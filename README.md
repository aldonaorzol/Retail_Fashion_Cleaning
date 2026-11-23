 <h1 align="center">Retail_Fashion_Cleaning</h1>

### Data analysis:
This notebook covers the first phase of __the Retail_Fashion project__: data cleaning and preparation. The analysis itself continues in the second part, available at the link: https://github.com/aldonaorzol/Retail_Fashion_Analysis


### Tools: 
PostgreSQL, Microsoft Excel,  Copilot, Claude

### Description:

The source data contained numerous data quality issues, including __inconsistent formats__, __duplicate records__, __missing values__, and __invalid entries__. Using SQL, I performed data profiling, data cleansing, and data validation tasks as part of the ETL workflow to enhance data integrity and prepare the datasets for reliable business analysis and reporting. The process included standardizing country names, validating numerical fields, identifying potential outliers, converting data types, standardizing formats, and verifying consistency across sales, product, and inventory datasets. Additional data quality checks were conducted to ensure data accuracy, completeness, and compliance with business rules.

### Purpose:

The goal of __Retail_Fashion_Cleaning__ is to prepare the data for analysis by recovering as many records as possible while keeping every transformation accurate and traceable.

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
-discount_pct: __31,322 NULLs about 12.0% of rows__
-order_date: __629 NULLs about 0.24% of rows__

I also noticed formats __dd.mm.yyyy, yyyy.mm, yyyy.mm.dd__, non-existent dates like __2024-13-40__, and __not_a_date values__ in the order_date column. 

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
3. Complete key and analytical columns. order_id, product_id and customer_id have no gaps, so JOIN operations and duplicate detection are not compromised by missing keys.
4. NULLs are counted here after PostgreSQL converted blank fields during import. In addition to the 629 NULLs, the column contains the text placeholder not_a_date in 661 records 0.25% of all rows. Together, this gives __1,290 rows about 0.49%__ with no usable date. The column also contains dates in invalid formats or with non-existent values, such as 2024-13-40. Because order_date is essential for time-based analysis, these issues are addressed during standardization.

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
The per-column check of products 2,500 records confirms that the record count is correct. It also shows that missing values occur in only one column: 
- launch_date: __91 NULLs about 3.6% of rows__
- All other columns product_id, category, sub_category, base_price contain no NULLs.

<img width="449" height="735" alt="image" src="https://github.com/user-attachments/assets/23dddca6-e5e8-4032-9257-20fe4403bb02" />

#### Explanations:

1. Complete key and analytical columns, product_id has no gaps, so JOIN operations with sales_orders and inventory are not compromised.

2. launch_date has 91 missing values that are documented, but __they do not affect this project the column is not used in the analysis__, so the affected rows are retained and no imputation or removal is required.  These records would need to be reviewed, as they could not be included in analyses such as product age or the effect of new launches on sales.

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
The per-column check of inventory 3,741 records confirms that the record count is correct. It also shows that NULLs occur in only one column last_stock_update __6 NULLs about 0.16% of rows__

All other columns product_id, warehouse_country, stock_quantit contain no NULLs. Inspection of the source file additionally revealed that last_stock_update contains invalid entries that a NULL check does not detect: the text placeholder not_a_date, a non-existent date 2024-13-40, month 13 and day 40, and partial dates without a day e.g. 2023-06, 2023-10, 2024-04, 2024-05, 2024-07, 2024-11. The same pattern was found in products.launch_date.

Check whether the product_id field contains duplicate values. If it does not, there is no need to filter the data based on the date of the last inventory update, and the last_stock_update column can be omitted.

#### Explanations:
1. product_id and warehouse_country have no gaps, so the composite key __product_id + warehouse_country__, already confirmed as __unique__, is fully populated and JOIN operations with products are not compromised. stock_quantity is also complete, so stock levels are available for every record.
2. __last_stock_update__ is not only missing, but also inconsistent. Besides 6 NULLs, the column contains the placeholder not_a_date, an invalid date, and partial dates (year and month only). A NULL-only check therefore understates the scale of the problem: __the number of unusable dates is higher than 6__.

