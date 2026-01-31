# 9. Best Practices

## Overview

Production Snowflake deployments require strategic schema design, cost optimization, and operational monitoring. Best practices span logical data modeling (star/snowflake schemas), warehouse sizing and auto-scaling, Time Travel/Fail-Safe configuration, and continuous observability through query profiling and resource monitors. Following these patterns ensures performance, reliability, and cost efficiency.

---

## 9.1 Schema Design Patterns

### Star Schema Design

Star schemas organize data into fact tables (metrics) surrounded by dimension tables (attributes). Facts contain foreign keys to dimensions and numeric measures. This denormalized structure optimizes query performance by minimizing joins.

**[Diagram: Star Schema Architecture]**
```
         ┌─────────────┐
         │   DIM_DATE  │
         │ (date_key)  │
         └──────┬──────┘
                │
┌───────────┐   │   ┌──────────────┐   ┌──────────────┐
│DIM_PRODUCT│◄──┴──►│  FACT_SALES  │◄──┤ DIM_CUSTOMER │
│(prod_key) │       │ (all_keys +  │   │ (cust_key)   │
└───────────┘       │  measures)   │   └──────────────┘
                    └──────┬───────┘
                            │
                     ┌──────┴───────┐
                     │  DIM_STORE   │
                     │ (store_key)  │
                     └──────────────┘
```

**Implementation Example:**
```sql
-- Dimension: Date
CREATE OR REPLACE TABLE dim_date (
    date_key INT PRIMARY KEY,
    calendar_date DATE NOT NULL,
    day_of_week VARCHAR(10),
    month_name VARCHAR(10),
    quarter INT,
    fiscal_year INT,
    is_weekend BOOLEAN,
    CONSTRAINT uk_date UNIQUE (calendar_date)
) CLUSTER BY (date_key);

-- Dimension: Product
CREATE OR REPLACE TABLE dim_product (
    product_key INT PRIMARY KEY,
    product_id VARCHAR(50) NOT NULL,
    product_name VARCHAR(200),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    unit_price DECIMAL(10,2),
    is_active BOOLEAN,
    effective_date DATE,
    end_date DATE,
    CONSTRAINT uk_product UNIQUE (product_id, effective_date)
) CLUSTER BY (product_key);

-- Dimension: Customer (SCD Type 2)
CREATE OR REPLACE TABLE dim_customer (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    customer_name VARCHAR(200),
    email VARCHAR(100),
    segment VARCHAR(50),
    region VARCHAR(50),
    effective_date DATE,
    end_date DATE,
    is_current BOOLEAN,
    CONSTRAINT uk_customer UNIQUE (customer_id, effective_date)
) CLUSTER BY (customer_key);

-- Fact Table: Sales
CREATE OR REPLACE TABLE fact_sales (
    sales_key INT AUTOINCREMENT PRIMARY KEY,
    date_key INT NOT NULL,
    product_key INT NOT NULL,
    customer_key INT NOT NULL,
    store_key INT NOT NULL,
    
    -- Measures
    quantity INT,
    unit_price DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    tax_amount DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    
    -- Metadata
    transaction_id VARCHAR(100),
    transaction_timestamp TIMESTAMP_NTZ,
    
    CONSTRAINT fk_date FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    CONSTRAINT fk_product FOREIGN KEY (product_key) REFERENCES dim_product(product_key),
    CONSTRAINT fk_customer FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key)
) CLUSTER BY (date_key, product_key);

-- Analytical query leveraging star schema
SELECT 
    d.month_name,
    p.category,
    c.segment,
    SUM(f.total_amount) AS revenue,
    COUNT(DISTINCT f.transaction_id) AS transaction_count
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_customer c ON f.customer_key = c.customer_key
WHERE d.fiscal_year = 2024
    AND c.is_current = TRUE
GROUP BY d.month_name, p.category, c.segment;
```

### Snowflake Schema Design

Snowflake schemas normalize dimensions into sub-dimensions, reducing data redundancy. While more complex than star schemas, they save storage costs for large dimension tables. Modern Snowflake (the platform) handles both patterns efficiently due to columnar storage.

