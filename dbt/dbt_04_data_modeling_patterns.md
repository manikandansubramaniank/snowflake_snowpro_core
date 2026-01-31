# 4. Data Modeling Patterns

## Overview
dbt enables modular, testable data transformations through layered modeling patterns. The staging → intermediate → marts pattern separates concerns: raw data normalization, business logic, and analytics-ready outputs. Dimensional modeling and SCD Type 2 snapshots provide proven frameworks for warehousing.

---

## 4.1 Staging → Intermediate → Marts Architecture

### Pattern Description
**Diagram: 3-Layer dbt Architecture**
```text
┌─────────────────────────────────────────────────────────────┐
│ Raw Sources (S3, APIs, databases)                           │
└───────────────────┬─────────────────────────────────────────┘
                    │ source('raw', 'table')
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ STAGING LAYER (stg_*)                                       │
│ - Light transformations (rename, type cast, dedupe)         │
│ - 1:1 with sources                                          │
│ - Materialized as views (cost-efficient)                    │
│ - Example: stg_orders, stg_customers                        │
└───────────────────┬─────────────────────────────────────────┘
                    │ ref('stg_orders')
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ INTERMEDIATE LAYER (int_*)                                  │
│ - Complex business logic                                    │
│ - Joins, aggregations, window functions                     │
│ - Reusable components                                       │
│ - Materialized as ephemeral or tables                       │
│ - Example: int_order_line_items_enriched                    │
└───────────────────┬─────────────────────────────────────────┘
                    │ ref('int_*')
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ MARTS LAYER (fct_*, dim_*)                                  │
│ - Analytics-ready models                                    │
│ - Fact tables (metrics) + Dimension tables (attributes)     │
│ - Materialized as tables or incrementals                    │
│ - Example: fct_orders, dim_customers                        │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    ↓ 
            BI Tools (Tableau, Looker)
```

**Staging Models:** Normalize source data with minimal transformations (rename columns, cast types, dedupe). Materialized as views to minimize compute costs since they're re-queried by downstream models.

**Intermediate Models:** Implement complex business logic (joins, window functions, CTEs). Use ephemeral materialization for simple transforms or tables for expensive computations that downstream models reuse.

**Marts Models:** Deliver analytics-ready fact and dimension tables following dimensional modeling (star/snowflake schemas). Materialized as tables for query performance or incrementals for large datasets.

---

### Syntax: dbt_project.yml - Models Configuration

