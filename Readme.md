Resource: <br/>
https://www.w3schools.com/postgresql/index.php <br/>
https://www.postgresql.org/docs/current/tutorial.html


### Create a new table, add a column to an existing Table

```sql
CREATE TABLE cars (
  brand VARCHAR(255),
  model VARCHAR(255),
  year INT
);

INSERT INTO cars (brand, model, year)
VALUES
  ('Volvo', 'p1800', 1968),
  ('BMW', 'M1', 1978),
  ('Toyota', 'Celica', 1975);

ALTER TABLE cars  -- ALTER TABLE statement is used to add, delete, or modify columns in an existing table and also used to add and drop various constraints on an existing table.
ADD color VARCHAR(255);
```

### Update data/row in a Table

```sql
UPDATE cars
SET color = 'red'
WHERE brand = 'Volvo';  -- Without the WHERE clause, ALL records will be updated

-- Update multiple columns
UPDATE cars
SET color = 'white', year = 1970
WHERE brand = 'Toyota';
```

### Update column properties

```sql
ALTER TABLE cars
ALTER COLUMN year TYPE VARCHAR(4);  -- Change the year column from INT to VARCHAR(4)

ALTER TABLE cars
ALTER COLUMN color TYPE VARCHAR(30); -- Change the color column's maximum allowed characters

ALTER TABLE cars
DROP COLUMN color;  -- Remove the color column
```

### Delete a row in a Table

**If you omit the WHERE clause, all records in the table will be deleted! The WHERE clause specifies which record(s) should be deleted.**

```sql
DELETE FROM cars
WHERE brand = 'Volvo';

DELETE FROM cars; -- delete all rows in a table without deleting the table (!Caution when use)
```

The DELETE statement removes rows one at a time and records an entry in the transaction log for each deleted row. TRUNCATE TABLE removes the data by deallocating the data pages used to store the table data and records only the page deallocations in the transaction log. DELETE command is slower than TRUNCATE command

```sql
TRUNCATE FROM cars; -- delete all records in the cars table
```

The DROP TABLE statement is used to drop an existing table in a database. It will result in loss of all information stored in the table!

```sql
DROP TABLE cars; -- delete the cars table
```

### SERIAL vs UUID for primary key

```sql
CREATE TABLE categories (
  category_id UUID NOT NULL PRIMARY KEY,  -- or "category_id SERIAL NOT NULL PRIMARY KEY"
  category_name VARCHAR(255),
  description VARCHAR(255),
  customer_id INT,
  order_date DATE
);

INSERT INTO orders (order_id, customer_id, order_date)
VALUES
  (10248, 90, '2021-07-04'),
  (10249, 81, '2021-07-05');
```


| Scenario | Use `SERIAL` | Use `UUID` |
|----------|------------|----------|
| Simple applications, local database | ✓ | X |
| High-performance applications with frequent inserts | ✓ | X |
| Multi-database, distributed systems | X | ✓ |
| Microservices, event-driven architecture | X | ✓ |
| Public-facing API (avoid exposing IDs) | X | ✓ |
| Sharding or database federation | X | ✓ |
| Security-sensitive applications | X | ✓ |

--- 

**Use `UUID`** if your app is distributed, sharded, or needs better security.

### Select Data

```sql
SELECT customer_name, country FROM customers; -- get customer_name and country columns from customers table

SELECT * FROM customers; -- select all columns
```

### Operators

- ```=``` : Equal to
- ```<``` : Less than
- ```>``` : Greater than
- ```<=``` : Less than or equal to
- ```>=``` : Greater than or equal to
- ```<>``` : Not equal to
- ```!=``` : Not equal to
- ```LIKE``` : Check if a value matches a pattern (case sensitive)
- ```ILIKE``` : Check if a value matches a pattern (case insensitive)
- ```AND``` : Logical AND
- ```OR``` : Logical OR
- ```IN``` : Check if a value is between a range of values
- ```BETWEEN``` : Check if a value is between a range of values
- ```IS NULL``` : Check if a value is NULL
- ```NOT``` : Makes a negative result e.g. NOT LIKE, NOT IN, NOT BETWEEN

The percent sign ```%```, represents zero, one, or multiple characters.
The underscore sign ```_```, represents one single character.

