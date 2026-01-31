# 2. dbt Architecture

## Project Structure

dbt projects follow a standardized directory layout promoting modularity and maintainability. Each folder serves a specific purpose in the transformation pipeline.

### Standard Project Layout

```text
my_dbt_project/
│
├── dbt_project.yml          # Project config, model defaults
├── packages.yml             # External package dependencies
├── .gitignore               # Exclude target/, logs/, dbt_packages/
│
├── models/                  # SQL transformation models
│   ├── staging/             # 1:1 with source tables (clean, rename)
│   │   ├── _sources.yml     # Source definitions + freshness tests
│   │   ├── stg_customers.sql
│   │   └── stg_orders.sql
│   │
│   ├── intermediate/        # Business logic, joins, filters
│   │   ├── _models.yml      # Model-level tests/docs
│   │   └── int_customer_orders.sql
│   │
│   ├── marts/               # End-user analytics models
│   │   ├── finance/
│   │   │   ├── _models.yml
│   │   │   ├── fct_revenue.sql
│   │   │   └── dim_products.sql
│   │   │
│   │   └── marketing/
│   │       ├── fct_campaigns.sql
│   │       └── dim_customers.sql
│   │
│   └── schema.yml           # Deprecated: use _models.yml per folder
│
├── tests/                   # Singular tests (custom SQL assertions)
│   ├── assert_positive_revenue.sql
│   └── assert_unique_customer_emails.sql
│
├── macros/                  # Reusable Jinja functions
│   ├── _macros.yml          # Macro documentation
│   ├── generate_schema_name.sql
│   └── custom_tests.sql
│
├── seeds/                   # CSV reference data (<1000 rows)
│   ├── country_codes.csv
│   └── product_categories.csv
│
├── snapshots/               # SCD Type 2 snapshots
│   └── snap_customers.sql
│
├── analyses/                # Ad-hoc queries (compiled but not run)
│   └── customer_cohort_analysis.sql
│
├── docs/                    # Custom documentation markdown
│   └── overview.md
│
├── target/                  # Compiled SQL, manifest.json (gitignored)
├── logs/                    # Debug logs (gitignored)
└── dbt_packages/            # Installed packages (gitignored)
```

### Diagram: dbt Project Organization

```text
┌──────────────────────────────────────────────────────────┐
│                   dbt_project.yml                        │
│  (Defines materialization defaults, schemas, vars)       │
└────────────────────────┬─────────────────────────────────┘
                         │
         ┌───────────────┼──────────────┐
         │               │              │
    ┌────▼────┐     ┌────▼────┐    ┌────▼────┐
    │ Sources │     │ Models  │    │  Seeds  │
    │ (YAML)  │     │  (SQL)  │    │  (CSV)  │
    └────┬────┘     └────┬────┘    └────┬────┘
         │               │              │
         │         ┌─────▼─────┐        │
         └────────►│  Tests    │◄───────┘
                   │  (YAML)   │
                   └─────┬─────┘
                         │
                   ┌─────▼─────┐
                   │   Docs    │
                   │ (Auto-gen)│
                   └───────────┘
```

## Parsing & Execution Lifecycle

dbt execution follows a multi-stage lifecycle: parse → compile → execute → document. Understanding this flow is critical for debugging and optimization.

### Stage 1: Parsing
- Reads `dbt_project.yml`, `profiles.yml`, and all model/source/test YAML files.
- Builds node dependency graph from `{{ ref() }}` and `{{ source() }}` references.
- Validates syntax, checks for circular dependencies.
- Output: In-memory DAG representing project structure.

### Stage 2: Compilation
- Renders Jinja templates (variables, macros, control flow).
- Resolves `{{ ref() }}` to actual table names based on target environment.
- Generates raw SQL files in `target/compiled/`.
- Does NOT execute SQL against warehouse.

### Stage 3: Execution
- Executes compiled SQL in topological order (dependencies first).
- Creates/updates tables/views based on materialization strategy.
- Logs query results, timings, row counts.
- Writes metadata to `target/run/`.

