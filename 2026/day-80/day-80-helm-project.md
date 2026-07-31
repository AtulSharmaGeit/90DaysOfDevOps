# Helm Project: Multi-Environment Deployment and CI/CD
> Two days of Helm - chart basics and a custom chart for the AI-BankApp. Today you bring it all together: environment-specific values for dev, staging, and production, Helm hooks for pre-install validation, chart packaging, and full CI/CD integration with GitHub Actions and ArgoCD.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

## Table of Contents
1. [Create Environment-Specific Values](#1-create-environment-specific-values)
2. [Add Helm Hooks and Tests](#2-add-helm-hooks)
3. [Package and Version the Chart](#3-package-and-version-the-chart)
4. [Helm in the AI-BankApp GitOps Pipeline](#4-understand-helm-in-the-ai-bankapp-gitops-pipeline)
5. [Helm Best Practices for Production](#5-helm-best-practices-for-production)
6. [Clean Up and Review](#6-clean-up-and-review)

## 1. Create Environment-Specific Values
One chart, three environments. The AI-BankApp runs with fundamentally different resource profiles and feature flags in dev, staging, and production.

### Navigate to our Helm Chart
Go to our chart directory (where `Chart.yaml` and `values.yaml` exist).
```bash
cd helm-chart/bankapp
```

### Create the Dev Values File
`bankapp/values-dev.yaml` — minimal resources, no autoscaling, Kind-compatible storage:
```yaml
bankapp:
  replicaCount: 1
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "latest"
    pullPolicy: Always
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "250m"
  autoscaling:
    enabled: false

mysql:
  enabled: true
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "250m"
  persistence:
    size: 2Gi
    storageClass: standard

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1.5Gi"
      cpu: "1000m"
  persistence:
    size: 5Gi
    storageClass: standard

storageClass:
  create: false
```

### Create the Staging Values File
`bankapp/values-staging.yaml` — pinned image tag, light autoscaling, gp3 storage:
```yaml
bankapp:
  replicaCount: 2
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 3
    targetCPUUtilization: 75

mysql:
  enabled: true
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  persistence:
    size: 5Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: StagingPass@456
  mysqlUser: root
  mysqlPassword: StagingPass@456

storageClass:
  create: true
```

### Create the Prod Values File
`bankapp/values-prod.yaml` — full resources, aggressive autoscaling, large storage, gateway enabled:
```yaml
bankapp:
  replicaCount: 4
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 70

mysql:
  enabled: true
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  persistence:
    size: 20Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "2Gi"
      cpu: "900m"
    limits:
      memory: "2.5Gi"
      cpu: "1500m"
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: ProdSecure@789
  mysqlUser: root
  mysqlPassword: ProdSecure@789

storageClass:
  create: true

gateway:
  enabled: true
```

### Environment Comparison
| Setting | Dev | Staging | Production |
|---|---|---|---|
| BankApp replicas | 1 (fixed) | 2–3 (HPA) | 2–4 (HPA) |
| Image tag | `latest` | `v1.2.0` | `v1.2.0` |
| Autoscaling | Disabled | Enabled (75% CPU) | Enabled (70% CPU) |
| MySQL storage | 2 Gi | 5 Gi | 20 Gi |
| MySQL memory limit | 256 Mi | 512 Mi | 1 Gi |
| Ollama memory limit | 1.5 Gi | Default | 2.5 Gi |
| Gateway | Disabled | Disabled | Enabled |
| StorageClass | `standard` (Kind) | `gp3` (EBS) | `gp3` (EBS) |

### Deploy and Verify
```bash
# Dev — deploy to Kind cluster
helm install bankapp-dev bankapp/ -f values-dev.yaml -n dev --create-namespace
 
# Staging — verify replica count without deploying
helm template bankapp-staging bankapp/ -f values-staging.yaml | grep "replicas:"
 
# Production — verify replica count without deploying
helm template bankapp-prod bankapp/ -f values-prod.yaml | grep "replicas:"
```
Same chart, three completely different deployments.

## 2. Add Helm Hooks
The AI-BankApp uses init containers to wait for MySQL. Helm hooks run Kubernetes Jobs at specific points in the release lifecycle. They complement the init containers already in the BankApp Deployment - hooks run at the Helm level (before resources are created), while init containers run at the pod level (before the container starts).

Helm hooks offer another approach -- running pre-install jobs.

### Navigate to Templates Directory
Go to our chart’s templates folder:
```bash
cd helm-chart/bankapp/templates
```

### Create the Pre-Install Job
`bankapp/templates/pre-install-job.yaml`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "bankapp.fullname" . }}-db-ready
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
        - name: db-check
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Waiting for MySQL to be ready..."
              until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do
                echo "MySQL not ready, retrying in 3s..."
                sleep 3
              done
              echo "MySQL is ready!"
          resources:
            requests: { memory: "32Mi", cpu: "50m" }
            limits: { memory: "64Mi", cpu: "100m" }
      restartPolicy: Never
  backoffLimit: 10
```

### How hooks work in the AI-BankApp context:
- `helm.sh/hook: pre-install,pre-upgrade` -- runs before install and before upgrade
- This ensures MySQL is up before the BankApp Deployment is created
- `helm.sh/hook-delete-policy: before-hook-creation` -- deletes the old job before creating a new one on re-runs
- Combined with init containers in the Deployment, this provides defense-in-depth

**Available hook types:**
| Hook | Timing | Common Use Case |
|---|---|---|
| `pre-install` | Before any resources are created | Wait for dependencies, validate cluster state |
| `post-install` | After all resources are created | Run database migrations |
| `pre-upgrade` | Before upgrade applies | Pre-flight checks, backups |
| `post-upgrade` | After upgrade completes | Smoke tests, cache warmup |
| `pre-delete` | Before `helm uninstall` | Database backup before teardown |
| `test` | On `helm test` | Connectivity and health checks |

### Create the Helm test
Create a new folder for tests if not present:
```bash
mkdir -p tests
```
Create the test file:
`bankapp/templates/tests/test-connection.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "bankapp.fullname" . }}-test
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ['sh', '-c', 'wget -qO- http://{{ include "bankapp.fullname" . }}-service:8080/actuator/health']
  restartPolicy: Never
```

### Deploy and run the test
Deploy our chart (example for dev):
```bash
helm install bankapp-dev ../ -f ../values-dev.yaml -n dev --create-namespace
```
Run the test:
```bash
helm test bankapp-dev -n dev
```
This will hit the Spring Boot health endpoint (`/actuator/health`) and confirm the app is running.

## 3. Package and Version the Chart
### Lint the Chart
Linting checks for errors or bad practices in our chart.
```bash
helm lint bankapp/
```
- If successful, we’ll see `0 chart(s) linted, 0 chart(s) failed`.
- If errors appear, fix them before packaging (e.g., missing values, bad indentation).

### Package the Chart
Create a `.tgz` archive of our chart:
```bash
helm package bankapp/
```
- This generates `bankapp-0.1.0.tgz` in our current directory.
- The version comes from `Chart.yaml`.

### Bump the Version
After adding hooks, the chart structure has changed - bump both versions:
```bash
vim bankapp/Chart.yaml
```
Update:
```yaml
version: 0.2.0        # Chart structure changed (added hooks)
appVersion: "1.1.0"   # Application version updated
```
- `version`: Helm chart version (structure/config changes).
- `appVersion`: Application version (e.g., BankApp release).

### Re-Package
Run packaging again:
```bash
helm package bankapp/
```
Now we’ll have:
- `bankapp-0.1.0.tgz`
- `bankapp-0.2.0.tgz`

### Install from a Package
Instead of installing from the chart directory, install directly from the `.tgz` file:
```bash
helm install my-bankapp bankapp-0.2.0.tgz -f bankapp/values-dev.yaml -n bankapp --create-namespace
```
- This proves our packaged chart works independently.
- Use `helm list -n bankapp` to confirm installation.

### Create a Chart Repository Index
This step prepares our chart for sharing (e.g., via GitHub Pages).
```bash
mkdir chart-repo
cp bankapp-*.tgz chart-repo/
helm repo index chart-repo/ --url https://your-username.github.io/helm-charts
cat chart-repo/index.yaml
```
- `index.yaml` lists available chart versions.
- Push `chart-repo/` to GitHub Pages → others can add our repo with:
    ```bash
    helm repo add my-bankapp https://your-username.github.io/helm-charts
    ```

## 4. Understand Helm in the AI-BankApp GitOps Pipeline
The AI-BankApp uses a GitOps pipeline. Study how Helm could integrate:

### Current pipeline (from `.github/workflows/gitops-ci.yml`):
```
Developer pushes code
  -> GitHub Actions builds Docker image
  -> Tags with git commit SHA
  -> Updates image tag in k8s/bankapp-deployment.yml via sed
  -> Commits the change back to the repo
  -> ArgoCD detects the change and syncs to EKS
```
This works, but editing raw manifests with `sed` is fragile - it can break YAML structure and doesn't scale across environments.

### With Helm, the pipeline becomes:
```
Developer pushes code
  -> GitHub Actions builds Docker image
  -> Tags with git commit SHA
  -> Updates image.tag in helm-chart/values.yaml (or values-prod.yaml)
  -> Commits the change back to the repo
  -> ArgoCD detects the change and runs helm upgrade on EKS
```
Only the values file is touched. Templates are never modified by CI.

### Update GitHub Actions workflow
In `.github/workflows/gitops-ci.yml`, replace the sed step with a Helm values update.

Here is how the CI step would look with Helm (reference pattern):
```yaml
# In the GitHub Actions workflow
- name: Update Helm values with new image tag
  run: |
    TAG=${{ steps.tag.outputs.sha_short }}
    yq -i '.bankapp.image.tag = "'$TAG'"' helm-chart/bankapp/values-prod.yaml

- name: Commit updated Helm values
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add helm-chart/bankapp/values-prod.yaml
    git diff --staged --quiet || git commit -m "ci: update bankapp image to $TAG [skip ci]"
    git push
```
- `yq` is preferred over `sed` for YAML mutations — it understands YAML structure and preserves formatting, while `sed` operates on raw text and can silently corrupt indentation.

### Update ArgoCD Application
Currently:
```yaml
# Current: raw manifests
source:
  path: k8s
```
Switch to Helm:
```yaml
# With Helm
source:
  path: helm-chart/bankapp
  helm:
    valueFiles:
      - values-prod.yaml
```

ArgoCD natively supports Helm charts - it renders templates and applies the result, tracking drift against the rendered output.

### What are the advantages of ArgoCD syncing a Helm chart vs raw manifests?
**Ans.** Raw Manifests vs. Helm in a GitOps Pipeline
| Dimension | Raw Manifests + ArgoCD | Helm Chart + ArgoCD |
|---|---|---|
| **CI mutation** | `sed` edits YAML files directly | `yq` updates a single values key |
| **Multi-environment** | Separate manifest directories | Separate values files, one chart |
| **Drift detection** | Against static YAML | Against rendered Helm output |
| **Rollback** | `git revert` + ArgoCD sync | `helm rollback` or revert the values commit |
| **Secret injection** | Hardcoded or patched in CI | `--set` flags at deploy time, never committed |
| **Upgrade safety** | No built-in mechanism | `--atomic` rolls back on failure |

## 5. Helm Best Practices for Production
Review these patterns used in production AI-BankApp deployments:

### 1. Always Use `helm upgrade --install`
This ensures our release is created if missing, or upgraded if it already exists.
```bash
helm upgrade --install bankapp bankapp/ \
  -f bankapp/values-prod.yaml \
  --set bankapp.image.tag=$GIT_SHA \
  -n bankapp --create-namespace \
  --wait --timeout 300s \
  --atomic
```
| Flag | Effect |
|---|---|
| `--install` | Creates the release if it doesn't exist; upgrades if it does — idempotent |
| `--set bankapp.image.tag=$GIT_SHA` | Pins the deployment to an exact git commit, not a mutable tag |
| `--wait` | Blocks until all pods, PVCs, and services reach a ready state |
| `--atomic` | Automatically rolls back to the previous revision if the upgrade fails |
 
This prevents half-deployed states in CI/CD and ensures every pipeline run is idempotent.

### 2. Use `helm diff` Before Upgrading
Install the plugin once:
```bash
helm plugin install https://github.com/databus23/helm-diff
```
Check changes before applying:
```bash
helm diff upgrade bankapp bankapp/ -f bankapp/values-prod.yaml
```
- Shows exactly what would change (replicas, resources, secrets, etc.) before we commit to the upgrade.
- Prevents blind upgrades and surprises in production.

### 3. Add Resource Quotas per Namespace
Define namespace limits in `templates/resourcequota.yaml`:
```yaml
# Add to templates/resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ include "bankapp.fullname" . }}-quota
  namespace: {{ .Release.Namespace }}
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```
- Prevents one app from consuming all cluster resources.
- Enforces fair usage across namespaces.

### 4. Never Store Real Secrets in `values.yaml`
The `secrets` block in `values.yaml` is acceptable for local development defaults. In production, always inject credentials at deploy time via CI/CD pipeline secrets:
```bash
helm upgrade --install bankapp bankapp/ \
  -f values-prod.yaml \
  --set secrets.mysqlRootPassword=$MYSQL_ROOT_PASSWORD \
  --set secrets.mysqlPassword=$MYSQL_PASSWORD
```
 
For a more robust solution, use an external secrets manager:
| Option | How It Works |
|---|---|
| **External Secrets Operator** | Syncs secrets from AWS Secrets Manager, GCP Secret Manager, or Vault into Kubernetes Secrets |
| **Sealed Secrets** | Encrypts secrets into `SealedSecret` objects safe to commit to Git |
| **HashiCorp Vault** | Full secrets lifecycle management with dynamic credentials |
 
All three keep plaintext credentials out of Git history entirely.

## 6. Clean Up and Review
### Check what we have Deployed
```bash
helm list -A
```
- This shows all Helm releases across namespaces.
- Verify that `bankapp-dev`, `bankapp-staging`, or `bankapp-prod` are present.

### Reflect and Document the 3-Day Helm Journey
| Day | Concept | AI-BankApp Connection |
|-----|---------|----------------------|
| 78 | Helm install, repos, values, upgrade, rollback | Deployed MySQL for the BankApp via Bitnami chart |
| 79 | Custom chart from scratch, Go templates | Converted 12 raw `k8s/` manifests into a Helm chart |
| 80 | Multi-env values, hooks, packaging, CI/CD | Production-ready chart with dev/staging/prod configs |

### When would we use Helm vs raw manifests vs Kustomize?
| Approach | Best For | Tradeoffs |
|---|---|---|
| **Raw manifests** | Simple, single-environment apps | No templating, no version history, manual multi-env management |
| **Helm** | Multi-environment, complex apps with dependencies and lifecycle hooks | Requires learning Go template syntax; chart maintenance overhead |
| **Kustomize** | Overlaying and patching existing manifests without full templating | Simpler than Helm for small changes; less powerful for complex parameterisation |
 
For the AI-BankApp - three services, HPA, hooks, and three environments - Helm is the right tool. Kustomize would be appropriate if you wanted to apply environment patches on top of the existing `k8s/` manifests without rewriting them.

### Clean up resources
When we’re done testing:
```bash
helm uninstall bankapp-dev -n dev
kubectl delete namespace dev
kind delete cluster --name tws-cluster
```
- Removes Helm release.
- Deletes namespace.
- Tears down Kind cluster.


