# Kargo Full Setup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a complete Kargo + ArgoCD promotion pipeline from scratch with two nginx images (web1/web2), three environments (dev/acc/prod), per-environment ghcr.io registry paths, and automated crane-based image promotion.

**Architecture:** kargo-demo-2 is the source repo — GitHub Actions builds web1 and web2 on every push to main and pushes SHA-tagged images to `ghcr.io/mikakolk12/dev/`. Kargo watches both images, creates freight, and promotes through dev → acc → prod. Each promotion for acc/prod uses `crane copy` to move images between registry paths and tags them with `YY.M.sha`. The GitOps repo (kargo-demo) is updated via kustomize-set-image and committed to stage-specific branches that ArgoCD deploys.

**Tech Stack:** Kargo v1+, ArgoCD, Kustomize, GitHub Actions, ghcr.io, crane, k3s (Rancher Desktop), nginx

## Global Constraints

- GitHub org/user: `MikaKolk12` (lowercase in registry paths: `mikakolk12`)
- GitOps repo URL: `https://github.com/MikaKolk12/kargo-demo`
- Source repo URL: `https://github.com/MikaKolk12/kargo-demo-2`
- Kargo namespace: `kargo-demo`
- ArgoCD namespace: `argocd`
- Stage names: `dev`, `acc`, `prod` (not test/uat)
- Stage branch names: `stage/dev`, `stage/acc`, `stage/prod`
- ArgoCD app names: `kargo-demo-dev`, `kargo-demo-acc`, `kargo-demo-prod`
- K8s namespaces per stage: `kargo-demo-dev`, `kargo-demo-acc`, `kargo-demo-prod`
- Dev image tag: 7-char git SHA (e.g. `abc123f`)
- Acc/prod image tag: `YY.M.sha7` (e.g. `26.6.abc123f`)
- NodePorts: dev-web1=32080, dev-web2=32083, acc-web1=32081, acc-web2=32084, prod-web1=32082, prod-web2=32085
- Secret `kargo-demo-ghcr` in namespace `kargo-demo`: user populates the token value manually
- Secret `kargo-demo-repo` in namespace `kargo-demo`: user populates the GitHub PAT manually
- Working directory for kargo-demo tasks: `C:\Users\MKOLK04\Documents\peridos\kargo-demo`
- Working directory for kargo-demo-2 tasks: `C:\Users\MKOLK04\Documents\peridos\kargo-demo-2`

---

### Task 1: Build kargo-demo-2 source repo (web1, web2, GitHub Actions)

**Files (kargo-demo-2):**
- Create: `web1/index.html`
- Create: `web1/Dockerfile`
- Create: `web2/index.html`
- Create: `web2/Dockerfile`
- Create: `.github/workflows/build.yml`

**Produces:** Two nginx images pushed to `ghcr.io/mikakolk12/dev/web1:<sha>` and `ghcr.io/mikakolk12/dev/web2:<sha>` on every push to main.

- [ ] **Step 1: Create web1/index.html**

```html
<!DOCTYPE html>
<html>
<body>
  <h1>web1</h1>
  <p>Environment: served by nginx</p>
</body>
</html>
```
File: `C:\Users\MKOLK04\Documents\peridos\kargo-demo-2\web1\index.html`

- [ ] **Step 2: Create web1/Dockerfile**

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```
File: `C:\Users\MKOLK04\Documents\peridos\kargo-demo-2\web1\Dockerfile`

- [ ] **Step 3: Create web2/index.html**

```html
<!DOCTYPE html>
<html>
<body>
  <h1>web2</h1>
  <p>Environment: served by nginx</p>
</body>
</html>
```
File: `C:\Users\MKOLK04\Documents\peridos\kargo-demo-2\web2\index.html`

- [ ] **Step 4: Create web2/Dockerfile**

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```
File: `C:\Users\MKOLK04\Documents\peridos\kargo-demo-2\web2\Dockerfile`

- [ ] **Step 5: Create GitHub Actions workflow**

