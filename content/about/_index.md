---
title: "About"
description: "Karthik B Hegde — DevOps Engineer"
showDate: false
showReadingTime: false
showWordCount: false
---

## Who I Am

DevOps Engineer with 3+ years of hands-on experience at **Appsndevices Technologies**, building and automating infrastructure for mobile and enterprise SaaS products. I manage **daily deployments across 4 platforms** (Android, iOS, Windows, macOS), maintain cloud infrastructure across **multiple providers**, and build the CI/CD pipelines that a team of **10+ engineers** relies on every day.

MCA graduate from **B.M.S College of Engineering, Bengaluru** (2020–2022).

---

## Professional Experience

### DevOps Engineer — Appsndevices Technologies Pvt. Ltd.
**Nov 2022 – Present** | Bengaluru, India

**CI/CD & Release Engineering**
- Designed and maintained CI/CD pipelines primarily using **GitHub Actions** (primary) and **Jenkins** — shipping daily releases to **Android, iOS, Windows, and macOS** from a single codebase
- Built automated deployment workflows serving a team of **10+ engineers**, reducing manual deployment effort and improving release consistency across all platforms
- Managed secrets across pipelines using **GitHub Secrets**, **Infisical**, and **Google Secret Manager**

**Infrastructure & Cloud**
- Managed cloud infrastructure across **DigitalOcean** (Droplets, networking, S3-compatible storage), **Google Cloud Platform** (BigQuery, data streaming, Compute Engine), and **AWS** (EC2, S3, IAM, VPC, Lambda)
- Provisioned infrastructure using **Pulumi** (primary IaC tool), with experience in **Terraform** and **Ansible** for reproducible, version-controlled environments
- Performed **cloud cost optimization** — right-sizing instances, cleaning up unused resources, and reducing infrastructure spend