### Stage 4: Artifact Generation
- Produces `manifest.json` (full project metadata, 10-100MB for large projects).
- Creates `run_results.json` (execution stats, errors, timings).
- Generates `catalog.json` (warehouse column metadata) via `dbt docs generate`.

### Diagram: dbt Run Lifecycle

```text
┌──────────────┐
│  dbt run     │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 1. PARSE                             │
│ • Read dbt_project.yml, profiles.yml │
│ • Parse models/*.sql for {{ ref() }} │
│ • Build DAG (nodes + edges)          │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 2. COMPILE                           │
│ • Render Jinja ({{ var() }}, loops)  │
│ • Resolve {{ ref('model') }} to DB   │
│ • Write target/compiled/**/*.sql     │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 3. EXECUTE                           │
│ • Run SQL in topological order       │
│ • CREATE TABLE / CREATE VIEW         │
│ • Write target/run/**/*.sql          │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 4. ARTIFACTS                         │
│ • Generate manifest.json             │
│ • Write run_results.json             │
│ • Update catalog.json (docs only)    │
└──────────────────────────────────────┘
```

## Selectors & Node Dependency Graph

Selectors enable targeted execution of model subsets using graph traversal syntax. Critical for incremental development and CI/CD optimization.

### Selector Syntax Reference

| Selector | Description | Example |
|----------|-------------|---------|
| `<model>` | Exact model name | `dbt run --select fct_orders` |
| `+<model>` | Model + all upstream dependencies | `dbt run --select +fct_orders` |
| `<model>+` | Model + all downstream dependencies | `dbt run --select stg_customers+` |
| `+<model>+` | Model + full dependency tree | `dbt run --select +fct_orders+` |
| `n+<model>` | Model + n-levels upstream | `dbt run --select 2+fct_orders` |
| `<model>+n` | Model + n-levels downstream | `dbt run --select stg_customers+2` |
| `tag:<tag>` | All models with tag | `dbt run --select tag:nightly` |
| `path:<path>` | All models in directory | `dbt run --select path:models/staging` |
| `<model1> <model2>` | Union (space-separated) | `dbt run --select stg_orders stg_customers` |
| `<selector1>,<selector2>` | Union (comma-separated) | `dbt run --select tag:daily,path:marts` |
| `@<model>` | Model + direct parents/children only | `dbt run --select @fct_orders` |
| `*` | All models | `dbt run --select "*"` (escape in shell) |
| `config.materialized:incremental` | Models with specific config | `dbt run --select config.materialized:incremental` |
| `source:<source>` | All models using source | `dbt run --select source:salesforce` |
| `exposure:<exposure>` | Models feeding exposure | `dbt run --select +exposure:weekly_dashboard` |
| `state:modified` | Modified models vs prior state | `dbt run --select state:modified --state ./prod-run` |
| `state:new` | New models not in prior state | `dbt run --select state:new --state ./prod-run` |

### Selector Syntax Template

```bash
# ============================================================================
# SELECTOR SYNTAX
# ============================================================================

dbt <command> --select <selector1> [<selector2> ...] [--exclude <selector>]

# Operators:
# +        Upstream dependencies
# n+       N-levels upstream
# +n       N-levels downstream
# @        Direct parents/children only
# ,        Union (OR)
# (space)  Union (OR)

# Methods:
# tag:<tag>                          Models with tag
# path:<directory>                   Models in directory
# config.<key>:<value>               Models with config value
# source:<source>                    Models using source
# exposure:<exposure>                Models feeding exposure
# result:<status>                    Models with last run status (success/error/skipped)
# state:modified                     Modified vs prior state
# state:new                          New models

# Examples:
dbt run --select +fct_orders                    # Orders + dependencies
dbt run --select stg_customers+                 # Customers + downstream
dbt run --select tag:nightly                    # Tagged models
dbt run --select path:models/marts              # Folder-level selection
dbt run --select +fct_orders --exclude stg_*    # Exclude staging
dbt test --select config.severity:warn          # Warn-level tests only
```

### Graph Traversal Example

