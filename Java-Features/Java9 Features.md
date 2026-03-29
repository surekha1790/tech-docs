# Java9 Features 

- ### Java Platform Module System (JPMS):
   Java9 introduced module structure for strong encapsulation and scalability for small application.
   > It requires module-info.java file in each module.
   
   > module-info.java is descriptor file which contains module name, dependencies, export packages.
  
   #### Project Structure:
         project-root/
        ├── app/
        │   ├── module-info.java
        │   └── com/example/Main.java
        └── utils/
        ├── module-info.java
        └── com/util/Helper.java

   **Example module-info.java from app module**
        
      module app {
         requires java.base; // optional, included by default
         requires utils;
      }
- ### Jshell:
   It is interactive console used to write and run java snippet without saving.
- ### Improved JavaDocs: 
- ### Collection factory method improvements:
  Java9 introduced few more new methods in Collections to create immutable collection.
  
  **Example:**

      list = Collections.unmodifiableList(list);
      List.of(1,2,3), Set.of(2,3,4), Map.of(1,2,3,4)
- ### Private and Static Methods in Interface
   Allowing to create private and static methods in interface. Private methods are not accessed from implemented classes,
   and they are helper methods encapsulated within interface
   
   * Private methods are not accessible from outside and acts as helper methods for default implementations.
   * Static methods are also used as helper/utility methods which can be accessed outside using interface name only
     not by any implementation class instance.
  #### ✅ Valid Use Cases
    * Utility or helper methods tied to the interface (e.g., parsing, validation)
    * Test data or demo setup methods 
    * Replacing separate Util classes

- ### Stream apis improvements
      takewhile, dropwhile, iterate methods are introduced
- ### Try with Resources:
      BufferedReader br = new BufferedReader(inputString);
      try (br) {
         return br.readLine();
      }
- ### GC Improvements:
   * G1 become default GC
   * introduced -Xlog to print unified logs for Structured, consistent, and filterable logging and easy monitoring
   * Full GC was single threaded earlier, but now it is multithreaded.
  
      
    xlog:gc*,ergo*=trace,age*=trace,phases*=debug,safepoint:%LOG_DIR%/gc.log:time:filecount=20
    ✅ Log Tags:
        gc*	         Logs all GC-related events (like start, end, memory usage, etc.)
        ergo*	 Logs GC ergonomics (automatic tuning decisions by JVM)
        age*	 Logs object aging info (e.g., promotion from young to old generation)
        phases*	 Logs internal GC phases (like marking, compacting, cleanup)
        safepoint    Logs JVM safepoints (where GC or other VM tasks run safely)