**Networking & Security**
- Configured **Nginx reverse proxies** for application routing and load distribution
- Managed **DNS records** across Cloudflare and cloud provider DNS services
- Set up and automated **SSL/TLS certificates** (Let's Encrypt) for secure HTTPS across services
- Configured **firewalls and security groups** — iptables, UFW, and cloud provider security lists/ACLs
- Managed **IAM and RBAC policies** across cloud providers for least-privilege access control

**Containers & Orchestration**
- Managed **Docker-based build environments** and **Kubernetes deployments** ensuring scalable and stable application releases
- Deployed and maintained multiple containerized services in production

**Data & Messaging**
- Managed **Apache Kafka clusters** for distributed event streaming — troubleshooting replication, connector, and consumer pipeline issues
- Worked with **PostgreSQL** including logical replication using dump/restore extensions
- Used **Redis** for caching and application performance optimization
- Managed **Google BigQuery** for data analytics and streaming pipelines

**Monitoring, Logging & Reliability**
- Set up and maintained **Prometheus + Grafana** monitoring stacks — configured exporters, built dashboards, defined alert rules, and used them to identify performance bottlenecks in production
- Used **CloudWatch** and cloud-native logging for centralized log management
- Built **disaster recovery and backup automation** — automated database backups, tested restore procedures, and maintained DR documentation
- Authored **operational runbooks and documentation** for incident response and system maintenance
- Supported production environments through **on-call rotations**, resolving CI/CD and infrastructure-related incidents with root cause analysis
- Managed **cron jobs and scheduled automation** for maintenance tasks, cleanup, and health checks

**AI-Assisted Engineering**
- Hands-on practitioner of **Claude Code** for agentic coding workflows — using AI agents for scripting, debugging, configuration generation, code review, and operational analysis
- Experience with **prompt engineering** and **agentic coding patterns** — designing effective prompts for complex multi-step automation tasks, validating AI outputs for correctness and reliability
- Early adopter of AI-assisted DevOps: using AI tools not as a crutch, but as a force multiplier for engineering velocity

**Process & Collaboration**
- Collaborated with development, QA, and product teams in **Agile (Scrum/Kanban)** workflows
- Participated in sprint planning, delivering DevOps automation improvements within sprint cycles

---

## Technical Skills

### CI/CD & Release Engineering
{{< badge >}}GitHub Actions (primary){{< /badge >}}
{{< badge >}}Jenkins{{< /badge >}}
{{< badge >}}Multi-platform releases (Android/iOS/Windows/macOS){{< /badge >}}

### Containers & Orchestration
{{< badge >}}Docker{{< /badge >}}
{{< badge >}}Kubernetes{{< /badge >}}
{{< badge >}}ArgoCD{{< /badge >}}

### Infrastructure as Code
{{< badge >}}Pulumi (primary){{< /badge >}}
{{< badge >}}Terraform{{< /badge >}}
{{< badge >}}Ansible{{< /badge >}}

### Cloud Platforms
{{< badge >}}DigitalOcean{{< /badge >}}
{{< badge >}}Google Cloud Platform{{< /badge >}}
{{< badge >}}AWS{{< /badge >}}
{{< badge >}}Oracle Cloud{{< /badge >}}
{{< badge >}}Cloudflare{{< /badge >}}

### Networking & Security
{{< badge >}}Nginx (reverse proxy){{< /badge >}}
{{< badge >}}DNS Management{{< /badge >}}
{{< badge >}}SSL/TLS (Let's Encrypt){{< /badge >}}
{{< badge >}}Firewalls / Security Groups{{< /badge >}}
{{< badge >}}IAM / RBAC{{< /badge >}}

### Secrets Management
{{< badge >}}GitHub Secrets{{< /badge >}}
{{< badge >}}Infisical{{< /badge >}}
{{< badge >}}Google Secret Manager{{< /badge >}}

### Programming & Scripting
{{< badge >}}Python{{< /badge >}}
{{< badge >}}Bash{{< /badge >}}
{{< badge >}}Linux (Ubuntu, RHEL){{< /badge >}}
{{< badge >}}Cron / Scheduled Automation{{< /badge >}}

### Monitoring & Observability
{{< badge >}}Prometheus (setup + maintenance){{< /badge >}}
{{< badge >}}Grafana (dashboards + alerting){{< /badge >}}
{{< badge >}}CloudWatch{{< /badge >}}
{{< badge >}}VictoriaMetrics{{< /badge >}}
{{< badge >}}Uptime Kuma{{< /badge >}}

### Data & Messaging
{{< badge >}}Apache Kafka{{< /badge >}}
{{< badge >}}PostgreSQL (replication){{< /badge >}}
{{< badge >}}Redis{{< /badge >}}
{{< badge >}}BigQuery{{< /badge >}}

### AI-Assisted Engineering
{{< badge >}}Claude Code (agentic coding){{< /badge >}}
{{< badge >}}Prompt Engineering{{< /badge >}}
{{< badge >}}AI-Assisted DevOps Workflows{{< /badge >}}

### Operational Excellence
{{< badge >}}Disaster Recovery / Backup Automation{{< /badge >}}
{{< badge >}}Cost Optimization{{< /badge >}}
{{< badge >}}Runbooks / Documentation{{< /badge >}}
{{< badge >}}On-Call / Incident Response{{< /badge >}}

### Tools & Workflow
{{< badge >}}Git{{< /badge >}}
{{< badge >}}JIRA{{< /badge >}}
{{< badge >}}Scrum / Kanban{{< /badge >}}

---

## Education

**Master of Computer Applications (MCA)**
B.M.S College of Engineering, Bengaluru — 2020–2022

---

## Certifications

- **Docker Certified Associate (DCA)** — *In Progress*

---

## This Site Is the Portfolio

This website is itself a DevOps project. The infrastructure behind it demonstrates the skills listed above:

{{< mermaid >}}
graph LR
    A[Git Push] --> B[GitHub Actions]
    B --> C[Build Hugo]
    C --> D[Cloudflare Pages CDN]
    D --> E[karthikhegde.in]

    F[Git Push] --> G[GitHub Actions]
    G --> H[Build Docker Image]
    H --> I[GHCR]
    I --> J[ArgoCD]
    J --> K[K3s on Oracle Cloud]
    K --> L[Grafana Dashboards]
    K --> M[Uptime Kuma]
    K --> N[VictoriaMetrics]

    O[Terraform] --> P[Oracle Cloud VM]
    O --> Q[Cloudflare DNS + Tunnel]
{{< /mermaid >}}

| Layer | Stack | Cost |
|---|---|---|
| Static site | Hugo + Blowfish on Cloudflare Pages (global CDN, 99.99% SLA) | $0 |
| Live infra | K3s on Oracle Cloud ARM (4 CPUs, 24GB RAM) with ArgoCD GitOps | $0 |
| Monitoring | VictoriaMetrics + Grafana + Uptime Kuma + UptimeRobot | $0 |
| IaC | Terraform (Oracle Cloud + Cloudflare providers) | $0 |
| CI/CD | GitHub Actions → GHCR → ArgoCD auto-sync | $0 |
| Security | Cloudflare Tunnel (zero open ports), Trivy container scanning | $0 |
| **Total** | | **$0/month** |

[Read how I built it →](/blog)

---

## Get in Touch

Open to new opportunities, consulting engagements, and collaborations.

{{< button href="mailto:hello@karthikhegde.in" >}}
Email Me
{{< /button >}}
{{< button href="https://github.com/KarthikHegde91" target="_blank" >}}
GitHub
{{< /button >}}
{{< button href="https://www.linkedin.com/in/karthik-hegde-9112b6198" target="_blank" >}}
LinkedIn
{{< /button >}}
