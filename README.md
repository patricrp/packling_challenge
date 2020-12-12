# packling_challenge

SQL Challenge for Data Analysts

> Load Data in a relational database

I decided to use BigQuery for the challenge. After downloading both csv, I created a project called "PackLink-challenge" to start with the test. 

I tried to load clients data and saw that the csv was a string separated by semicolons, so as a new table a get only a column with all the data: id + estimated deliverables volume + created date. I was using SPLIT function to separate the records but what I got was 3 records per row. The UNNEST function wasn't an option since it allows arrays type and not string ones. 

After, I used REGEX to get id and date, but had some issues fetching the nullable field "estimated deliverables volume".

SELECT string_field_0,
REGEXP_EXTRACT(string_field_0, r"^\w{8}-\w{4}-\w{4}-\w{4}-\w{12}") as id,
REGEXP_EXTRACT(string_field_0, r"\d{2}\/\d{2}\/\d{4}$") as date
from  `packlink.clients_base` 