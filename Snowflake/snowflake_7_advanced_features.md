# 7. Advanced Features

## Time Travel

Time Travel enables querying historical data for 0-90 days (Enterprise Edition) using AT or BEFORE clauses. Supports data recovery, auditing, and creating historical reports. Storage costs apply for retained data beyond Fail-Safe period.

### Time Travel Retention

| Edition | Max Retention | Default | Transient Tables |
|---------|---------------|---------|------------------|
| **Standard** | 1 day | 1 day | 0-1 day |
| **Enterprise** | 90 days | 1 day | 0-1 day |
| **Business Critical** | 90 days | 1 day | 0-1 day |

### Syntax: Time Travel Queries

```sql
-- Query table state at specific timestamp
SELECT * FROM orders
  AT(TIMESTAMP => '2025-01-30 10:00:00'::TIMESTAMP_NTZ);

-- Query table state offset from current time
SELECT * FROM orders
  AT(OFFSET => -3600);  -- 1 hour ago (seconds)

-- Query table before specific statement
SELECT * FROM orders
  BEFORE(STATEMENT => '01a9f2b3-0000-1234-0000-0123456789ab');

-- Clone table from historical state
CREATE TABLE orders_snapshot
  CLONE orders
  AT(TIMESTAMP => '2025-01-30 00:00:00'::TIMESTAMP_NTZ);
```

### Time Travel Examples

```sql
-- Basic Time Travel: Query yesterday's data
SELECT COUNT(*) 
FROM sales.fact_orders
  AT(OFFSET => -86400);  -- 24 hours ago

-- Compare current vs historical data
SELECT 
  current_data.order_count AS current_count,
  historical_data.order_count AS yesterday_count,
  current_data.order_count - historical_data.order_count AS delta
FROM (
  SELECT COUNT(*) AS order_count FROM sales.fact_orders
) current_data,
(
  SELECT COUNT(*) FROM sales.fact_orders AT(OFFSET => -86400) AS order_count
) historical_data;

-- Audit trail: Find deleted rows
SELECT * FROM sales.customers
  AT(OFFSET => -3600)  -- 1 hour ago
MINUS
SELECT * FROM sales.customers;
-- Returns: Rows deleted in last hour

-- Restore accidentally deleted data
INSERT INTO sales.customers
SELECT * FROM sales.customers
  AT(TIMESTAMP => '2025-01-31 09:00:00'::TIMESTAMP_NTZ)
WHERE customer_id IN (1001, 1002, 1003);
```

### Configure Retention

```sql
-- Set retention at account level (ACCOUNTADMIN only)
ALTER ACCOUNT SET DATA_RETENTION_TIME_IN_DAYS = 30;

-- Set retention at database level
ALTER DATABASE prod_db SET DATA_RETENTION_TIME_IN_DAYS = 90;

-- Set retention at table level
ALTER TABLE sales.fact_orders 
  SET DATA_RETENTION_TIME_IN_DAYS = 7;

-- View retention settings
SHOW PARAMETERS LIKE 'DATA_RETENTION%' IN TABLE sales.fact_orders;
```

---

## Fail-Safe

Fail-Safe is a 7-day data recovery period after Time Travel expiration. Available only for permanent tables (not transient/temporary). Recovery requires Snowflake Support intervention (not user-accessible). No query access to Fail-Safe data.

### Fail-Safe vs Time Travel

```
Timeline for Permanent Table (7-day Time Travel, 7-day Fail-Safe):

Day 0: Data created
 ↓
Days 0-7: Time Travel Period
- User can query historical data
- User can clone from historical state
- User can undrop tables
 ↓
Days 8-14: Fail-Safe Period
- No user access to data
- Recovery requires Snowflake Support
- Additional storage costs
 ↓
Day 15+: Data permanently deleted
```

### Storage Cost Calculation