**[Diagram: Snowflake Schema Architecture]**
```
                  ┌────────────┐
                  │ DIM_REGION │
                  │(region_key)│
                  └─────┬──────┘
                        │
         ┌──────────────┴──────────────┐
         │      DIM_CUSTOMER           │
         │ (cust_key, region_key_fk)   │
         └──────────────┬──────────────┘
                        │
         ┌──────────────┴───────────────┐
         │        FACT_SALES            │
         │  (cust_key_fk + measures)    │
         └──────────────┬───────────────┘
                        │
         ┌──────────────┴───────────────┐
         │       DIM_PRODUCT            │
         │  (prod_key, cat_key_fk)      │
         └──────────────┬───────────────┘
                        │
                  ┌─────┴──────┐
                  │DIM_CATEGORY│
                  │(cat_key)   │
                  └────────────┘
```

**Implementation with Normalized Dimensions:**
```sql
-- Sub-dimension: Category
CREATE OR REPLACE TABLE dim_category (
    category_key INT PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL,
    department VARCHAR(100),
    division VARCHAR(100)
);

-- Dimension: Product (references Category)
CREATE OR REPLACE TABLE dim_product (
    product_key INT PRIMARY KEY,
    product_name VARCHAR(200),
    category_key INT NOT NULL,  -- Foreign key to sub-dimension
    brand VARCHAR(100),
    CONSTRAINT fk_category FOREIGN KEY (category_key) REFERENCES dim_category(category_key)
);

-- Query with additional join for normalized dimension
SELECT 
    cat.department,
    cat.category_name,
    SUM(f.total_amount) AS revenue
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_category cat ON p.category_key = cat.category_key
GROUP BY cat.department, cat.category_name;
```

### Slowly Changing Dimensions (SCD) Types

**SCD Type 1: Overwrite**
Updates dimension records in place, losing historical values.

```sql
-- Update customer segment (SCD Type 1)
UPDATE dim_customer_type1
SET segment = 'Premium',
    last_modified = CURRENT_TIMESTAMP()
WHERE customer_id = 'CUST_12345';
```

**SCD Type 2: Historical Tracking**
Maintains full history with effective dates and current flags. Requires surrogate keys.

```sql
-- Insert new version, expire old (SCD Type 2)
BEGIN TRANSACTION;

-- Expire current record
UPDATE dim_customer
SET end_date = CURRENT_DATE(),
    is_current = FALSE
WHERE customer_id = 'CUST_12345' 
    AND is_current = TRUE;

-- Insert new version
INSERT INTO dim_customer (
    customer_id, customer_name, segment, region,
    effective_date, end_date, is_current
) VALUES (
    'CUST_12345', 'John Doe', 'Premium', 'West',
    CURRENT_DATE(), '9999-12-31', TRUE
);

COMMIT;
```

---

## 9.2 Cost Optimization

### Warehouse Sizing and Auto-Scaling

Virtual warehouses consume credits proportional to size and uptime. Right-sizing requires analyzing query patterns, concurrency needs, and cost constraints. Auto-scaling adds clusters for concurrency while auto-suspend pauses idle warehouses.

**Syntax: ALTER WAREHOUSE (Complete Parameter Reference)**