```yaml
name: Build and Push

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  OWNER: mikakolk12

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
    - uses: actions/checkout@v4

    - name: Get short SHA
      id: sha
      run: echo "short=$(echo $GITHUB_SHA | head -c7)" >> $GITHUB_OUTPUT

    - name: Log in to ghcr.io
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and push web1
      uses: docker/build-push-action@v5
      with:
        context: web1
        push: true
        tags: ghcr.io/${{ env.OWNER }}/dev/web1:${{ steps.sha.outputs.short }}

    - name: Build and push web2
      uses: docker/build-push-action@v5
      with:
        context: web2
        push: true
        tags: ghcr.io/${{ env.OWNER }}/dev/web2:${{ steps.sha.outputs.short }}
```
File: `C:\Users\MKOLK04\Documents\peridos\kargo-demo-2\.github\workflows\build.yml`

- [ ] **Step 6: Commit and push**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo-2"
git add .
git commit -m "feat: add web1 and web2 nginx apps with GitHub Actions CI"
git push origin main
```

- [ ] **Step 7: Verify CI ran**

Go to `https://github.com/MikaKolk12/kargo-demo-2/actions` and confirm the workflow succeeded. Both packages should appear at `https://github.com/MikaKolk12?tab=packages`.

- [ ] **Step 8: Make packages public**

For each package (`dev/web1`, `dev/web2`), go to:
`https://github.com/users/MikaKolk12/packages/container/dev%2Fweb1/settings`
`https://github.com/users/MikaKolk12/packages/container/dev%2Fweb2/settings`
Set visibility to **Public** so Kargo can discover images without auth.

---

### Task 2: Rebuild GitOps repo base manifests (web1 + web2)

**Files (kargo-demo):**
- Delete: `base/deploy.yaml`, `base/service.yaml` (replaced by per-app files)
- Create: `base/deploy-web1.yaml`
- Create: `base/deploy-web2.yaml`
- Create: `base/service-web1.yaml`
- Create: `base/service-web2.yaml`
- Modify: `base/kustomization.yaml`

**Produces:** Base Kustomize layer defining both web1 and web2 deployments and services.

- [ ] **Step 1: Delete old single-app base files**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo"
Remove-Item base\deploy.yaml
Remove-Item base\service.yaml
```

- [ ] **Step 2: Create base/deploy-web1.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kargo-demo-web1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kargo-demo-web1
  template:
    metadata:
      labels:
        app: kargo-demo-web1
    spec:
      containers:
      - name: nginx
        image: ghcr.io/mikakolk12/dev/web1
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
          readOnly: true
      volumes:
      - name: content
        configMap:
          name: kargo-demo-web1-content
```

- [ ] **Step 3: Create base/deploy-web2.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kargo-demo-web2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kargo-demo-web2
  template:
    metadata:
      labels:
        app: kargo-demo-web2
    spec:
      containers:
      - name: nginx
        image: ghcr.io/mikakolk12/dev/web2
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
          readOnly: true
      volumes:
      - name: content
        configMap:
          name: kargo-demo-web2-content
```

- [ ] **Step 4: Create base/service-web1.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kargo-demo-web1
spec:
  type: NodePort
  selector:
    app: kargo-demo-web1
  ports:
  - port: 3000
    targetPort: 80
    nodePort: 32000
```

Note: nodePort 32000 is a placeholder. Each stage overlay patches this to the correct port.

- [ ] **Step 5: Create base/service-web2.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kargo-demo-web2
spec:
  type: NodePort
  selector:
    app: kargo-demo-web2
  ports:
  - port: 3000
    targetPort: 80
    nodePort: 32001
```

- [ ] **Step 6: Update base/kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deploy-web1.yaml
- deploy-web2.yaml
- service-web1.yaml
- service-web2.yaml
```

