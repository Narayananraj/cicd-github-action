# CI/CD Learning Repo

A hands-on collection of production-style GitHub Actions CI/CD pipelines, built while learning DevOps, MLOps, and Cloud practices — covering containerized backend deploys, mobile app releases, and desktop app distribution. Each workflow is a real, runnable example, not a toy — the goal is practical, real-world capability.

## Why this repo exists

Most CI/CD tutorials stop at "build and push." These examples go further:
- Secret validation before the expensive steps run
- Retry logic for known-flaky third-party downloads
- Slack success/failure notifications on every pipeline
- Checksums, artifact retention, and cleanup of sensitive files
- Cost-consciousness baked into every design choice, not bolted on after

## Workflow examples in this repo

| File | What it teaches | Deploy target |
|---|---|---|
| `k8s-deploy.yml` | Docker build → ECR push → `kubectl` deploy → rollout verification → Slack notify | AWS EKS (`ap-south-1`) |
| `build-android.yml` | Signed AAB build → Google Play internal track | Google Play |
| `build-ios.yml` | Xcode archive/export with manual signing → TestFlight, using a Depot macOS runner | TestFlight |
| `build-desktop.yml` | Multi-OS (macOS + Windows) Rust build → notarization/signing → S3 release with signed update manifest | S3 / DigitalOcean Spaces |
| `build-androidtv.yml` | Gradle-based native Android TV signed release with caching | Google Play |

Each workflow file has inline comments explaining *why* a step exists, not just *what* it does — especially around retry logic, secret handling, and signing.

## Core concepts covered so far

- **Secrets & signing**: keystores, provisioning profiles, code signing certs, OIDC vs static AWS keys
- **Kubernetes deploys**: `kubectl set image`, rollout status checks, namespace-scoped deploys
- **Container registries**: ECR login, image tagging strategy (SHA + latest), lifecycle policies
- **Build reliability**: retry wrappers for flaky network downloads (CocoaPods, WebRTC prebuilt binaries)
- **Notifications**: Slack webhook (simple pass/fail) vs Slack Bot API (file upload + threaded release notes)
- **Caching**: Gradle and Cargo dependency caching to cut runner minutes
- **Release management**: version/channel inference from git tags, signed update manifests (Ed25519)

## Cost optimization checklist

Applied across these pipelines — use this as a reference when adding a new workflow:

- [ ] Use OIDC role assumption instead of long-lived cloud provider keys
- [ ] Cache dependencies (Gradle `~/.gradle`, Cargo `~/.cargo`, npm/pnpm store) keyed on lockfile hash
- [ ] Set an ECR/registry lifecycle policy to expire untagged images
- [ ] Right-size runners — don't default to the largest Depot/self-hosted tier without benchmarking
- [ ] Use `continue-on-error: true` only where a failure is genuinely non-fatal (e.g., re-upload of an existing version)
- [ ] Set resource `requests/limits` on K8s deployments so autoscalers don't over-provision
- [ ] Prefer GitHub-hosted runners over paid runner services unless build time is an actual bottleneck

## Roadmap — topics to add next

- [ ] Helm-based K8s deploy (replace raw `kubectl set image` with `helm upgrade --install`)
- [ ] ArgoCD GitOps example (CI only builds/pushes; ArgoCD syncs the cluster)
- [ ] Terraform-triggered infra pipeline (plan on PR, apply on merge, remote state via S3 + DynamoDB)
- [ ] OPA Gatekeeper / Kyverno policy-as-code checks in CI
- [ ] Kubecost/OpenCost integration for cost visibility dashboards
- [ ] Canary/blue-green deploy pattern with Argo Rollouts
- [ ] Fastlane consolidation for iOS/Android/Desktop signing logic

## How to use this repo

1. Each workflow file is standalone — copy the one you need into `.github/workflows/` in your own project.
2. Replace placeholder secret names and package/bundle identifiers with your own.
3. Read the inline comments before removing anything that looks like "unnecessary" retry logic or validation — most of it exists because of a real failure encountered while building these.