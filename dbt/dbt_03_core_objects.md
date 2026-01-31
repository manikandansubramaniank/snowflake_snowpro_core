# 3. Core dbt Objects

## Models

Models are SELECT statements that materialize as database objects (tables, views, incrementals, ephemerals). Each `.sql` file in `models/` defines a transformation. Models reference other models via `{{ ref('model_name') }}` and sources via `{{ source('schema', 'table') }}`.

### Model Materialization Types

| Type | Description | Use Case | Rebuild Behavior |
|------|-------------|----------|------------------|
| **view** | Virtual table (no storage) | Lightweight transformations, always fresh data | Recreates view on each run |
| **table** | Physical table | Heavy aggregations, frequently queried | Drops and recreates table |
| **incremental** | Appends/updates new data only | Large fact tables, event logs | Processes new/changed rows only |
| **ephemeral** | CTE injected into downstream models | Reusable logic without persistence | Not materialized, compiled inline |

### Diagram: Model Materialization Strategy Decision Tree

```text
Start: Choose Materialization
    │
    ├─ Data < 100K rows? ────────────────→ VIEW (fast, always fresh)
    │
    ├─ Complex aggregations? ─────────────→ TABLE (pre-compute for speed)
    │
    ├─ Millions of rows, time-series? ────→ INCREMENTAL (process deltas)
    │
    └─ Shared logic, no direct queries? ──→ EPHEMERAL (compile-time CTE)

Decision Factors:
• Query performance needs
• Data volume growth
• Freshness requirements
• Warehouse compute costs
```

### Model Configuration: Complete Syntax

```sql
-- ============================================================================
-- MODEL CONFIG BLOCK (in model .sql file)
-- ============================================================================

{{
  config(
    materialized='table' | 'view' | 'incremental' | 'ephemeral',
    schema='<schema_override>',
    database='<database_override>',
    alias='<table_name_override>',
    tags=['<tag1>', '<tag2>'],
    enabled=true | false,
    pre_hook=['<sql>'],
    post_hook=['<sql>'],
    full_refresh=true | false,
    
    -- Incremental-specific
    unique_key='<column>' | ['<col1>', '<col2>'],
    incremental_strategy='merge' | 'append' | 'delete+insert' | 'insert_overwrite',
    on_schema_change='append_new_columns' | 'fail' | 'sync_all_columns' | 'ignore',
    
    -- Adapter-specific (Snowflake)
    cluster_by=['<column1>', '<column2>'],
    transient=true | false,
    copy_grants=true | false,
    secure=true | false,
    
    -- BigQuery-specific
    partition_by={
      'field': '<column>',
      'data_type': 'date' | 'timestamp' | 'datetime' | 'int64',
      'granularity': 'day' | 'hour' | 'month' | 'year'
    },
    cluster_by=['<column1>', '<column2>'],
    
    -- Redshift-specific
    dist='even' | 'all' | 'auto' | '<column>',
    sort=['<column1>', '<column2>'],
    sort_type='compound' | 'interleaved',
    
    -- Docs
    persist_docs={'relation': true, 'columns': true},
    
    -- Grants
    grants={'select': ['<role1>', '<role2>']},
    
    -- Meta
    meta={'owner': '<team>', 'priority': 'high'}
  )
}}

-- Model SQL
SELECT ...
```

