# 1. Introduction to dbt

## What is dbt?

dbt (data build tool) transforms raw data into analytics-ready models using modular SQL + Jinja templating. It follows software engineering best practices (version control, tests, documentation) in ELT pipelines. Works with Snowflake, BigQuery, Redshift, Postgres, Databricks via adapters.

**Key Value Proposition**: dbt enables analytics engineers to work like software engineers — write modular transformations, test data quality, and deploy with CI/CD.

## ELT vs ETL

**ETL (Extract-Transform-Load)**: Data transformed before warehouse load (e.g., Informatica, Talend). **ELT (Extract-Load-Transform)**: Raw data loaded first, transformed in warehouse using dbt. Modern data warehouses (Snowflake, BigQuery) leverage compute power for in-warehouse transformations, making ELT more efficient.

### Diagram: ELT Flow with dbt

```text
┌─────────────────┐
│ Source Systems  │
│ (APIs, DBs)     │
└────────┬────────┘
         │ Extract & Load (Fivetran, Airbyte, Stitch)
         ↓
┌─────────────────┐
│ Data Warehouse  │
│ (Raw/Landing)   │
└────────┬────────┘
         │ dbt Transformation Layer
         ↓
┌─────────────────┐     ┌──────────────┐     ┌──────────┐
│ Staging Models  │ --→ │ Intermediate │ --→ │   Marts  │
│ (clean, rename) │     │ (business    │     │   (star  │ --→ BI Tools
│                 │     │  logic)      │     │   schema)│     (Tableau,
└─────────────────┘     └──────────────┘     └──────────┘      Looker)
                                             
```

## Core Concepts

### Models
SQL SELECT statements that create tables/views/incremental models in the warehouse. Each model is a `.sql` file in the `models/` directory. Models reference other models using `{{ ref('model_name') }}` to build dependency graphs automatically.

### Sources
Upstream raw tables cataloged in YAML with metadata (freshness, descriptions). Reference via `{{ source('schema', 'table') }}` to track lineage. Sources enable freshness tests to alert when data pipelines break.

### Tests
Assertions on data quality: `unique`, `not_null`, `relationships`, `accepted_values`, or custom SQL. Tests run via `dbt test` and fail builds on violations. Generic tests defined in `schema.yml`, singular tests as SQL files.

### Documentation
Auto-generated from YAML metadata (`description` fields) and model SQL. `dbt docs generate` creates a static site with lineage DAG. Serves as single source of truth for data catalog.

### Seeds
CSV files loaded as static tables (e.g., country codes, mappings). Use `dbt seed` to materialize. Useful for small reference data (<1000 rows) versioned in Git.

### Snapshots
Type 2 Slowly Changing Dimensions (SCD) tracking historical changes. Capture point-in-time state of mutable source tables. Use `timestamp` or `check` strategies.

## dbt_project.yml Complete Configuration

`dbt_project.yml` is the project's root config file defining project name, version, model paths, and global settings.

### Syntax Template

