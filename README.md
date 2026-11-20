 <h1 align="center">Retail_Fashion_Cleaning</h1>

### Data analysis:
This notebook covers the first phase of the Retail_Fashion project: data cleaning and preparation. The analysis itself continues in the second part, available at the link: https://github.com/aldonaorzol/Retail_Fashion_Analysis


### Tools: 
PostgreSQL, Microsoft Excel,  Copilot, Claude

### Description:

The source data contained numerous data quality issues, including inconsistent formats, duplicate records, missing values, and invalid entries. Using SQL, I performed data profiling, data cleansing, and data validation tasks as part of the ETL workflow to enhance data integrity and prepare the datasets for reliable business analysis and reporting. The process included standardizing country names, validating numerical fields, identifying potential outliers, converting data types, standardizing formats, and verifying consistency across sales, product, and inventory datasets. Additional data quality checks were conducted to ensure data accuracy, completeness, and compliance with business rules.

### Purpose:

The goal of Retail_Fashion_Cleaning is to prepare the data for analysis by recovering as many records as possible while keeping every transformation accurate and traceable.

### Data Structure:


<img width="700" height="205" alt="image" src="https://github.com/user-attachments/assets/31a1bb37-654e-41f6-bdbc-e1d95ef343ed" />


### Variable types:
__Identifiers:__ order_id, customer_id, product_id;

__Categorical:__ country, status, category, sub_category, warehouse_country;

__Numeric:__ quantity, unit_price, discount_pct, base_price, stock_quantity;

__Temporal:__ order_date, launch_date, last_stock_update;


<h2 align="center">Data Profiling</h2>

### Import verification

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
-discount_pct: __31,322 NULLs__ (about 12.0% of rows)
-order_date: __629 NULLs__ (about 0.24% of rows)

I also noticed formats dd.mm.yyyy, yyyy.mm, yyyy.mm.dd, non-existent dates like 2024-13-40, and not_a_date values in the order_date column. 

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
1. __discount_pct (12.0% missing)__ missing discount is ambiguous: it may mean “no discount was applied” or “the value was not recorded”. These two cases have different consequences for revenue calculations, so the interpretation must be verified (e.g. by comparing unit_price with base_price from products) before choosing a treatment, such as assuming 0% or flagging the rows. Replacing blanks with 0 without evidence could overstate revenue.
2. __order_date (629 missing)__ the share is small, but the date is essential for time-based analysis (trends, seasonality, year-over-year comparisons). Rows without a date cannot be assigned to a period and would silently drop out of time-based aggregations. They should be flagged and, if the date cannot be recovered, excluded from time-series analysis, while still being counted in totals where appropriate.
3. Complete key and analytical columns. order_id, product_id and customer_id have no gaps, so JOIN operations and duplicate detection are not compromised by missing keys. quantity and unit_price are also complete, so revenue can be calculated for every row.
4. NULLs are counted here after PostgreSQL converted blank fields during import. In addition to the 629 NULLs, the column contains the text placeholder not_a_date in 661 records (0.25% of all rows). Together, this gives __1,290 rows (about 0.49%)__ with no usable date. The column also contains dates in invalid formats or with non-existent values, such as 2024-13-40. Because order_date is essential for time-based analysis, these issues are addressed during standardization.

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

<img width="783" height="78" alt="image" src="https://github.com/user-attachments/assets/155236b4-59b0-46ed-8a3c-57089674ffdb" />

#### Insights: 
The per-column check of products (2,500 records) confirms that the record count is correct. It also shows that missing values occur in only one column:
- launch_date: __91 NULLs (about 3.6% of rows)__
All other columns (product_id, category, sub_category, base_price) contain no NULLs.

<img width="449" height="735" alt="image" src="https://github.com/user-attachments/assets/23dddca6-e5e8-4032-9257-20fe4403bb02" />

#### Explanations:

1.Complete key and analytical columns. product_id has no gaps, so JOIN operations with sales_orders and inventory are not compromised. category, sub_category and base_price are also complete, so sales can be analyzed by product segment and compared against selling prices without any imputation.
2. launch_date (91 missing). The missing values are documented, but __they do not affect this project: the column is not used in the analysis,__ so the affected rows are retained and no imputation or removal is required. Should the scope change in the future (e.g. analyzing product age or the effect of new launches on sales), these records would need to be reviewed, as they could not be included in such analysis.

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

<img width="771" height="74" alt="image" src="https://github.com/user-attachments/assets/82c45176-0db4-47fe-827b-116f5d79fa5b" />

#### Insights:
The per-column check of inventory (3,741 records) confirms that the record count is correct. It also shows that NULLs occur in only one column last_stock_update __6 NULLs (about 0.16% of rows)__

