## Create a Basic Shipping project tables (Fill them with mock-data)

```sql
-- Drop tables if they exist
DROP TABLE IF EXISTS payments, order_items, orders, products, customers CASCADE;

-- Customers table
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(100),
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    registered_at TIMESTAMP DEFAULT NOW()
);

-- Products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    description TEXT,
    price NUMERIC(10, 2),
    stock_quantity INTEGER
);

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    order_date TIMESTAMP DEFAULT NOW(),
    status VARCHAR(50) DEFAULT 'Pending'
);

-- Order items (many-to-many between orders and products)
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    product_id INT REFERENCES products(id),
    quantity INT,
    price_at_purchase NUMERIC(10,2)
);

-- Payments table
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    payment_date TIMESTAMP DEFAULT NOW(),
    amount NUMERIC(10, 2),
    method VARCHAR(50),
    status VARCHAR(50)
);


-- Insert customers
INSERT INTO customers (full_name, email, phone) VALUES
('Alice Johnson', 'alice@example.com', '555-1234'),
('Bob Smith', 'bob@example.com', '555-2345'),
('Charlie Lee', 'charlie@example.com', '555-3456');

-- Insert products
INSERT INTO products (name, description, price, stock_quantity) VALUES
('Wireless Mouse', 'A comfortable wireless mouse', 25.99, 50),
('Gaming Keyboard', 'RGB backlit mechanical keyboard', 79.99, 30),
('USB-C Cable', '1m USB-C to USB-C cable', 9.99, 100),
('Laptop Stand', 'Adjustable aluminum laptop stand', 39.95, 20);

-- Insert orders
INSERT INTO orders (customer_id, order_date, status) VALUES
(1, NOW() - INTERVAL '10 days', 'Shipped'),
(2, NOW() - INTERVAL '5 days', 'Processing'),
(1, NOW() - INTERVAL '1 day', 'Pending');

-- Insert order items
INSERT INTO order_items (order_id, product_id, quantity, price_at_purchase) VALUES
(1, 1, 2, 25.99),   -- Alice bought 2 Wireless Mouse
(1, 3, 1, 9.99),    -- Alice bought 1 USB-C Cable
(2, 2, 1, 79.99),   -- Bob bought 1 Gaming Keyboard
(3, 4, 1, 39.95);   -- Alice bought 1 Laptop Stand

-- Insert payments
INSERT INTO payments (order_id, payment_date, amount, method, status) VALUES
(1, NOW() - INTERVAL '9 days', 61.97, 'Credit Card', 'Paid'),
(2, NOW() - INTERVAL '4 days', 79.99, 'PayPal', 'Paid'),
(3, NULL, NULL, NULL, 'Unpaid'); -- Not paid yet

-- Visualize tables
SELECT * FROM customers;
SELECT * FROM products;
SELECT * FROM orders;
SELECT * FROM order_items;
SELECT * FROM payments;
```

## Create a Postgres Function
### get_customer_info_with_orders_by_id(p_customer_id INT)
```sql
CREATE OR REPLACE FUNCTION get_customer_info_with_orders_by_id(p_customer_id INT)
RETURNS TABLE(
  customer_name VARCHAR,
  customer_email VARCHAR,
  order_id INT,
  order_date TIMESTAMP,
  order_status VARCHAR,
  product_name VARCHAR,
  quantity INT,
  price_at_purchase NUMERIC,
  payment_status VARCHAR,
  payment_amount NUMERIC
)
AS $$
BEGIN
  RETURN QUERY
  SELECT c.full_name,
    c.email,
    o.id,
    o.order_date,
    o.status,
    p.name,
    oi.quantity,
    oi.price_at_purchase,
    pay.status,
    pay.amount
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  LEFT JOIN order_items oi ON o.id = oi.order_id
  LEFT JOIN products p ON p.id = oi.product_id
  LEFT JOIN payments pay ON pay.order_id = o.id
  WHERE c.id = p_customer_id
  ORDER BY o.order_date DESC;
END;
$$ LANGUAGE plpgsql; 

SELECT * FROM get_customer_info_with_orders_by_id(1);
```
OUTPUT:

