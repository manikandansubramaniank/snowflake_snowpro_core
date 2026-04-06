# Complete Beginner's Guide to Cloud Data Integration (CDI)

> **IICS Service:** Cloud Data Integration (CDI)
> **Difficulty:** Beginner
> **Estimated Reading Time:** 35 minutes
> **Prerequisites:** An IICS organization account (free trial is sufficient), basic understanding of what a database table and a CSV file are
> **Learning Objectives:**
> - Explain what Cloud Data Integration is and how it fits within the IICS platform
> - Define and distinguish between the four main CDI asset types: Mappings, Mapping Tasks, Synchronization Tasks, and Data Transfer Tasks
> - Build a simple Mapping in the Mapping Designer using Source, Target, and an intermediate Transformation
> - Execute a Mapping using a Mapping Task and monitor the job in the Monitor service
> - Configure a Synchronization Task for quick table-to-table data movement
> - Set up a Data Transfer Task for bulk file-based data loading
> - Choose the correct CDI asset type for a given business scenario

---

## 🧭 What Is Cloud Data Integration (CDI)?

**Cloud Data Integration (CDI)** (the IICS service that lets you design, run, and manage data movement and transformation jobs through a browser-based interface) is the core workhorse of the Informatica Intelligent Cloud Services platform.

Think of CDI as a visual assembly line for data:
- You tell it **where to get** the data (a database, a file, an API)
- You tell it **what to do** with the data (filter it, clean it, combine it, reshape it)
- You tell it **where to put** the result (another database, a data warehouse, a file)

All of this happens without writing code — you drag, drop, and configure using a browser.

> 💡 **What is IICS?**
> **Informatica Intelligent Cloud Services (IICS)** is Informatica's cloud-native platform for data integration, data quality, API management, and application integration. Think of IICS as a suite of specialized services — CDI is one of the most commonly used services in that suite.

### Where CDI Fits in the Data Pipeline

In most organizations, raw data lives in many places — ERP systems like SAP, CRM tools like Salesforce, flat files on SFTP servers, cloud databases like Snowflake or Amazon Redshift. CDI's job is to **extract** that data, **transform** it into a usable shape, and **load** it into a destination. This process is traditionally called **ETL (Extract, Transform, Load)** or **ELT (Extract, Load, Transform)** depending on when the transformations happen.

| Term | Meaning |
|------|---------|
| **ETL** | Extract data, Transform it in-flight, then Load to target |
| **ELT** | Extract data, Load it raw into the target, then Transform inside the target system |
| **CDI** | Supports both ETL and ELT patterns depending on configuration |

---

## 🏗️ The Four CDI Asset Types — Choosing the Right Tool

CDI offers four distinct asset types. Each is designed for a specific use case. Choosing the wrong one leads to unnecessary complexity or poor performance.

| Asset Type | Best For | Complexity | Transformations? | Typical Use Case |
|-----------|---------|-----------|-------------------|-----------------|
| **Mapping** | Complex data movement with transformations | Medium–High | Yes — full transformation palette | Loading a data warehouse with business logic |
| **Mapping Task** | Running a Mapping | Low (it wraps a Mapping) | N/A — it runs whatever the Mapping defines | Scheduling or manually triggering a Mapping |
| **Synchronization Task** | Simple table-to-table replication | Low | Minimal — basic field mapping and filtering | Copying a Salesforce table to Snowflake as-is |
| **Data Transfer Task** | Bulk file-based data movement | Low | None | Moving CSV files from S3 to Azure Blob Storage |

> ⚠️ **Key Distinction:** A Mapping is a **design artifact** — it defines the logic. A Mapping Task is an **execution artifact** — it runs the logic. You always need BOTH to process data using Mappings. Synchronization Tasks and Data Transfer Tasks are self-contained — they combine design and execution in one object.

---

## 🔌 Foundations: Connections, Secure Agents, and Runtime Environments

