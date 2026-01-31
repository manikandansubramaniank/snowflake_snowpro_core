# 6. Deployment & Orchestration

## Overview
dbt deployment bridges development models to production warehouses via CLI commands (dbt Core) or managed jobs (dbt Cloud). State-based selection optimizes CI/CD by running only modified models. Artifacts (manifest, run_results) enable lineage tracking, slim CI, and auditing.

---

## 6.1 Deployment Patterns

### dbt Core CLI vs dbt Cloud

**Diagram: Deployment Architectures**
```text
┌─────────────────────────────────────────────────────────────────┐
│ dbt Core CLI (Self-Managed)                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Developer → Git Push → GitHub Actions → dbt run → Snowflake    │
│                ↓                           ↓                    │
│         profiles.yml (local/CI)      manifest.json artifacts    │
│                                                                 │
│  Pros: Free, full control, integrate any orchestrator           │
│  Cons: Manual infrastructure, no managed scheduler              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ dbt Cloud (Managed Platform)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Developer → Git Push → dbt Cloud Job → Snowflake               │
│                ↓              ↓             ↓                   │
│          Auto-sync    Scheduler + IDE    Lineage + Docs         │
│                                                                 │
│  Pros: Managed infra, IDE, scheduler, observability             │
│  Cons: Subscription cost, less orchestration flexibility        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Hybrid: dbt Core + External Orchestrator (Airflow/Dagster)      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Airflow DAG → BashOperator(dbt run) → Sensors → Downstream     │
│        ↓                    ↓                      ↓            │
│  Complex workflows   State artifacts      BI refresh triggers   │
│                                                                 │
│  Use Case: Multi-tool pipelines (dbt + Python + Spark)          │
└─────────────────────────────────────────────────────────────────┘
```

**dbt Core CLI:** Free, self-hosted. Integrate with GitHub Actions, GitLab CI, Jenkins, Airflow. Full control over infra.

**dbt Cloud:** SaaS platform with IDE, scheduler, docs hosting, lineage DAG. Paid tiers for teams.

**Hybrid:** dbt Core in external orchestrators (Airflow/Dagster) for complex multi-tool workflows.

---

## 6.2 dbt Core CLI Commands

### Syntax: Core Commands

```bash
# ─────────────────────────────────────────────────────────────
# SYNTAX TEMPLATE
# ─────────────────────────────────────────────────────────────

# Run models
dbt run [options]
  --select <selector>         # Run specific models
  --exclude <selector>        # Skip models
  --full-refresh             # Rebuild incremental models from scratch
  --vars '{<key>: <value>}'  # Override variables
  --target <target_name>     # Use specific profile target
  --threads <N>              # Parallel execution (default: 4)
  --fail-fast                # Stop on first error
  --state <path>             # State artifacts for comparison

# Test data quality
dbt test [options]
  --select <selector>
  --store-failures           # Save failing rows
  --fail-fast

# Install dependencies
dbt deps
  --lock                     # Create/update packages-lock.yml

# Generate documentation
dbt docs generate
  --no-compile               # Skip compilation (reuse manifest)
  --target-path <path>       # Output directory (default: target/)

dbt docs serve
  --port <port>              # Serve docs on localhost (default: 8080)

# Snapshot SCD Type 2 tables
dbt snapshot [options]
  --select <snapshot_name>

# List models (dry-run)
dbt ls [options]
  --select <selector>        # Preview selection
  --output <format>          # Output: json, name, path
  --resource-type model      # Filter: model, test, snapshot, source

# Compile SQL without running
dbt compile [options]
  --select <selector>

# Build (run + test)
dbt build [options]
  --select <selector>        # Run models then tests atomically

# Source freshness checks
dbt source freshness [options]
  --select source:<name>
  --output <file>            # Save results to JSON

# Clean generated files
dbt clean
  # Removes target/, dbt_packages/, logs/

# Debug connection
dbt debug
  --config-dir <path>        # Override profiles.yml location
```

---

