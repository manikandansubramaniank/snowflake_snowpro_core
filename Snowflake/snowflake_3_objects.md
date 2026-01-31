# 3. Snowflake Objects

## Object Hierarchy

Snowflake organizes objects in a three-level namespace: `DATABASE.SCHEMA.OBJECT`. All objects exist within schemas, which exist within databases. Proper object organization enables role-based access control, cost tracking via tags, and environment separation (dev/staging/prod).

```
ACCOUNT
 ├── DATABASE: analytics_db
 │    ├── SCHEMA: sales
 │    │    ├── TABLE: fact_orders
 │    │    ├── VIEW: v_monthly_sales
 │    │    ├── STREAM: orders_stream
 │    │    └── TASK: refresh_aggregates
 │    ├── SCHEMA: marketing
 │    └── SCHEMA: finance
 ├── DATABASE: staging_db
 └── WAREHOUSE: analytics_wh (not in namespace hierarchy)
```

---

## Tables

Snowflake supports three table types with different durability and cost characteristics. All tables use micro-partitions and support Time Travel (duration varies by type).

### Table Types Comparison

| Type | Fail-Safe | Time Travel | Storage Cost | Use Case |
|------|-----------|-------------|--------------|----------|
| **Permanent** | 7 days | 0-90 days | 100% | Production data requiring disaster recovery |
| **Transient** | None | 0-1 day | ~30% | Staging, dev, temporary ETL |
| **Temporary** | None | 0-1 day | ~30% | Session-specific, auto-dropped on session end |

### Syntax: CREATE TABLE

#### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] [ { TRANSIENT | TEMPORARY | TEMP } ] TABLE [ IF NOT EXISTS ] <table_name> (
  <col_name> <col_type>
    [ DEFAULT { <expr> | IDENTITY | NULL } ]
    [ { AUTOINCREMENT | IDENTITY } [ ( <start_num>, <step_num> ) ] ]
    [ COLLATE '<collation>' ]
    [ { NOT NULL | NULL } ]
    [ INLINE CONSTRAINT <constraint_name> { UNIQUE | PRIMARY KEY | FOREIGN KEY REFERENCES <ref_table> ( <ref_col> ) } ]
    [ COMMENT '<string>' ]
  [ , <col_name> <col_type> ... ]
  [ , CONSTRAINT <constraint_name> { UNIQUE | PRIMARY KEY | FOREIGN KEY } ( <col_list> ) ]
)
  [ CLUSTER BY ( <expr> [ , <expr> ... ] ) ]
  [ STAGE_FILE_FORMAT = ( { FORMAT_NAME = '<fmt_name>' | TYPE = <type> [ formatTypeOptions ] } ) ]
  [ STAGE_COPY_OPTIONS = ( copyOptions ) ]
  [ DATA_RETENTION_TIME_IN_DAYS = <num> ]
  [ MAX_DATA_EXTENSION_TIME_IN_DAYS = <num> ]
  [ CHANGE_TRACKING = TRUE | FALSE ]
  [ DEFAULT_DDL_COLLATION = '<collation>' ]
  [ COPY GRANTS ]
  [ WITH ROW ACCESS POLICY <policy_name> ON ( <col_list> ) ]
  [ WITH AGGREGATION POLICY <policy_name> ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ]
[ AS <query> ];

-- Create from existing table
CREATE TABLE <n> CLONE <source_table>
  [ { AT | BEFORE } ( TIMESTAMP => <ts> | OFFSET => <time_diff> | STATEMENT => <id> ) ];

