 <h1 align="center">Retail_Fashion_Cleaning</h1>

### Data analysis:
This notebook covers the first phase of the Retail_Fashion project: data cleaning and preparation. The analysis itself continues in the second part, available at the link: https://github.com/aldonaorzol/Retail_Fashion_Analysis


### Tools: 
PostgreSQL, Microsoft Excel,  Copilot, Claude

### Description:

The source data contained numerous data quality issues, including __inconsistent formats__, __duplicate records__, __missing values__, and __invalid entries__. Using SQL, I performed data profiling, data cleansing, and data validation tasks as part of the ETL workflow to enhance data integrity and prepare the datasets for reliable business analysis and reporting. The process included standardizing country names, validating numerical fields, identifying potential outliers, converting data types, standardizing formats, and verifying consistency across sales, product, and inventory datasets. Additional data quality checks were conducted to ensure data accuracy, completeness, and compliance with business rules.

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

### Import verification:

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
-discount_pct: 31,322 NULLs (about __12.0%__ of rows)
-order_date: 629 NULLs (about __0.24%__ of rows)

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
1. __discount_pct (12.0% missing)__ missing discount is ambiguous. It may mean no discount was applied or the value was not recorded. These two cases have different consequences for revenue calculations, so the interpretation must be verified before choosing a treatment, such as assuming 0% or flagging the rows. Replacing blanks with 0 without evidence could overstate revenue.
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

1.Complete key and analytical columns. product_id has no gaps, so JOIN operations with sales_orders and inventory are not compromised.

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

Check whether the `product_id` field contains duplicate values. If it does not, there is no need to filter the data based on the date of the last inventory update, and the `last_stock_update` column can be omitted.

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

The results confirm that 780 order_id values appear twice. A comparison across all columns showed that every repeated order_id consists of identical rows, so these are true duplicates, not conflicting versions of the same order. They are removed in the next step, keeping one row per order_id.

<img width="964" height="265" alt="image" src="https://github.com/user-attachments/assets/3f0436cf-f540-4945-9270-52392fa700e0" />


<img width="349" height="764" alt="image" src="https://github.com/user-attachments/assets/589aad2c-3051-4688-b8f6-faba5aeb337b" />

The results show that the inventory table does not require deduplication, as the same product may appear in more than one warehouse.

#### Explanations:
1. product_id is the primary key in the catalog. It must be unique, so any repeated value would be an error (e.g. a doubled import or a merge of several sources). Duplicates in products would multiply rows in every JOIN with sales_orders and inventory, inflating revenue, quantities and all KPIs built on them. This was not the case here.
2. A repeated order_id is not automatically an error. If the table is at order-line level, one order can legitimately contain several products. Before removing anything, it was necessary to check whether the repeated rows are full duplicates (identical in all columns) or different line items / conflicting versions of the same order.
3. The two cases require different treatment. Full duplicates can be safely removed (keeping one record). Rows sharing a key but differing in values need a rule for choosing the correct version, such as the most recent record, or manual review.
4. In inventory, a repeated product_id is expected. The correct uniqueness check is the composite key (product_id + warehouse_country), which passed.
5. Why it matters for the analysis. Duplicates inflate totals and counts, distort averages and shares, and break the one-to-many relationship between products and sales_orders.
6. The product_id field contains no duplicate values; therefore, there is no need to filter the data following the most recent update of stock levels. So I decided to omit the column last_stock_update in my analysis.
  
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

I decided to use:
- CTE, because it lets me split the cleaning into clear, named steps (trimming blank values, classifying the format, then converting to DATE) instead of nesting subqueries. The status is calculated once and reused for the conversion, so the logic is easy to read, test, and change in one place.
- CASE function, because the dates arrive in several different text formats, and each one needs its own rule. CASE checks the formats in a defined order and applies the matching TO_DATE pattern only to values that pass validation, so invalid values never reach the conversion and cannot break the view.
- Give a flag, because replacing a value with NULL without explanation would hide what happened to the record. The order_date_status flag shows which format each date had and which values were unrecognized or missing, so the cleaning stays transparent, can be audited, and lets me decide separately how to treat each group in the analysis.

After:

<img width="196" height="536" alt="image" src="https://github.com/user-attachments/assets/6533c9b9-dc94-4e5d-8eda-d485cc059a14" />