**PARAMETERS (Common Across Commands)**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `--select` | String | All models | Node selector: model names, tags, paths, graph operators (+, @) | `tag:daily` or `staging.*+` |
| `--exclude` | String | None | Exclude nodes from selection | `tag:deprecated` |
| `--full-refresh` | Boolean | `false` | Rebuild incremental models/snapshots completely | N/A |
| `--vars` | JSON | `{}` | Override project variables at runtime | `'{start_date: "2024-01-01"}'` |
| `--target` | String | `default` | Profile target (dev/prod/ci) | `prod` |
| `--threads` | Integer | `4` | Parallel thread count | `8` |
| `--fail-fast` | Boolean | `false` | Stop execution on first error | N/A |
| `--state` | Path | None | Directory with prior run artifacts (manifest.json) | `./prod-artifacts/` |
| `--defer` | Boolean | `false` | Use prod artifacts for unbuilt upstream refs (with --state) | N/A |

---

**BASIC EXAMPLES**

```bash
# Run all models
dbt run

# Run specific model
dbt run --select customers

# Run model + downstream
dbt run --select customers+

# Run models in staging folder
dbt run --select staging.*

# Full refresh incremental model
dbt run --full-refresh --select fct_orders

# Override variables
dbt run --vars '{lookback_days: 30, env: dev}'

# Use production target
dbt run --target prod --threads 16
```

---

**ADVANCED EXAMPLES**

```bash
# Slim CI: Run only modified models
dbt run --select state:modified+ --state ./prod-artifacts/ --defer

# Run all marts tagged 'daily'
dbt run --select tag:daily --exclude tag:deprecated

# Run models matching pattern
dbt run --select fct_* dim_customers

# Atomic build (run + test together)
dbt build --select +fct_orders  # Run upstream, fct_orders, then tests

# List models without running
dbt ls --select tag:pii --output name

# Compile and inspect SQL
dbt compile --select fct_revenue
# Check target/compiled/<project>/models/fct_revenue.sql

# Source freshness check
dbt source freshness --select source:ecommerce --output target/freshness.json

# Clean all generated files
dbt clean
```

---

## 6.3 State-Based Selection

### Concept
State-based selection compares current project state to prior artifacts (manifest.json from production). Enables **slim CI** by running only modified models, reducing build times by 90%+.

**Diagram: State Comparison**
```text
Prod Artifacts (manifest.json from last prod run)
  ↓
┌──────────────────────────────────────────────────┐
│ dbt run --select state:modified+ --state ./prod/ │
└──────────────────────────────────────────────────┘
  ↓
Compares:
  - Model SQL (checksum changes)
  - Config changes (+materialized, +tags, etc.)
  - Upstream dependencies (ref() changes)
  ↓
Runs ONLY:
  - Modified models
  - Downstream models (+ operator)
  - New models (not in prod manifest)
```

**Use Cases:**
- **CI Pipelines:** Test only changed models on PRs
- **Incremental Deployments:** Deploy only affected models to prod
- **Cost Optimization:** Avoid re-running unchanged models

---

### Syntax: State Selectors

```bash
# ─────────────────────────────────────────────────────────────
# STATE SELECTOR SYNTAX
# ─────────────────────────────────────────────────────────────

dbt run --select <state_selector> --state <artifact_path>

# State Selectors:
state:modified             # Models with code/config changes
state:modified+            # Modified models + downstream
state:new                  # Models not in comparison manifest
state:modified.body        # SQL body changed (ignoring config)
state:modified.configs     # Config changed (ignoring SQL)

# Combine with other selectors:
state:modified+ tag:daily  # Modified daily models + downstream
```

**PARAMETERS**

| Selector | Description | Example Use Case |
|----------|-------------|------------------|
| `state:modified` | Models with SQL or config changes | PR validation (run changed models only) |
| `state:modified+` | Modified + all downstream models | CI/CD pipeline (test impact) |
| `state:new` | Models added since comparison state | Deploy new models separately |
| `state:modified.body` | SQL changed (config unchanged) | Detect logic changes only |
| `state:modified.configs` | Config changed (SQL unchanged) | Audit materialization changes |

**--defer Flag:** Use production refs for unbuilt upstream models. If `state:modified` skips model A but model B depends on it, `--defer` uses prod table for `ref('model_a')`.

---

**BASIC EXAMPLE: Slim CI**

```bash
# Step 1: Download prod artifacts (manifest.json)
aws s3 cp s3://dbt-artifacts/prod/manifest.json ./prod-artifacts/

# Step 2: Run only modified models
dbt run --select state:modified+ --state ./prod-artifacts/

# Result: Only changed models + downstream are built (90% faster)
```

---

**ADVANCED EXAMPLE: GitHub Actions Slim CI**