```yaml
# ─────────────────────────────────────────────────────────────
# SYNTAX TEMPLATE
# ─────────────────────────────────────────────────────────────
models:
  <project_name>:
    <folder_name>:
      +materialized: view | table | incremental | ephemeral
      +schema: <schema_name>
      +database: <database_name>
      +alias: <table_alias>
      +tags: [<tag1>, <tag2>]
      +meta: {<key>: <value>}
      +docs:
        show: true | false
      +persist_docs:
        relation: true | false
        columns: true | false
      +full_refresh: true | false
      +on_schema_change: ignore | fail | append_new_columns | sync_all_columns
      +grants:
        select: [<role1>, <role2>]
        insert: [<role>]
      +enabled: true | false
      
      # Incremental-specific
      +unique_key: <column_name> | [<col1>, <col2>]
      +incremental_strategy: merge | append | delete+insert | insert_overwrite
      +merge_exclude_columns: [<col1>, <col2>]
      +merge_update_columns: [<col1>, <col2>]
      
      # Adapter-specific (Snowflake example)
      +cluster_by: [<column1>, <column2>]
      +transient: true | false
      +copy_grants: true | false
      +secure: true | false
      +snowflake_warehouse: <warehouse_name>
      +query_tag: <tag_value>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `+materialized` | Enum | `view` | Persistence strategy: view (query-time), table (stored), incremental (upsert), ephemeral (CTE) | `incremental` |
| `+schema` | String | Project default | Target schema name; supports `{{ target.schema }}_suffix` | `analytics` |
| `+database` | String | Target default | Target database/catalog name | `prod_warehouse` |
| `+alias` | String | Model filename | Override table/view name in warehouse | `customer_master` |
| `+tags` | Array | `[]` | Metadata for selection (e.g., `dbt run --select tag:pii`) | `['pii', 'daily']` |
| `+meta` | Object | `{}` | Custom metadata for docs/integrations | `{owner: 'analytics'}` |
| `+persist_docs` | Object | `{relation: false, columns: false}` | Write descriptions to warehouse comments | `{relation: true}` |
| `+full_refresh` | Boolean | `false` | Force full rebuild (overrides --full-refresh flag) | `true` |
| `+on_schema_change` | Enum | `ignore` | Handle column changes in incremental: ignore, fail, append_new_columns, sync_all_columns | `append_new_columns` |
| `+grants` | Object | `{}` | Warehouse-level permissions by role | `{select: ['analyst']}` |
| `+enabled` | Boolean | `true` | Toggle model execution | `false` |
| `+unique_key` | String/Array | `NULL` | Primary key for incremental merge/delete+insert | `order_id` |
| `+incremental_strategy` | Enum | `merge` | Upsert logic: merge (update+insert), append (insert only), delete+insert (delete matching keys then insert) | `delete+insert` |
| `+merge_exclude_columns` | Array | `[]` | Columns excluded from UPDATE in merge strategy | `['created_at']` |
| `+merge_update_columns` | Array | All cols | Columns to UPDATE in merge (if not specified, all non-key columns) | `['status', 'updated_at']` |
| `+cluster_by` | Array | `[]` | Snowflake/BigQuery clustering keys for query optimization | `['event_date', 'user_id']` |
| `+transient` | Boolean | `false` | Snowflake transient table (no Fail-safe) | `true` |
| `+copy_grants` | Boolean | `false` | Snowflake: copy grants from old table to new on replacements | `true` |
| `+secure` | Boolean | `false` | Snowflake secure view (encrypts definition) | `true` |
| `+snowflake_warehouse` | String | Profile default | Override warehouse for specific model | `TRANSFORMING_L` |
| `+query_tag` | String | dbt-generated | Tag for query history/monitoring | `dbt_marts` |

---

**BASIC EXAMPLE**

```yaml
# dbt_project.yml
models:
  jaffle_shop:
    staging:
      +materialized: view
      +schema: staging
    
    marts:
      +materialized: table
      +schema: analytics
```

---

**ADVANCED EXAMPLE**

```yaml
# dbt_project.yml
models:
  ecommerce_analytics:
    staging:
      +materialized: view
      +schema: staging
      +tags: ['hourly']
      +persist_docs:
        relation: true
        columns: true
    
    intermediate:
      +materialized: ephemeral
      +schema: intermediate
      +tags: ['internal']
    
    marts:
      core:
        +materialized: incremental
        +schema: analytics_core
        +unique_key: id
        +incremental_strategy: merge
        +on_schema_change: append_new_columns
        +cluster_by: ['event_date', 'user_id']
        +tags: ['prod', 'daily']
        +grants:
          select: ['ANALYST_ROLE', 'BI_TOOL_ROLE']
        +persist_docs:
          relation: true
      
      finance:
        +materialized: table
        +schema: analytics_finance
        +tags: ['pii', 'finance']
        +transient: true
        +copy_grants: true
        +snowflake_warehouse: TRANSFORMING_XL
