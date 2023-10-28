Answer the following questions and provide the SQL queries used to find the answer.

    
**Question 1: Which cities and countries have the highest level of transaction revenues on the site?**


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
	WHERE totaltransactionrevenue IS NOT NULL
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
    totalrev DESC;
--20 rows affected.
```

**Assumptions:**

* Column 'all_sessions.totaltransactionrevenue' represents the dollar value of a completed transaction.
* Rows where the value of the 'all_sessions.country' column is '(not set)' can be excluded as this information is needed for the purpose of classifying the transaction revenue.
* Rows where the values of the 'all_sessions.city' column are '(not set)' or 'not available in demo dataset' can be excluded as this information is needed for the purpose of classifying transaction revenue.



**Answer:**
The top 5 cities and countries with the highest level of transaction revenue are:

1. San Francisco, United States
1. Sunnyvale, United States
1. Atlanta, United States
1. Palo Alto, United States
1. Tel Aviv-Yafo, Israel

**Sample Output:**

![q1_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/942f2cb4-223b-472d-b1ee-0fdac386a131)


**Data Quality Concerns:**

* The same all_sessions.city values correspond to different countries. This isn't always an issue, e.g. London is a city in Canada and in the UK. 

    However, in this database there are some obvious issues e.g. New York is listed as city under the United States and Canada. So is Los Angeles.
    
    There are 34 cities that would require a closer look. One-off cleaning isn't the most efficient approach. The best way to correct this would be if there was data available with accurate city and country data for reference.

* The all_sessions table has 15,134 rows. Filtering down to rows with transaction revenue leaves 81 rows. Filtering down further to remove country and city values that are not proper names leaves 56 rows. **

**Question 2: What is the average number of products ordered from visitors in each city and country?**


**SQL Queries:**
```postgresql
WITH q2_clean AS (
	SELECT
		(CASE 
			WHEN country='(not set)' THEN 'NULL'
			ELSE country
		END) AS country,
		(CASE
			WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
			ELSE city
		END) AS city,
		AVG(total_ordered::INT) OVER (PARTITION BY country, city) AS avg_total_ordered
	FROM 
        all_sessions alls
	JOIN sales_by_sku sbs ON alls.productsku=sbs.productsku
	WHERE 
        (sbs.total_ordered::INT) > 0
	GROUP BY 
        country, 
        city, 
        total_ordered
)
SELECT 
    country, 
    city, 
    ROUND(MAX(avg_total_ordered),0) AS avg_total_ordered
FROM 
    q2_clean
WHERE country <> 'NULL' AND city <> 'NULL'
GROUP BY 
    country, 
    city
ORDER BY 
    country;