### Duplicated records

<img width="601" height="389" alt="image" src="https://github.com/user-attachments/assets/b58700ad-133a-4db1-ad39-1b2d75f8759d" />

#### Insights:
I checked and compared the number of rows with the number of distinct key values in each table:
- sales_orders: 260,780 rows vs. 260,000 distinct order_id values, so __780 surplus rows about 0.30% of the table__.
- products: 2,500 rows and 2,500 distinct product_id values, so the key is unique and no duplicates exist.
- inventory: __3,741 rows vs. 2,500__ distinct product_id values.

<img width="782" height="690" alt="image" src="https://github.com/user-attachments/assets/b8233935-e21b-4ec9-8981-84ce80f031a5" />

The results confirm that 780 order_id values appear twice. A comparison across all columns showed that every repeated order_id consists of identical rows, so these are true duplicates, not conflicting versions of the same order. They are removed in the next step, keeping one row per order_id.

<img width="964" height="265" alt="image" src="https://github.com/user-attachments/assets/3f0436cf-f540-4945-9270-52392fa700e0" />


<img width="349" height="764" alt="image" src="https://github.com/user-attachments/assets/589aad2c-3051-4688-b8f6-faba5aeb337b" />

The results show that the inventory table does not require deduplication, as the same product may appear in more than one warehouse.

#### Explanations:
1. product_id is __the primary key__ in the catalog. It must be unique, so any repeated value would be an error e.g. a doubled import or a merge of several sources. Duplicates in products would multiply rows in every JOIN with sales_orders and inventory, inflating revenue, quantities and all KPIs built on them. This was not the case here.
2. A repeated order_id is not automatically an error. If the table is at order-line level, one order can legitimately contain several products. Before removing anything, it was necessary to check whether the repeated rows are full duplicates identical in all columns or different line items / conflicting versions of the same order.
3. __The two cases require different treatment__. Full duplicates can be safely removed keeping one record. Rows sharing a key but differing in values need a rule for choosing the correct version, such as the most recent record, or manual review.
4. In inventory, a repeated product_id is expected. The correct uniqueness check is the composite key product_id + warehouse_country, which passed.
5. Why it matters for the analysis. Duplicates inflate totals and counts, distort averages and shares, and break the one-to-many relationship between products and sales_orders.
6. The product_id field contains no duplicate values; therefore, there is no need to filter the data following the most recent update of stock levels. So I decided to omit the column last_stock_update in my analysis.
  
### Do the data require standardization?

<img width="816" height="467" alt="image" src="https://github.com/user-attachments/assets/2add5a63-a247-4cc8-9a5e-74ae24544f09" />

#### Insights:
Unstandardized data does not cause errors. It silently distorts results. My queries and charts will run without any problem, and the numbers will look credible, but they will be wrong. When the data is aggregated, France will appear as three separate rows, France, france and FR, each holding only a fraction of the actual sales. The country ranking will therefore depend on how consistently someone typed the name, not on real sales.

<img width="183" height="522" alt="image" src="https://github.com/user-attachments/assets/111310b7-219d-4e53-b58f-cd3c2291e781" />
<img width="164" height="496" alt="image" src="https://github.com/user-attachments/assets/0093749a-84cb-4bef-b655-a7f2ffea40ae" />
<img width="180" height="321" alt="image" src="https://github.com/user-attachments/assets/2601630c-d4c4-4e25-a1d6-df601956e6c2" />


#### Explanations:
1. Why standardization is necessary. The country column stores the same country in several forms: different letter case __France, france__, full names versus codes __France, FR__, non-standard codes __GER instead of DE__ and colloquial names __Holland instead of Netherlands__. SQL treats each of these as a distinct value, so grouping, filtering and counting operate on the spelling rather than on the actual country.

2. Status values. The values __Done, Complete and Completed__ have the same meaning and should be mapped to a single value. The same applies to __Ship and Shipped__.