```

---

## 4.2 Dimensional Modeling

### Star Schema Pattern
Dimensional modeling organizes data into fact tables (measurements) and dimension tables (context). Star schema denormalizes dimensions for query performance; snowflake schema normalizes dimensions to reduce redundancy.

**Diagram: Star Schema Example**
```text
        ┌─────────────────┐
        │  dim_customers  │
        │─────────────────│
        │ customer_key PK │
        │ customer_id NK  │◄──┐
        │ name            │   │
        │ email           │   │
        │ segment         │   │
        └─────────────────┘   │
                              │
        ┌─────────────────┐   │      ┌─────────────────┐
        │   dim_products  │   │      │    dim_dates    │
        │─────────────────│   │      │─────────────────│
        │ product_key PK  │◄──┼──┐   │ date_key PK     │◄─┐
        │ product_id NK   │   │  │   │ date            │  │
        │ name            │   │  │   │ month           │  │
        │ category        │   │  │   │ quarter         │  │
        │ price           │   │  │   │ year            │  │
        └─────────────────┘   │  │   └─────────────────┘  │
                              │  │                        │
                      ┌───────┴──┴────────────────────────┴──────┐
                      │         fct_orders                       │
                      │──────────────────────────────────────────│
                      │ order_key PK                             │
                      │ customer_key FK → dim_customers          │
                      │ product_key FK → dim_products            │
                      │ order_date_key FK → dim_dates            │
                      │ quantity (measure)                       │
                      │ revenue (measure)                        │
                      │ discount (measure)                       │
                      └──────────────────────────────────────────┘
```

**Fact Tables (`fct_*`):** Contain numeric measures (revenue, quantity) and foreign keys to dimensions. Use incremental materialization for append-only event data.

**Dimension Tables (`dim_*`):** Contain descriptive attributes (customer name, product category). Materialized as tables for lookup performance. Use surrogate keys (customer_key) to handle slowly changing dimensions.

---

### Syntax: Fact Table (Incremental)

```sql
-- models/marts/core/fct_orders.sql

{{
  config(
    materialized='incremental',
    unique_key='order_id',
    cluster_by=['order_date']
  )
}}

