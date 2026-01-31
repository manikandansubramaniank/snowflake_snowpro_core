# 10. Real-world Examples

## Overview

This section demonstrates complete, production-ready Snowflake implementations across common enterprise scenarios. Each example provides end-to-end workflows with full SQL syntax, configuration details, and SDLC integration patterns. Examples cover database migration, cloud ETL pipelines, analytical query optimization, and Agile sprint delivery.

---

## 10.1 Oracle to Snowflake Migration

### Migration Overview

Migrating from Oracle to Snowflake requires schema conversion, data extraction/loading, and query rewriting. Key considerations include data type mappings (Oracle NUMBER → Snowflake DECIMAL), sequence replacement (Oracle SEQUENCE → Snowflake AUTOINCREMENT), and procedural code migration (PL/SQL → Snowflake stored procedures).

**[Diagram: Migration Architecture]**
```
┌──────────────────┐  1. Export    ┌──────────────────┐
│  Oracle Database │──────────────►│  AWS S3 Staging  │
│  (Source)        │   (EXPDP/     │  (Parquet/CSV)   │
└──────────────────┘    sqlldr)    └────────┬─────────┘
                                            │
                                   2. COPY INTO
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │   Snowflake     │
                                   │   (Target)      │
                                   └─────────────────┘
                                            │
                               3. Transform │
                                (dbt/SQL)   │
                                            ▼
                                   ┌────────────────┐
                                   │  Production    │
                                   │  Analytics DB  │
                                   └────────────────┘
```

### Phase 1: Schema Mapping and DDL Conversion

**Oracle Schema (Source):**
```sql
-- Oracle DDL (original)
CREATE TABLE employees (
    employee_id NUMBER PRIMARY KEY,
    first_name VARCHAR2(50) NOT NULL,
    last_name VARCHAR2(50) NOT NULL,
    email VARCHAR2(100) UNIQUE,
    hire_date DATE,
    salary NUMBER(10,2),
    department_id NUMBER,
    commission_pct NUMBER(2,2),
    manager_id NUMBER,
    created_date DATE DEFAULT SYSDATE,
    CONSTRAINT fk_dept FOREIGN KEY (department_id) 
        REFERENCES departments(department_id),
    CONSTRAINT chk_salary CHECK (salary > 0)
);

CREATE SEQUENCE emp_seq START WITH 1000 INCREMENT BY 1 CACHE 20;

CREATE INDEX idx_emp_dept ON employees(department_id);
CREATE INDEX idx_emp_hire ON employees(hire_date);
```

**Converted Snowflake Schema:**
```sql
-- Snowflake DDL (converted)
CREATE OR REPLACE TABLE employees (
    employee_id NUMBER AUTOINCREMENT PRIMARY KEY,  -- Oracle SEQUENCE → AUTOINCREMENT
    first_name VARCHAR(50) NOT NULL,               -- VARCHAR2 → VARCHAR
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    hire_date DATE,
    salary DECIMAL(10,2),                          -- NUMBER(10,2) → DECIMAL
    department_id NUMBER,
    commission_pct DECIMAL(4,2),                   -- NUMBER(2,2) → DECIMAL(4,2) for precision
    manager_id NUMBER,
    created_date TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),  -- SYSDATE → CURRENT_TIMESTAMP
    
    -- Foreign key (not enforced, informational only in Snowflake)
    CONSTRAINT fk_dept FOREIGN KEY (department_id) 
        REFERENCES departments(department_id),
    
    -- Check constraints supported
    CONSTRAINT chk_salary CHECK (salary > 0)
) 
CLUSTER BY (department_id);  -- Snowflake optimization replacing Oracle index

-- Indexes replaced by clustering keys (Snowflake auto-maintains)
-- No manual CREATE INDEX needed in Snowflake
```

**Data Type Mapping Reference:**

| Oracle Type | Snowflake Type | Notes |
|-------------|----------------|-------|
| VARCHAR2(N) | VARCHAR(N) | Max 16MB |
| NUMBER | NUMBER | Default precision/scale |
| NUMBER(P,S) | DECIMAL(P,S) | Explicit precision |
| DATE | DATE or TIMESTAMP_NTZ | Oracle DATE includes time |
| TIMESTAMP | TIMESTAMP_NTZ | No timezone |
| CLOB | VARCHAR(16MB) | LOBs as VARCHAR |
| BLOB | BINARY | Binary data |
| ROWID | Not supported | Use surrogate keys |

### Phase 2: Data Extraction and Staging

**Oracle Data Export to S3 (Using AWS DMS or Custom Script):**
```bash
#!/bin/bash
# Oracle export to S3 using Data Pump + S3 upload

# 1. Oracle Data Pump export
expdp system/password@ORCL \
    directory=DATA_PUMP_DIR \
    dumpfile=employees_export_%U.dmp \
    logfile=employees_export.log \
    tables=HR.EMPLOYEES \
    parallel=4 \
    compression=ALL

# 2. Convert dump to Parquet (using external tool or AWS Glue)
# Assume conversion produces employees.parquet

# 3. Upload to S3
aws s3 cp employees.parquet s3://migration-bucket/oracle-exports/employees/ \
    --storage-class INTELLIGENT_TIERING
```

