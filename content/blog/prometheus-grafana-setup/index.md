---
title: "Setting Up Production Monitoring with Prometheus & Grafana from Scratch"
description: "A practical guide to building a monitoring stack that actually helps you find problems — covering golden signals, dashboard design, and alerting"
date: 2026-03-10
draft: false
tags: ["Prometheus", "Grafana", "Monitoring", "Alerting", "CloudWatch", "Observability"]
showTableOfContents: true
---

When I first set up monitoring for our production services, I made every mistake in the book — dashboards full of vanity metrics, alerts that fired constantly, and no clear path from "something is broken" to "here's what to fix."

This post covers what I learned about building a monitoring stack that actually helps you find and fix problems, based on running Prometheus and Grafana in production for over two years.

## Why We Needed It

Before centralized monitoring, our incident response process was:

1. User reports something is broken (usually via support ticket)
2. Engineer SSH's into production servers and manually checks logs
3. 2-3 hours of blind debugging later, someone finds the issue
4. Fix is deployed, and we hope it doesn't happen again

We had no visibility into application health, no alerting, and no unified view of system metrics. CloudWatch gave us basic EC2 metrics, but application-level insights were nonexistent.

## Architecture

{{< mermaid >}}
flowchart TB
    subgraph "Instrumented Services"
        A[API Servers\n/metrics endpoint]
        B[Background Workers\n/metrics endpoint]
        C[Node Exporter\nHost metrics]
        D[PostgreSQL Exporter]
        E[Kafka Exporter]
    end

    F[Prometheus\nScrape every 15s] --> A & B & C & D & E
    G[CloudWatch\nAWS-native metrics]

    F --> H[Grafana]
    G --> H
    F --> I[Alertmanager]
    I --> J[Slack]
    I --> K[Email]
{{< /mermaid >}}

Prometheus scrapes metrics from all instrumented services every 15 seconds. Grafana visualizes them. Alertmanager handles routing, deduplication, and notification delivery.

## What to Monitor: The Golden Signals

The biggest mistake I made early on was monitoring the wrong things. CPU usage, memory usage, and disk I/O are useful for capacity planning, but they're terrible for incident detection.

Google's SRE book defines four golden signals that actually tell you if your service is healthy:

### 1. Latency

**What:** How long requests take to process.
**How:** Prometheus histograms with bucket boundaries at key percentiles.
**Why:** A slow service feels broken to users even if it's technically "up."

Track p50 (median), p95, and p99 latency separately. The p99 is where the problems hide — 1% of your users might be having a terrible experience while your average looks fine.

### 2. Traffic

**What:** How many requests your service is handling.
**How:** Counter metric incremented on every request, broken down by endpoint and status code.
**Why:** Sudden traffic drops often mean something upstream is broken. Traffic spikes can predict saturation.

### 3. Errors

**What:** The rate of requests that fail.
**How:** HTTP 5xx response count as a percentage of total responses.
**Why:** This is the most direct signal that something is broken. A 5% error rate means 1 in 20 users is seeing failures.

### 4. Saturation

**What:** How "full" your service is.
**How:** Connection pool usage, queue depth, thread pool utilization.
**Why:** Saturation predicts future problems. A connection pool at 90% capacity will start failing requests soon.

## Dashboard Design That Actually Works

### The Three-Level Hierarchy

**Level 1: Infrastructure Overview**
One dashboard, all hosts. Green/yellow/red status indicators. This is what you check first during an incident.

**Level 2: Service Dashboards**
Per-service golden signals. API latency, error rates, throughput. This is where you drill down when the overview shows a problem.

**Level 3: Detail Dashboards**
Per-endpoint latency histograms, database query performance, Kafka consumer lag. This is where you diagnose root causes.

### Design Principles

- **Consistent time ranges** — Default to 6 hours for operational dashboards, 7 days for trend analysis
- **Red/yellow/green thresholds** — Make status instantly visible without reading numbers
- **Put the most important panels at the top** — Golden signals first, infrastructure metrics below
- **Use template variables** — One dashboard template that works for any service, filtered by a dropdown

## Alerting: The Hard Part

Setting up alerts is easy. Setting up alerts that people actually respond to is incredibly hard.

### The Alert Fatigue Problem

In my first iteration, I set up 40+ alert rules. Within a week, the team was ignoring all of them. Here's what I learned:

**Alert on symptoms, not causes.** "API latency > 500ms for 5 minutes" is actionable. "CPU > 80%" is not — CPU can spike during legitimate traffic bursts without affecting users.

**Every alert must have a runbook.** If the engineer receiving the alert doesn't know what to do next, the alert is useless. Each alert links to a remediation document with specific steps.

**Use severity levels ruthlessly:**
- **Critical** — Page immediately, user impact confirmed. We have fewer than 5 of these
- **Warning** — Investigate within 1 hour. These are leading indicators
- **Info** — Review during business hours. Capacity planning signals

**Tune aggressively.** After the initial setup, I spent two weeks adjusting thresholds based on false positive rates. If an alert fires more than twice without requiring action, the threshold is wrong.

### Example Alert Rules

```yaml
# High API latency
- alert: HighAPILatency
  expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "API p95 latency above 500ms"
    runbook: "https://wiki.internal/runbooks/high-latency"

# Error rate spike
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
  for: 3m
  labels:
    severity: critical
  annotations:
    summary: "5xx error rate above 5%"
    runbook: "https://wiki.internal/runbooks/error-rate"
```

## CloudWatch Integration

For AWS-managed services, CloudWatch provides metrics without any instrumentation. Rather than replacing CloudWatch with Prometheus, I use both:

- **CloudWatch** for EC2 instance metrics, Lambda invocations, S3 request counts, RDS performance
- **Prometheus** for application-level metrics, custom business metrics, Kafka and PostgreSQL exporters

Grafana's CloudWatch data source makes it trivial to put both on the same dashboard. This gives you a unified view without the overhead of exporting AWS metrics through Prometheus.

## Results

After six months of running this stack:

- **Incidents are detected in minutes** instead of hours — automated alerts replace user reports
- **Mean Time to Resolve dropped significantly** — dashboards + runbooks give engineers a clear investigation path
- **Proactive capacity planning** — trend dashboards show saturation approaching before it causes outages
- **The team trusts the monitoring** — fewer than 2 false positives per week means alerts are taken seriously

## Lessons Learned

1. **Start with the golden signals, not vanity metrics** — CPU usage alone is meaningless without application context
2. **Prometheus is not for long-term storage** — For retention beyond 2 weeks, use VictoriaMetrics or Thanos. Prometheus's local storage is designed for recent data
3. **Alert fatigue kills monitoring programs** — 5 well-tuned alerts are infinitely more valuable than 50 noisy ones
4. **Runbooks are as important as the alerts themselves** — An alert without remediation steps just creates panic
5. **Invest in dashboard templates** — Building one reusable template saves hours compared to creating per-service dashboards from scratch

---

Monitoring isn't about collecting metrics — it's about reducing the time between "something is broken" and "here's the fix." The tools are the easy part; the design decisions are what matter.
