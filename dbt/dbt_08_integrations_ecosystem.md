# 8. Integrations & Ecosystem

## Overview
dbt integrates with data warehouses (via adapters), orchestration platforms (Airflow/Dagster/Meltano), and BI tools (Tableau/Looker/Power BI). Understanding adapter-specific optimizations and orchestration patterns is critical for production deployments.

---

## 8.1 Database Adapters

### What are Adapters?
Adapters translate dbt's generic SQL to warehouse-specific dialects. Each adapter supports unique features (Snowflake clustering, BigQuery partitioning, Redshift sort keys). dbt Labs maintains core adapters; community maintains others.

**Diagram: Adapter Architecture**
```text
dbt Core (generic SQL + Jinja)
     ↓ adapter plugin
dbt-snowflake / dbt-bigquery / dbt-redshift
     ↓ warehouse-specific SQL
Snowflake (CLUSTER BY) / BigQuery (PARTITION BY) / Redshift (SORTKEY)
     ↓ execution
Materialized models in warehouse
```

**Core Adapters (dbt Labs)**
- **Snowflake**: dbt-snowflake
- **BigQuery**: dbt-bigquery
- **Redshift**: dbt-redshift
- **Postgres**: dbt-postgres
- **Databricks**: dbt-databricks
- **Spark**: dbt-spark

**Community Adapters**: DuckDB, ClickHouse, Trino, Oracle, SQLServer, Teradata

### Installation
```bash
# Install adapter via pip
pip install dbt-snowflake==1.7.0
pip install dbt-bigquery==1.7.0
pip install dbt-redshift==1.7.0

# Verify installation
dbt --version
# Output: Plugins: snowflake-1.7.0, bigquery-1.7.0
```

---

## 8.2 Snowflake Adapter

### Syntax: Snowflake-Specific Configs

**SYNTAX TEMPLATE**
```yaml
models:
  <project>:
    <folder>:
      +materialized: table | view | incremental | ephemeral
      +snowflake_warehouse: <warehouse_name>
      +cluster_by: [<expr1>, <expr2>]
      +automatic_clustering: true | false
      +transient: true | false
      +copy_grants: true | false
      +secure: true | false
      +query_tag: <tag_string>
      +tags:
        <key>: <value>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| snowflake_warehouse | String | profile default | Compute warehouse for execution | TRANSFORMING_WH |
| cluster_by | Array | [] | Clustering keys (improves pruning) | ['order_date', 'region'] |
| automatic_clustering | Boolean | false | Auto-reclustering (Enterprise edition) | true |
| transient | Boolean | false | Transient table (no fail-safe, 1-day Time Travel) | true |
| copy_grants | Boolean | false | Preserve grants on table replace | true |
| secure | Boolean | false | Secure view (hides definition from non-owners) | true |
| query_tag | String | - | Query tag for cost tracking | dbt_{{ model.name }} |
| tags | Object | {} | Snowflake object tags | {pii: 'true', cost_center: 'analytics'} |

**BASIC EXAMPLE**
```sql
-- models/marts/fct_sales.sql
{{
  config(
    materialized='table',
    cluster_by=['sale_date', 'store_id']
  )
}}

SELECT 
  sale_id,
  sale_date,
  store_id,
  revenue
FROM {{ ref('stg_sales') }}
```

**ADVANCED EXAMPLE**
```sql
-- models/marts/fct_customer_events.sql
{{
  config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    cluster_by=['event_timestamp::date', 'event_type'],
    automatic_clustering=true,
    snowflake_warehouse='TRANSFORMING_XL_WH',
    transient=false,
    copy_grants=true,
    query_tag='dbt:fct_customer_events:{{ invocation_id }}',
    tags={'pii': 'contains_email', 'retention': '7_years'}
  )
}}

SELECT 
  event_id,
  event_timestamp,
  event_type,
  user_id,
  MD5(email) AS email_hash  -- PII protection
FROM {{ ref('stg_events') }}

{% if is_incremental() %}
  WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