- [ ] **Step 7: Commit**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo"
git add base/
git commit -m "feat: rebuild base manifests for web1 and web2"
```

---

### Task 3: Rebuild stage overlays (dev, acc, prod)

**Files (kargo-demo):**
- Delete: `stages/test/`, `stages/uat/` directories and all contents
- Create: `stages/dev/kustomization.yaml`, `stages/dev/configmap-web1.yaml`, `stages/dev/configmap-web2.yaml`
- Create: `stages/acc/kustomization.yaml`, `stages/acc/configmap-web1.yaml`, `stages/acc/configmap-web2.yaml`
- Modify: `stages/prod/kustomization.yaml`, `stages/prod/configmap-web1.yaml`, `stages/prod/configmap-web2.yaml`

**Produces:** Three Kustomize overlays that patch NodePorts and provide environment-labelled HTML content.

- [ ] **Step 1: Remove old test and uat stage directories**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo"
Remove-Item -Recurse -Force stages\test
Remove-Item -Recurse -Force stages\uat
```

- [ ] **Step 2: Create stages/dev/configmap-web1.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kargo-demo-web1-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>dev - web1</h1>
    </body>
    </html>
```

- [ ] **Step 3: Create stages/dev/configmap-web2.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kargo-demo-web2-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>dev - web2</h1>
    </body>
    </html>
```

- [ ] **Step 4: Create stages/dev/kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base
- configmap-web1.yaml
- configmap-web2.yaml

patches:
- target:
    kind: Service
    name: kargo-demo-web1
  patch: |-
    - op: replace
      path: /spec/ports/0/nodePort
      value: 32080
- target:
    kind: Service
    name: kargo-demo-web2
  patch: |-
    - op: replace
      path: /spec/ports/0/nodePort
      value: 32083
```

- [ ] **Step 5: Create stages/acc/configmap-web1.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kargo-demo-web1-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>acc - web1</h1>
    </body>
    </html>
```

- [ ] **Step 6: Create stages/acc/configmap-web2.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kargo-demo-web2-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>acc - web2</h1>
    </body>
    </html>
```

- [ ] **Step 7: Create stages/acc/kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base
- configmap-web1.yaml
- configmap-web2.yaml

patches:
- target:
    kind: Service
    name: kargo-demo-web1
  patch: |-
    - op: replace
      path: /spec/ports/0/nodePort
      value: 32081
- target:
    kind: Service
    name: kargo-demo-web2
  patch: |-
    - op: replace
      path: /spec/ports/0/nodePort
      value: 32084
```

- [ ] **Step 8: Replace stages/prod/configmap.yaml with two configmaps**

Delete the old single configmap:
```powershell
Remove-Item "C:\Users\MKOLK04\Documents\peridos\kargo-demo\stages\prod\configmap.yaml"
```

Create `stages/prod/configmap-web1.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kargo-demo-web1-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>prod - web1</h1>
    </body>
    </html>
```

Create `stages/prod/configmap-web2.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kargo-demo-web2-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <body>
      <h1>prod - web2</h1>
    </body>
    </html>
```

- [ ] **Step 9: Replace stages/prod/kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base
- configmap-web1.yaml
- configmap-web2.yaml

patches:
- target:
    kind: Service
    name: kargo-demo-web1
  patch: |-
    - op: replace
      path: /spec/ports/0/nodePort
      value: 32082
- target:
    kind: Service
    name: kargo-demo-web2
  patch: |-
    - op: replace
      path: /spec/ports/0/nodePort
      value: 32085
```

- [ ] **Step 10: Commit**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo"
git add stages/
git commit -m "feat: rebuild stage overlays for dev/acc/prod with web1 and web2"
```

---

### Task 4: Write all Kargo and ArgoCD manifests

**Files (kargo-demo/kargo/):**
- Replace: `kargo/warehouse.yaml`
- Replace: `kargo/promotiontask.yaml`
- Create: `kargo/project.yaml`
- Create: `kargo/secret-git.yaml`
- Create: `kargo/secret-image.yaml`
- Create: `kargo/stages.yaml`
- Create: `kargo/applicationset.yaml`

**Produces:** All Kargo CRDs and ArgoCD ApplicationSet needed for the full pipeline.

- [ ] **Step 1: Create kargo/project.yaml**

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Project
metadata:
  name: kargo-demo
```

- [ ] **Step 2: Create kargo/secret-git.yaml**

This secret gives Kargo read/write access to the GitOps repo. The user populates the token after applying.

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-repo
  namespace: kargo-demo
  labels:
    kargo.akuity.io/cred-type: git
stringData:
  repoURL: https://github.com/MikaKolk12/kargo-demo
  username: MikaKolk12
  password: REPLACE_WITH_GITHUB_PAT
```