| customer_name | customer_email     | order_id | order_date         | order_status | product_name     | quantity | price_at_purchase | payment_status | payment_amount |
|---------------|--------------------|----------|---------------------|--------------|------------------|----------|-------------------|----------------|----------------|
| Alice Johnson | alice@example.com  | 3        | 2025-04-21 10:12:00 | Pending      | Laptop Stand     | 1        | 39.95             | Unpaid         | null           |
| Alice Johnson | alice@example.com  | 1        | 2025-04-11 14:00:00 | Shipped      | Wireless Mouse   | 2        | 25.99             | Paid           | 61.97          |
| Alice Johnson | alice@example.com  | 1        | 2025-04-11 14:00:00 | Shipped      | USB-C Cable       | 1        | 9.99              | Paid           | 61.97          |

---

## Aggregations & GROUP BY
### Find total spending per customer
```sql
-- Run on Server
SELECT 
    c.full_name,
    SUM(COALESCE(oi.quantity, 0) * COALESCE(oi.price_at_purchase, 0)) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id, c.full_name;

-- OR Use Function and call it on the client
CREATE OR REPLACE FUNCTION get_total_spendings_per_customer()
RETURNS TABLE(
  customer_name VARCHAR,
  total_spent NUMERIC
)
AS $$
BEGIN 
  RETURN QUERY
  SELECT 
      c.full_name,
      SUM(COALESCE(oi.quantity, 0) * COALESCE(oi.price_at_purchase, 0)) AS total_spent -- COALESCE for NULL handling, work with 0 instead of NULL values for calculations
  FROM customers c
  JOIN orders o ON o.customer_id = c.id
  JOIN order_items oi ON oi.order_id = o.id
  GROUP BY c.id, c.full_name;
END;
$$ LANGUAGE plpgsql; 

SELECT * FROM get_total_spendings_per_customer();
```

## Subqueries and CTEs 
### Show customers with more than 1 order
```sql
SELECT full_name, email FROM customers
WHERE id IN (
    SELECT customer_id FROM orders
    GROUP BY customer_id
    HAVING COUNT(*) > 1
);

-- OR alternate with CTE
WITH order_counts AS (
    SELECT customer_id, COUNT(*) AS total_orders
    FROM orders
    GROUP BY customer_id
)
SELECT c.full_name, oc.total_orders
FROM customers c
JOIN order_counts oc ON c.id = oc.customer_id
WHERE oc.total_orders > 1;
```

## Nested Joins & Data Shaping
### List each order with items in JSON format
```sql
SELECT
  o.id AS order_id,
  o.order_date,
  json_agg(json_build_object(
    'product', p.name,
    'quantity', oi.quantity,
    'price', oi.price_at_purchase
  )) AS items
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
GROUP BY o.id, o.order_date
```

OUTPUT:
| order_id | order_date     | items |
|---------------|--------------------|----------|
| 1 | 2025-04-12 06:56:15.34903  | [{"product" : "Wireless Mouse", "quantity" : 2, "price" : 25.99}, {"product" : "USB-C Cable", "quantity" : 1, "price" : 9.99}]        |
| 2 | 2025-04-17 06:56:15.34903  | [{"product" : "Gaming Keyboard", "quantity" : 1, "price" : 79.99}]        |
| 3 | 2025-04-21 06:56:15.34903  | [{"product" : "Laptop Stand", "quantity" : 1, "price" : 39.95}]        |


## UPSERT (Insert or Update)
### Add product, but if it exists update stock and price
```sql
INSERT INTO products (name, description, price, stock_quantity) 
VALUES ('USB-C Cable', 'Fast charging cable', 8.99, 80)
ON CONFLICT (name) DO UPDATE -- name should have UNIQUE tag, otherwise it will give "No unique or exclusion constraint matching the ON CONFLICT" error message
SET price = EXCLUDED.price, stock_quantity = products.stock_quantity + EXCLUDED.stock_quantity;

SELECT * FROM products;
```