-- Create external table
CREATE [ OR REPLACE ] EXTERNAL TABLE <n> (
  <col_name> <col_type> AS <expr>
  [ , ... ]
)
  LOCATION = @<stage_name> [ '/<path>/' ]
  [ PARTITION BY ( <part_col> [ , ... ] ) ]
  [ FILE_FORMAT = ( { FORMAT_NAME = '<fmt>' | TYPE = <type> } ) ]
  [ PATTERN = '<regex_pattern>' ]
  [ AUTO_REFRESH = TRUE | FALSE ]
  [ REFRESH_ON_CREATE = TRUE | FALSE ]
  [ COMMENT = '<string>' ];
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `OR REPLACE` | Flag | N/A | Replace existing table | `OR REPLACE` |
| `TRANSIENT` | Flag | FALSE | No Fail-Safe, reduced storage cost | `TRANSIENT` |
| `TEMPORARY` | Flag | FALSE | Session-scoped, auto-dropped | `TEMPORARY` |
| `IF NOT EXISTS` | Flag | N/A | Create only if doesn't exist | `IF NOT EXISTS` |
| `col_type` | Data Type | Required | INT, VARCHAR, VARIANT, TIMESTAMP_NTZ, etc. | `VARCHAR(100)` |
| `DEFAULT` | Expression | NULL | Default value for column | `DEFAULT 0` |
| `AUTOINCREMENT` | Sequence | NULL | Auto-incrementing values (start, step) | `AUTOINCREMENT(1,1)` |
| `NOT NULL` | Constraint | NULL | Enforce non-null values | `NOT NULL` |
| `UNIQUE` | Constraint | NULL | Enforce uniqueness (metadata only) | `UNIQUE` |
| `PRIMARY KEY` | Constraint | NULL | Identify unique rows (metadata only) | `PRIMARY KEY` |
| `FOREIGN KEY` | Constraint | NULL | Reference another table (metadata only) | `FOREIGN KEY REFERENCES orders(id)` |
| `CLUSTER BY` | Expression(s) | NULL | Clustering key for performance | `CLUSTER BY (order_date)` |
| `DATA_RETENTION_TIME_IN_DAYS` | Integer (0-90) | 1 | Time Travel retention | `7` |
| `CHANGE_TRACKING` | Boolean | FALSE | Enable change tracking for streams | `TRUE` |
| `COPY GRANTS` | Flag | FALSE | Copy privileges from replaced table | `COPY GRANTS` |
| `ROW ACCESS POLICY` | Policy Name | NULL | Apply row-level security | `WITH ROW ACCESS POLICY pii_policy ON (user_id)` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (sensitivity = 'high')` |
| `COMMENT` | String | NULL | Table description | `'Fact table for orders'` |

#### Basic Example

```sql
-- Simple permanent table
CREATE TABLE customers (
  customer_id INT PRIMARY KEY,
  first_name VARCHAR(50) NOT NULL,
  last_name VARCHAR(50) NOT NULL,
  email VARCHAR(100) UNIQUE,
  created_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Insert sample data
INSERT INTO customers VALUES
  (1, 'John', 'Doe', 'john@example.com', CURRENT_TIMESTAMP()),
  (2, 'Jane', 'Smith', 'jane@example.com', CURRENT_TIMESTAMP());
```

#### Advanced Example (SDLC: Multi-Environment Table Design)

```sql
-- Production fact table (permanent, clustered, change tracking)
CREATE OR REPLACE TABLE prod_analytics.sales.fact_orders (
  order_id BIGINT AUTOINCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  order_date DATE NOT NULL,
  order_timestamp TIMESTAMP_NTZ NOT NULL,
  region_code VARCHAR(10) NOT NULL,
  product_id INT NOT NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL,
  total_amount DECIMAL(12,2) NOT NULL,
  status VARCHAR(20) DEFAULT 'PENDING',
  created_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  updated_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES dim_customers(customer_id)
)
  CLUSTER BY (order_date, region_code)
  DATA_RETENTION_TIME_IN_DAYS = 30
  CHANGE_TRACKING = TRUE
  WITH TAG (
    environment = 'production',
    pii = 'false',
    compliance = 'SOX',
    refresh_freq = 'realtime'
  )
  COMMENT = 'Production orders fact table - real-time CDC from OLTP';

-- Staging transient table (cost-optimized)
CREATE TRANSIENT TABLE staging.sales.stg_orders (
  order_id BIGINT,
  customer_id INT,
  order_date DATE,
  region_code VARCHAR(10),
  total_amount DECIMAL(12,2),
  raw_data VARIANT,  -- Semi-structured staging
  load_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
)
  DATA_RETENTION_TIME_IN_DAYS = 1
  WITH TAG (environment = 'staging', auto_cleanup = 'true')
  COMMENT = 'Staging table for nightly ETL - 1-day retention';

-- Temporary session table (dev testing)
CREATE TEMPORARY TABLE temp_orders_analysis (
  order_date DATE,
  region_code VARCHAR(10),
  total_sales DECIMAL(15,2)
)
  COMMENT = 'Temporary analysis table - auto-dropped on session end';

-- Create table from query (CTAS)
CREATE TABLE sales_summary AS
SELECT 
  DATE_TRUNC('month', order_date) AS month,
  region_code,
  COUNT(*) AS order_count,
  SUM(total_amount) AS total_revenue
FROM fact_orders
GROUP BY 1, 2;

-- Zero-copy clone for QA testing
CREATE TABLE qa_analytics.sales.fact_orders
  CLONE prod_analytics.sales.fact_orders
  AT (OFFSET => -3600);  -- Clone state from 1 hour ago
```

---

## Views

Views are named queries stored in Snowflake's metadata. Secure views encrypt query definitions and prevent unauthorized access to underlying data. Materialized views pre-compute and store results for faster query performance.

### View Types

| Type | Storage | Performance | Use Case |
|------|---------|-------------|----------|
| **Standard** | No data | Query-time computation | Simple abstractions, no PII |
| **Secure** | No data | Query-time computation | PII data, compliance, prevent SQL inspection |
| **Materialized** | Stores results | Pre-computed | Expensive aggregations, BI dashboards |

### Syntax: CREATE VIEW

#### Complete Syntax Template

```sql
-- Standard view
CREATE [ OR REPLACE ] [ SECURE ] [ RECURSIVE ] VIEW [ IF NOT EXISTS ] <view_name>
  [ ( <column_list> ) ]
  [ COPY GRANTS ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ]
AS <select_statement>;

-- Materialized view
CREATE [ OR REPLACE ] [ SECURE ] MATERIALIZED VIEW [ IF NOT EXISTS ] <view_name>
  [ CLUSTER BY ( <expr> [ , ... ] ) ]
  [ COPY GRANTS ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ]
AS <select_statement>;
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `OR REPLACE` | Flag | N/A | Replace existing view | `OR REPLACE` |
| `SECURE` | Flag | FALSE | Encrypt definition, prevent SQL inspection | `SECURE` |
| `RECURSIVE` | Flag | FALSE | Allow recursive CTEs | `RECURSIVE` |
| `MATERIALIZED` | Flag | FALSE | Store results, auto-refresh | `MATERIALIZED` |
| `CLUSTER BY` | Expression(s) | NULL | Clustering for materialized views | `CLUSTER BY (order_date)` |
| `COPY GRANTS` | Flag | FALSE | Copy privileges from replaced view | `COPY GRANTS` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (pii = 'true')` |
| `COMMENT` | String | NULL | View description | `'Monthly sales aggregation'` |

#### Basic Example

```sql
-- Standard view
CREATE VIEW v_active_customers AS
SELECT 
  customer_id,
  first_name,
  last_name,
  email
FROM customers
WHERE status = 'ACTIVE';

-- Query view
SELECT * FROM v_active_customers LIMIT 10;
```

#### Advanced Example

```sql
-- Secure view for PII compliance
CREATE OR REPLACE SECURE VIEW prod.sales.v_customer_pii (
  customer_id COMMENT 'Unique customer identifier',
  full_name COMMENT 'Concatenated first and last name',
  masked_email COMMENT 'Partially masked email for privacy'
)
  WITH TAG (
    pii = 'true',
    compliance = 'GDPR',
    access_level = 'restricted'
  )
  COMMENT = 'Secure view with masked PII - GDPR compliant'
AS
SELECT 
  customer_id,
  first_name || ' ' || last_name AS full_name,
  REGEXP_REPLACE(email, '(.{2}).*(@.*)', '\\1***\\2') AS masked_email
FROM customers;

-- Materialized view for BI dashboard (auto-refresh)
CREATE OR REPLACE MATERIALIZED VIEW prod.sales.mv_monthly_revenue
  CLUSTER BY (month, region)
  WITH TAG (
    refresh_type = 'automatic',
    dashboard = 'executive'
  )
  COMMENT = 'Pre-aggregated monthly revenue - auto-refreshed'
AS
SELECT 
  DATE_TRUNC('month', order_date) AS month,
  region_code AS region,
  COUNT(DISTINCT customer_id) AS unique_customers,
  COUNT(*) AS total_orders,
  SUM(total_amount) AS total_revenue,
  AVG(total_amount) AS avg_order_value
FROM fact_orders
WHERE status = 'COMPLETED'
GROUP BY 1, 2;

-- Recursive view (hierarchical data)
CREATE OR REPLACE RECURSIVE VIEW v_employee_hierarchy AS
WITH RECURSIVE emp_cte AS (
  -- Anchor: Top-level managers
  SELECT employee_id, manager_id, full_name, 1 AS level
  FROM employees
  WHERE manager_id IS NULL
  
  UNION ALL
  
  -- Recursive: Employees reporting to managers
  SELECT e.employee_id, e.manager_id, e.full_name, cte.level + 1
  FROM employees e
  INNER JOIN emp_cte cte ON e.manager_id = cte.employee_id
)
SELECT * FROM emp_cte;
```

---

## Stages

Stages are named locations for storing data files. Internal stages store files within Snowflake, while external stages reference cloud storage (S3/Azure/GCS). Stages enable bulk data loading and unloading operations.

### Stage Types

| Type | Location | Use Case |
|------|----------|----------|
| **User Stage** | `@~` | Per-user temporary files (no CREATE required) |
| **Table Stage** | `@%table_name` | Per-table temporary files (no CREATE required) |
| **Named Internal** | `@stage_name` | Shared internal storage |
| **Named External** | `@stage_name` | Cloud storage (S3/Blob/GCS) |

### Syntax: CREATE STAGE

#### Complete Syntax Template

```sql
-- Internal stage
CREATE [ OR REPLACE ] [ TEMPORARY ] STAGE [ IF NOT EXISTS ] <stage_name>
  [ FILE_FORMAT = ( { FORMAT_NAME = '<fmt>' | TYPE = <type> [ formatTypeOptions ] } ) ]
  [ COPY_OPTIONS = ( copyOptions ) ]
  [ DIRECTORY = ( ENABLE = TRUE | FALSE ) ]
  [ ENCRYPTION = ( TYPE = 'SNOWFLAKE_SSE' | 'SNOWFLAKE_FULL' ) ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ];

-- External stage (S3)
CREATE [ OR REPLACE ] STAGE [ IF NOT EXISTS ] <stage_name>
  URL = 's3://<bucket>[/<path>/]'
  [ STORAGE_INTEGRATION = <integration_name> ]
  [ CREDENTIALS = ( AWS_KEY_ID = '<key>' AWS_SECRET_KEY = '<secret>' [ AWS_TOKEN = '<token>' ] ) ]
  [ ENCRYPTION = ( TYPE = 'AWS_SSE_S3' | 'AWS_SSE_KMS' [ KMS_KEY_ID = '<key_id>' ] | 'NONE' ) ]
  [ FILE_FORMAT = ( ... ) ]
  [ COPY_OPTIONS = ( ... ) ]
  [ DIRECTORY = ( ENABLE = TRUE | FALSE ) ]
  [ WITH TAG ( ... ) ]
  [ COMMENT = '<string>' ];

-- External stage (Azure)
CREATE STAGE <stage_name>
  URL = 'azure://<account>.blob.core.windows.net/<container>[/<path>/]'
  [ STORAGE_INTEGRATION = <integration_name> ]
  [ CREDENTIALS = ( AZURE_SAS_TOKEN = '<token>' ) ]
  [ ENCRYPTION = ( TYPE = 'AZURE_CSE' | 'NONE' ) ]
  [ ... ];

-- External stage (GCS)
CREATE STAGE <stage_name>
  URL = 'gcs://<bucket>[/<path>/]'
  [ STORAGE_INTEGRATION = <integration_name> ]
  [ ENCRYPTION = ( TYPE = 'GCS_SSE_KMS' [ KMS_KEY_ID = '<key>' ] | 'NONE' ) ]
  [ ... ];
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `TEMPORARY` | Flag | FALSE | Session-scoped stage | `TEMPORARY` |
| `URL` | String | NULL | Cloud storage path (external only) | `s3://mybucket/data/` |
| `STORAGE_INTEGRATION` | String | NULL | Named integration for cloud auth | `my_s3_integration` |
| `CREDENTIALS` | Key-Value | NULL | Cloud credentials (use integration instead) | `AWS_KEY_ID='xxx'` |
| `ENCRYPTION` | Config | SNOWFLAKE_SSE | Encryption type and settings | `TYPE='AWS_SSE_KMS'` |
| `FILE_FORMAT` | Config | NULL | Default file format for stage | `FORMAT_NAME='my_csv_format'` |
| `COPY_OPTIONS` | Config | NULL | Default COPY INTO options | `ON_ERROR='CONTINUE'` |
| `DIRECTORY` | Boolean | FALSE | Enable directory table for metadata | `ENABLE=TRUE` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (source='s3')` |
| `COMMENT` | String | NULL | Stage description | `'S3 raw data stage'` |

#### Basic Example

```sql
-- Create internal named stage
CREATE STAGE my_internal_stage
  FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1)
  COMMENT = 'Internal stage for CSV files';

