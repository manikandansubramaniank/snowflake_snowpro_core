# 7. Advanced Features

## Overview
Advanced dbt features enable code reusability (packages/macros), metadata management (exposures/metrics), and warehouse-specific optimizations. These tools bridge analytics engineering with semantic layers and production performance tuning.

---

## 7.1 Packages

### What are Packages?
Packages are reusable dbt projects installed via `packages.yml`. They provide macros, models, and tests from the community (dbt Hub) or private Git repos. Essential for avoiding code duplication across teams.

**Diagram: Package Dependency Chain**
```text
Your dbt Project
     ↓ dbt deps
dbt_utils (GitHub: dbt-labs/dbt-utils)
     ↓ imported macros
dbt_expectations (data quality tests)
     ↓ re_data (data observability)
Custom Internal Package (private GitLab repo)
```

### Syntax: packages.yml

**SYNTAX TEMPLATE**
```yaml
packages:
  # Hub packages
  - package: <org>/<package_name>
    version: <semantic_version>
    install-prerelease: true | false
  
  # Git packages
  - git: "<git_url>"
    revision: <branch | tag | commit_sha>
    subdirectory: <subdirectory_path>
    warn-unpinned: true | false
  
  # Local packages
  - local: <relative_path>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| package | String | - | Hub package identifier (org/name) | dbt-labs/dbt_utils |
| version | String/Array | latest | Semantic version or range | [">=1.0.0", "<2.0.0"] |
| install-prerelease | Boolean | false | Allow prerelease versions | true |
| git | String | - | Git repository URL (HTTPS/SSH) | https://github.com/org/repo.git |
| revision | String | main | Git branch, tag, or commit SHA | v1.2.3 or abc123 |
| subdirectory | String | - | Path to dbt project within repo | dbt_packages/my_package |
| warn-unpinned | Boolean | true | Warn if Git package not pinned to tag/commit | false |
| local | String | - | Relative path to local package | ../shared_macros |

**BASIC EXAMPLE**
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
  
  - package: calogica/dbt_expectations
    version: 0.10.0
```

**ADVANCED EXAMPLE**
```yaml
packages:
  # Pinned production packages
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
    install-prerelease: false
  
  - package: calogica/dbt_expectations
    version: 0.10.3
  
  # Internal package from private GitLab
  - git: "git@gitlab.company.com:analytics/dbt_company_macros.git"
    revision: v2.1.0  # Pinned to stable tag
    subdirectory: "dbt_project"
    warn-unpinned: false
  
  # Development package (local testing)
  - local: ../shared_transforms
```

**SDLC Use Case: Package Version Control in CI/CD**
```yaml
# .github/workflows/dbt_ci.yml
- name: Install dbt packages
  run: |
    dbt deps --lock  # Generate packages-lock.yml
    git diff --exit-code dbt_packages.lock || \
      (echo "Package lock changed - commit packages-lock.yml" && exit 1)
```

**Using Installed Packages**
```sql
-- models/staging/stg_customers.sql
{% from 'dbt_utils' import get_column_values %}

SELECT 
  customer_id,
  {{ dbt_utils.surrogate_key(['customer_id', 'order_date']) }} AS sk_customer_order,
  {{ dbt_utils.safe_cast('revenue', 'NUMERIC') }} AS revenue_numeric
FROM {{ source('raw', 'customers') }}
WHERE created_at >= {{ dbt_utils.date_spine(...) }}
```

---

## 7.2 Exposures

### What are Exposures?
Exposures document downstream dependencies (dashboards, ML models, apps) that consume dbt models. They enable lineage tracking and impact analysis when models change.

**Diagram: Exposure Lineage**
```text
dbt Models (marts.fct_orders)
     ↓ consumed by
Tableau Dashboard (sales_executive_summary)
     ↓ exposure definition
dbt Docs (lineage graph shows impact)
     ↓ dbt ls --select +exposure:sales_dashboard
Selective CI runs on affected models
```

### Syntax: Exposures (schema.yml)

