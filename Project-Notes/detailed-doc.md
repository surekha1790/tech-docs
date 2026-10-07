

## 1. Quote flow (Market Maker side)

A **quote** is a two-sided price from one Market Maker for one instrument: a **bid** (I buy at X) and an **ask** (I sell at Y), sent together.

**Key rule:** each MM has only **one live quote per instrument**. A new quote does not add to the old one. It **replaces** it on both sides.

### Flow

1. MM sends the quote in FIX (Quote / Mass Quote) to the **OEGW**.
2. OEGW converts it to **SBE** and sends it to the **ME**.
3. ME removes the MM's old bid and ask, then inserts the new bid and ask.
4. If the new bid or ask **crosses** the other side, it trades right away.
5. ME sends an ack to OEGW (FIX back to the MM), the event to **Kafka**, and the new top-of-book to the **MDA → TREP → MDGW**.

### Example: quote replace

Turbo on DAX. Book before (MM1 is the issuer):

| Bids | Asks |
|---|---|
| MM1: 500 @ 9.98 | MM1: 500 @ 10.02 |
| Retail A: 100 @ 9.95 | Retail B: 200 @ 10.05 |

Top of book = **9.98 / 10.02**.

The DAX goes up, so MM1 sends a new quote: **bid 10.08 × 500, ask 10.12 × 500**.

The ME:

- removes MM1's 9.98 bid and 10.02 ask
- inserts bid 10.08, then checks it against the asks: the best ask is now Retail B at 10.05. **10.08 ≥ 10.05, so it crosses.** MM1 buys 200 @ 10.05 from Retail B (the resting order's price wins).
- the remaining bid of 300 @ 10.08 rests
- inserts ask 500 @ 10.12 (nothing to cross)

Book after:

| Bids | Asks |
|---|---|
| MM1: 300 @ 10.08 | MM1: 500 @ 10.12 |
| Retail A: 100 @ 9.95 | |

New top of book = **10.08 / 10.12**, which gets published to the market.

### Points to say in an interview

- A replace usually **loses time priority**, because the new quote goes to the back of its price level. Some venues keep priority if only the size goes down. Check what your ME does.
- Quotes update far more often than orders (the price follows the underlying all the time), so the replace path must be very cheap. That's why it uses an O(1) lookup by MM + instrument, then unlink and insert.
- On a **halt** (Circuit Breaker) or a **knock-out**, quotes are pulled or rejected.

## 2. Order flow (an order comes in and matches)

1. A broker sends a **New Order Single (35=D)** in FIX to the OEGW.
2. OEGW checks the session, converts to SBE, and sends it to the ME.
3. ME **validates** the order: is the instrument active, not halted, and is the price inside the band? If not, it is rejected.
4. ME **matches** using **price-time priority**: best opposite price first, and at the same price, the earliest order first.
5. Anything left over **rests** on the book, or is cancelled if it's IOC/FOK.
6. ME sends events to the OEGW (**Execution Report 35=8**) for both sides, to Kafka (OQS saves them to the DB), and trades to the MDA (the market sees the print).

### Example: order matching

Asks on the book:

| Asks | Time |
|---|---|
| MM1: 300 @ 10.12 | 1st |
| Retail C: 100 @ 10.12 | 2nd |
| MM2: 400 @ 10.15 | 3rd |

A retail **buy 350 @ 10.13 (limit)** comes in:

- The best ask is 10.12, and 10.12 ≤ 10.13, so it trades. MM1 came first, so the buyer gets **300 from MM1** (MM1 fully filled).
- 50 are still needed. Next in line at 10.12 is Retail C, so the buyer gets **50 from C**. C keeps 50 resting.
- The buy is done (300 + 50 = 350). MM2 at 10.15 is untouched because it is above the 10.13 limit.

Result: **2 trades at 10.12**, exec reports go to the buyer, MM1 and C, and the best ask is now **50 @ 10.12**.

If the buy had been **500 @ 10.13** instead: it fills 400 at 10.12, then the 100 left over **rests as a bid at 10.13**. That becomes the new best bid.

## Quote vs order in one line

- **Order**: one side, many per participant, each one stays until filled or cancelled.
- **Quote**: both sides, one per MM per instrument, and every new quote **replaces** the old one.

Both are matched in the same order book with the same price-time rules.

## Why is the Kafka offset saved with the snapshot, and how?

"The snapshot is a copy of the order book, and the offset tells us exactly which events are already inside it. The offset is the position in the ME's outgoing event topic (placed, filled, cancelled), stored per partition.

Each snapshot file header stores the partition, the last Kafka offset, and the engine's own sequence number. We write it to a temp file, fsync, then rename, so a half-written snapshot is never loaded.

On restart we load the latest snapshot, seek Kafka to that offset, and apply only events with a higher sequence number. That gives no duplicates, no gaps, and a fast restart, because we replay at most about 3 seconds of events instead of the whole day."

## How LLM can achieve low latency ?
LLM is broker less messaging system. It does not have intermediate brokers like Kafka and it directly publish the message to the receiver.

## How to handle failures in LLM
The industry standard is three levels. First, A/B redundant feeds, so most losses are covered by the other line. Second, retransmission: a NAK to the sender or a request to a replay server for recent gaps. Third, for old gaps or a late join, load a snapshot and rejoin the live stream from its sequence number. Sequence numbers detect gaps, heartbeats detect silence, and duplicates are dropped by sequence number.

#### A/B feeds:
The same messages are sent on two separate networks (A and B).
The receiver takes whichever copy arrives first and drops the second.
Most single-packet losses are fixed here, with no delay and no request.

## How do snapshots and reconciliation work in your system?

### **Snapshot**

"The matching engine keeps every order book in memory, so we take a snapshot of each book about every 3 seconds.

The matching thread is the single writer, so it takes the snapshot between two events. That way the snapshot reflects an exact point in the stream.

Each snapshot has a header with the Security ID, the Kafka partition, the last offset and the engine's sequence number, followed by the book data: every resting order with its ID, side, price, remaining quantity and time priority.

We write it to local disk in a dated folder, using temp file, fsync, then rename, so a half-written file is never loaded. We also publish it as a snapshot event to Kafka, in the same partition as that book's events.

On restart we load the latest snapshot, seek Kafka to its offset, and apply only events with a higher sequence number. So we replay at most a few seconds of events, with no gaps and no duplicates."

### **Reconciliation**

"Recovery assumes the data is correct, and reconciliation proves it. The engine trades from memory, so a bug or lost event could make memory, Kafka or the database drift apart without anything crashing.

A reconciliation consumer builds its own copy of each book by applying the engine's events from Kafka. Because the snapshot event sits in the same partition right after the event it covers, when the consumer reaches it, both books are at exactly the same sequence number.

**Then it compares in two steps:**

**Quick check:** order count, total quantity per side, best bid and ask, and a checksum of all orders sorted by Order ID. If they match, we're done. That's almost always the case.
Detailed diff: if not, we match orders by Order ID and list the ones that are missing on either side or have a different remaining quantity, price, state or priority.

We also compare against the OQS database, so the stored record matches the engine.

Kafka is the source of truth and the tie-breaker. If the database is behind, we replay the missing events into OQS, which is safe because it's idempotent by sequence number. If the engine's own state is wrong, which is rare and serious, we alert, halt the instrument and rebuild that book from Kafka.

It runs continuously during the day, after every recovery or failover, and at end of day across the engine, database and clearing."

## How ME tier partition works
Instruments are sharded by the last character of the Security ID, which maps to a tier. OEGW routes orders by Security ID, and each tier owns only its books. Every tier is its own small cluster with one leader that matches and emits, plus sync servers that keep identical books from the same ordered inputs. RCMS detects leader failure by missed heartbeats and elects the most up-to-date sync server by majority, using term numbers to prevent two leaders. Because matching is deterministic, failover has no divergence and no duplicate trades, and a failure is contained to one tier."

Leader dies → heartbeats stop → RCMS marks it dead
→ RCMS elects the most up-to-date sync server (majority vote, new term)
→ new leader catches up the last events → starts matching
→ OEGW is told the new leader → orders flow again

RCMS is the cluster manager for each ME tier. It tracks which nodes are alive using heartbeats, elects one leader per tier, preferring the most up-to-date standby with majority agreement, and tells every node and the gateways who the leader is. Term numbers and stepping down without a majority prevent two leaders. Because matching is deterministic, the new leader's book is identical, so failover has no lost or duplicate trades.
