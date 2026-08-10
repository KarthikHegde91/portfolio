---
title: "Synced and Healthy: The Six-Day Outage Every Dashboard Reported as Fine"
description: "A wedged SQLite lock, a health probe that sat in Git for three weeks without ever deploying, and a GitOps loop that had never actually been closed"
date: 2026-08-11
draft: false
tags: ["Incident Response", "Postmortem", "Kubernetes", "ArgoCD", "GitOps", "Grafana", "SRE", "Observability"]
showTableOfContents: true
---

For six days `grafana.karthikhegde.in` returned 502.

For those same six days ArgoCD reported the monitoring application as **Synced** and **Healthy**. Kubernetes reported the Grafana pod as **Running, 1/1, zero restarts**. And the fix — the correct fix, the one that would have resolved this in about sixty seconds — had been committed to the repository three weeks earlier.

None of those three facts is a bug on its own. Together they describe a platform that had no mechanism left to tell me it was broken.

This is the sequel to [The Node That Wouldn't Come Back](/blog/k3s-node-recovery/). That post ended with a list of follow-ups. This one is largely about what happened to them.

## What broke

The platform is a single-node K3s cluster on Oracle Cloud's free tier — ArgoCD, Prometheus, VictoriaMetrics, Grafana and Uptime Kuma, all reachable through a Cloudflare Tunnel.

The symptom was narrow. Grafana returned 502. The status page and the ArgoCD UI, routed through the *same* tunnel and the *same* `cloudflared` pod, both served fine. So the tunnel was up, the node was up, K3s was up. Whatever was wrong lived in one hop: `cloudflared` → `grafana.monitoring.svc:3000`.

## The clue was the clock, not the status code

A 502 from Cloudflare is not very specific. Two quite different failures produce it, and I needed to know which:

1. **The Service has no endpoints.** The pod is gone, crash-looping, or stuck starting.
2. **The pod is there but not accepting connections.**

I probed the endpoint twice. Both times: 502 after **exactly 30 seconds**.

That timing settles it, and it is worth understanding why.

If a Service has no endpoints, `kube-proxy` installs an iptables `REJECT` rule for its ClusterIP. A connection attempt comes back **refused, immediately** — single-digit milliseconds. You cannot get a 30-second stall out of hypothesis 1.

A listener whose accept backlog is full behaves differently. It does not send a RST. The kernel simply **drops the SYN**. The client retransmits, gets nothing, and eventually gives up — and `cloudflared`'s default `connectTimeout` is 30 seconds.

So the delay, not the status code, told me the pod was present and had stopped accepting connections. Confirmed from the node itself, where `curl` to both the pod IP and the ClusterIP hung and timed out rather than being refused, while the Service endpoint was correct and pointed at the running pod.

**Reading the shape of a failure is often more diagnostic than reading its error code.** Two hypotheses, one status code, and the clock discriminated between them.

## Root cause — a SQLite lock nobody was left to notice

Grafana keeps its state in SQLite. On disk:

```
grafana.db          2.3M   written 5 Aug
grafana.db-journal  4.6K   written 9 Aug     ← stale rollback journal
```

An orphaned journal beside a database that had not been written in days is the signature of a transaction that never completed. Everything behind that lock queued up. From Grafana's own logs:

```
"Failed to lock and execute cleanup old anon devices"
  error="[sqlstore.max-retries-reached] retry 1: database is locked (5) (SQLITE_BUSY)"

"Request Completed" ... duration=67h10m35s
"Request Completed" ... duration=81h47m55s
```

Individual HTTP requests running for **sixty to eighty hours**. Each one held a goroutine and its memory. The accept backlog filled, new connections started getting dropped, and the process settled into a state it could not leave.

It got worse from there. The wedged process climbed to its 1 GiB cgroup ceiling and stayed there. With no swap configured, the kernel could not push anything out, so it entered permanent reclaim — evicting page cache and re-reading it from disk, forever. On a node with a single OCPU, that was enough to take the whole box down with it:

| | While wedged |
|---|---|
| CPU idle | **0.0 %** |
| iowait | **66.7 %** |
| IO pressure (`full`, 300s avg) | **79.6** |

`full` pressure at 79 means *every* runnable task was stalled on IO roughly four-fifths of the time. For a while I was convinced I was looking at an infrastructure problem — a failing volume, a noisy neighbour, throttled IOPS. It was one application holding one lock. **A single wedged process is entirely capable of impersonating a hardware fault.**

## Why nothing self-healed

The live Deployment had no `livenessProbe`. No `readinessProbe`. No `startupProbe`.

Without probes, "Ready" means only that the container process started. Kubernetes had no question it could ask that this pod would have failed. It saw `Running 1/1, restarts: 0` for six days and, given what it had been told to check, that was an honest answer.

A `livenessProbe` on `/api/health` with a three-failure threshold would have restarted this pod inside a minute. Which brings us to the actual problem.

## The real root cause — the fix had been written, and could not deploy

Three weeks earlier, on 23 July, I had hit a *related* Grafana failure and fixed it properly. The commit:

```
fix(grafana): raise memory to 1Gi, add health probes, Recreate strategy

Grafana was OOMKilled under the 256Mi limit, so the pod stopped serving and
cloudflared hung on it -> grafana.karthikhegde.in timed out. [...] add
/api/health startup/readiness/liveness probes so a non-serving pod fails
fast and self-heals instead of hanging [...]
```

I had diagnosed the class of failure correctly and written exactly the right remedy. Then it never deployed.

Here is how I know. On 11 August the live Deployment carried that commit's **resource values precisely** — `limits` of 1Gi and 500m, `requests` of 256Mi and 100m — while carrying **none** of its probes and **not** its `Recreate` strategy. That combination appears in no commit in the repository's history. It could only have arisen one way: during the July incident I patched the *live object* by hand to stop the OOMKill, then wrote the durable fix into Git.

**The hotfix stuck. The real fix never landed. And nothing in the system compared the two.**

### Why it could not deploy

Two structural reasons, both invisible.

**ArgoCD's directory source is not recursive by default.** The monitoring Application pointed at `k8s/monitoring/`, which contains only `namespace.yaml` at its top level — everything real lives in `grafana/`, `prometheus/` and `victoriametrics/` subdirectories. Without `directory.recurse: true`, ArgoCD synced the namespace and nothing else.

It then reported **Synced** and **Healthy**, month after month, and it was telling the truth: the one file it had been asked to compare did match Git. The status was accurate and completely uninformative. **A green light that is structurally incapable of turning red is not a green light.**

**The app-of-apps root had never been applied.** The repository is organised so a single `root` Application watches `k8s/apps/`, and every file there defines a child Application. That root object did not exist in the cluster. The three children were static objects someone had applied by hand once, long ago.

So nothing watched `k8s/apps/`. Every change I made to those files — including turning `recurse` on — was structurally unable to reach the cluster. I could push the fix to the fix and it would sit there too.

{{< mermaid >}}
flowchart TB
    A["Grafana SQLite lock<br/>around 5 Aug"] --> B["Requests hang 60-80 hours"]
    B --> C["Accept backlog fills<br/>SYNs silently dropped"]
    C --> D["cloudflared connect timeout<br/>502 after exactly 30s"]

    B --> E["Process pinned at its 1GiB ceiling<br/>no swap, permanent reclaim"]
    E --> F["Node starved<br/>zero CPU idle, 67 percent iowait"]

    A --> G{"Why no self-heal?"}
    G --> H["No liveness probe<br/>on the live Deployment"]
    H --> I["The probe was committed on 23 Jul"]
    I --> J["recurse false<br/>subdirectories never synced"]
    I --> K["No root Application<br/>k8s/apps was unmanaged"]
    J --> L["ArgoCD reports<br/>Synced and Healthy"]
    K --> L
    L --> M["Six days, no signal"]
{{< /mermaid >}}

## Two traps found on the way out

Applying the manifests was not quite a matter of running `kubectl apply`. Reasoning through the merge semantics first turned up two latent failures — neither of which actually fired, because I defused them before applying. Both would have been ugly.

**The `Recreate` merge trap.** Git asked for `strategy.type: Recreate`. The live Deployment had been created with no `strategy` block at all, so the API server had defaulted it to `RollingUpdate` *plus* a `rollingUpdate: {maxSurge, maxUnavailable}` sub-object. Because `rollingUpdate` was never in `last-applied-configuration`, a three-way merge has no reason to remove it — the patch sets `type` and leaves the sub-object in place, producing an object that fails validation:

```
Deployment.apps "grafana" is invalid: spec.strategy.rollingUpdate:
Forbidden: may not be specified when strategy `type` is 'Recreate'
```

That would have failed the **entire Application**, not just that resource. Server-side apply does not rescue you either, since `rollingUpdate` is owned by the field manager that created the object rather than by the applier. It needs a one-time patch that nulls the sub-object in the same request:

```bash
kubectl -n monitoring patch deploy grafana \
  -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
```

**A rollout deadlock waiting in the other two workloads.** Prometheus and VictoriaMetrics each mount a ReadWriteOnce volume and each hold a single-writer lock on it. Neither declared a `strategy`, so both defaulted to `RollingUpdate` — and at `replicas: 1` that default is actively wrong. `maxUnavailable` of 25 % rounds **down to 0** and `maxSurge` rounds **up to 1**, so Kubernetes starts the new pod and waits for it to become Ready before stopping the old one.

Both pods land on the same node, so RWO admits them both and there is no `Multi-Attach` error to point at. The new process simply cannot take the lock and exits, and the readiness probes I was in the middle of adding would have guaranteed it could never satisfy `maxUnavailable: 0`. The result is not a slow rollout — it is a permanent one: a `CrashLoopBackOff` pod beside a healthy old one, `rollout status` hanging forever, ArgoCD stuck `Progressing`, and `remote_write` still working off the old pod so nothing ever alerts.

Note the asymmetry. The only Deployment that already had `Recreate` was Grafana, which holds neither lock. The two that needed it did not have it.

## Timeline

| Date | Event |
|---|---|
| **23 Jul** | Earlier Grafana OOMKill. Live object patched by hand; durable fix (probes, 1Gi, `Recreate`) committed to Git. |
| **~5 Aug** | SQLite transaction interrupted. Journal orphaned; writes to `grafana.db` stop. |
| **5–11 Aug** | Requests queue behind the lock, running 60–80 h each. Accept backlog fills; 502 on every request. Node reaches 0 % CPU idle. |
| **5–11 Aug** | ArgoCD continues reporting `Synced` / `Healthy`. Pod continues reporting `Running 1/1`, 0 restarts. |
| **11 Aug** | 30-second 502 timing isolates the failure to a wedged listener rather than a missing endpoint. |
| **11 Aug** | Live spec inspected: no probes, `RollingUpdate`, resources matching a commit whose probes never applied. |
| **11 Aug** | Root cause found — no `root` Application; `recurse: false`; the whole subtree unmanaged. |
| **11 Aug** | Strategy patched, manifests applied, pod replaced. SQLite rolled back the stale journal on startup and recovered. |
| **11 Aug** | `root` Application created. GitOps loop closed for the first time. |

## What changed

| | Before | After |
|---|---|---|
| Grafana health checking | None at all | `startup` / `readiness` / `liveness` on `/api/health` |
| Grafana CPU limit | `500m` CFS quota — throttles even on an idle core | No limit; request only, for proportional scheduling |
| ArgoCD scope | `k8s/monitoring` top level — one file | `recurse: true`, the whole subtree |
| Application objects | Hand-applied, unmanaged, undeletable drift | Managed by the `root` app-of-apps |
| Repo access | SSH deploy key on a public repo | Public HTTPS — nothing left to expire |
| Prometheus / VictoriaMetrics rollout | `RollingUpdate` onto an RWO lock | `Recreate` |
| Monitoring PVCs | Prunable once tracked | `Prune=false` |

One detail worth its own note: the recovery restart silently moved Grafana from 13.1.1 to 13.1.3, because all three images tracked `:latest`. Nothing in the repository changed — the tag had moved underneath it. Pinning those tags to the exact versions running, so that the deployed version is a property of the repository rather than of when a pod last happened to restart, was the first follow-up out of this incident.

Recovery itself was one pod replacement. Grafana came back in under a second at ~30 ms, `"database": "ok"`, memory at **313 MiB** — proof that the 983 MiB I had been staring at was the wedge, not the workload. Node CPU idle went from 0 % to 88 %.

## Lessons learned

- **"Synced" means "matches what I told it to look at."** It does not mean "matches Git." `directory.recurse` defaults to `false`, and for months the application compared exactly one file and was honestly green about it. When a status can only ever be green, it is decoration.
- **`Ready` without a probe just means the process started.** Kubernetes will faithfully report a pod that has been dead to the world for six days as healthy, because nobody gave it a question that could fail.
- **Committing a fix is not shipping a fix.** The single most useful check I was missing: *can the reconciler actually reach the file I just changed?* Mine could not, and nothing said so. I now verify that a change lands, not merely that it merged.
- **Hotfixes outlive the durable fixes meant to replace them.** The emergency patch went onto the live object and survived; the proper fix went into Git and never arrived. Nothing in the system compared them, so the two drifted apart silently for three weeks — until the missing half was exactly what was needed.
- **Read the timing, not just the error.** A refused connection and a dropped SYN both surface as 502. One is instant and one takes thirty seconds, and that difference pointed straight at the real failure mode.
- **One wedged process can convincingly impersonate an infrastructure failure.** 0 % CPU idle and 67 % iowait looked like a disk problem. It was a lock.
- **Reason about the merge before you apply.** Two latent failures — an invalid `Recreate` merge and a permanent RWO rollout deadlock — were sitting in a change I was about to push during an outage. Thinking through what the API server would actually do with the patch cost ten minutes and avoided both.

---

The uncomfortable part of this one is not the SQLite lock. Locks happen; that is why liveness probes exist. What it exposed is that my previous postmortem ended with a set of correct, committed follow-ups, and I never verified that a single one of them had reached the cluster. I had confused *merged* with *deployed*, and GitOps made that confusion very easy to sustain, because the dashboard kept saying `Synced`.

The green light was the failure. Everything else was just the incident it hid.

{{< button href="/blog/k3s-node-recovery/" target="_self" >}}
Read the previous incident
{{< /button >}}

All manifests discussed here are public: [github.com/KarthikHegde91/infra](https://github.com/KarthikHegde91/infra). For how this stack was built, see [Setting Up Production Monitoring with Prometheus & Grafana](/blog/prometheus-grafana-setup/) and the [monitoring stack case study](/projects/monitoring-stack/).
