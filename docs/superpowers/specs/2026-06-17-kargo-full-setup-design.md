# Kargo Full Setup Design

## Goal
Build a complete Kargo + ArgoCD promotion pipeline from scratch with two source repos, two images (web1/web2), three environments (dev/acc/prod), per-environment registry paths, and automated image promotion via crane copy.

## Architecture
`kargo-demo-2` is the source code repo. GitHub Actions builds two nginx images on every push to `main` and pushes them to `ghcr.io/mikakolk12/dev/`. The Kargo Warehouse discovers new images and creates freight. Promotions copy images between registry paths (dev → acc → prod) using `crane copy` and update the GitOps repo (`kargo-demo`) via Kustomize. ArgoCD deploys from stage-specific branches.

## Tech Stack
Kargo, ArgoCD, Kustomize, GitHub Actions, ghcr.io (GitHub Container Registry), crane, k3s (Rancher Desktop), nginx

---

## Repositories

### kargo-demo-2 (source code)
- URL: `https://github.com/MikaKolk12/kargo-demo-2`
- Branch: `main` (single branch, trunk-based)
- Contents:
  - `web1/index.html` — simple nginx page, shows "web1"
  - `web1/Dockerfile` — nginx serving web1/index.html
  - `web2/index.html` — simple nginx page, shows "web2"
  - `web2/Dockerfile` — nginx serving web2/index.html
  - `.github/workflows/build.yml` — builds both images on push to main

### kargo-demo (GitOps)
- URL: `https://github.com/MikaKolk12/kargo-demo`
- Branch `main`: source manifests — edited by developers
- Branch `stage/dev`: rendered manifests for dev — written by Kargo only
- Branch `stage/acc`: rendered manifests for acc — written by Kargo only
- Branch `stage/prod`: rendered manifests for prod — written by Kargo only

---

## Registry Structure

| Environment | Image | Tag format | Example | Created by |
|---|---|---|---|---|
| dev | `ghcr.io/mikakolk12/dev/web1` | `<sha7>` | `abc123f` | GitHub Actions |
| dev | `ghcr.io/mikakolk12/dev/web2` | `<sha7>` | `abc123f` | GitHub Actions |
| acc | `ghcr.io/mikakolk12/acc/web1` | `YY.M.<sha7>` | `26.6.abc123f` | Kargo on promote |
| acc | `ghcr.io/mikakolk12/acc/web2` | `YY.M.<sha7>` | `26.6.abc123f` | Kargo on promote |
| prod | `ghcr.io/mikakolk12/prod/web1` | `YY.M.<sha7>` | `26.6.abc123f` | Kargo on promote |
| prod | `ghcr.io/mikakolk12/prod/web2` | `YY.M.<sha7>` | `26.6.abc123f` | Kargo on promote |

---

## GitOps Repo Structure (kargo-demo)

```
kargo-demo/
├── base/
│   ├── deploy-web1.yaml          nginx Deployment for web1
│   ├── deploy-web2.yaml          nginx Deployment for web2
│   ├── service-web1.yaml         NodePort Service for web1
│   ├── service-web2.yaml         NodePort Service for web2
│   └── kustomization.yaml        lists all base resources
├── stages/
│   ├── dev/
│   │   ├── configmap-web1.yaml   HTML content labelled "dev - web1"
│   │   ├── configmap-web2.yaml   HTML content labelled "dev - web2"
│   │   └── kustomization.yaml    patches NodePorts 32080/32083
│   ├── acc/
│   │   ├── configmap-web1.yaml   HTML content labelled "acc - web1"
│   │   ├── configmap-web2.yaml   HTML content labelled "acc - web2"
│   │   └── kustomization.yaml    patches NodePorts 32081/32084
│   └── prod/
│       ├── configmap-web1.yaml   HTML content labelled "prod - web1"
│       ├── configmap-web2.yaml   HTML content labelled "prod - web2"
│       └── kustomization.yaml    patches NodePorts 32082/32085
├── kargo/
│   ├── project.yaml              Kargo Project "kargo-demo"
│   ├── secret-git.yaml           Git credentials for Kargo (GitHub PAT)
│   ├── secret-image.yaml         Image credentials for Kargo (ghcr.io PAT, user populates)
│   ├── warehouse.yaml            Watches dev/web1 + dev/web2, NewestBuild strategy
│   ├── promotiontask.yaml        crane copy + kustomize-set-image + git + argocd
│   ├── stages.yaml               dev (direct from Warehouse), acc (from dev), prod (from acc)
│   └── applicationset.yaml       ArgoCD ApplicationSet for dev/acc/prod
└── registry/
    ├── namespace.yaml
    ├── deployment.yaml
    └── service.yaml
```