**SYNTAX TEMPLATE**
```yaml
exposures:
  - name: <exposure_name>
    type: dashboard | notebook | analysis | ml | application
    maturity: high | medium | low
    url: <dashboard_url>
    description: <markdown_description>
    
    depends_on:
      - ref('<model_name>')
      - source('<source_name>', '<table_name>')
    
    owner:
      name: <owner_name>
      email: <owner_email>
      slack: <slack_channel>
      team: <team_name>
    
    tags: [<tag1>, <tag2>]
    meta:
      <custom_key>: <custom_value>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| name | String | - | Unique exposure identifier (lowercase, underscores) | sales_executive_dashboard |
| type | Enum | - | Exposure category | dashboard, notebook, ml, application |
| maturity | Enum | - | Production readiness level | high, medium, low |
| url | String | - | Link to dashboard/report/app | https://tableau.company.com/sales |
| description | String | - | Purpose and consumers (supports Markdown) | "Executive sales metrics updated daily" |
| depends_on | Array | [] | List of ref() or source() dependencies | [ref('fct_orders'), ref('dim_customers')] |
| owner.name | String | - | Owner full name | Jane Doe |
| owner.email | String | - | Owner email address | jane.doe@company.com |
| owner.slack | String | - | Slack handle or channel | #analytics-alerts |
| owner.team | String | - | Owning team/department | Sales Analytics |
| tags | Array | [] | Searchable labels | ['prod', 'executive', 'daily'] |
| meta | Object | {} | Custom metadata (key-value pairs) | {refresh_frequency: 'hourly'} |

**BASIC EXAMPLE**
```yaml
exposures:
  - name: weekly_sales_report
    type: dashboard
    maturity: high
    url: https://lookerstudio.google.com/reporting/abc123
    description: Weekly sales performance by region
    
    depends_on:
      - ref('fct_sales')
      - ref('dim_regions')
    
    owner:
      name: Sales Team
      email: sales-analytics@company.com
```

**ADVANCED EXAMPLE**
```yaml
exposures:
  - name: customer_churn_model
    type: ml
    maturity: high
    url: https://mlflow.company.com/experiments/123
    description: |
      ## Customer Churn Prediction Model
      - **Algorithm**: XGBoost Classifier
      - **Features**: 30-day engagement metrics, support tickets, billing history
      - **Refresh**: Daily at 6am UTC via Airflow
      - **Consumers**: Retention team, customer success ops
    
    depends_on:
      - ref('fct_customer_engagement_30d')
      - ref('fct_support_tickets_summary')
      - ref('dim_customers_enriched')
      - source('billing', 'invoices')
    
    owner:
      name: Data Science Team
      email: ds-platform@company.com
      slack: '#ml-ops-alerts'
      team: Data Science & ML
    
    tags: ['ml', 'production', 'high-priority', 'daily']
    
    meta:
      model_version: 'v2.3.1'
      last_retrained: '2025-01-15'
      auc_score: 0.87
      feature_count: 45
      deployment: 'sagemaker'
```

**SDLC Use Case: Automated Exposure Impact Analysis**
```bash
# CI/CD: Check if PR changes affect production exposures
dbt ls --select +exposure:customer_churn_model --output json > affected_models.json

# Notify exposure owners via Slack if their dependencies change
if [ -s affected_models.json ]; then
  python scripts/notify_exposure_owners.py --exposures customer_churn_model
fi
```

---

## 7.3 Metrics (dbt Metrics - Legacy)

### What are Metrics?
**Note**: dbt Metrics (legacy) are deprecated as of dbt v1.6. Use **dbt Semantic Layer** (MetricFlow) instead. Metrics defined business logic (SUM, COUNT, AVG) with dimensions for consistent metric definitions across BI tools.

**Diagram: Metrics Evolution**
```text
Legacy dbt Metrics (v1.0-1.5)
     ↓ deprecated
dbt Semantic Layer v2 (MetricFlow, v1.6+)
     ↓ semantic_models + metrics
Integrated BI Queries (Tableau/Looker/Mode)
     ↓ GraphQL API
