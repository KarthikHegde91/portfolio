---
title: "$0/Month Portfolio Infrastructure"
description: "Production-grade DevOps portfolio with K3s, ArgoCD, Grafana, and monitoring — running entirely on free-tier services"
date: 2026-03-12
draft: false
tags: ["Hugo", "Cloudflare Pages", "Oracle Cloud", "K3s", "ArgoCD", "Terraform", "GitOps"]
showTableOfContents: true
---

## Overview

A production-grade DevOps portfolio and infrastructure showcase running at **$0/month** — proving that great DevOps is about smart architectural decisions, not budget.

{{< badge >}}Hugo{{< /badge >}}
{{< badge >}}Cloudflare Pages{{< /badge >}}
{{< badge >}}Oracle Cloud{{< /badge >}}
{{< badge >}}K3s{{< /badge >}}
{{< badge >}}ArgoCD{{< /badge >}}
{{< badge >}}Terraform{{< /badge >}}
{{< badge >}}GitHub Actions{{< /badge >}}

---

## Architecture

The infrastructure uses a **two-tier architecture** — static frontend on Cloudflare's CDN, and a live infrastructure layer on Oracle Cloud's free tier.

{{< mermaid >}}
flowchart TB
    subgraph "Tier 1: Static Site ($0)"
        A[Hugo + Blowfish] --> B[Cloudflare Pages\nGlobal CDN]
        B --> C[karthikhegde.in]
    end

    subgraph "Tier 2: Live Infrastructure ($0)"
        D[Oracle Cloud ARM\n4 CPUs / 24GB RAM] --> E[K3s Cluster]
        E --> F[ArgoCD]
        E --> G[Grafana]
        E --> H[VictoriaMetrics]
        E --> I[Uptime Kuma]
    end

    J[GitHub Actions] --> |Build & Push| K[GHCR]
    K --> |Auto-sync| F
    F --> E

    L[Cloudflare Tunnel] --> D

    subgraph "External Monitoring ($0)"
        M[UptimeRobot\n50 monitors]
    end
{{< /mermaid >}}

## Cost Breakdown

| Service | Provider | What It Does | Monthly Cost |
|---------|----------|-------------|-------------|
| Static hosting + CDN | Cloudflare Pages | Hugo site with global edge caching | $0 |
| DNS + SSL | Cloudflare | DNS management, free SSL certificates | $0 |
| Email forwarding | Cloudflare Email Routing | hello@karthikhegde.in → Gmail | $0 |
| Compute (4 ARM CPUs, 24GB RAM) | Oracle Cloud Always Free | K3s, ArgoCD, Grafana, monitoring | $0 |
| Container registry | GitHub Container Registry | Docker image storage (public repos) | $0 |
| CI/CD pipelines | GitHub Actions | Build, test, deploy automation | $0 |
| External monitoring | UptimeRobot | 50 monitors, 5-minute intervals | $0 |
| **Total** | | | **$0/month** |

The only cost is the domain registration: **~$10-12/year** for `karthikhegde.in`.

## Static Site Stack

- **Hugo** — Go-based static site generator (builds in <300ms)
- **Blowfish theme** — Tailwind CSS, dark mode, Mermaid diagrams, search, 100/100 Lighthouse
- **Custom cyberpunk theme** — Orbitron + Fira Code fonts, neon glow effects, glassmorphism, glitch animation
- **Cloudflare Pages** — Auto-deploys on git push, 500 builds/month, global CDN

## Infrastructure Stack

- **Oracle Cloud Always Free** — 4 ARM Ampere cores, 24GB RAM, 200GB disk — permanently free
- **K3s** — Lightweight Kubernetes distribution that fits comfortably in the free tier
- **ArgoCD** — GitOps controller with web UI — watches the Git repo and auto-syncs deployments
- **Cloudflare Tunnel** — Outbound-only encrypted tunnel, zero open ports on the VM

## Monitoring Stack

- **VictoriaMetrics** — Prometheus-compatible TSDB using 7x less RAM
- **Grafana** — Dashboards for infrastructure and application metrics
- **Uptime Kuma** — Self-hosted status page at `status.karthikhegde.in`
- **UptimeRobot** — External monitoring as a second pair of eyes (free tier: 50 monitors)

## CI/CD Pipeline

{{< mermaid >}}
flowchart LR
    A[Git Push] --> B[GitHub Actions]
    B --> C[Hugo Build]
    B --> D[Docker Build]
    C --> E[Cloudflare Pages\nAuto-deploy]
    D --> F[Push to GHCR]
    F --> G[ArgoCD Detects\nNew Image]
    G --> H[K3s Rollout]
{{< /mermaid >}}

## Lessons Learned

- **Oracle Cloud's free tier is the real deal** — 4 ARM CPUs and 24GB RAM is enough to run K3s + ArgoCD + Grafana + VictoriaMetrics + Uptime Kuma simultaneously
- **Cloudflare Tunnel eliminates firewall complexity** — No ports to open, no IP allowlisting
- **Hugo builds are incredibly fast** — Full site builds in under 300ms, Cloudflare Pages deploys in under 30 seconds
- **The `.io` TLD has sovereignty risk** — Research found Mauritius transfer creates TLD retirement risk; `.in` is a safe choice for Indian developers
