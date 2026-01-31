# 4. Data Loading & Unloading

## Data Loading Overview

Snowflake supports bulk loading (COPY INTO), continuous ingestion (Snowpipe), and external tables for querying cloud files. All methods leverage stages (internal or external) and file formats (CSV, JSON, Parquet, Avro, ORC, XML). Optimal performance requires parallelization, compression, and appropriate file sizing (100-250MB compressed per file).

```
Data Sources → Stages → COPY INTO / Snowpipe → Tables
            ↓
      (S3/Blob/GCS)
            ↓
      File Formats (CSV/JSON/Parquet)
            ↓
      Validation & Transformation
            ↓
      Target Tables (Permanent/Transient)
```

---

## COPY INTO (Bulk Loading)

COPY INTO loads data from staged files into tables. Supports transformations, error handling, and validation. Ideal for batch/scheduled loads (hourly/daily ETL).

### Syntax: COPY INTO (Table)

#### Complete Syntax Template

```sql
COPY INTO [ <namespace>. ] <table_name> [ ( <col_name> [ , <col_name> ... ] ) ]
  FROM { @<stage_name>[/<path>] | '<external_location>' }
  [ FILES = ( '<file_name>' [ , '<file_name>' ... ] ) ]
  [ PATTERN = '<regex_pattern>' ]
  [ FILE_FORMAT = ( { FORMAT_NAME = '<fmt_name>' | TYPE = <type> [ formatTypeOptions ] } ) ]
  [ COPY_OPTIONS ( 
      ON_ERROR = { CONTINUE | SKIP_FILE | SKIP_FILE_<num> | SKIP_FILE_<num>% | ABORT_STATEMENT }
      SIZE_LIMIT = <num>
      PURGE = TRUE | FALSE
      RETURN_FAILED_ONLY = TRUE | FALSE
      MATCH_BY_COLUMN_NAME = CASE_SENSITIVE | CASE_INSENSITIVE | NONE
      ENFORCE_LENGTH = TRUE | FALSE
      TRUNCATECOLUMNS = TRUE | FALSE
      FORCE = TRUE | FALSE
    ) ]
  [ VALIDATION_MODE = RETURN_<n>_ROWS | RETURN_ERRORS | RETURN_ALL_ERRORS ];
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `table_name` | String | Required | Target table name | `sales.fact_orders` |
| `FROM` | Stage/URL | Required | Source location (@stage or cloud URL) | `@my_stage/orders/` |
| `FILES` | String List | NULL | Specific file names to load | `FILES=('data001.csv', 'data002.csv')` |
| `PATTERN` | Regex | NULL | File pattern filter | `PATTERN='.*sales.*[.]csv'` |
| `FILE_FORMAT` | Config | NULL | File format specification | `TYPE=CSV FIELD_DELIMITER=','` |
| `ON_ERROR` | Enum | ABORT_STATEMENT | Error handling: CONTINUE (skip rows), SKIP_FILE (skip file on error), ABORT_STATEMENT (stop on error) | `ON_ERROR='SKIP_FILE_10'` |
| `SIZE_LIMIT` | Integer | NULL | Max data to load (bytes, 0=no limit) | `SIZE_LIMIT=1000000000` |
| `PURGE` | Boolean | FALSE | Delete files after successful load | `PURGE=TRUE` |
| `RETURN_FAILED_ONLY` | Boolean | FALSE | Return only failed rows in result | `RETURN_FAILED_ONLY=TRUE` |
| `MATCH_BY_COLUMN_NAME` | Enum | NONE | Match file columns to table by name | `MATCH_BY_COLUMN_NAME=CASE_INSENSITIVE` |
| `ENFORCE_LENGTH` | Boolean | TRUE | Truncate strings exceeding column length | `ENFORCE_LENGTH=FALSE` |
| `TRUNCATECOLUMNS` | Boolean | FALSE | Truncate text that exceeds target column length | `TRUNCATECOLUMNS=TRUE` |
| `FORCE` | Boolean | FALSE | Load files even if previously loaded | `FORCE=TRUE` |
| `VALIDATION_MODE` | Enum | NULL | Validate without loading: RETURN_n_ROWS (sample), RETURN_ERRORS (errors only), RETURN_ALL_ERRORS (all errors) | `VALIDATION_MODE=RETURN_10_ROWS` |

#### File Format Options (CSV)

```sql
FILE_FORMAT = (
  TYPE = CSV
  COMPRESSION = AUTO | GZIP | BZ2 | BROTLI | ZSTD | DEFLATE | RAW_DEFLATE | NONE
  RECORD_DELIMITER = '<char>' | NONE
  FIELD_DELIMITER = '<char>' | NONE
  SKIP_HEADER = <num>
  SKIP_BLANK_LINES = TRUE | FALSE
  DATE_FORMAT = '<string>'
  TIME_FORMAT = '<string>'
  TIMESTAMP_FORMAT = '<string>'
  BINARY_FORMAT = HEX | BASE64 | UTF8
  ESCAPE = '<char>' | NONE
  ESCAPE_UNENCLOSED_FIELD = '<char>' | NONE
  TRIM_SPACE = TRUE | FALSE
  FIELD_OPTIONALLY_ENCLOSED_BY = '<char>' | NONE
  NULL_IF = ( '<string>' [ , '<string>' ... ] )
  ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE | FALSE
  REPLACE_INVALID_CHARACTERS = TRUE | FALSE
  EMPTY_FIELD_AS_NULL = TRUE | FALSE
  SKIP_BYTE_ORDER_MARK = TRUE | FALSE
  ENCODING = '<string>' | UTF8
)
```

#### File Format Options (JSON)

```sql
FILE_FORMAT = (
  TYPE = JSON
  COMPRESSION = AUTO | GZIP | BZ2 | BROTLI | ZSTD | DEFLATE | RAW_DEFLATE | NONE
  DATE_FORMAT = '<string>'
  TIME_FORMAT = '<string>'
  TIMESTAMP_FORMAT = '<string>'
  BINARY_FORMAT = HEX | BASE64 | UTF8
  TRIM_SPACE = TRUE | FALSE
  NULL_IF = ( '<string>' [ , ... ] )
  FILE_EXTENSION = '<string>'
  ENABLE_OCTAL = TRUE | FALSE
  ALLOW_DUPLICATE = TRUE | FALSE
  STRIP_OUTER_ARRAY = TRUE | FALSE
  STRIP_NULL_VALUES = TRUE | FALSE
  REPLACE_INVALID_CHARACTERS = TRUE | FALSE
  IGNORE_UTF8_ERRORS = TRUE | FALSE
  SKIP_BYTE_ORDER_MARK = TRUE | FALSE
)
```

#### File Format Options (Parquet/Avro/ORC)

```sql
FILE_FORMAT = (
  TYPE = PARQUET | AVRO | ORC
  COMPRESSION = AUTO | LZO | SNAPPY | NONE
  BINARY_AS_TEXT = TRUE | FALSE
  TRIM_SPACE = TRUE | FALSE
  REPLACE_INVALID_CHARACTERS = TRUE | FALSE
  NULL_IF = ( '<string>' [ , ... ] )
)
```

#### Basic Example

```sql
-- Create target table
CREATE OR REPLACE TABLE sales.orders (
  order_id INT,
  customer_id INT,
  order_date DATE,
  total_amount DECIMAL(10,2)
);

