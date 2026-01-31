# 5. Testing & Quality

## Overview
dbt tests enforce data quality through SQL assertions executed after model builds. Four test types: singular (custom SQL), generic (reusable YAML), data (schema.yml), and unit tests (Python/SQL fixtures). Tests integrate into CI/CD pipelines to prevent bad data from reaching production.

---

## 5.1 Test Types

### Test Classification
**Diagram: dbt Test Hierarchy**
```text
┌───────────────────────────────────────────────────────────────┐
│ dbt Tests                                                     │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ DATA TESTS (schema.yml)                                 │  │
│  │ Run against models/sources after materialization        │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │                                                         │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │ GENERIC TESTS (reusable YAML)                     │  │  │
│  │  │ - unique, not_null, accepted_values, relationships│  │  │
│  │  │ - Custom generic tests in tests/generic/          │  │  │
│  │  │ - Applied in schema.yml                           │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  │                                                         │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │ SINGULAR TESTS (custom SQL in tests/)            │   │  │
│  │  │ - Bespoke assertions                             │   │  │
│  │  │ - SELECT returns 0 rows = pass                   │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ UNIT TESTS (Preview - dbt v1.8+)                        │  │
│  │ Test model logic with mock inputs before materialization│  │
│  │ - Defined in YAML with given/expect blocks              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ FRESHNESS TESTS (sources)                               │  │
│  │ Validate data recency for upstream sources              │  │
│  │ - warn_after / error_after thresholds                   │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘

Execution:
  dbt test                    # All tests
  dbt test --select model_name  # Model-specific
  dbt source freshness        # Freshness only
```

**Data Tests:** SQL assertions validating data quality (uniqueness, null constraints, referential integrity). Run with `dbt test`.

**Unit Tests:** Logic validation with mocked inputs/outputs (preview feature). Run with `dbt test --select test_type:unit`.

**Freshness Tests:** Timestamp-based checks ensuring sources are updated recently. Run with `dbt source freshness`.

---

## 5.2 Generic Tests (Built-in)

### Syntax: schema.yml Tests

```yaml
# ─────────────────────────────────────────────────────────────
# SYNTAX TEMPLATE
# ─────────────────────────────────────────────────────────────
version: 2

models:
  - name: <model_name>
    description: <model_description>
    columns:
      - name: <column_name>
        description: <column_description>
        tests:
          - unique:
              config:
                severity: error | warn
                error_if: <condition>
                warn_if: <condition>
                limit: <number>
                store_failures: true | false
                tags: [<tag1>]
          
          - not_null:
              config:
                severity: error | warn
                where: <sql_condition>
          
          - accepted_values:
              values: [<value1>, <value2>]
              quote: true | false
              config:
                severity: error | warn
          
          - relationships:
              to: ref('<parent_model>')
              field: <parent_column>
              config:
                severity: error | warn

    # Model-level tests
    tests:
      - <test_name>:
          config:
            severity: error | warn

sources:
  - name: <source_name>
    tables:
      - name: <table_name>
        columns:
          - name: <column_name>
            tests:
              - <test_config>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `severity` | Enum | `error` | Test outcome: `error` (exits non-zero), `warn` (logs but passes) | `warn` |
| `error_if` | String | `!=0` | SQL condition for error (overrides severity) | `">100"` (error if >100 failures) |
| `warn_if` | String | `!=0` | SQL condition for warning | `">10"` |
| `limit` | Integer | No limit | Max failing rows to show in logs | `100` |
| `store_failures` | Boolean | `false` | Save failing rows to table (audit trail) | `true` |
| `tags` | Array | `[]` | Test metadata for selection | `['critical', 'daily']` |
| `where` | String | None | Filter rows before testing (e.g., test only active records) | `"status = 'active'"` |
| `quote` | Boolean | `true` | Quote values in accepted_values SQL | `false` |
| `enabled` | Boolean | `true` | Toggle test execution | `false` |

**Built-in Generic Tests:**
- `unique`: Column has no duplicate values
- `not_null`: Column has no NULL values
- `accepted_values`: Column values match allowed list
- `relationships`: Foreign key constraint (column exists in parent table)

---

**BASIC EXAMPLE**

```yaml
# models/schema.yml
version: 2