```yaml
# ============================================================================
# COMPLETE dbt_project.yml SYNTAX
# ============================================================================

# Required: Project identifier (must be unique in dbt Cloud/Package ecosystem)
name: '<project_name>'

# Required: Project version (semantic versioning recommended)
version: '<version_string>'

# Required: Config file version (always 2 for modern dbt)
config-version: 2

# Required: Profile name from profiles.yml to use
profile: '<profile_name>'

# Optional: Model paths (default: ["models"])
model-paths: ["<path1>", "<path2>"]

# Optional: Analysis paths for ad-hoc queries (default: ["analyses"])
analysis-paths: ["<path>"]

# Optional: Test paths for singular tests (default: ["tests"])
test-paths: ["<path>"]

# Optional: Seed paths for CSV files (default: ["seeds"])
seed-paths: ["<path>"]

# Optional: Macro paths for Jinja macros (default: ["macros"])
macro-paths: ["<path>"]

# Optional: Snapshot paths for SCD Type 2 (default: ["snapshots"])
snapshot-paths: ["<path>"]

# Optional: Documentation paths (default: ["docs"])
docs-paths: ["<path>"]

# Optional: Asset paths for images/files in docs (default: ["assets"])
asset-paths: ["<path>"]

# Optional: Target path for compiled SQL/artifacts (default: "target")
target-path: "<path>"

# Optional: Paths to clean with dbt clean (default: ["target", "dbt_packages"])
clean-targets:
  - "<path1>"
  - "<path2>"

# Optional: Log path (default: "logs")
log-path: "<path>"

# Optional: Packages install path (default: "dbt_packages")
packages-install-path: "<path>"

# Optional: Query comment for warehouse query logs
query-comment:
  comment: '<comment_string>'
  append: true | false
  job_label: true | false

# Optional: Require explicit dbt version range
require-dbt-version: [">=1.0.0", "<2.0.0"]

# Optional: Global project variables accessible via {{ var('var_name') }}
vars:
  <var_name>: '<value>'
  <var_name>: <numeric_value>
  <var_name>: true | false

# Optional: Quoting configuration for object names
quoting:
  database: true | false
  schema: true | false
  identifier: true | false

# Optional: Model-level configurations (cascade to all models)
models:
  <project_name>:
    +materialized: table | view | incremental | ephemeral
    +schema: '<schema_override>'
    +database: '<database_override>'
    +alias: '<table_name_override>'
    +tags: ['<tag1>', '<tag2>']
    +meta:
      <key>: '<value>'
    +persist_docs:
      relation: true | false
      columns: true | false
    +grants:
      select: ['<role1>', '<role2>']
      insert: ['<role>']
      update: ['<role>']
    +enabled: true | false
    
    # Nested folder-level configs
    <folder_name>:
      +materialized: <type>
      +schema: '<schema>'
      
      # Model-specific overrides
      <model_name>:
        +materialized: <type>

# Optional: Seed configurations
seeds:
  <project_name>:
    +schema: '<schema>'
    +enabled: true | false
    +quote_columns: true | false
    +column_types:
      <column_name>: '<data_type>'

# Optional: Snapshot configurations
snapshots:
  <project_name>:
    +target_schema: '<schema>'
    +strategy: timestamp | check
    +unique_key: '<column>'
    +updated_at: '<timestamp_column>'  # for timestamp strategy
    +check_cols: ['<col1>', '<col2>']   # for check strategy
    +invalidate_hard_deletes: true | false

# Optional: Test configurations
tests:
  <project_name>:
    +severity: warn | error
    +error_if: '>100'
    +warn_if: '>10'
    +store_failures: true | false
    +schema: '<schema_for_failed_records>'

# Optional: Source configurations
sources:
  <project_name>:
    +enabled: true | false
    +database: '<database>'
    +schema: '<schema>'
    +quoting:
      database: true | false
      schema: true | false
      identifier: true | false

# Optional: Analysis configurations
analyses:
  <project_name>:
    +enabled: true | false

# Optional: Macro configurations
dispatch:
  - macro_namespace: <package_name>
    search_order: ['<project1>', '<project2>']

# Optional: On-run-start/end hooks (SQL to execute)
on-run-start:
  - '<sql_statement>'

on-run-end:
  - '<sql_statement>'
```

