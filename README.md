# Project faced

## Uploading csv to Supabase
- Multi-line comments causes problem parsing if the reading of csv is performed by csv library. pd.read_csv() is able to handle better.
- Supabase doesn't allow create or drop table via their supabase python library.
- If the constraints are in place at the data collection, the duplicated key will not happen at downstream.
- If the csv files are uploaded manually via web interface, the olist_products_dataset.csv will face issue. bigint for postgres database cannot be empty string. Even after changing the data type to string, the number of entries is 7264 instead of 32951. Need to understand why the data is unable to upload during the first attempt.

## Extract and load using meltano
- Need to convert a proper setup script so that the setup is not manual

## Data integrity check
### Check Criteria
Verifying that all foreign key relationships are valid, such as:
- All order_ids in order_items, payments, reviews, and products exist in orders
- All customer_ids in orders exist in customers
- All seller_ids in order_items and products exist in sellers
- All product_ids in order_items exist in products
- All customer_zip_code_prefix in customers exist in geolocation
- All seller_zip_code_prefix in sellers exist in geolocation

This ensures that every reference in a child table points to a valid record in the parent table.

### Outcome
- order_items.order_id -> orders.order_id: 0 missing
- order_payments.order_id -> orders.order_id: 0 missing
- order_reviews.order_id -> orders.order_id: 0 missing
- orders.customer_id -> customers.customer_id: 0 missing
- order_items.product_id -> products.product_id: 0 missing
- order_items.seller_id -> sellers.seller_id: 0 missing
- customers.customer_zip_code_prefix -> geolocation.geolocation_zip_code_prefix: 157 missing
- sellers.seller_zip_code_prefix -> geolocation.geolocation_zip_code_prefix: 7 missing

## Data Duplicates
The `olist_geolocation_dataset` contains multiple entries for the same zip code prefix, often with different latitude/longitude pairs.
This is because a single zip code prefix can cover a large area, and the dataset provides multiple sampled points for each prefix to better represent the geographic spread.

### Outcome
Total rows: 1000163
Unique zip code prefixes: 19015
Unique (zip, lat, lng): 720154
Rows per zip code prefix (top 5):
24220    1146
24230    1102
38400     965
35500     907
11680     879
Name: geolocation_zip_code_prefix, dtype: int64
