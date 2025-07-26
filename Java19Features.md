# Java19 Features

- ### HashMap changes
   Create hashmap object with number of elements
      
      // for 120 mappings: 120 / 0.75 = 160
      Map<String, Integer> map = new HashMap<>(160);
- ### Switch Pattern Matching
     ```When``` keyword is introduced
       
      case String s when s.length() > 5 
- ### Record with pattern matching

      private void print(Object object) {
          //if (object instanceof Position position) {
         if (object instanceof Position(int x, int y)) {
           System.out.println("object is a position, x = " + x + ", y = " + y);
         }
           // else ...
      }
- ### Virtual Threads

- ### Structured Concurrency