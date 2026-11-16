 <h1 align="center">Retail_Fashion_Cleaning</h1>

### Data analysis:
This notebook covers the first phase of the Retail_Fashion project: data cleaning and preparation. The analysis itself continues in the second part, available at the link: https://github.com/aldonaorzol/Retail_Fashion_Analysis


### Tools: 
PostgreSQL, Microsoft Excel,  Copilot, Claude

### Description:

The source data contained numerous data quality issues, including inconsistent formats, duplicate records, missing values, and invalid entries. Using SQL, I performed data profiling, data cleansing, and data validation tasks as part of the ETL workflow to enhance data integrity and prepare the datasets for reliable business analysis and reporting. The process included standardizing country names, validating numerical fields, identifying potential outliers, converting data types, standardizing formats, and verifying consistency across sales, product, and inventory datasets. Additional data quality checks were conducted to ensure data accuracy, completeness, and compliance with business rules.

### Data Structure:


<img width="700" height="205" alt="image" src="https://github.com/user-attachments/assets/31a1bb37-654e-41f6-bdbc-e1d95ef343ed" />



### Variable types:
__Identifiers:__ order_id, customer_id, product_id;

__Categorical:__ country, status, category, sub_category, warehouse_country;

__Numeric:__ quantity, unit_price, discount_pct, base_price, stock_quantity;

__Temporal:__ order_date, launch_date, last_stock_update;


<h2 align="center">Data Quality</h2>

### Did I import all the records correctly?

### Sales_orders:

```
SELECT 'sales_orders' AS table_name,
       COUNT(*)                             AS all_records,
       COUNT(*) - COUNT(order_id)           AS order_id_nulls,
       COUNT(*) - COUNT(customer_id)        AS customer_id_nulls,
       COUNT(*) - COUNT(order_date)         AS order_date_nulls,
       COUNT(*) - COUNT(country)            AS country_nulls,
       COUNT(*) - COUNT(product_id)         AS product_id_nulls,
       COUNT(*) - COUNT(quantity)           AS quantity_nulls,
       COUNT(*) - COUNT(unit_price)         AS unit_price_nulls,
       COUNT(*) - COUNT(discount_pct)       AS discount_pct_nulls,
       COUNT(*) - COUNT(status)             AS status_nulls
FROM sales_orders;
```

<img width="1204" height="73" alt="image" src="https://github.com/user-attachments/assets/9d0d19d9-4195-4063-a40f-8552aa18003e" />

#### Insights:
The per-column check of sales_orders (260,780 records) confirms that the record count is correct. It also shows that missing values are concentrated in two columns:
-discount_pct: 31,322 NULLs (about 12.0% of rows)
-order_date: 629 NULLs (about 0.24% of rows)

I also noticed not_a_date values in the order_date column. 

<img width="319" height="68" alt="image" src="https://github.com/user-attachments/assets/756b6f26-d81d-4fa0-affb-77a8b5820ecc" />

```
SELECT
    COUNT(*)                                                             AS all_records,
    COUNT(*) FILTER (WHERE LOWER(TRIM(order_date::text)) = 'not_a_date') AS not_a_date_cnt,
    ROUND(100.0 * COUNT(*) FILTER (WHERE LOWER(TRIM(order_date::text)) = 'not_a_date')
          / COUNT(*), 2)                                                 AS not_a_date_pct
FROM sales_orders;
```

<img width="611" height="654" alt="image" src="https://github.com/user-attachments/assets/d2ce39c6-c81a-4903-ba12-fa0ee6e20265" />

#### Explanations:
1. discount_pct (12.0% missing). A missing discount is ambiguous: it may mean “no discount was applied” or “the value was not recorded”. These two cases have different consequences for revenue calculations, so the interpretation must be verified (e.g. by comparing unit_price with base_price from products) before choosing a treatment, such as assuming 0% or flagging the rows. Replacing blanks with 0 without evidence could overstate revenue.
2. order_date (629 missing). The share is small, but the date is essential for time-based analysis (trends, seasonality, year-over-year comparisons). Rows without a date cannot be assigned to a period and would silently drop out of time-based aggregations. They should be flagged and, if the date cannot be recovered, excluded from time-series analysis, while still being counted in totals where appropriate.
3. Complete key and analytical columns. order_id, product_id and customer_id have no gaps, so JOIN operations and duplicate detection are not compromised by missing keys. quantity and unit_price are also complete, so revenue can be calculated for every row.
4. Scope of the check. NULLs are counted here after PostgreSQL converted blank fields during import. In addition to the 629 NULLs, the column contains the text placeholder not_a_date in 661 records (0.25% of all rows). Together, this gives 1,290 rows (about 0.49%) with no usable date. The column also contains dates in invalid formats or with non-existent values, such as 2024-13-40. Because order_date is essential for time-based analysis, these issues are addressed during standardization.
  

