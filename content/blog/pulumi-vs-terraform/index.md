---
title: "Pulumi vs Terraform: What I Learned After Using Both in Production"
description: "A practical comparison from someone who's used both IaC tools in production — when to choose each and how to migrate"
date: 2026-03-06
draft: false
tags: ["Pulumi", "Terraform", "Infrastructure as Code", "IaC", "DevOps"]
showTableOfContents: true
---

I started with Terraform because everyone recommended it. I switched to Pulumi because I needed something more flexible. After using both in production for real infrastructure, here's what I wish someone had told me from the start.

This isn't a "Pulumi is better than Terraform" post. Both are excellent tools. The right choice depends on your team, your infrastructure complexity, and how you think about code.

## Context

My infrastructure spans multiple cloud providers (DigitalOcean, GCP, AWS) with Kubernetes clusters, managed databases, DNS, SSL certificates, and CI/CD pipelines. I started with Terraform for basic provisioning and gradually migrated to Pulumi as the infrastructure grew more complex.

## Terraform: The Industry Standard

### What It Does Well

**Massive ecosystem.** Terraform has providers for virtually every cloud service, SaaS product, and infrastructure component. If it has an API, there's probably a Terraform provider for it.

**HCL is purpose-built for infrastructure.** HashiCorp Configuration Language is declarative by design. You describe what you want, and Terraform figures out how to get there:

```hcl
resource "digitalocean_droplet" "web" {
  image  = "ubuntu-22-04-x64"
  name   = "web-server"
  region = "blr1"
  size   = "s-2vcpu-4gb"

  tags = ["web", "production"]
}

resource "digitalocean_firewall" "web" {
  name        = "web-firewall"
  droplet_ids = [digitalocean_droplet.web.id]

  inbound_rule {
    protocol         = "tcp"
    port_range       = "443"
    source_addresses = ["0.0.0.0/0"]
  }
}
```

**Plan before apply.** `terraform plan` shows you exactly what will change before making any modifications. This safety net is invaluable in production.

**State management is well-understood.** Remote state backends (S3, GCS, Terraform Cloud) are battle-tested with locking and versioning.

### Where It Struggles

**Limited programming constructs.** Need a for-loop with conditional logic? HCL's `for_each` and `count` work for simple cases, but complex dynamic infrastructure becomes unwieldy:

```hcl
# This gets messy fast
resource "aws_security_group_rule" "ingress" {
  for_each = {
    for rule in var.ingress_rules :
    "${rule.port}-${rule.protocol}" => rule
    if rule.enabled
  }
  # ...
}
```

**Module reuse is clunky.** Terraform modules are powerful but passing variables between modules and handling conditional module inclusion requires verbose workarounds.

**State file management.** The state file is a single point of failure. State drift, state corruption, and state conflicts in team environments are real problems that require careful process discipline.

## Pulumi: Infrastructure as Real Code

### What It Does Well

**Real programming languages.** Pulumi lets you write infrastructure in Python, TypeScript, Go, or C#. This means you get loops, conditionals, functions, classes, and the full standard library:

```python
import pulumi
import pulumi_digitalocean as do

# Create servers dynamically based on configuration
environments = ["staging", "production"]

for env in environments:
    droplet = do.Droplet(
        f"web-{env}",
        image="ubuntu-22-04-x64",
        name=f"web-{env}",
        region="blr1",
        size="s-2vcpu-4gb" if env == "staging" else "s-4vcpu-8gb",
        tags=["web", env],
    )

    do.Firewall(
        f"fw-{env}",
        name=f"{env}-firewall",
        droplet_ids=[droplet.id],
        inbound_rules=[
            do.FirewallInboundRuleArgs(
                protocol="tcp",
                port_range="443",
                source_addresses=["0.0.0.0/0"],
            )
        ],
    )
```

This is natural Python code. No new syntax to learn, no DSL limitations to work around.

**Better abstractions.** You can create reusable components as actual classes with encapsulation, inheritance, and composition — much more powerful than Terraform modules:

```python
class WebServer(pulumi.ComponentResource):
    def __init__(self, name, env, size, opts=None):
        super().__init__("custom:WebServer", name, {}, opts)

        self.droplet = do.Droplet(...)
        self.firewall = do.Firewall(...)
        self.dns = cloudflare.Record(...)

        self.register_outputs({"ip": self.droplet.ipv4_address})
```

