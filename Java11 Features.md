# Java11 Features

After Java8, this is the long supported version. From Java11 JDK is no longer free for commercial use.
- ### New methods in String
- ### New methods in Files
- ### toArray method
   used to convert collection to an array
- ### not predicate method
      .filter(Predicate.not(String::isBlank))
- ### Local Variable Syntax in lamba
      .map((@Nonnull var x) -> x.toUpperCase())
- ### Z GC was introduced
  GC pause times never exceeded 10 ms. Only for Linux.