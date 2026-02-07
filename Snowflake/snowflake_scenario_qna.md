# 50 Scenario-Based Snowflake Data Engineer — Questions & Answers

## Table of Contents
- [Fundamental Questions & Answers (1-10)](#fundamental-questions--answers)
- [Intermediate Questions & Answers (11-35)](#intermediate-questions--answers)
- [Advanced Questions & Answers (36-50)](#advanced-questions--answers)

---

## Fundamental Questions & Answers

---

### Question 1: Initial Data Load from S3

**Difficulty**: Fundamental | **Primary Skill**: AWS S3 | **Topic**: External Stages and COPY Command

**Scenario**:  
You've been tasked with loading 50GB of customer data from CSV files stored in an S3 bucket (s3://company-data/customers/) into a new Snowflake table. The CSV files have headers, are pipe-delimited, and some text fields contain commas. This is a one-time historical load, and data quality is critical—you need to ensure all records load successfully or identify any problematic records.

**Question**:  
What steps would you take to set up the external stage, create the target table, and load the data? How would you handle potential errors during the load process?

---

**✅ Answer**:

I would follow a systematic 5-step approach:

**Step 1 — Create a Storage Integration (recommended over raw credentials)**

```sql
CREATE STORAGE INTEGRATION s3_customer_integration
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = 'S3'
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-access-role'
    STORAGE_ALLOWED_LOCATIONS = ('s3://company-data/customers/');
```

This is more secure than embedding AWS keys directly. You then `DESC INTEGRATION s3_customer_integration` to retrieve `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID`, and configure the AWS trust policy accordingly.

**Step 2 — Create a File Format**

```sql
CREATE OR REPLACE FILE FORMAT csv_pipe_format
    TYPE = 'CSV'
    FIELD_DELIMITER = '|'
    SKIP_HEADER = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'   -- handles commas inside text fields
    NULL_IF = ('NULL', 'null', '')
    ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE
    TRIM_SPACE = TRUE;
```

Key points: `FIELD_OPTIONALLY_ENCLOSED_BY` ensures commas inside quoted text fields are not treated as delimiters. `ERROR_ON_COLUMN_COUNT_MISMATCH` catches structural file issues early.

**Step 3 — Create External Stage and Target Table**

```sql
CREATE OR REPLACE STAGE s3_customer_stage
    STORAGE_INTEGRATION = s3_customer_integration
    URL = 's3://company-data/customers/'
    FILE_FORMAT = csv_pipe_format;

-- Verify files are accessible
LIST @s3_customer_stage;

-- Create target table
CREATE OR REPLACE TABLE customers (
    customer_id     NUMBER,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address         VARCHAR(500),
    city            VARCHAR(100),
    state           VARCHAR(50),
    zip_code        VARCHAR(20),
    signup_date     DATE,
    status          VARCHAR(20)
);
```

**Step 4 — Validate Before Loading (Dry Run)**

```sql
-- Test with VALIDATION_MODE to preview errors without actually loading
COPY INTO customers
FROM @s3_customer_stage
VALIDATION_MODE = 'RETURN_ERRORS';

-- Or check first N rows
COPY INTO customers
FROM @s3_customer_stage
VALIDATION_MODE = 'RETURN_10_ROWS';
```

This catches issues before any data is committed.

**Step 5 — Load with Error Handling**

```sql
-- For critical one-time load, use ABORT_STATEMENT to get clean load
COPY INTO customers
FROM @s3_customer_stage
ON_ERROR = 'ABORT_STATEMENT'
PURGE = FALSE;                         -- Keep source files intact for a one-time load

-- If some bad rows are expected, use CONTINUE and check rejected records
COPY INTO customers
FROM @s3_customer_stage
ON_ERROR = 'CONTINUE';

-- Post-load validation
SELECT COUNT(*) AS loaded_rows FROM customers;

-- Check load history for errors
SELECT
    file_name,
    status,
    rows_parsed,
    rows_loaded,
    error_count,
    first_error_message,
    first_error_line_num
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'CUSTOMERS',
    START_TIME => DATEADD('hour', -1, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```

**Summary of ON_ERROR Options**:

| Option | Behavior | Best For |
|--------|----------|----------|
| `ABORT_STATEMENT` | Stops on first error, loads nothing | Critical loads needing 100% success |
| `CONTINUE` | Skips bad rows, loads the rest | Tolerance for some data loss |
| `SKIP_FILE` | Skips entire file with errors | File-level granularity |
| `SKIP_FILE_<n>%` | Skips file if error % exceeds n | Threshold-based tolerance |

---

### Question 2: Basic dbt Model Creation

**Difficulty**: Fundamental | **Primary Skill**: dbt | **Topic**: dbt Core Concepts

**Scenario**:  
Your team is adopting dbt for data transformations in Snowflake. You need to create your first dbt model that transforms raw order data from a staging table into a cleaned orders table. The transformation should filter out cancelled orders, convert timestamps to dates, and rename columns to follow company naming conventions (snake_case).

**Question**:  
How would you structure this dbt model? What materialization strategy would you choose and why? How would you test the output?

---

**✅ Answer**:

**Model File — `models/staging/stg_orders.sql`**

```sql
{{ config(
    materialized='table',
    schema='staging',
    tags=['staging', 'daily']
) }}

WITH source AS (
    SELECT * FROM {{ source('raw_data', 'orders') }}
),

cleaned AS (
    SELECT
        OrderID             AS order_id,
        CustomerID          AS customer_id,
        ProductID           AS product_id,
        OrderAmount         AS order_amount,
        OrderTimestamp::DATE AS order_date,
        OrderStatus         AS order_status,
        ShippingAddress     AS shipping_address,
        CreatedAt           AS created_at,
        UpdatedAt           AS updated_at
    FROM source
    WHERE UPPER(OrderStatus) != 'CANCELLED'
)

SELECT * FROM cleaned
```

**Materialization Choice — `table`**:

I chose `table` over `view` because:
- The staging table is queried by multiple downstream models. A table avoids re-executing the query each time.
- For a view, every downstream query re-scans the raw data, which is wasteful for large datasets.
- `table` is better for performance and caching; `view` is better for small, simple transformations where you want to always see latest data.

When would I use `view`? For lightweight 1:1 column renames on small tables where cost of re-execution is negligible.

**Source Configuration — `models/staging/_sources.yml`**

```yaml
version: 2

sources:
  - name: raw_data
    database: RAW_DB
    schema: RAW_SCHEMA
    tables:
      - name: orders
        loaded_at_field: UpdatedAt
        freshness:
          warn_after: { count: 12, period: hour }
          error_after: { count: 24, period: hour }
        columns:
          - name: OrderID
            tests:
              - not_null
```

**Testing — `models/staging/_staging.yml`**

```yaml
version: 2

models:
  - name: stg_orders
    description: "Cleaned orders table - cancelled orders removed, timestamps converted to dates"
    columns:
      - name: order_id
        description: "Unique order identifier"
        tests:
          - unique
          - not_null

      - name: customer_id
        tests:
          - not_null

      - name: order_status
        tests:
          - accepted_values:
              values: ['PENDING', 'PROCESSING', 'SHIPPED', 'DELIVERED', 'RETURNED']

      - name: order_amount
        tests:
          - not_null
```

**Run & Test Commands**:

```bash
dbt run --select stg_orders          # Build the model
dbt test --select stg_orders         # Run all tests
dbt source freshness                 # Check source freshness
dbt docs generate && dbt docs serve  # Generate documentation
```

---

### Question 3: Virtual Warehouse Sizing

**Difficulty**: Fundamental | **Primary Skill**: Snowflake | **Topic**: Warehouse Configuration

**Scenario**:  
Your company has three different workloads: a nightly ETL process that loads 100GB of data (runs 2 hours), an analytics team running ad-hoc queries throughout the day (5-10 concurrent users), and a dashboard refresh process that runs every hour (takes 5 minutes). Currently, everything uses a single X-Large warehouse running 24/7, and monthly costs are $8,000.

**Question**:  
How would you optimize the warehouse configuration to reduce costs while maintaining performance?

---

**✅ Answer**:

**Current Problem Analysis**:

A single X-Large warehouse running 24/7:
- X-Large = 16 credits/hour × 24 hours × 30 days = **11,520 credits/month**
- At ~$3/credit = ~$34K (but at $8K, likely using standard pricing or smaller active time)
- Regardless, one warehouse for all workloads means: paying for idle time, no workload isolation, contention between ETL and analytics.

**Recommended Configuration — 3 Separate Warehouses**:

```sql
-- 1. ETL Warehouse: Needs compute power but only runs at night
CREATE WAREHOUSE ETL_WH
    WAREHOUSE_SIZE = 'LARGE'          -- Sufficient for 100GB load
    AUTO_SUSPEND = 60                 -- Suspend 1 min after ETL completes
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 1
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'Nightly ETL processing';

-- 2. Analytics Warehouse: Concurrent ad-hoc users during business hours
CREATE WAREHOUSE ANALYTICS_WH
    WAREHOUSE_SIZE = 'MEDIUM'         -- Good for ad-hoc queries
    AUTO_SUSPEND = 300                -- 5 min idle timeout
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3             -- Scale out for concurrency
    SCALING_POLICY = 'STANDARD'       -- Fast scaling for user experience
    COMMENT = 'Ad-hoc analytics queries';

-- 3. Dashboard Warehouse: Small, frequent, predictable workload
CREATE WAREHOUSE DASHBOARD_WH
    WAREHOUSE_SIZE = 'SMALL'          -- 5-min refresh doesn't need much
    AUTO_SUSPEND = 120                -- Suspend between hourly runs
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 1
    COMMENT = 'Hourly dashboard refresh';
```

**Cost Comparison**:

| Warehouse | Size | Credits/Hr | Active Hours/Day | Monthly Credits | Monthly Cost |
|-----------|------|------------|------------------|-----------------|-------------|
| ETL_WH | Large | 8 | 2 hrs (nightly) | 8 × 2 × 30 = 480 | ~$1,440 |
| ANALYTICS_WH | Medium | 4 | ~10 hrs (business hrs) | 4 × 10 × 22 = 880 | ~$2,640 |
| DASHBOARD_WH | Small | 2 | ~1.5 hrs (5 min × 24) | 2 × 1.5 × 30 = 90 | ~$270 |
| **Total** | | | | **1,450** | **~$4,350** |

**Savings: ~$3,650/month (45% reduction)**

**Key Principles**:
- **Scale UP** for complex queries (increase warehouse size = more compute per node)
- **Scale OUT** for concurrent users (multi-cluster = more parallel capacity)
- **Auto-suspend aggressively** — idle warehouses cost money
- **Never run 24/7** unless there's continuous load

---

### Question 4: Simple SQL Window Function

**Difficulty**: Fundamental | **Primary Skill**: SQL | **Topic**: Window Functions

**Scenario**:  
Your sales team wants a report showing each sales transaction alongside the running total of sales per customer. The source table (SALES) contains columns: transaction_id, customer_id, transaction_date, and amount.

**Question**:  
Write a SQL query that produces this report. How would you ensure the running total restarts for each customer?

---

**✅ Answer**:

```sql
SELECT
    customer_id,
    transaction_id,
    transaction_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY transaction_date, transaction_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM sales
ORDER BY customer_id, transaction_date, transaction_id;
```

**How It Works**:

- **`PARTITION BY customer_id`**: Restarts the running total for each customer. Without this, the running total would span the entire table.
- **`ORDER BY transaction_date, transaction_id`**: Defines the accumulation order. I include `transaction_id` as a tiebreaker when a customer has multiple transactions on the same date, ensuring deterministic results.
- **`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`**: Explicitly defines the window frame to include all rows from the start of the partition up to and including the current row. This is technically the default when ORDER BY is specified, but being explicit avoids ambiguity.

**Sample Output**:

| customer_id | transaction_id | transaction_date | amount | running_total |
|-------------|---------------|------------------|--------|---------------|
| C001 | T101 | 2024-01-05 | 150.00 | 150.00 |
| C001 | T108 | 2024-01-12 | 200.00 | 350.00 |
| C001 | T115 | 2024-02-01 | 75.00 | 425.00 |
| C002 | T102 | 2024-01-06 | 300.00 | 300.00 |
| C002 | T110 | 2024-01-15 | 50.00 | 350.00 |

**Important Note on Window Frames**:
- `ROWS BETWEEN` counts physical rows — exact and predictable
- `RANGE BETWEEN` (the default without explicit frame) groups rows with the same ORDER BY value — which can produce unexpected results with duplicate dates

---

### Question 5: Python Snowflake Connection

**Difficulty**: Fundamental | **Primary Skill**: Python | **Topic**: Snowflake Connector Basics

**Scenario**:  
You need to create a Python script that connects to Snowflake, executes a simple SELECT query from the CUSTOMERS table, and writes the results to a CSV file. The script will run daily via cron job, and credentials should not be hardcoded.

**Question**:  
How would you implement this Python script?

---

**✅ Answer**:

```python
import os
import logging
import pandas as pd
import snowflake.connector
from datetime import datetime

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('snowflake_export.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

def get_snowflake_connection():
    """Create Snowflake connection using environment variables."""
    try:
        conn = snowflake.connector.connect(
            account=os.environ['SNOWFLAKE_ACCOUNT'],
            user=os.environ['SNOWFLAKE_USER'],
            password=os.environ['SNOWFLAKE_PASSWORD'],
            warehouse=os.environ.get('SNOWFLAKE_WAREHOUSE', 'COMPUTE_WH'),
            database=os.environ.get('SNOWFLAKE_DATABASE', 'PROD_DB'),
            schema=os.environ.get('SNOWFLAKE_SCHEMA', 'PUBLIC'),
            role=os.environ.get('SNOWFLAKE_ROLE', 'DATA_ENGINEER')
        )
        logger.info("Connected to Snowflake successfully.")
        return conn
    except Exception as e:
        logger.error(f"Failed to connect to Snowflake: {e}")
        raise

def export_customers_to_csv():
    """Query CUSTOMERS table and write to CSV."""
    query = """
        SELECT
            customer_id,
            first_name,
            last_name,
            email,
            signup_date,
            status
        FROM customers
        WHERE status = 'ACTIVE'
        ORDER BY customer_id
    """

    output_file = f"customers_export_{datetime.now().strftime('%Y%m%d')}.csv"

    conn = None
    try:
        conn = get_snowflake_connection()
        cursor = conn.cursor()

        logger.info("Executing query...")
        cursor.execute(query)

        # Option 1: Using pandas (recommended for medium datasets)
        df = cursor.fetch_pandas_all()
        df.to_csv(output_file, index=False, encoding='utf-8')

        logger.info(f"Exported {len(df)} rows to {output_file}")

    except snowflake.connector.errors.ProgrammingError as e:
        logger.error(f"SQL Error: {e.msg}")
        raise
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        raise
    finally:
        if conn:
            conn.close()
            logger.info("Snowflake connection closed.")

if __name__ == '__main__':
    export_customers_to_csv()
```

**Setting Up Environment Variables**:

```bash
# .env file (never commit to git)
export SNOWFLAKE_ACCOUNT='xy12345.us-east-1'
export SNOWFLAKE_USER='etl_service_user'
export SNOWFLAKE_PASSWORD='secure_password_here'
export SNOWFLAKE_WAREHOUSE='EXPORT_WH'
export SNOWFLAKE_DATABASE='PROD_DB'
export SNOWFLAKE_SCHEMA='PUBLIC'

# Cron entry (runs daily at 6 AM)
0 6 * * * source /path/to/.env && python /path/to/export_script.py
```

**Security Best Practices**:
- Never hardcode credentials — use environment variables, AWS Secrets Manager, or HashiCorp Vault
- Use a service account with minimal privileges (SELECT only on needed tables)
- Consider key-pair authentication instead of password for automated scripts
- The `.env` file should be in `.gitignore` and have restricted file permissions (`chmod 600`)

---

### Question 6: Time Travel Recovery

**Difficulty**: Fundamental | **Primary Skill**: Snowflake | **Topic**: Time Travel Feature

**Scenario**:  
A developer accidentally ran a DELETE statement without a WHERE clause on the PRODUCTS table at 2:30 PM, removing all 50,000 product records. The mistake was discovered at 3:15 PM.

**Question**:  
How would you recover the deleted data?

---

**✅ Answer**:

**Step 1 — Verify the Damage**

```sql
-- Check current state (should be 0 rows)
SELECT COUNT(*) FROM products;

-- Confirm the table still exists (it wasn't DROPped)
SHOW TABLES LIKE 'PRODUCTS';
```

**Step 2 — Query Historical Data Using Time Travel**

```sql
-- View data as it was BEFORE the accidental DELETE (e.g., 2:25 PM)
SELECT COUNT(*) FROM products AT(TIMESTAMP => '2024-01-15 14:25:00'::TIMESTAMP_NTZ);
-- Should return 50,000
```

**Step 3 — Create Recovery Table (Safe Approach)**

```sql
-- Don't overwrite directly — create a recovery table first for verification
CREATE TABLE products_recovery AS
SELECT * FROM products AT(TIMESTAMP => '2024-01-15 14:25:00'::TIMESTAMP_NTZ);

-- Verify the recovery
SELECT COUNT(*) FROM products_recovery;   -- Should be 50,000

-- Spot-check some records
SELECT * FROM products_recovery LIMIT 20;
```

**Step 4 — Restore Data**

```sql
-- Option A: TRUNCATE and INSERT (preserves table structure, grants, constraints)
TRUNCATE TABLE products;
INSERT INTO products SELECT * FROM products_recovery;

-- Option B: SWAP (atomic rename — faster for large tables)
ALTER TABLE products SWAP WITH products_recovery;

-- Verify final state
SELECT COUNT(*) FROM products;  -- 50,000
```

**Step 5 — Cleanup**

```sql
DROP TABLE IF EXISTS products_recovery;
```

**Important Distinctions**:

| Situation | Recovery Method |
|-----------|----------------|
| Rows deleted (DELETE) | Time Travel with `AT` or `BEFORE` clause |
| Table dropped (DROP TABLE) | `UNDROP TABLE products;` |
| Table truncated (TRUNCATE) | Time Travel with `AT` or `BEFORE` clause |

**Alternative Time Travel Syntax**:

```sql
-- Using OFFSET (seconds ago) instead of timestamp
SELECT * FROM products AT(OFFSET => -3600);  -- 1 hour ago

-- Using STATEMENT (query_id of the destructive operation)
SELECT * FROM products BEFORE(STATEMENT => '01b2c3d4-0001-a1b2-0000-00000000c3d4');
```

**Retention Period Awareness**:
- Standard Edition: 1 day
- Enterprise Edition: Up to 90 days (configurable per table)
- After Time Travel expires, Fail-safe provides 7 additional days (Snowflake support only)

---

### Question 7: dbt Source Configuration

**Difficulty**: Fundamental | **Primary Skill**: dbt | **Topic**: Source Management

**Scenario**:  
Your dbt project references several raw tables in Snowflake's RAW_DATA schema directly using database.schema.table syntax, making it difficult to switch between environments.

**Question**:  
How would you configure dbt sources to make your project more maintainable?

---

**✅ Answer**:

**Create `models/staging/_sources.yml`**:

```yaml
version: 2

sources:
  - name: raw_data
    description: "Raw data ingested from production systems"
    database: "{{ env_var('RAW_DATABASE', 'RAW_DB') }}"
    schema: RAW_SCHEMA
    
    freshness:
      warn_after: { count: 12, period: hour }
      error_after: { count: 24, period: hour }
    loaded_at_field: _loaded_at   # Common metadata column

    tables:
      - name: customers
        description: "Raw customer records from CRM system"
        columns:
          - name: customer_id
            description: "Unique customer identifier"
            tests:
              - unique
              - not_null

      - name: orders
        description: "Raw order transactions"
        columns:
          - name: order_id
            tests:
              - unique
              - not_null

      - name: products
        description: "Product catalog from inventory system"
        identifier: product_master   # Actual table name if different
        columns:
          - name: product_id
            tests:
              - unique
              - not_null
```

**Using Sources in Models**:

```sql
-- BEFORE (hardcoded — fragile)
SELECT * FROM RAW_DB.RAW_SCHEMA.customers

-- AFTER (using source function — maintainable)
SELECT * FROM {{ source('raw_data', 'customers') }}
```

**Benefits**:

1. **Single point of configuration**: Change database/schema in one place — all models update automatically.
2. **Environment flexibility**: Use `env_var()` to swap databases between dev, staging, and prod.
3. **Freshness monitoring**: `dbt source freshness` checks if source data is stale.
4. **Documentation**: Source descriptions auto-populate in `dbt docs`.
5. **Lineage tracking**: dbt knows the full DAG from raw sources → final models.
6. **Testing at source**: Catch data issues before they propagate downstream.

**Running Freshness Checks**:

```bash
dbt source freshness   # Checks all sources
dbt source freshness --select source:raw_data  # Check specific source
```

---

### Question 8: Basic Query Performance Issue

**Difficulty**: Fundamental | **Primary Skill**: SQL | **Topic**: Query Optimization Basics

**Scenario**:  
A dashboard query joining TRANSACTIONS (500M rows) with CUSTOMERS (100K rows) filtered on last 30 days takes 45 seconds. No errors, just slow.

**Question**:  
What initial steps would you take to diagnose and improve performance?

---

**✅ Answer**:

**Step 1 — Examine the Query Profile**

In Snowflake UI → Query History → Click the query → Query Profile tab. Look for:
- **Percentage of partitions scanned** — If scanning 80%+ of partitions for a 30-day filter on a multi-year table, clustering is needed
- **Bytes spilled to local/remote disk** — Indicates memory pressure (need larger warehouse or query rewrite)
- **Join explosion** — Unexpected row multiplication from bad join logic
- **Full table scan** on TRANSACTIONS — The 30-day filter should eliminate most data

**Step 2 — Check Filter Effectiveness**

```sql
-- How selective is the filter?
SELECT
    COUNT(*) AS total_rows,
    COUNT(CASE WHEN transaction_date >= CURRENT_DATE - 30 THEN 1 END) AS filtered_rows,
    ROUND(filtered_rows / total_rows * 100, 2) AS filter_pct
FROM transactions;
-- If filtered_rows is ~5% of total, the filter should be very effective with clustering
```

**Step 3 — Check Clustering on transaction_date**

```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('transactions', '(transaction_date)');
-- Look at: average_depth, average_overlaps
-- average_depth close to 1 = well clustered
-- average_depth > 5 = poorly clustered, needs reclustering
```

**Step 4 — Optimize the Query**

```sql
-- BEFORE: Potentially inefficient
SELECT t.*, c.customer_name, c.region
FROM transactions t
JOIN customers c ON t.customer_id = c.customer_id
WHERE t.transaction_date >= CURRENT_DATE - 30;

-- AFTER: Force early filtering with CTE
WITH recent_transactions AS (
    SELECT *
    FROM transactions
    WHERE transaction_date >= CURRENT_DATE - 30
)
SELECT rt.*, c.customer_name, c.region
FROM recent_transactions rt
INNER JOIN customers c ON rt.customer_id = c.customer_id;
```

**Step 5 — Add Clustering Key (if not present)**

```sql
ALTER TABLE transactions CLUSTER BY (transaction_date);
-- This enables partition pruning — Snowflake will only scan partitions containing last 30 days
```

**Step 6 — Consider Materialized View for Dashboards**

```sql
CREATE MATERIALIZED VIEW mv_recent_transactions AS
SELECT t.*, c.customer_name, c.region
FROM transactions t
INNER JOIN customers c ON t.customer_id = c.customer_id
WHERE t.transaction_date >= CURRENT_DATE - 30;
-- Dashboard queries hit this MV directly — much faster
```

**Expected Improvement**: From 45 seconds → 3-5 seconds with clustering + early filtering.

---

### Question 9: File Format JSON Handling

**Difficulty**: Fundamental | **Primary Skill**: AWS S3 | **Topic**: Semi-Structured Data Loading

**Scenario**:  
You need to load JSON files from S3 containing user events with nested attributes. Files arrive hourly at s3://events-bucket/raw/YYYY/MM/DD/HH/.

**Question**:  
How would you set up the stage and file format? What table structure would you use?

---

**✅ Answer**:

**Step 1 — Create JSON File Format**

```sql
CREATE OR REPLACE FILE FORMAT json_events_format
    TYPE = 'JSON'
    STRIP_OUTER_ARRAY = TRUE      -- Each file contains a JSON array [...]; this unwraps it
    DATE_FORMAT = 'AUTO'
    TIMESTAMP_FORMAT = 'AUTO'
    COMPRESSION = 'AUTO';         -- Auto-detect gzip, bzip2, etc.
```

**Step 2 — Create External Stage**

```sql
CREATE OR REPLACE STAGE s3_events_stage
    STORAGE_INTEGRATION = s3_events_integration
    URL = 's3://events-bucket/raw/'
    FILE_FORMAT = json_events_format;

-- Verify
LIST @s3_events_stage/2024/01/15/;
```

**Step 3 — Create Target Table with VARIANT**

```sql
CREATE OR REPLACE TABLE raw_events (
    raw_data    VARIANT,          -- Stores entire JSON object
    filename    VARCHAR,          -- Source file tracking
    load_time   TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

Using `VARIANT` is ideal because:
- It handles schema evolution naturally (new fields added by source don't break anything)
- No need to define every column upfront
- Snowflake internally stores VARIANT data in columnar format for fast querying

**Step 4 — Load Data**

```sql
COPY INTO raw_events (raw_data, filename)
FROM (
    SELECT
        $1,                         -- The JSON object
        METADATA$FILENAME           -- Captures the source file path
    FROM @s3_events_stage/2024/01/15/
)
PATTERN = '.*\.json'
ON_ERROR = 'CONTINUE';
```

**Step 5 — Query Nested JSON Data**

```sql
-- Extract top-level fields
SELECT
    raw_data:user_id::STRING          AS user_id,
    raw_data:event_type::STRING       AS event_type,
    raw_data:timestamp::TIMESTAMP     AS event_timestamp,
    raw_data:session_id::STRING       AS session_id
FROM raw_events
WHERE load_time >= CURRENT_DATE;

-- Extract nested properties object
SELECT
    raw_data:user_id::STRING                     AS user_id,
    raw_data:event_type::STRING                  AS event_type,
    raw_data:properties:page_url::STRING         AS page_url,
    raw_data:properties:referrer::STRING          AS referrer,
    raw_data:properties:device_type::STRING       AS device_type,
    raw_data:properties:utm_source::STRING        AS utm_source
FROM raw_events;
```

**Schema Evolution Handling**: When the source adds new fields (e.g., `properties:experiment_id`), they're automatically stored in VARIANT. No ALTER TABLE needed — just update your downstream queries or dbt models to extract the new fields.

---

### Question 10: Role-Based Access Control

**Difficulty**: Fundamental | **Primary Skill**: Snowflake | **Topic**: Security and RBAC

**Scenario**:  
Onboarding 5 analytics users who need read-only access to ANALYTICS schema but not FINANCE schema. They need a dedicated ANALYTICS_WH warehouse.

**Question**:  
How would you set up roles and permissions?

---

**✅ Answer**:

```sql
-- Step 1: Create the custom role
CREATE ROLE analytics_reader;

-- Step 2: Grant database-level access
GRANT USAGE ON DATABASE prod_db TO ROLE analytics_reader;

-- Step 3: Grant schema-level access (ANALYTICS only, NOT finance)
GRANT USAGE ON SCHEMA prod_db.analytics TO ROLE analytics_reader;

-- Step 4: Grant table/view access in ANALYTICS schema
GRANT SELECT ON ALL TABLES IN SCHEMA prod_db.analytics TO ROLE analytics_reader;
GRANT SELECT ON ALL VIEWS IN SCHEMA prod_db.analytics TO ROLE analytics_reader;

-- Step 5: Grant FUTURE privileges (auto-applies to new tables/views)
GRANT SELECT ON FUTURE TABLES IN SCHEMA prod_db.analytics TO ROLE analytics_reader;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA prod_db.analytics TO ROLE analytics_reader;

-- Step 6: Grant warehouse usage
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE analytics_reader;

-- Step 7: Assign role to users
GRANT ROLE analytics_reader TO USER analyst_alice;
GRANT ROLE analytics_reader TO USER analyst_bob;
GRANT ROLE analytics_reader TO USER analyst_carol;
GRANT ROLE analytics_reader TO USER analyst_dave;
GRANT ROLE analytics_reader TO USER analyst_eve;

-- Step 8: Set up role hierarchy (best practice)
GRANT ROLE analytics_reader TO ROLE sysadmin;  -- SYSADMIN can assume this role
```

**Role Hierarchy Best Practice**:

```
ACCOUNTADMIN
    └── SYSADMIN
            ├── analytics_reader    (read-only analytics)
            ├── finance_reader      (read-only finance)
            └── data_engineer       (read/write all schemas)
```

**Verification**:

```sql
-- Verify grants
SHOW GRANTS TO ROLE analytics_reader;

-- Test as the role
USE ROLE analytics_reader;
SELECT * FROM prod_db.analytics.some_table LIMIT 5;   -- Should work ✅
SELECT * FROM prod_db.finance.some_table LIMIT 5;     -- Should fail ❌
```

**Why FUTURE GRANTS are critical**: Without them, when a data engineer creates a new table in the ANALYTICS schema tomorrow, the analytics team won't have access to it until someone manually runs another GRANT. FUTURE GRANTS solve this by automatically applying SELECT permission to any new objects created in the schema.

---

## Intermediate Questions & Answers

---

### Question 11: Incremental dbt Model Design

**Difficulty**: Intermediate | **Primary Skill**: dbt | **Topic**: Incremental Materialization

**Scenario**:  
You're building a DAILY_SALES_SUMMARY table aggregating 2 billion row TRANSACTIONS table (growing 5-10M rows daily). Full refresh takes 3 hours; you need hourly updates.

**Question**:  
How would you design this as an incremental dbt model? How would you handle late-arriving data?

---

**✅ Answer**:

```sql
-- models/marts/fct_daily_sales_summary.sql

{{ config(
    materialized='incremental',
    unique_key=['transaction_date', 'store_id', 'product_category'],
    incremental_strategy='merge',
    on_schema_change='append_new_columns',
    tags=['marts', 'hourly']
) }}

WITH source_transactions AS (
    SELECT
        transaction_date,
        store_id,
        product_category,
        SUM(amount) AS total_sales,
        COUNT(*) AS transaction_count,
        COUNT(DISTINCT customer_id) AS unique_customers,
        AVG(amount) AS avg_transaction_value
    FROM {{ ref('stg_transactions') }}

    {% if is_incremental() %}
        -- LOOKBACK WINDOW: Re-process last 3 days to catch late-arriving data
        WHERE transaction_date >= (
            SELECT DATEADD('day', -3, MAX(transaction_date))
            FROM {{ this }}
        )
    {% endif %}

    GROUP BY transaction_date, store_id, product_category
)

SELECT
    {{ dbt_utils.generate_surrogate_key(
        ['transaction_date', 'store_id', 'product_category']
    ) }} AS summary_id,
    transaction_date,
    store_id,
    product_category,
    total_sales,
    transaction_count,
    unique_customers,
    avg_transaction_value,
    CURRENT_TIMESTAMP() AS last_updated_at
FROM source_transactions
```

**How It Works**:

1. **First run**: `is_incremental()` returns FALSE → processes all data (full load).
2. **Subsequent runs**: `is_incremental()` returns TRUE → only processes last 3 days.
3. **`merge` strategy**: Matches on `unique_key` — if a row exists for the same (date, store, category), it UPDATEs. If new, it INSERTs. This handles late-arriving data that changes previously computed aggregates.

**Why 3-Day Lookback?**
Late-arriving data (e.g., offline POS transactions synced with delay) can modify past dates. Without a lookback window, those changes would be missed. 3 days balances data accuracy vs. processing cost.

**Scheduling**:
```bash
# Hourly incremental (fast — only 3 days of data)
dbt run --select fct_daily_sales_summary

# Weekly full refresh (ensures long-term accuracy)
dbt run --full-refresh --select fct_daily_sales_summary
```

**Performance Comparison**:

| Mode | Rows Processed | Runtime | Credits |
|------|---------------|---------|---------|
| Full Refresh | 2 billion | ~3 hours | ~180 |
| Incremental (3-day window) | ~15-30 million | ~5-10 minutes | ~5-10 |

---

### Question 12: Clustering Key Strategy

**Difficulty**: Intermediate | **Primary Skill**: Snowflake | **Topic**: Performance Optimization

**Scenario**:  
EVENTS table: 800M rows, 2TB, 5 years of data. Queries filter by event_date (last 90 days) and user_id. Queries take 30+ seconds with extensive partition scanning.

**Question**:  
Should you implement clustering keys? What strategy would you choose?

---

**✅ Answer**:

**Step 1 — Diagnose Current Clustering**

```sql
-- Check clustering quality on event_date
SELECT SYSTEM$CLUSTERING_INFORMATION('events', '(event_date)');

-- Example bad output:
-- { "cluster_by_keys": "(event_date)",
--   "total_partition_count": 25000,
--   "average_overlaps": 12.5,
--   "average_depth": 8.2 }
-- average_depth > 5 = poorly clustered → lots of unnecessary partition scanning
```

**Step 2 — Implement Clustering**

```sql
-- Cluster on event_date (primary filter in most queries)
ALTER TABLE events CLUSTER BY (event_date);

-- For queries that also filter on user_id, consider compound key:
-- ALTER TABLE events CLUSTER BY (event_date, user_id);
-- CAUTION: user_id has very high cardinality → may not help much
-- Rule: Put lower-cardinality column first
```

**Why `event_date` as the clustering key?**
- It's the most common filter in queries (last 90 days).
- Date has moderate cardinality (1,825 distinct values for 5 years).
- Partition pruning can eliminate ~95% of data (90 days out of 1,825 = ~5% scanned).

**Step 3 — Monitor Reclustering**

```sql
-- Check automatic reclustering costs
SELECT
    table_name,
    start_time,
    credits_used,
    num_bytes_reclustered,
    num_rows_reclustered
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE table_name = 'EVENTS'
    AND start_time >= DATEADD('day', -7, CURRENT_DATE)
ORDER BY start_time DESC;
```

**Step 4 — Verify Improvement**

```sql
-- After reclustering completes, check again
SELECT SYSTEM$CLUSTERING_INFORMATION('events', '(event_date)');
-- average_depth should now be 1-2

-- Test query performance
SELECT COUNT(*), SUM(event_value)
FROM events
WHERE event_date >= CURRENT_DATE - 90;
-- Check Query Profile: partitions scanned should drop dramatically
```

**Trade-offs**:

| Benefit | Cost |
|---------|------|
| 70-90% fewer partitions scanned per query | Ongoing reclustering credits (~$50-200/month for 2TB) |
| Queries go from 30s → 3-5s | Slight delay for DML operations |
| Better result cache utilization | Initial reclustering may take hours |

**When NOT to Cluster**:
- Tables < 1GB (too small to benefit)
- Tables with heavy random inserts and no dominant query pattern
- Tables that are mostly INSERT-only and already naturally ordered by time

---

### Question 13: Complex SQL Cohort Analysis

**Difficulty**: Intermediate | **Primary Skill**: SQL | **Topic**: Advanced SQL Techniques

**Scenario**:  
Marketing needs cohort analysis: customer retention by signup month, showing purchases in month 0 through month 12.

**Question**:  
Write the SQL query for this cohort retention report.

---

**✅ Answer**:

```sql
WITH cohorts AS (
    -- Assign each customer to their signup month cohort
    SELECT
        customer_id,
        DATE_TRUNC('month', signup_date) AS cohort_month
    FROM customers
),

customer_orders AS (
    -- Get each customer's order months and calculate months since signup
    SELECT
        c.customer_id,
        c.cohort_month,
        DATE_TRUNC('month', o.order_date) AS order_month,
        DATEDIFF('month', c.cohort_month, DATE_TRUNC('month', o.order_date)) AS months_since_signup
    FROM cohorts c
    INNER JOIN orders o
        ON c.customer_id = o.customer_id
),

cohort_activity AS (
    -- Count distinct active customers per cohort per period
    SELECT
        cohort_month,
        months_since_signup,
        COUNT(DISTINCT customer_id) AS active_customers
    FROM customer_orders
    WHERE months_since_signup BETWEEN 0 AND 12
    GROUP BY cohort_month, months_since_signup
),

cohort_sizes AS (
    -- Total customers per cohort (for retention percentage)
    SELECT
        cohort_month,
        COUNT(DISTINCT customer_id) AS cohort_size
    FROM cohorts
    GROUP BY cohort_month
)

SELECT
    ca.cohort_month,
    cs.cohort_size,
    ca.months_since_signup,
    ca.active_customers,
    ROUND(ca.active_customers / cs.cohort_size * 100, 2) AS retention_pct
FROM cohort_activity ca
JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
ORDER BY ca.cohort_month, ca.months_since_signup;
```

**Sample Output**:

| cohort_month | cohort_size | months_since_signup | active_customers | retention_pct |
|-------------|-------------|---------------------|------------------|---------------|
| 2024-01-01 | 1,200 | 0 | 850 | 70.83% |
| 2024-01-01 | 1,200 | 1 | 520 | 43.33% |
| 2024-01-01 | 1,200 | 2 | 410 | 34.17% |
| 2024-02-01 | 1,500 | 0 | 1,100 | 73.33% |
| 2024-02-01 | 1,500 | 1 | 680 | 45.33% |

**Pivot for Dashboard View** (optional):

```sql
SELECT
    cohort_month,
    cohort_size,
    MAX(CASE WHEN months_since_signup = 0 THEN retention_pct END) AS month_0,
    MAX(CASE WHEN months_since_signup = 1 THEN retention_pct END) AS month_1,
    MAX(CASE WHEN months_since_signup = 2 THEN retention_pct END) AS month_2,
    -- ... continue to month_12
    MAX(CASE WHEN months_since_signup = 12 THEN retention_pct END) AS month_12
FROM (/* main query above */) pivoted
GROUP BY cohort_month, cohort_size
ORDER BY cohort_month;
```

---

### Question 14: Snowpipe Automation

**Difficulty**: Intermediate | **Primary Skill**: Snowflake | **Topic**: Continuous Data Ingestion

**Scenario**:  
JSON log files land in S3 every 5 minutes. You need automatic loading into Snowflake within 10 minutes.

**Question**:  
How would you implement Snowpipe and monitor it?

---

**✅ Answer**:

**Step 1 — Create Target Table and Stage**

```sql
CREATE TABLE raw_app_logs (
    raw_data    VARIANT,
    filename    VARCHAR,
    load_time   TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

CREATE STAGE s3_logs_stage
    STORAGE_INTEGRATION = s3_integration
    URL = 's3://app-logs/production/'
    FILE_FORMAT = (TYPE = 'JSON' STRIP_OUTER_ARRAY = TRUE);
```

**Step 2 — Create Snowpipe**

```sql
CREATE PIPE app_logs_pipe
    AUTO_INGEST = TRUE
    COMMENT = 'Auto-ingest JSON logs from S3'
AS
COPY INTO raw_app_logs (raw_data, filename)
FROM (
    SELECT $1, METADATA$FILENAME
    FROM @s3_logs_stage
)
PATTERN = '.*\.json';
```

**Step 3 — Get SQS ARN for S3 Event Notification**

```sql
-- Get the notification channel ARN
SHOW PIPES LIKE 'app_logs_pipe';
-- Copy the 'notification_channel' value (looks like arn:aws:sqs:us-east-1:...)
```

**Step 4 — Configure S3 Event Notification (AWS Console or CLI)**

```json
// S3 Bucket → Properties → Event Notifications → Create
{
  "Event": "s3:ObjectCreated:*",
  "Filter": { "Key": { "FilterRules": [{ "Name": "suffix", "Value": ".json" }] } },
  "Queue": "arn:aws:sqs:us-east-1:123456789:sf-snowpipe-<account>"
}
```

**Step 5 — Monitoring**

```sql
-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('app_logs_pipe');
-- Returns: {"executionState":"RUNNING", "pendingFileCount":0, ...}

-- Check recent load history
SELECT
    file_name,
    stage_location,
    status,
    row_count,
    first_error_message,
    pipe_received_time,
    last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'RAW_APP_LOGS',
    START_TIME => DATEADD('hour', -6, CURRENT_TIMESTAMP())
))
ORDER BY pipe_received_time DESC;

-- Check pipe usage/costs
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE pipe_name = 'APP_LOGS_PIPE'
    AND start_time >= DATEADD('day', -7, CURRENT_DATE)
ORDER BY start_time DESC;
```

**Snowpipe Cost Model**: Snowpipe uses serverless compute (no warehouse needed). Charges are based on file count and compute time. For frequent small files, Snowpipe is cheaper than scheduling a dedicated warehouse. For large batch loads, scheduled COPY with a warehouse is often more cost-effective.

---

### Question 15: Data Quality Framework in dbt

**Difficulty**: Intermediate | **Primary Skill**: dbt | **Topic**: Testing and Data Quality

**Scenario**:  
50+ dbt models, data quality issues slipping into production. Duplicate order records caused incorrect revenue.

**Question**:  
How would you design a comprehensive testing strategy?

---

**✅ Answer**:

**Layer 1 — Schema Tests (`_schema.yml`)**

```yaml
version: 2

models:
  - name: fct_orders
    description: "Fact table for orders"
    columns:
      - name: order_id
        tests:
          - unique
          - not_null

      - name: order_line_key
        tests:
          - unique    # Catches the duplicate issue!
          - not_null

      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id

      - name: order_status
        tests:
          - accepted_values:
              values: ['pending', 'shipped', 'delivered', 'returned', 'cancelled']

      - name: order_amount
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 1000000    # Flag outliers
```

**Layer 2 — Custom Singular Tests**

```sql
-- tests/assert_no_duplicate_order_lines.sql
-- Catch the exact issue: duplicate order records
SELECT
    order_id,
    line_number,
    COUNT(*) AS duplicate_count
FROM {{ ref('fct_orders') }}
GROUP BY order_id, line_number
HAVING COUNT(*) > 1

-- tests/assert_daily_revenue_within_bounds.sql
-- Detect anomalous revenue swings (> 50% day-over-day change)
WITH daily_revenue AS (
    SELECT
        order_date,
        SUM(order_amount) AS revenue,
        LAG(SUM(order_amount)) OVER (ORDER BY order_date) AS prev_day_revenue
    FROM {{ ref('fct_orders') }}
    WHERE order_date >= CURRENT_DATE - 30
    GROUP BY order_date
)
SELECT *
FROM daily_revenue
WHERE revenue > prev_day_revenue * 1.5
   OR revenue < prev_day_revenue * 0.5
```

**Layer 3 — Row Count Reconciliation**

```sql
-- tests/assert_row_count_source_vs_target.sql
WITH source_count AS (
    SELECT COUNT(*) AS cnt FROM {{ source('raw_data', 'orders') }}
    WHERE order_date >= CURRENT_DATE - 1
),
target_count AS (
    SELECT COUNT(*) AS cnt FROM {{ ref('stg_orders') }}
    WHERE order_date >= CURRENT_DATE - 1
)
SELECT
    s.cnt AS source_rows,
    t.cnt AS target_rows,
    s.cnt - t.cnt AS difference
FROM source_count s, target_count t
WHERE ABS(s.cnt - t.cnt) > s.cnt * 0.01   -- Alert if > 1% difference
```

**Layer 4 — Test Configuration**

```yaml
# dbt_project.yml
tests:
  +severity: error              # Default: fail the run
  +store_failures: true         # Save failed records to schema for investigation
  +schema: dbt_test_audit       # Store failures in dedicated schema
```

**Execution Strategy**:

```bash
# CI/CD: Run tests on PR (before merge to main)
dbt test --select state:modified+   # Only test changed models

# Production: Run tests after every dbt run
dbt run && dbt test

# Source freshness: Run before dbt run
dbt source freshness && dbt run && dbt test
```

---

### Question 16: Python ETL Orchestration

**Difficulty**: Intermediate | **Primary Skill**: Python | **Topic**: Pipeline Automation

**Scenario**:  
Multi-step ETL: SFTP → S3 → Snowflake COPY → Validation → Email notification. Must handle partial failures.

**Question**:  
How would you architect this?

---

**✅ Answer**:

```python
import os
import logging
import paramiko
import boto3
import snowflake.connector
import smtplib
from email.mime.text import MIMEText
from datetime import datetime
from time import sleep

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')

class ETLPipeline:
    def __init__(self):
        self.results = {}
        self.errors = []

    def step_1_download_from_sftp(self, sftp_host, sftp_user, sftp_key, remote_dir, local_dir):
        """Download CSV files from SFTP server."""
        logger.info("Step 1: Downloading from SFTP...")
        downloaded_files = []
        transport = paramiko.Transport((sftp_host, 22))
        try:
            transport.connect(username=sftp_user, pkey=paramiko.RSAKey.from_private_key_file(sftp_key))
            sftp = paramiko.SFTPClient.from_transport(transport)

            for filename in sftp.listdir(remote_dir):
                if filename.endswith('.csv'):
                    remote_path = f"{remote_dir}/{filename}"
                    local_path = f"{local_dir}/{filename}"
                    sftp.get(remote_path, local_path)
                    downloaded_files.append(local_path)
                    logger.info(f"  Downloaded: {filename}")

            self.results['sftp_files'] = downloaded_files
            return downloaded_files
        except Exception as e:
            logger.error(f"SFTP download failed: {e}")
            raise
        finally:
            transport.close()

    def step_2_upload_to_s3(self, local_files, bucket, prefix):
        """Upload files to S3 with retry."""
        logger.info("Step 2: Uploading to S3...")
        s3 = boto3.client('s3')
        uploaded = []
        for filepath in local_files:
            filename = os.path.basename(filepath)
            s3_key = f"{prefix}/{datetime.now().strftime('%Y/%m/%d')}/{filename}"

            for attempt in range(3):  # Retry up to 3 times
                try:
                    s3.upload_file(filepath, bucket, s3_key)
                    uploaded.append(s3_key)
                    logger.info(f"  Uploaded: {s3_key}")
                    break
                except Exception as e:
                    if attempt == 2:
                        raise
                    logger.warning(f"  Retry {attempt+1} for {filename}: {e}")
                    sleep(2 ** attempt)

        self.results['s3_files'] = uploaded
        return uploaded

    def step_3_load_to_snowflake(self, stage_path):
        """Trigger COPY INTO from S3 stage."""
        logger.info("Step 3: Loading into Snowflake...")
        conn = snowflake.connector.connect(
            account=os.environ['SF_ACCOUNT'],
            user=os.environ['SF_USER'],
            password=os.environ['SF_PASSWORD'],
            warehouse='ETL_WH', database='RAW_DB', schema='STAGING'
        )
        try:
            cursor = conn.cursor()
            cursor.execute(f"""
                COPY INTO staging_table
                FROM @s3_stage/{stage_path}
                FILE_FORMAT = csv_format
                ON_ERROR = 'CONTINUE'
            """)
            result = cursor.fetchone()
            rows_loaded = result[3] if result else 0
            self.results['rows_loaded'] = rows_loaded
            logger.info(f"  Loaded {rows_loaded} rows into Snowflake")
            return rows_loaded
        finally:
            conn.close()

    def step_4_validate(self):
        """Run data validation queries."""
        logger.info("Step 4: Validating data...")
        conn = snowflake.connector.connect(
            account=os.environ['SF_ACCOUNT'], user=os.environ['SF_USER'],
            password=os.environ['SF_PASSWORD'],
            warehouse='ETL_WH', database='RAW_DB', schema='STAGING'
        )
        try:
            cursor = conn.cursor()
            # Check for nulls in required columns
            cursor.execute("SELECT COUNT(*) FROM staging_table WHERE customer_id IS NULL AND load_date = CURRENT_DATE")
            null_count = cursor.fetchone()[0]

            # Check row count
            cursor.execute("SELECT COUNT(*) FROM staging_table WHERE load_date = CURRENT_DATE")
            row_count = cursor.fetchone()[0]

            passed = null_count == 0 and row_count > 0
            self.results['validation'] = {'passed': passed, 'null_count': null_count, 'row_count': row_count}
            logger.info(f"  Validation: {'PASSED' if passed else 'FAILED'} (nulls={null_count}, rows={row_count})")
            return passed
        finally:
            conn.close()

    def step_5_notify(self, success):
        """Send email notification."""
        subject = f"ETL Pipeline {'SUCCESS' if success else 'FAILED'} - {datetime.now().strftime('%Y-%m-%d')}"
        body = f"Results:\n{self.results}\n\nErrors:\n{self.errors if self.errors else 'None'}"
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = 'etl@company.com'
        msg['To'] = 'data-team@company.com'

        with smtplib.SMTP(os.environ['SMTP_HOST'], 587) as server:
            server.starttls()
            server.login(os.environ['SMTP_USER'], os.environ['SMTP_PASS'])
            server.send_message(msg)
        logger.info(f"  Notification sent: {subject}")

    def run(self):
        """Orchestrate the full pipeline with error handling at each step."""
        overall_success = True
        try:
            files = self.step_1_download_from_sftp('/remote/data', '/tmp/etl')
            self.step_2_upload_to_s3(files, 'data-bucket', 'raw/customers')
            self.step_3_load_to_snowflake('raw/customers')
            validation_passed = self.step_4_validate()
            if not validation_passed:
                overall_success = False
                self.errors.append("Data validation failed")
        except Exception as e:
            overall_success = False
            self.errors.append(str(e))
            logger.error(f"Pipeline failed: {e}")
        finally:
            self.step_5_notify(overall_success)

        return overall_success
```

**Key Design Patterns**: Modular steps (each can be retried independently), retry with exponential backoff for network operations, checkpoint-like results tracking, notification regardless of outcome, credentials via environment variables.

---

### Question 17: Warehouse Spillage Resolution

**Difficulty**: Intermediate | **Primary Skill**: Snowflake | **Topic**: Performance Troubleshooting

**Scenario**:  
Nightly aggregation query failing with "resources exceeded." Query Profile shows local and remote disk spillage on 500GB data with X-Large warehouse.

**Question**:  
What causes disk spillage and how would you resolve it?

---

**✅ Answer**:

**What Causes Spillage?**

Snowflake processes data in memory. When operations (JOINs, GROUP BY, ORDER BY, window functions) require more memory than the warehouse provides, data "spills" to local SSD first, then to remote storage. Spillage degrades performance exponentially.

**Memory per Warehouse Size** (approximate):

| Size | Nodes | Memory | Best For |
|------|-------|--------|----------|
| X-Small | 1 | ~8GB | Small queries |
| Small | 1 | ~16GB | Light workloads |
| Medium | 2-4 | ~32-64GB | Moderate joins |
| Large | 4-8 | ~64-128GB | Complex queries |
| X-Large | 8-16 | ~128-256GB | Heavy aggregations |
| 2X-Large | 16-32 | ~256-512GB | Very large datasets |

**Short-Term Fix — Scale Up**:

```sql
ALTER WAREHOUSE etl_wh SET WAREHOUSE_SIZE = '2X-LARGE';
-- Rerun the query — spillage should reduce or eliminate
```

**Long-Term Fixes — Query Optimization**:

```sql
-- BEFORE: One giant query processing everything
SELECT
    region, product_category, order_month,
    SUM(amount), COUNT(*), AVG(amount),
    RANK() OVER (PARTITION BY region ORDER BY SUM(amount) DESC)
FROM fact_orders
JOIN dim_products ON ...
JOIN dim_customers ON ...
JOIN dim_regions ON ...
GROUP BY region, product_category, order_month;

-- AFTER: Break into stages to reduce memory pressure at each step
CREATE TEMPORARY TABLE tmp_pre_agg AS
SELECT
    product_id,
    customer_id,
    region_id,
    DATE_TRUNC('month', order_date) AS order_month,
    SUM(amount) AS total_amount,
    COUNT(*) AS order_count
FROM fact_orders
WHERE order_date >= DATEADD('year', -1, CURRENT_DATE)  -- Filter early!
GROUP BY 1, 2, 3, 4;

-- Then join with dimension tables on the smaller pre-aggregated dataset
SELECT
    r.region_name,
    p.product_category,
    t.order_month,
    SUM(t.total_amount),
    SUM(t.order_count),
    RANK() OVER (PARTITION BY r.region_name ORDER BY SUM(t.total_amount) DESC)
FROM tmp_pre_agg t
JOIN dim_products p ON t.product_id = p.product_id
JOIN dim_regions r ON t.region_id = r.region_id
GROUP BY 1, 2, 3;
```

**Key Strategies**:
1. **Filter early**: Add WHERE clauses before JOINs/GROUP BY
2. **Pre-aggregate**: Reduce rows before expensive operations
3. **Break into temp tables**: Let Snowflake process smaller chunks
4. **Add clustering keys**: Reduce data scanned for filtered queries
5. **Avoid SELECT ***: Only include needed columns

---

### Question 18: CDC Implementation

**Difficulty**: Intermediate | **Primary Skill**: Snowflake | **Topic**: Change Data Capture

**Scenario**:  
Sync data from Oracle (50M rows, 100K daily changes) to Snowflake. Full daily loads take 4 hours.

**Question**:  
How would you implement CDC?

---

**✅ Answer**:

**Approach: Timestamp-Based CDC with MERGE**

**Step 1 — Extract Changed Records from Oracle**

```sql
-- Oracle query (run by extraction tool)
SELECT * FROM oracle_schema.customers
WHERE last_modified_date > :last_sync_timestamp;
```

**Step 2 — Stage in S3 and Load to Snowflake Staging**

```sql
COPY INTO stg_customers
FROM @s3_oracle_stage/customers/
FILE_FORMAT = parquet_format
ON_ERROR = 'ABORT_STATEMENT';
```

**Step 3 — MERGE into Target Table**

```sql
MERGE INTO prod.customers AS target
USING stg_customers AS source
ON target.customer_id = source.customer_id

WHEN MATCHED AND source.last_modified_date > target.last_modified_date THEN
    UPDATE SET
        target.customer_name    = source.customer_name,
        target.email            = source.email,
        target.address          = source.address,
        target.status           = source.status,
        target.last_modified_date = source.last_modified_date,
        target.etl_updated_at   = CURRENT_TIMESTAMP()

WHEN NOT MATCHED THEN
    INSERT (customer_id, customer_name, email, address, status, last_modified_date, etl_updated_at)
    VALUES (source.customer_id, source.customer_name, source.email, source.address,
            source.status, source.last_modified_date, CURRENT_TIMESTAMP());
```

**Step 4 — Handle Deletes**

```sql
-- Option A: Soft delete flag
UPDATE prod.customers
SET is_deleted = TRUE, etl_updated_at = CURRENT_TIMESTAMP()
WHERE customer_id NOT IN (SELECT customer_id FROM stg_customers_full_snapshot)
  AND is_deleted = FALSE;

-- Option B: Maintain a delete audit table in Oracle
-- Extract deleted IDs and apply in Snowflake
```

**Step 5 — Track Watermark**

```sql
-- Store last successful sync timestamp
MERGE INTO etl.sync_watermarks AS t
USING (SELECT 'oracle.customers' AS source_table, CURRENT_TIMESTAMP() AS last_sync) AS s
ON t.source_table = s.source_table
WHEN MATCHED THEN UPDATE SET t.last_sync = s.last_sync
WHEN NOT MATCHED THEN INSERT VALUES (s.source_table, s.last_sync);
```

**Performance Comparison**:

| Approach | Data Extracted | Runtime | Source Impact |
|----------|---------------|---------|---------------|
| Full Load | 50M rows | ~4 hours | Heavy |
| CDC (timestamp) | ~100K rows | ~10 minutes | Minimal |

**Alternative: Snowflake Streams (for Snowflake-to-Snowflake CDC)**:

```sql
CREATE STREAM customers_stream ON TABLE raw.customers;
-- Stream tracks INSERT, UPDATE, DELETE with METADATA$ACTION, METADATA$ISUPDATE
```

---

### Question 19: Cost Optimization Analysis

**Difficulty**: Intermediate | **Primary Skill**: Snowflake | **Topic**: Cost Management

**Scenario**:  
Monthly Snowflake bill jumped from $15K to $32K. Finance wants a breakdown and optimization plan.

**Question**:  
What queries would you run to analyze costs?

---

**✅ Answer**:

```sql
-- 1. Top consuming warehouses (find the culprits)
SELECT
    warehouse_name,
    SUM(credits_used) AS total_credits,
    ROUND(SUM(credits_used) * 3.00, 2) AS estimated_cost,
    ROUND(SUM(credits_used) * 100.0 /
        (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
         WHERE start_time >= DATEADD('month', -1, CURRENT_DATE)), 1) AS pct_of_total
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('month', -1, CURRENT_DATE)
GROUP BY warehouse_name
ORDER BY total_credits DESC;

-- 2. Idle warehouse detection
SELECT
    warehouse_name,
    SUM(credits_used) AS credits_consumed,
    COUNT(DISTINCT DATE_TRUNC('hour', start_time)) AS active_hours,
    ROUND(SUM(credits_used) / NULLIF(COUNT(DISTINCT DATE_TRUNC('hour', start_time)), 0), 2) AS credits_per_hour
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('month', -1, CURRENT_DATE)
GROUP BY warehouse_name
HAVING active_hours < 10   -- Barely used
ORDER BY credits_consumed DESC;

-- 3. Most expensive queries
SELECT
    query_id,
    query_text,
    user_name,
    warehouse_name,
    warehouse_size,
    ROUND(execution_time / 60000, 1) AS runtime_minutes,
    ROUND(bytes_scanned / POWER(1024, 3), 2) AS gb_scanned,
    partitions_scanned,
    partitions_total,
    ROUND(bytes_spilled_to_local_storage / POWER(1024, 3), 2) AS gb_spilled
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('month', -1, CURRENT_DATE)
    AND execution_time > 60000    -- > 1 minute
ORDER BY execution_time DESC
LIMIT 20;

-- 4. Month-over-month trend (to understand the spike)
SELECT
    DATE_TRUNC('week', start_time) AS week,
    warehouse_name,
    SUM(credits_used) AS weekly_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('month', -3, CURRENT_DATE)
GROUP BY week, warehouse_name
ORDER BY week, weekly_credits DESC;
```

**Typical Findings & Actions**:

| Finding | Impact | Action |
|---------|--------|--------|
| DEV_WH running 24/7 with auto_suspend=3600 | ~$2K/month wasted | Set auto_suspend=120 |
| 3 zombie warehouses (0 queries) | ~$1K/month | Suspend/drop them |
| Nightly ETL on 2X-Large (only needs Large) | ~$3K/month | Right-size to Large |
| 5 queries with massive spillage | ~$4K/month | Optimize queries, add clustering |
| Developers using ACCOUNTADMIN warehouse | ~$2K/month | Enforce role-based warehouse access |

---

### Question 20: dbt Snapshot Implementation

**Difficulty**: Intermediate | **Primary Skill**: dbt | **Topic**: Slowly Changing Dimensions

**Scenario**:  
PRODUCTS table gets frequent price/description/category updates. Business needs historical tracking.

**Question**:  
How would you implement dbt snapshots?

---

**✅ Answer**:

```sql
-- snapshots/products_snapshot.sql
{% snapshot products_snapshot %}

{{
    config(
        target_database='analytics_db',
        target_schema='snapshots',
        unique_key='product_id',
        strategy='timestamp',
        updated_at='last_updated_at',
        invalidate_hard_deletes=True
    )
}}

SELECT
    product_id,
    product_name,
    description,
    category,
    subcategory,
    price,
    cost,
    supplier_id,
    is_active,
    last_updated_at
FROM {{ source('raw_data', 'products') }}

{% endsnapshot %}
```

**How It Works**:

1. **First run**: Inserts all products with `dbt_valid_from = now()`, `dbt_valid_to = NULL` (current records).
2. **Subsequent runs**: Compares `last_updated_at` to the snapshot. If changed → closes old record (`dbt_valid_to = now()`) and inserts new record (`dbt_valid_from = now()`).

**Result: SCD Type 2 Table**:

| product_id | product_name | price | dbt_valid_from | dbt_valid_to | dbt_scd_id |
|-----------|--------------|-------|----------------|--------------|------------|
| P001 | Widget A | 9.99 | 2024-01-01 | 2024-03-15 | abc123 |
| P001 | Widget A | 12.99 | 2024-03-15 | NULL | def456 |

**Querying Historical State**:

```sql
-- What was the price of product P001 on Feb 1, 2024?
SELECT *
FROM analytics_db.snapshots.products_snapshot
WHERE product_id = 'P001'
    AND '2024-02-01' >= dbt_valid_from
    AND '2024-02-01' < COALESCE(dbt_valid_to, '9999-12-31');
-- Returns: price = 9.99

-- All current products
SELECT * FROM analytics_db.snapshots.products_snapshot
WHERE dbt_valid_to IS NULL;
```

**Running**: `dbt snapshot` (separate from `dbt run`). Schedule to run before your `dbt run` to capture state changes first.

---

### Question 21: Multi-Environment dbt Strategy

**Difficulty**: Intermediate | **Primary Skill**: dbt | **Topic**: Development Workflow

**Scenario**:  
5 data engineers overwriting each other's work in DEV. No clear promotion path to production.

**Question**:  
How would you structure multi-environment dbt?

---

**✅ Answer**:

**`profiles.yml`**:

```yaml
analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: DEV_DB
      schema: "dbt_{{ env_var('DBT_USER', 'default') }}"   # dbt_john, dbt_sarah
      warehouse: DEV_WH
      role: DEV_ROLE

    staging:
      type: snowflake
      database: STAGING_DB
      schema: analytics
      warehouse: STAGING_WH
      role: STAGING_ROLE

    prod:
      type: snowflake
      database: PROD_DB
      schema: analytics
      warehouse: PROD_WH
      role: PROD_ROLE
```

**Developer Workflow**:

1. **Local development**: Each developer runs `dbt run` → output goes to `DEV_DB.dbt_john` (isolated)
2. **Pull Request**: CI pipeline runs `dbt run --target staging && dbt test` → validates in STAGING_DB
3. **Merge to main**: CD pipeline runs `dbt run --target prod` → deploys to PROD_DB.analytics

**Custom Schema Macro** (`macros/generate_schema_name.sql`):

```sql
{% macro generate_schema_name(custom_schema_name, node) %}
    {%- set default_schema = target.schema -%}
    {%- if target.name == 'prod' -%}
        {{ custom_schema_name | default(default_schema) }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | default('default') }}
    {%- endif -%}
{% endmacro %}
```

This ensures prod uses clean schema names (e.g., `analytics.staging`) while dev uses developer-prefixed schemas (e.g., `dbt_john_staging`).

---

### Question 22: Lateral Flatten JSON

**Difficulty**: Intermediate | **Primary Skill**: SQL | **Topic**: Semi-Structured Data Processing

**Scenario**:  
EVENTS table has VARIANT column with "tags" array. Need each tag as a separate row.

**Question**:  
Write the FLATTEN query and discuss performance.

---

**✅ Answer**:

```sql
-- Basic FLATTEN
SELECT
    e.event_id,
    e.event_data:user_id::STRING       AS user_id,
    e.event_data:event_type::STRING    AS event_type,
    e.event_data:timestamp::TIMESTAMP  AS event_timestamp,
    f.index                             AS tag_position,
    f.value::STRING                     AS tag
FROM events e,
LATERAL FLATTEN(input => e.event_data:tags) f
WHERE e.event_date >= CURRENT_DATE - 30;

-- With OUTER => TRUE (keep events with empty/null tags)
SELECT
    e.event_id,
    f.value::STRING AS tag
FROM events e,
LATERAL FLATTEN(input => e.event_data:tags, OUTER => TRUE) f;
```

**How LATERAL FLATTEN Works**:
- `LATERAL` allows the FLATTEN function to reference columns from the `events` table.
- `FLATTEN` expands the array into rows — one row per array element.
- For `tags = ["sale", "homepage", "mobile"]`, you get 3 rows with the same event_id.
- `f.index` = position in array (0, 1, 2), `f.value` = the element value.

**Performance Tips**:
- Always filter the base table first (`WHERE event_date >= ...`) before FLATTEN
- For frequently queried flattened data, create a materialized view:

```sql
CREATE MATERIALIZED VIEW mv_event_tags AS
SELECT
    event_id,
    event_data:user_id::STRING AS user_id,
    f.value::STRING AS tag,
    event_date
FROM events,
LATERAL FLATTEN(input => event_data:tags) f;
```

---

### Question 23: Stream and Task Orchestration

**Difficulty**: Intermediate | **Primary Skill**: Snowflake | **Topic**: Native Orchestration

**Scenario**:  
Automatically trigger transformations when new ORDERS records arrive.

**Question**:  
Implement using Streams and Tasks.

---

**✅ Answer**:

```sql
-- Step 1: Create stream on source table
CREATE OR REPLACE STREAM orders_stream ON TABLE raw.orders
    APPEND_ONLY = FALSE;  -- Track INSERTs, UPDATEs, and DELETEs

-- Step 2: Create transformation task
CREATE OR REPLACE TASK process_new_orders
    WAREHOUSE = ETL_WH
    SCHEDULE = '5 MINUTE'     -- Check every 5 minutes
    WHEN SYSTEM$STREAM_HAS_DATA('orders_stream')   -- Only run if there's new data
AS
    MERGE INTO analytics.orders_summary AS target
    USING (
        SELECT
            DATE_TRUNC('day', order_date) AS summary_date,
            product_category,
            SUM(CASE WHEN METADATA$ACTION = 'INSERT' THEN amount ELSE 0 END) AS new_amount,
            COUNT(CASE WHEN METADATA$ACTION = 'INSERT' THEN 1 END) AS new_orders
        FROM orders_stream
        GROUP BY 1, 2
    ) AS source
    ON target.summary_date = source.summary_date
       AND target.product_category = source.product_category
    WHEN MATCHED THEN UPDATE SET
        target.total_amount = target.total_amount + source.new_amount,
        target.order_count = target.order_count + source.new_orders,
        target.last_updated = CURRENT_TIMESTAMP()
    WHEN NOT MATCHED THEN INSERT
        (summary_date, product_category, total_amount, order_count, last_updated)
    VALUES (source.summary_date, source.product_category, source.new_amount, source.new_orders, CURRENT_TIMESTAMP());

-- Step 3: Create dependent child task for customer metrics
CREATE OR REPLACE TASK update_customer_metrics
    WAREHOUSE = ETL_WH
    AFTER process_new_orders     -- Runs only after parent completes
AS
    MERGE INTO analytics.customer_metrics AS target
    USING (/* customer aggregation query */) AS source
    ON target.customer_id = source.customer_id
    WHEN MATCHED THEN UPDATE SET ...
    WHEN NOT MATCHED THEN INSERT ...;

-- Step 4: Resume tasks (tasks are created in SUSPENDED state)
ALTER TASK update_customer_metrics RESUME;
ALTER TASK process_new_orders RESUME;    -- Resume parent LAST

-- Monitoring
SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'PROCESS_NEW_ORDERS',
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP())
))
ORDER BY scheduled_time DESC;
```

**Key Points**:
- `WHEN SYSTEM$STREAM_HAS_DATA()` prevents the task from running (and consuming credits) when there's no new data.
- Stream offset is consumed after successful task completion — if the task fails, data remains in the stream for retry.
- Resume child tasks first, parent task last.

---

### Question 24: Complex Join Optimization

**Difficulty**: Intermediate | **Primary Skill**: SQL | **Topic**: Query Performance

**Scenario**:  
5-table join query (ORDERS 10M, CUSTOMERS 500K, PRODUCTS 100K, ORDER_ITEMS 50M, SHIPMENTS 12M) taking 3 minutes.

**Question**:  
How would you optimize this?

---

**✅ Answer**:

```sql
-- OPTIMIZED APPROACH: Filter early, pre-aggregate, then join dimensions

