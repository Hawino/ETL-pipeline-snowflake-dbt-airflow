# Setup Snowflake

Snowflake adalah gudang data cloud (cloud data warehouse) yang skalabel, fleksibel dan berperforma tinggi. Snowflake sendiri layanannya berbayar tetapi kita bisa mencoba dengan trial selama 30 hari. 

1. Buat akun di https://www.snowflake.com/en/
2. Pada menu bar di sebelah kiri, pilih Projects > Worksheets. Buat project baru berupa SQL Worksheet
3. Berikut adalah contoh code untuk membuat database, warehouse, role, user hingga schema 
```
-- create accounts
use role accountadmin;

drop warehouse if exists dbt_wh;
drop database if exists dbt_db;
drop role if exists dbt_role;

create database if not exists dbt_db;
create warehouse dbt_wh with warehouse_size='x-small';
create role if not exists dbt_role;

show grants on warehouse dbt_wh;
grant usage on warehouse dbt_wh to role dbt_role;
grant role dbt_role to user xxx; -- ganti ke username akun snowflake sendiri
grant usage on warehouse dbt_wh to role dbt_role;
grant all on database dbt_db to role dbt_role;

use role dbt_role;

create schema if not exists dbt_db.dbt_schema;
```
4. Setelah code dirunning maka akan muncul tampilan database seperti ini
![image](https://github.com/Hawino/ETL-pipeline-snowflake-dbt-airflow/assets/160495569/25d6df17-9510-47e6-95e8-78c981d63828)