3. Why unambiguous values matter. Unambiguous values are important because I will be analyzing sales by country. The order status determines whether an order was actually fulfilled, generated revenue for the company and should be included in the sales results. Inconsistent status values would cause part of the real sales to be excluded or misclassified, which would distort the results for each country.

<h2 align="center">Data Cleaning</h2>

I decided to clean my data in PostgreSQL using Views. I made my decision based on:
- the raw data won't be changed; if any mistakes occur, correction will be possible.
- the data will always be updated; views are updated automatically if the database is updated 
- a view is a clear record of what I have done with the data. This makes the result easy to reproduce and explain.

### DEDUPLICATION - order_id

***Problem:*** 
The sales_orders table contained multiple rows with the same order_id, indicating duplicate records. Because each order should be represented only once, these duplicates could cause double-counting in downstream analyses and reporting.

To ensure data consistency, the dataset needed to be deduplicated while preserving one representative record for each order_id. This required identifying duplicate rows within each order group and filtering the dataset to keep only the first occurrence of every unique order.

Before:

<img width="783" height="544" alt="image" src="https://github.com/user-attachments/assets/8ab356bc-d8dd-4de4-abba-e2f037f0e1c1" />

***Solutions:***

```
CREATE OR REPLACE VIEW dedup_orders AS
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

***Explanation of the Solution:***

1. Cannot filter by ROW_NUMBER() in the same SELECT statement, because window functions are evaluated after the WHERE clause. A __CTE__ allows to calculate the number first and then filter by it.
2. __ROW_NUMBER()__ assigns each row a sequential number within its group. It takes no arguments, because the numbering does not depend on the values of the records.
3. __OVER (...)__ indicates that this is a window function: it is calculated for each row separately, based on a “window” of related rows, rather than collapsing the rows into one.
4. __PARTITION BY__ order_id splits the rows into groups that share the same order_id.
5. __ORDER BY ...__ sets the order of rows within a group. The row that comes first in that order gets number 1.

### DATE FORMAT STANDARDIZATION - order_date 

***Problem:***
order_date was stored as text. 99.01% of records 257,414 were in the DD.MM.YYYY format and __0.27% 694 values__ were partial dates in the __YYYY-MM format__. 
The remaining __0.73% 1,892__ of 260,000 records had __no usable date__: 629 NULLs, 660 not_a_date placeholders and 603 non-existent dates e.g. 2024-13-40.

<img width="394" height="170" alt="image" src="https://github.com/user-attachments/assets/b3bb7774-8851-4087-a5b0-585db9cf1445" />

Before:

<img width="789" height="686" alt="image" src="https://github.com/user-attachments/assets/4116ab4b-3ee0-4e4c-bc08-ef18df013866" />


***Solutions:***

```
CREATE OR REPLACE VIEW date_clean AS
SELECT *,
    CASE
        WHEN order_date ~ '^\d{2}\.\d{2}\.\d{4}$'
            THEN TO_DATE(order_date, 'DD.MM.YYYY')

        WHEN order_date ~ '^\d{4}-\d{2}$'
            THEN TO_DATE(order_date || '-01', 'YYYY-MM-DD')

        ELSE NULL
    END AS order_date_val,

    CASE
        WHEN LOWER(order_date) = 'not_a_date'
            THEN 'INVALID_DATE'
        ELSE 'OK'
    END AS order_date_flag,

    (order_date ~ '^\d{4}-\d{2}$') AS is_date_approximate