**profiles.yml (Snowflake)**
```yaml
my_project:
  target: prod
  outputs:
    prod:
      type: snowflake
      account: abc12345.us-east-1  # Snowflake account locator
      user: dbt_service_account
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: TRANSFORMER
      warehouse: TRANSFORMING_WH
      database: ANALYTICS_DB
      schema: DBT_PROD
      threads: 8
      client_session_keep_alive: true
      query_tag: dbt_{{ target.name }}
    
    dev:
      type: snowflake
      account: abc12345.us-east-1
      user: "{{ env_var('USER') }}"
      authenticator: externalbrowser  # SSO authentication
      role: DEVELOPER
      warehouse: TRANSFORMING_WH
      database: ANALYTICS_DEV
      schema: DBT_{{ env_var('USER') | upper }}
      threads: 4
```

---

## 8.3 BigQuery Adapter

### Syntax: BigQuery-Specific Configs

**SYNTAX TEMPLATE**
```yaml
models:
  <project>:
    <folder>:
      +materialized: table | view | incremental | ephemeral
      +partition_by:
        field: <date_column>
        data_type: date | timestamp | datetime
        granularity: day | hour | month | year
        time_ingestion_partitioning: true | false
      +cluster_by: [<column1>, <column2>, <column3>, <column4>]
      +require_partition_filter: true | false
      +partition_expiration_days: <days>
      +labels:
        <key>: <value>
      +kms_key_name: <kms_key_path>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| partition_by.field | String | - | Column for partitioning (DATE/TIMESTAMP) | order_date, created_at |
| partition_by.data_type | Enum | - | Column data type | date, timestamp, datetime |
| partition_by.granularity | Enum | day | Partition granularity | day, hour, month, year |
| partition_by.time_ingestion_partitioning | Boolean | false | Partition by ingestion time (_PARTITIONTIME) | true |
| cluster_by | Array | [] | Clustering columns (max 4, order matters) | ['customer_id', 'product_id'] |
| require_partition_filter | Boolean | false | Require WHERE clause on partition column | true (cost control) |
| partition_expiration_days | Integer | - | Auto-delete partitions after N days | 365 (compliance) |
| labels | Object | {} | BigQuery resource labels | {team: 'analytics', env: 'prod'} |
| kms_key_name | String | - | Customer-managed encryption key | projects/.../cryptoKeys/... |

**BASIC EXAMPLE**
```sql
-- models/marts/fct_orders.sql
{{
  config(
    materialized='table',
    partition_by={
      'field': 'order_date',
      'data_type': 'date'
    },
    cluster_by=['customer_id', 'region']
  )
}}

SELECT 
  order_id,
  order_date,
  customer_id,
  region,
  revenue
FROM {{ ref('stg_orders') }}
```

**ADVANCED EXAMPLE**
```sql
-- models/marts/fct_events_partitioned.sql
{{
  config(
    materialized='incremental',
    unique_key='event_id',
    partition_by={
      'field': 'event_timestamp',
      'data_type': 'timestamp',
      'granularity': 'hour'
    },
    cluster_by=['event_type', 'user_id', 'device_type'],
    require_partition_filter=true,
    partition_expiration_days=90,
    labels={
      'team': 'data_engineering',
      'environment': 'production',
      'pii': 'true',
      'cost_center': 'analytics'
    }
  )
}}

SELECT 
  event_id,
  event_timestamp,
  event_type,
  user_id,
  device_type,
  event_properties
FROM {{ ref('stg_events') }}

{% if is_incremental() %}
  WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

**Incremental Strategy: BigQuery Merge**
```sql
-- models/marts/dim_customers.sql
{{
  config(
    materialized='incremental',
    unique_key='customer_id',
    incremental_strategy='merge',
    partition_by={'field': 'updated_at', 'data_type': 'timestamp'},
    on_schema_change='sync_all_columns'
  )
}}

SELECT 
  customer_id,
  customer_name,
  email,
  CURRENT_TIMESTAMP() AS updated_at
FROM {{ ref('stg_customers') }}

{% if is_incremental() %}
  WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**profiles.yml (BigQuery)**
```yaml
my_project:
  target: prod
  outputs:
    prod:
      type: bigquery
      method: service-account  # or oauth, oauth-secrets
      project: my-gcp-project
      dataset: dbt_prod  # Default dataset/schema
      threads: 8
      keyfile: /path/to/service-account.json
      location: US  # BigQuery region
      priority: interactive  # interactive | batch
      maximum_bytes_billed: 1000000000  # Cost control
      timeout_seconds: 300
      labels:
        environment: production
        team: analytics
    
    dev:
      type: bigquery
      method: oauth
      project: my-gcp-project
      dataset: dbt_dev_{{ env_var('USER') | replace('.', '_') }}
      threads: 4
      location: US