***What I have done:***

1. I standardized order_date in a view built on CTEs. Blank values are treated as NULL, and each text value is classified with a strict regular expression (literal dots escaped as \.) that checks the valid ranges for month (01–12) and day (01–31).
2. Only values that match a pattern are converted to the DATE type with TO_DATE, using the format specific to each pattern. For day-first formats I confirmed the order by checking that the day reaches 31 while the month never exceeds 12.
3. Values that match no pattern (e.g., not_a_date, 2024-13-40) become NULL with the status UNRECOGNIZED, and empty values get source_null. Together they account for 0.73% of all records. I kept them in the view, flagged, and excluded them only from time-based analysis.
4. Records in the yyyy-mm format were assigned the first day of the month and are flagged, so they should be used only for monthly or yearly analysis.
5. The order_date_status column documents how each date was interpreted so the cleaning can be audited.

<img width="1155" height="695" alt="image" src="https://github.com/user-attachments/assets/56620d74-971b-4004-a75e-b50fd6a7aa46" />

### DATA STANDARDIZATION - country

<img width="619" height="583" alt="image" src="https://github.com/user-attachments/assets/cdbffa8e-cf31-468e-b8a2-8e8a7621857f" />

<img width="162" height="532" alt="image" src="https://github.com/user-attachments/assets/9a0fd34b-43cd-410b-b493-e0a986a588fe" />

***Solutions:***

```
CREATE OR REPLACE VIEW clear_country AS
SELECT *,
    CASE
        WHEN UPPER(TRIM(country)) IN ('AT', 'AUSTRIA')                            THEN 'Austria'
        WHEN UPPER(TRIM(country)) IN ('CZ', 'CZECH', 'CZECHIA', 'CZECH REPUBLIC') THEN 'Czech'
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
FROM date_clean;
```
<img width="173" height="293" alt="image" src="https://github.com/user-attachments/assets/6ae65975-952f-45f7-9969-91509b7430a9" />

***Explanation of my choices:***

1. Country names were standardized using the CASE statement, which allows multiple variants of the same country name to be mapped to a single, consistent value. This approach is transparent, easy to maintain, and simple to extend with additional mappings when needed, without modifying the source data.

2. The UPPER() function was used to eliminate inconsistencies related to letter casing. As a result, values such as "Germany", "GERMANY", and "germany" are treated as the same country.

3. The TRIM() function removes leading and trailing spaces from text values. This prevents entries such as "Poland " or " Germany" from being interpreted as separate and distinct records.

4. The ELSE 'UNMAPPED' clause makes it possible to identify values that are not covered by the existing mapping rules. This helps detect data entry errors, misspellings, or new country name variations that should be included in the standardization process. It is a safer approach than assigning unknown values to an existing category, as it reduces the risk of incorrect data classification and improves overall data quality.

### DATA STANDARDIZATION - status

<img width="643" height="628" alt="image" src="https://github.com/user-attachments/assets/c20692a3-a971-4157-af09-757b73a1ede2" />

