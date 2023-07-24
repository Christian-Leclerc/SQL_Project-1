#                                   STARTING WITH QUESTIONS
*by Christian Leclerc b.ing, Data Analyst, [LHL Bootcamp](https://www.lighthouselabs.ca)*

---
## *Question 1:* Which cities and countries have the highest level of transaction revenues on the site?

### STRATEGY
Multiple issue with the timeframe difference between the tables makes it difficult to join them adequately. The strategy is to extract the most revenues we can that can be connected to the column `country` and `city`.

**Step 1:**
Extract the data from [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) TABLE where it's impossible to match [clean_analytics](cleaning_data.md#clean_analytics-view) VIEW (which is only starting from 2017-05-01) and where the column `totaltransactionrevenue` IS NULL (to be added on Step 2).
[QA. RAN A2](QA.md#qa.-ran-A2) - With `productquantity` not NULL anymore, we can calculate a minimum revenue for these sessions (assuming 1 product sold per session).

**Step 2:**
Extract and **UNION ALL** the data from [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) TABLE where `totaltransactionrevenue` IS NOT NULL but BEFORE analytics timeframe of 2017-05-01 (no match).

**Step 3:**
Extract and **UNION ALL** the data from [clean_analytics](cleaning_data.md#clean_analytics-view) VIEW this time, that cannot be found in [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) TABLE (except where `totaltransactionrevenue` IS NOT NULL AND `visit_date` >= 2017-05-01) AND ONLY WHERE the `fullvisitor_id` EXISTS in [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) because it's the only way to get the `country and city` associated. The information can be found in the new table [visitor_details](cleaning_data.md#the-visitor_details-table).

Note: With this strategy, we are not considering a lot of revenues from the original analytics table, but instead of showing lots of NULL countries and cities, we prefer analysing what we have.

#### SQL Queries:

Let's first create a view to UNION all source of revenues
```sql
-- Setup a revenues selection where no possible match with analytics (totaltransactionrevenue IS NULL)

DROP VIEW IF EXISTS summary_revenues
;
CREATE VIEW summary_revenues AS
	SELECT
		country, city,
		(productquantity * productprice) AS revenue -- (QA. RAN A2)
	FROM
		clean_all_sessions
	WHERE 
		totaltransactionrevenue IS NULL
	-- 14672 rows
	-- Total revenue = 431,984.08 $

UNION ALL 

-- Add the revenue where we don't have a match with analytics (totaltransactionrevenue IS NOT NULL)
-- AND visit_date < 2017-05-01 	
	SELECT
		country, city,
		totaltransactionrevenue AS revenue -- (QA. RAN A2)
	FROM
		clean_all_sessions
	WHERE
		totaltransactionrevenue IS NOT NULL
		AND visit_date < '2017-05-01'
	-- 59 rows
	-- Total revenue = 10,965.53 $

UNION ALL

-- Revenues from clean_analytics not found in clean_all_sessions 
-- but where we can tag a country, city based on existing fullvisitor_id in visitor_details

	SELECT
		vd.country, -- cannot be NULL
		vd.city, -- Could be NULL (QA. RAN A1)
		ca.revenue
	FROM
		clean_analytics ca
	JOIN 
		visitor_details vd ON ca.fullvisitor_id = vd.fullvisitor_id
		AND ca.visit_date = vd.last_visit
		-- each visit_date has been added to the table with same fullvisitor_id
	WHERE
		country IS NOT NULL -- requirement
	-- 475 rows
	-- Total revenue = 50,992.63 $
-- end summary_revenues
```

Let's now use the summary_revenues VIEW to answer specific questions
```sql

-- Answer: (Assumption, Top 5 countries and cities)
SELECT
	CASE
		WHEN type = 'country' THEN 'Country: ' || location
		WHEN type = 'city' THEN 'City: ' || location
	END AS "Top 5", -- Assumption Top 5
	total_revenue AS "Total revenues"
FROM (
	SELECT
		type,
		location,
		total_revenue,
        -- to rank each country/city and extract Top 5
		ROW_NUMBER() OVER (PARTITION BY type ORDER BY total_revenue DESC) AS rn
	FROM (
		SELECT -- To show country and city on same column
			'country' AS type,
			country AS location,
			SUM(revenue) AS total_revenue
		FROM 
			summary_revenues
		GROUP BY
			country
		
		UNION ALL

		SELECT -- To show country and city on same column
			'city' AS type,
			city AS location,
			SUM(revenue) AS total_revenue
		FROM
			summary_revenues
		WHERE
			city IS NOT NULL -- Half of the data missing
		GROUP BY
			city
	) AS subqueries
) AS ranked
WHERE
	rn <= 5 -- TOP 5, as per assumption
ORDER BY 
	type DESC,
	total_revenue DESC
;


```

#### Answer: 
![Answer Q-1](images/Answer%20Q-1.png)

## *Question 2:* What is the average number of products ordered from visitors in each city and country?

### STRATEGY

This question is a little bit tricky in the present context as:

- We have assume 1 product per row in [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) table where the data was missing, but we needed something to calculate a minimal revenue. IMPOSSIBLE to find the data in [clean_analytics](cleaning_data.md#clean_analytics-view) view.
- For the rows with a `totaltransactionrevenue`, we have to use the productquantity WHERE `visit_date` < 2017-05-01 (before analytics timeframe).
Althought, we could do the same for the rows AFTER 2017-05-01, we will use the process in Step 3 to extract the data from [clean_analytics](cleaning_data.md#clean_analytics-view) view .
- For the rest of the rows coming from [clean_analytics](cleaning_data.md#clean_analytics-view) view, it's impossible to know WHICH product was sold, but we have the data for the quantity (`unitssold`). Thats all we need !


**Step 1:**
Similar process as Question #1 but calculate the average number of `productquantity` including the rows with `totaltransactionrevenue` NOT NULL BEFORE 2017-05-01. 

**Step 2:**
SELECT in [clean_analytics](cleaning_data.md#clean_analytics-view) view the rows that match [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) table with "sessions" identified with same `visit_id` and `fullvisitor_id` to extract the `unitssold` of all the product sold (with revenue). Use same process in Question #1 Step 3 to associate each row with a country and a city. This will include all rows with `totaltransactionrevenue` NOT NULL from [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) table AFTER 2017-05-01.

#### SQL Queries:
Notes: 
+ Only the code for Country is SHOWN (replace country with city)
+ Because of the great numbers of `productquantity` = 1 from our assumption, the Average seems totally biased. We recalculate the average EXCLUDING these product just for comparaison.

**AVERAGE BIASED QUERY:**
```sql

-- Setup an average product sold selection where no possible match with analytics (totaltransactionrevenue IS NULL)
-- AND totaltransactionrevenue IS NOT NULL but BEFORE 2017-05-01
WITH all_products AS (
	SELECT
		country, city,
		productquantity AS product_sold
	FROM
		clean_all_sessions
	WHERE 
		totaltransactionrevenue IS NULL 
		OR (totaltransactionrevenue IS NOT NULL AND visit_date < '2017-05-01' )
	--  14,731 rows
	--  Total products = 14,867 

UNION ALL

-- Unitssold from clean_analytics not found in clean_all_sessions (except for rows
-- with totaltransactionrevenue NOT NULL AFTER 2017-05-01)
-- but where we can tag a country, city based on existing fullvisitor_id in visitor_details

	SELECT
		vd.country, -- cannot be NULL
		vd.city, -- Could be NULL (QA. RAN A1)
		ca.unitssold AS product_sold
	FROM
		clean_analytics ca
	JOIN 
		visitor_details vd ON ca.fullvisitor_id = vd.fullvisitor_id
		AND ca.visit_date = vd.last_visit
		-- each visit_date has been added to the table with same fullvisitor_id
	WHERE
		country IS NOT NULL -- requirement
) 	-- 475 rows
	-- Total product = 8,022

-- Answer: (Only Country shown, same for City)
SELECT 
	country, -- or city
	SUM(product_sold) AS total_product,
	ROUND(AVG(product_sold), 2) AS "Average Country Product Count" -- Or city
FROM 
	all_products
-- WHERE
    -- city IS NOT NULL
GROUP BY 
	country -- Or city
ORDER BY
	3 DESC
```
**AVERAGE LESS BIASED QUERY:**
```sql

-- Setup an average product sold selection where no possible match with analytics (totaltransactionrevenue IS NULL) AND quantity > 1
-- AND totaltransactionrevenue IS NOT NULL but BEFORE 2017-05-01
WITH all_products AS (
	SELECT
		country, city,
		productquantity AS product_sold
	FROM
		clean_all_sessions
	WHERE 		
                (totaltransactionrevenue IS NULL AND productquantity >1) -- CHANGE
		OR (totaltransactionrevenue IS NOT NULL AND visit_date < '2017-05-01' )
	--  64 rows
	--  Total products = 200

UNION ALL

-- Unitssold from clean_analytics not found in clean_all_sessions (except for rows
-- with totaltransactionrevenue NOT NULL AFTER 2017-05-01)
-- but where we can tag a country, city based on existing fullvisitor_id in visitor_details

	SELECT
		vd.country, -- cannot be NULL
		vd.city, -- Could be NULL (QA. RAN A1)
		ca.unitssold AS product_sold
	FROM
		clean_analytics ca
	JOIN 
		visitor_details vd ON ca.fullvisitor_id = vd.fullvisitor_id
		AND ca.visit_date = vd.last_visit
		-- each visit_date has been added to the table with same fullvisitor_id
	WHERE
		country IS NOT NULL -- requirement
) 	-- 475 rows
	-- Total product = 8,022

-- Answer: (Only Country shown, same for City)
SELECT 
	country, -- or city
	SUM(product_sold) AS total_product,
	ROUND(AVG(product_sold), 2) AS "Average Country Product Count" -- Or city
FROM 
	all_products
-- WHERE
    -- city IS NOT NULL
GROUP BY 
	country -- Or city
ORDER BY
	3 DESC
```

#### Answer: 
Less Biased Average Product per Country and City (only a portion of the result shown for City result).

![Less Biased Average Product per Country](images/Q2%20-%20Less%20Biased%20Average%20Product%20per%20Country.png)

![Less Biased Average Product per City](images/Q2%20-%20Less%20Biased%20Average%20Product%20per%20City.png)

Biased Average Product per Country and City (only a portion of the result shown).

![Less Biased Average Product per Country](images/Q2%20-%20Biased%20Average%20Product%20per%20Country.png)

![Less Biased Average Product per City](images/Q2%20-%20Biased%20Average%20Product%20per%20City.png)

## *Question 3:* Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?

### STRATEGY

This time, we cannot use the [clean_analytics](cleaning_data.md#clean_analytics-view) view data to know what type of product was sold (unknown data). Even if we extract the quantity (`unitssold`) for "sessions" that match in [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) table, the quantity will not reflect the reality since there is only one product per row in clean_all_sessions table; we need to rely on this last table only.

First, we will create a `desc_statistics` VIEW that will summarize what we can get from the [clean_all_sessions](cleaning_data.md#clean_all_sessions-table) TABLE. We will then create specific queries to answer the next questions.

#### SQL Queries:
**Descriptive statistics VIEW**

```sql
DROP VIEW IF EXISTS desc_statistics
;
CREATE VIEW desc_statistics AS
-- Count total product
	WITH product_counts AS (
		SELECT
		country,
		city,
		v2productcategory,
		SUM(productquantity) AS total_product_quantity
	FROM
		clean_all_sessions
	GROUP BY
		country, city, v2productcategory
	),
-- Calculate average of total_count for city
	avg_counts_city AS (
		SELECT
			country,
			city,
			AVG(total_product_quantity) AS avg_product_quantity_city
		FROM
			product_counts
		GROUP BY
			country, city
	),
-- Calculate average of total_count for country
avg_counts_country AS (
	SELECT
		country,
		AVG(total_product_quantity) AS avg_product_quantity_country
	FROM
		product_counts
	GROUP BY
		country
	),
-- Give a rank for city per category base on total product
	city_category_ranks AS (
		SELECT
			country,
			city,
			v2productcategory,
			total_product_quantity,
			RANK() OVER (PARTITION BY country, city ORDER BY total_product_quantity DESC) AS city_category_rank
		FROM
			product_counts
	),
-- Give a rank for country and city base on average product
	country_city_ranks AS (
		SELECT
			country,
			city,
			v2productcategory,
			total_product_quantity,
			RANK() OVER (PARTITION BY country ORDER BY avg_product_quantity_city DESC) AS country_city_rank
		FROM
			product_counts
		JOIN avg_counts_city USING (country, city)
	)
-- JOIN all together
	SELECT
		pc.country,
		pc.city,
		pc.v2productcategory,
		pc.total_product_quantity,
		acc.avg_product_quantity_city,
		acco.avg_product_quantity_country,
		ccr.city_category_rank,
		ccr2.country_city_rank
	FROM
		product_counts pc
	JOIN avg_counts_city acc USING (country, city)
	JOIN avg_counts_country acco USING (country)
	JOIN city_category_ranks ccr USING (country, city, v2productcategory)
	JOIN country_city_ranks ccr2 USING (country, city, v2productcategory)
```
Use previous build [desc_statistics](starting_with_questions.md#sql-queries-2) VIEW and show:
- How many products were sold per country, city and category
- With only TOP 1 rank country, city

If we have a "nebulous" category, we can then deep-dive category name with the `v2productname` directly from clean_all_sessions.

#### SQL Queries:
TOP 1's per city
```sql
SELECT 
	v2productcategory,
	country,
	city,
	total_product_quantity
FROM
	desc_statistics -- VIEW created in Question #3
WHERE -- Only TOP 1 categories
		city_category_rank = 1
ORDER BY
	total_product_quantity DESC
```
TOP 1's per country
```sql
-- TOP 1 cat per country
SELECT 
	v2productcategory,
	country,
	SUM(productquantity) AS total_per_country
FROM
	clean_all_sessions -- So we can count WHERE city IS NULL
GROUP BY 1, 2
ORDER BY
	total_per_country DESC
```


Deep dive YouTube Category
```sql
SELECT
	v2productname,
	SUM(productquantity) AS product_count
FROM
	clean_all_sessions -- to get the product name
WHERE
	v2productcategory = 'YouTube'
	AND city IS NOT NULL -- to match desc_statistics VIEW
GROUP BY 
	v2productname
ORDER BY
	product_count DESC
```

#### Answer: 
WINNER WINNER CHICKEN DINNER !!! - Men's T-Shirts

TOP 1 categories per city
(only a portion of the result shown)
![Q3 - TOP 1 categories per City](images/Q3%20-%20TOP%201%20cat%20per%20City.png)

TOP 1 categories per country
![Q3 - TOP 1 categories per Country](images/Q3%20-%20TOP%201%20cat%20per%20Country.png)

TOP Categories Bar Chart
![Q3 - TOP Categories](images/Q3%20-%20TOP%20categories.png)

Since YouTube category is not really a category, let's deep dive its content
![Q3 - YouTube Category](images/Q3%20-%20Deep-dive%20YouTube%20category.png)
We see that inside the YouTube category, Men'S and Women'S shirt (or clothes related items) are not the top ones. It's more Office related.

## *Question 4:* What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?

### STRATEGY

We will first use the `product_sku` and `productquantity` to calculate the number of distinct products to figure out TOP selling ones.
We will then compare the overall result to the `productinfo` table where we assume that the overall quantity of products is recorded (thougth no way to group them in country or city). 
Note that only the "sessions" where `totaltransactionrevenue` IS NULL will be use. As mention in Question #1, the session with revenue have only one row per multiple product and we can't find the data from any other table. These represent only 81 rows out of 15129.
Most of the remaining rows have productquantity of 1 because we replaced the NULL value with 1 to calculate revenues. In this case, calculating TOP 1 selling product per city, where half of the city is NULL is pointless.
**We will only find the Top-selling product per country.**


#### SQL Queries:
Let's look at the overall TOP selling products
```sql
-- TOP selling product overall
SELECT
	product_SKU,
	v2productname,
	SUM(productquantity) AS total_quantity
FROM
	clean_all_sessions
WHERE
	country != '(not set)'
	AND totaltransactionrevenue IS NULL
GROUP BY
	product_SKU,
	v2productname
ORDER BY
	total_quantity DESC
```
Than we can group them by country and retain only TOP 1 per country
```sql

-- TOP selling product by country
WITH rank_per_country AS ( -- find the ranking per country
	SELECT
		country,
		product_SKU,
		v2productname,
		SUM(productquantity) AS total_quantity,
		RANK() OVER(PARTITION BY country ORDER BY SUM(productquantity) DESC) AS rk
	FROM
		clean_all_sessions
	WHERE
		country != '(not set)' -- for visual purpose
		AND totaltransactionrevenue IS NULL -- as per assumption
	GROUP BY
		country,
		product_SKU,
		v2productname
)
	SELECT *
	FROM 
		rank_per_country
	WHERE
		rk = 1 -- requirement
	ORDER BY
	total_quantity DESC
```
#### Answer: 
TOP Selling product overall (for comparaison)
![Q4 - TOP Selling Products](images/Q4%20-%20TOP%20Selling%20Product.png)
TOP selling product per Country
![Q4 - TOP Selling Product per Country](images/Q4%20-%20TOP%20Selling%20Product%20per%20Country.png)
We can see that the TOP selling product overall is the Google Men's-T-Shirts, and it is also TOP selling country with the same product. It match the Category Men's-T-Shirts we found in Question #3.
We could think that because the majority of this product is sold in USA, means that only this quantity have an impact on the distribution, but on the contrary, many countries have this product as TOP selling. Let's look specifically that product selling per country.

![Q4 - Men Shirt distribution per country](images/Q4%20-%20Men%20Shirt%20distribution%20per%20country.png)
The same product is Rank 1 on many countries. (Except where quantity is 1, only in alphabetical order).

## *Question 5:* Can we summarize the impact of revenue generated from each city/country?

### STRATEGY

We could create the same VIEW as [desc_statistics](starting_with_questions.md#sql-queries-2) , but starting from the [summary_revenues](starting_with_questions.md#sql-queries) VIEW and removing the category. We should get a nice `revenue_statistic` VIEW we can use.

#### SQL Queries:
**Descriptive statistic for revenues VIEW**
```sql
DROP VIEW IF EXISTS revenue_statistics
;
CREATE VIEW revenue_statistics AS
-- Count total revenue
	WITH product_revenues AS (
		SELECT
			country,
			city,
			SUM(revenue) AS total_revenue
		FROM
			summary_revenues -- re-use VIEW
		GROUP BY
			country, city
	),
-- Calculate average revenue per city
	avg_revenue_city AS (
		SELECT
			country,
			city,
			AVG(total_revenue) AS avg_revenue_city
		FROM
			product_revenues
		GROUP BY
			country, city
	),
-- Calculate average of total_revenue for country
avg_revenue_country AS (
	SELECT
		country,
		AVG(total_revenue) AS avg_revenue_country
	FROM
		product_revenues
	GROUP BY
		country
	),
-- Give a rank for city base on total revenue
	city_ranks AS (
		SELECT
			country,
			city,
			total_revenue,
			RANK() OVER (PARTITION BY country, city ORDER BY total_revenue DESC) AS city_rank
		FROM
			product_revenues
	),
-- Give a rank for country and city base on average revenue
	country_city_ranks AS (
		SELECT
			country,
			city,
			total_revenue,
			RANK() OVER (PARTITION BY country ORDER BY avg_revenue_city DESC) AS country_city_rank
		FROM
			product_revenues
		JOIN avg_revenue_city USING (country, city)
	)
-- JOIN all together
	SELECT
		pr.country,
		pr.city,
		pr.total_revenue,
		ROUND(arc.avg_revenue_city, 2) AS avg_revenue_city,
		ROUND(arc2.avg_revenue_country, 2) AS avg_revenue_country,
		cr.city_rank,
		ccr.country_city_rank
	FROM
		product_revenues pr
	JOIN avg_revenue_city arc USING (country, city)
	JOIN avg_revenue_country arc2 USING (country)
	JOIN city_ranks cr USING (country, city)
	JOIN country_city_ranks ccr USING (country, city)
	ORDER BY 
		country, city
-------- end revenue_statistics VIEW
```

Select summary of revenues country/city
```sql
SELECT *
FROM 
	revenue_statistics
ORDER BY
	avg_revenue_city DESC
```

#### Answer: 
Only top 20 shown.
VIEW revenue_statistics
![Q5 - Summary of revenues](images/Q5%20-%20Summary%20of%20revenues.png)