-- Load CSV files from internal stage
COPY INTO sales.orders
  FROM @my_internal_stage/orders/
  FILE_FORMAT = (
    TYPE = CSV
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1
    NULL_IF = ('NULL', 'null', '')
  )
  ON_ERROR = 'CONTINUE';

-- Check load status
SELECT * FROM TABLE(VALIDATE(sales.orders, JOB_ID => '_last'));
```

#### Advanced Example (SDLC: Production ETL with S3)

```sql
-- Create external stage for S3
CREATE OR REPLACE STAGE prod.raw_data.s3_orders_stage
  URL = 's3://mycompany-datalake/raw/orders/'
  STORAGE_INTEGRATION = aws_s3_integration
  FILE_FORMAT = (
    FORMAT_NAME = 'csv_orders_format'
  );

-- Create file format
CREATE OR REPLACE FILE FORMAT csv_orders_format
  TYPE = CSV
  COMPRESSION = GZIP
  FIELD_DELIMITER = '|'
  RECORD_DELIMITER = '\n'
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  TRIM_SPACE = TRUE
  ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE
  ESCAPE = '\\'
  ESCAPE_UNENCLOSED_FIELD = '\\'
  DATE_FORMAT = 'YYYY-MM-DD'
  TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS'
  NULL_IF = ('\\N', 'NULL', 'null', '');