WITH recent_orders AS (
    -- Step 1: Filter FIRST to reduce from 10M to ~300K rows
    SELECT order_id, customer_id, order_date, order_status
    FROM orders
    WHERE order_date >= CURRENT_DATE - 30
),

order_items_agg AS (
    -- Step 2: Pre-aggregate items (50M → ~300K after filtering)
    SELECT
        oi.order_id,
        SUM(oi.quantity) AS total_quantity,
        SUM(oi.quantity * oi.unit_price) AS total_amount,
        COUNT(DISTINCT oi.product_id) AS unique_products
    FROM order_items oi
    INNER JOIN recent_orders ro ON oi.order_id = ro.order_id  -- Filter pushdown
    GROUP BY oi.order_id
),

shipment_info AS (
    -- Step 3: Get latest shipment per order
    SELECT
        order_id,
        MAX(shipped_date) AS last_shipped_date,
        MAX(delivered_date) AS delivered_date
    FROM shipments
    WHERE order_id IN (SELECT order_id FROM recent_orders)
    GROUP BY order_id
)

-- Step 4: Join pre-aggregated results with small dimension tables
SELECT
    ro.order_id,
    ro.order_date,
    c.customer_name,
    c.region,
    oia.total_quantity,
    oia.total_amount,
    oia.unique_products,
    si.last_shipped_date,
    si.delivered_date
