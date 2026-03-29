# GC Knowledge Check

1. #### What is Garbage Collection (GC), and why is it important?

Answer: Garbage Collection is an automatic memory management process that reclaims memory occupied by objects no longer in use. It prevents memory leaks and dangling pointers, improving application stability and security.

2. #### Why is GC a concern in low-latency applications?

Answer: GC can introduce pauses (Stop-the-World events), which block application threads. In low-latency applications, even millisecond-level pauses are unacceptable as they can cause SLA violations or financial losses.

3. #### Explain the concept of GC pauses and how they affect application latency.

Answer: GC pauses occur when the collector stops the application threads to reclaim memory. These pauses can result in latency spikes, making real-time response guarantees difficult to meet.

4. #### What are Stop-the-World (STW) events, and how do they relate to GC?

Answer: STW events are points during GC where all application threads are halted. Most GC algorithms require at least brief STW phases, impacting application responsiveness.

5. #### What are the different types of GC algorithms you're familiar with?

Answer: Mark-and-Sweep, Copying, Generational, Concurrent Mark Sweep (CMS), G1GC, ZGC, Shenandoah, Reference Counting, and Manual Memory Management.

6. #### What is a generational GC, and how does it optimize memory management?

Answer: Generational GC divides memory into young and old generations. Most objects die young, so frequent and quick collections of the young generation improve efficiency and reduce pause times.

7. #### Compare throughput-oriented vs latency-oriented GC strategies.

Answer: Throughput-oriented GC (like Parallel GC) maximizes total application throughput with longer pauses. Latency-oriented GC (like ZGC or Shenandoah) aims for low pause times, even at the cost of some throughput.

8. #### Can you explain the trade-offs between compacting and non-compacting GCs?

Answer: Compacting GCs reduce fragmentation but incur longer pause times. Non-compacting GCs avoid pauses but risk fragmentation, which can hurt allocation performance over time.

9. #### What is "concurrent GC"? How does it help with latency-sensitive workloads?

Answer: Concurrent GC runs alongside the application, minimizing pause times by performing most work without stopping the application threads.

10. #### What is memory fragmentation, and how can it impact low-latency systems?

Answer: Fragmentation occurs when memory is freed in non-contiguous chunks. It leads to inefficient allocation, longer GC times, or allocation failures, increasing latency.

### Java-Specific

11. #### Compare G1GC, ZGC, and Shenandoah in terms of latency.

Answer:

G1GC: Medium latency, good for general use.

ZGC: Sub-millisecond pauses, highly scalable.

Shenandoah: Also low-pause, supports concurrent compaction.

12. #### What tuning parameters have you used for G1GC to reduce latency?

Answer: -XX:MaxGCPauseMillis, -XX:+UseStringDeduplication, -XX:InitiatingHeapOccupancyPercent, and sizing parameters like -Xms, -Xmx.

13. #### What is the effect of -Xms and -Xmx settings on GC behavior?

Answer: Setting both to the same value prevents heap resizing, reducing GC overhead and improving predictability in low-latency environments.

14. #### How would you analyze GC logs for latency spikes?

Answer: Use tools like GCViewer, GCEasy, or JFR. Look for long STW phases, Full GC frequency, heap occupancy, and promotion failures.

15. #### How do you reduce Full GC frequency in Java applications?

Answer: Increase heap size, tune survivor ratio, reduce object promotion, or use a low-pause GC like ZGC.

### Scenario-Based

20. #### You're noticing latency spikes and GC is the cause—what do you do?

Answer: Analyze GC logs, tune heap and GC parameters, reduce allocation rate, use object pooling, or switch to a concurrent GC.

21. #### How would you redesign a microservice to minimize GC impact?

Answer: Use immutable objects, pre-allocated buffers, minimize allocation rate, and prefer stateless design.

22. #### How do object allocation patterns influence GC behavior and latency?

Answer: Frequent allocations create pressure on young gen GC. Large or long-lived objects increase Old Gen size, triggering longer pauses.

23. #### Have you used object pooling to reduce GC pressure? How do you manage reuse safely?

Answer: Yes. Use thread-local or bounded pools. Ensure proper reset and avoid leaks. Use with caution in highly concurrent environments.

24. #### How do you monitor GC in production?

Answer: Use JFR, VisualVM, Prometheus with exporters, ELK for logs, or Application Performance Monitoring (APM) tools.

Advanced

25. #### What is escape analysis and how can it reduce GC pressure?

Answer: Escape analysis determines if an object can be stack-allocated instead of heap-allocated, reducing GC workload.

26. #### What techniques ensure real-time guarantees with GC?

Answer: Use real-time GC (like ZGC), pin critical objects, pre-allocate memory, or avoid GC via custom allocators.

27. #### What’s your experience with zero-allocation design patterns?

Answer: Avoid runtime object creation in hot paths. Use flyweight patterns, buffer reuse, and static allocation.

28. #### How do pre-touching memory and NUMA awareness help with latency?

Answer: Pre-touch reduces page faults. NUMA-awareness ensures memory is close to CPU cores, reducing access latency.

29. #### Would you ever disable GC in a low-latency application?

Answer: Rarely, only in tightly controlled environments with fixed memory usage. Requires manual memory management.