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

<h2 align="center">Data Quality</h2>
Assess the quality of the data by checking:

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

```
SELECT COUNT(order_id) AS all_orders, COUNT(DISTINCT order_id) AS distinct_orders
FROM sales_orders;
```
<img width="242" height="69" alt="image" src="https://github.com/user-attachments/assets/7a5cfaa4-5c58-4e85-b4f9-8d393c126465" />

As we can see, the data requires deduplication.
  
- Whether the values stored in each column are reasonable (e.g., prices, quantities).

  quantity has outliner 
- Whether the data is consistent across the dataset.


<h2 align="center">TABLE sales_orders</h2>

### DEDUPLICATION - order_id 

Problem:
The dataset contained duplicate records identified by the same order_id. Duplicate orders could lead to inaccurate reporting, including overstated sales volumes, revenue, and order counts. To ensure data quality and reliable analysis, duplicate records needed to be identified and removed.

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
Solutions:
* Groups records by order_id using PARTITION BY order_id.
* Assigns a sequential number (rn) to each row within the same order_id group using the ROW_NUMBER() function.
* The first occurrence of each order_id receives rn = 1, while subsequent duplicates receive rn = 2, rn = 3, etc.
* The WHERE rn = 1 filter keeps only the first occurrence of each order.
* The result is saved as a View named dedup_orders, providing a deduplicated dataset without modifying the original source table.

### Date Format Standardization - order_date

Problem:
The order_date column was stored as TEXT and mixed several formats (YYYY-MM-DD, YYYY/MM/DD, DD-MM-YYYY, DD.MM.YYYY, YYYY-MM). Some values were also impossible dates, such as 2024-13-40.

```
WITH parsed AS (
    SELECT
        CASE
            -- DD.MM.YYYY lub DD-MM-YYYY
            WHEN order_date ~ '^(0[1-9]|[12]\d|3[01])[.-](0[1-9]|1[0-2])[.-]\d{4}$'
                THEN TO_DATE(order_date, 'DD-MM-YYYY')
            -- YYYY-MM-DD
            WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'
                THEN TO_DATE(order_date, 'YYYY-MM-DD')
            -- YYYY-MM
            WHEN order_date ~ '^\d{4}-(0[1-9]|1[0-2])$'
                THEN TO_DATE(order_date || '-01', 'YYYY-MM-DD')
        END AS order_date_val
    FROM dedup_orders
)
SELECT
    COUNT(*)                                                      AS all_values,
    COUNT(*) - COUNT(order_date_val)                              AS null_values,
    ROUND(100.0 * (COUNT(*) - COUNT(order_date_val)) / COUNT(*), 2) AS percentage_of_all_nulls
FROM parsed;
```

<img width="386" height="76" alt="image" src="https://github.com/user-attachments/assets/99a8de21-86fd-4078-8183-fdcf1527c4ed" />

```
CREATE VIEW order_date_val AS
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
Solutions:
- To be able to refer back to the first part of the query, I used a CTE.
- I performed a simulation that allowed me to determine what percentage of all records consist of invalid values. As we can see above, this figure is 0.73%. Consequently, I decided to leave them as NULL values ​​and exclude them from the final analysis.
- The query standardizes the date format in the column and converts the data type from text to DATE, ensuring data consistency and accuracy for further analysis.
- Using the code '^(0[1-9]|[12]\d|3[01])\.(0[1-9]|1[0-2])\.\d{4}$', I excluded invalid dates such as 2024-13-40.
- I standardized the date format to YYYY-MM-DD using the `CASE` function.

### UNIQUE CUSTOMER INDENTIFIER - customer_id
No changes are required.

### DATA STANDARDIZATION - country

Problem:
The country column in the orders data contained the same country written in many different ways: codes (PL, DE), full names (Poland, Germany), local-language names (Polska, Deutschland), alternative names (Holland, Czechia) and inconsistent capitalization or stray spaces. This made grouping and reporting by country unreliable, since one country was split across several categories.

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
    END AS clear_country
FROM order_date_val;
```
Solutions:
- I built a CASE expression that maps every variant to a single standardized country name.
- Each value is first normalized with UPPER(TRIM(country)), so the mapping does not depend on capitalization or leading/trailing spaces.

### UNIQUE PRODUCT INDENTIFIER - product_id
No changes are required.

### OUTLIERS - quantity

Problem:
There is an outlier in the quantity column. To make an informed decision, I calculated what percentage of records contain erroneous data.

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
FROM clear_country;
```

<img width="352" height="71" alt="image" src="https://github.com/user-attachments/assets/63805611-6d82-4022-a226-527b2c870dce" />

```
SELECT quantity, COUNT(*)
FROM clear_country
GROUP BY quantity
ORDER BY 2 DESC;
```

<img width="178" height="417" alt="image" src="https://github.com/user-attachments/assets/7c844665-a011-4960-8366-397167e09bf0" />

```
CREATE VIEW quantity_cl AS
SELECT *,
    CASE
        WHEN quantity IN ('0', '608') THEN NULL
        ELSE quantity
    END AS quantity_cl
FROM clear_country;
```

Solutions:
- Calculated the percentage of records containing outlier values (quantity = 608 and quantity = 0) to assess their impact on the dataset.
- As these values accounted for only 1% of all records and were identified as data quality issues, they were replaced with NULL values.

### unit_price

Problem:
The unit_price column was stored as TEXT and had to be converted to NUMERIC before analysis. It also contained zero values, which are implausible for unit prices (the lowest non-zero price is about 30) and likely represent missing data. 

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
CREATE VIEW unit_price_cl AS
SELECT *,
    CASE
	WHEN unit_price = '0.0' THEN NULL
	ELSE unit_price
	END AS unit_price_cl
FROM quantity_cl;
```

Solutions:
- I did not find any specific dependencies (e.g., the zero values were not concentrated in a particular country, product, or date).
- Impossible values accounted for 0.62% of all records, so I decided to replace them with NULL rather than delete the rows.

### discount_pct
Problem: 
```
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE discount_pct !~ '^\d+(\.\d+)?$') AS corrupted,
    ROUND(100.0 * COUNT(*) FILTER (WHERE discount_pct !~ '^\d+(\.\d+)?$') / COUNT(*), 2) AS pct
FROM quantity_cl
WHERE discount_pct IS NOT NULL;
```
<img width="264" height="70" alt="image" src="https://github.com/user-attachments/assets/b9d35beb-dd8d-4f9d-8086-fff854f53216" />

```
CREATE VIEW discount_cl AS
SELECT *,
    CASE
        -- zwykłe liczby
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
FROM quantity_cl;
```

Solution:
- I opened the file in a notebook to verify whether the issue occurred during the data import process or if the data had been retrieved incorrectly. It turned out that the issue originated from the source data. In a real-world scenario, I would verify whether the data had been imported correctly from the source system. However, in this case, the dataset was artificially generated by AI, making such verification impossible.
- I decided to replace month names with numerical values. I am aware that this approach may introduce inaccuracies because a value such as "09.Mar" could be interpreted as either 09.30 or 09.03.

### status
Problem:
Solutions:



<h2 align="center">TABLE products</h2>

### product_id
Problem:
Solutions:
### category
Problem:
Solutions:
### sub_category
Problem:
Solutions:
### base_price
Problem:
Solutions:
### launch_date
Problem:
Solutions:

<h2 align="center">TABLE inventory</h2>

### product_id
Problem:
Solutions:
### warehouse_country
Problem:
Solutions:
### stock_quantity
Problem:
Solutions:
### last_stock_update
Problem:
Solutions:
