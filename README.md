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

### DEDUPLICATION - order_id and customer_id

***Problem:***
The dataset contained duplicate records identified by the same order_id. Duplicate orders could lead to inaccurate reporting, including overstated sales volumes, revenue, and order counts. To ensure data quality and reliable analysis, duplicate records needed to be identified and removed.

```
CREATE OR REPLACE VIEW dedup AS
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY order_id, customer_id
               ORDER BY order_id, customer_id
           ) AS rnn
    FROM status_cl
)
SELECT *
FROM ranked
WHERE rnn = 1;
```
***Solutions:***
* Groups records by order_id using PARTITION BY order_id.
* Assigns a sequential number (rn) to each row within the same order_id group using the ROW_NUMBER() function.
* The first occurrence of each order_id receives rn = 1, while subsequent duplicates receive rn = 2, rn = 3, etc.
* The WHERE rn = 1 filter keeps only the first occurrence of each order.
* The result is saved as a View named dedup_orders, providing a deduplicated dataset without modifying the original source table.

### DATE FORMAT STANDARIZATION - order_date

***Problem:***
The order_date column mixed several formats (YYYY-MM-DD, YYYY/MM/DD, DD-MM-YYYY, DD.MM.YYYY, YYYY-MM). Some values were also impossible dates, such as 2024-13-40.

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
***Solutions:***
- To be able to refer back to the first part of the query, I used a CTE.
- I simulated to determine what percentage of all records contain invalid values. As shown above, this figure is 0.73%. Consequently, I left them as NULL values and excluded them from the final analysis.
- Using the code '^(0[1-9]|[12]\d|3[01]).(0[1-9]|1[0-2]).\d{4}$', I excluded invalid dates such as 2024-13-40.
- I standardized the date format to YYYY-MM-DD using the CASE function.

### UNIQUE CUSTOMER IDENTIFIER - customer_id
***Problem:***

```
SELECT
    COUNT(*) AS all_rows,
    COUNT(customer_id) AS non_null,
    COUNT(DISTINCT customer_id) AS unique_ids,
    MIN(customer_id::INT) AS min_id,
    MAX(customer_id::INT) AS max_id
FROM sales_orders;
```
<img width="421" height="71" alt="image" src="https://github.com/user-attachments/assets/9b5b0d8a-5c46-406a-bf72-016a410de22a" />

***Solutions:***
- The customer_id column contains no NULL values (260,780 rows, all non-null).
- It contains 59,200 unique values in the range 1–59,999, so it is not unique.
- There are gaps in the ID range (about 799 missing numbers).

### DATA STANDARDIZATION - country

***Problem:***
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
***Solutions:***
- I built a CASE expression that maps every variant to a single standardized country name.
- Each value is first normalized with UPPER(TRIM(country)), so the mapping does not depend on capitalization or leading/trailing spaces.

### UNIQUE PRODUCT IDENTIFIER - product_id
***Problem:***


***Solutions:***
No changes are required.

### OUTLIERS - quantity

***Problem:***
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

***Solutions:***
- Calculated the percentage of records containing outlier values (quantity = 608 and quantity = 0) to assess their impact on the dataset.
- As these values accounted for only 1% of all records and were identified as data quality issues, they were replaced with NULL values.

### REPLACING INVALID VALUES WITH NULL - unit_price

***Problem:***
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

***Solutions:***
- I did not find any specific dependencies (e.g., the zero values were not concentrated in a particular country, product, or date).
- Impossible values accounted for 0.62% of all records, so I decided to replace them with NULL rather than delete the rows.

### DATA RECOVERY - discount_pct

***Problem:***
The discount_pct column was corrupted in 49.2% of records. The column contains dates, values ​​of 0.0, and NULLs. It is 	.....
Check that valid values fall between 0 and 100

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

***Solution:***
- I opened the file in a notebook to verify whether the issue occurred during the data import process or if the data had been retrieved incorrectly. It turned out that the issue originated from the source data. In a real-world scenario, I would verify whether the data had been imported correctly from the source system. However, in this case, the dataset was artificially generated by AI, making such verification impossible.
- I decided to replace month names with numerical values. I am aware that this approach may introduce inaccuracies because a value such as "09.Mar" could be interpreted as either 09.30 or 09.03.

### DATA STANDARDIZATION - status

***Problem:***
The status column contained the same order status written in many different ways: SHIP, shipped, SHIPPED, DONE, done, Done, complete, Completed, COMPLETED, CANCELLED. The inconsistent spelling and capitalization split each status into several categories, which made counting and grouping orders by status unreliable.