```sql
-- Time Travel storage (user-accessible)
-- = (Current table size × Time Travel days) × Daily change rate

-- Fail-Safe storage (Snowflake-managed)
-- = (Current table size × 7 days) × Daily change rate

-- Example:
-- Table: 100GB, 10% daily change, 30-day Time Travel
-- Time Travel storage: 100GB × 30 × 0.10 = 300GB
-- Fail-Safe storage: 100GB × 7 × 0.10 = 70GB
-- Total: 370GB additional storage beyond current table
```

---

## Zero-Copy Cloning

Cloning creates instant snapshots of databases, schemas, or tables without data duplication. Uses metadata pointers; data copies occur only on write (copy-on-write). Ideal for dev/test environments, backups, and data experimentation.

### Syntax: CREATE CLONE

#### Complete Syntax Template

```sql
-- Clone database
CREATE DATABASE <n> CLONE <source_db>
  [ AT ( TIMESTAMP => <ts> | OFFSET => <time_diff> | STATEMENT => <query_id> ) ]
  [ BEFORE ( STATEMENT => <query_id> ) ];

-- Clone schema
CREATE SCHEMA <n> CLONE <source_schema>
  [ AT | BEFORE ( ... ) ];

-- Clone table
CREATE TABLE <n> CLONE <source_table>
  [ AT | BEFORE ( ... ) ]
  [ COPY GRANTS ];
```

#### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `AT TIMESTAMP` | Timestamp | Current | Clone from specific point in time | `AT(TIMESTAMP => '2025-01-30 10:00:00')` |
| `AT OFFSET` | Integer | Current | Clone from seconds ago | `AT(OFFSET => -3600)` |
| `AT STATEMENT` | Query ID | Current | Clone from query execution point | `AT(STATEMENT => '01a9...')` |
| `BEFORE STATEMENT` | Query ID | Current | Clone before statement execution | `BEFORE(STATEMENT => '01a9...')` |
| `COPY GRANTS` | Flag | FALSE | Clone privileges with table | `COPY GRANTS` |

#### Basic Example

```sql
-- Clone table instantly
CREATE TABLE orders_backup CLONE sales.orders;

-- Clone schema with all objects
CREATE SCHEMA sales_backup CLONE sales;

-- Clone entire database
CREATE DATABASE prod_backup CLONE prod_db;

-- Clone from 24 hours ago
CREATE TABLE orders_yesterday CLONE sales.orders
  AT(OFFSET => -86400);
```

#### Advanced Example (SDLC: Environment Cloning)

```sql
-- Clone production to staging (zero-copy, instant)
CREATE DATABASE staging_db CLONE prod_db
  COMMENT = 'Production clone for sprint testing';

-- Clone production to QA with historical state
CREATE DATABASE qa_db CLONE prod_db
  AT(TIMESTAMP => DATEADD(day, -1, CURRENT_TIMESTAMP()))
  COMMENT = 'QA environment - yesterday production state';

-- Clone specific tables for feature development
CREATE SCHEMA dev_db.feature_branch_123 CLONE prod_db.sales;

-- Swap tables (blue-green deployment)
-- Step 1: Clone production table
CREATE TABLE sales.orders_v2 CLONE sales.orders;

-- Step 2: Modify new version
ALTER TABLE sales.orders_v2 ADD COLUMN new_feature VARCHAR;

-- Step 3: Swap tables
ALTER TABLE sales.orders RENAME TO sales.orders_old;
ALTER TABLE sales.orders_v2 RENAME TO sales.orders;

-- Step 4: Drop old version (after validation)
DROP TABLE sales.orders_old;
```

### Cloning Automation (Python)

