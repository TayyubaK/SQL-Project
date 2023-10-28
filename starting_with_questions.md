Answer the following questions and provide the SQL queries used to find the answer.

    
### **Question 1: Which cities and countries have the highest level of transaction revenues on the site?**


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

* The all_sessions table has 15,134 rows. Filtering down to rows with transaction revenue leaves 81 rows. Filtering down further to remove country and city values that are not proper names leaves 56 rows.

### **Question 2: What is the average number of products ordered from visitors in each city and country?**


**SQL Queries:**
```sql
WITH q2_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city,
        AVG(total_ordered::INT) OVER (
            PARTITION BY country, city) AS avg_total_ordered
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
1. Brno, Czechia
1. Riyadh, Saudi Arabia

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


### **Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?**


**SQL Queries:**
```sql
WITH clean_cat AS (
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
        FROM 
            all_sessions) cat_split
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
        FROM 
            all_sessions) cat_split
    WHERE second_category IS NOT NULL
        AND second_category NOT IN
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
        FROM 
            all_sessions) cat_split
        WHERE second_category IS NOT NULL
            AND third_category IS NULL
            AND second_category IN
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
        FROM 
            all_sessions) cat_split
        WHERE main_category LIKE 'Home%'
            AND second_category IN
                ('Limited Supply', 'Shop by Brand', 'Brands', 'Nest')
            AND third_category IS NOT NULL
    ORDER BY v2productcategory
),
main_group AS (
    SELECT 
        DISTINCT ON( ao.country, ao.city, ao.productsku, ao.v2productcategory)ao.*, 
        clean_cat.main_category
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
    LEFT JOIN clean_cat ON ao.v2productcategory=clean_cat.v2productcategory
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
),
q3_final AS (
    SELECT 
        country, 
        city, 
        main_category, 
        main_cat_count, 
        cat_rank
    FROM 
        main_cat_count_tbl
    ORDER BY country, city, cat_rank
)
SELECT 
    main_category, 
    count(main_category) AS count_rank_1
FROM 
    q3_final
WHERE 
    cat_rank=1
GROUP BY 
    main_category
ORDER BY 
    count_rank_1 DESC;

--8 rows affected.
```
**Assumptions:**

* There are no NULL or blank values for the all_sessions.productsku column. There are no NULL or blank values for all_sessions.v2productcategory. Therefore, each row has a productsku and v2productcategory value.
* all_sessions.v2categoryname cleaned to extract category information past the 'home/'

**Answer:**

Nest-USA and Apparel are the top product categories by a wide margin. They each rank #1 for ten cities/countries.

And if the final query is modified to 'WHERE cat_rank=2', Nest-USA and Apparel also rank #2 for 2 other cities/countries.

**Sample Output:**

![q3_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/0f12b478-93e8-46b4-bbce-683da0bb97c7)



### **Question 4: What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?**


**SQL Queries:**

Create tmp_alls_products with 1:1 relationship between productsku and v2productname

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
Use tmp_alls_products and sales_by_sku to answer the question:
```sql
WITH sales_sku AS (
    SELECT 
        sbs.*, 
        tmp_alls_products.v2productname
    FROM 
        sales_by_sku sbs
    LEFT JOIN tmp_alls_products ON sbs.productsku=tmp_alls_products.productsku
    WHERE (total_ordered::INT) <> 0
        AND v2productname IS NOT NULL
),
country_city_order AS (
    SELECT 
        DISTINCT(ss.productsku),
        (ss.total_ordered::INT), 
        ss.v2productname, 
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        CASE
            WHEN city IN ('not available in demo dataset','(not set)') THEN 'NULL'
            ELSE city
        END AS city
    FROM 
        sales_sku ss
    JOIN all_sessions alls ON ss.productsku=alls.productsku 
    WHERE (alls.transactions::INT)=1
),
all_ranked AS (
    SELECT 
        country, 
        city, 
        productsku, 
        total_ordered, 
        v2productname,
        DENSE_RANK() OVER (
            PARTITION BY country, city 
            ORDER BY total_ordered DESC) AS ranking
    FROM 
        country_city_order
    WHERE country <> 'NULL'
        AND city <> 'NULL'
    ORDER BY 
        country, 
        city, 
        total_ordered DESC
)
SELECT *
FROM all_ranked
WHERE ranking=1
ORDER BY
    country, city ASC;

--14 rows affected.
```

**Assumptions:**

* productsku is a unique identifier for a product. 
* One product name can potentially have multiple productsku values as names may overlap, however each productsku should have only 1 name associated with it.
* Top-selling defined as most quantity ordered

**Answer:**

**Sample Output:**

![q4_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/fab6045b-dfa0-446e-add2-2eb9ec51fa14)

**Data Quality Concerns:**

* Using sales_by_sku table ultimately results in dropping 

### **Question 5: Can we summarize the impact of revenue generated from each city/country?**

**SQL Queries:**

Query #1 - Look at revenue at the country-level.
```sql
WITH q1_clean AS (
    SELECT
        CASE 
            WHEN country='(not set)' THEN 'NULL'
            ELSE country
        END AS country,
        SUM(ROUND((totaltransactionrevenue::numeric)/1000000,2)) AS totalrev
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
    totalrev DESC;

--5 rows affected
```

Query #2 - Look at city-level ranking

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
    AND city <> 'NULL';

--20 rows affected
```

**Assumptions:**
* Column 'all_sessions.totaltransactionrevenue' represents the dollar value of a completed transaction.

Query #1
* Rows where the value of the 'all_sessions.country' column is '(not set)' can be excluded as this information is needed for the purpose of classifying the transaction revenue.
* Rows where the values of the 'all_sessions.city' column are '(not set)' or 'not available in demo dataset' can **_included_** in the results as the purpose of query #1 is to look at the country-level. 

Query #2
* Rows where the value of the 'all_sessions.country' column is '(not set)' can be excluded as this information is needed for the purpose of classifying the transaction revenue.
* Rows where the values of the 'all_sessions.city' column are '(not set)' or 'not available in demo dataset' can be **_excluded_** as this information is needed for the purpose of classifying transaction revenue. 

    Note, this results in a revenue $6,092.56 difference between the country-level and city-level analysis. The entire amount belongs to the U.S. but since we cannot assign a city to it, the true total sum and percentage of U.S. city revenue may be understated.

**Answer:**
* The United States is the biggest market by a wide margin. Revenue from the U.S. makes up 92.11% of total revenue from all countries. The #2 ranked city is Israel accounting for 4.22%.
* Looking at the city-level revenue, even the #4 ranked city in the U.S. generates 7.42% of revenue, which is comparable to the #1 ranked city of Tel Aviv-Yafo, Israel.
* The U.S. and its cities are very important to the revenue generation on the site. Other markets have yet to grow significantly.



**Sample Output:**

Query #1

![q5_ans_query1](https://github.com/TayyubaK/SQL-Project/assets/143013434/5553404e-d2f3-4f9a-941b-c9a5368e989d)

Query #2

![q5_ans_query2](https://github.com/TayyubaK/SQL-Project/assets/143013434/646e3db4-8cff-47d5-9c7e-d06c25edfb51)

**Data Quality Concerns:**
* The same all_sessions.city values correspond to different countries. This isn't always an issue, e.g. London is a city in Canada and in the UK.

    However, in this database there are some obvious issues. E.g. There's a city named 'New York' under Canada. A judgement can't be made about whether the country is correct or the city.

    There are 34 cities that would require a closer look. One-off cleaning isn't the most efficient approach. The best way to correct this would be if there was data available with accurate city and country data for reference.





