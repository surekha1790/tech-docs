### Changes in interface and why ?
* default, static, private methods are introduced.
* default methods are helpful to add new methods to interface without disturbing implementations. 
* static methods are factory or utility methods which belongs to interface. It can be called directly with interface and
  can not be overridden by subclasses. So, behaviour will not be changed.
  Ex: Collections interface contains static utility methods like sort, reverse, max, unmodifiableList
  ```java
    List<String> list = List.of("a", "b");                         // factory
    Comparator<User> c = Comparator.comparing(User::getName);      // helper
    Path p = Path.of("/tmp/file.txt");
    ```
* private methods are helper methods to share logic

### Sealed Interfaces
* Restrictions to which class can implement the interface
* It is really useful in switch pattern matching where it does not need default case as compiler knows which classes can implement
* If new class is added to the list then switch block will fail.

### Interface vs Abstract Class
* interface fields are always public static final, No Constructor, No protected or final methods
* Use when class need several roles Notifier is interface and email, sms notifiers are implementations.

### Multiple Inheritance
* If class extends one class and implements one interface then class wins
    ```
    class BaseOrder { public String info() { return "Base order"; } }
        class Order extends BaseOrder implements Trackable {
        // info() comes from BaseOrder, Trackable's default is ignored
    } 
    ```
* If it is chain implementation then parent wins
    ```
    interface Payable extends Trackable { default String info() { return "Payment"; } }
    class Order implements Trackable, Payable { }   // uses Payable.info()
   ```
  * Otherwise, must override or call with InterfaceName.super.methodName
      
    ```
    class Order implements Trackable, Payable {
      @Override
      public String info() {
          return Trackable.super.info() + " | " + Payable.super.info();   // combine both
      }
    }
    ```
### Why records can not be used for JPA Entity
* Records are immutable and final
* It does not have no arg constructor, only all args constructor
* JPA need no arg constructor to initialise and fill the values
* JPA require updates but records does not support