-- Put file into internal stage
PUT file:///tmp/data.csv @my_internal_stage;

-- List files in stage
LIST @my_internal_stage;
```

#### Advanced Example (SDLC: Multi-Cloud Stage Configuration)

```sql
-- AWS S3 external stage with storage integration (recommended)
CREATE OR REPLACE STAGE prod.raw_data.s3_orders_stage
  URL = 's3://mycompany-data-lake/orders/'
  STORAGE_INTEGRATION = aws_s3_integration
  FILE_FORMAT = (
    FORMAT_NAME = 'parquet_format'
  )
  COPY_OPTIONS = (
    ON_ERROR = 'SKIP_FILE',
    SIZE_LIMIT = 1000000000,  -- 1GB per file
    PURGE = FALSE
  )
  DIRECTORY = (ENABLE = TRUE)
  WITH TAG (
    cloud_provider = 'aws',
    region = 'us-east-1',
    cost_center = 'data_engineering'
  )
  COMMENT = 'Production S3 stage for raw order data - Parquet format';

-- Azure Blob Storage stage
CREATE OR REPLACE STAGE prod.raw_data.azure_events_stage
  URL = 'azure://myaccount.blob.core.windows.net/events/'
  STORAGE_INTEGRATION = azure_blob_integration
  FILE_FORMAT = (TYPE = 'JSON' COMPRESSION = 'GZIP')
  DIRECTORY = (ENABLE = TRUE)
  COMMENT = 'Azure Blob stage for event stream data - JSON/GZIP';