```sql
-- SYNTAX TEMPLATE
ALTER WAREHOUSE [ IF EXISTS ] <name> SET
    [ WAREHOUSE_SIZE = X-SMALL | SMALL | MEDIUM | LARGE | X-LARGE | 
                       2X-LARGE | 3X-LARGE | 4X-LARGE | 5X-LARGE | 6X-LARGE ]
    [ MAX_CLUSTER_COUNT = <num> ]
    [ MIN_CLUSTER_COUNT = <num> ]
    [ SCALING_POLICY = STANDARD | ECONOMY ]
    [ AUTO_SUSPEND = <num> ]
    [ AUTO_RESUME = TRUE | FALSE ]
    [ INITIALLY_SUSPENDED = TRUE | FALSE ]
    [ RESOURCE_MONITOR = <monitor_name> ]
    [ COMMENT = '<string>' ]
    [ ENABLE_QUERY_ACCELERATION = TRUE | FALSE ]
    [ QUERY_ACCELERATION_MAX_SCALE_FACTOR = <num> ]

-- Or rename/suspend/resume
ALTER WAREHOUSE <name> RENAME TO <new_name>;
ALTER WAREHOUSE <name> SUSPEND;
ALTER WAREHOUSE <name> RESUME [ IF SUSPENDED ];
ALTER WAREHOUSE <name> ABORT ALL QUERIES;
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| WAREHOUSE_SIZE | Enum | (current) | Compute capacity (X-SMALL to 6X-LARGE) | LARGE |
| MAX_CLUSTER_COUNT | Int | 1 | Max clusters for auto-scaling (1-10) | 5 |
| MIN_CLUSTER_COUNT | Int | 1 | Min clusters always running (1-10) | 1 |
| SCALING_POLICY | Enum | STANDARD | Cluster spin-up speed (STANDARD=immediate, ECONOMY=conserve) | ECONOMY |
| AUTO_SUSPEND | Int | 600 | Idle seconds before suspension (NULL=never) | 300 |
| AUTO_RESUME | Bool | TRUE | Auto-resume on query submission | TRUE |
| RESOURCE_MONITOR | String | NULL | Monitor name for credit limits | 'prod_limit_monitor' |
| ENABLE_QUERY_ACCELERATION | Bool | FALSE | Accelerate eligible queries with additional compute | TRUE |
| QUERY_ACCELERATION_MAX_SCALE_FACTOR | Int | 8 | Max scale factor for query acceleration (0-100) | 16 |

**BASIC EXAMPLE**
```sql
-- Resize warehouse for peak load
ALTER WAREHOUSE compute_wh SET WAREHOUSE_SIZE = LARGE;

-- Enable auto-suspend after 5 minutes idle
ALTER WAREHOUSE compute_wh SET AUTO_SUSPEND = 300;
```

**ADVANCED EXAMPLE (SDLC: Production Multi-Cluster Warehouse)**
```sql
-- Production warehouse: auto-scaling for concurrency, aggressive auto-suspend
ALTER WAREHOUSE prod_analytics_wh SET
    WAREHOUSE_SIZE = X-LARGE,
    MAX_CLUSTER_COUNT = 5,
    MIN_CLUSTER_COUNT = 1,
    SCALING_POLICY = STANDARD,        -- Fast cluster spin-up
    AUTO_SUSPEND = 120,                -- Suspend after 2 min idle
    AUTO_RESUME = TRUE,
    ENABLE_QUERY_ACCELERATION = TRUE,
    QUERY_ACCELERATION_MAX_SCALE_FACTOR = 16,
    RESOURCE_MONITOR = 'prod_monthly_limit',
    COMMENT = 'Production analytics warehouse - auto-scaling enabled';

-- Monitor warehouse status
SELECT 
    name,
    state,
    size,
    running,
    queued,
    auto_suspend,
    auto_resume,
    scaling_policy
FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_METERING_HISTORY(
    DATE_RANGE_START => DATEADD(day, -7, CURRENT_DATE()),
    WAREHOUSE_NAME => 'PROD_ANALYTICS_WH'
));
```

### Resource Monitors

Resource monitors cap credit consumption per warehouse or account. They trigger notifications or suspend warehouses when thresholds are reached. Essential for cost control in production environments.

**Syntax: CREATE RESOURCE MONITOR**

```sql
-- SYNTAX TEMPLATE
CREATE [ OR REPLACE ] RESOURCE MONITOR <name> WITH
    CREDIT_QUOTA = <num>
    [ FREQUENCY = MONTHLY | DAILY | WEEKLY | YEARLY | NEVER ]
    [ START_TIMESTAMP = '<timestamp>' | IMMEDIATELY ]
    [ END_TIMESTAMP = '<timestamp>' ]
    [ NOTIFY_USERS = ( <user1>, <user2>, ... ) ]
    [ TRIGGERS 
        ON <percent> PERCENT DO NOTIFY
        [ ON <percent> PERCENT DO SUSPEND ]
        [ ON <percent> PERCENT DO SUSPEND_IMMEDIATE ]
    ];

-- Assign to warehouse
ALTER WAREHOUSE <name> SET RESOURCE_MONITOR = <monitor_name>;

-- Assign to account (monitors all warehouses)
ALTER ACCOUNT SET RESOURCE_MONITOR = <monitor_name>;
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| CREDIT_QUOTA | Number | Required | Max credits per period | 1000 |
| FREQUENCY | Enum | MONTHLY | Reset period | DAILY |
| START_TIMESTAMP | Timestamp | IMMEDIATELY | When quota tracking starts | '2024-01-01 00:00:00' |
| END_TIMESTAMP | Timestamp | NULL | When monitor expires | '2024-12-31 23:59:59' |
| NOTIFY_USERS | List | NULL | Users to notify at thresholds | ('admin@company.com') |
| ON X PERCENT DO NOTIFY | Trigger | NULL | Send alert at X% usage | 80 |
| ON X PERCENT DO SUSPEND | Trigger | NULL | Suspend at X% (graceful) | 100 |
| ON X PERCENT DO SUSPEND_IMMEDIATE | Trigger | NULL | Suspend at X% (immediately) | 110 |

**EXAMPLE: Multi-Tier Alerting**
```sql
-- Create resource monitor with escalating actions
CREATE OR REPLACE RESOURCE MONITOR prod_monthly_limit WITH
    CREDIT_QUOTA = 5000
    FREQUENCY = MONTHLY
    START_TIMESTAMP = '2024-01-01 00:00:00'
    NOTIFY_USERS = ('dba@company.com', 'finance@company.com')
    TRIGGERS
        ON 75 PERCENT DO NOTIFY                -- Alert at 75%
        ON 90 PERCENT DO NOTIFY                -- Alert at 90%
        ON 100 PERCENT DO SUSPEND              -- Graceful suspend at 100%
        ON 110 PERCENT DO SUSPEND_IMMEDIATE;   -- Immediate suspend at 110%

-- Assign to production warehouses
ALTER WAREHOUSE prod_etl_wh SET RESOURCE_MONITOR = prod_monthly_limit;
ALTER WAREHOUSE prod_analytics_wh SET RESOURCE_MONITOR = prod_monthly_limit;

-- View monitor status
SHOW RESOURCE MONITORS;
SELECT * FROM TABLE(INFORMATION_SCHEMA.RESOURCE_MONITORS());
```

### Storage Cost Optimization

**Time Travel Retention Tuning:**
```sql
-- Reduce Time Travel for transient staging data (lower storage costs)
ALTER TABLE staging.temp_orders SET DATA_RETENTION_TIME_IN_DAYS = 1;

-- Standard retention for production tables
ALTER TABLE production.orders SET DATA_RETENTION_TIME_IN_DAYS = 7;

-- Maximum retention for compliance tables
ALTER TABLE compliance.audit_log SET DATA_RETENTION_TIME_IN_DAYS = 90;
```

**Table Type Selection:**
```sql
-- TRANSIENT tables: No Fail-Safe, ideal for staging/temp data
CREATE TRANSIENT TABLE staging.daily_extract (
    extract_date DATE,
    data_json VARIANT
) DATA_RETENTION_TIME_IN_DAYS = 0;  -- Zero Time Travel for max savings

-- TEMPORARY tables: Session-scoped, auto-dropped
CREATE TEMPORARY TABLE session_temp AS
SELECT * FROM large_table WHERE region = 'WEST';
```

**Clustering Key Optimization:**
Proper clustering reduces micro-partition scanning, lowering compute costs.