```text
Given DAG:
  stg_orders ──┐
               ├──→ int_orders_customers ──→ fct_orders ──→ exposure:revenue_dashboard
  stg_customers┘

Selector: +fct_orders
Result:   stg_orders, stg_customers, int_orders_customers, fct_orders

Selector: stg_customers+
Result:   stg_customers, int_orders_customers, fct_orders

Selector: +fct_orders+
Result:   All 5 nodes (sources to exposures)

Selector: 1+fct_orders
Result:   int_orders_customers, fct_orders

Selector: @fct_orders
Result:   int_orders_customers, fct_orders, exposure:revenue_dashboard
```

## profiles.yml Complete Configuration

`profiles.yml` defines warehouse connection credentials, typically stored in `~/.dbt/` (outside project repo for security). Each adapter (Snowflake, BigQuery, Redshift) has unique parameters.

### Syntax Template

```yaml
# ============================================================================
# COMPLETE profiles.yml SYNTAX
# ============================================================================

<profile_name>:
  target: <default_target>
  
  outputs:
    <target_name>:
      type: snowflake | bigquery | redshift | postgres | databricks
      threads: <integer>
      
      # Common parameters (varies by adapter)
      account: '<account_identifier>'
      user: '<username>'
      password: '<password>'
      role: '<warehouse_role>'
      database: '<database_name>'
      warehouse: '<warehouse_name>'
      schema: '<schema_name>'
      
      # Optional common parameters
      retry_on_database_errors: true | false
      connect_retries: <integer>
      connect_timeout: <seconds>
      retry_all: true | false
```

### Parameter Reference Table (Snowflake Adapter)

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `type` | String | (required) | Adapter type | `snowflake` |
| `account` | String | (required) | Snowflake account identifier | `abc12345.us-east-1` |
| `user` | String | (required) | Username | `dbt_user` |
| `password` | String | (required) | Password (use env_var) | `{{ env_var('DBT_PASSWORD') }}` |
| `role` | String | (required) | Warehouse role | `TRANSFORMER` |
| `database` | String | (required) | Target database | `ANALYTICS` |
| `warehouse` | String | (required) | Compute warehouse | `TRANSFORMING` |
| `schema` | String | (required) | Default schema | `dbt_prod` |
| `threads` | Integer | `4` | Parallel model execution | `8` |
| `client_session_keep_alive` | Boolean | `false` | Keep session alive | `true` |
| `query_tag` | String | `NULL` | Query tag for monitoring | `dbt_{{ invocation_id }}` |
| `connect_retries` | Integer | `0` | Connection retry attempts | `3` |
| `connect_timeout` | Integer | `10` | Connection timeout (seconds) | `30` |
| `retry_on_database_errors` | Boolean | `false` | Retry on transient errors | `true` |
| `retry_all` | Boolean | `false` | Retry all failed queries | `false` |
| `reuse_connections` | Boolean | `false` | Reuse connections across runs | `true` |
| `authenticator` | String | `snowflake` | Auth method | `externalbrowser` |
| `private_key_path` | String | `NULL` | Path to private key file | `/keys/key.p8` |
| `private_key_passphrase` | String | `NULL` | Key passphrase | `{{ env_var('KEY_PASS') }}` |
| `session_parameters` | Object | `{}` | Snowflake session params | `{QUERY_TAG: "dbt"}` |

### Basic Example

```yaml
default:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: abc12345.us-east-1
      user: my_user
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: TRANSFORMER
      database: DEV_DB
      warehouse: DEV_WH
      schema: dbt_my_user
      threads: 4
```

### Advanced Example (Multi-Environment)