FROM dedup_orders;
```
***Explanation of the Solution:***
- __CASE__ expressions were used to identify supported date formats and apply the appropriate conversion rule.
- Values in the DD.MM.YYYY format are converted directly to the SQL DATE type using __TO_DATE()__.
- Partial dates stored as YYYY-MM are converted by assigning the first day of the month (YYYY-MM-01) and flagged as approximate.
- Records that do not match either supported format are converted to NULL, preventing invalid values from affecting time-based analyses.
- The order_date_flag column identifies records containing the not_a_date placeholder, while is_date_approximate distinguishes month-level dates from exact dates.

After:

<img width="1202" height="417" alt="image" src="https://github.com/user-attachments/assets/ebfbf71a-4c58-459e-9b62-8c26b857c8d2" />

### DATA STANDARDIZATION - country

***Problem:***
The country column contained multiple representations of the same country, including abbreviations, alternative country names, local-language names, and inconsistent formatting. For example, Germany appeared as DE, GER, Germany, and Deutschland, while Poland appeared as PL, POL, Poland, and Polska.

As a result, grouping or aggregating data by country would produce fragmented results, where one country could be counted as several separate categories. To improve data quality and ensure accurate analysis, the country values had to be normalized into a unified format.

Before:
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

After:

<img width="173" height="293" alt="image" src="https://github.com/user-attachments/assets/6ae65975-952f-45f7-9969-91509b7430a9" />


***Explanation of the Solution:***

1. Country names were standardized using the __CASE__ statement, which allows multiple variants of the same country name to be mapped to a single, consistent value. This approach is transparent, easy to maintain, and simple to extend with additional mappings when needed, without modifying the source data.

2. The __UPPER()__ function was used to eliminate inconsistencies related to letter casing. As a result, values such as "Germany", "GERMANY", and "germany" are treated as the same country.

3. The __TRIM()__ function removes leading and trailing spaces from text values. This prevents entries such as "Poland " or " Germany" from being interpreted as separate and distinct records.

4. The __ELSE 'UNMAPPED'__ clause makes it possible to identify values that are not covered by the existing mapping rules. This helps detect data entry errors, misspellings, or new country name variations that should be included in the standardization process. It is a safer approach than assigning unknown values to an existing category, as it reduces the risk of incorrect data classification and improves overall data quality.

### DATA STANDARDIZATION - status

***Problem:***

The status column lacked standardization and contained multiple textual representations of the same business process stage. Equivalent statuses were stored under different values, including abbreviations (SHIP), alternative forms (SHIPPED), and synonymous completion statuses (DONE, COMPLETE, COMPLETED). The data also contained formatting inconsistencies such as different letter casing and potential leading or trailing spaces.

Without standardization, aggregations based on order status would produce duplicated categories and distorted analytical results. Therefore, a data-cleaning step was required to normalize status values into a consistent and analysis-ready format.

Before:
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

After:

<img width="146" height="120" alt="image" src="https://github.com/user-attachments/assets/80682117-55bd-4a49-be65-2e308c620541" />

***Explanation of the Solution:***

1. The __CASE__ statement was used to standardize the values in the status column. As a result, different variants representing the same order status, such as "SHIP" and "SHIPPED" or "DONE", "COMPLETE", and "COMPLETED", are mapped to a single, __consistent category__. Standardizing status values improves data quality and ensures more reliable analysis and reporting.

2. The __UPPER()__ function eliminates inconsistencies caused by differences in letter casing, ensuring that values such as "shipped", "Shipped", and "SHIPPED" are treated identically.

3. The __TRIM()__ function removes leading and trailing spaces from text values, preventing entries such as "SHIPPED " or " COMPLETED" from being incorrectly interpreted as separate statuses.

4. Using a __VIEW__ allows the transformation to be performed without modifying the source data. This preserves the original dataset while providing users with clean and standardized values for analysis.

5. The __ELSE NULL__ clause preserves information about statuses that are not covered by the existing mapping rules. This makes it easier to identify unexpected values, misspellings, or new status variants that should be included in the standardization process, rather than incorrectly assigning them to an existing category.

### DATA STANDARDIZATION - warehouse_country

***Problem:***

The warehouse_country column lacked standardization and contained multiple variations of the same country, including ISO codes, abbreviations, local-language names, and full country names. As a result, inventory records belonging to the same country could be split across multiple categories, leading to inaccurate aggregations and reporting. A data-cleaning step was required to normalize the values into a consistent and analysis-ready format.

Before:

<img width="429" height="520" alt="image" src="https://github.com/user-attachments/assets/af185b36-f0e3-4d06-81c1-29414620ddc2" />

<img width="198" height="440" alt="image" src="https://github.com/user-attachments/assets/d22f64d5-227f-4d51-aa05-2d5c2d132cd3" />

***Solutions:***

```
CREATE VIEW warehouse_country_cl AS
SELECT *,
    CASE
        WHEN UPPER(TRIM(warehouse_country)) IN ('CZ', 'CZECH', 'CZECHIA', 'CZECH REPUBLIC') THEN 'Czech'
        WHEN UPPER(TRIM(warehouse_country)) IN ('DE', 'GER', 'GERMANY', 'DEUTSCHLAND')      THEN 'Germany'
        WHEN UPPER(TRIM(warehouse_country)) IN ('PL', 'POL', 'POLAND', 'POLSKA')            THEN 'Poland'
    ELSE 'UNMAPPED'
    END AS warehouse_country_cl