All other columns (product_id, warehouse_country, stock_quantity) contain no NULLs. Inspection of the source file additionally revealed that last_stock_update contains invalid entries that a NULL check does not detect: the text placeholder not_a_date, a non-existent date __(2024-13-40, month 13 and day 40),__ and partial dates without a day __(e.g. 2023-06, 2023-10, 2024-04, 2024-05, 2024-07, 2024-11).__ The same pattern was found in products.launch_date.

#### Explanations:
1. Complete key and analytical columns. product_id and warehouse_country have no gaps, so the composite key (product_id + warehouse_country), already confirmed as unique, is fully populated and JOIN operations with products are not compromised. stock_quantity is also complete, so stock levels are available for every record.
2. last_stock_update is not only missing, but also inconsistent. Besides 6 NULLs, the column contains the placeholder not_a_date, an invalid date and partial dates (year and month only). A NULL-only check therefore understates the scale of the problem: the number of unusable dates is higher than 6.
3. Impact on the project. last_stock_update will be used in the analysis (e.g. to assess how current the inventory data is), so these records cannot be ignored. Rows with not_a_date, NULL or a non-existent date have no usable update date and will be converted to NULL and flagged. Partial dates (year and month) need an explicit rule: either keep them at month-level precision or set them to the first day of the month, with the decision documented. Dates will be parsed into a proper DATE type and checked against the expected range.

### Duplicated records

<img width="601" height="389" alt="image" src="https://github.com/user-attachments/assets/b58700ad-133a-4db1-ad39-1b2d75f8759d" />

#### Insights:
I check compared the number of rows with the number of distinct key values in each table:
- sales_orders: 260,780 rows vs. 260,000 distinct order_id values, so __780 surplus rows (about 0.30% of the table).__
- products: 2,500 rows and 2,500 distinct product_id values, so the key is unique and no duplicates exist.
- inventory: __3,741 rows vs. 2,500__ distinct product_id values.

<img width="782" height="690" alt="image" src="https://github.com/user-attachments/assets/b8233935-e21b-4ec9-8981-84ce80f031a5" />

The results confirm that real duplicate records exist in the sales_orders table. They will be removed in the next step.

<img width="964" height="265" alt="image" src="https://github.com/user-attachments/assets/3f0436cf-f540-4945-9270-52392fa700e0" />


<img width="349" height="764" alt="image" src="https://github.com/user-attachments/assets/589aad2c-3051-4688-b8f6-faba5aeb337b" />

The results show that the products table does not require deduplication, as the same product may appear in more than one warehouse.

#### Explanations:
1. product_id is the primary key in the catalog. It must be unique, so any repeated value would be an error (e.g. a doubled import or a merge of several sources). Duplicates in products would multiply rows in every JOIN with sales_orders and inventory, inflating revenue, quantities and all KPIs built on them. This was not the case here.
2. A repeated order_id is not automatically an error. If the table is at order-line level, one order can legitimately contain several products. Before removing anything, it was necessary to check whether the repeated rows are full duplicates (identical in all columns) or different line items / conflicting versions of the same order.
3. The two cases require different treatment. Full duplicates can be safely removed (keeping one record). Rows sharing a key but differing in values need a rule for choosing the correct version, such as the most recent record, or manual review.
4. In inventory, a repeated product_id is expected. The correct uniqueness check is the composite key (product_id + warehouse_country), which passed.
5. Why it matters for the analysis. Duplicates inflate totals and counts, distort averages and shares, and break the one-to-many relationship between products and sales_orders.
  
### Do the data require standardization?

<img width="816" height="467" alt="image" src="https://github.com/user-attachments/assets/2add5a63-a247-4cc8-9a5e-74ae24544f09" />

#### Insights:
Unstandardized data does not cause errors; it silently distorts results. My queries and charts will run without any problem, and the numbers will look credible, but they will be wrong. When the data is aggregated, France will appear as three separate rows, France, france and FR, each holding only a fraction of the actual sales. The country ranking will therefore depend on how consistently someone typed the name, not on real sales.

<img width="183" height="522" alt="image" src="https://github.com/user-attachments/assets/111310b7-219d-4e53-b58f-cd3c2291e781" />
<img width="164" height="496" alt="image" src="https://github.com/user-attachments/assets/0093749a-84cb-4bef-b655-a7f2ffea40ae" />
<img width="180" height="321" alt="image" src="https://github.com/user-attachments/assets/2601630c-d4c4-4e25-a1d6-df601956e6c2" />


