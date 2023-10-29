What issues will you address by cleaning the data?


Queries:
Below, provide the SQL queries you used to clean your data.



### tmp_clean_cat

Table was created to clean all_sessions.v2productcategory column. The v2productcategory column had many instances of 'Home/' followed by other values. This didn't allow for proper grouping as each value was distinct. 

The temp table takes first (main), second, or third delimited value from the original v2productcategory value as the main_category. This allows for better grouping based on the main_category.

It also retains the unique v2productcategory values that are in all_sessions so that JOINs are possible.


Data Issues Remaining:

* Values such as '${escCatTitle}' and '(not set)' were not filtered out as they may have been needed to JOIN with table all_sessions on v2productcategory
* Some categories are not straight-cut. The delimiter '/' was used but some categories names are messier than others.

    For example, these v2productcategory values have the main_category values: 
    1. Electronics --> main_category = Electronics
    1. Accessories --> main_category = Accessories
    1. Drinkware   --> main_category = Drinkware
    1. Home/Accessories --> main_category = Accessories
    1. Home/Accessories/Drinkware --> main_category = Accessories
    1. Home/Electronics/ --> main_category = Electronics
    1. Home/Electronics/Accessories/Drinkware --> main_category = Electronics


   Should #5 be Accessories or Drinkware? Should #7 be Electronics, Accessories, or Drinkware?

**Sample Output:**

