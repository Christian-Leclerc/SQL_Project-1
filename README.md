# Final-Project-Transforming-and-Analyzing-Data-with-SQL
*by Christian Leclerc b.ing, Data Analyst, [LHL Bootcamp](https://www.lighthouselabs.ca)*
<br>
***Note:***
This file is better viewed using [GitHub](https://github.com) preview screen (or any Markdown viewer) to be able to use the relative links.

## Project/Goals
This repository contains the code and documentation for my data analysis project on ***Transforming and Analysing Data with SQL.***
The project aimed to demonstrate my comprehension and skills related to the content of the *Data Analyst - FLEX [LHL Bootcamp](https://www.lighthouselabs.ca)* program so far, by analysing a fictional ecommerce dataset using PostgreSQL.

My goal, as a student, is to identify my strengths and weaknesses during the project along with increasing my knowledge of the coding and the tools available.

## Table of Contents
- [Project Background](#project-background-and-ressource)
- [Process](#process)
- [Results](#results)
- [Challenges](#challenges)
- [Future Goals](#future-goals)
- [Tech stack](#tech-stack)

## Project Background and Ressources
The dataset is compose of:
- raw data as .csv files
- each files represent a table to be imported in pgAdmin 4:
    + **all_sessions:** FACT table with transactional information about each product purchased
    + **analytics:** FACT table with more detailed transactional information within each session
    + **product:** DIMENSION table with information about each product sold
    + **sales_report:** REPORT table representing a time capture of the product table
    + **sales_by_sku:** REPORT  table identical to sales_report but with only the total 
    ordered product.

The gitHub repo contains .md files (Markdown) with instructions and questions to answer about the dataset.
The dataset are given without context on what they represent, without units of measurement nor how each table relate to each other. This is not a normal real world environment (I hope), thought the messy dataset is totally plausible...

## Process
### Step 1: Exploratory Data Analysis (EDA)
This is where I try to make sense of the dataset. Using different combination of SQL queries such as : **Common table expressions (CTE), subqueries, JOIN, UNION, COUNT and SUM**. The idea is to scrub the data to get meaningful numbers and some statictics like **MEAN and MODE** but also some ***"feelings"*** and hypothesis that can turn into assumptions that are needed before starting to [clean the data](#step-2-cleaning-and-transforming).
It would be pointless to attached any code related to this portion of the process because of the astronomical quantity of code lines used to make sense of the data (I'm still a student without experience). Also, it's not in the scope of the project.

Therefore, my approach was:
**To include in the [cleaning_data.md](cleaning_data.md#cleaning-data) file a section where I present all my [assumptions per table](cleaning_data.md#assumption-assessment-per-table) along with all the QA Risk Assessment Number [(QA. RAN X#)](QA.md) that would need to be mitigate at the end of the process [(QAing)](#step-4-qaing).**

<p align="center">
<img src="images/EDA-TOC.png" alt="EDA-TOC" width="400" >
</p>

### Step 2: Cleaning and transforming
>Please consult the [cleaning_data.md](cleaning_data.md) file for a comprehensive overview of my cleaning process and assumption descriptions. 

Based on my assumptions, I can then use different approach to "prepare" the dataset for my upcoming analysis. Normally, it is not recommended to ALTER directly the raw data as it could change, or get updated (not in the project scope) and also, because you want to be able to crosscheck or [QAing](#step-4-qaing) the result of your cleaning process.

Therefore, my approach was:
**- To only create new tables to ease the creation of the Entity-Relationship Diagram [(ERD)](readme.md#erd);**
**- To create copy of tables when it was possible to create a Primary Key (PK) or composite PK but only after cleaning the table;**
**- Create cleaned VIEWS of the raw data tables in accordance with my assumptions and risk mitigation.**

After cleaning, here is the result of the ERD:

![ERD](images/Schema.png) <a name="erd"></a>

### Step 3: Analysing

> Please consult the [starting_with_questions.md](starting_with_questions.md) file for a complete description of my analysis process including the queries.

Once the data is cleaned, I would normally go directly to [QAing](#step-4-qaing) as the next step is to push the model in the data warehouse (also out of scope of this project). Since the subject is SQL, this is where I need to show my skills with this type of coding for getting insights about the data. 

Therefore, my approach for analysing with PostgreSQL was:
- Create CTE's and VIEWs as much as possible, specific to the questions, but also having in mind the overall project. Meaning, sometimes, the same CTE or View can be re-use for multiple questions.
- Most of my assumptions where about the lack of data for calculating the revenues and the timeframe difference between the two fact table. To have enough data for the analysis, I had to extract them from both table with proper conditions and reassemble them in a meaningfull way.
- Because of missing data and the way I executed my assumption in the analysis, the result may not be comparable from question to question. Meaning, for example, you could have the total of product sold for a country on a question that doesn't match the total of product sold for the same country on another question. It depends on how you need to group the result and this information might not be available for each table.


### Step 4: QAing

> Please consult the [QA.md](QA.md) file for a comprehensive overview of my entire QA process and a description of each risk to be addressed.

This is probably the most important part of the full process, but also, as a student, the more complex one. The primary objective is to ensure the model's robustness before proceeding to deploy it in the data warehouse. Moreover, within the context of this project, it is crucial to validate the accuracy of the data cleaning process to avoid the notorious "garbage in, garbage out" problem. Additionally, we must assess the integrity and consistency of the data, workflow and related elements.

Therefore, my approach to this process was:
- To create a series of TEST (CTE queries) to mitigate each risk.
- Each CTE would calculate a specific attribute that SHOULD NOT happen:
    Not happen, means its OK so `fail_count = 0`, Else `fail_count = 1`.
- A last CTE would UNION ALL previous "checks" with a `check_name` and the SUM of the `fail_count`.
- The final TEST would then evaluate each check from the last CTE and give a PASS or FAIL status (globally or for each check).

## Results
> For full results and number, refer to [starting_with_questions.md](starting_with_questions.md) file.

Here is a summary of the findings in response to the project's questions:

The results revealed the top 5 cities and countries (my assumption by reading the question) with the highest transaction revenues: USA first, Canada second, yeah!!!. However, it's important to remember that this data might not capture the complete revenue picture due to the exclusion of some rows with unknown cities.

To provide a more accurate assessment on average product sold, two separate queries were performed: one that included all rows with an adjusted `productquantity` equal to 1 (more biased) and one that included less rows (less biased).
However, it's crucial to note that this number might be skewed due to the assumption made regarding rows with missing data.

The results revealed various product categories sold in each city and country. Interestingly, **"Men's T-Shirts"** emerged as the top-selling product category across several cities and countries, indicating its popularity among customers. However, within the **"YouTube"** category, a deeper dive demonstrated that it primarily consists of **office-related products**, rather than men's or women's clothing items.

Analyzing the impact of revenue generated per city and country involved creating a revenue_statistics view based on data from the summary_revenues view. The analysis showcased the total revenue and average revenue per city and country. The "Google Men's T-Shirts" product consistently ranked high in revenue across different cities and countries, indicating its significant contribution to overall revenue.

It's essential to recognize that the analysis had limitations, especially in dealing with missing or unknown data, which might have impacted the results. Despite these challenges, the analysis provided valuable insights into transaction revenues, product sales, and revenue distribution across various cities and countries.

## Challenges 

The biggest concern about this projet is not the the clarity of the tasks nor the intention behind the goals, but the dataset itself. I have spent more time (probably 80% of the project) just trying to figure out what the columns in the table meant. Trying to relate tables with each other always ending up with some huge data hole.
Althougth, I have learn a lot about myself and the way I plan my work when confronting this mountain of missing data. For example, I restarted from scratch at least 3 time before I finally read again the question to answer and didn't read them the same way. It's important to understand where you want to go before trying to code and lose your precious time.


## Future Goals

If I knew more about the context of the tables and have the complete set of data, I would have spend more time creating a real workflow where I could have test my model with raw data updates to validate my model and consistency and integrity of data. I would also create a QA test before sending the data to the data warehouse (again fictional).

## Tech stack
+ **[pgAdmin 4](https://www.pgadmin.org/download/)**
+ **[VSCode](https://code.visualstudio.com)**
+ **[GitHub](https://github.com)**
+ GitHub Destop
---
