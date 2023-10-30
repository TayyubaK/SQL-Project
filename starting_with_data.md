### **Question 1: How long were people who made transactions on the site?**

**SQL Queries:**

Query #1 - Max time and how much was spent for that time, min time and how much was spent for that time, and average time.
```sql
WITH time_to_trans AS (
    SELECT 
        DISTINCT(fullvisitorid), 
        ROUND((totaltransactionrevenue::NUMERIC)/1000000, 2) AS trans_rev,
        MAKE_INTERVAL(secs => (timeonsite::INT)) AS timeonsite
    FROM 
        all_sessions
    WHERE (transactions::INT) = 1
    ORDER BY fullvisitorid, trans_rev DESC
)
SELECT 
    MAX(timeonsite) AS max_time,
    (SELECT trans_rev FROM time_to_trans WHERE timeonsite='00:31:00') AS max_trans_value,
    MIN(timeonsite) AS min_time, 
    (SELECT trans_rev FROM time_to_trans WHERE timeonsite='00:01:23') AS min_trans_value,
    AVG(timeonsite) AS avg_seconds
FROM time_to_trans;
```
![q6_ans_query1](https://github.com/TayyubaK/SQL-Project/assets/143013434/8224b545-71d7-4594-9191-ac548026f88e)

Query #2 - Time on site for the 5 highest revenue transactions
```sql
SELECT 
    DISTINCT(fullvisitorid), 
    ROUND((totaltransactionrevenue::NUMERIC)/1000000, 2) AS trans_rev,
    MAKE_INTERVAL(secs => (timeonsite::INT)) AS timeonsite
FROM 
    all_sessions
WHERE (transactions::INT) = 1
ORDER BY trans_rev DESC
LIMIT 5;
```

![q6_ans_query2](https://github.com/TayyubaK/SQL-Project/assets/143013434/89661b6b-f449-4b49-b4f2-d97ae85bd14f)

Query #3 - Time on site for the 5 lowest revenue transactions

```sql
SELECT 
    DISTINCT(fullvisitorid), 
    ROUND((totaltransactionrevenue::NUMERIC)/1000000, 2) AS trans_rev,
    MAKE_INTERVAL(secs => (timeonsite::INT)) AS timeonsite
FROM 
    all_sessions
WHERE (transactions::INT) = 1
ORDER BY trans_rev ASC
LIMIT 5;
```

![q6_ans_query3](https://github.com/TayyubaK/SQL-Project/assets/143013434/93288373-256b-4550-ac17-3da9e0725f33)

**Assumptions:**

* all_sessions.timeonsite represents time spent on the site by a visitor 

**Answer:** 

Of the people who made transactions, the maximum time spent on the site was 31 minutes and the minimum time spent was 1m 23s. The average time on the site was 9m 29s. 

Being on the site longer does not correspond with greater spending. The visitor with the highest transaction revenue was on the site for 4m 18s, while the visitor with the lowest transaction revenue was on the site for 4m 38s. 

### **Question 2: Which city, country do the most views come from (regardless of transactions)? Does it line up with the highest revenue places?**

**SQL Queries:**
```sql
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
ORDER BY viewcount_rank ASC;
--282 rows affected.
```
**Assumptions:**
* Rows where the value of the 'all_sessions.country' column is '(not set)' can be excluded as this information is needed for the purpose of classifying page views.
* Rows where the values of the 'all_sessions.city' column are '(not set)' or 'not available in demo dataset' can be excluded as this information is needed for the purpose of classifying page views.
* Views is defined as a page was visited. This doesn't consider how many pages were viewed; only considers that a view record was created by a visit. 

**Answer:**

The most views for the site are from U.S. cities. This is generally in-line with where most revenue is generated (see question 5, starting_with_questions.md). 

The ranking differs between transaction revenue and page views. For example, San Franciso, U.S. ranks #1 for revenue but it is ranked #3 for page views. It could be of interest for further analysis.

**Sample Output:**

![q7_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/ea2252d7-8926-4174-978f-4a6dfbc7d30d)


### **Question 3: How did visitors who made transactions hear about the site?**

**SQL Queries:**
```sql
SELECT 
    DISTINCT(channelgrouping), 
    COUNT(DISTINCT fullvisitorid) AS count_ordering_visitors,
    SUM(COUNT(DISTINCT fullvisitorid)) OVER () AS sum_total_ordering_visitors,
    ROUND((100*(COUNT(DISTINCT fullvisitorid)/SUM(COUNT(DISTINCT fullvisitorid)) OVER ())),2) AS percent_channel
FROM 
    all_sessions
WHERE (transactions::INT) = 1
GROUP BY 
    channelgrouping
ORDER BY count_ordering_visitors DESC

--4 rows affected
```
**Assumptions**

* all_sessions.fullvisitorid is a unique identifier for each visitor
* all_sessions.visitid is an identifier for each visit session
* all_sessions.channelgrouping captures how visitors heard about the site
* all_sessions.transactions=1 indicates when a transaction has been made
* 'Paid Search' represents when company pay for links to their site to be suggested first

**Answer:**

* Most visitors, at 40%, belong to the 'Referral' channel. 
* The 'Direct' and 'Organic Search' channels are comparable at 28.75% and 27.50% , respectively. 
* 'Paid Search' brings in only 3.75% of visitors. This could indicate an area to evaluate in terms of the cost of a paid search vs. the payoff from it.

**Sample Output:**

![q8_ans](https://github.com/TayyubaK/SQL-Project/assets/143013434/a483813d-ab93-432a-9cea-c00b47341287)

