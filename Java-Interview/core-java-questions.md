### Explain JVM memory areas
<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/808d4061-b04d-4de5-9387-cb79d9daedee" />

### What happens when heap is full
Objects reside in heap area and GC is performed on this in phases. When heap is unable to allocate memory for a new object that means heap is full.
Generally, GC is performed when heap is full, if memory is not claimed back even after full GC then it means heap is full and results Out of memory error.

### OutOfMemory vs StackOverFlow
**OutofMemory:** It can be thrown by Heap area of Metaspace. When heap area is unable to allocate memory for a new object even after GC then it throws OOM error.
                 JVM can dump the heap using -XX:+HeapDumpOnOutOfMemoryError
**StackOverFlow:** Each methood call pushes a frame into the stack, when frames exceeded the stack size then it throws StackOverFlow error. Generally due to infinite loop.

### What are common causes of memory leaks in Java?
– How do you detect and fix them?
    - Objects are unused but still showing as referenced might be due to static variables
    - Caches without eviction
    - Remove values from thread local variables

We detect them using heap dumps, GC logs, and profilers like MAT or VisualVM, and fix them by removing strong references, adding eviction, and proper resource management.

### Explain Hashmap, hash collision and rehashing

- Initial capacity of map is 16 and **threshold = capacity * 0.75**, means when number of elements reaches the threshold then map size will be doubled.
- While inserting key into the map hashcode will be calculated and then index will be calculated **(n-1) & hash** and insert the element
- **Hash Collison:**
   * There is a possibility that different keys may have same hashcode and goes to the same bucket, this is where hash collision occurs.
   * In such cases multiple keys will be stored in the same bucket in linked list structure.
   * From Java8, red black tree is being used in place of linked list but not from initial collision.
   * When there is a collision, it checks if the **linked list size is > 8 and total array length is < 64** then it prefer to double the array size instead of treefying as it is cheap operation.
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
- ``` It uses comparator to compare elements ```
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

### What happens when two interfaces contain identical default methods and a class implements both? How can it be resolved?
If a class implements two interfaces that define the same default method, Java reports a compilation error because the implementation is ambiguous. 
We resolve it by overriding the method in the implementing class and optionally invoking a specific interface implementation using InterfaceName.super.method().
If a superclass provides a concrete implementation, the class method takes precedence over the interface defaults.

###
The Liskov Substitution Principle states that a subclass or implementation must be usable wherever its parent type is expected without breaking the program's correctness or behavioral contract.
Inheritance is still useful for code reuse, specialization, and polymorphism. LSP ensures that these benefits don't come at the cost of unexpected behavior.
For example, if a Penguin inherits from a Bird class that promises every bird can fly, it violates LSP if its fly() method throws an unsupported-operation exception. 
We should redesign the abstraction so that only flying birds implement a flying capability.
```
    interface Bird {
        void eat();
    }
    
    interface FlyingBird extends Bird {
        void fly();
    }
    
    class Sparrow implements FlyingBird {
        public void eat() {}
        public void fly() {
            System.out.println("Sparrow flying");
        }
    }
    
    class Penguin implements Bird {
        public void eat() {}
    }
```

###  Difference between `synchronized` and `ReentrantLock`. What are the trade-offs?
Both synchronized and ReentrantLock provide mutual exclusion and are reentrant. synchronized is simpler, and the JVM releases its monitor automatically. 
ReentrantLock offers additional capabilities such as tryLock(), timed and interruptible acquisition, optional fairness, and multiple condition variables.
The trade-off is simplicity and safer automatic cleanup versus greater control and flexibility.
I generally prefer synchronized for straightforward critical sections and use ReentrantLock when I specifically need its advanced locking features.
I also benchmark performance rather than assuming one is always faster.

### What is the `volatile` keyword?
```volatile``` is a Java keyword that guarantees visibility of updates to a shared variable across threads and provides memory-ordering guarantees under the Java Memory Model. However, it does not provide mutual exclusion or make compound operations atomic. For example, ```volatile``` int count does not make ```count++``` thread-safe. I use volatile for simple flags and use ```AtomicInteger```, synchronized blocks, or locks when atomic updates or compound operations are required.

### Difference between sequential and parallel streams?
Sequential streams process elements one by one, while parallel streams divide work across multiple threads. Parallel streams can improve performance for large, independent, CPU-intensive operations, but they introduce overhead and concurrency concerns. I choose between them based on workload size, thread safety, ordering requirements and performance measurements.

### Why collections are not immutable even declared as final in immutable class
```final``` keyword is not enough to make the collection immutable because when we return the collection 
with getter, and it returns the same reference, and we can add more elements in it.
So, to avoid this we need to create unmodifiable list.
```this.subjects = Collections.unmodifiableList(new ArrayList<>(subjects));```

### Can we write validations in record 
Yes, we can create compact constructor to perform any validation in record.
```
public record Voter(String name, int age) {

    public Voter {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name is required");
        }
        if (age < 18) {
            throw new IllegalArgumentException("Voter must be at least 18 years old");
        }
        name = name.trim();
    }
}
```

### Why do we need public in above example for compact constructor
It should be accessible to create instantiation or before instantiating.
### Can we have private constructor and other methods like static and overridden ?
Yes, private constructor can be added, but it should call another constructor of the record. Canonical or Compact
constructor can not be private as it should be accessible from outside.
record can also have normal methods, static, and overridden methods as it can implement interfaces.
### Can record extend class ?
No, because it has already extending one class ```java.lang.Record``` 

### How and Why record is final and immutable ?
record is by default final and all fields are public final. No class extend record.
record can implement interfaces.