-- GCS stage with encryption
CREATE OR REPLACE STAGE prod.raw_data.gcs_logs_stage
  URL = 'gcs://mycompany-logs/application-logs/'
  STORAGE_INTEGRATION = gcs_integration
  ENCRYPTION = (
    TYPE = 'GCS_SSE_KMS'
    KMS_KEY_ID = 'projects/myproject/locations/us/keyRings/myring/cryptoKeys/mykey'
  )
  FILE_FORMAT = (TYPE = 'CSV' COMPRESSION = 'AUTO')
  COMMENT = 'GCS stage for application logs - KMS encrypted';

-- Check stage configuration
DESC STAGE s3_orders_stage;

-- Query stage metadata (directory table)
SELECT *
FROM DIRECTORY(@s3_orders_stage)
WHERE LAST_MODIFIED >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY LAST_MODIFIED DESC;
```

---

## Streams

Streams capture change data (CDC) from tables, views, or directory tables. Streams track inserts, updates, and deletes enabling incremental processing without scanning entire tables.

### Syntax: CREATE STREAM

#### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] STREAM [ IF NOT EXISTS ] <stream_name>
  ON TABLE <table_name>
  [ AT ( { TIMESTAMP => <timestamp> | OFFSET => <time_diff> | STATEMENT => <query_id> | STREAM => '<stream_name>' } ) ]
  [ BEFORE ( STATEMENT => <query_id> ) ]
  [ APPEND_ONLY = TRUE | FALSE ]
  [ SHOW_INITIAL_ROWS = TRUE | FALSE ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ];

-- Stream on view
CREATE STREAM <stream_name> ON VIEW <view_name> [ ... ];

-- Stream on external table (directory)
CREATE STREAM <stream_name> ON EXTERNAL TABLE <table_name> [ ... ];

-- Stream on another stream
CREATE STREAM <stream_name> ON STREAM <source_stream> [ ... ];
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `ON TABLE/VIEW` | Object Name | Required | Source table or view | `ON TABLE fact_orders` |
| `AT` | Point-in-Time | Current | Stream starting point | `AT(OFFSET => -3600)` |
| `APPEND_ONLY` | Boolean | FALSE | Track only inserts (no updates/deletes) | `TRUE` |
| `SHOW_INITIAL_ROWS` | Boolean | FALSE | Include existing rows on creation | `TRUE` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (cdc='enabled')` |
| `COMMENT` | String | NULL | Stream description | `'CDC stream for orders'` |

