# tazama-v4-ci-fork

Fork of [`tazama-lf/cloud-infrastructure-deploy`](https://github.com/tazama-lf/cloud-infrastructure-deploy) aligned for the **AbliPay first deployment** of the Tazama v4 anti-fraud platform.

Consumed by the `gcp-tazama` Terraform workspace in [`AbliPayTech/gitops`](https://github.com/AbliPayTech/gitops) (PR #189) via ArgoCD app-of-apps.

## Changes vs upstream

All changes are on the `dev` branch (upstream content lives on `dev`, not `main`).

| Change | Upstream | Fork (`dev`) | Scope |
|---|---|---|---|
| `targetRevision` | `main` | `dev` | All 47 ArgoCD Application manifests |
| `repoURL` | `github.com/cloud-infrastructure-deploy` (typo, missing `tazama-lf/`) | `github.com/tazama-lf/cloud-infrastructure-deploy` | All 46 per-service Applications |
| `project` | `default` | `tazama` | All 47 Application manifests |
| `namespace` | `staging` | `tazama` | All 46 per-service Applications (app-of-apps stays in `argocd`) |

See [ADR-0001](docs/adr/0001-ablipay-deployment-alignment.md) for the rationale.

## Deployment configuration

### ArgoCD topology

```
app-of-apps (apps/app-of-apps-staging.yaml)
 └── 46 per-service Applications (apps/staging/*-application.yaml)
      └── each syncs k8s/overlays/staging/<service> (Kustomize)
           └── deploys to the tazama namespace
```

- **Project**: `tazama` (AppProject defined in AbliPay-Infra `gcp/tazama/02-argocd-project.tf`)
- **Namespace**: `tazama`
- **Images**: `tazamaorg/*` (Docker Hub, public)
- **Dependencies** (deployed separately via AbliPay-Infra): NATS, Valkey, PostgreSQL (in-cluster first), Keycloak (disabled)

### Endpoints

| Endpoint | Backend | Purpose |
|---|---|---|
| `https://tazama.ablipay.dev` | `tms-service` :3000 | Anti-fraud TMS API - pre-transaction screening on the quote step |
| `https://tazama-admin.ablipay.dev` | `admin-service` :3000 | Admin Services API - condition/configuration management |

Served via the external gateway (non-mTLS) for the first deployment. Moving behind `secure-external-gateway` (`*.secure.ablipay.dev`, mTLS) is a documented later refinement.

### Database

Tazama v4 uses PostgreSQL (5 databases: `eventhistory`, `rawhistory`, `evaluation`, `configuration`, `simulation`). Connection strings are isolated in the `tazama-db-config` Secret (AbliPay-Infra `gcp/tazama/07-db-config.tf`) so the in-cluster → Cloud SQL migration is a config change.

## Usage

```bash
# ArgoCD parent app (defined in AbliPay-Infra gcp/tazama/03-parent-app.tf)
# repoURL: https://github.com/AbliPayTech/tazama-v4-ci-fork.git
# targetRevision: dev
# path: apps/staging
```

## Upstream tracking

Rebase from upstream `dev` periodically:

```bash
git remote add upstream https://github.com/tazama-lf/cloud-infrastructure-deploy.git
git fetch upstream
git rebase upstream/dev
```

Note: rebasing may re-introduce the alignment changes above - re-apply if needed.