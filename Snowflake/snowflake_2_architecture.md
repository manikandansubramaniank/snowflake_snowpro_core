# 2. Snowflake Architecture

## Architecture Overview

Snowflake's architecture separates storage, compute, and services into three independent layers enabling elastic scaling, multi-tenancy, and pay-per-use billing. Compute scales horizontally (multi-cluster warehouses) and vertically (warehouse sizes) without impacting storage. Cloud Services manages metadata, authentication, and query optimization across all workloads.

## Three-Layer Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                   CLOUD SERVICES LAYER                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │ Auth &       │ │ Query        │ │ Metadata     │            │
│  │ Access Ctrl  │ │ Optimizer    │ │ Manager      │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │ Transaction  │ │ Security     │ │ Statistics   │            │
│  │ Manager      │ │ Manager      │ │ Collector    │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
└────────────────────────────────────────────────────────────────┘
                             ↕
┌────────────────────────────────────────────────────────────────┐
│               QUERY PROCESSING LAYER (COMPUTE)                 │
│                                                                │
│  Virtual Warehouse 1  │  Virtual Warehouse 2  │  VW 3          │
│  ┌─────────────────┐  │  ┌─────────────────┐  │ ┌──────────┐   │
│  │ Query Exec Eng  │  │  │ Multi-Cluster   │  │ │ Single   │   │
│  │ - Local Cache   │  │  │ ┌────┐ ┌────┐   │  │ │ Cluster  │   │
│  │ - Result Cache  │  │  │ │Cls1│ │Cls2│   │  │ │          │   │
│  │ Size: X-SMALL   │  │  │ └────┘ └────┘   │  │ │ XL       │   │
│  └─────────────────┘  │  └─────────────────┘  │ └──────────┘   │
└────────────────────────────────────────────────────────────────┘
                             ↕
┌────────────────────────────────────────────────────────────────┐
│               DATABASE STORAGE LAYER                           │
│                                                                │
│  Database 1           │  Database 2           │  Database N    │
│  ┌─────────────────┐  │  ┌─────────────────┐  │                │
│  │ Table A         │  │  │ Table X         │  │  Stored in:    │
│  │ ┌─────────────┐ │  │  │ ┌─────────────┐ │  │  - AWS S3      │
│  │ │Micro-Part 1 │ │  │  │ │Micro-Part 1 │ │  │  - Azure       │
│  │ │Micro-Part 2 │ │  │  │ │Micro-Part 2 │ │  │    Blob        │
│  │ │Micro-Part N │ │  │  │ │...          │ │  │  - GCP GCS     │
│  │ └─────────────┘ │  │  │ └─────────────┘ │  │                │
│  └─────────────────┘  │  └─────────────────┘  │                │
└────────────────────────────────────────────────────────────────┘
```

### Layer 1: Cloud Services

Manages infrastructure-level services across all user accounts:
- **Authentication & Authorization**: User login, MFA, SSO (SAML), role-based access control
- **Metadata Management**: Table definitions, statistics, transaction logs, Time Travel metadata
- **Query Optimization**: Parse SQL, generate execution plans, prune micro-partitions
- **Security**: Encryption key management, network policies, data masking enforcement
- **Infrastructure Orchestration**: Resource allocation, load balancing, fault tolerance

### Layer 2: Query Processing (Virtual Warehouses)

Independent compute clusters execute queries without sharing resources:
- **Elasticity**: Resize warehouses (X-Small to 6X-Large) or scale out (multi-cluster)
- **Isolation**: Each warehouse has dedicated CPU/memory, no contention
- **Caching**: Result cache (24 hours), local SSD cache (warehouse-specific)
- **Auto-Suspend/Resume**: Warehouses pause after inactivity (configurable), resume automatically on query
- **Billing**: Per-second compute charges, billed separately from storage

### Layer 3: Database Storage

Columnar, compressed storage layer managed entirely by Snowflake:
- **Micro-Partitions**: Immutable 50-500MB compressed chunks (automatic partitioning)
- **Columnar Format**: Each column stored separately, optimized for analytics
- **Compression**: Automatic compression (avg 10:1 ratio, varies by data type)
- **Cloud Object Storage**: Data persisted in S3/Blob/GCS with 99.999999999% durability
- **Pruning**: Metadata enables skipping irrelevant micro-partitions during queries

---

## Virtual Warehouses

Virtual warehouses (VWs) are named compute clusters executing SQL queries. Each warehouse operates independently with dedicated CPU, memory, and temporary storage. Warehouses can be resized dynamically or scaled horizontally via multi-cluster mode.

### Warehouse Sizes

| Size | Credits/Hour | Servers | Use Case |
|------|--------------|---------|----------|
| **X-Small** | 1 | 1 | Dev/test, infrequent queries |
| **Small** | 2 | 2 | Small datasets, ad-hoc queries |
| **Medium** | 4 | 4 | Moderate workloads, dashboards |
| **Large** | 8 | 8 | Production ETL, heavy queries |
| **X-Large** | 16 | 16 | Complex joins, large scans |
| **2X-Large** | 32 | 32 | Massive datasets, batch processing |
| **3X-Large** | 64 | 64 | Extreme workloads, ML training |
| **4X-Large** | 128 | 128 | Reserved for largest queries |
| **5X-Large** | 256 | 256 | Specialized use cases |
| **6X-Large** | 512 | 512 | Maximum size available |

**Scaling Guidelines:**
- **Scale Up** (larger size): Complex single queries, large joins, aggregations
- **Scale Out** (multi-cluster): High concurrency, many simultaneous users
- **Cost**: Doubling size = 2x credits/hour but often <2x faster (diminishing returns)

### Multi-Cluster Warehouses

Automatically add/remove clusters based on query queue depth:
- **Min Clusters**: Minimum running clusters (cost control)
- **Max Clusters**: Maximum clusters for peak load (concurrency limit)
- **Scaling Policy**: `STANDARD` (favor cost) or `ECONOMY` (favor performance)
- **Use Case**: BI dashboards with unpredictable user load

---

## Micro-Partitions

Snowflake automatically organizes table rows into immutable micro-partitions of 50-500MB (compressed). Each micro-partition stores all columns for its rows in columnar format. Metadata (min/max values, null counts) enables query pruning.

### Micro-Partition Diagram

```
Table: SALES (100M rows, 50GB uncompressed)
┌──────────────────────────────────────────────────┐
│ Micro-Partition 1 (rows 1-500K, 75MB compressed) │
│ ┌─────────┬─────────┬─────────┬─────────┐        │
│ │ ORDER_ID│ CUST_ID │ AMOUNT  │ DATE    │        │
│ │ (INT)   │ (INT)   │(DECIMAL)│ (DATE)  │        │
│ │ Min:1   │ Min:100 │ Min:$10 │Min:2024-│        │
│ │ Max:500K│ Max:9K  │ Max:$5K │Max:2024-│        │
│ └─────────┴─────────┴─────────┴─────────┘        │
└──────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────┐
│ Micro-Partition 2 (rows 500K-1M, 68MB)           │
│ [Column metadata: Min/Max values, nulls, etc.]   │
└──────────────────────────────────────────────────┘
...
┌──────────────────────────────────────────────────┐
│ Micro-Partition 200 (last partition, 71MB)       │
└──────────────────────────────────────────────────┘

Total: 200 micro-partitions, 15GB compressed (10:1 ratio)
```

### Partition Pruning Example

```sql
-- Query: SELECT SUM(amount) FROM sales WHERE date = '2024-12-25';
-- Snowflake scans metadata, prunes 190 partitions, reads only 10 partitions
-- Performance: 95% less data scanned → faster query, lower cost
```

### Key Characteristics
- **Automatic**: No manual partition definition required
- **Immutable**: Once written, micro-partitions never change (new writes create new partitions)
- **Columnar**: Each column stored separately for efficient analytics
- **Metadata-Driven**: Min/max values enable pruning without reading data

---

## Clustering Keys

Clustering keys organize micro-partitions to optimize query performance. Snowflake automatically maintains clustering as data is inserted/updated. Define clustering on high-cardinality columns frequently used in WHERE/JOIN clauses.

### Clustering Depth

Clustering depth measures data distribution quality (lower = better):
- **Depth 0**: Perfect clustering, all matching rows in single micro-partition
- **Depth 1-10**: Good clustering, minimal partition scanning
- **Depth >100**: Poor clustering, consider manual reclustering

### When to Cluster
- Large tables (>1TB) with selective queries
- High-cardinality columns (date, ID) in WHERE clauses
- Queries consistently filter/join on same columns
- Tables with frequent DML (inserts degrading natural clustering)

---

## Syntax: CREATE WAREHOUSE

### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] WAREHOUSE [ IF NOT EXISTS ] <name>
  [ WITH ]
  [ WAREHOUSE_TYPE = STANDARD | SNOWPARK-OPTIMIZED ]
  [ WAREHOUSE_SIZE = XSMALL | SMALL | MEDIUM | LARGE | XLARGE | XXLARGE | XXXLARGE | X4LARGE | X5LARGE | X6LARGE ]
  [ MAX_CLUSTER_COUNT = <num> ]
  [ MIN_CLUSTER_COUNT = <num> ]
  [ SCALING_POLICY = STANDARD | ECONOMY ]
  [ AUTO_SUSPEND = <num> ]
  [ AUTO_RESUME = TRUE | FALSE ]
  [ INITIALLY_SUSPENDED = TRUE | FALSE ]
  [ RESOURCE_MONITOR = <monitor_name> ]
  [ ENABLE_QUERY_ACCELERATION = TRUE | FALSE ]
  [ QUERY_ACCELERATION_MAX_SCALE_FACTOR = <num> ]
  [ MAX_CONCURRENCY_LEVEL = <num> ]
  [ STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = <num> ]
  [ STATEMENT_TIMEOUT_IN_SECONDS = <num> ]
  [ TAG <tag_name> = '<tag_value>' [ , ... ] ]
  [ COMMENT = '<string>' ]
```

### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `OR REPLACE` | Flag | N/A | Replace existing warehouse if exists | `OR REPLACE` |
| `IF NOT EXISTS` | Flag | N/A | Create only if doesn't exist | `IF NOT EXISTS` |
| `name` | String | Required | Warehouse identifier | `analytics_wh` |
| `WAREHOUSE_TYPE` | Enum | STANDARD | Warehouse type: STANDARD or SNOWPARK-OPTIMIZED | `STANDARD` |
| `WAREHOUSE_SIZE` | Enum | XSMALL | Compute size (XSMALL to X6LARGE) | `LARGE` |
| `MAX_CLUSTER_COUNT` | Integer (1-10) | 1 | Maximum clusters for multi-cluster (Enterprise+) | `5` |
| `MIN_CLUSTER_COUNT` | Integer (1-10) | 1 | Minimum running clusters | `1` |
| `SCALING_POLICY` | Enum | STANDARD | `STANDARD` (favor cost) or `ECONOMY` (favor perf) | `ECONOMY` |
| `AUTO_SUSPEND` | Integer | 600 | Seconds of inactivity before suspension (NULL = never) | `300` |
| `AUTO_RESUME` | Boolean | TRUE | Automatically resume on query submission | `TRUE` |
| `INITIALLY_SUSPENDED` | Boolean | FALSE | Create warehouse in suspended state | `TRUE` |
| `RESOURCE_MONITOR` | String | NULL | Assigned resource monitor for cost control | `dev_monitor` |
| `ENABLE_QUERY_ACCELERATION` | Boolean | FALSE | Enable serverless query acceleration | `TRUE` |
| `QUERY_ACCELERATION_MAX_SCALE_FACTOR` | Integer (0-100) | 8 | Max scale factor for acceleration | `16` |
| `MAX_CONCURRENCY_LEVEL` | Integer (1-8) | 8 | Max concurrent queries (statement queue beyond limit) | `4` |
| `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | Integer | 0 | Timeout for queued statements (0 = no timeout) | `1800` |
| `STATEMENT_TIMEOUT_IN_SECONDS` | Integer | 172800 | Timeout for running statements (2 days default) | `3600` |
| `TAG` | Key-Value | NULL | Metadata tags for governance | `cost_center = 'analytics'` |
| `COMMENT` | String | NULL | Warehouse description | `'Production ETL warehouse'` |

### Basic Example

```sql
-- Create simple warehouse with defaults
CREATE WAREHOUSE dev_wh
  WAREHOUSE_SIZE = XSMALL;

-- Verify creation
SHOW WAREHOUSES LIKE 'dev_wh';

-- Use warehouse for queries
USE WAREHOUSE dev_wh;
SELECT CURRENT_WAREHOUSE();
```

### Advanced Example (SDLC: Multi-Environment Warehouse Design)

```sql
-- Production multi-cluster warehouse (high concurrency)
CREATE OR REPLACE WAREHOUSE prod_analytics_wh
  WAREHOUSE_TYPE = STANDARD
  WAREHOUSE_SIZE = LARGE
  MAX_CLUSTER_COUNT = 5
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = ECONOMY  -- Favor performance over cost
  AUTO_SUSPEND = 300        -- Suspend after 5 min idle
  AUTO_RESUME = TRUE
  ENABLE_QUERY_ACCELERATION = TRUE
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 16
  MAX_CONCURRENCY_LEVEL = 8
  STATEMENT_TIMEOUT_IN_SECONDS = 3600
  RESOURCE_MONITOR = prod_monitor
  TAG (
    environment = 'production',
    cost_center = 'analytics',
    sla = '99.9'
  )
  COMMENT = 'Production analytics warehouse - 24x7 uptime';

-- Development warehouse (cost-optimized)
CREATE OR REPLACE WAREHOUSE dev_wh
  WAREHOUSE_SIZE = XSMALL
  AUTO_SUSPEND = 60         -- Aggressive suspension (1 min)
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 900  -- 15 min timeout
  TAG (
    environment = 'development',
    auto_cleanup = 'true'
  )
  COMMENT = 'Development warehouse - auto-suspend enabled';

-- ETL batch processing warehouse (large single queries)
CREATE OR REPLACE WAREHOUSE etl_batch_wh
  WAREHOUSE_SIZE = XXLARGE
  MAX_CLUSTER_COUNT = 1     -- No multi-cluster, large single queries
  AUTO_SUSPEND = 600
  AUTO_RESUME = FALSE       -- Manual resume for scheduled jobs
  INITIALLY_SUSPENDED = TRUE
  RESOURCE_MONITOR = etl_monitor
  TAG (workload_type = 'batch_etl')
  COMMENT = 'Nightly ETL batch processing';

-- BI dashboard warehouse (high concurrency, bursty load)
CREATE OR REPLACE WAREHOUSE bi_dashboard_wh
  WAREHOUSE_SIZE = MEDIUM
  MAX_CLUSTER_COUNT = 10    -- Scale to 10 clusters during peak
  MIN_CLUSTER_COUNT = 2     -- Keep 2 clusters running always
  SCALING_POLICY = STANDARD -- Favor cost, slower scale-out
  AUTO_SUSPEND = 120
  AUTO_RESUME = TRUE
  ENABLE_QUERY_ACCELERATION = TRUE
  TAG (
    workload_type = 'bi_interactive',
    users = '500+'
  )
  COMMENT = 'Self-service BI dashboards - auto-scaling';
```

---

## Syntax: ALTER WAREHOUSE

### Complete Syntax Template

```sql
ALTER WAREHOUSE [ IF EXISTS ] <name>
  { SET
      [ WAREHOUSE_SIZE = { XSMALL | SMALL | ... } ]
      [ MAX_CLUSTER_COUNT = <num> ]
      [ MIN_CLUSTER_COUNT = <num> ]
      [ SCALING_POLICY = { STANDARD | ECONOMY } ]
      [ AUTO_SUSPEND = <num> ]
      [ AUTO_RESUME = { TRUE | FALSE } ]
      [ ENABLE_QUERY_ACCELERATION = { TRUE | FALSE } ]
      [ QUERY_ACCELERATION_MAX_SCALE_FACTOR = <num> ]
      [ MAX_CONCURRENCY_LEVEL = <num> ]
      [ STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = <num> ]
      [ STATEMENT_TIMEOUT_IN_SECONDS = <num> ]
      [ RESOURCE_MONITOR = <monitor_name> ]
      [ TAG <tag_name> = '<tag_value>' [ , ... ] ]
      [ COMMENT = '<string>' ]
  | UNSET {
      TAG <tag_name> [ , ... ]
    | RESOURCE_MONITOR
    | COMMENT
    }
  | SUSPEND
  | RESUME [ IF SUSPENDED ]
  | ABORT ALL QUERIES
  | RENAME TO <new_name>
  }
```

### Parameters

Same as CREATE WAREHOUSE, plus:
- `SUSPEND`: Immediately suspend warehouse (stop compute, keep metadata)
- `RESUME`: Resume suspended warehouse
- `ABORT ALL QUERIES`: Cancel all running queries on warehouse
- `RENAME TO`: Change warehouse name

### Examples

```sql
-- Scale up warehouse for large query
ALTER WAREHOUSE analytics_wh SET WAREHOUSE_SIZE = XLARGE;