```python
# clone_environments.py
import snowflake.connector
from datetime import datetime

class EnvironmentCloner:
    def __init__(self, account, user, password, role='SYSADMIN'):
        self.conn = snowflake.connector.connect(
            account=account,
            user=user,
            password=password,
            role=role
        )
        self.cursor = self.conn.cursor()
    
    def clone_prod_to_dev(self, sprint_number):
        """Clone production database to development for sprint testing"""
        dev_db = f"dev_sprint_{sprint_number}_db"
        
        # Drop existing dev database
        self.cursor.execute(f"DROP DATABASE IF EXISTS {dev_db}")
        
        # Clone production (instant, zero-copy)
        self.cursor.execute(f"""
            CREATE DATABASE {dev_db}
              CLONE prod_db
              COMMENT = 'Dev environment - Sprint {sprint_number} - {datetime.now()}'
        """)
        
        # Grant access to dev team
        self.cursor.execute(f"""
            GRANT ALL ON DATABASE {dev_db} TO ROLE dev_team
        """)
        
        print(f"✅ Cloned prod_db → {dev_db}")
        return dev_db
    
    def clone_table_for_experiment(self, table_name, experiment_name):
        """Clone table for data science experiment"""
        clone_name = f"{table_name}_{experiment_name}"
        
        self.cursor.execute(f"""
            CREATE TABLE {clone_name} CLONE {table_name}
              COMMENT = 'Experiment: {experiment_name}'
        """)
        
        print(f"✅ Cloned {table_name} → {clone_name}")
        return clone_name
    
    def restore_from_time_travel(self, table_name, hours_ago):
        """Restore table to state N hours ago"""
        backup_name = f"{table_name}_backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
        
        # Backup current state
        self.cursor.execute(f"""
            CREATE TABLE {backup_name} CLONE {table_name}
        """)
        
        # Restore to historical state
        self.cursor.execute(f"""
            CREATE OR REPLACE TABLE {table_name}_restored CLONE {table_name}
              AT(OFFSET => -{hours_ago * 3600})
        """)
        
        print(f"✅ Restored {table_name} to {hours_ago} hours ago")
        print(f"📦 Backup: {backup_name}")
        
        return backup_name

# Usage
if __name__ == '__main__':
    cloner = EnvironmentCloner(
        account='myorg.us-east-1',
        user='admin_user',
        password='***'
    )
    
    # Clone prod to dev for sprint 45
    cloner.clone_prod_to_dev(sprint_number=45)
    
    # Clone table for ML experiment
    cloner.clone_table_for_experiment('sales.fact_orders', 'price_elasticity_v2')
    
    # Restore table after accidental deletion
    cloner.restore_from_time_travel('sales.customers', hours_ago=2)
```

---

## UNDROP (Object Recovery)

UNDROP restores dropped tables, schemas, or databases within the Time Travel retention period. Uses Time Travel metadata to reconstruct objects.

### Syntax: UNDROP

```sql
-- Undrop table
UNDROP TABLE <table_name>;

-- Undrop schema
UNDROP SCHEMA <schema_name>;

-- Undrop database
UNDROP DATABASE <database_name>;
```

### Example

```sql
-- Accidentally drop table
DROP TABLE sales.orders;

-- Realize mistake, restore immediately
UNDROP TABLE sales.orders;

-- Restore dropped schema
UNDROP SCHEMA sales;

-- Restore dropped database
UNDROP DATABASE prod_db;

-- View dropped objects
SHOW TABLES HISTORY LIKE '%orders%';
SHOW SCHEMAS HISTORY;
SHOW DATABASES HISTORY;
```

---

## Materialized Views

Materialized views pre-compute and store query results for faster performance. Automatically refresh on base table changes. Ideal for expensive aggregations used in BI dashboards.

### Syntax: CREATE MATERIALIZED VIEW

#### Complete Syntax Template

