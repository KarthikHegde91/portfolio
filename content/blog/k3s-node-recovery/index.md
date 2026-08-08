---
title: "The Node That Wouldn't Come Back: Recovering a K3s Cluster on Oracle's Free Tier"
description: "A five-day outage on free-tier ARM — capacity exhaustion, a cloud-init assumption that only holds on first boot, and a metrics target pinned to an IP that no longer existed"
date: 2026-08-05
draft: false
tags: ["Incident Response", "Postmortem", "K3s", "Oracle Cloud", "Prometheus", "cloud-init", "ArgoCD", "SRE"]
showTableOfContents: true
---

On 29 July my K3s node stopped responding. Rebuilding it should have been a ten-minute `terraform apply`. It took five days.

Not because the recovery itself was complicated, but because three assumptions I had never actually tested turned out to be wrong at the same time — and two of them failed *silently*. The cluster came back looking healthy while quietly missing half its telemetry. This is what broke, why each assumption held right up until it didn't, and what I changed so the same sequence can't repeat.

## What broke

The platform is a single-node K3s cluster on Oracle Cloud's Always-Free ARM tier, running ArgoCD, the Prometheus/VictoriaMetrics/Grafana stack, and Uptime Kuma behind a Cloudflare Tunnel. Single node means no HA — a node loss is a total outage. That is a deliberate trade-off of running on a free tier, and I knew it going in.

What I had not thought through was the recovery path. I assumed losing the node was survivable because everything is in code: `terraform apply`, wait for cloud-init, ArgoCD reconciles, done.

Every step of that assumption failed.

## The failure cascade

{{< mermaid >}}
flowchart TB
    A[Node unreachable<br/>29 Jul] --> B{Relaunch instance}
    B -->|Free-tier ARM capacity<br/>exhausted in region| C[Cannot rebuild<br/>on demand]
    C --> D[GitHub Actions retry loop<br/>polls until capacity frees]
    D --> E[Instance relaunched<br/>2 Aug]

    E --> F[SSH key absent]
    E --> G[Host metrics silent]
    E --> H[Monitoring app stuck]

    F -->|cloud-init runcmd is<br/>per-instance, not per-boot| F2[Fixed by injecting<br/>the key from bootcmd]
    G -->|scrape target pinned<br/>to the old private IP| G2[Fixed by Kubernetes API<br/>service discovery]
    H -->|leftover recovery Job<br/>blocking reconciliation| H2[Fixed by pruning<br/>the temporary artifacts]

    F2 --> Z[Cluster restored<br/>3 Aug]
    G2 --> Z
    H2 --> Z
{{< /mermaid >}}

The first failure was loud and obvious. The other three only surfaced *after* the node came back, which is the part worth paying attention to.

## Root cause 1 — free-tier capacity is not capacity you own

Oracle's Always-Free ARM tier gives you 4 vCPU and 24 GB of Ampere A1. It does not give you a *reservation*. When I tried to relaunch, the region had no free-tier ARM capacity available, and the API returned an out-of-capacity error. There is no queue and no ETA — you retry until someone else releases capacity.

The honest framing: I had been treating a best-effort allocation as though it were guaranteed infrastructure. Terraform can describe the instance perfectly and still be unable to create it.

Sitting there re-running `terraform apply` by hand was not a plan, so I wrote a temporary GitHub Actions workflow to do the polling for me — check for a running instance, and if there isn't one, attempt the launch.

That retry loop then produced two failures of its own, which is a useful reminder that recovery tooling written under pressure is still code, and still wrong the first time:

- **An empty instance count was not treated as zero.** The check for "do I already have an instance" returned an empty string rather than `0`, the conditional fell through, and the launch step never ran at all. The workflow reported success while doing nothing — the worst possible outcome for a retry loop.
- **The schedule didn't fire reliably.** GitHub Actions `schedule:` triggers are best-effort and get delayed or dropped under load. Waiting on a cron that might not run, to catch capacity that might last minutes, doesn't work. I replaced it with a single job that loops internally every three minutes for about five hours.

Capacity freed on 2 August and the instance came back.

## Root cause 2 — cloud-init runs per-instance, not per-boot

With the instance running, I couldn't SSH in. The key wasn't there.

The node is provisioned by `bootstrap/cloud-init.yaml`, which does its work in a `runcmd:` block. What I had internalised as "cloud-init runs at boot" is not quite what cloud-init does. Modules have a *frequency*, and `runcmd` defaults to `per-instance` — it runs once for a given instance ID and is skipped on every subsequent boot.

A relaunch is not a first boot. cloud-init saw an instance it considered already-configured, correctly decided there was nothing to do, and skipped the block entirely. No error, no warning. Exactly the behaviour it documents.

The fix is to run the key injection from `bootcmd`, which executes **every boot** rather than once per instance:

```yaml
# runcmd  → per-instance: runs once, skipped on relaunch
# bootcmd → per-boot:     runs every single boot
bootcmd:
  - <inject the authorized key here>
```

This is a nice example of a class of bug I now actively look for: **configuration that is only exercised on the happy path.** The `runcmd` block had worked perfectly for months, because in those months the instance was only ever created once. The defect was always there; nothing had asked the question.

## Root cause 3 — a hardcoded target is a hidden dependency on instance identity

The cluster was back, ArgoCD was syncing, Grafana was up. And host metrics were flatlined.

The rebuilt instance had come up with a different private IP. My Prometheus scrape config for `node_exporter` pointed at a static target — the old IP, baked in at setup time:

```yaml
      # Monitor host system via node_exporter
      - job_name: "node-exporter"
        static_configs:
          # Set to your K3s node's internal VCN IP, e.g. 10.0.x.x
          - targets: ["NODE_INTERNAL_IP:9100"]
```

Prometheus did exactly what it was told: it kept trying to scrape an address that no longer belonged to anything. The target went down, and because I had no alerting rules on scrape health at the time, nothing told me. CPU, memory, disk and network metrics for the host simply stopped arriving, and the dashboards showed a gap rather than an error.

The fix was to stop naming the node and start *discovering* it, using the Kubernetes API as the source of truth:

```yaml
      # Monitor host system via node_exporter — auto-discover the node's
      # current IP via the K8s API and scrape it on :9100, so this survives
      # instance relaunches (no hardcoded IP to go stale).
      - job_name: "node-exporter"
        scheme: http
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - source_labels: [__address__]
            regex: '([^:]+):.*'
            target_label: __address__
            replacement: '${1}:9100'
```

`kubernetes_sd_configs` with `role: node` asks the API server which nodes exist right now. The relabel rewrites whatever address comes back to point at port 9100. No IP appears anywhere in the config, so a future relaunch changes nothing.

The general principle: **anything you hardcode becomes an undeclared dependency on the thing it names.** A static IP in a scrape config is a dependency on instance identity, and it will hold until the first time that identity changes.

## Root cause 4 — recovery artifacts outliving the recovery

One more, smaller but instructive. The temporary `ssh-key-injector` Job I had applied to regain access was still sitting in the cluster. Because it was applied out-of-band and wasn't in Git, it conflicted with ArgoCD's desired state for the monitoring application and blocked that app from syncing.

The tool I used to fix the outage became the thing preventing the platform from returning to normal. Under GitOps, anything applied by hand is drift by definition — and the reconciler is right to complain about it.

## Timeline

| Date | Event |
|---|---|
| **29 Jul** | Node unreachable. Temporary `ssh-key-injector` Job applied to restore access. |
| **31 Jul** | Relaunch blocked — no free-tier ARM capacity in region. Wrote a GitHub Actions retry workflow to poll for capacity. |
| **31 Jul** | Retry workflow bug: empty instance count not treated as zero, so the launch step never executed. |
| **31 Jul** | Actions `schedule:` firing unreliably. Replaced with an internal 3-minute loop running ~5h per job. |
| **2 Aug** | Capacity freed. Clean relaunch, preserving data and node identity. |
| **2 Aug** | SSH key absent — `cloud-init` `runcmd` skipped as per-instance. Moved injection to `bootcmd`. |
| **2 Aug** | Removed the leftover `ssh-key-injector` Job, unblocking the monitoring app sync. |
| **3 Aug** | Host metrics found to be silent — scrape target pinned to the old IP. Switched to Kubernetes API service discovery. |
| **3 Aug** | Recovery workflows removed. Cluster restored and reconciling normally. |

## What changed

| | Before | After |
|---|---|---|
| Node key injection | `runcmd` — per-instance, skipped on relaunch | `bootcmd` — runs every boot |
| `node_exporter` target | Static private IP, stale after any rebuild | Kubernetes API service discovery, IP-independent |
| Capacity handling | Manual `terraform apply`, retried by hand | Understood as best-effort; retry is a known procedure |
| Recovery artifacts | Applied ad hoc, left running | Treated as drift, pruned as part of closing the incident |

## Lessons learned

- **A single node is a documented trade-off; an untested recovery path is not.** I knew a node loss meant an outage. I had never actually rebuilt from scratch, so I didn't know that rebuilding was where the real problems lived. Infrastructure-as-code proves you *can* describe the system, not that you can restore it.
- **Free-tier capacity is best-effort allocation, not reserved capacity.** Fine for a personal platform, as long as "the region might simply have nothing for me right now" is part of the plan rather than a surprise.
- **Read the frequency, not just the block.** `runcmd` is per-instance and `bootcmd` is per-boot. That distinction is documented, unremarkable, and invisible for as long as you only ever boot an instance once.
- **Silent failures are the expensive ones.** The node being down was obvious within minutes. Host metrics quietly not arriving could have gone unnoticed indefinitely, because a missing series looks like a gap in a graph rather than an error. The gap in my setup wasn't the stale IP — it was having **no alerting on scrape health**, which is now the top item on my list.
- **Hardcoding a value creates a dependency you never declared.** The static IP was a dependency on instance identity that nothing recorded and nothing tested.
- **Clean up the tools you used to recover.** Under GitOps, out-of-band fixes are drift. Leaving them behind means the reconciler stays unhappy long after the incident is nominally over.

---

None of the individual failures here were exotic. Each one is documented behaviour, and each fix is a few lines. What made it a five-day incident was that they were *layered* — capacity blocked the rebuild, the rebuild silently skipped provisioning, and the provisioning gap silently broke observability. You cannot see the third problem until you have solved the second.

That's the argument for actually exercising a restore rather than assuming one. All of this was recoverable precisely because the platform is defined in code — but "defined in code" and "verified to rebuild" are different claims, and only one of them had been tested.

{{< button href="/projects/zero-cost-infra/" target="_self" >}}
See the full infrastructure case study
{{< /button >}}

The manifests, Terraform and monitoring config discussed here are all public: [github.com/KarthikHegde91/infra](https://github.com/KarthikHegde91/infra). For how the detection stack was built in the first place, see [Setting Up Production Monitoring with Prometheus & Grafana](/blog/prometheus-grafana-setup/) and the [monitoring stack case study](/projects/monitoring-stack/).