models:
  - name: customers
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      
      - name: email
        tests:
          - not_null
      
      - name: status
        tests:
          - accepted_values:
              values: ['active', 'inactive', 'suspended']
```

```bash
dbt test --select customers
```

---

**ADVANCED EXAMPLE: Custom Severity + Store Failures**

```yaml
# models/marts/schema.yml
version: 2

models:
  - name: fct_orders
    description: "Order fact table with revenue metrics"
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - unique:
              config:
                severity: error
                store_failures: true
                tags: ['critical']
          - not_null:
              config:
                severity: error
      
      - name: customer_id
        description: "Foreign key to dim_customers"
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id
              config:
                severity: warn  # Don't fail pipeline on orphan records
                where: "order_date >= '2024-01-01'"  # Only test recent orders
      
      - name: status
        tests:
          - accepted_values:
              values: ['pending', 'processing', 'shipped', 'delivered', 'cancelled']
              config:
                severity: error
      
      - name: total_amount
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
              config:
                severity: error
                error_if: ">10"  # Error if >10 negative amounts
                warn_if: ">0"    # Warn if any negative amounts

    # Model-level test
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - order_id
            - line_item_id
          config:
            severity: error
```

**Store Failures Configuration:**
```yaml
# dbt_project.yml
tests:
  +store_failures: true
  +schema: test_failures  # Create test_failures schema for failed rows
```

---

## 5.3 Singular Tests (Custom SQL)

### Concept
Singular tests are bespoke SQL queries in `tests/` folder. Test passes if query returns 0 rows. Use for complex business logic that doesn't fit generic tests.

**Pattern:** `SELECT` statement returns failing rows; dbt counts them.

---

### Syntax: Singular Test

```sql
-- ─────────────────────────────────────────────────────────────
-- SYNTAX TEMPLATE (tests/<test_name>.sql)
-- ─────────────────────────────────────────────────────────────
-- Test description: <what_this_validates>
-- Expected: 0 rows (SELECT returns failing rows)

{{ config(
    severity='error' | 'warn',
    tags=['<tag1>', '<tag2>'],
    store_failures=true | false
) }}

SELECT
  <failing_row_identifier>,
  <diagnostic_columns>
FROM {{ ref('<model_name>') }}
WHERE <failure_condition>
```

---

**BASIC EXAMPLE: Revenue Validation**

```sql
-- tests/assert_revenue_positive.sql
-- Validate all order line items have positive revenue

SELECT
  order_id,
  line_item_id,
  revenue
FROM {{ ref('fct_order_items') }}
WHERE revenue < 0
```

**Execution:**
```bash
dbt test --select assert_revenue_positive
# PASS: 0 rows returned
# FAIL: N rows returned (shows failing records)
```

---

**ADVANCED EXAMPLE: Cross-Model Validation**

```sql
-- tests/assert_orders_match_payments.sql
-- Ensure total order amount matches sum of payments

{{ config(
    severity='error',
    tags=['critical', 'finance'],
    store_failures=true
) }}

WITH order_totals AS (
  SELECT
    order_id,
    SUM(total_amount) AS order_total
  FROM {{ ref('fct_orders') }}
  GROUP BY 1
),

payment_totals AS (
  SELECT
    order_id,
    SUM(payment_amount) AS payment_total
  FROM {{ ref('fct_payments') }}
  GROUP BY 1
)

SELECT
  COALESCE(o.order_id, p.order_id) AS order_id,
  o.order_total,
  p.payment_total,
  ABS(COALESCE(o.order_total, 0) - COALESCE(p.payment_total, 0)) AS discrepancy
FROM order_totals o
FULL OUTER JOIN payment_totals p
  ON o.order_id = p.order_id
WHERE ABS(COALESCE(o.order_total, 0) - COALESCE(p.payment_total, 0)) > 0.01
  -- Allow 1 cent rounding differences
```

---

## 5.4 Custom Generic Tests

### Concept
Create reusable test macros in `tests/generic/` for organization-specific validations. Define once, apply via YAML.

**Pattern:** Macro accepts `model` and test parameters; returns SQL SELECT.

---

### Syntax: Custom Generic Test

```sql
-- ─────────────────────────────────────────────────────────────
-- SYNTAX TEMPLATE (tests/generic/<test_name>.sql)
-- ─────────────────────────────────────────────────────────────
{% test <test_name>(model, column_name, <param1>, <param2>) %}

SELECT
  {{ column_name }} AS failing_column,
  COUNT(*) AS failure_count
FROM {{ model }}
WHERE <failure_condition>
GROUP BY 1
HAVING COUNT(*) > 0

{% endtest %}
```

**PARAMETERS (macro signature):**
- `model`: Model reference (automatically passed by dbt)
- `column_name`: Column being tested (if column-level test)
- Custom parameters: Defined in schema.yml, passed to macro

---

**BASIC EXAMPLE: Custom Range Test**

```sql
-- tests/generic/within_range.sql
{% test within_range(model, column_name, min_value, max_value) %}

SELECT
  {{ column_name }},
  COUNT(*) AS violations
FROM {{ model }}
WHERE {{ column_name }} < {{ min_value }}
   OR {{ column_name }} > {{ max_value }}
GROUP BY 1

{% endtest %}
```

**Usage in schema.yml:**
```yaml
models:
  - name: products
    columns:
      - name: price
        tests:
          - within_range:
              min_value: 0
              max_value: 10000
```

---

**ADVANCED EXAMPLE: Percentage Threshold Test**

```sql
-- tests/generic/fail_if_null_pct_exceeds.sql
{% test fail_if_null_pct_exceeds(model, column_name, threshold_pct) %}

WITH validation AS (
  SELECT
    COUNT(*) AS total_rows,
    SUM(CASE WHEN {{ column_name }} IS NULL THEN 1 ELSE 0 END) AS null_rows,
    (SUM(CASE WHEN {{ column_name }} IS NULL THEN 1 ELSE 0 END)::FLOAT / COUNT(*)) * 100 AS null_pct
  FROM {{ model }}
)

SELECT
  total_rows,
  null_rows,
  null_pct,
  {{ threshold_pct }} AS threshold_pct
FROM validation
WHERE null_pct > {{ threshold_pct }}

{% endtest %}
```

**Usage:**
```yaml
models:
  - name: customers
    columns:
      - name: email
        tests:
          - fail_if_null_pct_exceeds:
              threshold_pct: 5  # Fail if >5% emails are NULL
              config:
                severity: warn
```

---

## 5.5 Source Freshness Tests

### Concept
Freshness tests validate that source tables are updated within expected timeframes. Use `loaded_at_field` timestamp to check staleness. Configured in `sources.yml`.

**When to Use:** External data feeds, API imports, scheduled ETL jobs where delays indicate upstream failures.

---

### Syntax: Source Freshness Configuration

```yaml
# ─────────────────────────────────────────────────────────────
# SYNTAX TEMPLATE (models/staging/<source>/<source_name>.yml)
# ─────────────────────────────────────────────────────────────
version: 2

sources:
  - name: <source_name>
    description: <source_description>
    database: <database_name>
    schema: <schema_name>
    loaded_at_field: <timestamp_column>
    freshness:
      warn_after:
        count: <number>
        period: minute | hour | day
      error_after:
        count: <number>
        period: minute | hour | day
    
    tables:
      - name: <table_name>
        description: <table_description>
        loaded_at_field: <override_timestamp>
        freshness:
          warn_after: {count: <N>, period: <unit>}
          error_after: {count: <N>, period: <unit>}
        columns:
          - name: <column_name>
            tests:
              - not_null
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `loaded_at_field` | String | Required | Timestamp column indicating when row was loaded/updated | `updated_at` or `_loaded_at` |
| `warn_after` | Object | None | Threshold for warning: `{count: N, period: 'hour'}` | `{count: 24, period: 'hour'}` |
| `error_after` | Object | None | Threshold for error: `{count: N, period: 'day'}` | `{count: 2, period: 'day'}` |
| `count` | Integer | Required | Number of time units | `12` |
| `period` | Enum | Required | Time unit: `minute`, `hour`, `day` | `hour` |

**Behavior:**
- dbt queries `MAX(loaded_at_field)` from source table
- Compares to current timestamp
- **WARN** if `MAX(loaded_at_field) < NOW() - warn_after`
- **ERROR** if `MAX(loaded_at_field) < NOW() - error_after`

---

**BASIC EXAMPLE**

```yaml
# models/staging/ecommerce/sources.yml
version: 2

sources:
  - name: ecommerce
    database: raw
    schema: public
    loaded_at_field: updated_at  # Default for all tables
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    
    tables:
      - name: orders
        description: "Raw orders from Shopify"
      
      - name: customers
        description: "Customer master data"
```

**Execution:**
```bash
dbt source freshness
# Output:
# 15:30:00 | PASS freshness of ecommerce.orders
# 15:30:01 | WARN freshness of ecommerce.customers (stale for 18 hours)
```

---

**ADVANCED EXAMPLE: Multi-Source with Overrides**

```yaml
# models/staging/sources.yml
version: 2

sources:
  - name: erp_system
    database: raw_erp
    schema: prod
    freshness:
      warn_after: {count: 6, period: hour}
      error_after: {count: 12, period: hour}
    
    tables:
      - name: invoices
        loaded_at_field: last_modified_timestamp
        freshness:
          warn_after: {count: 1, period: hour}  # Override: more critical
          error_after: {count: 4, period: hour}
        columns:
          - name: invoice_id
            tests:
              - unique
              - not_null
      
      - name: inventory_snapshots
        loaded_at_field: snapshot_time
        freshness:
          warn_after: {count: 24, period: hour}  # Override: daily batch
          error_after: {count: 48, period: hour}

  - name: external_api
    database: raw_external
    schema: api_data
    tables:
      - name: weather_data
        loaded_at_field: fetched_at
        freshness:
          warn_after: {count: 30, period: minute}  # Real-time API
          error_after: {count: 2, period: hour}
```

**CI/CD Integration:**
```bash
# Run freshness checks before dbt run
dbt source freshness --output target/sources.json
if [ $? -ne 0 ]; then
  echo "Source freshness failed - aborting pipeline"
  exit 1
fi

dbt run
```

---

## 5.6 Unit Tests (Preview)

### Concept
Unit tests validate model logic with mocked inputs before materialization. Define fixtures in YAML with `given` (mock upstream refs/sources) and `expect` (expected output). Preview feature in dbt v1.8+.

**When to Use:** Test complex SQL logic (CTEs, window functions) without running full pipeline.

---

### Syntax: Unit Test Configuration

```yaml
# ─────────────────────────────────────────────────────────────
# SYNTAX TEMPLATE (models/<folder>/schema.yml or separate unit_tests.yml)
# ─────────────────────────────────────────────────────────────
unit_tests:
  - name: <test_name>
    description: <test_description>
    model: <model_name>
    given:
      - input: ref('<upstream_model>')
        rows:
          - {<column1>: <value1>, <column2>: <value2>}
          - {<column1>: <value3>, <column2>: <value4>}
      
      - input: source('<source_name>', '<table_name>')
        rows:
          - {<column>: <value>}
    
    expect:
      rows:
        - {<output_col1>: <expected_value1>, <output_col2>: <expected_value2>}
    
    overrides:
      macros:
        <macro_name>: <override_sql>
      vars:
        <var_name>: <override_value>
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `name` | String | Required | Unique test identifier | `test_revenue_calculation` |
| `model` | String | Required | Target model to test | `fct_orders` |
| `given` | Array | Required | Mock inputs: `input` (ref/source), `rows` (fixture data) | See examples |
| `expect` | Object | Required | Expected output: `rows` (array of result rows) | See examples |
| `overrides` | Object | Optional | Override macros/vars for test context | `{vars: {is_test: true}}` |

---

**BASIC EXAMPLE**

```yaml
# models/marts/unit_tests.yml
unit_tests:
  - name: test_order_totals
    description: "Verify order total calculation"
    model: fct_orders
    
    given:
      - input: ref('stg_order_items')
        rows:
          - {order_id: 1, item_id: 1, quantity: 2, unit_price: 10.00}
          - {order_id: 1, item_id: 2, quantity: 1, unit_price: 5.00}
          - {order_id: 2, item_id: 3, quantity: 3, unit_price: 8.00}
    
    expect:
      rows:
        - {order_id: 1, total_amount: 25.00}
        - {order_id: 2, total_amount: 24.00}
