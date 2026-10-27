# Virtual Threads, Scoped Values & Structured Concurrency

> Status as of Java 25 (LTS) / Java 26:
> **Virtual Threads**: final (Java 21) · **Scoped Values**: final (Java 25) · **Structured Concurrency**: still
> preview (`--enable-preview`)

---

## Project Loom

**Q: What is Project Loom?**
An OpenJDK project to make concurrency simple and scalable. Its goals are:

- Make blocking cheap
- Scale to millions of concurrent tasks
- Keep the simple "thread-per-request" style, with no reactive code needed

It delivered three features:

| Feature                | Version                     | Status            |
|------------------------|-----------------------------|-------------------|
| Virtual Threads        | 19 preview → **21 final**   | Final             |
| Scoped Values          | 20 incubator → **25 final** | Final             |
| Structured Concurrency | 19 incubator → 21+ preview  | Preview (Java 26) |

---

## Virtual Threads

### Problems with platform threads

**Q: Why are platform threads expensive?**

- Each platform thread maps **1:1 to an OS thread**.
- Each one reserves a large **fixed stack** (about 1 MB by default), so thousands of threads use a lot of memory.
- When a thread blocks on I/O, it **holds the OS thread** and does nothing useful.
- **Context switching** is costly: it goes into kernel mode, saves and restores registers, and disturbs CPU caches.
- The result is that we pool threads, and the pool size limits throughput (for example, 200 Tomcat threads means 200
  concurrent requests).

### What virtual threads are

**Q: What is a virtual thread?**
A lightweight thread **managed by the JVM**, not the OS.

- It costs only a few **KB** to create and **grows its stack dynamically**.
- Its stack frames live in **heap memory**.
- You can have **millions** of them.
- It is still `java.lang.Thread`, so existing APIs work.

**Q: How does a virtual thread work internally?**

- The JVM **mounts** a virtual thread on a **carrier thread**, which is a platform thread from a `ForkJoinPool`.
- When the virtual thread blocks (I/O, `sleep`, a lock, `BlockingQueue.take()`), the JVM:
    1. **Unmounts** it and copies its stack frames to the heap
    2. **Frees the carrier thread** to run another virtual thread
    3. **Remounts** it later, possibly on a *different* carrier, when the blocking operation completes
- This is why blocking is cheap: the OS thread is never wasted on waiting.

```
Virtual threads:   VT1  VT2  VT3  VT4 ... VT1,000,000
                     \    |    |   /
Carrier threads:     [ CT1 ] [ CT2 ] ... (≈ number of CPU cores)
                        |       |
OS threads:          [ OS1 ] [ OS2 ]
```

**Q: How many carrier threads are there?**
By default, as many as there are CPU cores. You can tune this with `-Djdk.virtualThreadScheduler.parallelism=N`.

### Creating virtual threads

```java
// 1. Start directly
Thread.startVirtualThread(() -> System.out.println("Hello"));

// 2. Builder (named threads: worker-0, worker-1, ...)
Thread t = Thread.ofVirtual().name("worker-", 0).start(() -> doWork());
t.join();

// 3. Executor (recommended): one new virtual thread per task
try(var executor = Executors.newVirtualThreadPerTaskExecutor())
{
    for(int i = 0;i< 10_000;i++) {
        int id = i;
         executor.submit(() ->{
             Thread.sleep(Duration.ofSeconds(1));   // blocking is fine
             return id;
       });
    }
}   // close() waits for all tasks to finish

// Check the thread type
Thread.currentThread().isVirtual();
```

> 10,000 tasks that each sleep 1 second finish in about **1 second** with virtual threads. A fixed pool of 200 platform
> threads takes about 50 seconds.

### When to use them

**Q: When should you use virtual threads?**

- **I/O-bound** work: REST calls, DB queries, file and network I/O, microservices
- High-concurrency servers using the thread-per-request model

