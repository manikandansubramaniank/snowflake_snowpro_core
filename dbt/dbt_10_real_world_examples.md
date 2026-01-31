# 10. Real-World Examples

## Overview
This section demonstrates complete dbt implementations: e-commerce analytics pipeline (staging → marts), SCD Type 2 customer dimension, and Snowflake CDC integration. Each example includes full configs, SDLC workflows, and production patterns.

---

## 10.1 E-Commerce Analytics Pipeline

### Business Requirements
Build a production-grade analytics layer for an e-commerce platform with:
- **Sources**: Raw orders, customers, products from Fivetran → Snowflake
- **Models**: Staging → Intermediate → Marts (star schema)
- **Metrics**: Daily revenue, customer LTV, product performance
- **Deployment**: Hourly incremental updates, daily full refresh

**Diagram: Complete Pipeline Architecture**
```text
Raw Data (Fivetran → Snowflake RAW schema)
     ↓ source() definitions
Staging Layer (light transformations, rename columns)
     ├── stg_orders (view)
     ├── stg_customers (view)
     └── stg_products (view)
     ↓ ref() + business logic
Intermediate Layer (joins, calculations)
     ├── int_orders_enriched (ephemeral)
     └── int_customer_metrics (table)
     ↓ dimensional modeling
Marts Layer (star schema, aggregations)
     ├── fct_orders (incremental, clustered)
     ├── dim_customers (SCD Type 2 snapshot)
     ├── dim_products (table)
     └── fct_daily_revenue (aggregated table)
     ↓ exposure definitions
BI Tools (Tableau, Looker)
```

---

### Project Structure

```text
ecommerce_analytics/
├── dbt_project.yml
├── packages.yml
├── profiles.yml
├── models/
│   ├── staging/
│   │   ├── _sources.yml
│   │   ├── stg_orders.sql
│   │   ├── stg_customers.sql
│   │   └── stg_products.sql
│   ├── intermediate/
│   │   ├── int_orders_enriched.sql
│   │   └── int_customer_metrics.sql
│   └── marts/
│       ├── _models.yml
│       ├── fct_orders.sql
│       ├── dim_customers.sql
│       ├── dim_products.sql
│       ├── fct_daily_revenue.sql
│       └── exposures.yml
├── macros/
│   └── generate_surrogate_key.sql
├── tests/
│   └── assert_positive_revenue.sql
└── snapshots/
    └── customers_snapshot.sql
```

---

### Configuration Files

**dbt_project.yml**
```yaml
name: 'ecommerce_analytics'
version: '1.0.0'
config-version: 2

profile: 'ecommerce'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

# Global configuration
models:
  ecommerce_analytics:
    +materialized: view
    +snowflake_warehouse: TRANSFORMING_WH
    +query_tag: "dbt:{{ model.name }}:{{ target.name }}"
    
    # Staging: lightweight views
    staging:
      +materialized: view
      +schema: staging
      +tags: ['staging', 'hourly']
    
    # Intermediate: ephemeral or tables
    intermediate:
      +materialized: ephemeral
      +schema: intermediate
      +tags: ['intermediate']
      
      int_customer_metrics:
        +materialized: table  # Exception: reused by multiple marts
    
    # Marts: production tables/incremental
    marts:
      +materialized: table
      +schema: marts
      +tags: ['marts', 'prod']
      +transient: false
      +copy_grants: true
      
      fct_orders:
        +materialized: incremental
        +unique_key: order_id
        +incremental_strategy: merge
        +cluster_by: ['order_date', 'customer_id']
        +tags: ['fact', 'hourly']
      
      fct_daily_revenue:
        +materialized: table
        +cluster_by: ['revenue_date']
        +tags: ['aggregate', 'daily']
      
      dim_customers:
        +materialized: table
        +cluster_by: ['customer_id']
        +tags: ['dimension', 'daily']
      
      dim_products:
        +materialized: table
        +cluster_by: ['product_id']
        +tags: ['dimension', 'daily']

snapshots:
  ecommerce_analytics:
    +target_schema: snapshots
    +strategy: timestamp
    +unique_key: customer_id
    +updated_at: updated_at
    +tags: ['scd2', 'daily']

vars:
  revenue_threshold: 1000000  # $1M for high-value customer segment
  lookback_days: 90  # Customer LTV calculation window
```

**packages.yml**
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
  
  - package: calogica/dbt_expectations
    version: 0.10.3