FROM recent_orders ro
INNER JOIN customers c ON ro.customer_id = c.customer_id
INNER JOIN order_items_agg oia ON ro.order_id = oia.order_id
LEFT JOIN shipment_info si ON ro.order_id = si.order_id
ORDER BY ro.order_date DESC;
```

**Why This Is Faster**:
- Filters 10M → 300K rows before any join
- Pre-aggregates 50M order_items to 300K grouped rows
- Joins small result sets with small dimension tables
- Expected: 3 minutes → 10-15 seconds

---

### Question 25–35: (Condensed Answers)

### Question 25: External Table Performance

**✅ Answer**: External tables are slower because data stays in S3 (no micro-partitions, no result cache, no clustering). Optimize by: defining partitions in DDL, using Parquet format, ensuring 100-1000MB file sizes, and creating materialized views on frequently queried external table data. Use native tables for frequent queries; external tables for archival or multi-platform data access.

---

### Question 26: dbt Macro Development

**✅ Answer**:

```sql
-- macros/generate_date_spine.sql
{% macro generate_date_spine(start_date, end_date, granularity='day') %}
WITH date_spine AS (
    SELECT
        DATEADD(
            '{{ granularity }}',
            ROW_NUMBER() OVER (ORDER BY SEQ4()) - 1,
            '{{ start_date }}'::DATE
        ) AS date_{{ granularity }}
    FROM TABLE(GENERATOR(ROWCOUNT =>
        DATEDIFF('{{ granularity }}', '{{ start_date }}'::DATE, '{{ end_date }}'::DATE) + 1
    ))
)
SELECT * FROM date_spine
{% endmacro %}