#### Explanations:
1. Why standardization is necessary. The country column stores the same country in several forms: different letter case __(France, france),__ full names versus codes __(France, FR),__ non-standard codes __(GER instead of DE)__ and colloquial names __(Holland instead of Netherlands).__ SQL treats each of these as a distinct value, so grouping, filtering and counting operate on the spelling rather than on the actual country.

2. Status values. The values __(Done, Complete and Completed)__ have the same meaning and should be mapped to a single value. The same applies to __(Ship and Shipped)__.

3. Why unambiguous values matter. Unambiguous values are important because I will be analyzing sales by country. The order status determines whether an order was actually fulfilled, generated revenue for the company and should be included in the sales results. Inconsistent status values would cause part of the real sales to be excluded or misclassified, which would distort the results for each country.


I decided to clean my data in PostgreSQL using Views. I made my decision based on:
- the raw data won't be changed; if any mistakes occur, correction will be possible.
- the data will always be updated; views are updated automatically if the database is updated 
- a view is a clear record of what I have done with the data. This makes the result easy to reproduce and explain.

### DEDUPLICATION - order_id

***Problem:*** 
Duplicated orders in order_id column. I veryfying this issue above. 

Before:

<img width="783" height="544" alt="image" src="https://github.com/user-attachments/assets/8ab356bc-d8dd-4de4-abba-e2f037f0e1c1" />

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

After:

<img width="857" height="171" alt="image" src="https://github.com/user-attachments/assets/9821975f-bd5b-4cf1-9a1c-068d6426ab55" />

***Why this query is appropriate:***

1. Cannot filter by `ROW_NUMBER()` in the same `SELECT` statement, because window functions are evaluated after the `WHERE` clause. A CTE allows to calculate the number first and then filter by it.
2. ROW_NUMBER() assigns each row a sequential number within its group. It takes no arguments, because the numbering does not depend on the values of the records.
3. OVER (...) indicates that this is a window function: it is calculated for each row separately, based on a “window” of related rows, rather than collapsing the rows into one.
4. PARTITION BY order_id splits the rows into groups that share the same order_id.
5. ORDER BY ... sets the order of rows within a group. The row that comes first in that order gets number 1.

### DATE FORMAT STANDARDIZATION - order_date 

***Problem:***
The order_date column mixed several text formats: YYYY-MM-DD, DD-MM-YYYY, DD.MM.YYYY and YYYY-MM. Some values were also impossible dates, such as 2024-13-40. 

<img width="394" height="170" alt="image" src="https://github.com/user-attachments/assets/b3bb7774-8851-4087-a5b0-585db9cf1445" />

Before:

<img width="789" height="686" alt="image" src="https://github.com/user-attachments/assets/4116ab4b-3ee0-4e4c-bc08-ef18df013866" />


***Solutions:***

```
CREATE OR REPLACE VIEW date_clean AS
WITH s AS (
    SELECT d.*, NULLIF(TRIM(d.order_date), '') AS od
    FROM dedup_orders d
),
t AS (
    SELECT s.*,
        CASE
            WHEN od IS NULL THEN 'source_null'
            WHEN od ~ '^(0[1-9]|[12]\d|3[01])\.(0[1-9]|1[0-2])\.\d{4}$' THEN 'dd.mm.yyyy'
            WHEN od ~ '^(0[1-9]|[12]\d|3[01])-(0[1-9]|1[0-2])-\d{4}$'   THEN 'dd-mm-yyyy'
            WHEN od ~ '^(0[1-9]|[12]\d|3[01])/(0[1-9]|1[0-2])/\d{4}$'   THEN 'dd/mm/yyyy'
            WHEN od ~ '^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$'   THEN 'yyyy-mm-dd'
            WHEN od ~ '^\d{4}/(0[1-9]|1[0-2])/(0[1-9]|[12]\d|3[01])$'   THEN 'yyyy/mm/dd'
            WHEN od ~ '^\d{4}\.(0[1-9]|1[0-2])\.(0[1-9]|[12]\d|3[01])$' THEN 'yyyy.mm.dd'
            WHEN od ~ '^\d{4}-(0[1-9]|1[0-2])$'                         THEN 'yyyy-mm'
            ELSE 'UNRECOGNIZED'
        END AS order_date_status
    FROM s
)
SELECT t.*,
    CASE order_date_status
        WHEN 'dd.mm.yyyy' THEN TO_DATE(od, 'DD.MM.YYYY')
        WHEN 'dd-mm-yyyy' THEN TO_DATE(od, 'DD-MM-YYYY')
        WHEN 'dd/mm/yyyy' THEN TO_DATE(od, 'DD/MM/YYYY')
        WHEN 'yyyy-mm-dd' THEN TO_DATE(od, 'YYYY-MM-DD')
        WHEN 'yyyy/mm/dd' THEN TO_DATE(od, 'YYYY/MM/DD')
        WHEN 'yyyy.mm.dd' THEN TO_DATE(od, 'YYYY.MM.DD')
        WHEN 'yyyy-mm'    THEN TO_DATE(od || '-01', 'YYYY-MM-DD')
    END AS order_date_val
FROM t;
```
After:

<img width="196" height="536" alt="image" src="https://github.com/user-attachments/assets/6533c9b9-dc94-4e5d-8eda-d485cc059a14" />

<img width="473" height="142" alt="image" src="https://github.com/user-attachments/assets/c04362c8-5e2e-46da-b3c8-3542b9ad4335" />

***What I have done:***

1. I standardized order_date in a view built on CTEs. Blank values are treated as NULL, and each text value is classified with a strict regular expression (literal dots escaped as \.) that checks the valid ranges for month (01–12) and day (01–31).
2. Only values that match a pattern are converted to the DATE type with TO_DATE, using the format specific to each pattern. For day-first formats I confirmed the order by checking that the day reaches 31 while the month never exceeds 12.
3. Values that match no pattern (e.g., not_a_date, 2024-13-40) become NULL with the status UNRECOGNIZED, and empty values get source_null. Together they account for 0.73% of all records. I kept them in the view, flagged, and excluded them only from time-based analysis.
4. Records in the yyyy-mm format were assigned the first day of the month and are flagged, so they should be used only for monthly or yearly analysis.
5. The order_date_status column documents how each date was interpreted, so the cleaning can be audited..

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
    ELSE 'UNMAPPED'
    END AS clear_country
FROM orders_clean;
```

 weryfikacja czy przebiegło prawidłowo sprawdz i zrób zrzut !!!!!!!
```
SELECT country, COUNT(*) AS n_rows
FROM (
    SELECT country, CASE ... END AS clear_country
    FROM orders_clean
) t
WHERE clear_country IS NULL OR clear_country = 'UNMAPPED'
GROUP BY country
ORDER BY n_rows DESC;
```

- I built a CASE expression that maps every variant to a single standardized country name.
- Each value is first normalized with UPPER(TRIM(country)), so the mapping does not depend on capitalization or leading/trailing spaces.



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

### Validate referential integrity - product_id exists in the products table

#### Explanations:
1. I'm looking for orphan records, rows in the child table (sales_orders, stock) whose product_id doesn't exist in the parent table (products). After an INNER JOIN, such rows disappear from the results, and after a LEFT JOIN, they receive empty product attributes (category, base price), so sales per category are underestimated without any warning.


Pomysły na usprawnienia na gromadzenie danych w przyszłości!!!!


OUTLIERS
image
Insights:
1038 records with 0 values are complaints. They are in the table to show the proper number of orders.
1570 recods with 608 values (0,60%) needs deeper verifying. I decided to assess statistical significance in revenue context
Odstęp od reszty danych: pozostałe wartości quantity mieszczą się w przedziale 0–13 (średnia 3,19, odchylenie 1,50) !!!!
Test częstości: przy rozkładzie Poissona dopasowanym do reszty prawdopodobieństwo wartości ≥ 608 jest praktycznie zerowe (p < 10⁻³⁰⁰, test dwumianowy p ≈ 0). Oczekiwana liczba takich rekordów to 0, a jest ich 1 570.

Wpływ na wyniki: te rekordy podnoszą średnią quantity z 3,19 do 6,83 i stanowią 53,6% sumy sprzedanych sztuk. Rekordy z 608 wyglądają jak losowa próbka danych, a nie wyróżniona grupa: Rozkład jest równomierny po produktach (1 152 różne), klientach (1 542 różnych) i latach. Zamówienie 608 sztuk przy typowej cenie ~180 byłoby naprawdę duże i trudno uwierzyć, że zdarza się z taką samą częstością w każdym kraju i roku. To wskazuje na artefakt, na przykład kod błędu, wartość zastępczą (sentinel) przy braku danych lub zepsuty import.

typy danych The data includes unrealistic values such as zero prices, outliers, impossible dates and unreadable discount values. These problematic records made up a small share of rows, except in discount_pct.
-należy uwzględnić jedynie najnoszą wersje weryfikacji stoku uwzględniając że te same produkty mogą tystępować w kilku magazynach.

Cleaning Plan:
Sales_orders

Products:

Convert data types: product_id to INT, base_price to decimal,
launch_date to date - remove column
Check price logic: no zero or negative base_price