```

**profiles.yml**
```yaml
ecommerce:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: abc12345.us-east-1
      user: "{{ env_var('USER') }}"
      authenticator: externalbrowser
      role: DEVELOPER
      warehouse: TRANSFORMING_WH
      database: ANALYTICS_DEV
      schema: DBT_{{ env_var('USER') | upper }}
      threads: 4
      client_session_keep_alive: true
    
    prod:
      type: snowflake
      account: abc12345.us-east-1
      user: dbt_service_account
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: TRANSFORMER
      warehouse: TRANSFORMING_WH
      database: ANALYTICS_PROD
      schema: DBT_PROD
      threads: 8
      client_session_keep_alive: true
      query_tag: "dbt:prod:{{ invocation_id }}"
```

---

### Source Definitions

**models/staging/_sources.yml**
```yaml
version: 2

sources:
  - name: raw_ecommerce
    description: Raw e-commerce data loaded by Fivetran
    database: RAW_DATA
    schema: fivetran_ecommerce
    
    freshness:
      warn_after: {count: 2, period: hour}
      error_after: {count: 6, period: hour}
    
    loaded_at_field: _fivetran_synced
    
    tables:
      - name: orders
        description: Customer orders with line items
        columns:
          - name: order_id
            description: Unique order identifier
            tests:
              - unique
              - not_null
          
          - name: customer_id
            description: FK to customers table
            tests:
              - not_null
              - relationships:
                  to: source('raw_ecommerce', 'customers')
                  field: customer_id
          
          - name: order_date
            description: Order timestamp (UTC)
            tests:
              - not_null
          
          - name: order_amount_cents
            description: Total order value in cents
            tests:
              - not_null
              - dbt_expectations.expect_column_values_to_be_between:
                  min_value: 0
                  max_value: 10000000  # $100k max order
      
      - name: customers
        description: Customer master data
        columns:
          - name: customer_id
            description: Unique customer identifier
            tests:
              - unique
              - not_null
          
          - name: customer_email
            description: Customer email address
            tests:
              - unique
              - not_null
          
          - name: created_at
            description: Customer registration timestamp
            tests:
              - not_null
      
      - name: products
        description: Product catalog
        columns:
          - name: product_id
            description: Unique product SKU
            tests:
              - unique
              - not_null
          
          - name: product_name
            description: Product display name
          
          - name: category
            description: Product category
          
          - name: price_cents
            description: Product price in cents
```

---

### Staging Models

**models/staging/stg_orders.sql**
```sql
{{
  config(
    materialized='view'
  )
}}

WITH source AS (
  SELECT * FROM {{ source('raw_ecommerce', 'orders') }}
),

renamed AS (
  SELECT 
    -- IDs
    order_id,
    customer_id,
    product_id,
    
    -- Timestamps
    order_date,
    _fivetran_synced AS loaded_at,
    
    -- Amounts (convert cents to dollars)
    ROUND(order_amount_cents / 100.0, 2) AS order_amount,
    ROUND(discount_cents / 100.0, 2) AS discount_amount,
    ROUND(tax_cents / 100.0, 2) AS tax_amount,
    ROUND(shipping_cents / 100.0, 2) AS shipping_amount,
    
    -- Derived
    ROUND((order_amount_cents - discount_cents + tax_cents + shipping_cents) / 100.0, 2) AS total_amount,
    
    -- Flags
    CASE 
      WHEN order_status = 'completed' THEN TRUE
      ELSE FALSE
    END AS is_completed,
    
    CASE 
      WHEN refunded_at IS NOT NULL THEN TRUE
      ELSE FALSE
    END AS is_refunded,
    
    -- Metadata
    order_status,
    refunded_at
  
  FROM source
)

SELECT * FROM renamed
```

**models/staging/stg_customers.sql**
```sql
{{
  config(
    materialized='view'
  )
}}

WITH source AS (
  SELECT * FROM {{ source('raw_ecommerce', 'customers') }}
),

renamed AS (
  SELECT 
    -- IDs
    customer_id,
    
    -- PII
    customer_email,
    {{ dbt_utils.surrogate_key(['customer_email']) }} AS email_hash,
    
    -- Attributes
    TRIM(first_name) AS first_name,
    TRIM(last_name) AS last_name,
    CONCAT(TRIM(first_name), ' ', TRIM(last_name)) AS full_name,
    
    -- Address
    UPPER(country_code) AS country_code,
    state,
    city,
    postal_code,
    
    -- Timestamps
    created_at AS customer_created_at,
    updated_at,
    _fivetran_synced AS loaded_at,
    
    -- Segments
    CASE 
      WHEN lifetime_value_cents >= {{ var('revenue_threshold') }} THEN 'high_value'
      WHEN lifetime_value_cents >= 50000 THEN 'medium_value'
      ELSE 'low_value'
    END AS customer_segment
  
  FROM source
)

SELECT * FROM renamed
```

**models/staging/stg_products.sql**
```sql
{{
  config(
    materialized='view'
  )
}}