```sql
SELECT * FROM cars
WHERE brand != 'Volvo';  -- Return all records where the brand is NOT 'Volvo'

SELECT * FROM cars
WHERE model LIKE 'M%'; -- Return all records where the model STARTS with a capital 'M' (case sensitive)

SELECT * FROM cars
WHERE brand NOT ILIKE 'm%'; -- Return all records where the brand does NOT start with 'm' (case insensitive)

SELECT * FROM customers
WHERE customer_name LIKE '%A%'; -- Return all customers with a name that contains the letter 'A'

SELECT * FROM customers
WHERE customer_name LIKE '%en'; -- Return all customers with a name that ends with the phrase 'en'

SELECT * FROM customers
WHERE city LIKE 'L_nd__'; -- Return all customers from a city that starts with 'L' followed by one wildcard character (character or number), then 'nd' and then two wildcard characters

SELECT * FROM cars
WHERE brand = 'Volvo' AND year = 1968; -- Return all records where the brand is 'Volvo' and the year is 1968

SELECT * FROM cars
WHERE brand = 'Volvo' OR year = 1975; -- Return all records where the brand is 'Volvo' OR the year is 1975

SELECT * FROM cars
WHERE brand IN ('Volvo', 'Mercedes', 'Ford'); -- Return all records where the brand is present in this list: ('Volvo', 'Mercedes', 'Ford')

SELECT * FROM cars
WHERE brand NOT IN ('Volvo', 'Mercedes', 'Ford'); -- Return all records where the brand is NOT present in this list: ('Volvo', 'Mercedes', 'Ford')

SELECT * FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders); -- Return all customers that have an order in the orders table

SELECT * FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders); -- Return all customers that have NOT placed any orders in the orders table

SELECT * FROM cars
WHERE year BETWEEN 1970 AND 1980; -- Return all records where the year is between 1970 and 1980, includes 1970 and 1980

SELECT * FROM Products
WHERE product_name BETWEEN 'Pavlova' AND 'Tofu'; -- Select all products alphabetically between 'Pavlova' and 'Tofu'

SELECT * FROM orders
WHERE order_date BETWEEN '2023-04-12' AND '2023-05-05'; -- Select all orders between 12. of April 2023 and 5. of May 2023

SELECT * FROM cars
WHERE year NOT BETWEEN 1970 AND 1980; -- Return all records where the year is NOT between 1970 and 1980 (the result would not include cars made in 1970 and 1980)

SELECT * FROM cars
WHERE model IS NULL; -- Return all records where the model is NULL

SELECT * FROM cars
WHERE model IS NOT NULL; -- Return all records where the model is NOT null
```

### Select Distinct

The SELECT DISTINCT statement is used to return only distinct (**different**) **values**. Inside a table, a column often contains many duplicate values and sometimes you only want to list the different (distinct) values.

```sql
SELECT DISTINCT country FROM customers; -- Select only the DISTINCT values from the country column in the customers table

SELECT COUNT(DISTINCT country) FROM customers; -- Return the number of different countries there are in the customers table
```

### Aliases

SQL aliases are used to give a table, or a column in a table, a temporary name. Aliases are often used to make column names more readable.

```sql
SELECT customer_id AS id
FROM customers;

SELECT customer_id id -- you can skip the AS keyword and get the same result
FROM customers;

SELECT product_name || unit AS product -- Concatenate two fields and call them product
FROM products;

SELECT product_name || ' ' || unit AS product -- Concatenate, with space
FROM products;
```

### Column Operation Keywords/Clauses

```sql
SELECT * FROM products
ORDER BY price; -- Sort the table by price, tring values will order alphabetically

SELECT * FROM products
ORDER BY price DESC; -- Sort the table by price, in descending order

SELECT * FROM customers
LIMIT 20; -- Return only the 20 first records from the customers table

SELECT * FROM customers
LIMIT 20 OFFSET 40; -- Return 20 records, starting from the 41th record

SELECT MIN(price) FROM products; -- Return the lowest price in the products table

SELECT MAX(price) FROM products; -- Return the highest price in the products table

SELECT MIN(price) AS lowest_price
FROM products;  -- Return the lowest price, and name the column lowest_price

SELECT COUNT(customer_id)
FROM customers;  -- Return the number of customers from the customers table, NULL values are not counted.

SELECT COUNT(customer_id)
FROM customers
WHERE city = 'London'; -- Return the number of customers from London

SELECT SUM(quantity)
FROM order_details; -- Return the total amount of ordered items, NULL values are ignored

SELECT AVG(price)
FROM products; -- Return the average price of all the products in the products table, NULL values are ignored

SELECT AVG(price)::NUMERIC(10,2)
FROM products; -- Return the average price of all the products, rounded to 2 decimals, if the result is 28.8663636363636364, output will be 28.87
```