-- Usage in a model:
-- {{ generate_date_spine('2020-01-01', '2025-12-31', 'day') }}
```

---

### Question 27: Python Data Validation

**✅ Answer**: Create validation functions for row count comparison, null checks, range validation, and referential integrity. Use snowflake-connector for executing validation SQL. Store results in an audit table. Send Slack/email alerts on failure. Consider Great Expectations for a more sophisticated framework with expectation suites and data docs.

---

### Question 28: Secure Data Sharing

**✅ Answer**:

```sql
-- Create secure views to filter PII and limit to 12 months
CREATE SECURE VIEW shared_orders AS
SELECT order_id, customer_id, order_date, amount
FROM orders WHERE order_date >= DATEADD('month', -12, CURRENT_DATE);

CREATE SECURE VIEW shared_customers AS
SELECT customer_id, first_name, last_name, 'REDACTED' AS email, city, state
FROM customers;

-- Create share and grant access
CREATE SHARE partner_share;
GRANT USAGE ON DATABASE sales_db TO SHARE partner_share;
GRANT USAGE ON SCHEMA sales_db.shared TO SHARE partner_share;
GRANT SELECT ON VIEW shared_orders TO SHARE partner_share;
GRANT SELECT ON VIEW shared_customers TO SHARE partner_share;
ALTER SHARE partner_share ADD ACCOUNTS = partner_account_id;
-- Partner runs: CREATE DATABASE sales_from_partner FROM SHARE our_account.partner_share;
```

No data duplication — queries run on consumer's compute.

---

### Question 29: Query Result Caching Strategy

**✅ Answer**: Snowflake's result cache stores exact query results for 24 hours. Cache hit requires: exact same SQL text, no underlying table changes, no volatile functions like `CURRENT_TIMESTAMP()`. To maximize hits: standardize query text, use `DATE_TRUNC('hour', CURRENT_TIMESTAMP())` instead of `NOW()`, schedule data refreshes at fixed times, and use materialized views for dashboard queries. Result cache is free (no warehouse needed) and shared across all users.

---

### Question 30: Error Handling in dbt

**✅ Answer**: Implement source freshness checks before runs. Use severity levels (warn vs. error) on tests. Enable `--store-failures` to save failed rows. Use `on-run-start` hooks for prerequisite validation. Use `--fail-fast` in production. For recovery, use `dbt run --select result:error+` to rerun only failed models. Integrate with orchestration tools (Airflow/Prefect) for retry logic and alerting.

---

### Question 31: S3 File Organization Strategy

**✅ Answer**: Use hierarchical partitioning: `s3://bucket/app/table/year=YYYY/month=MM/day=DD/`. File naming: `{table}_{timestamp}_{seq}.parquet`. Target 100-250MB per file. Benefits: Snowflake external tables can prune partitions, reducing scan costs by 90%+. Use Parquet with Snappy compression. Implement S3 lifecycle policies for archival to Glacier.