WITH source AS (
  SELECT * FROM {{ source('raw_ecommerce', 'products') }}
),

renamed AS (
  SELECT 
    -- IDs
    product_id,
    
    -- Attributes
    product_name,
    category,
    subcategory,
    brand,
    
    -- Pricing
    ROUND(price_cents / 100.0, 2) AS price,
    ROUND(cost_cents / 100.0, 2) AS cost,
    ROUND((price_cents - cost_cents) / 100.0, 2) AS profit_margin,
    
    -- Flags
    is_active,
    
    -- Timestamps
    created_at,
    updated_at,
    _fivetran_synced AS loaded_at
  
  FROM source
)

SELECT * FROM renamed
```

---

### Intermediate Models

**models/intermediate/int_orders_enriched.sql**
```sql
{{
  config(
    materialized='ephemeral'
  )
}}

WITH orders AS (
  SELECT * FROM {{ ref('stg_orders') }}
),

customers AS (
  SELECT * FROM {{ ref('stg_customers') }}
),

products AS (
  SELECT * FROM {{ ref('stg_products') }}
),

enriched AS (
  SELECT 
    -- Order details
    o.order_id,
    o.order_date,
    o.order_amount,
    o.discount_amount,
    o.tax_amount,
    o.shipping_amount,
    o.total_amount,
    o.is_completed,
    o.is_refunded,
    
    -- Customer details
    c.customer_id,
    c.customer_email,
    c.full_name,
    c.country_code,
    c.customer_segment,
    c.customer_created_at,
    
    -- Product details
    p.product_id,
    p.product_name,
    p.category,
    p.brand,
    p.price,
    p.cost,
    p.profit_margin,
    
    -- Calculated fields
    DATEDIFF(day, c.customer_created_at, o.order_date) AS days_since_first_order,
    
    CASE 
      WHEN DATEDIFF(day, c.customer_created_at, o.order_date) <= 30 THEN 'new_customer'
      WHEN DATEDIFF(day, c.customer_created_at, o.order_date) <= 90 THEN 'returning_customer'
      ELSE 'loyal_customer'
    END AS customer_tenure_bucket
  
  FROM orders o
  LEFT JOIN customers c ON o.customer_id = c.customer_id
  LEFT JOIN products p ON o.product_id = p.product_id
  
  WHERE o.is_completed = TRUE
    AND o.is_refunded = FALSE
)

SELECT * FROM enriched
```

**models/intermediate/int_customer_metrics.sql**
```sql
{{
  config(
    materialized='table'
  )
}}

WITH orders_enriched AS (
  SELECT * FROM {{ ref('int_orders_enriched') }}
),

customer_aggregates AS (
  SELECT 
    customer_id,
    
    -- Order counts
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT DATE(order_date)) AS distinct_order_days,
    
    -- Revenue metrics
    SUM(total_amount) AS lifetime_revenue,
    AVG(total_amount) AS avg_order_value,
    
    -- Recency
    MAX(order_date) AS last_order_date,
    DATEDIFF(day, MAX(order_date), CURRENT_DATE()) AS days_since_last_order,
    
    -- First/last order
    MIN(order_date) AS first_order_date,
    DATEDIFF(day, MIN(order_date), MAX(order_date)) AS customer_lifespan_days,
    
    -- Frequency
    CASE 
      WHEN COUNT(DISTINCT order_id) >= 10 THEN 'frequent'
      WHEN COUNT(DISTINCT order_id) >= 3 THEN 'occasional'
      ELSE 'one_time'
    END AS purchase_frequency_segment,
    
    -- Engagement
    CASE 
      WHEN DATEDIFF(day, MAX(order_date), CURRENT_DATE()) <= 30 THEN 'active'
      WHEN DATEDIFF(day, MAX(order_date), CURRENT_DATE()) <= 90 THEN 'at_risk'
      ELSE 'churned'
    END AS customer_status
  
  FROM orders_enriched
  GROUP BY 1
)

SELECT * FROM customer_aggregates
```

---

### Marts Layer (Fact Tables)

**models/marts/fct_orders.sql**
```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge',
    cluster_by=['order_date', 'customer_id'],
    on_schema_change='sync_all_columns'
  )
}}

WITH orders_enriched AS (
  SELECT * FROM {{ ref('int_orders_enriched') }}
),

