# 8. Integration & Ecosystem

## Overview

Snowflake integrates seamlessly with modern data stacks via native connectors, ODBC/JDBC drivers, and partner integrations. The platform supports bidirectional data flow with BI tools (Tableau, Power BI, Looker), ETL/ELT frameworks (dbt, Informatica, Fivetran), and custom applications through Python, Spark, and REST APIs. All integrations leverage Snowflake's security model (OAuth, key-pair auth) and support role-based access control.

---

## 8.1 Business Intelligence (BI) Integrations

### Tableau Integration

Snowflake provides native Tableau connectors with live query and extract modes. Live connections push query execution to Snowflake warehouses, eliminating data movement. Extracts leverage Snowflake's query optimization for fast initial loads.

**[Diagram: Tableau-Snowflake Architecture]**
```
┌──────────────┐  ODBC/JDBC   ┌─────────────────┐
│   Tableau    │◄────────────►│  Snowflake VW   │
│   Desktop/   │  SQL Queries │  (Compute)      │
│   Server     │◄─────────────┤  Result Cache   │
└──────────────┘  Live Data   └─────────────────┘
```

**Connection Configuration:**
```yaml
# Tableau Connection String (tableau.tdc file)
<?xml version='1.0' encoding='utf-8'?>
<connection-customization class='snowflake' enabled='true' version='7.7'>
  <vendor name='snowflake'/>
  <driver name='snowflake'/>
  <customizations>
    <customization name='CAP_ODBC_METADATA_SUPPRESS_EXECUTED_QUERY' value='yes'/>
    <customization name='CAP_QUERY_GROUP_BY_DEGREE' value='yes'/>
    <customization name='CAP_QUERY_TOP_N' value='yes'/>
    <customization name='CAP_SELECT_TOP_INTO' value='yes'/>
  </customizations>
</connection-customization>
```

**SDLC Use Case: CI/CD Dashboard Deployment**
In DevOps pipelines, use Tableau REST API + Snowflake to automate dashboard publishing:
1. dbt runs transformations → updates `ANALYTICS.PROD.DASHBOARD_METRICS` view
2. GitHub Action triggers Tableau Server workbook refresh via REST API
3. QA environment uses `ANALYTICS.QA.DASHBOARD_METRICS` (separate warehouse)
4. Production deploy swaps connection from QA to PROD schema

```python
# GitHub Action: Automated Tableau Refresh
import tableauserverclient as TSC
import snowflake.connector

# Refresh Snowflake materialized view
ctx = snowflake.connector.connect(
    account='myaccount',
    warehouse='TABLEAU_WH',
    database='ANALYTICS',
    schema='PROD'
)
ctx.cursor().execute("ALTER MATERIALIZED VIEW dashboard_metrics REFRESH")

# Trigger Tableau extract refresh
tableau_auth = TSC.PersonalAccessTokenAuth('token_name', 'token_secret', site_id='')
server = TSC.Server('https://tableau.company.com', use_server_version=True)
server.auth.sign_in(tableau_auth)
workbook = server.workbooks.get_by_id('workbook-uuid')
server.workbooks.refresh(workbook)
```

### Power BI Integration

Power BI connects via native Snowflake connector (DirectQuery or Import mode). DirectQuery enables real-time dashboards without data duplication. Import mode caches data in Power BI for faster rendering but requires scheduled refreshes.

**Power BI Connection Parameters:**
```
# Direct Query Mode (Live Connection)
Server: myaccount.snowflakecomputing.com
Warehouse: POWERBI_WH
Database: SALES_DB
Authentication: Username/Password or AAD OAuth

# Import Mode (Scheduled Refresh)
Data Refresh: Via Power BI Gateway
Incremental Refresh: Filter on UPDATED_AT >= @RefreshDate
```

**Advanced Example: Incremental Refresh Policy**
```sql
-- Create date-partitioned table for Power BI incremental refresh
CREATE OR REPLACE TABLE sales_transactions (
    transaction_id NUMBER,
    transaction_date DATE,
    amount DECIMAL(10,2),
    customer_id NUMBER
) CLUSTER BY (transaction_date);

-- Power BI M Query for Incremental Refresh
let
    Source = Snowflake.Databases("myaccount.snowflakecomputing.com", "SALES_DB", [Warehouse="POWERBI_WH"]),
    FilteredRows = Table.SelectRows(Source, each [TRANSACTION_DATE] >= RangeStart and [TRANSACTION_DATE] < RangeEnd)
in
    FilteredRows
```