```yaml
# .github/workflows/dbt_slim_ci.yml
name: dbt Slim CI

on:
  pull_request:
    branches: [main]

jobs:
  slim-ci:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout PR branch
        uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dbt
        run: pip install dbt-snowflake==1.7.0
      
      - name: Configure profiles
        run: |
          mkdir -p ~/.dbt
          echo "${{ secrets.DBT_PROFILES_YML }}" > ~/.dbt/profiles.yml
      
      - name: Download prod artifacts
        run: |
          mkdir -p ./prod-artifacts
          aws s3 cp s3://my-dbt-bucket/prod/manifest.json ./prod-artifacts/
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Install dependencies
        run: dbt deps
      
      - name: Run modified models (slim CI)
        run: |
          dbt run --select state:modified+ --state ./prod-artifacts/ --defer --target dev
      
      - name: Test modified models
        run: |
          dbt test --select state:modified+ --state ./prod-artifacts/ --target dev
      
      - name: Comment PR with results
        if: always()
        uses: actions/github-script@v6
        with:
          script: |
            const runResults = require('./target/run_results.json');
            const summary = `## dbt Slim CI Results\n\nModels run: ${runResults.results.length}\nStatus: ${runResults.success ? '✅ PASS' : '❌ FAIL'}`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: summary
            });
```

---

## 6.4 Artifacts

### Types and Usage
dbt generates JSON artifacts in `target/` directory after each run. These enable state comparison, lineage, auditing, and integrations.

**Diagram: Artifact Ecosystem**
```text
dbt run/test/docs generate
         ↓
┌─────────────────────────────────────────┐
│ target/ Artifacts                       │
├─────────────────────────────────────────┤
│ manifest.json      → Project state      │
│ run_results.json   → Execution log      │
│ catalog.json       → Column metadata    │
│ sources.json       → Source freshness   │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│ Consumers                               │
├─────────────────────────────────────────┤
│ • dbt Cloud (lineage DAG)               │
│ • State comparison (slim CI)            │
│ • BI tools (metadata import)            │
│ • Data catalogs (Monte Carlo, Datafold) │
│ • Custom monitoring dashboards          │
└─────────────────────────────────────────┘
```

---

### Artifact Files

| File | Generated By | Contains | Use Cases |
|------|-------------|----------|-----------|
| `manifest.json` | `run`, `compile`, `test`, `docs generate` | Full project graph: models, tests, macros, configs, dependencies | State comparison, lineage, impact analysis |
| `run_results.json` | `run`, `test`, `snapshot`, `seed` | Execution results: timing, status, row counts, errors | CI/CD reporting, SLA monitoring, cost tracking |
| `catalog.json` | `docs generate` | Column-level metadata: types, descriptions, stats | Data catalog integration, column lineage |
| `sources.json` | `source freshness` | Freshness test results | Alerting on stale data, SLA compliance |

---

**manifest.json Structure (Key Fields):**
```json
{
  "metadata": {
    "dbt_version": "1.7.0",
    "generated_at": "2024-01-31T15:30:00Z",
    "env": {"DBT_CLOUD_PROJECT_ID": "123"}
  },
  "nodes": {
    "model.project.fct_orders": {
      "unique_id": "model.project.fct_orders",
      "name": "fct_orders",
      "resource_type": "model",
      "package_name": "project",
      "path": "marts/fct_orders.sql",
      "config": {
        "materialized": "incremental",
        "tags": ["daily", "prod"]
      },
      "checksum": {"name": "sha256", "checksum": "abc123..."},
      "depends_on": {
        "nodes": ["model.project.stg_orders"]
      }
    }
  }
}
```

**run_results.json Structure:**
```json
{
  "metadata": {"generated_at": "2024-01-31T16:00:00Z"},
  "elapsed_time": 42.5,
  "success": true,
  "results": [
    {
      "unique_id": "model.project.fct_orders",
      "status": "success",
      "execution_time": 12.3,
      "rows_affected": 15420,
      "message": "CREATE TABLE"
    }
  ]
}
```

---

### Artifact Management Best Practices

**1. Store in Object Storage (S3/GCS/Azure Blob)**
```bash
# After prod run, upload artifacts
dbt run --target prod
aws s3 cp target/manifest.json s3://dbt-artifacts/prod/manifest.json
aws s3 cp target/run_results.json s3://dbt-artifacts/prod/run_results_$(date +%Y%m%d_%H%M%S).json
```

**2. Version Artifacts by Git Commit**
```bash
# Tag with commit hash
COMMIT_SHA=$(git rev-parse HEAD)
aws s3 cp target/manifest.json s3://dbt-artifacts/prod/${COMMIT_SHA}/manifest.json
```

**3. Retention Policy**
```bash
# Keep last 30 days of run_results, only latest manifest
aws s3api put-bucket-lifecycle-configuration --bucket dbt-artifacts --lifecycle-configuration '{
  "Rules": [{
    "Id": "delete-old-run-results",
    "Prefix": "prod/run_results",
    "Status": "Enabled",
    "Expiration": {"Days": 30}
  }]
}'
```

---

## 6.5 dbt Cloud Jobs

### Concept
dbt Cloud provides managed scheduling, IDE, and observability. Jobs are scheduled runs with environment configs (target, threads, timeout).

**Job Types:**
- **Production Job:** Scheduled (cron), runs on `main` branch
- **CI Job:** Triggered on PR, uses slim CI with `state:modified+`
- **Ad-Hoc Job:** Manual trigger from IDE or API

---

### dbt Cloud Job Configuration

**Visual Workflow (dbt Cloud UI):**
```text
Create Job → Configure:
  ├── Environment: Prod / Dev / CI
  ├── Commands:
  │     dbt run --select tag:daily
  │     dbt test --select tag:critical
  ├── Schedule: Cron (0 6 * * *) or event-based
  ├── Git: Branch (main), auto-fetch on merge
  ├── Threads: 8
  ├── Target Path: target/
  └── Run Timeout: 3600s