Consistent metric definitions across tools
```

### Syntax: Metrics (Legacy - for reference)

**SYNTAX TEMPLATE (Legacy)**
```yaml
metrics:
  - name: <metric_name>
    label: <display_label>
    model: ref('<model_name>')
    description: <metric_description>
    
    calculation_method: count | count_distinct | sum | average | min | max | median
    expression: <sql_expression>
    
    timestamp: <timestamp_column>
    time_grains: [day, week, month, quarter, year]
    
    dimensions:
      - <dimension_column_1>
      - <dimension_column_2>
    
    filters:
      - field: <column_name>
        operator: '=' | '!=' | '>' | '<' | '>=' | '<=' | 'in' | 'not in'
        value: <filter_value>
    
    meta:
      <custom_key>: <custom_value>
```

**PARAMETERS (Legacy)**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| name | String | - | Unique metric identifier | total_revenue |
| label | String | name | Human-readable display name | "Total Revenue (USD)" |
| model | String | - | Source model reference | ref('fct_orders') |
| description | String | - | Metric purpose and calculation logic | "Sum of order amounts excluding refunds" |
| calculation_method | Enum | - | Aggregation function | sum, count, count_distinct, average |
| expression | String | - | Column or SQL expression to aggregate | order_amount, CASE WHEN ... |
| timestamp | String | - | Date/timestamp column for time-series | order_date, created_at |
| time_grains | Array | [] | Supported time aggregation levels | [day, week, month, year] |
| dimensions | Array | [] | Grouping columns (must exist in model) | [customer_region, product_category] |
| filters | Array | [] | WHERE clause conditions | [{field: status, operator: '=', value: 'completed'}] |
| meta | Object | {} | Custom metadata | {team: 'finance', sla: 'daily'} |

**BASIC EXAMPLE (Legacy)**
```yaml
metrics:
  - name: total_orders
    label: "Total Orders"
    model: ref('fct_orders')
    description: "Count of all completed orders"
    
    calculation_method: count
    expression: order_id
    
    timestamp: order_date
    time_grains: [day, week, month]
    
    dimensions:
      - customer_region
```

**ADVANCED EXAMPLE (Legacy)**
```yaml
metrics:
  - name: monthly_recurring_revenue
    label: "MRR - Monthly Recurring Revenue"
    model: ref('fct_subscriptions')
    description: |
      Sum of subscription amounts for active subscriptions.
      Excludes trials and one-time purchases.
    
    calculation_method: sum
    expression: subscription_amount
    
    timestamp: subscription_start_date
    time_grains: [day, week, month, quarter, year]
    
    dimensions:
      - subscription_tier
      - customer_segment
      - billing_country
    
    filters:
      - field: subscription_status
        operator: '='
        value: 'active'
      - field: subscription_type
        operator: 'not in'
        value: ['trial', 'one_time']
    
    meta:
      team: 'finance'
      refresh_cadence: 'hourly'
      dashboard_url: 'https://looker.company.com/mrr'
```

### Modern Alternative: dbt Semantic Layer (MetricFlow)

**Syntax: Semantic Models + Metrics (dbt v1.6+)**
```yaml
# models/marts/semantic_models.yml
semantic_models:
  - name: orders
    model: ref('fct_orders')
    
    dimensions:
      - name: order_date
        type: time
        type_params:
          time_granularity: day
      
      - name: customer_region
        type: categorical
    
    measures:
      - name: order_count
        agg: count
      
      - name: revenue
        agg: sum
        expr: order_amount

# models/marts/metrics.yml
metrics:
  - name: total_revenue
    type: simple
    label: "Total Revenue"
    type_params:
      measure: revenue
    
  - name: revenue_growth_rate
    type: derived
    label: "Revenue Growth Rate (MoM)"
    type_params:
      expr: (current_revenue - previous_revenue) / previous_revenue
      metrics:
        - total_revenue
```

**SDLC Use Case: Metrics Governance**
```python
# scripts/validate_metrics.py
# Ensure all production metrics have owner, description, tests
import yaml

with open('models/marts/metrics.yml') as f:
    metrics = yaml.safe_load(f)

