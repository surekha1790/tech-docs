# CompletableFuture — A Complete Beginner's Guide

---

## 1. What problem does it solve?

A `CompletableFuture<T>` represents **a value that isn't ready yet but will be** — the result of some work happening (usually) on another thread. Instead of blocking and waiting for that work, you attach **callbacks** that run automatically when the value arrives. This lets you:

- Run slow operations (network calls, DB queries, file I/O) **without freezing** the calling thread.
- **Compose** multiple async steps into a readable pipeline instead of nested callbacks ("callback hell").
- **Fan out** many tasks in parallel and combine their results.
- Handle **errors** and **timeouts** declaratively.

Think of it as a promise: *"Here's a handle to a result. When it's done, do X with it; if it fails, do Y."*

---

## 2. Future vs CompletableFuture

Java 5's `Future` could represent an async result, but it was crippled: the only way to read it was `get()`, which **blocks**. You couldn't chain, combine, or react to completion.

| Capability | `Future` (Java 5) | `CompletableFuture` (Java 8+) |
|---|---|---|
| Get result | `get()` — blocks | `get()`, `join()`, or non-blocking callbacks |
| Chain transformations | No | `thenApply`, `thenCompose`, … |
| Combine multiple | No | `thenCombine`, `allOf`, `anyOf` |
| Complete manually | No | `complete(value)` |
| Handle errors | try/catch around `get()` | `exceptionally`, `handle` |
| Timeouts | `get(timeout)` only | `orTimeout`, `completeOnTimeout` |

`CompletableFuture` implements both `Future` and `CompletionStage` (the interface that defines all the `then*` composition methods).

---

## 3. Creating a CompletableFuture

### `supplyAsync` — run work that returns a value

The most common entry point. Runs the supplier on a thread pool and gives you a future for its result.

```java
CompletableFuture<Integer> cf =
    CompletableFuture.supplyAsync(() -> expensiveCalculation());
```

### `runAsync` — run work that returns nothing

For side-effect-only tasks (logging, sending an event). Returns `CompletableFuture<Void>`.

```java
CompletableFuture<Void> cf =
    CompletableFuture.runAsync(() -> System.out.println("background task"));
```

### `completedFuture` — an already-finished future

Handy for tests, defaults, or returning a value through an async API without doing async work.

```java
CompletableFuture<String> cf = CompletableFuture.completedFuture("ready");
System.out.println(cf.join());   // ready
```

### `new CompletableFuture<>()` + `complete(...)` — fill it in yourself

Create an empty future and complete it later from any thread (e.g. from a callback-based library or an event listener).

```java
CompletableFuture<String> cf = new CompletableFuture<>();
// ... somewhere else, when the value is ready:
cf.complete("filled");
System.out.println(cf.join());   // filled
```

Related manual-completion methods: `completeExceptionally(throwable)` to fail it, and `completeAsync(supplier, executor)` to complete it from a supplier.

> **Verified output:** `completedFuture` → `ready`, manual `complete` → `filled`.

---

## 4. Getting the result out

Usually you should **not** block — you attach callbacks (Sections 5+). But when you must extract the value:

### `join()` — block and return the value (unchecked)

Throws an **unchecked** `CompletionException` on failure. Preferred inside streams and lambdas because it needs no `try/catch`.

```java
int v = cf.join();
```

### `get()` — block and return the value (checked)

Throws **checked** `InterruptedException` and `ExecutionException`, so you must handle them. Supports a timeout overload: `get(500, TimeUnit.MILLISECONDS)`.

```java
try {
    int v = cf.get(500, TimeUnit.MILLISECONDS);
} catch (InterruptedException | ExecutionException | TimeoutException e) { /* ... */ }
```

### `getNow(fallback)` — don't block; return a fallback if not ready

```java
CompletableFuture<Integer> slow = CompletableFuture.supplyAsync(() -> { sleep(200); return 1; });
System.out.println(slow.getNow(-99));   // -99  (wasn't done yet)
```