```sql
-- Analyze clustering quality
SELECT SYSTEM$CLUSTERING_INFORMATION('sales', '(order_date, customer_id)');

-- Add clustering key to poorly clustered table
ALTER TABLE sales CLUSTER BY (order_date, customer_id);

-- Monitor clustering depth (lower = better)
SELECT 
    table_name,
    clustering_key,
    average_depth,
    average_overlaps
FROM TABLE(INFORMATION_SCHEMA.AUTOMATIC_CLUSTERING_HISTORY(
    DATE_RANGE_START => DATEADD(day, -7, CURRENT_DATE())
));
```

---

## 9.3 Backup and Disaster Recovery

### Time Travel for Data Recovery

Time Travel enables querying historical data states without physical backups. Critical for recovering from accidental deletes, updates, or schema changes.

**Syntax: Querying Historical Data**

```sql
-- Query table as of timestamp
SELECT * FROM orders AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);

-- Query table as of 1 hour ago
SELECT * FROM orders AT(OFFSET => -3600);

-- Query table before specific statement
SELECT * FROM orders BEFORE(STATEMENT => '01a234b5-6789-0123-cdef-456789abcdef');

-- Restore deleted table
UNDROP TABLE orders;

-- Restore dropped database
UNDROP DATABASE analytics_db;

-- Clone table from historical point
CREATE TABLE orders_backup CLONE orders 
    AT(TIMESTAMP => '2024-01-15 00:00:00'::TIMESTAMP);
```

**SDLC Use Case: Production Incident Recovery**
```sql
-- Scenario: Accidental mass update at 2024-01-20 14:30
-- Step 1: Verify bad data
SELECT COUNT(*) FROM orders WHERE order_status = 'INVALID';

-- Step 2: Clone table before incident
CREATE TABLE orders_recovery CLONE orders
    BEFORE(TIMESTAMP => '2024-01-20 14:25:00'::TIMESTAMP);

-- Step 3: Validate recovered data
SELECT COUNT(*) FROM orders_recovery WHERE order_status = 'INVALID';
-- Expected: 0

-- Step 4: Restore production table (requires DROP + RENAME)
BEGIN TRANSACTION;
    ALTER TABLE orders RENAME TO orders_corrupted;
    ALTER TABLE orders_recovery RENAME TO orders;
COMMIT;

-- Step 5: Verify production restored
SELECT MAX(updated_at) FROM orders;  -- Should be < 14:30
```

### Zero-Copy Cloning for Environments

Zero-copy clones create instant, space-efficient copies for dev/test/QA environments.

```sql
-- Clone production database to QA (instant, no storage overhead initially)
CREATE DATABASE qa_analytics CLONE production_analytics;

-- Clone specific schema with data as of yesterday
CREATE SCHEMA dev_analytics.staging CLONE production_analytics.staging
    AT(OFFSET => -86400);  -- 24 hours ago

-- Grant access to dev team
GRANT USAGE ON DATABASE qa_analytics TO ROLE developer;
GRANT SELECT ON ALL TABLES IN SCHEMA qa_analytics.public TO ROLE developer;
```

### Replication for Disaster Recovery

Snowflake's replication feature creates live copies across regions/clouds for failover.

```sql
-- Enable replication for primary database (ACCOUNTADMIN required)
ALTER DATABASE production_db ENABLE REPLICATION TO ACCOUNTS myorg.dr_account;

-- On DR account: Create replica database
CREATE DATABASE production_db_replica AS REPLICA OF myorg.prod_account.production_db;

-- Refresh replica (manual or scheduled)
ALTER DATABASE production_db_replica REFRESH;

-- Promote replica to primary (failover scenario)
ALTER DATABASE production_db_replica PRIMARY;
```

---

## 9.4 Monitoring and Auditing

### Query Performance Monitoring

**Syntax: SHOW Commands with Filtering**

```sql
-- SHOW WAREHOUSES (all warehouses in account)
SHOW WAREHOUSES;
SHOW WAREHOUSES LIKE 'PROD%';

-- SHOW TABLES with filtering
SHOW TABLES IN SCHEMA analytics.public;
SHOW TABLES LIKE 'fact_%' IN DATABASE analytics;

-- SHOW SCHEMAS
SHOW SCHEMAS IN DATABASE analytics;
SHOW SCHEMAS IN ACCOUNT;

-- Filter options
SHOW TABLES 
    IN DATABASE analytics 
    STARTS WITH 'dim_' 
    LIMIT 10;
```

