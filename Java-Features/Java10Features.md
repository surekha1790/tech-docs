# Java10 Features

- ### Collection, Optional api changes
   New methods introduced
   * copyOf --> copy collection to unmodifiable collection
   * toUnmodifiableList/Set/Map --> to collect stream into unmodifiable collection
   * OrElseThrow in Optional class

- ### Local Variable Type inference:
  'var' is used instead of type declaration.
  * it can be used only for local variables
  * Can not be used for method parameters, return types
  * Can not be used for variable with null values


  **Examples:**

      var idToNameMap = new HashMap<Integer, String>();
      var message = "Hello, Java 10";
