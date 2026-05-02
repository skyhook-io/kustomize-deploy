# Kustomize Deploy
 
High-level deployment action that handles both GitOps (ArgoCD) and direct kubectl deployments with automatic detection.
 
## Features

- 🔍 **Auto-detects GitOps** — Fixed relative path to `../../argocd/<environment>.yaml|.yml` from the overlay (Skyhook layout), else `managed-by` on built manifests
- 📝 **Updates manifests** - Sets image tags, labels, and environment variables
- 🚀 **Dual mode** - GitOps commit or kubectl apply
- ⏳ **Wait for rollout** - Monitors deployment status
- 🎯 **Namespace management** - Creates namespaces as needed

## Usage

### Single image deployment (recommended - separate image and tag)
```yaml
- name: Deploy to Kubernetes
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    image: myregistry.io/backend
    tag: v1.2.3
    environment: production
```

### Single image (backwards compatible - image:tag format)
```yaml
- name: Deploy to Kubernetes
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    image: myregistry.io/backend:v1.2.3
    environment: production
```

### Multiple images deployment
```yaml
- name: Deploy service with migrator
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    images_json: |
      [
        {"name": "myregistry.io/backend", "newTag": "v1.2.3"},
        {"name": "myregistry.io/backend-migrator", "newTag": "v1.2.3"}
      ]
    environment: production
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `working_directory` | Working directory for operations | ❌ | '.' |
| `overlay_dir` | Path to kustomize overlay directory | ✅ | - |
| `service_name` | Service name | ✅ | - |
| `image` | Container image name | ❌* | - |
| `tag` | Image tag | ❌* | - |
| `images_json` | Multiple images as JSON array | ❌* | - |
| `ensure_service_image` | Ensure an image entry for `service_name` is included (adds it if missing) | ❌ | `false` |
| `update_all_images` | Apply `tag` to every image entry in `kustomization.yaml`. Mutually exclusive with `image`/`images_json`/`ensure_service_image`. Requires `tag` | ❌ | `false` |
| `environment` | Environment name | ✅ | - |
| `actor` | User deploying | ❌ | `${{ github.actor }}` |
| `run_id` | Run ID for tracking | ❌ | `${{ github.run_id }}` |
| `detect_gitops` | Auto-detect GitOps mode from manifests | ❌ | `true` |
| `force_mode` | Force deployment mode (gitops, kubectl, or auto) | ❌ | `auto` |
| `commit_message` | Commit message for GitOps | ❌ | auto |
| `create_namespace` | Create namespace if it does not exist | ❌ | `true` |
| `wait_timeout` | Timeout for waiting on deployments (seconds) | ❌ | `120` |
| `env_patches` | Environment file patches (JSON format) | ❌ | - |
| `enable_helm` | Pass `--enable-helm` to `kustomize build` (required for overlays using `helmCharts:`). Requires `helm` binary on the runner; fails fast if missing. Set to `false` to skip. | ❌ | `true` |

\* **Image input options** (choose one):
  - Option 1: `image` (with embedded tag, e.g., `registry.io/app:v1.2.3`)
  - Option 2: `image` + `tag` (separate, e.g., `image: registry.io/app`, `tag: v1.2.3`)
  - Option 3: `images_json` (for multiple images)
  - Option 4: `tag` + `update_all_images: true` (bump every image entry in `kustomization.yaml` to the same tag - monorepo releases). Mutually exclusive with `image`/`images_json`/`ensure_service_image`.

## Outputs

| Output | Description |
|--------|-------------|
| `mode` | Deployment mode used (gitops or kubectl) |
| `namespace` | Kubernetes namespace |
| `deployment` | Primary deployment name |
| `managed_by` | Value of `app.kubernetes.io/managed-by` from `kustomize build` when step 2 runs; **empty** when GitOps is chosen via Skyhook Application file only (step 1) |

## How It Works

This action acts as an **orchestrator** that delegates to specialized sub-actions:

### 1. Update Phase
- **kustomize-edit** validates and normalizes all image inputs (handles image:tag format, separate params, or multi-image JSON)
- Sets image tags, version labels, and deployment metadata
- Patches environment variables in config files (if env_patches provided)

### 2. Inspection Phase
- **kustomize-inspect** extracts namespace and workloads from kustomization
- Gets primary deployment name
- Validates kustomization builds successfully

### 3. Detection Phase
Auto mode (`force_mode: auto`, `detect_gitops: true`) picks **gitops** vs **kubectl** in order:

1. **Skyhook layout (fixed relative path)** — Detection runs with working directory = `working_directory`/`overlay_dir` (same folder as your overlay `kustomization.yaml`). From there it checks **only**:
   - `../../argocd/<environment>.yaml` **or**
   - `../../argocd/<environment>.yml`  
   `<environment>` is the `environment` input (e.g. `staging`, `production`).  
   Example: `overlay_dir: deploy/overlays/production` → looks for `deploy/argocd/production.yaml` or `deploy/argocd/production.yml`.  
   There is **no** extra input to customize this path. If your repo uses a different folder depth or Argo CD manifest location, this check does nothing and detection continues with step 2 (use **`managed-by` labels** or **`force_mode: gitops`**).  
   When this file exists: mode is **gitops**. The `managed_by` output is **not** set (empty); use **`force_mode`** / downstream logic keyed on **`mode`** if you need explicit handling.
2. **Otherwise** — inspect built manifests for:
```yaml
app.kubernetes.io/managed-by: argocd  # or flux
```
   If the label is **argocd** or **flux**, mode is **gitops**; else **kubectl**.

### 4. Deploy Phase

**GitOps Mode:**
- Commits changes to git
- Pushes to trigger ArgoCD/Flux sync

**Kubectl Mode:**
- **kustomize-apply** applies manifests to cluster
- Waits for rollout to complete

> **Architecture Note:** This action focuses on deployment orchestration. All image input handling (validation, normalization, format conversion) is delegated to `kustomize-edit`, ensuring a single source of truth.

## Examples

### Auto-detect mode (recommended - separate image and tag)
```yaml
- name: Deploy service
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/${{ inputs.environment }}
    service_name: ${{ inputs.service }}
    image: ${{ inputs.registry }}/${{ inputs.service }}
    tag: ${{ inputs.tag }}
    environment: ${{ inputs.environment }}
