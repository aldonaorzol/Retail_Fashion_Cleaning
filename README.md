 <h1 align="center">Retail_Fashion_Cleaning</h1>

### Data analysis:
https://github.com/aldonaorzol/Retail_Fashion_Analysis

### Tools: 
PostgreSQL, Microsoft Excel,  Copilot, Claude

### Description:
The source data contained numerous data quality issues, including inconsistent formats, duplicate records, missing values, and invalid entries. Using SQL, I performed data profiling, data cleansing, and data validation tasks as part of the ETL workflow to enhance data integrity and prepare the datasets for reliable business analysis and reporting. The process included standardizing country names, validating numerical fields, identifying potential outliers, converting data types, standardizing formats, and verifying consistency across sales, product, and inventory datasets. Additional data quality checks were conducted to ensure data accuracy, completeness, and compliance with business rules.

### Splitting values ​​into columns in Excel:

Data -> Text to columns -> Comma

Before


<img width="793" height="570" alt="image" src="https://github.com/user-attachments/assets/b75342d6-697a-4168-a12b-9e16b9bb3819" />


After

<img width="688" height="599" alt="image" src="https://github.com/user-attachments/assets/8057c7c8-1bb6-45a5-98f1-876d1c8ec4f1" />

The pictures present the sales_orders table. I followed the same approach for the other tables. 

### Create database Retail_fashion in PostgreSQL:

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

I opened the source file in a text editor (Notepad) to verify the delimiter. It is a semicolon (;).
Import settings: Format: CSV, Encoding: UTF-8, Delimiter: ;

<h2 align="center">Data Quality</h2>

- Did I import all the records correctly?
```
SELECT 'sales_orders' AS table_name,
       COUNT(*)        AS all_records,
       COUNT(order_id) AS without_null
FROM sales_orders

UNION ALL

SELECT 'products',
       COUNT(*),
       COUNT(product_id)
FROM products

UNION ALL

SELECT 'inventory',
       COUNT(*),
       COUNT(product_id)
FROM inventory;
```

<img width="325" height="119" alt="image" src="https://github.com/user-attachments/assets/eed79eeb-3ce9-4bff-af8d-9a935a5818f2" />

The table above confirms that all records were successfully imported into PostgreSQL. Row counts match the source files, and the key columns contain no NULL values.

- Whether there are any missing values.

```
SELECT *
FROM sales_orders
WHERE TRIM(order_id) = ''
   OR TRIM(customer_id) = ''
   OR TRIM(order_date) = ''
   OR TRIM(product_id) = ''
   OR TRIM(quantity) = ''
   OR TRIM(unit_price) = '';

SELECT *
FROM products
WHERE TRIM(product_id) = ''
   OR TRIM(category) = ''
   OR TRIM(sub_category) = ''
   OR TRIM(base_price) = ''
   OR TRIM(launch_date) = '';

SELECT *
FROM inventory
WHERE TRIM(product_id) = ''
   OR TRIM(warehouse_country) = ''
   OR TRIM(stock_quantity) = ''
   OR TRIM(last_stock_update) = '';
```
The validation checked for empty strings (= '') and returned 0 rows, even though the source file clearly had missing values. PostgreSQL had converted those blanks to NULL during import.

- Whether duplicate records exist.

sales_orders table
```
SELECT COUNT(order_id) AS all_orders, COUNT(DISTINCT order_id) AS distinct_orders
FROM sales_orders;
```
<img width="242" height="69" alt="image" src="https://github.com/user-attachments/assets/7a5cfaa4-5c58-4e85-b4f9-8d393c126465" />

inventory table

```
SELECT
COUNT(product_id) AS all_products, 
COUNT(DISTINCT product_id) AS distinct_products
FROM inventory;
```

<img width="267" height="68" alt="image" src="https://github.com/user-attachments/assets/91743a41-d283-4f9d-a8f7-df7b4c366949" />

As we can see, the data requires deduplication.
  
- Whether the values stored in each column are reasonable (e.g., prices, quantities).