Before you build anything in CDI, three foundational components must be in place. Skipping this section is the #1 reason beginners get stuck.

### Connections

A **Connection** (a saved configuration that stores the credentials, host address, and connectivity details for a specific external system) is how IICS knows how to reach your data sources and targets.

**To create a Connection:**
1. From the IICS home page, click **Administrator** in the left navigation
2. Click **Connections** in the left panel
3. Click **New Connection**
4. Select the **Connection Type** (e.g., Snowflake, Oracle, Salesforce, Flat File)
5. Fill in the required fields — these vary by type but typically include: host, port, database name, username, and password
6. Under **Runtime Environment**, select the agent group that has network access to this system
7. Click **Test Connection** to verify — you should see "The test for this connection was successful"
8. Click **Save**

### Secure Agents

A **Secure Agent** (a lightweight program you install on a machine — either on-premises or in the cloud — that actually executes your IICS jobs and connects to your data sources) is the engine that does the work. IICS itself is a cloud control plane, but the Secure Agent is what reads from your Oracle database or writes to your Snowflake warehouse.

> 💡 **Why "Secure"?**
> The agent initiates all communication **outbound** to the IICS cloud. No inbound firewall ports need to be opened, which is why Informatica calls it "secure." Your IT team only needs to allow outbound HTTPS traffic from the agent machine.

### Runtime Environments

A **Runtime Environment** (a logical grouping of one or more Secure Agents that IICS uses to decide which machine runs a given job) allows you to organize agents by purpose — for example, one Runtime Environment for development jobs and another for production jobs.

**To verify your Runtime Environment is healthy:**
1. Go to **Administrator** → **Runtime Environments**
2. Your agent group should show a status of **"Up and Running"**
3. If it shows "Down" or "Unavailable," the Secure Agent service on that machine needs to be restarted

---

## 🗺️ Deep Dive: Mappings — The Core of CDI

A **Mapping** (a reusable design artifact that defines how data moves from one or more sources to one or more targets, with optional transformations applied in between) is where you spend most of your time in CDI.

### The Mapping Designer Canvas

The **Mapping Designer** (the browser-based visual workspace where you build and edit Mappings) opens as a canvas with a toolbar at the top, a palette of transformations you can add, and a properties panel at the bottom.

When you first create a Mapping, the canvas shows a default **Source** transformation and a default **Target** transformation connected by a line. Your job is to configure them and optionally add transformations between them.

### Core Transformations for Beginners

IICS provides dozens of transformations, but as a beginner, focus on these five:

| Transformation | What It Does | Real-World Analogy |
|---------------|-------------|-------------------|
| **Source** | Reads data from an external system | The loading dock where raw materials arrive |
| **Target** | Writes data to an external system | The shipping dock where finished goods leave |
| **Filter** | Removes rows that don't meet a condition | A quality inspector rejecting defective items |
| **Expression** | Calculates new values or modifies existing fields | A worker assembling parts into a new product |
| **Joiner** | Combines rows from two data streams based on a key | Merging two inventory lists into one by matching product IDs |

### Step-by-Step: Building a Mapping with a Filter and Expression

**Scenario:** You have an `EMPLOYEES` table in Oracle. You want to load only active employees into Snowflake, and you need to add a computed column `FULL_NAME` that concatenates `FIRST_NAME` and `LAST_NAME`.

1. Navigate to **Data Integration** → click **New** → **Mappings** → **Mapping**
2. Name it `m_employees_oracle_to_snowflake` → click **Create**
3. The Mapping Designer canvas opens with default Source and Target

**Configure the Source:**
4. Click the **Source** transformation on the canvas
5. In the **Properties** panel → **Source** tab → click **Select** next to Connection → choose your Oracle connection
6. Click **Select** next to Source Object → browse to the `EMPLOYEES` table → click **OK**
7. Go to the **Fields** tab to confirm all columns are imported