-- Load with transformations and error handling
COPY INTO prod.sales.fact_orders (
  order_id,
  customer_id,
  order_date,
  order_timestamp,
  total_amount,
  status,
  load_timestamp
)
  FROM (
    SELECT 
      $1::INT AS order_id,
      $2::INT AS customer_id,
      TO_DATE($3, 'YYYY-MM-DD') AS order_date,
      TO_TIMESTAMP($4, 'YYYY-MM-DD HH24:MI:SS') AS order_timestamp,
      $5::DECIMAL(12,2) AS total_amount,
      UPPER($6::VARCHAR) AS status,
      CURRENT_TIMESTAMP() AS load_timestamp
    FROM @s3_orders_stage/2025/01/
  )
  PATTERN = '.*orders.*[.]csv[.]gz'
  FILE_FORMAT = (FORMAT_NAME = 'csv_orders_format')
  ON_ERROR = 'SKIP_FILE_5%'  -- Skip file if >5% error rate
  SIZE_LIMIT = 10000000000    -- 10GB max per execution
  PURGE = TRUE                -- Delete files after load
  FORCE = FALSE;              -- Skip already loaded files

-- Validation (dry run, no data loaded)
COPY INTO prod.sales.fact_orders
  FROM @s3_orders_stage/2025/01/
  FILE_FORMAT = (FORMAT_NAME = 'csv_orders_format')
  VALIDATION_MODE = 'RETURN_ERRORS';