### Model Config Parameter Table

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `materialized` | Enum | `view` | Persistence strategy | `incremental` |
| `schema` | String | `target.schema` | Schema override | `analytics` |
| `database` | String | `target.database` | Database override | `prod_db` |
| `alias` | String | `<filename>` | Table name override | `fct_orders_v2` |
| `tags` | Array | `[]` | Selection tags | `["nightly", "finance"]` |
| `enabled` | Boolean | `true` | Enable/disable model | `false` |
| `pre_hook` | Array | `[]` | SQL before model creation | `["DELETE FROM {{ this }}"]` |
| `post_hook` | Array | `[]` | SQL after model creation | `["GRANT SELECT ON {{ this }}"]` |
| `full_refresh` | Boolean | `false` | Force full rebuild (incremental) | `true` |
| `unique_key` | String/Array | `NULL` | Primary key for merge (incremental) | `order_id` or `["date", "id"]` |
| `incremental_strategy` | Enum | `merge` | Incremental logic | `delete+insert` |
| `on_schema_change` | Enum | `ignore` | Handle schema changes | `sync_all_columns` |
| `cluster_by` | Array | `[]` | Clustering columns (Snowflake/BQ) | `["order_date", "customer_id"]` |
| `transient` | Boolean | `false` | Transient table (Snowflake) | `true` |
| `copy_grants` | Boolean | `false` | Preserve grants on refresh | `true` |
| `secure` | Boolean | `false` | Secure view (Snowflake) | `true` |
| `partition_by` | Object | `NULL` | Partition config (BigQuery) | `{field: "date", data_type: "date"}` |
| `dist` | String | `even` | Distribution key (Redshift) | `customer_id` |
| `sort` | Array | `[]` | Sort keys (Redshift) | `["date", "id"]` |
| `persist_docs` | Object | `{}` | Store descriptions | `{relation: true, columns: true}` |
| `grants` | Object | `{}` | DCL permissions | `{select: ["analyst_role"]}` |
| `meta` | Object | `{}` | Custom metadata | `{owner: "analytics_team"}` |

### Basic Model Example (View)

```sql
-- models/staging/stg_customers.sql
{{
  config(
    materialized='view',
    schema='staging',
    tags=['staging', 'daily']
  )
}}

SELECT
    id AS customer_id,
    first_name || ' ' || last_name AS full_name,
    email,
    created_at,
    updated_at
FROM {{ source('salesforce', 'account') }}
WHERE deleted_at IS NULL
```

### Advanced Model Example (Incremental with Merge)

```sql
-- models/marts/finance/fct_orders.sql
{{
  config(
    materialized='incremental',
    schema='analytics',
    unique_key='order_id',
    incremental_strategy='merge',
    on_schema_change='sync_all_columns',
    cluster_by=['order_date'],
    tags=['finance', 'production'],
    post_hook=[
      "GRANT SELECT ON {{ this }} TO ROLE analyst_role"
    ],
    meta={
      'owner': 'finance_team',
      'sla_hours': 2
    }
  )
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
    {% endif %}
),

customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
),

final AS (
    SELECT
        orders.order_id,
        orders.customer_id,
        customers.full_name,
        orders.order_date,
        orders.total_amount,
        orders.status,
        orders.updated_at
    FROM orders
    LEFT JOIN customers USING (customer_id)
)

SELECT * FROM final
```

### Ephemeral Model Example

```sql
-- models/intermediate/_int_order_line_items.sql
-- Ephemeral: compiled as CTE, not materialized
{{
  config(
    materialized='ephemeral'
  )
}}

SELECT
    order_id,
    product_id,
    quantity,
    unit_price,
    quantity * unit_price AS line_total
FROM {{ source('ecommerce', 'order_line_items') }}
```

```sql
-- models/marts/fct_order_summary.sql
-- References ephemeral model (injected as CTE)
SELECT
    order_id,
    COUNT(*) AS line_item_count,
    SUM(line_total) AS order_total
FROM {{ ref('_int_order_line_items') }}
GROUP BY 1
```

## Sources

Sources define upstream raw tables with metadata for lineage, freshness tests, and documentation. Declared in `_sources.yml` files.

### Source Configuration: Complete Syntax