```

```sql
-- models/marts/fct_orders.sql
SELECT
  order_id,
  SUM(quantity * unit_price) AS total_amount
FROM {{ ref('stg_order_items') }}
GROUP BY 1
```

**Execution:**
```bash
dbt test --select test_type:unit
# Output:
# 16:00:00 | PASS test_order_totals
```

---

**ADVANCED EXAMPLE: Multi-Input with Overrides**

```yaml
unit_tests:
  - name: test_customer_segmentation
    description: "Test segment logic with various customer scenarios"
    model: dim_customers
    
    given:
      - input: ref('stg_customers')
        rows:
          - {customer_id: 1, name: 'Alice', created_at: '2024-01-01'}
          - {customer_id: 2, name: 'Bob', created_at: '2024-06-01'}
      
      - input: ref('stg_orders')
        rows:
          - {customer_id: 1, order_date: '2024-01-15', total: 500}
          - {customer_id: 1, order_date: '2024-02-01', total: 600}
          - {customer_id: 2, order_date: '2024-06-05', total: 50}
    
    expect:
      rows:
        - {customer_id: 1, name: 'Alice', segment: 'High Value', lifetime_value: 1100}
        - {customer_id: 2, name: 'Bob', segment: 'Low Value', lifetime_value: 50}
    
    overrides:
      vars:
        high_value_threshold: 1000
        is_unit_test: true
```

---

## 5.7 SDLC Integration: Automated Test Gates

### Use Case: GitHub Actions CI/CD
Block merges if dbt tests fail. Run tests on every PR to prevent bad data logic from reaching production.

**Pattern:** `dbt test` in CI → exit non-zero on failure → block merge.

---

**GitHub Actions Workflow:**

```yaml
# .github/workflows/dbt_ci.yml
name: dbt CI

on:
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dbt
        run: |
          pip install dbt-snowflake==1.7.0
      
      - name: Setup dbt profiles
        run: |
          mkdir -p ~/.dbt
          echo "${{ secrets.DBT_PROFILES_YML }}" > ~/.dbt/profiles.yml
      
      - name: Install dependencies
        run: dbt deps
      
      - name: Run tests on modified models
        run: |
          dbt test --select state:modified+ --state ./prod-artifacts/ --fail-fast
          # --fail-fast: Stop on first failure for faster feedback
      
      - name: Run critical tests always
        if: always()
        run: |
          dbt test --select tag:critical
      
      - name: Check source freshness
        run: |
          dbt source freshness --select source:ecommerce --output target/freshness.json
      
      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: dbt-test-failures
          path: target/run_results.json
```

**Branch Protection Rule (GitHub):**
```yaml
Settings → Branches → main → Require status checks to pass:
  ✅ dbt CI / test
```

---

**Advanced: Store Test Failures in Snowflake**

```yaml
# dbt_project.yml
tests:
  +store_failures: true
  +schema: test_failures_{{ target.name }}  # Separate schema per environment

# models/schema.yml (critical tests)
models:
  - name: fct_orders
    columns:
      - name: order_id
        tests:
          - unique:
              config:
                severity: error
                store_failures: true
                tags: ['critical']
