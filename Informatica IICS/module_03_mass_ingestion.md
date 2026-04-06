# Complete Beginner's Guide to Mass Ingestion in IICS

> **IICS Service:** Mass Ingestion
> **Difficulty:** Beginner
> **Estimated Reading Time:** 25 minutes
> **Prerequisites:** An IICS organization account (free trial is sufficient), a source database (e.g., Oracle, SQL Server, or MySQL) and a cloud target (e.g., Snowflake, Databricks, Amazon S3) accessible from your Secure Agent
> **Learning Objectives:**
> - Explain what Mass Ingestion is and when to use it instead of CDI Mappings
> - Distinguish between the two ingestion modes: Initial Load and Incremental Load
> - Configure a Mass Ingestion Job to replicate multiple tables from a source database to a cloud target in a single operation
> - Set up an Ingestion Task with scheduling and monitoring
> - Understand how change data capture (CDC) works at a conceptual level within Mass Ingestion
> - Troubleshoot common Mass Ingestion failures

---

## 🧭 What Is Mass Ingestion and Why Does It Exist?

**Mass Ingestion** (the IICS service designed to replicate large numbers of database tables from a source system to a cloud target in bulk, with minimal configuration per table) solves a very specific problem: **getting lots of data from "here" to "there" as fast and simply as possible.**

### The Problem Mass Ingestion Solves

Imagine you're migrating from an on-premises Oracle database to Snowflake. Your Oracle system has 200 tables. Using CDI Mappings, you'd need to:

1. Create 200 Connections? No — just 2 (one source, one target). But...
2. Create 200 Mappings (one per table)
3. Create 200 Mapping Tasks (one per Mapping)
4. Configure field mappings for all 200 tables
5. Manage 200 jobs in the Monitor

That's a huge amount of repetitive work for what is essentially the same operation: "copy this table from Oracle to Snowflake."

**Mass Ingestion replaces all of that with a single job.** You select your source, your target, check the tables you want, and run it. IICS automatically:
- Discovers the schema of each selected table
- Creates the target tables if they don't exist
- Maps fields automatically
- Loads all the data

> 💡 **Mass Ingestion vs. CDI Mappings — When to Use Which**
> - Use **Mass Ingestion** when you need to replicate tables as-is with no transformation — just get the data from source to target
> - Use **CDI Mappings** when you need to filter rows, join tables, compute new columns, or apply any business logic during the move
> - A common pattern: Use Mass Ingestion for the initial data landing (raw zone), then use CDI Mappings to transform the landed data into a refined format

---

## 🔄 Two Modes of Ingestion: Initial vs. Incremental

Mass Ingestion supports two fundamental modes, and understanding the difference is critical.

### Initial Load

**Initial Load** (a one-time, full copy of all rows in the selected source tables to the target) is used when:
- You're setting up replication for the first time
- You want to completely refresh the target with a fresh copy of the source
- The target tables don't exist yet (Mass Ingestion creates them)

Think of Initial Load as photocopying an entire filing cabinet — you get a complete duplicate.

### Incremental Load (Change Data Capture)

**Incremental Load** (an ongoing process that captures only the rows that have changed — inserted, updated, or deleted — since the last ingestion run and applies those changes to the target) is used after the Initial Load to keep the target in sync without re-copying everything.

