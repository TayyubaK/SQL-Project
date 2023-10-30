# Final-Project-Transforming-and-Analyzing-Data-with-SQL

## Project/Goals
This purpose of this project was to analyze a set of 5 tables related to an e-commerce site to determine relationships and extract information. 

The goals of the project were:

* Review and select necessary data
* Use SQL queries to:

    * clean the data for use without major alteration to the source data
    * extract necessary information from the tables
    * QA queries and the result set (QA.md)

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
* The 


## Challenges 
* Many columns were not clear on the basis of their names and data alone. A SME would have been helpful or some additional context about the source of the data.
* Timeline to deliver results which creates a need to evaluate the scope/effort of different approaches to data cleaning  

## Future Goals

* Create a data dictionary to define columns and their characteristics
* Separate columns into their own tables (and normalize). For example, the 'all_sessions' has 32 columns. Many columns interfere with the relationship of other columns. 
* Do more exploratory querying