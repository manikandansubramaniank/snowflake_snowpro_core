# 5. Querying & Performance Optimization

## SnowSQL Basics

Snowflake SQL is ANSI-compliant with extensions for semi-structured data (JSON, Avro, Parquet), window functions, and time-series operations. All queries execute on virtual warehouses with automatic parallelization and result caching.

### Query Execution Flow

```
Query Submission → Cloud Services (Parse/Optimize)
                        ↓
        Query Plan Generation (Metadata Pruning)
                        ↓
    Warehouse Assignment (Auto-Resume if Suspended)
                        ↓
    Parallel Execution (Micro-Partition Scanning)
                        ↓
        Result Caching (24-hour TTL)
                        ↓
    Return Results to Client
```

---

## Joins

Snowflake supports inner, outer, cross, and lateral joins with automatic optimization for large datasets. Join elimination and pushdown filters improve performance.

### Join Types & Syntax

```sql
-- INNER JOIN (matching rows only)
SELECT o.order_id, c.customer_name, o.total_amount
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;

-- LEFT OUTER JOIN (all left rows, nulls for unmatched right)
SELECT c.customer_id, c.customer_name, COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- RIGHT OUTER JOIN (all right rows, nulls for unmatched left)
SELECT o.order_id, c.customer_name
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.customer_id;

-- FULL OUTER JOIN (all rows from both tables)
SELECT 
  COALESCE(c.customer_id, o.customer_id) AS customer_id,
  c.customer_name,
  o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id;

-- CROSS JOIN (Cartesian product)
SELECT d.date, p.product_id
FROM dim_date d
CROSS JOIN dim_product p
WHERE d.year = 2025;

-- LATERAL JOIN (correlated subquery per row)
SELECT c.customer_id, recent_orders.order_id
FROM customers c,
LATERAL (
  SELECT order_id, order_date
  FROM orders
  WHERE customer_id = c.customer_id
  ORDER BY order_date DESC
  LIMIT 5
) recent_orders;
```

### Join Optimization

```sql
-- Anti-pattern: Large table on left, small on right
SELECT o.*
FROM orders o  -- 1B rows
LEFT JOIN small_lookup l ON o.lookup_key = l.key  -- 1K rows

-- Pattern: Broadcast small table (automatic in Snowflake)
SELECT o.*
FROM orders o
LEFT JOIN small_lookup l ON o.lookup_key = l.key;
-- Snowflake auto-broadcasts small_lookup to all compute nodes

-- Pushdown filter (filter before join)
SELECT o.order_id, c.customer_name
FROM orders o
INNER JOIN (
  SELECT customer_id, customer_name
  FROM customers
  WHERE country = 'US'  -- Filter reduces join size
) c ON o.customer_id = c.customer_id;
```

---

## Window Functions

Window functions perform calculations across row sets without collapsing results (unlike GROUP BY). Support partitioning, ordering, and frame specifications.

### Syntax: Window Functions

```sql
<function_name> ( [ <expression> ] )
  OVER (
    [ PARTITION BY <expr> [ , ... ] ]
    [ ORDER BY <expr> [ ASC | DESC ] [ , ... ] ]
    [ { ROWS | RANGE } BETWEEN <frame_start> AND <frame_end> ]
  )
```

### Window Frame Specification

| Frame Type | Description | Example |
|------------|-------------|---------|
| `ROWS` | Physical row offset | `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` |
| `RANGE` | Logical value range | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |
| `UNBOUNDED PRECEDING` | First row of partition | Start of window |
| `CURRENT ROW` | Current row | Current position |
| `N PRECEDING` | N rows before current | `3 PRECEDING` |
| `N FOLLOWING` | N rows after current | `1 FOLLOWING` |
| `UNBOUNDED FOLLOWING` | Last row of partition | End of window |

### Ranking Functions

```sql
-- ROW_NUMBER: Unique sequential number per partition
SELECT 
  customer_id,
  order_date,
  total_amount,
  ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
FROM orders;

-- RANK: Rank with gaps for ties
SELECT 
  product_id,
  sales_amount,
  RANK() OVER (ORDER BY sales_amount DESC) AS rank
FROM product_sales;

-- DENSE_RANK: Rank without gaps
SELECT 
  product_id,
  sales_amount,
  DENSE_RANK() OVER (ORDER BY sales_amount DESC) AS dense_rank
FROM product_sales;

-- NTILE: Divide rows into N buckets
SELECT 
  customer_id,
  total_purchases,
  NTILE(4) OVER (ORDER BY total_purchases DESC) AS quartile
FROM customer_metrics;
```

### Aggregate Window Functions

```sql
-- Running total
SELECT 
  order_date,
  daily_revenue,
  SUM(daily_revenue) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM daily_sales;

-- Moving average (7-day)
SELECT 
  order_date,
  daily_revenue,
  AVG(daily_revenue) OVER (
    ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7d
FROM daily_sales;

-- Cumulative percentage
SELECT 
  product_id,
  revenue,
  SUM(revenue) OVER (ORDER BY revenue DESC) / 
    SUM(revenue) OVER () AS cumulative_pct
FROM product_revenue;
```

### Value Functions

```sql
-- LAG: Previous row value
SELECT 
  order_date,
  revenue,
  LAG(revenue, 1) OVER (ORDER BY order_date) AS prev_day_revenue,
  revenue - LAG(revenue, 1) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_sales;

-- LEAD: Next row value
SELECT 
  order_date,
  revenue,
  LEAD(revenue, 1) OVER (ORDER BY order_date) AS next_day_revenue
FROM daily_sales;

-- FIRST_VALUE / LAST_VALUE
SELECT 
  customer_id,
  order_date,
  total_amount,
  FIRST_VALUE(order_date) OVER (
    PARTITION BY customer_id ORDER BY order_date
  ) AS first_order_date,
  LAST_VALUE(order_date) OVER (
    PARTITION BY customer_id 
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS last_order_date
FROM orders;
```

### Advanced Example: Customer Cohort Analysis

```sql
WITH customer_orders AS (
  SELECT 
    customer_id,
    order_date,
    total_amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_number,
    FIRST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS cohort_date,
    SUM(total_amount) OVER (
      PARTITION BY customer_id 
      ORDER BY order_date 
      ROWS UNBOUNDED PRECEDING
    ) AS lifetime_value
  FROM orders
)
SELECT 
  DATE_TRUNC('month', cohort_date) AS cohort_month,
  order_number,
  COUNT(DISTINCT customer_id) AS customers,
  AVG(lifetime_value) AS avg_ltv,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY lifetime_value) AS median_ltv
FROM customer_orders
GROUP BY cohort_month, order_number
ORDER BY cohort_month, order_number;
```

---

## Semi-Structured Data

Snowflake natively handles JSON, Avro, Parquet, ORC, and XML in VARIANT columns. No pre-processing or schema definition required.

### VARIANT Data Type

```sql
-- Create table with VARIANT column
CREATE TABLE events (
  event_id INT,
  event_timestamp TIMESTAMP_NTZ,
  event_data VARIANT  -- Stores JSON/Avro/Parquet
);

-- Load JSON data
COPY INTO events
FROM @json_stage
FILE_FORMAT = (TYPE = JSON);

-- Query JSON fields (dot notation)
SELECT 
  event_id,
  event_data:user_id::INT AS user_id,
  event_data:event_type::STRING AS event_type,
  event_data:metadata:ip_address::STRING AS ip_address
FROM events;

-- Array access (zero-indexed)
SELECT 
  event_data:product_ids[0]::INT AS first_product_id,
  event_data:product_ids[1]::INT AS second_product_id
FROM events;
```

### Semi-Structured Functions

#### FLATTEN (Array Expansion)

```sql
-- FLATTEN: Explode array into rows
SELECT 
  event_id,
  value::INT AS product_id
FROM events,
LATERAL FLATTEN(INPUT => event_data:product_ids);

-- Nested FLATTEN (nested arrays)
SELECT 
  event_id,
  nested_value:category::STRING AS category,
  nested_value:price::DECIMAL(10,2) AS price
FROM events,
LATERAL FLATTEN(INPUT => event_data:products) AS outer_flatten,
LATERAL FLATTEN(INPUT => value:variants) AS nested_value;
```

#### GET_PATH / GET

```sql
-- GET_PATH: Extract value by path
SELECT 
  event_id,
  GET_PATH(event_data, 'user.profile.age')::INT AS age
FROM events;

-- GET: Extract value by key
SELECT 
  GET(event_data, 'event_type')::STRING AS event_type
FROM events;
```

#### PARSE_JSON / TRY_PARSE_JSON

```sql
-- PARSE_JSON: String to VARIANT
SELECT 
  PARSE_JSON('{"name": "John", "age": 30}') AS parsed_json;

-- TRY_PARSE_JSON: Null on error (no exception)
SELECT 
  TRY_PARSE_JSON(invalid_json_column) AS parsed
FROM raw_data;
```

#### OBJECT_CONSTRUCT / ARRAY_CONSTRUCT

```sql
-- OBJECT_CONSTRUCT: Build JSON object
SELECT 
  OBJECT_CONSTRUCT(
    'customer_id', customer_id,
    'full_name', first_name || ' ' || last_name,
    'orders', order_count
  ) AS customer_json
FROM customers;

-- ARRAY_CONSTRUCT: Build JSON array
SELECT 
  customer_id,
  ARRAY_CONSTRUCT(order_id_1, order_id_2, order_id_3) AS order_ids
FROM customer_orders;

-- ARRAY_AGG: Aggregate into array
SELECT 
  customer_id,
  ARRAY_AGG(order_id) AS all_order_ids
FROM orders
GROUP BY customer_id;
```

### Advanced Example: E-Commerce Event Analytics

```sql
-- Raw JSON events
CREATE TABLE ecommerce_events (
  event_id INT,
  event_timestamp TIMESTAMP_NTZ,
  raw_event VARIANT
);

-- Sample data
INSERT INTO ecommerce_events
SELECT 
  1,
  '2025-01-31 10:00:00',
  PARSE_JSON('{
    "user_id": 12345,
    "event_type": "purchase",
    "cart": {
      "items": [
        {"product_id": 101, "quantity": 2, "price": 29.99},
        {"product_id": 205, "quantity": 1, "price": 49.99}
      ],
      "total": 109.97,
      "currency": "USD"
    },
    "metadata": {
      "device": "mobile",
      "country": "US"
    }
  }');

-- Flatten cart items
SELECT 
  event_id,
  raw_event:user_id::INT AS user_id,
  raw_event:event_type::STRING AS event_type,
  item.value:product_id::INT AS product_id,
  item.value:quantity::INT AS quantity,
  item.value:price::DECIMAL(10,2) AS price,
  item.value:quantity::INT * item.value:price::DECIMAL(10,2) AS line_total,
  raw_event:cart.total::DECIMAL(10,2) AS cart_total,
  raw_event:metadata.device::STRING AS device,
  raw_event:metadata.country::STRING AS country
FROM ecommerce_events,
LATERAL FLATTEN(INPUT => raw_event:cart.items) AS item;

-- Aggregate metrics
SELECT 
  raw_event:metadata.country::STRING AS country,
  COUNT(*) AS purchase_count,
  SUM(raw_event:cart.total::DECIMAL(10,2)) AS total_revenue,
  AVG(raw_event:cart.total::DECIMAL(10,2)) AS avg_order_value
FROM ecommerce_events
WHERE raw_event:event_type::STRING = 'purchase'
GROUP BY country;
```

---

## Query Optimization

### Micro-Partition Pruning