## Indexing
### Create index on email (if not already unique)
```sql
-- Used for read performance improvements in large tables
CREATE INDEX idx_customers_email ON customers(email);
```

## Materialized Views
### Cache top 5 customers by spending
```sql
-- Fast dashboard load time in frontend/client.
CREATE MATERIALIZED VIEW top_customers AS
SELECT 
    c.id, c.full_name, SUM(oi.quantity * oi.price_at_purchase) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
GROUP BY c.id
ORDER BY total_spent DESC
LIMIT 5;

-- Refresh View
REFRESH MATERIALIZED VIEW top_customers;

SELECT * FROM top_customers
```

## Trigger for Audit Log
### Track changes to product stock
```sql
CREATE TABLE product_stock_audit(
    id SERIAL PRIMARY KEY,
    product_id INT,
    old_stock INT,
    new_stock INT,
    changed_at TIMESTAMP DEFAULT NOW()
);

-- Create log_stock_change function 
CREATE OR REPLACE FUNCTION log_stock_change()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.stock_quantity <> OLD.stock_quantity THEN -- Not equal
        INSERT INTO product_stock_audit (prodcut_id, old_stock, new_stock)
        VALUES (OLD.id, OLD_stock_quantity, NEW.stock_quantity);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create Trigger
CREATE TRIGGER trigger_log_stock_change
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION log_stock_change();
```

## Custom Function for JSON Output (for frontend)
### get_order_detail_json(p_order_id INT)
```sql
CREATE OR REPLACE FUNCTION get_order_detail_json(p_order_id INT)
RETURNS JSON AS $$
DECLARE
    result JSON;
BEGIN
    SELECT json_build_object(
        'order_id', o.id,
        'customer', c.full_name,
        'items', json_agg(json_build_object(  -- json_agg: JSON Array
            'product', p.name,
            'quantity', oi.quantity,
            'price', oi.price_at_purchase
        )),
        'total_price', SUM(oi.quantity * oi.price_at_purchase)
    ) INTO result
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    JOIN order_items oi ON o.id = oi.order_id
    JOIN products p ON p.id = oi.product_id
    WHERE o.id = p_order_id
    GROUP BY o.id, c.full_name ;

    RETURN result;
END;
$$ LANGUAGE plpgsql;


SELECT get_order_detail_json(1);
```

## Refresh Materialized Views on a Schedule in Supabase

### Option 1: Supabase Edge Function + Scheduled Webhook (via Supabase Scheduler)

#### Lets create a products_view Materialized view (snapshot)
```sql
CREATE MATERIALIZED VIEW products_view AS
SELECT
    p.id,
    p.name,
    p.description,
    p.price,
    p.stock_quantity,
    c.name AS category_name,
    AVG(r.rating) AS avg_rating,
    COUNT(r.id) AS rating_count
FROM products p
LEFT JOIN product_categories c ON p.category_id = c.id
LEFT JOIN product_reviews r ON p.id = r.product_id
GROUP BY p.id, c.name;
```

#### Create Edge Function to Refresh the View
```ts
// supabase/functions/refresh_product_view.ts
import { serve } from "https://deno.land/std/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js";

serve(async (req) => {
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")! // needs elevated permissions
  );

  // Call raw SQL to refresh view
  const { error } = await supabase.rpc("refresh_products_mv");
  if (error) return new Response(JSON.stringify(error), { status: 500 });

  return new Response("Materialized view refreshed successfully!");
});
```

#### Create the RPC Function in Supabase SQL Editor
```sql
CREATE OR REPLACE FUNCTION refresh_products_mv()
RETURNS void AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY products_view;
END;
$$ LANGUAGE plpgsql;
```

#### Schedule Cron Job via Supabase Scheduler
- Go to your Supabase Project > Edge Functions > Scheduler
- Create a new job like:
```makefile
Name: refresh-products-daily
Function: refresh_product_view
Frequency: Every 15 mins / hourly / nightly
```

