# Java8 Features

- ### Functional Interface : 
    Interface which contains only one abstract method and any number of non-abstract methods.

    **Example**:

      @FunctionalInterface
      public interface MyFunctionalInterface {
        void doSomething();
      }

   #### ✅ Why use ```@FunctionalInterface```?
    * **Lambda Expressions** : It is used to implement lambda expressions.
    * **Compile time check** : Compiler make sure it has only one interface and if anyone add accidentally it gives compile time error
    * **Readability**        : Improve readability as it helps to understand that it is a functional interface by looking at the annotation

   #### ❌ Is it mandatory?
    > No it is not mandatory to add annotation, but it causes adding more abstract methods accidentally which will no longer be a functional interface.
- ### Lambda Expressions:
    
    It is mainly used to implement functional interface. It is concise form of implementing anonymous functions. 

    #### ✅ Why Use Lambdas?
     * To implement functional interfaces
     * Avoid boilerplate code
     * Short and clean code
     * Uses in streams
     * Functional programming in Java
    
      **Examples:**
   
        MyFunctionalInterface mfi = () -> {System.out.println("do something")};
        mfi.doSomething();
- ### Streams:
    It is a pipeline of operation that process sequence of elements such as Collections (List, Set, Map).
    
    >      There are two types of operations: Intermediate and terminal

   #### ✅ Stream Methods
  * filter, map, flatmap, foreach, collect ...
   
        **Examples:**
  
           int sum = nums.stream()
                       .filter(n -> n % 2 == 0)
                       .mapToInt(n -> n)
                       .sum();
- ### Method References:
 
    Method references are to make method call concise and avoid boilerplate code.
    
     It can be used with static method, instance methods and constructors
    
            **Examples:**

               String::toLowerCase
               System.out::println
- ### Default Methods:

    Before Java 8, interfaces could not have any implementation — they could only declare methods. 
    If a new method was added to an interface, all implementing classes would break unless they implemented.
    
  > If a class implements multiple interfaces with conflicting default methods, it must override the method to resolve the conflict.
                
      Example:
  
          interface InterfaceA {
              default void show() {
                  System.out.println("InterfaceA show()");
               }
           }

            interface InterfaceB {
                default void show() {
                  System.out.println("InterfaceB show()");
                }
             }

             class MyClass implements InterfaceA, InterfaceB {
               // Compilation error if this method is not overridden

                @Override
                public void show() {
                    // Must resolve the conflict explicitly
                    System.out.println("MyClass resolving conflict");
                    InterfaceA.super.show(); // Optionally call one of the interface's default methods
                    InterfaceB.super.show(); // You can call both if needed
              }
      }
        
   This also resolves multiple inheritance problem.  
- ### Optional:
   To handle Null Pointer Exceptions and null more concise and less error-prone way
  
   There are some methods available in Optional class to handle nulls
      
      Optional.isPresent, Optional.ifPresent, Optional.orElse, Optional.map, Optional.ofNullable, Optional.of