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
        * starting_with _data.md
    * QA queries and the result set
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
    * Created temp tables to clean data that required more extensive cleaning
* Reviewed query results

## Results
(fill in what you discovered this data could tell you and how you used the data to answer those questions)

The data tables provide information on an e-commerce website. 

The site sells a wide varierty of products from electronics, to apparel, to office etc. Visitors from many U.S. cities drive a large part of the revenue generation. Nest and apparel products seem to be large favourites for shoppers. 

* all_sessions
    * transactions level information for visitors and products
    * visitor information (e.g. unique id for visitors, country, city)
    * some analytics
        * channel the visitor belongs to
        * when they visited, for how long, and page views



## Challenges 
* Many columns were not clear on the basis of their names and data alone. A SME would have been helpful or some additional context about the source of the data.
* Tables have large gaps in data (whole columns of nulls, columns of nearly all nulls). This can affect the robustness of the results.
* Timeline to deliver results created a need to evaluate the scope/effort of different approaches to data cleaning. This affects the quality and extent of cleaning that can be done.  

## Future Goals

* Create a data dictionary to define columns and their characteristics
* Separate columns into their own tables (and normalize). For example, the 'all_sessions' has 32 columns. Many columns interfere with the relationship of other columns. 
* Do more exploratory querying with the analytics and products table