WITH orders AS (
  SELECT * FROM {{ ref('stg_orders') }}
  {% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
  {% endif %}
),

customers AS (
  SELECT * FROM {{ ref('dim_customers') }}
),

products AS (
  SELECT * FROM {{ ref('dim_products') }}
)

SELECT
  o.order_id,
  c.customer_key,
  p.product_key,
  o.order_date::DATE AS order_date_key,
  o.quantity,
  o.unit_price * o.quantity AS revenue,
  o.discount_amount,
  o.updated_at
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
LEFT JOIN products p ON o.product_id = p.product_id
```

---

### Syntax: Dimension Table (SCD Type 1)

```sql
-- models/marts/core/dim_customers.sql

{{
  config(
    materialized='table',
    unique_key='customer_key'
  )
}}

SELECT
  {{ dbt_utils.generate_surrogate_key(['customer_id']) }} AS customer_key,
  customer_id,
  first_name || ' ' || last_name AS name,
  email,
  CASE
    WHEN lifetime_value > 1000 THEN 'High Value'
    WHEN lifetime_value > 500 THEN 'Medium Value'
    ELSE 'Low Value'
  END AS segment,
  created_at,
  updated_at
FROM {{ ref('stg_customers') }}
```

---

## 4.3 Incremental Models

### Incremental Strategies
Incremental models process only new/changed data since the last run. Three strategies: **merge** (upsert via MERGE), **append** (INSERT only), **delete+insert** (DELETE matching keys then INSERT).

**When to Use:**
- **Merge:** Updates existing rows (e.g., order status changes). Requires unique_key.
- **Append:** Insert-only event logs (clickstream, IoT). No unique_key needed.
- **Delete+insert:** Full row replacement when partial updates are complex or merge is slow.

---

### Syntax: Incremental Model Configuration

```sql
-- ─────────────────────────────────────────────────────────────
-- SYNTAX TEMPLATE
-- ─────────────────────────────────────────────────────────────
{{
  config(
    materialized='incremental',
    unique_key='<column_name>' | ['<col1>', '<col2>'],
    incremental_strategy='merge' | 'append' | 'delete+insert' | 'insert_overwrite',
    on_schema_change='ignore' | 'fail' | 'append_new_columns' | 'sync_all_columns',
    merge_exclude_columns=['<col1>', '<col2>'],
    merge_update_columns=['<col1>', '<col2>'],
    cluster_by=['<column1>', '<column2>'],
    partition_by={
      'field': '<column_name>',
      'data_type': 'date' | 'timestamp' | 'int64',
      'granularity': 'day' | 'hour' | 'month' | 'year'
    },
    full_refresh=false
  )
}}

SELECT ...
FROM {{ ref('upstream_model') }}
{% if is_incremental() %}
  WHERE <filter_condition>
{% endif %}
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `unique_key` | String/Array | `NULL` | Column(s) for merge/delete+insert to identify matching rows | `'order_id'` or `['customer_id', 'order_date']` |
| `incremental_strategy` | Enum | `merge` | Upsert logic: merge (UPDATE/INSERT via MERGE), append (INSERT only), delete+insert (DELETE then INSERT), insert_overwrite (partition overwrite) | `merge` |
| `on_schema_change` | Enum | `ignore` | Handle new columns: ignore (skip), fail (error), append_new_columns (add to table), sync_all_columns (add/remove columns) | `append_new_columns` |
| `merge_exclude_columns` | Array | `[]` | Columns NOT updated in merge (e.g., created_at timestamp) | `['created_at', 'source_system']` |
| `merge_update_columns` | Array | All non-key | Explicit columns to UPDATE in merge (overrides default "all columns") | `['status', 'updated_at', 'quantity']` |
| `cluster_by` | Array | `[]` | Snowflake/BigQuery: columns for physical clustering | `['event_date', 'user_id']` |
| `partition_by` | Object | `{}` | BigQuery: partition spec for cost optimization | `{field: 'event_date', data_type: 'date', granularity: 'day'}` |
| `full_refresh` | Boolean | `false` | Force full rebuild (overrides --full-refresh CLI flag) | `true` |

**Jinja Macro: `is_incremental()`**
Returns `true` if model exists and running in incremental mode (not full-refresh). Use to filter only new data.

---

**BASIC EXAMPLE: Append Strategy**

```sql
-- models/marts/events/fct_pageviews.sql
{{
  config(
    materialized='incremental',
    incremental_strategy='append',
    cluster_by=['event_date']
  )
}}

SELECT
  event_id,
  user_id,
  page_url,
  event_timestamp,
  event_timestamp::DATE AS event_date
FROM {{ ref('stg_pageviews') }}
{% if is_incremental() %}
  -- Only process events after the max timestamp in the table
  WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

---

**ADVANCED EXAMPLE: Merge Strategy with Schema Evolution**

```sql
-- models/marts/core/fct_orders_incremental.sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns',
    merge_exclude_columns=['created_at'],
    merge_update_columns=['status', 'updated_at', 'total_amount'],
    cluster_by=['order_date', 'customer_id'],
    tags=['prod', 'hourly']
  )
}}

WITH new_orders AS (
  SELECT
    order_id,
    customer_id,
    order_date,
    status,
    total_amount,
    created_at,
    updated_at
  FROM {{ ref('stg_orders') }}
  {% if is_incremental() %}
    -- Incremental filter: process orders updated since last run
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
  {% endif %}
)

SELECT
  order_id,
  customer_id,
  order_date,
  status,
  total_amount,
  created_at,
  updated_at
FROM new_orders

-- Merge behavior:
-- 1. If order_id exists → UPDATE status, updated_at, total_amount (exclude created_at)
-- 2. If order_id new → INSERT all columns
-- 3. New columns in source → append_new_columns adds them to table schema
```

---

**DELETE+INSERT EXAMPLE**

```sql
-- models/marts/finance/fct_daily_revenue.sql
{{
  config(
    materialized='incremental',
    unique_key=['order_date', 'region'],
    incremental_strategy='delete+insert'
  )
}}

SELECT
  order_date,
  region,
  SUM(revenue) AS total_revenue,
  COUNT(DISTINCT order_id) AS order_count