### Parameter Reference Table

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `name` | String | (required) | Unique project identifier | `my_dbt_project` |
| `version` | String | `1.0.0` | Semantic version | `1.2.0` |
| `config-version` | Integer | `2` | Config file schema version (always 2) | `2` |
| `profile` | String | (required) | Profile name from profiles.yml | `snowflake_prod` |
| `model-paths` | Array | `["models"]` | Directories containing model SQL files | `["models", "custom_models"]` |
| `analysis-paths` | Array | `["analyses"]` | Ad-hoc query directories | `["analyses"]` |
| `test-paths` | Array | `["tests"]` | Singular test directories | `["tests"]` |
| `seed-paths` | Array | `["seeds"]` | CSV seed file directories | `["seeds", "data"]` |
| `macro-paths` | Array | `["macros"]` | Jinja macro directories | `["macros"]` |
| `snapshot-paths` | Array | `["snapshots"]` | Snapshot directories | `["snapshots"]` |
| `docs-paths` | Array | `["docs"]` | Documentation markdown directories | `["docs"]` |
| `asset-paths` | Array | `["assets"]` | Static asset directories for docs | `["assets"]` |
| `target-path` | String | `target` | Compiled SQL output directory | `target` |
| `clean-targets` | Array | `["target", "dbt_packages"]` | Paths cleaned by `dbt clean` | `["target", "logs"]` |
| `log-path` | String | `logs` | Log file directory | `logs` |
| `packages-install-path` | String | `dbt_packages` | Installed package directory | `dbt_modules` |
| `query-comment` | Object | `{}` | Comment appended to warehouse queries | `{comment: "dbt_user"}` |
| `require-dbt-version` | Array | `[]` | Enforce dbt version range | `[">=1.0.0", "<2.0.0"]` |
| `vars` | Object | `{}` | Global variables accessible in Jinja | `{start_date: "2023-01-01"}` |
| `quoting.database` | Boolean | `false` | Quote database names | `true` |
| `quoting.schema` | Boolean | `false` | Quote schema names | `true` |
| `quoting.identifier` | Boolean | `false` | Quote table/column names | `true` |
| `models.<folder>.+materialized` | Enum | `view` | Persistence strategy | `incremental` |
| `models.<folder>.+schema` | String | `target.schema` | Schema override | `analytics` |
| `models.<folder>.+database` | String | `target.database` | Database override | `prod_db` |
| `models.<folder>.+alias` | String | `<filename>` | Table name override | `fct_orders` |
| `models.<folder>.+tags` | Array | `[]` | Tags for selection | `["nightly", "pii"]` |
| `models.<folder>.+meta` | Object | `{}` | Custom metadata | `{owner: "data_team"}` |
| `models.<folder>.+persist_docs` | Object | `{}` | Persist descriptions to warehouse | `{relation: true}` |
| `models.<folder>.+grants` | Object | `{}` | DCL permissions | `{select: ["analyst_role"]}` |
| `models.<folder>.+enabled` | Boolean | `true` | Enable/disable models | `false` |
| `seeds.<seed>.+quote_columns` | Boolean | `false` | Quote CSV column names | `true` |
| `seeds.<seed>.+column_types` | Object | `{}` | Override inferred types | `{id: "varchar(50)"}` |
| `snapshots.<snap>.+strategy` | Enum | (required) | SCD strategy | `timestamp` |
| `snapshots.<snap>.+unique_key` | String | (required) | Primary key column | `customer_id` |
| `snapshots.<snap>.+updated_at` | String | (required for timestamp) | Timestamp column | `updated_at` |
| `snapshots.<snap>.+check_cols` | Array | (required for check) | Columns to monitor | `["status", "email"]` |
| `tests.<test>.+severity` | Enum | `error` | Test failure severity | `warn` |
| `tests.<test>.+store_failures` | Boolean | `false` | Store failed rows in table | `true` |
| `on-run-start` | Array | `[]` | SQL executed before dbt run | `["GRANT USAGE ON SCHEMA..."]` |
| `on-run-end` | Array | `[]` | SQL executed after dbt run | `["CALL sp_log_run()"]` |

### Basic Example

```yaml
name: 'analytics_dbt'
version: '1.0.0'
config-version: 2

profile: 'default'

model-paths: ["models"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  analytics_dbt:
    +materialized: view
    +schema: analytics
```

### Advanced Example

```yaml
name: 'ecommerce_analytics'
version: '2.1.0'
config-version: 2

profile: 'snowflake_prod'

model-paths: ["models", "custom_models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds", "reference_data"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
log-path: "logs"
packages-install-path: "dbt_packages"

clean-targets:
  - "target"
  - "dbt_packages"
  - "logs"

require-dbt-version: [">=1.5.0", "<2.0.0"]

vars:
  start_date: '2023-01-01'
  fiscal_year_start_month: 4
  enable_pii_masking: true

query-comment:
  comment: "dbt_{{ var('dbt_cloud_project_id', 'local') }}_{{ invocation_id }}"
  append: true

quoting:
  database: false
  schema: false
  identifier: false

models:
  ecommerce_analytics:
    +materialized: view
    +persist_docs:
      relation: true
      columns: true
    +meta:
      owner: 'data_engineering'
      
    staging:
      +materialized: view
      +schema: staging
      +tags: ['staging', 'daily']
      
    intermediate:
      +materialized: ephemeral
      +tags: ['intermediate']
      
    marts:
      +materialized: table
      +schema: analytics
      +tags: ['marts', 'production']
      
      finance:
        +schema: finance
        +grants:
          select: ['finance_role', 'analyst_role']
        +tags: ['finance', 'pii']
        
      marketing:
        +materialized: incremental
        +schema: marketing
        +tags: ['marketing']
        
        fct_campaign_performance:
          +cluster_by: ['campaign_date']
          +unique_key: 'campaign_id'

seeds:
  ecommerce_analytics:
    +schema: seed_data
    +quote_columns: false
    
    country_codes:
      +column_types:
        country_code: varchar(2)
        country_name: varchar(100)

snapshots:
  ecommerce_analytics:
    +target_schema: snapshots
    +strategy: timestamp
    +unique_key: id
    +updated_at: updated_at

tests:
  ecommerce_analytics:
    +severity: error
    +store_failures: true
    +schema: test_failures

on-run-start:
  - "{{ create_schema_if_not_exists() }}"
  - "GRANT USAGE ON SCHEMA {{ target.schema }} TO ROLE analyst_role"

on-run-end:
  - "{{ log_run_metadata() }}"
```

