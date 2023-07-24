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

Validate that the number of rows with composite PK(visit_id, fullvisitor_id, product_sku) is smaller than the original based on the assumption that each row tagged to a `product_sku` cannot have a the same information for these columns.

#### QA. RAN A4 <a name="qa.-ran-a4"></a>

**Description of risk:**
>**Action:**
>+ TRUNCATE v2productcategory values to only keep the words before the last "/" , if exits, THEN remove all remaining "/".
>+ Replace "${escCatTitle}" category with "(not set)".
>+ Bonus action: After previous actions completed, if possible, we could try replacing the v2productcategory "(not set)" with a category name that exist in the v2productname.

Working with REGEX can create disaster when altering table. The goal here was to reduce the number of category AND replace the missing ones.

***Mitigation:***

- Validate that the process indeed reduce the number of categories.
- Crosscheck the data with the original table to make sure to loss of information (or wrong alteration).

#### QA. RAN A5 <a name="qa.-ran-a5"></a>

**Description of risk:**
>**Action:**
>+ Remove all rows with `productprice` = 0.

The risk here is more an assumption issue. We assume that these are errors, but if not, we are loosing a lot of information.

***Mitigation:***

Crosscheck the quantity of part from original tables compare to the `productinfo` total quantity. Not to compare similar quantity, but to see the impact of these rows if we would have considered them not errors (with at least a quantity of 1).


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


---
##**QUERIES AND PROCESS** <a name="queries-and-process"></a>



---
Mitigation:

+   number of rows with totaltransactionrevenue = ???? 58??
+ number of rows with revenue = 
+ validate total ordered product per sku at least not higher than total sold from productinfo
+ validate that clean_all_sessions row count is good


QA Process:
Describe your QA process and include the SQL queries used to execute it.