FROM {{ ref('fct_orders') }}
{% if is_incremental() %}
  WHERE order_date > CURRENT_DATE - 7  -- Reprocess last 7 days
{% endif %}
GROUP BY 1, 2

-- Strategy: DELETE WHERE (order_date, region) IN (new data) THEN INSERT
-- Useful when aggregations make partial updates complex
```

---

## 4.4 Snapshots (SCD Type 2)

### Concept
Snapshots capture historical states of mutable source tables using Slowly Changing Dimension Type 2. Each row gets `dbt_valid_from` and `dbt_valid_to` timestamps to track when the record was active. Enables point-in-time analysis (e.g., "What was customer segment on 2024-01-15?").

**Diagram: SCD Type 2 Snapshot**
```text
Source Table (mutable):
┌────────────┬────────┬─────────┬────────────┐
│ customer_id│  name  │ segment │ updated_at │
├────────────┼────────┼─────────┼────────────┤
│ 123        │ Alice  │ Premium │ 2024-02-01 │ ← Current state
└────────────┴────────┴─────────┴────────────┘

Snapshot Table (immutable history):
┌────────────┬────────┬─────────┬─────────────────┬───────────────┬────────────┐
│ customer_id│  name  │ segment │ dbt_valid_from  │ dbt_valid_to  │ dbt_scd_id │
├────────────┼────────┼─────────┼─────────────────┼───────────────┼────────────┤
│ 123        │ Alice  │ Standard│ 2024-01-01      │ 2024-01-15    │ hash1      │
│ 123        │ Alice  │ Premium │ 2024-01-15      │ NULL          │ hash2      │ ← Active
└────────────┴────────┴─────────┴─────────────────┴───────────────┴────────────┘
```

**Use Cases:** Customer attributes, product prices, account statuses, any dimension requiring audit trails.

---

### Syntax: Snapshot Configuration

```sql
-- ─────────────────────────────────────────────────────────────
-- SYNTAX TEMPLATE (snapshots/<snapshot_name>.sql)
-- ─────────────────────────────────────────────────────────────
{% snapshot <snapshot_name> %}

{{
  config(
    target_schema='<schema_name>',
    target_database='<database_name>',
    unique_key='<primary_key_column>',
    strategy='timestamp' | 'check',
    
    -- Timestamp strategy
    updated_at='<updated_at_column>',
    
    -- Check strategy
    check_cols='all' | [<col1>, <col2>],
    
    -- Optional
    invalidate_hard_deletes=true | false,
    tags=[<tag1>, <tag2>],
    grant_access_to=[{database_role: <role_name>}]
  )
}}

SELECT * FROM {{ source('<source_name>', '<table_name>') }}

{% endsnapshot %}
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `target_schema` | String | Required | Schema where snapshot table is created | `snapshots` or `{{ target.schema }}_snapshots` |
| `target_database` | String | Profile default | Database/catalog for snapshot table | `analytics_db` |
| `unique_key` | String | Required | Primary key to identify unique rows | `customer_id` |
| `strategy` | Enum | Required | Change detection: `timestamp` (use updated_at column) or `check` (compare column values) | `timestamp` |
| `updated_at` | String | Required (timestamp) | Column indicating when row was last modified (timestamp strategy only) | `updated_at` or `modified_date` |
| `check_cols` | String/Array | Required (check) | Columns to monitor for changes: `'all'` or `['col1', 'col2']` (check strategy only) | `['status', 'segment']` or `'all'` |
| `invalidate_hard_deletes` | Boolean | `false` | Set `dbt_valid_to` when source row is deleted (requires unique_key) | `true` |
| `tags` | Array | `[]` | Metadata for selection/docs | `['daily', 'pii']` |
| `grant_access_to` | Array | `[]` | Snowflake: grant roles access to snapshot | `[{database_role: 'ANALYST'}]` |

**Generated Columns (added by dbt):**
- `dbt_valid_from`: Timestamp when row became active
- `dbt_valid_to`: Timestamp when row was superseded (NULL if current)
- `dbt_scd_id`: Unique hash for each historical record
- `dbt_updated_at`: Timestamp of last dbt snapshot run