--206 rows affected.
```
**Assumptions:**
* Column sales_by_sku.total_ordered represents the number of products ordered
* Rows where the value of the 'all_sessions.country' column is '(not set)' can be excluded as this information is needed for the purpose of classifying the transaction revenue.
* Rows where the values of the 'all_sessions.city' column are '(not set)' or 'not available in demo dataset' can be excluded as this information is needed for the purpose of classifying transaction revenue.

**Answer:**

The city/country with the highest average number of products ordered, with a value of 319 are:
1. Brno, Czechia - avg_total_ordered = 319
1. Riyadh, Saudi Arabia - avg_total_ordered = 319

The city/country with the lowest average number of products ordered, with a value of 1 are:
1. Rosario, Argentina
1. Mountain View, Australia
1. Singapore, France
1. Villeneuve-d'Ascq
1. Vilnius, Lithuania
1. Oslo, Norway
1. Lahore, Pakistan
1. Piscataway Township, United States
1. Kansas City, United States
1. Greer, United States


**Sample Output:**

![q2_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/e96d7d19-e885-4f92-8979-98dcd4e08088)

**Data Quality Concerns:**
* The same all_sessions.city values correspond to different countries. This isn't always an issue, e.g. London is a city in Canada and in the UK.

    However, in this database there are some obvious issues. E.g. There's a row where city Singapore is under country France.

    There are 34 cities that would require a closer look. One-off cleaning isn't the most efficient approach. The best way to correct this would be if there was data available with accurate city and country data for reference.


**Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?**


SQL Queries:
```postgresql
WITH tmp_clean_cat AS (
SELECT DISTINCT(v2productcategory), main_category
FROM(
	SELECT DISTINCT(v2productcategory),
		  split_part(v2productcategory, '/', 1) AS main_category,
		  split_part(v2productcategory, '/', 2) AS second_category,
		  split_part(v2productcategory, '/', 3) AS third_category,
		  split_part(v2productcategory, '/', 4) AS fourth_category
	FROM all_sessions) cat_split
WHERE main_category NOT LIKE 'Home%'

UNION

SELECT DISTINCT(v2productcategory), second_category
FROM(
	SELECT DISTINCT(v2productcategory),
		  split_part(v2productcategory, '/', 1) AS main_category,
		  NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
		  split_part(v2productcategory, '/', 3) AS third_category,
		  split_part(v2productcategory, '/', 4) AS fourth_category
	FROM all_sessions) cat_split
WHERE second_category IS NOT NULL
	AND second_category NOT IN
		('Limited Supply', 'Shop by Brand', 'Brands', 'Nest','Men''s T-Shirts')

UNION

SELECT DISTINCT(v2productcategory), second_category
FROM(
	SELECT DISTINCT(v2productcategory),
		  split_part(v2productcategory, '/', 1) AS main_category,
		  NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
		  NULLIF(split_part(v2productcategory, '/', 3),'') AS third_category,
		  split_part(v2productcategory, '/', 4) AS fourth_category
	FROM all_sessions) cat_split
WHERE second_category IS NOT NULL
	AND third_category IS NULL
	AND second_category IN
		('Limited Supply', 'Shop by Brand', 'Brands')

UNION

SELECT DISTINCT(v2productcategory), third_category
FROM(
	SELECT DISTINCT(v2productcategory),
		  split_part(v2productcategory, '/', 1) AS main_category,
		  NULLIF(split_part(v2productcategory, '/', 2),'') AS second_category,
		  NULLIF(split_part(v2productcategory, '/', 3),'') AS third_category,
		  split_part(v2productcategory, '/', 4) AS fourth_category
	FROM all_sessions) cat_split
WHERE main_category LIKE 'Home%'
	AND second_category IN
		('Limited Supply', 'Shop by Brand', 'Brands', 'Nest')
	AND third_category IS NOT NULL
ORDER BY v2productcategory
),
main_group AS (
	SELECT DISTINCT ON( ao.country, ao.city, ao.productsku, ao.v2productcategory)ao.*, 
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
		FROM all_sessions
		WHERE (transactions::int)=1) ao
	LEFT JOIN tmp_clean_cat 
		ON ao.v2productcategory=tmp_clean_cat.v2productcategory
	WHERE country <> 'NULL'
	AND city <> 'NULL'
),
main_cat_count_tbl AS (
	SELECT country, 
		city, 
		main_category, 
		COUNT(main_category) AS main_cat_count,
		DENSE_RANK() OVER 
			(PARTITION BY country, city 
			 ORDER BY COUNT(main_category) DESC) AS cat_rank
	FROM main_group
	GROUP BY country, city, main_category
),
q3_final AS (
	SELECT country, city, main_category, main_cat_count, cat_rank
	FROM main_cat_count_tbl
	ORDER BY country, city, cat_rank)

SELECT main_category, count(main_category) AS count_rank_1
FROM q3_final
WHERE cat_rank=1
GROUP BY main_category
ORDER BY count_rank_1 DESC
```
**Assumptions**

**Answer:**

**Sample Output:**



**Question 4: What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?**


SQL Queries:



Answer:





**Question 5: Can we summarize the impact of revenue generated from each city/country?**

SQL Queries:



Answer:







