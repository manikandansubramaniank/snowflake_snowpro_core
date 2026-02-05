# Snowflake Data Engineering Interview Questions & Answers
*Generated on: February 5, 2026*  
*Content based on latest industry trends and Snowflake documentation*

---

## Table of Contents
1. [Snowflake Architecture & Fundamentals](#architecture--fundamentals)
2. [SQL & Query Optimization](#sql--query-optimization)
3. [Data Loading & Transformation](#data-loading--transformation)
4. [DBT Integration](#dbt-integration)
5. [Security & Governance](#security--governance)
6. [Performance & Cost Optimization](#performance--cost-optimization)
7. [Advanced Features](#advanced-features)
8. [Python & Airflow Integration](#python--airflow-integration)
9. [Scenario-Based Questions](#scenario-based-questions)

---

## Architecture & Fundamentals

### Q1: Explain Snowflake's three-layer architecture and how it differs from traditional data warehouses.
**Difficulty:** Beginner  
**Topics:** `Snowflake` `Architecture` `Cloud`

**Question:**
What are the three main architectural layers in Snowflake, and how does this design provide advantages over traditional data warehouses?

**Answer:**
Snowflake's architecture consists of three distinct, independently scalable layers:

1. **Database Storage Layer**: This is where all structured and semi-structured data is stored in a compressed, columnar format. Data is automatically organized into micro-partitions (50-500 MB compressed). This layer uses the underlying cloud provider's storage (AWS S3, Azure Blob, GCP Cloud Storage).

2. **Query Processing Layer (Compute)**: Virtual warehouses execute queries using clusters of compute resources. Each warehouse operates independently, allowing unlimited concurrent workloads without resource contention. Warehouses can be started, stopped, and resized on-demand.

3. **Cloud Services Layer**: The "brain" of Snowflake that handles authentication, access control, metadata management, query optimization, query compilation, and transaction management. This layer coordinates all activities across the platform.

**Key Differentiator**: Unlike traditional architectures where storage and compute are tightly coupled, Snowflake's separation allows independent scaling of each layer. You can spin up multiple warehouses to handle different workloads simultaneously without affecting each other, and storage scales infinitely without compute changes.

**Key Points:**
- Storage and compute scale independently (pay only for what you use)
- No resource contention between workloads
- Near-zero administration compared to traditional systems
- Automatic micro-partitioning eliminates manual indexing

**Common Mistakes:**
- Confusing virtual warehouses with databases (warehouses are compute, not storage)
- Assuming larger warehouses always mean faster queries (depends on query complexity)

---

### Q2: What is micro-partitioning in Snowflake, and how does it optimize query performance?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Storage` `Performance`

**Question:**
Explain how Snowflake's micro-partitioning works and its role in query optimization.

**Answer:**
Micro-partitioning is Snowflake's automatic method of organizing data into small, immutable storage units. Each micro-partition contains 50-500 MB of uncompressed data (stored compressed) and is organized in a columnar format.

**How it works:**
- Data is automatically divided into micro-partitions based on insertion order
- Each partition stores metadata including: min/max values for each column, distinct value counts, and NULL counts
- Partitions are immutable—updates create new partitions; old ones are retained for Time Travel

**Query Optimization Benefits:**
1. **Partition Pruning**: Snowflake uses metadata to skip irrelevant partitions entirely. For a query with `WHERE created_at > '2024-01-01'`, Snowflake skips all partitions where max(created_at) < '2024-01-01'.

2. **Column Pruning**: Only required columns are read from storage due to columnar format.

3. **No Manual Indexing**: The metadata serves as a lightweight "index" without maintenance overhead.

**Example:**
```sql
-- This query benefits from partition pruning
SELECT * FROM orders 
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';
-- Snowflake only scans partitions containing January 2024 data
```

**Key Points:**
- Partitioning is automatic—no manual intervention required
- Metadata enables intelligent query pruning without indexes
- Works best when filter columns have natural clustering (e.g., date columns)

---

### Q3: What are the different types of tables in Snowflake, and when would you use each?
**Difficulty:** Beginner  
**Topics:** `Snowflake` `Tables` `Storage`

**Question:**
Describe Permanent, Transient, and Temporary tables in Snowflake, including their Time Travel and Fail-safe characteristics.

**Answer:**

| Table Type | Time Travel | Fail-safe | Use Case |
|------------|-------------|-----------|----------|
| **Permanent** | 0-90 days (Enterprise: up to 90) | 7 days | Production data requiring full protection |
| **Transient** | 0-1 day | None | Staging/ETL data that can be recreated |
| **Temporary** | 0-1 day | None | Session-scoped working data |

**Permanent Tables:**
- Default table type with full CDP (Continuous Data Protection)
- Time Travel retention configurable up to 90 days (Enterprise edition)
- 7-day Fail-safe period after Time Travel expires
- Highest storage costs due to data retention

**Transient Tables:**
- Created with `CREATE TRANSIENT TABLE`
- No Fail-safe period—data not recoverable after Time Travel expires
- Lower storage costs than permanent tables
- Ideal for staging areas, ETL intermediate data, or data that can be regenerated

**Temporary Tables:**
- Created with `CREATE TEMPORARY TABLE`
- Exist only within the session that created them
- Automatically dropped when session ends
- No visibility to other sessions or users
- Best for session-specific calculations or intermediate results

**Example:**
```sql
-- Staging table that can be recreated from source
CREATE TRANSIENT TABLE staging.raw_events (
    event_id STRING,
    event_data VARIANT,
    load_timestamp TIMESTAMP
);

-- Temporary working table for complex calculations
CREATE TEMPORARY TABLE temp_calculations AS
SELECT customer_id, SUM(amount) as total
FROM orders GROUP BY customer_id;
```

**Key Points:**
- Choose table type based on data criticality and recovery requirements
- Transient tables reduce storage costs by ~50% compared to permanent tables
- Temporary tables are perfect for CTEs that need to be materialized

---

### Q4: What is Zero-Copy Cloning in Snowflake, and how does it work internally?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Cloning` `Storage`

**Question:**
Explain the concept of Zero-Copy Cloning and its practical applications in data engineering.

**Answer:**
Zero-Copy Cloning creates an instant copy of any database, schema, or table without physically duplicating the underlying data. It works by creating new metadata pointers to the same micro-partitions.

**How It Works Internally:**
1. When you clone an object, Snowflake creates new metadata entries pointing to the existing micro-partitions
2. No actual data is copied—only metadata is duplicated
3. Changes to either the clone or source create new micro-partitions owned exclusively by that object
4. Storage costs only incur when data in either object is modified

**Syntax:**
```sql
-- Clone a table
CREATE TABLE orders_dev CLONE orders;

-- Clone an entire database at a point in time
CREATE DATABASE prod_backup CLONE production
    AT(TIMESTAMP => '2024-01-15 10:00:00'::timestamp);

-- Clone a schema
CREATE SCHEMA test_schema CLONE prod_schema;
```

**Practical Applications:**
1. **Development/Testing Environments**: Create instant production replicas without storage costs
2. **Backup Snapshots**: Point-in-time clones for disaster recovery
3. **Data Science Sandboxes**: Safe experimentation without affecting production
4. **A/B Testing**: Compare different data transformations side-by-side

**Important Considerations:**
- Privileges are NOT cloned (must be granted separately)
- When cloning with Time Travel, historical data must still be available
- Tasks in cloned schemas are suspended by default
- Streams on cloned tables start fresh (no historical change data)

**Key Points:**
- Cloning is instantaneous regardless of data size
- Free until data is modified in either source or clone
- Combines with Time Travel for point-in-time clones

**Common Mistakes:**
- Expecting privileges to transfer with clones
- Forgetting that streams don't carry over change data

---

### Q5: What is the difference between Snowflake's Result Cache, Metadata Cache, and Warehouse Cache?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Caching` `Performance`

**Question:**
Explain the three types of caching in Snowflake and when each is used.

**Answer:**
Snowflake employs three distinct caching mechanisms to optimize query performance:

**1. Result Cache (Query Result Cache)**
- **Location**: Cloud Services Layer
- **Duration**: 24 hours
- **How it works**: Stores complete query results for identical queries
- **Conditions**: Same query text, same role, same data (no changes to underlying tables)
- **Cost**: Free (no warehouse compute needed)

```sql
-- First execution: uses warehouse compute
SELECT COUNT(*) FROM large_table WHERE status = 'active';

-- Second identical execution within 24 hours: instant result from cache
SELECT COUNT(*) FROM large_table WHERE status = 'active';
```

**2. Metadata Cache**
- **Location**: Cloud Services Layer
- **How it works**: Stores metadata about micro-partitions (min/max values, row counts)
- **Use cases**: COUNT(*), MIN(), MAX() on clustered columns
- **Cost**: Often free if query can be answered entirely from metadata

```sql
-- Can be answered from metadata cache (no table scan)
SELECT MIN(order_date), MAX(order_date) FROM orders;
SELECT COUNT(*) FROM orders;
```

**3. Warehouse Cache (Local Disk Cache)**
- **Location**: SSD storage on warehouse compute nodes
- **Duration**: While warehouse is running (lost on suspend)
- **How it works**: Caches raw table data read from remote storage
- **Benefit**: Subsequent queries reading same data avoid network I/O

**Optimization Tips:**
- Keep warehouses running for repeated workloads to leverage warehouse cache
- Use consistent query patterns to maximize result cache hits
- Set `USE_CACHED_RESULT = FALSE` when testing query performance

**Key Points:**
- Result cache provides instant results for repeated identical queries
- Metadata cache enables sub-second responses for aggregate queries
- Warehouse cache reduces I/O for repeated access patterns

---

## SQL & Query Optimization

### Q6: How do you analyze and optimize a slow-running query in Snowflake?
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Performance` `Query Optimization`

**Question:**
Walk through your approach to diagnosing and fixing a slow query in Snowflake.

**Answer:**
A systematic approach to query optimization involves analyzing the Query Profile and addressing specific bottlenecks:

**Step 1: Access the Query Profile**
```sql
-- Find the query ID from history
SELECT query_id, query_text, total_elapsed_time, bytes_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_text ILIKE '%your_table%'
ORDER BY start_time DESC
LIMIT 10;
```

**Step 2: Analyze the Query Profile**
Look for these key metrics in the Query Profile UI:
- **Percentage Scanned from Cache**: Low values indicate cold cache
- **Bytes Sent Over Network**: High values suggest partition pruning issues
- **Spillage to Local/Remote Storage**: Indicates memory pressure
- **Most Expensive Nodes**: Identify bottleneck operations

**Step 3: Common Optimization Patterns**

*Issue: High TableScan time (poor partition pruning)*
```sql
-- BAD: Function on column prevents pruning
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- GOOD: Direct comparison enables pruning
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
```

*Issue: Cartesian product or row explosion*
```sql
-- Check for missing/incorrect JOIN conditions
-- Ensure JOINs have proper ON clauses
SELECT o.*, c.customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id; -- Ensure this exists
```

*Issue: Disk spillage*
```sql
-- Increase warehouse size or optimize query
-- Check for large aggregations or sorts
-- Consider pre-aggregating data
```

**Step 4: Implement Solutions**
1. **Add clustering keys** for frequently filtered columns
2. **Use materialized views** for expensive repeated calculations
3. **Enable Search Optimization Service** for point lookups
4. **Right-size the warehouse** based on workload

**Key Points:**
- Always start with the Query Profile before making changes
- Focus on the "Most Expensive Nodes" section
- Disk spillage is often a sign to increase warehouse size
- Partition pruning is the biggest performance lever

---

### Q7: Explain clustering keys in Snowflake and when to use them.
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Clustering` `Performance`

**Question:**
What are clustering keys, how do they work, and when should you implement them?

**Answer:**
Clustering keys define the order in which data is organized within micro-partitions. While Snowflake automatically clusters data by insertion order, explicit clustering keys can dramatically improve query performance for specific access patterns.

**When to Use Clustering Keys:**
- Tables larger than 1 TB
- Queries consistently filter on specific columns
- Current partition pruning is inefficient (check Query Profile)
- Filter columns have high cardinality but predictable patterns

**Best Practices for Key Selection:**
```sql
-- Single column clustering (most common)
ALTER TABLE orders CLUSTER BY (order_date);

-- Multi-column clustering (order matters - put lowest cardinality first)
ALTER TABLE events CLUSTER BY (event_type, event_date);

-- Expression-based clustering
ALTER TABLE logs CLUSTER BY (TO_DATE(created_at));
```

**Column Selection Guidelines:**
1. **Cardinality**: Choose columns with medium cardinality (not too unique, not too few values)
2. **Query patterns**: Prioritize columns used in WHERE clauses and JOINs
3. **Column order**: Put lower cardinality columns first

**Monitoring Clustering:**
```sql
-- Check clustering depth
SELECT SYSTEM$CLUSTERING_DEPTH('my_table');

-- Check clustering information
SELECT SYSTEM$CLUSTERING_INFORMATION('my_table', '(cluster_column)');
```

**Automatic vs Manual Reclustering:**
- As of May 2020, manual reclustering is deprecated
- Snowflake automatically maintains clustering in the background
- Reclustering incurs serverless compute costs

**Example Impact:**
```sql
-- Without clustering: Scans 500 GB
SELECT * FROM events WHERE event_date = '2024-01-15';

-- With clustering on event_date: Scans 2 GB (99% pruned)
SELECT * FROM events WHERE event_date = '2024-01-15';
```

**Key Points:**
- Only cluster large tables with clear access patterns
- Monitor clustering depth to ensure maintenance is effective
- Clustering incurs ongoing maintenance costs

**Common Mistakes:**
- Over-clustering small tables (added cost, minimal benefit)
- Choosing high-cardinality columns (e.g., unique IDs)
- Not monitoring clustering effectiveness over time

---

### Q8: What is the Search Optimization Service, and when should you use it?
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Performance` `Search Optimization`

**Question:**
Explain Snowflake's Search Optimization Service and its ideal use cases.

**Answer:**
The Search Optimization Service (SOS) creates and maintains a search access path—a persistent data structure that tracks which values might be found in each micro-partition. It significantly improves performance for selective point lookup queries.

**Ideal Use Cases:**
1. **Point lookups**: Queries returning one or few rows from large tables
2. **Equality predicates**: `WHERE user_id = 'abc123'`
3. **IN clauses**: `WHERE status IN ('pending', 'active')`
4. **Substring searches**: `WHERE name LIKE '%smith%'`
5. **Semi-structured data queries**: Searching within VARIANT columns

**Enabling Search Optimization:**
```sql
-- Enable on entire table
ALTER TABLE customers ADD SEARCH OPTIMIZATION;

-- Enable on specific columns (more efficient)
ALTER TABLE customers ADD SEARCH OPTIMIZATION 
ON EQUALITY(customer_id), SUBSTRING(customer_name);

-- Check optimization status
SELECT * FROM TABLE(INFORMATION_SCHEMA.SEARCH_OPTIMIZATION_HISTORY(
    TABLE_NAME => 'customers'
));
```

**How It Works:**
1. Creates a search access path that maps values to micro-partitions
2. Maintenance service updates the path as data changes
3. Queries check the path to identify which partitions to scan
4. Significantly reduces data scanned for selective queries

**When NOT to Use:**
- Analytical queries scanning large data ranges
- Small tables (overhead outweighs benefits)
- Columns with very low cardinality
- Tables with constant bulk updates

**Cost Considerations:**
- Storage cost for the search access path
- Serverless compute for maintenance
- Monitor with SEARCH_OPTIMIZATION_HISTORY function

**Example:**
```sql
-- Without SOS: Full table scan (100M rows)
SELECT * FROM customers WHERE email = 'john@example.com';
-- With SOS: Scans only relevant partitions (~100 rows)
```

**Key Points:**
- Best for OLTP-style point lookups on large tables
- Works transparently—no query changes needed
- Combine with clustering for maximum optimization

---

## Data Loading & Transformation

### Q9: Compare COPY INTO and Snowpipe for data loading. When would you use each?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Data Loading` `Snowpipe`

**Question:**
Explain the differences between COPY INTO and Snowpipe, and provide guidance on when to use each approach.

**Answer:**

| Feature | COPY INTO | Snowpipe |
|---------|-----------|----------|
| **Trigger** | Manual/scheduled execution | Event-driven (automatic) |
| **Latency** | Batch (minutes to hours) | Near real-time (seconds to minutes) |
| **Compute** | Virtual warehouse | Serverless |
| **Best for** | Bulk historical loads | Continuous streaming data |
| **Cost model** | Warehouse credits | Per-file serverless credits |

**COPY INTO - Batch Loading:**
```sql
-- Stage files and load in bulk
COPY INTO my_table
FROM @my_stage/data/
FILE_FORMAT = (TYPE = 'PARQUET')
PATTERN = '.*[.]parquet'
ON_ERROR = 'CONTINUE';

-- With transformation during load
COPY INTO my_table (id, name, created_at)
FROM (
    SELECT $1:id, $1:name, CURRENT_TIMESTAMP()
    FROM @my_stage/data/
)
FILE_FORMAT = (TYPE = 'JSON');
```

**Snowpipe - Continuous Loading:**
```sql
-- Create pipe with auto-ingest
CREATE PIPE my_pipe
AUTO_INGEST = TRUE
AS
COPY INTO my_table
FROM @my_stage/incoming/
FILE_FORMAT = (TYPE = 'JSON');

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('my_pipe');

-- View load history
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'my_table', 
    START_TIME => DATEADD(hours, -24, CURRENT_TIMESTAMP())
));
```

**When to Use COPY INTO:**
- Initial bulk data migrations
- Scheduled batch loads (hourly, daily)
- Loading historical data
- When you need transformation during load
- Cost-sensitive scenarios with predictable volumes

**When to Use Snowpipe:**
- Real-time or near-real-time requirements
- Continuous streaming from sources like Kafka, S3 events
- Unpredictable file arrival patterns
- When you need automatic ingestion without scheduling

**Snowpipe Streaming (High-Performance):**
```sql
-- For highest throughput streaming scenarios
-- Uses Snowflake Ingest SDK
-- Sub-second latency possible
```

**Key Points:**
- COPY INTO uses warehouse credits; Snowpipe uses serverless credits
- Snowpipe has a small file overhead—better for many small files
- Both track loaded files to prevent duplicates

---

### Q10: How do you handle semi-structured data (JSON, Parquet, Avro) in Snowflake?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Semi-structured Data` `VARIANT`

**Question:**
Explain how to load, query, and optimize semi-structured data in Snowflake.

**Answer:**
Snowflake natively supports semi-structured data through the VARIANT data type, which can store JSON, Avro, ORC, Parquet, and XML up to 128 MB per value (increased from 16 MB in 2025).

**Loading Semi-Structured Data:**
```sql
-- Create table with VARIANT column
CREATE TABLE events (
    event_id INT,
    event_data VARIANT,
    load_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Load JSON files
COPY INTO events (event_id, event_data)
FROM (
    SELECT $1:id::INT, $1
    FROM @my_stage/events/
)
FILE_FORMAT = (TYPE = 'JSON');

-- Load Parquet (columnar, highly compressed)
COPY INTO events
FROM @my_stage/data/
FILE_FORMAT = (TYPE = 'PARQUET');
```

**Querying VARIANT Data:**
```sql
-- Dot notation for nested access
SELECT 
    event_data:user:id::STRING AS user_id,
    event_data:user:email::STRING AS email,
    event_data:items[0]:product_name::STRING AS first_product
FROM events;

-- FLATTEN for arrays
SELECT 
    e.event_id,
    item.value:product_id::STRING AS product_id,
    item.value:quantity::INT AS quantity
FROM events e,
LATERAL FLATTEN(input => e.event_data:items) item;

-- Recursive FLATTEN for deeply nested structures
SELECT *
FROM events,
LATERAL FLATTEN(input => event_data, RECURSIVE => TRUE) f
WHERE f.key = 'error_code';
```

**Optimization Techniques:**

1. **Extract frequently queried fields to columns:**
```sql
ALTER TABLE events ADD COLUMN user_id STRING 
AS (event_data:user:id::STRING);
```

2. **Enable Search Optimization on VARIANT paths:**
```sql
ALTER TABLE events ADD SEARCH OPTIMIZATION 
ON EQUALITY(event_data:user:id);
```

3. **Use OBJECT_CONSTRUCT for building JSON:**
```sql
SELECT OBJECT_CONSTRUCT(
    'user_id', user_id,
    'total', SUM(amount),
    'orders', ARRAY_AGG(OBJECT_CONSTRUCT('id', order_id, 'date', order_date))
) AS user_summary
FROM orders
GROUP BY user_id;
```

**Key Points:**
- VARIANT stores data efficiently with automatic type detection
- Parquet is most efficient for analytical workloads
- Extract hot paths to typed columns for best performance

**Common Mistakes:**
- Not casting VARIANT values (returns VARIANT, not expected type)
- Forgetting the double-colon cast syntax (::STRING, ::INT)
- Over-extracting columns instead of querying VARIANT directly

---

## DBT Integration

### Q11: What are the different incremental strategies in dbt for Snowflake, and when should you use each?
**Difficulty:** Advanced  
**Topics:** `dbt` `Snowflake` `Incremental Models`

**Question:**
Explain the merge, delete+insert, and append strategies for dbt incremental models on Snowflake.

**Answer:**
dbt provides several incremental strategies for Snowflake, each with distinct trade-offs:

**1. Merge Strategy (Default)**
Uses Snowflake's MERGE INTO statement to handle inserts and updates atomically.

```sql
-- models/orders_incremental.sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

SELECT 
    order_id,
    customer_id,
    order_date,
    amount,
    updated_at
FROM {{ source('raw', 'orders') }}
{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**Best for:**
- Tables with updates (SCD Type 1)
- Deduplication requirements
- Smaller incremental batches

**2. Delete+Insert Strategy**
Deletes matching records first, then inserts new records.

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='delete+insert'
) }}
```

**Best for:**
- Large batch updates
- Partitioned data (can be more efficient than merge)
- When you need to replace entire partitions

**3. Append Strategy (Insert-Only)**
Simply appends new records without checking for duplicates.

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='append'
) }}

SELECT * FROM {{ source('raw', 'events') }}
{% if is_incremental() %}
WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

**Best for:**
- Immutable event logs
- Time-series data
- When source data never updates

**Advanced: Incremental Predicates**
Limit the scan on the target table for better performance:

```yaml
# models/schema.yml
models:
  - name: my_incremental_model
    config:
      materialized: incremental
      unique_key: id
      incremental_strategy: merge
      cluster_by: ['event_date']
      incremental_predicates:
        - "DBT_INTERNAL_DEST.event_date > dateadd(day, -7, current_date)"
```

**Performance Comparison:**
| Strategy | Insert Performance | Update Performance | Best For |
|----------|-------------------|-------------------|----------|
| Merge | Moderate | Good | Mixed workloads |
| Delete+Insert | Good | Good (batch) | Large partitions |
| Append | Excellent | N/A | Event streams |

**Key Points:**
- Always define `unique_key` for merge and delete+insert
- Use `incremental_predicates` to limit target table scans
- Append is fastest but doesn't handle updates

---

### Q12: How do you implement Slowly Changing Dimensions (SCD) Type 2 in dbt?
**Difficulty:** Advanced  
**Topics:** `dbt` `Snowflake` `SCD`

**Question:**
Explain how to implement SCD Type 2 using dbt snapshots with Snowflake.

**Answer:**
dbt snapshots track historical changes to records, perfect for implementing SCD Type 2 dimensions.

**Snapshot Configuration:**
```sql
-- snapshots/customer_snapshot.sql
{% snapshot customers_snapshot %}

{{
    config(
        target_database='analytics',
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='updated_at',
        invalidate_hard_deletes=True
    )
}}

SELECT
    customer_id,
    customer_name,
    email,
    segment,
    updated_at
FROM {{ source('raw', 'customers') }}

{% endsnapshot %}
```

**Generated Columns:**
- `dbt_scd_id`: Unique identifier for each version
- `dbt_updated_at`: When the record version was created
- `dbt_valid_from`: Start of validity period
- `dbt_valid_to`: End of validity period (NULL = current)

**Strategy Options:**

1. **Timestamp Strategy** (recommended):
```sql
strategy='timestamp',
updated_at='updated_at'  -- Column that changes when record updates
```

2. **Check Strategy** (when no timestamp available):
```sql
strategy='check',
check_cols=['name', 'email', 'status']  -- Columns to monitor for changes
-- Or check all columns:
check_cols='all'
```

**Querying SCD Type 2:**
```sql
-- Current records only
SELECT * FROM {{ ref('customers_snapshot') }}
WHERE dbt_valid_to IS NULL;

-- Point-in-time query
SELECT * FROM {{ ref('customers_snapshot') }}
WHERE '2024-06-15'::DATE BETWEEN dbt_valid_from AND COALESCE(dbt_valid_to, '9999-12-31');

-- Full history for a customer
SELECT * FROM {{ ref('customers_snapshot') }}
WHERE customer_id = 'C123'
ORDER BY dbt_valid_from;
```

**Handling Hard Deletes:**
```sql
invalidate_hard_deletes=True  -- Closes records that disappear from source
```

**Best Practices:**
1. Run snapshots frequently (at least daily)
2. Use timestamp strategy when possible (more efficient)
3. Create a view for current records only
4. Add tests for unique `dbt_scd_id` and non-null `dbt_valid_from`

**Key Points:**
- Snapshots maintain full history automatically
- Run `dbt snapshot` before other transformations
- Use timestamp strategy for better performance

---

### Q13: What are dbt best practices for Snowflake project structure and materialization?
**Difficulty:** Intermediate  
**Topics:** `dbt` `Snowflake` `Best Practices`

**Question:**
Describe the recommended project structure and materialization strategy for dbt with Snowflake.

**Answer:**

**Recommended Directory Structure:**
```
dbt_project/
├── models/
│   ├── staging/           # Views - light transformations
│   │   ├── stg_customers.sql
│   │   └── _staging__models.yml
│   ├── intermediate/      # Views/Ephemeral - business logic
│   │   └── int_orders_enriched.sql
│   └── marts/            # Tables - final models
│       ├── core/
│       │   └── dim_customers.sql
│       └── finance/
│           └── fct_revenue.sql
├── snapshots/            # SCD Type 2
├── macros/               # Reusable SQL
├── tests/                # Custom tests
└── dbt_project.yml
```

**Materialization Strategy:**
```yaml
# dbt_project.yml
models:
  my_project:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: ephemeral  # Or view
    marts:
      +materialized: table
      +schema: marts
      core:
        +materialized: incremental
        +incremental_strategy: merge
```

**Materialization Guidelines:**
| Layer | Materialization | Rationale |
|-------|----------------|-----------|
| Staging | View | Always fresh, minimal storage |
| Intermediate | Ephemeral/View | Reduce duplication |
| Marts (small) | Table | Fast queries |
| Marts (large) | Incremental | Efficient updates |

**Snowflake-Specific Configurations:**
```sql
{{ config(
    materialized='incremental',
    cluster_by=['order_date'],
    query_tag='dbt_orders_model',
    transient=true  -- For non-critical tables
) }}
```

**Separate Dev and Prod:**
```yaml
# profiles.yml
my_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      schema: "{{ env_var('DBT_USER') }}_dev"
      warehouse: DEV_WH
    prod:
      type: snowflake
      schema: prod
      warehouse: PROD_WH
```

**Key Points:**
- Views for staging (always fresh, no storage cost)
- Incremental for large fact tables
- Use `generate_schema_name` macro for environment separation
- Set `transient: true` for intermediate tables to save storage

---

## Security & Governance

### Q14: Explain Snowflake's Role-Based Access Control (RBAC) model.
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Security` `RBAC`

**Question:**
Describe Snowflake's RBAC hierarchy and best practices for implementing access control.

**Answer:**
Snowflake uses a hierarchical RBAC model where privileges are granted to roles, and roles are granted to users or other roles.

**System-Defined Roles:**
```
ORGADMIN (organization-level)
    └── ACCOUNTADMIN (account-level, most privileged)
            ├── SECURITYADMIN (manages grants and users)
            │       └── USERADMIN (manages users and roles)
            └── SYSADMIN (manages all objects)
                    └── PUBLIC (default role for all users)
```

**Creating a Role Hierarchy:**
```sql
-- Create functional roles
CREATE ROLE data_engineer;
CREATE ROLE data_analyst;
CREATE ROLE data_scientist;

-- Create object access roles
CREATE ROLE raw_data_reader;
CREATE ROLE analytics_reader;
CREATE ROLE analytics_writer;

-- Build hierarchy
GRANT ROLE raw_data_reader TO ROLE data_engineer;
GRANT ROLE analytics_reader TO ROLE data_analyst;
GRANT ROLE analytics_writer TO ROLE data_scientist;

-- Grant to system hierarchy
GRANT ROLE data_engineer TO ROLE sysadmin;
GRANT ROLE data_analyst TO ROLE sysadmin;
GRANT ROLE data_scientist TO ROLE sysadmin;

-- Grant to users
GRANT ROLE data_analyst TO USER john_smith;
```

**Best Practices:**

1. **Principle of Least Privilege:**
```sql
-- Grant specific privileges, not ownership
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE data_analyst;
GRANT USAGE ON DATABASE analytics TO ROLE data_analyst;
GRANT USAGE ON SCHEMA analytics.public TO ROLE data_analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.public TO ROLE data_analyst;
```

2. **Use Future Grants:**
```sql
-- Auto-grant on future objects
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.public TO ROLE data_analyst;
```

3. **Managed Access Schemas:**
```sql
-- Only schema owner can grant privileges
CREATE SCHEMA sensitive_data WITH MANAGED ACCESS;
```

4. **Service Accounts:**
```sql
-- Create dedicated roles for applications
CREATE ROLE etl_service;
CREATE USER etl_user PASSWORD = '...' DEFAULT_ROLE = etl_service;
GRANT ROLE etl_service TO USER etl_user;
```

**Key Points:**
- Never use ACCOUNTADMIN for regular operations
- Create role hierarchies that mirror organizational structure
- Use FUTURE GRANTS for automated privilege management

---

### Q15: How do you implement Row-Level Security and Dynamic Data Masking in Snowflake?
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Security` `RLS` `Masking`

**Question:**
Explain how to implement fine-grained access control using Row Access Policies and Masking Policies.

**Answer:**

**Row Access Policies (Row-Level Security):**
Restrict which rows users can see based on their role or attributes.

```sql
-- Create mapping table for entitlements
CREATE TABLE security.region_access (
    role_name VARCHAR,
    region VARCHAR
);

INSERT INTO security.region_access VALUES
    ('SALES_EAST', 'East'),
    ('SALES_WEST', 'West'),
    ('SALES_MANAGER', 'East'),
    ('SALES_MANAGER', 'West');

-- Create row access policy
CREATE OR REPLACE ROW ACCESS POLICY sales_region_policy
AS (region_col VARCHAR) RETURNS BOOLEAN ->
    CURRENT_ROLE() = 'ADMIN'
    OR EXISTS (
        SELECT 1 FROM security.region_access
        WHERE role_name = CURRENT_ROLE()
        AND region = region_col
    );

-- Apply policy to table
ALTER TABLE sales.orders 
ADD ROW ACCESS POLICY sales_region_policy ON (region);
```

**Dynamic Data Masking (Column-Level Security):**
Mask sensitive data based on user roles.

```sql
-- Create masking policy for emails
CREATE OR REPLACE MASKING POLICY email_mask
AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'PII_VIEWER') THEN val
        WHEN CURRENT_ROLE() IN ('ANALYST') THEN REGEXP_REPLACE(val, '.+@', '***@')
        ELSE '***MASKED***'
    END;

-- Create masking policy for SSN
CREATE OR REPLACE MASKING POLICY ssn_mask
AS (val STRING) RETURNS STRING ->
    CASE
        WHEN IS_ROLE_IN_SESSION('PII_ADMIN') THEN val
        ELSE 'XXX-XX-' || RIGHT(val, 4)
    END;

-- Apply policies to columns
ALTER TABLE customers 
MODIFY COLUMN email SET MASKING POLICY email_mask;

ALTER TABLE customers 
MODIFY COLUMN ssn SET MASKING POLICY ssn_mask;
```

**Conditional Masking (Multiple Columns):**
```sql
-- Mask based on another column's value
CREATE OR REPLACE MASKING POLICY conditional_email_mask
AS (email VARCHAR, visibility VARCHAR) RETURNS VARCHAR ->
    CASE
        WHEN CURRENT_ROLE() = 'ADMIN' THEN email
        WHEN visibility = 'public' THEN email
        ELSE '***MASKED***'
    END;

-- Apply with two columns
ALTER TABLE contacts 
MODIFY COLUMN email 
SET MASKING POLICY conditional_email_mask 
USING (email, visibility_status);
```

**Testing Policies:**
```sql
-- Simulate query with different role
SELECT POLICY_CONTEXT('email_mask', 'ANALYST', 'test@example.com');

-- Test row access
SELECT * FROM sales.orders;  -- Results vary by role
```

**Key Points:**
- Row access policies filter rows; masking policies transform columns
- Use mapping tables for flexible entitlements
- Test with `IS_ROLE_IN_SESSION()` for role hierarchy awareness
- Same column cannot have both row access and masking policy

---

## Performance & Cost Optimization

### Q16: What strategies would you use to optimize Snowflake costs?
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Cost Optimization` `Credits`

**Question:**
Describe a comprehensive approach to reducing Snowflake spend while maintaining performance.

**Answer:**

**1. Virtual Warehouse Optimization:**

```sql
-- Right-size warehouses based on workload
ALTER WAREHOUSE etl_wh SET 
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 60           -- Suspend after 1 minute idle
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3
    SCALING_POLICY = 'ECONOMY'; -- Use ECONOMY for cost savings

-- Create workload-specific warehouses
CREATE WAREHOUSE reporting_wh
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 120;

CREATE WAREHOUSE etl_wh
    WAREHOUSE_SIZE = 'LARGE'
    AUTO_SUSPEND = 60;
```

**2. Resource Monitors:**
```sql
-- Set up alerts and limits
CREATE RESOURCE MONITOR monthly_limit
    WITH CREDIT_QUOTA = 1000
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND;

ALTER WAREHOUSE analytics_wh 
SET RESOURCE_MONITOR = monthly_limit;
```

**3. Storage Optimization:**
```sql
-- Use transient tables for non-critical data
CREATE TRANSIENT TABLE staging.temp_data (...);

-- Reduce Time Travel retention where appropriate
ALTER TABLE staging.events 
SET DATA_RETENTION_TIME_IN_DAYS = 1;

-- Clean up old clones
DROP TABLE old_backup_clone;

-- Monitor storage
SELECT 
    TABLE_CATALOG,
    TABLE_SCHEMA,
    TABLE_NAME,
    BYTES / (1024*1024*1024) AS SIZE_GB,
    TIME_TRAVEL_BYTES / (1024*1024*1024) AS TIME_TRAVEL_GB
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
ORDER BY BYTES DESC
LIMIT 20;
```

**4. Query Optimization:**
```sql
-- Identify expensive queries
SELECT 
    query_id,
    query_text,
    total_elapsed_time / 1000 AS seconds,
    credits_used_cloud_services,
    bytes_scanned / (1024*1024*1024) AS gb_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC
LIMIT 20;

-- Add clustering where beneficial
ALTER TABLE large_fact_table CLUSTER BY (date_key);
```

**5. Serverless Feature Review:**
```sql
-- Check automatic clustering costs
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time > DATEADD(day, -30, CURRENT_TIMESTAMP());

-- Consider disabling for rarely queried tables
ALTER TABLE archive_data SUSPEND RECLUSTER;
```

**Credit Consumption by Warehouse Size:**
| Size | Credits/Hour | Use Case |
|------|-------------|----------|
| X-Small | 1 | Light queries, dev |
| Small | 2 | Standard reporting |
| Medium | 4 | Moderate ETL |
| Large | 8 | Complex transforms |
| X-Large | 16 | Heavy analytics |

**Key Points:**
- Auto-suspend is the #1 cost saver—set aggressively
- Match warehouse size to workload complexity
- Use Resource Monitors to prevent runaway costs
- Regularly review and clean up unused objects

---

### Q17: How do you monitor and troubleshoot warehouse queuing issues?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Performance` `Concurrency`

**Question:**
Explain how to identify and resolve query queuing problems in Snowflake.

**Answer:**
Query queuing occurs when a warehouse cannot process all concurrent queries, causing them to wait.

**Identifying Queuing Issues:**
```sql
-- Check warehouse load history
SELECT 
    start_time,
    warehouse_name,
    AVG(avg_running) AS avg_running_queries,
    AVG(avg_queued_load) AS avg_queued_queries,
    AVG(avg_blocked) AS avg_blocked_queries
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY 1, 2
HAVING AVG(avg_queued_load) > 0
ORDER BY start_time DESC;

-- Find queries that were queued
SELECT 
    query_id,
    warehouse_name,
    queued_provisioning_time,
    queued_overload_time,
    total_elapsed_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE queued_overload_time > 0
AND start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
ORDER BY queued_overload_time DESC;
```

**Solutions:**

**1. Multi-Cluster Warehouses (Enterprise Edition):**
```sql
-- Auto-scale based on demand
ALTER WAREHOUSE analytics_wh SET
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 4
    SCALING_POLICY = 'STANDARD';  -- Add clusters proactively
    -- or 'ECONOMY' to minimize clusters
```

**2. Workload Separation:**
```sql
-- Separate warehouses by workload type
CREATE WAREHOUSE bi_reporting_wh WAREHOUSE_SIZE = 'MEDIUM';
CREATE WAREHOUSE etl_loading_wh WAREHOUSE_SIZE = 'LARGE';
CREATE WAREHOUSE adhoc_queries_wh WAREHOUSE_SIZE = 'SMALL';

-- Assign users/applications to specific warehouses
ALTER USER bi_service SET DEFAULT_WAREHOUSE = bi_reporting_wh;
```

**3. Query Optimization:**
```sql
-- Reduce query runtime to free up resources faster
-- Add clustering, materialized views, or optimize SQL
```

**4. Statement Queuing Parameters:**
```sql
-- Set timeout for queued queries
ALTER WAREHOUSE my_wh SET 
    STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes max queue time

-- Set timeout for running queries  
ALTER WAREHOUSE my_wh SET
    STATEMENT_TIMEOUT_IN_SECONDS = 3600;  -- 1 hour max runtime
```

**Key Points:**
- Monitor `WAREHOUSE_LOAD_HISTORY` regularly
- Multi-cluster warehouses handle concurrency automatically
- Separate workloads to prevent interference
- Queue timeouts prevent runaway waiting

---

## Advanced Features

### Q18: Compare Dynamic Tables, Streams and Tasks, and Materialized Views for data pipelines.
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Dynamic Tables` `Streams` `Tasks`

**Question:**
When would you use each approach for building data pipelines in Snowflake?

**Answer:**

**Feature Comparison:**

| Feature | Dynamic Tables | Streams + Tasks | Materialized Views |
|---------|---------------|-----------------|-------------------|
| **Setup Complexity** | Low (declarative) | High (procedural) | Low |
| **Refresh Control** | Target lag | Scheduled/event-driven | Automatic |
| **Flexibility** | Medium | High | Low |
| **CDC Support** | Automatic | Manual via streams | No |
| **Multi-table Joins** | Yes | Yes | Limited |
| **Cost Model** | Warehouse/serverless | Warehouse/serverless | Serverless |

**Dynamic Tables (Recommended for most pipelines):**
```sql
-- Declarative pipeline - just define the result
CREATE OR REPLACE DYNAMIC TABLE customer_orders
    TARGET_LAG = '10 minutes'
    WAREHOUSE = transform_wh
AS
SELECT 
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- Chain dynamic tables
CREATE DYNAMIC TABLE customer_segments
    TARGET_LAG = DOWNSTREAM  -- Refresh when downstream needs it
    WAREHOUSE = transform_wh
AS
SELECT *,
    CASE WHEN total_spent > 10000 THEN 'VIP' ELSE 'Standard' END AS segment
FROM customer_orders;
```

**Streams + Tasks (Complex transformations):**
```sql
-- Create stream for CDC
CREATE OR REPLACE STREAM orders_stream ON TABLE raw_orders;

-- Create task for processing
CREATE OR REPLACE TASK process_orders
    WAREHOUSE = etl_wh
    SCHEDULE = 'USING CRON 0 * * * * UTC'  -- Hourly
    WHEN SYSTEM$STREAM_HAS_DATA('orders_stream')
AS
MERGE INTO processed_orders t
USING (
    SELECT * FROM orders_stream
    WHERE METADATA$ACTION = 'INSERT'
) s
ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...;

-- Start the task
ALTER TASK process_orders RESUME;
```

**Materialized Views (Simple aggregations):**
```sql
-- Best for single-table aggregations
CREATE MATERIALIZED VIEW daily_sales_mv AS
SELECT 
    order_date,
    SUM(amount) AS total_sales,
    COUNT(*) AS order_count
FROM orders
GROUP BY order_date;
```

**When to Use Each:**

| Scenario | Recommended Approach |
|----------|---------------------|
| Multi-table joins with freshness SLA | Dynamic Tables |
| Complex CDC logic (deletes, SCD2) | Streams + Tasks |
| Simple single-table aggregation | Materialized View |
| Real-time requirements (<1 min) | Streams + Tasks |
| Minimal maintenance desired | Dynamic Tables |
| Need procedural logic | Tasks with Stored Procedures |

**Key Points:**
- Dynamic Tables reduce pipeline complexity significantly
- Streams + Tasks offer most flexibility for complex scenarios
- Materialized Views are limited but fully automatic
- Dynamic Tables can source from Streams for hybrid approaches

---

### Q19: Explain Time Travel and Fail-safe in Snowflake.
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Time Travel` `Fail-safe` `Data Recovery`

**Question:**
How do Time Travel and Fail-safe work together for data protection, and what are their limitations?

**Answer:**

**Time Travel (User-Controlled Recovery):**
Allows accessing historical data within a retention period.

```sql
-- Query data from a specific timestamp
SELECT * FROM orders
AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);

-- Query data from before a specific time
SELECT * FROM orders
BEFORE(TIMESTAMP => DATEADD(hours, -2, CURRENT_TIMESTAMP()));

-- Query data from a specific statement (before changes)
SELECT * FROM orders
BEFORE(STATEMENT => '01a2b3c4-5678-90ab-cdef-1234567890ab');

-- Restore dropped table
UNDROP TABLE accidentally_dropped_table;

-- Restore to previous state
CREATE TABLE orders_restored CLONE orders
AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);
```

**Retention Periods:**
| Edition | Permanent Tables | Transient/Temporary |
|---------|------------------|---------------------|
| Standard | 0-1 days | 0-1 days |
| Enterprise | 0-90 days | 0-1 days |

```sql
-- Set retention period
ALTER TABLE critical_data 
SET DATA_RETENTION_TIME_IN_DAYS = 90;

-- Check current setting
SHOW TABLES LIKE 'critical_data';
```

**Fail-safe (Snowflake-Managed Recovery):**
- Non-configurable 7-day period AFTER Time Travel expires
- Only Snowflake Support can recover data
- Only applies to permanent tables
- Used for disaster recovery scenarios

**Data Protection Timeline:**
```
|---- Active Data ----|---- Time Travel (1-90 days) ----|---- Fail-safe (7 days) ----|
                      ^                                  ^                            ^
                 Changes made                    Time Travel expires          Data permanently deleted
```

**Storage Cost Implications:**
```sql
-- Monitor Time Travel storage
SELECT 
    TABLE_NAME,
    ACTIVE_BYTES / (1024*1024*1024) AS ACTIVE_GB,
    TIME_TRAVEL_BYTES / (1024*1024*1024) AS TIME_TRAVEL_GB,
    FAILSAFE_BYTES / (1024*1024*1024) AS FAILSAFE_GB
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE TIME_TRAVEL_BYTES > 0
ORDER BY TIME_TRAVEL_BYTES DESC;
```

**Best Practices:**
1. Set appropriate retention based on recovery needs
2. Use shorter retention for staging/transient data
3. Create periodic clones for long-term backups
4. Monitor Time Travel storage costs

**Key Points:**
- Time Travel: Self-service recovery within retention period
- Fail-safe: Emergency recovery via Snowflake Support
- Transient tables skip Fail-safe (lower cost, less protection)
- Zero-Copy Cloning can extend protection beyond 90 days

---

## Python & Airflow Integration

### Q20: How do you connect Python to Snowflake and what are best practices?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Python` `Connector`

**Question:**
Explain the options for connecting Python to Snowflake and when to use each.

**Answer:**

**1. Snowflake Python Connector:**
```python
import snowflake.connector

# Basic connection
conn = snowflake.connector.connect(
    user='my_user',
    password='my_password',
    account='my_account.us-east-1',
    warehouse='compute_wh',
    database='my_db',
    schema='my_schema'
)

# Execute query
cursor = conn.cursor()
cursor.execute("SELECT * FROM customers LIMIT 10")
rows = cursor.fetchall()

# With pandas
import pandas as pd
cursor.execute("SELECT * FROM customers")
df = cursor.fetch_pandas_all()

# Parameterized queries (prevent SQL injection)
cursor.execute(
    "SELECT * FROM customers WHERE region = %s",
    ('East',)
)

# Bulk loading from pandas
from snowflake.connector.pandas_tools import write_pandas
write_pandas(conn, df, 'target_table')

conn.close()
```

**2. SQLAlchemy Integration:**
```python
from sqlalchemy import create_engine
import pandas as pd

# Connection string
engine = create_engine(
    'snowflake://{user}:{password}@{account}/{database}/{schema}?warehouse={warehouse}'.format(
        user='my_user',
        password='my_password',
        account='my_account',
        database='my_db',
        schema='my_schema',
        warehouse='compute_wh'
    )
)

# Read with pandas
df = pd.read_sql("SELECT * FROM customers", engine)

# Write with pandas
df.to_sql('new_table', engine, index=False, if_exists='replace')
```

**3. Snowpark Python (For in-Snowflake processing):**
```python
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, sum, avg

# Create session
connection_params = {
    "account": "my_account",
    "user": "my_user",
    "password": "my_password",
    "warehouse": "compute_wh",
    "database": "my_db",
    "schema": "my_schema"
}
session = Session.builder.configs(connection_params).create()

# DataFrame operations (executed in Snowflake)
df = session.table("orders")
result = df.filter(col("status") == "completed") \
           .group_by("customer_id") \
           .agg(sum("amount").alias("total_spent"))

# Write results
result.write.save_as_table("customer_totals")

session.close()
```

**Best Practices:**
1. **Use key-pair authentication for production:**
```python
conn = snowflake.connector.connect(
    user='service_account',
    account='my_account',
    private_key_file='/path/to/rsa_key.p8'
)
```

2. **Connection pooling for applications:**
```python
from snowflake.connector import SnowflakeConnection
# Use connection pools for web applications
```

3. **Use Snowpark for heavy transformations:**
- Pushes computation to Snowflake
- Avoids data movement
- Leverages Snowflake's optimization

**Key Points:**
- Python Connector for simple queries and data extraction
- Snowpark for complex transformations (runs in Snowflake)
- Always use parameterized queries
- Prefer key-pair auth over passwords

---

### Q21: How do you orchestrate Snowflake workflows with Apache Airflow?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Airflow` `Orchestration`

**Question:**
Describe how to use Airflow to orchestrate Snowflake data pipelines.

**Answer:**

**Airflow Snowflake Provider:**
```bash
pip install apache-airflow-providers-snowflake
```

**Connection Setup (Airflow UI or CLI):**
```python
# Connection ID: snowflake_default
# Connection Type: Snowflake
# Host: account_name (without .snowflakecomputing.com)
# Schema: my_schema
# Login: username
# Password: password
# Extra: {"warehouse": "compute_wh", "database": "my_db", "role": "my_role"}
```

**Basic DAG with SnowflakeOperator:**
```python
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.providers.snowflake.transfers.s3_to_snowflake import S3ToSnowflakeOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data_team',
    'depends_on_past': False,
    'email_on_failure': True,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'snowflake_etl_pipeline',
    default_args=default_args,
    schedule_interval='0 6 * * *',  # Daily at 6 AM
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:

    # Load data from S3
    load_raw_data = S3ToSnowflakeOperator(
        task_id='load_raw_data',
        snowflake_conn_id='snowflake_default',
        s3_keys=['data/{{ ds }}/events.parquet'],
        table='raw_events',
        schema='staging',
        stage='my_s3_stage',
        file_format='(TYPE = PARQUET)',
    )

    # Transform data
    transform_data = SnowflakeOperator(
        task_id='transform_events',
        snowflake_conn_id='snowflake_default',
        sql="""
            INSERT INTO analytics.processed_events
            SELECT 
                event_id,
                event_type,
                user_id,
                event_timestamp,
                '{{ ds }}' as load_date
            FROM staging.raw_events
            WHERE event_date = '{{ ds }}'
        """,
    )

    # Run dbt models
    run_dbt = BashOperator(
        task_id='run_dbt_models',
        bash_command='cd /dbt && dbt run --select marts.*',
    )

    # Data quality check
    quality_check = SnowflakeOperator(
        task_id='data_quality_check',
        snowflake_conn_id='snowflake_default',
        sql="""
            SELECT CASE 
                WHEN COUNT(*) > 0 THEN 'PASS' 
                ELSE 'FAIL' 
            END as result
            FROM analytics.processed_events
            WHERE load_date = '{{ ds }}'
        """,
    )

    load_raw_data >> transform_data >> run_dbt >> quality_check
```

**Advanced: Dynamic Warehouse Sizing:**
```python
from airflow.providers.snowflake.hooks.snowflake import SnowflakeHook

def scale_warehouse(size: str, **context):
    hook = SnowflakeHook(snowflake_conn_id='snowflake_default')
    hook.run(f"ALTER WAREHOUSE etl_wh SET WAREHOUSE_SIZE = '{size}'")

scale_up = PythonOperator(
    task_id='scale_up_warehouse',
    python_callable=scale_warehouse,
    op_kwargs={'size': 'LARGE'},
)

scale_down = PythonOperator(
    task_id='scale_down_warehouse',
    python_callable=scale_warehouse,
    op_kwargs={'size': 'SMALL'},
)

scale_up >> heavy_processing_task >> scale_down
```

**Key Points:**
- Use SnowflakeOperator for SQL execution
- S3ToSnowflakeOperator handles COPY INTO automatically
- Template variables like `{{ ds }}` for dynamic dates
- Scale warehouses programmatically for cost optimization

---

## Scenario-Based Questions

### Q22: How would you design a real-time analytics pipeline for e-commerce order data?
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Architecture` `Real-time`

**Question:**
Design an end-to-end pipeline for processing e-commerce orders with near real-time analytics requirements.

**Answer:**

**Architecture Overview:**
```
[Order Events] → [Kafka] → [Snowpipe Streaming] → [Raw Layer]
                                                      ↓
[Dynamic Tables] → [Staging] → [Business Logic] → [Marts]
                                                      ↓
                                              [BI Dashboard]
```

**Implementation:**

**1. Ingestion Layer (Snowpipe Streaming):**
```sql
-- Create landing table
CREATE TABLE raw.order_events (
    event_id STRING,
    order_id STRING,
    customer_id STRING,
    event_type STRING,  -- created, updated, shipped, delivered
    event_data VARIANT,
    event_timestamp TIMESTAMP_NTZ,
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- For Kafka connector or Snowpipe Streaming SDK
-- Events land here with sub-second latency
```

**2. Transformation Layer (Dynamic Tables):**
```sql
-- Staging: Deduplicate and structure events
CREATE OR REPLACE DYNAMIC TABLE staging.orders
    TARGET_LAG = '1 minute'
    WAREHOUSE = transform_wh
AS
WITH ranked_events AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY event_timestamp DESC) as rn
    FROM raw.order_events
    WHERE event_type = 'created' OR event_type = 'updated'
)
SELECT 
    order_id,
    customer_id,
    event_data:items AS items,
    event_data:total_amount::DECIMAL(10,2) AS total_amount,
    event_data:shipping_address AS shipping_address,
    event_timestamp AS last_updated_at
FROM ranked_events
WHERE rn = 1;

-- Business Logic: Enrich with customer data
CREATE OR REPLACE DYNAMIC TABLE intermediate.orders_enriched
    TARGET_LAG = '2 minutes'
    WAREHOUSE = transform_wh
AS
SELECT 
    o.*,
    c.customer_segment,
    c.lifetime_value,
    c.first_order_date
FROM staging.orders o
LEFT JOIN dims.customers c ON o.customer_id = c.customer_id;

-- Marts: Aggregations for dashboards
CREATE OR REPLACE DYNAMIC TABLE marts.hourly_sales
    TARGET_LAG = '5 minutes'
    WAREHOUSE = transform_wh
AS
SELECT 
    DATE_TRUNC('hour', last_updated_at) AS hour,
    customer_segment,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value
FROM intermediate.orders_enriched
WHERE last_updated_at > DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2;
```

**3. Real-time Dashboard (Streamlit in Snowflake):**
```python
# streamlit_app.py
import streamlit as st
from snowflake.snowpark import Session

st.title("Real-time Sales Dashboard")

# Get current metrics
df = session.sql("""
    SELECT * FROM marts.hourly_sales
    WHERE hour > DATEADD(hour, -24, CURRENT_TIMESTAMP())
    ORDER BY hour DESC
""").to_pandas()

# Display metrics
col1, col2, col3 = st.columns(3)
col1.metric("Orders (24h)", df['ORDER_COUNT'].sum())
col2.metric("Revenue (24h)", f"${df['TOTAL_REVENUE'].sum():,.2f}")
col3.metric("Avg Order", f"${df['AVG_ORDER_VALUE'].mean():,.2f}")

# Chart
st.line_chart(df.set_index('HOUR')['TOTAL_REVENUE'])
```

**4. Alerting (Tasks):**
```sql
CREATE OR REPLACE TASK alert_on_anomaly
    WAREHOUSE = monitor_wh
    SCHEDULE = 'USING CRON */5 * * * * UTC'
AS
CALL alert_if_sales_drop();  -- Stored procedure that sends alerts
```

**Key Design Decisions:**
- Snowpipe Streaming for sub-second ingestion latency
- Dynamic Tables for declarative transformations with freshness SLAs
- Separate warehouses for ingestion, transformation, and serving
- Target lag cascade: raw → staging (1 min) → marts (5 min)

---

### Q23: How would you handle a scenario where a team member accidentally deleted critical production data?
**Difficulty:** Intermediate  
**Topics:** `Snowflake` `Time Travel` `Recovery`

**Question:**
Walk through the recovery process for accidentally deleted data.

**Answer:**

**Immediate Response Steps:**

**1. Assess the Situation:**
```sql
-- Find the deletion query
SELECT query_id, query_text, start_time, user_name
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_text ILIKE '%DELETE%critical_table%'
    OR query_text ILIKE '%TRUNCATE%critical_table%'
    OR query_text ILIKE '%DROP%critical_table%'
ORDER BY start_time DESC
LIMIT 10;
```

**2. Recovery Options by Scenario:**

**Scenario A: DELETE statement removed rows**
```sql
-- Find data before deletion using query_id
SELECT * FROM critical_table
BEFORE(STATEMENT => '01a2b3c4-5678-90ab-cdef-1234567890ab');

-- Restore the deleted rows
INSERT INTO critical_table
SELECT * FROM critical_table
BEFORE(STATEMENT => '01a2b3c4-5678-90ab-cdef-1234567890ab')
WHERE id NOT IN (SELECT id FROM critical_table);
```

**Scenario B: TRUNCATE removed all data**
```sql
-- View data before truncate
SELECT * FROM critical_table
BEFORE(STATEMENT => 'truncate_query_id');

-- Create backup and swap
CREATE TABLE critical_table_restored CLONE critical_table
BEFORE(STATEMENT => 'truncate_query_id');

-- Swap tables
ALTER TABLE critical_table SWAP WITH critical_table_restored;
```

**Scenario C: DROP TABLE**
```sql
-- Simply undrop
UNDROP TABLE critical_table;

-- If table was recreated, need to rename first
ALTER TABLE critical_table RENAME TO critical_table_new;
UNDROP TABLE critical_table;
```

**Scenario D: Time Travel expired (within Fail-safe)**
```sql
-- Contact Snowflake Support with:
-- 1. Table name and fully qualified path
-- 2. Approximate time of deletion
-- 3. Query ID if known
-- 4. Business justification
```

**3. Verify Recovery:**
```sql
-- Compare row counts
SELECT 
    (SELECT COUNT(*) FROM critical_table) AS current_count,
    (SELECT COUNT(*) FROM critical_table 
     AT(TIMESTAMP => '2024-01-15 09:00:00'::TIMESTAMP)) AS expected_count;

-- Verify specific records
SELECT * FROM critical_table WHERE id = 'important_id';
```

**4. Prevention Measures:**
```sql
-- Implement row access policies
CREATE ROW ACCESS POLICY prevent_mass_delete
AS (id STRING) RETURNS BOOLEAN ->
    CURRENT_ROLE() = 'ADMIN' 
    OR (SELECT COUNT(*) FROM critical_table) > 1000;

-- Require MFA for destructive operations
ALTER ACCOUNT SET REQUIRE_MFA_FOR_OPERATIONS = TRUE;

-- Create regular backup clones
CREATE TASK daily_backup
    SCHEDULE = 'USING CRON 0 2 * * * UTC'
AS
CREATE OR REPLACE TABLE backups.critical_table_backup 
CLONE production.critical_table;
```

**Key Points:**
- Act quickly—Time Travel has limited retention
- Document the incident and query_id
- Use CLONE with AT clause for full table recovery
- Implement preventive measures post-recovery

---

### Q24: Design a cost-effective data warehouse for a startup with variable workloads.
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Architecture` `Cost Optimization`

**Question:**
How would you architect a Snowflake environment for a startup that needs to minimize costs while handling unpredictable query patterns?

**Answer:**

**Architecture Principles:**
1. Start small, scale as needed
2. Aggressive auto-suspend
3. Workload isolation
4. Leverage free tiers and optimizations

**Implementation:**

**1. Warehouse Strategy:**
```sql
-- Development warehouse (minimal cost)
CREATE WAREHOUSE dev_wh
    WAREHOUSE_SIZE = 'XSMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE;

-- Production queries (auto-scaling)
CREATE WAREHOUSE prod_wh
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 2
    SCALING_POLICY = 'ECONOMY';

-- ETL (sized for batch, quick suspend)
CREATE WAREHOUSE etl_wh
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE;
```

**2. Storage Optimization:**
```sql
-- Use transient tables for staging
CREATE TRANSIENT TABLE staging.raw_events (...);

-- Minimal Time Travel for non-critical data
ALTER TABLE staging.temp_data 
SET DATA_RETENTION_TIME_IN_DAYS = 0;

-- 1-day retention for most tables
ALTER DATABASE analytics 
SET DATA_RETENTION_TIME_IN_DAYS = 1;
```

**3. Resource Monitoring:**
```sql
-- Strict monthly budget
CREATE RESOURCE MONITOR startup_budget
    WITH CREDIT_QUOTA = 500
    FREQUENCY = MONTHLY
    TRIGGERS
        ON 50 PERCENT DO NOTIFY
        ON 80 PERCENT DO NOTIFY
        ON 95 PERCENT DO SUSPEND_IMMEDIATE;

-- Apply to all warehouses
ALTER WAREHOUSE dev_wh SET RESOURCE_MONITOR = startup_budget;
ALTER WAREHOUSE prod_wh SET RESOURCE_MONITOR = startup_budget;
ALTER WAREHOUSE etl_wh SET RESOURCE_MONITOR = startup_budget;
```

**4. Query Optimization:**
```sql
-- Leverage result cache (free)
-- Ensure queries are consistent for cache hits

-- Use materialized views sparingly for critical aggregations
CREATE MATERIALIZED VIEW daily_metrics AS
SELECT date, SUM(revenue) FROM orders GROUP BY date;

-- Avoid expensive serverless features initially
-- (automatic clustering, search optimization)
```

**5. Development Practices:**
```sql
-- Clone production for development (free until modified)
CREATE DATABASE dev_db CLONE prod_db;

-- Use LIMIT during development
SELECT * FROM large_table LIMIT 1000;

-- EXPLAIN before running expensive queries
EXPLAIN SELECT ...;
```

**6. Cost Monitoring Dashboard:**
```sql
-- Weekly cost review query
SELECT 
    DATE_TRUNC('week', start_time) AS week,
    warehouse_name,
    SUM(credits_used) AS total_credits,
    SUM(credits_used) * 3 AS estimated_cost_usd  -- Adjust rate
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;
```

**Estimated Monthly Costs:**
| Component | Usage | Credits | Est. Cost |
|-----------|-------|---------|-----------|
| Dev WH (2h/day) | 60h | 60 | $180 |
| Prod WH (8h/day) | 240h | 480 | $1,440 |
| ETL WH (1h/day) | 30h | 120 | $360 |
| Storage (1TB) | - | - | $23 |
| **Total** | - | ~660 | **~$2,000** |

**Key Points:**
- X-Small warehouses for development
- Aggressive auto-suspend (60 seconds)
- Transient tables for staging data
- Resource monitors to prevent surprises
- Regular cost reviews and optimization

---

### Q25: How would you migrate a 10TB Oracle data warehouse to Snowflake?
**Difficulty:** Advanced  
**Topics:** `Snowflake` `Migration` `Oracle`

**Question:**
Outline your approach to migrating a large Oracle data warehouse to Snowflake with minimal downtime.

**Answer:**

**Migration Phases:**

**Phase 1: Assessment & Planning (2-4 weeks)**
```
├── Inventory existing objects (tables, views, procedures)
├── Identify data volumes and growth patterns
├── Map Oracle data types to Snowflake equivalents
├── Analyze SQL/PL/SQL for conversion needs
├── Define success criteria and rollback plan
└── Estimate timeline and resources
```

**Data Type Mapping:**
| Oracle | Snowflake |
|--------|-----------|
| VARCHAR2 | VARCHAR |
| NUMBER | NUMBER/FLOAT |
| DATE | TIMESTAMP_NTZ |
| CLOB | VARCHAR (128MB limit) |
| BLOB | BINARY |
| RAW | BINARY |

**Phase 2: Schema Migration (1-2 weeks)**
```sql
-- Use Snowflake's SnowConvert or manual conversion
-- Oracle
CREATE TABLE orders (
    order_id NUMBER(10) PRIMARY KEY,
    order_date DATE,
    amount NUMBER(10,2)
);

-- Snowflake
CREATE TABLE orders (
    order_id NUMBER(10) NOT NULL,
    order_date TIMESTAMP_NTZ,
    amount NUMBER(10,2),
    PRIMARY KEY (order_id)  -- Informational only, not enforced
);
```

**Phase 3: Data Migration (2-4 weeks)**

**Option A: Direct Export/Import**
```bash
# Export from Oracle to Parquet (recommended format)
# Using tools like Apache Spark or Oracle Data Pump + conversion

# Stage in cloud storage
aws s3 cp ./oracle_export/ s3://migration-bucket/oracle-data/

# Load into Snowflake
```

```sql
-- Create external stage
CREATE STAGE oracle_migration_stage
    URL = 's3://migration-bucket/oracle-data/'
    CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');

-- Bulk load
COPY INTO orders
FROM @oracle_migration_stage/orders/
FILE_FORMAT = (TYPE = 'PARQUET')
ON_ERROR = 'CONTINUE';
```

**Option B: ETL Tool Migration**
- Fivetran, Airbyte, Matillion
- Handles type conversion automatically
- Built-in CDC for ongoing sync

**Phase 4: Validation (1-2 weeks)**
```sql
-- Row count validation
SELECT 'orders' AS table_name,
    (SELECT COUNT(*) FROM oracle_orders) AS oracle_count,
    (SELECT COUNT(*) FROM snowflake_orders) AS snowflake_count;

-- Aggregate validation
SELECT 
    SUM(amount) AS total_amount,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM orders;

-- Sample row comparison
SELECT * FROM orders WHERE order_id IN (1001, 1002, 1003);
```

**Phase 5: Parallel Run (2-4 weeks)**
```
[Oracle Source] → [CDC Stream] → [Snowflake]
       ↓                              ↓
  [Prod Apps]                  [Shadow Testing]
       ↓                              ↓
  [Compare Results Daily]
```

**Phase 6: Cutover (1 day)**
```
Hour 0: Announce maintenance window
Hour 1: Stop writes to Oracle
Hour 2: Final CDC sync to Snowflake
Hour 3: Validation checks
Hour 4: Switch application connections
Hour 5: Monitor and verify
Hour 6: Declare success or rollback
```

**PL/SQL to Snowflake Conversion:**
```sql
-- Oracle PL/SQL
CREATE OR REPLACE PROCEDURE process_orders IS
BEGIN
    FOR rec IN (SELECT * FROM pending_orders) LOOP
        UPDATE orders SET status = 'PROCESSED' WHERE id = rec.id;
    END LOOP;
END;

-- Snowflake JavaScript/Python
CREATE OR REPLACE PROCEDURE process_orders()
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
    var stmt = snowflake.createStatement({
        sqlText: `UPDATE orders SET status = 'PROCESSED' 
                  WHERE id IN (SELECT id FROM pending_orders)`
    });
    stmt.execute();
    return 'Success';
$$;

-- Or better: Use set-based SQL directly
UPDATE orders 
SET status = 'PROCESSED' 
WHERE id IN (SELECT id FROM pending_orders);
```

**Key Points:**
- Parquet format for fastest, most reliable data transfer
- Validate at every phase before proceeding
- Plan for 2-4 week parallel run
- Have clear rollback procedures
- Convert PL/SQL to set-based operations where possible

---

## Summary

This comprehensive guide covers the essential topics for Snowflake data engineering interviews:

**Beginner Topics (30%):**
- Architecture fundamentals
- Table types and storage
- Basic SQL operations
- Data loading basics

**Intermediate Topics (50%):**
- Query optimization
- Caching mechanisms
- Time Travel and cloning
- dbt integration
- Security and RBAC

**Advanced Topics (20%):**
- Dynamic Tables vs Streams/Tasks
- Cost optimization strategies
- Complex migration scenarios
- Real-time pipeline design

**Key Interview Tips:**
1. Always explain the "why" behind your answers
2. Use specific examples from your experience
3. Discuss trade-offs, not just features
4. Know current Snowflake capabilities (2024-2025 features)
5. Be prepared for scenario-based problem solving

---

*Document Version: 1.0*  
*Last Updated: February 2026*  
*Sources: Snowflake Documentation, Industry Best Practices, Real-world Implementation Experience*