---

### Question 32: Resource Monitor Setup

**✅ Answer**:

```sql
-- Account-level guard
CREATE RESOURCE MONITOR monthly_budget
    WITH CREDIT_QUOTA = 10000 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 50 PERCENT DO NOTIFY
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND
        ON 110 PERCENT DO SUSPEND_IMMEDIATE;
ALTER ACCOUNT SET RESOURCE_MONITOR = monthly_budget;

-- Warehouse-level guard (prevent runaway queries)
CREATE RESOURCE MONITOR etl_daily_limit
    WITH CREDIT_QUOTA = 200 FREQUENCY = DAILY
    TRIGGERS ON 100 PERCENT DO SUSPEND;
ALTER WAREHOUSE etl_wh SET RESOURCE_MONITOR = etl_daily_limit;

-- Also set query timeout
ALTER WAREHOUSE etl_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 7200;  -- 2 hour max
```

---

### Question 33: SCD Type 2 Implementation

**✅ Answer**: Two-step process: (1) UPDATE existing current records to set `effective_to = CURRENT_DATE - 1, is_current = FALSE` where attributes changed, (2) INSERT new version with `effective_from = CURRENT_DATE, is_current = TRUE`. Use dbt snapshots for automatic implementation. Always maintain surrogate key separate from natural key.

---

### Question 34: Python Batch Processing

**✅ Answer**: Use S3 paginator for lazy file listing, process in configurable batches (100 files), use `ThreadPoolExecutor` for parallel I/O, implement checkpointing (track processed files in JSON/DB), use `pandas.read_csv(chunksize=50000)` for large files, and load via Snowflake COPY (stage + COPY) instead of row-by-row INSERT for 10-100x better performance.

---

### Question 35: Multi-Cluster Warehouse Configuration

**✅ Answer**:

```sql
ALTER WAREHOUSE analytics_wh SET
    MIN_CLUSTER_COUNT = 1       -- Cost optimization during off-peak
    MAX_CLUSTER_COUNT = 4       -- Handle 25 concurrent users
    SCALING_POLICY = 'STANDARD' -- Scale immediately when queries queue
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE;
```

STANDARD = prioritizes speed (scales immediately), ECONOMY = prioritizes cost (waits ~6 min). For 25 users with 20-30 concurrent queries, MAX=4 with STANDARD provides best user experience. Monitor `WAREHOUSE_LOAD_HISTORY` to tune cluster count.

---

## Advanced Questions & Answers

---

### Question 36: Zero-Copy Cloning Architecture

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Database Architecture and Cost Optimization

**Scenario**:  
Migrating 10TB Oracle DW to Snowflake. Need DEV and UAT environments. Oracle copies cost \$50K+/month.

**Question**:  
Explain zero-copy cloning architecture and how to leverage it for multi-environment setup.

---

**✅ Answer**:

**How Zero-Copy Cloning Works Internally**:

Snowflake stores all data in immutable **micro-partitions** (50-500MB compressed columnar files). When you clone a table/database/schema, Snowflake creates new **metadata pointers** referencing the same micro-partitions — no data is physically copied.

```
PRODUCTION TABLE (10TB = ~25,000 micro-partitions)
├── Micro-partition 1 ────────┐
├── Micro-partition 2 ────┐   │
├── ...                   │   │
                          │   │
CLONE TABLE (0 bytes)     │   │
├── Pointer → ────────────┘   │
├── Pointer → ────────────────┘
└── ...
```

**Copy-on-Write**: When you modify data in the clone, only the affected micro-partitions are written as new files. Unchanged partitions remain shared.

**Multi-Environment Implementation**:

```sql
-- Clone entire production database for UAT (takes seconds, not hours)
CREATE DATABASE UAT_DB CLONE PROD_DB;

-- Clone for development
CREATE DATABASE DEV_DB CLONE PROD_DB;

-- Clone specific schema for a developer's sandbox
CREATE SCHEMA DEV_DB.john_sandbox CLONE PROD_DB.ANALYTICS;

-- Clone table at a specific point in time (Time Travel + Clone)
CREATE TABLE DEV_DB.PUBLIC.orders_test
    CLONE PROD_DB.PUBLIC.orders
    AT(TIMESTAMP => '2024-01-15 00:00:00'::TIMESTAMP);

-- Refresh UAT weekly (drop and re-clone)
DROP DATABASE IF EXISTS UAT_DB;
CREATE DATABASE UAT_DB CLONE PROD_DB;
```

**Storage Cost Analysis**:

| Environment | Oracle (Full Copy) | Snowflake (Clone) |
|------------|-------------------|-------------------|
| Production | 10TB = \$23,000/mo | 10TB = \$23,000/mo |
| UAT | 10TB = \$23,000/mo | ~200GB changes = \$460/mo |
| DEV | 10TB = \$23,000/mo | ~100GB changes = \$230/mo |
| **Total** | **\$69,000/mo** | **\$23,690/mo** |
| **Savings** | — | **\$45,310/mo (66%)** |

