# packling_challenge

SQL Challenge for Data Analysts

> Load Data in a relational database

I decided to use BigQuery for the challenge. After downloading both csv, I created a project called "PackLink-challenge" to start with the test. 

I loaded clients data to a table called "clients_base" and saw that the csv was a string separated by semicolons, so as a new table a get only a column with all the data: id + estimated deliverables volume + created date. 

To get structured schema I selected different string functions to split the string: 
- SPLIT function to separate the records. I got 3 records per row, as a kind of pivot table that wasn't very useful. 

```SQL - SPLIT function
SELECT SPLIT( string_field_0, ';' ) 
FROM `packlink.clients_base`
```

- UNNEST function to separate the string wasn't an option since it allows arrays type. 

```SQL - UNNEST function
SELECT string_field_0, UNNEST(string_field_0) AS test
FROM `packlink.clients_base`
```

After, I used REGEX to get id and date, but had some issues fetching the nullable field "estimated deliverables volume". 

```SQL
SELECT string_field_0,
    REGEXP_EXTRACT(string_field_0, r"^\w{8}-\w{4}-\w{4}-\w{4}-\w{12}") as id,
    REGEXP_EXTRACT(string_field_0, r"\d{2}\/\d{2}\/\d{4}$") as date
from  `packlink.clients_base` 
```

I had some problems getting id + estimated deliverables volume + date, so I created a new table for id and date with a null column for estimated deliverables volume with the following query

```SQL - creating clients table
CREATE TABLE `packlink.clients` AS 
SELECT 
    string_field_0,
    REGEXP_EXTRACT(string_field_0, r"^\w{8}-\w{4}-\w{4}-\w{4}-\w{12}") as id,
    REGEXP_EXTRACT(string_field_0, r"\d{2}\/\d{2}\/\d{4}$") as date,
    "NA" AS estimated_volume
FROM  `packlink.clients_base`;
```
I was able to get estimated deliverables volume in a separate query:

```SQL - estimated deliverables volume
SELECT
    REGEXP_EXTRACT( string_field_0, r"(\[\d{0,2}\ \-\ \d{1,5}\])" ) as vol
FROM  `packlink.clients_base`
```



    


I also tried to create a new empty table with same schema id + estimated deliverables volume + created date, all as string, to join later the csv. 