- [ ] **Step 3: Create kargo/secret-image.yaml**

This secret gives crane write access to ghcr.io for image copying. The user populates the token after applying.

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-ghcr
  namespace: kargo-demo
stringData:
  username: MikaKolk12
  password: REPLACE_WITH_GITHUB_PAT_WRITE_PACKAGES
```

- [ ] **Step 4: Replace kargo/warehouse.yaml**

Watches both dev images using `NewestBuild` strategy (works with SHA tags).

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: kargo-demo
  namespace: kargo-demo
spec:
  interval: 1m0s
  subscriptions:
  - image:
      repoURL: ghcr.io/mikakolk12/dev/web1
      imageSelectionStrategy: NewestBuild
      discoveryLimit: 5
  - image:
      repoURL: ghcr.io/mikakolk12/dev/web2
      imageSelectionStrategy: NewestBuild
      discoveryLimit: 5
```

- [ ] **Step 5: Replace kargo/promotiontask.yaml**

The task handles three cases via `ctx.stage`:
- `dev`: kustomize-set-image using dev registry paths directly
- `acc`: crane copy dev→acc with YY.M.sha tag, then kustomize-set-image with acc paths
- `prod`: crane copy acc→prod with same YY.M.sha tag, then kustomize-set-image with prod paths

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: demo-promo-process
  namespace: kargo-demo
spec:
  vars:
  - name: gitopsRepo
    value: https://github.com/MikaKolk12/kargo-demo
  - name: web1DevRepo
    value: ghcr.io/mikakolk12/dev/web1
  - name: web2DevRepo
    value: ghcr.io/mikakolk12/dev/web2
  steps:
  - uses: git-clone
    config:
      repoURL: ${{ vars.gitopsRepo }}
      checkout:
      - branch: main
        path: ./src
      - branch: stage/${{ ctx.stage }}
        create: true
        path: ./out

  - uses: git-clear
    config:
      path: ./out

  # --- dev stage: use dev images directly ---
  - as: update-dev
    uses: kustomize-set-image
    if: ${{ ctx.stage == "dev" }}
    config:
      path: ./src/base
      images:
      - image: ${{ vars.web1DevRepo }}
        tag: ${{ imageFrom(vars.web1DevRepo).Tag }}
      - image: ${{ vars.web2DevRepo }}
        tag: ${{ imageFrom(vars.web2DevRepo).Tag }}

  # --- acc stage: crane copy dev→acc with YY.M.sha tag, update kustomization ---
  - uses: exec
    if: ${{ ctx.stage == "acc" }}
    config:
      command: sh
      args:
      - -c
      - |
        SHA=${{ imageFrom(vars.web1DevRepo).Tag }}
        TAG=$(date +%y.%-m).${SHA}
        crane copy ghcr.io/mikakolk12/dev/web1:${SHA} ghcr.io/mikakolk12/acc/web1:${TAG} --all
        crane copy ghcr.io/mikakolk12/dev/web2:${SHA} ghcr.io/mikakolk12/acc/web2:${TAG} --all
        cd ./src/base
        kustomize edit set image ghcr.io/mikakolk12/dev/web1=ghcr.io/mikakolk12/acc/web1:${TAG}
        kustomize edit set image ghcr.io/mikakolk12/dev/web2=ghcr.io/mikakolk12/acc/web2:${TAG}

  # --- prod stage: crane copy acc→prod with same YY.M.sha tag, update kustomization ---
  - uses: exec
    if: ${{ ctx.stage == "prod" }}
    config:
      command: sh
      args:
      - -c
      - |
        SHA=${{ imageFrom(vars.web1DevRepo).Tag }}
        TAG=$(date +%y.%-m).${SHA}
        crane copy ghcr.io/mikakolk12/acc/web1:${TAG} ghcr.io/mikakolk12/prod/web1:${TAG} --all
        crane copy ghcr.io/mikakolk12/acc/web2:${TAG} ghcr.io/mikakolk12/prod/web2:${TAG} --all
        cd ./src/base
        kustomize edit set image ghcr.io/mikakolph12/dev/web1=ghcr.io/mikakolk12/prod/web1:${TAG}
        kustomize edit set image ghcr.io/mikakolph12/dev/web2=ghcr.io/mikakolk12/prod/web2:${TAG}

  - as: update-prod
    uses: kustomize-set-image
    if: ${{ ctx.stage == "prod" }}
    config:
      path: ./src/base
      images:
      - image: ghcr.io/mikakolk12/prod/web1
        tag: ${{ imageFrom(vars.web1DevRepo).Tag }}
      - image: ghcr.io/mikakolk12/prod/web2
        tag: ${{ imageFrom(vars.web2DevRepo).Tag }}

  # --- shared: build, commit, push, sync ---
  - uses: kustomize-build
    config:
      path: ./src/stages/${{ ctx.stage }}
      outPath: ./out

  - as: commit
    uses: git-commit
    config:
      path: ./out
      message: "Promoted ${{ ctx.stage }}: web1 + web2"

  - uses: git-push
    config:
      path: ./out

  - uses: argocd-update
    config:
      apps:
      - name: kargo-demo-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.gitopsRepo }}
          desiredRevision: ${{ task.outputs.commit.commit }}