### `isDone()`, `isCompletedExceptionally()`, `isCancelled()`

Non-blocking status checks.

> **Rule of thumb:** use `join()`/`get()` only at the *very end* of a pipeline, or never — prefer callbacks.

---

## 5. Transforming a result

These three take the result of a stage and do something with it. They differ by whether the callback **uses** the input and whether it **produces** an output.

### `thenApply` — transform the value (`T -> R`)

The workhorse. Like `map` on a stream.

```java
CompletableFuture.supplyAsync(() -> 10)
    .thenApply(v -> v * 2)     // 10 -> 20
    .thenApply(v -> "result=" + v);   // 20 -> "result=20"
```

### `thenAccept` — consume the value, return nothing (`T -> void`)

Use it as a terminal step that does something with the result (print, save) but passes nothing on. Returns `CompletableFuture<Void>`.

```java
CompletableFuture.supplyAsync(() -> 10)
    .thenAccept(v -> System.out.println("got " + v));   // got 10
```

### `thenRun` — run an action, ignore the value (`() -> void`)

For "when it's done, do this" where you don't care about the actual result.

```java
CompletableFuture.supplyAsync(() -> 10)
    .thenRun(() -> System.out.println("done"));   // done
```

**Use case:** `thenApply` to convert a DB entity to a DTO; `thenAccept` to write the response; `thenRun` to log "request complete."

---

## 6. Chaining dependent async work: `thenCompose`

Use `thenCompose` when your next step is **itself asynchronous** and depends on the previous result. Its callback returns a `CompletableFuture`, and `thenCompose` **flattens** it — so you get `CompletableFuture<R>`, not `CompletableFuture<CompletableFuture<R>>`.

```java
CompletableFuture<Integer> result =
    CompletableFuture.supplyAsync(() -> 10)
        .thenCompose(v -> CompletableFuture.supplyAsync(() -> v + 5));   // 15
```

**`thenApply` vs `thenCompose`** — this is the key distinction:

- `thenApply(fn)` — `fn` returns a **plain value**. (like `Stream.map`)
- `thenCompose(fn)` — `fn` returns **another future**. (like `Stream.flatMap`)

If you use `thenApply` where the callback returns a future, you end up with an awkward nested `CompletableFuture<CompletableFuture<R>>`. Reach for `thenCompose`.

**Use case:** `getUserId()` returns a future; then `fetchProfile(userId)` returns another future. Chain them with `thenCompose` so step 2 starts only after step 1's ID arrives.

---

## 7. Combining two futures

When two **independent** futures run in parallel and you want to merge their results once **both** finish.

### `thenCombine` — merge two results into a value (`(a, b) -> R`)

```java
var price = CompletableFuture.supplyAsync(() -> 100);
var tax   = CompletableFuture.supplyAsync(() -> 20);
int total = price.thenCombine(tax, (p, t) -> p + t).join();   // 120
```

### `thenAcceptBoth` — consume both results, return nothing (`(a, b) -> void`)

```java
price.thenAcceptBoth(tax, (p, t) -> System.out.println(p + "+" + t));
```

### `runAfterBoth` — run an action after both complete, ignore both results

```java
price.runAfterBoth(tax, () -> System.out.println("both done"));
```

**Use case:** fetch a product's price from one service and its stock from another **in parallel**, then `thenCombine` them into a single "product view."

---

## 8. Racing two futures

When you have two futures and only care about **whichever finishes first**.

### `applyToEither` — take the first result, transform it (`T -> R`)

```java
var fast = CompletableFuture.supplyAsync(() -> { sleep(30);  return "fast"; });
var slow = CompletableFuture.supplyAsync(() -> { sleep(200); return "slow"; });
String winner = fast.applyToEither(slow, s -> s.toUpperCase()).join();   // FAST
```

### `acceptEither` — consume the first result (`T -> void`)

```java
fast.acceptEither(slow, s -> System.out.println("first: " + s));
```

### `runAfterEither` — run an action after either completes