final AS (
  SELECT 
    -- Keys
    order_id,
    customer_id,
    product_id,
    
    -- Order details
    order_date,
    order_amount,
    discount_amount,
    tax_amount,
    shipping_amount,
    total_amount,
    
    -- Customer attributes
    customer_segment,
    customer_tenure_bucket,
    
    -- Product attributes
    category AS product_category,
    brand AS product_brand,
    profit_margin,
    
    -- Metadata
    CURRENT_TIMESTAMP() AS _updated_at
  
  FROM orders_enriched
  
  {% if is_incremental() %}
    -- Process last 7 days to handle late-arriving data
    WHERE order_date >= DATEADD(day, -7, (SELECT MAX(order_date) FROM {{ this }}))
  {% endif %}
)

SELECT * FROM final
```

**models/marts/fct_daily_revenue.sql**
```sql
{{
  config(
    materialized='table',
    cluster_by=['revenue_date']
  )
}}

WITH orders AS (
  SELECT * FROM {{ ref('fct_orders') }}
),

daily_aggregates AS (
  SELECT 
    DATE(order_date) AS revenue_date,
    
    -- Order metrics
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    
    -- Revenue metrics
    SUM(order_amount) AS gross_revenue,
    SUM(discount_amount) AS total_discounts,
    SUM(tax_amount) AS total_tax,
    SUM(shipping_amount) AS total_shipping,
    SUM(total_amount) AS net_revenue,
    
    -- Average metrics
    AVG(total_amount) AS avg_order_value,
    
    -- Product metrics
    COUNT(DISTINCT product_id) AS unique_products_sold,
    
    -- Customer segments
    COUNT(DISTINCT CASE WHEN customer_segment = 'high_value' THEN customer_id END) AS high_value_customers,
    SUM(CASE WHEN customer_segment = 'high_value' THEN total_amount ELSE 0 END) AS high_value_revenue,
    
    -- Metadata
    CURRENT_TIMESTAMP() AS _updated_at
  
  FROM orders
  GROUP BY 1
)

SELECT * FROM daily_aggregates
ORDER BY revenue_date DESC
```

---

### Marts Layer (Dimension Tables)

**models/marts/dim_customers.sql**
```sql
{{
  config(
    materialized='table',
    cluster_by=['customer_id']
  )
}}

WITH customers AS (
  SELECT * FROM {{ ref('stg_customers') }}
),

customer_metrics AS (
  SELECT * FROM {{ ref('int_customer_metrics') }}
),

final AS (
  SELECT 
    -- Keys
    c.customer_id,
    {{ dbt_utils.generate_surrogate_key(['c.customer_id', 'c.updated_at']) }} AS customer_dim_key,
    
    -- Attributes
    c.customer_email,
    c.email_hash,  -- PII protection
    c.full_name,
    c.country_code,
    c.state,
    c.city,
    c.customer_segment,
    
    -- Metrics
    COALESCE(m.total_orders, 0) AS total_orders,
    COALESCE(m.lifetime_revenue, 0) AS lifetime_revenue,
    COALESCE(m.avg_order_value, 0) AS avg_order_value,
    m.last_order_date,
    m.days_since_last_order,
    m.purchase_frequency_segment,
    m.customer_status,
    
    -- Timestamps
    c.customer_created_at,
    c.updated_at,
    CURRENT_TIMESTAMP() AS _loaded_at
  
  FROM customers c
  LEFT JOIN customer_metrics m ON c.customer_id = m.customer_id
)

SELECT * FROM final
```

**models/marts/dim_products.sql**
```sql
{{
  config(
    materialized='table',
    cluster_by=['product_id']
  )
}}

WITH products AS (
  SELECT * FROM {{ ref('stg_products') }}
),

final AS (
  SELECT 
    -- Keys
    product_id,
    {{ dbt_utils.generate_surrogate_key(['product_id', 'updated_at']) }} AS product_dim_key,
    
    -- Attributes
    product_name,
    category,
    subcategory,
    brand,
    
    -- Pricing
    price,
    cost,
    profit_margin,
    
    -- Flags
    is_active,
    
    -- Timestamps
    created_at,
    updated_at,
    CURRENT_TIMESTAMP() AS _loaded_at
  
  FROM products
)

SELECT * FROM final
```

---

### Model Documentation

**models/marts/_models.yml**
```yaml
version: 2

