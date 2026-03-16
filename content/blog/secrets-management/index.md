---
title: "Managing Secrets Across CI/CD Pipelines: GitHub Secrets, Infisical & GCP Secret Manager"
description: "Comparing three secrets management approaches and how to use them together for a robust pipeline security strategy"
date: 2026-03-08
draft: false
tags: ["Security", "Secrets Management", "GitHub Secrets", "Infisical", "Google Secret Manager", "CI/CD"]
showTableOfContents: true
---

Secrets management is one of those things that seems simple until you have API keys scattered across .env files on developer laptops, hardcoded tokens in CI config, and credentials shared over Slack messages that "someone will rotate later."

I've worked with three different secrets management tools in production — GitHub Secrets, Infisical, and Google Secret Manager. Each has a sweet spot, and using them together gives you a robust security posture without overcomplicating your pipeline.

## The Problem

When I inherited the CI/CD pipeline, secrets were everywhere:

- **Hardcoded in .env files** checked into private repos (yes, really)
- **Copy-pasted between team members** via Slack and email
- **Never rotated** — some API keys were over a year old
- **No audit trail** — no way to know who accessed what, or when
- **Environment drift** — staging and production had different secrets with no source of truth

This is a security incident waiting to happen. The goal was to centralize secrets management, enforce rotation, and maintain audit logs — without slowing down the development workflow.

## Three Tools Compared

| Feature | GitHub Secrets | Infisical | Google Secret Manager |
|---------|---------------|-----------|----------------------|
| **Scope** | Repository/Org-level | Project-level, cross-platform | GCP project-level |
| **Access control** | Repo collaborators | Role-based, granular | IAM-based |
| **Rotation** | Manual | Automatic + manual | Automatic + manual |
| **Audit log** | Limited (Enterprise only) | Full audit trail | Full audit trail |
| **CLI access** | `gh secret` | `infisical` CLI | `gcloud secrets` |
| **CI/CD integration** | Native in GitHub Actions | SDK/CLI injection | SDK/CLI injection |
| **Cost** | Free (included with GitHub) | Free tier available, self-hostable | Pay per access + storage |
| **Best for** | Simple pipeline secrets | Centralized team secrets | GCP-native workloads |

## GitHub Secrets: When It's Enough

GitHub Secrets is the simplest option and comes free with every GitHub repository. Secrets are encrypted at rest, masked in logs, and directly available in GitHub Actions workflows:

```yaml
- name: Deploy to production
  env:
    API_KEY: ${{ secrets.PRODUCTION_API_KEY }}
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  run: ./deploy.sh
```

### When GitHub Secrets Works Well

- **Small teams** with straightforward pipelines
- **CI/CD-only secrets** that don't need to be shared outside GitHub Actions
- **Repository-scoped secrets** where each repo has its own credentials

### Limitations

- **No rotation automation** — you have to manually update secrets when they expire
- **No audit log** on free/team plans — you can't see who accessed or changed a secret
- **No cross-platform access** — secrets are locked to GitHub Actions; your local dev environment, staging server, or other CI tools can't access them
- **No versioning** — updating a secret overwrites the old value with no history

## Infisical: The Centralized Solution

Infisical is where I moved the team's secrets management. It provides a centralized dashboard for all secrets across environments, with role-based access control, automatic rotation, and a full audit trail.

### Why Infisical

- **Single source of truth** — all secrets live in one place, organized by project and environment (dev, staging, production)
- **CLI injection** — developers pull secrets into their local environment with `infisical run -- npm start` instead of maintaining .env files
- **Self-hostable** — for organizations that need secrets to stay on-premises
- **Audit trail** — every secret access, modification, and rotation is logged with timestamps and user identity

### CI/CD Integration

In GitHub Actions, Infisical secrets are injected at build time:

```yaml
- name: Import secrets from Infisical
  uses: Infisical/secrets-action@v1
  with:
    domain: ${{ secrets.INFISICAL_URL }}
    token: ${{ secrets.INFISICAL_TOKEN }}
    env: production
    projectId: ${{ secrets.INFISICAL_PROJECT_ID }}
```

This keeps a minimal set of "bootstrap" secrets in GitHub Secrets (the Infisical token and URL), while all application secrets are managed centrally in Infisical.

### Local Development

No more .env files committed to repos:

```bash
# Pulls secrets from Infisical and injects them as environment variables
infisical run --env=dev -- npm start
```

New team members get access to the right secrets by being added to the Infisical project — no more "ask someone to send you the .env file."

## Google Secret Manager: For GCP-Native Workloads

For services running on GCP (Compute Engine, Cloud Run, Cloud Functions), Google Secret Manager integrates natively with IAM:

- **IAM-based access** — service accounts get access to specific secrets without API keys
- **Automatic rotation** — configure rotation schedules for supported secret types
- **Versioning** — every update creates a new version; you can roll back to previous values
- **Audit logging** — integrated with Cloud Audit Logs

### When to Use It

Google Secret Manager makes sense when:
- Your workload runs on GCP and uses service accounts
- You need tight IAM integration (e.g., "only this Cloud Run service can access this database password")
- You're already invested in the GCP ecosystem

## Our Hybrid Approach

After evaluating all three, we settled on a layered strategy:

{{< mermaid >}}
flowchart TB
    A[GitHub Secrets] --> |Bootstrap credentials| B[CI/CD Pipelines]
    C[Infisical] --> |Application secrets| B
    C --> |Application secrets| D[Developer Machines]
    C --> |Application secrets| E[Staging/Production Servers]
    F[Google Secret Manager] --> |GCP-specific secrets| G[GCP Services]
{{< /mermaid >}}

| Secret Type | Tool | Why |
|------------|------|-----|
| CI/CD pipeline credentials (tokens, signing keys) | GitHub Secrets | Simplest for GitHub Actions-only secrets |
| Application secrets (API keys, DB passwords, config) | Infisical | Centralized, audited, accessible everywhere |
| GCP service credentials (service account keys, API keys for GCP services) | Google Secret Manager | Native IAM integration, no credential files |

## Best Practices

Based on two years of managing secrets across these tools:

1. **Never hardcode secrets** — Not in code, not in config files, not in Dockerfiles. Ever. Use environment variable injection
2. **Rotate regularly** — Set calendar reminders if you don't have automatic rotation. 90-day maximum for any credential
3. **Least privilege** — Each service gets access only to the secrets it needs. No shared "master" credentials
4. **Audit access** — Review who has access to what quarterly. Remove access for people who've changed roles or left
5. **Separate by environment** — Dev, staging, and production secrets should never be the same values. If someone leaks dev credentials, production isn't compromised
6. **Use short-lived tokens where possible** — JWTs, temporary credentials, and session tokens are better than long-lived API keys
7. **Encrypt secrets in transit** — Use TLS for all secret retrieval. Never pass secrets as command-line arguments (they appear in process listings)

---

Secrets management isn't glamorous, but a leaked credential can undo months of good engineering work. The right tooling makes it painless enough that the team actually follows the security practices instead of working around them.