```java
fast.runAfterEither(slow, () -> System.out.println("one finished"));
```

**Use case:** query two redundant mirrors/replicas and use whichever responds first, to cut tail latency.

---

## 9. Combining many futures: `allOf`, `anyOf`

### `allOf(cf1, cf2, ...)` — wait for **all** to complete

Returns `CompletableFuture<Void>` — it signals completion but **does not carry the results**. To get the values, join each future afterward (they're already done, so it's instant).

```java
List<CompletableFuture<Integer>> futures = IntStream.rangeClosed(1, 5)
    .mapToObj(i -> CompletableFuture.supplyAsync(() -> i * i))
    .toList();

int sum = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream()
        .map(CompletableFuture::join)          // safe: all already complete
        .mapToInt(Integer::intValue).sum())
    .join();                                    // 55
```

**Reusable "allOf that returns results" helper:**

```java
static <T> CompletableFuture<List<T>> allResults(List<CompletableFuture<T>> fs) {
    return CompletableFuture.allOf(fs.toArray(new CompletableFuture[0]))
        .thenApply(v -> fs.stream().map(CompletableFuture::join).toList());
}
```

> Because `allOf` returns `Void`, the per-future `join` is the only way to retrieve values — but it's required *only when you need the results*. For pure side-effect tasks, `allOf(...).join()` alone is enough.

### `anyOf(cf1, cf2, ...)` — wait for the **first** to complete

Returns `CompletableFuture<Object>` with the first result (type-erased, since inputs may differ).

```java
var f1 = CompletableFuture.supplyAsync(() -> { sleep(150); return "slow"; });
var f2 = CompletableFuture.supplyAsync(() -> { sleep(40);  return "fast"; });
System.out.println(CompletableFuture.anyOf(f1, f2).join());   // fast
```

**Use case:** `allOf` to wait for a batch of enrichment calls before rendering a page; `anyOf` for "first responder wins" or a manual timeout race.

---

## 10. Error handling

If a stage throws, the exception propagates down the chain (wrapped in `CompletionException`) and later `then*` stages are **skipped** until a handler catches it.

### `exceptionally` — recover with a fallback value (runs only on failure)

```java
int v = CompletableFuture.<Integer>supplyAsync(() -> { throw new RuntimeException("boom"); })
    .exceptionally(ex -> -1)      // recover
    .join();                       // -1
```

### `handle` — see result **or** exception, always runs (`(res, err) -> R`)

```java
String out = CompletableFuture.<Integer>supplyAsync(() -> { throw new RuntimeException("x"); })
    .handle((res, err) -> err != null ? "recovered" : "ok:" + res)
    .join();                       // recovered
```

### `whenComplete` — peek at result/exception **without changing** it (`(res, err) -> void`)

Great for logging or cleanup. It does **not** transform the result — the original value (or exception) flows through unchanged.

```java
CompletableFuture.supplyAsync(() -> 42)
    .whenComplete((res, err) -> System.out.println("res=" + res + " err=" + err))
    .join();                       // prints res=42 err=null, returns 42
```

### `exceptionallyCompose` (Java 12+) — recover with **another future**

Like `exceptionally`, but the recovery is itself async.

```java
int v = CompletableFuture.<Integer>supplyAsync(() -> { throw new RuntimeException("fail"); })
    .exceptionallyCompose(ex -> CompletableFuture.supplyAsync(() -> 7))
    .join();                       // 7
```

**Which to use:**

| Method | Runs on success? | Runs on failure? | Can change result? | Returns |
|---|:---:|:---:|:---:|---|
| `exceptionally` | No | Yes | Yes (fallback) | value |
| `handle` | Yes | Yes | Yes | value |
| `whenComplete` | Yes | Yes | No (peek only) | same value |

> **Verified output:** `exceptionally` → `-1`, `handle` → `recovered`, `whenComplete` → `res=42 err=null`, `exceptionallyCompose` → `7`.

---

## 11. Timeouts (Java 9+)

### `orTimeout(t, unit)` — fail the future if it takes too long