```yaml
# ============================================================================
# SOURCES CONFIGURATION (models/_sources.yml)
# ============================================================================

version: 2

sources:
  - name: <source_name>
    description: '<human_readable_description>'
    database: '<database_name>'
    schema: '<schema_name>'
    loader: '<etl_tool_name>'
    loaded_at_field: '<timestamp_column>'
    
    meta:
      <custom_key>: '<custom_value>'
    
    freshness:
      warn_after: {count: <integer>, period: minute | hour | day}
      error_after: {count: <integer>, period: minute | hour | day}
      filter: '<sql_filter>'
    
    quoting:
      database: true | false
      schema: true | false
      identifier: true | false
    
    tags: ['<tag1>', '<tag2>']
    
    tables:
      - name: <table_name>
        description: '<table_description>'
        identifier: '<actual_table_name>'
        loaded_at_field: '<timestamp_column>'
        
        freshness:
          warn_after: {count: <int>, period: <unit>}
          error_after: {count: <int>, period: <unit>}
          filter: '<sql_filter>'
        
        quoting:
          identifier: true | false
        
        external:
          location: '<s3_path>'
          partitions:
            - name: <partition_column>
              data_type: <data_type>
        
        columns:
          - name: <column_name>
            description: '<column_description>'
            meta:
              <key>: '<value>'
            quote: true | false
            tests:
              - not_null
              - unique
              - accepted_values:
                  values: ['<val1>', '<val2>']
              - relationships:
                  to: source('<source>', '<table>')
                  field: <column>
```

### Source Parameter Table

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `name` | String | (required) | Source identifier | `salesforce` |
| `description` | String | `""` | Human-readable description | `Salesforce CRM data` |
| `database` | String | `target.database` | Source database | `raw` |
| `schema` | String | (required) | Source schema | `salesforce` |
| `loader` | String | `NULL` | ETL tool name | `fivetran` |
| `loaded_at_field` | String | `NULL` | Timestamp column for freshness | `_fivetran_synced` |
| `freshness.warn_after` | Object | `NULL` | Warning threshold | `{count: 12, period: hour}` |
| `freshness.error_after` | Object | `NULL` | Error threshold | `{count: 24, period: hour}` |
| `freshness.filter` | String | `NULL` | SQL filter for freshness check | `region = 'US'` |
| `quoting.database` | Boolean | `false` | Quote database name | `true` |
| `quoting.schema` | Boolean | `false` | Quote schema name | `true` |
| `quoting.identifier` | Boolean | `false` | Quote table name | `true` |
| `tables[].identifier` | String | `<name>` | Actual table name if different | `sf_account` |
| `columns[].tests` | Array | `[]` | Column-level tests | `[not_null, unique]` |
| `external.location` | String | `NULL` | External table path (Snowflake) | `s3://bucket/data/` |

### Source Configuration Example

```yaml
version: 2

sources:
  - name: ecommerce_db
    description: 'Postgres production database replicated via Fivetran'
    database: raw
    schema: ecommerce
    loader: fivetran
    loaded_at_field: _fivetran_synced
    
    freshness:
      warn_after: {count: 6, period: hour}
      error_after: {count: 12, period: hour}
    
    tags: ['production', 'ecommerce']
    
    tables:
      - name: orders
        description: 'Transactional order records'
        identifier: orders_raw  # Actual table name
        
        columns:
          - name: order_id
            description: 'Primary key'
            tests:
              - unique
              - not_null
          
          - name: customer_id
            description: 'Foreign key to customers'
            tests:
              - not_null
              - relationships:
                  to: source('ecommerce_db', 'customers')
                  field: customer_id
          
          - name: status
            description: 'Order status'
            tests:
              - accepted_values:
                  values: ['pending', 'processing', 'shipped', 'delivered', 'cancelled']
          
          - name: total_amount
            description: 'Order total in USD'
            tests:
              - not_null
              - dbt_utils.expression_is_true:
                  expression: '>= 0'
      
      - name: customers
        description: 'Customer master records'
        freshness:
          warn_after: {count: 24, period: hour}  # Override default
        
        columns:
          - name: customer_id
            tests:
              - unique
              - not_null
          
          - name: email
            tests:
              - not_null
              - unique
```

## Seeds

Seeds load CSV files as static reference tables. Useful for small datasets (<1000 rows) like country codes, category mappings. Version-controlled in Git.

### Seed Configuration Syntax

```yaml
# ============================================================================
# SEEDS CONFIGURATION (dbt_project.yml or seeds/_seeds.yml)
# ============================================================================

seeds:
  <project_name>:
    +schema: '<schema_override>'
    +enabled: true | false
    +quote_columns: true | false
    +column_types:
      <column_name>: '<sql_data_type>'
    +full_refresh: true | false
    
    <seed_name>:
      +column_types:
        <column>: '<type>'
```

