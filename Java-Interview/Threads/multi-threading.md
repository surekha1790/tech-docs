### volatile vs atomic

### synchronized vs reentrant lock

### ReadWriteLock vs StampedLock

```ReentrantReadWriteLock``` allows multiple concurrent readers or one exclusive writer. It is reentrant and suitable for
general-purpose read/write synchronization. 
```StampedLock``` additionally supports optimistic reads, where a thread reads without acquiring a read lock and validates 
afterward whether a write occurred. This can improve performance in read-heavy workloads. However, StampedLock is not reentrant, 
requires careful stamp management, and is more complex to use correctly.

### How do AtomicInteger and CAS work?
```AtomicInteger``` provides thread-safe integer operations using CAS, which stands for Compare-And-Swap. CAS atomically compares
the current value with an expected value and updates it only if they match. If another thread changes the value first,
CAS fails and the operation can retry. This enables lock-free updates for common operations, such as incrementing a shared counter,
without using ```synchronized```.

### What is the ABA problem ?
The ABA problem occurs when a thread reads value A, another thread changes it to B and then back to A, and the first thread's CAS 
succeeds because the current value matches its expected value. However, the state may have changed in between.
We can prevent this using ```AtomicStampedReference```, which associates a version stamp with the reference, so CAS checks both the value
and its version.

### AtomicLong vs LongAdder?
Under high contention, LongAdder keeps separate counters per thread and sums them on read, which is much faster for counters and metrics.
Use AtomicLong when you need an exact value on every read or a compareAndSet.

### Deadlock
Deadlock occurs when threads wait indefinitely for locks held by one another. I prevent it primarily by maintaining a consistent lock ordering
across the application, so circular wait cannot occur. I also minimize nested locks, keep critical sections short, and use tryLock() with
a timeout when appropriate. In production, I use thread dumps and JVM monitoring to detect deadlocks.
1. Maintain a consistent lock ordering
2. Use tryLock() with a timeout
3. Avoid nested locks