![Apple_logo](https://github.com/aryan1jaiswal/Apple_sales_analysis/blob/main/Apple.png)

# Apple Sales Analysis (Advanced)

## Overview
This portfolio project contains advanced SQL analysis on **1 million+ Apple sales records**
It includes 5 key tables:
- products
- category
- sales
- warranty
- stores

## Objectives
- Identify top-performing products by analyzing pricing, quantity sold, and category-wise leaders to guide product strategy.
- Evaluate sales trends and seasonality at the product, store, and country level to uncover high-growth and underperforming areas.
- Assess store performance and consistency using monthly sales stats, standard deviation, and growth metrics. 
- Analyze warranty claims and product reliability to determine claim rates by product, store, and region. 
- Understand product lifecycle performance by tracking sales from launch through different time intervals.

## Schema

```sql

-- Creating Tables

-- Stores & category are parent tables
DROP table if exists stores;
create table stores( 
Store_ID VARCHAR(10) PRIMARY KEY,
Store_Name VARCHAR, 
City VARCHAR,
Country VARCHAR
);

DROP table if exists category
create table category( 
category_id VARCHAR(10) PRIMARY KEY,
category_name VARCHAR(30)
);

Create Table products (
    Product_ID VARCHAR(10) PRIMARY KEY,
    Product_Name VARCHAR,
    Category_ID VARCHAR(10), -- fk
    Launch_Date DATE,
    Price FLOAT,
    Constraint f_category Foreign Key (Category_ID) References category(Category_ID)
);

Create Table Sales(
sale_id	    VARCHAR PRIMARY KEY,
sale_date   DATE,
store_id	VARCHAR(10), -- fk
product_id	VARCHAR(10),  -- fk
quantity    INT,
Constraint f_stores Foreign Key(store_id) References stores(store_id),
Constraint f_products Foreign Key (product_id) References Products(product_id)
);


create table warranty( 
claim_id	VARCHAR(10)  PRIMARY KEY,
claim_date	Date,
sale_id	    VARCHAR, -- fk
repair_status  VARCHAR(30),
Constraint f_sales Foreign Key(sale_id) References Sales(sale_id)
);

--- 
```

## SQL Queries

<details>
<summary><strong> Click to expand all SQL queries</strong></summary>

<br>

### EDA

``` sql
-- sales date < launch date. Updating sales date to launch date for these rows
UPDATE sales s
SET sale_date = p.launch_date
FROM products p
WHERE s.product_id = p.product_id
AND s.sale_date < p.launch_date;

--- sales date < warranty claim date. Updating sales date to claim date for these rows

UPDATE warranty w
SET claim_date = s.sale_date
FROM sales s
Where s.sale_id = w.sale_id
AND w.claim_date = s.sale_date
```

### 1. List all products that are among the top 3 highest unique prices within their category. If there are ties at a rank, include all tied products.

``` sql
select  category_name, product_name, price
from (
select *, DEnse_rank() Over(Partition By a.category_id Order by price desc) as rk
from products a JOIN category b
ON a.category_id = b.category_id
) abc
where rk <=3;
```

### 2. For each store and month (based on sale_date), report:number of transactions, total revenue, average revenue per sale.

``` sql
select a.store_id, EXTRACT(MOnth from sale_date) as month, 
COUNT(*) as no_of_transactions, 
SUM(price*quantity) as total_revenue,
ROUND(SUM(price*quantity)::numeric/COUNT(*),2) AS avg_revenue_per_sale
from stores a JOIN sales b
ON a.store_id = b.store_id
JOIN products c
ON b.product_id = c.product_id
group by a.store_id, month

```

### 3. For each store, find the product with the highest total quantity sold. Return: store_id, product_id, product_name, total_quantity_sold.

``` sql
with cte AS(
select *, Dense_Rank() OVER (Partition By store_id Order BY total_quantity_sold desc) as rk
from (
select a.store_id, b.product_id, product_name, SUM(quantity) as total_quantity_sold
from stores a left JOIN sales b 
ON a.store_id = b.store_id
JOIN products c
ON b.product_id = c.product_id
Group by a.store_id, b.product_id, product_name
) abc 
)
 select store_id, product_id, product_name, total_quantity_sold
 from cte 
 where rk = 1

```

### 4. For each product, compute: total number of sales, total number of warranty claims, claim rate (claims/sales), rounded to 2 decimals.

``` sql

select a.product_id, b.product_name, COUNT(*) as total_sales, 
COUNT(claim_id) as total_claims, 
ROUND(COUNT(claim_id)::numeric*100/COUNT(*),2) as claim_rate
from sales a RIght JOIN products b
ON a.product_id = b.product_id
LEFT JOIN warranty c
ON a.sale_id = c.sale_id
Group by a.product_id, b.product_name
```


### 5. For each product, compare the price at launch with the average sale price over time from the sales table. Return products where the average sale price differs from the launch price by more than 10%.

``` sql
with pricing as (
select a.product_id, price as launch_price, SUM(quantity*price)/ SUM(quantity) as avg_price 
from products a LEFT JOIN sales b 
ON a.product_id = b.product_id
Group by a.product_id
)

select *, (avg_price - launch_price)*100::numeric/launch_price as ratio
from pricing
where (avg_price - launch_price)*100/launch_price >10;

--pricing was consistent so avg_price was = to launch_price

```


### 6. For each category, calculate the total revenue generated and the percentage of the total company-wide revenue it represents. Return: category_name, category_revenue, revenue_percentage.

``` sql
with cte as (
select *
from products a LEFT JOIN sales B
ON a.product_id = b.product_id
LEFT JOIN category c
ON a.category_id = c.category_id
)

select category_name, 
SUM(price*quantity) as revenue, 
ROUND((SUM(price*quantity)::numeric)*100/
(select SUM(price*quantity) from products d LEFT JOIN sales e
ON d.product_id = e.product_id)::numeric,2) as revenue_percentage
from cte
group by category_name

```

### 7. For each category, compute total sales quantity before 2022 and after 2022. Return category name, quantity before 2022, and quantity after 2022. Filter categories where the growth is negative.

``` sql
select * 
from (
select category_name, 
SUM(case when sale_date < '2022-01-01' THEN quantity 
ELSE 0 END) AS sales_before_2022,
SUM(CASE WHEN sale_date > '2022-01-01' THEN quantity 
ELSE 0 END) AS sales_after_2022
from sales a Left JOIN products b
ON a.product_id = b.product_id
JOIN category c
ON b.category_id = c.category_id
GROUP BY category_name
) abcd
where sales_before_2022 > sales_after_2022;
```

### 8. Calculate the monthly running total of sales for each store over the past four years

``` sql
with cte as (
select *, To_char(sale_date, 'YYYY-MM') as month
from sales
where sale_date <= current_date - Interval '4 years'
), 
cte1 as (
select store_id, month, COUNT(*) as cnt
from cte
group by store_id, month
)

select *, SUM(cnt) over (partition by store_id order by month) as total_sales
from cte1

```

### 9. Analyse the year-on-year growth ratio for each store in USA

``` sql
with sales_data as (
select a.store_id, a.country, Extract(Year from sale_date) as year, 
COUNT(*) as current_sales, 
LAG(COUNT(*)) OVER (partition by a.store_id order by Extract(Year from sale_date)) as prev_sales
from stores a JOIN sales b
ON a.store_id = b.store_id
group by a.store_id, a.country, year
)

select *, ROUND((current_sales - prev_sales)::numeric/prev_sales *100,4) as growth_percent
from sales_data
where country = 'United States'
```

### 10. Analyze product sales trends over time, segmented into key periods: from launch to 6 months, 6-12 months, 12-18 months, and beyond 18 months.

``` sql
select sale_time, count(*) as sales_cnt,
SUM(revenue) as revenue
from (
select *, price*quantity as revenue, 
case when AGE(sale_date, launch_date) <= '6 mons' THEN 'Immediate Sales' 
WHEN AGE(sale_date, launch_date) <= '12 mons' AND AGE(sale_date, launch_date) > '6 mons' THEN 'Early Sales'
WHEN AGE(sale_date, launch_date) <= '18 mons' AND AGE(sale_date, launch_date) > '12 mons' THEN 'Mid Sales'
ELSE 'Late Sales'
END As sale_time
from products a join sales b
ON a.product_id = b.product_id
) abc
group by sale_time
order by sales_cnt desc
```

### 11. For each store, compute the standard deviation of monthly total sales quantity over the last 2 years. Include only stores that have sales in at least 18 months.

``` sql

with sales_counts as (
select store_id, To_char(sale_date, 'MM-YYYY') as sales_month, 
SUM(quantity)::numeric as sales_cnt, count(*) as cnt
from sales
where sale_date <= current_date - Interval '2 years'
Group by store_id, sales_month
), 

sales_atleast_18mons as (
select store_id
from sales_counts
Group by store_id
HAVING count(distinct sales_month) >= 18 
),

mean as (
select a.*, AVG(sales_cnt) OVER (Partition By a.store_id) as avg_sales
from sales_counts a JOIN sales_atleast_18mons b
ON a.store_id = b.store_id
)

select *, ROUND(SQRT(POWER((sales_cnt-avg_sales), 2)/cnt),4) as Standard_dev
from mean

```

### 12. For each month, find the product with the highest number of units sold. Return month, product name, product category and total units sold. Break ties alphabetically.

``` sql
with cte as (
select sales_month, product_name, units_sold, category_id,
Dense_rank() Over (Partition By sales_month order by units_sold desc, p.product_name) as rk
from 
(
select TO_char(sale_date, 'MM-YYYY') as sales_month, product_id, 
SUM(quantity) as units_sold
from sales
group by sales_month, product_id
) abc
JOIN products p
ON abc.product_id = p.product_id
)

select To_date( '01-' || sales_month, 'DD-MM-YYYY') as sales_month , 
product_name, category_name, units_sold
from cte a JOIN category c
ON a.category_id = c.category_id 
where rk =1
order by sales_month;
```


### 13. For each store, find the number of times total monthly sales more than doubled compared to the previous month. 

``` sql
with sales_counts as (
select store_id, To_Char(sale_date, 'MM-YYYY') as sales_month, SUM(quantity)::numeric as sales_cnt
from sales
GROUP BY store_id, sales_month
), 
grouped_sales as 
(
select store_id,
To_date('01-' || sales_month, 'DD-MM-YYYY') as sales_month1, sales_cnt
from sales_counts
ORder by store_id, sales_month1
),
previous_sales as 
(
select *, 
LAG(sales_cnt) OVER (Partition By store_id) as prev_sales
from grouped_sales
)

select *
from previous_sales
where sales_cnt/prev_sales >= 2;
```

### 14. Find all products that have had at least 12 consecutive months of sales. Return product name and the starting month of this 12-month streak.

``` sql
with product_sales as (
select product_id, DATE_TRUNC('Month', sale_date)::date as sales_month
from sales
GROUP BY product_id, sales_month
), 
row_num as 
(
select *,
Row_number() Over (Partition BY product_id Order by sales_month) as rn
from product_sales
),
grouped_sales as 
(
select *, sales_month - Interval '1 month' * (rn-1) as grp_key
from row_num
),
consecutive_sales as (
select product_id, grp_key, MIN(sales_month) as start_date, COUNT(*) as streak
from grouped_sales
GROUP BY product_id, grp_key
HAVING COUNT(*) >= 12
)

select start_date, b.product_name, streak
FROM consecutive_sales a
LEFT JOIN products b
ON a.product_id = b.product_id

```


