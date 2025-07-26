# Java14 Features

- ### Record class 
   To remove boilerplate code in pojo classes new type Record class is introduced.
   It creates Immutable class.

      public record User(int id, String password) { };
   This automatically adds constructor, getters,toString, equals and hashcode methods.
- ### Helpful NullPointerExceptions 
    More meaningful and helpful message with NPE exceptions.
  > java.lang.NullPointerException: Cannot store to int array because "a" is null

- ### ZGC to windows and mac OS
   ZGC was introduced in Java11 only for linux/x64. As it received positive feedback on ZGC,
   ported support to windows and macOS.