FROM inventory;
```

After:

<img width="610" height="416" alt="image" src="https://github.com/user-attachments/assets/7b7d22f1-9c15-4a77-9992-7084e0912a54" />

***Explanation of the Solution:***
- I created a view warehouse_country_cl with a __CASE__ expression that maps every variant to a single standardized country name.
- Each value is first normalized with __UPPER(TRIM(warehouse_country))__, so the mapping does not depend on capitalization or leading/trailing spaces. __IN (...)__ lists group all known variants of a country under one label.
- The original column is kept next to the new clear_country column, so the cleaning stays transparent and auditable, and the raw table is never modified.


### DATA OUTLIERS - quantity

***Problem:***

Apart from two groups of values __0 and 608__, quantity ranges from 1 to 13 units. Excluding 608, the mean is 3.19 and the standard deviation is 1.50. I examined both groups in the same way, using the order status, the price and the distribution across products, customers, countries and years.

quantity = 0 __1,038 records, about 0.40% of rows__

1. The order status does not explain these records: the share of cancelled orders among them 5.3% is close to the rest of the data 6.1%, so they cannot be assumed to be cancellations.
2. Their meaning could not be verified. They may represent cancelled or returned items, but also data-entry errors or placeholder values.
3. They add nothing to revenue, but they distort the order count, the average quantity and the average order value.
4. They are flagged is_zero_quantity and excluded from quantity-based and average-order metrics. They stay in the view, so the decision can be reversed.

quantity = 608 __1,570 records, 0.60% of rows__

1. The value lies about 404 standard deviations above the mean of the remaining records, and the next highest quantity is 13.
2. It appears evenly across 1,152 products, 1,542 customers, all countries and all years, and it does not differ from other orders in price mean 182.6 vs 183.2 or status 5.2% cancelled vs 6.1%. A genuine bulk order would not be expected to show such a uniform pattern, which points to a placeholder or a data error.
3. Inventory does not support orders of this size: the largest stock level in the inventory table is 567 units.
4. Their impact is substantial: they raise the average quantity from 3.19 to 6.83 and account for 53.6% of all units sold.
5. In a production environment I would confirm their meaning with the data owner. This was not possible because the dataset is synthetic.
6. They are flagged is_quantity_anomaly and excluded from the final analytical dataset to prevent distortion of business metrics.


Before:

<img width="632" height="461" alt="image" src="https://github.com/user-attachments/assets/5fc28e3c-f2e6-48d0-b933-0010cb6deed3" />

***Solutions:***
```
CREATE OR REPLACE VIEW quantity_fl AS
SELECT *,
       (quantity::int = 0)   AS is_zero_quantity,
       (quantity::int = 608) AS is_quantity_anomaly