for metric in metrics.get('metrics', []):
    assert 'description' in metric, f"Missing description: {metric['name']}"
    assert 'meta' in metric and 'owner' in metric['meta'], f"Missing owner: {metric['name']}"
```

---

## 7.4 Jinja Macros

### What are Macros?
Macros are reusable SQL functions written in Jinja. They enable DRY (Don't Repeat Yourself) code, dynamic SQL generation, and cross-database compatibility. Stored in `macros/` directory.

**Diagram: Macro Execution Flow**
```text
Model SQL (calls macro)
     ↓ dbt compile
Jinja Renderer (substitutes macro logic)
     ↓ SQL output
Compiled SQL (macros/target/compiled/)
     ↓ dbt run
Warehouse Execution (Snowflake/BigQuery)
```

### Syntax: Macros

**SYNTAX TEMPLATE**
```sql
{# macros/macro_name.sql #}
{% macro macro_name(param1, param2='default_value', param3=None) %}
  
  {# Macro logic with Jinja control structures #}
  {% if param3 %}
    -- SQL code when param3 is provided
  {% else %}
    -- Default SQL code
  {% endif %}
  
  {# Return SQL or value #}
  {{ return(sql_statement) }}

{% endmacro %}
```

**PARAMETERS (Macro Definition)**

| Component | Description | Example |
|-----------|-------------|---------|
| macro_name | Unique identifier (called via {{ macro_name() }}) | grant_select_on_schemas |
| param1, param2 | Positional or keyword arguments | schema_name, role_list=['analyst'] |
| default_value | Default for optional parameters | role_list=['analyst', 'developer'] |
| return() | Explicitly return value (optional) | {{ return(sql_string) }} |
| Jinja tags | {% if %}, {% for %}, {% set %} control structures | {% for role in role_list %} |

**BASIC EXAMPLE**
```sql
{# macros/cents_to_dollars.sql #}
{% macro cents_to_dollars(column_name, decimal_places=2) %}
  ROUND({{ column_name }} / 100.0, {{ decimal_places }})
{% endmacro %}

-- Usage in model:
SELECT 
  order_id,
  {{ cents_to_dollars('amount_cents') }} AS amount_dollars
FROM {{ ref('stg_orders') }}
```

**ADVANCED EXAMPLE: Cross-Database Date Truncation**
```sql
{# macros/date_trunc_adapter.sql #}
{% macro date_trunc(datepart, column) %}
  
  {# Adapter-specific SQL based on target warehouse #}
  {% if target.type == 'snowflake' %}
    DATE_TRUNC('{{ datepart }}', {{ column }})
  
  {% elif target.type == 'bigquery' %}
    DATE_TRUNC({{ column }}, {{ datepart.upper() }})
  
  {% elif target.type == 'redshift' or target.type == 'postgres' %}
    DATE_TRUNC('{{ datepart }}', {{ column }})
  
  {% else %}
    {{ exceptions.raise_compiler_error("Unsupported adapter: " ~ target.type) }}
  
  {% endif %}

{% endmacro %}

-- Usage:
SELECT 
  {{ date_trunc('month', 'order_date') }} AS order_month,
  COUNT(*) AS order_count
FROM {{ ref('fct_orders') }}
GROUP BY 1
```

**ADVANCED EXAMPLE: Generate Surrogate Keys**
```sql
{# macros/generate_schema_name.sql - Override default schema naming #}
{% macro generate_schema_name(custom_schema_name, node) -%}
  
  {%- set default_schema = target.schema -%}
  
  {%- if custom_schema_name is none -%}
    {{ default_schema }}
  
  {%- elif target.name == 'prod' -%}
    {# Production: use custom schema directly #}
    {{ custom_schema_name | trim }}
  
  {%- else -%}
    {# Dev/Staging: prefix with user schema #}
    {{ default_schema }}_{{ custom_schema_name | trim }}
  
  {%- endif -%}

{%- endmacro %}
```

**Macro Testing**
```sql
{# macros/test_cents_to_dollars.sql #}
{% macro test_cents_to_dollars() %}
  
  {% set result = cents_to_dollars('100') %}
  
  {% if result == '1.00' %}
    {{ log("Test passed: cents_to_dollars(100) = 1.00", info=True) }}
  {% else %}
    {{ exceptions.raise_compiler_error("Test failed: expected 1.00, got " ~ result) }}
  {% endif %}

{% endmacro %}

-- Run via: dbt run-operation test_cents_to_dollars
```

**SDLC Use Case: Macro Library Versioning**
```yaml
# dbt_project.yml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ['my_company_utils', 'dbt_utils']  # Override dbt_utils macros

# macros/override_surrogate_key.sql
{% macro my_company_utils__surrogate_key(field_list) %}
  {# Custom implementation with MD5 hashing #}
  MD5(CONCAT_WS('||', {{ field_list | join(", ") }}))
{% endmacro %}
```

---

## 7.5 dbt + Snowflake Optimizations

### What are Snowflake Optimizations?
Snowflake-specific configs (clustering, transient tables, query tags) reduce compute costs and improve query performance. Essential for production workloads.

**Diagram: Snowflake Compute Optimization**
```text
Raw Data (100M rows, unclustered)
     ↓ dbt incremental model
Clustered Table (cluster_by=['order_date', 'region'])
     ↓ Snowflake micro-partitions (automatic)
Query: WHERE order_date = '2025-01-01'
     ↓ Partition pruning (scans 0.1% of data)
10x faster queries, 90% cost reduction
```

### Syntax: Snowflake-Specific Configs

**SYNTAX TEMPLATE**
```yaml
# dbt_project.yml or model config block
models:
  <project_name>:
    <folder>:
      +materialized: table | view | incremental
      +snowflake_warehouse: <warehouse_name>
      +cluster_by: [<column1>, <column2>, ...]
      +automatic_clustering: true | false
      +transient: true | false
      +database: <database_name>
      +schema: <schema_name>
      +alias: <table_alias>
      +query_tag: <query_tag_string>
      +copy_grants: true | false
      +secure: true | false  # Secure views
      +tags:
        <tag_key>: <tag_value>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| snowflake_warehouse | String | profile default | Warehouse for model execution | TRANSFORMING_WH, REPORTING_WH |
| cluster_by | Array | [] | Clustering keys for table optimization | ['order_date', 'customer_id'] |
| automatic_clustering | Boolean | false | Enable Snowflake automatic re-clustering | true (production tables) |
| transient | Boolean | false | Transient table (no fail-safe, 1-day Time Travel) | true (staging, temporary models) |
| database | String | profile default | Target database | ANALYTICS_DB, PROD_DB |
| schema | String | models | Target schema | STAGING, MARTS |
| alias | String | model name | Table/view alias name | fct_orders_v2 |
| query_tag | String | - | Snowflake query tag for monitoring | dbt:{{ model.name }}:{{ target.name }} |
| copy_grants | Boolean | false | Preserve existing grants on table replace | true (production) |
| secure | Boolean | false | Create secure view (hides definition) | true (PII, sensitive logic) |
| tags | Object | {} | Snowflake object tags (key-value) | {pii: 'true', owner: 'analytics'} |

**BASIC EXAMPLE**
```sql
-- models/marts/fct_orders.sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    cluster_by=['order_date', 'customer_region'],
    transient=false
  )
}}

SELECT 
  order_id,
  order_date,
  customer_region,
  order_amount
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
  WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**ADVANCED EXAMPLE: Production Optimized Config**
```yaml
# dbt_project.yml
models:
  my_project:
    marts:
      +materialized: incremental
      +snowflake_warehouse: TRANSFORMING_XL_WH
      +cluster_by: ['event_date', 'user_id']
      +automatic_clustering: true
      +transient: false
      +copy_grants: true
      +query_tag: "dbt:{{ model.name }}:{{ invocation_id }}"
      +tags:
        environment: production
        pii: contains_pii
        cost_center: analytics_engineering
      
      fct_user_events:
        +cluster_by: ['event_timestamp::date', 'event_type', 'user_id']
        +pre-hook: |
          ALTER WAREHOUSE {{ target.warehouse }} SET WAREHOUSE_SIZE = 'X-LARGE';
        +post-hook: |
          ALTER WAREHOUSE {{ target.warehouse }} SET WAREHOUSE_SIZE = 'SMALL';
          GRANT SELECT ON {{ this }} TO ROLE ANALYST_ROLE;
```

**Incremental Strategy: Snowflake Merge**
```sql
-- models/marts/fct_daily_metrics.sql
{{
  config(
    materialized='incremental',
    unique_key='metric_date',
    incremental_strategy='merge',
    cluster_by=['metric_date'],
    on_schema_change='sync_all_columns',
    snowflake_warehouse='TRANSFORMING_WH'
  )
}}

SELECT 
  DATE(event_timestamp) AS metric_date,
  COUNT(DISTINCT user_id) AS daily_active_users,
  SUM(revenue) AS daily_revenue,
  CURRENT_TIMESTAMP() AS _updated_at
FROM {{ ref('fct_events') }}

{% if is_incremental() %}
  WHERE DATE(event_timestamp) >= (
    SELECT DATEADD(day, -3, MAX(metric_date)) FROM {{ this }}
  )
{% endif %}

GROUP BY 1
```

**Query Tag for Cost Tracking**
```sql
-- macros/generate_query_tag.sql
{% macro generate_query_tag() %}
  {% set tag_dict = {
    'model': model.name,
    'target': target.name,
    'invocation_id': invocation_id,
    'user': target.user,
    'dbt_version': dbt_version
  } %}
  
  {{ return(tojson(tag_dict)) }}
{% endmacro %}

-- Model usage:
{{ config(query_tag=generate_query_tag()) }}
```

**SDLC Use Case: Warehouse Auto-Scaling**
```sql
-- macros/warehouse_management.sql
{% macro set_warehouse_size(size='SMALL') %}
  {% if target.name == 'prod' and execute %}
    {% do run_query("ALTER WAREHOUSE " ~ target.warehouse ~ " SET WAREHOUSE_SIZE = '" ~ size ~ "'") %}
    {{ log("Set warehouse to " ~ size, info=True) }}
  {% endif %}
{% endmacro %}

-- Model with auto-scaling:
{{ set_warehouse_size('X-LARGE') }}

SELECT ...  -- Heavy transformation

{{ set_warehouse_size('SMALL') }}
```

**Snowflake Zero-Copy Cloning for Dev**
```bash
# scripts/create_dev_env.sh
#!/bin/bash
snowsql -q "
CREATE DATABASE DEV_${USER}_DB CLONE PROD_DB;
GRANT OWNERSHIP ON DATABASE DEV_${USER}_DB TO ROLE DEVELOPER;
"

# dbt profiles.yml uses DEV_${USER}_DB dynamically
```

---

## 7.6 Production Workflow Integration

**Complete SDLC Example: GitHub Actions + dbt Cloud**
```yaml
# .github/workflows/dbt_prod_deploy.yml
name: dbt Production Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install dbt packages
        run: dbt deps
      
      - name: Run dbt slim CI (modified models only)
        run: |
          dbt build --select state:modified+ --state ./prod_artifacts --defer
      
      - name: Run full refresh on weekends
        if: github.event.schedule == '0 2 * * 0'
        run: dbt run --full-refresh --select tag:full_refresh_weekly
      
      - name: Upload artifacts to S3
        run: |
          aws s3 cp target/manifest.json s3://dbt-artifacts/prod/manifest.json
          aws s3 cp target/run_results.json s3://dbt-artifacts/prod/run_results.json
      
      - name: Notify Slack on failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text":"dbt production run failed - check GitHub Actions"}'
```

---

## Summary

Advanced dbt features enable:
- **Packages**: Reusable code across projects (dbt_utils, dbt_expectations)
- **Exposures**: Downstream dependency tracking (dashboards, ML models)
- **Metrics**: Consistent business logic definitions (legacy → Semantic Layer)
- **Macros**: DRY SQL, cross-database compatibility
- **Snowflake Optimizations**: Clustering, transient tables, query tags for cost control

---