**Add a Filter Transformation:**
8. On the canvas toolbar, click the **+** icon → select **Filter**
9. Drag from the Source output port (right side) to the Filter input port (left side) to connect them
10. Click the Filter transformation → in Properties → **Filter** tab → click the **+** icon to add a filter condition
11. Set the condition: `STATUS = 'ACTIVE'`

**Add an Expression Transformation:**
12. Click **+** → select **Expression**
13. Connect the Filter output to the Expression input
14. Click the Expression → **Expression** tab → click **+** to add a new output field
15. Name it `FULL_NAME`, set the datatype to `String(200)`
16. Set the expression: `CONCAT(CONCAT(FIRST_NAME, ' '), LAST_NAME)`

**Configure the Target:**
17. Connect the Expression output to the Target input
18. Click the Target → **Target** tab → select your Snowflake connection and target table
19. Go to **Field Mapping** → click **Auto Map** to map matching field names
20. Manually map `FULL_NAME` to the corresponding target column

**Validate and Save:**
21. Click the **three-dot menu (⋯)** → **Validate** — fix any errors shown in the validation panel
22. Click **Save**

> 💡 **Why Validate?**
> Validation checks your Mapping for structural errors — missing connections, incompatible data types, unlinked fields — before you spend time creating a Mapping Task and running it. Always validate after every significant change.

---

## 🚀 Deep Dive: Mapping Tasks — Running Your Mappings

A **Mapping Task** (a runnable job that wraps a Mapping and adds execution settings such as the runtime environment, session properties, and schedule) is the "play button" for your Mapping.

### Creating and Running a Mapping Task

1. Navigate to **Data Integration** → click **New** → **Tasks** → **Mapping Task**
2. Name it `mt_employees_oracle_to_snowflake`
3. Under **Mapping**, click **Select** → choose `m_employees_oracle_to_snowflake`
4. Under **Runtime Environment**, select the Secure Agent group with access to both Oracle and Snowflake
5. (Optional) Under **Schedule**, click **New Schedule** to set a recurring run time
6. Click **Save**
7. Click **Run** to execute immediately

### Monitoring Execution

1. From the left navigation, click **Monitor**
2. Click **All Jobs** or **My Jobs**
3. Find your Mapping Task — the status will show: Running, Success, Warning, or Failed
4. Click the job name to view details: row counts (source rows read, target rows written, rows rejected), duration, and error messages if any

> ⚠️ **If you see zero rows written but no error:** Check that your Filter condition isn't filtering out ALL rows. Run a quick query on your source data to confirm matching rows exist.

---

## 🔄 Deep Dive: Synchronization Tasks — Quick Table Replication

A **Synchronization Task** (a self-contained CDI asset that copies data from a source to a target with minimal configuration — no separate Mapping needed) is designed for straightforward replication scenarios where you don't need complex transformations.

### When to Use a Synchronization Task Instead of a Mapping

| Criteria | Use Synchronization Task | Use Mapping |
|----------|------------------------|-------------|
| Simple field renaming or filtering | ✅ Yes | Also works, but overkill |
| Complex multi-source joins | ❌ No | ✅ Yes |
| Lookups against reference tables | ❌ No | ✅ Yes |
| Quick proof-of-concept replication | ✅ Yes — fastest setup | Works, but slower to configure |
| Need to reuse logic across jobs | ❌ Limited | ✅ Yes — Mappings are reusable |

### Creating a Synchronization Task

1. Navigate to **Data Integration** → click **New** → **Tasks** → **Synchronization Task**
2. Name it `st_salesforce_accounts_to_snowflake`
3. Select your **Source Connection** (e.g., Salesforce) and **Source Object** (e.g., Account)
4. Select your **Target Connection** (e.g., Snowflake) and **Target Object** (e.g., STG_ACCOUNT)
5. On the **Field Mapping** screen, map source fields to target fields — use **Auto Map** for matching names
6. (Optional) Add a simple filter in the **Filters** section — e.g., `Industry = 'Technology'`
7. Under **Runtime Environment**, select your agent group
8. Click **Save** → **Run**

