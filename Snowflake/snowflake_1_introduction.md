# 1. Introduction to Snowflake

## What is Snowflake?

Snowflake is a cloud-native data platform delivering elastic compute, storage, and analytics as a service. It separates storage from compute, eliminating infrastructure management while enabling per-second billing. Built for multi-cloud (AWS, Azure, GCP), it handles structured and semi-structured data (JSON, Avro, Parquet) natively.

## Key Features

- **Elastic Scalability**: Scale compute up/down independently of storage without downtime
- **Zero-Copy Cloning**: Instant table/schema/database copies without duplicating data
- **Time Travel**: Query historical data with 1-90 days retention (configurable)
- **Secure Data Sharing**: Share live data across accounts/organizations without ETL
- **Multi-Cloud**: Deploy on AWS, Azure, or GCP with consistent experience
- **ACID Compliance**: Full transactional support with automatic concurrency control
- **Per-Second Billing**: Pay only for compute used, suspend when idle

## Cloud-Native vs Traditional Data Warehouses

| Aspect | Traditional (On-Prem) | Snowflake (Cloud-Native) |
|--------|----------------------|--------------------------|
| **Architecture** | Tightly coupled storage/compute | Separated layers, independent scaling |
| **Scaling** | Manual hardware provisioning (weeks) | Instant vertical/horizontal scaling (seconds) |
| **Maintenance** | Manual patches, backups, tuning | Fully managed, zero-administration |
| **Billing** | Fixed upfront CapEx | Per-second OpEx (compute), flat storage rate |
| **Concurrency** | Resource contention, complex workload mgmt | Unlimited virtual warehouses, no contention |
| **Data Sharing** | ETL pipelines, data copies | Live, secure sharing without movement |

## Snowflake Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  CLOUD SERVICES LAYER (Shared, Multi-Tenant)                │
│  - Authentication & Access Control                          │
│  - Query Parsing & Optimization                             │
│  - Metadata Management                                      │
│  - Transaction Management                                   │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  QUERY PROCESSING LAYER (Elastic Compute)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Virtual     │  │  Virtual     │  │  Virtual     │       │
│  │  Warehouse 1 │  │  Warehouse 2 │  │  Warehouse N │       │
│  │  (X-Small)   │  │  (Large)     │  │  (Multi-Clst)│       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  DATABASE STORAGE LAYER (Columnar, Compressed)              │
│  - Micro-Partitions (50-500MB compressed)                   │
│  - Automatic Clustering & Pruning                           │
│  - Cloud Object Storage (S3/Blob/GCS)                       │
└─────────────────────────────────────────────────────────────┘
```

**Three-Layer Separation:**
1. **Cloud Services**: Manages authentication, metadata, query optimization, and transaction coordination
2. **Query Processing**: Independent virtual warehouses execute queries without resource contention
3. **Database Storage**: Immutable micro-partitions stored in cloud object storage with automatic compression

---

## Syntax: CREATE DATABASE

### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] [ TRANSIENT ] DATABASE [ IF NOT EXISTS ] <name>
  [ CLONE <source_db> 
      [ { AT | BEFORE } ( TIMESTAMP => <timestamp> | OFFSET => <time_difference> | STATEMENT => <id> ) ] ]
  [ DATA_RETENTION_TIME_IN_DAYS = <num> ]
  [ MAX_DATA_EXTENSION_TIME_IN_DAYS = <num> ]
  [ DEFAULT_DDL_COLLATION = '<collation_spec>' ]
  [ TAG <tag_name> = '<tag_value>' [ , <tag_name> = '<tag_value>' ... ] ]
  [ COMMENT = '<string>' ]
```

### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `OR REPLACE` | Flag | N/A | Drop existing database if exists, create new | `OR REPLACE` |
| `TRANSIENT` | Flag | FALSE | Skip Fail-Safe period, reduce storage costs (70% savings) | `TRANSIENT` |
| `IF NOT EXISTS` | Flag | N/A | Create only if database doesn't exist (no error) | `IF NOT EXISTS` |
| `name` | String | Required | Database identifier (max 255 chars, alphanumeric + underscore) | `analytics_db` |
| `CLONE` | String | NULL | Clone from existing database (zero-copy) | `CLONE prod_db` |
| `AT/BEFORE` | Timestamp | NULL | Clone from specific point in Time Travel window | `AT(TIMESTAMP => '2025-01-15 10:00:00')` |
| `DATA_RETENTION_TIME_IN_DAYS` | Integer (0-90) | 1 | Time Travel retention period (Enterprise: 0-90, Standard: 0-1) | `7` |
| `MAX_DATA_EXTENSION_TIME_IN_DAYS` | Integer (0-90) | 14 | Maximum days to extend retention beyond standard | `30` |
| `DEFAULT_DDL_COLLATION` | String | NULL | Default collation for new string columns | `'en-ci'` (case-insensitive) |
| `TAG` | Key-Value | NULL | Metadata tags for governance/cost tracking | `environment = 'dev'` |
| `COMMENT` | String | NULL | Database description (max 2000 chars) | `'Production finance database'` |