**Limitations**:
- External tables and stages cannot be cloned
- Privileges/grants are NOT copied — must re-apply via scripts
- Cloned objects share data but are fully independent (changes don't sync back)
- Internal stage data is NOT cloned

**Best Practice — Instant Rollback Before Risky Changes**:

```sql
CREATE TABLE orders_backup CLONE orders;        -- Instant backup, 0 bytes
ALTER TABLE orders ADD COLUMN new_col VARCHAR;   -- Risky change
-- If something goes wrong:
ALTER TABLE orders SWAP WITH orders_backup;      -- Instant rollback
```

---

### Question 37: Large-Scale Data Migration Strategy

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Enterprise Migration Planning

**Scenario**:  
500TB Oracle migration, 2,000 tables, 6-month timeline, 2-month parallel run.

**Question**:  
Design comprehensive migration strategy.

---

**✅ Answer**:

**Phase 1: Assessment & Planning (Weeks 1-4)**

- Inventory all 2,000 tables: sizes, row counts, growth rates, dependencies
- Prioritize: Critical (200 tables) → High (500) → Medium (800) → Low (500)
- Data type mapping document:

| Oracle | Snowflake |
|--------|-----------|
| NUMBER(p,s) | NUMBER(p,s) |
| VARCHAR2(n) | VARCHAR(n) |
| CLOB | VARCHAR(16MB) or VARIANT |
| DATE | TIMESTAMP_NTZ (Oracle DATE includes time!) |
| RAW | BINARY |
| XMLTYPE | VARIANT |

- Network plan: 500TB at 1Gbps = ~46 days continuous transfer; plan for 60 days with overhead

**Phase 2: Proof of Concept (Weeks 5-7)**

Migrate 10 representative tables (mix of sizes and complexity). Benchmark top 20 business queries comparing Oracle vs. Snowflake performance. Document SQL syntax differences (ROWNUM → ROW_NUMBER(), NVL → COALESCE, SYSDATE → CURRENT_TIMESTAMP()).

**Phase 3: Historical Load (Weeks 8-15)**

```
Extract Strategy:
  Oracle → Data Pump Export → Convert to Parquet → Upload to S3 → COPY INTO Snowflake

S3 Structure:
  s3://migration/critical/orders/year=2019/orders_2019_001.parquet
  s3://migration/critical/orders/year=2020/orders_2020_001.parquet
```

```sql
COPY INTO prod.orders
FROM @s3_migration_stage/critical/orders/
FILE_FORMAT = (TYPE=PARQUET)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

Process parallel extractions: 5-10 tables simultaneously, prioritizing critical tables first.

**Phase 4: CDC Setup (Weeks 16-19)**

Implement Oracle GoldenGate or AWS DMS for ongoing change capture. Changes stream to S3, then Snowpipe or scheduled COPY loads into Snowflake staging tables. MERGE statements apply changes to production tables.

**Phase 5: Parallel Run (Weeks 20-23)**

```sql
-- Daily reconciliation for each critical table
SELECT
    'ORDERS' AS table_name,
    (SELECT COUNT(*) FROM oracle_link.orders) AS oracle_count,
    (SELECT COUNT(*) FROM snowflake.orders) AS sf_count,
    ABS(oracle_count - sf_count) AS row_difference,
    (SELECT SUM(amount) FROM oracle_link.orders) AS oracle_sum,
    (SELECT SUM(amount) FROM snowflake.orders) AS sf_sum,
    ABS(oracle_sum - sf_sum) AS amount_difference;
```

Shift read traffic gradually: Week 1 = 10% Snowflake, Week 4 = 50%, Week 6 = 90%.

**Phase 6: Cutover (Week 24)**

Friday 8PM: Stop Oracle writes → Final CDC sync (zero lag) → Final reconciliation → Switch connection strings → Smoke test dashboards → Monitor 48 hours → Keep Oracle read-only for 2 weeks as safety net.

**Key Risks & Mitigations**:
- **Data loss**: Checksums + row counts + aggregate validation on every table
- **Performance regression**: Pre-benchmark top 50 queries; optimize with clustering keys
- **Rollback**: Oracle stays read-only for 2 weeks; connection revert possible in 1 hour

---

### Question 38: Query Optimization for Extreme Scale

**Difficulty**: Advanced | **Primary Skill**: SQL | **Topic**: Performance Engineering

**Scenario**:  
50B row fact table (20TB). Monthly report: 45 min on 4X-Large, 300 credits. Target: <10 min, <100 credits.

**Question**:  
Systematic optimization approach.

---

**✅ Answer**:

**Step 1 — Diagnose via Query Profile**:
- Scanning 45,000 of 50,000 partitions (90% — terrible pruning)
- 200GB disk spillage
- 8 dimension joins before any aggregation

**Step 2 — Add Clustering Key (30-50% improvement)**:

```sql
ALTER TABLE fact_sales CLUSTER BY (date_key);
-- Monthly filter now scans ~2,000 partitions (4%) instead of 45,000 (90%)
```

**Step 3 — Build Pre-Aggregation Summary Table (The game-changer)**:

```sql
-- One-time build: 50B rows → ~200M daily summary rows (250x reduction)
CREATE TABLE fact_sales_daily AS
SELECT
    date_key, product_id, region_id, channel_id,
    SUM(amount) AS total_amount,
    SUM(quantity) AS total_quantity,
    COUNT(*) AS txn_count,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM fact_sales
GROUP BY 1, 2, 3, 4;

-- Daily incremental maintenance
INSERT INTO fact_sales_daily
SELECT date_key, product_id, region_id, channel_id,
       SUM(amount), SUM(quantity), COUNT(*), COUNT(DISTINCT customer_id)
FROM fact_sales
WHERE date_key = CURRENT_DATE - 1
GROUP BY 1, 2, 3, 4;
```

**Step 4 — Rewrite Report Against Summary**:

```sql
SELECT
    p.category, r.region_name,
    SUM(s.total_amount) AS revenue,
    SUM(s.txn_count) AS transactions,
    RANK() OVER (PARTITION BY r.region_name ORDER BY SUM(s.total_amount) DESC) AS rank
FROM fact_sales_daily s
JOIN dim_product p ON s.product_id = p.product_id
JOIN dim_region r ON s.region_id = r.region_id
WHERE s.date_key BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY 1, 2;
```

**Results**:

| Metric | Before | After Clustering | After Summary Table | Target |
|--------|--------|-----------------|-------------------|--------|
| Rows processed | 50B | 50B | 200M | — |
| Partitions scanned | 45,000 | 2,000 | 200 | — |
| Runtime | 45 min | 25 min | **5 min** | <10 min ✅ |
| Warehouse | 4X-Large | 4X-Large | **X-Large** | — |
| Credits | 300 | 200 | **40** | <100 ✅ |

---

### Question 39: dbt Advanced Dependency Management

**Difficulty**: Advanced | **Primary Skill**: dbt | **Topic**: Complex DAG Architecture

**Scenario**:  
300+ models, complex cross-layer dependencies, models running before dependencies ready.

**Question**:  
How to architect reliable dependency management?

---

**✅ Answer**:

**Layered Project Structure**:

```
models/
├── staging/           # 1:1 with sources, views only
│   ├── stg_orders.sql           → {{ source('raw', 'orders') }}
│   └── stg_customers.sql       → {{ source('raw', 'customers') }}
├── intermediate/      # Business logic, reusable components
│   ├── int_orders_enriched.sql  → {{ ref('stg_orders') }} + {{ ref('stg_customers') }}
│   └── int_customer_ltv.sql     → {{ ref('stg_orders') }}
├── marts/             # Business entities, tables
│   ├── finance/
│   │   ├── fct_revenue.sql      → {{ ref('int_orders_enriched') }}
│   │   └── dim_customers.sql    → {{ ref('int_customer_ltv') }}
│   └── marketing/
│       └── fct_campaigns.sql
└── reporting/         # BI-ready, views
    └── rpt_dashboard.sql        → {{ ref('fct_revenue') }} + {{ ref('dim_customers') }}
```

**Dependency Rules**:
1. Staging → Only references `source()`
2. Intermediate → References staging or other intermediate
3. Marts → References intermediate or staging (never raw sources)
4. Reporting → References marts only
5. **No backward references** (mart ↛ staging)

**Orchestration with Selectors**:

```bash
dbt run --select tag:staging           # Phase 1
dbt run --select tag:intermediate      # Phase 2
dbt run --select tag:marts             # Phase 3
dbt run --select tag:reporting         # Phase 4

# Run specific chain
dbt run --select stg_orders+           # This model + ALL downstream
dbt run --select +fct_revenue          # ALL upstream + this model
```

**Handling Incremental Inter-Dependencies**:

When `fct_revenue` (incremental) depends on `dim_customers` (also incremental), use a 7-day lookback window in `fct_revenue` to cover potential gaps from late dim_customers updates:

```sql
{% if is_incremental() %}
    WHERE o.order_date >= (SELECT DATEADD('day', -7, MAX(order_date)) FROM {{ this }})
{% endif %}
```

**Airflow Integration for Complex Scheduling**:

```python
staging = BashOperator(task_id='dbt_staging', bash_command='dbt run --select tag:staging')
marts = BashOperator(task_id='dbt_marts', bash_command='dbt run --select tag:marts')
tests = BashOperator(task_id='dbt_test', bash_command='dbt test')

staging >> marts >> tests   # Explicit ordering
```

---

### Question 40: Multi-Cloud Data Integration

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Cross-Cloud Architecture

**Scenario**:  
Integrate BigQuery (5TB) and GCS (10TB) from GCP into AWS-based Snowflake. Daily syncs.

**Question**:  
Design architecture with cost/performance trade-offs.

---

**✅ Answer**:

**Recommended Architecture**:

```
BigQuery → Scheduled Export to GCS (Parquet) → Snowflake External Stage on GCS → Snowpipe → Raw Tables → dbt → Analytics
```

**Implementation**:

```sql
-- 1. Storage integration for GCS
CREATE STORAGE INTEGRATION gcs_integration
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = GCS
    ENABLED = TRUE
    STORAGE_ALLOWED_LOCATIONS = ('gcs://partner-data/exports/');

-- 2. Stage
CREATE STAGE gcs_partner_stage
    STORAGE_INTEGRATION = gcs_integration
    URL = 'gcs://partner-data/exports/'
    FILE_FORMAT = (TYPE=PARQUET);

-- 3. Auto-ingest pipe
CREATE PIPE gcs_daily_pipe AUTO_INGEST = TRUE
AS COPY INTO raw.partner_data FROM @gcs_partner_stage;
```

**Monthly Cost Estimate** (15TB transfer):

| Component | Cost |
|-----------|------|
| GCP egress (15TB × \$0.12/GB) | ~\$1,800 |
| Snowflake compute (COPY + dbt) | ~\$1,500 |
| Snowflake storage (~6TB compressed) | ~\$240 |
| **Total** | **~\$3,540/month** |

**Strategic Alternative**: If egress costs grow significantly, consider a Snowflake account on GCP with cross-cloud replication to AWS. Replication is typically cheaper than continuous cross-cloud egress for large volumes.

**Optimization**: Use Parquet with Snappy compression (3-5x compression ratio), ensure files are 100-250MB each, schedule exports during GCP off-peak hours.

---

### Question 41: Advanced Security Implementation

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Security and Governance (HIPAA)

**Scenario**:  
Healthcare company: row-level security, column masking, audit logging, de-identified data sharing for research.

**Question**:  
Design complete security architecture.

---

**✅ Answer**:

**1. Row Access Policy (Users See Only Assigned Patients)**:

```sql
CREATE OR REPLACE ROW ACCESS POLICY patient_row_policy
    AS (patient_id VARCHAR) RETURNS BOOLEAN ->
    CURRENT_ROLE() IN ('HEALTHCARE_ADMIN')
    OR EXISTS (
        SELECT 1 FROM security.user_patient_access
        WHERE user_name = CURRENT_USER()
          AND patient_id = patient_row_policy.patient_id
    );

ALTER TABLE patient_records ADD ROW ACCESS POLICY patient_row_policy ON (patient_id);
```

**2. Dynamic Data Masking (SSN, DOB)**:

```sql
CREATE MASKING POLICY mask_ssn AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('HEALTHCARE_ADMIN') THEN val
        WHEN CURRENT_ROLE() = 'HEALTHCARE_PROVIDER' THEN 'XXX-XX-' || RIGHT(val, 4)
        ELSE '***-**-****'
    END;

CREATE MASKING POLICY mask_dob AS (val DATE) RETURNS DATE ->
    CASE
        WHEN CURRENT_ROLE() IN ('HEALTHCARE_ADMIN', 'HEALTHCARE_PROVIDER') THEN val
        WHEN CURRENT_ROLE() = 'RESEARCHER' THEN DATE_FROM_PARTS(YEAR(val), 1, 1)
        ELSE NULL
    END;

ALTER TABLE patient_records MODIFY COLUMN ssn SET MASKING POLICY mask_ssn;
ALTER TABLE patient_records MODIFY COLUMN dob SET MASKING POLICY mask_dob;
```

**3. Audit Logging via ACCESS_HISTORY**:

```sql
SELECT query_id, user_name, role_name, start_time,
       direct_objects_accessed, base_objects_accessed
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE ARRAY_CONTAINS('PATIENT_RECORDS'::VARIANT, 
    TRANSFORM(base_objects_accessed, x -> x:objectName))
AND start_time >= DATEADD('day', -7, CURRENT_DATE)
ORDER BY start_time DESC;
```

**4. Secure Data Sharing for Research (De-Identified)**:

```sql
CREATE SECURE VIEW research.aggregated_patients AS
SELECT
    FLOOR(DATEDIFF('year', dob, CURRENT_DATE) / 10) * 10 AS age_group,
    state, diagnosis_code,
    COUNT(*) AS patient_count,
    AVG(treatment_days) AS avg_treatment_days
FROM patient_records
GROUP BY 1, 2, 3
HAVING COUNT(*) >= 10;    -- K-anonymity: suppress small groups

CREATE SHARE research_share;
GRANT USAGE ON DATABASE research_db TO SHARE research_share;
GRANT USAGE ON SCHEMA research_db.research TO SHARE research_share;
GRANT SELECT ON VIEW research.aggregated_patients TO SHARE research_share;
ALTER SHARE research_share ADD ACCOUNTS = research_partner_acct;
```

**5. RBAC Hierarchy**:

```sql
CREATE ROLE HEALTHCARE_ADMIN;      -- Full access
CREATE ROLE HEALTHCARE_PROVIDER;   -- Patient data with partial SSN masking
CREATE ROLE RESEARCHER;            -- Aggregated data only
CREATE ROLE BILLING_ANALYST;       -- Financial data, masked PII

GRANT ROLE HEALTHCARE_PROVIDER TO ROLE HEALTHCARE_ADMIN;
GRANT ROLE RESEARCHER TO ROLE HEALTHCARE_ADMIN;
```

**Performance Impact**: Row access policies add ~10-15% query overhead. Acceptable for HIPAA compliance. Optimize with clustering on patient_id if queries are patient-specific.

---

### Question 42: Real-Time Streaming Data Pipeline

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Streaming Architecture

**Scenario**:  
50K events/second, need <1 min latency. Current 15-min batch is insufficient.

**Question**:  
Design real-time streaming architecture.

---

**✅ Answer**:

**Architecture**:

```
App Events → Kafka (8-16 partitions)
    → Snowflake Kafka Connector (Snowpipe Streaming mode)
    → raw.events (VARIANT column)
    → Stream → Transform Task (1 min) → analytics.events_processed
    → Stream → Alert Task (1 min)    → alerts.fraud_alerts → Email/Slack
```

**Implementation**:

```sql
-- Raw table for ingestion
CREATE TABLE raw.events (
    raw_data    VARIANT,
    ingest_ts   TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Stream captures new arrivals
CREATE STREAM events_stream ON TABLE raw.events;

-- Transformation task: runs every minute, only when new data
CREATE TASK transform_events
    WAREHOUSE = STREAMING_WH
    SCHEDULE = '1 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('events_stream')
AS
INSERT INTO analytics.events_processed
SELECT
    raw_data:event_id::STRING       AS event_id,
    raw_data:event_type::STRING     AS event_type,
    raw_data:user_id::STRING        AS user_id,
    raw_data:amount::FLOAT          AS amount,
    raw_data:timestamp::TIMESTAMP   AS event_ts,
    ingest_ts
FROM events_stream;

ALTER TASK transform_events RESUME;

-- Fraud detection task (child of transform)
CREATE TASK detect_fraud
    WAREHOUSE = ALERT_WH
    AFTER transform_events
AS
INSERT INTO alerts.fraud_alerts
SELECT *
FROM analytics.events_processed
WHERE event_ts > DATEADD('minute', -2, CURRENT_TIMESTAMP())
  AND (amount > 10000
    OR user_id IN (
        SELECT user_id FROM analytics.events_processed
        WHERE event_ts > DATEADD('hour', -1, CURRENT_TIMESTAMP())
        GROUP BY user_id HAVING COUNT(*) > 50
    ));

ALTER TASK detect_fraud RESUME;
```

**Throughput Math**: 50K events/sec × 1KB avg = ~50MB/sec = **~4.3TB/day**. Kafka with 8-16 partitions handles this; Snowpipe Streaming provides sub-minute latency without S3 staging files.

**Monitoring**:

```sql
-- Check task execution history
SELECT name, state, scheduled_time, completed_time,
       DATEDIFF('second', scheduled_time, completed_time) AS runtime_sec
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'TRANSFORM_EVENTS',
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -6, CURRENT_TIMESTAMP())
))
ORDER BY scheduled_time DESC;
```

---

### Question 43: Tag-Based Governance at Scale

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Data Governance

**Scenario**:  
500+ tables, PII scattered across many columns. Manual masking unsustainable.

**Question**:  
Implement automatic tag-based classification and masking.

---

**✅ Answer**:

```sql
-- 1. Create classification tags
CREATE TAG pii_type ALLOWED_VALUES 'NAME', 'EMAIL', 'SSN', 'PHONE', 'ADDRESS', 'DOB';