```

**API Trigger (via dbt Cloud API):**
```bash
# Trigger job via API
curl -X POST \
  https://cloud.getdbt.com/api/v2/accounts/{account_id}/jobs/{job_id}/run/ \
  -H "Authorization: Token ${DBT_CLOUD_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "cause": "API trigger from Airflow",
    "git_sha": "abc123",
    "schema_override": "dev_schema"
  }'
```

**Response:**
```json
{
  "data": {
    "id": 98765,
    "href": "https://cloud.getdbt.com/#/accounts/123/projects/456/runs/98765/",
    "status": 10  // 10 = Queued, 3 = Success, 20 = Failed
  }
}
```

---

## 6.6 SDLC Integration: GitHub Actions + dbt Cloud

### Use Case: Hybrid CI/CD Pipeline
Use GitHub Actions for slim CI on PRs, dbt Cloud for production scheduling.

**Pattern:**
- **PR → GitHub Actions:** Slim CI (`state:modified+`)
- **Merge → dbt Cloud:** Production job trigger via webhook
- **Schedule → dbt Cloud:** Daily full refresh at 6 AM UTC

---

### GitHub Actions + dbt Cloud Workflow

```yaml
# .github/workflows/dbt_prod_deploy.yml
name: dbt Production Deploy

on:
  push:
    branches: [main]

jobs:
  trigger-dbt-cloud:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger dbt Cloud Job
        run: |
          RESPONSE=$(curl -X POST \
            https://cloud.getdbt.com/api/v2/accounts/${{ secrets.DBT_CLOUD_ACCOUNT_ID }}/jobs/${{ secrets.DBT_CLOUD_JOB_ID }}/run/ \
            -H "Authorization: Token ${{ secrets.DBT_CLOUD_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "cause": "GitHub Actions deploy from main",
              "git_branch": "main",
              "steps_override": [
                "dbt deps",
                "dbt run --select state:modified+ --defer --state prod",
                "dbt test --select state:modified+"
              ]
            }')
          
          echo "Response: $RESPONSE"
          RUN_ID=$(echo $RESPONSE | jq -r '.data.id')
          echo "dbt Cloud Run ID: $RUN_ID"
      
      - name: Wait for dbt Cloud Job
        run: |
          while true; do
            STATUS=$(curl -s \
              https://cloud.getdbt.com/api/v2/accounts/${{ secrets.DBT_CLOUD_ACCOUNT_ID }}/runs/$RUN_ID/ \
              -H "Authorization: Token ${{ secrets.DBT_CLOUD_API_TOKEN }}" \
              | jq -r '.data.status')
            
            if [ "$STATUS" == "3" ]; then
              echo "✅ dbt Cloud job succeeded"
              exit 0
            elif [ "$STATUS" == "20" ]; then
              echo "❌ dbt Cloud job failed"
              exit 1
            else
              echo "⏳ Job running... (status: $STATUS)"
              sleep 30
            fi
          done
