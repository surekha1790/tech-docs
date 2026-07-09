### Explain JVM memory areas
<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/808d4061-b04d-4de5-9387-cb79d9daedee" />

### What happens when heap is full
Objects reside in heap area and GC is performed on this in phases. When heap is unable to allocate memory for a new object that means heap is full.
Generally, GC is performed when heap is full, if memory is not claimed back even after full GC then it means heap is full and results Out of memory error.

### OutOfMemory vs StackOverFlow
**OutofMemory:** It can be thrown by Heap area of Metaspace. When heap area is unable to allocate memory for a new object even after GC then it throws OOM error.
                 JVM can dump the heap using -XX:+HeapDumpOnOutOfMemoryError
**StackOverFlow:** Each methood call pushes a frame into the stack, when frames exceeded the stack size then it throws StackOverFlow error. Generally due to infinite loop.

### Explain Hashmap, hash collision and rehashing

- Initial capacity of map is 16 and **threshold = capacity * 0.75**, means when number of elements reaches the threshold then map size will be doubled.
- While inserting key into the map hashcode will be calculated and then index will be calculated **(n-1) & hash** and insert the element
- **Hash Collison:**
   * There is a possibility that different keys may have same hashcode and goes to the same bucket, this is where hash collision occurs.
   * In such cases multiple keys will be stored in the same bucket in linked list structure.
   * From Java8, red black tree is being used in place of linked list but not from intial collision.
   * When there is a collision, it check if the **linked list size is > 8 and total array length is < 64** then it prefer to double the array size instead of treefying as it is cheap operation.
   * When array **length is >= 64** then it will convert the particular bucket into red black tree. Treefying is not per hashmap instead it is per bucket.
 
 - **Rehashing**
     * When table size grows than the threshold then it increases the table size to 32 (double).
     * Every entry should be redistributed according to the new size.
     * Jav8 does this in smart way to avoid calculating hash for all elements.
     * It calculates **hash & oldcapacity (16)**, if it is 0 then it stays in the same index otherwise moves to **index + oldcapacity (16)**
 - Hash is not calculated everytime. Each node caches the hash during insertion and same will be used by JVM during resizing.  

### What is Red Black tree in Hashmap
It is self balanced binary serach tree. If it is a plain binary search tree and inserting sorted order elements then tree grows in one directon and complexity becomes O(n) which is worst case for hashmap.
So, to avoid this hashmap uses red black AVL tree and it applies left/right rotations to balance the tree. Root element is black and no two red should be in same line.
    10        10                20           20                   20 
      \         \               / \          / \                  / \ 
       20        20   ==>      10  30       10  30    ==>        10  40
                   \                              \                  / \  
                    30                             40                30 50

### Difference between Hashmap, synchronized hashmap and Concurrent hashmap
- All stores key value pair
- Hashmap an Synchronized hash map allows one null key and null value but concurrent hashmap does not allow
- Hashmap is not thread-safe but syncronized and concurrent hashmaps are thread safe.
- Synchronized hashmap locks the entire map, even for read actions so multiple threads are blocked.
- Concurrent map locking is per bucket level and no locking on read.

### Why concurrent hashmap is fast
- It does not acquire lock on entire map rather locks only particular bucket.
- No locking while reading
- Insertion into empty bucket never locks, it acquires lock only when there is a collision.
- It does not allow null because `get()` returns null, so it means no element rather than creating ambiguity.
                    