models:
  - name: fct_orders
    description: Fact table of completed, non-refunded orders with customer and product dimensions
    
    config:
      tags: ['fact', 'hourly', 'prod']
    
    columns:
      - name: order_id
        description: Unique order identifier (primary key)
        tests:
          - unique
          - not_null
      
      - name: customer_id
        description: FK to dim_customers
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id
      
      - name: product_id
        description: FK to dim_products
        tests:
          - not_null
          - relationships:
              to: ref('dim_products')
              field: product_id
      
      - name: order_date
        description: Order timestamp (UTC)
        tests:
          - not_null
      
      - name: total_amount
        description: Total order value (order + tax + shipping - discounts)
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 100000
  
  - name: dim_customers
    description: Customer dimension with lifetime metrics
    
    columns:
      - name: customer_id
        description: Unique customer identifier (primary key)
        tests:
          - unique
          - not_null
      
      - name: customer_email
        description: Customer email address (PII)
        tests:
          - unique
          - not_null
      
      - name: lifetime_revenue
        description: Total revenue from customer across all orders
        tests:
          - not_null
  
  - name: dim_products
    description: Product dimension with pricing
    
    columns:
      - name: product_id
        description: Unique product SKU (primary key)
        tests:
          - unique
          - not_null
  
  - name: fct_daily_revenue
    description: Daily revenue aggregates for executive reporting
    
    columns:
      - name: revenue_date
        description: Date of revenue aggregation (primary key)
        tests:
          - unique
          - not_null
      
      - name: net_revenue
        description: Total net revenue (gross - discounts + tax + shipping)
        tests:
          - not_null
```

---

### Custom Tests

**tests/assert_positive_revenue.sql**
```sql
-- Singular test: Ensure no negative revenue days
SELECT 
  revenue_date,
  net_revenue
FROM {{ ref('fct_daily_revenue') }}
WHERE net_revenue < 0
```

---

### Exposures

**models/marts/exposures.yml**
```yaml
version: 2

exposures:
  - name: executive_revenue_dashboard
    type: dashboard
    maturity: high
    url: https://tableau.company.com/revenue_exec
    description: |
      Executive dashboard showing daily/weekly/monthly revenue trends,
      customer segments, and product performance.
    
    depends_on:
      - ref('fct_daily_revenue')
      - ref('dim_customers')
      - ref('dim_products')
    
    owner:
      name: Sales Analytics Team
      email: sales-analytics@company.com
      slack: '#sales-analytics'
    
    tags: ['executive', 'daily']
  
  - name: customer_segmentation_model
    type: ml
    maturity: high
    url: https://sagemaker.aws.amazon.com/projects/customer-segment
    description: ML model for customer churn prediction and LTV forecasting
    
    depends_on:
      - ref('dim_customers')
      - ref('int_customer_metrics')
    
    owner:
      name: Data Science Team
      email: data-science@company.com
    
    tags: ['ml', 'daily']
```

---

## 10.2 SCD Type 2: Customer Dimension

### What is SCD Type 2?
Slowly Changing Dimension Type 2 captures historical changes by creating new rows with validity periods. Essential for tracking customer attribute changes (email, address, segment).

**Diagram: SCD Type 2 Timeline**
```text
Customer ID 123 History:
┌─────────────┬────────────────┬─────────────────┬───────────────┬──────────────────┐
│ customer_id │ customer_email │ customer_segment│ dbt_valid_from│ dbt_valid_to     │
├─────────────┼────────────────┼─────────────────┼───────────────┼──────────────────┤
│ 123         │ old@email.com  │ low_value       │ 2024-01-01    │ 2024-06-15       │ ← Historical
│ 123         │ new@email.com  │ low_value       │ 2024-06-15    │ 2024-11-20       │ ← Historical
│ 123         │ new@email.com  │ high_value      │ 2024-11-20    │ NULL             │ ← Current
└─────────────┴────────────────┴─────────────────┴───────────────┴──────────────────┘
```

### Syntax: Snapshots

**snapshots/customers_snapshot.sql**
```sql
{% snapshot customers_snapshot %}

{{
  config(
    target_schema='snapshots',
    strategy='timestamp',
    unique_key='customer_id',
    updated_at='updated_at',
    invalidate_hard_deletes=True,
    tags=['scd2', 'daily']
  )
}}

SELECT 
  customer_id,
  customer_email,
  full_name,
  country_code,
  state,
  customer_segment,
  updated_at,
  _fivetran_synced
FROM {{ source('raw_ecommerce', 'customers') }}

{% endsnapshot %}
```

**PARAMETERS**

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| target_schema | String | - | Schema for snapshot table | snapshots |
| strategy | Enum | - | Change detection method | timestamp, check |
| unique_key | String | - | Primary key for tracking records | customer_id, order_id |
| updated_at | String | - | Timestamp column for change detection | updated_at, modified_at |
| check_cols | Array | - | Columns to monitor for changes (check strategy) | ['email', 'segment'] |
| invalidate_hard_deletes | Boolean | False | Set dbt_valid_to for deleted source records | True |

**Run Snapshot**
```bash
# Daily snapshot execution
dbt snapshot --select customers_snapshot

# Full refresh (rebuild from scratch)
dbt snapshot --select customers_snapshot --full-refresh