The data includes unrealistic values such as zero prices, outliers, impossible dates and unreadable discount values. These problematic records made up a small share of rows, except in discount_pct.

<h2 align="center">TABLE sales_orders</h2>

### DEDUPLICATION - order_id

***Problem:***
- The column contains no NULL values (260,780 rows, all non-null).
- It contains 260,000 unique values in the range 1–260,000, so it is not unique (780 duplicated rows).

```
SELECT
    COUNT(*) AS all_rows,
    COUNT(order_id) AS non_null,
    COUNT(DISTINCT order_id) AS unique_ids,
    MIN(order_id::INT) AS min_id,
    MAX(order_id::INT) AS max_id
FROM sales_orders;
```

<img width="426" height="69" alt="image" src="https://github.com/user-attachments/assets/adf1423f-1468-4fd1-a225-5144b83a4f45" />

***Solutions:***

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

- Groups records by order_id using PARTITION BY
- Assigns a sequential number (rn) to each row within the same group using the ROW_NUMBER() function.
- The WHERE rn = 1 filter keeps only the first occurrence 
- The result is saved as a View, providing a deduplicated dataset without modifying the source table.

### DATE FORMAT STANDARDIZATION - order_date 

***Problem:***
The order_date column mixed several text formats: YYYY-MM-DD, YYYY/MM/DD, DD-MM-YYYY, DD.MM.YYYY and YYYY-MM. Some values were also impossible dates, such as 2024-13-40. 

```
WITH parsed AS (
    SELECT
        order_date,
        CASE
            -- DD.MM.YYYY lub DD-MM-YYYY
            WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])[.-](0[1-9]|1[0-2])[.-]\d{4}$'
                THEN TO_DATE(REPLACE(order_date, '.', '-'), 'DD-MM-YYYY')

            -- YYYY-MM-DD
            WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'
                THEN TO_DATE(order_date, 'YYYY-MM-DD')

            -- YYYY/MM/DD
            WHEN order_date ~ '^\d{4}/(0[1-9]|1[0-2])/(0[1-9]|[12]\d|3[01])$'
                THEN TO_DATE(order_date, 'YYYY/MM/DD')

            -- YYYY-MM
            WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])$'
                THEN TO_DATE(order_date || '-01', 'YYYY-MM-DD')

            ELSE NULL
        END AS order_date_val
    FROM dedup_orders
)
SELECT
    COUNT(*)                                                        AS all_values,
    COUNT(*) - COUNT(order_date_val)                                AS null_values,
    ROUND(100.0 * (COUNT(*) - COUNT(order_date_val)) / COUNT(*), 2) AS percentage_of_all_nulls
FROM parsed;
```

<img width="386" height="76" alt="image" src="https://github.com/user-attachments/assets/99a8de21-86fd-4078-8183-fdcf1527c4ed" />

***Solutions:***

```
CREATE VIEW orders_clean AS
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

        -- YYYY/MM/DD 
        WHEN order_date ~ '^\d{4}/(0[1-9]|1[0-2])/(0[1-9]|[12]\d|3[01])$'
            THEN TO_DATE(order_date, 'YYYY/MM/DD')

        -- YYYY-MM (I set the 1st day of the month)
        WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])$'
            THEN TO_DATE(order_date || '-01', 'YYYY-MM-DD')

        -- incorrect records (e.g. 2024-13-40) -> NULL
        ELSE NULL
    END AS order_date_val,

    CASE
        -- a genuine absence in the source (distinguishing it from unrecognized values)
        WHEN order_date IS NULL THEN 'source_null'
        WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])\.(0[1-9]|1[0-2])\.\d{4}$' THEN 'dd.mm.yyyy'
        WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])-(0[1-9]|1[0-2])-\d{4}$'   THEN 'dd-mm-yyyy'
        WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'   THEN 'yyyy-mm-dd'
        WHEN order_date ~ '^\d{4}/(0[1-9]|1[0-2])/(0[1-9]|[12]\d|3[01])$'   THEN 'yyyy/mm/dd'
        WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])$'                         THEN 'yyyy-mm'
        -- does not match any pattern (e.g., 2024-13-40, not_a_date)
        ELSE 'UNRECOGNIZED'
    END AS order_date_status
FROM dedup_orders;
```