```

### Auto-detect mode (backwards compatible - image:tag format)
```yaml
- name: Deploy service
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/${{ inputs.environment }}
    service_name: ${{ inputs.service }}
    image: ${{ inputs.registry }}/${{ inputs.service }}:${{ inputs.tag }}
    environment: ${{ inputs.environment }}
```

### Force GitOps mode
```yaml
- name: Deploy via GitOps
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    image: myregistry.io/backend
    tag: v1.2.3
    environment: production
    force_mode: gitops
    commit_message: "Deploy backend v1.2.3 to production"
```

### Force kubectl mode
```yaml
- name: Direct deployment
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/staging
    service_name: frontend
    image: myregistry.io/frontend
    tag: latest
    environment: staging
    force_mode: kubectl
    wait_timeout: 300
```

### With namespace creation
```yaml
- name: Deploy to new namespace
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/feature
    service_name: api
    image: myregistry.io/api
    tag: feature-123
    environment: feature-123
    create_namespace: true
```

### With environment variable patches
```yaml
- name: Deploy with env patches
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    image: myregistry.io/backend
    tag: v1.2.3
    environment: production
    env_patches: |
      {
        "container.env": {
          "SENTRY_RELEASE": "v1.2.3",
          "BUILD_ID": "${{ github.run_id }}"
        }
      }
```

### With multiple images (e.g., app + migrator)
```yaml
- name: Deploy service with migrator
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: backend
    images_json: |
      [
        {"name": "myregistry.io/backend", "newTag": "v1.2.3"},
        {"name": "myregistry.io/backend-migrator", "newTag": "v1.2.3"}
      ]
    environment: production