#### Basic Example

```sql
-- Create stream on table
CREATE STREAM orders_stream
  ON TABLE fact_orders
  COMMENT = 'Capture changes to orders table';

-- Insert/update/delete data
INSERT INTO fact_orders VALUES (1001, 100, CURRENT_DATE(), 'US', 50);
UPDATE fact_orders SET quantity = 55 WHERE order_id = 1001;
DELETE FROM fact_orders WHERE order_id = 1000;

-- Query stream (shows changes since last consume)
SELECT 
  order_id,
  customer_id,
  quantity,
  METADATA$ACTION AS dml_type,  -- INSERT, UPDATE, DELETE
  METADATA$ISUPDATE AS is_update,
  METADATA$ROW_ID AS row_id
FROM orders_stream;

-- Consume stream (process changes)
INSERT INTO orders_history
SELECT * FROM orders_stream;
-- Stream offset advances, changes no longer visible
```

#### Advanced Example (SDLC: Real-Time CDC Pipeline)

```sql
-- Enable change tracking on source table
ALTER TABLE prod.sales.fact_orders SET CHANGE_TRACKING = TRUE;

-- Create stream for incremental ETL
CREATE OR REPLACE STREAM prod.cdc.orders_cdc_stream
  ON TABLE prod.sales.fact_orders
  APPEND_ONLY = FALSE  -- Track all DML (insert/update/delete)
  WITH TAG (
    cdc_type = 'full',
    target = 'data_warehouse',
    refresh_freq = 'realtime'
  )
  COMMENT = 'CDC stream for real-time order processing';

-- Create target table for materialized changes
CREATE TABLE prod.warehouse.fact_orders_materialized (
  order_id BIGINT PRIMARY KEY,
  customer_id INT,
  order_date DATE,
  total_amount DECIMAL(12,2),
  status VARCHAR(20),
  updated_at TIMESTAMP_NTZ,
  is_deleted BOOLEAN DEFAULT FALSE
);

-- Create task to merge stream changes every 5 minutes
CREATE OR REPLACE TASK prod.cdc.merge_orders_task
  WAREHOUSE = cdc_wh
  SCHEDULE = '5 MINUTE'
AS
MERGE INTO prod.warehouse.fact_orders_materialized AS target
USING (
  SELECT 
    order_id,
    customer_id,
    order_date,
    total_amount,
    status,
    CURRENT_TIMESTAMP() AS updated_at,
    METADATA$ACTION = 'DELETE' AS is_deleted
  FROM prod.cdc.orders_cdc_stream
) AS source
ON target.order_id = source.order_id
WHEN MATCHED AND source.is_deleted THEN
  UPDATE SET is_deleted = TRUE, updated_at = source.updated_at
WHEN MATCHED THEN
  UPDATE SET 
    customer_id = source.customer_id,
    order_date = source.order_date,
    total_amount = source.total_amount,
    status = source.status,
    updated_at = source.updated_at
WHEN NOT MATCHED THEN
  INSERT (order_id, customer_id, order_date, total_amount, status, updated_at, is_deleted)
  VALUES (source.order_id, source.customer_id, source.order_date, 
          source.total_amount, source.status, source.updated_at, source.is_deleted);

-- Resume task
ALTER TASK merge_orders_task RESUME;

-- Monitor stream lag
SELECT 
  SYSTEM$STREAM_GET_TABLE_TIMESTAMP('prod.cdc.orders_cdc_stream') AS stream_timestamp,
  DATEDIFF(second, stream_timestamp, CURRENT_TIMESTAMP()) AS lag_seconds
;
```