#### Optional - Refresh After Insert Trigger (semi-live)
```sql
CREATE OR REPLACE FUNCTION refresh_mv_on_product_insert()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM refresh_products_mv();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_refresh_mv
AFTER INSERT ON products
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_mv_on_product_insert();
```

## Filtering Products

### 1. **Use SQL `WHERE` + Indexes**

```sql
SELECT * FROM products
WHERE 
  name ILIKE '%laptop%' AND
  price BETWEEN 500 AND 1500 AND
  stock_quantity > 0
ORDER BY price ASC
LIMIT 20 OFFSET 0;
```

### Add proper indexes:

```sql
CREATE INDEX idx_products_name ON products (name);
CREATE INDEX idx_products_price ON products (price);
CREATE INDEX idx_products_stock ON products (stock_quantity);
```

> 📌 Postgres will skip full scans and use B-trees or bitmap indexes for fast retrieval.

---

### 2. **Search + Filter API in Supabase**
In your Supabase client:

```ts
const { data } = await supabase
  .from('products')
  .select('*')
  .ilike('name', `%laptop%`)
  .gt('stock_quantity', 0)
  .gte('price', 500)
  .lte('price', 1500)
  .order('price', { ascending: true })
  .range(0, 19);  // Pagination
```

---

### 3. **Paginate Always (even on filters)**

```ts
.range((page - 1) * perPage, page * perPage - 1)
```

Or in raw SQL:

```sql
LIMIT 20 OFFSET 0;
```

> Prevents slowdowns and allows infinite scroll or pagination

---

### 4. **Add Indexes on Filter Fields**
Add `btree` indexes on:
- `name`, `price`, `stock_quantity`, `category_id`, `rating`

And optionally **GIN Index** for advanced text search:

```sql
CREATE INDEX idx_products_name_gin ON products USING gin (to_tsvector('english', name));
```

Then query with:

```sql
SELECT * FROM products
WHERE to_tsvector('english', name) @@ to_tsquery('wireless & mouse');
```

---

## What About Materialized Views?

Materialized views are best for **pre-filtered summary data**, not for **highly dynamic filters**.

| Scenario                        | Use MV?     | Why? |
|---------------------------------|-------------|------|
| Product search with name/price | ❌ No       | Needs dynamic filters |
| Top 10 best-selling products   | ✅ Yes      | Static summary, cacheable |
| Dashboard summary cards        | ✅ Yes      | Perfect MV use |
| Per-user recommendations       | ☑️ Maybe    | If precomputed per user |

### Solution:
**Use normal indexed tables for filters + pagination. Use MVs for dashboards, charts, or popular views.**

---

## Fast Dynamic Filtering With Supabase

### Example Supabase API Query:

```ts
const query = supabase.from("products").select("*");

if (filter.name) query.ilike("name", `%${filter.name}%`);
if (filter.minPrice) query.gte("price", filter.minPrice);
if (filter.maxPrice) query.lte("price", filter.maxPrice);
if (filter.inStockOnly) query.gt("stock_quantity", 0);

query.order("price", { ascending: true }).range(0, 19);

const { data, error } = await query;
```

This is clean, fast, and professional.

---

## Summary

| Feature                | Recommended |
|------------------------|-------------|
| Server-side filters    | ✅ Yes       |
| Index filter columns   | ✅ Yes       |
| Use MVs for dashboards | ✅ Yes       |
| Use pagination         | ✅ Yes       |
| Filter on MVs          | ❌ Not for dynamic UI filters |
| Client-side filtering  | ❌ Never (on large sets) |

---

We can build it query-by-query.

