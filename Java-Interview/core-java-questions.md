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
   * From Java8, red black tree is being used in place of linked list but not from initial collision.
   * When there is a collision, it check if the **linked list size is > 8 and total array length is < 64** then it prefer to double the array size instead of treefying as it is cheap operation.
   * When array **length is >= 64** then it will convert the particular bucket into red black tree. Treefying is not per hashmap instead it is per bucket.
 
 - **Rehashing**
     * When table size grows than the threshold then it increases the table size to 32 (double).
     * Every entry should be redistributed according to the new size.
     * Jav8 does this in smart way to avoid calculating hash for all elements.
     * It calculates **hash & oldcapacity (16)**, if it is 0 then it stays in the same index otherwise moves to **index + old capacity (16)**
 - Hash is not calculated everytime. Each node caches the hash during insertion and same will be used by JVM during resizing.  


### What is Red Black tree in Hashmap
It is self balanced binary search tree. If it is a plain binary search tree and inserting sorted order elements then tree grows in one direction and complexity becomes O(n) which is worst case for hashmap.
So, to avoid this hashmap uses red black AVL tree, and it applies left/right rotations to balance the tree. Root element is black and no two red should be in same line.
    10        10                20           20                   20 
      \         \               / \          / \                  / \ 
       20        20   ==>      10  30       10  30    ==>        10  40
                   \                              \                  / \  
                    30                             40                30 50
### Explain TreeHashSet internal implementation
- It uses self balanced red black tree.
- root is always black.
- small elements left side and bigger elements in right side.
- no two red nodes should be in a row.
- rebalances the tree to keep complexity O(logn)
- null nodes should be black
- It uses comparator to compare elements
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

### What is memory leak 
- When any object is unused but still reachable in heap area which can not be collected by GC.
- Analyze heap dump and identity the memory leak.
- GC Root is always alive, ex: static variables, thread stacks

### What is the `var` keyword? Can it be used as a generic type?
```var``` was introduced in Java 10 for local variable type inference. The compiler infers the variable's static type from its initializer, improving readability without changing Java's static typing.
It cannot be used as a generic type argument, type parameter, field type, method parameter, or return type, but it can be used to declare local variables whose values are generic collections.

### What is effectively final variable
An effectively final variable is a local variable that is not explicitly declared final but is never reassigned after initialization. Java allows lambdas and anonymous classes to capture such variables.
This restriction applies to reassignment of the variable, not mutation of an object referenced by it.

### Why Lambda expression require effectively final ?
Java requires local variables captured by lambdas to be final or effectively final because lambdas capture their values, rather than providing shared access to reassigned local variables. This gives predictable behavior and avoids ambiguity about whether the lambda should observe the original or a subsequently assigned value. It also avoids requiring special shared-variable semantics for captured local variables. We can still mutate an object referenced by an effectively final variable, although that object may need separate thread-safety protection.

### What is Record class ? How to map fetched query results to record ?
Java Records are primarily intended for immutable data carriers, such as DTOs and query projections. Standard JPA entities require a no-argument constructor and generally need mutable state and proxy compatibility, 
so records are not appropriate for normal JPA entities. I use regular classes for entities and records for DTOs or read-only query projections.

### Can HashMap keys be mutable? Why or why not?
HashMap keys can be mutable objects. However, if a key's fields used by hashCode() or equals() change after insertion, the map may not find the entry because it was stored using the original hash. Therefore, immutable keys are preferred. Java records are often suitable keys when their components are themselves immutable and their equality and hash code remain stable.

### Can we define private methods in an interface?
Yes, Java supports private methods in interfaces from Java 9. They can be private instance methods or private static methods, and they help reuse common logic within default and static methods. 
They cannot be accessed or overridden by implementing classes, and they must have a method body.

### IoC vs DI
- Inversion of control is a principal. Framework control the object injection.
- When you inject object with new keyword then it is tightly coupled.
- Dependency injection is a technique spring uses IoC to inject dependencies.
- 
                    


