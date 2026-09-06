### Thread and Heap dump

* Use ```jcmd``` or ```jstack``` to take the heap/thread dump.
* Thread dump: ``` jcmd 12345 Thread.print > thread-dump.txt ``` and ```jstack 12345 > thread-dump.txt ```
* GC Heap dump: ```jcmd 12345 GC.heap_dump /tmp/heap.hprof ```
* In Thread dump, look for runnable, blocked, waiting threads
* Open Head dump using Eclipse MAT and check dominator tree, histogram, GC roots, retained heap
* JFR is a JVM tool which is like a black box recorder. We can start JFR to investigate what is happening in JVM during certain period.
* jcmd <PID> JFR.start duration=60s filename=app.jfr
* It is basically useful when thread dump and heap dump is not helping to identify the issue.