## Products:
```
SELECT 'products' AS table_name,
       COUNT(*)                          AS all_records,
       COUNT(*) - COUNT(product_id)      AS product_id_nulls,
       COUNT(*) - COUNT(category)        AS category_nulls,
       COUNT(*) - COUNT(sub_category)    AS sub_category_nulls,
       COUNT(*) - COUNT(base_price)      AS base_price_nulls,
       COUNT(*) - COUNT(launch_date)     AS launch_date_nulls
FROM products;
```

<img width="789" height="76" alt="image" src="https://github.com/user-attachments/assets/b4c7c36a-6857-498d-9598-5e31594e55c7" />

#### Insights: 
The per-column check of products (2,500 records) confirms that the record count is correct. It also shows that missing values occur in only one column:
- launch_date: 91 NULLs (about 3.6% of rows)
All other columns (product_id, category, sub_category, base_price) contain no NULLs.

<img width="483" height="707" alt="image" src="https://github.com/user-attachments/assets/45b9f255-f00c-42db-a1db-31d7075402fc" />

#### Explanations:

1.Complete key and analytical columns. product_id has no gaps, so JOIN operations with sales_orders and inventory are not compromised. category, sub_category and base_price are also complete, so sales can be analyzed by product segment and compared against selling prices without any imputation.
2. launch_date (91 missing). The missing values are documented, but they do not affect this project: the column is not used in the analysis, so the affected rows are retained and no imputation or removal is required. Should the scope change in the future (e.g. analyzing product age or the effect of new launches on sales), these records would need to be reviewed, as they could not be included in such analysis.

## Inventory:

```
SELECT 'inventory' AS table_name,
       COUNT(*)                              AS all_records,
       COUNT(*) - COUNT(product_id)          AS product_id_nulls,
       COUNT(*) - COUNT(warehouse_country)   AS warehouse_country_nulls,
       COUNT(*) - COUNT(stock_quantity)      AS stock_quantity_nulls,
       COUNT(*) - COUNT(last_stock_update)   AS last_stock_update_nulls
FROM inventory;
```
<img width="766" height="70" alt="image" src="https://github.com/user-attachments/assets/990ac658-296d-494b-b000-11de8ed1355a" />

#### Insights:
The per-column check of inventory (3,741 records) confirms that the record count is correct. It also shows that NULLs occur in only one column last_stock_update 6 NULLs (about 0.16% of rows)

All other columns (product_id, warehouse_country, stock_quantity) contain no NULLs. Inspection of the source file additionally revealed that last_stock_update contains invalid entries that a NULL check does not detect: the text placeholder not_a_date, a non-existent date (2024-13-40, month 13 and day 40), and partial dates without a day (e.g. 2023-06, 2023-10, 2024-04, 2024-05, 2024-07, 2024-11). The same pattern was found in products.launch_date.

#### Explanations:
1. Complete key and analytical columns. product_id and warehouse_country have no gaps, so the composite key (product_id + warehouse_country), already confirmed as unique, is fully populated and JOIN operations with products are not compromised. stock_quantity is also complete, so stock levels are available for every record.
2. last_stock_update is not only missing, but also inconsistent. Besides 6 NULLs, the column contains the placeholder not_a_date, an invalid date and partial dates (year and month only). A NULL-only check therefore understates the scale of the problem: the number of unusable dates is higher than 6.
3. Impact on the project. last_stock_update will be used in the analysis (e.g. to assess how current the inventory data is), so these records cannot be ignored. Rows with not_a_date, NULL or a non-existent date have no usable update date and will be converted to NULL and flagged. Partial dates (year and month) need an explicit rule: either keep them at month-level precision or set them to the first day of the month, with the decision documented. Dates will be parsed into a proper DATE type and checked against the expected range.

### Whether duplicate records exist.

<img width="601" height="389" alt="image" src="https://github.com/user-attachments/assets/b58700ad-133a-4db1-ad39-1b2d75f8759d" />

#### Insights:
I check compared the number of rows with the number of distinct key values in each table:
- sales_orders: 260,780 rows vs. 260,000 distinct order_id values, so 780 surplus rows (about 0.30% of the table).
- products: 2,500 rows and 2,500 distinct product_id values, so the key is unique and no duplicates exist.
- inventory: 3,741 rows vs. 2,500 distinct product_id values.

<img width="782" height="690" alt="image" src="https://github.com/user-attachments/assets/b8233935-e21b-4ec9-8981-84ce80f031a5" />

