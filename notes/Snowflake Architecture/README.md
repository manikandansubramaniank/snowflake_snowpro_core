# Snowflake Architecture and Concepts

## 📘 Table of Contents

1. [Traditional Database Architectures](#1-traditional-database-architectures)  
   - [Shared Disk Architecture](#shared-disk-architecture)  
   - [Share Nothing Architecture](#share-nothing-architecture)  
   - [Multi-Cluster Shared Data Architecture](#snowflake's-multi-cluster-shared-data-architecture)

2. [Snowflake’s Three Architecture Layers](#2-snowflake-s-three-architecture-layers)  
   - [Storage Layer Details](#storage-layer-details)  
   - [Compute Warehouses](#compute-warehouses)  
   - [Cloud Services Overview](#cloud-services-overview)  
   - [How It Works Together](#how-it-works-together)  
   - [Key Benefits](#key-benefits)

3. [Loading Data into Snowflake](#3-loading-data-into-snowflake)

4. [Snowflake Editions and Features](#4-snowflake-editions-and-features)  
   - [Standard Edition](#standard-edition)  
   - [Enterprise Edition](#enterprise-edition)  
   - [Business Critical Edition](#business-critical-edition-formerly-enterprise-for-sensitive-data)  
   - [Virtual Private Snowflake (VPS)](#virtual-private-snowflake-vps)

5. [Pricing Components](#5-pricing-components)  
   - [Compute Cost](#compute-cost)  
   - [Cloud Services Cost](#cloud-services-cost)  
   - [Storage Cost](#storage-cost)  
   - [Data Transfer Cost](#data-transfer-cost)

6. [Storage Monitoring](#6-storage-monitoring)

7. [Resource Monitors](#7-resource-monitors)

8. [Warehouse Scaling](#8-warehouse-scaling)  
   - [Warehouse Types](#types)  
   - [Warehouse Sizes](#size)  
   - [Multi-Cluster Warehouse](#multi-cluster-warehouse)  
   - [Scaling Policy](#scaling-policy)

9. [Object Hierarchy in Snowflake](#9-object-hierarchy-in-snowflake)

10. [SnowSQL Command Line Tool](#10-snowsql-command-line-tool)


## 1. Traditional Database Architectures

### Shared Disk Architecture

The Shared Disk Architecture is a type of database or computing architecture where multiple nodes (servers) share access to the same physical storage (disk), but each node has its own private memory and CPU.

### Key Components
1. **Multiple Compute Nodes:**
- Each node has its own CPU and memory.
- Nodes can process queries independently.
2. **Shared Storage:**
- All nodes access the same disk subsystem (e.g., SAN or NAS).
- Data consistency is managed through a coordination mechanism.
3. **Cluster Interconnect:**
- High-speed network connecting the nodes.
- Used for communication and coordination (e.g., locking, cache coherence).

### Advantages ✅
- **High availability:** If one node fails, others can continue.
- **Scalability:** Can add more nodes to improve performance.
- **Data consistency:** Centralized storage simplifies data management.
### Disadvantages ❌
- **Coordination overhead:** Locking and cache coherence can become bottlenecks.
- **Storage I/O contention:** All nodes accessing the same disk can lead to performance issues.
- **Complexity:** Requires sophisticated software to manage concurrency and consistency.

![Shared Disk vs Shared Nothing](https://github.com/manikandansubramaniank/snowflake_snowpro_core/blob/main/notes/Snowflake%20Architecture/shared-disk_vs_share-nothing.png) 

### Share Nothing Architecture

The Shared-Nothing Architecture is a distributed computing model where each node in the system is independent and self-sufficient, with no shared memory or disk.

### Key Characteristics
1. **Independent Nodes:**
- Each node has its own CPU, memory, and disk.
- No resource sharing between nodes.
2. **Data Partitioning:**
- Data is distributed across nodes.
- Each node processes its own portion of data.
3. **High Scalability:**
- Easily add more nodes to increase capacity.
- No bottlenecks from shared resources.
4. **Fault Isolation:**
- Failure in one node doesn’t affect others.
- Easier to maintain and recover.

### Advantages ✅
- **Scalability:** Ideal for big data and cloud-native systems.
- **Performance:** No contention for shared resources.
- **Resilience:** Faults are isolated to individual nodes.
### Disadvantages ❌
- **Complexity:** Requires sophisticated coordination and data distribution.
- **Data Movement:** Joins across nodes can be expensive.
- **Consistency:** Ensuring consistency across nodes can be challenging.

###  Snowflake's Multi-Cluster Shared Data Architecture:
- Combines centralized persistent storage with multiple independent compute clusters caching data locally for strong performance and scalability.

![Multi-Cluster Shared Data Architecture](https://github.com/manikandansubramaniank/snowflake_snowpro_core/blob/main/notes/Snowflake%20Architecture/multi_cluster_shared_disk.png)  

---

## 2. Snowflake’s Three Architecture Layers

The [Snowflake Architecture](https://docs.snowflake.com/en/user-guide/intro-key-concepts#snowflake-architecture) is a modern, cloud-native data platform design that separates compute, storage, and services layers. This separation allows for independent scaling, high concurrency, and efficient data sharing.

![Snowflake Architecture](https://github.com/manikandansubramaniank/snowflake_snowpro_core/blob/main/notes/Snowflake%20Architecture/architecture-overview.png)

1. **[Storage Layer Details](https://docs.snowflake.com/en/user-guide/intro-key-concepts#database-storage)**

- Stores structured and semi-structured data (e.g., JSON, Avro, Parquet).
- Automatically compressed, encrypted, and distributed across cloud storage (AWS S3, Azure Blob, or GCP).
- Fully managed by Snowflake—users don’t manage indexes or partitions.

2. **[Compute Warehouses](https://docs.snowflake.com/en/user-guide/intro-key-concepts#query-processing)**

- Independent compute clusters called Virtual Warehouses.
- Each warehouse can access the same data without contention.
- Supports multi-cluster warehouses for high concurrency.
- Warehouses can be paused/resumed and scaled independently.

3. **[Cloud Services Overview](https://docs.snowflake.com/en/user-guide/intro-key-concepts#cloud-services)**

- Manages metadata, query parsing, optimization, access control, and transactions.
- Acts as the brain of the platform.
- Enables features like:
    - Query optimization
    - Security and governance
    - Data sharing and replication

### How It Works Together
- A user submits a query.
- The Cloud Services Layer parses and optimizes it.
- A Virtual Warehouse (Compute Layer) executes the query.
- The Storage Layer provides the required data.
- Results are returned to the user.

### Key Benefits
- Separation of compute and storage for flexible scaling.
- Concurrency without contention using multi-cluster compute.
- Secure data sharing without data duplication.
- Support for semi-structured data natively.
- Cross-cloud and cross-region capabilities.

---

## 3. Loading Data into Snowflake

```sql

// Create database
CREATE DATABASE IF NOT EXISTS MANI_SN_DB;

USE DATABASE MANI_SN_DB;

CREATE SCHEMA IF NOT EXISTS SN_SCH_MA;

USE SCHEMA SN_SCH_MA;

// Create the table + meta data
CREATE OR REPLACE TABLE "MANI_SN_DB"."SN_SCH_MA"."LOAN_PAYMENT" (
  "Loan_ID" STRING,
  "loan_status" STRING,
  "Principal" STRING,
  "terms" STRING,
  "effective_date" STRING,
  "due_date" STRING,
  "paid_off_time" STRING,
  "past_due_days" STRING,
  "age" STRING,
  "education" STRING,
  "Gender" STRING);
  
  
 SELECT * FROM MANI_SN_DB.SN_SCH_MA.LOAN_PAYMENT;


 // Loading the data from S3 bucket
  
 COPY INTO LOAN_PAYMENT
    FROM s3://bucketsnowflakes3/Loan_payments_data.csv
    file_format = (type = csv 
                   field_delimiter = ',' 
                   skip_header=1);

```

---

## 4. Snowflake Editions and Features

### Snowflake Edition

[Snowflake Edition](https://docs.snowflake.com/en/user-guide/intro-editions)

Snowflake offers four main editions, each designed to meet different organizational needs, with increasing levels of features, security, and support:

1. **Standard Edition**

**Target:** Small to medium businesses or teams starting with Snowflake.
**Features:**
- Full SQL support
- Time Travel (1 day)
- Automatic encryption
- Basic security and governance
- Native support for semi-structured data (JSON, Avro, Parquet, etc.).

2. **Enterprise Edition**

**Target:** Larger organizations needing more advanced features.
- Includes all Standard features, plus:
- Time Travel up to 90 days
- Materialized views
- Multi-cluster warehouses
- Resource monitors
- Custom roles and access policies

3. **Business Critical Edition** (formerly Enterprise for Sensitive Data)

**Target:** Organizations with sensitive or regulated data (e.g., healthcare, finance).

- Includes all Enterprise features, plus:
- HIPAA and HITRUST compliance
- Enhanced encryption (Tri-Secret Secure)
- Network policies and private connectivity
- Account failover/failback for disaster recovery

4. **Virtual Private Snowflake (VPS)**

**Target:** Organizations with the highest security requirements (e.g., government, financial institutions).

- Includes all Business Critical features, plus:
- Dedicated Snowflake environment
- No shared infrastructure
- Isolated metadata and compute resources


| Edition               | Key Use Case                    | Features                                              |
|-----------------------|---------------------------------|-------------------------------------------------------|
| Standard              | Foundational                    | Encryption, 1-day time travel, disaster recovery      |
| Enterprise            | Large enterprises               | Multi-cluster warehouses, 90-day time travel          |
| Business Critical     | Compliance & regulation         | Enhanced security, failover, customer-managed keys    |
| Virtual Private       | Highest isolation & security    | Dedicated hardware and metadata                       |

---

## 5. Pricing Components

- **Compute:** Charged per second based on warehouse size and usage.
- **Storage:** Monthly rate for compressed data stored.
- **Data Transfer:** Free ingress; egress costs vary by region/cloud.

### Compute Cost

[Compute cost](https://docs.snowflake.com/en/user-guide/cost-exploring-compute)
Snowflake charges for compute using credits, which are consumed based on the size and duration of your virtual warehouse usage. Here's a breakdown of how it works:

Virtual Warehouse Sizes and Credit Usage
|Warehouse Size	|Compute Nodes|Credits per Hour|
|:--------------|:------------|:---------------|
|X-Small (XS)	|1            |1               |
|Small (S)		|2            |2               |
|Medium (M)		|4            |4               |
|Large (L)		|8            |8               |
|X-Large (XL)	|16           |16              |
|2X-Large (2XL)	|32           |32              |
|3X-Large (3XL)	|64           |64              |
|4X-Large (4XL)	|128          |128             |
|5X-Large (5XL)	|256          |256             |
|6X-Large (6XL)	|512          |512             |

Billing is per second, with a 60-second minimum per session

The price per credit depends on your Snowflake Edition and region:

Edition	Typical Credit Cost (US)
|Edition					|Cost(USD)	  |
|:--------------------------|:------------|
|Standard					|2.00 – 3.10|
|Enterprise					|3.00 – 4.65|
|Business Critical			|4.00 – 6.20|
|Virtual Private Snowflake	|6.00 – 9.30|

Prices vary by cloud provider (AWS, Azure, GCP) and region (e.g., US vs. Europe)

### Cloud Services cost

Snowflake uses a fair-use policy for cloud services:

- Free up to 10% of your daily compute credit usage.
- If your cloud services usage exceeds **10%**, the excess is billed at the same rate as compute credits.
- For most users, cloud services charges are minimal or free.
- Heavy use of features like cloning, metadata-heavy queries, or frequent DDL operations may increase cloud services usage.
- Monitoring this usage helps avoid unexpected costs.

### Storage Cost

Snowflake charges for storage based on the amount of data stored per month, measured in terabytes (TB). The cost depends on your cloud provider, region, and whether you're on an On Demand or Capacity plan.

1. **On Demand Pricing**
	- Pay-as-you-go model.
	- You are billed per second for compute and per TB per month for storage.
	- No upfront commitment.
	
	**Ideal for:**
	- Small teams or projects.
	- Irregular or unpredictable workloads.
	- Getting started with Snowflake.
	
	**Pros:** ✅ 
	- Flexible.
	- No long-term commitment.
	- Easy to manage.
	
	**Cons:** ❌ 
	- Higher per-credit cost
	- No volume discounts

2. **Capacity Pricing**
	- You pre-purchase credits (e.g., 100,000 credits for 1 year).
	- You get discounted rates compared to On Demand.
	
	**Ideal for:**
	- Large or predictable workloads.
	- Enterprises with consistent usage.
	- Budget planning and cost control.
	
	**Pros:** ✅ 
	- Lower cost per credit.
	- Predictable billing.
	- Better suited for enterprise-scale usage.
	
	**Cons:** ❌ 
	- Requires upfront c## 7. Resource Monitorsommitment
	- Unused credits may expire

|Feature		|On Demand		|Capacity Plan		 |
|:--------------|:--------------|:-------------------|
|Billing		|Per second		|Prepaid credits	 |
|Commitment		|None			|1–3 years typically |
|Cost per Credit|Higher			|Lower (discounted)  |
|Best For		|Flexibility	|Cost efficiency	 |


### Data Transfer cost

Snowflake charges data transfer (egress) fees when data is moved out of your Snowflake account to another region, cloud provider, or external system.

|Scenario											|Cost?		|
|:--------------------------------------------------|:----------|
|Within same region & cloud provider				|❌ Free	|
|Between regions (same cloud)						|✅ Yes		|
|Between cloud providers (e.g., AWS → Azure)		|✅ Yes		|
|To external systems (e.g., S3, GCS, on-prem)		|✅ Yes		|
|Within same account, same region					|❌ Free	|


---

## 6. Storage Monitoring

[Monitoring storage](https://docs.snowflake.com/en/user-guide/cost-exploring-data-storage) in Snowflake is essential for managing costs, optimizing performance, and ensuring compliance.

Snowflake storage is categorized into:

* **Data Storage**: For tables and schemas.
* **Stage Storage**: For files in internal stages (e.g., @my_stage).
* **Failsafe Storage**: For disaster recovery (7-day retention, not user-accessible).
* **Time Travel Storage**: For historical data (up to 90 days depending on edition).


#### Monitoring Storage Usage

Snowflake provides system views in the SNOWFLAKE.ACCOUNT_USAGE and INFORMATION_SCHEMA schemas.

```sql
-- SNOWFLAKE DB
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS;

-- CREATED_DB
SELECT * FROM DB_NAME.INFORMATION_SCHEMA.TABLE_STORAGE_METRICS;
```

---

## 7. Resource Monitors

[Resource Monitors](https://docs.snowflake.com/en/user-guide/resource-monitors) in Snowflake are tools used to track and control compute resource usage, especially to manage credit consumption and avoid unexpected costs.

Resource Monitors Do:
* Set credit limits for virtual warehouses.
* Monitor usage against those limits.
* Trigger notifications or actions (like suspending warehouses) when thresholds are reached.

**Key Components of a Resource Monitor**

|Component					|Description															|
|:--------------------------|:----------------------------------------------------------------------|
|Credit Quota				|Total number of credits allowed for the monitor.                       |
|Thresholds					|Percentage points (e.g., 50%, 75%, 100%) to trigger actions or alerts. |
|Actions					|Notify, Suspend, or Suspend Immediately when thresholds are hit.       |
|Assigned Warehouses		|Specific warehouses the monitor applies to.                            |

**Creating a Resource Monitor (SQL Example)**

```sql
CREATE RESOURCE MONITOR MY_MONITOR
  WITH CREDIT_QUOTA = 100
  TRIGGERS ON 50 PERCENT DO NOTIFY
           ON 75 PERCENT DO SUSPEND
           ON 100 PERCENT DO SUSPEND IMMEDIATE;

ALTER WAREHOUSE MY_WAREHOUSE SET RESOURCE_MONITOR = MY_MONITOR;
```

**Or use the Snowflake UI:**

Go to **Admin > Resource Monitors** to view and manage monitors.

![Resource Monitor Workflow](https://github.com/manikandansubramaniank/snowflake_snowpro_core/blob/main/notes/Snowflake%20Architecture/snowflake-resource-monitors.png)  

---

## 8. Warehouse Scaling

In Snowflake, a [Virtual Warehouse](https://docs.snowflake.com/en/user-guide/warehouses) is a virtual compute engine used to execute SQL queries, load data, and perform other operations. It’s a key part of Snowflake’s multi-cluster shared data architecture, separating storage from compute.

Warehouse can be created using ACCOUNTADMIN, SECURITYADMIN or SYSADMIN

**Types**

|Feature				|Standard Warehouse						|Snowpark-Optimized Warehouse								|
|:----------------------|:--------------------------------------|:----------------------------------------------------------|
|Purpose				|General-purpose compute (SQL, BI, ETL)	|Memory-intensive workloads (Snowpark, ML, UDFs)            |
|Memory per Node		|Standard memory						|Up to 16x or 64x more memory per node                      |
|Local Cache			|Standard cache							|10x larger local cache for faster repeated execution       |
|Resource Constraints	|Not configurable						|Configurable via RESOURCE_CONSTRAINT (e.g., MEMORY_16X_X86)|
|Startup Time			|Fast									|Slightly longer due to high memory provisioning            |
|Cost					|Lower (based on size and usage)		|Higher (due to enhanced memory and performance)            |
|Best For				|SQL queries, dashboards, light ETL		|Snowpark DataFrames, ML training/inference, large UDFs     |
|Multi-Cluster Support	|Yes									|Yes                                                        |
|Availability			|All Snowflake editions and clouds		|Available on AWS, Azure, GCP (some memory configs AWS-only)|

**Size**

|Warehouse size		|Standard (Credits/Hour)|Snowpark-Optimized (Credits/Hour)|
|:------------------|:----------------------|:--------------------------------|
|X-Small			|1             			|N/A                              |
|Small				|2             			|N/A                              |
|Medium				|4             			|6                                |
|Large				|8             			|12                               |
|X-Large			|16            			|24                               |
|2X-Large			|32            			|48                               |
|3X-Large			|64            			|96                               |
|4X-Large			|128           			|192                              |
|5X-Large			|256           			|384                              |
|6X-Large			|512           			|768                              |

**Multi Cluster Warehouse**

A [Multi-Cluster Warehouse](https://docs.snowflake.com/en/user-guide/warehouses-multicluster) is a Snowflake virtual warehouse that can scale horizontally by adding or removing clusters of compute nodes based on demand.

|Feature								|Description													|
|:--------------------------------------|:--------------------------------------------------------------|
|Concurrency Scaling					|Automatically adds clusters when query load increases.         |
|Min/Max Clusters						|You define the minimum and maximum number of clusters.         |
|Auto-Scaling							|Snowflake spins up/down clusters based on query queue length.  |
|Same Size Clusters						|All clusters are the same size (e.g., Large, X-Large).         |
|Shared Data Cache						|All clusters access the same underlying data.                  |

**Scaling Policy**

In Snowflake, the scaling policy of a warehouse determines how aggressively it adds or removes clusters in a multi-cluster warehouse setup. This affects both performance and cost efficiency.

|Policy		|Description																					|
|:----------|:----------------------------------------------------------------------------------------------|
|Standard	|Adds clusters quickly when queries are queued. Prioritizes performance.                        |
|Economy	|Adds clusters more conservatively. Prioritizes cost savings by avoiding unnecessary scaling.   |

**Sample Multi clustered Warehouse**

```sql
CREATE WAREHOUSE MY_WH
  WITH WAREHOUSE_SIZE = 'LARGE'
  WAREHOUSE_TYPE = 'STANDARD'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3
  SCALING_POLICY = 'ECONOMY';
```

---

## 9. Object Hierarchy in Snowflake

In Snowflake, an object is any named entity that you can create, manage, and use within your account. These objects are organized hierarchically and serve different purposes across data storage, compute, security, and governance.

![Object hierarchy](https://github.com/manikandansubramaniank/snowflake_snowpro_core/blob/main/notes/Snowflake%20Architecture/securable-objects-hierarchy.png)

**Database Objects**
These are used to store and manage data.

|Object								|Description												  |
|:----------------------------------|:------------------------------------------------------------|
|Database							|Top-level container for schemas and data.					  |
|Schema								|Logical grouping of tables, views, etc.                      |
|Table								|Stores structured data.                                      |
|View								|Virtual table based on a SQL query.                          |
|Materialized View					|Precomputed view for performance.                            |
|External Table						|References data in external storage (e.g., S3).              |
|Stage								|Temporary or permanent file storage (internal or external).  |
|File Format						|Defines how files are read/written (CSV, JSON, etc.).        |
|Sequence							|Generates unique numeric values.                             |
|Function / Procedure				|Custom logic using SQL or Snowpark (UDFs, stored procs).     |
																								
																								
**Compute Objects**            
                                                                  
|Object								|Description                                                  |
|:----------------------------------|:------------------------------------------------------------|
|Warehouse							|Virtual compute engine for running queries.                  |
|Resource Monitor					|Tracks and controls credit usage.                            |
|Task								|Automates SQL execution on a schedule or trigger.            |
|Pipe								|Enables continuous data ingestion via Snowpipe.              |
																								 
**Security & Access Control Objects**                                                            
|Object								|Description                                                  |
|:----------------------------------|:------------------------------------------------------------|
|Role								|Defines a set of privileges.                                 |
|User								|Represents a person or service accessing Snowflake.          |
|Masking Policy						|Controls how sensitive data is displayed.                    |
|Row Access Policy					|Controls row-level access to data.                           |
|Access Control Privileges			|Permissions granted to roles on objects.                     |
																								 
**Governance & Metadata Objects**                                                                
|Object								|Description                                                  |
|:----------------------------------|:------------------------------------------------------------|
|Tag								|Metadata label for classification and tracking.              |
|Policy								|Rules for data masking, access, etc.                         |
|Share								|Enables secure data sharing across accounts.                 |
|Stream								|Tracks changes (CDC) to a table for incremental processing.  |

---

## 10. SnowSQL Command Line Tool

- Command-line client for query execution and administration.
- Supports Windows, Linux, MacOS.
- Typical connection example:

```sql
snowsql -a <account_name> -u <username>
```

- Useful for scripting and loading/unloading data.
- Context switching with `USE` statements.

Guide: [SnowSQL User Guide](https://docs.snowflake.com/en/user-guide/snowsql)

---