Completes exceptionally with `TimeoutException` after the deadline.

```java
CompletableFuture.supplyAsync(() -> { sleep(500); return "late"; })
    .orTimeout(100, TimeUnit.MILLISECONDS)
    .join();   // throws CompletionException caused by TimeoutException
```

### `completeOnTimeout(fallback, t, unit)` — supply a default if too slow

Completes **normally** with the fallback instead of failing.

```java
String v = CompletableFuture.supplyAsync(() -> { sleep(500); return "late"; })
    .completeOnTimeout("default", 100, TimeUnit.MILLISECONDS)
    .join();   // "default"
```

**Use case:** cap how long you'll wait on a downstream service — either fail fast (`orTimeout`) or degrade gracefully with a default (`completeOnTimeout`).

> **Verified output:** `orTimeout` → `TimeoutException`, `completeOnTimeout` → `default`.

---

## 12. The `Async` suffix and the threading model

**Every** composition method has three forms:

```java
thenApply(fn)            // runs fn on the thread that completed the previous stage
thenApplyAsync(fn)       // runs fn on the default pool (ForkJoinPool.commonPool)
thenApplyAsync(fn, ex)   // runs fn on the executor YOU provide
```

- **No suffix:** the callback runs on whatever thread completed the previous stage (or the caller's thread if already done). Cheap, but a heavy callback can hog that thread.
- **`Async` (no executor):** reschedules onto the common `ForkJoinPool`. Good for CPU-bound work.
- **`Async` with executor:** runs on your pool. **Always pass your own executor for blocking I/O** — don't starve the common pool.

```java
ExecutorService ex = Executors.newFixedThreadPool(4);
CompletableFuture.supplyAsync(() -> load(), ex)      // I/O on your pool
    .thenApplyAsync(data -> transform(data), ex);    // keep it off commonPool

// remember to shut it down when done:
ex.shutdown();
```

> **Note on virtual threads (Java 21):** for blocking I/O, `Executors.newVirtualThreadPerTaskExecutor()` is an excellent executor to pass here — each task gets a cheap virtual thread and blocking no longer wastes an OS thread.

---

## 13. Naming pattern — decode any method name

Once you see the system, you can guess any method's behavior:

**Action type (what your callback does):**

- **`Apply`** → takes input, **returns a value** (`Function`)
- **`Accept`** → takes input, **returns nothing** (`Consumer`)
- **`Run`** → takes **no** input, returns nothing (`Runnable`)

**Arity / combinator (how many futures):**

- *(no suffix)* → operates on **one** future
- **`Compose`** → chains a **dependent** future (flattens)
- **`Combine` / `Both`** → needs **two** futures, waits for **both**
- **`Either`** → needs **two** futures, takes the **first**
- **`allOf` / `anyOf`** → **many** futures

**Threading:** add **`Async`** to run the callback on another thread (optionally your executor).

So `thenAcceptBothAsync` = after **both** futures finish, **consume** both results (no return), on a **separate** thread. You can now read any method name without memorizing it.

---

## 14. Real-world use cases

**Parallel service aggregation.** A product page needs price, inventory, and reviews from three services. Fire all three with `supplyAsync`, then `allOf(...).thenApply(...)` to assemble one response — total latency is the *slowest* call, not the *sum*.

```java
var price     = CompletableFuture.supplyAsync(() -> priceService.get(id), ex);
var inventory = CompletableFuture.supplyAsync(() -> stockService.get(id), ex);
var reviews   = CompletableFuture.supplyAsync(() -> reviewService.get(id), ex);

ProductView view = CompletableFuture.allOf(price, inventory, reviews)
    .thenApply(v -> new ProductView(price.join(), inventory.join(), reviews.join()))
    .join();
```

**Dependent pipeline.** Authenticate → fetch profile → fetch orders, each async and dependent: chain with `thenCompose`.

**Resilient calls.** Wrap a downstream call with `completeOnTimeout(defaultValue, ...)` and `exceptionally(...)` so a slow or failing dependency degrades gracefully instead of taking down the request.

**Fastest-of-N.** Query two replicas and `applyToEither` to use whichever answers first.

**Fire-and-forget.** Use `runAsync` for audit logging or metrics you don't need to wait on.

---

## 15. Common pitfalls & best practices

**Don't block in the middle of a chain.** Calling `join()`/`get()` between stages defeats the purpose. Compose with callbacks and block (if at all) only at the end.

**Always supply your own executor for blocking I/O.** The default `ForkJoinPool.commonPool` is sized for CPU work and shared JVM-wide; blocking it starves everything else.

**`allOf` returns `Void`.** It won't hand you results — join each future afterward (they're already done) or use the `allResults` helper.