---

## 📦 Deep Dive: Data Transfer Tasks — Bulk File Movement

A **Data Transfer Task** (a CDI asset designed specifically for moving files in bulk between file-based systems — such as FTP, SFTP, Amazon S3, Azure Blob Storage, or Google Cloud Storage — without any transformation) is the right tool when you need to move files without looking at or changing their contents.

### When to Use a Data Transfer Task

- Moving hundreds of CSV files from an SFTP server to Amazon S3
- Archiving processed files from one cloud bucket to another
- Receiving flat files from a vendor and staging them before processing

### Creating a Data Transfer Task

1. Navigate to **Data Integration** → click **New** → **Tasks** → **Data Transfer Task**
2. Name it `dt_sftp_to_s3_daily_files`
3. Select your **Source Connection** (e.g., SFTP connection) and specify the **Source Directory** path
4. Select your **Target Connection** (e.g., Amazon S3 connection) and specify the **Target Directory** path
5. Configure file handling options:
   - **Transfer Mode:** Choose between moving all files or only files matching a pattern (e.g., `*.csv`)
   - **If File Exists:** Choose whether to overwrite, skip, or rename
6. Under **Runtime Environment**, select your agent group
7. Click **Save** → **Run**

> 💡 **Data Transfer vs. Mapping for Files:**
> If you only need to MOVE a file from point A to point B, use a Data Transfer Task — it's faster to set up and optimized for file operations. If you need to READ the file contents, transform them, and write the result somewhere, use a Mapping with a Flat File source.

---

## 🔗 How It All Connects — A Typical CDI Pipeline

In production, CDI assets rarely run in isolation. Here's how a typical pipeline looks:

```
Data Transfer Task                Mapping Task                    Mapping Task
(Move CSV from SFTP → S3)  →  (Read CSV, clean data,       →  (Aggregate clean data
                                load to staging table)           into reporting table)
```

These tasks are chained together using **Taskflows** (an IICS orchestration asset that defines the execution sequence of multiple tasks, with branching, error handling, and dependency logic). Taskflows are covered in a separate module, but it's important to know that CDI assets are designed to be composed together.

---

## ⚠️ Common Mistakes & Troubleshooting

| Mistake | Why It Happens | How to Fix It |
|---------|---------------|---------------|
| Mapping validates successfully but Mapping Task fails at runtime | The Secure Agent cannot reach the source or target system (firewall, VPN, credentials expired) | Check **Administrator** → **Runtime Environments** to confirm agent is "Up and Running." Verify connection credentials by editing the Connection and clicking **Test Connection** |
| Synchronization Task writes zero rows with no error | The source object has no data matching the filter condition, or the source connection points to an empty table | Remove the filter temporarily and re-run. Query the source directly to confirm data exists |
| Data Transfer Task fails with "file not found" | The source directory path is wrong, or the file name pattern doesn't match any files | Double-check the directory path (case-sensitive on Linux). Verify file names against your pattern. Use a wildcard like `*` to test |
| "No Runtime Environment available" error when running any task | No Secure Agent is registered, or the agent service is stopped | Go to **Administrator** → **Runtime Environments** → verify status. If down, restart the Secure Agent service on the host machine |
| Fields show as "unmapped" in the Target and rows are silently dropped | Auto Map didn't match because field names differ in case or spelling | Open **Field Mapping** and manually drag source fields to target fields. IICS field mapping is case-sensitive |
| Mapping runs successfully but data in target is wrong | Expression transformation logic has an error (e.g., wrong function, data type mismatch) | Preview the Expression output using the **Data Preview** feature in the Mapping Designer before running the full task |

---

## 📝 Key Takeaways