**Alternative: Direct Oracle Query to CSV:**
```sql
-- Oracle SQL*Plus export to CSV
SET COLSEP ','
SET PAGESIZE 0
SET TRIMSPOOL ON
SET HEADSEP OFF
SET LINESIZE 32767
SET FEEDBACK OFF

SPOOL employees_export.csv
SELECT 
    employee_id,
    first_name,
    last_name,
    email,
    TO_CHAR(hire_date, 'YYYY-MM-DD') AS hire_date,
    salary,
    department_id,
    commission_pct,
    manager_id,
    TO_CHAR(created_date, 'YYYY-MM-DD HH24:MI:SS') AS created_date
FROM employees;
SPOOL OFF
```

### Phase 3: Loading Data into Snowflake

**Create Snowflake External Stage:**
```sql
-- SYNTAX: CREATE STAGE (complete parameters)
CREATE OR REPLACE STAGE oracle_migration_stage
    URL = 's3://migration-bucket/oracle-exports/'
    STORAGE_INTEGRATION = aws_migration_integration  -- Pre-configured S3 integration
    FILE_FORMAT = (
        TYPE = PARQUET
        COMPRESSION = AUTO
        BINARY_AS_TEXT = FALSE
    )
    DIRECTORY = (
        ENABLE = TRUE
        AUTO_REFRESH = FALSE
    )
    COMMENT = 'Oracle migration staging area';

-- Verify stage contents
LIST @oracle_migration_stage;

-- Sample data from stage
SELECT $1, $2, $3 FROM @oracle_migration_stage/employees/ LIMIT 10;
```

**Load Data with COPY INTO:**
```sql
-- COPY INTO with validation and error handling
COPY INTO employees
FROM @oracle_migration_stage/employees/
FILE_FORMAT = (TYPE = PARQUET)
PATTERN = '.*employees.*\\.parquet'
ON_ERROR = CONTINUE              -- Log errors, continue loading
RETURN_FAILED_ONLY = TRUE        -- Only return failed rows
VALIDATION_MODE = RETURN_ERRORS; -- Dry-run validation

-- Review errors
SELECT 
    rejected_record,
    error,
    file,
    line
FROM TABLE(VALIDATE(employees, JOB_ID => '_last'));

-- Production load after validation
COPY INTO employees
FROM @oracle_migration_stage/employees/
FILE_FORMAT = (TYPE = PARQUET)
PATTERN = '.*employees.*\\.parquet'
ON_ERROR = ABORT_STATEMENT       -- Fail on any error
PURGE = TRUE;                    -- Delete source files after load

-- Verify row counts
SELECT COUNT(*) AS snowflake_count FROM employees;
-- Compare with Oracle: SELECT COUNT(*) FROM HR.EMPLOYEES;
```

### Phase 4: Migrating Oracle PL/SQL to Snowflake Stored Procedures

**Oracle PL/SQL Procedure:**
```sql
-- Oracle PL/SQL (original)
CREATE OR REPLACE PROCEDURE calculate_annual_bonus (
    p_department_id IN NUMBER,
    p_bonus_pct IN NUMBER,
    p_total_bonus OUT NUMBER
) AS
    v_dept_total NUMBER;
BEGIN
    SELECT SUM(salary * p_bonus_pct / 100)
    INTO v_dept_total
    FROM employees
    WHERE department_id = p_department_id;
    
    p_total_bonus := v_dept_total;
    
    UPDATE employees
    SET commission_pct = p_bonus_pct
    WHERE department_id = p_department_id;
    
    COMMIT;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        p_total_bonus := 0;
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
```

**Snowflake JavaScript Stored Procedure (Converted):**
```sql
-- Snowflake JavaScript procedure (converted)
CREATE OR REPLACE PROCEDURE calculate_annual_bonus(
    p_department_id NUMBER,
    p_bonus_pct FLOAT
)
RETURNS FLOAT
LANGUAGE JAVASCRIPT
EXECUTE AS CALLER
AS
$$
    // Calculate total bonus for department
    var sql_select = `
        SELECT COALESCE(SUM(salary * :1 / 100), 0) AS dept_total
        FROM employees
        WHERE department_id = :2
    `;
    
    var stmt_select = snowflake.createStatement({
        sqlText: sql_select,
        binds: [P_BONUS_PCT, P_DEPARTMENT_ID]
    });
    
    var result_set = stmt_select.execute();
    result_set.next();
    var total_bonus = result_set.getColumnValue('DEPT_TOTAL');
    
    // Update employee commission percentages
    var sql_update = `
        UPDATE employees
        SET commission_pct = :1
        WHERE department_id = :2
    `;
    
    var stmt_update = snowflake.createStatement({
        sqlText: sql_update,
        binds: [P_BONUS_PCT, P_DEPARTMENT_ID]
    });
    
    try {
        stmt_update.execute();
    } catch (err) {
        throw new Error("Bonus update failed: " + err.message);
    }
    
    return total_bonus;
$$;

-- Execute procedure
CALL calculate_annual_bonus(10, 5.0);  -- Returns total bonus for dept 10
```

