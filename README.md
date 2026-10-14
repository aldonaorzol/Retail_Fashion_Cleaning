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

```
CREATE TABLE sales_orders (
    order_id INT,
    order_date DATE,
    customer_id INT,
    country VARCHAR(50),
    product_id INT,
    quantity INT,
    unit_price NUMERIC(10,2),
    discount_pct NUMERIC(10,2),
    status VARCHAR(20)
);
```

I open my folder in notebook to veryfy delimiters. It is semicolon ; 
Import data -> 
