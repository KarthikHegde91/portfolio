---
title: "Kafka & Data Pipeline"
description: "Production event streaming, log aggregation, and analytics data pipeline with Apache Kafka, PostgreSQL, Redis, and BigQuery"
date: 2026-03-11
draft: false
tags: ["Apache Kafka", "PostgreSQL", "Redis", "BigQuery", "Data Pipeline"]
showTableOfContents: true
---

## Overview

Production-grade data pipeline handling **event streaming**, **log aggregation**, and **analytics data flow** across multiple services — processing real-time events and feeding them into PostgreSQL for operational data and BigQuery for analytics.

{{< badge >}}Apache Kafka{{< /badge >}}
{{< badge >}}PostgreSQL{{< /badge >}}
{{< badge >}}Redis{{< /badge >}}
{{< badge >}}BigQuery{{< /badge >}}
{{< badge >}}Docker{{< /badge >}}

---

## The Problem

As the application grew, several data challenges emerged:

- **Tight coupling between services** — Direct API calls between services created cascading failures
- **No real-time event processing** — Changes in one system took minutes to propagate to others
- **Analytics queries impacting production** — Running analytics on the production database caused performance degradation
- **Log data was scattered** — No centralized way to aggregate and search logs across services

## Architecture

{{< mermaid >}}
flowchart TB
    subgraph "Producers"
        A[Application Services]
        B[Backend APIs]
        C[Background Workers]
    end

    subgraph "Kafka Cluster"
        D[Topic: events]
        E[Topic: logs]
        F[Topic: analytics]
    end

    subgraph "Consumers"
        G[Event Processor]
        H[Log Aggregator]
        I[Analytics Pipeline]
    end

    subgraph "Storage"
        J[(PostgreSQL\nOperational DB)]
        K[(Redis\nCache Layer)]
        L[(BigQuery\nAnalytics Warehouse)]
    end

    A --> D
    B --> D & E
    C --> E & F

    D --> G --> J
    E --> H --> J
    F --> I --> L

    G --> K
    J --> |Replication| J2[(PostgreSQL\nReplica)]
{{< /mermaid >}}

## Kafka Setup

### Cluster Configuration

- **Multi-broker setup** for fault tolerance
- **Topic partitioning** designed around consumer throughput requirements
- **Retention policies** tuned per topic — events retained for 7 days, logs for 30 days, analytics for 3 days (already persisted to BigQuery)

### Topic Design

| Topic | Partitions | Purpose | Retention |
|-------|-----------|---------|-----------|
| `events` | Based on consumer count | Service-to-service event streaming | 7 days |
| `logs` | Based on throughput | Centralized log aggregation | 30 days |
| `analytics` | Optimized for BigQuery sink | Analytics event stream | 3 days |

## Data Flow

### Event Streaming
Services publish domain events (user actions, state changes, system events) to Kafka. Consumer services pick up events asynchronously — decoupling the producer from downstream processing.

### Log Aggregation
Application logs are published to a dedicated Kafka topic, consumed by a log aggregator that structures and stores them for search and analysis.

### Analytics Pipeline
High-value analytics events flow through Kafka into BigQuery, where they power dashboards and business intelligence queries — completely isolated from the production database.

## PostgreSQL

- **Primary-replica replication** for read scaling and disaster recovery
- **Automated backups** with tested restore procedures
- **Connection pooling** to handle high-concurrency workloads

## Redis

- **Caching layer** for frequently accessed data — reducing PostgreSQL load
- **Session management** for application state
- **Rate limiting** counters for API endpoints

## Monitoring

Key Kafka metrics tracked:

- **Consumer lag** — How far behind consumers are from the latest offset
- **Throughput** — Messages per second across topics
- **Broker health** — Disk usage, replication status, partition leadership
- **End-to-end latency** — Time from produce to consume

## Lessons Learned

- **Partition count matters more than you think** — Under-partitioned topics become bottlenecks; over-partitioned topics waste resources. Start conservative, scale up based on consumer throughput
- **Consumer group management is critical** — Rebalancing during deployments can cause message processing delays; use cooperative rebalancing where possible
- **Separate analytics from production early** — Running BigQuery-style queries on PostgreSQL will eventually bring production down. Kafka makes the separation clean
- **Monitor consumer lag religiously** — It's the first signal that something is wrong in the pipeline
- **Redis is not a database** — Use it for caching and ephemeral data, not as a primary store. Always have a fallback to PostgreSQL