### Phase 5: Query Rewriting

**Oracle SQL with Hierarchical Query:**
```sql
-- Oracle hierarchical query (CONNECT BY)
SELECT 
    LEVEL,
    employee_id,
    first_name || ' ' || last_name AS full_name,
    manager_id,
    SYS_CONNECT_BY_PATH(last_name, ' -> ') AS hierarchy_path
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id
ORDER SIBLINGS BY last_name;
```

**Snowflake Recursive CTE (Converted):**
```sql
-- Snowflake recursive CTE (equivalent)
WITH RECURSIVE employee_hierarchy AS (
    -- Anchor: Top-level managers
    SELECT 
        1 AS level,
        employee_id,
        first_name || ' ' || last_name AS full_name,
        manager_id,
        last_name AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: Subordinates
    SELECT 
        eh.level + 1,
        e.employee_id,
        e.first_name || ' ' || e.last_name,
        e.manager_id,
        eh.hierarchy_path || ' -> ' || e.last_name
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy
ORDER BY hierarchy_path;
```

**SDLC Integration: Automated Migration Testing**
```python
# pytest: Validate Oracle-to-Snowflake migration
import pytest
import cx_Oracle  # Oracle connector
import snowflake.connector

@pytest.fixture(scope='module')
def oracle_conn():
    conn = cx_Oracle.connect('hr/password@localhost:1521/ORCL')
    yield conn
    conn.close()

@pytest.fixture(scope='module')
def snowflake_conn():
    conn = snowflake.connector.connect(
        account='myaccount',
        user='migration_test',
        warehouse='MIGRATION_WH',
        database='MIGRATED_HR'
    )
    yield conn
    conn.close()

def test_row_count_match(oracle_conn, snowflake_conn):
    """Verify row counts match between Oracle and Snowflake"""
    oracle_cur = oracle_conn.cursor()
    oracle_cur.execute("SELECT COUNT(*) FROM employees")
    oracle_count = oracle_cur.fetchone()[0]
    
    sf_cur = snowflake_conn.cursor()
    sf_cur.execute("SELECT COUNT(*) FROM employees")
    sf_count = sf_cur.fetchone()[0]
    
    assert oracle_count == sf_count, \
        f"Row count mismatch: Oracle={oracle_count}, Snowflake={sf_count}"

def test_salary_sum_match(oracle_conn, snowflake_conn):
    """Verify aggregate calculations match"""
    oracle_cur = oracle_conn.cursor()
    oracle_cur.execute("SELECT SUM(salary) FROM employees")
    oracle_sum = float(oracle_cur.fetchone()[0])
    
    sf_cur = snowflake_conn.cursor()
    sf_cur.execute("SELECT SUM(salary) FROM employees")
    sf_sum = float(sf_cur.fetchone()[0])
    
    # Allow 0.01% variance for floating point differences
    assert abs(oracle_sum - sf_sum) / oracle_sum < 0.0001, \
        f"Salary sum mismatch: Oracle={oracle_sum}, Snowflake={sf_sum}"
```

---

## 10.2 AWS S3 + Snowpipe Real-time ETL

### Architecture Overview

Snowpipe enables continuous, event-driven data ingestion from S3. When new files land in S3, SQS notifications trigger Snowpipe to automatically load data without manual intervention. Ideal for streaming logs, IoT telemetry, and real-time analytics.

**[Diagram: Snowpipe Architecture]**
```
┌─────────────┐  1. Upload   ┌──────────────┐  2. SQS Event  ┌─────────────┐
│ Application │─────────────►│   AWS S3     │───────────────►│  Snowflake  │
│ (Logs/Data) │              │   Bucket     │                │  Snowpipe   │
└─────────────┘              └──────────────┘                └───────┬─────┘
                                                                     │
                                                            3. Auto-COPY
                                                                     │
                                                                     ▼
                                                            ┌────────────────┐
                                                            │  Target Table  │
                                                            │  (Snowflake)   │
                                                            └────────────────┘
```

### Step 1: S3 Event Configuration

**Create S3 Bucket with Event Notification:**
```bash
# AWS CLI: Configure S3 event notification to SQS
aws s3api create-bucket \
    --bucket my-app-logs \
    --region us-east-1

# Create SQS queue for S3 events
aws sqs create-queue \
    --queue-name snowpipe-s3-events \
    --attributes MessageRetentionPeriod=86400

# Get queue ARN
QUEUE_ARN=$(aws sqs get-queue-attributes \
    --queue-url https://sqs.us-east-1.amazonaws.com/123456789/snowpipe-s3-events \
    --attribute-names QueueArn \
    --query 'Attributes.QueueArn' \
    --output text)

# Configure S3 bucket notification
aws s3api put-bucket-notification-configuration \
    --bucket my-app-logs \
    --notification-configuration '{
        "QueueConfigurations": [{
            "QueueArn": "'$QUEUE_ARN'",
            "Events": ["s3:ObjectCreated:*"],
            "Filter": {
                "Key": {
                    "FilterRules": [{
                        "Name": "prefix",
                        "Value": "logs/application/"
                    }, {
                        "Name": "suffix",
                        "Value": ".json"
                    }]
                }
            }
        }]
    }'
```