### Seed Parameter Table

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `+schema` | String | `target.schema` | Schema for seed table | `seed_data` |
| `+enabled` | Boolean | `true` | Enable seed | `false` |
| `+quote_columns` | Boolean | `false` | Quote column names | `true` |
| `+column_types` | Object | `{}` | Override inferred types | `{id: "VARCHAR(50)"}` |
| `+full_refresh` | Boolean | `false` | Force rebuild | `true` |

### Seed Example

```yaml
# dbt_project.yml
seeds:
  ecommerce_analytics:
    +schema: reference
    
    country_codes:
      +column_types:
        country_code: VARCHAR(2)
        country_name: VARCHAR(100)
```

```csv
# seeds/country_codes.csv
country_code,country_name,region
US,United States,North America
CA,Canada,North America
GB,United Kingdom,Europe
DE,Germany,Europe
```

```bash
# Load seed
dbt seed

# Reference in model
SELECT * FROM {{ ref('country_codes') }}
```

## Snapshots

Snapshots implement SCD Type 2 (Slowly Changing Dimension) to track historical changes. Capture point-in-time state of mutable source tables.

### Snapshot Strategies

| Strategy | Mechanism | Use Case |
|----------|-----------|----------|
| **timestamp** | Compare `updated_at` column | Tables with reliable update timestamps |
| **check** | Compare column values (hash) | Tables without timestamps, detect any column change |

### Snapshot Configuration: Complete Syntax

```sql
-- ============================================================================
-- SNAPSHOT SYNTAX (snapshots/snap_<name>.sql)
-- ============================================================================

{% snapshot <snapshot_name> %}

{{
  config(
    target_database='<database>',
    target_schema='<schema>',
    unique_key='<primary_key_column>',
    strategy='timestamp' | 'check',
    
    -- For timestamp strategy
    updated_at='<timestamp_column>',
    
    -- For check strategy
    check_cols='all' | ['<col1>', '<col2>'],
    
    -- Optional
    invalidate_hard_deletes=true | false,
    tags=['<tag1>', '<tag2>']
  )
}}

SELECT * FROM {{ source('<source>', '<table>') }}

{% endsnapshot %}
```

### Snapshot Parameter Table

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `target_database` | String | `target.database` | Snapshot database | `snapshots` |
| `target_schema` | String | (required) | Snapshot schema | `snapshots` |
| `unique_key` | String | (required) | Primary key column | `customer_id` |
| `strategy` | Enum | (required) | SCD strategy | `timestamp` |
| `updated_at` | String | (required for timestamp) | Timestamp column | `updated_at` |
| `check_cols` | String/Array | `all` | Columns to monitor (check strategy) | `["email", "status"]` |
| `invalidate_hard_deletes` | Boolean | `false` | Set end date for deleted rows | `true` |
| `tags` | Array | `[]` | Selection tags | `["daily"]` |

### Snapshot Example (Timestamp Strategy)

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}

{{
  config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
    invalidate_hard_deletes=true,
    tags=['daily', 'snapshots']
  )
}}

SELECT * FROM {{ source('ecommerce_db', 'customers') }}

{% endsnapshot %}
```

**Result**: Creates `snapshots.snap_customers` with SCD Type 2 columns:
- `dbt_valid_from`: Row start date
- `dbt_valid_to`: Row end date (NULL for current)
- `dbt_scd_id`: Unique row ID
- `dbt_updated_at`: Source timestamp

### Snapshot Example (Check Strategy)

```sql
-- snapshots/snap_product_prices.sql
{% snapshot snap_product_prices %}

{{
  config(
    target_schema='snapshots',
    unique_key='product_id',
    strategy='check',
    check_cols=['price', 'discount_rate'],
    tags=['daily']
  )
}}

SELECT
    product_id,
    product_name,
    price,
    discount_rate
FROM {{ source('ecommerce_db', 'products') }}