```
CREATE VIEW status_cl AS
SELECT *,
    CASE
	WHEN UPPER(TRIM(status)) IN ('SHIP', 'SHIPPED')                THEN 'Shipped'
	WHEN UPPER(TRIM(status)) IN ('DONE', 'COMPLETE', 'COMPLETED')  THEN 'Completed'
	WHEN UPPER(TRIM(STATUS)) IN ('CANCELLED')                      THEN 'Cancelled'
    END AS status_cl
FROM discount_cl;
```

***Solutions:***
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
  
### DATA PROFILING - category
***Problem:***
Spelling and capitalization are consistent (each value begins with an uppercase letter); I see no typos, variations such as "women" / "WOMEN" / "Woman," or empty fields.

```
SELECT category, COUNT(*) AS cnt
FROM products
GROUP BY category
ORDER BY cnt DESC;
```

<img width="189" height="170" alt="image" src="https://github.com/user-attachments/assets/d269c20d-9bdc-442e-b679-656daf64429f" />

***Solutions:***
- I profiled the column with a GROUP BY query that lists every distinct value together with its number of records.
- The result showed exactly five categories (Women, Men, Kids, Shoes, Accessories), with consistent spelling and capitalization and no duplicate variants.
- The column required no cleaning.

### DATA PROFILING - sub_category

***Problem:***
This column appears consistent in terms of spelling, but it has a structural issue that is worth describing.
Overlapping categories. This is the main problem:
- Shoes is a parent category for Sneakers, Boots, Heels, Sandals, and Loafers.
- Tops overlaps with Shirts and T-Shirts.
- Bottoms overlaps with Jeans.

DO ROZWAŻENIA
```
CASE
    WHEN subcategory IN ('Sneakers','Boots','Heels','Sandals','Loafers','Shoes') THEN 'Footwear'
    WHEN subcategory IN ('Tops','Shirts','T-Shirts') THEN 'Tops'
    WHEN subcategory IN ('Jeans','Bottoms') THEN 'Bottoms'
    WHEN subcategory IN ('Bags','Belts','Hats','Scarves','Jewelry') THEN 'Accessories'
    WHEN subcategory IN ('Lingerie','Sleepwear') THEN 'Underwear & Sleepwear'
    ELSE subcategory
END AS category_group
```

***Solutions:***
- I see about 25 values, all formatted consistently, capitalized first letter, no variations like "jeans" vs. "JEANS," and no typos.

### DATA PROFILING - base_price
***Problem:***
These prices appear reliable and do not require cleaning. The distribution ranges roughly from 31 to 410, with most values ​​falling between 60 and 300; there are no negative values, text, or unusual formats. I do not see any zeros or extreme outliers here.

```
SELECT
    COUNT(*)                                              AS all_rows,
    COUNT(base_price)                                          AS non_null,
    MIN(base_price::NUMERIC)                                   AS min_price,
    MAX(base_price::NUMERIC)                                   AS max_price
FROM products;
```

<img width="366" height="67" alt="image" src="https://github.com/user-attachments/assets/9239d0e8-2fa5-4525-b4b0-537076b6113d" />

***Solutions:***
- Before using base_price in revenue and margin calculations, I needed to confirm that the column was complete and contained realistic numeric values.
- I ran a profiling query that checks the total row count, the number of non-NULL values, and the minimum and maximum price after casting to NUMERIC. The cast succeeded for every row, which confirms that all values are numeric. The prices fell between about 31 and 410, with no NULLs, and no zero or negative values.

### launch_date

***Problem:***
These dates appear consistent: they all follow the DD.MM.YYYY format, with leading zeros and a dot as a separator. I see no mixed formats, YYYY-MM patterns, or invalid values ​​such as 2024-13-40. The range spans the years 2015–2024, which is realistic (there are no dates from the future or from 1900).

```
SELECT
    CASE
        WHEN launch_date ~ '^\d{2}\.\d{2}\.\d{4}$' THEN 'DD.MM.YYYY'
        WHEN launch_date ~ '^\d{2}-\d{2}-\d{4}$'   THEN 'DD-MM-YYYY'
        WHEN launch_date ~ '^\d{4}-\d{2}-\d{2}$'   THEN 'YYYY-MM-DD'
        WHEN launch_date ~ '^\d{4}/\d{2}/\d{2}$'   THEN 'YYYY/MM/DD'
        WHEN launch_date ~ '^\d{4}-\d{2}$'         THEN 'YYYY-MM'
        WHEN launch_date IS NULL                   THEN 'NULL'
        ELSE 'other'
    END AS format,
    COUNT(*) AS cnt,
    MIN(launch_date) AS example
FROM products
GROUP BY 1
ORDER BY cnt DESC;
```