### Step 2: Snowflake Storage Integration

```sql
-- SYNTAX: CREATE STORAGE INTEGRATION (complete parameters)
CREATE OR REPLACE STORAGE INTEGRATION s3_app_logs_integration
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = S3
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/snowflake-s3-access-role'
    STORAGE_ALLOWED_LOCATIONS = (
        's3://my-app-logs/logs/application/',
        's3://my-app-logs/logs/errors/'
    )
    STORAGE_BLOCKED_LOCATIONS = (
        's3://my-app-logs/sensitive/'
    )
    COMMENT = 'Integration for application log ingestion';

-- Retrieve Snowflake IAM user for S3 trust relationship
DESC STORAGE INTEGRATION s3_app_logs_integration;
-- Copy STORAGE_AWS_IAM_USER_ARN and STORAGE_AWS_EXTERNAL_ID for AWS IAM policy

-- Create external stage using integration
CREATE OR REPLACE STAGE app_logs_stage
    URL = 's3://my-app-logs/logs/application/'
    STORAGE_INTEGRATION = s3_app_logs_integration
    FILE_FORMAT = (
        TYPE = JSON
        COMPRESSION = AUTO
        STRIP_OUTER_ARRAY = TRUE
        STRIP_NULL_VALUES = TRUE
    );
```

**AWS IAM Trust Policy (configure in AWS Console):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789:user/snowflake-iam-user"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "SNOWFLAKE_EXTERNAL_ID_FROM_DESC"
        }
      }
    }
  ]
}
```

### Step 3: Target Table and Snowpipe Creation

**Create Target Table for JSON Logs:**
```sql
-- Application log table (semi-structured + flattened columns)
CREATE OR REPLACE TABLE application_logs (
    log_id NUMBER AUTOINCREMENT PRIMARY KEY,
    ingested_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    filename VARCHAR(500),
    
    -- Flattened JSON fields
    timestamp TIMESTAMP_NTZ,
    log_level VARCHAR(20),
    service_name VARCHAR(100),
    user_id VARCHAR(50),
    request_id VARCHAR(100),
    endpoint VARCHAR(200),
    response_time_ms NUMBER,
    status_code NUMBER,
    
    -- Original JSON (for unstructured fields)
    raw_log VARIANT,
    
    -- Optional metadata
    error_message VARCHAR(1000),
    stack_trace VARCHAR(5000)
) CLUSTER BY (timestamp, service_name);
```

**Create Snowpipe with SQS Notification:**
```sql
-- SYNTAX: CREATE PIPE (complete parameters)
CREATE OR REPLACE PIPE app_logs_pipe
    AUTO_INGEST = TRUE                    -- Enable SQS auto-ingestion
    AWS_SNS_TOPIC = 'arn:aws:sqs:us-east-1:123456789:snowpipe-s3-events'
    INTEGRATION = 'NOTIFICATION_INTEGRATION_NAME'  -- Optional: For SNS instead of SQS
    ERROR_INTEGRATION = 'ERROR_NOTIFICATION_INTEGRATION'  -- Optional: Error alerts
    COMMENT = 'Auto-ingest application logs from S3'
AS
COPY INTO application_logs (
    timestamp,
    log_level,
    service_name,
    user_id,
    request_id,
    endpoint,
    response_time_ms,
    status_code,
    raw_log,
    error_message,
    stack_trace
)
FROM (
    SELECT 
        TO_TIMESTAMP($1:timestamp::STRING, 'YYYY-MM-DD HH24:MI:SS'),
        $1:level::STRING,
        $1:service::STRING,
        $1:user_id::STRING,
        $1:request_id::STRING,
        $1:endpoint::STRING,
        $1:response_time::NUMBER,
        $1:status_code::NUMBER,
        $1,  -- Full JSON
        $1:error:message::STRING,
        $1:error:stack_trace::STRING
    FROM @app_logs_stage
)
FILE_FORMAT = (TYPE = JSON)
ON_ERROR = CONTINUE;  -- Log errors but continue processing

-- Retrieve Snowpipe SQS ARN for S3 notification
SHOW PIPES LIKE 'APP_LOGS_PIPE';
SELECT SYSTEM$PIPE_STATUS('APP_LOGS_PIPE');
```

**Grant SQS Permissions (AWS CLI):**
```bash
# Allow S3 to send messages to SQS
aws sqs set-queue-attributes \
    --queue-url https://sqs.us-east-1.amazonaws.com/123456789/snowpipe-s3-events \
    --attributes '{
        "Policy": "{
            \"Version\": \"2012-10-17\",
            \"Statement\": [{
                \"Effect\": \"Allow\",
                \"Principal\": {\"Service\": \"s3.amazonaws.com\"},
                \"Action\": \"sqs:SendMessage\",
                \"Resource\": \"arn:aws:sqs:us-east-1:123456789:snowpipe-s3-events\",
                \"Condition\": {
                    \"ArnLike\": {\"aws:SourceArn\": \"arn:aws:s3:::my-app-logs\"}
                }
            }]
        }"
    }'
