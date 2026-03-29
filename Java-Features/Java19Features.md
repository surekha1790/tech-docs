# Java19 Features

- ### HashMap changes
   Create hashmap object with number of elements
      
      // for 120 mappings: 120 / 0.75 = 160
      Map<String, Integer> map = new HashMap<>(160);
- ### Switch Pattern Matching
     ```When``` keyword is introduced
       
      case String s when s.length() > 5 
- ### Record with pattern matching

      private void print(Object object) {
          //if (object instanceof Position position) {
         if (object instanceof Position(int x, int y)) {
           System.out.println("object is a position, x = " + x + ", y = " + y);
         }
           // else ...
      }
- ### Virtual Threads
  * Virtual thread more light-weight thread compared to traditional threads. 
  * It is designed to make concurrency simpler, more scalable and memory efficient.
  * These threads are managed by JVM unlike platform threads managed by OS.
  * A Million virtual threads can be created without causing any issue with I/O operations.
  * Life cycle can be easily managed by JVM to start, yield, resume, stop virtual threads.
  * If there is priority for OS thread then it yields the virtual thread and let the OS thread finish the work then resume it back.
  * It uses very low memory compared to platform threads
  * It is useful when costly I/O operations are present like DB calls, HTTP requests
  * Useful when short-lived I/O blocking operations
  * High concurrency is required
  
    Not ideal for:
    CPU-bound tasks (no performance gain over platform threads)
  
        Thread virtualThread = Thread.ofVirtual().start(() -> {
            System.out.println("Hello from a virtual thread!");
        });
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
             executor.submit(() -> System.out.println("Task 1 in a virtual thread"));
             executor.submit(() -> System.out.println("Task 2 in a virtual thread"));
        } 

Heavy ThreadLocal usage

- ### Structured Concurrency