**QUERY_HISTORY View (Key Columns)**

Access via `INFORMATION_SCHEMA.QUERY_HISTORY` or `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`.

```sql
-- SYNTAX: Query History Analysis
SELECT
    query_id,
    query_text,
    user_name,
    role_name,
    warehouse_name,
    warehouse_size,
    execution_status,
    error_code,
    error_message,
    start_time,
    end_time,
    total_elapsed_time,
    bytes_scanned,
    rows_produced,
    compilation_time,
    execution_time,
    queued_provisioning_time,
    queued_overload_time,
    transaction_blocked_time,
    credits_used_cloud_services,
    query_type,
    session_id,
    query_tag
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
    AND warehouse_name = 'PROD_WH'
ORDER BY total_elapsed_time DESC
LIMIT 100;
```

**KEY QUERY_HISTORY COLUMNS**

| Column | Type | Description | Use Case |
|--------|------|-------------|----------|
| query_id | String | Unique query identifier | Drill-down debugging |
| query_text | String | Full SQL statement | Identify expensive queries |
| total_elapsed_time | Number | Milliseconds from submit to complete | Performance analysis |
| bytes_scanned | Number | Data volume read | Storage/clustering optimization |
| warehouse_size | String | Warehouse size during execution | Right-sizing analysis |
| credits_used_cloud_services | Number | Cloud services credits consumed | Cost attribution |
| execution_status | String | SUCCESS, FAIL, RUNNING | Error rate monitoring |
| query_tag | String | User-defined query label | Cost allocation by team/app |
| queued_overload_time | Number | Milliseconds queued due to concurrency | Auto-scaling trigger |

**ADVANCED EXAMPLE: Performance Dashboard Query**
```sql
-- Top 10 slowest queries in last 24 hours with cost attribution
WITH query_metrics AS (
    SELECT
        query_id,
        LEFT(query_text, 100) AS query_preview,
        user_name,
        warehouse_name,
        warehouse_size,
        total_elapsed_time / 1000.0 AS duration_seconds,
        bytes_scanned / POWER(1024, 3) AS gb_scanned,
        credits_used_cloud_services,
        queued_overload_time / 1000.0 AS queue_seconds,
        execution_status,
        start_time
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP())
        AND execution_status = 'SUCCESS'
        AND warehouse_name IS NOT NULL
)
SELECT
    query_id,
    query_preview,
    user_name,
    warehouse_name,
    warehouse_size,
    ROUND(duration_seconds, 2) AS duration_sec,
    ROUND(gb_scanned, 2) AS gb_scanned,
    ROUND(credits_used_cloud_services, 4) AS credits_used,
    ROUND(queue_seconds, 2) AS queue_sec,
    start_time
FROM query_metrics
ORDER BY duration_seconds DESC
LIMIT 10;
```

### Access Auditing

Track user activity and privilege changes via `ACCESS_HISTORY` and `GRANTS_TO_USERS` views.

```sql
-- Recent table access patterns
SELECT
    user_name,
    query_id,
    object_name,
    object_domain,
    privilege,
    query_start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE query_start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
    AND object_name = 'SENSITIVE_DATA_TABLE'
ORDER BY query_start_time DESC;

-- Track privilege grants/revokes
SELECT
    created_on,
    modified_on,
    privilege,
    granted_on,
    name AS object_name,
    grantee_name,
    granted_by,
    deleted_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE grantee_name = 'ANALYST_ROLE'
    AND deleted_on IS NULL  -- Currently active grants
ORDER BY created_on DESC;
```

### Cost and Usage Dashboards

**Warehouse Metering:**
```sql
-- Daily credit consumption by warehouse
SELECT
    warehouse_name,
    DATE(start_time) AS usage_date,
    SUM(credits_used) AS total_credits,
    SUM(credits_used_compute) AS compute_credits,
    SUM(credits_used_cloud_services) AS cloud_services_credits,
    AVG(avg_running) AS avg_warehouses_running
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY warehouse_name, DATE(start_time)
ORDER BY usage_date DESC, total_credits DESC;
```

