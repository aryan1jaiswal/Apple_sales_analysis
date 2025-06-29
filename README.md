# Apple_sales_analysis

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