- I used a CTE to parse dates first and then calculate the share of invalid values on top of the parsed result.
- I measured what percentage of all records could not be converted to a date. This is 0.73% (NULL values and unrecognized formats combined), so I left them as NULL and excluded them from time-based analysis.
- Each format was matched with a regular expression with literal dots escaped (\.), so impossible dates such as 2024-13-40 matched no pattern and became NULL.
- I converted the text values to the DATE type using CASE (the default display format is YYYY-MM-DD).

### DATA STANDARDIZATION - country

***Problem:***
The country column in the orders data contained the same country written in many different ways: codes (PL, DE), full names (Poland, Germany), local-language names (Polska, Deutschland), alternative names (Holland, Czechia) and inconsistent capitalization or stray spaces. This made grouping and reporting by country unreliable, since one country was split across several categories.

```
SELECT DISTINCT country
FROM sales_orders
ORDER BY country;
```

<img width="144" height="491" alt="image" src="https://github.com/user-attachments/assets/aac07b53-5e42-4a18-91dc-f0e25ffd508f" />

***Solutions:***

```
CREATE VIEW clear_country AS
SELECT *,
    CASE
        WHEN UPPER(TRIM(country)) IN ('AT', 'AUSTRIA')                            THEN 'Austria'
        WHEN UPPER(TRIM(country)) IN ('CZ', 'CZECH', 'CZECHIA', 'CZECH REPUBLIC') THEN 'The Czech Republic'
        WHEN UPPER(TRIM(country)) IN ('DE', 'GER', 'GERMANY', 'DEUTSCHLAND')      THEN 'Germany'
        WHEN UPPER(TRIM(country)) IN ('ES', 'SPAIN')                              THEN 'Spain'
        WHEN UPPER(TRIM(country)) IN ('FR', 'FRANCE')                             THEN 'France'
		WHEN UPPER(TRIM(country)) IN ('IT', 'ITALY')                              THEN 'Italy'
        WHEN UPPER(TRIM(country)) IN ('NL', 'NETHERLANDS', 'HOLLAND')             THEN 'The Netherlands'
        WHEN UPPER(TRIM(country)) IN ('PL', 'POL', 'POLAND', 'POLSKA')            THEN 'Poland'
        WHEN UPPER(TRIM(country)) IN ('SE', 'SWEDEN')                             THEN 'Sweden'
        WHEN UPPER(TRIM(country)) IN ('SK', 'SLOVAK', 'SLOVAKIA')                 THEN 'Slovakia'
    ELSE NULL
    END AS clear_country
FROM orders_clean;
```
- I built a CASE expression that maps every variant to a single standardized country name.
- Each value is first normalized with UPPER(TRIM(country)), so the mapping does not depend on capitalization or leading/trailing spaces.

### OUTLIERS - quantity

***Problem:***
The quantity column contains an outlier. To make an informed decision, I calculated what percentage of records contain erroneous data.

```
SELECT
    COUNT(*) AS total_records,
    COUNT(*) FILTER (
        WHERE quantity IN ('608', '0')
    ) AS outliers,
    ROUND(
        COUNT(*) FILTER (
            WHERE quantity IN ('608', '0')
        ) * 100.0
        / COUNT(*),
        2
    ) AS outlier_percentage
FROM sales_orders; 
```

<img width="352" height="71" alt="image" src="https://github.com/user-attachments/assets/63805611-6d82-4022-a226-527b2c870dce" />

```
SELECT quantity, COUNT(*)
FROM sales_orders
GROUP BY quantity
ORDER BY 2 DESC;
```

<img width="178" height="417" alt="image" src="https://github.com/user-attachments/assets/7c844665-a011-4960-8366-397167e09bf0" />