FROM status_cl;
```

After:

<img width="328" height="348" alt="image" src="https://github.com/user-attachments/assets/c1a331a7-e382-4944-9fa6-1d731559cff1" />

***Explanation of the Solution:***
1. The expression __quantity::int = 0__ creates the is_zero_quantity flag. Records with a quantity equal to zero are identified because their business meaning cannot be verified. Although they do not contribute to revenue, they can distort metrics such as order count, average quantity, and average order value.

2. The expression __quantity::int = 608__ creates the is_quantity_anomaly flag. This value was classified as anomalous because it is far outside the normal distribution of sales quantities, where almost all orders contain between 1 and 13 units.

3. Boolean flags (TRUE/FALSE) were used instead of deleting records or replacing values. This makes the cleaning process fully transparent, reversible, and auditable, allowing future analysts to review or reconsider the assumptions behind the treatment of these observations.

4. Separating anomaly detection from anomaly removal follows a common data-quality practice. The view identifies records that may require special treatment, while the final analytical dataset can decide whether to include or exclude them depending on the business objective.

5. The solution preserves data lineage by keeping all original records available for investigation. This is particularly important because the true meaning of the anomalous values could not be confirmed with a data owner, and therefore a conservative approach was adopted.

6. By flagging both groups instead of deleting them, the dataset remains suitable for data-quality audits while preventing extreme values from distorting KPIs such as total units sold, average quantity per order, and trend analysis.

Both groups of records are handled consistently: they are identified, documented, and flagged, ensuring that analytical decisions are explicit and reproducible rather than hidden inside the cleaning process.

### QUALITY VALIDATION - unit_price

***Problem:***
- 8 records contained date values such as 28.mar and 26.kwi instead of numeric prices. These values resulted from an Excel conversion error and could not be reliably restored to their original numbers.
  
- 1,608 records had a price equal to 0. Unlike the date values, zero-priced transactions may represent valid business events e.g., freebies or promotions and therefore could not be automatically classified as errors.

Before analysis, it was necessary to distinguish between corrupted price values and potentially valid zero-priced transactions to prevent distortions in revenue and price-based metrics while preserving data transparency.

Before:

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

Data quality validation identified 8 records where the unit_price field contained date values e.g. __"28.mar", "26.kwi"__ instead of numeric prices. Since the field is intended to store monetary values only, these observations were classified as data entry or import errors and excluded from all price-related analyses.

Root cause: the source CSV was opened and saved in Excel with Polish regional settings, where a dot is read as a date separator. Values such as 24.8 were converted to dates displayed as 24.sie, so the original number was lost. I received these data in Excel file so I hadn't another option. 

<img width="335" height="364" alt="image" src="https://github.com/user-attachments/assets/5fcfd473-d4f8-489c-891d-169a1140df72" />

<img width="249" height="562" alt="image" src="https://github.com/user-attachments/assets/ebb33bb3-a6f3-4e35-98d2-4f8518b8881b" /> 

The remaining __1,608 records__ had a unit price equal to __zero__. As zero-priced transactions may represent valid business events such as returns, complaints, promotional orders, or missing values, they were investigated separately and retained until their business meaning was verified.

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

FROM quantity_fl
```

<img width="224" height="244" alt="image" src="https://github.com/user-attachments/assets/0a92ea37-2f09-49db-a4f3-4067f4ce6b56" />

***Explanation of the Solution:***
1. A regular expression identifies exactly the corrupted values. The pattern __^[0-9]{1,2}\.(sty|lut|...|gru)$__ matches the whole string in the form “day.Polish month abbreviation”. The anchors __^...$__ prevent partial matches, the escaped dot (\.) is read as a literal dot, and {1,2} covers days written with one or two digits. The profiling query that searched for every value not shaped like a number returned only these 8 values, and all of them match this pattern, so the rule covers the whole problem and nothing else.

2. CASE makes the conversion safe. A direct CAST of the whole column would fail on the first value such as 24.sie. Because CASE evaluates the cast only in the branch for values that did not match the date pattern, the view works for all other records.

