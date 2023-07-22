#                                   CLEANING DATA
*by Christian Leclerc b.ing, Data Analyst, [LHL Bootcamp](https://www.lighthouselabs.ca)*

---
***Notes:***
- This file is better viewed using [GitHub](https://github.com) preview screen (or any Markdown viewer) to be able to use the relative links.
- Refer to Risk Assessment Number (RAN X#) in [QA.md file](QA.md) for explanations and mitigation of each risk.
- Almost all column names were imported from raw data .csv files with new names without camelCase to be able to:
    +   differentiate time and date with datatype **TIME** and **DATE**
    +   to remove the usage of double quote ("") in SQL queries (when using **[pgAdmin 4](https://www.pgadmin.org/download/)**)

***Common Issues for most tables***
1.  Columns representing money amount are multiple of 1 million => 100,000,000 = 100.
    +   Need to be divided by 1,000,000 to show proper unit (assuming USD dollars)
2.  Columns with the product name have leading space in most of the values.
    +   Need to remove leading space.
3.  Column with time data (except datatype **DATE**) are stored in UNIX Epoch.
    +   Need to be converted in **TIMESTAMP** for readability and future queries.

---
## TABLE OF CONTENTS
+ [ASSUMPTION ASSESSMENT PER TABLE](cleaning_data.md#assumptions-assessment-per-table)
	+ [The `all_sessions` table](cleaning_data.md#a-the-all_sessions-table)
		<details>
  			<summary>Content</summary>

		+ [FINDINGS](cleaning_data.md#findings)
		+ [MISSING DATA](cleaning_data.md#missing-data-total-rows--15134)
		+ [DUPLICATES AND IRRELEVANT DATA](cleaning_data.md#duplicates-and-irrelevant-data)
		+ [OUTLIERS](cleaning_data.md#outliers)
		+ [STRATEGY FOR CLEANING](cleaning_data.md#strategy-for-cleaning)

		</details>
	+ [The `analytics` table](cleaning_data.md#b-the-analytics-table)
		<details>
  			<summary>Content</summary>

		+ FINDINGS
		+ MISSING DATA
		+ DUPLICATES AND IRRELEVANT DATA
		+ OUTLIERS
		+ STRATEGY FOR CLEANING

		</details>
	+ [The `productinfo` table](cleaning_data.md#c-the-productinfo-table)
		<details>
  			<summary>Content</summary>

		+ FINDINGS
		+ MISSING DATA
		+ DUPLICATES AND IRRELEVANT DATA
		+ OUTLIERS
		+ STRATEGY FOR CLEANING

		</details>
	+ [The `sales_report` table](cleaning_data.md#d-the-sales_report-table)
		<details>
  			<summary>Content</summary>

		+ FINDINGS
		+ MISSING DATA
		+ STRATEGY FOR CLEANING

		</details>
	+ [The `sales_by_sku` table](cleaning_data.md#e-the-sales_by_sku-table)
		<details>
  			<summary>Content</summary>

		+ FINDINGS
		+ MISSING DATA
		+ STRATEGY FOR CLEANING

		</details>
+ [NEW TABLES](cleaning_data.md#new-tables)
	+ [The `visitor_details` TABLE](cleaning_data.md#the-visitor_details-table)
	+ [The `visitors` TABLE](cleaning_data.md#the-visitors-table)
+ [QUERIES](cleaning_data.md#queries)
	+ [`clean_all_sessions` TABLE](cleaning_data.md#clean_all_sessions-table)
	+ [`clean_analytics` VIEW](cleaning_data.md#clean_analytics-view)
	+ [`clean_productinfo` VIEW](cleaning_data.md#clean_productinfo-view)

---
## ASSUMPTION ASSESSMENT PER TABLE <a name="assumptions assessment-per-table"></a>

### A. The `all_sessions` table <a name="a-the-all_sessions-table"></a>

>**Type of data**: Facts table (transactionals info and some visitors info)
**Timeframe**: 1 year from '2016-08-01' to '2017-08-01'

#### FINDINGS
		
There is 2 "stages" in `all_sessions` table:

**Stage 1**: Each row (session) should represent a transaction of one single product sold, uniquely identified has:
	
composite PK (`visit_id, fullvisitor_id, product_sku`)
					
Multiple instance of the same session (`visit_id, fullvisitor_id`) with different `product_sku` can exist (visitor buying more than one product per session).

**Stage 2**: Starting 2017-05-01, `analytics` table was created and a "bug" occured (assumption) while transfering or extracting the data from one table to the other.
Therefore, some single rows with one product actually represent the total revenue `(totaltransactionrevenue) = SUM(revenue)` from `analytics` table, but identified has a single part.
	
#### MISSING DATA (Total rows = 15134)
		
Stage 1 and Stage 2 sessions have multiple missing data:
- `productquantity` (only 53 NOT NULL)
- `country` (No NULL but 24 (not set))
- `city` (half of data missing)
- `totaltransactionrevenue` (only 81 rows NOT NULL)
- `productrevenue` (only 4 rows not NULL)

Stage 2 sessions (one row per multiple products) have multiple missing rows to be able to properly identify which product was sold (and how many).
Althougth we have the `totaltransactionrevenue` that match `analytics = SUM(revenue)`. *(validated)*

a.	We can replace some missing values in `city` IF only the info EXISTS for the same visitor (`fullvisitor_id`).
>			
>**Action**:	[(RAN A1)](qa.md#ran-a1)
>+ Replace all missing `city` values with existing value for same `fullvisitor_id` IF EXISTS
ELSE keep it NULL.
><br>
			
b.	Stage 1 sessions should have at least a `productquantity` of 1 instead of NULL to be able to calculate a minimal revenue from `productprice`.
Data cannot all be found in `analytics` due to timeframe difference.
>			
>**Action**:	[(RAN A2)](qa.md#ran-a2)
>+ Replace all `productquantity` NULL with 1.
><br>

#### DUPLICATES AND IRRELEVANT DATA
	
a.	Based on previous assumption, using triple composite PK (`visit_id, fullvisitor_id, product_sku`) we find that only 5 rows are duplicates.
>		
>**Action**:	[(RAN A3)](qa.md#ran-a2)
>+ Remove them to consolidate PK.
><br>

b.	Multiple rows have `productprice`= 0. Not usable to calculate revenue.

After analysis, all `sku_id` from `productinfo` table that matched `product_sku` of `all_sessions` table have an `orderedquantity` = 0. But not all product can be found. Meaning, it make sense that products with a `productprice` of 0 where not added to the dimension table `productinfo`. Probably cancelled order (or errors).

*Bonus assumption:* by looking at the product SKU with `productprice` = 0, it seems it is all the product starting with a number instead of G... numbers (Ex.: 9180766). It really looks like a "back order" problem. People where trying to buy a part that is out-of-stock. So they had to redo a new transaction to buy an alternative.
	
>**Action**: [(RAN A4)](qa.md#ran-a2)
>+ Remove all rows with `productprice` = 0.
><br>

#### OUTLIERS
	
#### STRATEGY FOR CLEANING
	
**Strategy**: 

Since we determined that we want a triple composite Primary Key (PK) made of `visit_id`, `fullvisitor_id` and `product_sku`, we cannot add that constraint directly in the raw data because of the presence of the duplicates.

For this small project, we could avoid using PK and just create a single VIEW with all the cleaning queries, but for the sake of having a nice [ERD](README.md#erd) and for educational perpective, we will create a duplicate of the all_sessions table and ALTER the data of that table with all the cleaning queries. 

Althought, it's not the best practice because, in real world,  we would need a FUNCTION to update this table each time the raw data is updated (out of scope of this project).

> **Action**: Create a copy of all_sessions called [clean_all_sessions](cleaning_data.md#clean-all-sessions-table) that includes all previous actions, and common issues, using UPDATE or ALTER command to change the data in the table.
Then add the constraint to get the triple composite PK.
---

### B. The `analytics` table <a name="b-the-analytics-table"></a>

>**Type of data**: Facts table (visitors info with transactions info)
**Timeframe**: 3 month (or a quarter) from '2017-05-01' to '2017-08-01'

#### FINDINGS

#### MISSING DATA (Total rows = 15134)

#### DUPLICATES AND IRRELEVANT DATA

#### OUTLIERS
	
#### STRATEGY FOR CLEANING

---

### C. The `productinfo` table <a name="c-the-productinfo-table"></a>

>**Type of data**: Facts table (visitors info with transactions info)
**Timeframe**: 3 month (or a quarter) from '2017-05-01' to '2017-08-01'

#### FINDINGS

#### MISSING DATA (Total rows = 15134)

#### DUPLICATES AND IRRELEVANT DATA

#### OUTLIERS
	
#### STRATEGY FOR CLEANING

---

### D. The `sales_report` table <a name="d-the-sales_report-table"></a>

>**Type of data**: Facts table (visitors info with transactions info)
**Timeframe**: 3 month (or a quarter) from '2017-05-01' to '2017-08-01'

#### FINDINGS

#### MISSING DATA (Total rows = 15134)

#### DUPLICATES AND IRRELEVANT DATA

#### OUTLIERS
	
#### STRATEGY FOR CLEANING

---

### E. The `sales_by_sku` table <a name="e-the-sales_by_sku-table"></a>

>**Type of data**: Facts table (visitors info with transactions info)
**Timeframe**: 3 month (or a quarter) from '2017-05-01' to '2017-08-01'

#### FINDINGS

#### MISSING DATA (Total rows = 15134)

#### DUPLICATES AND IRRELEVANT DATA

#### OUTLIERS
	
#### STRATEGY FOR CLEANING

---

## NEW TABLES <a name="new-tables"></a>

### The `visitor_details` table <a name="the-visitor_details-table"></a>

While trying to join `all_sessions` with `analytics` tables it became very obvious that a new table for visitors data was needed as there is a MANY-to-MANY relationship between the two tables: 
>Multiple visitors can have multiple sessions (in `all_sessions`)
AND 
same visitors session can have multiple product bought (in `analytics`)

For that purpose (and for a cleaner [ERD](README.md#erd)), the `visitor_details` table was created with only the data needed for our analysis. Thought, during EDA (Exploratory Data Analysis), we found multiple visitors with different country. Plausible, as a visitor can move from time-to-time. Therefore, my approach was to use a composite Key made of `fullvisitor_id` and `visit_date`. Together, they represent a unique identifier of each visit of each customer. Adding the `country` and `city` in the table, we have all the info needed for the analysis.

Pros:

+ Preserves historical data and allows to keep track of changes over time.
+ Provides a clear audit trail for visitors changes.
+ Simplifies data integrity, as each visitor record remains unchanged.

Cons:

+ Increases (or in this case, maintain) data duplication, as each visitor may have multiple records with different countries.
+ Requires additional logic to handle queries related to visitors, as we need to consider multiple records for a single visitor.

Note: Of course, the table is missing data for `city` as the data was not available.

### The `visitors` table <a name="the-visitors-table"></a>

To complete the "clean" [ERD](README.md#erd) and have a relation between the `clean_all_sessions`, the `visitor_details` and the `analytics` tables,  a new table is needed to represent each visitor with `fullvisitor_id` as the PK. Additional informations of the visitor can then be append to it such as (but not limited to):
+ Total number of visits
+ Total revenue
+ Total units bought
+ Total time spent on site
+ etc.

Note: Only the `fullvisitor_id` was populated for this project.

---
## QUERIES <a name="queries"></a>

### `clean_all_sessions` TABLE <a name="clean_all_sessions-table"></a>

>Refer to Risk Assessment Number (RAN X#) in [QA.md file](QA.md) for explanations and mitigation of each risk.
```sql
---------------- CLEANING DATA ----------------
----- Cleaning all_sessions table -----

---- Create a copy of all_sessions with cleaned data

--- Removing irrelevant data
-- (RAN A3) - Removing duplicates
-- (RAN A4) - Removing rows with productprice = 0

CREATE TABLE clean_all_sessions AS
	-- Using LAG to iterate each session identified with unique PK(visit_id, fullvisitor_id, product_sku)
	-- to find where product_SKU repeats and tag a new column prev_val with the previous value of product_sku (to be deleted)
	SELECT *
	FROM ( SELECT *, -- RAN A3 (part 1)
		LAG(product_sku) OVER (PARTITION BY visit_id, fullvisitor_id, product_sku ORDER BY visit_id) AS prev_val
		FROM all_sessions
		) AS irrelevant_data
	WHERE
		prev_val IS NULL
	-- Delete also rows WHERE productprice = 0
		AND productprice != 0 	-- RAN A4
; -- expected 14753 rows

-- DROP prev_val column
ALTER TABLE clean_all_sessions
DROP COLUMN prev_val
;
---- Convert datatypes and clean missing data

--- Clean missing data
-- (RAN A1) - Replace missing value of city from visitor_details table, if possible
UPDATE clean_all_sessions
SET city = visitor_details.city
FROM visitor_details
WHERE 
	clean_all_sessions.fullvisitor_id = visitor_details.fullvisitor_id
  	AND clean_all_sessions.visit_date = visitor_details.last_visit
;
-- (RAN A2) - Replace all productquantity with value = 1
UPDATE clean_all_sessions
SET productquantity =
	CASE
		WHEN productquantity IS NULL THEN 1
		ELSE productquantity
	END
;
--- Common issues: Convert datatype
ALTER TABLE clean_all_sessions
-- Convert Unix Epoch to datatype TIMESTAMP (easier to read)
ALTER COLUMN
	visit_id SET DATA TYPE TIMESTAMP USING TO_TIMESTAMP(visit_id),
-- Convert INT to NUMERIC, divided by 1,000,000.
ALTER COLUMN
	productprice SET DATA TYPE NUMERIC USING (productprice::NUMERIC / 1000000)
;
-- Then round productprice to 2 decimals
UPDATE clean_all_sessions
SET productprice = ROUND(productprice, 2)
;
--- Add the PK constraint
ALTER TABLE  clean_all_sessions
ADD CONSTRAINT 
	pk_clean_all_sessions
	PRIMARY KEY (visit_id, fullvisitor_id, product_sku)  -- RAN A3 (part 2)
; 
```

### `clean_analytics` VIEW <a name="clean-analytics-view"></a>



### `clean_productinfo` VIEW <a name="clean-productinfo-view"></a>


