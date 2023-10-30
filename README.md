# Final-Project-Transforming-and-Analyzing-Data-with-SQL

## Project/Goals
This purpose of this project was to analyze a set of 5 tables related to an e-commerce site to determine relationships and extract information. 

The goals of the project were:

* Review and select necessary data
* Use SQL queries to:

    * clean the data for use without major alteration to the source data
        
        * cleaning_data.md 
    * extract necessary information from the tables 
        * starting_with_questions.md
        * starting_with_data.md
    * QA the result set
        * QA.md

## Process
* Loaded .csv files with all columns set to 'character varying'
* Reviewed each table for number of columns and names
* Reviewed each column to determine 
    
    * null values
    * most appropriate data type
    * max & min values for columns best suited to be integer/numeric
    * distinct values for columns with text
* Selected columns to use for queries
    * Used CTEs, CAST and CASE to clean within queries
    * Created temp tables to clean data that required more extensive cleaning (see cleaning_data.md) 
        * tmp_alls_products - clean source for product SKU and product names
        * tmp_clean_cats - clean source for main product categories
* Reviewed query results

## Results
The data tables provide information on an e-commerce website which sells a wide varierty of products from electronics, to apparel, to office equipment etc. Visitors from many U.S. cities drive a large part of the revenue generation. Nest and apparel products seem to be large favourites for shoppers. 

Tables had data covering transactions/visitor information, analytics, and product focused details. For the scope/timeline of this project I stayed mostly within the 'all_sessions' and 'analytics' tables to extract country, city, revenue, and preferred product information.

Overview of tables:
* all_sessions (32 columns, 15,134 rows)
    * transactions level information for visitors and products (transactions, revenue etc.)
    * visitor information (e.g. unique id for visitors, date of visit, country, city)
    * some analytics
        * channel the visitor belongs to
        * when they visited, for how long, and page views
    * several nearly empty columns (e.g. productrefundamount) 

* analytics (14 columns, 4,301,122 rows)
    * information about unique visitors, their visit date time, page views.
    * some columns have no nulls 
* products (7 columns, 1,092 rows)
    * product-level information like SKU, name, sentimentscore, stock etc.
* sales_by_sku (2 columns, 462 rows)
    * Basic sales info for products: productsku and amount ordered
* sales_report (8 columns, 454 rows)
    * Bridges products and sales_by_sku to a degree 

I also created two temp tables:
* tmp_alls_products
    * Contains one name for each productsku, using the 'all_sessions' table as the main source
* tmp_clean_cats
    * Extracts 1 main category for each distinct category in the 'all_sessions' table. This allowed for greater insight and better grouping of product categories.

## Challenges 
* Many columns were not clear on the basis of their names and data alone. A SME would have been helpful or some additional context about the source of the data.
* Tables have large gaps in data (whole columns of nulls or nearly all nulls). This can affect the robustness of the results.
* Timeline to deliver results created a need to evaluate the scope/effort of different approaches to data cleaning. This affects the quality and extent of cleaning that can be done.  

## Future Goals

* Create a data dictionary to define columns and their characteristics
* Separate columns into their own tables (and normalize). For example, the 'all_sessions' has 32 columns covering a wide range.
    * Create better connections between tables. Many columns interfere with the relationship of other columns. 
    * Create a copy of the database for more deep cleaning
* Organize the project files based on questions addressed and cover analysis, data cleaning, question querying, and QA in the same file. Rather than separate files for questions, data cleaning, and QA.
    * A section to cover analysis results, table reviews, and why certain columns are used could be insightful.
* Do more exploratory querying with the 'analytics' and 'products' tables
* Re-do QA approach by using temp tables to store question result sets. Then QA queries can performed using the temp tables. It would make the QA.md file shorter. 