-- 2. Tag columns across tables
ALTER TABLE customers MODIFY COLUMN email SET TAG pii_type = 'EMAIL';
ALTER TABLE customers MODIFY COLUMN ssn SET TAG pii_type = 'SSN';
ALTER TABLE customers MODIFY COLUMN full_name SET TAG pii_type = 'NAME';
ALTER TABLE employees MODIFY COLUMN personal_email SET TAG pii_type = 'EMAIL';
ALTER TABLE employees MODIFY COLUMN phone SET TAG pii_type = 'PHONE';
-- (Script this for bulk application across 500+ tables)

-- 3. Single universal masking policy using SYSTEM$GET_TAG_ON_CURRENT_COLUMN
CREATE MASKING POLICY universal_pii_mask AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'COMPLIANCE') THEN val
        WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('pii_type') = 'SSN'
            THEN 'XXX-XX-' || RIGHT(val, 4)
        WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('pii_type') = 'EMAIL'
            THEN REGEXP_REPLACE(val, '(.{2}).*@', '\\1***@')
        WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('pii_type') IN ('NAME', 'PHONE', 'ADDRESS')
            THEN '***REDACTED***'
        ELSE val
    END;

-- 4. Attach policy to the TAG (auto-applies to ALL tagged columns everywhere)
ALTER TAG pii_type SET MASKING POLICY universal_pii_mask;
-- Now: ANY column tagged with pii_type across ALL tables is automatically masked
```

**Auto-Classification**:

```sql
-- Use Snowflake's built-in classifier to detect PII
SELECT EXTRACT_SEMANTIC_CATEGORIES('MY_DB.MY_SCHEMA.CUSTOMERS');
-- Returns: {"EMAIL": {"semantic_category": "EMAIL", "privacy_category": "IDENTIFIER"}, ...}
-- Script the results to auto-apply tags
```

**Scale Impact**: 1 policy + 1 tag = governs hundreds of columns across hundreds of tables. Adding a new table with an email column only requires tagging that one column — the masking policy applies automatically.

---

### Question 44: Parquet File Optimization for Snowflake

**Difficulty**: Advanced | **Primary Skill**: AWS S3 | **Topic**: File Format Optimization

**Scenario**:  
Spark outputs 2GB Parquet files with GZIP. Loading 5TB/day takes 2 hours. Post-load queries slow.

**Question**:  
Optimize file generation and loading.

---

**✅ Answer**:

**Spark Configuration Changes**:

```python
df.sort("event_date", "user_id")          # Pre-sort for natural clustering in Snowflake
  .repartition(200)                        # 5TB / 200 = ~25GB raw → ~125MB compressed per file
  .write.mode("overwrite")
  .option("compression", "snappy")         # Snappy > GZIP (faster decompress, Snowflake-optimized)
  .option("parquet.block.size", 268435456) # 256MB row groups
  .parquet("s3://bucket/output/")
```

**Snowflake COPY Optimization**:

```sql
COPY INTO target_table
FROM @s3_stage/output/
FILE_FORMAT = (TYPE=PARQUET COMPRESSION=SNAPPY)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE
FORCE = FALSE              -- Skip already-loaded files
PURGE = TRUE;              -- Clean up after successful load
```

**Why These Changes Help**:

| Change | Before | After | Impact |
|--------|--------|-------|--------|
| File size | 2GB | ~125MB | Better parallelism, fewer retries |
| Compression | GZIP | SNAPPY | 40-60% faster decompression |
| Pre-sorting | Random | event_date, user_id | Natural clustering → better partition pruning |
| File count | ~25 files | ~200 files | More parallel COPY workers |

**Expected Improvement**:
- Load time: 2 hours → **30-40 minutes**
- Query performance: **30-50% faster** due to pre-sorted data aligning with micro-partitions

---

### Question 45: Python Snowpark for Complex Transformations

**Difficulty**: Advanced | **Primary Skill**: Python | **Topic**: Snowpark DataFrame API

**Scenario**:  
Migrate pandas feature engineering from EMR to Snowpark to eliminate data movement.

**Question**:  
How to implement? Key differences from pandas?

---

**✅ Answer**:

```python
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, sum as sum_, avg, lag, datediff, count
from snowflake.snowpark.window import Window

session = Session.builder.configs({
    "account": "xy12345",
    "user": "ml_user",
    "password": "...",
    "warehouse": "ML_WH",
    "database": "ANALYTICS",
    "schema": "FEATURES"
}).create()

# Read data — LAZY, no data moves to client
orders = session.table("ORDERS")
customers = session.table("CUSTOMERS")

# Feature engineering — runs IN Snowflake
window = Window.partition_by("customer_id").order_by("order_date")

features = (
    orders
    .join(customers, orders["customer_id"] == customers["customer_id"])
    .with_column("prev_order_date", lag("order_date", 1).over(window))
    .with_column("days_between", datediff("day", col("prev_order_date"), col("order_date")))
    .group_by("customer_id")
    .agg(
        sum_("amount").alias("total_spend"),
        avg("days_between").alias("avg_order_frequency"),
        count("order_id").alias("total_orders")
    )
)

# See the generated SQL
features.explain()

# Write results back to Snowflake — data never left!
features.write.mode("overwrite").save_as_table("CUSTOMER_FEATURES")
```

**Custom UDF for Complex Logic**:

```python
from snowflake.snowpark.functions import udf
from snowflake.snowpark.types import FloatType

@udf(name="rfm_score", is_permanent=True, stage_location="@udf_stage",
     input_types=[FloatType(), FloatType(), FloatType()], return_type=FloatType(),
     packages=["numpy"], replace=True)
def rfm_score(recency: float, frequency: float, monetary: float) -> float:
    import numpy as np
    return float(np.dot([0.3, 0.3, 0.4], [recency, frequency, monetary]))

# Use in DataFrame
features = features.with_column("rfm", rfm_score(col("recency"), col("frequency"), col("monetary")))
```

**Key Differences from pandas**:

| Aspect | pandas | Snowpark |
|--------|--------|----------|
| Execution | Immediate (eager) | Deferred (lazy) until .collect()/.save_as_table() |
| Where runs | Local machine RAM | Snowflake warehouse compute |
| Data movement | Entire dataset loaded locally | Data never leaves Snowflake |
| Scale | Limited by local RAM | TB-scale via warehouse scaling |
| Debugging | Easy (print, breakpoints) | Harder (.show(5), .explain()) |
| Function coverage | Very broad | Covers ~80% of common operations |

**Best Practices**:
- Prefer DataFrame operations over UDFs (SQL pushdown is much faster)
- Use vectorized UDFs (batch processing) over scalar UDFs
- Avoid `.collect()` — it pulls data to the client (defeats the purpose)
- Use `.explain()` to inspect generated SQL for optimization opportunities

---

### Question 46: Disaster Recovery and Business Continuity

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: High Availability

**Scenario**:  
RPO <1 hour, RTO <4 hours, cross-region redundancy.

**Question**:  
Design DR architecture.

---

**✅ Answer**:

```sql
-- PRIMARY ACCOUNT (us-east-1): Create failover group
CREATE FAILOVER GROUP analytics_dr
    OBJECT_TYPES = DATABASES, WAREHOUSES, USERS, ROLES, INTEGRATIONS
    ALLOWED_DATABASES = PROD_DB, ANALYTICS_DB, SHARED_DB
    ALLOWED_ACCOUNTS = org_name.dr_account_us_west_2
    REPLICATION_SCHEDULE = '10 MINUTE';   -- RPO: max 10 minutes ✅

-- DR ACCOUNT (us-west-2): Create as replica
CREATE FAILOVER GROUP analytics_dr
    AS REPLICA OF org_name.primary_account.analytics_dr;

-- FAILOVER PROCEDURE (when primary region goes down):
-- Step 1: Promote DR to primary (~15-30 min)
ALTER FAILOVER GROUP analytics_dr PRIMARY;

-- Step 2: Validate critical data
SELECT COUNT(*) FROM PROD_DB.PUBLIC.ORDERS;  -- Verify data is present
SELECT MAX(created_at) FROM PROD_DB.PUBLIC.ORDERS;  -- Check replication lag

-- Step 3: Update application connection strings (1-2 hours with DNS/config changes)
-- Step 4: Run smoke tests on critical dashboards
-- Total RTO: ~2-3 hours ✅
```

**Layers of Protection**:

| Layer | Coverage | RPO | Who Triggers |
|-------|----------|-----|-------------|
| Cross-region replication | Region failure | 10 min | DBA team |
| Time Travel (90 days) | Accidental data changes | 0 (point-in-time) | Data engineer |
| Fail-safe (7 days) | After Time Travel expires | N/A | Snowflake support only |
| S3 cross-region replication | Independent raw data backup | Minutes | Automatic |

**Cost**: ~\$2/TB/month for replication storage + compute for replication sync. For 10TB = ~\$20/month replication + standby warehouse costs when not active.

**Testing Cadence**: Quarterly full failover drills. Monthly replication lag checks. Automated monitoring: alert if replication lag > 30 minutes.

---

### Question 47: Advanced SQL — Recursive CTEs and Graph Queries

**Difficulty**: Advanced | **Primary Skill**: SQL | **Topic**: Recursive Queries

**Scenario**:  
Employee hierarchy: full management chain, depth, headcount per manager. Handle circular references for 50K+ employees.

**Question**:  
Write recursive CTE with cycle detection.

---

**✅ Answer**:

```sql
-- Full hierarchy with management chain and cycle detection
WITH RECURSIVE org_hierarchy AS (
    -- Anchor: CEO (no manager)
    SELECT
        employee_id,
        name,
        manager_id,
        department,
        0 AS depth,
        name::VARCHAR(10000) AS management_chain,
        employee_id::VARCHAR(10000) AS path_ids
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: walk down the tree
    SELECT
        e.employee_id,
        e.name,
        e.manager_id,
        e.department,
        h.depth + 1,
        h.management_chain || ' > ' || e.name,
        h.path_ids || ',' || e.employee_id::VARCHAR
    FROM employees e
    INNER JOIN org_hierarchy h ON e.manager_id = h.employee_id
    WHERE h.depth < 20                                          -- Safety limit
      AND NOT CONTAINS(h.path_ids, e.employee_id::VARCHAR)      -- Cycle detection
)

SELECT
    employee_id,
    name,
    department,
    depth,
    management_chain
FROM org_hierarchy
ORDER BY management_chain;
```

**Headcount Per Manager**:

```sql
-- Using the hierarchy above, count all descendants per manager
WITH RECURSIVE subordinates AS (
    SELECT employee_id, manager_id, employee_id AS root_manager_id
    FROM employees

    UNION ALL

    SELECT e.employee_id, e.manager_id, s.root_manager_id
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.employee_id
    WHERE s.root_manager_id != e.employee_id  -- Prevent counting self in loop
)
SELECT
    root_manager_id AS manager_id,
    m.name AS manager_name,
    COUNT(DISTINCT employee_id) - 1 AS total_direct_and_indirect_reports
FROM subordinates s
JOIN employees m ON s.root_manager_id = m.employee_id
GROUP BY root_manager_id, m.name
ORDER BY total_direct_and_indirect_reports DESC;
```

**Performance for 50K+ Employees**:
- Recursive CTEs are O(N × depth) — fine for one-off queries
- For dashboards: materialize as daily table (`CREATE TABLE org_hierarchy_cache AS SELECT ...`)
- Snowflake's max recursion depth: 100 iterations (sufficient for any real organization)

---

### Question 48: Enterprise Data Mesh Architecture

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Data Architecture Design

**Scenario**:  
8 business units, centralized team bottleneck. Need data mesh with governance.

**Question**:  
Implement in Snowflake.

---

**✅ Answer**:

```sql
-- 1. Per-domain databases with standard schema pattern
CREATE DATABASE MARKETING_DOMAIN;
CREATE SCHEMA MARKETING_DOMAIN.RAW;
CREATE SCHEMA MARKETING_DOMAIN.STAGING;
CREATE SCHEMA MARKETING_DOMAIN.MARTS;
CREATE SCHEMA MARKETING_DOMAIN.SHARED;     -- Published data products

-- Repeat for SALES_DOMAIN, FINANCE_DOMAIN, PRODUCT_DOMAIN, etc.

-- 2. Cross-domain sharing via internal data marketplace
CREATE SHARE marketing_campaigns_share;
GRANT USAGE ON DATABASE MARKETING_DOMAIN TO SHARE marketing_campaigns_share;
GRANT USAGE ON SCHEMA MARKETING_DOMAIN.SHARED TO SHARE marketing_campaigns_share;
GRANT SELECT ON VIEW MARKETING_DOMAIN.SHARED.campaign_performance
    TO SHARE marketing_campaigns_share;

-- 3. Central governance (tags + masking, applied globally)
CREATE DATABASE GOVERNANCE;
CREATE TAG GOVERNANCE.TAGS.data_domain
    ALLOWED_VALUES 'marketing', 'sales', 'finance', 'product', 'hr', 'ops', 'supply_chain', 'customer_success';
CREATE TAG GOVERNANCE.TAGS.data_classification
    ALLOWED_VALUES 'public', 'internal', 'confidential', 'restricted';

-- Apply classification centrally
ALTER TABLE MARKETING_DOMAIN.SHARED.campaign_performance
    SET TAG GOVERNANCE.TAGS.data_classification = 'internal';

-- 4. Domain-specific warehouses for cost chargeback
CREATE WAREHOUSE MARKETING_WH COMMENT = 'Marketing domain compute';
CREATE WAREHOUSE SALES_WH COMMENT = 'Sales domain compute';

CREATE RESOURCE MONITOR marketing_budget
    WITH CREDIT_QUOTA = 2000 FREQUENCY = MONTHLY
    TRIGGERS ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO SUSPEND;
ALTER WAREHOUSE MARKETING_WH SET RESOURCE_MONITOR = marketing_budget;

-- 5. Monthly chargeback query
SELECT
    warehouse_name,
    SPLIT_PART(warehouse_name, '_', 1) AS domain,
    SUM(credits_used) AS monthly_credits,
    ROUND(SUM(credits_used) * 3.00, 2) AS estimated_cost
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY 1, 2
ORDER BY monthly_credits DESC;

