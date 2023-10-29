### **Risk Areas**
What are your risk areas? Identify and describe them.

* Columns with blank/near empty, non-conforming, and/or duplicate values, for example:
        
    * all_sessions.totaltransactionrevenue has only 81 rows with a value out of 15,134 rows
    * all_sessions.country has no nulls but some countries are listed as '(not set)'
    * all_sessions.productsku has duplicates in the column


* All data loaded as character varying data type. Correct data type must be cast for some functions.
* With CTEs and subqueries each part has to be checked to ensure each piece is working as intended.

### **QA Process**

Describe your QA process and include the SQL queries used to execute it.
* Compare counts and results from query against individual query results of the table(s)
* Run query piece-by-piece and evaluate the output

    * All columns appear as expected and with the correct data type
    * Row counts are reasonable and match to the change queried
    * Calculations are accurate
* Visual inspection, manual calculation/checks

### tmp_alls_products - QA
This temp table was needed because all_sessions.productsku (unique identifier for a product) had more than 1 name associated with it.

Check that the unique count of productsku values matches between all_sessions and the temp table.
```sql
SELECT COUNT(DISTINCT productsku)
FROM all_sessions
--Output -> 536

SELECT COUNT(DISTINCT productsku)
FROM tmp_alls_products
--Output -> 536

SELECT COUNT(productsku)
FROM tmp_alls_products
--Output -> 536; temp table has only 1 row for each productsku

SELECT 
    DISTINCT productsku
FROM 
    all_sessions
GROUP BY 
    productsku
HAVING 
    count(DISTINCT(v2productname))>1
ORDER BY productsku
--74 rows; all_sessions had 74 productsku values with different v2productname values

SELECT 
    DISTINCT productsku
FROM 
    tmp_alls_products
GROUP BY 
    productsku
HAVING 
    count(DISTINCT(v2productname))>1
ORDER BY productsku
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
    productsku, v2productname
-- 2 rows affected; 'Google Men's Pullover Hoodie' name had a count of 1, and 'Google Men's Pullover Hoodie Grey' had a count of 3.

--tmp_alls_products should only have the name with the higher count in all_sessions
SELECT 
    productsku, 
    v2productname, 
    COUNT(v2productname)
FROM 
    tmp_alls_products
WHERE 
    productsku='9182593'
GROUP BY 
    productsku, v2productname
--1 row affected; only 'Google Men's Pullover Hoodie Grey' is in the tmp table
```


### Question 1 - QA

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
)
--5
```
Count of unique countries in all_sessions table with transaction revenue: 5
```sql
SELECT 
    COUNT(DISTINCT country)
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL
--5

--The Q1 result set excludes '(not set)' values for country. Check that the count of 5 doesn't include '(not set)':
SELECT
    DISTINCT country
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL

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
)
--19
```
Count of unique cities in all_sessions table with transaction revenue: 19

```sql
SELECT 
    COUNT(DISTINCT city)
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL 
    AND city NOT IN ('(not set)','not available in demo dataset')
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
        SUM(ROUND((totaltransactionrevenue::numeric)/1000000,2)) AS totalrev
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
    totalrev DESC)
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
    AND city NOT IN ('(not set)','not available in demo dataset')
--8188.75
```

### Question 2 - QA

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

### Question 3 - QA