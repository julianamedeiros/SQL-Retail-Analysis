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