Snowflake uses metadata (min/max values, null counts) to skip irrelevant micro-partitions.

```sql
-- Good: Filter on clustered column (prunes partitions)
SELECT * FROM orders
WHERE order_date = '2025-01-31';
-- Scans: 10 partitions (out of 1000)

-- Bad: Filter on non-clustered column (full scan)
SELECT * FROM orders
WHERE customer_email = 'john@example.com';
-- Scans: 1000 partitions (no pruning)

-- Fix: Add clustering key
ALTER TABLE orders CLUSTER BY (customer_email);
-- Now scans: 5 partitions
```

### Clustering Optimization

```sql
-- Check clustering depth (lower = better)
SELECT SYSTEM$CLUSTERING_DEPTH('orders', '(order_date)');
-- Returns: {"totalDepth": 2, "average": 1.1}

-- Check clustering info
SELECT SYSTEM$CLUSTERING_INFORMATION('orders', '(order_date, region)');

-- Manual recluster (if automatic is slow)
ALTER TABLE orders RECLUSTER MAX_SIZE = 10000;

-- Suspend automatic reclustering (cost control)
ALTER TABLE orders SUSPEND RECLUSTER;
```

### Query Profile Analysis

```sql
-- Enable query profiling
ALTER SESSION SET USE_CACHED_RESULT = FALSE;

-- Run query
SELECT ...;

-- View query profile (Web UI: Query History → Query ID → Profile tab)
-- Analyze:
-- - Partition pruning effectiveness
-- - Join distribution (broadcast vs hash)
-- - Spillage to remote storage
-- - Cache hits
```

---

## Caching

Snowflake uses three cache layers: metadata cache, result cache, and warehouse cache.

### Cache Types

| Cache Type | Scope | Duration | Use Case |
|------------|-------|----------|----------|
| **Metadata Cache** | Account-wide | Persistent | Micro-partition metadata (min/max values) |
| **Result Cache** | Account-wide | 24 hours | Identical query results |
| **Warehouse Cache** | Warehouse-specific | Until warehouse suspended | Raw data cached in local SSD |

### Result Cache

```sql
-- First execution (cache miss)
SELECT COUNT(*) FROM orders;
-- Execution time: 2.5 seconds

-- Second identical execution (cache hit)
SELECT COUNT(*) FROM orders;
-- Execution time: 0.1 seconds (result cache)

-- Disable result cache for testing
ALTER SESSION SET USE_CACHED_RESULT = FALSE;

-- Result cache invalidated if:
-- - Table data changes (DML)
-- - 24 hours elapsed
-- - Different user privileges (different result set)
```

### Warehouse Cache (Local Disk)

```sql
-- First query (cache miss, reads from storage)
SELECT AVG(total_amount) FROM orders WHERE order_date >= '2025-01-01';
-- Execution time: 5 seconds, Partitions scanned: 100

-- Similar query (cache hit, reads from local SSD)
SELECT SUM(total_amount) FROM orders WHERE order_date >= '2025-01-01';
-- Execution time: 1 second (warehouse cache)

-- Cache cleared when:
-- - Warehouse suspended
-- - Warehouse resized
-- - Different warehouse used
```

### Cache Optimization Strategies

```sql
-- Pattern: Dedicated warehouse for BI dashboards (warm cache)
CREATE WAREHOUSE bi_dashboard_wh
  WAREHOUSE_SIZE = MEDIUM
  AUTO_SUSPEND = 600  -- Keep warm for 10 minutes
  AUTO_RESUME = TRUE;

-- Pattern: Suspend development warehouse (cold cache acceptable)
CREATE WAREHOUSE dev_wh
  AUTO_SUSPEND = 60  -- Aggressive suspension
  AUTO_RESUME = TRUE;

-- Pattern: Share result cache across users
-- (Same query, same privileges, within 24 hours)
-- No additional configuration needed (automatic)
```

---

## SDLC Use Case: QA Performance Benchmarking