3. NUMERIC is the right target type for prices. Revenue is calculated as quantity × unit_price, which needs a number, not text. NUMERIC stores values exactly, whereas floating-point types can introduce rounding errors in monetary calculations.

4. Corrupted prices are set to NULL, not guessed. A value like 24.sie could originally have been 24.8 or 24.08, and a price feeds straight into revenue, so any guess would add an error to the results. NULL is also not replaced with 0, because zero-priced records exist in the data as a separate group (potential freebies), and mixing the two would blur that distinction.

5. The records are flagged, not deleted. Only the price is invalid in these rows. Their quantity, country, date and status are still correct, so they remain available for non-price analysis, and only price-related calculations exclude them.

6. The __flag__ documents the reason. unit_price_flag (DATE_FLAG or OK) explains why a price is missing, so a NULL in unit_price_clean cannot be confused with a missing value from the source, and it shows exactly how many records were affected.


### RECOVERY PROCESS - discount_pct

***Problem:***
The discount_pct column was stored as text and contained several data quality issues. Profiling showed five types of values:
- correctly stored __numbers__ e.g. 0.0, 13.84: __44.70%__,
- numbers with __a % symbol__ e.g. 10%: __0.41%__,
- values __resembling dates__ with Polish month abbreviations e.g. lip.41, 01.cze: __42.88%__,
- missing values __NULL: 12.01%__.

Altogether, only __45.10%__ of the values were stored as numbers.

Root cause
The source file had been opened and saved in Excel with Polish regional settings, where a dot is read as a date separator. Discounts such as 7.41 were converted to dates (displayed as lip.41), and values such as 1.6 to 01.cze. A simulation of this conversion on the original file reproduced the observed shares (42.96% vs 42.88% in the database), which confirms the explanation.

Before:

<img width="862" height="721" alt="image" src="https://github.com/user-attachments/assets/166ba157-e352-4614-b5da-a0f8b01cc747" />

***Solutions:***

```
CREATE OR REPLACE VIEW discount_cl AS
SELECT *,
    ROUND(
        CASE
            WHEN discount_pct IS NULL OR TRIM(discount_pct) = '' THEN NULL
            WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?$' THEN discount_pct::numeric
            WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?%$' THEN REPLACE(discount_pct, '%', '')::numeric
            WHEN discount_pct ~ '^(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)\.[0-9]{2}$' THEN ARRAY_POSITION(
                          ARRAY['sty','lut','mar','kwi','maj','cze','lip','sie','wrz','paź','lis','gru'],
                          SPLIT_PART(discount_pct, '.', 1))
                      + SPLIT_PART(discount_pct, '.', 2)::numeric / 100
            ELSE NULL
        END,
    2) AS discount_pct_clean,
    CASE
        WHEN discount_pct IS NULL OR TRIM(discount_pct) = '' THEN 'MISSING'
        WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?$' THEN 'OK'
        WHEN discount_pct ~ '^[0-9]+(\.[0-9]+)?%$' THEN 'PERCENT_CONVERTED'
        WHEN discount_pct ~ '^(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)\.[0-9]{2}$' THEN 'RECOVERED_MONTH_YEAR'
        WHEN discount_pct ~ '^[0-9]{2}\.(sty|lut|mar|kwi|maj|cze|lip|sie|wrz|paź|lis|gru)$' THEN 'AMBIGUOUS_DAY_MONTH'
        ELSE 'OTHER_INVALID'
    END AS discount_pct_flag
FROM unit_price_cl;
```

After:

<img width="387" height="546" alt="image" src="https://github.com/user-attachments/assets/14c59842-738b-46ac-9e4c-7d8f28ec148c" />

***Results:***
1. I created a cleaning view discount_cl that keeps the original column and adds discount_pct_clean numeric, rounded to two decimals and discount_pct_flag.
2. After deduplication 260,000 records, a usable discount is available for 190,447 records 73.25%: 26,048 with a discount of 0 and 164,399 with a discount above 0. The remaining __69,553 records 26.75% are NULL__.
3. Discounts are used only for records with a documented value, selected by the flag. __Results based on discounts__ are reported for this __subset 73.25%__ and not for all orders.