-- Check load history
SELECT 
  file_name,
  file_size,
  row_count,
  row_parsed,
  first_error_message,
  first_error_line_number,
  status
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => 'PROD.SALES.FACT_ORDERS',
  START_TIME => DATEADD(day, -7, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```

---

## Snowpipe (Continuous Ingestion)

Snowpipe automatically loads data as files arrive in cloud storage. Uses serverless compute, event-driven architecture (S3 notifications, Azure Event Grid, GCS Pub/Sub), and micro-batch processing (typically <1 minute latency).

### Syntax: CREATE PIPE

#### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] PIPE [ IF NOT EXISTS ] <pipe_name>
  [ AUTO_INGEST = TRUE | FALSE ]
  [ ERROR_INTEGRATION = <integration_name> ]
  [ AWS_SNS_TOPIC = '<sns_topic_arn>' ]
  [ INTEGRATION = '<notification_integration_name>' ]
  [ WITH TAG ( <tag_name> = '<tag_value>' [ , ... ] ) ]
  [ COMMENT = '<string>' ]
AS
  COPY INTO <table_name>
  FROM @<stage_name>
  [ FILE_FORMAT = ( ... ) ]
  [ COPY_OPTIONS ( ... ) ];
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `AUTO_INGEST` | Boolean | FALSE | Enable automatic file detection via cloud events | `AUTO_INGEST=TRUE` |
| `ERROR_INTEGRATION` | String | NULL | Notification integration for errors | `email_error_integration` |
| `AWS_SNS_TOPIC` | String | NULL | AWS SNS topic ARN for S3 event notifications | `arn:aws:sns:us-east-1:123456789:topic` |
| `INTEGRATION` | String | NULL | Notification integration name (Azure/GCS) | `azure_event_grid_integration` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (ingestion='realtime')` |
| `COMMENT` | String | NULL | Pipe description | `'Continuous S3 to orders table'` |

#### Basic Example

```sql
-- Create pipe for manual triggering
CREATE OR REPLACE PIPE orders_pipe AS
  COPY INTO sales.orders
  FROM @my_stage/orders/
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1);

-- Manually trigger pipe
ALTER PIPE orders_pipe REFRESH;

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('orders_pipe');

-- View pipe load history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
  PIPE_NAME => 'ORDERS_PIPE'
));
```

#### Advanced Example (SDLC: Auto-Ingest from S3)

```sql
-- Step 1: Create notification integration (one-time setup)
CREATE OR REPLACE NOTIFICATION INTEGRATION s3_orders_notification
  TYPE = QUEUE
  NOTIFICATION_PROVIDER = AWS_SNS
  ENABLED = TRUE
  AWS_SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789012:snowpipe-orders'
  AWS_SNS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-s3-access';

-- Step 2: Create auto-ingest pipe
CREATE OR REPLACE PIPE prod.ingestion.orders_snowpipe
  AUTO_INGEST = TRUE
  AWS_SNS_TOPIC = 'arn:aws:sns:us-east-1:123456789012:snowpipe-orders'
  ERROR_INTEGRATION = error_notification
  WITH TAG (
    ingestion_type = 'continuous',
    sla = 'realtime',
    cost_center = 'data_engineering'
  )
  COMMENT = 'Auto-ingest orders from S3 - realtime CDC'
AS
  COPY INTO prod.sales.fact_orders (
    order_id, customer_id, order_date, total_amount, status, ingested_at
  )
  FROM (
    SELECT 
      $1::INT,
      $2::INT,
      $3::DATE,
      $4::DECIMAL(12,2),
      $5::VARCHAR,
      METADATA$FILENAME AS source_file,
      METADATA$FILE_ROW_NUMBER AS row_number,
      CURRENT_TIMESTAMP() AS ingested_at
    FROM @prod.stages.s3_orders_stage
  )
  FILE_FORMAT = (FORMAT_NAME = 'json_orders_format')
  ON_ERROR = 'SKIP_FILE';

-- Step 3: Configure S3 bucket notifications
-- (AWS Console or Terraform: S3 → Events → SNS topic)
-- Event type: s3:ObjectCreated:*
-- Prefix: raw/orders/
-- SNS topic: arn:aws:sns:us-east-1:123456789012:snowpipe-orders

-- Monitor pipe performance
SELECT 
  pipe_name,
  credits_used,
  bytes_inserted,
  files_inserted,
  AVG(bytes_inserted) / NULLIF(AVG(credits_used), 0) AS bytes_per_credit
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE pipe_name = 'ORDERS_SNOWPIPE'
  AND start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY pipe_name, credits_used, bytes_inserted, files_inserted;

-- Pause pipe (maintenance)
ALTER PIPE orders_snowpipe SET PIPE_EXECUTION_PAUSED = TRUE;

-- Resume pipe
ALTER PIPE orders_snowpipe SET PIPE_EXECUTION_PAUSED = FALSE;
```

---

## Cloud Storage Integrations

Storage integrations enable secure, delegated access to cloud storage without embedding credentials in SQL. Snowflake assumes IAM roles (AWS), service principals (Azure), or service accounts (GCS).

### AWS S3 Integration

```sql
-- Create storage integration
CREATE OR REPLACE STORAGE INTEGRATION aws_s3_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-s3-access'
  STORAGE_ALLOWED_LOCATIONS = (
    's3://mycompany-datalake/raw/',
    's3://mycompany-datalake/processed/'
  )
  STORAGE_BLOCKED_LOCATIONS = (
    's3://mycompany-datalake/raw/sensitive/'
  );

-- Get Snowflake IAM user ARN (for AWS trust policy)
DESC STORAGE INTEGRATION aws_s3_integration;
-- Returns: STORAGE_AWS_IAM_USER_ARN, STORAGE_AWS_EXTERNAL_ID

-- AWS IAM Trust Policy (attach to role)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "<STORAGE_AWS_IAM_USER_ARN>"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>"
        }
      }
    }
  ]
}

-- Create stage using integration
CREATE STAGE s3_data_stage
  URL = 's3://mycompany-datalake/raw/'
  STORAGE_INTEGRATION = aws_s3_integration;
```

### Azure Blob Storage Integration

```sql
-- Create storage integration
CREATE OR REPLACE STORAGE INTEGRATION azure_blob_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = AZURE
  ENABLED = TRUE
  AZURE_TENANT_ID = '<tenant_id>'
  STORAGE_ALLOWED_LOCATIONS = (
    'azure://myaccount.blob.core.windows.net/datalake/raw/'
  );

-- Get Snowflake service principal (for Azure consent)
DESC STORAGE INTEGRATION azure_blob_integration;
-- Returns: AZURE_CONSENT_URL

-- Grant Storage Blob Data Reader role to service principal in Azure Portal

-- Create stage
CREATE STAGE azure_data_stage
  URL = 'azure://myaccount.blob.core.windows.net/datalake/raw/'
  STORAGE_INTEGRATION = azure_blob_integration;
```

### GCS Integration

```sql
-- Create storage integration
CREATE OR REPLACE STORAGE INTEGRATION gcs_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = GCS
  ENABLED = TRUE
  STORAGE_ALLOWED_LOCATIONS = (
    'gcs://mycompany-datalake/raw/'
  );

-- Get Snowflake service account email
DESC STORAGE INTEGRATION gcs_integration;
-- Returns: STORAGE_GCP_SERVICE_ACCOUNT

-- Grant Storage Object Viewer role in GCP Console

-- Create stage
CREATE STAGE gcs_data_stage
  URL = 'gcs://mycompany-datalake/raw/'
  STORAGE_INTEGRATION = gcs_integration;
```

---

## Data Unloading (COPY INTO Location)

Unload data from tables to stages (internal or external) in CSV, JSON, or Parquet format. Useful for data export, archival, or cross-platform integration.

### Syntax: COPY INTO (Location)

```sql
COPY INTO { @<stage_name>[/<path>] | '<external_location>' }
  FROM { <table_name> | ( <query> ) }
  [ PARTITION BY <expr> ]
  [ FILE_FORMAT = ( ... ) ]
  [ HEADER = TRUE | FALSE ]
  [ MAX_FILE_SIZE = <num> ]
  [ OVERWRITE = TRUE | FALSE ]
  [ SINGLE = TRUE | FALSE ]
  [ INCLUDE_QUERY_ID = TRUE | FALSE ];
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `PARTITION BY` | Expression | NULL | Partition files by expression (creates subdirectories) | `PARTITION BY DATE(order_date)` |
| `HEADER` | Boolean | FALSE | Include header row in CSV files | `HEADER=TRUE` |
| `MAX_FILE_SIZE` | Integer | 16000000 | Max bytes per file (default 16MB) | `MAX_FILE_SIZE=100000000` |
| `OVERWRITE` | Boolean | FALSE | Overwrite existing files | `OVERWRITE=TRUE` |
| `SINGLE` | Boolean | FALSE | Write to single file (disables parallelism) | `SINGLE=TRUE` |
| `INCLUDE_QUERY_ID` | Boolean | FALSE | Append query ID to filenames | `INCLUDE_QUERY_ID=TRUE` |

#### Example

```sql
-- Unload to internal stage (CSV)
COPY INTO @my_stage/exports/orders_
  FROM sales.orders
  FILE_FORMAT = (TYPE = CSV COMPRESSION = GZIP HEADER = TRUE)
  MAX_FILE_SIZE = 50000000  -- 50MB per file
  OVERWRITE = TRUE;

-- Unload to S3 (Parquet, partitioned by date)
COPY INTO @s3_export_stage/orders/
  FROM (
    SELECT order_id, customer_id, order_date, total_amount
    FROM sales.orders
    WHERE order_date >= '2025-01-01'
  )
  PARTITION BY ('year=' || YEAR(order_date) || '/month=' || MONTH(order_date))
  FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY)
  HEADER = TRUE;

