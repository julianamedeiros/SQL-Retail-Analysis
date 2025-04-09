# SQL-Retail-Analysis
SQL-powered analysis of retail fashion data, exploring sales trends, customer behavior, and market insights to help answer stakeholders' BI questions.

*The goal and stakeholders are ficticious. This project was made for educational purposes.*

## Table of Contents:
1. [The Dataset](#the-dataset)
2. [Technologies Used](#technologies-used)
3. [Project Goal](#project-goal)
4. [Stakeholder and Business Context](#stakeholder-and-business-context)
5. [BI Questions](#bi-questions)
6. [Creating the Database](#creating-the-database)
7. [Exploratory Data Analysis (EDA)](#exploratory-data-analysis)
   - [Sales Analysis](#sales-analysis)
     - [Revenue](#revenue)
     - [Profit Margin](#profit-margin)
     - [Sales Volume and Product](#sales-volume-and-product)
     - [Returns](#returns)
     - [Store/Region Performance](#storeregion-performance)
   - [Customer Analysis](#customer-analysis)
8. [Data Visualization](#data-visualization)
9. [Key Outcomes](#key-outcomes)

## The Dataset
This dataset contains **sales simulations** for 2 years of a multinational fashion retail company, covering global sales transactions.

The data was downloaded via the API from: [Kaggle - Global Fashion Retail Stores Dataset](https://www.kaggle.com/datasets/ricgomes/global-fashion-retail-stores-dataset/data?select=customers.csv)

## Technologies Used:
- **PostgreSQL**: For database management and SQL querying
- **SQL**: Used for data manipulation and analysis
- **Power BI**: For data visualization and reporting

## Project Goal:
The primary objective of this project is to **analyze two years of transaction data** for a global fashion retail company. The goal is to provide actionable insights that will help **optimize sales and marketing strategies** for the upcoming year, with a focus on improving sales performance, understanding customer behavior, and identifying areas of improvement.

## Stakeholder and Business Context
The **Manager of Portuguese Stores** is the main stakeholder for this project. The goal is to support their decision-making by providing **data-driven insights** on:

- Sales performance trends across various stores and regions.
- Customer purchasing behavior to improve future marketing and sales strategies.

## BI Questions:
1. **Sales and Marketing Strategy**:
    - Which products and categories contribute the most to the revenue?  
    - Are there any regions or stores underperforming?
    - What season sees the highest sales?
    - Are certain products more prone to returns?

2. **Customer Behavior**:
    - Who are our top buyers?
    - What is the **average order value (AOV)** of customers?
    - Are return rates higher for certain age groups?

## Creating the database
Create db, tables and define relationships by a star schema.

```sql
CREATE DATABASE globalretail

CREATE TABLE customers (
	CustomerID int PRIMARY KEY,
	Name VARCHAR (50),
	Email text,
	Telephone VARCHAR (50),
	City text,
	Country VARCHAR (50),
	Gender VARCHAR(10),
	Date_of_birth DATE,
	Job_title TEXT
	
);

CREATE TABLE discounts (
	Start_date DATE,
	End_Date DATE,
	Discount DECIMAL,
	Description TEXT,
	Discount_perc DECIMAL,
	Category VARCHAR (20),
	Sub_category VARCHAR(50)
);

CREATE TABLE stores(
	StoreID int PRIMARY KEY,
	Country VARCHAR(50),
	City text,
	Store_name text,
	Number_of_employees INT,
	ZIP_code VARCHAR(30),
	Latitude FLOAT,
	Longitude FLOAT
);

CREATE TABLE employees(
	EmployeeID int PRIMARY KEY,
	StoreID int,
	Name VARCHAR(50),
	Position VARCHAR(20),
	CONSTRAINT fk_employees_store FOREIGN KEY (StoreID) REFERENCES stores(StoreID)
);

CREATE TABLE products(
	ProductID int PRIMARY KEY,
	Category VARCHAR(20),
	Sub_category VARCHAR(50),
	Description_PT text,
	Description_DE text,
	Description_FR text,
	Description_ES text,
	Description_EN text,
	Description_ZH text,
	Color VARCHAR(20),
	Sizes VARCHAR(20),
	Production_cost FLOAT
);


CREATE TABLE transactions (
    InvoiceID text,
    Line int,
    CustomerID int,
    ProductID int,
    Size VARCHAR(10),
    Color VARCHAR(20),
    Unit_price FLOAT,
    Quantity INT,
    Date TIMESTAMP,
    Discount DECIMAL,
    Line_total FLOAT,
    StoreID int,
    EmployeeID INT,
    Currency VARCHAR(3),
    Currency_symbol VARCHAR(3),
    Sku TEXT,
    Transaction_type VARCHAR(10),
    Payment_method VARCHAR(20),
    Invoice_total FLOAT,
    CONSTRAINT fk_transactions_customer FOREIGN KEY (CustomerID) REFERENCES customers(CustomerID),
    CONSTRAINT fk_transactions_product FOREIGN KEY (ProductID) REFERENCES products(ProductID),
    CONSTRAINT fk_transactions_store FOREIGN KEY (StoreID) REFERENCES stores(StoreID),
    CONSTRAINT fk_transactions_employee FOREIGN KEY (EmployeeID) REFERENCES employees(EmployeeID)
);
```

## Exploratory Data Analysis
This section outlines the analysis conducted on the dataset to explore sales trends, customer behavior, and more.

We begin by creating a **view** that filters transactions specifically from Portuguese stores. This view will be used in several other queries throughout the analysis to maintain consistency and avoid redundant code.

```sql
CREATE VIEW pt_transactions AS
select t.*, s.city
from transactions t
inner join stores s
on s.storeid = t.storeid
where s.country = 'Portugal'
```
## **Adressing data quality issues**

Some of the return invoices in the dataset are duplicated, causing returns to exceed the purchase value. This happens because a single invoice may have multiple return entries, leading to inflated return totals. The issue likely stems from data entry errors, where products are counted multiple times as returned.

To address this, I created a view that relates each sale to its return, keeping only the maximum (or minimum, in the case of negative values) return amount for each invoice. This ensures that no invoice or product is duplicated in the return calculations.

```sql
CREATE VIEW pt_revenue AS 
with sales as(
	select invoiceid as sale_id, invoice_total as t_sale
	from pt_transactions
	where transaction_type = 'Sale'
),
returns as (
	select invoiceid as return_id, min(invoice_total) as t_return
	from pt_transactions
	where transaction_type = 'Return'
	group by return_id
),
cte as (
	select distinct sale_id, t_sale, return_id, COALESCE(t_return, 0) t_return
	from sales s
	left join returns r
	on substring(sale_id, 5) = substring(return_id, 5)
)
select sale_id, t_sale, return_id, t_return, (t_sale + t_return) as t_invoice
from cte
```

## **Sales Analysis**
### Revenue

**Total revenue per year**
```sql
select ROUND(sum(t_invoice)::numeric, 2) as total_revenue, date_part('year', date) as year
from pt_revenue r
left join pt_transactions t
on r.sale_id = t.invoiceid
where t_invoice > 0
group by date_part('year', date)
limit 20
```

**Total revenue per month/year**
```sql
SELECT date_trunc('month', date) as month, round(sum(t_invoice)::numeric, 2) as monthly_revenue
from pt_revenue
left join pt_transactions
on sale_id = invoiceid
where t_invoice > 0 
group by month
```

**Seasonal net sales peak**
```sql
WITH CTE AS(
	SELECT date_trunc('month', date) as month, round(sum(t_invoice)::numeric, 2) as monthly_rev
	from pt_revenue
	left join pt_transactions
	on sale_id = invoiceid
	where t_invoice > 0 
	group by month
),
trends as (
	SELECT month, monthly_rev,
		LAG(monthly_rev) OVER(ORDER BY month) as prev_month_rev,
		LEAD(monthly_rev) OVER(ORDER BY month) as next_month_rev
	FROM cte
)
select *
from trends
where monthly_rev > prev_month_rev 
and monthly_rev > next_month_rev
```

**Minimum, maximum and mean of sales revenue**
Filtering t_invoice > 0.5 to avoid possible taxes and fees.
```sql
select min(t_invoice), max(t_invoice), ROUND(avg(t_invoice)::numeric, 2)
from pt_revenue
where t_invoice > 0.5
```

### Profit Margin

**Overall profit margin**
```sql
with temp as (
	select invoiceid, line_total, quantity, productid
	from pt_revenue
	left join pt_transactions
	on pt_revenue.sale_id = pt_transactions.invoiceid
	where return_id is null
),
cte as(
	select productid, invoiceid, (production_cost*quantity) as production_cost, line_total as sale_value, line_total - (production_cost*quantity) as profit_margin
	from temp
	left join products
	using(productid)
)
select sum(production_cost) as t_costs, sum(sale_value) as t_sales, sum(profit_margin) as profit, sum(profit_margin)*100.0/sum(sale_value) as profit_perc
from cte
```

**Profit margin per quarter**
```sql
with temp as (
	select invoiceid, line_total, quantity, productid, date_trunc('quarter', date) as quarter
	from pt_revenue
	left join pt_transactions
	on pt_revenue.sale_id = pt_transactions.invoiceid
	where return_id is null
),
cte as(
	select productid, invoiceid, (production_cost*quantity) as production_cost, 
		line_total as sale_value, line_total - (production_cost*quantity) as profit_margin,
		quarter
	from temp
	left join products
	using(productid)
)
select quarter, round(sum(production_cost)::numeric, 2) as t_costs, round(sum(sale_value)::numeric, 2) as t_sales, 
	round(sum(profit_margin)::numeric,2) as profit, 
	round((sum(profit_margin)*100.0/sum(sale_value))::numeric, 2) as profit_perc
from cte
group by quarter
order by quarter asc
```
**Profit margin per product sale**
```sql
with cte as (
	select productid, sale_id, quantity, line_total 
	from pt_revenue r 
	left join pt_transactions t
	on r.sale_id = t.invoiceid
	where return_id is null
),
temp as (
	select sale_id, productid, (production_cost*quantity) as cost, line_total as revenue,
		line_total-(production_cost*quantity) as profit
	from cte
	left join products
	using (productid)
)
select productid, round(sum(cost)::numeric, 2) as total_cost, round(sum(revenue)::numeric, 2) as total_revenue, 
	round(sum(profit)::numeric, 2) as total_profit,
	round((sum(profit)*100.0/sum(revenue))::numeric, 2) as profit_perc
from temp
group by productid
order by profit_perc desc
limit 5
```

**Profit margin per product subcategory**
```sql
with cte as (
	select productid, sale_id, quantity, line_total 
	from pt_revenue r 
	left join pt_transactions t
	on r.sale_id = t.invoiceid
	where return_id is null
),
temp as (
	select sale_id, productid, sub_category, (production_cost*quantity) as cost, line_total as revenue,
		line_total-(production_cost*quantity) as profit
	from cte
	left join products
	using (productid)
)
select sub_category, round(sum(cost)::numeric, 2) as total_cost, round(sum(revenue)::numeric, 2) as total_revenue, 
	round(sum(profit)::numeric, 2) as total_profit,
	round((sum(profit)*100.0/sum(revenue))::numeric, 2) as profit_perc
from temp
group by sub_category
order by profit_perc desc
```
**Profit margin per product category**
```sql
with cte as (
	select productid, sale_id, quantity, line_total 
	from pt_revenue r 
	left join pt_transactions t
	on r.sale_id = t.invoiceid
	where return_id is null
),
temp as (
	select sale_id, productid, category, (production_cost*quantity) as cost, line_total as revenue,
		line_total-(production_cost*quantity) as profit
	from cte
	left join products
	using (productid)
)
select category, round(sum(cost)::numeric, 2) as total_cost, round(sum(revenue)::numeric, 2) as total_revenue, 
	round(sum(profit)::numeric, 2) as total_profit,
	round((sum(profit)*100.0/sum(revenue))::numeric, 2) as profit_perc
from temp
group by category
order by profit_perc desc
limit 50
```
### Sales Volume and Product

**Top 10 net sold products**
```sql
with sales as (
	select r.sale_id, t_invoice, p.productid, p.unit_price, p.quantity, discount, p.line_total, date
	from pt_revenue r
	left join pt_transactions p
	on r.sale_id = p.invoiceid
	where t_invoice > 0
)
SELECT productid, category, sub_category, description_pt, sum(quantity) total_sold
from sales
left join products
using (productid)
group by productid, category, sub_category, description_pt
order by total_sold desc
limit 10
```

**Total net sales per quarter**
```sql
select date_trunc('quarter', date) as quarter, count(distinct sale_id) as total_sales
from pt_revenue
left join pt_transactions
on pt_revenue.sale_id = pt_transactions.invoiceid
where return_id is null
group by quarter
order by quarter asc
```

**Total products net sold per quarter**
```sql
select date_trunc('quarter', date) as quarter, count(productid) as t_sold_products
from pt_revenue
left join pt_transactions
on pt_revenue.sale_id = pt_transactions.invoiceid
where return_id is null
group by quarter
order by quarter asc
```

**Products net sold per subcategory and quarter**
```sql
with sales as (
	select r.sale_id, t_invoice, p.productid, p.unit_price, p.quantity, discount, p.line_total, date_trunc('quarter', date) as quarter
	from pt_revenue r
	left join pt_transactions p
	on r.sale_id = p.invoiceid
	where t_invoice > 0
)
SELECT sub_category, sum(quantity) as products_sold, quarter
from sales
left join products
using(productid)
group by sub_category, quarter
order by quarter, sub_category desc
```

**% Net sales per category comparison**
```sql
with sales as (
	select r.sale_id, t_invoice, p.productid, p.unit_price, p.quantity, discount, p.line_total, date
	from pt_revenue r
	left join pt_transactions p
	on r.sale_id = p.invoiceid
	where t_invoice > 0
),
cte as(
	SELECT category, sum(quantity) as t_per_cat
	from sales
	left join products
	using(productid)
	group by category
)
select category, round(t_per_cat*100.0/(select sum(quantity) from sales),2) as perc
from cte
order by perc desc
```

**% Contribution to Net Sales Revenue per Category**
```sql
with sales as (
	select r.sale_id, t_invoice, p.productid, p.unit_price, p.quantity, discount, p.line_total, date
	from pt_revenue r
	left join pt_transactions p
	on r.sale_id = p.invoiceid
	where t_invoice > 0
),
cte as(
	SELECT category, round(sum(line_total)::numeric,2) as revenue_per_category
	from sales
	left join products
	using(productid)
	group by category
)
select category, round(revenue_per_category*100.0/(select sum(line_total)::numeric from sales),2) as rev_per
from cte
order by rev_per desc
```
**% Contribution to Net Sales Revenue per Subcategory**
```sql
with sales as (
	select r.sale_id, t_invoice, p.productid, p.unit_price, p.quantity, discount, p.line_total, date
	from pt_revenue r
	left join pt_transactions p
	on r.sale_id = p.invoiceid
	where t_invoice > 0
),
cte as(
	SELECT sub_category, round(sum(line_total)::numeric,2) as revenue_per_sub_category
	from sales
	left join products
	using(productid)
	group by sub_category
)
select sub_category, round(revenue_per_sub_category*100.0/(select sum(line_total)::numeric from sales),2) as rev_per
from cte
order by rev_per desc
```
**% Contribution to Net Sales Revenue per Product**
```sql
with sales as (
	select r.sale_id, t_invoice, p.productid, p.unit_price, p.quantity, discount, p.line_total, date
	from pt_revenue r
	left join pt_transactions p
	on r.sale_id = p.invoiceid
	where t_invoice > 0
),
cte as(
	SELECT productid, round(sum(line_total)::numeric,2) as revenue_per_product, count(productid) as t_sold
	from sales
	left join products
	using(productid)
	group by productid
)
select productid, t_sold, round(revenue_per_product*100.0/(select sum(line_total)::numeric from sales),4) as rev_per
from cte
order by rev_per desc
limit 10
```

### Returns

**Overall returns rate**
```sql
with cte as(
	select sum(quantity) as t_products_returned
	from pt_transactions
	where transaction_type = 'Return'
),
sale as(
	select sum(quantity) as t_products_sold
	from pt_transactions
	where transaction_type = 'Sale'
)
select t_products_sold, t_products_returned, round(t_products_returned*100.0/t_products_sold,2) as return_perc
from cte
cross join sale
```

**Products with more than 50% return rate**
```sql
with cte as(
	select productid, sum(quantity) as t_sold
	from pt_transactions
	where transaction_type = 'Sale'
	group by productid
),
sale as(
	select productid, sum(quantity) as t_returned
	from pt_transactions
	where transaction_type = 'Return'
	group by productid
),
measures as(
	select productid, t_sold, t_returned, round(t_returned*100.0/t_sold, 2) as perc
	from cte
	right join sale
	using(productid)
	where t_returned*100.0/t_sold <=100
	and t_sold > 5
)
select m.*, category, sub_category, description_pt 
from measures m
left join products p
using(productid)
where perc >=50 
order by perc desc
```

**Top 10 product categories with high return rate**
```sql
with cte as(
	select productid, sum(quantity) as t_sold
	from pt_transactions
	where transaction_type = 'Sale'
	group by productid
),
sale as(
	select productid, sum(quantity) as t_returned
	from pt_transactions
	where transaction_type = 'Return'
	group by productid
),
joined as(
	select productid, t_sold, t_returned
	from cte
	right join sale
	using(productid)
)
select sub_category, sum(t_sold) as t_products_sold, sum(t_returned) t_products_returned, 
	round((sum(t_returned)*100.0/sum(t_sold))::numeric, 2) as return_rate
from joined
left join products
using(productid)
group by sub_category
order by return_rate desc
limit 10
```

**Returns per age group**
```sql
with cte as(
	select customerid, 
		CASE 
			WHEN 2025-date_part('year', date_of_birth) <25 then '18-24'
			WHEN 2025-date_part('year', date_of_birth) BETWEEN 25 and 34 then '25-34'
			WHEN 2025-date_part('year', date_of_birth) BETWEEN 35 and 44 then '35-44'
			WHEN 2025-date_part('year', date_of_birth) BETWEEN 45 and 54 then '45-54'
			WHEN 2025-date_part('year', date_of_birth) >54 then '55+'
		END as age
	from rfm
	left join customers
	using(customerid)
	where frequency_score = 5 and monetary_score = 5 and recency_score = 5
)
select age, count(invoiceid) as total_returns
from cte
left join pt_transactions
using(customerid)
where transaction_type = 'Return'
group by age
order by age 

```


### Store/Region Performance

**Quartely revenue per store and region**
```sql
select round(sum(t_invoice)::numeric, 2) as revenue, storeid, city, date_trunc('quarter', date) as quarter
from pt_revenue r
left join pt_transactions t
on sale_id = invoiceid
where t_invoice > 0
group by storeid, quarter, city
order by quarter, storeid
```
Categories


## **Customer Analysis**

**Total purchases, max spent, min, spent and AOV per customer**
```sql
with cte as(
	select distinct customerid, t_invoice 
	from pt_revenue r
	inner join pt_transactions t
	on r.sale_id = t.invoiceid
	where t_invoice > 0.5
)
select customerid, round(avg(t_invoice)::numeric, 2) as avg_spent, max(t_invoice) as max_spent, min(t_invoice) as min_spent, count(t_invoice) as total_purchases
from cte
group by customerid
order by avg_spent desc
```

**Customers from portuguese shops per age and gender**
```sql
with cte as (
	select r.*, customerid
	from pt_revenue r 
	left join pt_transactions t
	on r.sale_id = t.invoiceid
)
select customerid, 2025-date_part('year', date_of_birth) as age, gender, round(sum(t_invoice)::numeric, 2) as total_revenue
from cte
left join customers
using(customerid)
group by customerid, age, gender
order by total_revenue desc
```

**Max and min customers age**
```sql
with cte as (
	select r.*, customerid
	from pt_revenue r 
	left join pt_transactions t
	on r.sale_id = t.invoiceid
)
select max(2025-date_part('year', date_of_birth)) as max_age,  min(2025-date_part('year', date_of_birth)) as min_age
from cte
left join customers
using(customerid)
```

### **RFM segmentation**

```sql
with cte as(
	select customerid, 
		max(date_trunc('day', date)::date) as last_purchase_date, 
		count(distinct invoiceid) as frequency, 
		round(sum(invoice_total)::numeric, 2) as monetary
	from pt_transactions
	where transaction_type = 'Sale'
	group by customerid
), rfm as(
	select customerid, (current_date - last_purchase_date) as recency, frequency, monetary
	from cte
),
ntiles as (
SELECT customerid, recency, frequency, monetary,
	ntile(5) over(order by recency desc) as recency_score,
	ntile(5) over(order by frequency asc) as frequency_score,
	ntile(5) over(order by monetary asc) as monetary_score
from rfm
)
select *
from ntiles
order by recency_score desc, frequency_score desc, monetary_score desc

```

**Finding top customers**
```sql
with cte as(
	select customerid, 
		max(date_trunc('day', date)::date) as last_purchase_date, 
		count(distinct invoiceid) as frequency, 
		round(sum(invoice_total)::numeric, 2) as monetary
	from pt_transactions
	where transaction_type = 'Sale'
	group by customerid
), rfm as(
	select customerid, (current_date - last_purchase_date) as recency, frequency, monetary
	from cte
),
ntiles as (
SELECT customerid, recency, frequency, monetary,
	ntile(5) over(order by recency desc) as recency_score,
	ntile(5) over(order by frequency asc) as frequency_score,
	ntile(5) over(order by monetary asc) as monetary_score
from rfm
)
select customerid, recency_score, frequency_score, monetary_score
from ntiles
where recency_score = 5 and frequency_score = 5 and monetary_score = 5
order by recency_score desc, frequency_score desc, monetary_score desc
```

**Creating a view with the RFM segmentation**
```sql
CREATE VIEW rfm AS ( 
with cte as(
	select customerid, 
		max(date_trunc('day', date)::date) as last_purchase_date, 
		count(distinct invoiceid) as frequency, 
		round(sum(invoice_total)::numeric, 2) as monetary
	from pt_transactions
	where transaction_type = 'Sale'
	group by customerid
), rfm as(
	select customerid, (current_date - last_purchase_date) as recency, frequency, monetary
	from cte
),
ntiles as (
SELECT customerid, recency, frequency, monetary,
	ntile(5) over(order by recency desc) as recency_score,
	ntile(5) over(order by frequency asc) as frequency_score,
	ntile(5) over(order by monetary asc) as monetary_score
from rfm
)
select *
from ntiles
order by recency_score desc, frequency_score desc, monetary_score desc
)
```

### **Demographic analysis of top consumers**
**Max, Min and AVG age of top customers**
```sql
select avg(2025-date_part('year', date_of_birth)), max(2025-date_part('year', date_of_birth)), 
	min(2025-date_part('year', date_of_birth))
from rfm
left join customers
using(customerid)
where frequency_score = 5 and monetary_score = 5 and recency_score = 5
```
**Count top customers per age group**
```sql
with cte as(
	select customerid, 
		CASE 
			WHEN 2025-date_part('year', date_of_birth) <25 then '18-24'
			WHEN 2025-date_part('year', date_of_birth) BETWEEN 25 and 34 then '25-34'
			WHEN 2025-date_part('year', date_of_birth) BETWEEN 35 and 44 then '35-44'
			WHEN 2025-date_part('year', date_of_birth) BETWEEN 45 and 54 then '45-54'
			WHEN 2025-date_part('year', date_of_birth) >54 then '+55'
		END as age
	from rfm
	left join customers
	using(customerid)
	where frequency_score = 5 and monetary_score = 5 and recency_score = 5
)
select age, count(customerid)
from cte
group by age
order by age
```
**Gender ratio of top customers**
```sql
with cte as(
	select gender, COUNT(gender) as t_gender
	from rfm
	left join customers
	using(customerid)
	where frequency_score = 5 and monetary_score = 5 and recency_score = 5
	GROUP BY gender
), total as (
	select count(distinct customerid) as t_customers
	from rfm
	where frequency_score = 5 and monetary_score = 5 and recency_score = 5
)
select gender, round(t_gender*100 / t_customers, 2) as ratio
from cte
cross join total
```

## Data Visualization
To effectively present the insights from the analysis, a **Power BI report** was built to help the stakeholder make informed decisions. The report is designed to be **simple**, **intuitive**, and **direct**, with visualizations that specifically address the key BI questions. 

The following components are included in the report:
- **Revenue trends** over time (yearly, monthly).
- **Top-performing products** by category and subcategory.
- **Profit margins** across categories and subcategories.
- **Customer behavior analysis**, including top buyers and return rates.
- **Regional performance**, highlighting stores or areas that need attention.

The goal is for the report to provide actionable insights at a glance, allowing the stakeholder to easily identify trends and make decisions based on the visualized data.


## Key Outcomes

- - **Sales Peak**: The quarter of **2023/10** saw a peak in **sales**, and **revenue**, though the **profit margin** was average.
  
- **Top Subcategories by Revenue**:
  - **'Coats and Blazers'**: 13.2% of total sales revenue
  - **'Pants and Jeans'**: 12.5% of total sales revenue
  - **'Suits and Sets'**: 11.7% of total sales revenue

- **Most Profitable Products**: Products from the subcategory **'Baby (0-12 months)'** have the highest profit margins, despite lower sales volume and contribution to overall revenue.

- **Category Contribution to Sales Revenue**:
  - **Feminine**: 60.1% of total sales revenue
  - **Masculine**: 30.8% of total sales revenue
  - **Children**: 8.9% of total sales revenue

- **Profit Margin by Category**:
  - **Feminine**: 57% profit margin
  - **Masculine**: 57% profit margin
  - **Children**: 54.5% profit margin

- **Sales Peaks**: The months of **March**, **September**, **October**, and **December** saw significant sales activity.

- **Return Rate**: Overall, **5.5%** of all purchased products were returned.

- **Highest Return Rates by Subcategory**:
  - **'Baby (0-12 months)'**: 9.3% return rate
  - **'Pajamas'**: 8.8% return rate
  - **'Underwear and Pajamas'**: 8.2% return rate

- **Top Customers**: The highest-spending customers are **women aged 35-54**.

- **Return Rates by Age Group**:
  - Return rates are higher for age groups between **45-54** and **35-44**—groups that also have the highest purchases.

These insights are crucial for understanding customer behavior, sales trends, and areas of opportunity for future strategy adjustments.