| Feature                      | RPC Function (SQL)                      | GIN Full-Text Search                  |
|-----------------------------|-----------------------------------------|--------------------------------------|
| 🔍 Use case                 | Advanced multi-criteria filtering        | Fast keyword search (title, desc)    |
| 🧩 Filter by price, stock   | ✅ Excellent                            | ❌ Hard to do efficiently            |
| 🔤 Keyword search (name)    | ✅ OK (ILIKE) / ☑️ Good (with GIN used inside RPC) | ✅✅ Excellent                       |
| ⚙️ Custom logic             | ✅ Full control                         | ❌ Limited to text matching           |
| 📦 Return paginated result | ✅ Easy with LIMIT/OFFSET               | ✅ Easy                               |
| 🚀 Performance on large data| ✅ Good (if indexed properly)           | ✅✅ Very fast for text search        |
| 🔄 Dynamic combinations     | ✅ Perfect (filters, sort, pagination) | ☑️ Maybe (with combined strategies)   |
| 🔐 Supabase client use      | ❌ Needs RPC call (`supabase.rpc`)      | ✅ Native query methods available     |
| 🔧 Maintenance              | ✅ Central control in DB                | ✅ Just need index + query            |

---

## Use GIN Full-Text Search When:

- You want fast **search by keywords** in `name`, `description`, etc.
- You're building an **autocomplete or instant search bar**
- Filtering by price, stock isn't required **in the same query**

### Example GIN Use:

```sql
CREATE INDEX idx_products_search ON products
USING GIN (to_tsvector('english', name || ' ' || description));
```

```sql
SELECT *
FROM products
WHERE to_tsvector('english', name || ' ' || description)
      @@ plainto_tsquery('wireless mouse')
LIMIT 20 OFFSET 0;
```

> ⚠️ You can't filter price/stock as easily unless you combine the queries with CTEs or subqueries.

---

## Use RPC Function When:

- You want to **combine multiple filters**: `name`, `price`, `stock`, `category`, `rating`, etc.
- You want **pagination**, **sorting**, and optional filters
- You want **all logic centralized** in one secure SQL function
- You want to expose a **clean single endpoint** (like a REST route)

### Example: Multi-Filter RPC Function

```sql
CREATE OR REPLACE FUNCTION get_filtered_products(
    p_name TEXT DEFAULT NULL,
    p_min_price NUMERIC DEFAULT NULL,
    p_max_price NUMERIC DEFAULT NULL,
    p_in_stock BOOLEAN DEFAULT TRUE,
    p_limit INT DEFAULT 20,
    p_offset INT DEFAULT 0
)
RETURNS TABLE (
    id INT,
    name TEXT,
    price NUMERIC,
    stock_quantity INT,
    description TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT id, name, price, stock_quantity, description
    FROM products
    WHERE
        (p_name IS NULL OR name ILIKE '%' || p_name || '%') AND
        (p_min_price IS NULL OR price >= p_min_price) AND
        (p_max_price IS NULL OR price <= p_max_price) AND
        (NOT p_in_stock OR stock_quantity > 0)
    ORDER BY price ASC
    LIMIT p_limit OFFSET p_offset;
END;
$$ LANGUAGE plpgsql;
```

Then in Supabase:

```ts
const { data } = await supabase
  .rpc("get_filtered_products", {
    p_name: "mouse",
    p_min_price: 20,
    p_max_price: 100,
    p_limit: 10,
    p_offset: 0
  });
```

---

## Real-World Recommendation (Best of Both)

> Use **RPC** for your main filter & pagination endpoint  
> Use **GIN full-text** *inside that RPC* for keyword search

Yes, you can combine both like this:

```sql
AND (p_name IS NULL OR to_tsvector('english', name) @@ plainto_tsquery(p_name))
```

But keep in mind: **text search is exact/match-based**, not fuzzy unless using `pg_trgm`.

---

## Conclusion: Which Is More Efficient?

| Scenario                                         | Winner       |
|--------------------------------------------------|--------------|
| 🔎 Search by name/desc (fast keyword search)     | GIN          |
| 📦 Filter by price, stock, etc.                  | RPC          |
| 🧠 Combined smart filtering + pagination          | RPC (with GIN inside) |
| 🔥 Real-time performance at scale                 | RPC with indexes (and optional MV) |