The results confirm that real duplicate records exist in the sales_orders table. They will be removed in the next step.

```
SELECT COUNT(*) AS all_rows,
       COUNT(DISTINCT (product_id, warehouse_country)) AS distinct_keys,
       COUNT(DISTINCT (product_id, warehouse_country, stock_quantity, last_stock_update)) AS distinct_full_rows
FROM inventory;
```

<img width="964" height="265" alt="image" src="https://github.com/user-attachments/assets/3f0436cf-f540-4945-9270-52392fa700e0" />


<img width="349" height="764" alt="image" src="https://github.com/user-attachments/assets/589aad2c-3051-4688-b8f6-faba5aeb337b" />

The results confirm that products does not require a deduplication process, as its key is unique.

#### Explanations:
1. product_id is the primary key in the catalog. It must be unique, so any repeated value would be an error (e.g. a doubled import or a merge of several sources). Duplicates in products would multiply rows in every JOIN with sales_orders and inventory, inflating revenue, quantities and all KPIs built on them. This was not the case here.
2. A repeated order_id is not automatically an error. If the table is at order-line level, one order can legitimately contain several products. Before removing anything, it was necessary to check whether the repeated rows are full duplicates (identical in all columns) or different line items / conflicting versions of the same order.
3. The two cases require different treatment. Full duplicates can be safely removed (keeping one record). Rows sharing a key but differing in values need a rule for choosing the correct version, such as the most recent record, or manual review.
4. In inventory, a repeated product_id is expected. The correct uniqueness check is the composite key (product_id + warehouse_country), which passed.
5. Why it matters for the analysis. Duplicates inflate totals and counts, distort averages and shares, and break the one-to-many relationship between products and sales_orders.
  
### Do the data require standardization?



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
  
### DATE FORMAT STANDARDIZATION - launch_date kolumna nie potrzebna do analizy do usunięcia !!!!!

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
FROM inventory;
```

- I created a view (warehouse_country_cl) with a CASE expression that maps every variant to a single standardized country name.
- Each value is first normalized with UPPER(TRIM(warehouse_country)), so the mapping does not depend on capitalization or leading/trailing spaces. IN (...) lists group all known variants of a country under one label.
- The original column is kept next to the new clear_country column, so the cleaning stays transparent and auditable, and the raw table is never modified.

### DEDUPLICATION - product_id błędnie wykonana nie uwzględnia najnowszych danych, dane są zdeduplikowane losowo !!!!

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
   ROW_NUMBER() OVER (PARTITION BY product_id, clear_country ORDER BY product_id
   ) AS rn
FROM warehouse_country_cl
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
FROM inventory
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
FROM dedup_id;
```

- I validated and converted dates in DD.MM.YYYY and YYYY-MM-DD formats to the DATE type.
- I converted the 3 YYYY-MM records to the first day of the month.
- I set invalid values (not_a_date, 2024-13-40) to NULL.
- Missing values were kept as NULL.

***I skipped the columns that did not require cleaning.***

<h2 align="center">Preparing tables to POWER BI</h2> 
Dane są wyczyszczone i gotowe do analizy poniżej próbka jak wyglądają po czyszczeniu.

## Sales_order

```
SELECT 
order_id::INT AS order_id,
order_date_val AS order_date,
customer_id::INT AS customer_id,
clear_country AS country,
product_id::INT AS product_id,
quantity_cl::INT AS quantity,
unit_price_cl::DECIMAL AS unit_price,
discount_clean::DECIMAL AS discount,
status_cl AS status
FROM status_cl
WHERE rn = 1;
```

<img width="856" height="415" alt="image" src="https://github.com/user-attachments/assets/bc5b1eca-fd8d-4176-8e47-8a53f396a06f" />



## Products

```
SELECT 
product_id::INT AS product_id,
category,
sub_category,
base_price::DECIMAL AS base_price,
date_cl AS launch_date
FROM date_cl
```

<img width="500" height="418" alt="image" src="https://github.com/user-attachments/assets/aa854f03-cdd5-4289-b02c-c1c120d80747" />


## Inventory

```
SELECT
product_id::INT AS product_id,
clear_country AS warehouse_country,
stock_quantity::INT AS stock_quantity,
last_stock_update_cl AS last_stock_update
FROM last_stock_update_cl
WHERE rn = 1;
```

<img width="508" height="416" alt="image" src="https://github.com/user-attachments/assets/fb6b9115-5bd5-44c0-a3e4-86f5e75a286d" />

Pomysły na usprawnienia na gromadzenie danych w przyszłości!!!!
