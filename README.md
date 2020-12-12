# packling_challenge

SQL Challenge for Data Analysts

## 1.Load Data in a relational database

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


I repeated the process for shipments, creating an empty table called "shipments_base" to see all data and process it later. The shipments csv was a large file so I upload it to Google Cloud Storage and started the process creating a new table called "shipments_base" with autodetected schema and comma separated values to see raw data. I saw that the csv was a string separated by semicolons with the following info: shipment_reference + client_id + product + country + order_date + invoiced_weight + net_revenue + net_cost + shipment_source. I repeated this process, deleting the first row to avoid headers.

Working with **regexp** I took some assumptions, to work quickly in order to have a table to start with the challenge:

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

### Since all this process took me so long to create the relational database properly and shipments data is too heavy, I decided to move on with an approach for the data analysis. 

## 2. Make checks to see if the data is clean

With clients table I had an issue with the string fields client_id and date, so I had to used SUBSTRING method to get the cleaned ones: 

```SQL - SUBSTRING method
SELECT SUBSTRING(client_id, 2) AS client_id,
    estimated_volume,
    SUBSTRING(DATE, 1, 10) AS date
FROM clients;
```
I couldn't load on time shipments data, so I supposed I have to repeat the same process for the first and last field. From now on, all you can find is **my proposal on how to proceed the data analysis.**

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
- invoiced_weight
- revenue
- costs


Check how many duplicated there are in shipments for:
- shipment_reference


### If it is not clean, what could be done to make it clean? What are the steps are taken and the results obtained by these cleansing activities?

First, for client_id null values in clients table, we could delete this info because is the link with the shipments table. And if there are clients_id in the shipments table that are not included in clients one, we could estimate a deliveries volume based on a COUNT(shipment_reference) by client_id and minimum ordered date.

Check all shipments record without a weight, but with revenue and/or profit. We could estimate an average weight based on an average info from each source, knowing their revenue and profits for grouped weights:

```SQL - source, weight, revenue and profits
SELECT source,
    weight,
    net_revenue,
    net_profit,
    CASE WHEN weight < 5 THEN 'Lower than 5'
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

### Provide Consolidated information – total revenue, total revenue by country, total revenue by source

Once data is cleaned, we are able to provide consolidated information.  Since we have to calculated it for country and source, I include a ROLLUP with the GROUP BY.

The query should look like this:

```SQL - total revenue
SELECT 
    shipment_source,
    country,
    SUM(net_revenue) as revenue
FROM shipments
GROUP BY shipment_source, country WITH ROLLUP
```

### Identify the clients with the lowest profitability // Identify the clients with the highest profitability

Profitability could be calculated subtracting costs to revenue. If there is no requirement about date profitability, I decided to analyse all together to know the most and less profitability clients we have record for.

The query should show the top and bottom rows. I limit the query to show 5 clients of each type.

```SQL - Profitability

SELECT * 
FROM 
(SELECT prof.*, ROW_NUMBER() OVER (Order BY prof.profitability) as TopFive
   ,ROW_NUMBER() OVER (Order BY prof.profitability Desc) as BottomFive
   FROM 
   (SELECT
       client_id,
        SUM(net_revenue) - SUM(net_costs) as profitability
    FROM shipments
    GROUP BY client_id
   ) prof
)
WHERE TopFive <=5 OR BottomFive <=5
```


### Identify the clients who generate more shipments than estimated // Identify the clients who generate fewer shipments than estimated

To know if the clients are generating more or less shipments than estimated, we have to compare the number of shipments recorded in shipments with the estimated_deliveries_volume in the clients table

```SQL - shipments
SELECT shi.client_id,
    COUNT(shi.shipment_reference) as num_deliveries,
    shi.estimated_volume
FROM shipments shi
LEFT JOIN clients cli
WHERE shi.client_id = cli.client_id
GROUP BY shi.client_id
```

If we create two new series, one with the minimun and the other with the maximum volume of estimated deliveries, we can compare the number of shipment with both series. If the number of deliveries is less than the series minimum, those clients are using less than hired. On the other way, if the number of deliveries is higher than the max, those clients are using more deliveries than hired.