---

## NodePort Allocation

| Stage | web1 | web2 |
|---|---|---|
| dev | 32080 | 32083 |
| acc | 32081 | 32084 |
| prod | 32082 | 32085 |

---

## Kargo Flow

```
1. Developer pushes to kargo-demo-2 main
2. GitHub Actions → ghcr.io/mikakolk12/dev/web1:abc123f
                  → ghcr.io/mikakolk12/dev/web2:abc123f

3. Kargo Warehouse discovers both images (NewestBuild), creates freight

4. Promote freight → dev stage:
   - kustomize-set-image on base/ (dev/web1:sha + dev/web2:sha)
   - kustomize-build stages/dev → stage/dev branch
   - ArgoCD syncs → kargo-demo-dev namespace

5. Promote freight → acc stage:
   - crane copy dev/web1:sha → acc/web1:YY.M.sha
   - crane copy dev/web2:sha → acc/web2:YY.M.sha
   - kustomize-set-image on base/ (acc/web1:YY.M.sha + acc/web2:YY.M.sha)
   - kustomize-build stages/acc → stage/acc branch
   - ArgoCD syncs → kargo-demo-acc namespace

6. Promote freight → prod stage:
   - crane copy acc/web1:YY.M.sha → prod/web1:YY.M.sha
   - crane copy acc/web2:YY.M.sha → prod/web2:YY.M.sha
   - kustomize-set-image on base/ (prod/web1:YY.M.sha + prod/web2:YY.M.sha)
   - kustomize-build stages/prod → stage/prod branch
   - ArgoCD syncs → kargo-demo-prod namespace
```

## Promotion Task Variables

| Variable | Value |
|---|---|
| `gitopsRepo` | `https://github.com/MikaKolk12/kargo-demo` |
| `imageRepoWeb1Dev` | `ghcr.io/mikakolk12/dev/web1` |
| `imageRepoWeb2Dev` | `ghcr.io/mikakolk12/dev/web2` |
| `imageRepoWeb1Acc` | `ghcr.io/mikakolk12/acc/web1` |
| `imageRepoWeb2Acc` | `ghcr.io/mikakolk12/acc/web2` |
| `imageRepoWeb1Prod` | `ghcr.io/mikakolk12/prod/web1` |
| `imageRepoWeb2Prod` | `ghcr.io/mikakolk12/prod/web2` |

## Secrets Required

| Secret name | Namespace | Purpose | Who populates |
|---|---|---|---|
| `kargo-demo-repo` | kargo-demo | Git read/write for kargo-demo GitOps repo | User (GitHub PAT, `repo` scope) |
| `kargo-demo-ghcr` | kargo-demo | crane copy between ghcr.io paths | User (GitHub PAT, `write:packages` scope) |

## Cleanup Strategy

- Dev images: delete all `ghcr.io/mikakolk12/dev/web1` tags older than 30 days (short SHA tags accumulate rapidly)
- Acc/prod images: tag format `YY.M.sha` — delete entire month prefix when that month's releases are retired

## What is Applied from Scratch (nothing assumed pre-existing)

- ArgoCD ApplicationSet (`kargo/applicationset.yaml`)
- Kargo Project (`kargo/project.yaml`)
- Kargo git secret (`kargo/secret-git.yaml`)
- Kargo image secret (`kargo/secret-image.yaml`)
- Kargo Warehouse (`kargo/warehouse.yaml`)
- Kargo PromotionTask (`kargo/promotiontask.yaml`)
- Kargo Stages dev/acc/prod (`kargo/stages.yaml`)
- Local registry (`registry/`)
- Full base + stage overlays for web1 and web2