{% endsnapshot %}
```

## Tests

Tests validate data quality. Two types: generic (YAML-defined, reusable) and singular (SQL files, custom logic).

### Generic Tests

| Test | Description | Parameters |
|------|-------------|------------|
| `not_null` | Column has no NULLs | None |
| `unique` | Column values are unique | None |
| `accepted_values` | Column matches whitelist | `values: [...]` |
| `relationships` | Foreign key integrity | `to: ref('model'), field: column` |

### Test Configuration Syntax

```yaml
# ============================================================================
# TESTS CONFIGURATION (models/_models.yml)
# ============================================================================

version: 2

models:
  - name: <model_name>
    description: '<description>'
    
    columns:
      - name: <column_name>
        description: '<description>'
        tests:
          - not_null:
              config:
                severity: warn | error
                error_if: '>100'
                warn_if: '>10'
                store_failures: true
                where: '<sql_filter>'
          
          - unique:
              config:
                severity: error
          
          - accepted_values:
              values: ['<val1>', '<val2>']
              quote: true | false
          
          - relationships:
              to: ref('<model>') | source('<source>', '<table>')
              field: <column>
              config:
                severity: warn
```

### Test Parameter Table

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `severity` | Enum | `error` | Test failure severity | `warn` |
| `error_if` | String | `!=0` | Error condition SQL | `>100` |
| `warn_if` | String | `!=0` | Warning condition SQL | `>10` |
| `store_failures` | Boolean | `false` | Store failed rows in table | `true` |
| `where` | String | `NULL` | SQL filter for test | `status = 'active'` |
| `quote` | Boolean | `false` | Quote accepted values | `true` |

### Generic Test Example

```yaml
version: 2

models:
  - name: fct_orders
    description: 'Order fact table'
    
    columns:
      - name: order_id
        description: 'Primary key'
        tests:
          - unique:
              config:
                severity: error
          - not_null:
              config:
                severity: error
      
      - name: customer_id
        description: 'Foreign key to dim_customers'
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id
              config:
                severity: warn
                where: "status != 'deleted'"
      
      - name: order_status
        description: 'Order status'
        tests:
          - accepted_values:
              values: ['pending', 'shipped', 'delivered', 'cancelled']
              config:
                severity: error
      
      - name: total_amount
        description: 'Order total'
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: '>= 0'
              config:
                severity: error
                where: "order_status != 'cancelled'"
```

### Singular Test Example

```sql
-- tests/assert_revenue_positive.sql
-- Fails if any dates have negative revenue
SELECT
    order_date,
    SUM(total_amount) AS daily_revenue
FROM {{ ref('fct_orders') }}
GROUP BY 1
HAVING SUM(total_amount) < 0
```

### Custom Generic Test Example

```sql
-- macros/generic_tests/test_is_positive.sql
{% test is_positive(model, column_name) %}

SELECT *
FROM {{ model }}
WHERE {{ column_name }} <= 0

{% endtest %}
```

```yaml
# Usage in _models.yml
models:
  - name: fct_sales
    columns:
      - name: revenue
        tests:
          - is_positive
```

## Documentation

Documentation auto-generates from YAML `description` fields and model SQL. `dbt docs generate` creates static site with DAG visualization.

### Documentation Syntax

```yaml
# ============================================================================
# DOCUMENTATION (models/_models.yml)
# ============================================================================

version: 2

models:
  - name: <model_name>
    description: |
      Multi-line description supporting **markdown**.
      
      Lineage shows upstream {{ ref() }} dependencies.
    
    meta:
      owner: '<team_name>'
      
    columns:
      - name: <column_name>
        description: 'Column description'
        meta:
          <key>: '<value>'
```

### Doc Blocks (Reusable Descriptions)

```markdown
<!-- docs/overview.md -->
{% docs customer_table %}
Dimensional customer table with SCD Type 2 history.
Refreshed daily at 6am UTC via dbt Cloud job.
{% enddocs %}
```

```yaml
# models/_models.yml
models:
  - name: dim_customers
    description: '{{ doc("customer_table") }}'