### Looker Integration

Looker uses LookML to model Snowflake data with persistent derived tables (PDTs). PDTs materialize complex queries as tables in Snowflake, improving dashboard performance. Looker's SQL Runner allows direct query execution for ad-hoc analysis.

**LookML Connection Configuration:**
```yaml
# looker_connection.lkml
connection: "snowflake_prod" {
  host: "myaccount.snowflakecomputing.com"
  port: "443"
  database: "ANALYTICS"
  warehouse: "LOOKER_WH"
  schema: "PUBLIC"
  username: "looker_svc"
  password: "@{looker_password}"
  
  # Connection pooling
  max_connections: 50
  pool_timeout: 300
  
  # Query optimizations
  query_timezone: "America/New_York"
  ssl: true
}

# Persistent Derived Table (PDT)
view: daily_revenue {
  derived_table: {
    sql: 
      SELECT 
        DATE_TRUNC('day', order_date) AS order_day,
        SUM(amount) AS total_revenue,
        COUNT(DISTINCT customer_id) AS unique_customers
      FROM orders
      WHERE order_date >= DATEADD(day, -90, CURRENT_DATE)
      GROUP BY 1 ;;
    
    # Materialized in Snowflake
    datagroup_trigger: daily_refresh
    distribution_style: all
    sortkeys: ["order_day"]
  }
  
  dimension: order_day { type: date }
  measure: total_revenue { type: sum }
}
```

---

## 8.2 ETL/ELT Tool Integrations

### dbt (Data Build Tool)

dbt transforms data in Snowflake using SQL-based models, tests, and documentation. It compiles Jinja templates into SQL and orchestrates execution order via DAGs. dbt Cloud provides CI/CD integration for automated testing on pull requests.

**dbt Project Configuration:**

```yaml
# profiles.yml
snowflake_prod:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: myaccount
      user: dbt_dev_user
      password: "{{ env_var('DBT_DEV_PASSWORD') }}"
      role: DBT_DEV_ROLE
      warehouse: DBT_DEV_WH
      database: ANALYTICS_DEV
      schema: dbt_{{ env_var('USER') }}
      threads: 8
      client_session_keep_alive: true
      query_tag: dbt_dev
      
    prod:
      type: snowflake
      account: myaccount
      user: dbt_prod_user
      authenticator: oauth
      token: "{{ env_var('DBT_OAUTH_TOKEN') }}"
      role: DBT_PROD_ROLE
      warehouse: DBT_PROD_WH
      database: ANALYTICS
      schema: PUBLIC
      threads: 16
      query_tag: dbt_prod
```

**dbt Model with Snowflake-Specific Syntax:**
```sql
-- models/marts/finance/daily_revenue.sql
{{
  config(
    materialized='incremental',
    unique_key='revenue_date',
    cluster_by=['revenue_date'],
    pre_hook="ALTER SESSION SET TIMEZONE = 'America/New_York'",
    post_hook="GRANT SELECT ON {{ this }} TO ROLE ANALYST_ROLE"
  )
}}

SELECT
    DATE_TRUNC('day', order_timestamp) AS revenue_date,
    SUM(order_amount) AS total_revenue,
    COUNT(DISTINCT customer_id) AS unique_customers,
    CURRENT_TIMESTAMP() AS dbt_updated_at
FROM {{ ref('stg_orders') }}
WHERE order_status = 'COMPLETED'

{% if is_incremental() %}
    AND order_timestamp > (SELECT MAX(revenue_date) FROM {{ this }})
{% endif %}

GROUP BY 1
```

**SDLC Use Case: dbt CI/CD Pipeline**
```yaml
# .github/workflows/dbt_ci.yml
name: dbt CI/CD Pipeline
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  dbt_test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dbt
        run: pip install dbt-snowflake==1.7.0
      
      - name: dbt deps
        run: dbt deps
        
      - name: dbt build (PR environment)
        if: github.event_name == 'pull_request'
        run: |
          dbt build --target dev --select state:modified+
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          DBT_DEV_PASSWORD: ${{ secrets.DBT_DEV_PASSWORD }}
      
      - name: dbt build (Production)
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: dbt build --target prod
        env:
          DBT_OAUTH_TOKEN: ${{ secrets.DBT_OAUTH_TOKEN }}
```