### Scenario: Sprint Testing with Performance Metrics

```python
# qa_benchmark.py (pytest framework)
import snowflake.connector
import time
from contextlib import contextmanager

class PerformanceBenchmark:
    def __init__(self, account, user, password, warehouse):
        self.conn = snowflake.connector.connect(
            account=account,
            user=user,
            password=password,
            warehouse=warehouse,
            role='QA_ROLE'
        )
        self.cursor = self.conn.cursor()
    
    @contextmanager
    def measure_query(self, query_name):
        """Context manager for query performance measurement"""
        # Disable caches for accurate measurement
        self.cursor.execute("ALTER SESSION SET USE_CACHED_RESULT = FALSE")
        
        start_time = time.time()
        yield
        execution_time = time.time() - start_time
        
        # Get query stats
        self.cursor.execute("""
            SELECT 
              query_id,
              execution_time,
              bytes_scanned,
              bytes_written,
              partitions_scanned,
              partitions_total
            FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
            WHERE query_id = LAST_QUERY_ID()
        """)
        
        stats = self.cursor.fetchone()
        
        print(f"""
        Query: {query_name}
        Execution Time: {execution_time:.2f}s
        Warehouse Time: {stats[1] / 1000:.2f}s
        Bytes Scanned: {stats[2] / 1e9:.2f} GB
        Partitions Scanned: {stats[4]} / {stats[5]}
        Partition Pruning: {(1 - stats[4]/stats[5]) * 100:.1f}%
        """)
        
        # Performance assertions
        assert execution_time < 10, f"Query timeout: {execution_time}s"
        assert stats[4]/stats[5] < 0.1, "Poor partition pruning"
    
    def test_date_filter_performance(self):
        """Test clustered column query performance"""
        with self.measure_query("Date Filter Test"):
            self.cursor.execute("""
                SELECT COUNT(*), SUM(total_amount)
                FROM prod.sales.fact_orders
                WHERE order_date = '2025-01-30'
            """)
            result = self.cursor.fetchone()
        
        assert result[0] > 0, "No data found"
    
    def test_aggregation_performance(self):
        """Test window function performance"""
        with self.measure_query("Window Function Test"):
            self.cursor.execute("""
                SELECT 
                  customer_id,
                  order_date,
                  total_amount,
                  AVG(total_amount) OVER (
                    PARTITION BY customer_id 
                    ORDER BY order_date 
                    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
                  ) AS moving_avg
                FROM prod.sales.fact_orders
                WHERE order_date >= '2025-01-01'
                LIMIT 10000
            """)
    
    def test_semi_structured_performance(self):
        """Test VARIANT query performance"""
        with self.measure_query("JSON Query Test"):
            self.cursor.execute("""
                SELECT 
                  event_id,
                  event_data:user_id::INT AS user_id,
                  COUNT(*)
                FROM prod.events.user_events
                WHERE event_data:event_type::STRING = 'purchase'
                  AND event_timestamp >= '2025-01-30'
                GROUP BY event_id, user_id
                LIMIT 1000
            """)

# Run benchmarks
if __name__ == '__main__':
    bench = PerformanceBenchmark(
        account='myorg.us-east-1',
        user='qa_user',
        password='***',
        warehouse='qa_benchmark_wh'
    )
    
    bench.test_date_filter_performance()
    bench.test_aggregation_performance()
    bench.test_semi_structured_performance()
```

---

## Key Concepts

- **Window Function**: Calculation across row sets without collapsing results (ROW_NUMBER, RANK, LAG, etc.)
- **VARIANT**: Data type for semi-structured data (JSON, Avro, Parquet) with native querying
- **FLATTEN**: Function to expand arrays/objects into rows for analysis
- **Partition Pruning**: Skipping micro-partitions based on metadata (improves query performance)
- **Result Cache**: 24-hour cache for identical queries (account-wide, automatic)
- **Warehouse Cache**: Local SSD cache for recently accessed data (warehouse-specific)

---
