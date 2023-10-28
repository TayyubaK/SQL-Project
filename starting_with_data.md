### **Question 1: How long were people who made transactions on the site?**

**SQL Queries:**

Query #1 - Max time and how much was spent for that time, min time and how much was spent for that time, and average time.
```sql
WITH time_to_trans AS (
    SELECT 
        DISTINCT(fullvisitorid), 
        ROUND((totaltransactionrevenue::numeric)/1000000, 2) AS trans_rev,
        MAKE_INTERVAL(secs => (timeonsite::integer)) AS timeonsite
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
    ROUND((totaltransactionrevenue::numeric)/1000000, 2) AS trans_rev,
    MAKE_INTERVAL(secs => (timeonsite::integer)) AS timeonsite
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
    ROUND((totaltransactionrevenue::numeric)/1000000, 2) AS trans_rev,
    MAKE_INTERVAL(secs => (timeonsite::integer)) AS timeonsite
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

SQL Queries:

Answer:



Question 3: How did visitors who made transactions hear about the site?

SQL Queries:

Answer:

