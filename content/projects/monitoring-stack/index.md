---
title: "Monitoring & Observability Stack"
description: "Full-stack production monitoring with Prometheus, Grafana, CloudWatch, and alerting covering application and infrastructure metrics"
date: 2026-03-10
draft: false
tags: ["Prometheus", "Grafana", "CloudWatch", "VictoriaMetrics", "Alerting", "Observability"]
showTableOfContents: true
---

## Overview

End-to-end monitoring and observability stack covering **application metrics**, **infrastructure metrics**, and **alerting** — transforming incident response from reactive firefighting to proactive detection.

{{< badge >}}Prometheus{{< /badge >}}
{{< badge >}}Grafana{{< /badge >}}
{{< badge >}}CloudWatch{{< /badge >}}
{{< badge >}}VictoriaMetrics{{< /badge >}}
{{< badge >}}Alertmanager{{< /badge >}}

---

## The Problem

Before implementing centralized monitoring:

- **No visibility** into application health — issues were discovered by users, not engineers
- **Firefighting mode** — Every incident was reactive, often taking hours to diagnose
- **Scattered metrics** — Some in CloudWatch, some in application logs, no unified view
- **No alerting** — Manual checking of dashboards (when someone remembered to look)

## Architecture

{{< mermaid >}}
flowchart TB
    subgraph "Targets"
        A[Application Services\nAPI latency, errors, requests]
        B[Infrastructure\nCPU, memory, disk, network]
        C[Databases\nPostgreSQL, Redis]
        D[Kafka\nConsumer lag, throughput]
    end

    subgraph "Collection"
        E[Prometheus\nScrape & Store]
        F[CloudWatch\nAWS-native metrics]
    end

    subgraph "Visualization"
        G[Grafana\nDashboards]
    end

    subgraph "Alerting"
        H[Alertmanager\nRouting & Dedup]
        I[Slack / Email\nNotifications]
    end

    A & B & C & D --> E
    E --> G
    F --> G
    E --> H --> I
{{< /mermaid >}}

## Application Monitoring

### The Four Golden Signals

Following Google's SRE golden signals framework:

| Signal | What We Track | Why It Matters |
|--------|--------------|----------------|
| **Latency** | API response times (p50, p95, p99) | Detects slowdowns before users complain |
| **Traffic** | Requests per second by endpoint | Capacity planning, anomaly detection |
| **Errors** | HTTP 5xx rates, exception counts | Immediate signal that something is broken |
| **Saturation** | Queue depth, connection pool usage | Predicts problems before they hit |

### Key Application Metrics

- **API endpoint latency** — Histogram buckets for p50/p95/p99 response times
- **Error rates** — 5xx errors as a percentage of total requests
- **Request throughput** — Requests per second, broken down by endpoint and status code
- **Health checks** — Synthetic probes confirming service availability

## Infrastructure Monitoring

- **CPU utilization** — Per-core usage with alerting at 80% sustained
- **Memory** — Used vs available, swap usage (swap should always be near zero)
- **Disk** — Usage percentage, I/O wait, predictive alerts for disk full
- **Network** — Bandwidth, packet loss, connection counts

## Dashboard Design

### Principles

1. **Start with the golden signals** — Every service dashboard has latency, traffic, errors, saturation as the top row
2. **Use consistent time ranges** — Default to 6 hours for operational dashboards, 7 days for trends
3. **Red/Yellow/Green encoding** — Thresholds make status instantly visible
4. **Drill-down capability** — Overview → service → endpoint → individual request

### Dashboard Hierarchy

```
Infrastructure Overview (all hosts)
  └── Service Dashboard (per-service golden signals)
       └── Endpoint Detail (latency histograms, error breakdown)
            └── Kafka Dashboard (consumer lag, throughput)
                 └── Database Dashboard (connections, query performance)
```

## Alerting

### Alert Design Principles

- **Alert on symptoms, not causes** — "API latency > 500ms" not "CPU > 80%"
- **Include runbook links** — Every alert has a link to the remediation steps
- **Use severity levels** — Critical (page immediately), Warning (investigate within 1 hour), Info (review during business hours)
- **Avoid alert fatigue** — Tune thresholds aggressively; a noisy alert is worse than no alert

### Example Alert Rules

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| High API Latency | p95 > 500ms for 5 min | Warning | Check backend services |
| Error Rate Spike | 5xx > 5% of traffic for 3 min | Critical | Immediate investigation |
| Disk Almost Full | > 85% usage | Warning | Clean up or expand |
| Kafka Consumer Lag | > 10,000 messages for 10 min | Warning | Check consumer health |

## CloudWatch Integration

For AWS-managed services (EC2, S3, Lambda, RDS), CloudWatch provides native metrics without any instrumentation:

- **EC2** — CPU, network, disk I/O (complements node_exporter)
- **Lambda** — Invocation count, duration, errors, throttles
- **S3** — Request metrics, bucket size
- **RDS** — Database connections, read/write latency

Grafana's CloudWatch data source unifies these with Prometheus metrics on a single dashboard.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Mean Time to Detect (MTTD) | Hours (user reports) | Minutes (automated alerts) |
| Mean Time to Resolve (MTTR) | Hours (blind debugging) | Minutes (dashboards + runbooks) |
| Incidents discovered by | Users | Monitoring system |
| Dashboard coverage | None | All services and infrastructure |
| Alert response | Manual checking | Automated notifications to Slack/email |

## Lessons Learned

- **Start with the golden signals, not vanity metrics** — CPU usage alone tells you nothing; API latency tells you everything
- **Prometheus is not a long-term storage solution** — For retention beyond 2 weeks, use VictoriaMetrics or Thanos
- **Alert fatigue is real and dangerous** — 50 noisy alerts means all 50 get ignored. Be ruthless about tuning thresholds
- **Runbooks are as important as alerts** — An alert without a remediation guide just creates panic
- **CloudWatch and Prometheus complement each other** — Don't try to replace one with the other; use both for their strengths
