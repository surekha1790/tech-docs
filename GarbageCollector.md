# Garbage Collector

Object are created in heap area. GC helps in cleaning up and free up the space by removing unused objects.
This is done automatically in the background.

## Heap Structure:

Heap are is divided in few areas to maintain newly created and old objects.

1. #### Young Generation : 
    * ##### Eden Space:
        New object will be created here and minor GC will be performed frequently on this area.
    * ##### Survival Space:
        Objects survived after minor GCs are moved to this space
2. #### Old Generation:
      Objects survived after several minor GCs will be moved here. 
      Major GC will be performed here less frequently as it is costly operation.
3. #### Meta Space:
     PermGen Space is replaced by meta space in java8 which is auto growing memory. It stores class metadata.

## GC Types:

- ### Single GC
   * Single thread for all GC operations
   * Suitable for only small applications
   * stop-the-world pause time is more
- ### Parallel GC
   * Multithreaded on Young and Old generation.
   * High throughput.
   * Suitable for multi core CPUs.
   * still pause time is more.
   * not suitable for low latency applications.
- ### Concurrent Mark and Sweep GC:
   * Concurrently marks unused objects and sweeps them.
   * low pause time.
   * Only young generation will be compacted but not old generation space which causes Fragmentation issues.
    ##### Fragmentation issue:
     When object are being cleaned up from heap area, those spaces will be free but empty blocks will be increased
     That means, total space will be there, but it is fragmented as small discontinued spaces. 
     This is due to compact issue where free space won't be moved in old generation.
     Ex: [Live][ ][Live][ ][ ][Live]
- ### G1 GC:
   * Default from Java9
   * Heap area is split into regions other than generations.
   * concurrently marks, track live objects and sweep objects.
   * G1 can collect Young + parts of Old Gen together.
   * Low pause time.
   * Handles large heaps efficiently.
   * Suitable for low latency applications
   * CPU utilisation is a bit more
- ### ZGC:
   * Divides heap area different way not as generations
   * It treats heap ara uniformly not like generations
   * Uses color pointers and load barriers which is a piece of code to store metadata of objects.
      This metadata will be used to know about object state.
   * Concurrently marks and sweeps objects
   * Concurrently compact the space
   * very low pause time
   * Best for low latency applications
   * Heap size is large ~16TB

  #### Metadata bits track:
  * Marking state (marked or not). 
  * Remap state (needs updating or not). 
  * Finalizable object tracking. 
  * Barriers state (active/inactive).