### Basic Example

```sql
-- Create simple database with defaults
CREATE DATABASE analytics_db;

-- Verify creation
SHOW DATABASES LIKE 'analytics_db';
```

### Advanced Example (SDLC: CI/CD Database Provisioning)

```sql
-- Production database with extended retention
CREATE OR REPLACE DATABASE prod_analytics
  DATA_RETENTION_TIME_IN_DAYS = 30
  MAX_DATA_EXTENSION_TIME_IN_DAYS = 60
  DEFAULT_DDL_COLLATION = 'en-ci'
  TAG (
    environment = 'production',
    cost_center = 'analytics',
    compliance = 'SOX'
  )
  COMMENT = 'Production analytics database - Q1 2025';

-- Development transient database (lower cost)
CREATE TRANSIENT DATABASE dev_analytics
  DATA_RETENTION_TIME_IN_DAYS = 3
  TAG (
    environment = 'development',
    auto_cleanup = 'true'
  )
  COMMENT = 'Development analytics - Sprint 42';

-- Clone production for testing (zero-copy)
CREATE DATABASE qa_analytics
  CLONE prod_analytics
  AT (TIMESTAMP => DATEADD(day, -1, CURRENT_TIMESTAMP()));
```

---

## Syntax: CREATE SCHEMA

### Complete Syntax Template

```sql
CREATE [ OR REPLACE ] [ TRANSIENT ] SCHEMA [ IF NOT EXISTS ] <name>
  [ CLONE <source_schema>
      [ { AT | BEFORE } ( TIMESTAMP => <timestamp> | OFFSET => <time_difference> | STATEMENT => <id> ) ] ]
  [ WITH MANAGED ACCESS ]
  [ DATA_RETENTION_TIME_IN_DAYS = <num> ]
  [ MAX_DATA_EXTENSION_TIME_IN_DAYS = <num> ]
  [ DEFAULT_DDL_COLLATION = '<collation_spec>' ]
  [ TAG <tag_name> = '<tag_value>' [ , ... ] ]
  [ COMMENT = '<string>' ]
```

### Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `OR REPLACE` | Flag | N/A | Replace existing schema | `OR REPLACE` |
| `TRANSIENT` | Flag | FALSE | Skip Fail-Safe, child tables inherit | `TRANSIENT` |
| `IF NOT EXISTS` | Flag | N/A | No error if schema exists | `IF NOT EXISTS` |
| `name` | String | Required | Schema name (qualified: `db.schema` or unqualified) | `analytics.sales` |
| `CLONE` | String | NULL | Clone from existing schema | `CLONE prod.sales` |
| `WITH MANAGED ACCESS` | Flag | FALSE | Only schema owner can grant privileges (vs object owners) | `WITH MANAGED ACCESS` |
| `DATA_RETENTION_TIME_IN_DAYS` | Integer | 1 | Inheritable Time Travel retention | `7` |
| `MAX_DATA_EXTENSION_TIME_IN_DAYS` | Integer | 14 | Max extension days | `30` |
| `DEFAULT_DDL_COLLATION` | String | NULL | Default collation for schema objects | `'en-cs'` |
| `TAG` | Key-Value | NULL | Schema-level tags | `team = 'data_eng'` |
| `COMMENT` | String | NULL | Schema description | `'Sales data mart'` |

### Basic Example

```sql
-- Create schema in current database
CREATE SCHEMA sales_data;

-- Create schema in specific database
CREATE SCHEMA analytics_db.customer_360;
```

### Advanced Example (SDLC: Multi-Environment Schema Design)

```sql
-- Production managed schema (strict access control)
CREATE SCHEMA prod_analytics.sales
  WITH MANAGED ACCESS
  DATA_RETENTION_TIME_IN_DAYS = 30
  TAG (
    environment = 'production',
    data_classification = 'confidential',
    owner_team = 'sales_analytics'
  )
  COMMENT = 'Production sales fact and dimension tables';

-- Development transient schema
CREATE TRANSIENT SCHEMA dev_analytics.sales
  DATA_RETENTION_TIME_IN_DAYS = 3
  TAG (environment = 'development')
  COMMENT = 'Dev sales schema - ephemeral data';

-- Clone production schema for QA
CREATE SCHEMA qa_analytics.sales
  CLONE prod_analytics.sales
  AT (OFFSET => -3600);  -- 1 hour ago
```

