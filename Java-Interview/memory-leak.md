### Memory Leak:

Main reason for memory leak is an unused object is still reachable. So it keep stays in memory after multiple GCs.

### Detect memory leak:

Memory leak can be find using tools like visualVM and analyse MAT(Memory analyzer tool). 
Export a heap dump from here and check what is going with objects in heap area.

### How to take heap dump:
 * Auto generate when out of memory

        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/tmp/heapdump.hprof
 * Generate manually

        jcmd <pid> GC.heap_dump /tmp/heapdump.hprof
 * Export from visual vm

Import the heap dump into analyser and inspect the memory allocations

    Class	          Shallow Heap	  Retained Heap	  Objects
    byte[]	          1,024,000,000	  1,024,000,000	  1000
    java.util.HashMap 48              1,024,300,000   1

Here retained plays the main role which indicates how much memory will be freed up if this object is disappeared.
Now, open the Dominator Tree then figured out the leak.
Next open the Path to Roots and click on suspicious objects.

    byte[1048576]
    <- java.util.HashMap$Node.value
    <- java.util.HashMap.table
    <- java.util.HashMap
    <- MemoryLeakDemo.CACHE
    <- java.lang.Class
    <- System Class Loader
    <- GC Root

If the object is still reachable from GC Root then it can not be collected by GC.
### GC Roots:
GC Roots are the objects which are considered as "always alive".
Ex: Thread Stacks, Static Variables

Example Problems:
* static Hashmap
* static thread local
Here system keep inserting elements but never removing any.
* 

