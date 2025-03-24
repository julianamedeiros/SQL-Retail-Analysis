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
