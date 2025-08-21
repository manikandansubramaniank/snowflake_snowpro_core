# Snowflake Architecture and Concepts 

## 1. Traditional Database Architectures

- **Shared Disk Architecture:** All compute nodes share a central storage. Simple data management but has scalability limits due to network bottlenecks and single point of failure.
- **Shared Nothing Architecture:** Each node independently manages CPU, memory, and storage. Offers scalability and high availability but with more complex data management.
- **Snowflake's Multi-Cluster Shared Data Architecture:** Combines centralized persistent storage with multiple independent compute clusters caching data locally for strong performance and scalability.

![Database Architectures](https://docs.snowflake.com/en/_images/architecture-overview.png)  
*Source: Snowflake Architecture Overview*

Learn more: [Snowflake Architecture Overview](https://docs.snowflake.com/en/user-guide/architecture-overview.html)

---

## 2. Snowflake’s Three Architecture Layers

| Layer                | Description                                                  | Key Roles                                  |
|----------------------|--------------------------------------------------------------|--------------------------------------------|
| **Database Storage** | Decoupled, compressed columnar storage on cloud platforms like AWS S3 or Azure Blob. | Stores persistent data blobs efficiently. |
| **Compute Layer**    | Virtual warehouses made of scalable compute clusters (one or many nodes). | Executes query processing at scale.        |
| **Cloud Services Layer** | The "brain" handling authentication, access control, metadata, query parsing/optimization, and infrastructure management. | Coordinates system activities.              |

![Snowflake Layered Architecture](https://docs.snowflake.com/en/_images/architecture-overview.png)  
*Source: Snowflake Documentation*

References:  
- [Storage Layer Details](https://docs.snowflake.com/en/user-guide/architecture-data-storage.html)  
- [Compute Warehouses](https://docs.snowflake.com/en/user-guide/warehouses.html)  
- [Cloud Services Overview](https://docs.snowflake.com/en/user-guide/architecture-account.html)

---

## 3. Loading Data into Snowflake

- **Create Database and Table:**

```sql

CREATE DATABASE first_db;
CREATE TABLE "loan_payment" ( "ID" INT, "NAME" STRING, "AMOUNT" FLOAT );

```

- **Load Data from Cloud Storage:**

```sql

COPY INTO loan_payment
FROM 's3://your-bucket/loan_payment.csv'
FILE_FORMAT = (TYPE => 'CSV', FIELD_DELIMITER => ',', SKIP_HEADER => 1);

```

- **Query Validation:**

```sql

SELECT * FROM loan_payment;

```

![Loading Data Flow](https://docs.snowflake.com/_images/data-load-overview.png)  
*Source: Snowflake Data Loading Guide*

Details: [Snowflake Loading Data](https://docs.snowflake.com/en/user-guide/data-load-overview.html)

---

## 4. Snowflake Editions and Features

| Edition               | Key Use Case                    | Features                                              |
|-----------------------|---------------------------------|-------------------------------------------------------|
| Standard              | Foundational                    | Encryption, 1-day time travel, disaster recovery      |
| Enterprise            | Large enterprises               | Multi-cluster warehouses, 90-day time travel          |
| Business Critical     | Compliance & regulation         | Enhanced security, failover, customer-managed keys    |
| Virtual Private       | Highest isolation & security    | Dedicated hardware and metadata                       |

![Snowflake Editions](https://docs.snowflake.com/_images/editions-feature-matrix.png)  
*Source: Snowflake Editions Matrix*

More info: [Snowflake Editions](https://docs.snowflake.com/en/user-guide/editions.html)

---

## 5. Pricing Components

- **Compute:** Charged per second based on warehouse size and usage.
- **Storage:** Monthly rate for compressed data stored.
- **Data Transfer:** Free ingress; egress costs vary by region/cloud.

![Snowflake Pricing](https://www.snowflake.com/wp-content/themes/snowflake/assets/images/pricing-hero.png)  
*Source: Snowflake Pricing Page*

Official: [Snowflake Pricing](https://www.snowflake.com/pricing/)

---

## 6. Resource Monitors

- Define credit quotas on warehouses/account.
- Actions: notification, suspend warehouse (deferred/immediate).
- Enables cost control and credit usage tracking.

![Resource Monitor Workflow](https://docs.snowflake.com/_images/resource-monitor-workflow.png)  
*Source: Snowflake Resource Monitors*

Further details: [Resource Monitors Guide](https://docs.snowflake.com/en/user-guide/resource-monitors.html)

---

## 7. Warehouse Scaling

- **Single Cluster:** Fixed compute capacity.
- **Multi-Cluster:** Multiple clusters for scaling concurrent queries.
- **Scaling Modes:**  
  - Maximized (fixed clusters)  
  - Auto (dynamic min/max clusters)  
- **Scaling Policies:**  
  - Standard (performance prioritization)  
  - Economy (credit conservation)

![Warehouse Auto-scaling](https://docs.snowflake.com/_images/warehouses-auto-scaling.png)  
*Source: Snowflake Warehouses Auto-Scaling*

Learn: [Warehouses Auto-scaling](https://docs.snowflake.com/en/user-guide/warehouses-auto-scaling.html)

---

## 8. Object Hierarchy in Snowflake

- Account > Databases > Schemas > Tables/Views/Functions/Stages.
- Additional objects include Tasks, Streams, and Pipes.
- Roles and permissions manage access.

![Snowflake Object Hierarchy](https://docs.snowflake.com/_images/object_hierarchy.png)  
*Source: Snowflake Database Objects*

Read more: [Databases and Database Objects](https://docs.snowflake.com/en/user-guide/databases.html)

---

## 9. SnowSQL Command Line Tool

- Command-line client for query execution and administration.
- Supports Windows, Linux, MacOS.
- Typical connection example:

```sql
snowsql -a <account_name> -u <username>
```

- Useful for scripting and loading/unloading data.
- Context switching with `USE` statements.

![SnowSQL CLI](https://docs.snowflake.com/_images/snowsql-example.png)  
*Source: Snowflake SnowSQL Documentation*

Guide: [SnowSQL User Guide](https://docs.snowflake.com/en/user-guide/snowsql.html)

---

