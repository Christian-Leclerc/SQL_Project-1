#                                   CLEANING DATA
*by Christian Leclerc b.ing, Data Analyst, [LHL Bootcamp](https://www.lighthouselabs.ca)*

---
***Notes:***
- This file is better viewed using [GitHub](https://github.com) preview screen (or any Markdown viewer) to be able to use the relative links.
- Refer to *Risk Assessment Number* (**QA. RAN X#**) in [qa.md file](qa.md) for explanations and mitigation of each risk.
- Almost all column names were imported from raw data .csv files with new names without camelCase to be able to:
    +   differentiate `time and date` column with datatype **TIME** and **DATE**
    +   to remove the usage of double quote ("") in PSQL queries (when using **[pgAdmin 4](https://www.pgadmin.org/download/)**)

***Common Issues for most tables***
1.  Columns representing money amount are multiple of 1 million => 100,000,000 = 100.
    +   Need to be divided by 1,000,000 to show proper unit (assuming USD dollars)
2.  Columns with the product name have leading space in most of the values.
    +   Need to remove leading space.
3.  Column with time data (except datatype **DATE**) are stored in UNIX Epoch.
    +   Need to be converted in **TIMESTAMP** for readability and future queries.

---
## TABLE OF CONTENTS
+ [ASSUMPTION ASSESSMENT PER TABLE](cleaning_data.md#assumption-assessment-per-table)
	+ [The `all_sessions` table](cleaning_data.md#a-the-all_sessions-table)
		<details>
  			<summary>Content</summary>

		+ [FINDINGS](cleaning_data.md#findings)
		+ [MISSING DATA](cleaning_data.md#missing-data-total-rows--15134)
		+ [DUPLICATES AND IRRELEVANT DATA](cleaning_data.md#duplicates-and-irrelevant-data)
		+ [STRATEGY FOR CLEANING](cleaning_data.md#strategy-for-cleaning)

		</details>
	+ [The `analytics` table](cleaning_data.md#b-the-analytics-table)
		<details>
  			<summary>Content</summary>

		+ [FINDINGS](cleaning_data.md#findings-1)
		+ [MISSING DATA](cleaning_data.md#missing-data-total-rows--4301122)
		+ [DUPLICATES AND IRRELEVANT DATA](cleaning_data.md#duplicates-and-irrelevant-data-1)
		+ [STRATEGY FOR CLEANING](cleaning_data.md#strategy-for-cleaning-1)

		</details>
	+ [The `productinfo` table](cleaning_data.md#c-the-productinfo-table)
		<details>
  			<summary>Content</summary>

		+ [FINDINGS](cleaning_data.md#findings-2)
		+ [MISSING DATA](cleaning_data.md#missing-data-total-rows--1092)
		+ [DUPLICATES AND IRRELEVANT DATA](cleaning_data.md#duplicates-and-irrelevant-data-2)
		+ [STRATEGY FOR CLEANING](cleaning_data.md#strategy-for-cleaning-2)

		</details>
	+ [The `sales_report` table](cleaning_data.md#d-the-sales_report-table)
		<details>
  			<summary>Content</summary>

		+ [FINDINGS](cleaning_data.md#findings-3)
		+ [MISSING DATA](cleaning_data.md#missing-data-total-rows--454)
		+ [DUPLICATES AND IRRELEVANT DATA](cleaning_data.md#duplicates-and-irrelevant-data-3)

		</details>
	+ [The `sales_by_sku` table](cleaning_data.md#e-the-sales_by_sku-table)
		<details>
  			<summary>Content</summary>

		+ [FINDINGS](cleaning_data.md#findings-4)

		</details>
+ [NEW TABLES](cleaning_data.md#new-tables)
	+ [The `visitor_details` table](cleaning_data.md#the-visitor_details-table)
	+ [The `visitors` table](cleaning_data.md#the-visitors-table)
+ [QUERIES](cleaning_data.md#queries)
	+ [`clean_all_sessions` table](cleaning_data.md#clean_all_sessions-table)
	+ [`clean_analytics` view](cleaning_data.md#clean_analytics-view)
	+ [`clean_productinfo` view](cleaning_data.md#clean_productinfo-view)

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
>**Action**:	[(QA. RAN A1)](qa.md)
>+ Replace all missing `city` values with existing value for same `fullvisitor_id` IF EXISTS
ELSE keep it NULL.
><br>
			
b.	Stage 1 sessions should have at least a `productquantity` of 1 instead of NULL to be able to calculate a minimal revenue from `productprice`.
Data cannot all be found in `analytics` due to timeframe difference.
>			
>**Action**:	[(QA. RAN A2)](qa.md)
>+ Replace all `productquantity` NULL with 1.
><br>

#### DUPLICATES AND IRRELEVANT DATA
	
a.	Based on previous assumption, using triple composite PK (`visit_id, fullvisitor_id, product_sku`) we find that only 5 rows are duplicates.
>		
>**Action**:	[(QA. RAN A3)](qa.md)
>+ Remove the duplicates to consolidate PK.
><br>

b.	Multiple rows have `productprice`= 0. Not usable to calculate revenue.

After analysis, all `sku_id` from `productinfo` table that matched `product_sku` of `all_sessions` table with a `productprice` = 0 have an `orderedquantity` = 0. But not all product can be found. Meaning, it make sense that products with a `productprice` of 0 where not added to the dimension table `productinfo`. Probably cancelled order (or errors).

c. The `v2productcategory` column should be representing the **type of product** sold on the site. It looks more like the *URL path* of the page the visitor landed on:

Ex.:
 ![Source: images/v2productcategory](images/v2productcategory.png)
**Source: [images/v2productcategory](images/v2productcategory.png)**

Since it's the only source of information, we will have to deal with it. Althought, it is also very *messy*:
- Same `product_sku` can have multiple different category names.
- Most categories starts with \'Home/\', but than some categories only have a portion of the "path".
- There is some "(not set)" category name and some "${escCatTitle}" name&nbsp;**\***.<br>
>**\*** The text "${escCatTitle}" likely represents a **placeholder** for the category title of a product in the website's database. When the website renders product information, this placeholder would be replaced with the actual category title of the product, ensuring that any special characters in the title are properly encoded or escaped for safe display on the website.
Source: [chatGPT](https://chat.openai.com)
Althought, something went wrong during the extract and we lost the information.

The need here is only to evaluate WHAT kind of product the visitor are buying for each country/city. The category name itself seems self-explanatory but to decrease the number of categories, instead of doing a full cleaning, we will TRUNCATE the name to only keep the last ***words*** after the last "/" if exists.
This way, some categories with one word will now match with the truncated categories.

>**Action**: [(QA. RAN A4)](qa.md)
>+ TRUNCATE v2productcategory values to only keep the words before the last "/" , if exits, THEN remove all remaining "/".
>+ Replace "${escCatTitle}" category with "(not set)".
>+ Bonus action: After previous actions completed, if possible, we could try replacing the v2productcategory "(not set)" with a category name that exist in the v2productname.
><br>

*Bonus assumption:* by looking at the product SKU with `productprice` = 0, it seems it is all the product starting with a number instead of G... numbers (Ex.: 9180766). It really looks like a "back order" problem. People where trying to buy a part that is out-of-stock. So they had to redo a new transaction to buy an alternative.
	
>**Action**: [(QA. RAN A5)](qa.md)
>+ Remove all rows with `productprice` = 0.
><br>
	
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

The `analytics` table contains the transactional information about each "action" taken by the visitor during a session:
- Each action (row) represent 1 "product" purchased or not (assumption: each time it is added to the cart, maybe). Thought, no information about the actual product, only the unit price.
- Each action contains (and repeats) some information about the visitor, the site behavior and end result of the purchase (or not), such as:
+ `visitnumber`: the number of visit this particular visitor has of all time on the site.
+ `visit_id` or `visitstarttime`: both represent the start time of the session and can be use to identify it.
+ `fullvisitor_id`: a number, probably generated with the visitor information, unique to the visitor, but repeated each row of the session.

None of the columns have unique values, therefore we can't use any one nor any combinations (composite) to create a Primary Key. Only the `fullvisitor_id` can be a Foreign Key of the **[visitors](cleaning_data.md#the-visitors-table)** table.

If we were to assume a pipeline for this business, we would say that the `analytics`table "feeds" the `all_sessions` table by providing only the information about each purchase that actually has a revenue for each product within each session. 
Apparently, something went wrong and either, the transfer didn't work properly (missing rows in `all_sessions`) or the business tried to change the pipeline behavior to have only one row in `all_sessions` that represent the total revenue of the session. Thought, we loose the information about the actual products that were bougth...

#### MISSING DATA (Total rows = 4,301,122)

To be able to answer any questions about revenue and quantity of units, we need to validate if we have enough data available. The most important columns are:
- `unitssold` (95,147 rows with a SUM over 400K units!).
- `revenue` (only 15,355 row with data out of 4 millions!).
- `unitprice` (No NULL, but 188,314 rows where unitprice = 0).

a.	It does not make sense to have 400k units sold in 3 month while the overall `productinfo` table only has 156K products ordered (which should represent the total ordered of product from `all_sessions` table (1 year)). After validation, every row with a `revenue` has a `unitssold`. Therefore:
>			
>**Action**:	[(QA. RAN B1)](qa.md)
>+ Remove all rows WITH `unitssold` WITHOUT `revenue`.
><br>

b.	If there is missing `unitssold` (deeper analysis) there is no point keeping the rows with `unitprice` = 0. If we do found a quantity, there is no way to figure out what product was sold, therefore:
>			
>**Action**:
>+ Remove all rows WITH `unitprice` = 0.
Note: after analysis, by doing previous action (keep only rows with revenue), no more rows with unitprice = 0 exists. No further action required.
><br>

#### DUPLICATES AND IRRELEVANT DATA

Even thought a lot of rows seems duplicates, they are actually a real selection from the visitor (we just don't see which product it is). After removing the data from previous actions, their should be no reason to have duplicates. If there is (for example, two products with same quantity and unitprice), we can't tell if its one or two products. Therefore, we won't proceed with a duplicate cleaning process.
	
#### STRATEGY FOR CLEANING

Since we can't have any PK, there is no reason to recreate a clean copy of the table like the `all_sessions`. We will only create a clean VIEW of the table. 
Also, in the same VIEW creation, we could apply any queries necessary to obtain the "missing" informations about the **country, city and category** needed for this analysis. To be evaluated.

>**Action:** Create a VIEW called [clean_analytics](cleaning_data.md#clean_analytics-view) which will include all actions and common issues. 
[(QA.RAN B2)](qa.md) Add any queries necessary to be able to answer the analysis questions.
><br>

---

### C. The `productinfo` table <a name="c-the-productinfo-table"></a>

>**Type of data**: Dimension table (product related informations)
**Timeframe**: None (unknown) - Assumption: Overall sold product (1 year).

#### FINDINGS

The `productinfo` table contains any relevant informations about a unique product: 

(Quantitative data)
- SKU number (Stock Keeping Unit)
- Product name
- Total ordered
- Stock level (how many product are left in stock)
- Leading time (Assumption: in weeks, to get a new PO (Purchase Order))

(Qualitative data transformed in quantitative data by the business (unknown rules))
- Sentiment score 
- Sentiment magnitude

The `sku_id` is unique and was already set to PK at creation.

#### MISSING DATA (Total rows = 1092)

During EDA, we found that out of 536 listed `product_sku` in `all_sessions`, only 389 actually EXISTS in the `productinfo`.
It is not necessarily and error, because the `productinfo` table could be in "pending" UPDATE from the other table (a monthly or quarterly plan to update the dimension tables from the business).
As for the `analytics` table, we can only assume that the number of distinct product sold is smaller than the toal from `all_sessions` due to the timeframe difference.
	
>**No action**:

#### DUPLICATES AND IRRELEVANT DATA

There is no duplicates in the table (having `sku_id` as PK).
Also, every product with `stocklevel` = 0 have an `orderedquanity` = 0. Business wise, it would be interessting to deep dive this table to extract some information about the business

>**Action**: Good potential table for data analysis questions related to stock management.
><br>
	
#### STRATEGY FOR CLEANING

Only the common issues can be fixed for the need of our analysis.

>**Action**: Create a VIEW called [clean_productinfo](cleaning_data.md#clean_productinfo-view) with the common issue fixed.
><br>


---

### D. The `sales_report` table <a name="d-the-sales_report-table"></a>

>**Type of data**: Report table (report from `productinfo`)
**Timeframe**: None (unknown).

#### FINDINGS

The content of `sales_report` is exactly the same as `productinfo` but with a smaller amount of product ordered. It suggest that it is a capture, in time, to represent the total ordered of product for that timeframe to be compared with the `stocklevel`. This information is added in a new column:
+ `ratio` which is the ratio of `total_ordered` / `stocklevel`.

It's a number, when greater than 1, indicates a problem: "Warning ! More part ordered than quantity in stock!".

#### MISSING DATA (Total rows = 454)

Less product than `productinfo`, normal since its a capture in time.

No reason to believe there is missing data in the report table.
		
>**No action**:

#### DUPLICATES AND IRRELEVANT DATA

The `stocklevel` of `sales_report` has exactly the same quantity then `productinfo.stocklevel`. Which prooves it is a capture of the quantity ordered with the actual stocklevel.
	
>**Action**: No need to clean a report table for our analysis purpose.
><br>

---

### E. The `sales_by_sku` table <a name="e-the-sales_by_sku-table"></a>

>**Type of data**: Report table (exactly the same as sales_report)
**Timeframe**: None (unknown)

#### FINDINGS

This report table is exactly the same as `sales_report` except it contains only 2 columns: `product_sku` and `total_ordered`. Not sure why this report is needed, it depends on the business needs for reporting and visualization (maybe a KPI).

No further analysis required.
>**No action**:	

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

>Refer to Risk Assessment Number (QA. RAN X#) in [qa.md file](QA.md) for explanations and mitigation of each risk.

### `clean_all_sessions` TABLE <a name="clean_all_sessions-table"></a>

```sql

---------------- CLEANING DATA ----------------
----- Cleaning all_sessions table -----

---- Create a copy of all_sessions with cleaned data

--- Removing irrelevant data
-- (QA. RAN A3) - Removing duplicates
-- (QA. RAN A4) - Removing rows with productprice = 0

/*
	Using LAG to iterate each session identified with unique PK(visit_id, fullvisitor_id, product_sku)
to find where product_SKU repeats and tag a new column prev_val with the previous value of product_sku (to be deleted).
*/
CREATE TABLE clean_all_sessions AS
	SELECT *
	FROM ( 
		SELECT *, -- QA. RAN A3 (part 1)
			LAG(product_sku) OVER (PARTITION BY visit_id, fullvisitor_id, product_sku ORDER BY visit_id) AS prev_val
		FROM 
			all_sessions
		) AS irrelevant_data
	WHERE
		prev_val IS NULL
	-- Delete also rows WHERE productprice = 0
		AND productprice != 0 	-- QA. RAN A4
;

-- DROP prev_val column --
ALTER TABLE clean_all_sessions
DROP COLUMN prev_val
;

---- Convert datatypes and clean missing data ----

--- Clean missing data ---

-- (QA. RAN A1) - Replace missing value of city from visitor_details table, if possible --

UPDATE clean_all_sessions AS cs
SET city = vd.city
FROM visitor_details AS vd
WHERE 
	cs.fullvisitor_id = vd.fullvisitor_id
  	AND cs.visit_date = vd.last_visit
	AND cs.city IN ('(not set)', 'not available in demo dataset')
;

-- (QA. RAN A2) - Replace all productquantity with value = 1 --

UPDATE clean_all_sessions
SET productquantity =
	CASE
		WHEN productquantity IS NULL THEN 1
		ELSE productquantity
	END
;

--- (QA. RAN 4) - Category multiple issues ---
/*
- Try to update category values of (not set) or ${escCatTitle} where a similar v2productname exists,
- then truncate v2productcategory to keep only words before last "/" and remove remaining "/".
- then try to update remaining "unset" value with a new category name where it exist in the productname.
*/

WITH get_mode_categories AS ( -- Create a CTE to extract the most frequent categories per productname
  	SELECT
    	t1.v2productname,
    	t1.v2productcategory,
		-- rank each productname where rank = 1 means the MODE
    	RANK() OVER (PARTITION BY t1.v2productname ORDER BY COUNT(*) DESC) AS rank
  	FROM clean_all_sessions AS t1
  	JOIN clean_all_sessions AS t2
    	ON t1.v2productname LIKE '%' || t2.v2productname || '%' -- to screen the whole column
  	WHERE 
		t1.v2productcategory NOT IN ('(not set)', '${escCatTitle}') 
    	AND t2.v2productcategory NOT IN ('(not set)', '${escCatTitle}') -- to make sure source is not empty
  	GROUP BY 
		t1.v2productname, t1.v2productcategory
)
UPDATE clean_all_sessions AS t
SET v2productcategory = gmc.v2productcategory
FROM 
	get_mode_categories AS gmc
WHERE 
	t.v2productname = gmc.v2productname
  	AND t.v2productcategory IN ('(not set)', '${escCatTitle}') -- to make sure we only update the missing ones
;

-- truncate v2productcategory to keep only words before last "/" and remove remaining "/".
UPDATE clean_all_sessions
SET v2productcategory = REGEXP_REPLACE(v2productcategory, '^.*/([^/]+)/?$', '\1')
;-- 54 new distinct categories instead of 73 :)

-- Update (not set) values with a category name that is found in the productname --

UPDATE clean_all_sessions AS t1
SET v2productcategory = 
    (
		SELECT 
			t2.v2productcategory
    	FROM 
			clean_all_sessions AS t2
    	WHERE 
			t2.v2productcategory <> '(not set)'
      		AND t1.v2productname LIKE '%' || t2.v2productcategory || '%' -- to screen the whole column
	 	LIMIT 1 -- anyone will do ! :)
	)
WHERE v2productcategory = '(not set)';
;

--- Common issues: Convert datatype ---

ALTER TABLE clean_all_sessions
-- Convert Unix Epoch to datatype TIMESTAMP (easier to read)
ALTER COLUMN
	visit_id SET DATA TYPE TIMESTAMP USING TO_TIMESTAMP(visit_id),
-- Convert INT to NUMERIC, divided by 1,000,000.
ALTER COLUMN
	productprice SET DATA TYPE NUMERIC USING (productprice::NUMERIC / 1000000)
ALTER COLUMN
	totaltransactionrevenue SET DATA TYPE NUMERIC USING (totaltransactionrevenue::NUMERIC / 1000000)
;

-- Then round productprice to 2 decimals --

UPDATE clean_all_sessions
SET 
	productprice = ROUND(productprice, 2)
	totaltransactionrevenue = ROUND(totaltransactionrevenue, 2)
;

--- Add the PK constraint ---

ALTER TABLE  clean_all_sessions
ADD CONSTRAINT 
	pk_clean_all_sessions
	PRIMARY KEY (visit_id, fullvisitor_id, product_sku)  -- QA. RAN A3 (part 2)
; 
```

### `clean_analytics` VIEW <a name="clean-analytics-view"></a>

```sql
--------------------------------------------
----- Cleaning analytics table -----

---- Create a VIEW of analytics with clean data and only necessary info

-- (QA. RAN B1) - Remove rows without revenue
-- Clean common issues

DROP VIEW IF EXISTS clean_analytics
;
CREATE OR REPLACE VIEW clean_analytics AS

	SELECT
		fullvisitor_id,
		visit_date,
		unitssold,
		TO_TIMESTAMP(visit_id) AS visit_id,
		ROUND((revenue::NUMERIC / 1000000), 2) AS revenue,
		ROUND((unitprice::NUMERIC / 1000000), 2) AS unitprice
	FROM 
		analytics
	WHERE 
		revenue IS NOT NULL -- (QA. RAN B1)
;
```

### `clean_productinfo` VIEW <a name="clean-productinfo-view"></a>

```sql
------------------------------------
----- Cleaning productinfo table -----

--- Clean common issues

DROP VIEW IF EXISTS clean_productinfo
;
CREATE VIEW clean_productinfo AS 
-- Remove leading space from productname
	SELECT
		sku_id, 
		TRIM(productname), -- Remove leading "spaces"
		orderedquantity, stocklevel, restockingleadtime,
		sentimentscore, sentimentmagnitude
	FROM
		productinfo
```

