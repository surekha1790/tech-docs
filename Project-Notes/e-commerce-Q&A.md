## Senior Java Developer — Interview Q&A
### E-commerce microservices project — questions **with answers**, plus a full idempotency map

Answers are written the way you'd say them in an interview: **what → why → trade-off**, grounded in this project.

---

# PART 1 — Idempotency in this project (complete map)

> This is the topic interviewers on this system will push hardest on, because it's what makes at-least-once Kafka safe. Here is **every** place we handle it and how.

### Summary table

| # | Where | Trigger it protects against | Mechanism | Key | Storage |
|---|---|---|---|---|---|
| 1 | **Order creation** (order-service) | Double-click / client retry creating duplicate orders | Insert-first behind a UNIQUE constraint; replay stored order | client `Idempotency-Key` header | `order_idempotency_keys` (V12) |
| 2 | **Stock restore on cancel** (inventory-service) | Duplicate `order.cancelled` restoring stock twice | Dedup ledger row written in the SAME tx as the restore | `stock-restore:<orderId>` | `processed_inventory_events` (V6) |
| 3 | **Wallet credit/debit** (customer-service) | Duplicate wallet events double-crediting/debiting | `existsByReferenceId` check before applying | `ORDER-WALLET-<orderId>`, `REFUND-<refundId>` | `wallet_transactions.reference_id` |
| 4 | **Stock deduct/restore** (inventory-service) | Concurrent orders overselling (not duplicate delivery) | Atomic conditional `UPDATE … WHERE quantity >= :qty` | product row | `inventory_items` |
| 5 | **product.created consumer** (inventory-service) | Duplicate `product.created` creating two rows | `existsByProductId` skip | productId | `inventory_items` |
| 6 | **user.registered consumer** (customer-service) | Duplicate registration events | `existsByAuthUserId` skip | authUserId | `customers` |
| 7 | **order.fulfilled consumer** (inventory-service) | Duplicate fulfilment events | No-op (stock already deducted at placement) | — | — |
| 8 | **Kafka producer** (all except auth) | Producer retries duplicating a message on a partition | `acks=all` → idempotent producer (implicit, Kafka 3.x) | producer id + sequence | broker |

### Detail per mechanism

**1. Order creation — `OrderIdempotencyService`**
The client (customer-ui) generates one `crypto.randomUUID()` per checkout and sends it as the `Idempotency-Key` header (reused on every retry). The service does **insert-first**: `saveAndFlush` a row in `order_idempotency_keys` guarded by the UNIQUE constraint `uq_oik_key`. First request → creates the order, stores `orderId` on the row. Retry with the same key → returns the existing order (replay); a concurrent duplicate loses the insert race and is resolved against the winner. *Why insert-first vs check-then-insert:* the UNIQUE constraint makes it atomic — a plain check-then-insert is a TOCTOU race two requests can both pass.

**2. Stock restore on cancel — `ProcessedInventoryEvent` ledger**
Kafka is at-least-once, so `order.cancelled` can arrive twice. Before restoring, the consumer checks `existsByEventKey("stock-restore:"+orderId)` as a fast path, does the restore, then **writes the ledger row in the same transaction**. The UNIQUE constraint `uq_pie_event_key` means a concurrent duplicate's transaction (restore + marker) fails atomically and rolls back — so the restore is applied **exactly once**. *The elegance:* bundling the marker insert with the stock change lets the DB's unique constraint enforce exactly-once, even under a race.

**3. Wallet credit/debit — `existsByReferenceId`**
`WalletServiceImpl.credit/debit` skip if a `wallet_transaction` with that `referenceId` already exists. Reference IDs are deterministic per business action: `ORDER-WALLET-<orderId>` (wallet used at checkout, published by order-service) and `REFUND-<refundId>` (refund credit, published by payment-service). So a redelivered event with the same referenceId is ignored — no double debit/credit.

**4. Stock deduct/restore — atomic conditional UPDATE**
`deductStock` runs `UPDATE inventory_items SET quantity = quantity - :qty WHERE product_id = :id AND quantity >= :qty`. This is a **concurrency guard**, not idempotency: it prevents overselling under simultaneous orders (the check-then-act race), returning 0 rows if stock is insufficient. Note the distinction — deduction itself isn't idempotent, but it's driven by a *synchronous* HTTP reserve call (not an at-least-once Kafka event), so duplicate delivery isn't the risk here; concurrency is.

**5–7. Naturally idempotent consumers**
`product.created` checks `existsByProductId` before creating; `product.updated`/`deleted` just set fields (idempotent by nature); `user.registered` checks `existsByAuthUserId`; `order.fulfilled` is a deliberate no-op because stock was already deducted at placement. These need no ledger because re-applying them has no additional effect.