### Informatica Integration

Informatica Intelligent Cloud Services (IICS) connects to Snowflake via ODBC/JDBC for ETL mappings. Native pushdown optimization executes transformations in Snowflake compute rather than Informatica servers. Use Snowflake warehouses for data loading and transformations.

**Informatica Connection Properties:**
```
Connection Type: Snowflake
Host: myaccount.snowflakecomputing.com
Port: 443
Database: SALES_DB
Schema: PUBLIC
Warehouse: INFORMATICA_WH
User: informatica_user
Authentication: Key Pair (RSA 2048-bit)
Private Key Path: /path/to/rsa_key.p8

# Advanced Settings
JDBC URL: jdbc:snowflake://myaccount.snowflakecomputing.com/?warehouse=INFORMATICA_WH&db=SALES_DB
Query Band: session=informatica_job_id=12345
Client Session Keep Alive: true
Network Timeout: 300
```

**Pushdown Optimization Example:**
```sql
-- Informatica mapping translates to Snowflake SQL (pushdown enabled)
-- Source: CUSTOMERS (10M rows), ORDERS (50M rows)
-- Transformation: Join + Aggregate → Executed in Snowflake warehouse

INSERT INTO customer_order_summary
SELECT 
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) AS total_orders,
    SUM(o.order_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= DATEADD(year, -1, CURRENT_DATE)
GROUP BY c.customer_id, c.customer_name;

-- Result: 300 seconds (Snowflake XL warehouse) vs 900 seconds (Informatica server processing)
```

### Fivetran Integration

Fivetran automates ELT pipelines from SaaS applications, databases, and event streams to Snowflake. Connectors sync data incrementally using change data capture (CDC) or timestamp-based detection. Fivetran manages schema evolution and data type mappings automatically.

**Fivetran Connector Configuration:**
```json
{
  "destination": {
    "service": "snowflake",
    "config": {
      "host": "myaccount.snowflakecomputing.com",
      "port": 443,
      "database": "FIVETRAN_DB",
      "user": "fivetran_user",
      "password": "encrypted_password",
      "role": "FIVETRAN_ROLE",
      "warehouse": "FIVETRAN_WH",
      "connection_type": "Directly"
    }
  },
  "source": {
    "service": "salesforce",
    "config": {
      "domain": "na1.salesforce.com",
      "api_version": "v58.0",
      "sync_mode": "INCREMENTAL",
      "replication_slot": "fivetran_salesforce_slot"
    }
  },
  "sync_frequency": "6_hours",
  "paused": false
}
```

---

## 8.3 Programmatic Access

### Python Connector

The Snowflake Python connector enables programmatic SQL execution, data loading, and result fetching. It supports multiple authentication methods (username/password, key pair, OAuth, SSO) and connection pooling for high-throughput applications.

**Syntax: snowflake.connector.connect()**

```python
import snowflake.connector

# SYNTAX TEMPLATE
conn = snowflake.connector.connect(
    account='<account_identifier>',
    user='<username>',
    password='<password>',  # OR authenticator / private_key
    warehouse='<warehouse_name>',
    database='<database_name>',
    schema='<schema_name>',
    role='<role_name>',
    authenticator='<auth_method>',  # snowflake (default), externalbrowser, oauth
    token='<oauth_token>',
    private_key='<rsa_private_key>',
    session_parameters={
        'QUERY_TAG': '<query_tag>',
        'TIMEZONE': '<timezone>',
        'ROWS_PER_RESULTSET': <num>
    },
    client_session_keep_alive=<boolean>,
    network_timeout=<seconds>,
    login_timeout=<seconds>,
    ocsp_response_cache_filename='<cache_file>',
    application='<app_name>',
    protocol='https',
    port=443
)
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| account | String | Required | Snowflake account identifier | 'myorg-myaccount' |
| user | String | Required | Username for authentication | 'john_doe' |
| password | String | None | Password (if not using key pair/OAuth) | 'SecurePass123!' |
| authenticator | String | 'snowflake' | Auth method: snowflake, externalbrowser, oauth, snowflake_jwt | 'externalbrowser' |
| warehouse | String | None | Default virtual warehouse | 'COMPUTE_WH' |
| database | String | None | Default database | 'ANALYTICS' |
| schema | String | None | Default schema | 'PUBLIC' |
| role | String | None | Default role | 'DATA_ENGINEER' |
| private_key | Bytes | None | RSA private key for key-pair auth | b'-----BEGIN PRIVATE KEY-----...' |
| token | String | None | OAuth access token | 'ya29.a0AfH6SMB...' |
| session_parameters | Dict | {} | Session-level settings | {'QUERY_TAG': 'python_etl'} |
| client_session_keep_alive | Bool | False | Keep session alive (heartbeat) | True |
| network_timeout | Int | None | Socket timeout (seconds) | 300 |
| login_timeout | Int | 120 | Login timeout (seconds) | 60 |
| application | String | 'PythonConnector' | Application identifier for logging | 'MyETLApp' |

**BASIC EXAMPLE**
```python
import snowflake.connector