***Solutions:***

```
CREATE VIEW quantity_cl AS
SELECT *,
    CASE
        WHEN quantity IN ('0', '608') THEN NULL
        ELSE quantity
    END AS quantity_cl
FROM clear_country; 
```

- Calculated the percentage of records containing outlier values (quantity = 608 and quantity = 0) to assess their impact on the dataset.
- As these values accounted for only 1% of all records and were identified as data quality issues, they were replaced with NULL values.

### REPLACING INVALID VALUES WITH NULL - unit_price

***Problem:***
The column contains zero values, which is an incorrect value for unit prices. It also contained 8 values (< 0.01% of 260,780 rows) stored as dates (e.g. 28.mar, 30.sty) instead of numbers.

```
SELECT
    COUNT(*)                                                  AS total_records,
    COUNT(*) FILTER (WHERE unit_price = '0.0')                    AS impossible_values,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE unit_price = '0.0') / COUNT(*),
        2
    )                                                         AS impo_val_per
FROM quantity_cl;
```

<img width="377" height="68" alt="image" src="https://github.com/user-attachments/assets/4ab0be09-8bb0-4f89-8238-e4fb10d496d4" />

```
SELECT order_id, unit_price
FROM sales_orders
WHERE unit_price !~ '^\d+(\.\d+)?$';
```

<img width="200" height="248" alt="image" src="https://github.com/user-attachments/assets/164e95ff-63a0-4447-95cf-9ef4cfcdd5f8" />


***Solutions:***

```
CREATE VIEW unit_price_cl AS
SELECT *,
    CASE
     -- non-numeric values ​​e.g., 28.mar, 30.sty -> NULL
     WHEN unit_price !~ '^\d+(\.\d+)?$' THEN NULL
    -- price 0.0 -> NULL
	WHEN unit_price = '0.0' THEN NULL
	ELSE unit_price
	END AS unit_price_cl
FROM quantity_cl; 
```
    

- I did not find any specific dependencies (e.g., the zero values were not concentrated in a particular country, product, or date).
- Impossible values accounted for 0.62% of all records, so I decided to replace them with NULL rather than delete the rows.

### DATA RECOVERY - discount_pct 

***Problem:***
The discount_pct column was corrupted in 49.2% of records. The column contains dates, values ​​of 0.0, and NULLs. 

```
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE discount_pct !~ '^\d+(\.\d+)?$') AS corrupted,
    ROUND(100.0 * COUNT(*) FILTER (WHERE discount_pct !~ '^\d+(\.\d+)?$') / COUNT(*), 2) AS pct
FROM sales_orders
WHERE discount_pct IS NOT NULL;
```
<img width="264" height="70" alt="image" src="https://github.com/user-attachments/assets/b9d35beb-dd8d-4f9d-8086-fff854f53216" />


***Solution:***

```
CREATE VIEW discount_cl AS
SELECT *,
    CASE
        -- ordinary numbers
        WHEN discount_pct ~ '^\d+(\.\d+)?$' THEN discount_pct::NUMERIC

        -- DD.mon  ->  D.MM  (np. 09.mar -> 9.03)
        WHEN discount_pct ~* '^\d{1,2}\.(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)$' THEN
            (SPLIT_PART(discount_pct, '.', 1)::INT
             + CASE LOWER(SPLIT_PART(discount_pct, '.', 2))
                 WHEN 'sty' THEN 0.01 WHEN 'lut' THEN 0.02 WHEN 'mar' THEN 0.03
                 WHEN 'kwi' THEN 0.04 WHEN 'maj' THEN 0.05 WHEN 'cze' THEN 0.06
                 WHEN 'lip' THEN 0.07 WHEN 'sie' THEN 0.08 WHEN 'wrz' THEN 0.09
                 WHEN 'paź' THEN 0.10 WHEN 'lis' THEN 0.11 WHEN 'gru' THEN 0.12
               END)::NUMERIC

        -- mon.NN  ->  M.NN  (np. sie.61 -> 8.61)
        WHEN discount_pct ~* '^(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)\.\d{1,2}$' THEN
            (CASE LOWER(SPLIT_PART(discount_pct, '.', 1))
                 WHEN 'sty' THEN 1 WHEN 'lut' THEN 2 WHEN 'mar' THEN 3
                 WHEN 'kwi' THEN 4 WHEN 'maj' THEN 5 WHEN 'cze' THEN 6
                 WHEN 'lip' THEN 7 WHEN 'sie' THEN 8 WHEN 'wrz' THEN 9
                 WHEN 'paź' THEN 10 WHEN 'lis' THEN 11 WHEN 'gru' THEN 12
             END
             || '.' || SPLIT_PART(discount_pct, '.', 2))::NUMERIC
    END AS discount_clean
FROM unit_price_cl;
```

