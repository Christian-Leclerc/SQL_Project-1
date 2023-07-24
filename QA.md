#                                   QA PROCESS
*by Christian Leclerc b.ing, Data Analyst, [LHL Bootcamp](https://www.lighthouselabs.ca)*

---

##**RISK ASSESSMENT NUMBERS** <a name="qa"></a>

###**QA. RAN A - `clean_all_sessions` TABLE**

***Notes:*** By doing any of the following actions in the [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) table, we made sure not to alter the original file (raw data).

#### QA. RAN A1 <a name="qa.-ran-a1"></a>

**Description of risk:**
>**Action:**
>
>- Replace all missing city values with existing value for same fullvisitor_id IF EXISTS
ELSE keep it NULL.

We could have change the relation from a `fullvisitor_id` and it's recorded `city`. Hence, modify the result when filtering data per city.

***Mitigation:***

- By creating a new `visitor_details` table to record all visitors info (including country, city, last visit), we make sure that the city is connected to a PK, or composite PK here, and can be remove from the original table ultimately.
- We can crosscheck the relation between fullvisitor_id, visit_date, country and city from the original table to see if we have the same relation in the other table.

#### QA. RAN A2 <a name="qa.-ran-a2"></a>

**Description of risk:**
>**Action:**
>
>Replace all `productquantity` NULL with 1.

The only risk here is to have accidentaly change a non NULL value with 1, hence loosing important data. This is more an assumption that each row should have at least a quantity of 1 for having been added to the table originally (Assumption that only sessions with revenues should be in this table).

***Mitigation:***

- We can crosscheck the original value of each session with the new table data.

#### QA. RAN A3 <a name="qa.-ran-a3"></a>

**Description of risk:**
>**Action:**
>+ Remove the duplicates to consolidate PK.

+ We could have removed to much rows during the process;
OR 
+ We could have identified these rows wrongly.

***Mitigation:***

- Validate that the number of rows with composite PK(visit_id, fullvisitor_id, product_sku) is smaller than the original based on the assumption that each row tagged to a `product_sku` cannot have a the same information for these columns.

#### QA. RAN A4 <a name="qa.-ran-a4"></a>

**Description of risk:**
>**Action:**
>+ TRUNCATE v2productcategory values to only keep the words before the last "/" , if exits, THEN remove all remaining "/".
>+ Replace "${escCatTitle}" category with "(not set)".
>+ Bonus action: After previous actions completed, if possible, we could try replacing the v2productcategory "(not set)" with a category name that exist in the v2productname.

Working with REGEX can create disaster when altering table. The goal here was to reduce the number of category AND replace the missing ones.

***Mitigation:***

- Validate that the process indeed reduce the number of categories.
- Crosscheck the data with the original table to make sure no loss of information (or wrong alteration).

#### QA. RAN A5 <a name="qa.-ran-a5"></a>

**Description of risk:**
>**Action:**
>+ Remove all rows with `productprice` = 0.

The risk here is more an assumption issue. We assume that these are errors, but if not, we are loosing a lot of information.

***Mitigation:***

- Crosscheck the quantity of part from original tables compare to the `productinfo` total quantity. Not to compare similar quantity, but to see the impact of these rows if we would have considered them not errors (with at least a quantity of 1).


###**QA. RAN B - `clean_analytics` TABLE**

#### QA. RAN B1 <a name="qa.-ran-b1"></a>

**Description of risk:**
>**Action:**
>+ Remove all rows WITH `unitssold` WITHOUT `revenue`.

This is also more an assumption issue. We assume by ignoring these units sold, the data make more sense concerning the quantity of part sold in the timeframe of the table (3 month) compare to the total part from `productinfo` (assuming 1 year).

***Mitigation:***

- Validate that the `clean_analytics` view contains the right information after the split.

#### QA. RAN B2 <a name="qa.-ran-b2"></a>

**Description of risk:**
>**Action:** 
Add any queries in the `clean_analytics` view necessary to be able to answer the analysis questions.

Duplicating informations from table to table is not a best practice. The goal is more to create relationship that can provide the information when joining them throught PK, FK, etc.

***Mitigation:***

- Do not add the country, city and category informations in the `clean_analytics` view. Use the new table `visitor_details` FK(fullvisitor_id) to join the table if needed. 

---
##**QUERIES AND PROCESS** <a name="queries-and-process"></a>

**Description of process:**

My approach to this task is only to create a series of test to answer each question in my TO DO List and report them all on a single query that would say if the test PASS or FAIL.

If I had more time, I would like to create a series of test to prepare future update of the raw data and validate that the content of my new tables and views are updated accordingly.

>TO DO LIST:
- [x] We can crosscheck the relation between fullvisitor_id, visit_date, country and city from the original table to see if we have the same relation in the other table.
- [ ] We can crosscheck the original value of each session with the new table data.
- [ ] Validate that the number of rows with composite PK(visit_id, fullvisitor_id, product_sku) is smaller than the original based on the assumption that each row tagged to a `product_sku` cannot have a the same information for these columns.
- [ ] Validate that the process indeed reduce the number of categories.
- [ ] Crosscheck the data with the original table to make sure no loss of information (or wrong alteration).
- [ ] Crosscheck the quantity of part from original tables compare to the `productinfo` total quantity. Not to compare similar quantity, but to see the impact of these rows if we would have considered them not errors (with at least a quantity of 1).
- [ ] Validate that the `clean_analytics` view contains the right information after the split.
><br>

**Queries**

```sql
-------------------------------------------
------------    QA process   --------------

-- [ ] We can crosscheck the relation between fullvisitor_id, visit_date, country and city 
-- from the original table to see if we have the same relation in the other tables.
WITH check_raw_country_city AS (
	SELECT -- Select all relations between fullvisitor_id, country and city
		a.fullvisitor_id, -- NOT DISTINCT
		a.country AS raw_country, vd.country ,
		a.city AS raw_city, vd.city,
		a.visit_date, vd.last_visit
	FROM all_sessions a
	JOIN visitor_details vd
	 	 ON a.fullvisitor_id = vd.fullvisitor_id
		AND a.visit_date = vd.last_visit
),
qa_country_city AS (
	SELECT
		'all_sessions' AS table,
		SUM(
			CASE 
				WHEN raw_country = country THEN 0 
				WHEN raw_country IN ('(not set)', 'not available in demo dataset') 
				 AND country IS NULL THEN 0
				ELSE 1 
			END
			) AS check_country,
		SUM(
			CASE 
				WHEN raw_city = city THEN 0 
				WHEN raw_city IN ('(not set)', 'not available in demo dataset', NULL)
				 AND city IS NULL THEN 0
				ELSE 1 
			END
			) AS check_city,
		CASE
			WHEN
				COUNT(DISTINCT(fullvisitor_id, visit_date)) = COUNT(DISTINCT(fullvisitor_id, last_visit)) THEN 0
				ELSE 1
			END AS id_count
	FROM
		check_raw_country_city
),
-- All other CTE checks....on going
-- then
qa_test AS ( -- collect all fail_count
	SELECT
		'check_raw_country_city' AS check_name,
		SUM(check_country + check_city + id_count) as fail_count
	FROM 
		qa_country_city
--UNION ALL
	--SELECT * FROM check 2
--UNION ALL
	--SELECT * FROM check 3
--UNION ALL
	--SELECT * FROM check 4
--UNION ALL
	--SELECT * FROM check 5
--UNION ALL
	--SELECT * FROM check 6
--UNION ALL
	--SELECT * FROM check 7
)
-- Final TEST
SELECT
	check_name,
	fail_count,
	CASE
		WHEN fail_count > 0 THEN 'FAIL'
		ELSE 'PASS'
	END
FROM qa_test
;
-- [ ] We can crosscheck the original value of each session with the new table data.


-- [ ] Validate that the number of rows with composite PK(visit_id, fullvisitor_id, product_sku) is smaller than the original based on the assumption that each row tagged to a `product_sku` cannot have a the same information for these columns.


-- [ ] Validate that the process indeed reduce the number of categories.
-- [ ] Crosscheck the data with the original table to make sure no loss of information (or wrong alteration).


-- [ ] Crosscheck the quantity of part from original tables compare to the `productinfo` total quantity. Not to compare similar quantity, but to see the impact of these rows if we would have considered them not errors (with at least a quantity of 1).


-- [ ] Validate that the `clean_analytics` view contains the right information after the split.
```

---
**Result of query:**

![QA result](images/Result%20of%20Final%20QA%20process.png)