<img width="223" height="322" alt="image" src="https://github.com/user-attachments/assets/f2dfa72e-d543-4044-b1f4-802af4924099" />

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
FROM clear_country;
```

<img width="146" height="120" alt="image" src="https://github.com/user-attachments/assets/80682117-55bd-4a49-be65-2e308c620541" />

***Explanation of my choices:***

1. The CASE statement was used to standardize the values in the status column. As a result, different variants representing the same order status, such as "SHIP" and "SHIPPED" or "DONE", "COMPLETE", and "COMPLETED", are mapped to a single, consistent category. Standardizing status values improves data quality and ensures more reliable analysis and reporting.

2. The UPPER() function eliminates inconsistencies caused by differences in letter casing, ensuring that values such as "shipped", "Shipped", and "SHIPPED" are treated identically.

3. The TRIM() function removes leading and trailing spaces from text values, preventing entries such as "SHIPPED " or " COMPLETED" from being incorrectly interpreted as separate statuses.

4. Using a VIEW allows the transformation to be performed without modifying the source data. This preserves the original dataset while providing users with clean and standardized values for analysis.

5. The ELSE NULL clause preserves information about statuses that are not covered by the existing mapping rules. This makes it easier to identify unexpected values, misspellings, or new status variants that should be included in the standardization process, rather than incorrectly assigning them to an existing category.

### DATA STANDARDIZATION - warehouse_country

<img width="429" height="520" alt="image" src="https://github.com/user-attachments/assets/af185b36-f0e3-4d06-81c1-29414620ddc2" />

<img width="198" height="440" alt="image" src="https://github.com/user-attachments/assets/d22f64d5-227f-4d51-aa05-2d5c2d132cd3" />

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


### DATA OUTLIERS - quantity

<img width="632" height="461" alt="image" src="https://github.com/user-attachments/assets/5fc28e3c-f2e6-48d0-b933-0010cb6deed3" />

Apart from two groups of values (0 and 608), quantity ranges from 1 to 13 units. Excluding 608, the mean is 3.19 and the standard deviation is 1.50. I examined both groups in the same way, using the order status, the price and the distribution across products, customers, countries and years.

quantity = 0 (1,038 records, about 0.40% of rows)

1. The order status does not explain these records: the share of cancelled orders among them (5.3%) is close to the rest of the data (6.1%), so they cannot be assumed to be cancellations.
2. Their meaning could not be verified. They may represent cancelled or returned items, but also data-entry errors or placeholder values.
3. They add nothing to revenue, but they distort the order count, the average quantity and the average order value.
4. They are flagged (is_zero_quantity) and excluded from quantity-based and average-order metrics. They stay in the view, so the decision can be reversed.

quantity = 608 (1,570 records, 0.60% of rows)

1. The value lies about 404 standard deviations above the mean of the remaining records, and the next highest quantity is 13.
2. It appears evenly across 1,152 products, 1,542 customers, all countries and all years, and it does not differ from other orders in price (mean 182.6 vs 183.2) or status (5.2% cancelled vs 6.1%). A genuine bulk order would not be expected to show such a uniform pattern, which points to a placeholder or a data error.
3. Inventory does not support orders of this size: the largest stock level in the inventory table is 567 units.
4. Their impact is substantial: they raise the average quantity from 3.19 to 6.83 and account for 53.6% of all units sold.
5. In a production environment I would confirm their meaning with the data owner. This was not possible because the dataset is synthetic.
6. They are flagged (is_quantity_anomaly) and excluded from the final analytical dataset to prevent distortion of business metrics.

```
CREATE OR REPLACE VIEW quantity_cl AS
SELECT *,
       (quantity::int = 0)   AS is_zero_quantity,
       (quantity::int = 608) AS is_quantity_anomaly
FROM status_cl;
```
Both groups are handled consistently: flagged, documented, and never deleted from the source.


### - unit_price

<img width="148" height="421" alt="image" src="https://github.com/user-attachments/assets/892790d7-d3fe-4dc5-87b0-2b2155427965" />

<img width="90" height="244" alt="image" src="https://github.com/user-attachments/assets/39ad0368-e012-4f8c-b57d-3b7dfb2a5bb2" />

<img width="280" height="74" alt="image" src="https://github.com/user-attachments/assets/cb51d2ca-f364-4232-af47-aa040afb8ae9" />

```
Select distinct(unit_price), COUNT(*)
FROM status_cl
GROUP BY unit_price
ORDER BY 2 DESC;

SELECT DISTINCT unit_price
FROM status_cl
WHERE unit_price !~ '^[0-9]+(\.[0-9]+)?$';
```

Data quality validation identified 8 records where the unit_price field contained date values (e.g., "28.mar", "26.kwi") instead of numeric prices. Since the field is intended to store monetary values only, these observations were classified as data entry or import errors and excluded from all price-related analyses.

Root cause: the source CSV was opened and saved in Excel with Polish regional settings, where a dot is read as a date separator. Values such as 24.8 were converted to dates (displayed as 24.sie), so the original number was lost.

<img width="335" height="364" alt="image" src="https://github.com/user-attachments/assets/5fcfd473-d4f8-489c-891d-169a1140df72" />

<img width="249" height="562" alt="image" src="https://github.com/user-attachments/assets/ebb33bb3-a6f3-4e35-98d2-4f8518b8881b" /> 

The remaining 1,608 records had a unit price equal to zero. As zero-priced transactions may represent valid business events such as returns, complaints, promotional orders, or missing values, they were investigated separately and retained until their business meaning was verified.

I could not identify any relationship between unit_price and either status or quantity. As with the previous anomaly, proper verification would require consultation with the data provider, but this was not possible because the dataset was synthetically generated.

Therefore, I decided to treat records with a unit_price of 0 as potential freebies, while records containing date values instead of prices were classified as data quality issues. Both groups were flagged for transparency, but only the records containing date values were considered invalid for price-related analyses.

***Solutions:***

```
CREATE VIEW unit_price_cl AS
SELECT
    *,
    CASE
        WHEN unit_price ~ '^[0-9]{1,2}\.(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)$'
            THEN NULL
        ELSE CAST(unit_price AS NUMERIC)
    END AS unit_price_clean,

    CASE
        WHEN unit_price ~ '^[0-9]{1,2}\.(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)$'
            THEN 'DATE_FLAG'
        ELSE 'OK'
    END AS unit_price_flag

