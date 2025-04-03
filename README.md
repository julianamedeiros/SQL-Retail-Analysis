# SQL-Retail-Analysis
SQL-powered analysis of retail fashion data, exploring sales trends, customer behavior, and market insights to help answer stakeholders' BI questions.
*The goal and stakeholders are ficticious. This project was made for educational purposes.

# Contents:
1. The dataset
2. Technologies Used
3. Project Goal
4. Stakeholder and Business Context
5. BI questions
6. Creating the Database
7. Exploratory Data Analysis
   7.1. Sales analytics
   7.2. Customer behaviour
8. Data visualization
9. Business insights


# The dataset
Sales simulation for 2 years of a multinational brand.
Downloaded through API. In: https://www.kaggle.com/datasets/ricgomes/global-fashion-retail-stores-dataset/data?select=customers.csv

# Technologies used:
- PostgreSQL
- SQL
- Power BI


# Project Goal:
This project aims to analyze two years of transaction data for a global fashion retail company to support Business Intelligence (BI) decision-making. The main objective is to provide actionable insights that help stakeholders optimize sales and marketing strategies for the upcoming year.

# Stakeholder & Business Context
The primary stakeholder is the **Manager of Portuguese Stores**, who needs data-driven insights to make informed decisions about:
- Sales performance trends
- Customer purchasing behavior

# BI questions:
1.  Sales and marketing strategy:
	- Which products and categories contribute the most and the less to the revenue?
	- Are there regions or stores that are underperforming?
	- What season sells the most? 
	- Are certain products more prone to returns?
	
2. Customer behaviour 
	- Who are our top buyers?
	- Whats the average order value (AOV) of customers?
	- Do younger customers buy different products than older ones?
	- Are return rates higher for certain age groups?

# Creating the database
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

# Exploratory Data Analysis

We start by creating a **view** to store a filter on transactions made in Portuguese stores. We will be using this filter in many other queries.

```sql
CREATE VIEW pt_transactions AS
select t.*, s.city
from transactions t
inner join stores s
on s.storeid = t.storeid
where s.country = 'Portugal'
```
**Adressing data quality issues**
Some data on return invoices are duplicated (the same value twice, or some of the products being returned twice), causing returns to exceed purchase (one invoice should have only one invoice total, but the issue cuases one invoice to have two or more invoice total, caused by the some product being counted twice). The problem is probablt related to data entry. My solution is to relate each sale with its return in a view, but keeping only the max (min is used in the query since its a negative value) returned value to each invoice (we then avoid duplication of the entire invoice, or duplication of some returned products).
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

## **1. Sales analytics**
**Sales, profit and revenue by month, quarter and year**

**- Minimum, maximum and mean of a sales revenue:**
Filtering t_invoice > 0.5 to avoid possible taxes and fees.
```sql
select min(t_invoice), max(t_invoice), ROUND(avg(t_invoice)::numeric, 2)
from pt_revenue
where t_invoice > 0.5
```

**-Total revenue per year:**
```sql
select ROUND(sum(t_invoice)::numeric, 2) as total_revenue, date_part('year', date) as year
from pt_revenue r
left join pt_transactions t
on r.sale_id = t.invoiceid
where t_invoice > 0
group by date_part('year', date)
limit 20
```
**-Total revenue per month/year:**
```sql
SELECT date_trunc('month', date) as month, round(sum(t_invoice)::numeric, 2) as monthly_revenue
from pt_revenue
left join pt_transactions
on sale_id = invoiceid
where t_invoice > 0 
group by month
```

**-Total quartely revenue:**
```sql
SELECT date_trunc('quarter', date) as quarter, round(sum(t_invoice)::numeric, 2) as quartely_revenue
from pt_revenue
left join pt_transactions
on sale_id = invoiceid
where t_invoice > 0 
group by quarter
```
**-Analysing seasonality trends (per month):**
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
**- Quartely revenue per store:**
```sql
select round(sum(t_invoice)::numeric, 2) as revenue, storeid, city, date_trunc('quarter', date) as quarter
from pt_revenue r
left join pt_transactions t
on sale_id = invoiceid
where t_invoice > 0
group by storeid, quarter, city
order by quarter, storeid
```
**- Top 10 most sold products:**
```sql
with sales as (
	select r.sale_id, t_invoice, p.productid, p.unit_price, p.quantity, discount, p.line_total, date
	from pt_revenue r
	left join pt_transactions p
	on r.sale_id = p.invoiceid
	where t_invoice > 0
)
SELECT productid, sum(quantity)
from sales
group by productid
order by sum desc
limit 10
```
**-Products sold per subcategory and quarter:**
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