```sql
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
| `SECURE` | Flag | FALSE | Encrypt view definition | `SECURE` |
| `CLUSTER BY` | Expression(s) | NULL | Clustering for MV | `CLUSTER BY (order_date)` |
| `COPY GRANTS` | Flag | FALSE | Copy privileges from replaced view | `COPY GRANTS` |
| `TAG` | Key-Value | NULL | Metadata tags | `WITH TAG (dashboard='executive')` |
| `COMMENT` | String | NULL | View description | `'Pre-aggregated sales metrics'` |

#### Basic Example

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW sales.mv_daily_sales AS
SELECT 
  DATE_TRUNC('day', order_date) AS day,
  SUM(total_amount) AS daily_revenue,
  COUNT(*) AS order_count,
  AVG(total_amount) AS avg_order_value
FROM sales.fact_orders
GROUP BY day;

-- Query materialized view (fast, pre-computed)
SELECT * FROM sales.mv_daily_sales
WHERE day >= '2025-01-01'
ORDER BY day DESC;

-- Refresh materialized view manually (optional)
ALTER MATERIALIZED VIEW sales.mv_daily_sales SUSPEND;
ALTER MATERIALIZED VIEW sales.mv_daily_sales RESUME;
```

#### Advanced Example (BI Dashboard Optimization)

```sql
-- Pre-aggregate executive dashboard metrics
CREATE OR REPLACE SECURE MATERIALIZED VIEW exec.mv_kpi_dashboard
  CLUSTER BY (metric_date, region)
  WITH TAG (
    dashboard = 'executive',
    refresh_freq = 'automatic',
    cost_center = 'analytics'
  )
  COMMENT = 'Executive KPI dashboard - auto-refreshed'
AS
SELECT 
  DATE_TRUNC('day', o.order_date) AS metric_date,
  c.region,
  c.customer_segment,
  COUNT(DISTINCT o.customer_id) AS active_customers,
  COUNT(o.order_id) AS total_orders,
  SUM(o.total_amount) AS total_revenue,
  AVG(o.total_amount) AS avg_order_value,
  SUM(CASE WHEN o.status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
  SUM(CASE WHEN o.status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders,
  CURRENT_TIMESTAMP() AS last_refreshed
FROM sales.fact_orders o
INNER JOIN sales.dim_customers c ON o.customer_id = c.customer_id
GROUP BY metric_date, region, customer_segment;

-- Create dependent MV (monthly rollup from daily)
CREATE MATERIALIZED VIEW exec.mv_monthly_kpis AS
SELECT 
  DATE_TRUNC('month', metric_date) AS month,
  region,
  SUM(total_revenue) AS monthly_revenue,
  AVG(avg_order_value) AS avg_order_value,
  SUM(total_orders) AS monthly_orders
FROM exec.mv_kpi_dashboard
GROUP BY month, region;

-- Show MV refresh history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.MATERIALIZED_VIEW_REFRESH_HISTORY(
  VIEW_NAME => 'EXEC.MV_KPI_DASHBOARD'
))
ORDER BY refresh_time DESC;
```

### Materialized View Limitations

- **No Joins**: Base query cannot contain joins (use nested MVs)
- **No CTEs**: Common Table Expressions not supported
- **No UDFs**: User-defined functions not allowed
- **Clustering**: Clustering key must be subset of GROUP BY columns
- **Storage**: MVs consume storage (similar to table)

### MV vs View Performance

```sql
-- Standard view (query-time computation)
CREATE VIEW v_daily_sales AS
SELECT 
  DATE_TRUNC('day', order_date) AS day,
  SUM(total_amount) AS revenue
FROM fact_orders  -- 10B rows
GROUP BY day;

SELECT * FROM v_daily_sales WHERE day = '2025-01-31';
-- Execution time: 15 seconds (scans 10B rows)

-- Materialized view (pre-computed)
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
  DATE_TRUNC('day', order_date) AS day,
  SUM(total_amount) AS revenue
FROM fact_orders
GROUP BY day;

SELECT * FROM mv_daily_sales WHERE day = '2025-01-31';
-- Execution time: 0.5 seconds (scans 365 rows)
```

---

## Cross-Account Data Sharing

Share live data across organizations without data movement. Secure, governed, and monetizable (Snowflake Marketplace).

### Secure Share Architecture

