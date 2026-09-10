# 1. AbliPay Deployment Alignment

## Status

Proposed - 2026-09-09 (open PR, pending first ArgoCD deploy validation).

## Context

AbliPay is deploying the Tazama v4 anti-fraud platform onto its `ablipay-dev` GKE cluster as a pre-transaction screening layer (evaluation on the quote step). The deployment consumes this fork of `tazama-lf/cloud-infrastructure-deploy` via ArgoCD app-of-apps.

The upstream repository is self-labeled "Still A Work In Progress" and ships manifests that do not match AbliPay's deployment model:

- **`targetRevision: main`** everywhere, but all content lives on the `dev` branch - ArgoCD would sync an empty/near-empty tree.
- **`repoURL` typo** in `rule-001-application.yaml`: `github.com/cloud-infrastructure-deploy` (missing the `tazama-lf/` org prefix) - the Application would fail to resolve.
- **`project: default`** - AbliPay uses dedicated ArgoCD AppProjects per workload (see `gcp/ablipay/02-argocd-project.tf` pattern).
- **`namespace: staging`** - AbliPay deploys into a dedicated `tazama` namespace managed by Terraform.

### Forces

- **ArgoCD correctness**: Applications must point at a branch that exists (`dev`) and a resolvable repo URL.
- **Tenancy isolation**: A dedicated AppProject + namespace keeps the anti-fraud platform isolated from Mojaloop workloads.
- **Terraform-managed namespace**: The `tazama` namespace is created by Terraform (`gcp/tazama/01-namespace.tf`), so ArgoCD must not create it (`CreateNamespace=false`).
- **Minimal fork delta**: Only the alignment changes needed to deploy; no functional changes to the Tazama manifests themselves.

### Options considered

| Option | Pros | Cons |
|--------|------|------|
| **Fork + patch manifests** (chosen) | Small, reviewable delta; tracks upstream; ArgoCD consumes a known-good branch | Fork drift on upstream rebase; manual re-apply of alignment |
| **Kustomize overlay in AbliPay-Infra** | No fork; patches applied at sync time | Duplicates the 46-manifest structure; overlay must track upstream paths; harder to review |
| **Helm chart from scratch** | Full control | No upstream v4 Helm chart exists (`tazama-helm` is an empty stub); significant effort (rejected in ABL-900) |

**Decision: fork + patch manifests.** Smallest delta that makes the upstream Kustomize-based deployment consumable by AbliPay's ArgoCD pattern.

## Decision

### 1. Fork and align the ArgoCD manifests

Fork `tazama-lf/cloud-infrastructure-deploy` as `AbliPayTech/tazama-v4-ci-fork` (default branch `dev`) and apply four alignment changes to all Application manifests:

1. `targetRevision: main` → `dev` (47 files)
2. `repoURL` typo fix: add `tazama-lf/` org prefix (46 files)
3. `project: default` → `tazama` (47 files)
4. `namespace: staging` → `tazama` (46 per-service files; app-of-apps stays in `argocd`)

### 2. Deployment topology

- Parent app-of-apps (`apps/app-of-apps-staging.yaml`) → 46 per-service Applications (`apps/staging/*-application.yaml`)
- Each service syncs `k8s/overlays/staging/<service>` (Kustomize) into the `tazama` namespace
- Consumed by the `gcp-tazama` Terraform workspace (AbliPay-Infra PR #189)

### 3. Endpoints

| Endpoint | Backend |
|---|---|
| `https://tazama.ablipay.dev` | `tms-service` :3000 |
| `https://tazama-admin.ablipay.dev` | `admin-service` :3000 |

Served via the external gateway (non-mTLS) for the first deployment; mTLS via `secure-external-gateway` is a later refinement.

## Consequences

### Positive

- ArgoCD can sync the full Tazama stack from a known-good branch (`dev`) with a resolvable repo URL.
- Dedicated `tazama` AppProject + namespace isolates the anti-fraud platform.
- No functional changes to Tazama manifests - upstream behavior preserved.
- Small, reviewable delta that tracks upstream.

### Negative

- Fork drift: upstream changes to `dev` may conflict with the alignment patches on rebase.
- The alignment is fork-specific - upstream PRs to fix the same issues would reduce the delta to zero.

### Risks

- Upstream `dev` is a moving target; pinning `targetRevision: dev` means ArgoCD picks up upstream changes on sync. Mitigation: review upstream commits before sync, or pin a specific commit SHA.
- The `tazamaorg/*` image tags in the Kustomize overlays may lag the v4 canonical stack (`tazama-stack`). Verify image versions during the first deploy (ABL-897).