---

# Fully optimized PostgreSQL RPC function usage

## 1. Enable `pg_trgm` + Index Setup

```sql
-- Enable pg_trgm extension (needed for fuzzy search)
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Index for trigram fuzzy matching (on product name)
CREATE INDEX idx_products_name_trgm ON products
USING GIN (name gin_trgm_ops);

-- Index for category
CREATE INDEX idx_products_category_id ON products (category_id);

-- Index for price, stock, rating
CREATE INDEX idx_products_price ON products (price);
CREATE INDEX idx_products_stock ON products (stock_quantity);
CREATE INDEX idx_products_rating ON products (rating);
```

> These indexes will make WHERE filters super fast, even with thousands of products.

---

## 2. Create `get_filtered_products` RPC Function

```sql
CREATE OR REPLACE FUNCTION get_filtered_products(
  p_name TEXT DEFAULT NULL,
  p_min_price NUMERIC DEFAULT NULL,
  p_max_price NUMERIC DEFAULT NULL,
  p_in_stock BOOLEAN DEFAULT TRUE,
  p_category_id INT DEFAULT NULL,
  p_min_rating NUMERIC DEFAULT NULL,
  p_limit INT DEFAULT 20,
  p_offset INT DEFAULT 0
)
RETURNS TABLE (
  id INT,
  name TEXT,
  description TEXT,
  price NUMERIC,
  stock_quantity INT,
  rating NUMERIC,
  category_id INT
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    id, name, description, price, stock_quantity, rating, category_id
  FROM products
  WHERE
    (p_name IS NULL OR name % p_name) AND  -- pg_trgm fuzzy match
    (p_min_price IS NULL OR price >= p_min_price) AND
    (p_max_price IS NULL OR price <= p_max_price) AND
    (NOT p_in_stock OR stock_quantity > 0) AND
    (p_category_id IS NULL OR category_id = p_category_id) AND
    (p_min_rating IS NULL OR rating >= p_min_rating)
  ORDER BY price ASC
  LIMIT p_limit OFFSET p_offset;
END;
$$ LANGUAGE plpgsql;
```

> `%` is the **pg_trgm operator** for fuzzy matching. `name % 'mouse'` will match “mouze”, “mousse”, etc.

---

## 3. Call It from Supabase Client (RPC)

```ts
const { data, error } = await supabase.rpc("get_filtered_products", {
  p_name: "mouse",
  p_min_price: 10,
  p_max_price: 100,
  p_in_stock: true,
  p_category_id: 2,
  p_min_rating: 4,
  p_limit: 10,
  p_offset: 0
});
```

---

## Supabase `.select()` Filtering vs RPC Function

| Feature                          | Supabase `.from().select()`             | Supabase `rpc("get_filtered_products")` |
|----------------------------------|----------------------------------------|------------------------------------------|
| ✅ Filtering on columns          | Yes (with `.eq`, `.gte`, `.ilike`)     | Yes (custom logic)                       |
| 🔍 Fuzzy search (`pg_trgm`)      | ❌ Not supported natively              | ✅ Supported                              |
| 🔥 Performance on large datasets | ❌ Client builds complex query strings | ✅ Query pre-compiled in DB               |
| 📦 Complex logic in SQL          | ❌ Hard to maintain                    | ✅ Clean + versioned in SQL               |
| 🔐 Security (RLS)                | ✅ Supported                           | ✅ Supported (like normal tables/views)   |
| 🧪 Testing / reuse               | ❌ Repeated in code                    | ✅ Centralized, testable logic            |
| 💡 Pagination + filters combo    | ☑️ Possible but verbose               | ✅ Simple and clean                       |
| 🧠 Maintainability               | ❌ JS and query strings everywhere     | ✅ Logic in SQL with defaults             |

---

## Professional Recommendation

