# Problems Faced in BDE Project

## Extract and Load Using Meltano

- Need to convert to a proper setup script so that the setup is not manual.

## Uploading CSV to Supabase

- Multi-line comments cause parsing problems if the CSV reading is performed by the CSV library. `pd.read_csv()` handles this better.
- Supabase doesn't allow creating or dropping tables via their Supabase Python library.
- If constraints are in place at data collection, duplicated keys won't occur downstream.
- If CSV files are uploaded manually via the web interface, `olist_products_dataset.csv` faces issues. BIGINT for PostgreSQL database cannot be an empty string. Even after changing the data type to string, the number of entries is 7264 instead of 32951. Need to understand why the data couldn't upload during the first attempt.

## Data Integrity Check

### Check Criteria

Verifying that all foreign key relationships are valid, such as:

- All `order_ids` in `order_items`, `payments`, `reviews`, and `products` exist in `orders`.
- All `customer_ids` in `orders` exist in `customers`.
- All `seller_ids` in `order_items` and `products` exist in `sellers`.
- All `product_ids` in `order_items` exist in `products`.
- All `customer_zip_code_prefix` in `customers` exist in `geolocation`.
- All `seller_zip_code_prefix` in `sellers` exist in `geolocation`.

This ensures that every reference in a child table points to a valid record in the parent table.

### Outcome

- `order_items.order_id` -> `orders.order_id`: 0 missing
- `order_payments.order_id` -> `orders.order_id`: 0 missing
- `order_reviews.order_id` -> `orders.order_id`: 0 missing
- `orders.customer_id` -> `customers.customer_id`: 0 missing
- `order_items.product_id` -> `products.product_id`: 0 missing
- `order_items.seller_id` -> `sellers.seller_id`: 0 missing
- `customers.customer_zip_code_prefix` -> `geolocation.geolocation_zip_code_prefix`: 157 missing
- `sellers.seller_zip_code_prefix` -> `geolocation.geolocation_zip_code_prefix`: 7 missing

## Data Duplicates

The `olist_geolocation_dataset` contains multiple entries for the same zip code prefix, often with different latitude/longitude pairs.
This is because a single zip code prefix can cover a large area, and the dataset provides multiple sampled points for each prefix to better represent the geographic spread.

### Outcome

- Total rows: 1,000,163
- Unique zip code prefixes: 19,015
- Unique (zip, lat, lng): 720,154
- Rows per zip code prefix (top 5):
  - 24220: 1,146
  - 24230: 1,102
  - 38400: 965
  - 35500: 907
  - 11680: 879

## Relational Data

### CASE #1

Tables:
Geolocation => `olist_geolocation_dataset` Total Records: 1,000,163
Customers => `olist_customers_dataset` Total Records: 99,441

Table Geolocation cannot be join with Customer table as is because of the above issue where same zip code, same city, and same state with different latitude or langitude.

Query:
WITH GEO AS
(SELECT geolocation_zip_code_prefix, geolocation_city, geolocation_state,
COUNT(geolocation_lat) cnt_lat, COUNT(geolocation_lng) cnt_lng
FROM dsai-468212.olist_staging.public_olist_geolocation_dataset
GROUP BY geolocation_zip_code_prefix, geolocation_city, geolocation_state
HAVING cnt_lat > 1 AND cnt_lng > 1
)
SELECT \*
FROM dsai-468212.olist_staging.public_olist_customers_dataset C, GEO g
WHERE customer_zip_code_prefix = geolocation_zip_code_prefix
AND customer_city = geolocation_city
AND customer_state = geolocation_state
ORDER BY customer_id

Outcome => Generating 98,905 records of customers with having multiple lat or lng

### Solutions:

Getting only 1 of Latitude or Langitude like example using MAX / MIN / MEDIAN function

Query:
SELECT geolocation_zip_code_prefix, geolocation_city, geolocation_state,
MAX(geolocation_lat) cnt_lat, MAX(geolocation_lng) cnt_lng
FROM dsai-468212.olist_staging.public_olist_geolocation_dataset
GROUP BY geolocation_zip_code_prefix, geolocation_city, geolocation_state

Outcome => it will generating only 27,914 records out of 1,000,163 can be used as New Dimension Table of GeoLocation

### CASE #2

Tables:
Geolocation => `olist_geolocation_dataset` Total Records: 1,000,163
Customers => `olist_customers_dataset` Total Records: 99,441

