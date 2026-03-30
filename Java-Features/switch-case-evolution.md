### Switch Case Evolution

* Earlier in Java5 and Java7 switch case was improved by adding enums and strings.

        `switch(enum) {
            case MON : system.out.println("");
            break;
         }

         switch(enum) {
             case "MON" : system.out.println("");
             break;
        }`

#### From Java 12 there were couple major changes has been made

* Syntax changed to 
    `case -> `
* No `break` statement required
* Switch can return a value, use `yield` in case of multi line body
* Multiple conditions with same body can be grouped together 
 

    ` String value = switch(enm) {
      case SUN, SAT -> "weekend";
      case MON, TUE -> "weekday";
    } `

* Pattern Matching introduced and can handle null values also
  

    Object obj = "hello";
    switch (obj) {
        case null -> System.out.println("Null value");
        case String s -> System.out.println(s.toUpperCase());
        case Integer i -> System.out.println(i * 2);
        default -> System.out.println("Unknown");
    }

* `When` keyword is introduced for condition check

      switch (obj) {
          case String s when s.length() > 5 -> System.out.println("Long string");
          case String s -> System.out.println("Short string");
          default -> System.out.println("Other");
      }

* Record Pattern also allowed to match from Java21


      record Point(int x, int y) {}

      Object obj = new Point(10, 20);
      switch (obj) {
        case Point(int x, int y) -> System.out.println("Point: " + x + ", " + y);
        default -> System.out.println("Other");
      }

* Modern switch compiler can check necessary cases which eliminates the `default` case

#### Important point to note

    switch (obj) {
        case Object o -> System.out.println("Any object");
        case String s -> System.out.println("String");
    }

  This is Dominance case as Object matches any non null object so String case is unreachable.

    switch (obj) {
        case Object o -> System.out.println("Any object");
        case String s -> System.out.println("String");
    }

* Introducing Primitive type matching from Java 23 (still under preview till Java26)


    switch (value) {
        case 0 -> System.out.println("Zero");
        case 1 -> System.out.println("One");
        case int i -> System.out.println("Other int: " + i);
    }
  This allows to match all other integer values with type