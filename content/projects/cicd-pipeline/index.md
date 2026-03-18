---
title: "Multi-Platform CI/CD Pipeline"
description: "Automated GitHub Actions pipeline shipping to Android, iOS, Windows & macOS from a single Flutter codebase"
date: 2026-03-13
draft: false
tags: ["GitHub Actions", "CI/CD", "Flutter", "Docker", "Shorebird"]
showTableOfContents: true
impact: "2hr manual builds → 15min automated"
---

## Overview

Fully automated CI/CD pipeline that builds, tests, signs, and deploys a Flutter application to **4 platforms** (Android, iOS, Windows, macOS) from a single codebase — serving a team of **10+ engineers** with **daily releases**.

{{< badge >}}GitHub Actions{{< /badge >}}
{{< badge >}}Flutter{{< /badge >}}
{{< badge >}}Shorebird{{< /badge >}}
{{< badge >}}Fastlane{{< /badge >}}
{{< badge >}}Docker{{< /badge >}}

---

## The Problem

Before automation, the release process was painful:

- **Manual builds took ~2 hours per platform** — building for all 4 meant a full day lost
- **Frequent build failures** from inconsistent local environments across the team
- **Release coordination was error-prone** — syncing versions, changelogs, and signing across Android, iOS, Windows, and macOS led to missed deadlines
- **No rollback capability** — a bad release meant rebuilding and resubmitting from scratch

## Solution

A GitHub Actions-based CI/CD system with **platform-specific workflows** that trigger on push to release branches.

### Architecture

{{< mermaid >}}
flowchart LR
    A[Git Push] --> B[GitHub Actions]
    B --> C[Build Matrix]
    C --> D[Android\nAAB + APK]
    C --> E[iOS\nIPA + TestFlight]
    C --> F[Windows\nMSIX]
    C --> G[macOS\nDMG]
    D --> H[Google Play Store]
    E --> I[App Store Connect]
    F --> J[GitHub Release]
    G --> J
    D --> J
    B --> K[Shorebird\nOTA Patch]
{{< /mermaid >}}

### Key Features

- **Matrix builds** — All 4 platforms build in parallel, reducing total pipeline time
- **Retry logic** — Flaky builds (especially iOS signing) automatically retry up to 3 times using `nick-fields/retry`
- **Artifact validation** — Build outputs are verified before upload to prevent deploying corrupt artifacts
- **Shorebird code push** — Over-the-air patches for Flutter apps without going through app store review
- **Automated signing** — Keystore/certificate management via base64-encoded secrets with decode verification
- **Error reporting** — On failure, build logs are captured and dumped to GitHub Step Summary for quick debugging

### Platform-Specific Workflows

| Platform | Build Tool | Deploy Target | Special Handling |
|----------|-----------|---------------|-----------------|
| Android | Gradle | Play Store + GitHub Release | Heap size tuning (4GB) for large builds |
| iOS | Xcode + Fastlane | TestFlight + GitHub Release | Code signing with `continue-on-error` for TestFlight |
| Windows | Flutter build + PowerShell | GitHub Release | `$LASTEXITCODE` checks for PowerShell error handling |
| macOS | Xcode + Fastlane | GitHub Release | DMG packaging with notarization |

## Results

| Metric | Before | After |
|--------|--------|-------|
| Build time per platform | ~2 hours (manual) | 15-20 minutes (automated) |
| Release frequency | Weekly (at best) | Daily across all 4 platforms |
| Build failure rate | High (environment inconsistency) | Low (containerized, consistent) |
| Rollback time | Hours (rebuild + resubmit) | Minutes (Shorebird OTA patch) |
| Team productivity | 1 engineer dedicated to releases | Fully automated, zero manual intervention |

## Lessons Learned

- **iOS signing is the hardest part** — Certificate and provisioning profile management needs robust error handling and `continue-on-error` patterns
- **Gradle heap matters** — Android builds OOM frequently without explicit heap allocation (`-Xmx4G`)
- **Artifact validation is non-negotiable** — Always verify build outputs exist and have non-zero size before uploading
- **Shorebird is a game-changer** — OTA patches bypass app store review for critical fixes

{{< button href="/blog/multi-platform-cicd/" target="_self" >}}
Read the full blog post
{{< /button >}}