**Storage Costs:**
```sql
-- Database storage growth over time
SELECT
    usage_date,
    database_name,
    ROUND(average_database_bytes / POWER(1024, 4), 2) AS avg_tb_stored,
    ROUND(average_failsafe_bytes / POWER(1024, 4), 2) AS avg_tb_failsafe
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY
WHERE usage_date >= DATEADD(month, -3, CURRENT_TIMESTAMP())
ORDER BY usage_date DESC, avg_tb_stored DESC;
```

**SDLC Use Case: Automated Cost Alerting**
```python
# Airflow DAG: Daily cost anomaly detection
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import requests

def check_cost_anomaly(**context):
    """Alert if daily credits exceed 110% of 7-day average"""
    
    # Query passed from SnowflakeOperator via XCom
    result = context['ti'].xcom_pull(task_ids='get_daily_credits')
    today_credits = result[0][0]
    avg_credits = result[0][1]
    
    if today_credits > avg_credits * 1.10:
        # Send Slack alert
        requests.post(
            'https://hooks.slack.com/services/YOUR/WEBHOOK/URL',
            json={
                'text': f'⚠️ Cost Alert: Today={today_credits:.2f} credits, 7-day avg={avg_credits:.2f}'
            }
        )

with DAG(
    'snowflake_cost_monitoring',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False
) as dag:
    
    get_credits = SnowflakeOperator(
        task_id='get_daily_credits',
        sql="""
            SELECT 
                SUM(CASE WHEN usage_date = CURRENT_DATE() THEN credits_used ELSE 0 END) AS today,
                AVG(CASE WHEN usage_date >= DATEADD(day, -7, CURRENT_DATE()) 
                    THEN credits_used ELSE NULL END) AS avg_7day
            FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
            WHERE usage_date >= DATEADD(day, -7, CURRENT_DATE())
        """,
        snowflake_conn_id='snowflake_prod',
        do_xcom_push=True
    )
    
    check_alert = PythonOperator(
        task_id='check_cost_anomaly',
        python_callable=check_cost_anomaly,
        provide_context=True
    )
    
    get_credits >> check_alert
```

---

## 9.5 Security Best Practices

### Principle of Least Privilege

Grant minimal permissions required for each role. Use role hierarchies and avoid granting ACCOUNTADMIN unless absolutely necessary.

```sql
-- Role hierarchy: Analyst → Data Engineer → Admin
CREATE ROLE analyst_role;
CREATE ROLE data_engineer_role;
GRANT ROLE analyst_role TO ROLE data_engineer_role;

-- Analyst: Read-only access
GRANT USAGE ON DATABASE analytics TO ROLE analyst_role;
GRANT USAGE ON SCHEMA analytics.public TO ROLE analyst_role;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.public TO ROLE analyst_role;

-- Data Engineer: Write access + warehouse management
GRANT ALL PRIVILEGES ON SCHEMA analytics.staging TO ROLE data_engineer_role;
GRANT OPERATE ON WAREHOUSE etl_wh TO ROLE data_engineer_role;

-- Never grant this unless required
-- GRANT ROLE ACCOUNTADMIN TO USER risky_user;  -- ❌ AVOID
```

### Network Policies

Restrict access to Snowflake from approved IP ranges.

```sql
-- Create network policy for corporate IPs
CREATE NETWORK POLICY corp_office_policy
    ALLOWED_IP_LIST = ('203.0.113.0/24', '198.51.100.5')
    BLOCKED_IP_LIST = ('192.0.2.0/24')
    COMMENT = 'Corporate office + VPN access only';

-- Apply to account
ALTER ACCOUNT SET NETWORK_POLICY = corp_office_policy;

-- Apply to specific user
ALTER USER remote_contractor SET NETWORK_POLICY = vpn_only_policy;
```

### Multi-Factor Authentication (MFA)