FROM status_cl
```

<img width="224" height="244" alt="image" src="https://github.com/user-attachments/assets/0a92ea37-2f09-49db-a4f3-4067f4ce6b56" />

### - discount_pct

<img width="862" height="721" alt="image" src="https://github.com/user-attachments/assets/166ba157-e352-4614-b5da-a0f8b01cc747" />

Problem
The discount_pct column was stored as text and contained several data quality issues. Profiling showed five types of values:
- correctly stored numbers (e.g. 0.0, 13.84): 44.70%,
- numbers with a % symbol (e.g. 10%): 0.41%,
- values resembling dates with Polish month abbreviations (e.g. lip.41, 01.cze): 42.88%,
- missing values (NULL): 12.01%.

Altogether, only 45.10% of the values were stored as numbers.

Root cause
The source file had been opened and saved in Excel with Polish regional settings, where a dot is read as a date separator. Discounts such as 7.41 were converted to dates (displayed as lip.41), and values such as 1.6 to 01.cze. A simulation of this conversion on the original file reproduced the observed shares (42.96% vs 42.88% in the database), which confirms the explanation.

***Solutions:***

```
CREATE OR REPLACE VIEW discount_cl AS
SELECT *,
    ROUND(
        CASE
            WHEN discount_pct IS NULL OR TRIM(discount_pct) = ''                   THEN NULL
            WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?$'                              THEN discount_pct::numeric
            WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?%$'                             THEN REPLACE(discount_pct, '%', '')::numeric
            WHEN discount_pct ~ '^(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)\.[0-9]{2}$'
                 THEN ARRAY_POSITION(
                          ARRAY['sty','lut','mar','kwi','maj','cze','lip','sie','wrz','paź','lis','gru'],
                          SPLIT_PART(discount_pct, '.', 1))
                      + SPLIT_PART(discount_pct, '.', 2)::numeric / 100
            ELSE NULL
        END,
    2) AS discount_pct_clean,

    CASE
        WHEN discount_pct IS NULL OR TRIM(discount_pct) = ''                       THEN 'MISSING'
        WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?$'                                  THEN 'OK'
        WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?%$'                                 THEN 'PERCENT_CONVERTED'
        WHEN discount_pct ~ '^(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)\.[0-9]{2}$'
                                                                                   THEN 'RECOVERED_MONTH_YEAR'
        WHEN discount_pct ~ '^[0-9]{2}\.(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)$'
                                                                                   THEN 'AMBIGUOUS_DAY_MONTH'
        ELSE 'OTHER_INVALID'
    END AS discount_pct_flag
FROM unit_price_cl;
```

I created a cleaning view (discount_cl) that keeps the original column and adds discount_pct_clean (numeric, rounded to two decimals) and discount_pct_flag.

After deduplication (260,000 records), a usable discount is available for 190,447 records (73.25%): 26,048 with a discount of 0 and 164,399 with a discount above 0. The remaining 69,553 records (26.75%) are NULL.

Discounts are used only for records with a documented value, selected by the flag. Results based on discounts are reported for this subset (73.25%) and not for all orders.

<img width="684" height="158" alt="image" src="https://github.com/user-attachments/assets/18d21cf1-f87d-4532-bee8-8b21701d3a9e" />


***The data has been cleaned and is ready for analysis. The sample below shows what it looks like after cleaning.***


### Recommendations for future data collection

<img width="720" height="649" alt="image" src="https://github.com/user-attachments/assets/e59ddac8-bf46-4063-ab6f-fe489579ad38" />