### **Join**

A JOIN clause is used to combine rows from two or more tables, based on a related column between them.

```products``` table
```bash
product_id |  product_name  | category_id
------------+----------------+-------------
         33 | Geitost        |           4
         34 | Sasquatch Ale  |           1
         35 | Steeleye Stout |           1
         36 | Inlagd Sill    |           8
```

```categories``` table

```bash
 category_id | category_name
-------------+----------------
           1 | Beverages
           2 | Condiments
           3 | Confections
           4 | Dairy Products
```

! The category_id column in the products table **refers** to the category_id in the categories table. 

```sql
SELECT product_id, product_name, category_name
FROM products
INNER JOIN categories ON products.category_id = categories.category_id;
```

```Output```

```bash
 product_id |  product_name  | category_name
------------+----------------+----------------
         33 | Geitost        | Dairy Products
         34 | Sasquatch Ale  | Beverages
         35 | Steeleye Stout | Beverages
         36 | Inlagd Sill    | Seafood
```

- ```INNER JOIN``` : Returns records that have matching values in both tables
- ```LEFT JOIN``` : Returns all records from the left table, and the matched records from the right table
- ```RIGHT JOIN``` : Returns all records from the right table, and the matched records from the left table
- ```FULL JOIN``` : Returns all records when there is a match in either left or right table

### **Inner Join**

```JOIN``` and ```INNER JOIN``` will give the same result!

```testproducts``` table

```bash
testproduct_id |      product_name      | category_id
----------------+------------------------+-------------
              1 | Johns Fruit Cake       |           3
              2 | Marys Healthy Mix      |           9
              3 | Peters Scary Stuff     |          10
              4 | Jims Secret Recipe     |          11
              5 | Elisabeths Best Apples |          12
              6 | Janes Favorite Cheese  |           4
              7 | Billys Home Made Pizza |          13
              8 | Ellas Special Salmon   |           8
              9 | Roberts Rich Spaghetti |           5
            10 | Mias Popular Ice        |          14
```

```categories``` table

```bash
 category_id | category_name  |                       description
-------------+----------------+------------------------------------------------------------
           1 | Beverages      | Soft drinks, coffees, teas, beers, and ales
           2 | Condiments     | Sweet and savory sauces, relishes, spreads, and seasonings
           3 | Confections    | Desserts, candies, and sweet breads
           4 | Dairy Products | Cheeses
           5 | Grains/Cereals | Breads, crackers, pasta, and cereal
           6 | Meat/Poultry   | Prepared meats
           7 | Produce        | Dried fruit and bean curd
           8 | Seafood        | Seaweed and fish
```

* By using INNER JOIN we will not get the records where there is not a match, we will only get the records that matches both tables

```sql
SELECT testproduct_id, product_name, category_name
FROM testproducts
INNER JOIN categories ON testproducts.category_id = categories.category_id; 
```

```Output```

```bash
testproduct_id |      product_name      | category_name
----------------+------------------------+----------------
              1 | Johns Fruit Cake       | Confections
              6 | Janes Favorite Cheese  | Dairy Products
              8 | Ellas Special Salmon   | Seafood
              9 | Roberts Rich Spaghetti | Grains/Cereals
```

### **Left Join**

The ```LEFT JOIN``` keyword selects **ALL** records from the "left" table, and the **matching records** from the "**right**" table. The result is **0** records from the right side if there is no match.

```testproducts``` table

```bash
 testproduct_id |      product_name      | category_id
----------------+------------------------+-------------
              1 | Johns Fruit Cake       |           3
              2 | Marys Healthy Mix      |           9
              3 | Peters Scary Stuff     |          10
              4 | Jims Secret Recipe     |          11
              5 | Elisabeths Best Apples |          12
              6 | Janes Favorite Cheese  |           4
              7 | Billys Home Made Pizza |          13
              8 | Ellas Special Salmon   |           8
              9 | Roberts Rich Spaghetti |           5
            10 | Mias Popular Ice        |          14
```