-- Enable multi-cluster for peak load
ALTER WAREHOUSE bi_wh SET
  MAX_CLUSTER_COUNT = 10
  MIN_CLUSTER_COUNT = 3
  SCALING_POLICY = ECONOMY;

-- Reduce auto-suspend timeout
ALTER WAREHOUSE dev_wh SET AUTO_SUSPEND = 60;

-- Manually suspend warehouse
ALTER WAREHOUSE etl_wh SUSPEND;

-- Resume warehouse
ALTER WAREHOUSE etl_wh RESUME;

-- Abort all queries (useful for runaway queries)
ALTER WAREHOUSE analytics_wh ABORT ALL QUERIES;
```

---

## Syntax: ALTER TABLE CLUSTER BY

### Complete Syntax Template

```sql
-- Add clustering key
ALTER TABLE <table_name> CLUSTER BY ( <expr1> [ , <expr2> , ... ] );

-- Remove clustering key
ALTER TABLE <table_name> DROP CLUSTERING KEY;

-- Recluster table (manual optimization)
ALTER TABLE <table_name> RECLUSTER [ MAX_SIZE = <num> ] [ WHERE <condition> ];

-- Suspend automatic reclustering
ALTER TABLE <table_name> SUSPEND RECLUSTER;

-- Resume automatic reclustering
ALTER TABLE <table_name> RESUME RECLUSTER;
```

### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `CLUSTER BY` | Expression(s) | NULL | Columns/expressions for clustering | `order_date, region_id` |
| `MAX_SIZE` | Integer (MB) | 1000 | Max data to recluster per operation | `5000` |
| `WHERE` | Condition | NULL | Selective reclustering filter | `WHERE year = 2024` |
| `SUSPEND/RESUME` | Flag | N/A | Control automatic reclustering | `SUSPEND RECLUSTER` |

### Basic Example

```sql
-- Create table
CREATE TABLE sales (
  order_id INT,
  customer_id INT,
  order_date DATE,
  amount DECIMAL(10,2)
);

-- Add clustering key
ALTER TABLE sales CLUSTER BY (order_date);

-- Check clustering depth
SELECT SYSTEM$CLUSTERING_DEPTH('sales', '(order_date)');
-- Returns: {"totalDepth":3,"average":1.2} (good clustering)
```

### Advanced Example

```sql
-- Large table with poor clustering
CREATE TABLE event_logs (
  event_id BIGINT,
  user_id INT,
  event_timestamp TIMESTAMP,
  event_type VARCHAR,
  payload VARIANT
);

-- Insert 10B rows over time (clustering degrades)
-- Check current clustering
SELECT SYSTEM$CLUSTERING_DEPTH('event_logs', '(event_timestamp, event_type)');
-- Returns: {"totalDepth":245} (poor clustering, many partitions scanned)

-- Add composite clustering key
ALTER TABLE event_logs CLUSTER BY (DATE(event_timestamp), event_type);

-- Manual recluster (specific partition)
ALTER TABLE event_logs RECLUSTER
  MAX_SIZE = 10000
  WHERE DATE(event_timestamp) >= '2025-01-01';

-- Monitor reclustering progress
SELECT 
  table_name,
  clustering_key,
  SYSTEM$CLUSTERING_DEPTH(table_name, clustering_key) AS depth
FROM information_schema.tables
WHERE table_name = 'EVENT_LOGS';
```

---

## SDLC Use Case: Elastic Warehouse Scaling in Agile Sprints

### Scenario: Sprint Testing with Dynamic Warehouse Sizing

During 2-week Agile sprints, teams test data pipelines with varying compute needs. Snowflake warehouses scale dynamically to match workload intensity.

### Sprint Workflow

```python
# sprint_warehouse_manager.py
import snowflake.connector
from datetime import datetime