**Prefer `join()` inside lambdas/streams.** It throws unchecked exceptions, so it composes cleanly where `get()` forces `try/catch`.

**Exceptions are wrapped.** A failure surfaces as `CompletionException` (from `join`) or `ExecutionException` (from `get`); unwrap with `.getCause()` to see the real error.

**`thenApply` vs `thenCompose`.** If your callback returns a future, use `thenCompose` to avoid nesting.

**Remember to shut down executors** (`ex.shutdown()`), or the JVM may not exit.

**An unhandled exception is silent** until someone reads the future. Add `exceptionally`/`handle`, or at least `whenComplete` to log it.

---

## 16. Quick reference table

| Method | Callback shape | Waits for | Returns | Use when |
|---|---|---|---|---|
| `supplyAsync` | `() -> T` | — | `CF<T>` | start async work with a result |
| `runAsync` | `() -> void` | — | `CF<Void>` | start async side-effect work |
| `completedFuture` | — | — | `CF<T>` | already-known value |
| `complete` | — | — | `boolean` | fill an empty future manually |
| `join` / `get` | — | this | `T` | block for the value (end of chain) |
| `getNow` | — | — | `T` | value now or fallback, no blocking |
| `thenApply` | `T -> R` | this | `CF<R>` | transform the result |
| `thenAccept` | `T -> void` | this | `CF<Void>` | consume the result |
| `thenRun` | `() -> void` | this | `CF<Void>` | run action, ignore result |
| `thenCompose` | `T -> CF<R>` | this + inner | `CF<R>` | chain a dependent future |
| `thenCombine` | `(T,U) -> R` | both | `CF<R>` | merge two parallel results |
| `thenAcceptBoth` | `(T,U) -> void` | both | `CF<Void>` | consume two results |
| `runAfterBoth` | `() -> void` | both | `CF<Void>` | act after both finish |
| `applyToEither` | `T -> R` | first | `CF<R>` | transform the fastest result |
| `acceptEither` | `T -> void` | first | `CF<Void>` | consume the fastest result |
| `runAfterEither` | `() -> void` | first | `CF<Void>` | act after either finishes |
| `allOf` | — | all | `CF<Void>` | barrier over many futures |
| `anyOf` | — | first | `CF<Object>` | first-of-many wins |
| `exceptionally` | `Throwable -> T` | this (on error) | `CF<T>` | fallback value on failure |
| `handle` | `(T,Throwable) -> R` | this | `CF<R>` | handle success or failure |
| `whenComplete` | `(T,Throwable) -> void` | this | `CF<T>` | peek/log without changing result |
| `exceptionallyCompose` | `Throwable -> CF<T>` | this (on error) | `CF<T>` | async recovery |
| `orTimeout` | — | this / deadline | `CF<T>` | fail if too slow |
| `completeOnTimeout` | — | this / deadline | `CF<T>` | default if too slow |

*Every `then*`, `applyToEither`, `acceptEither`, `runAfter*`, and `handle`/`whenComplete` method also has `...Async` variants that run the callback on another thread (optionally a supplied executor).*

---

*All examples verified on OpenJDK 21. `CF<T>` is shorthand for `CompletableFuture<T>`. `sleep(ms)` is a helper wrapping `Thread.sleep`.*
