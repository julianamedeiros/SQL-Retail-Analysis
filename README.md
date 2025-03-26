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
	- What season sells the most? (comparative)
	- Are certain products more prone to returns, and why? (Defective items? Wrong sizing?)
	
2. Customer behaviour (cohort by age and sex)
	- Who are our top buyers? (track customer purchases)
	- Whats the average order value of customers?
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

**1. Analysing frequency**
**- Sales per store/city (one city has only one store):**
```sql
select city, count(distinct invoiceid) as t_sales
from pt_transactions
where transaction_type = 'Sale'
group by city
```

**- Portuguese customers per age:**
(we analyse from the transactions table because some customers might have other nationality and purchase in portuguese shops. We are considering only transactions = 'sale').
```sql
with cte as(
select c.customerid, (2025-date_part('year', date_of_birth)) as age
from customers c
right join pt_transactions pt
on c.customerid = pt.customerid
)
select age, count(distinct customerid) t
from cte
group by age
order by t desc
```
**- Portuguese customers per gender (M/F/D):**
```sql
select gender, count(distinct pt.customerid) t
from customers c 
right join pt_transactions pt
on c.customerid = pt.customerid
group by gender
order by t desc
```
**- Total purchases per customer:**
```sql
select p.customerid, (2025-date_part('year', date_of_birth)) as age, count(distinct invoiceid) as t_purchases
from pt_transactions p
left join customers c
on p.customerid = c.customerid
where transaction_type = 'Sale'
group by p.customerid, age, gender
order by t_purchases desc
```
**- Minimum, maximum and mean of a sales transaction:**
Filtering t_invoice > 0.5 to avoid possible taxes and fees.
```sql
select min(t_invoice), max(t_invoice), ROUND(avg(t_invoice)::numeric, 2)
from pt_revenue
where t_invoice > 0.5
```

**- Total spent on sales transactions per customer:**

```sql
select customerid, round(sum(distinct t_invoice)::numeric, 2) as sum
from pt_revenue r
left join pt_transactions t
on r.sale_id = t.invoiceid
where t_invoice > 0.5
group by customerid
order by sum desc
limit 5
```


**- Average of total spent in sales transactions per customer:**
```sql
with cte as(
	select distinct customerid, t_invoice 
	from pt_revenue r
	inner join pt_transactions t
	on r.sale_id = t.invoiceid
	where t_invoice > 0.5
)
select customerid, round(avg(t_invoice)::numeric, 2) as avg_spent
from cte
group by customerid
order by avg_spent desc

```

**2. Analysing correlation**
- Correlation between age and total purchases:
```sql
with cte as (
select count(distinct c.customerid) as t, (2025-date_part('year', date_of_birth)) as age
from customers c
right join pt_transactions pt
on c.customerid = pt.customerid
where transaction_type = 'Sale'
group by age
order by t desc
)
select CORR(t, age)
from cte
```
**The result is a coeficient of 0.06, suggesting that there is no relationship between customers' age and total purchases.**

- Correlation between age and total spent:
```sql
with cte as(
	select distinct customerid, t_invoice 
	from pt_revenue r
	inner join pt_transactions t
	on r.sale_id = t.invoiceid
	where t_invoice > 0.5
),
ages as (
	select customerid, (2025-date_part('year', date_of_birth)) as age, round(sum(t_invoice)::numeric, 2) as t_spent
	from cte
	inner join customers
	using (customerid)
	group by customerid, age
)
select CORR(age, t_spent)
from ages
```
**The result is a coeficient of 0.05, suggesting that there is no relationship between customers' age and total spent.**




# Data visualization
Building a report in Power BI to be used by the stakeholder. It should be simple and direct to answer BI questions.

- Data distribution: historiogram for customer buying (age and total purchase), spending habits of customers (total spent per transaction)
