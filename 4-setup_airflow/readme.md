# Setup Airflow

Airflow adalah platform pengatur alur kerja (workflow orchestration) open-source yang populer digunakan untuk penjadwalan dan pengelolaan data pipeline.
Untuk mengintegrasikan DBT dengan Airflow, menggunakan library dari Astro.

## Setup at Command Prompt
1. Download dan install Astro dengan code `winget install -e --id Astronomer.Astro`.
2. Setelah selesai install Astro, bukan terminal baru dan buat direktori untuk Astro dengan code `cd dbt_dag`. Lanjut dengan code `astro dev init`
3. Running code `code .` untuk masuk ke VSCode

## Setup at VSCode
1. Setup Dockerfile dengan code:
```
FROM quay.io/astronomer/astro-runtime:11.4.0

RUN python -m venv dbt_venv && source dbt_venv/bin/activate && \
    pip install --no-cache-dir dbt-snowflake && deactivate
   ```
\
2. Setup requirement.txt dengan code:
```
# Astro Runtime includes the following pre-installed providers packages: https://docs.astronomer.io/astro/runtime-image-architecture#provider-packages
astronomer-cosmos
apache-airflow-providers-snowflake
```
\
3. Di dalam folder dags, buat file dbt_dags.py dengan code:
```
import os
from datetime import datetime

from cosmos import DbtDag, ProjectConfig, ProfileConfig, ExecutionConfig
from cosmos.profiles import SnowflakeUserPasswordProfileMapping


profile_config = ProfileConfig(
    profile_name="default",
    target_name="dev",
    profile_mapping=SnowflakeUserPasswordProfileMapping(
        conn_id="snowflake_conn", 
        profile_args={"database": "dbt_db", "schema": "dbt_schema"},
    )
)

dbt_snowflake_dag = DbtDag(
    project_config=ProjectConfig(f"{os.environ['AIRFLOW_HOME']}/dags/dbt/data_pipeline"),
    operator_args={"install_deps": True},
    profile_config=profile_config,
    execution_config=ExecutionConfig(dbt_executable_path=f"{os.environ['AIRFLOW_HOME']}/dbt_venv/bin/dbt",),
    schedule_interval="@daily",
    start_date=datetime(2024, 6, 2), # ganti dengan tanggal hari ini
    catchup=False,
    dag_id="dbt_dag",
)
```
\
4. Buat folder dengan nama dbt di dalam folder dags. Kemudian buat lagi folder data_pipeline di dalam folder dbt.
\
5. Copy semua folder dan file dari VSCode saat melakukan setup DBT ke dalam folder data_pipeline seperti contoh berikut:
![image](https://github.com/Hawino/ETL-pipeline-snowflake-dbt-airflow/assets/160495569/36eec958-a1e4-4cff-bf69-66b09b03c082)
\
6. Setelah itu kembali ke Command Prompt dan running code `astro dev start` untuk masuk ke dalam Airflow UI. Program akan otomatis membuka Airflow UI, jika tidak terbuka otomatis, di bagian bawah setelah running code ada info link, usernam, dan pass seperti ini:
![image](https://github.com/Hawino/ETL-pipeline-snowflake-dbt-airflow/assets/160495569/88b84774-1e1e-410d-97cb-0c16fd8baa70)
Untuk user dan pass di Airflow UI, gunakan admin

## Setup di Airflow UI
1. Setelah masuk ke dalam Airflow UI, di menubar atas Admin > Connections. Buat Connections baru. Masukan data yang diperlukan seperti username Snowflake, password hingga Account ID Snowflake seperti ini
<img width="960" alt="screencapture-localhost-8080-connection-edit-1-2024-06-03-20_54_15" src="https://github.com/Hawino/ETL-pipeline-snowflake-dbt-airflow/assets/160495569/1f8d029e-8c45-4508-9225-0437b938aa7f">

\
2. Pada Menubar, klik DAGs. Selanjutnya akan muncul dag yg kita setup sebelumnya, yaitu dbt_dag. Klik dbt_dag.

\
3. Unpause DAG dan trigger DAG. Proses otomasi Airflow berhasil jika seluruh indikator berwarna hijau.
<img width="944" alt="Screenshot 2024-06-03 210534" src="https://github.com/Hawino/ETL-pipeline-snowflake-dbt-airflow/assets/160495569/d86490d6-28ab-4119-b2ca-c67908b4142c">