**- Percentage comparison between sales for each category:**
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
**-Percentage of total sales revenue contribution per category**:
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
**-Percentage of total sales revenue contribution per sub category:**
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
**-Percentage of total sales revenue contribution per product:**
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

**-Overall profit margin:**
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

**- Profit margin per quarter:**
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
**- Profit margin per product sale:**
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

**-Profit margin per product subcategory:**
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
**-Profit margin per product category:**
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

**- Total sales per quarter:**
```sql
select date_trunc('quarter', date) as quarter, count(distinct sale_id) as total_sales
from pt_revenue
left join pt_transactions
on pt_revenue.sale_id = pt_transactions.invoiceid
where return_id is null
group by quarter
order by quarter asc
```

**- Total products sold per quarter:**
```sql
select date_trunc('quarter', date) as quarter, count(productid) as t_sold_products
from pt_revenue
left join pt_transactions
on pt_revenue.sale_id = pt_transactions.invoiceid
where return_id is null
group by quarter
order by quarter asc
```

**-Total returned products:**
```sql
with cte as(
	select sum(quantity) as t_returned
	from pt_transactions
	where transaction_type = 'Return'
),
sale as(
	select sum(quantity) as t_sold
	from pt_transactions
	where transaction_type = 'Sale'
)
select t_sold, t_returned
from cte
cross join sale
```

**-Returns rate**:
```sql
with cte as(
	select sum(quantity) as t_returned
	from pt_transactions
	where transaction_type = 'Return'
),
sale as(
	select sum(quantity) as t_sold
	from pt_transactions
	where transaction_type = 'Sale'
)
select round(t_returned*100.0/t_sold,2) as return_perc
from cte
cross join sale
```

**-Top 10 products with high return rate:**
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
)
select productid, t_sold, t_returned, round(t_returned*100.0/t_sold, 2) as perc
from cte
right join sale
using(productid)
where t_returned*100.0/t_sold <=100
and t_sold > 5
order by perc desc
limit 10
```

**-Top 10 product categories with high return rate:**
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

## **2. Customer analytics**

**- Total purchases, max spent, min, spent and AOV per customer:**
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

**- Customers from portuguese shops per age and gender-**
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

**- Max and min customers age:**
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
**-RFM analysis:**
```sql
with cte as(
	select customerid, 
		max(date_trunc('day', date)::date) as last_purchase_date, 
		count(distinct invoiceid) as frequency, 
		round(sum(invoice_total)::numeric, 2) as monetary
	from pt_transactions
	where transaction_type = 'Sale'
	group by customerid
)
select customerid, (current_date - last_purchase_date) as recency, frequency, monetary
from cte
order by monetary desc
```
**- Defining RFM scores:**
Calculating percentiles Q1, Q2, Q3 and Q4.
```sql
with cte as(
	select customerid, 
		max(date_trunc('day', date)::date) as last_purchase_date, 
		count(distinct invoiceid) as frequency, 
		round(sum(invoice_total)::numeric, 2) as monetary
	from pt_transactions
	where transaction_type = 'Sale'
	group by customerid
),
rfm as(
	select customerid, (current_date - last_purchase_date) as recency, frequency, monetary
	from cte
	order by monetary desc
)
select max(recency), min(recency), avg(recency),
	max(frequency), min(frequency), avg(frequency),
	max(monetary), min(monetary), avg(monetary)
from rfm
```

Output (max, min, and avg):
- Recency: 823, 16, 243.0
- Frequency: 30, 1, 3.8
- Monetary: 8650, 2.5, 516.9




# Data visualization
Building a report in Power BI to be used by the stakeholder. It should be simple and direct to answer BI questions.

- Data distribution: historiogram for customer buying (age and total purchase), spending habits of customers (total spent per transaction)

# BI insights
- The quarter of 2023/10 had a peak in sales, products sold, and revenue, but the profit margin was average.
- Products from the subcategory 'Coats and Blazer', 'Pants and Jeans' and 'Suits and Sets' contribute the most to the total sales revenue (respectivelly 13.2%, 12.5%, 11.7%).
- Products from the subcategory 'Baby(0-12 months)' are the most profitable, even though their sales volume and contribution to total sales revenue is lower than average.
- Products from the category 'Feminine' contribute 60.1% to the total sales revenue, Masculine 30.8% and Children 8.9%.
- Products from the category 'Feminine' and 'Masculine' have a 57% of profit margin, while 'Children' have 54.5% of profit margin.
- Sales peak in the months of March, September, October and December.
- 5.5% of all bought products were returned.
- Products from the subcatgory 'Baby (0-12months)', 'Pajamas' and 'Underwear and pajamas' have the highers return rate (respectively 9.3%, 8.8%, and 8.2%).