---

## Tasks

Tasks schedule SQL statements or stored procedures to run automatically. Tasks support dependencies (DAGs), error handling, and conditional execution enabling complex ELT workflows.

### Syntax: CREATE TASK

#### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] TASK [ IF NOT EXISTS ] <task_name>
  WAREHOUSE = <warehouse_name>
  [ SCHEDULE = '{ <num> MINUTE | USING CRON <expr> <time_zone> }' ]
  [ AFTER <predecessor_task> [ , <predecessor_task> ... ] ]
  [ WHEN <boolean_expr> ]
  [ USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE = <size> ]
  [ USER_TASK_TIMEOUT_MS = <num> ]
  [ SUSPEND_TASK_AFTER_NUM_FAILURES = <num> ]
  [ ERROR_INTEGRATION = <integration_name> ]
  [ ALLOW_OVERLAPPING_EXECUTION = TRUE | FALSE ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ]
  [ COPY GRANTS ]
AS <sql_statement>;

-- Serverless task (no warehouse required)
CREATE TASK <task_name>
  SCHEDULE = '10 MINUTE'
  USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE = 'XSMALL'
AS <sql_statement>;
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `WAREHOUSE` | String | NULL | Warehouse for execution (or serverless) | `etl_wh` |
| `SCHEDULE` | String | NULL | Cron expression or minute interval | `USING CRON 0 9 * * * America/Los_Angeles` |
| `AFTER` | Task Name(s) | NULL | Predecessor task(s) for DAG | `AFTER task1, task2` |
| `WHEN` | Boolean Expr | NULL | Conditional execution (skip if false) | `WHEN SYSTEM$STREAM_HAS_DATA('my_stream')` |
| `USER_TASK_MANAGED_*` | String | NULL | Serverless task warehouse size | `XSMALL` |
| `USER_TASK_TIMEOUT_MS` | Integer | 3600000 | Task timeout in milliseconds (1 hour default) | `7200000` |
| `SUSPEND_TASK_AFTER_NUM_FAILURES` | Integer | NULL | Auto-suspend after N failures | `3` |
| `ERROR_INTEGRATION` | String | NULL | Notification integration for errors | `email_error_integration` |
| `ALLOW_OVERLAPPING_EXECUTION` | Boolean | FALSE | Allow concurrent task runs | `TRUE` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (dag='etl_pipeline')` |
| `COMMENT` | String | NULL | Task description | `'Hourly sales aggregation'` |

#### Basic Example

```sql
-- Simple scheduled task
CREATE TASK refresh_sales_summary
  WAREHOUSE = etl_wh
  SCHEDULE = '60 MINUTE'
AS
  INSERT INTO sales_summary
  SELECT 
    DATE_TRUNC('hour', order_timestamp) AS hour,
    SUM(total_amount) AS total_sales
  FROM fact_orders
  WHERE order_timestamp >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
  GROUP BY 1;

-- Resume task (tasks are created suspended)
ALTER TASK refresh_sales_summary RESUME;

-- Check task status
SHOW TASKS LIKE 'refresh_sales_summary';

