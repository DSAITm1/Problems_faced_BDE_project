# Project faced

## Uploading csv to Supabase
- Multi-line comments causes problem parsing if the reading of csv is performed by csv library. pd.read_csv() is able to handle better.
- Supabase doesn't allow create or drop table via their supabase python library.
- If the constraints are in place at the data collection, the duplicated key will not happen at downstream.
- If the csv files are uploaded manually via web interface, the olist_products_dataset.csv will face issue. bigint for postgres database cannot be empty string. Even after changing the data type to string, the number of entries is 7264 instead of 32951. Need to understand why the data is unable to upload during the first attempt.

## Extract and load using meltano
- Need to convert a proper setup script so that the setup is not manual
