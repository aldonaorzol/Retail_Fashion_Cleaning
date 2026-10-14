Data cleaning

Sales_orders

Excel:
1. Splitting values ​​into columns. 
Data -> Text to columns -> Comma

Before
<img width="793" height="570" alt="image" src="https://github.com/user-attachments/assets/b75342d6-697a-4168-a12b-9e16b9bb3819" />


After

<img width="688" height="599" alt="image" src="https://github.com/user-attachments/assets/8057c7c8-1bb6-45a5-98f1-876d1c8ec4f1" />

PostgreSQL:

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
```

I open my folder in notebook to veryfy delimiters. It is semicolon ; 
Import data -> 