```yaml
analytics_prod:
  target: dev
  
  outputs:
    dev:
      type: snowflake
      account: abc12345.us-east-1
      user: "{{ env_var('DBT_USER') }}"
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: DEVELOPER
      database: DEV_DB
      warehouse: DEV_WH
      schema: "dbt_{{ env_var('DBT_USER') }}"
      threads: 4
      query_tag: "dbt_dev_{{ invocation_id }}"
      client_session_keep_alive: true
      
    ci:
      type: snowflake
      account: abc12345.us-east-1
      user: "{{ env_var('DBT_CI_USER') }}"
      password: "{{ env_var('DBT_CI_PASSWORD') }}"
      role: CI_RUNNER
      database: CI_DB
      warehouse: CI_WH
      schema: "dbt_ci_{{ env_var('GITHUB_RUN_ID', 'local') }}"
      threads: 8
      query_tag: "dbt_ci_{{ env_var('GITHUB_SHA', 'unknown') }}"
      connect_retries: 3
      retry_on_database_errors: true
      
    prod:
      type: snowflake
      account: abc12345.us-east-1
      user: "{{ env_var('DBT_PROD_USER') }}"
      password: "{{ env_var('DBT_PROD_PASSWORD') }}"
      role: TRANSFORMER
      database: PROD_DB
      warehouse: PROD_WH
      schema: analytics
      threads: 16
      query_tag: "dbt_prod_{{ invocation_id }}"
      client_session_keep_alive: true
      session_parameters:
        QUERY_TAG: 'dbt_production'
        STATEMENT_TIMEOUT_IN_SECONDS: 7200
```

### BigQuery Adapter Example

```yaml
bigquery_prod:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: service-account
      project: my-gcp-project
      dataset: dbt_dev
      threads: 4
      keyfile: /path/to/service-account.json
      location: US
      priority: interactive
      timeout_seconds: 300
      maximum_bytes_billed: 1000000000  # 1GB limit
```

## dbt_project.yml: Models & Sources Configuration

Project-level model configurations cascade to folders/files. Source configs define upstream data catalog.

### Models Configuration Syntax

```yaml
# ============================================================================
# MODELS CONFIGURATION IN dbt_project.yml
# ============================================================================

models:
  <project_name>:
    +materialized: table | view | incremental | ephemeral
    +schema: '<schema_override>'
    +database: '<database_override>'
    +alias: '<table_name_override>'
    +tags: ['<tag1>', '<tag2>']
    +meta:
      <key>: '<value>'
    +pre-hook: ['<sql>']
    +post-hook: ['<sql>']
    +persist_docs:
      relation: true | false
      columns: true | false
    +grants:
      select: ['<role>']
      insert: ['<role>']
    +full_refresh: true | false
    +enabled: true | false
    
    # Adapter-specific (Snowflake example)
    +cluster_by: ['<column>']
    +copy_grants: true | false
    +secure: true | false
    
    # Nested folder configs
    <folder>:
      +materialized: <type>
      
      <model>:
        +materialized: <type>
```

### Models Config Parameter Table

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `+materialized` | Enum | `view` | Persistence strategy | `incremental` |
| `+schema` | String | `target.schema` | Schema override | `analytics` |
| `+database` | String | `target.database` | Database override | `prod_db` |
| `+alias` | String | `<filename>` | Table name override | `fct_orders_v2` |
| `+tags` | Array | `[]` | Selection tags | `["nightly", "pii"]` |
| `+meta` | Object | `{}` | Custom metadata | `{owner: "team"}` |
| `+pre-hook` | Array | `[]` | SQL before model | `["GRANT SELECT..."]` |
| `+post-hook` | Array | `[]` | SQL after model | `["CALL sp_log()"]` |
| `+persist_docs` | Object | `{}` | Store descriptions | `{relation: true}` |
| `+grants` | Object | `{}` | DCL permissions | `{select: ["role"]}` |
| `+full_refresh` | Boolean | `false` | Force full rebuild | `true` |
| `+enabled` | Boolean | `true` | Enable model | `false` |
| `+cluster_by` (Snowflake) | Array | `[]` | Clustering keys | `["date", "id"]` |
| `+copy_grants` (Snowflake) | Boolean | `false` | Preserve grants on refresh | `true` |
| `+secure` (Snowflake) | Boolean | `false` | Create secure view | `true` |

### Sources Configuration Syntax

