What issues will you address by cleaning the data?


Queries:
Below, provide the SQL queries you used to clean your data.

* Identify columns to use, then find:
    * nulls
    * duplicates
    * anomalies

* All data was loaded as character varying. For queries, CAST to the appropriate data type for use. 


### tmp_clean_cat

Table was created to clean all_sessions.v2productcategory column. The v2productcategory column had many instances of 'Home/' followed by other values. This didn't allow for proper grouping as each value was distinct. 

The temp table takes first (main), second, or third delimited value from the original v2productcategory value as the main_category. This allows for better grouping based on the main_category.

It also retains the unique v2productcategory values that are in all_sessions so that JOINs are possible.


**Data Quality Concerns:**

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

Temp table was created to clean all_sessions.productsku and all_sessions.v2product name columns. Product SKU is a unique product identifier. Several productsku values may have the same name (for a common product), but a productsku cannot have multiple names. 

The all_sessions table had 74 unique productsku values where mutliple names existed.

Created tmp_alls_products for 1:1 relationship between productsku and v2productname

**Data Quality Concerns:**

* Where multiple names existed for a productsku, the approach was to take the name with the highest count. This may not be the best/correct choice. Product names are also columns in tables products and sales_report, and this choice may not align with those columns. 

```sql
--Step 1: Add productsku and v2productname values that already have a 1:1 relationship.
CREATE TEMP TABLE tmp_alls_products AS
SELECT 
    DISTINCT(dup.psku) AS productsku, 
    alls.v2productname
FROM (	
    SELECT 
        DISTINCT(productsku) AS psku
    FROM 
        all_sessions
    GROUP BY 
        productsku
    HAVING 
        count(DISTINCT(v2productname))=1
    ORDER BY 
        productsku) dup
JOIN all_sessions alls ON alls.productsku=dup.psku
GROUP BY 
    dup.psku,
    alls.v2productname
ORDER BY 
dup.psku;

--462 rows

--Step 2: Find all productsku values with multiple v2productname values. 
--Count the number of times each name appears and rank according to frequency, keep only rank #1.
SELECT *
FROM (
    SELECT 
        DISTINCT(dup.psku) AS productsku, 
        alls.v2productname, 
        COUNT(v2productname) AS name_count,
        DENSE_RANK() OVER (
            PARTITION BY dup.psku
            ORDER BY COUNT(v2productname) DESC) AS name_rank
    FROM (	
        SELECT 
            DISTINCT(productsku) AS psku
        FROM 
            all_sessions
        GROUP BY 
            productsku
        HAVING 
            count(DISTINCT(v2productname))>1
        ORDER BY productsku) dup
    JOIN all_sessions alls ON alls.productsku=dup.psku
    GROUP BY 
        dup.psku,
        alls.v2productname
    ORDER BY 
        dup.psku) rank
WHERE name_rank=1;

--76 rows (2 products are tied for rank #1)

--Step 3: Clean tied product names and add final list to existing tmp_alls_products table
INSERT INTO tmp_alls_products (productsku, v2productname)
SELECT DISTINCT(productsku), 
	CASE
		WHEN v2productname = 'Google Women''s Tee Purple' 
			THEN 'Google Women''s Long Sleeve Tee Lavender'
		WHEN v2productname = 'Google Four Color EDC Flashlight'
			THEN 'Google Flashlight'
		ELSE v2productname
	END
FROM (
	SELECT * 
	FROM (
		SELECT DISTINCT(dup.psku) AS productsku, 
			alls.v2productname, 
			COUNT(v2productname) AS name_count,
			DENSE_RANK() OVER (PARTITION BY dup.psku
							   ORDER BY COUNT(v2productname) DESC) AS name_rank
		FROM (	
			SELECT DISTINCT(productsku) AS psku
			FROM all_sessions
			GROUP BY productsku
			HAVING count(DISTINCT(v2productname))>1
			ORDER BY productsku) dup
		JOIN all_sessions alls ON alls.productsku=dup.psku
		GROUP BY dup.psku,alls.v2productname
		ORDER BY dup.psku) rank
WHERE name_rank=1) rank_dup;

--Resulting tmp_alls_products table has 536 rows in total.
```



### **starting_with_questions - Question 1**

**In CTE 'q1_clean':**
* CASE expression used to update '(not set)' values for the country column to 'NULL'.
* CASE expression used to update '(not set)' and 'not available in demo dataset' values for the city column to 'NULL'.
* Values for the totaltransactionrevenue column were: 
    * updated to data type numberic from character varying data type.
    * divided by 1,000,000 because unit cost in the dataset needed to be divided by 1,000,000, so related values needed the same consideration.
    * rounded to 2 decimal places since the column represents a dollar value.

**In the SELECT query:**
* NULL country and city values were removed

**Data Quality Concerns:**

* As some cities do not seem to be assigned to the correct country, e.g. New York under Canada. However, without a "true" data source of country/city data to reference, these can't be corrected. In the grouping, the revenue generated by New York, Canada is attributed to country Canada instead of New York, United States. 

    Also it's not possible to determine if the transaction revenue belongs to the country or the city. 

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
WHERE country <> 'NULL' AND city <> 'NULL'
GROUP BY 
    country, 
    city, 
    totalrev
ORDER BY 
    totalrev DESC;
```

* Check country names for issues
```sql
SELECT 
    DISTINCT(country) 
FROM 
    all_sessions
ORDER BY 
    country
--No issues found
```
* Check city names for issues

    The same all_sessions.city values correspond to different countries. This isn't always an issue, e.g. London is a city in Canada and in the UK. 

    However, in this database there are some obvious issues e.g. New York is listed as city under the United States and Canada. So is Los Angeles.

    There are 34 cities that would require a closer look. One-off cleaning isn't the most efficient approach. The best way to correct this would be if there was data available with accurate city and country data for reference.
```sql
WITH city_issues AS (
    SELECT 
        DISTINCT(city)
    FROM 
        all_sessions
    WHERE city NOT IN ('(not set)', 'not available in demo dataset')
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

### **starting_with_questions - Question 2**
Used tmp_alls_products (data cleaning for it is described above)

**In CTE 'unit_data':**
* DISTINCT ON removes rows that have the same values for all columns in the grouping, keeping only one representative. This helps to eliminate some duplication.
* CASE expression used to update '(not set)' values for the country column to 'NULL'.
* CASE expression used to update '(not set)' and 'not available in demo dataset' values for the city column to 'NULL'.
* WHERE conditions
    * transactions column CAST from character varying to integer data type and filtered for value of 1
    * totaltransactionrevenue column CAST from character varying to integer data type and filtered to remove NULL
    * units_sold column CAST from character varying to integer data type and filtered to remove NULL

End result are rows with no empty nulls. The 'NULL' values remaining are text.

**In CTE 'q2_clean':**

* 'NULL' text values are removed for country and city columns

Result set now has no anomalies for country and city columns.

**In the SELECT query:****
* units_sold column CAST from character varying to integer data type so that average can be calculated and value can be rounded to 2 decimal places

**Data Quality Concerns:**

* Although some duplicates are removed with the DISTINCT ON, there are some rows remaining where a column differs from the whole but the transaction otherwise seems to be a duplicate. The dataset needs a better identifier for unique transactions for robust removal of duplicates.


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
),
q2_clean AS (
	SELECT * 
    FROM 
        unit_data
	WHERE country <> 'NULL'
	    AND city <> 'NULL'
)
SELECT 
    DISTINCT ON (country, city)
    country,
	city,
	ROUND(AVG(units_sold::INT) OVER (
		PARTITION BY country, city),2) AS avg_total_ordered
FROM 
    q2_clean
GROUP BY 
    country, 
    city,
    units_sold;

--28 rows affected
```
### **starting_with_questions - Question 3**
Used tmp_clean_cat (data cleaning for it is described above)

**In CTE 'main_group':**
* CASE expression used to update '(not set)' values for the country column to 'NULL'.
* CASE expression used to update '(not set)' and 'not available in demo dataset' values for the city column to 'NULL'.
* WHERE conditions
    * transactions column CAST from character varying to integer data type and filtered for value of 1
    * 'NULL' text values are removed for country and city columns (in the outer query)

End result are rows with no blanks and no anomalies for country and city columns. Subsequent queries are not for data cleaning.

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
        WHERE (transactions::int)=1) ao
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
    cat_rank=1
GROUP BY 
    main_category
ORDER BY 
    count_rank_1 DESC;
--8 rows affected.
```

### **starting_with_questions - Question 4**
**In CTE 'main_group':**
* CASE expression used to update '(not set)' values for the country column to 'NULL'.
* CASE expression used to update '(not set)' and 'not available in demo dataset' values for the city column to 'NULL'.
* productquantity CAST from charactering varying to integer data type and filtered for values greater than 0

* WHERE conditions
    * transactions column CAST from character varying to integer data type and filtered for value of 1
    * 'NULL' text values are removed for country and city columns (in the outer query)

End result are rows with no blanks and no anomalies for country and city columns. Subsequent queries are not for data cleaning.
```sql
WITH main_group AS (
    SELECT 
        DISTINCT ON( 
            ao.country, 
            ao.city, 
            ao.productsku, 
            ao.v2productcategory, 
            ao.productquantity)ao.*,
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
),
rank_tbl AS (
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
SELECT * 
FROM 
    rank_tbl 
WHERE top_prod_rank=1;
```

**starting_with_questions - Question 5**

**starting_with_data - Question 1**

**starting_with_data - Question 2**

**starting_with_data - Question 3**
