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
--Using tmp_alls_products which was created for data cleaning (see cleaningdata.md)

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
**Assumptions:**
* Rows where the value of the 'all_sessions.country' column is '(not set)' can be excluded as this information is needed for the purpose of classifying the transaction revenue.
* Rows where the values of the 'all_sessions.city' column are '(not set)' or 'not available in demo dataset' can be excluded as this information is needed for the purpose of classifying transaction revenue.

**Answer:**

* The average number of products ordered by visitors in each city and country ranges between 1 to 5.50. 


**Sample Output:**

![q2_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/edc98a4d-39da-4801-8340-544283fb38c6)


**Data Quality Concerns:**
* The same all_sessions.city values correspond to different countries. This isn't always an issue, e.g. London is a city in Canada and in the UK.

    However, in this database there are some obvious issues. E.g. There's a row where city Singapore is under country France.

    There are 34 cities that would require a closer look. One-off cleaning isn't the most efficient approach. The best way to correct this would be if there was data available with accurate city and country data for reference.


### **Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?**


**SQL Queries:**
```sql
--Using tmp_clean_cat which was created for data cleaning (see cleaningdata.md)

WITH main_group AS (
    SELECT 
        DISTINCT ON( ao.country, ao.city, ao.productsku, ao.v2productcategory)ao.*, 
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

Use tmp_alls_products and all_sessions  tables to answer the question:
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

**Assumptions:**

* Top-selling defined as most quantity ordered
* Rows where the value of the 'all_sessions.country' column is '(not set)' can be excluded as this information is needed for the purpose of classifying products.
* Rows where the values of the 'all_sessions.city' column are '(not set)' or 'not available in demo dataset' can be excluded as this information is needed for the purpose of classifying products.

**Answer:**

* Nest products appear to be quite popular.
* Results are in-line with the top categories (question 3): Nest-USA and apparel

**Sample Output:**

![q4_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/5e3b6187-93e5-4561-bc2c-4f55b6b22aa4)

**Data Quality Concerns:**



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
* The United States is the biggest market by a wide margin. Revenue from the U.S. makes up 92.11% of total revenue from all countries. The #2 ranked country is Israel accounting for 4.22%.
* Looking at the city-level revenue, even the #4 ranked city in the U.S. generates 7.42% of revenue, which is slightly higher than the #1 ranked city of Tel Aviv-Yafo, Israel with 7.35%.
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