```yaml
# ============================================================================
# SOURCES CONFIGURATION (in models/_sources.yml)
# ============================================================================

version: 2

sources:
  - name: <source_name>
    description: '<description>'
    database: '<database>'
    schema: '<schema>'
    loader: '<etl_tool>'
    loaded_at_field: '<timestamp_column>'
    meta:
      <key>: '<value>'
    
    freshness:
      warn_after: {count: <int>, period: minute | hour | day}
      error_after: {count: <int>, period: minute | hour | day}
    
    quoting:
      database: true | false
      schema: true | false
      identifier: true | false
    
    tables:
      - name: <table_name>
        description: '<description>'
        identifier: '<actual_table_name>'
        loaded_at_field: '<timestamp_column>'
        freshness:
          warn_after: {count: <int>, period: <unit>}
          error_after: {count: <int>, period: <unit>}
        columns:
          - name: <column_name>
            description: '<description>'
            tests:
              - not_null
              - unique
```

### Sources Config Example

```yaml
version: 2

sources:
  - name: salesforce
    description: 'Salesforce production data loaded via Fivetran'
    database: raw
    schema: salesforce
    loader: fivetran
    loaded_at_field: _fivetran_synced
    
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    
    tables:
      - name: account
        description: 'Salesforce accounts (companies/organizations)'
        identifier: sf_account  # Actual table name if different
        columns:
          - name: id
            description: 'Salesforce account ID'
            tests:
              - unique
              - not_null
          
          - name: name
            description: 'Account name'
            tests:
              - not_null
      
      - name: opportunity
        freshness:
          warn_after: {count: 6, period: hour}  # Override default
        columns:
          - name: account_id
            tests:
              - relationships:
                  to: source('salesforce', 'account')
                  field: id
```

## SDLC Use Case: Git Branching Strategy

**Scenario**: Team follows trunk-based development with feature branches, CI/CD validates PRs before merge.

### Branching Model

```text
main (protected)
  │
  ├── release/v1.2.0 (long-lived, production deploys)
  │
  ├── develop (integration branch)
  │     │
  │     ├── feature/add-customer-ltv
  │     ├── feature/fix-revenue-bug
  │     └── feature/new-marketing-metrics
  │
  └── hotfix/urgent-revenue-fix (merge to main + develop)
```

### Workflow

```bash
# 1. Create feature branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/add-customer-ltv

# 2. Develop models in personal schema
dbt run --select +fct_customer_ltv --target dev

# 3. Run tests
dbt test --select fct_customer_ltv

# 4. Commit changes
git add models/marts/finance/fct_customer_ltv.sql
git add models/marts/finance/_models.yml
git commit -m "feat(finance): Add customer LTV model with 12-month lookback"

# 5. Push and open PR
git push origin feature/add-customer-ltv
# Open PR: feature/add-customer-ltv → develop

# 6. CI runs on PR (GitHub Actions)
# - dbt compile (validate syntax)
# - dbt run --select state:modified --state ./prod-state (slim CI)
# - dbt test --select state:modified+

# 7. Peer review, approve, merge to develop

# 8. Deploy to staging from develop
# CI/CD: dbt run --target staging --select state:modified

# 9. Merge develop → release/v1.3.0 → main
# 10. Production deploy from main
# CI/CD: dbt run --target prod --full-refresh (scheduled)
```

### GitHub Actions: Slim CI Example

```yaml
# .github/workflows/dbt_slim_ci.yml
name: dbt Slim CI

on:
  pull_request:
    branches: [develop]

jobs:
  slim-ci:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Download production manifest
        run: |
          aws s3 cp s3://dbt-artifacts/prod/manifest.json ./prod-state/
      
      - name: Install dbt
        run: pip install dbt-snowflake==1.5.0
      
      - name: Run modified models only
        run: |
          dbt run \
            --select state:modified+ \
            --state ./prod-state \
            --target ci
      
      - name: Test modified models
        run: |
          dbt test \
            --select state:modified+ \
            --state ./prod-state \
            --target ci
```

## Key Takeaways

- dbt project structure separates staging/intermediate/marts for clear data flow.
- Parsing creates DAG, compilation renders Jinja, execution runs SQL, artifacts enable state comparison.
- Selectors (`+model`, `tag:`, `state:modified`) enable targeted runs critical for CI/CD.
- `profiles.yml` defines multi-environment connections (dev/ci/prod) with adapter-specific params.
- Branching strategies (trunk-based, Gitflow) integrate dbt into modern SDLC workflows.

---
