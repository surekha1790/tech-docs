# Java13 Features

- ### Switch Expression
  * yield statement is introduced to return value from switch
    
        var result = switch (operation) {
           case "doubleMe" -> {
             yield me * 2;
           }
           case "squareMe" -> {
             yield me * me;
           }
           default -> me;
        };
- ### Z GC improvements
   * Returns Uncommit/Unused heap space back to OS
   * maximum supported heap size of 16TB. Earlier, 4TB
   * Only for Linux/x64