<img width="269" height="167" alt="image" src="https://github.com/user-attachments/assets/9bef0570-40f1-454a-b4e1-6e0a07899d3e" />


***Solutions:***
- Zdecydowana większość (95,84%) jest poprawna w formacie DD.MM.YYYY. Kolumna jest prawie spójna, więc nie trzeba jej przebudowywać, tylko obsłużyć wyjątki.
- 91 NULL-i (3,64%) to braki. Zostaw je jako NULL, ale odnotuj skalę. To najliczniejszy problem.
- 6 rekordów other: przykład not_a_date to jawnie błędny tekst. Zamień na NULL.
- 4 rekordy YYYY-MM-DD: przykład 2024-13-... wygląda na niemożliwą datę (miesiąc 13). Regex formatu je złapał, ale TO_DATE je odrzuci, więc część z nich (lub wszystkie) będzie NULL. Sprawdź je ręcznie.
- 3 rekordy YYYY-MM (np. 2017-05): da się sparsować na 1. dzień miesiąca, ale mają niższą precyzję, więc oznacz je flagą.

<h2 align="center">TABLE inventory</h2>

### DEDUPLICATION - product_id
***Problem:***
Solutions:

### DATA STANDARDIZATION - warehouse_country
Problem:
The country column in the inventory table described the same warehouse location in many different ways: ISO codes (PL, DE, CZ), full English names (Poland, Germany), local-language names (Polska, Deutschland), alternative names (Czechia) and inconsistent capitalization or stray spaces. One country was therefore split across several categories, which made grouping and reporting stock by country unreliable.

```
CREATE VIEW warehouse_country_cl AS
SELECT *,
    CASE
        WHEN UPPER(TRIM(country)) IN ('CZ', 'CZECH', 'CZECHIA', 'CZECH REPUBLIC') THEN 'The Czech Republic'
        WHEN UPPER(TRIM(country)) IN ('DE', 'GER', 'GERMANY', 'DEUTSCHLAND')      THEN 'Germany'
        WHEN UPPER(TRIM(country)) IN ('PL', 'POL', 'POLAND', 'POLSKA')            THEN 'Poland'
    END AS clear_country
FROM inventory;
```
***Solutions:***
- I created a view (warehouse_country_cl) with a CASE expression that maps every variant to a single standardized country name.
- Each value is first normalized with UPPER(TRIM(country)), so the mapping does not depend on capitalization or leading/trailing spaces. IN (...) lists group all known variants of a country under one label.
- The original column is kept next to the new clear_country column, so the cleaning stays transparent and auditable, and the raw table is never modified.

### stock_quantity

***Problem:***
The values ​​are whole numbers, mostly ranging from 0 to approximately 570, with no text, fractions, or negative values. Structurally, it looks clean.
Before using stock_quantity in inventory analysis, I needed to confirm that the column was complete, numeric and within a realistic range. The column was stored as TEXT, and it contained many zero values, so I had to decide whether they represent missing data or genuine stock-outs.

```
SELECT
    COUNT(*)                                             AS all_rows,
    COUNT(stock_quantity)                                AS non_null,
    COUNT(*) FILTER (WHERE stock_quantity = '0')         AS zeros,
    ROUND(100.0 * COUNT(*) FILTER (WHERE stock_quantity = '0') / COUNT(*), 2) AS zeros_pct,
    MIN(stock_quantity::INT), MAX(stock_quantity::INT)
FROM inventory;
```

<img width="474" height="73" alt="image" src="https://github.com/user-attachments/assets/a14eaf8b-f00d-4a91-9daf-f96d76922592" />


***Solutions:***
I ran a profiling query that checks the total row count, the number of non-NULL values, the number and percentage of zero values, and the minimum and maximum quantity after casting to INT. The cast succeeded for every row, which confirms that all values are whole numbers. Zeros were treated as valid “out of stock” levels rather than errors, because they sit naturally at the start of the distribution and, unlike placeholder codes such as 608 in quantity, they do not stand apart from the rest of the data.

### last_stock_update

***Problem:***
This sample appears consistent: all values ​​follow the DD.MM.YYYY format, with leading zeros and a dot as a separator. The range—roughly May 2023 to December 2024—is realistic (there are no dates from the distant future or from the year 1900). The first number frequently exceeds 12 (e.g., 25.01.2024, 31.08.2024, 28.12.2023), which confirms the format is DD.MM rather than MM.DD.

```

```

***Solutions:***


<h2 align="center">Results:</h2>