**Q: When are they *not* helpful?**

- **CPU-bound** work (image processing, heavy calculation). Virtual threads are not faster; they only make *waiting*
  cheap. Use a platform thread pool or parallel streams for that.

### Pinning

**Q: What is pinning?**
A virtual thread is **pinned** when it can't unmount from its carrier while blocked, so the carrier is blocked too.

- **Before Java 24:** blocking inside a `synchronized` block or method pinned the thread, so people replaced
  `synchronized` with `ReentrantLock`.
- **Java 24+ (JEP 491):** `synchronized` **no longer pins**.
- **It still pins** when blocking inside **native code** (JNI or Foreign Function calls).
- You can detect it with the JFR event `jdk.VirtualThreadPinned`.

### Best practices and gotchas

**Q: Should you pool virtual threads?**
**No.** They are cheap, so create one per task. To **limit concurrency** (for example, a database that allows 20
connections), use a `Semaphore`:

```java
private static final Semaphore DB_LIMIT = new Semaphore(20);

void query() throws InterruptedException {
    DB_LIMIT.acquire();
    try {
        runQuery();
    } finally {
        DB_LIMIT.release();
    }
}
```
### What is Semaphore
* Semaphore is to limit number threads 
* Threads acquire before entering and release it after
* So, external api will not be overwhelmed with many virtual threads
* Once the semaphore limit is reached then threads will wait and enter only when it is free to enter.

**Q: Is `ThreadLocal` safe with virtual threads?**
It works, but with millions of threads each one holds its own copy, which uses a lot of memory and is easy to leak. *
*Prefer Scoped Values.**

**Other facts:**

- Virtual threads are always **daemon** threads with fixed `NORM_PRIORITY`. Calls to `setPriority()` and
  `setDaemon(false)` are ignored or throw.
- Don't use them for long CPU loops.
- Thread dump that includes virtual threads: `jcmd <pid> Thread.dump_to_file -format=json dump.json`

**Q: How do you enable them in Spring Boot (3.2+)?**

```properties
spring.threads.virtual.enabled=true
```

Tomcat, `@Async` and scheduled tasks then run on virtual threads.

### Platform vs virtual threads

|               | Platform Thread             | Virtual Thread            |
|---------------|-----------------------------|---------------------------|
| Managed by    | OS                          | JVM                       |
| Memory        | ~1 MB fixed stack           | Few KB, grows dynamically |
| Count         | Thousands                   | Millions                  |
| Blocking cost | Expensive (holds OS thread) | Cheap (unmounts)          |
| Pooling       | Yes                         | No, one per task          |
| Best for      | CPU-bound work              | I/O-bound work            |
| Daemon        | Configurable                | Always daemon             |

---

## Scoped Values (final in Java 25)

**Q: What is a Scoped Value?**
A way to share **immutable** data with code called inside a **bounded scope**. Typical data is the current user, a
request ID or a transaction context. It is a modern replacement for `ThreadLocal`.

**Q: What's wrong with `ThreadLocal`?**

- It is **mutable**: anyone can call `set()`, so data flow is hard to follow.
- It has **unbounded lifetime**: if you forget `remove()`, you get memory leaks, and pooled threads can leak data
  between requests.
- **Inheritance is expensive**: `InheritableThreadLocal` copies the data into every child thread, which is bad with
  millions of virtual threads.

**Q: How do you use Scoped Values?**

```java
public class RequestHandler {
    private static final ScopedValue<String> USER = ScopedValue.newInstance();

    void handle(String user) {
        ScopedValue.where(USER, user)
                .run(() -> service());   // USER is bound only inside this run()
        // USER is unbound again here, with no cleanup needed
    }

    void service() {
        System.out.println("User: " + USER.get());   // readable anywhere in the call chain
    }
}
```

**Return a value with `call()`:**

```java
String result = ScopedValue.where(USER, "admin").call(() -> fetchData());
```