***Explanation of the Solution:***
1. __CASE__ handles each type of value with its own rule.The conditions are checked in a fixed order, and only the result expression of the matching branch is evaluated. A text value is therefore cast to numeric only after it has passed the matching pattern, so invalid values cannot break the view. Anything unexpected falls into ELSE and is not silently converted.

2. Anchored regular expressions __(^...$)__ validate the whole string. Matching the entire value, not just a part of it, prevents false positives e.g. a string containing a number plus other characters. Escaping the dot (\.) ensures it is read as a literal dot, not as any character.

3. __REPLACE__, __ARRAY_POSITION__ and __SPLIT_PART__ recover what can be recovered exactly. Excel reads a value such as 7.41 as July 1941 and displays it as lip.41. Splitting the text at the dot SPLIT_PART, turning the Polish month abbreviation back into its number ARRAY_POSITION and adding the year part divided by 100 restores the original 7.41 without guessing. Values with a % sign are converted by removing the symbol REPLACE. This raised the share of usable discounts from 45.10% to 73.25% of records.

4. Values that cannot be recovered reliably are not guessed. A value like 01.cze could be 1.6 or 1.06, so it is set to NULL and flagged as AMBIGUOUS_DAY_MONTH instead of being replaced with an arbitrary number. Blanks are also kept as NULL and not replaced with 0, because a missing discount may mean either “no discount” or “not recorded”, and assuming 0 could overstate revenue.

5. __ROUND(..., 2)__ gives the column a consistent precision. Discounts are expressed with two decimal places, so rounding keeps the recovered values month + year/100 in the same format as the original ones and avoids inconsistencies in comparisons and aggregations. ROUND on a numeric value keeps the column type unchanged, which allows the view to be replaced without breaking dependent views.

6. The __flag__ column documents every decision. discount_pct_flag is built with the same conditions, in the same order, as discount_pct_clean, so the value and its explanation always match. It separates missing values MISSING from recovered RECOVERED_MONTH_YEAR, ambiguous AMBIGUOUS_DAY_MONTH, converted PERCENT_CONVERTED and invalid OTHER_INVALID ones. This makes it possible to analyse each group separately and to report how reliable the discount data is.

***To sum up***
Why not a simpler approach? Casting the column directly would raise an error on the first invalid value. Replacing all problematic values with 0 would hide the problem and overstate revenue, and deleting the rows would remove valid orders from the analysis. The chosen approach keeps every record, recovers the values that can be recovered exactly, and clearly marks the rest.

### The data has been cleaned and is ready for analysis.

<img width="1164" height="548" alt="image" src="https://github.com/user-attachments/assets/47993d2a-68b0-4215-99a8-3b538fb08eca" />

<img width="362" height="667" alt="image" src="https://github.com/user-attachments/assets/c9f41034-bfaf-4758-b9ce-985d807172d6" />

<img width="331" height="670" alt="image" src="https://github.com/user-attachments/assets/43375075-902e-4c86-ae1a-c3a19824bc37" />


### Recommendations for future data collection

<img width="833" height="611" alt="image" src="https://github.com/user-attachments/assets/b56fdaa7-0300-4fb4-852a-2eed7e5c0736" />

Expected outcome: improved data quality, more reliable reporting, reduced manual data cleaning effort.

### Take-Home messages

1. Dirty data distorts results silently. Duplicates, inconsistent labels and placeholder values e.g. quantity = 608, 0.6% of records but 53.6% of units sold would have inflated totals and averages if they had not been found and handled.

2. Flagging beats deleting. Cleaning in SQL views kept the raw data intact, and flags make every exclusion visible, auditable and reversible.

3. Most problems can be prevented at the source. Opening the CSV in Excel corrupted 42.88% of discounts into dates, so exporting directly from the system and validating data on entry would remove most of the cleaning work.
