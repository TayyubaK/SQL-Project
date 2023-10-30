### **Risk Areas**
What are your risk areas? Identify and describe them.
The major risk areas are blanks, duplicates, data anomalies and data types.
* Columns with blank/near empty, non-conforming, and/or duplicate values, for example:
        
    * all_sessions.totaltransactionrevenue has only 81 rows with a value out of 15,134 rows
    * all_sessions.country has no nulls but some countries are listed as '(not set)'
    * all_sessions.productsku has duplicates in the column

* All data loaded to pgAdmin as character varying. Correct data type must be cast for some functions. 

* With CTEs and subqueries can create a lot of nesting and potential to lose data unintentionally. Each part has to be checked to ensure each piece is working as intended.


### **QA Process**

General Approach: 
* To address the risk of blank and/or duplicate values, check that these values were removed/updated correctly
* Compare counts and results from query against individual query results of the table(s)
* To address data type risks, review queries and ensure data types selected are appropriate for the context of the column and the calculations being applied.
* For CTE and subsqueries, run query piece-by-piece and evaluate the output

    * All columns appear as expected and with the correct data type
    * Row counts are reasonable and changes in each step match expectations
    * Calculations are accurate
* Visual inspection, manual calculation/checks, reviewing subsets of the final answer

### **tmp_alls_products - QA**
This temp table was needed because all_sessions.productsku (unique identifier for a product) had more than 1 name associated with it.

Check that the unique count of productsku values matches between all_sessions and the temp table.
```sql
SELECT COUNT(DISTINCT productsku)
FROM all_sessions;
--Output -> 536

SELECT COUNT(DISTINCT productsku)
FROM tmp_alls_products;
--Output -> 536

SELECT COUNT(productsku)
FROM tmp_alls_products;
--Output -> 536; temp table has only 1 row for each productsku
```
```sql
SELECT 
    DISTINCT productsku
FROM 
    all_sessions
GROUP BY 
    productsku
HAVING 
    count(DISTINCT(v2productname))>1
ORDER BY productsku;
--74 rows affected; all_sessions had 74 productsku values with different v2productname values

SELECT 
    DISTINCT productsku
FROM 
    tmp_alls_products
GROUP BY 
    productsku
HAVING 
    count(DISTINCT(v2productname))>1
ORDER BY productsku;
--0 rows; all productsku and v2productname values have a 1:1 relationship in the temp table
```
Manually check a value - productsku='9182593'
```sql
SELECT 
    productsku, 
    v2productname, 
    COUNT(v2productname)
FROM 
    all_sessions
WHERE 
    productsku='9182593'
GROUP BY 
    productsku, v2productname;
-- 2 rows affected; 
--'Google Men's Pullover Hoodie' name had a count of 1
--'Google Men's Pullover Hoodie Grey' had a count of 3

--tmp_alls_products should only have the name with the higher count in all_sessions.
SELECT 
    productsku, 
    v2productname, 
    COUNT(v2productname)
FROM 
    tmp_alls_products
WHERE 
    productsku='9182593'
GROUP BY 
    productsku, v2productname;
--1 row affected; 
--only 'Google Men's Pullover Hoodie Grey' is in the temp table
```

### **tmp_clean_cat - QA**
This table was created because all_sessions.v2productcategory had the majority of categories starting with 'Home/', which didn't allow for drilling down.

Check that the unique count of v2productcategory values matches between all_sessions and the temp table.
```sql
SELECT COUNT(DISTINCT v2productcategory)
FROM all_sessions;
--Output -> 74

SELECT COUNT(DISTINCT v2productcategory)
FROM tmp_clean_cat;
--Output -> 74

SELECT COUNT(v2productcategory)
FROM tmp_clean_cat;
--Output -> 74; temp table has only 1 row for each unique v2productcategory that was in all_sessions

SELECT 
    DISTINCT v2productcategory
FROM 
    tmp_clean_cat
GROUP BY 
    v2productcategory
HAVING 
    count(DISTINCT(main_category))=1
ORDER BY v2productcategory;
--74 rows; each v2productcategory has 1 value for main_category (new column unique to the temp table)
```

### **Main Subset - Transaction Data and Country/City**