```

```bash
# Generate and serve docs
dbt docs generate  # Creates catalog.json
dbt docs serve     # Localhost:8080
```

## SDLC Use Case: Pull Request Workflow

**Scenario**: Developer adds new model `fct_customer_ltv`, must pass CI checks before merge.

### PR Workflow

```bash
# 1. Create feature branch
git checkout -b feature/customer-ltv

# 2. Add model
cat > models/marts/finance/fct_customer_ltv.sql <<'EOF'
{{
  config(
    materialized='table',
    schema='analytics',
    tags=['finance', 'ltv']
  )
}}

WITH customer_orders AS (
    SELECT
        customer_id,
        SUM(total_amount) AS total_revenue,
        COUNT(*) AS order_count
    FROM {{ ref('fct_orders') }}
    WHERE order_date >= DATEADD(month, -12, CURRENT_DATE)
    GROUP BY 1
)

SELECT
    c.customer_id,
    c.full_name,
    c.email,
    co.total_revenue,
    co.order_count,
    co.total_revenue / NULLIF(co.order_count, 0) AS avg_order_value,
    co.total_revenue AS customer_ltv_12m
FROM {{ ref('dim_customers') }} c
LEFT JOIN customer_orders co USING (customer_id)
EOF

# 3. Add tests/docs
cat > models/marts/finance/_models.yml <<'EOF'
version: 2

models:
  - name: fct_customer_ltv
    description: 'Customer lifetime value (12-month lookback)'
    columns:
      - name: customer_id
        description: 'Primary key'
        tests:
          - unique
          - not_null
      
      - name: customer_ltv_12m
        description: 'Total revenue last 12 months'
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: '>= 0'
EOF

# 4. Run and test locally
dbt run --select +fct_customer_ltv
dbt test --select fct_customer_ltv

# 5. Commit and push
git add models/marts/finance/
git commit -m "feat(finance): Add customer LTV model"
git push origin feature/customer-ltv

# 6. Open PR → GitHub Actions CI
```

### GitHub Actions PR Validation

```yaml
# .github/workflows/dbt_pr_check.yml
name: dbt PR Checks

on:
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dbt
        run: |
          pip install dbt-snowflake==1.5.0
          pip install sqlfluff==2.1.0
      
      - name: Lint SQL
        run: sqlfluff lint models/ --dialect snowflake
      
      - name: dbt deps
        run: dbt deps
      
      - name: dbt compile
        run: dbt compile --target ci
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.DBT_ACCOUNT }}
          DBT_SNOWFLAKE_USER: ${{ secrets.DBT_USER }}
          DBT_SNOWFLAKE_PASSWORD: ${{ secrets.DBT_PASSWORD }}
      
      - name: Download prod manifest
        run: |
          aws s3 cp s3://artifacts/prod/manifest.json ./prod-state/
      
      - name: Run modified models
        run: |
          dbt run \
            --select state:modified+ \
            --defer \
            --state ./prod-state \
            --target ci
      
      - name: Test modified models
        run: |
          dbt test \
            --select state:modified+ \
            --target ci
      
      - name: Generate docs
        run: dbt docs generate --target ci
      
      - name: Upload docs artifact
        uses: actions/upload-artifact@v3
        with:
          name: dbt-docs
          path: target/
```

### PR Review Checklist

- [ ] Model follows naming convention (`stg_`, `int_`, `fct_`, `dim_`)
- [ ] Config block includes materialization, schema, tags
- [ ] Primary key has `unique` + `not_null` tests
- [ ] Foreign keys have `relationships` tests
- [ ] Description added to model and key columns
- [ ] CI passes: compile + run + test
- [ ] Peer reviewed for SQL quality, performance

## Key Takeaways

- Models use 4 materialization types: view (default), table, incremental, ephemeral.
- Sources catalog raw tables with freshness tests; `{{ source() }}` enables lineage tracking.
- Seeds load small CSV reference data (<1000 rows) version-controlled in Git.
- Snapshots implement SCD Type 2 using `timestamp` or `check` strategies.
- Tests (generic + singular) enforce data quality; generic tests reusable across models.
- Documentation auto-generates from YAML descriptions; serve with `dbt docs serve`.
- PR workflows integrate dbt into SDLC with automated lint/compile/test gates.

---
