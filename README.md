# CI/CD Learning Repo

A hands-on collection of production-style GitHub Actions CI/CD pipelines, built while learning DevOps, MLOps, and Cloud practices — covering an EKS-deployed backend service and a full multi-platform app release pipeline (Android, Android TV, iOS — two variants — and Desktop for macOS/Windows). Each workflow is a real, runnable example, not a toy — the goal is practical, real-world capability for mid-to-senior DevOps/Cloud roles.

## Why this repo exists

Most CI/CD tutorials stop at "build and push." These examples go further:
- Secret validation before the expensive steps run
- Retry logic for known-flaky third-party downloads
- Slack success/failure notifications on every pipeline (webhook and Bot API variants)
- Checksums, artifact retention, and cleanup of sensitive files (e.g. decoded keystores)
- Cost-consciousness baked into every design choice, not bolted on after

## Workflow examples in this repo

| File | What it teaches | Deploy target |
|---|---|---|
| `web-service-deployment.yml` | Docker build → ECR push (OIDC, no static keys) → `kubectl set image` → rollout verification → Slack notify | AWS EKS (`ap-south-1`) |
| `android-deployment.yml` | Gradle signed AAB/APK build on a Depot runner → Google Play internal track | Google Play |
| `androidtv-deployment.yml` | Gradle wrapper generation + dependency caching → signed AAB/APK → checksums → Google Play internal track → Slack notify | Google Play |
| `ios-deployment.yml` | Xcode archive/export with manual signing on a self-hosted macOS runner → TestFlight | TestFlight |
| `ios-depoloyment-depot.yml` | Expo/React Native prebuild → CocoaPods install with retry → Xcode archive on a Depot macOS runner → TestFlight → Slack Bot file upload with threaded release summary | TestFlight |
| `desktop-windows-mac-deployment.yml` | Multi-job pipeline: release metadata from git tag → secret/certificate preflight → macOS universal Rust build (sign + notarize + DMG) → Windows build (Inno Setup installer) → checksums → Ed25519-signed `latest.json` update manifest → publish | S3 / DigitalOcean Spaces + GitHub Releases |

Each workflow file has inline comments explaining *why* a step exists, not just *what* it does — especially around retry logic, secret handling, and signing.

## Core concepts covered so far

- **Secrets & signing**: Android keystores, Apple certificates/provisioning profiles (including broadcast/share extension profiles), App Store Connect API keys, Ed25519 manifest signing, OIDC vs static AWS keys
- **Kubernetes deploys**: `kubectl set image`, rollout status checks, namespace-scoped deploys, ECR image tagging (SHA + `latest`)
- **Multi-platform mobile builds**: native Gradle (Android/Android TV) vs Expo/React Native prebuild (iOS via Depot), same target platform built two different ways
- **Build reliability**: retry wrappers for flaky network downloads (CocoaPods, WebRTC prebuilt binaries), Sentry source-map upload with graceful fallback when a secret is missing
- **Runner strategy**: GitHub-hosted vs Depot-hosted (Ubuntu + macOS) vs self-hosted in-house macOS runners, and when each makes sense
- **Notifications**: Slack webhook (simple pass/fail attachment) vs Slack Bot API (file upload + threaded release notes)
- **Caching**: Gradle (`~/.gradle`) and Cargo (`~/.cargo`, sccache) dependency caching keyed on lockfile/wrapper hashes
- **Release management**: version/channel inference from git tags (release vs beta/rc/alpha), signed update manifests, draft GitHub Releases with auto-generated notes

## Cost optimization checklist

Applied across these pipelines — use this as a reference when adding a new workflow:

- [ ] Use OIDC role assumption instead of long-lived cloud provider keys (`web-service-deployment.yml`)
- [ ] Cache dependencies (Gradle `~/.gradle`, Cargo `~/.cargo`, npm/pnpm store) keyed on lockfile/wrapper hash
- [ ] Right-size runners — Depot Ubuntu/macOS tiers per job instead of defaulting to the largest tier; self-hosted only where Apple's macOS licensing/hardware needs make it worth it
- [ ] Use `continue-on-error: true` only where a failure is genuinely non-fatal (e.g., re-upload of an existing Play/TestFlight version), never on a security scan
- [ ] Set an ECR/registry lifecycle policy to expire untagged images
- [ ] Trigger builds on git tags, not every push, for expensive mobile/desktop release jobs
- [ ] Clean up decoded secrets (keystores, `.p12`, provisioning profiles) from runner disk with `if: always()` so nothing lingers even on failure
- [ ] Prefer GitHub-hosted runners over paid runner services (Depot/self-hosted) unless build time or platform requirement (macOS) is an actual bottleneck

## Roadmap — topics to add next

- [ ] Helm-based EKS deploy (replace raw `kubectl set image` with `helm upgrade --install`)
- [ ] ArgoCD GitOps example — CI only builds/pushes; ArgoCD syncs the cluster (ApplicationSets with Git generator + Image Updater for automated ECR sync)
- [ ] Terraform-triggered infra pipeline (plan on PR, apply on merge, remote state via S3 + DynamoDB)
- [ ] OPA Gatekeeper / Kyverno policy-as-code checks in CI, including admission-level enforcement of Cosign-signed images
- [ ] Kubecost/OpenCost integration for cost visibility dashboards
- [ ] Canary/blue-green deploy pattern with Argo Rollouts
- [ ] Consolidate the two iOS workflows (native Xcode vs Expo/Depot) into a single Fastlane-driven pipeline shared across iOS/Android/Desktop signing logic
- [ ] Prometheus + Grafana observability stack on EKS
- [ ] Vault on EKS via Helm for secrets management (replacing base64-only Kubernetes Secrets)

## How to use this repo

1. Each workflow file is standalone — copy the one you need into `.github/workflows/` in your own project.
2. Replace placeholder secret names, package/bundle identifiers, and cluster/region values with your own.
3. Read the inline comments before removing anything that looks like "unnecessary" retry logic or validation — most of it exists because of a real failure encountered while building these.