As many questions ask to consider orders for city/country, several queries use the same conditions to identify rows with transction data and country/city information.
```sql
--Check number of rows indicating a transaction (transactions=1)
SELECT 
    CASE 
        WHEN country='(not set)' THEN 'NULL'
        ELSE country
    END AS country,
    CASE
        WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
        ELSE city
    END AS city
FROM 
    all_sessions
WHERE (transactions::INT)=1;
--81 rows affected
```
```sql
--add in condition for totaltransactionrevenue 
SELECT 
    CASE 
        WHEN country='(not set)' THEN 'NULL'
        ELSE country
    END AS country,
    CASE
        WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
        ELSE city
    END AS city
FROM 
    all_sessions
WHERE (transactions::INT)=1
    AND totaltransactionrevenue IS NOT NULL;
--81 rows affected; this means that rows that have transactions=1, also have a value for totaltransactionrevenue
```
```sql
--Check how many rows with transaction data in all_sessions have countries that are '(not set)'
SELECT 
    country, 
    city
FROM all_sessions
WHERE (transactions::INT)=1
    AND totaltransactionrevenue IS NOT NULL
    AND country='(not set)';
--0 rows affected; there were no rows with transaction data where the country column had '(not set)'
```
```sql
--Check how many rows with transaction data in all_sessions have cites that are '(not set)' or 'not available in demo dataset'
SELECT 
    country, 
    city
FROM all_sessions
WHERE (transactions::int)=1
    AND totaltransactionrevenue IS NOT NULL
    AND city IN ('not available in demo dataset','(not set)');
--25 rows affected.
```
```sql
--Check how many rows have a 'NULL' value for country or city after CASE update
WITH null_check AS (
    SELECT 
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city
    FROM 
        all_sessions
    WHERE (transactions::int)=1
)
SELECT * 
FROM null_check
WHERE country = 'NULL'
    OR city = 'NULL';
--25 rows affected; this matches with the number of cities that had '(not set)' or 'not available in demo dataset'
```
```sql
--Check how many rows remain after 'NULL' country/city values are removed
WITH null_check AS (
    SELECT 
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city
    FROM 
        all_sessions
    WHERE (transactions::int)=1
)
SELECT * 
FROM null_check
WHERE country <> 'NULL'
    AND city <> 'NULL';
--56 rows affected; 
--81 rows with transaction data, 25 rows of those had cities that would be updated to 'NULL'. 
--81-25 = 56 so the query correctly removes rows with country/city data anomalies.
```

### **starting_with_questions - Question 1**

Check counts of country, city, and sum of transaction revenue.

**Count of distinct countries in result set should match distinct countries with transaction revenue values.**

Count of unique countries in result set: 5
```sql
SELECT COUNT(DISTINCT country) FROM ( 
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city,
        SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM 
        all_sessions
    WHERE 
        totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country, 
        city
    ORDER BY 
        totalrev DESC
)
SELECT * 
FROM 
    q1_clean
WHERE 
    country <> 'NULL' AND city <> 'NULL'
GROUP BY 
    country, 
    city, 
    totalrev
ORDER BY 
    totalrev DESC
);
--5
```
Count of unique countries in all_sessions table with transaction revenue: 5
```sql
SELECT 
    COUNT(DISTINCT country)
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL;
--5

--The Q1 result set excludes '(not set)' values for country. Check that the count of 5 doesn't include '(not set)':
SELECT
    DISTINCT country
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL;

--1. Australia
--2. Canada
--3. Israel
--4. Switzerland
--5. United States
```

**Count of distinct cities in result set should match distinct cities with transaction revenue values, excluding '(not set)' and 'not available in demo dataset' values.** 

Count of unique cities in result set: 19

```sql
SELECT COUNT(DISTINCT city) FROM (
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city,
        SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM 
        all_sessions
    WHERE 
        totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country, 
        city
    ORDER BY 
        totalrev DESC
)
SELECT * 
FROM 
    q1_clean
WHERE 
    country <> 'NULL' AND city <> 'NULL'
GROUP BY 
    country, 
    city, 
    totalrev
ORDER BY 
    totalrev DESC
);
--19
```
Count of unique cities in all_sessions table with transaction revenue: 19

```sql
SELECT 
    COUNT(DISTINCT city)
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL 
    AND city NOT IN ('(not set)','not available in demo dataset');
--19
```
**Sum of total revenue in result set should match sum of transaction revenue values where county and city aren't '(not set)' and 'not available in demo dataset'.**

Sum of transaction revenue from result set: 8,188.75
```sql
SELECT SUM(totalrev) FROM (
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city,
        SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM 
        all_sessions
    WHERE 
        totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country, 
        city
    ORDER BY 
        totalrev DESC
)
SELECT * 
FROM 
    q1_clean
WHERE 
    country <> 'NULL' AND city <> 'NULL'
GROUP BY 
    country, 
    city, 
    totalrev
ORDER BY 
    totalrev DESC
);
--8188.75
```

Sum of transaction revenue from all_sessions table: 8,188.75

```sql
SELECT 
    SUM(ROUND(totaltransactionrevenue::NUMERIC/1000000,2))
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL
    AND country NOT IN ('(not set)','not available in demo dataset')
    AND city NOT IN ('(not set)','not available in demo dataset');
--8188.75
```

### **starting_with_questions - Question 2**

**Check CTE 'unit_data'** query by uncommenting the matching opening and ending queries. The uncommented part stays for all checks.
* 7 columns queried, and 7 columns in the result set
* Check 1: All values for transactions should be 1.
* Check 2: There should be no rows where country is '(not set)' or city is 'not available in demo dataset' or '(not set)'
* Check 3: There should be no rows where units_sold is NULL
* Check 4: Check total row count
* Check 5: Check how many countries were updated to 'NULL'
* Check 6: Check how many cities were updated to 'NULL'