# Simple connection with username/password
conn = snowflake.connector.connect(
    account='myaccount',
    user='analyst',
    password='MyPassword123',
    warehouse='COMPUTE_WH',
    database='SALES_DB'
)

cur = conn.cursor()
cur.execute("SELECT COUNT(*) FROM orders")
result = cur.fetchone()
print(f"Total orders: {result[0]}")

cur.close()
conn.close()
```

**ADVANCED EXAMPLE (SDLC: Production ETL with Key-Pair Auth)**
```python
import snowflake.connector
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
import os
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Load RSA private key from environment
with open(os.environ['SNOWFLAKE_PRIVATE_KEY_PATH'], 'rb') as key_file:
    private_key = serialization.load_pem_private_key(
        key_file.read(),
        password=os.environ['SNOWFLAKE_KEY_PASSPHRASE'].encode(),
        backend=default_backend()
    )

# Production connection with key-pair auth + session params
conn = snowflake.connector.connect(
    account='prodaccount',
    user='etl_service_account',
    private_key=private_key.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    ),
    warehouse='ETL_PROD_WH',
    database='DATAWAREHOUSE',
    schema='STAGING',
    role='ETL_ROLE',
    session_parameters={
        'QUERY_TAG': 'daily_etl_pipeline',
        'TIMEZONE': 'UTC',
        'STATEMENT_TIMEOUT_IN_SECONDS': 3600
    },
    client_session_keep_alive=True,
    network_timeout=600,
    application='ProductionETL_v2.1'
)

try:
    # Execute multi-statement transaction
    cur = conn.cursor()
    
    # Begin transaction
    cur.execute("BEGIN TRANSACTION")
    
    # Truncate staging table
    cur.execute("TRUNCATE TABLE staging.temp_orders")
    logger.info("Staging table truncated")
    
    # Bulk insert from internal stage
    cur.execute("""
        COPY INTO staging.temp_orders
        FROM @staging.etl_stage/orders/
        FILE_FORMAT = (TYPE = 'PARQUET')
        PATTERN = '.*orders_[0-9]{8}\\.parquet'
        ON_ERROR = 'ABORT_STATEMENT'
    """)
    logger.info(f"Loaded {cur.rowcount} rows into staging")
    
    # Merge into production table
    cur.execute("""
        MERGE INTO production.orders AS target
        USING staging.temp_orders AS source
        ON target.order_id = source.order_id
        WHEN MATCHED THEN 
            UPDATE SET 
                order_status = source.order_status,
                updated_at = CURRENT_TIMESTAMP()
        WHEN NOT MATCHED THEN
            INSERT (order_id, customer_id, order_amount, order_status, created_at)
            VALUES (source.order_id, source.customer_id, source.order_amount, 
                    source.order_status, CURRENT_TIMESTAMP())
    """)
    logger.info(f"Merged {cur.rowcount} rows into production")
    
    # Commit transaction
    cur.execute("COMMIT")
    logger.info("Transaction committed successfully")
    
except Exception as e:
    cur.execute("ROLLBACK")
    logger.error(f"ETL failed, transaction rolled back: {str(e)}")
    raise
finally:
    cur.close()
    conn.close()
```

### Snowflake Connector for Spark

The Snowflake Connector for Spark enables bidirectional data transfer between Spark DataFrames and Snowflake tables. It uses Snowflake's internal stages for efficient bulk loading and supports pushdown optimization for filters and aggregations.

**Syntax: Spark Connector Configuration**

```python
from pyspark.sql import SparkSession