```

---

## 8.4 Redshift Adapter

### Syntax: Redshift-Specific Configs

**SYNTAX TEMPLATE**
```yaml
models:
  <project>:
    <folder>:
      +materialized: table | view | incremental | ephemeral
      +dist: all | even | auto | <column_name>
      +sort: [<column1>, <column2>]
      +sort_type: compound | interleaved
      +bind: true | false
      +backup: true | false
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| dist | String/Enum | auto | Distribution style (data distribution across nodes) | all, even, auto, customer_id |
| sort | Array | [] | Sort keys for query optimization | ['order_date', 'customer_id'] |
| sort_type | Enum | compound | Sort key type | compound (default), interleaved |
| bind | Boolean | true | Use bind variables (prevents plan caching issues) | false |
| backup | Boolean | true | Include table in automated snapshots | false (temp tables) |

**BASIC EXAMPLE**
```sql
-- models/marts/fct_sales.sql
{{
  config(
    materialized='table',
    dist='customer_id',
    sort=['sale_date']
  )
}}

SELECT 
  sale_id,
  sale_date,
  customer_id,
  revenue
FROM {{ ref('stg_sales') }}
```

**ADVANCED EXAMPLE**
```sql
-- models/marts/fct_orders.sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    dist='customer_id',
    sort=['order_date', 'customer_id'],
    sort_type='compound',
    bind=false,
    backup=true
  )
}}

SELECT 
  order_id,
  order_date,
  customer_id,
  order_amount,
  GETDATE() AS _updated_at
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
  WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**profiles.yml (Redshift)**
```yaml
my_project:
  target: prod
  outputs:
    prod:
      type: redshift
      host: my-redshift-cluster.abc123.us-east-1.redshift.amazonaws.com
      port: 5439
      user: dbt_service_account
      password: "{{ env_var('REDSHIFT_PASSWORD') }}"
      dbname: analytics
      schema: dbt_prod
      threads: 8
      keepalives_idle: 240  # TCP keepalive
      connect_timeout: 10
      search_path: public,dbt_prod  # Schema search order
      sslmode: require
    
    dev:
      type: redshift
      host: my-redshift-cluster.abc123.us-east-1.redshift.amazonaws.com
      port: 5439
      user: "{{ env_var('USER') }}"
      password: "{{ env_var('REDSHIFT_PASSWORD') }}"
      dbname: analytics
      schema: dbt_dev_{{ env_var('USER') }}
      threads: 4
```

---

## 8.5 Orchestration: Airflow

### What is Airflow Integration?
Apache Airflow orchestrates dbt runs as DAG tasks. Use `dbt-airflow` or `astronomer-cosmos` for native integration. Enables scheduling, retries, sensors, and dependencies with other data pipelines.

**Diagram: Airflow + dbt Pipeline**
```text
Airflow Scheduler (cron: daily 2am UTC)
     ↓ trigger DAG
Extract Task (Fivetran/Airbyte → Snowflake)
     ↓ ExternalTaskSensor (wait for completion)
dbt Seed Task (dbt seed)
     ↓ BashOperator
dbt Run Task (dbt run --select staging+)
     ↓ BashOperator  
dbt Test Task (dbt test)
     ↓ BashOperator
dbt Docs Generate (dbt docs generate)
     ↓ on_failure_callback
Slack Notification (alert data team)
```

### Syntax: Airflow DAG with dbt

**BASIC EXAMPLE**
```python
# dags/dbt_daily_pipeline.py
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email': ['alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5)
}

