# packling_challenge

SQL Challenge for Data Analysts

## Load Data in a relational database

### BIGQUERY

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


### Python - Pandas

I loaded both raw csv as dataframes. Using string, split function with semicolon as separator which returned a series of arrays.

```Python - pandas
df_clients = pd.read_csv('input/clients.csv')
df_clients.head()

df = pd.DataFrame(df_clients['shipment_reference;client_id;product;country;order_date;invoiced_weight;net_revenue;net_cost;shipment_source'].str.split(';'))

df_ship = pd.read_csv('input/shipments.csv', error_bad_lines=False)
df_ship.head()

df_1 = pd.DataFrame(df_ship['id;estimated_deliveries_volumes;created_at'].str.split(';'))
```

After facing some issues with iterrows() function, I decided to move on with MySQL.

### MySQL

After downloading and install MySQL, I created tables "clients" and "shipments" to import raw data. All fields were VARCHAR(200) without Primary Key and nullable values. I had to use ' as string indicator and semicolon as separator. 

With clients table I had an issue with the string, so I had to used SUBSTRING method: 

```SQL - SUBSTRING method
SELECT SUBSTRING(client_id, 2) AS client_id,
estimated_volume,
SUBSTRING(DATE, 1, 10) AS date
FROM clients;
```

## Make checks to see if the data is clean

With clients table I had an issue with the string fields client_id and date, so I had to used SUBSTRING method to get the cleaned ones: 

```SQL - SUBSTRING method
SELECT SUBSTRING(client_id, 2) AS client_id,
estimated_volume,
SUBSTRING(DATE, 1, 10) AS date
FROM clients;
```

Checking how many null values are in clients for:

- client_id
- estimated_volume
- date

Check how many duplicated there are in clients for:
- client_id
- client_id and date

Checking how many null values there are in shipments for:

- shipment_reference
- client_id
- date
- product
- country
- source
- weight
- revenue
- costs


Check how many duplicated there are in shipments for:
- shipment_reference


### If it is not clean, what could be done to make it clean? What are the steps are taken and the results obtained by these cleansing activities?

First, for client_id null values in clients table, we could estimate a deliveries volume based on a COUNT(shipment_reference) by client_id. 

If there are records in clients without client_id, we could delete this info because is the link with shipments table.

Check all shipments record without a weight, but with revenue and/or profit. We could estimate an average weight based on an average info from each source, knowing their revenue and profits for grouped weights:

```SQL - source, weight, revenue and profits
SELECT source,
weight,
net_revenue,
net_profit,
CASE WHEN weight < 5 THEN 'Lower than 5',
WHEN weight >= 5 AND weight < 10 THEN '5 - 10'
WHEN weight >= 10 AND weight < 25 THEN '10 - 25'
WHEN weight >= 25 AND weight < 50 THEN '25 - 50'
ELSE 'Higer than 50'
END as AVG_weight
FROM shipments
GROUP BY source
```
Also, knowing this information we could add an average revenue or profit if null, based on their weight.


Check if the number of COUNT(shipments_reference) for each client_id is the same as estimated in clients table because it could be a reason for a deviation. In case it's not the same, create a table with all clients that have more deliveries than hired and how many are.