#Orders Analytics in Snowflake
# Orders / Customers / Products — SQL Query Reference

This document describes four queries built against the `ORDERS`, `CUSTOMERS`, and `PRODUCTS` tables.

## Environment

Built in **Snowflake**, in the `KASI_MART` database, `ORDERS` schema.

| Table | Rows | Columns |
|---|---|---|
| `CUSTOMERS` | 50 | `customer_id`, `customer_name`, `email`, `province`, `signup_date` |
| `PRODUCTS` | 20 | `product_id`, `product_name`, `category`, `unit_price` |
| `ORDERS` | 150 | `order_id`, `customer_id`, `product_id`, `order_date`, `quantity` |

---

## Setup: creating the database, schema, and tables

### 1. Create the database and schema

```sql
CREATE DATABASE "KASI_MART";
CREATE SCHEMA "KASI_MART"."ORDERS";
```

This gives `KASI_MART` three schemas by default (`INFORMATION_SCHEMA`, `PUBLIC`, and the new `ORDERS` schema), with the three data tables living under `KASI_MART.ORDERS`.

### 2. Create the tables

```sql
CREATE TABLE "KASI_MART"."ORDERS"."CUSTOMERS" (
    customer_id   VARCHAR,
    customer_name VARCHAR,
    email         VARCHAR,
    province      VARCHAR,
    signup_date   DATE
);

CREATE TABLE "KASI_MART"."ORDERS"."PRODUCTS" (
    product_id   VARCHAR,
    product_name VARCHAR,
    category     VARCHAR,
    unit_price   NUMBER
);

CREATE TABLE "KASI_MART"."ORDERS"."ORDERS" (
    order_id    VARCHAR,
    customer_id VARCHAR,
    product_id  VARCHAR,
    order_date  DATE,
    quantity    NUMBER
);
```

### 3. Load data from CSV

Each table was populated from a matching CSV file (`customers.csv`, `products.csv`, `orders.csv`) using Snowsight's **Load Data into Table** wizard (Database Explorer → table → **Load Data**), which:

1. Uploads the CSV and lets you confirm/edit the inferred schema (column names, data types) against a preview of the file.
2. Lets you choose an error-handling behavior for bad rows (e.g. "Only load valid data from the file").
3. Generates and runs the underlying `COPY INTO` statement.

The wizard's generated SQL looks like this (shown here for `CUSTOMERS`; `PRODUCTS` and `ORDERS` follow the same pattern):

```sql
CREATE TEMP FILE FORMAT "KASI_MART"."ORDERS"."temp_file_format"
    TYPE = CSV
    SKIP_HEADER = 1
    FIELD_DELIMITER = ','
    TRIM_SPACE = TRUE
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    REPLACE_INVALID_CHARACTERS = TRUE
    DATE_FORMAT = AUTO
    TIME_FORMAT = AUTO
    TIMESTAMP_FORMAT = AUTO;

COPY INTO "KASI_MART"."ORDERS"."CUSTOMERS"
FROM (
    SELECT $1, $2, $3, $4, $5
    FROM '@"KASI_MART"."ORDERS"."__snowflake_temp_import_files__"'
)
FILES = ('customers.csv')
FILE_FORMAT = 'KASI_MART.ORDERS.temp_file_format'
ON_ERROR = ABORT_STATEMENT;
```

Same process for `products.csv` → `PRODUCTS` and `orders.csv` → `ORDERS`, each matched to its table's column order. After loading, `KASI_MART.ORDERS.Tables` shows all three tables with their row counts (50 / 20 / 150) confirming the load succeeded.

---

## Query 1 — Order detail with line revenue

Every order joined to customer name, product name, category, and a calculated `LINE_REVENUE` (quantity × unit price).

```sql
SELECT 
    o.ORDER_ID,
    o.QUANTITY,
    c.CUSTOMER_NAME,
    p.PRODUCT_NAME,
    p.CATEGORY,
    (o.QUANTITY * p.UNIT_PRICE) AS LINE_REVENUE
FROM ORDERS o
INNER JOIN CUSTOMERS c ON o.customer_id = c.CUSTOMER_ID
INNER JOIN PRODUCTS p ON o.product_id = p.PRODUCT_ID;
```

**Notes:** No aggregation — one row per order line. `LINE_REVENUE` is a simple row-level calculation, not a `SUM`.

---

## Query 2 — Total revenue per customer

```sql
SELECT
    o.ORDER_ID,
    c.CUSTOMER_NAME,
    SUM(o.QUANTITY * p.UNIT_PRICE) AS LINE_REVENUE
FROM ORDERS o
INNER JOIN CUSTOMERS c ON o.CUSTOMER_ID = c.CUSTOMER_ID
INNER JOIN PRODUCTS p ON o.PRODUCT_ID = p.PRODUCT_ID
GROUP BY o.ORDER_ID, c.CUSTOMER_NAME;
```

**Notes:** Grouped by `ORDER_ID` and `CUSTOMER_NAME`, so this actually returns revenue **per order**, not a single total per customer. If you want one row per customer summing *all* their orders, drop `ORDER_ID` from both the `SELECT` and `GROUP BY`:

```sql
SELECT
    c.CUSTOMER_NAME,
    SUM(o.QUANTITY * p.UNIT_PRICE) AS TOTAL_REVENUE
FROM ORDERS o
INNER JOIN CUSTOMERS c ON o.CUSTOMER_ID = c.CUSTOMER_ID
INNER JOIN PRODUCTS p ON o.PRODUCT_ID = p.PRODUCT_ID
GROUP BY c.CUSTOMER_NAME;
```

---

## Query 3 — Total revenue per product category

```sql
SELECT 
    p.CATEGORY,
    SUM(o.QUANTITY * p.UNIT_PRICE) AS LINE_REVENUE
FROM PRODUCTS p
INNER JOIN ORDERS o ON p.PRODUCT_ID = o.PRODUCT_ID
GROUP BY p.CATEGORY
ORDER BY p.CATEGORY;
```

**Notes:** One row per category, revenue summed across all orders containing products in that category.

---

## Query 4 — Top 5 customers by total spend

```sql
SELECT 
    c.CUSTOMER_ID,
    c.CUSTOMER_NAME,
    SUM(o.QUANTITY * p.UNIT_PRICE) AS LINE_REVENUE
FROM CUSTOMERS c
INNER JOIN ORDERS o ON c.CUSTOMER_ID = o.CUSTOMER_ID
INNER JOIN PRODUCTS p ON o.PRODUCT_ID = p.PRODUCT_ID
GROUP BY c.CUSTOMER_ID, c.CUSTOMER_NAME
ORDER BY LINE_REVENUE DESC
LIMIT 5;
```

**Notes:** Added `ORDER BY LINE_REVENUE DESC` — without it, `LIMIT 5` just returns *some* 5 customers, not necessarily the top spenders. Also qualified `UNIT_PRICE` as `p.UNIT_PRICE` for clarity (it was unprefixed in the original, which works only because it's unambiguous, but explicit is safer).

---