## SDLC Use Case: GitHub Repository Onboarding

**Scenario**: New analytics engineer joins team, needs to set up local dbt environment.

### Workflow

```bash
# 1. Clone repository
git clone https://github.com/company/analytics-dbt.git
cd analytics-dbt

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. Install dbt adapter (Snowflake example)
pip install dbt-snowflake

# 4. Configure profiles.yml (create in ~/.dbt/)
cat > ~/.dbt/profiles.yml <<EOF
default:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: abc12345.us-east-1
      user: "{{ env_var('DBT_USER') }}"
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: ANALYST
      database: DEV_DB
      warehouse: DEV_WH
      schema: dbt_{{ env_var('DBT_USER') }}
      threads: 4
EOF

# 5. Set environment variables
export DBT_USER=your_username
export DBT_PASSWORD=your_password

# 6. Install dbt packages
dbt deps

# 7. Verify connection
dbt debug

# 8. Compile models (no warehouse execution)
dbt compile

# 9. Create feature branch
git checkout -b feature/new-metrics

# 10. Develop models, run in dev schema
dbt run --select +my_new_model

# 11. Test changes
dbt test --select my_new_model

# 12. Generate documentation
dbt docs generate
dbt docs serve  # View at http://localhost:8080

# 13. Commit and push PR
git add models/marts/my_new_model.sql
git commit -m "feat: Add new customer lifetime value model"
git push origin feature/new-metrics

# 14. Open PR → CI/CD runs dbt compile + test → Peer review → Merge
```

### GitHub Actions CI/CD Example

```yaml
# .github/workflows/dbt_ci.yml
name: dbt CI

on:
  pull_request:
    branches: [main]
    paths:
      - 'models/**'
      - 'tests/**'
      - 'macros/**'

jobs:
  dbt-test:
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
      
      - name: Install packages
        run: dbt deps
        
      - name: Compile models
        run: dbt compile
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.DBT_ACCOUNT }}
          DBT_SNOWFLAKE_USER: ${{ secrets.DBT_USER }}
          DBT_SNOWFLAKE_PASSWORD: ${{ secrets.DBT_PASSWORD }}
      
      - name: Run tests
        run: dbt test --profiles-dir ./ci
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.DBT_ACCOUNT }}
```

### profiles.yml for CI Environment

```yaml
# ci/profiles.yml
default:
  target: ci
  outputs:
    ci:
      type: snowflake
      account: "{{ env_var('DBT_SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('DBT_SNOWFLAKE_USER') }}"
      password: "{{ env_var('DBT_SNOWFLAKE_PASSWORD') }}"
      role: CI_RUNNER
      database: CI_DB
      warehouse: CI_WH
      schema: dbt_ci_{{ env_var('GITHUB_RUN_ID', 'local') }}
      threads: 8
      query_tag: "dbt_ci_{{ env_var('GITHUB_SHA', 'unknown') }}"
```

## Key Takeaways

- dbt enables analytics engineering with software SDLC practices (Git, tests, CI/CD).
- `dbt_project.yml` is the central config file defining project structure and model defaults.
- Models reference each other via `{{ ref() }}` and sources via `{{ source() }}` for automatic lineage.
- Tests ensure data quality; documentation auto-generates from YAML metadata.
- GitHub workflows integrate dbt into modern DevOps pipelines with automated testing gates.

---
