### Difference between UNION and JOIN
  * Both are used to get combined results from two or more tables.
  * JOIN works based on primary key, foreign key relationship and it combines columns side by side.
  * UNION works on two tables without any join condition and it combines rows.
  * **JOIN**: ``` SELECT c.name, o.amount FROM customers c JOIN orders o ON c.id = o.customer_id; ```
  * **UNION:** ``` SELECT email FROM customers UNION SELECT email FROM suppliers; ```
  * Join Types: Inner join, Natural Join, Left Join, Right Join, Full Join
  * Union Types: UNION, UNION ALL

### Rules for UNION
* Same number of columns
* Same datatypes
* Same column order
* Same column names
* UNION ALL is faster as it includes duplicates, UNION needs to filter duplicates, so it is slower than UNION ALL
  
