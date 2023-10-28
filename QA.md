What are your risk areas? Identify and describe them.



QA Process:
Describe your QA process and include the SQL queries used to execute it.

* Identify columns to use, find:
    * nulls
    * duplicates
    * anomalies

**Question 1**

Check counts of country, city, and sum of transaction revenue.

**Count of distinct countries in result set should match distinct countries with transaction revenue values.**

Count of unique countries in result set: 5
```postgresql
SELECT COUNT(DISTINCT country) FROM ( WITH q1_clean AS (
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
```postgresql
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

```postgresql
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

```postgresql
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
```postgresql
SELECT SUM(totalrev) FROM
(WITH q1_clean AS (
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

```postgresql
SELECT 
    SUM(ROUND(totaltransactionrevenue::NUMERIC/1000000,2))
FROM 
    all_sessions
WHERE totaltransactionrevenue IS NOT NULL
	AND country NOT IN ('(not set)','not available in demo dataset')
	AND city NOT IN ('(not set)','not available in demo dataset')
--8188.75
```