Similar as above but this time Customer table all having Prefix Zip Code, City and State - some of this value do not have reference to Geolocation table as example below

Query:
SELECT COUNT(\*)
FROM dsai-468212.olist_staging.public_olist_customers_dataset C
WHERE NOT EXISTS (SELECT 1
FROM dsai-468212.olist_staging.public_olist_geolocation_dataset G
WHERE customer_zip_code_prefix = geolocation_zip_code_prefix
AND customer_city = geolocation_city
AND customer_state = geolocation_state)

Outcome => Generating 318 records that having value of zip, city, state but not exists in Geolocation table

### Solutions:

Using LEFT JOIN of Customer table so that regardless if value not match in Geolocation will still showing the Customer, the downside is performance because as we know the columns are having values, by right should using INNER JOIN but it will not display for Customer that not match againts the Geolocation table.

### CASE #3

Tables:
Geolocation => `olist_geolocation_dataset` Total Records: 1,000,163
Sellers => `olist_sellers_dataset` Total Records: 3,095

Table Geolocation cannot be join with Sellers or Sellers table as is because of the above issue where same zip code, same city, and same state with different latitude or langitude.

Query:
WITH GEO AS
(SELECT geolocation_zip_code_prefix, geolocation_city, geolocation_state,
COUNT(geolocation_lat) cnt_lat, COUNT(geolocation_lng) cnt_lng
FROM dsai-468212.olist_staging.public_olist_geolocation_dataset
GROUP BY geolocation_zip_code_prefix, geolocation_city, geolocation_state
HAVING cnt_lat > 1 AND cnt_lng > 1
)
SELECT \*
FROM dsai-468212.olist_staging.public_olist_sellers_dataset C, GEO g
WHERE seller_zip_code_prefix = geolocation_zip_code_prefix
AND seller_city = geolocation_city
AND seller_state = geolocation_state
ORDER BY seller_id

### Outcome

Generating 2,954 records of sellers with having multiple lat or lng

### Solutions:

Same as CASE no.1

### CASE #4

Tables:
Geolocation => `olist_geolocation_dataset` Total Records: 1,000,163
Sellers => `olist_sellers_dataset` Total Records: 3,095

Similar as above but this time Sellers table all having Prefix Zip Code, City and State - some of this value do not have reference to Geolocation table as example below

Query:
SELECT COUNT(\*)
FROM dsai-468212.olist_staging.public_olist_sellers_dataset C
WHERE NOT EXISTS (SELECT 1
FROM dsai-468212.olist_staging.public_olist_geolocation_dataset G
WHERE seller_zip_code_prefix = geolocation_zip_code_prefix
AND seller_city = geolocation_city
AND seller_state = geolocation_state)

### Outcome

Generating 137 records of seller that having value of zip, city, state but not exists in Geolocation table

### Solutions:

Same as CASE no.2 to used LEFT JOIN of Sellers table.

### CASE #5

Tables:
Orders => `olist_orders_dataset` Total Records: 99,441

3 columns below are empty

Query:
SELECT COUNT(_) FROM dsai-468212.olist_staging.public_olist_orders_dataset WHERE order_approved_at = '';
SELECT COUNT(_) FROM dsai-468212.olist_staging.public_olist_orders_dataset WHERE order_delivered_carrier_date = '';
SELECT COUNT(\*) FROM dsai-468212.olist_staging.public_olist_orders_dataset WHERE order_delivered_customer_date = '';

### Outcome

Generating 160 records empty of order_approved_at
Generating 1,783 records empty of order_delivered_carrier_date
Generating 2,965 records empty of order_delivered_customer_date

### Solutions:

Are we treat the records as ORDER that still DRAFT or we should DELETE it ?

### CASE #6

Tables:
Orders => `olist_orders_dataset` Total Records: 99,441
Columns: order_purchase_timestamp, order_approved_at, order_delivered_carrier_date, order_delivered_customer_date, order_estimated_delivery_date

Order Items => `olist_order_items_dataset` Total Records: 112,650
Columns: shipping_limit_date

Above tables having column date but datatype is STRING and need to be converted into DATETIME

## Data Issues

- **Geolocation Problems**: Some geolocation (lat, lon) points plot outside Brazil, landing in other countries or the ocean, so they're bad geocodes.
- **Customer ID Confusion**: There are two fields: `customer_id` and `customer_unique_id`. `customer_id` changes with every order. For any analysis about real customers, always use `customer_unique_id`.
