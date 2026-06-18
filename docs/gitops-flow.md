# GitOps Promotion Flows

## Flow 1 — Feature

```mermaid
flowchart TD
    A([Merge feature/* → main]) --> B

    subgraph CI ["kargo-demo-2 — GitHub Actions"]
        B["Build image\nghcr.io/mikakolk12/web1:sha-abc1234"]
    end

    B --> C

    subgraph K8S ["Kargo"]
        C["Warehouse discovers\nsha-abc1234"]
        C --> D["Auto-promote → dev\nimage: sha-abc1234"]
        D --> E{"Promote to acc?"}
        E --> |Manual in Kargo UI| F["Promote → acc\nimage: sha-abc1234"]
        F --> G
    end

    subgraph GHA ["kargo-demo — GitHub Actions\non push to stage/acc"]
        G["1. Retag image → 26.6.1\n2. Create GitHub tag 26.6.1\n   in kargo-demo-2\n3. Update acc manifests\n   to use 26.6.1"]
    end

    G --> H

    subgraph K8S2 ["Kargo"]
        H{"Promote to prod?"}
        H --> |Manual in Kargo UI| I["Promote → prod\nimage: sha-abc1234"]
        I --> J
    end

    subgraph GHA2 ["kargo-demo — GitHub Actions\non push to stage/prod"]
        J["Read CalVer 26.6.1\nfrom stage/acc\nUpdate prod manifests\nNo new tag created"]
    end

    style CI fill:#dbeafe,stroke:#3b82f6
    style K8S fill:#dcfce7,stroke:#22c55e
    style GHA fill:#fef9c3,stroke:#eab308
    style K8S2 fill:#dcfce7,stroke:#22c55e
    style GHA2 fill:#fef9c3,stroke:#eab308
```

## Flow 2 — Hotfix

```mermaid
flowchart TD
    A([Merge hotfix/* → main]) --> B

    subgraph CI ["kargo-demo-2 — GitHub Actions"]
        B["Build image\nghcr.io/mikakolk12/web1:hotfix-def5678"]
    end

    B --> C

    subgraph KARGO ["Kargo"]
        C["Warehouse discovers\nhotfix-def5678"]
        C --> D{"Deploy to?"}
        D --> |Direct promote ACC| E["Deploy → acc\nimage: hotfix-def5678\n(skips dev)"]
        D --> |Direct promote PROD| F["Deploy → prod\nimage: hotfix-def5678\n(skips dev + acc)"]
    end

    style CI fill:#dbeafe,stroke:#3b82f6
    style KARGO fill:#dcfce7,stroke:#22c55e
```

## Tag naming

| Flow    | CI tag             | After acc promotion | After prod promotion |
|---------|--------------------|---------------------|----------------------|
| Feature | `sha-abc1234`      | `26.6.1`            | `26.6.1` (same)      |
| Hotfix  | `hotfix-def5678`   | `hotfix-def5678`    | `hotfix-def5678`     |

## Secrets required in kargo-demo repo

| Secret        | Purpose                                              |
|---------------|------------------------------------------------------|
| `CALVER_PAT`  | GitHub PAT (repo scope) to create tags in kargo-demo-2 and read/push images to GHCR |