Enforce MFA for privileged accounts via account settings (ACCOUNTADMIN required).

```sql
-- Require MFA for all users
ALTER ACCOUNT SET MFA_ENROLLMENT = REQUIRED;

-- Check MFA status
SELECT 
    name,
    has_mfa,
    mfa_enrollment_time
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL;
```

---

## 9.6 Data Lifecycle Management

### Archival Strategy

Move cold data to cheaper storage tiers or external stages for long-term retention.

```sql
-- Archive orders older than 2 years to S3
COPY INTO @s3_archive_stage/orders/year=2022/
FROM (
    SELECT * FROM orders WHERE YEAR(order_date) = 2022
)
FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY)
HEADER = TRUE
OVERWRITE = TRUE;

-- Delete archived data from active table
DELETE FROM orders WHERE YEAR(order_date) = 2022;
```

### Automated Data Purging

```sql
-- Create task to purge staging data older than 7 days
CREATE OR REPLACE TASK purge_staging_data
    WAREHOUSE = 'ADMIN_WH'
    SCHEDULE = 'USING CRON 0 2 * * * America/New_York'  -- Daily 2am ET
AS
    DELETE FROM staging.temp_data
    WHERE created_at < DATEADD(day, -7, CURRENT_DATE());

-- Enable task
ALTER TASK purge_staging_data RESUME;

-- Monitor task history
SELECT 
    name,
    state,
    scheduled_time,
    completed_time,
    error_code,
    error_message
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'PURGE_STAGING_DATA',
    SCHEDULED_TIME_RANGE_START => DATEADD(day, -7, CURRENT_TIMESTAMP())
))
ORDER BY scheduled_time DESC;
```

---

## Glossary of Best Practices Terms

- **Star Schema**: Denormalized design with central fact table surrounded by dimension tables
- **Snowflake Schema**: Normalized design where dimensions have sub-dimensions (not to be confused with Snowflake the platform)
- **SCD Type 2**: Slowly Changing Dimension tracking full history with effective dates
- **Auto-Scaling**: Automatic addition/removal of warehouse clusters based on workload
- **Auto-Suspend**: Automatic warehouse suspension after idle period to reduce costs
- **Resource Monitor**: Credit quota enforcement with notification/suspension triggers
- **Clustering Key**: Column(s) used to co-locate related rows in micro-partitions
- **Clustering Depth**: Measure of micro-partition overlap (lower is better)
- **Zero-Copy Clone**: Instant table/schema/database copy using metadata pointers
- **Time Travel**: Ability to query/restore data from past (1-90 days retention)
- **Fail-Safe**: 7-day recovery period after Time Travel expires (Snowflake-managed)
- **Transient Table**: Table without Fail-Safe period, lower storage costs
- **Temporary Table**: Session-scoped table, auto-dropped at session end
- **Query Tag**: User-defined label for cost allocation and query grouping
- **Query Profile**: Visual execution plan showing performance bottlenecks
- **Micro-Partition Pruning**: Skipping irrelevant partitions via metadata, reduces scanning
- **Result Caching**: Reusing query results for identical queries (24-hour TTL)
- **Warehouse Credit**: Unit of compute consumption (varies by warehouse size)
- **Cloud Services Credit**: Background processing credit (metadata, optimization, etc.)
- **Network Policy**: IP whitelist/blacklist restricting account access
- **Multi-Factor Authentication (MFA)**: Additional authentication layer beyond password
- **ACCOUNTADMIN**: Highest privilege role, should be restricted
- **Least Privilege**: Security principle granting minimal required permissions
- **Role Hierarchy**: Parent roles inherit child role privileges
- **Replication**: Cross-region/account database copies for disaster recovery
- **Failover**: Promoting replica database to primary during outage
- **Access History**: Audit log of object-level access (queries against tables/views)
- **Warehouse Metering**: Credit consumption tracking per warehouse
- **Storage Metering**: Database storage usage tracking (active, Time Travel, Fail-Safe)
- **CRON Expression**: Schedule syntax for task execution (minute, hour, day, month, weekday)