```

### Step 4: Monitoring and Validation

**Monitor Snowpipe Execution:**
```sql
-- Check Snowpipe load history
SELECT 
    pipe_name,
    file_name,
    row_count,
    row_parsed,
    first_error_message,
    first_error_line_number,
    last_load_time,
    status
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'APPLICATION_LOGS',
    START_TIME => DATEADD(hour, -24, CURRENT_TIMESTAMP())
))
WHERE pipe_name = 'APP_LOGS_PIPE'
ORDER BY last_load_time DESC;

-- Check Snowpipe status
SELECT SYSTEM$PIPE_STATUS('APP_LOGS_PIPE');

-- Manually refresh Snowpipe (if AUTO_INGEST temporarily disabled)
ALTER PIPE app_logs_pipe REFRESH;

-- Pause/Resume Snowpipe
ALTER PIPE app_logs_pipe SET PIPE_EXECUTION_PAUSED = TRUE;
ALTER PIPE app_logs_pipe SET PIPE_EXECUTION_PAUSED = FALSE;
```

**Query Ingested Logs:**
```sql
-- Recent error logs
SELECT 
    timestamp,
    service_name,
    endpoint,
    status_code,
    error_message
FROM application_logs
WHERE log_level = 'ERROR'
    AND timestamp >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
ORDER BY timestamp DESC;

-- Endpoint performance metrics
SELECT 
    endpoint,
    COUNT(*) AS request_count,
    AVG(response_time_ms) AS avg_response_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time_ms) AS p95_response_ms,
    SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) AS error_count
FROM application_logs
WHERE timestamp >= DATEADD(day, -1, CURRENT_TIMESTAMP())
GROUP BY endpoint
ORDER BY error_count DESC, avg_response_ms DESC;
```

**SDLC Use Case: CI/CD Log Monitoring Dashboard**
```sql
-- Create materialized view for real-time dashboard
CREATE OR REPLACE MATERIALIZED VIEW mv_service_health AS
SELECT 
    DATE_TRUNC('minute', timestamp) AS minute,
    service_name,
    COUNT(*) AS total_requests,
    SUM(CASE WHEN status_code >= 200 AND status_code < 300 THEN 1 ELSE 0 END) AS success_count,
    SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) AS error_count,
    AVG(response_time_ms) AS avg_response_ms
FROM application_logs
WHERE timestamp >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
GROUP BY DATE_TRUNC('minute', timestamp), service_name;

-- Refresh every 5 minutes
ALTER MATERIALIZED VIEW mv_service_health SET AUTO_REFRESH = TRUE;

-- Grafana query for dashboard
SELECT 
    minute,
    service_name,
    (success_count * 100.0 / NULLIF(total_requests, 0)) AS success_rate,
    error_count,
    avg_response_ms
FROM mv_service_health
WHERE minute >= DATEADD(hour, -4, CURRENT_TIMESTAMP())
ORDER BY minute DESC;
```

---

## 10.3 Advanced Analytics Queries

### Window Functions for Time-Series Analysis

**Scenario: Customer Cohort Retention Analysis**
```sql
-- Calculate monthly cohort retention rates
WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', first_purchase_date) AS cohort_month
    FROM (
        SELECT 
            customer_id AS user_id,
            MIN(order_date) AS first_purchase_date
        FROM orders
        GROUP BY customer_id
    )
),
monthly_activity AS (
    SELECT DISTINCT
        o.customer_id AS user_id,
        DATE_TRUNC('month', o.order_date) AS activity_month
    FROM orders o
),
cohort_activity AS (
    SELECT 
        uc.cohort_month,
        ma.activity_month,
        DATEDIFF(month, uc.cohort_month, ma.activity_month) AS months_since_cohort,
        COUNT(DISTINCT ma.user_id) AS active_users
    FROM user_cohorts uc
    JOIN monthly_activity ma ON uc.user_id = ma.user_id
    GROUP BY uc.cohort_month, ma.activity_month
),
cohort_sizes AS (
    SELECT 
        cohort_month,
        COUNT(DISTINCT user_id) AS cohort_size
    FROM user_cohorts
    GROUP BY cohort_month
)
SELECT 
    ca.cohort_month,
    cs.cohort_size AS month_0_users,
    ca.months_since_cohort,
    ca.active_users,
    ROUND(ca.active_users * 100.0 / cs.cohort_size, 2) AS retention_rate
FROM cohort_activity ca
JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
WHERE ca.cohort_month >= '2024-01-01'
ORDER BY ca.cohort_month, ca.months_since_cohort;
```

### Complex JSON Parsing with FLATTEN

**Scenario: Parse Nested JSON Event Stream**
```sql
-- Sample JSON structure:
-- {
--   "event_id": "evt_123",
--   "user_id": "usr_456",
--   "items": [
--     {"product_id": "P001", "quantity": 2, "price": 29.99},
--     {"product_id": "P002", "quantity": 1, "price": 49.99}
--   ]
-- }