-- View task history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
  TASK_NAME => 'REFRESH_SALES_SUMMARY'
))
ORDER BY SCHEDULED_TIME DESC
LIMIT 10;
```

#### Advanced Example (SDLC: Multi-Step ELT DAG)

```sql
-- Root task: Extract new files from S3 stage
CREATE OR REPLACE TASK prod.etl.task_01_extract
  WAREHOUSE = etl_wh
  SCHEDULE = 'USING CRON 0 */4 * * * America/New_York'  -- Every 4 hours
  USER_TASK_TIMEOUT_MS = 7200000  -- 2-hour timeout
  SUSPEND_TASK_AFTER_NUM_FAILURES = 3
  WITH TAG (
    dag = 'daily_etl',
    step = '01_extract',
    criticality = 'high'
  )
  COMMENT = 'Extract: Load new files from S3 to staging'
AS
  COPY INTO staging.raw_data.orders
  FROM @prod.stages.s3_orders_stage
  FILE_FORMAT = (FORMAT_NAME = 'parquet_format')
  PATTERN = '.*orders.*[.]parquet'
  ON_ERROR = 'SKIP_FILE';

-- Child task: Transform (depends on extract)
CREATE OR REPLACE TASK prod.etl.task_02_transform
  WAREHOUSE = etl_wh
  AFTER prod.etl.task_01_extract
  WHEN SYSTEM$STREAM_HAS_DATA('staging.streams.orders_stream')  -- Only run if changes exist
  WITH TAG (dag = 'daily_etl', step = '02_transform')
  COMMENT = 'Transform: Clean and enrich orders data'
AS
  INSERT INTO prod.curated.orders_cleaned
  SELECT 
    order_id,
    customer_id,
    order_date,
    COALESCE(total_amount, 0) AS total_amount,
    UPPER(status) AS status,
    CURRENT_TIMESTAMP() AS processed_at
  FROM staging.streams.orders_stream
  WHERE total_amount > 0;

-- Child task: Load to fact table (depends on transform)
CREATE OR REPLACE TASK prod.etl.task_03_load
  WAREHOUSE = etl_wh
  AFTER prod.etl.task_02_transform
  WITH TAG (dag = 'daily_etl', step = '03_load')
  COMMENT = 'Load: Merge into production fact table'
AS
  MERGE INTO prod.warehouse.fact_orders AS target
  USING prod.curated.orders_cleaned AS source
  ON target.order_id = source.order_id
  WHEN MATCHED THEN
    UPDATE SET 
      total_amount = source.total_amount,
      status = source.status
  WHEN NOT MATCHED THEN
    INSERT (order_id, customer_id, order_date, total_amount, status)
    VALUES (source.order_id, source.customer_id, source.order_date, 
            source.total_amount, source.status);

-- Final task: Refresh materialized views (depends on load)
CREATE OR REPLACE TASK prod.etl.task_04_refresh_mv
  WAREHOUSE = etl_wh
  AFTER prod.etl.task_03_load
  WITH TAG (dag = 'daily_etl', step = '04_refresh')
  COMMENT = 'Refresh: Update materialized views for BI'
AS
  CALL SYSTEM$REFRESH_MATERIALIZED_VIEW('prod.sales.mv_monthly_revenue');

-- Resume all tasks (bottom-up: leaf tasks first, root task last)
ALTER TASK prod.etl.task_04_refresh_mv RESUME;
ALTER TASK prod.etl.task_03_load RESUME;
ALTER TASK prod.etl.task_02_transform RESUME;
ALTER TASK prod.etl.task_01_extract RESUME;  -- Root task resumed last

-- Monitor DAG execution
SELECT 
  name AS task_name,
  state,
  schedule,
  predecessors,
  PARSE_JSON(task_history):query_text AS last_query
FROM TABLE(INFORMATION_SCHEMA.TASK_DEPENDENTS(
  TASK_NAME => 'PROD.ETL.TASK_01_EXTRACT',
  RECURSIVE => TRUE
));
```

---

## Stored Procedures & Functions

Stored procedures (JavaScript/SQL/Python) execute logic with transaction control. User-defined functions (UDFs) return scalar values or table results. External functions call remote APIs.

### Syntax: CREATE PROCEDURE

```sql
CREATE [ OR REPLACE ] PROCEDURE <proc_name> ( [ <arg_name> <arg_type> [ , ... ] ] )
  RETURNS <return_type>
  LANGUAGE { JAVASCRIPT | SQL | PYTHON }
  [ EXECUTE AS { CALLER | OWNER } ]
  [ WITH TAG ( ... ) ]
  [ COMMENT = '<string>' ]
AS
$$
  -- Procedure code