| Use case                         | Recommended Strategy          |
|----------------------------------|-------------------------------|
| Basic list with 1-2 filters      | `.from().select()`            |
| Search bar, fuzzy text           | `rpc() + pg_trgm + GIN`       |
| Multi-filter product list        | ✅ `rpc()` (fast, maintainable) |
| Dashboard or repeated queries    | `rpc()` or Materialized Views |

---

# Personalized Product Advice

## Problem Breakdown: Personalized Product Advice

- Per-user recommendations (based on order behavior)
- Filtered by user purchase stats: min/max prices, categories, brands, etc.
- Needs to load fast on dashboard (real-time feel)

---

### Professional Architecture

| Layer                     | Responsibility                                | Tool                         |
|--------------------------|-----------------------------------------------|------------------------------|
| 🔄 Materialized View     | Precompute **user profile insights**          | `user_order_stats_mv`       |
| ⚙️ RPC Function          | Fetch personalized recommendations dynamically| `get_personalized_products(user_id)` |
| 🧠 pg_trgm + GIN Indexes | Optional fuzzy match for product name         | Indexes                     |
| 🧪 Optional Caching      | For anonymous users or low-changing users     | Redis / App memory / SSR    |

---

## 1. Materialized View – `user_order_stats_mv`

Precompute **user-based purchase behavior** so it loads fast.

```sql
CREATE MATERIALIZED VIEW user_order_stats_mv AS
SELECT 
  c.id AS user_id,
  MIN(oi.price_at_purchase) AS min_price,
  MAX(oi.price_at_purchase) AS max_price,
  COUNT(DISTINCT p.category_id) AS category_diversity,
  ARRAY_AGG(DISTINCT p.category_id) AS favorite_categories
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
GROUP BY c.id;
```

You can refresh this nightly or via trigger after checkout.

---

## 2. RPC Function – `get_personalized_products`

Use the stats in the MV to fetch personalized products dynamically.

```sql
CREATE OR REPLACE FUNCTION get_personalized_products(p_user_id INT)
RETURNS TABLE (
  id INT,
  name TEXT,
  price NUMERIC,
  rating NUMERIC,
  category_id INT
) AS $$
BEGIN
  RETURN QUERY
  SELECT p.id, p.name, p.price, p.rating, p.category_id
  FROM products p
  JOIN user_order_stats_mv stats ON stats.user_id = p_user_id
  WHERE 
    p.price BETWEEN stats.min_price AND stats.max_price
    AND p.category_id = ANY(stats.favorite_categories)
    AND p.stock_quantity > 0
  ORDER BY rating DESC, price ASC
  LIMIT 10;
END;
$$ LANGUAGE plpgsql;
```
    
## 3. Call It from Supabase Client (RPC)

```ts
const { data } = await supabase.rpc("get_personalized_products", {
  p_user_id: currentUser.id
});
```

This will instantly return 5–10 products tailored to the user.

---

## Optional Enhancements

### 1. Include `pg_trgm` Matching for Similar Product Names

```sql
AND p.name % 'wireless mouse'
```

### 2. Background Refresh for MV

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_stats_mv;
```

Use Supabase Edge Functions + Scheduler (as we discussed earlier).

### 3. SSR/CSR Fallback Caching (Optional for Web)

If a user hasn’t ordered yet or isn’t logged in, fallback to:

```sql
SELECT * FROM products ORDER BY rating DESC LIMIT 10;
```

---

### Final Verdict: Which Approach Is Best?

| Approach                       | Performance | Personalization | Scalability | Use case                             |
|-------------------------------|-------------|------------------|-------------|--------------------------------------|
| Materialized View Only        | ✅ Fast      | ❌ Static         | ✅ High     | Global dashboards (sales, popular)  |
| RPC + MV (hybrid approach)    | ✅✅ Fast     | ✅ Smart          | ✅✅ Best   | Personalized, real-time dashboards   |
| Supabase .select() dynamic    | ❌ Slow      | ✅ Smart          | ☑️ OK       | Small data only                      |
| Dynamic view (no MV)          | ❌ Slowest   | ✅ Smart          | ❌ Bad      | Not suitable at scale                |

---
