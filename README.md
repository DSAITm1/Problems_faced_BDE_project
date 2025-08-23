# Lesson_learnt_BDE_project

## Uploading csv to Supabase
- multi-line comments causes problem parsing if the reading of csv is performed by csv library. pd.read_csv() is able to handle better.
- Supabase doesn't allow create or drop table via their supabase python library.
- If the constraints are in place at the data collection, the duplicated key will not happen at downstream.
