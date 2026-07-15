# portfolio — karthikhegde.in

Source for my personal portfolio and DevOps blog, **[karthikhegde.in](https://karthikhegde.in)** — a static site built with **Hugo** and deployed on **Cloudflare Pages** (part of my [$0/month infrastructure](https://github.com/KarthikHegde91/infra)).

![Hugo](https://img.shields.io/badge/built_with-Hugo-ff4088)
![Cloudflare Pages](https://img.shields.io/badge/deploy-Cloudflare_Pages-orange)
![Cost](https://img.shields.io/badge/hosting-%240-brightgreen)

---

## What's here

A DevOps/SRE portfolio with project write-ups and technical blog posts drawn from real work:

**Projects**
- Multi-platform CI/CD pipeline (ship to 4 platforms from one codebase)
- Kafka data pipeline
- Monitoring & observability stack
- Zero-cost ($0/month) infrastructure

**Blog**
- Managing secrets across CI/CD pipelines (GitHub Secrets / Infisical / GCP Secret Manager)
- Prometheus + Grafana setup
- Pulumi vs. Terraform
- Multi-platform CI/CD lessons learned

---

## Tech

- **Hugo** static site generator (`hugo.toml`, `config/_default/`)
- Custom layouts (`layouts/`) and styling (`assets/css/`, incl. a cyberpunk scheme)
- Content in Markdown under `content/`
- Deployed continuously on **Cloudflare Pages**

## Structure

```
.
├── config/_default/     # Hugo config (params, menus, languages)
├── content/             # Markdown: projects, blog, about, contact, services
├── layouts/             # custom partials (head, footer, home)
├── assets/css/          # styles + color schemes
├── static/              # static assets (incl. resume PDF)
├── resume/              # LaTeX resume source
└── hugo.toml            # site entry config
```

## Run locally

```bash
# Requires Hugo extended (https://gohugo.io)
git clone --recurse-submodules https://github.com/KarthikHegde91/portfolio.git
cd portfolio
hugo server -D        # live-reload dev server at http://localhost:1313
```

> Uses a theme as a git submodule — clone with `--recurse-submodules` (or run `git submodule update --init`).

---

**Author:** Karthik B Hegde — DevOps / SRE Engineer, Bengaluru
[karthikhegde.in](https://karthikhegde.in) · [LinkedIn](https://www.linkedin.com/in/karthik-hegde-9112b6198) · [GitHub](https://github.com/KarthikHegde91)