---

## SDLC Use Case: CI/CD Onboarding for Fintech

In Agile fintech environments, Snowflake integrates with CI/CD pipelines for automated database lifecycle management.

### Scenario: Feature Branch Database Provisioning

**Workflow:**
1. **PR Opened** → GitHub Actions triggers database creation
2. **Developer Testing** → Isolated environment per feature branch
3. **PR Merged** → Database dropped, changes promoted to staging
4. **Production Deploy** → dbt migrations apply schema changes

### GitHub Actions Example

```yaml
# .github/workflows/snowflake-provision.yml
name: Snowflake Feature DB Provisioning

on:
  pull_request:
    types: [opened, reopened]
    branches:
      - main

jobs:
  provision-database:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install Snowflake CLI
        run: pip install snowflake-connector-python

      - name: Create Feature Database
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
        run: |
          python - <<EOF
          import snowflake.connector
          import os
          
          # Extract branch name from PR
          branch = os.getenv('GITHUB_HEAD_REF').replace('/', '_')
          db_name = f"feature_{branch}"
          
          conn = snowflake.connector.connect(
              account=os.getenv('SNOWFLAKE_ACCOUNT'),
              user=os.getenv('SNOWFLAKE_USER'),
              password=os.getenv('SNOWFLAKE_PASSWORD'),
              role='SYSADMIN'
          )
          
          cursor = conn.cursor()
          
          # Create transient database for feature
          cursor.execute(f"""
              CREATE OR REPLACE TRANSIENT DATABASE {db_name}
                DATA_RETENTION_TIME_IN_DAYS = 3
                TAG (
                  environment = 'feature',
                  pr_number = '{os.getenv("GITHUB_PR_NUMBER")}',
                  created_by = 'github_actions'
                )
                COMMENT = 'Feature branch DB - PR #{os.getenv("GITHUB_PR_NUMBER")}'
          """)
          
          # Clone staging schema structure
          cursor.execute(f"""
              CREATE SCHEMA {db_name}.analytics
                CLONE staging_db.analytics
          """)
          
          print(f"✅ Created database: {db_name}")
          
          cursor.close()
          conn.close()
          EOF

      - name: Run dbt Tests
        run: |
          dbt test --profiles-dir ./profiles --target feature_${{ github.head_ref }}
```

### Cleanup on PR Merge

```yaml
# .github/workflows/snowflake-cleanup.yml
name: Cleanup Feature Database

on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Drop Feature Database
        run: |
          python - <<EOF
          import snowflake.connector
          import os
          
          branch = os.getenv('GITHUB_HEAD_REF').replace('/', '_')
          db_name = f"feature_{branch}"
          
          conn = snowflake.connector.connect(
              account=os.getenv('SNOWFLAKE_ACCOUNT'),
              user=os.getenv('SNOWFLAKE_USER'),
              password=os.getenv('SNOWFLAKE_PASSWORD'),
              role='SYSADMIN'
          )
          
          conn.cursor().execute(f"DROP DATABASE IF EXISTS {db_name}")
          print(f"🗑️ Dropped database: {db_name}")
          EOF
```

### Benefits
- **Isolation**: Each feature branch gets dedicated database, preventing conflicts
- **Cost Optimization**: TRANSIENT databases skip Fail-Safe (70% storage savings), auto-cleanup after merge
- **Velocity**: Developers test schema changes without impacting shared environments
- **Audit Trail**: TAG metadata tracks which PR created each database

---

## Quick Start Checklist

1. **Account Setup**: Provision Snowflake account (AWS/Azure/GCP region)
2. **User Creation**: Create admin user with `ACCOUNTADMIN` role
3. **Database Creation**: `CREATE DATABASE <name>` for production
4. **Schema Design**: Organize tables into logical schemas
5. **Warehouse Provisioning**: Create compute warehouses for workloads
6. **Access Control**: Define roles and grant privileges (RBAC)
7. **CI/CD Integration**: Connect to GitHub/GitLab for automated deployments

---

## Key Concepts

- **Virtual Warehouse**: Independent compute cluster for query execution (scales separately from storage)
- **Micro-Partition**: Immutable 50-500MB compressed columnar data chunk (automatic, no manual tuning)
- **Time Travel**: Historical data access for queries, clones, and undrop operations
- **Fail-Safe**: 7-day data recovery period after Time Travel expiration (Snowflake-managed)
- **Zero-Copy Clone**: Instant copy using metadata pointers, no data duplication until changes occur

---