SELECT 
    raw_data:event_id::STRING AS event_id,
    raw_data:user_id::STRING AS user_id,
    item.value:product_id::STRING AS product_id,
    item.value:quantity::NUMBER AS quantity,
    item.value:price::FLOAT AS price,
    item.value:quantity::NUMBER * item.value:price::FLOAT AS line_total
FROM events,
LATERAL FLATTEN(input => raw_data:items) AS item
WHERE event_date >= CURRENT_DATE() - 7;

-- Aggregate sales by product
WITH flattened_events AS (
    SELECT 
        raw_data:event_id::STRING AS event_id,
        item.value:product_id::STRING AS product_id,
        item.value:quantity::NUMBER AS quantity,
        item.value:price::FLOAT AS price
    FROM events,
    LATERAL FLATTEN(input => raw_data:items) AS item
)
SELECT 
    product_id,
    SUM(quantity) AS total_quantity_sold,
    SUM(quantity * price) AS total_revenue,
    COUNT(DISTINCT event_id) AS unique_transactions
FROM flattened_events
GROUP BY product_id
ORDER BY total_revenue DESC;
```

### Performance Optimization Example

**Before Optimization (Slow Query):**
```sql
-- Unoptimized: Full table scan, no clustering
SELECT 
    customer_id,
    SUM(order_amount) AS total_spent
FROM orders  -- 100M rows, not clustered
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY customer_id;

-- Query Profile shows:
-- - Partitions scanned: 12,543
-- - Bytes scanned: 45 GB
-- - Execution time: 18 seconds
```

**After Optimization:**
```sql
-- 1. Add clustering key on frequently filtered column
ALTER TABLE orders CLUSTER BY (order_date);

-- 2. Wait for automatic clustering (or force it)
-- SELECT SYSTEM$CLUSTERING_INFORMATION('orders', '(order_date)');

-- 3. Re-run same query
SELECT 
    customer_id,
    SUM(order_amount) AS total_spent
FROM orders  -- Now clustered by order_date
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY customer_id;

-- Query Profile shows:
-- - Partitions scanned: 1,342 (89% reduction via pruning)
-- - Bytes scanned: 5.2 GB (88% reduction)
-- - Execution time: 2.1 seconds (88% faster)

-- 3. Further optimize with result caching
-- Second execution returns instantly from cache (< 100ms)
```

---

## 10.4 Agile Sprint Demo: E-commerce Analytics Platform

### Sprint Goal

Deliver production-ready e-commerce analytics with real-time order tracking, customer segmentation, and executive dashboards. Complete SDLC workflow: requirements → schema design → ETL → queries → CI/CD deployment → monitoring.

**Sprint Deliverables (2-week sprint):**
1. Star schema for orders, customers, products
2. Snowpipe for real-time order ingestion
3. dbt models for business logic transformations
4. Materialized views for dashboard queries
5. GitHub Actions CI/CD pipeline
6. Query monitoring and alerting

### Day 1-2: Schema Design and DDL

**Dimension Tables:**
```sql
-- Dimension: Date
CREATE OR REPLACE TABLE dim_date (
    date_key NUMBER PRIMARY KEY,
    calendar_date DATE UNIQUE NOT NULL,
    year NUMBER,
    quarter NUMBER,
    month NUMBER,
    month_name VARCHAR(10),
    day_of_month NUMBER,
    day_of_week NUMBER,
    day_name VARCHAR(10),
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    fiscal_year NUMBER,
    fiscal_quarter NUMBER
) CLUSTER BY (date_key);

-- Dimension: Customer (SCD Type 2)
CREATE OR REPLACE TABLE dim_customer (
    customer_key NUMBER AUTOINCREMENT PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    email VARCHAR(100),
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    segment VARCHAR(50),  -- Bronze, Silver, Gold, Platinum
    lifetime_value DECIMAL(10,2),
    acquisition_channel VARCHAR(50),
    effective_date DATE NOT NULL,
    end_date DATE,
    is_current BOOLEAN,
    CONSTRAINT uk_customer UNIQUE (customer_id, effective_date)
) CLUSTER BY (customer_key);

-- Dimension: Product
CREATE OR REPLACE TABLE dim_product (
    product_key NUMBER PRIMARY KEY,
    product_id VARCHAR(50) UNIQUE NOT NULL,
    product_name VARCHAR(200),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    unit_cost DECIMAL(10,2),
    unit_price DECIMAL(10,2),
    is_active BOOLEAN
) CLUSTER BY (product_key);