with DAG(
    'dbt_daily_transform',
    default_args=default_args,
    description='Daily dbt transformations',
    schedule_interval='0 2 * * *',  # 2am UTC daily
    start_date=datetime(2025, 1, 1),
    catchup=False,
    tags=['dbt', 'analytics']
) as dag:
    
    dbt_run = BashOperator(
        task_id='dbt_run',
        bash_command='cd /opt/dbt && dbt run --profiles-dir . --target prod'
    )
    
    dbt_test = BashOperator(
        task_id='dbt_test',
        bash_command='cd /opt/dbt && dbt test --profiles-dir . --target prod'
    )
    
    dbt_run >> dbt_test  # Task dependency
```

**ADVANCED EXAMPLE: Astronomer Cosmos**
```python
# dags/dbt_advanced_pipeline.py
from airflow import DAG
from cosmos import DbtTaskGroup, ExecutionConfig, ProfileConfig, ProjectConfig
from cosmos.profiles import SnowflakeUserPasswordProfileMapping
from airflow.operators.empty import EmptyOperator
from datetime import datetime

# Cosmos configuration
profile_config = ProfileConfig(
    profile_name='my_project',
    target_name='prod',
    profile_mapping=SnowflakeUserPasswordProfileMapping(
        conn_id='snowflake_default',
        profile_args={
            'database': 'ANALYTICS_DB',
            'schema': 'DBT_PROD'
        }
    )
)

project_config = ProjectConfig(
    dbt_project_path='/opt/dbt/my_project',
    models_relative_path='models',
    seeds_relative_path='seeds',
    snapshots_relative_path='snapshots',
    manifest_path='/opt/dbt/target/manifest.json'  # For task-level lineage
)

execution_config = ExecutionConfig(
    dbt_executable_path='/usr/local/bin/dbt',
    execution_mode='local'  # or 'docker', 'kubernetes'
)

with DAG(
    'dbt_cosmos_pipeline',
    start_date=datetime(2025, 1, 1),
    schedule='0 2 * * *',
    catchup=False,
    tags=['dbt', 'cosmos']
) as dag:
    
    start = EmptyOperator(task_id='start')
    
    # dbt models as individual Airflow tasks (automatic dependency graph)
    dbt_models = DbtTaskGroup(
        group_id='dbt_transformations',
        project_config=project_config,
        profile_config=profile_config,
        execution_config=execution_config,
        operator_args={
            'install_deps': True,  # dbt deps before run
            'append_env': True,
            'retries': 2
        }
    )
    
    end = EmptyOperator(task_id='end')
    
    start >> dbt_models >> end
```

**ADVANCED EXAMPLE: Selective dbt Runs with State**
```python
# dags/dbt_slim_ci.py
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.sensors.external_task import ExternalTaskSensor
from datetime import datetime, timedelta

with DAG(
    'dbt_slim_ci',
    schedule_interval='@hourly',
    start_date=datetime(2025, 1, 1),
    catchup=False
) as dag:
    
    # Wait for upstream Fivetran sync
    wait_for_ingestion = ExternalTaskSensor(
        task_id='wait_for_fivetran',
        external_dag_id='fivetran_sync',
        external_task_id='sync_complete',
        timeout=3600,
        mode='poke',
        poke_interval=60
    )
    
    # dbt deps (install packages)
    dbt_deps = BashOperator(
        task_id='dbt_deps',
        bash_command='cd /opt/dbt && dbt deps'
    )
    
    # Selective run: only modified models
    dbt_slim_run = BashOperator(
        task_id='dbt_slim_run',
        bash_command='''
            cd /opt/dbt
            dbt run \
              --select state:modified+ \
              --state /opt/dbt/prod_artifacts \
              --defer \
              --target prod
        '''
    )
    
    # Test only affected models
    dbt_slim_test = BashOperator(
        task_id='dbt_slim_test',
        bash_command='''
            cd /opt/dbt
            dbt test \
              --select state:modified+ \
              --state /opt/dbt/prod_artifacts \
              --target prod
        '''
    )
    
    wait_for_ingestion >> dbt_deps >> dbt_slim_run >> dbt_slim_test