- **CDI is the data movement engine of IICS** — it handles extraction, transformation, and loading through a visual, browser-based interface
- **Mappings + Mapping Tasks are a two-part system** — the Mapping defines the logic; the Mapping Task executes it. You always need both
- **Synchronization Tasks are the shortcut** — use them for simple source-to-target replication without complex transformations
- **Data Transfer Tasks are for files, not data** — they move files between file systems without inspecting or transforming the content
- **Connections, Secure Agents, and Runtime Environments are the foundation** — every CDI asset depends on these three components being correctly configured before anything runs

---

## 🧪 Practice Exercise

**Scenario:** Your company receives a daily CSV file containing new product records from a supplier. The file has columns: `product_id`, `product_name`, `category`, `unit_price`, `is_active`. You need to load only active products (where `is_active = 'Y'`) into a database table, and you want to add a computed column `price_with_tax` that multiplies `unit_price` by 1.08.

**Your Task:**
1. Create a **Connection** for a Flat File source pointing to a local CSV file (create a sample CSV with 10 rows, 3 of which have `is_active = 'N'`)
2. Create a **Connection** for your target database (use any database available in your trial org)
3. Create a **Mapping** named `m_exercise_products_csv_to_db`:
   - Source: your CSV flat file connection
   - Add a **Filter** transformation with condition: `is_active = 'Y'`
   - Add an **Expression** transformation with a new output field: `price_with_tax = unit_price * 1.08`
   - Target: your database connection and table
4. **Validate** the Mapping — resolve any errors
5. Create a **Mapping Task** named `mt_exercise_products_csv_to_db` and **Run** it
6. Check **Monitor** → **My Jobs** — confirm the job succeeded with exactly 7 rows written (the 7 active products)

**Expected Outcome:** Job status shows "Success." The target table contains 7 rows, each with a correctly calculated `price_with_tax` value. The 3 inactive products from the CSV were filtered out.

---

## 📚 Glossary

| Term | Definition |
|------|-----------|
| **Cloud Data Integration (CDI)** | The IICS service for designing, running, and managing data movement and transformation jobs |
| **Mapping** | A reusable design artifact that defines how data moves from source to target with optional transformations |
| **Mapping Task** | A runnable job that wraps a Mapping and adds execution settings like schedule and runtime environment |
| **Mapping Designer** | The browser-based visual canvas in IICS where you build and edit Mappings |
| **Synchronization Task** | A self-contained CDI asset for simple source-to-target data replication without complex transformations |
| **Data Transfer Task** | A CDI asset for moving files in bulk between file-based systems without transformation |
| **Source Transformation** | The Mapping component that reads data from an external system |
| **Target Transformation** | The Mapping component that writes data to an external system |
| **Filter Transformation** | A Mapping component that removes rows that do not meet a specified condition |
| **Expression Transformation** | A Mapping component that calculates new values or modifies existing fields using functions |
| **Joiner Transformation** | A Mapping component that combines rows from two data streams based on a matching key |
| **Connection** | A saved configuration storing credentials and connectivity details for a specific external system |
| **Secure Agent** | A lightweight program installed on a machine that executes IICS jobs and connects to data sources |
| **Runtime Environment** | A logical grouping of one or more Secure Agents used to run IICS jobs |
| **Validate** | A pre-run check in the Mapping Designer that verifies a Mapping has no structural errors |
| **ETL** | Extract, Transform, Load — a data integration pattern where data is transformed before loading into the target |
| **ELT** | Extract, Load, Transform — a pattern where data is loaded raw into the target and transformed there |
| **Taskflow** | An IICS orchestration asset that chains multiple tasks in a defined execution sequence with error handling |
| **Monitor** | The IICS service where you view the status, logs, and row counts of running and completed jobs |

---

## 🔗 What to Learn Next

- **Intermediate Transformations in CDI:** Lookup, Router, Aggregator, Normalizer — expand your transformation toolkit for complex business logic
- **Taskflows and Orchestration:** Learn how to chain Mappings, Synchronization Tasks, and Data Transfer Tasks into automated, dependency-aware workflows
- **Parameters and Reusability:** Use input and in-out parameters to make your Mappings configurable across environments (dev, test, prod) without editing them
