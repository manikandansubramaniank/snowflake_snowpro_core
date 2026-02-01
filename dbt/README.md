# dbt Technical Documentation

<div align="center">

![dbt](https://img.shields.io/badge/dbt-FF694B?style=for-the-badge&logo=dbt&logoColor=white)
![Data Transformation](https://img.shields.io/badge/ELT-Data%20Transformation-orange?style=for-the-badge)
![Documentation](https://img.shields.io/badge/Documentation-Complete-success?style=for-the-badge)

**Comprehensive Technical Guide for dbt (data build tool)**

*Integrating Analytics Engineering, CI/CD, and Production Best Practices*

</div>

---

## 📚 About This Guide

This repository contains comprehensive technical documentation for **dbt (data build tool)**, designed for analytics engineers, data engineers, and data platform teams. Each guide integrates real-world Software Development Life Cycle (SDLC) methodologies, CI/CD workflows, and production deployment patterns.

### ✨ Key Features
- 📖 **Beginner to Advanced**: Progressive learning path from fundamentals to enterprise patterns
- 💻 **Complete Syntax Reference**: Full parameter tables with defaults and examples
- 🔄 **SDLC Integration**: CI/CD pipelines, GitHub Actions, and state management
- 🏢 **Production Ready**: Deployment, orchestration, and cost optimization
- 🌐 **Multi-Platform**: Snowflake, BigQuery, Redshift, and cloud orchestration
- ✅ **Best Practices**: Testing, documentation, and data quality patterns

---

## 📋 Table of Contents

- [Documentation Index](#-documentation-index)
  - [1. Introduction to dbt](#1-introduction-to-dbt)
  - [2. dbt Architecture](#2-dbt-architecture)
  - [3. Core Objects](#3-core-objects)
  - [4. Data Modeling Patterns](#4-data-modeling-patterns)
  - [5. Testing & Quality Assurance](#5-testing--quality-assurance)
  - [6. Deployment & Orchestration](#6-deployment--orchestration)
  - [7. Advanced Features](#7-advanced-features)
  - [8. Integrations & Ecosystem](#8-integrations--ecosystem)
  - [9. Production Best Practices](#9-production-best-practices)
  - [10. Real-World Examples](#10-real-world-examples)

---

## 📖 Documentation Index

### 1. Introduction to dbt
**📄 [dbt_01_introduction.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md)**

Foundational concepts covering dbt fundamentals, core concepts, and getting started with analytics engineering.

#### Topics Covered:
- [What is dbt?](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#what-is-dbt)
- [ELT vs ETL](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#elt-vs-etl)
  - [Diagram: ELT Flow with dbt](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#diagram-elt-flow-with-dbt)
- [Core Concepts](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#core-concepts)
  - [Models](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#models)
  - [Sources](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#sources)
  - [Tests](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#tests)
  - [Documentation](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#documentation)
  - [Seeds](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#seeds)
  - [Snapshots](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#snapshots)
- [dbt_project.yml Complete Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#dbt_projectyml-complete-configuration)
  - [Syntax Template](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#syntax-template)
  - [Parameter Reference Table](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#parameter-reference-table)
  - Basic and Advanced Examples
- [SDLC Use Case: GitHub Repository Onboarding](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#sdlc-use-case-github-repository-onboarding)
  - GitHub Actions CI/CD workflows
  - Environment provisioning patterns
- [Key Takeaways](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_01_introduction.md#key-takeaways)

---

### 2. dbt Architecture
**📄 [dbt_02_architecture.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md)**

Deep dive into dbt's project structure, execution lifecycle, and dependency graph management.

#### Topics Covered:
- [Project Structure](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#project-structure)
  - [Standard Project Layout](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#standard-project-layout)
  - [Diagram: dbt Project Organization](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#diagram-dbt-project-organization)
- [Parsing & Execution Lifecycle](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#parsing--execution-lifecycle)
  - [Stage 1: Parsing](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#stage-1-parsing)
  - [Stage 2: Compilation](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#stage-2-compilation)
  - [Stage 3: Execution](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#stage-3-execution)
  - [Stage 4: Artifact Generation](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#stage-4-artifact-generation)
  - [Diagram: dbt Run Lifecycle](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#diagram-dbt-run-lifecycle)
- [Selectors & Node Dependency Graph](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#selectors--node-dependency-graph)
  - [Selector Syntax Reference](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#selector-syntax-reference)
  - [Graph Traversal Examples](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#graph-traversal-example)
- [profiles.yml Complete Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#profilesyml-complete-configuration)
  - [Snowflake Adapter](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#parameter-reference-table-snowflake-adapter)
  - [BigQuery Adapter Example](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_02_architecture.md#bigquery-adapter-example)
  - Multi-environment setup patterns

---

### 3. Core Objects
**📄 [dbt_03_core_objects.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md)**

Complete reference for dbt's core building blocks: models, sources, seeds, and snapshots.

#### Topics Covered:
- [Models](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#models)
  - [Model Materialization Types](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#model-materialization-types)
    - Table, View, Incremental, Ephemeral
  - [Diagram: Model Materialization Strategy Decision Tree](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#diagram-model-materialization-strategy-decision-tree)
  - [Model Configuration: Complete Syntax](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#model-configuration-complete-syntax)
  - [Model Config Parameter Table](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#model-config-parameter-table)
  - [Basic Model Example (View)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#basic-model-example-view)
  - [Advanced Model Example (Incremental with Merge)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#advanced-model-example-incremental-with-merge)
  - [Ephemeral Model Example](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#ephemeral-model-example)
- [Sources](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#sources)
  - [Source Configuration: Complete Syntax](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#source-configuration-complete-syntax)
  - [Source Parameter Table](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#source-parameter-table)
- [Seeds](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#seeds)
  - [Seed Configuration Syntax](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#seed-configuration-syntax)
  - CSV reference data management
- [Snapshots](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#snapshots)
  - [Snapshot Strategies](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#snapshot-strategies)
    - Timestamp and Check strategies
  - [Snapshot Configuration: Complete Syntax](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_03_core_objects.md#snapshot-configuration-complete-syntax)

---

### 4. Data Modeling Patterns
**📄 [dbt_04_data_modeling_patterns.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md)**

Industry-standard modeling patterns for building production-grade analytics pipelines.

#### Topics Covered:
- [Overview](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#overview)
- [4.1 Staging → Intermediate → Marts Architecture](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#41-staging--intermediate--marts-architecture)
  - [Pattern Description](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#pattern-description)
  - Layer-by-layer modeling approach
- [4.2 Dimensional Modeling](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#42-dimensional-modeling)
  - [Star Schema Pattern](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#star-schema-pattern)
  - [Syntax: Fact Table (Incremental)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#syntax-fact-table-incremental)
  - [Syntax: Dimension Table (SCD Type 1)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#syntax-dimension-table-scd-type-1)
- [4.3 Incremental Models](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#43-incremental-models)
  - [Incremental Strategies](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#incremental-strategies)
    - Merge, Delete+Insert, Insert Overwrite
  - [Syntax: Incremental Model Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#syntax-incremental-model-configuration)
- [4.4 Snapshots (SCD Type 2)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#44-snapshots-scd-type-2)
  - [Syntax: Snapshot Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#syntax-snapshot-configuration)
- [4.5 SDLC Integration: Feature Flags](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#45-sdlc-integration-feature-flags)
- [Summary](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#summary)
- [Quick Reference Commands](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_04_data_modeling_patterns.md#quick-reference-commands)

---

### 5. Testing & Quality Assurance
**📄 [dbt_05_testing_quality.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md)**

Comprehensive testing framework for ensuring data quality and pipeline reliability.

#### Topics Covered:
- [Overview](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#overview)
- [5.1 Test Types](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#51-test-types)
  - [Test Classification](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#test-classification)
- [5.2 Generic Tests (Built-in)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#52-generic-tests-built-in)
  - [Syntax: schema.yml Tests](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#syntax-schemayml-tests)
    - unique, not_null, accepted_values, relationships
- [5.3 Singular Tests (Custom SQL)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#53-singular-tests-custom-sql)
  - [Syntax: Singular Test](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#syntax-singular-test)
- [5.4 Custom Generic Tests](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#54-custom-generic-tests)
  - [Syntax: Custom Generic Test](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#syntax-custom-generic-test)
- [5.5 Source Freshness Tests](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#55-source-freshness-tests)
  - [Syntax: Source Freshness Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#syntax-source-freshness-configuration)
- [5.6 Unit Tests (Preview)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#56-unit-tests-preview)
  - [Syntax: Unit Test Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#syntax-unit-test-configuration)
- [5.7 SDLC Integration: Automated Test Gates](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#57-sdlc-integration-automated-test-gates)
  - GitHub Actions CI/CD test automation
- [5.8 Test Selection Patterns](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_05_testing_quality.md#58-test-selection-patterns)

---

### 6. Deployment & Orchestration
**📄 [dbt_06_deployment_orchestration.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md)**

Production deployment patterns, state management, and orchestration with external tools.

#### Topics Covered:
- [Overview](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#overview)
- [6.1 Deployment Patterns](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#61-deployment-patterns)
  - [dbt Core CLI vs dbt Cloud](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#dbt-core-cli-vs-dbt-cloud)
- [6.2 dbt Core CLI Commands](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#62-dbt-core-cli-commands)
  - [Syntax: Core Commands](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#syntax-core-commands)
- [6.3 State-Based Selection](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#63-state-based-selection)
  - [Syntax: State Selectors](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#syntax-state-selectors)
  - Slim CI patterns
- [6.4 Artifacts](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#64-artifacts)
  - [Artifact Files](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#artifact-files)
    - manifest.json, run_results.json, catalog.json
  - [Artifact Management Best Practices](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#artifact-management-best-practices)
- [6.5 dbt Cloud Jobs](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#65-dbt-cloud-jobs)
  - [dbt Cloud Job Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#dbt-cloud-job-configuration)
- [6.6 SDLC Integration: GitHub Actions + dbt Cloud](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#66-sdlc-integration-github-actions--dbt-cloud)
  - Hybrid CI/CD pipeline patterns
- [6.7 External Orchestrators](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#67-external-orchestrators)
  - [Airflow Integration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_06_deployment_orchestration.md#airflow-integration)

---

### 7. Advanced Features
**📄 [dbt_07_advanced_features.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md)**

Advanced dbt capabilities including packages, macros, exposures, and platform-specific optimizations.

#### Topics Covered:
- [Overview](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#overview)
- [7.1 Packages](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#71-packages)
  - [What are Packages?](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#what-are-packages)
  - [Syntax: packages.yml](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#syntax-packagesyml)
    - dbt_utils, dbt_expectations, codegen
- [7.2 Exposures](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#72-exposures)
  - [Syntax: Exposures (schema.yml)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#syntax-exposures-schemayml)
  - Dashboard and report dependencies
- [7.3 Metrics (dbt Metrics - Legacy)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#73-metrics-dbt-metrics---legacy)
  - [Syntax: Metrics (Legacy - for reference)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#syntax-metrics-legacy---for-reference)
  - [Modern Alternative: dbt Semantic Layer (MetricFlow)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#modern-alternative-dbt-semantic-layer-metricflow)
- [7.4 Jinja Macros](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#74-jinja-macros)
  - [What are Macros?](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#what-are-macros)
  - [Syntax: Macros](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#syntax-macros)
    - Custom macro development
- [7.5 dbt + Snowflake Optimizations](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#75-dbt--snowflake-optimizations)
  - [Syntax: Snowflake-Specific Configs](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#syntax-snowflake-specific-configs)
    - Clustering, transient tables, query tags
- [7.6 Production Workflow Integration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_07_advanced_features.md#76-production-workflow-integration)

---

### 8. Integrations & Ecosystem
**📄 [dbt_08_integrations_ecosystem.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md)**

Integration patterns with cloud data warehouses, orchestration platforms, and BI tools.

#### Topics Covered:
- [Overview](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#overview)
- [8.1 Database Adapters](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#81-database-adapters)
  - [What are Adapters?](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#what-are-adapters)
  - [Installation](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#installation)
- [8.2 Snowflake Adapter](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#82-snowflake-adapter)
  - [Syntax: Snowflake-Specific Configs](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#syntax-snowflake-specific-configs)
- [8.3 BigQuery Adapter](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#83-bigquery-adapter)
  - [Syntax: BigQuery-Specific Configs](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#syntax-bigquery-specific-configs)
- [8.4 Redshift Adapter](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#84-redshift-adapter)
  - [Syntax: Redshift-Specific Configs](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#syntax-redshift-specific-configs)
- [8.5 Orchestration: Airflow](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#85-orchestration-airflow)
  - [Syntax: Airflow DAG with dbt](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#syntax-airflow-dag-with-dbt)
- [8.6 Orchestration: Dagster](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#86-orchestration-dagster)
  - [Syntax: Dagster Integration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#syntax-dagster-integration)
- [8.7 Orchestration: Meltano](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#87-orchestration-meltano)
  - [Syntax: meltano.yml](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#syntax-meltanoyml)
- [8.8 BI Tool Integration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_08_integrations_ecosystem.md#88-bi-tool-integration)
  - Looker, Tableau, Power BI, Mode Analytics

---

### 9. Production Best Practices
**📄 [dbt_09_production_best_practices.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md)**

Production-grade patterns for state management, cost optimization, and monitoring.

#### Topics Covered:
- [Overview](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#overview)
- [9.1 State Management & Artifacts](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#91-state-management--artifacts)
  - [Key Artifacts](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#key-artifacts)
    - manifest.json, run_results.json, catalog.json
  - [Syntax: State-Based Selectors](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#syntax-state-based-selectors)
- [9.2 Selective Runs](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#92-selective-runs)
  - [Syntax: Selection & Exclusion](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#syntax-selection--exclusion)
- [9.3 Cost Optimization](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#93-cost-optimization)
  - [Strategy 1: Incremental Materialization](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#strategy-1-incremental-materialization)
  - [Strategy 2: Warehouse Auto-Scaling](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#strategy-2-warehouse-auto-scaling)
  - [Strategy 3: Query Result Caching](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#strategy-3-query-result-caching)
  - [Strategy 4: Partition Pruning](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#strategy-4-partition-pruning)
  - [Cost Monitoring](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#cost-monitoring)
- [9.4 Logging & Monitoring](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#94-logging--monitoring)
  - [Syntax: Logging Configuration](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#syntax-logging-configuration)
  - [Structured Logging Example](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#structured-logging-example)
  - [Custom Logging Macro](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_09_production_best_practices.md#custom-logging-macro)

---

### 10. Real-World Examples
**📄 [dbt_10_real_world_examples.md](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md)**

Practical implementation scenarios demonstrating complete dbt projects in production.

#### Topics Covered:
- [Overview](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#overview)
- [10.1 E-Commerce Analytics Pipeline](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#101-e-commerce-analytics-pipeline)
  - [Business Requirements](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#business-requirements)
  - [Project Structure](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#project-structure)
  - [Configuration Files](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#configuration-files)
  - [Source Definitions](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#source-definitions)
  - [Staging Models](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#staging-models)
  - [Intermediate Models](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#intermediate-models)
  - [Marts Layer (Fact Tables)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#marts-layer-fact-tables)
  - [Marts Layer (Dimension Tables)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#marts-layer-dimension-tables)
  - [Model Documentation](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#model-documentation)
  - [Custom Tests](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#custom-tests)
  - [Exposures](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#exposures)
- [10.2 SCD Type 2: Customer Dimension](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#102-scd-type-2-customer-dimension)
  - [What is SCD Type 2?](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#what-is-scd-type-2)
  - [Syntax: Snapshots](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#syntax-snapshots)
- [10.3 Snowflake CDC (Change Data Capture)](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#103-snowflake-cdc-change-data-capture)
  - [What is CDC?](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#what-is-cdc)
  - [Setup: Snowflake Streams](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#setup-snowflake-streams)
  - [dbt Model: Process CDC Stream](https://github.com/manikandansubramaniank/snowflake_snowpro_core/tree/main/dbt/dbt_10_real_world_examples.md#dbt-model-process-cdc-stream)

---