```

---

## 6.7 External Orchestrators

### Airflow Integration

**Pattern:** BashOperator executes `dbt run` commands. Use XComs to pass artifacts between tasks.

```python
# dags/dbt_daily_pipeline.py
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.sensors.external_task import ExternalTaskSensor
from datetime import datetime, timedelta

default_args = {
    'owner': 'data_team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=5)
}

with DAG(
    'dbt_daily_pipeline',
    default_args=default_args,
    schedule_interval='0 6 * * *',  # 6 AM UTC daily
    catchup=False,
    tags=['dbt', 'analytics']
) as dag:

    # Check source freshness
    check_freshness = BashOperator(
        task_id='check_source_freshness',
        bash_command="""
        cd /opt/dbt/project &&
        dbt source freshness --select source:ecommerce --output target/freshness.json
        """,
        env={
            'DBT_PROFILES_DIR': '/opt/dbt/.dbt',
            'DBT_TARGET': 'prod'
        }
    )

    # Install dependencies
    dbt_deps = BashOperator(
        task_id='dbt_deps',
        bash_command='cd /opt/dbt/project && dbt deps'
    )

    # Run staging models
    run_staging = BashOperator(
        task_id='run_staging_models',
        bash_command="""
        cd /opt/dbt/project &&
        dbt run --select staging.* --target prod --threads 8
        """,
        env={'DBT_PROFILES_DIR': '/opt/dbt/.dbt'}
    )

    # Run mart models (incremental)
    run_marts = BashOperator(
        task_id='run_mart_models',
        bash_command="""
        cd /opt/dbt/project &&
        dbt run --select marts.* --target prod --threads 16 --exclude tag:full_refresh
        """,
        env={'DBT_PROFILES_DIR': '/opt/dbt/.dbt'}
    )

    # Test data quality
    test_models = BashOperator(
        task_id='test_models',
        bash_command="""
        cd /opt/dbt/project &&
        dbt test --select tag:critical --target prod --store-failures
        """
    )

    # Snapshots (SCD Type 2)
    run_snapshots = BashOperator(
        task_id='run_snapshots',
        bash_command='cd /opt/dbt/project && dbt snapshot --target prod'
    )

    # Generate docs
    generate_docs = BashOperator(
        task_id='generate_docs',
        bash_command="""
        cd /opt/dbt/project &&
        dbt docs generate --target prod &&
        aws s3 sync target/ s3://dbt-docs-bucket/
        """
    )

    # Define dependencies
    check_freshness >> dbt_deps >> run_staging
    run_staging >> [run_marts, run_snapshots]
    run_marts >> test_models >> generate_docs
```

---

### Dagster Integration

**Pattern:** dbt assets as Dagster software-defined assets. Dagster parses manifest.json for lineage.

```python
# dagster_dbt_project.py
from dagster import asset, AssetExecutionContext, Definitions
from dagster_dbt import DbtCliResource, dbt_assets

dbt_resource = DbtCliResource(
    project_dir="/opt/dbt/project",
    profiles_dir="/opt/dbt/.dbt",
    target="prod"
)

@dbt_assets(manifest="/opt/dbt/project/target/manifest.json")
def dbt_analytics_models(context: AssetExecutionContext, dbt: DbtCliResource):
    """All dbt models as Dagster assets"""
    yield from dbt.cli(["run"], context=context).stream()

@asset(deps=[dbt_analytics_models])
def export_to_s3(context):
    """Export mart tables to S3 after dbt completes"""
    context.log.info("Exporting marts to S3...")
    # Export logic here

defs = Definitions(
    assets=[dbt_analytics_models, export_to_s3],
    resources={"dbt": dbt_resource}
)
```

---

## 6.8 CLI Reference: Advanced Patterns

### Selector Syntax

```bash
# ─────────────────────────────────────────────────────────────
# SELECTOR SYNTAX REFERENCE
# ─────────────────────────────────────────────────────────────

# BASIC SELECTORS
<model_name>           # Specific model: customers
<folder>.*             # All models in folder: staging.*
<folder>.*+            # Folder + downstream: staging.*+