```categories``` table

```bash
 category_id | category_name  |                       description
-------------+----------------+------------------------------------------------------------
           1 | Beverages      | Soft drinks, coffees, teas, beers, and ales
           2 | Condiments     | Sweet and savory sauces, relishes, spreads, and seasonings
           3 | Confections    | Desserts, candies, and sweet breads
           4 | Dairy Products | Cheeses
           5 | Grains/Cereals | Breads, crackers, pasta, and cereal
           6 | Meat/Poultry   | Prepared meats
           7 | Produce        | Dried fruit and bean curd
           8 | Seafood        | Seaweed and fish
```

```sql
SELECT testproduct_id, product_name, category_name
FROM testproducts
LEFT JOIN categories ON testproducts.category_id = categories.category_id;
```

```Output```

```bash
 testproduct_id |      product_name      | category_name
----------------+------------------------+----------------
              1 | Johns Fruit Cake       | Confections
              2 | Marys Healthy Mix      |
              3 | Peters Scary Stuff     |
              4 | Jims Secret Recipe     |
              5 | Elisabeths Best Apples |
              6 | Janes Favorite Cheese  | Dairy Products
              7 | Billys Home Made Pizza |
              8 | Ellas Special Salmon   | Seafood
              9 | Roberts Rich Spaghetti | Grains/Cereals
             10 | Mias Popular Ice       |
```

### **Right Join**

The ```RIGHT JOIN``` keyword makes the same thing with ```LEFT JOIN``` but for right side table will have **ALL** records.

### **Full Join**

The ```FULL JOIN``` keyword selects **ALL** records from both tables, even if there is not a match. For rows with a match the values from both tables are available, if there is not a match the empty fields will get the value **NULL**.

### **Cross Join**

The CROSS JOIN keyword matches ALL records from the "left" table with EACH record from the "right" table. That means that all records from the "right" table will be returned for each record in the "left" table.

```sql
SELECT testproduct_id, product_name, category_name
FROM testproducts
CROSS JOIN categories;
```