```

**SDLC Use Case: Blue/Green Deployment with Airflow**
```python
# dags/dbt_blue_green_deploy.py
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import BranchPythonOperator
from datetime import datetime

def check_test_results(**context):
    """Route to green schema if tests pass, rollback if fail"""
    test_results = context['task_instance'].xcom_pull(task_ids='dbt_test_blue')
    if test_results == 0:  # Success
        return 'swap_to_green'
    else:
        return 'rollback_blue'

with DAG('dbt_blue_green', schedule_interval='@daily', start_date=datetime(2025, 1, 1)) as dag:
    
    dbt_run_blue = BashOperator(
        task_id='dbt_run_blue',
        bash_command='dbt run --target blue'
    )
    
    dbt_test_blue = BashOperator(
        task_id='dbt_test_blue',
        bash_command='dbt test --target blue',
        do_xcom_push=True
    )
    
    check_tests = BranchPythonOperator(
        task_id='check_tests',
        python_callable=check_test_results
    )
    
    swap_schemas = BashOperator(
        task_id='swap_to_green',
        bash_command='''
            snowsql -q "
                ALTER SCHEMA ANALYTICS_DB.DBT_PROD RENAME TO DBT_OLD;
                ALTER SCHEMA ANALYTICS_DB.DBT_BLUE RENAME TO DBT_PROD;
                ALTER SCHEMA ANALYTICS_DB.DBT_OLD RENAME TO DBT_BLUE;
            "
        '''
    )
    
    dbt_run_blue >> dbt_test_blue >> check_tests >> swap_schemas
```

---

## 8.6 Orchestration: Dagster

### What is Dagster Integration?
Dagster integrates dbt via `dagster-dbt` library. Models become Dagster assets with automatic lineage, asset checks (tests), and partitioning support.

**Diagram: Dagster + dbt Assets**
```text
Dagster Asset Graph
     ↓ dbt models → assets
stg_orders (dbt model) → Dagster Asset
     ↓ upstream dependency
fct_orders (dbt incremental) → Dagster Asset
     ↓ asset checks
dbt tests → Dagster Asset Checks
     ↓ materialization
Snowflake tables updated
```

### Syntax: Dagster Integration

**BASIC EXAMPLE**
```python
# definitions.py
from dagster import Definitions
from dagster_dbt import DbtCliResource, dbt_assets

dbt_project_dir = "/opt/dbt/my_project"
dbt_resource = DbtCliResource(project_dir=dbt_project_dir)