![tmp_clean_cat sample](https://github.com/TayyubaK/SQL-Project/assets/143013434/1afa660e-ed76-4d6b-95ef-fcbf6268de98)


```sql
CREATE TEMP TABLE tmp_clean_cat AS
SELECT 
    DISTINCT(v2productcategory), 
    main_category
FROM(
    SELECT 
        DISTINCT(v2productcategory),
        split_part(v2productcategory, '/', 1) AS main_category,
        split_part(v2productcategory, '/', 2) AS second_category,
        split_part(v2productcategory, '/', 3) AS third_category,
        split_part(v2productcategory, '/', 4) AS fourth_category
    FROM all_sessions) cat_split
WHERE main_category NOT LIKE 'Home%'

UNION

SELECT 
    DISTINCT(v2productcategory), 
    second_category
FROM(
    SELECT 
        DISTINCT(v2productcategory),
        split_part(v2productcategory, '/', 1) AS main_category,
        NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
        split_part(v2productcategory, '/', 3) AS third_category,
        split_part(v2productcategory, '/', 4) AS fourth_category
    FROM all_sessions) cat_split
WHERE second_category IS NOT NULL AND second_category NOT IN
    ('Limited Supply', 'Shop by Brand', 'Brands', 'Nest','Men''s T-Shirts')

UNION

SELECT 
    DISTINCT(v2productcategory), 
    second_category
FROM(
    SELECT 
        DISTINCT(v2productcategory),
        split_part(v2productcategory, '/', 1) AS main_category,
        NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
        NULLIF(split_part(v2productcategory, '/', 3),'') AS third_category,
        split_part(v2productcategory, '/', 4) AS fourth_category
    FROM all_sessions) cat_split
WHERE second_category IS NOT NULL
    AND third_category IS NULL AND second_category IN
        ('Limited Supply', 'Shop by Brand', 'Brands')

UNION

SELECT 
    DISTINCT(v2productcategory), 
    third_category
FROM(
    SELECT 
        DISTINCT(v2productcategory),
        split_part(v2productcategory, '/', 1) AS main_category,
        NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
        NULLIF(split_part(v2productcategory, '/', 3),'') AS third_category,
        split_part(v2productcategory, '/', 4) AS fourth_category
    FROM all_sessions) cat_split
WHERE main_category LIKE 'Home%' AND second_category IN
    ('Limited Supply', 'Shop by Brand', 'Brands', 'Nest')
    AND third_category IS NOT NULL
ORDER BY v2productcategory
--74 rows affected
```


### tmp_alls_products




**Question 1**

**Data cleaning:** 

In the q1_clean CTE:
* CASE expression used to update '(not set)' values for the country column to 'NULL'.
* CASE expression used to update '(not set)' and 'not available in demo dataset' values for the city column to 'NULL'.
* Values for the totaltransactionrevenue column were: 
    * updated to data type numberic from character varying data type.
    * divided by 1,000,000 because unit cost in the dataset needed to be divided by 1,000,000, so related values needed the same consideration.
    * rounded to 2 decimal places since the column represents a dollar value.

In the query:
* NULL country and city values were removed

**SQL Queries:**
```sql
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
    totalrev DESC

```

**Data Quality Issues**
* Identify columns to use, find:
    * nulls
    * duplicates
    * anomalies

**Question 1**

| Table Name    | Column        | Risk Areas  |
| ------------- | ------------- |-------------|
| all_sessions  | country       | <li> similar names <li> nulls            |
| all_sessions  | city          | <li> similar names <li> nulls            | 
| all_sessions  | totaltransactionrevenue | <li> duplicates <li> data type | 

* Check country names for issues
```postgresql
SELECT 
    DISTINCT(country) 
FROM 
    all_sessions
ORDER BY 
    country
```
* The same all_sessions.city values correspond to different countries. This isn't always an issue, e.g. London is a city in Canada and in the UK. 

    However, in this database there are some obvious issues e.g. New York is listed as city under the United States and Canada. So is Los Angeles.

    There are 34 cities that would require a closer look. One-off cleaning isn't the most efficient approach. The best way to correct this would be if there was data available with accurate city and country data for reference.
```sql
WITH city_issues AS (
	SELECT 
        DISTINCT(city)
	FROM 
        all_sessions
	WHERE 
        city NOT IN ('(not set)', 'not available in demo dataset')
	GROUP BY 
        city
	HAVING 
        COUNT(DISTINCT country)>1
	ORDER BY 
        city
)
SELECT 
    DISTINCT(alls.country), 
    city_issues.city
FROM 
    city_issues
JOIN all_sessions alls ON city_issues.city=alls.city
ORDER BY 
    city

--34 rows affected.
```

**Question 3**

Created a temp table to clean up category names
```sql
CREATE TEMP TABLE tmp_clean_cat AS
SELECT 
    DISTINCT(v2productcategory), 
    main_category
FROM(
	SELECT 
        DISTINCT(v2productcategory),
		split_part(v2productcategory, '/', 1) AS main_category,
		split_part(v2productcategory, '/', 2) AS second_category,
		split_part(v2productcategory, '/', 3) AS third_category,
		split_part(v2productcategory, '/', 4) AS fourth_category
    FROM all_sessions) cat_split
WHERE main_category NOT LIKE 'Home%'

UNION

SELECT 
    DISTINCT(v2productcategory), 
    second_category
FROM(
    SELECT 
        DISTINCT(v2productcategory),
		split_part(v2productcategory, '/', 1) AS main_category,
		NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
		split_part(v2productcategory, '/', 3) AS third_category,
		split_part(v2productcategory, '/', 4) AS fourth_category
    FROM all_sessions) cat_split
WHERE second_category IS NOT NULL AND second_category NOT IN
			('Limited Supply', 'Shop by Brand', 'Brands', 'Nest','Men''s T-Shirts')

UNION

SELECT 
    DISTINCT(v2productcategory), 
    second_category
FROM(
	SELECT 
        DISTINCT(v2productcategory),
		split_part(v2productcategory, '/', 1) AS main_category,
		NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
		NULLIF(split_part(v2productcategory, '/', 3),'') AS third_category,
		split_part(v2productcategory, '/', 4) AS fourth_category
	FROM all_sessions) cat_split
WHERE second_category IS NOT NULL
		AND third_category IS NULL AND second_category IN
			('Limited Supply', 'Shop by Brand', 'Brands')

UNION

SELECT 
    DISTINCT(v2productcategory), 
    third_category
FROM(
	SELECT 
        DISTINCT(v2productcategory),
		split_part(v2productcategory, '/', 1) AS main_category,
		NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
		NULLIF(split_part(v2productcategory, '/', 3),'') AS third_category,
		split_part(v2productcategory, '/', 4) AS fourth_category
	FROM all_sessions) cat_split
WHERE main_category LIKE 'Home%' AND second_category IN
			('Limited Supply', 'Shop by Brand', 'Brands', 'Nest')
		AND third_category IS NOT NULL
ORDER BY v2productcategory
--74 rows affected
```