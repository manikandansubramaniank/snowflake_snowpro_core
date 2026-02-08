# Snowflake SQL Complete Cheatsheet

> **Comprehensive Reference Guide for Data Engineers and Analysts**  
> Last Updated: February 2026 | Snowflake Version: Latest

## Introduction

This cheatsheet provides complete coverage of Snowflake SQL syntax, commands, and best practices. It serves as a quick reference for both beginners and experienced practitioners, covering everything from basic DDL operations to advanced query optimization techniques.

**Who This Is For:**
- Data Engineers building data pipelines
- Data Analysts writing complex queries
- Database Administrators managing Snowflake environments
- Developers integrating applications with Snowflake
- Anyone preparing for SnowPro certification

**How to Use This Guide:**
- Use the table of contents to jump to specific topics
- Each section includes syntax, parameters, examples, and best practices
- Code examples are production-ready and can be adapted to your needs
- Best practices are based on real-world implementations

---

## Table of Contents

### [1. Database Objects](#1-database-objects)
- [1.1 Databases](#11-databases)
- [1.2 Schemas](#12-schemas)
- [1.3 Tables](#13-tables)
- [1.4 Views](#14-views)
- [1.5 Sequences](#15-sequences)
- [1.6 File Formats](#16-file-formats)
- [1.7 Stages](#17-stages)
- [1.8 Pipes](#18-pipes)
- [1.9 Streams](#19-streams)
- [1.10 Tasks](#110-tasks)

### [2. Data Types](#2-data-types)
- [2.1 Numeric Types](#21-numeric-types)
- [2.2 String Types](#22-string-types)
- [2.3 Date and Time Types](#23-date-and-time-types)
- [2.4 Semi-Structured Types](#24-semi-structured-types)
- [2.5 Geospatial Types](#25-geospatial-types)

### [3. SQL Commands](#3-sql-commands)
- [3.1 DDL Commands](#31-ddl-commands)
- [3.2 DML Commands](#32-dml-commands)
- [3.3 DQL Commands](#33-dql-commands)
- [3.4 DCL Commands](#34-dcl-commands)
- [3.5 TCL Commands](#35-tcl-commands)

### [4. Functions](#4-functions)
- [4.1 Aggregate Functions](#41-aggregate-functions)
- [4.2 String Functions](#42-string-functions)
- [4.3 Date/Time Functions](#43-datetime-functions)
- [4.4 Numeric Functions](#44-numeric-functions)
- [4.5 Conditional Functions](#45-conditional-functions)
- [4.6 Window Functions](#46-window-functions)
- [4.7 Semi-Structured Functions](#47-semi-structured-functions)
- [4.8 Table Functions](#48-table-functions)

### [5. Snowflake Features](#5-snowflake-features)
- [5.1 Time Travel](#51-time-travel)
- [5.2 Zero-Copy Cloning](#52-zero-copy-cloning)
- [5.3 Data Sharing](#53-data-sharing)
- [5.4 Query Optimization](#54-query-optimization)
- [5.5 Virtual Warehouses](#55-virtual-warehouses)
- [5.6 Security](#56-security)

### [6. Best Practices](#6-best-practices)
### [7. Quick Reference Tables](#7-quick-reference-tables)

---

## 1. Database Objects

### 1.1 Databases

Databases are the top-level containers in Snowflake's three-tier architecture.

#### Basic Syntax

```sql
-- Create database
CREATE [ OR REPLACE ] [ TRANSIENT ] DATABASE [ IF NOT EXISTS ] database_name
    [ DATA_RETENTION_TIME_IN_DAYS = <integer> ]
    [ COMMENT = '<string>' ];

-- Examples
CREATE DATABASE sales_db;
CREATE DATABASE analytics_db DATA_RETENTION_TIME_IN_DAYS = 7;
CREATE TRANSIENT DATABASE staging_db DATA_RETENTION_TIME_IN_DAYS = 1;

-- Clone database with Time Travel
CREATE DATABASE sales_backup 
CLONE sales_db AT (TIMESTAMP => '2026-02-01 12:00:00'::TIMESTAMP);
```

#### Management Commands

```sql
-- Show and describe
SHOW DATABASES;
DESCRIBE DATABASE sales_db;

-- Alter database
ALTER DATABASE sales_db SET DATA_RETENTION_TIME_IN_DAYS = 14;
ALTER DATABASE old_name RENAME TO new_name;

-- Drop and undrop
DROP DATABASE sales_db;
UNDROP DATABASE sales_db;

-- Set context
USE DATABASE sales_db;
```

---

### 1.2 Schemas

Schemas organize database objects within a database.

```sql
-- Create schema
CREATE SCHEMA sales_db.customers 
    WITH MANAGED ACCESS  -- Enhanced security
    DATA_RETENTION_TIME_IN_DAYS = 7;

-- Create transient schema
CREATE TRANSIENT SCHEMA sales_db.staging;

-- Management
SHOW SCHEMAS IN DATABASE sales_db;
ALTER SCHEMA sales_db.customers SET DATA_RETENTION_TIME_IN_DAYS = 14;
DROP SCHEMA sales_db.customers;
```

---

### 1.3 Tables

#### Permanent Tables

```sql
-- Basic table
CREATE TABLE customers (
    customer_id NUMBER(38,0) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    PRIMARY KEY (customer_id)
);

-- Table with clustering
CREATE TABLE sales (
    sale_id NUMBER,
    sale_date DATE,
    customer_id NUMBER,
    amount DECIMAL(10,2)
) CLUSTER BY (sale_date);

-- Table with auto-increment
CREATE TABLE orders (
    order_id NUMBER AUTOINCREMENT START 1000 INCREMENT 1,
    customer_id NUMBER,
    order_date DATE,
    total_amount DECIMAL(10,2)
);

-- CTAS (Create Table As Select)
CREATE TABLE high_value_customers AS
SELECT customer_id, SUM(amount) as total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 10000;
```

#### Temporary Tables

```sql
-- Session-scoped temporary table
CREATE TEMPORARY TABLE session_data (
    user_id NUMBER,
    session_id VARCHAR(100)
);

-- Temporary table from query
CREATE TEMPORARY TABLE temp_results AS
SELECT * FROM large_table WHERE date = CURRENT_DATE();
```

#### Transient Tables

```sql
-- No fail-safe, lower storage costs
CREATE TRANSIENT TABLE staging_orders (
    order_id NUMBER,
    amount DECIMAL(10,2),
    loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
) DATA_RETENTION_TIME_IN_DAYS = 0;
```

#### External Tables

```sql
-- External table on S3
CREATE EXTERNAL TABLE ext_customer_data (
    customer_id NUMBER AS (value:c1::NUMBER),
    first_name VARCHAR AS (value:c2::VARCHAR),
    filename VARCHAR AS (metadata$filename::VARCHAR)
)
LOCATION = @my_s3_stage/customer_data/
FILE_FORMAT = (TYPE = JSON)
AUTO_REFRESH = TRUE;

-- With partitioning
CREATE EXTERNAL TABLE ext_sales (
    sale_id NUMBER AS (value:sale_id::NUMBER),
    amount DECIMAL AS (value:amount::DECIMAL(10,2)),
    sale_date DATE AS (to_date(metadata$filename, 'sales/year=YYYY/month=MM/DD.parquet'))
)
PARTITION BY (sale_date)
LOCATION = @s3_stage/sales/
FILE_FORMAT = (TYPE = PARQUET)
AUTO_REFRESH = TRUE;
```

#### Dynamic Tables

```sql
-- Declarative materialization
CREATE DYNAMIC TABLE customer_metrics
    TARGET_LAG = '1 hour'
    WAREHOUSE = compute_wh
    REFRESH_MODE = AUTO
AS
SELECT 
    customer_id,
    COUNT(*) as order_count,
    SUM(amount) as total_spent
FROM orders
GROUP BY customer_id;

-- Incremental refresh
CREATE DYNAMIC TABLE recent_sales
    TARGET_LAG = '5 minutes'
    WAREHOUSE = realtime_wh
    REFRESH_MODE = INCREMENTAL
AS
SELECT * FROM sales 
WHERE sale_date > DATEADD(day, -7, CURRENT_TIMESTAMP());
```

---

### 1.4 Views

#### Standard Views

```sql
-- Basic view
CREATE VIEW active_customers AS
SELECT customer_id, first_name, last_name, email
FROM customers
WHERE status = 'ACTIVE';

-- Complex view
CREATE VIEW customer_lifetime_value AS
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name as name,
    COUNT(o.order_id) as total_orders,
    SUM(o.amount) as lifetime_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name;
```

#### Materialized Views

```sql
-- Pre-computed results
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
    DATE_TRUNC('day', order_date) as sale_date,
    product_id,
    SUM(amount) as total_amount,
    COUNT(*) as order_count
FROM orders
GROUP BY DATE_TRUNC('day', order_date), product_id;

-- With clustering
CREATE MATERIALIZED VIEW mv_customer_summary
    CLUSTER BY (customer_id)
AS
SELECT customer_id, COUNT(*), SUM(amount)
FROM orders
GROUP BY customer_id;
```

#### Secure Views

```sql
-- Hides definition for security/sharing
CREATE SECURE VIEW secure_customer_data AS
SELECT 
    customer_id,
    first_name,
    last_name,
    CASE 
        WHEN CURRENT_ROLE() = 'ADMIN' THEN email
        ELSE '***MASKED***'
    END as email
FROM customers;
```

---

### 1.5 Sequences

```sql
-- Create sequence
CREATE SEQUENCE seq_customer_id START = 1000 INCREMENT = 1;

-- Use in table
CREATE TABLE customers (
    customer_id NUMBER DEFAULT seq_customer_id.NEXTVAL,
    name VARCHAR(100)
);

-- Use in INSERT
INSERT INTO customers (customer_id, name)
VALUES (seq_customer_id.NEXTVAL, 'John Doe');

-- Get current value
SELECT seq_customer_id.CURRVAL;

-- Management
ALTER SEQUENCE seq_customer_id SET INCREMENT = 10;
DROP SEQUENCE seq_customer_id;
```

---

### 1.6 File Formats

```sql
-- CSV format
CREATE FILE FORMAT csv_format
    TYPE = CSV
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    NULL_IF = ('NULL', 'null', '')
    COMPRESSION = AUTO;

-- JSON format
CREATE FILE FORMAT json_format
    TYPE = JSON
    STRIP_OUTER_ARRAY = TRUE
    COMPRESSION = GZIP;

-- Parquet format
CREATE FILE FORMAT parquet_format
    TYPE = PARQUET
    COMPRESSION = SNAPPY;

-- Use in COPY
COPY INTO table_name
FROM @stage/file.csv
FILE_FORMAT = (FORMAT_NAME = 'csv_format');
```

---

### 1.7 Stages

#### Internal Stages

```sql
-- Named internal stage
CREATE STAGE my_stage
    FILE_FORMAT = (TYPE = CSV);

-- User stage (automatic): @~
-- Table stage (automatic): @%table_name
```

#### External Stages

```sql
-- S3 stage with integration
CREATE STAGE s3_stage
    URL = 's3://my-bucket/data/'
    STORAGE_INTEGRATION = aws_integration
    FILE_FORMAT = (TYPE = CSV);

-- Azure stage
CREATE STAGE azure_stage
    URL = 'azure://account.blob.core.windows.net/container/path/'
    STORAGE_INTEGRATION = azure_integration;

-- GCS stage
CREATE STAGE gcs_stage
    URL = 'gcs://my-bucket/data/'
    STORAGE_INTEGRATION = gcs_integration;

-- List files
LIST @my_stage;
LIST @my_stage PATTERN = '.*\.csv';

-- Remove files
REMOVE @my_stage/old_file.csv;
```

---

### 1.8 Pipes

```sql
-- Auto-ingest pipe for S3
CREATE PIPE auto_pipe
    AUTO_INGEST = TRUE
    AWS_SNS_TOPIC = 'arn:aws:sns:region:account:topic'
AS
COPY INTO target_table
FROM @s3_stage/
FILE_FORMAT = (TYPE = JSON);

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('auto_pipe');

-- Pause/resume pipe
ALTER PIPE auto_pipe SET PIPE_EXECUTION_PAUSED = TRUE;
ALTER PIPE auto_pipe SET PIPE_EXECUTION_PAUSED = FALSE;

-- Manual refresh
ALTER PIPE auto_pipe REFRESH;

-- Pipe usage history
SELECT * FROM TABLE(
    INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
        PIPE_NAME => 'AUTO_PIPE'
    )
);
```

---

### 1.9 Streams

```sql
-- Create stream for CDC
CREATE STREAM customer_stream ON TABLE customers;

-- Append-only stream
CREATE STREAM sales_stream 
ON TABLE sales
APPEND_ONLY = TRUE;

-- Query stream
SELECT 
    *,
    METADATA$ACTION,
    METADATA$ISUPDATE,
    METADATA$ROW_ID
FROM customer_stream;

-- Consume stream
INSERT INTO customer_history
SELECT * FROM customer_stream;

-- Check if stream has data
SELECT SYSTEM$STREAM_HAS_DATA('customer_stream');
```

---

### 1.10 Tasks

```sql
-- Scheduled task
CREATE TASK daily_aggregation
    WAREHOUSE = compute_wh
    SCHEDULE = 'USING CRON 0 2 * * * UTC'
AS
INSERT INTO daily_metrics
SELECT CURRENT_DATE() - 1, SUM(amount)
FROM sales
WHERE sale_date = CURRENT_DATE() - 1;

-- Conditional task (stream-triggered)
CREATE TASK process_changes
    WAREHOUSE = compute_wh
    SCHEDULE = '5 MINUTE'
    WHEN SYSTEM$STREAM_HAS_DATA('customer_stream')
AS
INSERT INTO customer_history
SELECT * FROM customer_stream;

-- Task DAG (dependencies)
CREATE TASK root_task
    WAREHOUSE = compute_wh
    SCHEDULE = '1 HOUR'
AS
    CALL extract_data();

CREATE TASK child_task
    WAREHOUSE = compute_wh
    AFTER root_task
AS
    CALL transform_data();

-- Resume tasks (resume children first, then root)
ALTER TASK child_task RESUME;
ALTER TASK root_task RESUME;

-- Task history
SELECT * FROM TABLE(
    INFORMATION_SCHEMA.TASK_HISTORY(
        TASK_NAME => 'DAILY_AGGREGATION'
    )
);
```

---

## 2. Data Types

### 2.1 Numeric Types

```sql
-- Fixed-point
NUMBER(38,0)          -- Default precision: 38 digits
NUMBER(10,2)          -- 10 digits total, 2 after decimal
DECIMAL(10,2)         -- Synonym for NUMBER
NUMERIC(10,2)         -- Synonym for NUMBER

-- Integer types (synonyms for NUMBER)
INT, INTEGER, BIGINT, SMALLINT, TINYINT, BYTEINT

-- Floating-point
FLOAT, FLOAT4, FLOAT8
DOUBLE, DOUBLE PRECISION, REAL

-- Examples
CREATE TABLE products (
    product_id NUMBER(10,0),
    price DECIMAL(10,2),
    quantity INT,
    weight FLOAT
);
```

### 2.2 String Types

```sql
-- Variable length (max 16MB)
VARCHAR(100)          -- Up to 100 characters
STRING               -- Synonym, no limit specified
TEXT                 -- Synonym

-- Fixed length
CHAR(10)             -- Always 10 characters
CHARACTER(10)        -- Synonym

-- Binary
BINARY(100)          -- Fixed binary data
VARBINARY            -- Variable binary data

-- Examples
CREATE TABLE users (
    username VARCHAR(50),
    bio TEXT,
    profile_pic BINARY
);
```

### 2.3 Date and Time Types

```sql
-- Date/Time types
DATE                 -- Date only (YYYY-MM-DD)
TIME                 -- Time only (HH:MI:SS)
DATETIME             -- Alias for TIMESTAMP_NTZ
TIMESTAMP            -- Alias for TIMESTAMP_NTZ

-- Timestamp variants
TIMESTAMP_NTZ        -- No timezone (default)
TIMESTAMP_LTZ        -- Local timezone
TIMESTAMP_TZ         -- With timezone

-- Examples
CREATE TABLE events (
    event_date DATE,
    event_time TIME,
    created_ntz TIMESTAMP_NTZ,
    created_ltz TIMESTAMP_LTZ,
    created_tz TIMESTAMP_TZ
);

INSERT INTO events VALUES (
    CURRENT_DATE(),
    CURRENT_TIME(),
    CURRENT_TIMESTAMP(),
    CURRENT_TIMESTAMP(),
    '2026-02-07 10:30:00 -0800'::TIMESTAMP_TZ
);
```

### 2.4 Semi-Structured Types

```sql
-- VARIANT: stores semi-structured data
VARIANT              -- JSON, Avro, ORC, Parquet, XML

-- OBJECT: key-value pairs
OBJECT               -- JSON object

-- ARRAY: ordered list
ARRAY                -- JSON array

-- Examples
CREATE TABLE json_data (
    id NUMBER,
    data VARIANT,
    metadata OBJECT,
    tags ARRAY
);

INSERT INTO json_data 
SELECT 
    1,
    PARSE_JSON('{"name": "John", "age": 30}'),
    OBJECT_CONSTRUCT('key1', 'value1', 'key2', 'value2'),
    ARRAY_CONSTRUCT('tag1', 'tag2', 'tag3');

-- Query VARIANT
SELECT 
    data:name::STRING,
    data:age::NUMBER
FROM json_data;
```

### 2.5 Geospatial Types

```sql
-- Geospatial types (GeoJSON)
GEOGRAPHY            -- WGS 84 coordinate system
GEOMETRY             -- Planar coordinate system

-- Examples
CREATE TABLE locations (
    location_id NUMBER,
    point GEOGRAPHY,
    polygon GEOGRAPHY
);

INSERT INTO locations VALUES (
    1,
    TO_GEOGRAPHY('POINT(-122.35 37.55)'),
    TO_GEOGRAPHY('POLYGON((-122.4 37.8, -122.4 37.7, -122.3 37.7, -122.3 37.8, -122.4 37.8))')
);

-- Distance calculation
SELECT ST_DISTANCE(
    TO_GEOGRAPHY('POINT(-122.35 37.55)'),
    TO_GEOGRAPHY('POINT(-122.40 37.60)')
);
```

---

## 3. SQL Commands

### 3.1 DDL Commands

#### CREATE

```sql
-- See sections 1.1-1.10 for detailed CREATE syntax
CREATE DATABASE db_name;
CREATE SCHEMA schema_name;
CREATE TABLE table_name (...);
CREATE VIEW view_name AS SELECT ...;
```

#### ALTER

```sql
-- Alter table
ALTER TABLE customers ADD COLUMN phone VARCHAR(20);
ALTER TABLE customers DROP COLUMN phone;
ALTER TABLE customers RENAME COLUMN old_name TO new_name;
ALTER TABLE customers ALTER COLUMN email SET NOT NULL;

-- Alter table clustering
ALTER TABLE sales CLUSTER BY (sale_date, region);
ALTER TABLE sales DROP CLUSTERING KEY;

-- Suspend/resume clustering
ALTER TABLE sales SUSPEND RECLUSTER;
ALTER TABLE sales RESUME RECLUSTER;

-- Alter other objects
ALTER DATABASE db_name SET DATA_RETENTION_TIME_IN_DAYS = 14;
ALTER SCHEMA schema_name RENAME TO new_schema_name;
ALTER VIEW view_name RENAME TO new_view_name;
```

#### DROP

```sql
-- Drop with CASCADE/RESTRICT
DROP TABLE customers;
DROP TABLE IF EXISTS customers;
DROP TABLE customers CASCADE;  -- Drop dependent objects

DROP DATABASE db_name CASCADE;
DROP SCHEMA schema_name CASCADE;
DROP VIEW view_name;
```

#### TRUNCATE

```sql
-- Remove all rows, faster than DELETE
TRUNCATE TABLE staging_table;
TRUNCATE TABLE IF EXISTS temp_data;
```

#### RENAME

```sql
-- Rename objects
ALTER TABLE old_table_name RENAME TO new_table_name;
ALTER VIEW old_view_name RENAME TO new_view_name;
ALTER SCHEMA old_schema RENAME TO new_schema;
```

#### CLONE

```sql
-- Zero-copy clone (instant)
CREATE TABLE customers_backup CLONE customers;

-- Clone at specific time
CREATE TABLE customers_jan CLONE customers 
AT (TIMESTAMP => '2026-01-31 23:59:59'::TIMESTAMP);

-- Clone with offset
CREATE TABLE customers_yesterday CLONE customers 
AT (OFFSET => -86400);  -- 24 hours ago

-- Clone database
CREATE DATABASE sales_dev CLONE sales_prod;

-- Clone schema
CREATE SCHEMA analytics_backup CLONE analytics;
```

#### UNDROP

```sql
-- Restore dropped objects (within retention period)
UNDROP TABLE customers;
UNDROP DATABASE sales_db;
UNDROP SCHEMA analytics_schema;
```

---

### 3.2 DML Commands

#### INSERT

```sql
-- Single row
INSERT INTO customers (customer_id, name, email)
VALUES (1, 'John Doe', 'john@example.com');

-- Multiple rows
INSERT INTO customers VALUES 
    (1, 'John Doe', 'john@example.com'),
    (2, 'Jane Smith', 'jane@example.com'),
    (3, 'Bob Johnson', 'bob@example.com');

-- Insert from SELECT
INSERT INTO customer_archive
SELECT * FROM customers
WHERE created_date < DATEADD(year, -2, CURRENT_DATE());

-- Insert with column list
INSERT INTO orders (order_id, customer_id, amount)
SELECT order_id, customer_id, total FROM staging_orders;

-- OVERWRITE variant (replaces all data)
INSERT OVERWRITE INTO target_table
SELECT * FROM source_table;
```

#### UPDATE

```sql
-- Basic update
UPDATE customers
SET email = 'newemail@example.com'
WHERE customer_id = 1;

-- Update multiple columns
UPDATE customers
SET 
    email = 'updated@example.com',
    updated_at = CURRENT_TIMESTAMP()
WHERE customer_id = 1;

-- Update from another table
UPDATE customers c
SET email = s.new_email
FROM staging_customers s
WHERE c.customer_id = s.customer_id;

-- Conditional update
UPDATE products
SET price = price * 1.1
WHERE category = 'Electronics' AND stock_quantity > 0;
```

#### DELETE

```sql
-- Delete with condition
DELETE FROM customers WHERE customer_id = 1;

-- Delete all rows (use TRUNCATE instead for better performance)
DELETE FROM staging_table;

-- Delete with subquery
DELETE FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE status = 'INACTIVE'
);

-- Delete with join
DELETE FROM order_items
USING orders
WHERE order_items.order_id = orders.order_id
    AND orders.status = 'CANCELLED';
```

#### MERGE

```sql
-- Basic MERGE (UPSERT)
MERGE INTO target_customers t
USING source_customers s
ON t.customer_id = s.customer_id
WHEN MATCHED THEN UPDATE SET
    t.first_name = s.first_name,
    t.last_name = s.last_name,
    t.email = s.email,
    t.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT
    (customer_id, first_name, last_name, email, created_at)
VALUES
    (s.customer_id, s.first_name, s.last_name, s.email, CURRENT_TIMESTAMP());

-- MERGE with DELETE
MERGE INTO inventory i
USING inventory_updates u
ON i.product_id = u.product_id
WHEN MATCHED AND u.quantity = 0 THEN DELETE
WHEN MATCHED THEN UPDATE SET i.quantity = u.quantity
WHEN NOT MATCHED THEN INSERT (product_id, quantity) VALUES (u.product_id, u.quantity);

-- MERGE with conditional update
MERGE INTO sales_fact f
USING sales_staging s
ON f.sale_id = s.sale_id
WHEN MATCHED AND s.amount != f.amount THEN UPDATE SET
    f.amount = s.amount,
    f.updated_flag = TRUE
WHEN NOT MATCHED THEN INSERT
    (sale_id, customer_id, amount, sale_date)
VALUES
    (s.sale_id, s.customer_id, s.amount, s.sale_date);
```

#### COPY INTO

```sql
-- Load from stage
COPY INTO customers
FROM @my_stage/customers.csv
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1);

-- Load with transformations
COPY INTO orders (order_id, customer_id, order_date, amount)
FROM (
    SELECT 
        $1::NUMBER,
        $2::NUMBER,
        TO_DATE($3, 'YYYY-MM-DD'),
        $4::DECIMAL(10,2)
    FROM @my_stage/orders.csv
)
FILE_FORMAT = (TYPE = CSV);

-- Load with pattern matching
COPY INTO sales
FROM @s3_stage/sales/
PATTERN = '.*2026.*\.csv'
FILE_FORMAT = (FORMAT_NAME = 'csv_format');

-- Load with validation
COPY INTO customers
FROM @my_stage/customers.csv
FILE_FORMAT = (TYPE = CSV)
VALIDATION_MODE = RETURN_ERRORS;

-- Load with error handling
COPY INTO customers
FROM @my_stage/customers.csv
FILE_FORMAT = (TYPE = CSV)
ON_ERROR = 'CONTINUE'  -- Options: CONTINUE, SKIP_FILE, ABORT_STATEMENT
RETURN_FAILED_ONLY = TRUE;

-- Unload data (COPY INTO location)
COPY INTO @my_stage/customer_export/
FROM customers
FILE_FORMAT = (TYPE = CSV COMPRESSION = GZIP)
HEADER = TRUE
SINGLE = FALSE  -- Multiple files
MAX_FILE_SIZE = 5368709120;  -- 5GB per file
```

---

### 3.3 DQL Commands

#### SELECT Basics

```sql
-- Basic SELECT
SELECT * FROM customers;
SELECT customer_id, first_name, last_name FROM customers;

-- DISTINCT
SELECT DISTINCT region FROM sales;
SELECT DISTINCT region, country FROM sales;

-- WHERE clause
SELECT * FROM customers WHERE region = 'West';
SELECT * FROM orders WHERE amount > 1000 AND status = 'COMPLETED';

-- LIMIT and OFFSET
SELECT * FROM customers LIMIT 10;
SELECT * FROM customers LIMIT 10 OFFSET 20;  -- Skip first 20

-- ORDER BY
SELECT * FROM customers ORDER BY last_name;
SELECT * FROM sales ORDER BY sale_date DESC, amount DESC;

-- GROUP BY
SELECT region, COUNT(*), SUM(amount)
FROM sales
GROUP BY region;

-- HAVING
SELECT customer_id, SUM(amount) as total
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 10000;

-- SELECT with aliases
SELECT 
    customer_id as id,
    first_name || ' ' || last_name as full_name,
    email
FROM customers;
```

#### JOINs

```sql
-- INNER JOIN
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- LEFT JOIN (LEFT OUTER JOIN)
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;

-- RIGHT JOIN
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;

-- FULL OUTER JOIN
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id;

-- CROSS JOIN
SELECT p.product_name, s.store_name
FROM products p
CROSS JOIN stores s;

-- Self JOIN
SELECT 
    e1.employee_id,
    e1.name as employee_name,
    e2.name as manager_name
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.employee_id;

-- Multiple JOINs
SELECT 
    c.name,
    o.order_id,
    p.product_name,
    oi.quantity
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;

-- LATERAL JOIN
SELECT c.customer_id, c.name, t.recent_orders
FROM customers c,
LATERAL (
    SELECT COUNT(*) as recent_orders
    FROM orders o
    WHERE o.customer_id = c.customer_id
        AND o.order_date >= DATEADD(month, -1, CURRENT_DATE())
) t;
```

#### Subqueries

```sql
-- Scalar subquery
SELECT customer_id, name,
    (SELECT COUNT(*) FROM orders o 
     WHERE o.customer_id = c.customer_id) as order_count
FROM customers c;

-- IN subquery
SELECT * FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM orders WHERE amount > 1000
);

-- EXISTS subquery
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
        AND o.amount > 1000
);

-- NOT EXISTS
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- Correlated subquery
SELECT customer_id, name
FROM customers c
WHERE (
    SELECT SUM(amount) FROM orders o 
    WHERE o.customer_id = c.customer_id
) > 10000;

-- FROM subquery
SELECT customer_id, total_spent
FROM (
    SELECT customer_id, SUM(amount) as total_spent
    FROM orders
    GROUP BY customer_id
) subquery
WHERE total_spent > 5000;
```

#### CTEs (Common Table Expressions)

```sql
-- Basic CTE
WITH customer_totals AS (
    SELECT customer_id, SUM(amount) as total_spent
    FROM orders
    GROUP BY customer_id
)
SELECT c.name, ct.total_spent
FROM customers c
JOIN customer_totals ct ON c.customer_id = ct.customer_id;

-- Multiple CTEs
WITH 
    high_value AS (
        SELECT customer_id FROM orders 
        GROUP BY customer_id 
        HAVING SUM(amount) > 10000
    ),
    recent_orders AS (
        SELECT customer_id, COUNT(*) as order_count
        FROM orders
        WHERE order_date >= DATEADD(month, -3, CURRENT_DATE())
        GROUP BY customer_id
    )
SELECT c.name, r.order_count
FROM customers c
JOIN high_value h ON c.customer_id = h.customer_id
JOIN recent_orders r ON c.customer_id = r.customer_id;

-- Recursive CTE (org hierarchy)
WITH RECURSIVE employee_hierarchy AS (
    -- Anchor: top-level employees
    SELECT employee_id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: employees reporting to current level
    SELECT e.employee_id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level, name;
```

#### QUALIFY Clause

```sql
-- Filter results of window functions
SELECT 
    customer_id,
    order_id,
    amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) as rn
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) = 1;

-- Top 3 per category
SELECT product_name, category, revenue
FROM sales
QUALIFY ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) <= 3;

-- QUALIFY with RANK
SELECT employee_id, department, salary
FROM employees
QUALIFY RANK() OVER (PARTITION BY department ORDER BY salary DESC) <= 3;
```

#### Set Operations

```sql
-- UNION (removes duplicates)
SELECT customer_id FROM customers_2025
UNION
SELECT customer_id FROM customers_2026;

-- UNION ALL (keeps duplicates)
SELECT customer_id FROM customers_2025
UNION ALL
SELECT customer_id FROM customers_2026;

-- INTERSECT
SELECT customer_id FROM customers_west
INTERSECT
SELECT customer_id FROM customers_premium;

-- EXCEPT (MINUS)
SELECT customer_id FROM all_customers
EXCEPT
SELECT customer_id FROM inactive_customers;
```

---

### 3.4 DCL Commands

#### GRANT

```sql
-- Grant object privileges
GRANT SELECT ON TABLE customers TO ROLE analyst_role;
GRANT SELECT, INSERT, UPDATE ON TABLE orders TO ROLE data_engineer;
GRANT ALL ON TABLE products TO ROLE admin_role;

-- Grant on database/schema
GRANT USAGE ON DATABASE sales_db TO ROLE analyst_role;
GRANT USAGE ON SCHEMA sales_db.analytics TO ROLE analyst_role;

-- Grant on warehouse
GRANT USAGE, OPERATE ON WAREHOUSE compute_wh TO ROLE analyst_role;
GRANT MODIFY ON WAREHOUSE compute_wh TO ROLE admin_role;

-- Grant role to user
GRANT ROLE analyst_role TO USER john_doe;

-- Grant role to role (role hierarchy)
GRANT ROLE junior_analyst TO ROLE senior_analyst;

-- Grant with GRANT OPTION
GRANT SELECT ON TABLE customers TO ROLE manager_role WITH GRANT OPTION;

-- Grant future privileges
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales_db.analytics TO ROLE analyst_role;
GRANT USAGE ON FUTURE SCHEMAS IN DATABASE sales_db TO ROLE analyst_role;
```

#### REVOKE

```sql
-- Revoke object privileges
REVOKE SELECT ON TABLE customers FROM ROLE analyst_role;
REVOKE INSERT, UPDATE ON TABLE orders FROM ROLE data_engineer;

-- Revoke on warehouse
REVOKE USAGE ON WAREHOUSE compute_wh FROM ROLE analyst_role;

-- Revoke role from user
REVOKE ROLE analyst_role FROM USER john_doe;

-- Revoke role from role
REVOKE ROLE junior_analyst FROM ROLE senior_analyst;
```

#### Role Management

```sql
-- Create role
CREATE ROLE data_engineer;
CREATE ROLE analyst;

-- Grant role hierarchy
GRANT ROLE analyst TO ROLE senior_analyst;
GRANT ROLE senior_analyst TO ROLE admin;

-- Set default role for user
ALTER USER john_doe SET DEFAULT_ROLE = analyst;

-- Use role
USE ROLE analyst;

-- Show grants
SHOW GRANTS TO ROLE analyst;
SHOW GRANTS ON TABLE customers;
SHOW GRANTS TO USER john_doe;

-- Show roles
SHOW ROLES;

-- Drop role
DROP ROLE analyst;
```

---

### 3.5 TCL Commands

```sql
-- BEGIN TRANSACTION
BEGIN TRANSACTION;
BEGIN;  -- Short form

-- Perform DML operations
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

-- COMMIT
COMMIT;

-- ROLLBACK
BEGIN;
UPDATE customers SET status = 'INACTIVE' WHERE last_order < '2025-01-01';
ROLLBACK;  -- Undo changes

-- Savepoints
BEGIN;
    INSERT INTO orders VALUES (1, 100);
    SAVEPOINT sp1;
    
    INSERT INTO orders VALUES (2, 200);
    SAVEPOINT sp2;
    
    INSERT INTO orders VALUES (3, 300);
    
    ROLLBACK TO SAVEPOINT sp2;  -- Undo last insert only
COMMIT;

-- Autocommit (default is ON in Snowflake)
ALTER SESSION SET AUTOCOMMIT = FALSE;
-- Now need explicit COMMIT
ALTER SESSION SET AUTOCOMMIT = TRUE;
```

---

## 4. Functions

### 4.1 Aggregate Functions

```sql
-- Basic aggregates
SELECT 
    COUNT(*) as total_rows,
    COUNT(DISTINCT customer_id) as unique_customers,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MIN(amount) as min_amount,
    MAX(amount) as max_amount,
    MEDIAN(amount) as median_amount
FROM orders;

-- String aggregation
SELECT 
    customer_id,
    LISTAGG(product_name, ', ') WITHIN GROUP (ORDER BY product_name) as products
FROM purchases
GROUP BY customer_id;

SELECT 
    region,
    ARRAY_AGG(DISTINCT store_name) WITHIN GROUP (ORDER BY store_name) as stores
FROM store_locations
GROUP BY region;

-- Statistical functions
SELECT 
    product_category,
    STDDEV(price) as price_stddev,
    VARIANCE(price) as price_variance,
    STDDEV_POP(price) as pop_stddev,
    VAR_POP(price) as pop_variance
FROM products
GROUP BY product_category;

-- Approximate functions (faster for large datasets)
SELECT 
    APPROX_COUNT_DISTINCT(customer_id) as approx_unique_customers,
    APPROX_PERCENTILE(amount, 0.5) as approx_median,
    APPROX_TOP_K(product_id, 10) as top_products
FROM sales;

-- GROUPING SETS, ROLLUP, CUBE
SELECT region, product_category, SUM(amount)
FROM sales
GROUP BY GROUPING SETS ((region), (product_category), (region, product_category), ());

SELECT region, product_category, SUM(amount)
FROM sales
GROUP BY ROLLUP (region, product_category);

SELECT region, product_category, SUM(amount)
FROM sales
GROUP BY CUBE (region, product_category);
```

### 4.2 String Functions

```sql
-- Concatenation
SELECT 
    CONCAT(first_name, ' ', last_name) as full_name,
    first_name || ' ' || last_name as full_name_alt
FROM customers;

-- Case conversion
SELECT 
    UPPER(name) as upper_name,
    LOWER(name) as lower_name,
    INITCAP(name) as title_case
FROM customers;

-- Substring and trimming
SELECT 
    SUBSTRING(email, 1, 5) as email_prefix,
    LEFT(email, 5) as left_5,
    RIGHT(email, 10) as right_10,
    TRIM(name) as trimmed,
    LTRIM(name) as left_trimmed,
    RTRIM(name) as right_trimmed,
    TRIM(name, ' .') as trim_chars
FROM customers;

-- String manipulation
SELECT 
    LENGTH(name) as name_length,
    CHAR_LENGTH(name) as char_length,
    REPLACE(email, '@old.com', '@new.com') as new_email,
    SPLIT(full_address, ',') as address_parts,
    SPLIT_PART(email, '@', 2) as domain,
    POSITION('@' IN email) as at_position,
    CONTAINS(description, 'premium') as is_premium,
    STARTSWITH(product_code, 'PRD-') as is_product,
    ENDSWITH(filename, '.csv') as is_csv
FROM data_table;

-- Pattern matching
SELECT 
    name,
    REGEXP_REPLACE(phone, '[^0-9]', '') as clean_phone,
    REGEXP_SUBSTR(email, '[^@]+') as email_user,
    REGEXP_COUNT(text, 'error') as error_count,
    REGEXP_LIKE(email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}$') as valid_email
FROM customers;

-- String construction
SELECT 
    REPEAT('*', 5) as stars,
    LPAD(account_number, 10, '0') as padded_account,
    RPAD(name, 20, '.') as padded_name,
    REVERSE(text) as reversed
FROM accounts;

-- Encoding
SELECT 
    BASE64_ENCODE(data) as encoded,
    BASE64_DECODE_STRING(encoded_data) as decoded,
    HEX_ENCODE(binary_data) as hex,
    HEX_DECODE_STRING(hex_data) as decoded_hex
FROM data_table;
```

### 4.3 Date/Time Functions

```sql
-- Current date/time
SELECT 
    CURRENT_DATE() as today,
    CURRENT_TIME() as now_time,
    CURRENT_TIMESTAMP() as now_timestamp,
    SYSDATE() as system_date,
    GETDATE() as get_date;

-- Date arithmetic
SELECT 
    DATEADD(DAY, 7, CURRENT_DATE()) as one_week_later,
    DATEADD(MONTH, -3, CURRENT_DATE()) as three_months_ago,
    DATEADD(YEAR, 1, CURRENT_DATE()) as next_year,
    DATEADD(HOUR, 5, CURRENT_TIMESTAMP()) as five_hours_later;

SELECT 
    DATEDIFF(DAY, start_date, end_date) as days_between,
    DATEDIFF(MONTH, '2025-01-01', '2026-02-01') as months_between,
    DATEDIFF(YEAR, birth_date, CURRENT_DATE()) as age;

-- Date parts
SELECT 
    YEAR(order_date) as year,
    MONTH(order_date) as month,
    DAY(order_date) as day,
    QUARTER(order_date) as quarter,
    DAYOFWEEK(order_date) as dow,
    DAYOFYEAR(order_date) as doy,
    WEEKOFYEAR(order_date) as week,
    HOUR(created_at) as hour,
    MINUTE(created_at) as minute,
    SECOND(created_at) as second;

-- Date truncation
SELECT 
    DATE_TRUNC('DAY', created_at) as day_start,
    DATE_TRUNC('WEEK', created_at) as week_start,
    DATE_TRUNC('MONTH', created_at) as month_start,
    DATE_TRUNC('QUARTER', created_at) as quarter_start,
    DATE_TRUNC('YEAR', created_at) as year_start,
    DATE_TRUNC('HOUR', created_at) as hour_start;

-- Date formatting and parsing
SELECT 
    TO_CHAR(order_date, 'YYYY-MM-DD') as formatted_date,
    TO_CHAR(created_at, 'YYYY-MM-DD HH24:MI:SS') as formatted_timestamp,
    TO_DATE('2026-02-07', 'YYYY-MM-DD') as parsed_date,
    TO_TIMESTAMP('2026-02-07 10:30:00', 'YYYY-MM-DD HH24:MI:SS') as parsed_timestamp,
    TO_TIMESTAMP_NTZ('2026-02-07 10:30:00') as timestamp_ntz;

-- Timezone conversion
SELECT 
    CONVERT_TIMEZONE('America/Los_Angeles', 'America/New_York', timestamp_col) as ny_time,
    CONVERT_TIMEZONE('UTC', created_at) as utc_time;

-- Date construction
SELECT 
    DATE_FROM_PARTS(2026, 2, 7) as constructed_date,
    TIME_FROM_PARTS(10, 30, 0) as constructed_time,
    TIMESTAMP_FROM_PARTS(2026, 2, 7, 10, 30, 0) as constructed_timestamp;

-- Last day of period
SELECT 
    LAST_DAY(CURRENT_DATE()) as last_day_of_month,
    LAST_DAY(CURRENT_DATE(), 'YEAR') as last_day_of_year,
    LAST_DAY(CURRENT_DATE(), 'QUARTER') as last_day_of_quarter;
```

### 4.4 Numeric Functions

```sql
-- Rounding
SELECT 
    ROUND(123.456, 2) as rounded,        -- 123.46
    ROUND(123.456, 0) as rounded_int,    -- 123
    ROUND(123.456, -1) as rounded_tens,  -- 120
    CEIL(123.456) as ceiling,            -- 124
    CEILING(123.456) as ceiling_alt,     -- 124
    FLOOR(123.456) as floor_val,         -- 123
    TRUNC(123.456, 1) as truncated;      -- 123.4

-- Absolute and sign
SELECT 
    ABS(-123.45) as absolute,            -- 123.45
    SIGN(-123.45) as sign_val,           -- -1
    SIGN(0) as zero_sign,                -- 0
    SIGN(123.45) as positive_sign;       -- 1

-- Mathematical functions
SELECT 
    POWER(2, 10) as power_result,        -- 1024
    POW(2, 10) as pow_result,            -- 1024
    SQRT(16) as square_root,             -- 4
    MOD(10, 3) as modulo,                -- 1
    EXP(1) as exponential,               -- 2.718...
    LN(10) as natural_log,
    LOG(10, 100) as log_base_10;

-- Trigonometric
SELECT 
    PI() as pi_value,
    SIN(PI()/2) as sine,
    COS(0) as cosine,
    TAN(PI()/4) as tangent,
    ASIN(1) as arcsine,
    ACOS(0) as arccosine,
    ATAN(1) as arctangent;

-- Random
SELECT 
    RANDOM() as random_number,           -- Random 64-bit integer
    UNIFORM(1, 100, RANDOM()) as random_1_to_100,
    NORMAL(50, 10, RANDOM()) as normal_dist;

-- Bitwise operations
SELECT 
    BITAND(12, 10) as bit_and,
    BITOR(12, 10) as bit_or,
    BITXOR(12, 10) as bit_xor,
    BITNOT(12) as bit_not;

-- DIV0 (division with zero handling)
SELECT 
    DIV0(10, 2) as div_normal,     -- 5
    DIV0(10, 0) as div_by_zero,    -- 0 (instead of error)
    DIV0NULL(10, 0) as div_null;   -- NULL
```

### 4.5 Conditional Functions

```sql
-- CASE expressions
SELECT 
    product_id,
    price,
    CASE 
        WHEN price < 10 THEN 'Budget'
        WHEN price < 50 THEN 'Standard'
        WHEN price < 100 THEN 'Premium'
        ELSE 'Luxury'
    END as price_tier,
    CASE category
        WHEN 'Electronics' THEN 'Tech'
        WHEN 'Clothing' THEN 'Apparel'
        ELSE 'Other'
    END as category_group
FROM products;

-- IFF (inline IF)
SELECT 
    customer_id,
    IFF(total_spent > 1000, 'VIP', 'Standard') as customer_tier,
    IFF(status = 'ACTIVE', 1, 0) as is_active
FROM customers;

-- IFNULL, NVL
SELECT 
    IFNULL(middle_name, 'N/A') as middle_name,
    NVL(phone, 'No phone') as phone,
    NVL2(email, 'Has email', 'No email') as email_status
FROM customers;

-- COALESCE (return first non-null)
SELECT 
    COALESCE(mobile_phone, home_phone, work_phone, 'No phone') as contact_phone,
    COALESCE(nickname, first_name, 'Unknown') as display_name
FROM customers;

-- NULLIF (return NULL if equal)
SELECT 
    NULLIF(column1, 0) as safe_division,  -- NULL if 0, otherwise value
    NULLIF(status, 'DELETED') as active_status
FROM data_table;

-- ZEROIFNULL
SELECT 
    ZEROIFNULL(revenue) as revenue,
    ZEROIFNULL(cost) as cost
FROM financial_data;

-- GREATEST, LEAST
SELECT 
    GREATEST(price1, price2, price3) as max_price,
    LEAST(price1, price2, price3) as min_price
FROM price_comparison;

-- DECODE (Oracle-style CASE)
SELECT 
    DECODE(status, 
        'A', 'Active',
        'I', 'Inactive',
        'P', 'Pending',
        'Unknown'
    ) as status_description
FROM accounts;
```

### 4.6 Window Functions

```sql
-- ROW_NUMBER (unique sequential number)
SELECT 
    customer_id,
    order_date,
    amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as order_sequence
FROM orders;

-- RANK and DENSE_RANK
SELECT 
    product_name,
    category,
    revenue,
    RANK() OVER (PARTITION BY category ORDER BY revenue DESC) as rank,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY revenue DESC) as dense_rank,
    PERCENT_RANK() OVER (PARTITION BY category ORDER BY revenue DESC) as percent_rank
FROM product_sales;

-- NTILE (divide into buckets)
SELECT 
    customer_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) as quartile,
    NTILE(10) OVER (ORDER BY total_spent DESC) as decile
FROM customer_lifetime_value;

-- LAG and LEAD (access adjacent rows)
SELECT 
    sale_date,
    daily_revenue,
    LAG(daily_revenue, 1) OVER (ORDER BY sale_date) as previous_day,
    LEAD(daily_revenue, 1) OVER (ORDER BY sale_date) as next_day,
    daily_revenue - LAG(daily_revenue, 1) OVER (ORDER BY sale_date) as day_over_day_change
FROM daily_sales;

-- FIRST_VALUE and LAST_VALUE
SELECT 
    employee_id,
    department,
    salary,
    FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) as highest_salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as lowest_salary
FROM employees;

-- NTH_VALUE
SELECT 
    product_id,
    sale_date,
    revenue,
    NTH_VALUE(revenue, 2) OVER (PARTITION BY product_id ORDER BY sale_date) as second_day_revenue
FROM daily_product_sales;

-- Cumulative aggregates
SELECT 
    order_date,
    daily_amount,
    SUM(daily_amount) OVER (ORDER BY order_date) as running_total,
    AVG(daily_amount) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as seven_day_avg,
    COUNT(*) OVER (ORDER BY order_date) as cumulative_count
FROM daily_orders;

-- Moving window
SELECT 
    sale_date,
    amount,
    SUM(amount) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
    ) as five_day_total,
    AVG(amount) OVER (
        ORDER BY sale_date 
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) as trailing_7_day_avg
FROM sales;

-- Multiple windows
SELECT 
    department,
    employee_id,
    salary,
    AVG(salary) OVER w1 as dept_avg,
    RANK() OVER w2 as salary_rank
FROM employees
WINDOW 
    w1 AS (PARTITION BY department),
    w2 AS (PARTITION BY department ORDER BY salary DESC);
```

### 4.7 Semi-Structured Functions

```sql
-- PARSE_JSON
SELECT 
    PARSE_JSON('{"name": "John", "age": 30, "city": "NYC"}') as json_data;

-- Access JSON elements
SELECT 
    data:name::STRING as name,
    data:age::NUMBER as age,
    data:address.city::STRING as city,
    data:tags[0]::STRING as first_tag
FROM json_table;

-- GET and GET_PATH
SELECT 
    GET(data, 'name')::STRING as name,
    GET_PATH(data, 'address.city')::STRING as city
FROM json_table;

-- FLATTEN (explode arrays)
SELECT 
    id,
    f.value::STRING as tag
FROM json_table,
LATERAL FLATTEN(input => data:tags) f;

SELECT 
    customer_id,
    f.value:product_id::NUMBER as product_id,
    f.value:quantity::NUMBER as quantity
FROM orders,
LATERAL FLATTEN(input => order_items) f;

-- OBJECT_CONSTRUCT
SELECT 
    OBJECT_CONSTRUCT(
        'customer_id', customer_id,
        'name', first_name || ' ' || last_name,
        'total_orders', order_count
    ) as customer_json
FROM customers;

-- OBJECT_CONSTRUCT_KEEP_NULL
SELECT 
    OBJECT_CONSTRUCT_KEEP_NULL(
        'id', customer_id,
        'email', email,  -- Included even if NULL
        'phone', phone
    ) as customer_data
FROM customers;

-- ARRAY_CONSTRUCT
SELECT 
    ARRAY_CONSTRUCT(
        'tag1', 'tag2', 'tag3'
    ) as tags;

-- ARRAY_AGG (aggregate into array)
SELECT 
    customer_id,
    ARRAY_AGG(OBJECT_CONSTRUCT('product', product_name, 'amount', amount)) 
        WITHIN GROUP (ORDER BY order_date) as purchase_history
FROM orders
GROUP BY customer_id;

-- JSON/ARRAY manipulation
SELECT 
    ARRAY_SIZE(tags) as tag_count,
    ARRAY_CONTAINS('premium'::VARIANT, tags) as has_premium,
    ARRAY_SLICE(tags, 0, 2) as first_two_tags,
    ARRAY_CAT(tags1, tags2) as combined_tags,
    ARRAY_INTERSECTION(tags1, tags2) as common_tags,
    ARRAY_DISTINCT(tags) as unique_tags,
    ARRAY_SORT(tags) as sorted_tags
FROM product_data;

-- TYPEOF (get data type)
SELECT 
    TYPEOF(data:name) as name_type,
    TYPEOF(data:age) as age_type,
    TYPEOF(data:tags) as tags_type
FROM json_table;

-- IS_* functions
SELECT 
    IS_ARRAY(data:tags) as is_array_col,
    IS_OBJECT(data:address) as is_object_col,
    IS_INTEGER(data:age) as is_integer_col,
    IS_DECIMAL(data:price) as is_decimal_col,
    IS_VARCHAR(data:name) as is_varchar_col,
    IS_NULL_VALUE(data:optional_field) as is_null_col
FROM json_table;

-- STRIP_NULL_VALUE
SELECT 
    STRIP_NULL_VALUE(data) as clean_data
FROM json_table;

-- CHECK_JSON, CHECK_XML
SELECT 
    CHECK_JSON(raw_string) as json_validation,
    CHECK_XML(xml_string) as xml_validation
FROM raw_data;
```

### 4.8 Table Functions

```sql
-- FLATTEN (array/object expansion)
SELECT *
FROM TABLE(FLATTEN(INPUT => PARSE_JSON('["a", "b", "c"]')));

-- GENERATOR (generate rows)
SELECT SEQ4() as sequence_num, UNIFORM(1, 100, RANDOM()) as random_val
FROM TABLE(GENERATOR(ROWCOUNT => 1000));

-- RESULT_SCAN (query previous query results)
SELECT * FROM sales WHERE region = 'West';
-- Get query ID from result
SELECT * FROM TABLE(RESULT_SCAN('01a2b3c4-5678-90ab-cdef-1234567890ab'));
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- SPLIT_TO_TABLE (split string into rows)
SELECT 
    id,
    f.value::STRING as tag
FROM products,
TABLE(SPLIT_TO_TABLE(tags_string, ',')) f;

-- EXTERNAL_TABLE_FILES (list external table files)
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES(
        TABLE_NAME => 'EXT_SALES'
    )
);

-- TASK_HISTORY
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.TASK_HISTORY(
        TASK_NAME => 'DAILY_ETL',
        SCHEDULED_TIME_RANGE_START => DATEADD('day', -7, CURRENT_TIMESTAMP())
    )
);

-- COPY_HISTORY
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.COPY_HISTORY(
        TABLE_NAME => 'CUSTOMERS',
        START_TIME => DATEADD('day', -1, CURRENT_TIMESTAMP())
    )
);

-- QUERY_HISTORY
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.QUERY_HISTORY(
        END_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP()),
        END_TIME_RANGE_END => CURRENT_TIMESTAMP()
    )
);

-- WAREHOUSE_LOAD_HISTORY
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.WAREHOUSE_LOAD_HISTORY(
        WAREHOUSE_NAME => 'COMPUTE_WH',
        DATE_RANGE_START => DATEADD('day', -7, CURRENT_TIMESTAMP())
    )
);
```

---

## 5. Snowflake Features

### 5.1 Time Travel

```sql
-- Query as of specific timestamp
SELECT * FROM customers
AT (TIMESTAMP => '2026-02-06 10:00:00'::TIMESTAMP);

-- Query as of offset (seconds ago)
SELECT * FROM customers
AT (OFFSET => -3600);  -- 1 hour ago

-- Query before statement
SELECT * FROM customers
BEFORE (STATEMENT => '01a2b3c4-5678-90ab-cdef-1234567890ab');

-- Clone at point in time
CREATE TABLE customers_backup CLONE customers
AT (TIMESTAMP => '2026-02-01 00:00:00'::TIMESTAMP);

-- Compare current vs historical
SELECT 
    current.*,
    historical.status as old_status
FROM customers current
LEFT JOIN customers AT (OFFSET => -86400) historical
ON current.customer_id = historical.customer_id
WHERE current.status != historical.status;

-- Restore dropped table
UNDROP TABLE customers;

-- Set retention period
ALTER TABLE customers SET DATA_RETENTION_TIME_IN_DAYS = 30;
```

### 5.2 Zero-Copy Cloning

```sql
-- Clone table (instant, no storage duplication initially)
CREATE TABLE customers_dev CLONE customers;

-- Clone with Time Travel
CREATE TABLE customers_yesterday CLONE customers
AT (OFFSET => -86400);

-- Clone schema
CREATE SCHEMA analytics_dev CLONE analytics;

-- Clone database
CREATE DATABASE sales_test CLONE sales_prod;

-- Clone preserves:
-- - All data
-- - Table structure
-- - Constraints
-- - Privileges (with COPY GRANTS)

CREATE TABLE archived_2025 CLONE customers
AT (TIMESTAMP => '2025-12-31 23:59:59'::TIMESTAMP)
COPY GRANTS;
```

### 5.3 Data Sharing

```sql
-- Create share
CREATE SHARE customer_data_share;

-- Grant usage on database/schema
GRANT USAGE ON DATABASE sales_db TO SHARE customer_data_share;
GRANT USAGE ON SCHEMA sales_db.public TO SHARE customer_data_share;

-- Grant select on tables/views (must be SECURE views)
GRANT SELECT ON TABLE sales_db.public.customers TO SHARE customer_data_share;
GRANT SELECT ON VIEW sales_db.public.secure_sales_view TO SHARE customer_data_share;

-- Add accounts to share
ALTER SHARE customer_data_share 
ADD ACCOUNTS = account_id1, account_id2;

-- Show shares
SHOW SHARES;

-- Describe share
DESC SHARE customer_data_share;

-- Drop share
DROP SHARE customer_data_share;

-- Consumer: Create database from share
CREATE DATABASE shared_data FROM SHARE provider_account.customer_data_share;
```

### 5.4 Query Optimization

#### Clustering Keys

```sql
-- Add clustering key
ALTER TABLE sales CLUSTER BY (sale_date, region);

-- Multi-column clustering
ALTER TABLE events CLUSTER BY (event_date, event_type, user_id);

-- Linear clustering
ALTER TABLE logs CLUSTER BY (LINEAR(timestamp));

-- Check clustering information
SELECT SYSTEM$CLUSTERING_INFORMATION('sales', '(sale_date, region)');

-- Automatic clustering (enabled by default for Enterprise)
ALTER TABLE sales RESUME RECLUSTER;
ALTER TABLE sales SUSPEND RECLUSTER;
```

#### Search Optimization

```sql
-- Enable search optimization
ALTER TABLE customers ADD SEARCH OPTIMIZATION;

-- On specific columns
ALTER TABLE customers ADD SEARCH OPTIMIZATION ON EQUALITY(email), SUBSTRING(name);

-- Check status
SELECT SYSTEM$SEARCH_OPTIMIZATION_PROGRESS('CUSTOMERS');

-- Drop search optimization
ALTER TABLE customers DROP SEARCH OPTIMIZATION;
```

#### Result Caching

```sql
-- Automatically enabled (24-hour cache)
-- Use same query to hit cache
SELECT * FROM sales WHERE sale_date = CURRENT_DATE();

-- Bypass cache
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
SELECT * FROM sales WHERE sale_date = CURRENT_DATE();
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
```

#### Query Profile and Optimization

```sql
-- Explain plan
EXPLAIN USING TEXT SELECT * FROM large_table WHERE date_col = '2026-02-07';

-- Query history with performance
SELECT 
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    rows_produced
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE execution_time > 60000  -- Queries over 1 minute
ORDER BY execution_time DESC;

-- Identify scan-heavy queries
SELECT 
    query_text,
    bytes_scanned / 1024 / 1024 / 1024 as gb_scanned,
    execution_time / 1000 as seconds
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE bytes_scanned > 1000000000  -- > 1GB
ORDER BY bytes_scanned DESC;
```

### 5.5 Virtual Warehouses

```sql
-- Create warehouse
CREATE WAREHOUSE compute_wh
    WAREHOUSE_SIZE = XSMALL
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'General compute warehouse';

-- Sizes: XSMALL, SMALL, MEDIUM, LARGE, XLARGE, 2XLARGE, 3XLARGE, 4XLARGE, 5XLARGE, 6XLARGE

-- Multi-cluster warehouse (Enterprise)
CREATE WAREHOUSE analytics_wh
    WAREHOUSE_SIZE = LARGE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 5
    SCALING_POLICY = STANDARD
    AUTO_SUSPEND = 600
    AUTO_RESUME = TRUE;

-- Alter warehouse
ALTER WAREHOUSE compute_wh SET WAREHOUSE_SIZE = MEDIUM;
ALTER WAREHOUSE compute_wh SET AUTO_SUSPEND = 600;
ALTER WAREHOUSE compute_wh SET MAX_CLUSTER_COUNT = 3;

-- Resume/suspend
ALTER WAREHOUSE compute_wh RESUME;
ALTER WAREHOUSE compute_wh SUSPEND;

-- Use warehouse
USE WAREHOUSE compute_wh;

-- Show warehouses
SHOW WAREHOUSES;

-- Warehouse usage
SELECT *
FROM TABLE(
    INFORMATION_SCHEMA.WAREHOUSE_METERING_HISTORY(
        DATE_RANGE_START => DATEADD('day', -30, CURRENT_TIMESTAMP())
    )
);

-- Drop warehouse
DROP WAREHOUSE compute_wh;
```

### 5.6 Security

#### Network Policies

```sql
-- Create network policy
CREATE NETWORK POLICY office_policy
    ALLOWED_IP_LIST = ('192.168.1.0/24', '10.0.0.0/8')
    BLOCKED_IP_LIST = ('192.168.1.99');

-- Apply to account
ALTER ACCOUNT SET NETWORK_POLICY = office_policy;

-- Apply to user
ALTER USER john_doe SET NETWORK_POLICY = office_policy;
```

#### Row Access Policies

```sql
-- Create mapping table
CREATE TABLE user_regions (
    username VARCHAR,
    region VARCHAR
);

-- Create row access policy
CREATE ROW ACCESS POLICY region_policy AS (region VARCHAR) RETURNS BOOLEAN ->
    region = (SELECT region FROM user_regions WHERE username = CURRENT_USER())
    OR CURRENT_ROLE() IN ('ADMIN');

-- Apply policy
ALTER TABLE sales ADD ROW ACCESS POLICY region_policy ON (region);

-- Drop policy
ALTER TABLE sales DROP ROW ACCESS POLICY region_policy;
```

#### Column Masking Policies

```sql
-- Create masking policy
CREATE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
    CASE 
        WHEN CURRENT_ROLE() IN ('ADMIN', 'HR') THEN val
        ELSE REGEXP_REPLACE(val, '.+@', '***@')
    END;

-- Apply to column
ALTER TABLE customers MODIFY COLUMN email 
    SET MASKING POLICY email_mask;

-- Unset policy
ALTER TABLE customers MODIFY COLUMN email 
    UNSET MASKING POLICY;
```

---

## 6. Best Practices

### Performance Optimization

```sql
-- ✅ DO: Use clustering for large tables with predictable filters
ALTER TABLE sales CLUSTER BY (sale_date, region);

-- ✅ DO: Use materialized views for expensive aggregations
CREATE MATERIALIZED VIEW mv_daily_metrics AS
SELECT DATE_TRUNC('day', created_at), COUNT(*), SUM(amount)
FROM transactions
GROUP BY DATE_TRUNC('day', created_at);

-- ❌ DON'T: SELECT * from large tables
-- Instead: SELECT specific columns
SELECT customer_id, name, email FROM customers;

-- ✅ DO: Use QUALIFY for window function filtering
SELECT customer_id, order_id, amount
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) = 1;

-- ✅ DO: Use COPY INTO for bulk loading
COPY INTO customers FROM @my_stage/customers.csv
FILE_FORMAT = (TYPE = CSV);

-- ❌ DON'T: Use many single-row INSERTs
-- Instead: Batch inserts or use COPY INTO

-- ✅ DO: Use transient tables for temporary/staging data
CREATE TRANSIENT TABLE staging_orders (...);

-- ✅ DO: Leverage result caching for repeated queries
-- Same query within 24 hours uses cached results automatically
```

### Cost Optimization

```sql
-- ✅ Use AUTO_SUSPEND for warehouses
CREATE WAREHOUSE wh AUTO_SUSPEND = 300;  -- 5 minutes

-- ✅ Use appropriate warehouse sizes
-- Start small, scale up if needed
-- XSMALL for light queries, LARGE+ for heavy analytics

-- ✅ Use transient tables when fail-safe not needed
CREATE TRANSIENT TABLE logs (...);

-- ✅ Monitor credit usage
SELECT *
FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_METERING_HISTORY(
    DATE_RANGE_START => DATEADD('day', -30, CURRENT_TIMESTAMP())
));

-- ✅ Use multi-cluster warehouses for concurrent workloads
CREATE WAREHOUSE analytics_wh
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 5;
```

### Security Best Practices

```sql
-- ✅ Use roles for access control
CREATE ROLE analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ROLE analyst;
GRANT ROLE analyst TO USER john_doe;

-- ✅ Use SECURE views for data sharing
CREATE SECURE VIEW shared_view AS
SELECT non_sensitive_columns FROM table;

-- ✅ Implement row-level security
CREATE ROW ACCESS POLICY AS (...);
ALTER TABLE sensitive_data ADD ROW ACCESS POLICY policy_name ON (column);

-- ✅ Mask sensitive columns
CREATE MASKING POLICY ssn_mask AS (val STRING) RETURNS STRING ->
    CASE WHEN CURRENT_ROLE() = 'ADMIN' THEN val ELSE 'XXX-XX-' || RIGHT(val, 4) END;

-- ✅ Use managed access schemas
CREATE SCHEMA sensitive WITH MANAGED ACCESS;
```

---

## 7. Quick Reference Tables

### Data Type Sizes

| Type | Max Size |
|------|----------|
| VARCHAR | 16 MB |
| NUMBER | 38 digits precision |
| BINARY | 8 MB |
| VARIANT | 16 MB compressed |

### Warehouse Sizes

| Size | Credits/Hour | Relative Performance |
|------|--------------|---------------------|
| XSMALL | 1 | 1x |
| SMALL | 2 | 2x |
| MEDIUM | 4 | 4x |
| LARGE | 8 | 8x |
| XLARGE | 16 | 16x |
| 2XLARGE | 32 | 32x |
| 3XLARGE | 64 | 64x |
| 4XLARGE | 128 | 128x |

### Time Travel Retention

| Edition | Max Retention |
|---------|--------------|
| Standard | 1 day |
| Enterprise | 90 days |
| Transient Objects | 0-1 day |
| Temporary Objects | 0-1 day |

### Common Patterns

```sql
-- Slowly Changing Dimension Type 2
MERGE INTO dimension d
USING staging s ON d.id = s.id AND d.is_current = TRUE
WHEN MATCHED AND (d.col1 != s.col1 OR d.col2 != s.col2) THEN
    UPDATE SET is_current = FALSE, end_date = CURRENT_DATE()
WHEN NOT MATCHED THEN
    INSERT (id, col1, col2, start_date, end_date, is_current)
    VALUES (s.id, s.col1, s.col2, CURRENT_DATE(), NULL, TRUE);

-- Deduplication
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) as rn
    FROM raw_data
)
WHERE rn = 1;

-- Running total
SELECT 
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_total
FROM sales;

-- Pivot
SELECT *
FROM sales
PIVOT (SUM(amount) FOR region IN ('East', 'West', 'North', 'South'))
AS pivoted;

-- Unpivot
SELECT *
FROM quarterly_sales
UNPIVOT (sales FOR quarter IN (q1, q2, q3, q4))
AS unpivoted;
```

### SQL Dialect Differences

| Feature | Snowflake | PostgreSQL | MySQL | Oracle |
|---------|-----------|------------|-------|--------|
| String Concat | \|\| or CONCAT() | \|\| | CONCAT() | \|\| |
| Sequence | NEXTVAL | nextval() | AUTO_INCREMENT | .NEXTVAL |
| Top N | LIMIT | LIMIT | LIMIT | ROWNUM |
| Date Add | DATEADD() | INTERVAL | DATE_ADD() | ADD_MONTHS() |
| NVL | NVL() | COALESCE() | IFNULL() | NVL() |

### Resources

- **Official Documentation**: https://docs.snowflake.com
- **SnowPro Certification**: https://www.snowflake.com/certifications/
- **Community**: https://community.snowflake.com
- **Snowflake University**: Free training courses
- **Partner Connect**: Pre-built integrations

---

**End of Cheatsheet**

*This comprehensive guide covers Snowflake SQL from basics to advanced features. Bookmark this page and use the table of contents to quickly find what you need. For the latest updates, always refer to the official Snowflake documentation.*