-- Fact Table: Orders
CREATE OR REPLACE TABLE fact_orders (
    order_key NUMBER AUTOINCREMENT PRIMARY KEY,
    order_id VARCHAR(50) UNIQUE NOT NULL,
    order_date_key NUMBER NOT NULL,
    customer_key NUMBER NOT NULL,
    product_key NUMBER NOT NULL,
    
    -- Measures
    quantity NUMBER,
    unit_price DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    tax_amount DECIMAL(10,2),
    shipping_amount DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    
    -- Metadata
    order_status VARCHAR(20),
    payment_method VARCHAR(50),
    shipping_method VARCHAR(50),
    order_timestamp TIMESTAMP_NTZ,
    
    CONSTRAINT fk_order_date FOREIGN KEY (order_date_key) REFERENCES dim_date(date_key),
    CONSTRAINT fk_customer FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),
    CONSTRAINT fk_product FOREIGN KEY (product_key) REFERENCES dim_product(product_key)
) CLUSTER BY (order_date_key, customer_key);
```

### Day 3-5: dbt Transformations

**dbt Model: Customer Lifetime Value**
```sql
-- models/marts/finance/customer_lifetime_value.sql
{{
  config(
    materialized='incremental',
    unique_key='customer_id',
    cluster_by=['segment'],
    tags=['finance', 'customer']
  )
}}

WITH customer_orders AS (
    SELECT 
        c.customer_id,
        c.email,
        c.segment,
        COUNT(f.order_id) AS total_orders,
        SUM(f.total_amount) AS lifetime_revenue,
        MIN(f.order_timestamp) AS first_order_date,
        MAX(f.order_timestamp) AS last_order_date,
        DATEDIFF(day, MIN(f.order_timestamp), MAX(f.order_timestamp)) AS customer_tenure_days
    FROM {{ ref('dim_customer') }} c
    JOIN {{ ref('fact_orders') }} f ON c.customer_key = f.customer_key
    WHERE c.is_current = TRUE
    
    {% if is_incremental() %}
        AND f.order_timestamp > (SELECT MAX(last_updated) FROM {{ this }})
    {% endif %}
    
    GROUP BY c.customer_id, c.email, c.segment
),
rfm_score AS (
    SELECT 
        customer_id,
        NTILE(5) OVER (ORDER BY last_order_date DESC) AS recency_score,
        NTILE(5) OVER (ORDER BY total_orders) AS frequency_score,
        NTILE(5) OVER (ORDER BY lifetime_revenue) AS monetary_score
    FROM customer_orders
)
SELECT 
    co.customer_id,
    co.email,
    co.segment,
    co.total_orders,
    co.lifetime_revenue,
    co.first_order_date,
    co.last_order_date,
    co.customer_tenure_days,
    rfm.recency_score,
    rfm.frequency_score,
    rfm.monetary_score,
    (rfm.recency_score + rfm.frequency_score + rfm.monetary_score) / 3.0 AS avg_rfm_score,
    CURRENT_TIMESTAMP() AS last_updated
FROM customer_orders co
JOIN rfm_score rfm ON co.customer_id = rfm.customer_id
```

### Day 6-8: CI/CD Pipeline

**GitHub Actions Workflow:**
```yaml
# .github/workflows/snowflake_deploy.yml
name: Snowflake CI/CD Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
  SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
  SNOWFLAKE_PRIVATE_KEY: ${{ secrets.SNOWFLAKE_PRIVATE_KEY }}

jobs:
  validate_sql:
    name: Validate SQL Syntax
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install sqlfluff
        run: pip install sqlfluff
      
      - name: Lint SQL files
        run: sqlfluff lint models/ --dialect snowflake
  
  dbt_test_dev:
    name: dbt Test (Dev Environment)
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    needs: validate_sql
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup dbt
        run: pip install dbt-snowflake==1.7.0
      
      - name: dbt deps
        run: dbt deps
      
      - name: dbt build (dev)
        run: dbt build --target dev --select state:modified+
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ env.SNOWFLAKE_ACCOUNT }}
          DBT_DEV_PASSWORD: ${{ secrets.DBT_DEV_PASSWORD }}
      
      - name: dbt test
        run: dbt test --target dev
  
  dbt_deploy_prod:
    name: dbt Deploy (Production)
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: validate_sql
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup dbt
        run: pip install dbt-snowflake==1.7.0
      
      - name: dbt deps
        run: dbt deps
      
      - name: dbt build (production)
        run: dbt build --target prod
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ env.SNOWFLAKE_ACCOUNT }}
          DBT_OAUTH_TOKEN: ${{ secrets.DBT_OAUTH_TOKEN }}
      
      - name: Notify Slack
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-Type: application/json' \
            -d '{"text":"✅ Production deployment successful"}'
```

### Day 9-10: Dashboard Queries and Materialized Views

**Materialized View: Executive Dashboard Metrics**
```sql
-- Materialized view for real-time executive dashboard
CREATE OR REPLACE MATERIALIZED VIEW mv_executive_metrics AS
SELECT 
    d.calendar_date,
    d.year,
    d.month_name,
    COUNT(DISTINCT f.order_id) AS total_orders,
    COUNT(DISTINCT f.customer_key) AS unique_customers,
    SUM(f.total_amount) AS daily_revenue,
    AVG(f.total_amount) AS avg_order_value,
    SUM(f.quantity) AS units_sold,
    
    -- Year-over-year comparison
    LAG(SUM(f.total_amount), 365) OVER (ORDER BY d.calendar_date) AS revenue_prior_year,
    (SUM(f.total_amount) - LAG(SUM(f.total_amount), 365) OVER (ORDER BY d.calendar_date)) 
        / NULLIF(LAG(SUM(f.total_amount), 365) OVER (ORDER BY d.calendar_date), 0) * 100 AS yoy_growth_pct