**Built-in state management.** Pulumi Cloud handles state by default — no S3 bucket to configure, no locking to set up. You can also self-manage state if needed.

**Strong typing.** With TypeScript or Python type hints, your IDE catches infrastructure mistakes before you even run the code.

### Where It Struggles

**Smaller community.** Terraform has been around longer and has more Stack Overflow answers, blog posts, and copy-pasteable examples.

**Learning curve for non-developers.** If your team includes ops engineers who are comfortable with YAML/HCL but not Python/TypeScript, Pulumi's "everything is code" approach can be a barrier.

**Provider parity.** While Pulumi can use Terraform providers via a bridge, some providers lag behind their Terraform counterparts in documentation and edge-case support.

## Side-by-Side: The Same Infrastructure

Here's the same infrastructure defined in both tools — a Kubernetes namespace with resource quotas:

### Terraform

```hcl
resource "kubernetes_namespace" "app" {
  metadata {
    name = "my-app"
    labels = {
      environment = var.environment
      managed-by  = "terraform"
    }
  }
}

resource "kubernetes_resource_quota" "app" {
  metadata {
    name      = "app-quota"
    namespace = kubernetes_namespace.app.metadata[0].name
  }
  spec {
    hard = {
      "requests.cpu"    = "4"
      "requests.memory" = "8Gi"
      "limits.cpu"      = "8"
      "limits.memory"   = "16Gi"
    }
  }
}
```

### Pulumi (Python)

```python
import pulumi_kubernetes as k8s

ns = k8s.core.v1.Namespace(
    "app",
    metadata=k8s.meta.v1.ObjectMetaArgs(
        name="my-app",
        labels={
            "environment": env,
            "managed-by": "pulumi",
        },
    ),
)

quota = k8s.core.v1.ResourceQuota(
    "app-quota",
    metadata=k8s.meta.v1.ObjectMetaArgs(
        name="app-quota",
        namespace=ns.metadata.name,
    ),
    spec=k8s.core.v1.ResourceQuotaSpecArgs(
        hard={
            "requests.cpu": "4",
            "requests.memory": "8Gi",
            "limits.cpu": "8",
            "limits.memory": "16Gi",
        },
    ),
)
```

For simple resources like this, both are roughly equivalent. The difference shows up when you need conditional logic, dynamic resource generation, or complex dependencies.

## When I'd Choose Each

### Choose Terraform When:

- **Your team is ops-heavy** and more comfortable with declarative config than programming
- **You're using well-trodden paths** — standard AWS/GCP/Azure setups where Terraform modules already exist
- **You need maximum community support** — more examples, more modules, more answered questions
- **Simple infrastructure** — if `for_each` and `count` handle your dynamic needs, Terraform's simplicity is a strength

### Choose Pulumi When:

- **Your team writes code daily** — developers who already think in Python/TypeScript will be productive immediately
- **Complex infrastructure logic** — conditional resources, dynamic generation from config files, custom validation
- **Reusable components** — when you need proper abstractions that go beyond Terraform modules
- **Multi-cloud with shared patterns** — defining common infrastructure patterns as typed classes that work across providers

## Migration Tips

If you're considering moving from Terraform to Pulumi:

1. **Don't migrate everything at once.** Start with new infrastructure in Pulumi while keeping existing Terraform resources
2. **Pulumi can import Terraform state.** Use `pulumi import` to bring existing resources under Pulumi management without recreating them
3. **Pulumi's Terraform bridge** lets you use Terraform providers directly — you don't lose access to the ecosystem
4. **Both can coexist.** It's perfectly valid to use Terraform for some infrastructure and Pulumi for others. Use the right tool for each job

---

The IaC landscape isn't about picking a winner — it's about understanding the trade-offs. I use Pulumi as my primary tool because my work involves complex, dynamic infrastructure where real programming constructs save significant time. But I'd reach for Terraform in a heartbeat for a straightforward cloud setup where modules already exist.

The worst choice is no IaC at all. Pick either one, and you're already ahead of clicking through cloud consoles.