# Production schedule (daily 3am)
0 3 * * * dbt snapshot --target prod
```

**Query Historical Data**
```sql
-- Current state (active rows)
SELECT *
FROM snapshots.customers_snapshot
WHERE dbt_valid_to IS NULL;

-- Point-in-time query (state as of 2024-06-01)
SELECT *
FROM snapshots.customers_snapshot
WHERE dbt_valid_from <= '2024-06-01'
  AND (dbt_valid_to > '2024-06-01' OR dbt_valid_to IS NULL);

-- Customer history (all changes for customer 123)
SELECT 
  customer_id,
  customer_email,
  customer_segment,
  dbt_valid_from,
  dbt_valid_to,
  dbt_updated_at
FROM snapshots.customers_snapshot
WHERE customer_id = 123
ORDER BY dbt_valid_from;

-- Segment migration analysis
WITH customer_changes AS (
  SELECT 
    customer_id,
    customer_segment AS old_segment,
    LEAD(customer_segment) OVER (PARTITION BY customer_id ORDER BY dbt_valid_from) AS new_segment,
    dbt_valid_from AS change_date
  FROM snapshots.customers_snapshot
)

SELECT 
  old_segment,
  new_segment,
  COUNT(*) AS customer_count
FROM customer_changes
WHERE new_segment IS NOT NULL
  AND old_segment != new_segment
GROUP BY 1, 2
ORDER BY 3 DESC;
```

---

## 10.3 Snowflake CDC (Change Data Capture)

### What is CDC?
Change Data Capture streams database changes (INSERT/UPDATE/DELETE) for real-time synchronization. Snowflake Streams track DML operations for incremental processing.

**Diagram: CDC Pipeline**
```text
Source Table (OLTP database)
     ↓ Fivetran CDC
Snowflake RAW Table + Stream
     ↓ METADATA$ACTION (INSERT/UPDATE/DELETE)
dbt Incremental Model (process stream)
     ↓ merge logic
Analytics Table (ANALYTICS schema)
     ↓ BI queries
Real-time dashboards (5-min latency)
```

### Setup: Snowflake Streams

**Create Stream on Source Table**
```sql
-- Run as Snowflake admin
CREATE OR REPLACE STREAM raw_data.fivetran_ecommerce.orders_stream
  ON TABLE raw_data.fivetran_ecommerce.orders
  APPEND_ONLY = FALSE  -- Capture INSERT, UPDATE, DELETE
  SHOW_INITIAL_ROWS = FALSE;

-- Check stream status
SHOW STREAMS LIKE 'orders_stream' IN SCHEMA raw_data.fivetran_ecommerce;

-- View pending changes
SELECT 
  METADATA$ACTION,
  METADATA$ISUPDATE,
  METADATA$ROW_ID,
  *
FROM raw_data.fivetran_ecommerce.orders_stream
LIMIT 10;
```

### dbt Model: Process CDC Stream

**models/staging/stg_orders_cdc.sql**
```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge',
    on_schema_change='sync_all_columns',
    cluster_by=['order_date'],
    tags=['cdc', 'realtime']
  )
}}

WITH stream_data AS (
  SELECT 
    -- Metadata
    METADATA$ACTION AS stream_action,  -- INSERT, DELETE
    METADATA$ISUPDATE AS stream_is_update,  -- TRUE for UPDATE operations
    
    -- Order data
    order_id,
    customer_id,
    product_id,
    order_date,
    order_amount_cents,
    order_status,
    updated_at
  
  FROM {{ source_stream('raw_ecommerce', 'orders_stream') }}
  
  {% if is_incremental() %}
    -- Only process new stream records
    WHERE TRUE  -- Stream automatically filters to new changes
  {% endif %}
),

transformed AS (
  SELECT 
    order_id,
    customer_id,
    product_id,
    order_date,
    ROUND(order_amount_cents / 100.0, 2) AS order_amount,
    order_status,
    updated_at,
    
    -- CDC metadata
    stream_action,
    stream_is_update,
    CURRENT_TIMESTAMP() AS _processed_at
  
  FROM stream_data
)

SELECT * FROM transformed
```

**Macro: Source Stream Helper**
```sql
-- macros/source_stream.sql
{% macro source_stream(source_name, stream_name) %}
  {% set stream_relation = source(source_name, stream_name) %}
  {{ return(stream_relation) }}
{% endmacro %}
```

**Source Definition for Streams**
```yaml
# models/staging/_sources.yml
sources:
  - name: raw_ecommerce
    tables:
      - name: orders_stream
        description: Snowflake stream on orders table (CDC)
        identifier: ORDERS_STREAM
        database: RAW_DATA
        schema: fivetran_ecommerce