-- 6. Role hierarchy
-- ACCOUNTADMIN → GOVERNANCE_ADMIN (central team)
--     ├── MARKETING_ADMIN → MARKETING_ENGINEER, MARKETING_ANALYST
--     ├── SALES_ADMIN → SALES_ENGINEER, SALES_ANALYST
--     └── FINANCE_ADMIN → FINANCE_ENGINEER, FINANCE_ANALYST
```

**Data Product Contract** (enforced per domain):
- **SLA**: Data refreshed by 6 AM daily
- **Quality**: dbt tests pass (unique, not_null, freshness)
- **Schema**: Documented in dbt docs; versioned in Git
- **Access**: Published in SHARED schema only after testing

---

### Question 49: Python Advanced Error Recovery Pipeline

**Difficulty**: Advanced | **Primary Skill**: Python | **Topic**: Production-Grade Pipeline Engineering

**Scenario**:  
15 API sources with different auth/formats/rate limits. Need partial failure handling, retries, DLQ, checkpointing, observability.

**Question**:  
Design production-grade pipeline framework.

---

**✅ Answer**:

```python
import time, json, logging, boto3, snowflake.connector
from datetime import datetime, timedelta
from concurrent.futures import ThreadPoolExecutor, as_completed

logger = logging.getLogger(__name__)

# ---- PATTERN 1: Circuit Breaker ----
class CircuitBreaker:
    """Prevents repeated calls to a failing source."""
    def __init__(self, threshold=5, recovery_timeout=300):
        self.threshold = threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.state = "CLOSED"       # CLOSED=normal, OPEN=blocked, HALF_OPEN=testing
        self.last_failure_time = None

    def can_execute(self):
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.recovery_timeout):
                self.state = "HALF_OPEN"
                return True     # Allow one test request
            return False
        return True             # HALF_OPEN: try one

    def record_success(self):
        self.failure_count = 0
        self.state = "CLOSED"

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        if self.failure_count >= self.threshold:
            self.state = "OPEN"
            logger.warning(f"Circuit breaker OPEN after {self.failure_count} failures")


# ---- PATTERN 2: Retry with Exponential Backoff ----
def retry_with_backoff(func, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries + 1):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries:
                raise
            delay = base_delay * (2 ** attempt)   # 1s, 2s, 4s
            logger.warning(f"Attempt {attempt+1} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)


# ---- PATTERN 3: Dead Letter Queue ----
class DeadLetterQueue:
    """Store failed records in S3 for later reprocessing."""
    def __init__(self, bucket, prefix):
        self.s3 = boto3.client('s3')
        self.bucket = bucket
        self.prefix = prefix

    def send(self, source_name, records, error):
        key = f"{self.prefix}/{source_name}/{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        payload = {
            "source": source_name,
            "error": str(error),
            "timestamp": datetime.now().isoformat(),
            "record_count": len(records),
            "records": records
        }
        self.s3.put_object(Bucket=self.bucket, Key=key, Body=json.dumps(payload))
        logger.info(f"DLQ: {len(records)} records from {source_name} → {key}")


# ---- PATTERN 4: Checkpoint Manager ----
class CheckpointManager:
    """Track last processed ID per source for restart capability."""
    def __init__(self, sf_conn):
        self.conn = sf_conn

    def get(self, source):
        row = self.conn.cursor().execute(
            "SELECT last_processed_id FROM pipeline.checkpoints WHERE source_name = %s",
            (source,)
        ).fetchone()
        return row[0] if row else None

    def save(self, source, last_id):
        self.conn.cursor().execute("""
            MERGE INTO pipeline.checkpoints t
            USING (SELECT %s AS src, %s AS lid) s ON t.source_name = s.src
            WHEN MATCHED THEN UPDATE SET last_processed_id = s.lid, updated_at = CURRENT_TIMESTAMP()
            WHEN NOT MATCHED THEN INSERT VALUES (s.src, s.lid, CURRENT_TIMESTAMP())
        """, (source, last_id))


# ---- MAIN ORCHESTRATOR ----
class PipelineOrchestrator:
    def __init__(self, sources, sf_conn):
        self.sources = sources
        self.breakers = {s['name']: CircuitBreaker(s.get('threshold', 5)) for s in sources}
        self.dlq = DeadLetterQueue("pipeline-dlq-bucket", "failed")
        self.checkpoint = CheckpointManager(sf_conn)
        self.results = {}

    def run(self):
        for source in self.sources:
            name = source['name']
            cb = self.breakers[name]

            if not cb.can_execute():
                logger.warning(f"SKIPPING {name}: circuit breaker OPEN")
                self.results[name] = {"status": "skipped", "reason": "circuit_breaker"}
                continue

            try:
                last_id = self.checkpoint.get(name)
                data = retry_with_backoff(
                    lambda: self.extract(source, last_id),
                    max_retries=source.get('max_retries', 3)
                )
                loaded, failed = self.transform_and_load(source, data)

                if failed:
                    self.dlq.send(name, failed, "transform_or_load_failure")

                if data:
                    self.checkpoint.save(name, data[-1].get('id', ''))

                cb.record_success()
                self.results[name] = {"status": "success", "loaded": loaded, "failed": len(failed)}

            except Exception as e:
                cb.record_failure()
                logger.error(f"FAILED {name}: {e}")
                self.results[name] = {"status": "failed", "error": str(e)}

        self.log_results()
        return self.results

    def extract(self, source, last_id):
        """Extract data from API source (implementation varies per source type)."""
        # ... source-specific extraction logic
        pass

    def transform_and_load(self, source, data):
        """Transform and load to Snowflake. Return (loaded_count, failed_records)."""
        # ... transformation and MERGE/COPY logic
        pass

    def log_results(self):
        """Log run results to Snowflake audit table and send alerts."""
        success = sum(1 for r in self.results.values() if r['status'] == 'success')
        failed = sum(1 for r in self.results.values() if r['status'] == 'failed')
        logger.info(f"Pipeline complete: {success} succeeded, {failed} failed")
        # Send Slack/PagerDuty alert if any failures
```

**Key Patterns Summary**:
- **Circuit Breaker**: Stops hammering a failing source after 5 consecutive failures; retries after 5 min cooldown
- **Retry with Backoff**: 1s → 2s → 4s delays between retries for transient errors
- **Dead Letter Queue**: Preserves failed records in S3 for manual reprocessing or automated replay
- **Checkpointing**: Saves last processed ID per source; enables restart from failure point
- **Idempotency**: Use MERGE in Snowflake (not INSERT) so reprocessing doesn't create duplicates

---

### Question 50: Strategic Cost Optimization at Enterprise Scale

**Difficulty**: Advanced | **Primary Skill**: Snowflake | **Topic**: Enterprise Cost Management

**Scenario**:  
\$500K/month Snowflake spend across 200 warehouses. CFO wants 30% reduction (\$150K/month).

**Question**:  
Design comprehensive cost optimization strategy.

---

**✅ Answer**:

**Phase 1: Analysis (Weeks 1-2)**

```sql
-- Top 10 cost-consuming warehouses
SELECT warehouse_name, SUM(credits_used) AS credits,
       ROUND(SUM(credits_used) * 3.00, 2) AS cost,
       ROUND(SUM(credits_used) * 100.0 /
           (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
            WHERE start_time >= DATEADD('month', -1, CURRENT_DATE)), 1) AS pct
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('month', -1, CURRENT_DATE)
GROUP BY warehouse_name ORDER BY credits DESC LIMIT 10;

-- Zombie warehouses (consuming credits but barely used)
WITH wh_credits AS (
    SELECT warehouse_name, SUM(credits_used) AS credits
    FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
    WHERE start_time >= DATEADD('month', -1, CURRENT_DATE) GROUP BY 1
),
wh_queries AS (
    SELECT warehouse_name, COUNT(*) AS query_count
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE start_time >= DATEADD('month', -1, CURRENT_DATE) GROUP BY 1
)
SELECT c.warehouse_name, c.credits, COALESCE(q.query_count, 0) AS queries,
       CASE WHEN COALESCE(q.query_count, 0) < 10 THEN 'ZOMBIE' ELSE 'ACTIVE' END AS status
FROM wh_credits c LEFT JOIN wh_queries q ON c.warehouse_name = q.warehouse_name
ORDER BY c.credits DESC;

-- Top 20 most expensive queries
SELECT query_id, user_name, warehouse_name, warehouse_size,
       ROUND(execution_time / 60000, 1) AS runtime_min,
       ROUND(bytes_scanned / POWER(1024, 4), 2) AS tb_scanned,
       ROUND(bytes_spilled_to_local_storage / POWER(1024, 3), 2) AS gb_spilled
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('month', -1, CURRENT_DATE)
  AND execution_time > 60000
ORDER BY execution_time DESC LIMIT 20;
```

**Phase 2: Quick Wins (Weeks 2-4) — Target: \$60-80K savings**

| Action | Savings | Effort |
|--------|---------|--------|
| Fix auto-suspend (60s ETL, 300s analytics, 120s ad-hoc) | \$20-30K | Low |
| Eliminate 15-20 zombie warehouses | \$10-15K | Low |
| Right-size 10 over-provisioned warehouses | \$15-20K | Medium |
| Storage cleanup (drop unused tables, reduce Time Travel) | \$10-15K | Low |

```sql
-- Example quick fixes
ALTER WAREHOUSE dev_wh_01 SET AUTO_SUSPEND = 120;
ALTER WAREHOUSE etl_staging SET WAREHOUSE_SIZE = 'LARGE';  -- Was X-Large
ALTER TABLE archive_data SET DATA_RETENTION_TIME_IN_DAYS = 1;  -- Was 90
DROP TABLE IF EXISTS abandoned_test_table;
```

**Phase 3: Structural Changes (Weeks 4-8) — Target: \$40-60K savings**

| Action | Savings | Effort |
|--------|---------|--------|
| Consolidate 200 → ~50 warehouses (multi-cluster) | \$15-20K | High |
| Optimize top 20 expensive queries | \$15-20K | Medium |
| Convert full-refresh models to incremental | \$10-15K | Medium |
| Implement caching strategy for dashboards | \$5-10K | Low |

**Phase 4: Governance (Weeks 8-12) — Sustain Savings**

```sql
-- Mandatory resource monitors per department
CREATE RESOURCE MONITOR eng_monthly WITH CREDIT_QUOTA = 3000 FREQUENCY = MONTHLY
    TRIGGERS ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO SUSPEND;

-- Query tagging for cost attribution
ALTER SESSION SET QUERY_TAG = 'team:engineering;project:data_pipeline';
```

- Monthly cost review meetings with department heads
- No new warehouses without governance approval
- Automated alerts for cost anomalies (>20% week-over-week increase)

**Expected Outcome**:

| Category | Current | Optimized | Savings |
|----------|---------|-----------|---------|
| ETL (40%) | \$200K | \$130K | \$70K |
| BI Dashboards (25%) | \$125K | \$85K | \$40K |
| Ad-hoc Analytics (20%) | \$100K | \$70K | \$30K |
| Data Science (15%) | \$75K | \$65K | \$10K |
| **Total** | **\$500K** | **\$350K** | **\$150K (30%) ✅** |

---

## Quick Reference Summary

| # | Topic | Difficulty | Skill | Key Concept |
|---|-------|-----------|-------|-------------|
| 1 | S3 Data Load | Fund. | AWS S3 | COPY INTO, VALIDATION_MODE, ON_ERROR |
| 2 | dbt Models | Fund. | dbt | Materializations, ref(), schema tests |
| 3 | Warehouse Sizing | Fund. | Snowflake | Separate warehouses, auto-suspend |
| 4 | Window Functions | Fund. | SQL | PARTITION BY, ROWS BETWEEN |
| 5 | Python Connector | Fund. | Python | snowflake-connector, env variables |
| 6 | Time Travel | Fund. | Snowflake | AT/BEFORE clause, recovery workflow |
| 7 | dbt Sources | Fund. | dbt | source(), freshness checks |
| 8 | Query Performance | Fund. | SQL | Query Profile, clustering, filter pushdown |
| 9 | JSON Loading | Fund. | AWS S3 | VARIANT, STRIP_OUTER_ARRAY, nested access |
| 10 | RBAC | Fund. | Snowflake | Roles, grants, FUTURE GRANTS |
| 11 | Incremental Models | Inter. | dbt | is_incremental(), lookback window, merge |
| 12 | Clustering Keys | Inter. | Snowflake | SYSTEM\$CLUSTERING_INFORMATION, reclustering |
| 13 | Cohort Analysis | Inter. | SQL | DATE_TRUNC, DATEDIFF, retention pivoting |
| 14 | Snowpipe | Inter. | Snowflake | AUTO_INGEST, S3 event notifications |
| 15 | Data Quality | Inter. | dbt | Custom tests, store_failures, severity |
| 16 | ETL Orchestration | Inter. | Python | Modular steps, retry, error handling |
| 17 | Spillage | Inter. | Snowflake | Memory limits, pre-aggregation, temp tables |
| 18 | CDC | Inter. | Snowflake | MERGE, watermark tracking, timestamp-based |
| 19 | Cost Analysis | Inter. | Snowflake | WAREHOUSE_METERING_HISTORY, idle detection |
| 20 | Snapshots | Inter. | dbt | SCD Type 2, timestamp strategy |
| 21 | Multi-Environment | Inter. | dbt | profiles.yml, generate_schema_name |
| 22 | LATERAL FLATTEN | Inter. | SQL | VARIANT arrays, OUTER => TRUE |
| 23 | Streams & Tasks | Inter. | Snowflake | STREAM_HAS_DATA, task chaining |
| 24 | Join Optimization | Inter. | SQL | CTEs, pre-aggregation, filter early |
| 25 | External Tables | Inter. | AWS S3 | Native vs external, partition pruning |
| 26 | dbt Macros | Inter. | dbt | Jinja, GENERATOR, date spine |
| 27 | Data Validation | Inter. | Python | Validation functions, Great Expectations |
| 28 | Data Sharing | Inter. | Snowflake | SECURE VIEW, CREATE SHARE |
| 29 | Result Caching | Inter. | Snowflake | Exact match, 24hr cache, materialized views |
| 30 | Error Handling | Inter. | dbt | Hooks, store_failures, fail-fast |
| 31 | S3 Organization | Inter. | AWS S3 | Partitioning, file sizing, naming |
| 32 | Resource Monitors | Inter. | Snowflake | Credit quotas, SUSPEND thresholds |
| 33 | SCD Type 2 | Inter. | SQL | UPDATE + INSERT, effective dates |
| 34 | Batch Processing | Inter. | Python | ThreadPoolExecutor, checkpointing |
| 35 | Multi-Cluster WH | Inter. | Snowflake | STANDARD vs ECONOMY scaling |
| 36 | Zero-Copy Cloning | Adv. | Snowflake | Metadata pointers, copy-on-write |
| 37 | Migration Strategy | Adv. | Snowflake | 6-phase approach, CDC, reconciliation |
| 38 | Extreme Scale | Adv. | SQL | Pre-aggregation summary tables |
| 39 | dbt DAG Management | Adv. | dbt | Layered structure, tags, selectors |
| 40 | Multi-Cloud | Adv. | Snowflake | GCS integration, egress costs |
| 41 | HIPAA Security | Adv. | Snowflake | Row access, masking, ACCESS_HISTORY |
| 42 | Streaming | Adv. | Snowflake | Snowpipe Streaming, Kafka, sub-minute |
| 43 | Tag Governance | Adv. | Snowflake | Tag-based masking, auto-classification |
| 44 | Parquet Optimization | Adv. | AWS S3 | Snappy, 125MB files, pre-sorting |
| 45 | Snowpark | Adv. | Python | Lazy evaluation, SQL pushdown, UDFs |
| 46 | Disaster Recovery | Adv. | Snowflake | Failover groups, 10-min replication |
| 47 | Recursive CTEs | Adv. | SQL | Hierarchy traversal, cycle detection |
| 48 | Data Mesh | Adv. | Snowflake | Domain databases, central governance |
| 49 | Error Recovery | Adv. | Python | Circuit breaker, DLQ, checkpointing |
| 50 | Cost Optimization | Adv. | Snowflake | 4-phase approach, \$150K savings |
