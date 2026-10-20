
<h1 align="center">Retail_Fashion_Cleaning</h1>

### Excel:
1. Splitting values ​​into columns. 
Data -> Text to columns -> Comma

Before
<img width="793" height="570" alt="image" src="https://github.com/user-attachments/assets/b75342d6-697a-4168-a12b-9e16b9bb3819" />


After

<img width="688" height="599" alt="image" src="https://github.com/user-attachments/assets/8057c7c8-1bb6-45a5-98f1-876d1c8ec4f1" />

The pictures present the sales_orders table. I followed the same approach for the other tables. 

### PostgreSQL:

Create database Retail_fashion


Because the dataset is heavily polluted with data quality issues, a staging table with columns stored as TEXT was created to allow the data to be imported successfully.

```
CREATE TABLE sales_orders (
    order_id TEXT,
    order_date TEXT,
    customer_id TEXT,
    country TEXT,
    product_id TEXT,
    quantity TEXT,
    unit_price TEXT,
    discount_pct TEXT,
    status TEXT
);

CREATE TABLE products (
    product_id TEXT,
    category TEXT,
    sub_category TEXT,
    base_price TEXT,
    launch_date TEXT
);

CREATE TABLE inventory (
    product_id TEXT,
    warehouse_country TEXT,
    stock_quantity TEXT,
    last_stock_update TEXT
);
```

I opened my folder in notebook to veryfy delimiters. It is semicolon ; 
Import data -> Format - CSV, Encoding - UTF8

<h2 align="center">Sales_order</h2>

### Deduplication - order_id

```
SELECT COUNT(order_id) AS all_orders, COUNT(DISTINCT order_id) AS distinct_orders
FROM sales_orders;
```
<img width="242" height="69" alt="image" src="https://github.com/user-attachments/assets/7a5cfaa4-5c58-4e85-b4f9-8d393c126465" />

As we can see, the data requires deduplication. 

```
CREATE VIEW dedup_orders AS
WITH dedup_orders AS (
    SELECT *,
       ROW_NUMBER() OVER ( PARTITION BY order_id
	   ORDER BY order_id
    ) AS rn
    FROM sales_orders
)
SELECT *
FROM dedup_orders
WHERE rn = 1;
```
Description:
* Groups records by order_id using PARTITION BY order_id.
* Assigns a sequential number (rn) to each row within the same order_id group using the ROW_NUMBER() function.
* The first occurrence of each order_id receives rn = 1, while subsequent duplicates receive rn = 2, rn = 3, etc.
* The WHERE rn = 1 filter keeps only the first occurrence of each order.
* The result is saved as a View named dedup_orders, providing a deduplicated dataset without modifying the original source table.

### Date Format Standardization - order_date

Dane posiadają różne formaty takie jak YYYY-MM-DD, DD.MM.YYYY oraz błędne dane jak 2024-13-40, 2024-12.
W celu umożliwienie przeprowadzenia analizy ustandaryzowałam daty do formatu DD.MM.YYYY, w Rekordach gdzie nie było wskazanych dni wstawiłam 1. 

data quality validation

```
SELECT *,
    CASE
        -- DD.MM.YYYY
        WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])\.(0[1-9]|1[0-2])\.\d{4}$'
            THEN TO_DATE(order_date, 'DD.MM.YYYY')

        -- DD-MM-YYYY
        WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])-(0[1-9]|1[0-2])-\d{4}$'
            THEN TO_DATE(order_date, 'DD-MM-YYYY')

        -- YYYY-MM-DD
        WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'
            THEN TO_DATE(order_date, 'YYYY-MM-DD')

        -- YYYY-MM (ustawiamy 1. dzień miesiąca)
        WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])$'
            THEN TO_DATE(order_date || '-01', 'YYYY-MM-DD')

        -- błędne rekordy (np. 2024-13-40) -> NULL
        ELSE NULL
    END AS order_date_val
FROM dedup_orders;
```


```
WITH parsed AS (
    SELECT
        order_date,
        CASE
            -- DD.MM.YYYY
            WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])\.(0[1-9]|1[0-2])\.\d{4}$'
                THEN TO_DATE(order_date, 'DD.MM.YYYY')

            -- DD-MM-YYYY
            WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])-(0[1-9]|1[0-2])-\d{4}$'
                THEN TO_DATE(order_date, 'DD-MM-YYYY')

            -- YYYY-MM-DD
            WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'
                THEN TO_DATE(order_date, 'YYYY-MM-DD')

            -- YYYY-MM (1. dzień miesiąca)
            WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])$'
                THEN TO_DATE(order_date || '-01', 'YYYY-MM-DD')

            ELSE NULL
        END AS order_date_val
    FROM dedup_orders
)
SELECT
    COUNT(*)                                                    AS all_values,
    COUNT(*) FILTER (WHERE order_date_val IS NULL)              AS null_values,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE order_date_val IS NULL) / COUNT(*),
        2
    )                                                           AS percentage_of_all_nulls
FROM parsed;
```

<img width="386" height="76" alt="image" src="https://github.com/user-attachments/assets/99a8de21-86fd-4078-8183-fdcf1527c4ed" />