@dbt_assets(manifest=dbt_project_dir + "/target/manifest.json")
def my_dbt_assets(context, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()

defs = Definitions(
    assets=[my_dbt_assets],
    resources={"dbt": dbt_resource}
)
```

**ADVANCED EXAMPLE: Partitioned Assets**
```python
# definitions.py
from dagster import (
    Definitions, 
    DailyPartitionsDefinition,
    AssetExecutionContext
)
from dagster_dbt import DbtCliResource, dbt_assets

daily_partition = DailyPartitionsDefinition(start_date="2025-01-01")

@dbt_assets(
    manifest="/opt/dbt/target/manifest.json",
    partitions_def=daily_partition,
    op_tags={"team": "analytics", "sla": "4_hours"}
)
def partitioned_dbt_models(context: AssetExecutionContext, dbt: DbtCliResource):
    partition_date = context.partition_key
    
    # Run dbt with partition-specific vars
    yield from dbt.cli(
        [
            "build",
            "--select", "tag:partitioned",
            "--vars", f"{{partition_date: {partition_date}}}"
        ],
        context=context
    ).stream()

defs = Definitions(
    assets=[partitioned_dbt_models],
    resources={
        "dbt": DbtCliResource(
            project_dir="/opt/dbt",
            profiles_dir="/opt/dbt",
            target="prod"
        )
    }
)
```

---

## 8.7 Orchestration: Meltano

### What is Meltano Integration?
Meltano is an ELT orchestration platform with native dbt integration. Combines extractors (taps), loaders (targets), and dbt transformations in a single config file.

**Diagram: Meltano Pipeline**
```text
Meltano Schedule (daily cron)
     ↓ tap-postgres (extractor)
Raw JSON files (data/raw/)
     ↓ target-snowflake (loader)
Snowflake RAW schema
     ↓ dbt transformations
ANALYTICS schema (marts)
```

### Syntax: meltano.yml

**BASIC EXAMPLE**
```yaml
# meltano.yml
version: 1
project_id: my_meltano_project

plugins:
  extractors:
    - name: tap-postgres
      variant: transferwise
      pip_url: pipelinewise-tap-postgres
      config:
        host: postgres.example.com
        port: 5432
        dbname: production
        user: readonly
        password: ${POSTGRES_PASSWORD}
  
  loaders:
    - name: target-snowflake
      variant: datamill-co
      pip_url: target-snowflake
      config:
        account: abc12345
        warehouse: LOADING_WH
        database: RAW_DATA
        schema: postgres_replica
  
  transformers:
    - name: dbt-snowflake
      pip_url: dbt-core~=1.7.0 dbt-snowflake~=1.7.0
      config:
        profiles_dir: transform/profiles
        target: prod

schedules:
  - name: daily_etl
    interval: '0 2 * * *'  # 2am UTC
    job: tap-postgres-to-snowflake-dbt
    
jobs:
  - name: tap-postgres-to-snowflake-dbt
    tasks:
      - tap-postgres target-snowflake
      - dbt-snowflake:run
      - dbt-snowflake:test
```

**Run Meltano Pipeline**
```bash
# Install dependencies
meltano install

# Run extraction + loading + transformation
meltano run tap-postgres target-snowflake dbt-snowflake:run dbt-snowflake:test

# Schedule via cron or Airflow
meltano schedule run daily_etl
```

---

## 8.8 BI Tool Integration

### Tableau
**dbt Exposures for Tableau Dashboards**
```yaml
# models/marts/exposures.yml
exposures:
  - name: executive_sales_dashboard
    type: dashboard
    maturity: high
    url: https://tableau.company.com/#/views/SalesExec/Dashboard1
    description: Executive sales metrics (revenue, orders, customers)
    
    depends_on:
      - ref('fct_sales')
      - ref('dim_customers')
      - ref('dim_products')
    
    owner:
      name: Sales Analytics Team
      email: sales-analytics@company.com
```

**Tableau Integration via dbt Docs**
```bash
# Generate dbt docs with lineage
dbt docs generate

# Tableau users reference dbt docs catalog for column descriptions
# https://dbt-docs.company.com/#!/model/model.my_project.fct_sales
```

### Looker
**LookML Integration**
```lookml
# looker_models/fct_orders.view.lkml
# Auto-generated from dbt via dbt2looker
view: fct_orders {
  sql_table_name: ANALYTICS.DBT_PROD.FCT_ORDERS ;;
  
  dimension: order_id {
    type: string
    sql: ${TABLE}.order_id ;;
    primary_key: yes
  }
  
  dimension_group: order_date {
    type: time
    timeframes: [date, week, month, year]
    sql: ${TABLE}.order_date ;;
  }
  
  measure: total_revenue {
    type: sum
    sql: ${TABLE}.order_amount ;;
    value_format_name: usd
  }
}
```

**dbt2looker Configuration**
```yaml
# dbt2looker.yml
connection_name: snowflake_prod
output_dir: looker/views
model_path: models/marts
tags_to_include: ['looker']
```

### Power BI
**Power BI Integration via DirectQuery**
```python
# Power BI M Language query (DirectQuery to Snowflake)
let
    Source = Snowflake.Databases("abc12345.snowflakecomputing.com", "ANALYTICS_DB"),
    DBT_PROD_Schema = Source{[Name="DBT_PROD"]}[Data],
    FCT_SALES_Table = DBT_PROD_Schema{[Name="FCT_SALES"]}[Data]
in
    FCT_SALES_Table
```

**dbt Model Documentation → Power BI Metadata**
```yaml
# models/marts/fct_sales.yml
models:
  - name: fct_sales
    description: Daily sales transactions with customer and product dimensions
    columns:
      - name: sale_id
        description: Unique sale identifier (primary key)
        tests:
          - unique
          - not_null
      
      - name: sale_date
        description: Transaction date (UTC)
        meta:
          power_bi_data_type: date
      
      - name: revenue
        description: Sale amount in USD
        meta:
          power_bi_format: currency
          power_bi_aggregation: sum
```

---

## 8.9 Metadata Integrations

### dbt + DataHub (LinkedIn)
```yaml
# .github/workflows/push_to_datahub.yml
name: Push dbt Metadata to DataHub

on:
  push:
    branches: [main]

jobs:
  metadata_sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Generate dbt artifacts
        run: |
          dbt compile
          dbt docs generate
      
      - name: Push to DataHub
        run: |
          pip install acryl-datahub
          datahub ingest -c datahub_config.yml
```

**datahub_config.yml**
```yaml
source:
  type: dbt
  config:
    manifest_path: target/manifest.json
    catalog_path: target/catalog.json
    target_platform: snowflake
    platform_instance: prod

sink:
  type: datahub-rest
  config:
    server: https://datahub.company.com
    token: ${DATAHUB_TOKEN}
```

### dbt + Monte Carlo (Data Observability)
```yaml
# Install Monte Carlo agent
# Monitors dbt test failures, data freshness, schema changes

# Monte Carlo config
monitors:
  - type: dbt_test
    description: Monitor all dbt test failures
    schedule: "0 * * * *"  # Hourly
    alert_channels: [slack, pagerduty]
  
  - type: freshness
    table: ANALYTICS.DBT_PROD.FCT_ORDERS
    threshold: 6 hours
    schedule: "*/30 * * * *"  # Every 30 min
```

---

## 8.10 SDLC Example: Multi-Tool Orchestration

**Complete Pipeline: Fivetran → Airflow → dbt → Looker**
```python
# airflow/dags/full_etl_pipeline.py
from airflow import DAG
from airflow.providers.fivetran.operators.fivetran import FivetranOperator
from airflow.providers.fivetran.sensors.fivetran import FivetranSensor
from cosmos import DbtTaskGroup, ProfileConfig, ProjectConfig
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG(
    'full_etl_pipeline',
    schedule_interval='0 1 * * *',  # 1am UTC
    start_date=datetime(2025, 1, 1),
    catchup=False
) as dag:
    
    # Step 1: Fivetran sync (Salesforce → Snowflake)
    fivetran_sync = FivetranOperator(
        task_id='fivetran_salesforce_sync',
        connector_id='{{ var.value.fivetran_salesforce_connector }}',
        fivetran_conn_id='fivetran_default'
    )
    
    # Step 2: Wait for Fivetran completion
    wait_for_sync = FivetranSensor(
        task_id='wait_for_fivetran',
        connector_id='{{ var.value.fivetran_salesforce_connector }}',
        fivetran_conn_id='fivetran_default',
        poke_interval=60,
        timeout=3600
    )
    
    # Step 3: dbt transformations (via Cosmos)
    dbt_transform = DbtTaskGroup(
        group_id='dbt_models',
        project_config=ProjectConfig('/opt/dbt/my_project'),
        profile_config=ProfileConfig(
            profile_name='my_project',
            target_name='prod'
        )
    )
    
    # Step 4: Refresh Looker dashboards
    refresh_looker = BashOperator(
        task_id='refresh_looker_dashboards',
        bash_command='''
            curl -X POST https://looker.company.com/api/dashboards/123/refresh \
              -H "Authorization: Bearer $LOOKER_TOKEN"
        '''
    )
    
    fivetran_sync >> wait_for_sync >> dbt_transform >> refresh_looker
```

---

## Summary

**Adapters**: Snowflake (clustering), BigQuery (partitioning), Redshift (sort keys) require adapter-specific optimizations for production performance.

**Orchestration**: 
- **Airflow**: BashOperator or Cosmos for dbt tasks
- **Dagster**: dbt models → Dagster assets with lineage
- **Meltano**: Unified ELT config (extractors + dbt)

**BI Tools**: Tableau/Looker/Power BI consume dbt models via DirectQuery or LookML generation.

**SDLC Best Practice**: Use exposures to track downstream dependencies, orchestrate dbt with state artifacts for slim CI/CD.
 
 ---