> 💡 **What is Change Data Capture (CDC)?**
> **CDC** is a technique for identifying and capturing only the data that has changed in a source system since the last extraction. Instead of re-reading millions of rows, CDC reads only the new or modified rows. This is dramatically faster and less resource-intensive. Mass Ingestion supports CDC through database log-based methods (reading the database's transaction log) for supported source systems.

| Mode | What It Copies | When to Use | Performance Impact on Source |
|------|---------------|-------------|----------------------------|
| **Initial Load** | All rows in selected tables | First-time setup, full refresh | Higher — reads entire tables |
| **Incremental Load** | Only changed rows (inserts, updates, deletes) | Ongoing synchronization | Lower — reads only transaction logs |

### Typical Workflow

```
Day 1:  Run Initial Load (full copy of 200 tables → Snowflake)
Day 2+: Run Incremental Load on a schedule (every hour / every 15 minutes)
        Only changed rows are transferred
```

---

## 🏗️ Components of a Mass Ingestion Setup

Before creating a Mass Ingestion Job, these components must be in place:

| Component | What It Is | Where to Configure |
|-----------|-----------|-------------------|
| **Source Connection** | Connection to your source database (Oracle, SQL Server, MySQL, PostgreSQL, etc.) | **Administrator** → **Connections** |
| **Target Connection** | Connection to your cloud target (Snowflake, Databricks, Amazon S3, Azure Data Lake Storage, Google Cloud Storage) | **Administrator** → **Connections** |
| **Secure Agent** | The installed agent that reads from the source and writes to the target | **Administrator** → **Runtime Environments** |
| **Mass Ingestion Job** | The configuration that defines which tables to replicate and how | **Mass Ingestion** service in IICS |

> ⚠️ **Important:** Not all source/target combinations are supported for Mass Ingestion. Check the Informatica documentation for your specific combination. Common supported paths include: Oracle → Snowflake, SQL Server → Databricks, MySQL → Amazon S3, PostgreSQL → Azure Data Lake Storage.

---

## 🖱️ Step-by-Step: Creating a Mass Ingestion Job

### Choosing Your Ingestion Approach

When you create a Mass Ingestion Job, the first decision is the **Job Type**:

| Job Type | Description |
|----------|------------|
| **Database Ingestion** | Replicates tables from a relational database (Oracle, SQL Server, etc.) to a cloud target |
| **File Ingestion** | [VERIFY: confirm if "File Ingestion" is an actual Mass Ingestion job type or if file-based bulk ingestion uses a different mechanism] |

For this guide, we'll focus on **Database Ingestion**, which is the most common use case.

### Creating the Job

1. From the IICS home page, click **Mass Ingestion** in the left navigation
2. Click **New** → **Mass Ingestion Job** [VERIFY: confirm exact menu path — it may be "New" → "Database Ingestion and Replication" depending on IICS version]
3. Name it `mi_oracle_to_snowflake_full_replication`

### Configuring the Source

4. Under **Source**, select your **Source Connection** (e.g., your Oracle connection)
5. Choose the **Schema** — this is the database schema (namespace) containing the tables you want to replicate (e.g., `HR`, `SALES`)
6. IICS will display a list of all tables in that schema
7. **Select the tables** you want to replicate:
   - Check individual tables, or
   - Use **Select All** to replicate the entire schema
   - You can use filters or search to find specific tables

> 💡 **Table Selection Best Practice**
> Start with a small subset (3–5 tables) for your first Mass Ingestion Job. Verify they load correctly before scaling up to the full set. This makes troubleshooting much easier if something goes wrong.

### Configuring the Target

8. Under **Target**, select your **Target Connection** (e.g., your Snowflake connection)
9. Specify the **Target Schema** where the replicated tables will be created (e.g., `RAW_ORACLE`)
10. Configure target table options:
    - **Table Name Prefix/Suffix**: Optionally add a prefix (e.g., `SRC_`) to distinguish ingested tables from transformed tables
    - **If Table Exists**: Choose whether to drop and recreate, truncate and reload, or append

### Configuring Ingestion Mode

11. Select the **Ingestion Mode**:
    - **Initial Load**: For the first run — copies all data
    - **Incremental Load**: For ongoing sync — captures only changes
    - **Initial + Incremental**: Performs a full load first, then switches to incremental mode automatically

### Configuring the Runtime

12. Under **Runtime Environment**, select the Secure Agent group that has network access to both the source database and the cloud target
13. (Optional) Set **Scheduling**: configure a cron-style schedule for automatic execution

### Saving and Running

14. Click **Save**
15. Review the summary — confirm the table list, source, target, and mode
16. Click **Run**

---

## 📊 Monitoring Mass Ingestion Jobs

Mass Ingestion provides its own monitoring dashboard that shows more detail than the general IICS Monitor.

### Accessing the Dashboard

1. Navigate to **Mass Ingestion** → click on your job name
2. The dashboard shows:
   - **Overall Status**: Running, Completed, Failed, or Partially Completed
   - **Per-Table Status**: Each selected table has its own status with row counts
   - **Throughput**: Rows per second being processed
   - **Error Details**: If any table fails, click it to see the specific error

### Key Metrics to Watch

| Metric | What It Tells You |
|--------|------------------|
| **Source Rows Read** | Total rows read from the source table |
| **Target Rows Written** | Total rows successfully written to the target |
| **Rejected Rows** | Rows that failed to load (data type issues, constraint violations) |
| **Duration** | Elapsed time for the job or individual table |
| **Lag (Incremental mode)** | How far behind the target is compared to the source |

> ⚠️ **"Partially Completed" Status:** This means some tables succeeded and others failed. Don't re-run the entire job — check which tables failed, fix the issue (often a data type mismatch or permission problem), and re-run only those tables.

---

## 🔧 Configuration Options and Tuning

### Handling Schema Changes

What happens if someone adds a new column to a source table after you've set up the Mass Ingestion Job?

- **Initial Load mode**: The next full load will pick up the new column automatically (since it re-reads the full schema)
- **Incremental Load mode**: Behavior depends on configuration — some setups require you to **restart the ingestion** for that table to capture schema changes. [VERIFY: confirm exact behavior for schema evolution in Mass Ingestion incremental mode]

### Parallelism

Mass Ingestion can process multiple tables in parallel to improve overall throughput:

- The number of tables processed concurrently depends on the Secure Agent's resources (CPU, memory, network bandwidth)
- You can typically configure the **degree of parallelism** in the job settings [VERIFY: confirm exact setting name for parallelism in Mass Ingestion]
- For Initial Load of large tables, IICS may also parallelize the extraction of a single table by splitting it into partitions based on the primary key

### Data Type Mapping

When Mass Ingestion creates target tables, it automatically maps source data types to target data types. For example:

| Oracle Source Type | Snowflake Target Type |
|-------------------|----------------------|
| VARCHAR2(100) | VARCHAR(100) |
| NUMBER(10,2) | NUMBER(10,2) |
| DATE | TIMESTAMP_NTZ |
| CLOB | VARCHAR(16777216) |

In most cases, the automatic mapping is correct. However, large object types (CLOBs, BLOBs) may require special handling or may not be supported for all target types.

---

## ⚠️ Common Mistakes & Troubleshooting

| Mistake | Why It Happens | How to Fix It |
|---------|---------------|---------------|
| Initial Load succeeds but Incremental Load shows zero changes | CDC is not enabled on the source database, or the database's transaction log has been purged | Verify that supplemental logging (Oracle) or CDC is enabled (SQL Server) on the source database. Contact your DBA to ensure logs are retained long enough |
| Job fails with "table already exists" error | The target table was manually created with a different schema than what Mass Ingestion expects | Either drop the target table and let Mass Ingestion recreate it, or set the "If Table Exists" option to "Truncate and Reload" |
| Some tables load successfully but others fail with data type errors | The source table has data types not supported by the target (e.g., spatial data types, custom types) | Check the error log for the specific unsupported type. Exclude those tables from Mass Ingestion and use a CDI Mapping with explicit type conversion instead |
| Job runs very slowly for large tables | The source table has no primary key or index, forcing a full sequential scan | Ensure source tables have primary keys. For Initial Load, IICS uses the primary key to partition the table for parallel reads |
| Incremental Load misses some changes | The Secure Agent was offline during the change capture window, and the source database transaction logs were purged | Ensure the Secure Agent runs continuously for Incremental mode. Set log retention on the source database to at least 24–48 hours beyond your expected downtime |

---

## 📝 Key Takeaways

- **Mass Ingestion is for bulk table replication without transformation** — it's the fastest way to get many tables from a source database to a cloud target
- **Two modes serve different purposes:** Initial Load for the first full copy; Incremental Load (CDC) for ongoing synchronization of changes
- **Mass Ingestion creates target tables automatically** — you don't need to define schemas in the target before running
- **Start small (3–5 tables), verify, then scale** — this avoids diagnosing problems across hundreds of tables simultaneously
- **CDC requires source database configuration** — supplemental logging (Oracle) or CDC enabling (SQL Server) must be set up by a DBA before Incremental Load will work

---

## 🧪 Practice Exercise

**Scenario:** Your team needs to replicate three tables (`CUSTOMERS`, `ORDERS`, `PRODUCTS`) from a source database into a cloud data warehouse for analytics.

**Your Task:**
1. Verify your source database **Connection** works by clicking **Test Connection** in **Administrator** → **Connections**
2. Verify your target cloud warehouse **Connection** works the same way
3. Create a **Mass Ingestion Job** named `mi_exercise_3_tables`:
   - Source: your database connection → select the schema containing `CUSTOMERS`, `ORDERS`, `PRODUCTS`
   - Target: your cloud warehouse connection → specify a target schema (e.g., `RAW_ZONE`)
   - Mode: **Initial Load**
   - Select only the three tables listed above
4. **Save** and **Run** the job
5. Monitor the job dashboard — confirm all three tables show "Completed" status
6. Connect to your target cloud warehouse and run `SELECT COUNT(*) FROM RAW_ZONE.CUSTOMERS` (and for the other two tables) — verify the row counts match the source

**Expected Outcome:** All three tables appear in the target schema with matching row counts. Job status shows "Completed" for all three tables.

---

## 📚 Glossary

| Term | Definition |
|------|-----------|
| **Mass Ingestion** | The IICS service for replicating large numbers of database tables to a cloud target in bulk with minimal per-table configuration |
| **Initial Load** | A one-time full copy of all rows in selected source tables to the target |
| **Incremental Load** | An ongoing process that captures and applies only the changed rows (inserts, updates, deletes) to the target |
| **Change Data Capture (CDC)** | A technique for identifying and capturing only the data that has changed in a source system since the last extraction |
| **Transaction Log** | A database's internal record of all changes (inserts, updates, deletes) — CDC reads this log instead of re-scanning entire tables |
| **Supplemental Logging** | An Oracle database feature that must be enabled for CDC to capture sufficient column data for replication |
| **Schema** | A namespace within a database that groups related tables (e.g., the `HR` schema contains `EMPLOYEES`, `DEPARTMENTS`) |
| **Degree of Parallelism** | The number of tables (or partitions of a table) processed concurrently during ingestion |
| **Secure Agent** | A lightweight program installed on a machine that executes IICS jobs and connects to data sources |
| **Runtime Environment** | A logical grouping of one or more Secure Agents used to run IICS jobs |
| **Connection** | A saved configuration in IICS storing credentials and connectivity details for a specific system |

---

## 🔗 What to Learn Next

- **Advanced CDC Configuration:** Setting up Oracle supplemental logging, SQL Server CDC, and PostgreSQL logical replication for Incremental Load
- **Post-Ingestion Transformation with CDI:** Learn how to build CDI Mappings that read from the raw tables created by Mass Ingestion and transform them into analytics-ready models
- **Orchestrating Mass Ingestion with Taskflows:** Chain Mass Ingestion Jobs with CDI Mapping Tasks in a Taskflow to build an end-to-end pipeline: ingest → transform → publish