```

### Monorepo release: bump every image to the same tag
```yaml
- name: Deploy monorepo release
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: platform
    tag: v2.3.0
    update_all_images: true
    environment: production
```

When `update_all_images: true`, the action reads `kustomization.yaml`, takes every entry under `images:`, and rewrites its `newTag` to the supplied `tag` (preserving `newName` if set). Errors if `image`, `images_json`, or `ensure_service_image` are also set. Fails fast if `tag` is missing, `kustomization.yaml` has no `images[]`, or any entry uses `digest:`.

### Real-world example: Service with database migrations
```yaml
- name: Deploy API with DB migrator
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/production
    service_name: api
    images_json: |
      [
        {"name": "europe-docker.pkg.dev/myproject/api", "newTag": "${{ github.sha }}"},
        {"name": "europe-docker.pkg.dev/myproject/api-migrator", "newTag": "${{ github.sha }}"}
      ]
    environment: production
    env_patches: |
      {
        "container.env": {
          "SENTRY_RELEASE": "${{ github.sha }}",
          "DEPLOYMENT_ID": "${{ github.run_id }}"
        }
      }
```

## Multiple Images Support

The `images_json` input allows you to deploy services with multiple container images in a single deployment. This is useful for:

- **Database migrations**: Deploy your app alongside a migrator image that runs as an init container or Job
- **Sidecar containers**: Update multiple images that run together in the same pod
- **Multi-container services**: Services with multiple containers (e.g., app + proxy, app + logging agent)

### Format

The `images_json` input expects a JSON array:

```json
[
  {
    "name": "registry.io/app",
    "newTag": "v1.2.3"
  },
  {
    "name": "registry.io/app-migrator",
    "newTag": "v1.2.3"
  }
]
```

**Required fields:**
- `name` - Full image name/repository (e.g., `gcr.io/project/image`)
- `newTag` - Tag to deploy (e.g., `v1.2.3`, `latest`, `sha-abc123`)

**Mutual Exclusivity:**
Provide either `images_json` OR `image` (with or without tag), not both. Input validation is handled by the underlying `kustomize-edit` action.

### Common Pattern: Service + Migrator

Many services follow this pattern:
1. Build both `myapp` and `myapp-migrator` images with the same tag
2. Deploy both with `images_json`
3. Migrator runs as Kubernetes Job or init container before the main app starts

```yaml
- name: Deploy with migrator
  uses: skyhook-io/kustomize-deploy@v1
  with:
    overlay_dir: deploy/overlays/${{ inputs.environment }}
    service_name: ${{ inputs.service }}
    images_json: |
      [
        {"name": "registry.io/${{ inputs.service }}", "newTag": "${{ inputs.tag }}"},
        {"name": "registry.io/${{ inputs.service }}-migrator", "newTag": "${{ inputs.tag }}"}
      ]
    environment: ${{ inputs.environment }}
```

## Prerequisites

### For GitOps mode
- Write access to the repository
- ArgoCD configured to watch the repository

### For kubectl mode
- Authenticated kubectl context
- Appropriate RBAC permissions
- Cluster access (use cloud-login action first)

## Error Handling

The action will fail if:
- Overlay directory doesn't exist or is missing kustomization.yaml
- Invalid image inputs (validated by kustomize-edit)
- Kustomization build fails (invalid manifests)
- Git push fails (GitOps mode)
- Kubectl apply fails (kubectl mode)
- Deployment doesn't become ready within timeout

## Notes

- **Skyhook Application path** is not configurable; it must match the layout above or rely on step 2 (`managed-by`) / `force_mode`.
- Always use with cloud-login action for kubectl mode
- GitOps mode requires repository write permissions
- **Image format options:**
  - Single image with embedded tag: `image: registry.io/app:v1.2.3` (backwards compatible)
  - Single image with separate tag: `image: registry.io/app` + `tag: v1.2.3` (recommended)
  - Multiple images: `images_json` with array of `{"name":"...","newTag":"..."}`
- Supports both Deployment and Rollout resources
  