---

**BASIC EXAMPLE: Timestamp Strategy**

```sql
-- snapshots/customers_snapshot.sql
{% snapshot customers_snapshot %}

{{
  config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at'
  )
}}

SELECT * FROM {{ source('raw', 'customers') }}

{% endsnapshot %}
```

**Execution:**
```bash
dbt snapshot  # Run all snapshots
dbt snapshot --select customers_snapshot  # Run specific snapshot
```

---

**ADVANCED EXAMPLE: Check Strategy with Hard Deletes**

```sql
-- snapshots/product_prices_snapshot.sql
{% snapshot product_prices_snapshot %}

{{
  config(
    target_schema='snapshots',
    target_database='analytics',
    unique_key='product_id',
    strategy='check',
    check_cols=['price', 'currency', 'discount_pct'],
    invalidate_hard_deletes=true,
    tags=['finance', 'daily']
  )
}}

SELECT
  product_id,
  product_name,
  price,
  currency,
  discount_pct,
  category,
  CURRENT_TIMESTAMP AS snapshot_timestamp
FROM {{ source('ecommerce', 'products') }}

{% endsnapshot %}

-- Usage: Query current prices
-- SELECT * FROM snapshots.product_prices_snapshot
-- WHERE dbt_valid_to IS NULL

-- Usage: Historical prices on specific date
-- SELECT * FROM snapshots.product_prices_snapshot
-- WHERE '2024-01-15' BETWEEN dbt_valid_from AND COALESCE(dbt_valid_to, '9999-12-31')
```

---

## 4.5 SDLC Integration: Feature Flags

### Use Case
Feature flags allow teams to merge incomplete models into `main` without activating them in production. Use `enabled` config to toggle models based on environment or custom variables.

**Pattern:** Toggle models via dbt_project.yml + environment variables.

```yaml
# dbt_project.yml
models:
  project:
    experiments:
      +enabled: "{{ var('enable_experiments', false) }}"
      +schema: experiments
```

```bash
# Development: Enable experimental models
dbt run --vars '{enable_experiments: true}'

# Production: Disabled by default
dbt run  # Skips experiments/ folder
```

**Advanced: Per-Model Flags**

```sql
-- models/marts/experimental_revenue_model.sql
{{
  config(
    enabled=var('enable_revenue_v2', false),
    materialized='table'
  )
}}

SELECT ...
```

```yaml
# profiles.yml or dbt_project.yml
vars:
  enable_revenue_v2: false  # Default off

  # Override in dev profile
  dev:
    enable_revenue_v2: true
```

**GitHub Actions Integration:**
```yaml
# .github/workflows/dbt_ci.yml
- name: Run dbt with feature flags
  run: |
    dbt run --select state:modified+ --vars '{
      enable_experiments: true,
      enable_revenue_v2: ${{ github.event.inputs.enable_new_models }}
    }'
```

---

## Summary

**Key Patterns:**
- **Staging → Intermediate → Marts:** Separation of concerns for maintainability
- **Dimensional Modeling:** Star schema with fact/dimension tables for analytics
- **Incremental Models:** Process only new data (merge/append/delete+insert strategies)
- **Snapshots (SCD Type 2):** Capture historical states of mutable dimensions
- **Feature Flags:** Control model deployment via `enabled` config + variables

**SDLC Best Practice:** Use feature branches for new modeling patterns, enable via flags in dev, promote to prod after validation.

---

## Quick Reference Commands

```bash
# Full refresh incremental model
dbt run --full-refresh --select fct_orders

# Run only incremental models
dbt run --select config.materialized:incremental

# Execute snapshots
dbt snapshot
dbt snapshot --select customers_snapshot

# Test feature flag models
dbt run --vars '{enable_experiments: true}' --select experiments.*

# Selective run by tag
dbt run --select tag:daily
```