**8. Kafka producer idempotence**
With `acks=all` and retries, the Kafka 3.x client enables the idempotent producer by default (producer id + per-partition sequence numbers), so a producer retry can't write the same record twice within a partition. auth-service is the exception (`acks=1`), so it doesn't get this.

### The consistent pattern
For anything that **mutates state from an at-least-once event**, we use a **dedup key + unique constraint, written in the same transaction as the mutation**. That turns "at-least-once delivery" into "exactly-once effect." Concurrency (not delivery) is handled separately by atomic conditional updates.

---

# PART 2 — Q&A by theme

## Architecture

**Q: Walk me through the architecture.**
A single API gateway fronts ~14 Spring Boot services (auth, order, payment, inventory, shipping, product, customer…), each with its **own Postgres database**. Services talk **synchronously via REST/Feign** when an immediate answer is needed (order → inventory stock check) and **asynchronously via Kafka** when they can be decoupled (order.placed → shipping/customer). Config is centralized in Spring Cloud Config. The trade-off we accepted: independent scaling and fault isolation in exchange for distributed-transaction complexity and eventual consistency.

**Q: Why database-per-service?**
Loose coupling — a service owns and can evolve its schema independently, and one DB failing doesn't take down others. The cost is no cross-service joins and no ACID across services, which is exactly why we use the saga pattern for multi-service operations.

**Q: What does the gateway do?**
It's the single entry point: routing, **JWT validation once at the edge**, injecting `X-User-*` headers so downstream services don't parse tokens, CORS, and audience-based authorization (`/api/admin/**` requires an admin-audience token). Validating auth once keeps every service simple.

## Kafka & event-driven

**Q: Which delivery semantic do you use and why?**
**At-least-once** (Spring Kafka commits the offset *after* the listener succeeds, via `AckMode.BATCH`). At-most-once risks loss; exactly-once is only Kafka-internal and can't cover our DB/HTTP side effects. So we pair at-least-once with **idempotent consumers** → "effectively once."

**Q: If offsets aren't auto-committed, how are they committed?**
The Spring Kafka container commits them for us via `commitSync` after the poll batch is processed — not Kafka's timer-based auto-commit. That "process then commit" ordering is what makes it at-least-once.

**Q: A consumer crashes after processing but before commit — what happens?**
Offsets are cumulative positions. On restart it resumes from the last committed offset and reprocesses everything after it, in order — no message lost, but the already-processed one is redelivered. That's the duplicate our idempotency handles.

**Q: A message keeps failing — where does it go?**
Our `DefaultErrorHandler` retries 3× with backoff, then `DeadLetterPublishingRecoverer` publishes it to `<topic>.DLT`, then the offset commits and the partition moves on. Crucially, the listener must **let the exception propagate** — we removed the `try/catch → log` swallows so failures actually reach the handler.

