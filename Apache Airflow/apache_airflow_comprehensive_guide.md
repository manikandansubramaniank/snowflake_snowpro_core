# Apache Airflow: Comprehensive Technical Guide
## From Beginner to Advanced

**Version:** 1.0  
**Author:** Apache Airflow Architecture Team  
**Last Updated:** February 2026  
**Target Audience:** Data Engineers, DevOps Engineers, Software Developers  
**Reference:** All content grounded in official Apache Airflow documentation (https://airflow.apache.org/docs/)

---

## Table of Contents

1. [Introduction to Apache Airflow](#1-introduction-to-apache-airflow)
2. [Airflow Architecture](#2-airflow-architecture)
3. [Airflow Objects](#3-airflow-objects)
4. [Data Ingestion and ETL](#4-data-ingestion-and-etl)
5. [Scheduling and Performance](#5-scheduling-and-performance)
6. [Security and Governance](#6-security-and-governance)
7. [Advanced Features](#7-advanced-features)
8. [Integration and Ecosystem](#8-integration-and-ecosystem)
9. [Best Practices](#9-best-practices)
10. [Real-World Examples](#10-real-world-examples)
11. [Glossary](#11-glossary)

---

## 1. Introduction to Apache Airflow

### 1.1 What is Apache Airflow?

**Apache Airflow** is an open-source platform for authoring, scheduling, and monitoring workflows programmatically. Per official docs, it defines workflows as **DAGs (Directed Acyclic Graphs)** using Python code, enabling version control, testing, and collaboration. Unlike traditional cron jobs, Airflow provides observability, dependency management, and scalability for complex data pipelines.

**Key Features:**
- **Programmatic workflow definition:** DAGs as Python code enable version control and dynamic generation
- **Extensible architecture:** 200+ pre-built operators for databases, cloud platforms, and APIs
- **Rich UI:** Web interface for monitoring, troubleshooting, and manual interventions
- **Scalability:** Supports multiple executors (Local, Celery, Kubernetes) for horizontal scaling
- **Observability:** Built-in logging, metrics, SLA monitoring, and alerting

**Diagram: Airflow Workflow Hierarchy**
```
DAG (Workflow)
  ├── Task 1 (PythonOperator)
  ├── Task 2 (BashOperator)
  └── Task 3 (SnowflakeOperator)
       └── Dependencies: Task 1 >> Task 2 >> Task 3
```

### 1.2 Cloud-Native vs Cron

**Cron Limitations:**
- No dependency management between jobs
- Limited observability and error handling
- Manual retry logic required
- No centralized monitoring or alerting

**Airflow Advantages:**
- Task dependencies with automatic scheduling
- Retry/backoff strategies built-in
- Centralized UI for all pipelines
- Distributed execution across workers

### 1.3 SDLC Use Case: CI/CD Onboarding for Fintech

**Scenario:** A fintech startup needs to onboard 50+ daily data pipelines for regulatory reporting.

**Implementation:**
- Store DAG files in GitHub with branch protection
- Use GitHub Actions to validate DAG syntax on pull requests
- Deploy DAGs to Airflow via CI/CD pipeline
- Monitor pipeline SLAs for compliance deadlines

**Benefits:** Version-controlled pipelines, automated testing, reduced manual errors, audit trail for compliance.

### 1.4 DAG Constructor: Complete Syntax

```python
# SYNTAX TEMPLATE
from airflow import DAG
from datetime import datetime, timedelta

dag = DAG(
    dag_id="<string>",                          # Required: Unique identifier
    description="<string>",                     # Optional: Human-readable description
    schedule_interval="<cron|timedelta|None>",  # Optional: When to run (default: None)
    start_date=<datetime>,                      # Required: When DAG becomes active
    end_date=<datetime>,                        # Optional: When DAG stops running
    catchup=<bool>,                             # Optional: Backfill missed runs (default: True)
    default_args=<dict>,                        # Optional: Default params for all tasks
    max_active_runs=<int>,                      # Optional: Concurrent DAG runs (default: 16)
    max_active_tasks=<int>,                     # Optional: Concurrent tasks (default: 16)
    dagrun_timeout=<timedelta>,                 # Optional: Max DAG run duration
    sla_miss_callback=<callable>,               # Optional: Function for SLA misses
    default_view="<tree|graph|duration|gantt>", # Optional: Default UI view
    orientation="<LR|TB|RL|BT>",                # Optional: Graph direction (default: LR)
    concurrency=<int>,                          # Deprecated: Use max_active_tasks
    tags=<list>,                                # Optional: Tags for grouping (default: [])
    doc_md="<markdown>",                        # Optional: Markdown documentation
    is_paused_upon_creation=<bool>,             # Optional: Start paused (default: None)
    render_template_as_native_obj=<bool>,       # Optional: Native types for templates
    access_control=<dict>,                      # Optional: RBAC permissions
    params=<dict>,                              # Optional: User-defined parameters
    owner_links=<dict>,                         # Optional: Links for owners
)
```

### 1.5 DAG Parameters Reference

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `dag_id` | String | None (Required) | Unique identifier for the DAG | `"etl_pipeline_v1"` |
| `description` | String | None | Brief description shown in UI | `"Daily ETL for sales data"` |
| `schedule_interval` | Cron/Timedelta/None | None | Execution frequency; None = manual only | `"@daily"` or `"0 2 * * *"` |
| `start_date` | datetime | None (Required) | Date when DAG becomes schedulable | `datetime(2024,1,1)` |
| `end_date` | datetime | None | Date when DAG stops running | `datetime(2025,12,31)` |
| `catchup` | Boolean | True | Whether to backfill runs between start_date and now | `False` |
| `default_args` | Dict | {} | Default arguments inherited by all tasks | `{"retries": 3}` |
| `max_active_runs` | Integer | 16 | Maximum concurrent DAG runs | `5` |
| `max_active_tasks` | Integer | 16 | Maximum concurrent tasks per DAG run | `10` |
| `dagrun_timeout` | timedelta | None | Maximum duration for a DAG run before timeout | `timedelta(hours=4)` |
| `sla_miss_callback` | Callable | None | Function called when task misses SLA | `notify_team` |
| `default_view` | String | "tree" | Default view in Airflow UI | `"graph"` |
| `orientation` | String | "LR" | Graph direction: LR/TB/RL/BT | `"TB"` |
| `tags` | List | [] | Tags for filtering in UI | `["prod", "etl"]` |
| `doc_md` | String | None | Markdown documentation | `"# Pipeline Docs"` |
| `is_paused_upon_creation` | Boolean | None | Whether to start DAG paused | `True` |
| `render_template_as_native_obj` | Boolean | False | Render templates as native Python types | `True` |
| `access_control` | Dict | {} | Role-based permissions | `{"Admin": {"can_read"}}` |
| `params` | Dict | {} | User-defined runtime parameters | `{"env": "prod"}` |
| `owner_links` | Dict | {} | Links associated with owners | `{"John": "slack://..."}` |

### 1.6 Basic DAG Example

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

# Minimal DAG for daily backup
with DAG(
    dag_id="daily_backup",
    schedule_interval="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["backup"]
) as dag:
    backup_task = BashOperator(
        task_id="run_backup",
        bash_command="tar -czf /backup/data_$(date +%Y%m%d).tar.gz /data"
    )
```

### 1.7 Advanced DAG Example

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def send_sla_alert(context):
    """Custom SLA callback for critical pipelines"""
    print(f"SLA missed for {context['task_instance'].task_id}")

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email': ['alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(hours=1),
    'execution_timeout': timedelta(hours=2),
}

with DAG(
    dag_id="prod_etl_pipeline",
    description="Production ETL with SLA monitoring and auto-retry",
    schedule_interval="0 2 * * *",  # 2 AM daily
    start_date=datetime(2024, 1, 1),
    end_date=datetime(2025, 12, 31),
    catchup=True,  # Backfill historical data
    max_active_runs=3,
    max_active_tasks=10,
    dagrun_timeout=timedelta(hours=4),
    sla_miss_callback=send_sla_alert,
    default_args=default_args,
    tags=["prod", "etl", "critical"],
    doc_md="""
    # Production ETL Pipeline
    Ingests sales data from Oracle, transforms with Spark, loads to Snowflake.
    **SLA:** Must complete within 4 hours for daily reports.
    """,
    is_paused_upon_creation=False,
    render_template_as_native_obj=True,
    params={'env': 'production', 'batch_size': 10000}
) as dag:
    # Tasks defined here
    pass
```

---

## 2. Airflow Architecture

### 2.1 Core Components

Per official docs, Airflow architecture consists of four primary components that work together to orchestrate workflows.

**Diagram: Airflow Architecture**
```
┌─────────────────────────────────────────────────────────┐
│                      Web Server                         │
│              (Flask UI on port 8080)                    │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│                   Scheduler                             │
│      (Monitors DAGs, triggers task execution)           │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│                  Executor                               │
│   (LocalExecutor, CeleryExecutor, KubernetesExecutor)   │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│              Worker Nodes                               │
│          (Execute individual tasks)                     │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│              Metadata Database                          │
│       (PostgreSQL/MySQL - stores state)                 │
└─────────────────────────────────────────────────────────┘
```

#### 2.1.1 Scheduler

The **Scheduler** monitors DAG files, parses them, and triggers task execution based on schedules and dependencies.

**Key Functions:**
- Parse DAG definitions from Python files
- Create DAG runs based on schedule_interval
- Monitor task dependencies and trigger execution
- Handle retries and task state updates

#### 2.1.2 Executor

The **Executor** determines *how* tasks run.

| Executor | Use Case | Concurrency | Setup Complexity |
|----------|----------|-------------|------------------|
| SequentialExecutor | Development only | 1 task at a time | Minimal |
| LocalExecutor | Single-machine | Multi-threaded | Low |
| CeleryExecutor | Distributed | High (horizontal scaling) | Medium |
| KubernetesExecutor | Cloud-native | Dynamic pods | High |

#### 2.1.3 Web Server

The **Web Server** provides the Flask-based UI for monitoring, triggering, and debugging workflows.

#### 2.1.4 Metadata Database

The **Metadata Database** (PostgreSQL or MySQL) stores all Airflow state: DAG definitions, task instances, execution history, connections, variables, and user accounts.

### 2.2 SDLC Use Case: Scaling in Sprint Testing

**Scenario:** QA team needs to test 20 parallel data pipelines during 2-week sprints without impacting production.

**Implementation:**
- Deploy test Airflow environment with CeleryExecutor (5 workers)
- Use separate metadata database for test isolation
- Configure max_active_tasks=20 for parallel execution
- Auto-scale Celery workers based on queue depth

**Benefits:** Isolated testing environment, parallel execution reduces test time from 4 hours to 30 minutes.

---

## 3. Airflow Objects

### 3.1 PythonOperator: Complete Syntax

```python
# SYNTAX TEMPLATE
from airflow.operators.python import PythonOperator

task = PythonOperator(
    task_id="<string>",                  # Required: Task identifier
    python_callable=<callable>,          # Required: Python function to execute
    op_args=<list>,                      # Optional: Positional arguments (default: [])
    op_kwargs=<dict>,                    # Optional: Keyword arguments (default: {})
    templates_dict=<dict>,               # Optional: Templates for Jinja rendering
    templates_exts=<list>,               # Optional: File extensions to template
    show_return_value_in_logs=<bool>,    # Optional: Log return value (default: True)
    # BaseOperator parameters
    owner="<string>",                    # Optional: Task owner (default: "airflow")
    retries=<int>,                       # Optional: Number of retries (default: 0)
    retry_delay=<timedelta>,             # Optional: Delay between retries
    execution_timeout=<timedelta>,       # Optional: Task timeout
    pool="<string>",                     # Optional: Resource pool (default: "default_pool")
    pool_slots=<int>,                    # Optional: Pool slots required (default: 1)
)
```

### 3.2 PythonOperator Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `python_callable` | Callable | None (Required) | Python function to execute | `my_function` |
| `op_args` | List | [] | Positional arguments for callable | `[1, 2, 3]` |
| `op_kwargs` | Dict | {} | Keyword arguments for callable | `{"key": "value"}` |
| `templates_dict` | Dict | None | Dictionary passed to callable with rendered templates | `{"sql": "SELECT * FROM {{ params.table }}"}` |
| `show_return_value_in_logs` | Boolean | True | Whether to log return value | `False` |

### 3.3 Python Operator Examples

**Basic Example:**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def greet(name):
    print(f"Hello, {name}!")
    return f"Greeting sent to {name}"

with DAG(
    dag_id="python_basic",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False
) as dag:
    
    greet_task = PythonOperator(
        task_id="greet_user",
        python_callable=greet,
        op_kwargs={"name": "Data Engineer"}
    )
```

**Advanced Example:**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import pandas as pd

def extract_data(**context):
    exec_date = context['execution_date']
    print(f"Extracting data for {exec_date}")
    data = {'id': [1, 2, 3], 'value': [100, 200, 300]}
    df = pd.DataFrame(data)
    return df.to_json()

def transform_data(ti, **context):
    data_json = ti.xcom_pull(task_ids='extract')
    df = pd.read_json(data_json)
    df['value'] = df['value'] * 2
    return df.to_json()

with DAG(
    dag_id="python_advanced",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False,
    default_args={'retries': 2, 'retry_delay': timedelta(minutes=5)}
) as dag:
    
    extract = PythonOperator(
        task_id="extract",
        python_callable=extract_data,
        provide_context=True,
        execution_timeout=timedelta(minutes=30)
    )
    
    transform = PythonOperator(
        task_id="transform",
        python_callable=transform_data,
        provide_context=True
    )
    
    extract >> transform
```

### 3.4 BashOperator: Complete Syntax

```python
# SYNTAX TEMPLATE
from airflow.operators.bash import BashOperator

task = BashOperator(
    task_id="<string>",                  # Required: Task identifier
    bash_command="<string>",             # Required: Bash command/script
    env=<dict>,                          # Optional: Environment variables
    append_env=<bool>,                   # Optional: Append to existing env (default: False)
    output_encoding="<string>",          # Optional: Output encoding (default: "utf-8")
    skip_exit_code=<int>,                # Optional: Exit code to skip task (default: 99)
    cwd="<string>",                      # Optional: Working directory
)
```

### 3.5 BashOperator Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `bash_command` | String | None (Required) | Bash command or script to execute | `"echo 'Hello World'"` |
| `env` | Dict | None | Environment variables for command | `{"PATH": "/usr/bin"}` |
| `append_env` | Boolean | False | Append env vars to existing environment | `True` |
| `skip_exit_code` | Integer | 99 | Exit code that marks task as skipped | `100` |
| `cwd` | String | None | Working directory for command execution | `"/tmp"` |

### 3.6 BashOperator Examples

**Basic Example:**
```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG(
    dag_id="bash_basic",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False
) as dag:
    
    hello_task = BashOperator(
        task_id="print_date",
        bash_command="date"
    )
```

**Advanced Example:**
```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

with DAG(
    dag_id="bash_advanced",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 2 * * *",
    catchup=False,
    default_args={'retries': 3}
) as dag:
    
    backup_script = BashOperator(
        task_id="backup_database",
        bash_command="""
        set -e
        echo "Starting backup for {{ ds }}"
        BACKUP_DIR=/backups/{{ ds }}
        mkdir -p $BACKUP_DIR
        pg_dump -h db.example.com -U user mydb > $BACKUP_DIR/dump.sql
        tar -czf $BACKUP_DIR/dump.tar.gz $BACKUP_DIR/dump.sql
        rm $BACKUP_DIR/dump.sql
        echo "Backup completed"
        """,
        env={'PGPASSWORD': '{{ var.value.db_password }}'},
        cwd='/tmp',
        execution_timeout=timedelta(hours=1)
    )
```

### 3.7 S3KeySensor: Complete Syntax

```python
# SYNTAX TEMPLATE
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

task = S3KeySensor(
    task_id="<string>",                  # Required: Task identifier
    bucket_key="<string>",               # Required: S3 key to check
    bucket_name="<string>",              # Required: S3 bucket name
    wildcard_match=<bool>,               # Optional: Use wildcards in key (default: False)
    aws_conn_id="<string>",              # Optional: Airflow connection (default: "aws_default")
    poke_interval=<int>,                 # Optional: Seconds between pokes (default: 60)
    timeout=<int>,                       # Optional: Sensor timeout seconds (default: 604800)
    mode="<poke|reschedule>",            # Optional: Sensor mode (default: "poke")
)
```

### 3.8 S3KeySensor Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `bucket_key` | String | None (Required) | S3 object key to check | `"data/file.csv"` |
| `bucket_name` | String | None (Required) | S3 bucket name | `"my-data-bucket"` |
| `wildcard_match` | Boolean | False | Enable wildcard patterns in key | `True` |
| `aws_conn_id` | String | "aws_default" | Airflow AWS connection ID | `"aws_prod"` |
| `poke_interval` | Integer | 60 | Seconds between checks | `300` |
| `timeout` | Integer | 604800 (7 days) | Sensor timeout in seconds | `3600` |
| `mode` | String | "poke" | Sensor mode: poke or reschedule | `"reschedule"` |

### 3.9 S3KeySensor Examples

**Basic Example:**
```python
from airflow import DAG
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from datetime import datetime

with DAG(
    dag_id="s3_sensor_basic",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@hourly",
    catchup=False
) as dag:
    
    wait_for_file = S3KeySensor(
        task_id="wait_for_data_file",
        bucket_name="my-data-bucket",
        bucket_key="incoming/data.csv",
        poke_interval=60,
        timeout=3600
    )
```

**Advanced Example:**
```python
from airflow import DAG
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from datetime import datetime

with DAG(
    dag_id="s3_sensor_advanced",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 */6 * * *",
    catchup=False
) as dag:
    
    wait_for_daily_file = S3KeySensor(
        task_id="wait_for_daily_file",
        bucket_name="data-lake",
        bucket_key="raw/sales/{{ ds }}/*.parquet",
        wildcard_match=True,
        aws_conn_id="aws_production",
        poke_interval=300,
        timeout=21600,
        mode="reschedule"
    )
```

### 3.10 Connections

**Connections** store credentials for external systems.

**CLI Connection Management:**
```bash
# Add connection
airflow connections add 'snowflake_prod' \
    --conn-type 'snowflake' \
    --conn-host 'account.snowflakecomputing.com' \
    --conn-login 'etl_user' \
    --conn-password 'secure_pass' \
    --conn-schema 'ANALYTICS'

# List connections
airflow connections list

# Delete connection
airflow connections delete snowflake_prod
```

### 3.11 Variables

**Variables** store key-value configuration accessible across DAGs.

```python
from airflow.models import Variable

# Set variable
Variable.set("api_endpoint", "https://api.example.com")

# Get variable
value = Variable.get("api_endpoint")

# JSON variables
Variable.set("config", {"batch_size": 1000}, serialize_json=True)
config = Variable.get("config", deserialize_json=True)
```

### 3.12 SDLC Use Case: GitHub Actions for DAG Versioning

**GitHub Actions Workflow:**
```yaml
name: Airflow DAG CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install Airflow
        run: |
          pip install apache-airflow==2.8.0
          pip install -r requirements.txt
      
      - name: Validate DAG syntax
        run: |
          python -m py_compile dags/*.py
          airflow dags list
      
      - name: Run DAG integrity tests
        run: pytest tests/

  deploy:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Airflow
        run: |
          rsync -avz dags/ airflow@prod-server:/opt/airflow/dags/
```

---

## 4. Data Ingestion and ETL

### 4.1 TaskFlow API with @task Decorator

```python
# SYNTAX TEMPLATE
from airflow.decorators import task

@task(
    task_id="<string>",                        # Optional: Override function name
    multiple_outputs=<bool>,                   # Optional: Return dict as separate XComs
    pool="<string>",                           # Optional: Resource pool
    retries=<int>,                             # Optional: Number of retries
    retry_delay=<timedelta>,                   # Optional: Delay between retries
)
def my_function(param1, param2):
    return result
```

### 4.2 @task Decorator Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `task_id` | String | Function name | Override task ID | `"custom_task_id"` |
| `multiple_outputs` | Boolean | False | Return dict as multiple XCom keys | `True` |
| `pool` | String | "default_pool" | Resource pool name | `"etl_pool"` |
| `retries` | Integer | 0 | Retry attempts | `3` |

### 4.3 TaskFlow API Examples

**Basic Example:**
```python
from airflow.decorators import dag, task
from datetime import datetime

@dag(
    dag_id="taskflow_basic",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False
)
def simple_etl():
    
    @task
    def extract():
        return [1, 2, 3, 4, 5]
    
    @task
    def transform(data):
        return [x * 2 for x in data]
    
    @task
    def load(data):
        print(f"Loading {len(data)} records: {data}")
    
    data = extract()
    transformed = transform(data)
    load(transformed)

simple_etl_dag = simple_etl()
```

**Advanced Example:**
```python
from airflow.decorators import dag, task
from datetime import datetime, timedelta
import pandas as pd

@dag(
    dag_id="taskflow_advanced_etl",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 2 * * *",
    catchup=False,
    default_args={'retries': 3, 'retry_delay': timedelta(minutes=5)},
    tags=["etl", "production"]
)
def advanced_etl_pipeline():
    
    @task(pool="extraction_pool", pool_slots=2)
    def extract_from_oracle(**context):
        exec_date = context['ds']
        print(f"Extracting data for {exec_date}")
        data = {'id': range(1, 1001), 'value': range(100, 100100, 100)}
        df = pd.DataFrame(data)
        return df.to_json()
    
    @task(multiple_outputs=True)
    def validate_data(data_json):
        df = pd.read_json(data_json)
        return {
            'record_count': len(df),
            'is_valid': len(df) > 0,
            'min_value': int(df['value'].min())
        }
    
    @task
    def transform_data(data_json, validation_results):
        if not validation_results['is_valid']:
            raise ValueError("Data validation failed")
        df = pd.read_json(data_json)
        df['value_scaled'] = df['value'] / 100
        return df.to_json()
    
    raw_data = extract_from_oracle()
    validation = validate_data(raw_data)
    transformed = transform_data(raw_data, validation)

advanced_etl_dag = advanced_etl_pipeline()
```

### 4.4 SnowflakeOperator: Complete Syntax

```python
# SYNTAX TEMPLATE
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator

task = SnowflakeOperator(
    task_id="<string>",                    # Required: Task identifier
    sql="<string|list>",                   # Required: SQL query or list of queries
    snowflake_conn_id="<string>",          # Optional: Connection ID (default: "snowflake_default")
    warehouse="<string>",                  # Optional: Snowflake warehouse
    database="<string>",                   # Optional: Database name
    schema="<string>",                     # Optional: Schema name
    role="<string>",                       # Optional: Snowflake role
    autocommit=<bool>,                     # Optional: Autocommit mode (default: True)
    parameters=<dict|list>,                # Optional: Query parameters
)
```

### 4.5 SnowflakeOperator Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `sql` | String/List | None (Required) | SQL query or list of queries | `"SELECT * FROM table"` |
| `snowflake_conn_id` | String | "snowflake_default" | Airflow connection ID | `"snowflake_prod"` |
| `warehouse` | String | None | Snowflake warehouse name | `"ETL_WH"` |
| `database` | String | None | Database name | `"ANALYTICS"` |
| `schema` | String | None | Schema name | `"PUBLIC"` |
| `role` | String | None | Snowflake role | `"ETL_ROLE"` |
| `autocommit` | Boolean | True | Enable autocommit | `False` |
| `parameters` | Dict/List | None | Query bind parameters | `{"id": 123}` |

### 4.6 SnowflakeOperator Examples

**Basic Example:**
```python
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime

with DAG(
    dag_id="snowflake_basic",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False
) as dag:
    
    create_table = SnowflakeOperator(
        task_id="create_staging_table",
        sql="""
        CREATE TABLE IF NOT EXISTS staging.daily_sales (
            sale_id INT,
            amount DECIMAL(10,2),
            sale_date DATE
        )
        """
    )
```

**Advanced Example:**
```python
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime, timedelta

with DAG(
    dag_id="snowflake_advanced_etl",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 3 * * *",
    catchup=False
) as dag:
    
    truncate_staging = SnowflakeOperator(
        task_id="truncate_staging",
        snowflake_conn_id="snowflake_prod",
        warehouse="ETL_WH",
        database="ANALYTICS",
        schema="STAGING",
        sql="TRUNCATE TABLE IF EXISTS staging.daily_sales"
    )
    
    load_data = SnowflakeOperator(
        task_id="load_daily_sales",
        snowflake_conn_id="snowflake_prod",
        warehouse="ETL_WH",
        database="ANALYTICS",
        schema="STAGING",
        sql="""
        COPY INTO staging.daily_sales
        FROM @s3_stage/sales/{{ ds }}/
        FILE_FORMAT = (TYPE = 'PARQUET')
        """
    )
    
    merge_to_prod = SnowflakeOperator(
        task_id="merge_to_production",
        sql="""
        MERGE INTO public.sales AS target
        USING staging.daily_sales AS source
        ON target.sale_id = source.sale_id
        WHEN MATCHED THEN UPDATE SET amount = source.amount
        WHEN NOT MATCHED THEN INSERT VALUES (source.sale_id, source.amount, source.sale_date)
        """
    )
    
    truncate_staging >> load_data >> merge_to_prod
```

### 4.7 SDLC Use Case: Nightly ETL in Sprints

**Scenario:** E-commerce company runs nightly ETL to sync sales data from Oracle to Snowflake.

**Implementation:**
```python
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime

with DAG(
    dag_id="nightly_sales_etl",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 1 * * *",
    catchup=False,
    tags=["production", "etl"]
) as dag:
    
    extract_sales = SnowflakeOperator(
        task_id="extract_sales",
        sql="CALL extract_oracle_sales('{{ ds }}')"
    )
    
    transform_data = SnowflakeOperator(
        task_id="transform_sales",
        sql="CALL transform_sales_data('{{ ds }}')"
    )
    
    load_to_warehouse = SnowflakeOperator(
        task_id="load_to_warehouse",
        sql="CALL load_sales_to_warehouse('{{ ds }}')"
    )
    
    extract_sales >> transform_data >> load_to_warehouse
```

---

## 5. Scheduling and Performance

### 5.1 Cron Expressions

**Cron Format:** `minute hour day_of_month month day_of_week`

**Common Presets:**
- `@once` - Run once, then never again
- `@hourly` - Every hour (0 * * * *)
- `@daily` - Daily at midnight (0 0 * * *)
- `@weekly` - Weekly on Sunday (0 0 * * 0)
- `@monthly` - First of month (0 0 1 * *)
- `None` - Manual trigger only

**Custom Cron Examples:**

| Expression | Description |
|------------|-------------|
| `0 2 * * *` | 2 AM daily |
| `0 */6 * * *` | Every 6 hours |
| `30 14 * * 1-5` | 2:30 PM on weekdays |
| `0 0 1 */3 *` | Quarterly |
| `0 9-17 * * 1-5` | Hourly during business hours |

### 5.2 Pools

**Pools** limit concurrent tasks for specific resources.

**Creating Pools:**
```bash
# Via CLI
airflow pools set database_pool 5 "PostgreSQL connection pool"
airflow pools set api_pool 3 "External API rate limit"

# View pools
airflow pools list
```

**Using Pools in Tasks:**
```python
task = PythonOperator(
    task_id="query_database",
    python_callable=my_function,
    pool="database_pool",
    pool_slots=2
)
```

### 5.3 SLAs

**SLAs** define expected task completion times.

```python
from datetime import timedelta

def sla_miss_callback(dag, task_list, blocking_task_list, slas, blocking_tis):
    print(f"SLA missed for tasks: {task_list}")

with DAG(
    dag_id="sla_monitored_dag",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    sla_miss_callback=sla_miss_callback
) as dag:
    
    critical_task = PythonOperator(
        task_id="critical_processing",
        python_callable=process_data,
        sla=timedelta(hours=2)
    )
```

### 5.4 Performance Example

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def process_partition(partition_id):
    print(f"Processing partition {partition_id}")

with DAG(
    dag_id="high_performance_etl",
    schedule_interval="0 2 * * *",
    start_date=datetime(2024, 1, 1),
    max_active_runs=2,
    max_active_tasks=20,
    dagrun_timeout=timedelta(hours=4),
    default_args={'retries': 3, 'execution_timeout': timedelta(minutes=30)}
) as dag:
    
    for i in range(10):
        task = PythonOperator(
            task_id=f"process_partition_{i}",
            python_callable=process_partition,
            op_kwargs={'partition_id': i},
            pool="etl_pool",
            priority_weight=10 - i
        )
```

---

## 6. Security and Governance

### 6.1 Role-Based Access Control (RBAC)

Per official docs, RBAC controls user access to DAGs, connections, variables, and admin functions.

**Predefined Roles:**
- **Admin:** Full access to all features
- **User:** Can view/edit DAGs, trigger runs
- **Viewer:** Read-only access to DAGs and logs
- **Op:** Operational tasks (clear tasks, mark success/failed)

### 6.2 Secrets Backends

**Secrets backends** integrate with external secret managers to avoid storing credentials in Airflow metadata DB.

**Configuration (airflow.cfg):**
```ini
[secrets]
backend = airflow.providers.amazon.aws.secrets.secrets_manager.SecretsManagerBackend
backend_kwargs = {"connections_prefix": "airflow/connections"}
```

**AWS Secrets Manager Example:**
```json
{
  "conn_type": "postgres",
  "host": "db.example.com",
  "schema": "analytics",
  "login": "etl_user",
  "password": "secure_password",
  "port": 5432
}
```

### 6.3 Variable Masking

**Variables** containing sensitive data should be masked in logs.

```python
from airflow.models import Variable

# Set variable (auto-masked if name contains "password", "secret", etc.)
Variable.set("api_password", "secret_value_123")

# Retrieve variable
api_pass = Variable.get("api_password")
```

### 6.4 SDLC Use Case: DevSecOps Access Control

**Scenario:** Fintech company requires strict separation of dev/QA/prod environments.

**Implementation:**
- **Dev:** All engineers have Admin role
- **QA:** QA team has User role
- **Prod:** Only SRE team has Admin role; data engineers have Op role

**Secrets Management:**
```bash
aws secretsmanager create-secret \
    --name airflow/connections/snowflake_prod \
    --secret-string '{
        "conn_type": "snowflake",
        "host": "prod.snowflakecomputing.com",
        "login": "prod_user",
        "password": "secure_pass"
    }'
```

---

## 7. Advanced Features

### 7.1 Task Groups

**Task Groups** organize related tasks into collapsible groups in the UI.

```python
# SYNTAX
from airflow.utils.task_group import TaskGroup

with TaskGroup(
    group_id="<string>",                # Required: Group identifier
    tooltip="<string>",                 # Optional: Hover text in UI
    prefix_group_id=<bool>,             # Optional: Prefix task IDs (default: True)
) as group:
    # Define tasks within group
    pass
```

**TaskGroup Parameters:**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `group_id` | String | None (Required) | Unique group identifier | `"extract_group"` |
| `tooltip` | String | None | Tooltip shown in UI | `"Data extraction tasks"` |
| `prefix_group_id` | Boolean | True | Prefix group_id to task IDs | `True` |

**Basic Example:**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime

with DAG(
    dag_id="taskgroup_basic",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False
) as dag:
    
    start = PythonOperator(
        task_id="start",
        python_callable=lambda: print("Starting")
    )
    
    with TaskGroup("processing_group") as processing:
        task1 = PythonOperator(
            task_id="task1",
            python_callable=lambda: print("Task 1")
        )
        task2 = PythonOperator(
            task_id="task2",
            python_callable=lambda: print("Task 2")
        )
        task1 >> task2
    
    end = PythonOperator(
        task_id="end",
        python_callable=lambda: print("Complete")
    )
    
    start >> processing >> end
```

### 7.2 Dynamic DAGs

**Dynamic DAGs** generate tasks programmatically based on runtime data.

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

DATABASES = ['db1', 'db2', 'db3', 'db4']

def process_database(db_name):
    print(f"Processing {db_name}")

with DAG(
    dag_id="dynamic_dag",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False
) as dag:
    
    for db in DATABASES:
        task = PythonOperator(
            task_id=f"process_{db}",
            python_callable=process_database,
            op_kwargs={'db_name': db}
        )
```

### 7.3 SDLC Use Case: Cloning DAGs for Test Environments

```python
import os

ENV = os.getenv('AIRFLOW_ENV', 'dev')
CONFIG = {
    'dev': {'schedule': None, 'retries': 0},
    'qa': {'schedule': '@daily', 'retries': 2},
    'prod': {'schedule': '@hourly', 'retries': 5}
}

with DAG(
    dag_id=f"{ENV}_etl_pipeline",
    start_date=datetime(2024, 1, 1),
    schedule_interval=CONFIG[ENV]['schedule'],
    default_args={'retries': CONFIG[ENV]['retries']},
    catchup=False
) as dag:
    # Tasks here
    pass
```

---

## 8. Integration and Ecosystem

### 8.1 BigQueryOperator: Complete Syntax

```python
# SYNTAX TEMPLATE
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator

task = BigQueryInsertJobOperator(
    task_id="<string>",                    # Required: Task identifier
    configuration=<dict>,                  # Required: BigQuery job configuration
    project_id="<string>",                 # Optional: GCP project ID
    location="<string>",                   # Optional: Dataset location (default: US)
    gcp_conn_id="<string>",                # Optional: Connection ID (default: "google_cloud_default")
    impersonation_chain=<list>,            # Optional: Service account impersonation
)
```

### 8.2 BigQueryOperator Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `configuration` | Dict | None (Required) | BigQuery job configuration | `{"query": {...}}` |
| `project_id` | String | None | GCP project ID | `"my-project"` |
| `location` | String | "US" | Dataset location | `"US"` |
| `gcp_conn_id` | String | "google_cloud_default" | Airflow GCP connection | `"gcp_prod"` |

### 8.3 BigQueryOperator Example

```python
from airflow import DAG
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from datetime import datetime

with DAG(
    dag_id="bigquery_etl",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False
) as dag:
    
    run_query = BigQueryInsertJobOperator(
        task_id="run_bigquery_job",
        configuration={
            "query": {
                "query": """
                    SELECT customer_id, SUM(amount) as total
                    FROM `project.dataset.sales`
                    WHERE date = '{{ ds }}'
                    GROUP BY customer_id
                """,
                "useLegacySql": False,
                "destinationTable": {
                    "projectId": "my-project",
                    "datasetId": "analytics",
                    "tableId": "daily_summary"
                },
                "writeDisposition": "WRITE_TRUNCATE"
            }
        },
        project_id="my-project",
        location="US"
    )
```

### 8.4 SDLC Use Case: CI/CD Dashboards

**Scenario:** Integrate Airflow with CI/CD for real-time pipeline monitoring.

**Implementation:**
- Expose Airflow metrics to Prometheus
- Create Grafana dashboards for DAG success rates
- Set up alerts for SLA violations
- Integrate with Slack for notifications

---

## 9. Best Practices

### 9.1 DAG Design Principles

**Modularity:** Break complex workflows into reusable components using TaskGroups.
**Idempotency:** Ensure tasks can be safely retried without side effects.
**Atomicity:** Each task should represent a single, well-defined operation.

### 9.2 Performance Optimization

**Pool Management:** Limit concurrent tasks accessing shared resources.
**Task Parallelization:** Use dynamic DAGs to process multiple partitions simultaneously.
**Sensor Optimization:** Use `mode="reschedule"` for long-running sensors to free worker slots.

### 9.3 Observability

**Logging:** Configure centralized logging to S3/GCS for long-term retention.
**Metrics:** Export Airflow metrics to Prometheus for monitoring.
**Alerting:** Set up SLA callbacks and email/Slack notifications for failures.

### 9.4 Cost Optimization

**Right-size pools:** Match pool sizes to actual resource constraints.
**Optimize catchup:** Set `catchup=False` for non-historical pipelines.
**Use spot instances:** Run Celery workers on AWS Spot/GCP Preemptible instances.

### 9.5 SDLC Use Case: Production Observability

**Scenario:** Monitor 100+ production DAGs with 99.9% uptime SLA.

**Implementation:**
```python
# airflow.cfg
[metrics]
statsd_on = True
statsd_host = statsd-exporter
statsd_port = 8125

[logging]
remote_logging = True
remote_base_log_folder = s3://my-logs/airflow
```

**Grafana Dashboard Metrics:**
- DAG success rate (last 24 hours)
- Task duration percentiles (p50, p95, p99)
- Scheduler lag (time between expected and actual DAG run)
- Pool utilization

---

## 10. Real-World Examples

### 10.1 Oracle-to-Snowflake ETL

```python
from airflow import DAG
from airflow.providers.oracle.operators.oracle import OracleOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.providers.snowflake.transfers.oracle_to_snowflake import OracleToSnowflakeOperator
from datetime import datetime, timedelta

with DAG(
    dag_id="oracle_to_snowflake_etl",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 2 * * *",
    catchup=False,
    default_args={
        'owner': 'data-team',
        'retries': 3,
        'retry_delay': timedelta(minutes=5),
        'execution_timeout': timedelta(hours=2)
    },
    tags=["etl", "oracle", "snowflake"]
) as dag:
    
    extract_oracle = OracleOperator(
        task_id="extract_from_oracle",
        oracle_conn_id="oracle_prod",
        sql="""
        SELECT * FROM sales
        WHERE sale_date = TO_DATE('{{ ds }}', 'YYYY-MM-DD')
        """
    )
    
    stage_to_s3 = PythonOperator(
        task_id="stage_to_s3",
        python_callable=lambda: print("Staging to S3")
    )
    
    load_to_snowflake = SnowflakeOperator(
        task_id="load_to_snowflake",
        snowflake_conn_id="snowflake_prod",
        warehouse="ETL_WH",
        database="ANALYTICS",
        sql="""
        COPY INTO staging.sales
        FROM @s3_stage/sales/{{ ds }}/
        FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"')
        """
    )
    
    validate_load = SnowflakeOperator(
        task_id="validate_load",
        sql="""
        SELECT COUNT(*) FROM staging.sales
        WHERE sale_date = '{{ ds }}'
        HAVING COUNT(*) > 0
        """
    )
    
    extract_oracle >> stage_to_s3 >> load_to_snowflake >> validate_load
```

### 10.2 S3 Ingestion Pipeline

```python
from airflow import DAG
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.operators.python import PythonOperator
from datetime import datetime

def process_s3_file(**context):
    file_key = context['ti'].xcom_pull(task_ids='wait_for_file')
    print(f"Processing file: {file_key}")

with DAG(
    dag_id="s3_ingestion_pipeline",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@hourly",
    catchup=False,
    tags=["ingestion", "s3"]
) as dag:
    
    wait_for_file = S3KeySensor(
        task_id="wait_for_file",
        bucket_name="incoming-data",
        bucket_key="uploads/{{ ds }}/data_*.csv",
        wildcard_match=True,
        aws_conn_id="aws_prod",
        poke_interval=300,
        timeout=3600,
        mode="reschedule"
    )
    
    process_file = PythonOperator(
        task_id="process_file",
        python_callable=process_s3_file,
        provide_context=True
    )
    
    archive_file = S3CopyObjectOperator(
        task_id="archive_file",
        source_bucket_name="incoming-data",
        source_bucket_key="uploads/{{ ds }}/data_*.csv",
        dest_bucket_name="archived-data",
        dest_bucket_key="archive/{{ ds }}/data.csv",
        aws_conn_id="aws_prod"
    )
    
    wait_for_file >> process_file >> archive_file
```

### 10.3 Analytics Pipeline with TaskGroups

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime

with DAG(
    dag_id="analytics_pipeline",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 3 * * *",
    catchup=False,
    tags=["analytics"]
) as dag:
    
    start = PythonOperator(
        task_id="start",
        python_callable=lambda: print("Starting analytics pipeline")
    )
    
    with TaskGroup("extract_sources") as extract:
        extract_db1 = PythonOperator(
            task_id="extract_database1",
            python_callable=lambda: print("Extract DB1")
        )
        extract_db2 = PythonOperator(
            task_id="extract_database2",
            python_callable=lambda: print("Extract DB2")
        )
    
    with TaskGroup("transform") as transform:
        clean_data = PythonOperator(
            task_id="clean_data",
            python_callable=lambda: print("Cleaning data")
        )
        aggregate_data = PythonOperator(
            task_id="aggregate_data",
            python_callable=lambda: print("Aggregating data")
        )
        clean_data >> aggregate_data
    
    with TaskGroup("load_targets") as load:
        load_warehouse = PythonOperator(
            task_id="load_warehouse",
            python_callable=lambda: print("Load warehouse")
        )
        load_dashboard = PythonOperator(
            task_id="load_dashboard",
            python_callable=lambda: print("Load dashboard")
        )
    
    end = PythonOperator(
        task_id="end",
        python_callable=lambda: print("Pipeline complete"),
        trigger_rule="all_done"
    )
    
    start >> extract >> transform >> load >> end
```

---

## 11. Glossary

**Airflow:** Open-source platform for authoring, scheduling, and monitoring workflows as DAGs.

**Catchup:** Process of backfilling DAG runs between start_date and current time.

**CeleryExecutor:** Distributed task executor using Celery for horizontal scaling.

**Connection:** Stored credentials and configuration for external systems.

**DAG (Directed Acyclic Graph):** Workflow definition containing tasks and dependencies; "directed" means tasks flow in one direction, "acyclic" means no loops.

**DAG Run:** Single execution instance of a DAG for a specific execution date.

**Execution Date:** Logical start time of a data interval (not when DAG actually runs).

**Executor:** Component that determines how tasks are executed (Local, Celery, Kubernetes).

**Hook:** Interface to external systems providing authentication and connection management.

**Idempotency:** Property ensuring repeated task executions produce same result without side effects.

**KubernetesExecutor:** Executor that spawns dynamic Kubernetes pods for each task.

**LocalExecutor:** Multi-threaded executor for single-machine Airflow deployments.

**Metadata Database:** PostgreSQL/MySQL database storing all Airflow state and configuration.

**Operator:** Template defining specific work (Python, Bash, SQL, etc.); tasks are instances of operators.

**Pool:** Resource limit controlling concurrent task execution for shared resources.

**RBAC (Role-Based Access Control):** Security model controlling user access to DAG and admin features.

**Reschedule Mode:** Sensor mode that releases worker slot between pokes for efficiency.

**Scheduler:** Core component that parses DAGs, creates runs, and triggers task execution.

**Sensor:** Specialized operator that waits for conditions before proceeding.

**SLA (Service Level Agreement):** Expected maximum task duration; triggers callbacks when exceeded.

**Task:** Unit of work within a DAG; instance of an operator.

**Task Instance:** Single execution of a task for a specific DAG run.

**TaskFlow API:** Python decorator-based API for simplified DAG authoring (Airflow 2.0+).

**TaskGroup:** Organizational construct for grouping related tasks in UI visualization.

**Trigger Rule:** Condition determining when task executes (all_success, one_failed, etc.).

**Variable:** Key-value configuration stored in metadata database, accessible across DAGs.

**Web Server:** Flask-based UI for monitoring, triggering, and debugging workflows.

**Worker:** Process that executes individual tasks as queued by scheduler and executor.

**XCom (Cross-Communication):** Mechanism for tasks to exchange small messages; not for large datasets.

---

## References

All content grounded in official Apache Airflow documentation: https://airflow.apache.org/docs/

**Additional Resources:**
- Airflow GitHub: https://github.com/apache/airflow
- Airflow Community: https://airflow.apache.org/community/
- Airflow Providers: https://airflow.apache.org/docs/apache-airflow-providers/

---

**Document End**