Check that valid values fall between 0 and 100

<img width="201" height="75" alt="image" src="https://github.com/user-attachments/assets/812cb051-90a5-4177-aa00-46fc8d9f2111" />


- I opened the file in a notebook to verify whether the issue occurred during the data import process or if the data had been retrieved incorrectly. It turned out that the issue originated from the source data. In a real-world scenario, I would verify whether the data had been imported correctly from the source system. However, in this case, the dataset was artificially generated by AI, making such verification impossible.
- I decided to replace month names with numerical values. I am aware that this approach may introduce inaccuracies because a value such as "09.Mar" could be interpreted as either 09.30 or 09.03.

### DATA STANDARDIZATION - status

***Problem:***
The status column contained the same order status written in many different ways: SHIP, shipped, SHIPPED, DONE, done, Done, complete, Completed, COMPLETED, CANCELLED. The inconsistent spelling and capitalization split each status into several categories, which made counting and grouping orders by status unreliable.

```
SELECT DISTINCT status
FROM sales_orders
ORDER BY status;
```

<img width="129" height="310" alt="image" src="https://github.com/user-attachments/assets/0c3397dc-bb84-4dfb-8f25-237ff37852ef" />


***Solutions:***

```
CREATE VIEW status_cl AS
SELECT *,
    CASE
	WHEN UPPER(TRIM(status)) IN ('SHIP', 'SHIPPED')                THEN 'Shipped'
	WHEN UPPER(TRIM(status)) IN ('DONE', 'COMPLETE', 'COMPLETED')  THEN 'Completed'
	WHEN UPPER(TRIM(STATUS)) IN ('CANCELLED')                      THEN 'Cancelled'
    ELSE NULL
    END AS status_cl
FROM discount_cl;
```

- I standardized the column into three consistent statuses (Shipped, Completed, Cancelled) using a CASE expression.
- Each value is first normalized with UPPER(TRIM(status)), so the mapping does not depend on capitalization or stray spaces.

<h2 align="center">TABLE products</h2>

### DATA PROFILING - product_id 
***Problem:***
Before using product_id as a key for joining the products, inventory, and sales_orders tables, I needed to confirm that the column was complete and unique.

```
SELECT
    COUNT(*) AS all_rows,
    COUNT(product_id) AS non_null,
    COUNT(DISTINCT product_id) AS unique_ids,
    MIN(product_id::INT) AS min_id,
    MAX(product_id::INT) AS max_id
FROM products;
```

<img width="424" height="70" alt="image" src="https://github.com/user-attachments/assets/3c813641-64f2-42ea-b59e-81f14bfbd990" />

***Solutions:***
- As we can see upper the column has unique values. 
- I ran a single profiling query that checks the row count, the number of non-NULL values, the number of distinct values, and the minimum and maximum ID
- The column required no cleaning.
  
### DATE FORMAT STANDARDIZATION - launch_date 

***Problem:***
- 91 NULL values ​​(3.64%), 6 records marked as 'not_a_date', 4 records containing an invalid date—2024-13-40 (following the YYYY-MM-DD pattern, but with month 13 and day 40)—and 3 records in YYYY-MM format (valid, but lacking a day).