```

- [ ] **Step 6: Create kargo/stages.yaml**

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: dev
  namespace: kargo-demo
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: kargo-demo
    sources:
      direct: true
  promotionTemplate:
    spec:
      steps:
      - task:
          name: demo-promo-process
        as: promo-process
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: acc
  namespace: kargo-demo
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: kargo-demo
    sources:
      stages:
      - dev
  promotionTemplate:
    spec:
      steps:
      - task:
          name: demo-promo-process
        as: promo-process
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: prod
  namespace: kargo-demo
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: kargo-demo
    sources:
      stages:
      - acc
  promotionTemplate:
    spec:
      steps:
      - task:
          name: demo-promo-process
        as: promo-process
```

- [ ] **Step 7: Create kargo/applicationset.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kargo-demo
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - stage: dev
      - stage: acc
      - stage: prod
  template:
    metadata:
      name: kargo-demo-{{stage}}
      annotations:
        kargo.akuity.io/authorized-stage: kargo-demo:{{stage}}
    spec:
      project: default
      source:
        repoURL: https://github.com/MikaKolk12/kargo-demo
        targetRevision: stage/{{stage}}
        path: .
      destination:
        server: https://kubernetes.default.svc
        namespace: kargo-demo-{{stage}}
      syncPolicy:
        syncOptions:
        - CreateNamespace=true
```

- [ ] **Step 8: Commit**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo"
git add kargo/
git commit -m "feat: add all Kargo and ArgoCD manifests for dev/acc/prod pipeline"
```

---

### Task 5: Clean up old cluster resources and apply everything from scratch

**Produces:** Cluster running the new dev/acc/prod pipeline with web1 and web2.

- [ ] **Step 1: Delete old Kargo stages and ArgoCD apps**

```powershell
kubectl delete stage test uat prod -n kargo-demo --ignore-not-found
kubectl delete applicationset kargo-demo -n argocd --ignore-not-found
kubectl delete application kargo-demo-test kargo-demo-uat kargo-demo-prod -n argocd --ignore-not-found
```

- [ ] **Step 2: Delete old Kargo Warehouse and PromotionTask**

```powershell
kubectl delete warehouse kargo-demo -n kargo-demo --ignore-not-found
kubectl delete promotiontask demo-promo-process -n kargo-demo --ignore-not-found
```

- [ ] **Step 3: Delete old Project and secrets (will be recreated)**

```powershell
kubectl delete project kargo-demo --ignore-not-found
kubectl delete secret kargo-demo-repo kargo-demo-ghcr -n kargo-demo --ignore-not-found
```

