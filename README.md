# CI/CD Pipelines — Chess Prime (Personal Project)

Production-grade GitHub Actions workflows covering backend deployment to AWS EKS and multi-platform application delivery for Android, Android TV, iOS, and Desktop (macOS/Windows).

## Overview

This repository contains the CI/CD pipeline definitions used to build, sign, and release the Chess Prime platform across every supported surface. Each workflow is designed for real operational use: secrets are validated before expensive steps run, signing artifacts are handled and cleaned up securely, failures are surfaced through Slack, and every design decision accounts for build cost and runner efficiency.

The pipelines are organized by deployment target rather than by platform convention, so each file can be read end-to-end as a complete release process — from source checkout to store submission or production rollout.

## Repository Structure

| File | Deployment Target | Description |
|---|---|---|
| `web-service-deployment.yml` | AWS EKS (`ap-south-1`) | Builds and pushes a Docker image to ECR using OIDC role assumption, deploys via `kubectl set image`, verifies rollout status, and reports build status to Slack. |
| `android-deployment.yml` | Google Play | Builds a signed Android App Bundle and APK using Gradle on a Depot-hosted runner, then publishes to the Google Play internal testing track. |
| `androidtv-deployment.yml` | Google Play | Generates the Gradle wrapper, applies dependency caching, builds and signs the Android TV release, generates build checksums, publishes to Google Play, and reports status to Slack. |
| `ios-deployment.yml` | TestFlight | Archives and exports the native iOS application using manual code signing on a self-hosted macOS runner, then uploads the build to TestFlight. |
| `ios-depoloyment-depot.yml` | TestFlight | Builds the Expo/React Native iOS application on a Depot-hosted macOS runner, including CocoaPods installation with retry handling, and uploads the release artifact to Slack with a threaded build summary. |
| `desktop-windows-mac-deployment.yml` | S3 / DigitalOcean Spaces + GitHub Releases | Multi-job pipeline that derives release metadata from the git tag, validates Apple signing credentials, builds and notarizes a universal macOS binary, builds a signed Windows installer, generates checksums, produces an Ed25519-signed update manifest, and publishes all release artifacts. |

## Engineering Practices

- **Secret validation before execution** — required secrets and variables are checked at the start of each workflow so failures surface immediately rather than after minutes of build time.
- **Secure signing material handling** — decoded keystores, certificates, and provisioning profiles are removed from the runner filesystem with `if: always()` cleanup steps.
- **OIDC over long-lived credentials** — AWS authentication uses short-lived, role-based OIDC tokens instead of static access keys.
- **Dependency caching** — Gradle and Cargo caches are keyed on lockfile and wrapper hashes to reduce redundant downloads across runs.
- **Retry handling for known network flakiness** — CocoaPods installs and WebRTC prebuilt binary downloads include bounded retry logic to prevent transient failures from breaking a release.
- **Runner selection by workload** — GitHub-hosted, Depot-hosted, and self-hosted macOS runners are each used where they are the most cost-appropriate choice for the job.
- **Release traceability** — checksums, versioned artifacts, and signed update manifests accompany every desktop release.
- **Build status visibility** — Slack notifications (webhook and Bot API) report success or failure with direct links to the workflow run.

## Cost Optimization Practices

- OIDC role assumption removes the operational cost of rotating and securing long-lived AWS credentials.
- Dependency and toolchain caching (Gradle, Cargo/sccache) reduces repeated download and compile time across runs.
- Release-triggered builds (git tags) keep expensive mobile and desktop jobs from running on every push.
- Runner tiers are matched to workload size rather than defaulting to the largest available tier.
- `continue-on-error` is scoped only to genuinely non-fatal steps, such as re-uploading an existing store version, and is never applied to security or validation steps.
- ECR image lifecycle policies and tagging strategy prevent unbounded storage growth.

## Roadmap

- Migrate the EKS deployment from `kubectl set image` to Helm-based releases.
- Introduce ArgoCD for GitOps-based cluster synchronization, including ApplicationSets and automated image updates.
- Add a Terraform-driven infrastructure pipeline with plan-on-PR and apply-on-merge stages.
- Introduce policy-as-code enforcement (OPA Gatekeeper / Kyverno) with admission-level verification of signed container images.
- Add cost visibility tooling (Kubecost/OpenCost) for the EKS environment.
- Consolidate the two iOS release workflows into a single Fastlane-driven pipeline shared across platforms.
- Add a Prometheus and Grafana observability stack for the EKS cluster.
- Introduce Vault on EKS for centralized secrets management.

## Usage

1. Copy the required workflow file into `.github/workflows/` in the target repository.
2. Replace placeholder secrets, variables, package identifiers, and cluster or region values with environment-specific configuration.
3. Review the inline comments before modifying retry logic, validation steps, or cleanup steps — each exists to address a specific failure mode encountered in production use.
