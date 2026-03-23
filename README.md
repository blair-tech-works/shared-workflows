# shared-workflows

Reusable GitHub Actions workflows for Blair Tech Works repos. Centralizes CI logic so app repos stay thin.

---

## Available Workflows

### `docker-build.yml` — Build & Push to GCP Artifact Registry

```mermaid
graph LR
    subgraph App Repo
        A[Push to main<br>or open PR]
    end

    subgraph shared-workflows
        B[docker-build.yml]
    end

    subgraph Steps
        C[Checkout code]
        D[Auth to GCP]
        E[Configure Docker for AR]
        F[Auto-detect tags]
        G[Build with Buildx + cache]
        H[Push to Artifact Registry]
    end

    subgraph GCP Artifact Registry
        I["us-west1-docker.pkg.dev/<br>sandbox-438722/apps/APP:TAG"]
    end

    A -->|workflow_call| B
    B --> C --> D --> E --> F --> G --> H --> I
```

**Auto-tagging logic:**

| Trigger | Tags Applied |
|---------|-------------|
| Pull Request | `pr-<number>` |
| Push to branch | `sha-<short>`, `latest` |
| GitHub Release | `<semver>`, `latest` |
| Explicit `tags` input | Whatever you specify |

**Usage in your app repo:**

```yaml
# .github/workflows/ci.yml
name: Build & Push
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    uses: blair-tech-works/shared-workflows/.github/workflows/docker-build.yml@main
    with:
      image-name: my-app          # → pushed to .../apps/my-app
    secrets:
      GCP_SA_KEY: ${{ secrets.GCP_SA_KEY }}
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | Yes | — | Image name (e.g. `fieldflow`) |
| `registry` | No | `us-west1-docker.pkg.dev` | AR host |
| `project-id` | No | `sandbox-438722` | GCP project |
| `ar-repo` | No | `apps` | AR repository name |
| `dockerfile` | No | `Dockerfile` | Dockerfile path |
| `build-args` | No | — | Newline-separated `KEY=VALUE` |
| `tags` | No | auto-detected | Comma-separated explicit tags |

---

### `bump-version.yml` — Auto Version Bump & Draft Release

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant BV as bump-version.yml
    participant Rel as GitHub Releases

    Dev->>GH: Merge PR to main
    GH->>BV: Triggers on push to main
    BV->>GH: Read package.json version
    BV->>GH: Bump patch (0.1.0 → 0.1.1)
    BV->>GH: Commit + push + tag
    BV->>Rel: Create draft release (v0.1.1)
    Note over Rel: Review & publish when ready
```

**Usage in your app repo:**

```yaml
# .github/workflows/release.yml
name: Bump Version & Release
on:
  push:
    branches: [main]

jobs:
  version:
    uses: blair-tech-works/shared-workflows/.github/workflows/bump-version.yml@main
    secrets:
      BOT_ACCESS_TOKEN: ${{ secrets.BOT_ACCESS_TOKEN }}
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `version-type` | No | `patch` | Semver bump: `patch`, `minor`, `major` |

---

## Full CI/CD Pipeline

Combining both workflows gives you a complete pipeline:

```mermaid
graph TB
    subgraph "App Repo (e.g. fieldflow)"
        A[Merge PR to main]
    end

    subgraph "shared-workflows"
        B[bump-version.yml]
        C[docker-build.yml]
    end

    subgraph "GitHub"
        D[v0.1.1 tag + draft release]
    end

    subgraph "GCP Artifact Registry"
        E[app:0.1.1 + app:latest]
    end

    subgraph "GKE Cluster (via nextjs-iac)"
        F[Flux detects new tag]
        G[Rolling update → live]
    end

    A --> B
    B --> D
    D -->|publish release| C
    C --> E
    E --> F --> G
```

## Required Secrets

Set these in your app repo (Settings → Secrets → Actions):

| Secret | Purpose |
|--------|---------|
| `GCP_SA_KEY` | Service account JSON with Artifact Registry Writer role |
| `BOT_ACCESS_TOKEN` | PAT with `repo` scope for version bump commits |