The ```CROSS JOIN``` method will return ALL categories for EACH testproduct, meaning that it will return 80 rows (10 * 8). [Output](https://www.w3schools.com/postgresql/postgresql_cross_join.php)

### **Union**

The UNION operator is used to combine the result-set of two or more queries. The queries in the union must follow these rules:

- They must have the **same number of columns**
- The columns must have the **same data types**
- The columns must be in the **same order**

```sql
SELECT product_id, product_name
FROM products
UNION
SELECT testproduct_id, product_name
FROM testproducts
ORDER BY product_id;
```

```Output```

```bash
product_id |           product_name
------------+----------------------------------
          1 | Johns Fruit Cake
          1 | Chais
          2 | Marys Healthy Mix
          2 | Chang
          3 | Peters Scary Stuff
          3 | Aniseed Syrup
          4 | Jims Secret Recipe
          4 | Chef Antons Cajun Seasoning
          5 | Chef Antons Gumbo Mix
          5 | Elisabeths Best Apples
          . | ...
          . | ...
          . | ...
```

With the ```UNION``` operator, if some rows in the two queries returns the exact same result, only one row will be listed, because ```UNION``` selects only **distinct** values. Use ```UNION ALL``` to return **duplicate** values!

### **Group By**

The ```GROUP BY``` clause is often used with aggregate functions like ```COUNT()```, ```MAX()```, ```MIN()```, ```SUM()```, ```AVG()``` to group the result-set by one or more columns.

```sql
SELECT COUNT(customer_id), country
FROM customers
GROUP BY country;
```

```Output```

```bash
 count |   country
-------+-------------
     3 | Argentina
     5 | Spain
     2 | Switzerland
     3 | Italy
     . | ...
     . | ...
     . | ...
```

```sql
SELECT customers.customer_name, COUNT(orders.order_id)
FROM orders
LEFT JOIN customers ON orders.customer_id = customers.customer_id
GROUP BY customer_name;
```

```Output```

```bash
           customer_name            | count
------------------------------------+-------
 Magazzini Alimentari Riuniti       |    10
 Wilman Kala                        |     8
 The Cracker Box                    |     3
 ...                                |     .
 ...                                |     .
 ...                                |     .
```

### **Having**

The ```HAVING``` clause was added to SQL because the ```WHERE``` clause **cannot** be used with **aggregate functions**. Aggregate functions are often used with ```GROUP BY``` clauses, and by adding ```HAVING``` we can write condition like we do with ```WHERE``` clauses.

```sql
-- List only countries that are represented more than 5 times
SELECT COUNT(customer_id), country
FROM customers
GROUP BY country
HAVING COUNT(customer_id) > 5;
```

```Output```

```bash
count | country
-------+---------
    13 | USA
    11 | France
     9 | Brazil
     7 | UK
    11 | Germany
```

```sql
-- Lists only orders with a total price of 400$ or more
SELECT order_details.order_id, SUM(products.price)
FROM order_details
LEFT JOIN products ON order_details.product_id = products.product_id
GROUP BY order_id
HAVING SUM(products.price) > 400.00;

-- Lists customers that have ordered for 1000$ or more
SELECT customers.customer_name, SUM(products.price)
FROM order_details
LEFT JOIN products ON order_details.product_id = products.product_id
LEFT JOIN orders ON order_details.order_id = orders.order_id
LEFT JOIN customers ON orders.customer_id = customers.customer_id
GROUP BY customer_name
HAVING SUM(products.price) > 1000.00;
```

### **Exists**

The ```EXISTS``` operator returns TRUE if the sub query returns one or more records. It properly handle **multiple** results, possibly by using an aggregation or limiting the subquery to one row. Without ```EXISTS``` operator query assumes that the subquery will return a **single value** (which is not guaranteed when a customer has multiple orders).

```sql
SELECT customers.customer_name
FROM customers
WHERE EXISTS (
  SELECT order_id
  FROM orders
  WHERE customer_id = customers.customer_id
);
```

```Output```

```bash
            customer_name
------------------------------------
 Alfreds Futterkiste
 Ana Trujillo Emparedados y helados
 ...
 ...
```

```sql
-- Return all customers that is NOT represented in the orders table
SELECT customers.customer_name
FROM customers
WHERE NOT EXISTS (
  SELECT order_id
  FROM orders
  WHERE customer_id = customers.customer_id
);
```
### **Any**

The ```ANY``` operator allows you to perform a **comparison** between a **single** column value and a **range of other values**.

- Returns a **Boolean** value as a result
- Returns **TRUE** if ANY of the sub query values meet the condition

```sql
-- List products that have ANY records in the order_details table with a quantity larger than 120
SELECT product_name
FROM products
WHERE product_id = ANY (
  SELECT product_id
  FROM order_details
  WHERE quantity > 120
);
```

### **All**

```ALL``` means that the condition will be **true** only if the operation is **true** for **all** values in the range.

- Returns a **Boolean** value as a result
- Returns **TRUE** if ALL of the sub query values meet the condition
- Is used with ```SELECT```, ```WHERE``` and ```HAVING``` statements

```sql
-- List the products if ALL the records in the order_details with quantity larger than 10.
SELECT product_name
FROM products
WHERE product_id = ALL (
  SELECT product_id
  FROM order_details
  WHERE quantity > 10
);
```

### **Case**

The ```CASE``` expression goes through **conditions** and returns a value when the first condition is met (like an **if-then-else** statement). Once a condition is **true**, it will **stop** reading and return the result. If no conditions are true, it returns the value in the ```ELSE``` clause. If there is no ELSE part and no conditions are true, it returns **NULL**.

```sql
-- Return specific values if the price meets a specific condition
SELECT product_name,
CASE
  WHEN price < 10 THEN 'Low price product'
  WHEN price > 50 THEN 'High price product'
ELSE
  'Normal product'
END
FROM products;

-- Same example, but with an alias for the case column
SELECT product_name,
CASE
  WHEN price < 10 THEN 'Low price product'
  WHEN price > 50 THEN 'High price product'
ELSE
  'Normal product'
END AS "price category"
FROM products;
```

...TO_BE_CONTINUED...
https://www.w3schools.com/sql/sql_constraints.asp

