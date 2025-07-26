# Java18 Features

- ### Switch Pattern Matching improvements
   * should cover all cases otherwise compile time error, means must have default case.
   * Compile time error as CharSequence also similar to String and covers the same case
          
         case CharSequence cs -> 
         case String s ->  
- ### Simple Web Server
   JWebServer introduced to run lightweight applications without any external servers.
   > Run jwebserver --port 8080 --directory . under project directory