- [ ] **Step 4: Apply Kargo Project first (creates the namespace)**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo"
kubectl apply -f kargo/project.yaml
```

Wait ~5 seconds for the namespace to be created:
```powershell
kubectl wait --for=condition=Ready project/kargo-demo --timeout=30s
```

- [ ] **Step 5: Apply secrets**

```powershell
kubectl apply -f kargo/secret-git.yaml
kubectl apply -f kargo/secret-image.yaml
```

- [ ] **Step 6: Apply Warehouse, PromotionTask, Stages**

```powershell
kubectl apply -f kargo/warehouse.yaml
kubectl apply -f kargo/promotiontask.yaml
kubectl apply -f kargo/stages.yaml
```

- [ ] **Step 7: Apply ApplicationSet**

```powershell
kubectl apply -f kargo/applicationset.yaml
```

- [ ] **Step 8: Apply local registry**

```powershell
kubectl apply -f registry/
# Wait and re-apply if namespace wasn't ready
kubectl apply -f registry/
```

- [ ] **Step 9: Verify Warehouse is discovering images**

```powershell
kubectl get warehouse kargo-demo -n kargo-demo -o jsonpath='{.status.conditions[?(@.type=="Ready")].message}'
```
Expected output: `Successfully discovered artifacts from 2 subscriptions`

- [ ] **Step 10: Verify Stages exist**

```powershell
kubectl get stages -n kargo-demo
```
Expected: `dev`, `acc`, `prod` all listed.

- [ ] **Step 11: Verify ArgoCD apps were created**

```powershell
kubectl get applications -n argocd | grep kargo-demo
```
Expected: `kargo-demo-dev`, `kargo-demo-acc`, `kargo-demo-prod`

- [ ] **Step 12: Push all changes to remote**

```powershell
cd "C:\Users\MKOLK04\Documents\peridos\kargo-demo"
git push origin main
```

---

### Task 6: Populate secrets and verify first promotion

**Produces:** Working end-to-end promotion from dev through acc to prod.

- [ ] **Step 1: Populate git secret**

The user must create a GitHub PAT with `repo` scope at `https://github.com/settings/tokens/new` and patch the secret:

```powershell
kubectl patch secret kargo-demo-repo -n kargo-demo `
  --type=merge `
  -p '{"stringData":{"password":"<YOUR_GITHUB_PAT>"}}'
```

- [ ] **Step 2: Populate image secret**

The user must create a GitHub PAT with `write:packages` scope and patch the secret:

```powershell
kubectl patch secret kargo-demo-ghcr -n kargo-demo `
  --type=merge `
  -p '{"stringData":{"password":"<YOUR_GITHUB_PAT_WRITE_PACKAGES>"}}'
```

- [ ] **Step 3: Verify Warehouse discovered freight**

```powershell
kubectl get freight -n kargo-demo
```
Expected: at least one freight item listed.

- [ ] **Step 4: Promote to dev via Kargo UI or CLI**

Open the Kargo UI (typically `https://localhost:31444` or check with `kubectl get svc -n kargo`).
Find the `kargo-demo` project and promote the latest freight to the `dev` stage.

Or via CLI:
```powershell
kargo promote --project kargo-demo --freight <freight-id> --stage dev
```

- [ ] **Step 5: Verify dev deployment**

```powershell
kubectl get pods -n kargo-demo-dev
```
Expected: pods for `kargo-demo-web1` and `kargo-demo-web2` running.

Open `http://localhost:32080` → should show "dev - web1"
Open `http://localhost:32083` → should show "dev - web2"

- [ ] **Step 6: Promote to acc**

In Kargo UI, promote the same freight to `acc` stage. The PromotionTask will:
1. Run `crane copy` to copy dev images to acc path with `26.6.sha` tag
2. Update kustomize, commit to `stage/acc`, trigger ArgoCD sync

Verify:
```powershell
kubectl get pods -n kargo-demo-acc
```
Open `http://localhost:32081` → should show "acc - web1"
Open `http://localhost:32084` → should show "acc - web2"

- [ ] **Step 7: Promote to prod**

Promote to `prod` stage. Verify:
```powershell
kubectl get pods -n kargo-demo-prod
```
Open `http://localhost:32082` → should show "prod - web1"
Open `http://localhost:32085` → should show "prod - web2"
