# Multi-Thread Interview Questions

1. #### What is Optimistic and Pessimistic Locking ?
    **Optimistic Lock:** Concurrency management mechanism where multiple transactions
    can be completed without interfering each other. It uses version field to identify the conflict during commit.
    
    **Pessimistic Lock**: Locking mechanism where it locks the read until the current thread complete the transaction.
    It causes blocking resource.
2. #### What are the main differences between optimistic and pessimistic locking?

     
      Optimistic Locking	       Pessimistic Locking
      -------------------          --------------------    
      No locks at DB level	        Uses DB locks
      Conflict checked at commit	Prevents conflict by locking
      Better performance	        Safer for high contention
      Uses @Version	                Uses lock hints or SELECT FOR UPDATE

3. #### Disadvantages of Optimistic and Pessimistic Locking

     
     Optimistic Locking	      
     -------------------      
     * Check during commit only 
     * Possibilities of Optimistic locking exceptions
     * Must handle exceptions
 
     Pessimistic Locking
     -------------------
     * Locks the operations
     * Chances of deadlock
     * reduces concurrency

4. #### When should use Optimistic and Pessimistic Locking
   
   * When low contention and  high performance is required then go for Optimistic Lock
   * When high contention and low concurrency then Pessimistic Locking

5. #### How Version works in Optimistic Locking
    A version field with ```@version``` annotation will be used which increments version for every update.
    During commit it checks the version with DB value, if there is a mismatch then throws exception.


    @Version
    private int version;

1. #### What are the different lock modes in JPA/Hibernate?
    * LockModeType.OPTIMISTIC
    * LockModeType.OPTIMISTIC_FORCE_INCREMENT
    * LockModeType.PESSIMISTIC_READ
    * LockModeType.PESSIMISTIC_WRITE
    * LockModeType.PESSIMISTIC_FORCE_INCREMENT
2. #### What kind of locks does pessimistic locking use?
   Database-level locks like:
   * Row-level exclusive locks
   * Table locks (rare)
   * Typically via SELECT ... FOR UPDATE
3. #### How do you handle deadlocks with pessimistic locking?
   * Use short transactions
   * Always acquire locks in the same order 
   * Set a lock timeout 
   * Use database diagnostics (SHOW ENGINE INNODB STATUS)
4. #### How does isolation level affect locking?
    Higher isolation levels (e.g., SERIALIZABLE) use more locking. Locking behavior is tightly coupled with isolation:
    * READ COMMITTED: avoids dirty reads 
    * REPEATABLE READ: avoids non-repeatable reads 
    * SERIALIZABLE: avoids phantom reads
5. #### What is thread leak ?
   * When a thread is running more than expected time and never released or terminated this is called 
   thread leak.
   * This causes resource blocking, increase memory usage and eventually causes out of memory issue.
   * It can occur when thread local data is not cleaned up.
   * Memory won't be freed up until thread dies

