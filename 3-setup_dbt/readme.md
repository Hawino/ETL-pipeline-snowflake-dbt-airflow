# Setup DBT

DBT merupakan singkatan dari Data Build Tool adalah sebuah framework transformasi data open-source yang dirancang khusus untuk database/datawarehouse cloud.

## Setup di Command Prompt
1. Buka Command Prompt dan ketik `python -m pip install dbt-core`.
2. Selanjutnya ketik `python -m pip install dbt-snowflake`.
3. Setelah selesai menginstall DBT, selanjutnya memulai program dbt dengan code `dbt init`.
4. Silakan isi untuk setup awal dbt, seperti
  - Enter a name for your project (letters, digits, underscore): data_pipeline
  - Which database would you like to use?
    [1] snowflake
    Enter a number: 1 (pilih angka yang ada snowflake)
  - account (https://<this_value>.snowflakecomputing.com): ... (masukan id akun Snowflake, ke Snowflake > check menubar di kiri > Admin > Account, kemudian arahkan kursor ke kolom Account anda)
  - Desired authentication type option (enter a number): 1
  - password (dev password): ... (masukan pass user Snowflake)
  - role (dev role): dbt_role
  - warehouse (warehouse name): dbt_wh
  - database (default database that dbt will build objects in): dbt_db
  - schema (default schema that dbt will build objects in): dbt_schema
  - threads (1 or more) [1]: 10
5. Selanjutnya pindah ke direktori `cd data_pipeline`
6. Setelah pindah direktori, ketik `code .`, maka kita akan otomatis berpindah ke VSCode untuk setup selajutnya

## Setup di VSCode
1. Setup file `dbt_project.yml` dengan code berikut
```
# Name your project! Project names should contain only lowercase characters
# and underscores. A good package name should reflect your organization's
# name or the intended use of these models
name: 'data_pipeline'
version: '1.0.0'

# This setting configures which "profile" dbt uses for this project.
profile: 'data_pipeline'

# These configurations specify where dbt should look for different types of files.
# The `model-paths` config, for example, states that models in this project can be
# found in the "models/" directory. You probably won't need to change these!
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:         # directories to be removed by `dbt clean`
  - "target"
  - "dbt_packages"


# Configuring models
# Full documentation: https://docs.getdbt.com/docs/configuring-models

# In this example config, we tell dbt to build all models in the example/
# directory as views. These settings can be overridden in the individual model
# files using the `{{ config(...) }}` macro.
models:
  data_pipeline:
    # Config indicated by + and applies to all files under models/example/
    staging:
      materialized: view
      snowflake_warehouse: dbt_wh
    marts:
      materialized: table
      snowflake_warehouse: dbt_wh
```
2. Buat file baru di luar folder dengan nama `packages.yml` dengan isi code
```
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```
3. Selanjutnya di bagian terminal, running code `dbt deps` untuk menginstall dbt packages yang sudah kita setup di poin 2.
4. Buat 2 folder baru pada folder Models, yaitu marts dan staging\
![image](https://github.com/Hawino/ETL-pipeline-snowflake-dbt-airflow/assets/160495569/da89e2ed-31e7-42f7-8659-5670f7be6fad)
5. Pada folder staging, akan membuat 3 file baru yaitu:
  - tpch_sources.yml
  - stg_tpch_orders.sql
  - stg_tpch_line_items.sql\

`tpch_sources.yml`
```
version: 2

sources:
  - name: tpch
    database: snowflake_sample_data
    schema: tpch_sf1
    tables:
      - name: orders
        columns:
          - name: o_orderkey
            tests:
              - unique
              - not_null
      - name: lineitem
        columns:
          - name: l_orderkey
            tests:
              - relationships:
                  to: source('tpch', 'orders')
                  field: o_orderkey
```

`stg_tpch_orders.sql`
```
select
    o_orderkey as order_key,
    o_custkey as customer_key,
    o_orderstatus as status_code,
    o_totalprice as total_price,
    o_orderdate as order_date
from
    {{ source('tpch', 'orders') }}
```

`stg_tpch_line_items.sql`
``` 
 select
    {{
        dbt_utils.generate_surrogate_key([
            'l_orderkey',
            'l_linenumber'
        ])
    }} as order_item_key,
	l_orderkey as order_key,
	l_partkey as part_key,
	l_linenumber as line_number,
	l_quantity as quantity,
	l_extendedprice as extended_price,
	l_discount as discount_percentage,
	l_tax as tax_rate
from
    {{ source('tpch', 'lineitem') }}
```

6. Pada folder marts, akan membuat 3 file baru yaitu:
  - int_order_items.sql
  - int_order_items_summary.sql
  - fct_orders.sql\
 `int_order_items.sql`
```
select
    line_item.order_item_key,
    line_item.part_key,
    line_item.line_number,
    line_item.extended_price,
    orders.order_key,
    orders.customer_key,
    orders.order_date,
    {{ discounted_amount('line_item.extended_price', 'line_item.discount_percentage') }} as item_discount_amount
from
    {{ ref('stg_tpch_orders') }} as orders
join
    {{ ref('stg_tpch_line_items') }} as line_item
        on orders.order_key = line_item.order_key
order by
    orders.order_date
```

`int_order_items_summary.sql`
```
select 
    order_key,
    sum(extended_price) as gross_item_sales_amount,
    sum(item_discount_amount) as item_discount_amount
from
    {{ ref('int_order_items') }}
group by
    order_key
```

`fct_orders.sql`
```
select
    orders.*,
    order_item_summary.gross_item_sales_amount,
    order_item_summary.item_discount_amount
from
    {{ref('stg_tpch_orders')}} as orders
join
    {{ref('int_order_items_summary')}} as order_item_summary
        on orders.order_key = order_item_summary.order_key
order by order_date
```

7. Setup Fungsi untuk nanti dipanggil di salah satu tabel, pada folder Makro, buat file `pricing.sql` dengan isi code
```
{% macro discounted_amount(extended_price, discount_percentage, scale=2) %}
    (-1 * {{extended_price}} * {{discount_percentage}})::decimal(16, {{ scale }})
{% endmacro %}
```

8. 