```

### Advanced: Handle Soft Deletes

**models/marts/fct_orders_with_deletes.sql**
```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge',
    cluster_by=['order_date']
  )
}}

WITH stream_data AS (
  SELECT 
    METADATA$ACTION AS action,
    order_id,
    customer_id,
    order_date,
    order_amount_cents,
    order_status,
    updated_at
  FROM {{ source_stream('raw_ecommerce', 'orders_stream') }}
),

transformed AS (
  SELECT 
    order_id,
    customer_id,
    order_date,
    ROUND(order_amount_cents / 100.0, 2) AS order_amount,
    order_status,
    
    -- Soft delete flag
    CASE 
      WHEN action = 'DELETE' THEN TRUE
      ELSE FALSE
    END AS is_deleted,
    
    updated_at,
    CURRENT_TIMESTAMP() AS _processed_at
  
  FROM stream_data
)

SELECT * FROM transformed
```

**Consumption: Query Active Records Only**
```sql
-- BI query: Exclude soft-deleted records
SELECT 
  order_date,
  SUM(order_amount) AS daily_revenue
FROM {{ ref('fct_orders_with_deletes') }}
WHERE is_deleted = FALSE
GROUP BY 1;
```

---

## 10.4 SDLC: Agile Sprint Demo

### Sprint Goal
Deploy e-commerce analytics pipeline to production with full testing and monitoring.

**Sprint Timeline (2 weeks)**
```text
Day 1-2: Setup
  - Initialize dbt project
  - Configure Snowflake connection
  - Define sources

Day 3-6: Development
  - Build staging models
  - Implement intermediate transformations
  - Create marts (fct_orders, dim_customers)

Day 7-8: Testing
  - Add generic tests (unique, not_null, relationships)
  - Write singular tests (assert_positive_revenue)
  - Configure freshness checks

Day 9: Documentation
  - Add model descriptions
  - Define exposures
  - Generate dbt docs

Day 10-11: CI/CD
  - GitHub Actions workflow (slim CI)
  - Production deployment
  - Artifact management

Day 12: Monitoring
  - Query tags for cost tracking
  - Slack alerts on failure
  - Performance dashboard

Day 13-14: Demo & Retrospective
```

**GitHub Workflow: Complete CI/CD**
```yaml
# .github/workflows/dbt_ci_cd.yml
name: dbt CI/CD Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

env:
  DBT_PROFILES_DIR: ./

jobs:
  # PR: Slim CI
  slim_ci:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install dbt-snowflake==1.7.0
          dbt deps
      
      - name: Download prod artifacts
        run: |
          mkdir -p prod_artifacts
          aws s3 sync s3://dbt-artifacts/prod/ prod_artifacts/
      
      - name: dbt build (modified models only)
        run: |
          dbt build \
            --select state:modified+ \
            --state prod_artifacts \
            --defer \
            --target ci \
            --fail-fast
        env:
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_CI_PASSWORD }}
      
      - name: Comment PR with summary
        if: always()
        run: |
          echo "## dbt Run Summary" > comment.md
          echo "Modified models: $(dbt ls --select state:modified+ --state prod_artifacts)" >> comment.md
          gh pr comment ${{ github.event.pull_request.number }} --body-file comment.md
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  
  # Main: Production Deployment
  prod_deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install dbt-snowflake==1.7.0
          dbt deps
      
      - name: Download prod artifacts
        run: |
          mkdir -p prod_artifacts
          aws s3 sync s3://dbt-artifacts/prod/ prod_artifacts/
      
      - name: dbt build (modified + new models)
        run: |
          dbt build \
            --select state:modified+ state:new \
            --state prod_artifacts \
            --target prod
        env:
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PROD_PASSWORD }}
      
      - name: Generate documentation
        run: dbt docs generate --target prod
      
      - name: Upload artifacts
        run: |
          aws s3 sync target/ s3://dbt-artifacts/prod/
          aws s3 sync target/ s3://dbt-docs-prod/ --exclude "*.json"
      
      - name: Notify Slack
        if: always()
        run: |
          STATUS="${{ job.status }}"
          if [ "$STATUS" == "success" ]; then
            curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
              -d '{"text":"✅ dbt production deployed successfully"}'
          else
            curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
              -d '{"text":"🚨 dbt production deployment FAILED - check logs"}'
          fi
```

**Production Deployment Checklist**
```markdown
## Pre-Deployment
- [ ] All PR reviews approved
- [ ] CI tests passing
- [ ] No merge conflicts
- [ ] dbt compile succeeds locally
- [ ] Run dbt test --select state:modified+

## Deployment
- [ ] Merge PR to main
- [ ] GitHub Actions runs production deployment
- [ ] Verify artifacts uploaded to S3
- [ ] Check Snowflake for new/updated tables