-- Download from internal stage to local
GET @my_stage/exports/ file:///tmp/exports/;
```

---

## Best Practices

### File Sizing
- **Optimal**: 100-250MB compressed per file
- **Min**: >10MB (avoid tiny files, poor parallelism)
- **Max**: <5GB (better error handling, faster retries)

### Compression
- **CSV/JSON**: GZIP (best balance), BROTLI (highest ratio)
- **Parquet/ORC**: Built-in columnar compression (Snappy recommended)
- **Avro**: Deflate or Snappy

### Parallelism
- COPY INTO parallelizes per file (more files = faster load)
- Snowpipe processes files in parallel (auto-scaling)
- Split large files into 100-250MB chunks for optimal throughput

### Error Handling
- **Development**: `ON_ERROR='CONTINUE'` (load all valid rows)
- **Production**: `ON_ERROR='SKIP_FILE_5%'` (skip file if >5% errors)
- **Critical**: `ON_ERROR='ABORT_STATEMENT'` (fail fast)

### Monitoring
```sql
-- Check COPY history
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => 'FACT_ORDERS',
  START_TIME => DATEADD(day, -1, CURRENT_TIMESTAMP())
));

-- Monitor Snowpipe
SELECT * FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
  PIPE_NAME => 'ORDERS_PIPE',
  START_TIME => DATEADD(hour, -24, CURRENT_TIMESTAMP())
));
```

---

## SDLC Use Case: Nightly ETL in Agile Sprints

### Scenario: Scheduled Batch Load from S3

```python
# nightly_etl.py (Airflow/Prefect DAG)
import snowflake.connector
from datetime import datetime, timedelta

