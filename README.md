# packling_challenge

SQL Challenge for Data Analysts

> Load Data in a relational database

I decided to use BigQuery for the challenge. After downloading both csv, I created a project called "PackLink-challenge" to start with the test. 

I loaded clients data to a table called "clients_base" with autodetected schema and comma separated values to see raw data. I saw that the csv was a string separated by semicolons, so as a new table a get only a column with all the data: id + estimated deliverables volume + created date. I repeated this process, deleting the first row to avoid headers.

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


I repeated the process for shipments, creating an empty table called "shipments_base" to see all data and process it later. The shipments csv was a large file so I upload it to Google Cloud Storage and started the process creating a new table called "shipments_base" with autodetected schema and comma separated values to see raw data. I saw that the csv was a string separated by semicolons with the following info: shipment_reference + client_id + product + country + order_date + invoiced_weight + net revenue + net_cost + shipment_source. I repeated this process, deleting the first row to avoid headers.

Working with REGES I took some assumptions, to work quickly in order to have a table to start with the challenge:

- All countries codes have 2 letters
- All product codes have 3 letters
- From timestamp string, I took only date + time
- From shipment_source, I got only first word, where 'eBay', 'CSV' and 'module_shopify' where null because of their regex requirements.


```SQL - shipments
SELECT 
    string_field_0,
    REGEXP_EXTRACT(string_field_0, r"(\w{2}\d{4}\w{3}\d{10})") as shipment_reference,
    REGEXP_EXTRACT(string_field_0, r"([a-z0-9]{8}\-[a-z0-9]{4}\-[a-z0-9]{4}\-[a-z0-9]{4}\-[a-z0-9]{12})") as client_id,
    REGEXP_EXTRACT(string_field_0, r"\b[A-Z]{3}\b") as product,
    REGEXP_EXTRACT(string_field_0, r"\b[A-Z]{2}\b") as country,
    REGEXP_EXTRACT(string_field_0, r"(\b\d{4}\-\d{2}\-\d{2}\ \d{2}\:\d{2}\:\d{2}\b)") as order_date,
    REGEXP_EXTRACT(string_field_0, r"(\b[A-Z][a-z]{3,})") as shipment_source
FROM  `packlink.shipments_base`
```

I got some issues to fetch weight, revenue and cost separately and since I was running out of time, I decided to move on with Python - pandas library, in order to get 2 cleaned datasets there and come back later with renewal csv. 