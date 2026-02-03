---
title: Analytics Architecture
layout: default
---

# My Analytics Architecture Methodology

This page outlines the analytics architecture methodology I use when working on **consumer facing software products**, particularly for **feature usage and performance telemetry**, but applicable to analytics systems across domains. 

The design prioritizes **scalability, cost efficiency, and fast stakeholder access**.

At a high level, the architecture follows a **directed, acyclic, layered data pipeline** where each layer has a clearly defined responsibility and consumer.

---   

## Core Design Principles

  - **Directed and acyclic pipelines**  
  Each dataset flows in one direction only. No circular dependencies, no implicit backfills.

  - **Incremental first design**  
  Pipelines append or update only new data using partition columns. Full table rebuilds are avoided unless absolutely necessary or cost negligible.

  - **Cost aware analytics**  
  Storage is cheap and compute is not. Expensive aggregations are computed once and reused downstream.

  - **Clear separation of concerns**  
  Raw ingestion, data preparation, and reporting are handled in distinct layers with the ability to version control and access control each layer to necessary parties.

---   

## Layered Architecture Overview

### 1. Raw `fact` Tables (Event Level Data)

This layer stores **raw, high volume event data** directly from production systems.

**Characteristics**
  - Hundreds of millions to billions of rows
  - Often exceeds 1 (one) TB in size
  - Append only, batch insert or merge jobs
  - Partitioned by event date (or shard date if working with sharded marts)
  - Partition expiry set to ~90 days
  - Clustered by commonly filtered columns

**Design Rationale**
  - Raw events are expensive to scan and rarely needed long term
  - Short retention minimizes storage costs, but allows for quick searching of specific, recent partitions
  - Avoid rebuilding or rescanning large sharded tables on every load, which is extremely inefficient and costly

This layer exists for **correctness and traceability**, not analytics consumption.

---   

### 2. Staging Tables (`stg`)

Staging tables prepare data for analytical use while still retaining relatively fine granularity.

**Characteristics**
  - Built directly from raw fact tables
  - Incremental insert or merge batch loads only
  - Always select the most recent data using the partition column
  - Clustered to optimize downstream queries
  - Optional partitioning and expiry depending on table size
  - Enforce testing to ensure data quality, alert when data does not flow cleanly to this layer

**Responsibilities**
  - Normalize and clean raw event data
  - Apply consistent data modeling and transformations
  - Reduce query complexity for downstream layers

**Design Rationale**
  - Rebuilding staging tables on every load does not scale
  - Incremental processing dramatically reduces compute cost
  - This layer is optimized for *analytics engineering*, not direct stakeholder access

---   

### 3. Reporting Tables (`rpt`)

Reporting tables are the **stakeholder facing analytics layer**.

**Characteristics**
  - Built from one or more staging tables
  - Highly aggregated
  - Primary keys are typically date based and/or high level dimensions (e.g. device type)
  - “All time” view with no partition expiry
  - Typically under 50GB, often **significantly** smaller
  - Optional clustering for dashboard filter performance; under 64MB its unneeded

**Responsibilities**
  - Serve dashboards, reports, and recurring analyses
  - Provide stable, trusted metrics
  - Eliminate the need for stakeholders to query large datasets

**Design Rationale**
  - Heavy aggregation drastically reduces storage requirements
  - Precomputing expensive logic keeps dashboards fast and predictable
  - Long term historical trends can be preserved without retaining raw events

---   

## Why This Architecture Works Well for Consumer Products

  - Handles **massive event volumes** without exploding costs
  - Keeps **dashboard latency low**, even at scale
  - Prevents BI tools from querying raw or semi raw data
  - Encourages metric consistency and reuse, preventing schema or metric drift
  - Scales cleanly as new features and data sources are added

This approach closely aligns with modern <a href="https://www.databricks.com/glossary/medallion-architecture" target="_blank" rel="noopener noreferrer">medallion style architectures</a> and <a href="https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/aggregate-fact-table-cube/" target="_blank" rel="noopener noreferrer">medallion style architectures</a>, adapted specifically for high volume product telemetry and performance data.

While this methodology was developed in a Google Cloud environment (BigQuery and Dataform), it is warehouse agnostic by design and translates cleanly to Snowflake with dbt, Databricks (Delta Lake), Amazon Redshift, Azure Synapse and other modern analytical platforms that support incremental processing and partition aware query optimization allowing for an equivalent to GCP style clustering.

---   

## Summary

This methodology emphasizes:
  - Incremental processing over full rebuilds
  - Aggregates over repeated computation, especially at the stakeholder facing report level
  - Clear data contracts between layers
  - Practical tradeoffs between data fidelity, cost, and usability

The result is an analytics system that is **fast, predictable, testable and sustainable**, even as product usage and data volumes grow.

*Velut arbor aevo!* 🌳