```

**Query Failed Test Results:**
```sql
-- Snowflake query to inspect failures
SELECT
  test_name,
  failing_row_count,
  execution_time
FROM test_failures_dev.unique_fct_orders_order_id
WHERE execution_time > CURRENT_TIMESTAMP - INTERVAL '1 hour'
ORDER BY execution_time DESC;
```

---

## 5.8 Test Selection Patterns

### Command Syntax

```bash
# ─────────────────────────────────────────────────────────────
# SYNTAX TEMPLATE
# ─────────────────────────────────────────────────────────────
dbt test [options]

# Options:
--select <selector>        # Run specific tests
--exclude <selector>       # Skip tests
--fail-fast               # Stop on first failure
--store-failures          # Override store_failures config
--indirect-selection      # Include indirect tests (default: eager)

# Selectors:
<test_name>               # Singular test by name
<model_name>              # All tests for model
tag:<tag_name>            # Tests with tag
test_type:generic         # Only generic tests
test_type:singular        # Only singular tests
test_type:unit            # Only unit tests
test_name:<test>          # Tests matching name
config.severity:error     # Tests with severity=error
source:<source_name>      # Source tests
state:modified            # Tests for modified models
```

---

**EXAMPLES**

```bash
# Run all tests
dbt test

# Run tests for specific model
dbt test --select customers

# Run tests for model + downstream
dbt test --select customers+

# Run only critical tests
dbt test --select tag:critical

# Run tests excluding warnings
dbt test --exclude config.severity:warn

# Run generic tests only
dbt test --select test_type:generic

# Run tests for modified models (CI)
dbt test --select state:modified+ --state ./prod-artifacts/

# Run freshness checks
dbt source freshness --select source:ecommerce

# Store failures for debugging
dbt test --select fct_orders --store-failures

# Fail fast (dev iteration)
dbt test --select staging.* --fail-fast
```

---

## Summary

**Test Types:**
- **Generic Tests:** Reusable YAML tests (unique, not_null, accepted_values, relationships)
- **Singular Tests:** Custom SQL assertions in `tests/` folder
- **Custom Generic Tests:** Organization-specific reusable tests via macros
- **Unit Tests:** Mock-based logic validation (preview feature)
- **Freshness Tests:** Source data recency validation

**Severity Levels:**
- `error`: Fails dbt test command (exit code 1)
- `warn`: Logs warning but passes (exit code 0)

**SDLC Best Practice:** Run `dbt test` in CI/CD; use `--fail-fast` for rapid feedback; store failures for audit trails.

---

## Quick Reference Commands

```bash
# Test everything
dbt test

# Model-specific tests
dbt test --select fct_orders

# Tag-based selection
dbt test --select tag:critical tag:daily

# Freshness checks
dbt source freshness

# Store failures for debugging
dbt test --store-failures --select fct_orders

# CI pipeline: test modified models
dbt test --select state:modified+ --state ./prod/ --fail-fast

# Unit tests only (v1.8+)
dbt test --select test_type:unit

# Exclude warnings
dbt test --exclude config.severity:warn
```

---

## Test Coverage Best Practices

**Column-Level Tests (schema.yml):**
- Primary keys: `unique` + `not_null`
- Foreign keys: `relationships` + `not_null`
- Enum columns: `accepted_values`
- Numeric columns: Custom `within_range` or `expression_is_true`

**Model-Level Tests:**
- Row counts: `dbt_utils.fewer_rows_than`, `at_least_one`
- Uniqueness: `dbt_utils.unique_combination_of_columns`
- Business logic: Singular tests in `tests/`

**Freshness:**
- Critical sources: `error_after` < SLA
- Batch sources: `warn_after` = expected cadence + buffer

**Unit Tests:**
- Complex CTEs, window functions, edge cases
- Mock dependencies to isolate logic

**CI/CD Gates:**
- `tag:critical` → blocking
- `tag:daily` → non-blocking (warn only)
- Store failures for postmortem analysis
