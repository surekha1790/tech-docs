# Java12 Features

- ### New methods in String and File
   * indent in string
   * mismatch in Files
- ### switch Expressions
    Removed boilerplate code and made more concise and readable.
    Removed break statement

      typeOfDay = switch (dayOfWeek) {
        case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Working Day";
        case SATURDAY, SUNDAY -> "Day Off";
      };
      
      case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> {
      // more logic
         System.out.println("Working Day")
      }
- ### InstanceOf pattern matching
    **Old Syntax:**

      Object obj = "Hello World!";
      if (obj instanceof String) {
         String s = (String) obj;
         int length = s.length();
      }
    
    **New Syntax:**

      if (obj instanceof String s) {
        int length = s.length();
      }
- ### JVM Changes
   Experimental low pause time GC is introduced **Shenandoah**. It reduces the GC pause times by doing evacuation work simultaneously .
