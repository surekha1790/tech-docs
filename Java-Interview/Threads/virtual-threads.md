
## What is Project Loom ?

* Main goal is to make blocking cheaper
* To make concurrency simple
* Scale to millions of concurrent tasks

Features in Project Loom:
- Virtual Threads
- StructuredTaskScope
- Scoped Values

### Virtual Threads

Virtual threads are introduced in Java19  as preview feature and made it as stable feature in Java21.

#### Disadvantages of Platform Thread:
* Heavy weight as each thread creates a fixed stack 
* So it takes huge memory to create large number of threads
* Platform thread is managed by OS and it creates OS thread for each request.
* n number of threads means n number of OS threads. When there is a block/waiting for something
  it blocks OS thread and waste resources. So thread block is very expensive.
* Context switching will also become costly with number of threads as it has to 
   - switch kernal mode,
   - save registers
   - restore state 
   - flush CPU cache.
* CPU can accommodate more threads to process but performance/memory will be a challenge as 
  each thread takes stack area and block OS threads.

#### Virtual Thread:
* Light-weight threads
* Handled by JVM
* Thread creation takes memory KBs
* These threads reside in JVM heap memory
* When virtual thread is created very less memory will be occupied (in KBs)
* Virtual thread is attached to OS thread (Carrier Thread) while running the task.
* If it is blocked or completed then OS thread will be released and picked up another OS thread 
  in case virtual thread is released from block.
* This is the reason blocking is not costly.
* Virtual thread stack grows dynamically.
* Each virtual thread method call creates a stack frame in heap area which is managed by JVM
* For each method call it creates a stack frame and connected with each other by linking
* During blocking, JVM captures frames, releases carrier thread and frames stays in heap area

### StructuredTaskScope

* Internally uses virtual threads
* manages thread life cycle like start, stop, fail and clean
* parent - child relation
* it manages automatic cancellation 
* Error propagation

      try(var scope = StructuredTaskScope.ShutdownOnFailure()) {
         var fork1 = scope.getUser();
         var fork2 = scope.getOrder();
         scope.join();
         scope.throwIfFailed();
      }
* Failure cancels siblings