# GRAPH OPERATORS
<model>+               # Model + downstream (children)
+<model>               # Upstream + model (parents)
+<model>+              # Full lineage (parents + model + children)
@<model>               # Model only (exclude up/downstream)

# TAG SELECTORS
tag:<tag_name>         # Models with tag: tag:daily
tag:<tag1>,<tag2>      # Multiple tags (OR): tag:daily,hourly

# PATH SELECTORS
path:<path>            # File path: path:models/staging/ecommerce/*

# CONFIG SELECTORS
config.<key>:<value>   # Config filter: config.materialized:incremental

# RESOURCE TYPE
resource_type:<type>   # Filter: resource_type:snapshot

# SOURCE SELECTORS
source:<name>          # Source: source:ecommerce
source:<name>.<table>  # Specific table: source:ecommerce.orders
source:*+              # All sources + downstream models

# STATE SELECTORS (requires --state)
state:modified         # Modified models
state:modified+        # Modified + downstream
state:new              # New models not in comparison state

# INTERSECTION (&)
<selector1> <selector2>  # Intersection (AND): tag:daily staging.*

# UNION (space)
<selector1>,<selector2>  # Union (OR): customers,orders

# EXCLUSION
--exclude <selector>   # Exclude: --exclude tag:deprecated
```

---

**ADVANCED EXAMPLES**

```bash
# Run all staging + downstream, exclude deprecated
dbt run --select staging.*+ --exclude tag:deprecated

# Run incremental models tagged 'hourly'
dbt run --select config.materialized:incremental tag:hourly

# Full lineage of specific model
dbt run --select +fct_orders+

# Test sources and immediate downstream
dbt test --select source:ecommerce+1

# Slim CI: modified models + critical tests
dbt build --select state:modified+ tag:critical --state ./prod/

# List all snapshots
dbt ls --resource-type snapshot --output name

# Run models in subfolder path
dbt run --select path:models/marts/finance/**

# Intersection: daily marts only
dbt run --select tag:daily marts.*
```

---

## Summary

**Deployment Options:**
- **dbt Core CLI:** Self-managed, integrate with GitHub Actions/Airflow/Dagster
- **dbt Cloud:** Managed scheduler, IDE, docs hosting, lineage DAG
- **Hybrid:** dbt Cloud for scheduling, CLI for orchestration

**State-Based Selection:**
- `state:modified+` enables **slim CI** (90%+ faster builds)
- Requires artifacts from production runs stored in S3/GCS
- Use `--defer` to reference prod tables for unbuilt upstream models

**Artifacts:**
- **manifest.json:** Project state, enables state comparison
- **run_results.json:** Execution logs, SLA monitoring
- **catalog.json:** Column metadata, data catalog integration
- **sources.json:** Freshness test results

**SDLC Best Practice:**
- PR → GitHub Actions slim CI (`state:modified+`)
- Merge → dbt Cloud production job trigger
- Store artifacts in S3/GCS for state comparison

---

## Quick Reference Commands

```bash
# Install dependencies
dbt deps

# Run models
dbt run --select tag:daily --target prod --threads 16

# Slim CI (state-based)
dbt run --select state:modified+ --state ./prod/ --defer

# Full refresh incremental
dbt run --full-refresh --select fct_orders

# Atomic build (run + test)
dbt build --select +fct_orders

# Test with stored failures
dbt test --select tag:critical --store-failures

# Source freshness
dbt source freshness --select source:ecommerce

# Generate docs
dbt docs generate && dbt docs serve --port 8080

# List models (dry-run)
dbt ls --select state:modified+ --state ./prod/ --output name

# Compile without running
dbt compile --select fct_revenue

# Clean artifacts
dbt clean
```

---

## Orchestration Cheat Sheet

| Orchestrator | Pattern | Pros | Cons |
|--------------|---------|------|------|
| **dbt Cloud** | Managed jobs with cron scheduler | Zero infra, built-in IDE/docs | Paid, limited cross-tool orchestration |
| **GitHub Actions** | BashOperator in YAML workflow | Free for CI/CD, Git-native | No advanced DAG features |
| **Airflow** | BashOperator with dbt CLI | Complex multi-tool DAGs, monitoring | Heavy infra, steep learning curve |
| **Dagster** | dbt assets from manifest.json | Software-defined assets, type-safe | Newer ecosystem, less mature |
| **Meltano** | dbt runner plugin | All-in-one ELT + dbt | Less flexible than Airflow |