```
SELECT launch_date, COUNT(*) AS rows_count
FROM products
WHERE launch_date IS NULL
   OR launch_date !~ '^\d{2}\.\d{2}\.\d{4}$'
GROUP BY launch_date
ORDER BY rows_count DESC;
```

<img width="233" height="188" alt="image" src="https://github.com/user-attachments/assets/8f3941fe-188b-4640-9165-19d20e2b1356" />


***Solutions:***

```
CREATE OR REPLACE VIEW date_cl AS
SELECT *,
    CASE
        -- DD.MM.YYYY
        WHEN launch_date ~ '^(0[1-9]|[12]\d|3[01])\.(0[1-9]|1[0-2])\.\d{4}$'
            THEN TO_DATE(launch_date, 'DD.MM.YYYY')

        -- YYYY-MM-DD
        WHEN launch_date ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'
            THEN TO_DATE(launch_date, 'YYYY-MM-DD')

        -- YYYY/MM/DD
        WHEN launch_date ~ '^\d{4}/(0[1-9]|1[0-2])/(0[1-9]|[12]\d|3[01])$'
            THEN TO_DATE(launch_date, 'YYYY/MM/DD')

        -- YYYY-MM (1st day of the month assumed)
        WHEN launch_date ~ '^\d{4}-(0[1-9]|1[0-2])$'
            THEN TO_DATE(launch_date || '-01', 'YYYY-MM-DD')

        -- incorrect records (e.g. 2024-13-40, not_a_date) -> NULL
        ELSE NULL
        END AS date_cl
FROM products;
```

- Created the view that converts the text into a date.
- Supported formats: `DD.MM.YYYY`, `YYYY-MM-DD`, `YYYY/MM/DD` and `YYYY-MM` (the 1st day of the month is assumed).
- Invalid and missing values (91 NULLs, 6 × not_a_date, 4 × 2024-13-40) are set to NULL.

<h2 align="center">TABLE inventory</h2>

### DATA STANDARDIZATION - warehouse_country

Problem:
The country column in the inventory table described the same warehouse location in many different ways: codes (PL, DE, CZ), full English names (Poland, Germany), local-language names (Polska, Deutschland), alternative names (Czechia) and inconsistent capitalization or stray spaces. One country was therefore split across several categories, which made grouping and reporting stock by country unreliable.

```
SELECT DISTINCT warehouse_country, COUNT (warehouse_country)
FROM inventory
GROUP BY warehouse_country;
```

<img width="243" height="443" alt="image" src="https://github.com/user-attachments/assets/94eaf4f1-af9b-4aef-ad1e-943a2dde88a4" />


***Solutions:***

```
CREATE VIEW warehouse_country_cl AS
SELECT *,
    CASE
        WHEN UPPER(TRIM(warehouse_country)) IN ('CZ', 'CZECH', 'CZECHIA', 'CZECH REPUBLIC') THEN 'The Czech Republic'
        WHEN UPPER(TRIM(warehouse_country)) IN ('DE', 'GER', 'GERMANY', 'DEUTSCHLAND')      THEN 'Germany'
        WHEN UPPER(TRIM(warehouse_country)) IN ('PL', 'POL', 'POLAND', 'POLSKA')            THEN 'Poland'
    ELSE NULL
    END AS clear_country
FROM dedup_id;
```

- I created a view (warehouse_country_cl) with a CASE expression that maps every variant to a single standardized country name.
- Each value is first normalized with UPPER(TRIM(warehouse_country)), so the mapping does not depend on capitalization or leading/trailing spaces. IN (...) lists group all known variants of a country under one label.
- The original column is kept next to the new clear_country column, so the cleaning stays transparent and auditable, and the raw table is never modified.

### DEDUPLICATION - product_id  !!!! uwzględnij warehouse_country !!!

***Problem:***
- The column contains no NULL values (3,741 rows, all non-null).
- It contains 2,500 unique values, so it is not unique (1,241 duplicated rows).
- The column is stored as text, so MIN and MAX are compared alphabetically ('1' to '999')
  