**Q: How do you keep ordering?**
Ordering is per-partition only, so we key events by `orderId` (all of an order's events land on one partition, processed in order) and keep the idempotent producer with `max.in.flight ≤ 5` so retries can't reorder.

## Distributed transactions / saga

**Q: You can't do one ACID transaction across order/payment/inventory. How do you stay consistent?**
The **saga pattern**: a sequence of local transactions, each with a **compensating transaction**. On payment failure we compensate in reverse — release inventory, cancel the order (which sits in `PENDING` as a semantic lock). It's eventual consistency, coordinated by events.

**Q: Why reserve inventory before payment?**
Releasing a hold is instant and free; refunding a payment costs fees and trust. Secure the scarce, hard-to-undo resource first, and it also closes the check-then-act race via the atomic guarded update.

**Q: Why not 2PC?**
Two-phase commit locks resources across services for the whole transaction, doesn't scale, and stalls if the coordinator dies. Sagas trade isolation for availability and scalability.

## Resilience

**Q: How do you prevent duplicate orders?**
Idempotency key at the API (Part 1 #1): client sends a stable `Idempotency-Key`, the server inserts-first behind a unique constraint, retries replay the original order. Client button-disabling is a nicety; the unique constraint is the guarantee.

**Q: How does `@Transactional` work, and what's the self-invocation trap?**
It's proxy-based AOP (CGLIB/JDK) — a proxy begins/commits/rolls-back around the method. A `this.method()` self-call bypasses the proxy, so the annotation is ignored. That's why `OrderIdempotencyService` calls the injected `OrderService` **proxy**, not a self method, so `placeOrder`'s transaction still applies.

**Q: Default rollback rules?**
Rolls back on unchecked exceptions and Errors only; **checked exceptions commit** unless you specify `rollbackFor`. Catching and swallowing an exception also commits — a trap.

## Security

**Q: Walk me through auth.**
auth-service issues a short-lived HS256 **access token** (claims: sub, email, roles, userType, scoped audience) plus a DB-stored **refresh token**. The gateway validates the access token once and forwards `X-User-*` headers. Services trust the headers.

**Q: Why two tokens?**
A stateless JWT can't be revoked, so the access token is short-lived to limit damage. The refresh token lives in the DB so it *can* be revoked (logout/password change) and rotated — giving a long session with revocation control, without a DB lookup on every request.

**Q: The gateway forwards `X-User-Id` and services trust it — risk?**
If a service is directly reachable, a caller could spoof `X-User-Id`. Mitigation: network-isolate services so only the gateway can reach them (private network / mTLS). This is the key operational control for this pattern.

**Q: How do you stop a customer token hitting admin APIs?**
Audience scoping: CUSTOMER tokens get `ecommerce-customer-api`, admin/B2B get `ecommerce-admin-api`, and the gateway requires the admin audience on `/api/admin/**` (403 otherwise) — defense-in-depth on top of role checks.

## Database & performance

**Q: How do you manage connection pools across many services?**
HikariCP is auto-configured; the real risk is the **sum of pools across all services and instances vs each Postgres server's `max_connections`** — size deliberately or use PgBouncer. We also disabled Open-Session-In-View so connections aren't held through response rendering.

**Q: How does inventory avoid overselling?**
An atomic compare-and-decrement: `UPDATE … SET quantity = quantity - :qty WHERE quantity >= :qty`. One concurrent order wins; the other gets 0 rows updated → insufficient-stock. No pessimistic locks needed.

## Java & Spring fundamentals (they'll still ask)

**Q: JVM memory areas?** Objects on the heap, references/frames on per-thread stacks, class metadata in **Metaspace** (off-heap, replaced PermGen in Java 8).

**Q: HashMap vs ConcurrentHashMap — why is CHM faster?** Per-bucket locking + lock-free reads vs one global mutex; CHM scales with cores.

**Q: `map` vs `flatMap`?** `map` is 1:1; `flatMap` transforms each element into a stream and flattens one level. Streams are lazy and fuse into a single pass.

**Q: Constructor vs field injection?** Constructor: immutability (`final`), guaranteed dependencies, testable without reflection, and it surfaces "too many dependencies" as a visible smell.

## Scenario questions (highest signal)

**Q: Trace an order end to end.**
Gateway authenticates → order-service validates → **synchronous** inventory deduct (blocks the order on failure to avoid overselling) → save order → publish `order.placed` → payment (wallet debit) → shipping/customer consume events. On payment failure, `order.cancelled` triggers the compensating stock restore (idempotent).

**Q: `order.cancelled` is delivered twice — is stock restored twice?**
No. The `processed_inventory_events` ledger keyed by `stock-restore:<orderId>`, written in the same transaction as the restore, makes it exactly-once via the unique constraint.

**Q: A service is down 2 hours then recovers — lost events? Duplicates?**
No loss — Kafka retains messages and `auto.offset.reset=earliest` + at-least-once redelivery replays from the last commit; duplicates are absorbed by idempotency. (If it were down > 7 days, its committed offsets could expire and it'd replay from the log start — still safe because idempotent.)

## Judgment / "what would you improve"

**Q: Biggest weakness and fix?**
Events are untyped `Map<String,Object>` with **no schema**, so producer/consumer field names can drift silently (we actually found `customerId` vs `authUserId` and `threshold` vs `lowStockThreshold` bugs). Fix: a schema registry + Avro/Protobuf or a shared event-DTO module + contract tests. Other gaps: no distributed tracing, deduct-on-order without a TTL for abandoned carts, and the `X-User-Id` trust needing network isolation.

**Q: 100× traffic — what breaks first?**
The database. Response: cache (Redis/Caffeine), read replicas, then sharding; make more work async through Kafka; add partitions for consumer parallelism.

---

### Interview strategy for this project
Interviewers will camp on **idempotency + saga + the order flow + Kafka semantics + gateway auth**. Lead with concrete stories from this codebase — e.g. *"we had a silent bug where a duplicate `order.cancelled` could restore stock twice; we fixed it with a dedup ledger keyed by orderId written in the same transaction as the restore, so the unique constraint enforces exactly-once."* A real war story beats a textbook definition every time.