class SnowflakeETL:
    def __init__(self, account, user, password, warehouse):
        self.conn = snowflake.connector.connect(
            account=account,
            user=user,
            password=password,
            warehouse=warehouse,
            role='ETL_ROLE'
        )
        self.cursor = self.conn.cursor()
    
    def load_daily_orders(self, load_date):
        """Load orders for specific date from S3"""
        year = load_date.year
        month = str(load_date.month).zfill(2)
        day = str(load_date.day).zfill(2)
        
        sql = f"""
        COPY INTO prod.sales.fact_orders (
          order_id, customer_id, order_date, total_amount, status
        )
        FROM (
          SELECT 
            $1::INT,
            $2::INT,
            $3::DATE,
            $4::DECIMAL(12,2),
            $5::VARCHAR
          FROM @prod.stages.s3_orders_stage/{year}/{month}/{day}/
        )
        PATTERN = '.*orders.*[.]parquet'
        FILE_FORMAT = (TYPE = PARQUET)
        ON_ERROR = 'SKIP_FILE_10%'
        PURGE = TRUE
        """
        
        try:
            result = self.cursor.execute(sql)
            rows_loaded = result.fetchone()[0]
            print(f"✅ Loaded {rows_loaded} rows for {load_date}")
            return rows_loaded
        except Exception as e:
            print(f"❌ Load failed for {load_date}: {e}")
            raise
    
    def validate_load(self, load_date):
        """Validate data quality post-load"""
        sql = f"""
        SELECT 
          COUNT(*) AS total_rows,
          COUNT(DISTINCT customer_id) AS unique_customers,
          SUM(total_amount) AS total_revenue,
          MIN(order_date) AS min_date,
          MAX(order_date) AS max_date
        FROM prod.sales.fact_orders
        WHERE order_date = '{load_date}'
        """
        
        result = self.cursor.execute(sql).fetchone()
        metrics = {
            'total_rows': result[0],
            'unique_customers': result[1],
            'total_revenue': float(result[2]),
            'min_date': result[3],
            'max_date': result[4]
        }
        
        print(f"📊 Validation: {metrics}")
        
        # Assertions
        assert metrics['total_rows'] > 0, "No rows loaded"
        assert metrics['min_date'] == load_date, "Date mismatch"
        
        return metrics

# Airflow DAG
if __name__ == '__main__':
    etl = SnowflakeETL(
        account='myorg.us-east-1',
        user='etl_user',
        password='***',
        warehouse='etl_wh'
    )
    
    # Load yesterday's data
    load_date = datetime.now().date() - timedelta(days=1)
    
    # Execute ETL
    etl.load_daily_orders(load_date)
    etl.validate_load(load_date)
```

---

## Key Concepts

- **COPY INTO**: Bulk load command for batch data ingestion from staged files
- **Snowpipe**: Serverless continuous ingestion using cloud event notifications
- **Storage Integration**: Secure delegated access to cloud storage without embedded credentials
- **File Format**: Specification for parsing CSV, JSON, Parquet, Avro, ORC, XML files
- **Stage**: Named location for data files (internal Snowflake storage or external cloud storage)

---