```sql
--Check 1: SELECT COUNT(transactions) FROM
--Check 2: SELECT COUNT(country), COUNT(city) FROM
--Check 3: SELECT COUNT(units_sold) FROM 
--Check 4: SELECT COUNT(*) FROM 
--Check 5: SELECT COUNT(country) FROM
--Check 6: SELECT COUNT(city) FROM
(SELECT 
        DISTINCT ON (
            alls.totaltransactionrevenue, 
            alls.transactions, 
            a.units_sold, 
            tmp_alls_products.productsku, 
            tmp_alls_products.v2productname, 
            alls.country, 
            alls.city)
        totaltransactionrevenue, 
        transactions, 
        units_sold, 
        tmp_alls_products.productsku,
        tmp_alls_products.v2productname,
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city
    FROM 
        analytics a
    JOIN all_sessions alls ON a.fullvisitorid=alls.fullvisitorid
    JOIN tmp_alls_products ON alls.productsku=tmp_alls_products.productsku
    WHERE transactions::INT=1
        AND(totaltransactionrevenue::INT) IS NOT NULL
        AND (units_sold::INT) IS NOT NULL
    ORDER BY totaltransactionrevenue DESC)
--Check 1: WHERE (transactions::INT)=1;
--Check 2: WHERE country IN ('(not set)') OR city IN ('not available in demo dataset','(not set)');
--Check 3: WHERE units_sold IS NULL; 
--Check 4: ---
--Check 5: WHERE country='NULL'
--Check 6: WHERE city='NULL'

--Check 1 output -> 39
--Check 2 output -> 0, 0 
--Check 3 output -> 0
--Check 4 output -> 39
--Check 5 output -> 0
--Check 6 output -> 11 
```
Compare results to querying all_sessions, analytics, tmp_alls_products
```sql
SELECT 
    DISTINCT(totaltransactionrevenue), 
    transactions, 
    units_sold, 
    tmp_alls_products.productsku, 
    tmp_alls_products.v2productname
FROM 
    all_sessions alls
JOIN analytics a ON alls.fullvisitorid=a.fullvisitorid
JOIN tmp_alls_products ON alls.productsku=tmp_alls_products.productsku
WHERE (transactions::INT)=1
    AND (a.units_sold::INT) > 0
    AND (a.units_sold::INT) IS NOT NULL
    AND totaltransactionrevenue IS NOT NULL

--39 rows
```
**Check CTE 'q2_clean'** - this filters out 'NULL' country and city values. Since the first CTE had 39 rows and 11 were updated to NULL (check #6 above), **expected result is 28 rows**.
```sql
WITH unit_data AS(
    SELECT 
        DISTINCT ON (
            alls.totaltransactionrevenue, 
            alls.transactions, 
            a.units_sold, 
            tmp_alls_products.productsku, 
            tmp_alls_products.v2productname, 
            alls.country, 
            alls.city)
        totaltransactionrevenue, 
        transactions, 
        units_sold, 
        tmp_alls_products.productsku,
        tmp_alls_products.v2productname,
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city
    FROM 
        analytics a
    JOIN all_sessions alls ON a.fullvisitorid=alls.fullvisitorid
    JOIN tmp_alls_products ON alls.productsku=tmp_alls_products.productsku
    WHERE (transactions::INT)=1
        AND(totaltransactionrevenue::INT) IS NOT NULL
        AND (units_sold::INT) IS NOT NULL
    ORDER BY totaltransactionrevenue DESC
)
--,
--q2_clean AS (
	SELECT * FROM unit_data
	WHERE country <> 'NULL'
	AND city <> 'NULL'

--Output -> 28 rows

```
Compare results to querying all_sessions, analytics, tmp_alls_products
```sql
SELECT 
    DISTINCT(totaltransactionrevenue), 
    transactions, 
    units_sold, 
    tmp_alls_products.productsku, 
    tmp_alls_products.v2productname
FROM 
    all_sessions alls
JOIN analytics a ON alls.fullvisitorid=a.fullvisitorid
JOIN tmp_alls_products ON alls.productsku=tmp_alls_products.productsku
WHERE (transactions::INT)=1
    AND (a.units_sold::INT) > 0
    AND (a.units_sold::INT) IS NOT NULL
    AND totaltransactionrevenue IS NOT NULL
--adding in additional conditions
	AND country NOT IN ('(not set)')
	AND city NOT IN ('not available in demo dataset','(not set)')

--Output -> 28 rows
```
Manual check - The query below keeps the same rows but a column 'city_unit_count' is added to sum the number units_sold for that city/country group. Each row represents an order. 

For example, the city_unit_count=10 for San Franciso, United Stated, and the number of rows is 6. If 10 products were ordered across 6 orders, then average is 1.66, which matches the result set of the query.
```sql
WITH unit_data AS(
    SELECT 
        DISTINCT ON (
            alls.totaltransactionrevenue, 
            alls.transactions, 
            a.units_sold,
			tmp_alls_products.productsku, 
            tmp_alls_products.v2productname, 
            alls.country,
			alls.city)
		totaltransactionrevenue, 
		transactions, 
		units_sold, 
		alls.productsku,
		tmp_alls_products.v2productname,
		CASE 
			WHEN country='(not set)' THEN 'NULL'
			ELSE country
		END AS country,
		CASE
			WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
			ELSE city
		END AS city
	FROM 
        analytics a
	JOIN all_sessions alls ON a.fullvisitorid=alls.fullvisitorid
	JOIN tmp_alls_products ON alls.productsku=tmp_alls_products.productsku
	WHERE transactions::INT=1
	AND (units_sold::INT) IS NOT NULL
	ORDER BY totaltransactionrevenue DESC
),
q2_clean AS (
	SELECT * 
    FROM 
        unit_data
	WHERE country <> 'NULL'
	    AND city <> 'NULL'
)
SELECT 
    country,
	city,
	SUM(units_sold::INT) OVER (
        PARTITION BY country, city) AS city_unit_count,
	ROUND(AVG(units_sold::INT) OVER (
		PARTITION BY country, city),2) AS avg_total_ordered
FROM 
    q2_clean
GROUP BY 
    country, 
    city, 
    v2productname, 
    units_sold;
--28 rows
```

### **starting_with_questions - Question 3**

* CTE 'main_group' has 56 rows, no nulls. This is ok as it aligns with 'Main Subset - Transaction Data and County/City' and 'tmp_clean_cat - QA' (see above).

```sql
SELECT DISTINCT(main_category), country, city FROM (
SELECT 
    ao.country, 
    ao.city,
    ao.productsku,
    ao.v2productcategory, 
    tmp_clean_cat.main_category
FROM (
    SELECT 
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city, 
        productsku, 
        v2productcategory
    FROM 
        all_sessions
    WHERE (transactions::INT)=1) ao
LEFT JOIN tmp_clean_cat ON ao.v2productcategory=tmp_clean_cat.v2productcategory
WHERE country <> 'NULL'
    AND city <> 'NULL'
)
--The query for main group has 56 rows in total. 
--Within those there are 41 distinct main categories for city and country.
--Visually inspecting the results, main_category='Apparel' has 12 rows for different cities/countries.
```

* CTE 'main_cat_count_tbl' reduces row count to 41 which meets expectations because it counts and ranks the main_category over country and city. As seen before, CTE 'main group' had 41 distinct main categories for city and country.
```sql
--Checking the query for main_cat_count_tbl for cat_rank=1, 2, or 3 and main_category='Apparel'
SELECT * FROM (
WITH main_group AS (
    SELECT 
        ao.country, 
        ao.city,
        ao.productsku,
        ao.v2productcategory, 
        tmp_clean_cat.main_category
    FROM (
        SELECT 
            CASE 
                WHEN country='(not set)' THEN 'NULL'
                ELSE country
            END AS country,
            CASE
                WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
                ELSE city
            END AS city, 
            productsku, 
            v2productcategory
        FROM 
            all_sessions
        WHERE (transactions::INT)=1) ao
    LEFT JOIN tmp_clean_cat ON ao.v2productcategory=tmp_clean_cat.v2productcategory
    WHERE country <> 'NULL'
        AND city <> 'NULL'
)
    SELECT 
        country, 
        city, 
        main_category, 
        COUNT(main_category) AS main_cat_count,
        DENSE_RANK() OVER (
            PARTITION BY country, city 
            ORDER BY COUNT(main_category) DESC) AS cat_rank
    FROM 
        main_group
    GROUP BY 
        country, 
        city, 
        main_category
)
WHERE cat_rank=1 --alternate: WHERE cat_rank=2
	AND main_category='Apparel'
--10 rows; this indicates 'Apparel is rank #1 in 10 cities
--alternate query has 2 rows; this indicates 'Apparel' is rank #2 in 2 cities
--setting condition for cat_rank=3 has 0 results
--CTE 'main_group' showed 'Apparel' had 12 rows. 
--The query for CTE 'main_cat_count_tbl' correctly assigned all 12 rows a rank within their country/city grouping
```

* SELECT query filters for #1 ranked categories for each country/city, and counts how many times the rank was #1. The result set shows that 'Apparel' was ranked #1 for ten country/city groups. This summary aligns with the results of the base CTE queries.

    Updating the SELECT query to 'WHERE cat_rank=2' shows 'Apparel' was ranked #2 for 2 country/city groups. The 12 rows have been maintained.
```sql
WITH main_group AS (
    SELECT 
        ao.country, 
        ao.city,
        ao.productsku,
        ao.v2productcategory, 
        tmp_clean_cat.main_category
    FROM (
        SELECT 
            CASE 
                WHEN country='(not set)' THEN 'NULL'
                ELSE country
            END AS country,
            CASE
                WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
                ELSE city
            END AS city, 
            productsku, 
            v2productcategory
        FROM 
            all_sessions
        WHERE (transactions::INT)=1) ao
    LEFT JOIN tmp_clean_cat ON ao.v2productcategory=tmp_clean_cat.v2productcategory
    WHERE country <> 'NULL'
        AND city <> 'NULL'
),
main_cat_count_tbl AS (
    SELECT 
        country, 
        city, 
        main_category, 
        COUNT(main_category) AS main_cat_count,
        DENSE_RANK() OVER (
            PARTITION BY country, city 
            ORDER BY COUNT(main_category) DESC) AS cat_rank
    FROM 
        main_group
    GROUP BY 
        country, 
        city, 
        main_category
)
SELECT 
    main_category, 
    count(main_category) AS count_rank_1
FROM 
    main_cat_count_tbl
WHERE 
    cat_rank=1 --alternate: cat_rank=2
GROUP BY 
    main_category
ORDER BY 
    count_rank_1 DESC;
--cat_rank=1 result set shows 'Apparel' with a value of 10
--cat_rank=2 result set shows 'Apparel' with a value of 2
--this summary result set has kept rows from the base subset intact
```

### **starting_with_questions - Question 4**

* CTE 'main_group' should only keep results where all_sessions.productquantity>0 and city data is clean.
```sql
--How many rows in all_sessions have productquantity greater than 0?
SELECT 
    country, 
    city, 
    productquantity
FROM all_sessions 
WHERE (productquantity::INT)>0  
--53 rows affected.

--How many of the 53 have country or city values that should be excluded?
SELECT 
    country, 
    city, 
    productquantity
FROM all_sessions 
WHERE (productquantity::INT)>0  
    AND country IN ('(not set)')
--0 rows affected

SELECT 
    country, 
    city, 
    productquantity
FROM all_sessions 
WHERE (productquantity::INT)>0  
    AND city IN ('not available in demo dataset','(not set)')
--21 rows affected

--How many rows result from the  CTE 'main_group' query?
SELECT COUNT(*) FROM(
SELECT 
    ao.country, 
    ao.city, 
    ao.productsku, 
    ao.v2productcategory, 
    ao.productquantity,
    tmp_clean_cat.main_category
FROM (
    SELECT 
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city, 
        productsku, 
        v2productcategory,
        (productquantity::INT)
    FROM all_sessions
    WHERE (productquantity::INT) > 0) ao
LEFT JOIN tmp_clean_cat 
    ON ao.v2productcategory=tmp_clean_cat.v2productcategory
WHERE country <> 'NULL'
    AND city <> 'NULL'
)
--Output --> 32
--There are 53 rows in all_sessions with a productquantity value greater than 0. 
--21 of those 53 rows have a value for city that will be 'NULL' and removed.
--53-21=32; The CTE query correctly filters for rows from all_sessions where productquantity is greater than 0 and 'NULL' city values are removed.
```
* CTE 'rank_tbl' should sum up the productquantity values and rank them for each country/city group

```sql
--How many distinct country/city/productname groupings are there in all_sessions and tmp_all_products with productquanity greater than 0?
SELECT DISTINCT ON (alls.country, alls.city, tap.v2productname)
    alls.country,
    alls.city,
    tap.v2productname,
    alls.productquantity
FROM all_sessions alls
JOIN tmp_alls_products tap ON alls.productsku=tap.productsku
WHERE (alls.productquantity::INT)>0  
    AND alls.city NOT IN ('not available in demo dataset','(not set)')
--30 rows affected
```
```sql
--How many rows result from the CTE 'rank_tbl' query?

SELECT COUNT(*) FROM (
WITH main_group AS (
    SELECT 
        ao.country, 
        ao.city, 
        ao.productsku, 
        ao.v2productcategory, 
        ao.productquantity,
        tmp_clean_cat.main_category
    FROM (
        SELECT 
            CASE 
                WHEN country='(not set)' THEN 'NULL'
                ELSE country
            END AS country,
            CASE
                WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
                ELSE city
            END AS city, 
            productsku, 
            v2productcategory,
            (productquantity::INT)
        FROM all_sessions
        WHERE (productquantity::INT) > 0) ao
    LEFT JOIN tmp_clean_cat 
        ON ao.v2productcategory=tmp_clean_cat.v2productcategory
    WHERE country <> 'NULL'
        AND city <> 'NULL'
)
SELECT 
	country,
	city,
	tmp_alls_products.v2productname,
	SUM(productquantity) AS prod_count,
	DENSE_RANK() OVER (
		PARTITION BY country, city 
		ORDER BY SUM(productquantity) DESC) AS top_prod_rank
FROM 
	main_group mg
LEFT JOIN tmp_alls_products ON mg.productsku=tmp_alls_products.productsku
GROUP BY 
	mg.country, 
	mg.city, 
	tmp_alls_products.v2productname
)
--Output without any WHERE condition --> 30
--Agreement between both queries

--Add in the WHERE condition at the end and that's why the result set output is 22
WHERE top_prod_rank=1
--Output --> 22
```
* SELECT query gives the final result of product counts ranked #1. Result set has 22 rows. This matches the row count of CTE 'rank_tbl' when the condition 'WHERE top_prod_rank=1' was added. 

* Manual Check: Final result set indicates a product count of 10 for 'Waze Dress Socks' in Madrid Spain. Confirm using all_sessions and tmp_alls_products:
```sql
SELECT DISTINCT ON (alls.country, alls.city, tap.v2productname)
    alls.country,
    alls.city,
    tap.v2productname,
    alls.productquantity
FROM all_sessions alls
JOIN tmp_alls_products tap ON alls.productsku=tap.productsku
WHERE (alls.productquantity::INT)>0  
    AND alls.city NOT IN ('not available in demo dataset','(not set)')
	AND city='Madrid'
	AND country='Spain'
--Output -->
-- | country  | city    | v2productname    | productquantity
-- | Spain    | Madrid  | Waze Dress Socks | 10

```

### **starting_with_questions - Question 5**
* Compare rows and countries in result set to the table
```sql
SELECT country FROM(
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM all_sessions
    WHERE totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country
    ORDER BY 
        totalrev DESC
)
SELECT 
    country, 
    totalrev,
    SUM(totalrev) OVER () sum_totalrev,
    ROUND((100*(totalrev/SUM(totalrev) OVER ())),2) AS percent_rev
FROM 
    q1_clean
WHERE country <> 'NULL'
GROUP BY 
    country, 
    totalrev
ORDER BY 
    totalrev DESC
)
--5 rows affected; 
--1. United States
--2. Israel
--3. Australia
--4. Canada
--5. Switzerland
```
```sql
SELECT 
    DISTINCT country
FROM all_sessions
WHERE (transactions::INT)>0  
    AND totaltransactionrevenue IS NOT NULL
	AND country NOT IN ('not available in demo dataset','(not set)')
--5 rows affected; 
--1. AustraliaUnited States
--2. CanadaIsrael
--3. Israel
--4. SwitzerlandCanada
--5. United StatesSwitzerland
```

* Compare totalrev from all_sessions table to result set

```SQL
SELECT 
    country, 
    ROUND((totaltransactionrevenue::NUMERIC/1000000),2) AS trans_rev
FROM 
    all_sessions
WHERE (transactions::INT)>0  
    AND totaltransactionrevenue IS NOT NULL
	AND country NOT IN ('not available in demo dataset','(not set)')
    AND country='Canada'
ORDER BY 
    country
--2 rows affected. trans_rev values are 67.99 and 82.16. Their sum is 150.15
```
```sql
SELECT country, totalrev FROM (
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM all_sessions
    WHERE totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country
    ORDER BY 
        totalrev DESC
)
SELECT 
    country, 
    totalrev,
    SUM(totalrev) OVER () sum_totalrev,
    ROUND((100*(totalrev/SUM(totalrev) OVER ())),2) AS percent_rev
FROM 
    q1_clean
WHERE country <> 'NULL'
GROUP BY 
    country, 
    totalrev
ORDER BY 
    totalrev DESC
)
WHERE country='Canada'
--1 row affected; shows Canada and totalrev of 150.15
```

* Compare sum of totalrev from all_sessions to result set
```sql
SELECT 
    ROUND(SUM(totaltransactionrevenue::NUMERIC/1000000),2) AS trans_rev
FROM 
    all_sessions
WHERE (transactions::INT)>0  
    AND totaltransactionrevenue IS NOT NULL
	AND country NOT IN ('not available in demo dataset','(not set)')
--14281.31
```
```sql
SELECT SUM(totalrev) FROM(
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM all_sessions
    WHERE totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country
    ORDER BY 
        totalrev DESC
)
SELECT 
    country, 
    totalrev,
    SUM(totalrev) OVER () sum_totalrev,
    ROUND((100*(totalrev/SUM(totalrev) OVER ())),2) AS percent_rev
FROM 
    q1_clean
WHERE country <> 'NULL'
GROUP BY 
    country, 
    totalrev
ORDER BY 
    totalrev DESC
)
--14281.31
```
* Manual check - For Canada, the result set shows a totalrev=150.15, and sum_totalrev=14281.31. These values have already been QA'd. 

    In the result set the percent_rev=1.05

    (150.15/14281.31)*100 = 1.05 (rounded)


* Compare rows and cities in result set to the table
```sql
SELECT DISTINCT ON (country, city)
	COUNT(*)
FROM 
    all_sessions
WHERE (transactions::INT)>0  
    AND totaltransactionrevenue IS NOT NULL
	AND city NOT IN ('not available in demo dataset','(not set)')
GROUP BY 
    country, 
    city;
--20 rows affected
```
```sql
SELECT COUNT(*) FROM (
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
        ELSE city
        END AS city,
    SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM 
        all_sessions
    WHERE totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country, 
        city
    ORDER BY 
        country, 
        city, 
        totalrev DESC
)
SELECT 
    country, 
    city, 
    totalrev,
    SUM(totalrev) OVER () sum_totalrev,
    ROUND((100*(totalrev/SUM(totalrev) OVER ())),2) AS percent_rev,
    DENSE_RANK() OVER (
        PARTITION BY country
        ORDER BY totalrev DESC) AS rev_rank
FROM 
    q1_clean
WHERE country <> 'NULL' 
    AND city <> 'NULL');
--Output --> 20
```

* Compare totalrev from all_sessions table to result set
```sql
SELECT 
    city, 
    ROUND((totaltransactionrevenue::NUMERIC/1000000),2) AS trans_rev
FROM 
    all_sessions
WHERE (transactions::INT)>0  
    AND totaltransactionrevenue IS NOT NULL
	AND city NOT IN ('not available in demo dataset','(not set)')
    AND country='Israel'
--> Output city=Tel Aviv-Yafo, trans_rev=602.00
```
```sql
SELECT city, totalrev FROM (
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
        ELSE city
        END AS city,
    SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM 
        all_sessions
    WHERE totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country, 
        city
    ORDER BY 
        country, 
        city, 
        totalrev DESC
)
SELECT 
    country, 
    city, 
    totalrev,
    SUM(totalrev) OVER () sum_totalrev,
    ROUND((100*(totalrev/SUM(totalrev) OVER ())),2) AS percent_rev,
    DENSE_RANK() OVER (
        PARTITION BY country
        ORDER BY totalrev DESC) AS rev_rank
FROM 
    q1_clean
WHERE country <> 'NULL' 
    AND city <> 'NULL'
)
WHERE country='Israel';
--Output --> city=Tel Aviv-Yafo, totrev=602.00
```

* Compare sum of totalrev from all_sessions to result set
```sql
SELECT 
    ROUND(SUM(totaltransactionrevenue::NUMERIC/1000000),2) AS trans_rev
FROM 
    all_sessions
WHERE (transactions::INT)>0  
    AND totaltransactionrevenue IS NOT NULL
	AND city NOT IN ('not available in demo dataset','(not set)')
--8188.75
```
```sql
SELECT SUM(totalrev) FROM(
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
        ELSE city
        END AS city,
    SUM(ROUND((totaltransactionrevenue::NUMERIC)/1000000,2)) AS totalrev
    FROM 
        all_sessions
    WHERE totaltransactionrevenue IS NOT NULL
    GROUP BY 
        country, 
        city
    ORDER BY 
        country, 
        city, 
        totalrev DESC
)
SELECT 
    country, 
    city, 
    totalrev,
    SUM(totalrev) OVER () sum_totalrev,
    ROUND((100*(totalrev/SUM(totalrev) OVER ())),2) AS percent_rev,
    DENSE_RANK() OVER (
        PARTITION BY country
        ORDER BY totalrev DESC) AS rev_rank
FROM 
    q1_clean
WHERE country <> 'NULL' 
    AND city <> 'NULL'
)
--8188.75
```

* Manual check - For Tel Aviv-Yafo, the result set shows a totalrev=602.00, and sum_totalrev=8188.75. These values have already been QA'd. 

    In the result set the percent_rev=7.35

    (602.00/8188.75)*100 = 7.35 (rounded)

### **starting_with_data - Question 1**
* Query #1 - Review full output
```sql
SELECT 
    DISTINCT fullvisitorid, 
	MAKE_INTERVAL(secs => (timeonsite::INT)) AS timeonsite,
	 ROUND((totaltransactionrevenue::NUMERIC)/1000000, 2) AS trans_rev
FROM 
    all_sessions
WHERE (transactions::INT) = 1
ORDER BY 
    timeonsite
--80 rows affected
--Row #1 shows timeonsite=00:01:23, transrev=200.00 (this matches calculated max and revenue for that transaction)
--Row #80 shows timeonsite=00:31:00, transrev=61.97 (this matches calculated max and revenue for that transaction)
```

* Query #2 & #3- Manual Check 
```sql
SELECT 
    DISTINCT fullvisitorid, 
	MAKE_INTERVAL(secs => (timeonsite::INT)) AS timeonsite,
	 ROUND((totaltransactionrevenue::NUMERIC)/1000000, 2) AS trans_rev
FROM 
    all_sessions
WHERE (transactions::INT) = 1
ORDER BY 
    trans_rev
--80 rows affected
--Rows 76-80 match query #2 result set for highest transaction revenues
--Rows 1-5 match query #3 result set for lowest transaction revneues
```

### **starting_with_data - Question 2**
* Compare result set count vs. querying the table
```sql
SELECT DISTINCT ON(country, city)
    country, 
    city, 
    pageviews
FROM 
all_sessions
WHERE pageviews IS NOT NULL  
AND country NOT IN ('(not set)')
AND city NOT IN ('not available in demo dataset','(not set)')
--282 rows affected
```
```sql
SELECT COUNT(*) FROM (
WITH q7_clean AS (
    SELECT DISTINCT ON (fullvisitorid, date(date), pageviews, country, city)
        fullvisitorid, 
        date(date), 
        pageviews, 
        totaltransactionrevenue, 
        transactions, 
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE        
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city
    FROM 
        all_sessions
    GROUP BY 
        fullvisitorid, 
        date(date), 
        pageviews, 
        totaltransactionrevenue, 
        transactions, 
        country, 
        city
    ORDER BY 
        fullvisitorid, 
        date(date)
),
clean_cc AS (
    SELECT 
        country, 
        city, 
        COUNT(pageviews) AS count_pageviews
    FROM 
        q7_clean
    WHERE country <> 'NULL'
        AND city <> 'NULL'
    GROUP BY 
        country, 
        city
    ORDER BY count_pageviews DESC
)
SELECT 
    country, 
    city, 
    count_pageviews,
    DENSE_RANK() OVER (
        ORDER BY count_pageviews DESC) AS viewcount_rank
FROM 
    clean_cc
GROUP BY 
    country, 
    city, 
    count_pageviews
ORDER BY viewcount_rank ASC
)
--Output --> 282
```
* Compare subset from table results to result set
```sql
SELECT COUNT(*) FROM (
    SELECT
        country, 
        city, 
        fullvisitorid, date(date),
        count(pageviews) AS count_pageviews
    FROM 
        all_sessions
    WHERE country NOT IN ('(not set)')
        AND city NOT IN ('not available in demo dataset','(not set)')
    GROUP BY 
        country, 
        city,
        fullvisitorid,
        date(date)
    ORDER BY count_pageviews DESC
)
WHERE country='India'
AND city='Bengaluru';
--Output --> 72
```
```sql
SELECT * FROM (
WITH q7_clean AS (
    SELECT DISTINCT ON (fullvisitorid, date(date), pageviews, country, city)
        fullvisitorid, 
        date(date), 
        pageviews, 
        totaltransactionrevenue, 
        transactions, 
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE        
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city
    FROM 
        all_sessions
    GROUP BY 
        fullvisitorid, 
        date(date), 
        pageviews, 
        totaltransactionrevenue, 
        transactions, 
        country, 
        city
    ORDER BY 
        fullvisitorid, 
        date(date)
),
clean_cc AS (
    SELECT 
        country, 
        city, 
        COUNT(pageviews) AS count_pageviews
    FROM 
        q7_clean
    WHERE country <> 'NULL'
        AND city <> 'NULL'
    GROUP BY 
        country, 
        city
    ORDER BY count_pageviews DESC
)
SELECT 
    country, 
    city, 
    count_pageviews,
    DENSE_RANK() OVER (
        ORDER BY count_pageviews DESC) AS viewcount_rank
FROM 
    clean_cc
GROUP BY 
    country, 
    city, 
    count_pageviews
ORDER BY viewcount_rank ASC
)
WHERE country='India'
AND city='Bengaluru'
--1 row affected. country=India, city=Bengaluru, count_pageviews=72, viewcount_rank=13
```

### **starting_with_data - Question 3**
* Check all_sessions for rows that meet criteria
```sql
SELECT 
    COUNT(DISTINCT fullvisitorid)
FROM all_sessions
WHERE (transactions::INT)=1
--Output --> 80
```
```sql
SELECT sum_total_ordering_visitors FROM (
(SELECT 
    DISTINCT(channelgrouping), 
    COUNT(DISTINCT fullvisitorid) AS count_ordering_visitors,
    SUM(COUNT(DISTINCT fullvisitorid)) OVER () AS sum_total_ordering_visitors,
    ROUND((100*(COUNT(DISTINCT fullvisitorid)/SUM(COUNT(DISTINCT fullvisitorid)) OVER ())),2) AS percent_channel
FROM 
    all_sessions
WHERE (transactions::INT) = 1
GROUP BY 
    channelgrouping
ORDER BY count_ordering_visitors DESC))
--Output --> 80 (4 rows of the same because of query structure)
```
* Check count of each channel if it was individually queried.
```sql
SELECT 
    (SELECT COUNT(DISTINCT fullvisitorid) AS count_ref_grp
    FROM all_sessions
    WHERE (transactions::INT)=1
        AND channelgrouping='Referral'), 
    (SELECT COUNT(DISTINCT fullvisitorid) AS count_dir_grp
    FROM all_sessions
    WHERE (transactions::INT)=1
        AND channelgrouping='Direct'),
    (SELECT COUNT(DISTINCT fullvisitorid) AS count_osearch_grp
    FROM all_sessions
    WHERE (transactions::INT)=1
        AND channelgrouping='Organic Search'),
    (SELECT COUNT(DISTINCT fullvisitorid) AS count_psearch_grp
        FROM all_sessions
    WHERE (transactions::INT)=1
    AND channelgrouping='Paid Search')
-- Output -->
-- count_ref_grp = 32 --> ((32/80)*100)=40
-- count_dir_grp = 23
-- count_osearch_grp=22
-- count_psearch_grp=3
-- 32+23+22+3=80
-- Results of this query align with answer result set
```
* Check percentage total of result set
```sql
SELECT SUM(percent_channel) FROM (
(SELECT 
    DISTINCT(channelgrouping), 
    COUNT(DISTINCT fullvisitorid) AS count_ordering_visitors,
    SUM(COUNT(DISTINCT fullvisitorid)) OVER () AS sum_total_ordering_visitors,
    ROUND((100*(COUNT(DISTINCT fullvisitorid)/SUM(COUNT(DISTINCT fullvisitorid)) OVER ())),2) AS percent_channel
FROM 
    all_sessions
WHERE (transactions::INT) = 1
GROUP BY 
    channelgrouping
ORDER BY count_ordering_visitors DESC))
--Output --> 100.00
```