## Post-Deployment
- [ ] Validate row counts (compare staging vs prod)
- [ ] Check Slack for deployment notification
- [ ] Verify BI dashboards reflect changes
- [ ] Monitor Snowflake query performance for 1 hour

## Rollback Plan
- [ ] Revert git commit if critical issues
- [ ] Restore previous artifacts from S3
- [ ] Run dbt build --full-refresh if needed
```

---

## Glossary

**Adapter**: Plugin translating dbt's generic SQL to warehouse-specific dialects (dbt-snowflake, dbt-bigquery).

**Artifact**: JSON metadata files (manifest.json, run_results.json) generated by dbt for state comparison and lineage.

**Clustering**: Organizing table data by specified columns to improve query performance via partition pruning (Snowflake/BigQuery).

**Config**: Settings applied to models, sources, or snapshots (e.g., +materialized, +cluster_by).

**DAG (Directed Acyclic Graph)**: Model dependency graph; dbt builds models in topological order.

**Defer**: Use production models for unbuilt upstream dependencies (--defer flag in CI/CD).

**Dimension Table**: Lookup table with descriptive attributes (dim_customers, dim_products) in star schema.

**dbt Cloud**: Managed dbt service with web IDE, scheduling, and observability.

**dbt Core**: Open-source CLI tool for running dbt projects locally or in CI/CD.

**Ephemeral**: Materialization creating CTEs instead of physical tables; used for simple transformations.

**Exposure**: Downstream dependency (dashboard, ML model) documented in YAML for impact analysis.

**Fact Table**: Central table in star schema containing measurable events (fct_orders, fct_revenue).

**Freshness**: Source data recency check; warns/errors if source hasn't updated within threshold.

**Incremental Model**: Materialization updating only new/changed rows, reducing compute costs (95%+ savings).

**Invocation ID**: Unique identifier for each dbt run; useful for query tagging and debugging.

**Jinja**: Templating language in dbt for dynamic SQL ({% if %}, {{ ref() }}, macros).

**Lineage**: Upstream/downstream dependency graph showing data flow from sources to exposures.

**Macro**: Reusable SQL function written in Jinja (e.g., surrogate_key, date_trunc).

**Manifest**: Artifact (manifest.json) containing compiled project graph, configs, and dependencies.

**Materialization**: Strategy for persisting models: table, view, incremental, ephemeral.

**Merge Strategy**: Incremental approach using MERGE/UPSERT to update existing rows and insert new ones.

**Model**: SQL SELECT statement defining a dataset; builds table/view in warehouse.

**Node**: Individual component in dbt DAG (model, test, source, snapshot, seed).

**Package**: Reusable dbt project installed via packages.yml (dbt_utils, dbt_expectations).

**Partition**: Dividing table into segments based on column values for query performance (BigQuery PARTITION BY).

**Profile**: Connection configuration (profiles.yml) specifying warehouse credentials and settings.

**ref()**: Jinja function referencing another model; establishes dependencies.

**Schema**: Logical grouping of tables in warehouse; dbt projects organize models by schema.

**SCD (Slowly Changing Dimension)**: Dimension table tracking historical changes; Type 2 creates new rows with validity periods.

**Seed**: Static CSV file loaded as table (dbt seed); useful for small reference data.

**Selector**: dbt syntax for choosing model subsets (tag:daily, state:modified+, +model_name).

**Slim CI**: Running only modified models using --select state:modified+ --defer for fast CI/CD.

**Snapshot**: dbt feature implementing SCD Type 2; tracks historical changes with dbt_valid_from/dbt_valid_to.

**Source**: External table/view defined in sources.yml; accessed via source() function.

**Source Freshness**: Check ensuring source data updated within expected timeframe.

**State**: Previous dbt run artifacts used for comparison (state:modified, result:error).

**Surrogate Key**: Synthetic primary key generated from multiple columns (MD5 hash, concatenation).

**Test**: Assertion validating data quality; generic (unique, not_null) or singular (custom SQL).

**Transient Table**: Snowflake table type with reduced storage costs (no fail-safe, 1-day Time Travel).

**Warehouse**: Compute resource in cloud data platforms (Snowflake warehouse, BigQuery slots).

---

## Summary

**E-Commerce Pipeline**: Complete staging → intermediate → marts architecture with incremental models, clustering, and BI exposures.

**SCD Type 2**: Snapshots track historical dimension changes with dbt_valid_from/dbt_valid_to columns.

**CDC Integration**: Snowflake Streams + dbt incremental models enable real-time change processing.

**SDLC**: GitHub Actions slim CI, production deployments, artifact management, and monitoring complete the production workflow.

---