$$;
```

### Syntax: CREATE FUNCTION

```sql
-- Scalar UDF
CREATE [ OR REPLACE ] [ SECURE ] FUNCTION <function_name> ( [ <arg_name> <arg_type> [ , ... ] ] )
  RETURNS <return_type>
  LANGUAGE { JAVASCRIPT | SQL | PYTHON }
  [ MEMOIZABLE ]
  [ WITH TAG ( ... ) ]
  [ COMMENT = '<string>' ]
AS
$$
  -- Function code
$$;

-- Table UDF (UDTF)
CREATE [ OR REPLACE ] FUNCTION <function_name> ( [ <arg_name> <arg_type> [ , ... ] ] )
  RETURNS TABLE ( <col_name> <col_type> [ , ... ] )
  LANGUAGE SQL
AS
$$
  -- SELECT statement returning table
$$;
```

#### Example: JavaScript Stored Procedure

```sql
CREATE OR REPLACE PROCEDURE archive_old_orders(days_old INT)
  RETURNS STRING
  LANGUAGE JAVASCRIPT
  EXECUTE AS CALLER
AS
$$
  var sql_delete = `
    DELETE FROM fact_orders
    WHERE order_date < DATEADD(day, -${DAYS_OLD}, CURRENT_DATE())
  `;
  
  var sql_archive = `
    INSERT INTO orders_archive
    SELECT * FROM fact_orders
    WHERE order_date < DATEADD(day, -${DAYS_OLD}, CURRENT_DATE())
  `;
  
  // Execute archive, then delete
  snowflake.execute({sqlText: sql_archive});
  var result = snowflake.execute({sqlText: sql_delete});
  
  return `Archived and deleted ${result.getNumRowsAffected()} rows`;
$$;

-- Call procedure
CALL archive_old_orders(365);
```

---

## SDLC Use Case: GitHub Actions for Object Versioning

### Scenario: Automated Schema Deployment via CI/CD

```yaml
# .github/workflows/snowflake-deploy.yml
name: Deploy Snowflake Objects

on:
  push:
    branches:
      - main
    paths:
      - 'sql/tables/**'
      - 'sql/views/**'
      - 'sql/tasks/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install Snowflake CLI
        run: pip install snowflake-connector-python

      - name: Deploy Tables
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
        run: |
          python - <<EOF
          import snowflake.connector
          import os
          import glob
          
          conn = snowflake.connector.connect(
              account=os.getenv('SNOWFLAKE_ACCOUNT'),
              user=os.getenv('SNOWFLAKE_USER'),
              password=os.getenv('SNOWFLAKE_PASSWORD'),
              role='SYSADMIN',
              warehouse='deployment_wh'
          )
          
          cursor = conn.cursor()
          
          # Deploy tables in order
          for sql_file in sorted(glob.glob('sql/tables/*.sql')):
              print(f"Deploying {sql_file}...")
              with open(sql_file) as f:
                  sql = f.read()
                  cursor.execute(sql)
              print(f"✅ Deployed {sql_file}")
          
          # Deploy views
          for sql_file in sorted(glob.glob('sql/views/*.sql')):
              print(f"Deploying {sql_file}...")
              with open(sql_file) as f:
                  cursor.execute(f.read())
              print(f"✅ Deployed {sql_file}")
          
          cursor.close()
          conn.close()
          EOF

      - name: Run Integration Tests
        run: |
          python tests/test_objects.py
```

### Versioned SQL Files

```sql
-- sql/tables/01_fact_orders.sql
CREATE OR REPLACE TABLE prod.sales.fact_orders (
  order_id BIGINT PRIMARY KEY,
  customer_id INT NOT NULL,
  order_date DATE NOT NULL,
  total_amount DECIMAL(12,2)
)
  CLUSTER BY (order_date)
  DATA_RETENTION_TIME_IN_DAYS = 30
  CHANGE_TRACKING = TRUE
  WITH TAG (version = 'v1.2.0', deployed_by = 'github_actions')
  COMMENT = 'Production fact table - version 1.2.0';

-- sql/views/01_v_monthly_sales.sql
CREATE OR REPLACE SECURE VIEW prod.sales.v_monthly_sales AS
SELECT 
  DATE_TRUNC('month', order_date) AS month,
  SUM(total_amount) AS revenue
FROM prod.sales.fact_orders
GROUP BY 1;
```

---

## Key Concepts

- **Micro-Partition**: Immutable 50-500MB columnar chunk with metadata (automatic, no manual partitioning)
- **Stream**: CDC mechanism tracking table changes (inserts, updates, deletes) for incremental processing
- **Task**: Scheduled SQL execution with dependencies (DAGs) for ELT automation
- **Stage**: Named location for data files (internal or external cloud storage)
- **Materialized View**: Pre-computed query results with automatic refresh for performance

---