# CONFIGURATION PARAMETERS
sfOptions = {
    # Connection
    "sfURL": "<account_identifier>.snowflakecomputing.com",
    "sfUser": "<username>",
    "sfPassword": "<password>",  # OR use sfAuthenticator + sfToken
    "sfDatabase": "<database_name>",
    "sfSchema": "<schema_name>",
    "sfWarehouse": "<warehouse_name>",
    "sfRole": "<role_name>",
    
    # Authentication
    "sfAuthenticator": "<auth_method>",  # snowflake (default), externalbrowser, oauth
    "sfToken": "<oauth_token>",
    "pem_private_key": "<path_to_key>",
    
    # Performance
    "parallelism": "<num_partitions>",  # Number of Spark partitions
    "use_cached_result": "<boolean>",   # Reuse query results if available
    "keep_column_case": "<sensitive|insensitive>",  # Preserve column name case
    "autopushdown": "<on|off>",         # Enable query pushdown to Snowflake
    
    # Data Loading
    "streaming_stage": "<stage_name>",  # Internal stage for temp data
    "purge": "<boolean>",               # Delete staged files after load
    "column_mapping": "<name|order>",   # Column matching strategy
    "truncate_table": "<on|off>",       # Truncate before write
    
    # Query Optimization
    "partition_size_in_mb": "<num>",    # Target partition size (default: 100MB)
    "time_output_format": "<format>",   # Timestamp format string
}
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| sfURL | String | Required | Snowflake account URL (without https://) | 'myaccount.snowflakecomputing.com' |
| sfUser | String | Required | Snowflake username | 'spark_user' |
| sfPassword | String | None | Password (if not using OAuth/key-pair) | 'SecurePass123' |
| sfWarehouse | String | Required | Virtual warehouse for compute | 'SPARK_WH' |
| sfDatabase | String | Required | Target database | 'ANALYTICS' |
| sfSchema | String | Required | Target schema | 'PUBLIC' |
| sfRole | String | None | Role to use | 'SPARK_ROLE' |
| sfAuthenticator | String | 'snowflake' | Auth method | 'externalbrowser' |
| parallelism | Int | 1 | Spark partition count | 8 |
| autopushdown | String | 'on' | Enable filter/aggregate pushdown | 'on' |
| purge | Bool | false | Delete staged files after load | true |
| truncate_table | String | 'off' | Truncate table before write | 'on' |

**BASIC EXAMPLE**
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("SnowflakeIntegration") \
    .config("spark.jars.packages", "net.snowflake:spark-snowflake_2.12:2.12.0-spark_3.4") \
    .getOrCreate()

sfOptions = {
    "sfURL": "myaccount.snowflakecomputing.com",
    "sfUser": "spark_user",
    "sfPassword": "MyPassword",
    "sfDatabase": "SALES_DB",
    "sfSchema": "PUBLIC",
    "sfWarehouse": "COMPUTE_WH"
}

# Read from Snowflake
df = spark.read \
    .format("snowflake") \
    .options(**sfOptions) \
    .option("dbtable", "orders") \
    .load()

df.show(10)
```

**ADVANCED EXAMPLE (SDLC: Spark ETL with Pushdown Optimization)**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, count, year, month
import os

# Production Spark session with Snowflake connector
spark = SparkSession.builder \
    .appName("Production_ETL_Snowflake") \
    .config("spark.jars.packages", "net.snowflake:spark-snowflake_2.12:2.12.0-spark_3.4") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.executor.memory", "8g") \
    .config("spark.executor.cores", "4") \
    .getOrCreate()

# Production connection with OAuth
sfOptions = {
    "sfURL": "prodaccount.snowflakecomputing.com",
    "sfUser": "spark_svc_account",
    "sfAuthenticator": "oauth",
    "sfToken": os.environ['SNOWFLAKE_OAUTH_TOKEN'],
    "sfDatabase": "ANALYTICS",
    "sfSchema": "STAGING",
    "sfWarehouse": "SPARK_PROD_WH",
    "sfRole": "SPARK_ETL_ROLE",
    
    # Performance optimizations
    "parallelism": "16",              # 16 Spark partitions
    "autopushdown": "on",             # Push filters to Snowflake
    "use_cached_result": "true",      # Reuse query cache
    "partition_size_in_mb": "128",    # 128MB partitions
    "keep_column_case": "sensitive",  # Preserve case
    "purge": "true"                   # Clean up staged files
}

# Read large table with filter pushdown
# Snowflake executes: SELECT * FROM orders WHERE order_year = 2024
orders_df = spark.read \
    .format("snowflake") \
    .options(**sfOptions) \
    .option("dbtable", "orders") \
    .load() \
    .filter(year(col("order_date")) == 2024)

# Spark aggregation (executed in Spark, not pushed down)
monthly_summary = orders_df.groupBy(
    year(col("order_date")).alias("order_year"),
    month(col("order_date")).alias("order_month")
).agg(
    sum(col("order_amount")).alias("total_revenue"),
    count(col("order_id")).alias("order_count")
)

# Write aggregated results back to Snowflake
monthly_summary.write \
    .format("snowflake") \
    .options(**sfOptions) \
    .option("dbtable", "monthly_revenue_summary") \
    .option("truncate_table", "on") \
    .option("column_mapping", "name") \
    .mode("overwrite") \
    .save()

print("ETL completed: Aggregated 2024 orders and wrote to monthly_revenue_summary")
spark.stop()
```

---

## 8.4 REST API Integration

### Snowflake SQL API

The SQL API executes SQL statements via HTTPS requests without persistent connections. It supports asynchronous execution for long-running queries and returns results in JSON format. Ideal for serverless applications, webhooks, and microservices.

**API Endpoint Structure:**
```
POST https://<account_identifier>.snowflakecomputing.com/api/v2/statements
Authorization: Bearer <oauth_token>  OR  Basic <base64_encoded_credentials>
Content-Type: application/json
X-Snowflake-Authorization-Token-Type: KEYPAIR (if using JWT)

Request Body:
{
  "statement": "<SQL_STATEMENT>",
  "timeout": <seconds>,
  "database": "<database_name>",
  "schema": "<schema_name>",
  "warehouse": "<warehouse_name>",
  "role": "<role_name>",
  "bindings": {
    "1": {"type": "FIXED", "value": "<value>"},
    "2": {"type": "TEXT", "value": "<value>"}
  },
  "parameters": {
    "QUERY_TAG": "<tag>",
    "TIMEZONE": "<tz>"
  },
  "resultSetMetaData": {
    "format": "json"  # or "arrow"
  }
}
```

**Example: Asynchronous Query Execution**
```python
import requests
import json
import time

# SQL API configuration
ACCOUNT = "myaccount"
API_URL = f"https://{ACCOUNT}.snowflakecomputing.com/api/v2/statements"
OAUTH_TOKEN = "your_oauth_token"

headers = {
    "Authorization": f"Bearer {OAUTH_TOKEN}",
    "Content-Type": "application/json",
    "X-Snowflake-Authorization-Token-Type": "OAUTH"
}

# Submit async query
query_request = {
    "statement": "SELECT customer_id, SUM(order_amount) FROM orders GROUP BY customer_id",
    "timeout": 3600,
    "database": "ANALYTICS",
    "schema": "PUBLIC",
    "warehouse": "API_WH",
    "role": "API_ROLE",
    "async": True,  # Asynchronous execution
    "parameters": {
        "QUERY_TAG": "api_aggregation_query"
    }
}

# POST query
response = requests.post(API_URL, headers=headers, json=query_request)
response_data = response.json()
statement_handle = response_data['statementHandle']

print(f"Query submitted: {statement_handle}")

# Poll for completion
status_url = f"{API_URL}/{statement_handle}"
while True:
    status_response = requests.get(status_url, headers=headers)
    status_data = status_response.json()
    
    if status_data['statementStatusUrl']:
        print(f"Query running... Rows processed: {status_data.get('rowsProduced', 0)}")
        time.sleep(5)
    else:
        print("Query completed")
        results = status_data['data']
        print(f"Total rows: {len(results)}")
        break
```

**SDLC Use Case: Serverless Lambda Function**
```python
# AWS Lambda function for on-demand Snowflake aggregation
import json
import requests
import os

def lambda_handler(event, context):
    """
    Triggered by API Gateway: /aggregate?metric=revenue&date=2024-01-15
    Returns aggregated metrics from Snowflake via SQL API
    """
    
    metric = event['queryStringParameters']['metric']
    date = event['queryStringParameters']['date']
    
    # SQL API request
    api_url = f"https://{os.environ['SNOWFLAKE_ACCOUNT']}.snowflakecomputing.com/api/v2/statements"
    headers = {
        "Authorization": f"Bearer {os.environ['SNOWFLAKE_OAUTH_TOKEN']}",
        "Content-Type": "application/json"
    }
    
    query = {
        "statement": f"""
            SELECT 
                SUM(order_amount) AS total_{metric},
                COUNT(*) AS order_count
            FROM orders
            WHERE order_date = ?
        """,
        "bindings": {
            "1": {"type": "DATE", "value": date}
        },
        "database": "ANALYTICS",
        "warehouse": "LAMBDA_WH",
        "timeout": 60
    }
    
    response = requests.post(api_url, headers=headers, json=query)
    result = response.json()
    
    return {
        'statusCode': 200,
        'body': json.dumps(result['data'][0])
    }
```

---

## 8.5 SDLC Integration Patterns

### CI/CD Pipeline Integration

**Pattern 1: Schema Change Management**
```yaml
# GitLab CI/CD: Automated schema migrations
stages:
  - validate
  - deploy_dev
  - deploy_prod

validate_sql:
  stage: validate
  script:
    - pip install sqlfluff
    - sqlfluff lint migrations/*.sql --dialect snowflake
  only:
    - merge_requests

deploy_dev:
  stage: deploy_dev
  script:
    - |
      python << EOF
      import snowflake.connector
      conn = snowflake.connector.connect(
          account=os.environ['SF_ACCOUNT'],
          user=os.environ['SF_USER'],
          private_key=os.environ['SF_PRIVATE_KEY'],
          warehouse='DEV_WH',
          database='DEV_DB',
          role='DEPLOY_ROLE'
      )
      with open('migrations/V001__create_tables.sql') as f:
          conn.cursor().execute(f.read())
      EOF
  environment:
    name: development

deploy_prod:
  stage: deploy_prod
  script:
    - snowsql -a $SF_ACCOUNT -u $SF_USER -f migrations/V001__create_tables.sql
  environment:
    name: production
  when: manual
  only:
    - main
```

**Pattern 2: Data Quality Testing**
```python
# pytest + Snowflake: Automated data validation
import pytest
import snowflake.connector

@pytest.fixture(scope='module')
def snowflake_conn():
    conn = snowflake.connector.connect(
        account='testaccount',
        user='test_user',
        warehouse='TEST_WH',
        database='QA_DB',
        schema='PUBLIC'
    )
    yield conn
    conn.close()

def test_orders_no_nulls(snowflake_conn):
    """Verify critical columns have no NULLs"""
    cur = snowflake_conn.cursor()
    cur.execute("""
        SELECT COUNT(*) FROM orders
        WHERE customer_id IS NULL OR order_amount IS NULL
    """)
    assert cur.fetchone()[0] == 0, "Found NULL values in critical columns"

def test_daily_order_count(snowflake_conn):
    """Verify daily order volume meets threshold"""
    cur = snowflake_conn.cursor()
    cur.execute("""
        SELECT COUNT(*) FROM orders
        WHERE order_date = CURRENT_DATE - 1
    """)
    assert cur.fetchone()[0] >= 1000, "Order count below expected threshold"
```

---

## Glossary of Integration Terms

- **Pushdown Optimization**: Executing transformations in Snowflake compute rather than external tools, reducing data movement
- **Persistent Derived Table (PDT)**: Looker feature materializing complex queries as physical tables in Snowflake
- **Incremental Refresh**: Power BI/Tableau pattern loading only new/changed data since last refresh
- **Change Data Capture (CDC)**: Fivetran technique tracking database changes for incremental sync
- **Query Pushdown**: Spark/Informatica delegating filter/aggregate operations to Snowflake
- **LookML**: Looker's modeling language defining metrics and dimensions on top of Snowflake tables
- **dbt Model**: SQL SELECT transformed into table/view via Jinja templating and DAG execution
- **OAuth Token**: Time-limited credential for API authentication, eliminating password storage
- **Key-Pair Authentication**: RSA public/private key auth for service accounts, more secure than passwords
- **Connection Pooling**: Reusing database connections across requests to reduce overhead
- **Client Session Keep-Alive**: Heartbeat mechanism preventing idle connection timeouts
- **Statement Handle**: Unique identifier for asynchronous SQL API queries
- **Result Set Metadata**: Schema information (column names, types) returned with query results
- **Query Tag**: Label attached to queries for cost tracking and performance monitoring
- **Snowflake Connector JAR**: Java library enabling Spark-Snowflake data transfer
- **Internal Stage**: Snowflake-managed storage for temporary data during bulk loads
