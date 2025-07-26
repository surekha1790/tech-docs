# Java15 Features

- ### Record class changes
   We can add canonical constructor.

      public record Person(String name, int age) {
         public Person {
          if(age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
          }
         }
      }
- ### Sealed Classes
    To add more control on inheritance. Sealed classes allows to define which subclasses can inherit the parent class/interface.

      public abstract sealed class Person
        permits Employee, Manager {

         //...
      }
      public final class Employee extends Person {
      }
      public non-sealed class Manager extends Person {
      }
   > It’s important to note that any class that extends a sealed class must itself be declared sealed, non-sealed, or final. This ensures the class hierarchy remains finite and known by the compiler.

- ### Hidden Classes
   Not really useful for developers but useful who works with JVM byte code. 
   * Runtime creation of classes which are not discoverable.
   * Can not be discovered by reflection.
  
- ### Pattern Matching in InstanceOf

      if (person instanceof Employee employee) {
       Date hireDate = employee.getHireDate();
       //...
      }
   
      if (person instanceof Employee employee && employee.getYearsOfService() > 5) {
         //...
      }