FROM fact_orders f
JOIN dim_date d ON f.order_date_key = d.date_key
WHERE d.calendar_date >= DATEADD(year, -2, CURRENT_DATE())
GROUP BY d.calendar_date, d.year, d.month_name;

-- Enable auto-refresh every hour
ALTER MATERIALIZED VIEW mv_executive_metrics SET AUTO_REFRESH = TRUE;
```

**Dashboard Query: Product Performance**
```sql
-- Top 10 products by revenue (last 30 days)
SELECT 
    p.product_name,
    p.category,
    p.brand,
    COUNT(f.order_id) AS order_count,
    SUM(f.quantity) AS units_sold,
    SUM(f.total_amount) AS total_revenue,
    AVG(f.total_amount) AS avg_sale_price,
    SUM(f.total_amount - (p.unit_cost * f.quantity)) AS gross_profit
FROM fact_orders f
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_date d ON f.order_date_key = d.date_key
WHERE d.calendar_date >= CURRENT_DATE() - 30
GROUP BY p.product_name, p.category, p.brand
ORDER BY total_revenue DESC
LIMIT 10;
```

### Sprint Review: Key Metrics Delivered

**Final Deliverables:**
1. ✅ 4 dimension tables + 1 fact table (star schema)
2. ✅ Snowpipe for real-time order ingestion (< 1 min latency)
3. ✅ 5 dbt models with incremental loads
4. ✅ 2 materialized views for dashboards (auto-refresh hourly)
5. ✅ CI/CD pipeline with automated testing
6. ✅ Query performance: 95% under 3 seconds

**Performance Benchmarks:**
```sql
-- Sprint performance validation query
SELECT 
    'Fact Table Row Count' AS metric,
    COUNT(*) AS value
FROM fact_orders

UNION ALL

SELECT 
    'Avg Query Time (Dashboard Queries)',
    AVG(total_elapsed_time) / 1000.0
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_tag = 'executive_dashboard'
    AND start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())

UNION ALL

SELECT 
    'Snowpipe Files Processed (24h)',
    COUNT(*)
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'FACT_ORDERS',
    START_TIME => DATEADD(day, -1, CURRENT_TIMESTAMP())
))
WHERE pipe_name = 'ORDER_INGESTION_PIPE';
```

---

## Glossary of Real-world Implementation Terms

- **Schema Conversion**: Translating DDL from source database to Snowflake syntax
- **Storage Integration**: Snowflake object enabling secure access to external cloud storage
- **Snowpipe**: Auto-ingestion service triggered by cloud storage events (S3, Azure, GCP)
- **SQS (Simple Queue Service)**: AWS message queue service for event-driven architectures
- **SCD Type 2**: Slowly Changing Dimension with full history tracking via effective dates
- **Zero-Copy Clone**: Instant data copy using metadata pointers, no storage duplication
- **Materialized View**: Pre-computed query results automatically refreshed
- **dbt Model**: SQL SELECT statement templated and orchestrated by dbt framework
- **Incremental Model**: dbt pattern loading only new/changed data since last run
- **CI/CD Pipeline**: Automated testing and deployment workflow (GitHub Actions, GitLab CI)
- **RFM Score**: Recency, Frequency, Monetary customer segmentation metric
- **FLATTEN**: Snowflake function expanding nested/array JSON into tabular rows
- **LATERAL JOIN**: SQL pattern for correlated subqueries with FLATTEN
- **Clustering Key**: Column(s) optimizing micro-partition organization for query pruning
- **Query Profile**: Visual execution plan showing bottlenecks and optimization opportunities
- **Result Caching**: Snowflake feature reusing identical query results (24-hour TTL)
- **Auto-Refresh**: Materialized view setting for automatic incremental updates
- **COPY INTO**: Bulk data loading command from stages to tables
- **ON_ERROR**: COPY INTO parameter controlling error handling (CONTINUE, ABORT, SKIP_FILE)
- **FILE_FORMAT**: Structured definition of file type, delimiters, compression, encoding
- **VALIDATION_MODE**: COPY INTO dry-run mode returning errors without loading data
- **PURGE**: COPY INTO flag deleting source files after successful load
- **Cohort Analysis**: Grouping users by shared attributes (e.g., signup month) for retention tracking
- **Window Function**: SQL analytic function computing across row partitions (LAG, NTILE, etc.)
- **Star Schema**: Denormalized data model with central fact table and surrounding dimensions
- **NTILE**: Window function dividing rows into equal buckets (e.g., quintiles for RFM scoring)
- **Query Tag**: User-defined label for cost attribution and query grouping
- **Sqlfluff**: SQL linter for syntax validation and style enforcement
- **State-based Selection**: dbt feature running only modified models and downstream dependencies
- **Failover**: Disaster recovery process promoting replica database to primary
