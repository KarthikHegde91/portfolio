---
title: "Shipping to 4 Platforms from a Single Codebase with GitHub Actions"
description: "How I automated multi-platform CI/CD for Android, iOS, Windows & macOS — cutting build times from 2 hours to 15 minutes"
date: 2026-03-13
draft: false
tags: ["GitHub Actions", "CI/CD", "Flutter", "Shorebird", "Android", "iOS", "Windows", "macOS"]
showTableOfContents: true
---

Building and releasing a Flutter application to **Android, iOS, Windows, and macOS** from a single codebase sounds straightforward in theory. In practice, it's one of the hardest CI/CD problems I've solved.

This post covers how I designed the pipeline, the platform-specific challenges I ran into, and what I'd do differently if I started over.

## The Problem

Our team of 10+ engineers works on a single Flutter codebase that ships to four platforms. Before automation, the release process looked like this:

- **Manual builds took ~2 hours per platform** — an engineer would sit at their machine running builds, signing artifacts, and uploading them to app stores
- **Build failures were frequent** — different local environments, different Xcode versions, different JDK versions. A build that worked on one machine would fail on another
- **Release coordination was a nightmare** — versioning across 4 platforms, updating changelogs, ensuring all platforms shipped the same code. Someone always missed something
- **Rollbacks were painful** — a bad release meant rebuilding, re-signing, re-submitting, and waiting for app store review

We needed a system that could build all four platforms in parallel, handle platform-specific signing and packaging, and deploy automatically — with zero manual intervention.

## The Architecture

{{< mermaid >}}
flowchart TB
    A[Developer pushes to release branch] --> B[GitHub Actions triggers]

    B --> C[Android Workflow]
    B --> D[iOS Workflow]
    B --> E[Windows Workflow]
    B --> F[macOS Workflow]

    C --> C1[Build AAB + APK]
    C1 --> C2[Upload to Play Store]
    C1 --> C3[Upload to GitHub Release]

    D --> D1[Build IPA via Fastlane]
    D1 --> D2[Upload to TestFlight]
    D1 --> D3[Upload to GitHub Release]

    E --> E1[Build MSIX via PowerShell]
    E1 --> E3[Upload to GitHub Release]

    F --> F1[Build DMG via Fastlane]
    F1 --> F3[Upload to GitHub Release]

    B --> G[Shorebird OTA Patch]
{{< /mermaid >}}

Each platform has its own workflow file (150-300 lines each), triggered by pushes to the release branch. All four run in parallel.

## Platform-Specific Challenges

### Android: Gradle Heap and Signing

Android builds are memory-hungry. The default Gradle heap size is insufficient for large Flutter projects. Without explicit configuration, you'll hit OutOfMemoryError on CI runners:

```yaml
env:
  GRADLE_OPTS: "-Xmx4G -Dorg.gradle.daemon=false"
```

For signing, the keystore is stored as a base64-encoded GitHub Secret. A critical lesson: **always verify the decode succeeded** before attempting to build:

```yaml
- name: Decode keystore
  run: |
    echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > android/app/keystore.jks
    [ -s android/app/keystore.jks ] || { echo "Keystore decode failed"; exit 1; }
```

### iOS: The Signing Nightmare

iOS code signing is the single hardest part of the entire pipeline. Certificates expire, provisioning profiles get revoked, and Apple's tooling provides cryptic error messages.

I use Fastlane's `match` for certificate management and `continue-on-error: true` for the TestFlight upload step — because TestFlight sometimes rejects builds for non-code reasons (compliance questionnaire changes, processing delays) and we don't want that to fail the entire pipeline.

### Windows: PowerShell Error Handling

Windows builds use PowerShell, which has different error handling semantics than Bash. The key gotcha: **PowerShell doesn't exit on error by default**. You need explicit `$LASTEXITCODE` checks:

```powershell
flutter build windows --release
if ($LASTEXITCODE -ne 0) {
    Write-Error "Flutter build failed with exit code $LASTEXITCODE"
    exit 1
}
```

### macOS: Notarization and DMG Packaging

macOS requires notarization for distribution outside the App Store. Fastlane handles the build and signing, but DMG packaging and notarization add significant build time (~5-10 minutes just for the notarization step).

## Retry Logic for Flaky Builds

CI builds are inherently flaky — network timeouts downloading dependencies, transient signing server issues, runner resource contention. I use `nick-fields/retry` for critical steps:

```yaml
- name: Build Android APK
  uses: nick-fields/retry@v3
  with:
    timeout_minutes: 30
    max_attempts: 3
    command: flutter build apk --release
```

This alone reduced our "red build" rate significantly. Most transient failures succeed on the second attempt.

## Error Reporting

When a build fails, the last thing you want is to dig through hundreds of lines of CI logs. I added a pattern where **on failure, the last 50 lines of build output are written to GitHub Step Summary**:

```yaml
- name: Capture error context
  if: failure()
  run: |
    echo "## Build Failed" >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
    tail -50 build.log >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
```

This gives you instant context in the GitHub Actions UI without opening log files.

## Shorebird: Over-the-Air Code Push

One of the most impactful additions was Shorebird — it lets you push Flutter code changes directly to users' devices without going through app store review. For critical bug fixes, this cuts the rollout time from **days (app store review)** to **minutes (OTA patch)**.

The Shorebird workflow runs alongside the regular builds, creating a patch that can be applied to the currently-released version.

## Results

| Metric | Before | After |
|--------|--------|-------|
| Build time (per platform) | ~2 hours manual | 15-20 minutes automated |
| Release frequency | Weekly at best | Daily across all 4 platforms |
| Build failure rate | High (environment inconsistency) | Low (containerized, reproducible) |
| Rollback time | Hours (rebuild + resubmit) | Minutes (Shorebird OTA) |
| Manual effort per release | ~1 full day | Zero |

## What I'd Do Differently

1. **Start with Fastlane earlier** — I initially wrote raw shell scripts for iOS/macOS builds. Fastlane abstracts away most of the signing complexity and was worth adopting on day one
2. **Use composite actions for shared logic** — There's significant duplication across the 4 workflows (checkout, Flutter setup, secret decoding). Composite actions would reduce maintenance overhead
3. **Invest in build caching from the start** — Gradle and Xcode caches cut build times by 30-40%. I added them late; should have been there from the beginning
4. **Set up Shorebird from day one** — The ability to push hotfixes without app store review is too valuable to delay

---

Building a CI/CD pipeline for 4 platforms is complex, but the investment pays for itself within weeks. The key is treating each platform's quirks as first-class concerns rather than afterthoughts.