class SprintWarehouseManager:
    def __init__(self, account, user, password, role='SYSADMIN'):
        self.conn = snowflake.connector.connect(
            account=account,
            user=user,
            password=password,
            role=role
        )
        self.cursor = self.conn.cursor()
    
    def provision_sprint_warehouse(self, sprint_number, team_name):
        """Create warehouse for sprint with auto-scaling"""
        wh_name = f"sprint_{sprint_number}_{team_name}_wh"
        
        self.cursor.execute(f"""
            CREATE OR REPLACE WAREHOUSE {wh_name}
              WAREHOUSE_SIZE = XSMALL
              MAX_CLUSTER_COUNT = 3
              MIN_CLUSTER_COUNT = 1
              SCALING_POLICY = ECONOMY
              AUTO_SUSPEND = 300
              AUTO_RESUME = TRUE
              TAG (
                sprint = '{sprint_number}',
                team = '{team_name}',
                created_at = '{datetime.now().isoformat()}'
              )
              COMMENT = 'Sprint {sprint_number} - {team_name} team testing'
        """)
        
        print(f"✅ Created warehouse: {wh_name}")
        return wh_name
    
    def scale_for_load_testing(self, wh_name):
        """Scale up for performance testing phase"""
        self.cursor.execute(f"""
            ALTER WAREHOUSE {wh_name} SET
              WAREHOUSE_SIZE = LARGE
              MAX_CLUSTER_COUNT = 5
        """)
        print(f"📈 Scaled {wh_name} to LARGE with 5-cluster max")
    
    def optimize_for_development(self, wh_name):
        """Return to cost-optimized settings after testing"""
        self.cursor.execute(f"""
            ALTER WAREHOUSE {wh_name} SET
              WAREHOUSE_SIZE = SMALL
              MAX_CLUSTER_COUNT = 2
              AUTO_SUSPEND = 60
        """)
        print(f"💰 Optimized {wh_name} for cost")
    
    def cleanup_sprint(self, sprint_number):
        """Drop warehouse after sprint completion"""
        self.cursor.execute(f"""
            SHOW WAREHOUSES LIKE 'sprint_{sprint_number}%'
        """)
        
        warehouses = self.cursor.fetchall()
        for wh in warehouses:
            wh_name = wh[0]
            self.cursor.execute(f"DROP WAREHOUSE IF EXISTS {wh_name}")
            print(f"🗑️ Dropped warehouse: {wh_name}")

# Usage in CI/CD pipeline
if __name__ == '__main__':
    manager = SprintWarehouseManager(
        account='myorg.us-east-1',
        user='cicd_user',
        password='***'
    )
    
    # Sprint kickoff (Day 1)
    wh = manager.provision_sprint_warehouse(sprint_number=42, team_name='data_eng')
    
    # Load testing phase (Day 7)
    manager.scale_for_load_testing(wh)
    
    # Post-testing optimization (Day 8)
    manager.optimize_for_development(wh)
    
    # Sprint retrospective (Day 14)
    manager.cleanup_sprint(sprint_number=42)
```

### Benefits
- **Cost Control**: XSMALL warehouses for development, scale only when needed
- **Performance**: Scale to LARGE during load testing, multi-cluster for concurrency tests
- **Isolation**: Each sprint gets dedicated warehouse, no resource contention
- **Automation**: CI/CD pipeline manages warehouse lifecycle

---

## Performance Optimization Patterns

### 1. Warehouse Sizing
```sql
-- Anti-pattern: Oversized warehouse for small queries
CREATE WAREHOUSE small_queries_wh WAREHOUSE_SIZE = XXLARGE;  -- ❌ Waste

-- Pattern: Right-size based on workload
CREATE WAREHOUSE small_queries_wh WAREHOUSE_SIZE = SMALL;    -- ✅ Optimal
```

### 2. Multi-Cluster Scaling
```sql
-- Pattern: BI dashboard with peak concurrency
CREATE WAREHOUSE bi_wh
  WAREHOUSE_SIZE = MEDIUM
  MAX_CLUSTER_COUNT = 10  -- Scale out for users
  MIN_CLUSTER_COUNT = 2;  -- Baseline capacity
```

### 3. Auto-Suspend Tuning
```sql
-- Dev: Aggressive suspension (cost)
ALTER WAREHOUSE dev_wh SET AUTO_SUSPEND = 60;

-- Prod: Moderate suspension (performance)
ALTER WAREHOUSE prod_wh SET AUTO_SUSPEND = 600;
```

---

## Key Concepts

- **Virtual Warehouse**: Independent compute cluster with dedicated CPU/memory (scales separately from storage)
- **Micro-Partition**: Immutable 50-500MB columnar chunk with metadata (automatic partitioning, no tuning)
- **Clustering Key**: Column(s) organizing micro-partitions for query pruning (improves large table performance)
- **Multi-Cluster Warehouse**: Auto-scaling warehouse adding/removing clusters based on queue depth (Enterprise feature)
- **Clustering Depth**: Metric measuring partition overlap (lower = better, indicates data distribution quality)

---