```
SELECT 
COUNT(*) AS all_records,
COUNT(product_id) AS without_null,
COUNT(DISTINCT product_id) AS unique_ids,
MIN(product_id),
MAX (product_id)
FROM inventory;
```

<img width="419" height="69" alt="image" src="https://github.com/user-attachments/assets/d5348b17-a00c-47a6-8cf4-21d9286ae2ab" />

***Solutions:***

```
CREATE VIEW dedup_id AS
WITH dedup_id AS (
SELECT *,
   ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY product_id
   ) AS rn
FROM inventory
)
SELECT *
FROM dedup_id
WHERE rn = 1;
```

- Groups records by product_id using PARTITION BY product_id.
- Assigns a sequential number (rn) to each row within the same product_id group using the ROW_NUMBER() function.
- The first occurrence of each product_id receives rn = 1, while subsequent duplicates receive rn = 2, rn = 3, etc.
- The WHERE rn = 1 filter keeps only the first occurrence of each product.

### DATE FORMAT STANDARDIZATION - last_stock_update

***Problem:***
- The column is stored as text, so the database does not enforce a date format.
- The query returned 8 groups of values that do not match the DD.MM.YYYY pattern (20 rows in total).
- 6 rows contain NULL values, 2 rows contain 'not_a_date', 6 rows contain 2024-13-40, 6 rows contain incomplete dates without a day

```
SELECT last_stock_update, COUNT(*) AS cnt
FROM warehouse_country_cl
WHERE last_stock_update IS NULL
   OR last_stock_update !~ '^\d{2}\.\d{2}\.\d{4}$'
GROUP BY last_stock_update
ORDER BY last_stock_update;
```

<img width="235" height="245" alt="image" src="https://github.com/user-attachments/assets/16d05bd3-8747-4e53-981c-343708e3c140" />

***Solutions:***

```
CREATE VIEW last_stock_update_cl AS
SELECT *,
    CASE
        --- DD.MM.YYYY
        WHEN last_stock_update ~ '^(0[1-9]|[12]\d|3[01])\.(0[1-9]|1[0-2])\.\d{4}$'
            THEN TO_DATE(last_stock_update, 'DD.MM.YYYY')
        --- YYYY-MM-DD
        WHEN last_stock_update ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'
            THEN TO_DATE(last_stock_update, 'YYYY-MM-DD')
        --- YYYY/MM/DD
        WHEN last_stock_update ~ '^\d{4}/(0[1-9]|1[0-2])/(0[1-9]|[12]\d|3[01])$'
            THEN TO_DATE(last_stock_update, 'YYYY/MM/DD')
        --- YYYY-MM (1st day of the month assumed)
        WHEN last_stock_update ~ '^\d{4}-(0[1-9]|1[0-2])$'
            THEN TO_DATE(last_stock_update || '-01', 'YYYY-MM-DD')
        --- incorrect records (e.g. 2024-13-40, not_a_date) -> NULL
        ELSE NULL
    END AS last_stock_update_cl
FROM warehouse_country_cl;
```

- I validated and converted dates in DD.MM.YYYY and YYYY-MM-DD formats to the DATE type.
- I converted the 3 YYYY-MM records to the first day of the month.
- I set invalid values (not_a_date, 2024-13-40) to NULL.
- Missing values were kept as NULL.

***I skipped the columns that did not require cleaning.***

### Preparing tables to POWER BI
## Sales_order

```
SELECT 
order_id::INT AS order_id,
order_date_val AS order_date,
clear_country AS country,
quantity_cl::INT AS quantity,
unit_price_cl::DECIMAL AS unit_price,
discount_clean::DECIMAL AS discount,
status_cl AS status
FROM status_cl
WHERE rn = 1;
```
## Products

## Inventory


<img width="645" height="415" alt="image" src="https://github.com/user-attachments/assets/a2901a31-ae83-4bbe-a672-8cfae4b9841a" />


<h2 align="center">Results:</h2>