**Bind several values:**

```java
ScopedValue.where(USER, "admin")
           .where(REQUEST_ID, "req-42")
           .run(() -> process());
```

**Rebind in a nested scope** (the outer value is restored afterwards):

```java
ScopedValue.where(USER, "admin").run(() ->{
        ScopedValue.where(USER, "guest")
                   .run(() ->USER.get());   // "guest"
        USER.get();                                               // "admin"
    });
```

**Check whether it is bound:**

```java
USER.isBound();             // false outside a scope
USER.orElse("anonymous");   // fallback (null is not allowed)
USER.get();                 // throws NoSuchElementException if unbound
```

**Q: How do Scoped Values work with child threads?**
Child tasks forked in a `StructuredTaskScope` **automatically inherit** the scoped values cheaply, with no copying.

### ThreadLocal vs ScopedValue

|                           | ThreadLocal                          | ScopedValue                            |
|---------------------------|--------------------------------------|----------------------------------------|
| Mutability                | Mutable (`set()`)                    | Immutable                              |
| Lifetime                  | Until `remove()` or thread death     | Only inside `run()`/`call()`           |
| Cleanup                   | Manual (leak risk)                   | Automatic                              |
| Child threads             | `InheritableThreadLocal` copies data | Shared cheaply via StructuredTaskScope |
| Cost with virtual threads | High                                 | Low                                    |

---

## Structured Concurrency (preview in Java 26)

**Q: What is Structured Concurrency?**
It treats a group of related concurrent tasks as **one unit of work**, with a parent-child relationship.

- It runs subtasks on **virtual threads** by default.
- If one subtask fails, the **others are cancelled automatically**.
- **Errors propagate** to the parent.
- **No thread leaks**: all subtasks finish before the scope closes.
- Thread dumps show the parent-child structure.

**Q: How do you use it?** (Java 25+ API; the old `ShutdownOnFailure` class was removed)

```java
Response handle() throws InterruptedException {
    try (var scope = StructuredTaskScope.open()) {        // default: all must succeed
        Subtask<User> user = scope.fork(() -> findUser());
        Subtask<Order> order = scope.fork(() -> findOrder());

        scope.join();   // waits for both; throws FailedException if either fails
        // (and the other is cancelled)

        return new Response(user.get(), order.get());
    }
}
```

**Q: Can you change the success or failure policy?**
Yes. Pass a `Joiner` to `StructuredTaskScope.open(joiner)`. For example, you can make the first successful result win,
or collect all results. The exact Joiner methods may still change while the API is in preview.

**Q: Why not just use `ExecutorService` + `Future`?**
With an executor, if `findUser()` fails, `findOrder()` **keeps running** and wastes resources. You have to cancel it
manually, and threads can leak. Structured concurrency does this cleanup for you.

---

## Quick Interview Answers

1. **Virtual vs platform thread?** Virtual threads are JVM-managed, cheap and number in the millions. They suit
   I/O-bound work.
2. **Are virtual threads faster?** No. They give more **throughput** for blocking work, not faster CPU execution.
3. **What is a carrier thread?** A platform thread from a ForkJoinPool that runs mounted virtual threads.
4. **What is pinning?** A virtual thread stuck on its carrier while blocking. Since Java 24, it happens only in native
   calls, not `synchronized`.
5. **Should you pool virtual threads?** No. Use a `Semaphore` to limit concurrency.
6. **ScopedValue vs ThreadLocal?** ScopedValue is immutable, bounded, needs no cleanup and is cheap with virtual
   threads.
7. **Is Structured Concurrency final?** Not yet. It is still preview in Java 26.
8. **Virtual threads in Spring Boot?** Set `spring.threads.virtual.enabled=true`.

### Use case in E-commerce application

* Product details page should call multiple services to get the information related to product such inventory, price, reviews
* Here, create a new Backend-to-frontend service and call all the required services using virtual threads.