```
PROVIDER ACCOUNT                    CONSUMER ACCOUNT
┌───────────────────┐                ┌─────────────────┐
│ prod_db.sales     │                │ shared_sales_db │
│ ├── fact_orders   │  ──Share──>    │ ├── v_orders    │
│ ├── dim_customers │                │ ├── v_customers │
│ └── v_analytics   │                │ └── v_analytics │
└───────────────────┘                └─────────────────┘
          ↓                                    ↓
      Data stored                        Metadata only
      in provider                        (no data copy)
```

### Advanced Sharing Example

```sql
-- PROVIDER: Create share with secure views
CREATE SHARE partner_sales_share
  COMMENT = 'Sales data share for partner accounts';

-- Grant database and schema
GRANT USAGE ON DATABASE prod_db TO SHARE partner_sales_share;
GRANT USAGE ON SCHEMA prod_db.sales TO SHARE partner_sales_share;

-- Create secure view with filters (share only US region)
CREATE OR REPLACE SECURE VIEW prod_db.sales.v_orders_shared AS
SELECT 
  order_id,
  order_date,
  total_amount,
  status
FROM prod_db.sales.fact_orders
WHERE region = 'US'
  AND order_date >= DATEADD(year, -1, CURRENT_DATE());

GRANT SELECT ON VIEW prod_db.sales.v_orders_shared TO SHARE partner_sales_share;

-- Add consumer accounts
ALTER SHARE partner_sales_share 
  ADD ACCOUNTS = partner_account_id_1, partner_account_id_2;

-- CONSUMER: Access shared data
CREATE DATABASE partner_data
  FROM SHARE provider_account.partner_sales_share;

-- Query shared data
SELECT 
  DATE_TRUNC('month', order_date) AS month,
  SUM(total_amount) AS revenue
FROM partner_data.sales.v_orders_shared
GROUP BY month;
```

---

## SDLC Use Case: Zero-Copy Cloning for Test Environments

### Scenario: Instant Environment Provisioning in CI/CD

```yaml
# .github/workflows/clone-test-env.yml
name: Clone Test Environment

on:
  workflow_dispatch:
    inputs:
      environment_name:
        description: 'Environment name (e.g., qa-sprint-45)'
        required: true
      clone_from:
        description: 'Source database to clone'
        required: true
        default: 'prod_db'

jobs:
  clone-environment:
    runs-on: ubuntu-latest
    steps:
      - name: Clone Database
        run: |
          python - <<EOF
          import snowflake.connector
          import os
          
          conn = snowflake.connector.connect(
              account=os.getenv('SNOWFLAKE_ACCOUNT'),
              user=os.getenv('SNOWFLAKE_USER'),
              password=os.getenv('SNOWFLAKE_PASSWORD'),
              role='SYSADMIN'
          )
          
          env_name = "${{ github.event.inputs.environment_name }}"
          source_db = "${{ github.event.inputs.clone_from }}"
          
          cursor = conn.cursor()
          
          # Clone database (zero-copy, instant)
          cursor.execute(f"""
              CREATE OR REPLACE DATABASE {env_name}_db
                CLONE {source_db}
                COMMENT = 'Test environment - {env_name}'
          """)
          
          # Grant access to QA team
          cursor.execute(f"""
              GRANT ALL ON DATABASE {env_name}_db TO ROLE qa_team
          """)
          
          print(f"✅ Environment {env_name}_db created successfully")
          
          cursor.close()
          conn.close()
          EOF
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
```

---

## Key Concepts

- **Time Travel**: Query historical data 0-90 days using AT/BEFORE clauses (configurable retention)
- **Fail-Safe**: 7-day recovery period after Time Travel (Snowflake Support only)
- **Zero-Copy Clone**: Instant table/schema/database snapshots using metadata pointers
- **UNDROP**: Restore dropped objects within Time Travel retention window
- **Materialized View**: Pre-computed query results with automatic refresh on base table changes
- **Data Sharing**: Live cross-account data access without data movement or ETL

---
