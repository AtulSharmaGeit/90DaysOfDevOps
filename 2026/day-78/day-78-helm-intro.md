# Introduction to Helm and Chart Basics
> You've deployed applications with raw Kubernetes manifests - writing Deployments, Services, ConfigMaps, and Secrets by hand. The AI-BankApp project has 12 YAML files in its `k8s/` directory. Managing those across dev, staging, and production with slightly different configurations is painful. Helm solves this.
 
Helm is the package manager for Kubernetes. It lets you template, package, version, and deploy Kubernetes applications as reusable units called charts. This guide covers installing Helm, understanding chart structure, and deploying your first application - MySQL, which the AI-BankApp depends on - using a community chart.
 
## Table of Contents
1. [Understanding Helm](#1-understand-helm-concepts)
2. [Install Helm and Explore the AI-BankApp](#2-install-helm-and-explore-the-ai-bankapp)
3. [Deploy MySQL Using a Helm Chart](#3-deploy-mysql-using-a-helm-chart)
4. [Customise Deployments with Values Files](#4-customize-a-deployment-with-values-files)
5. [Manage Releases - Upgrade, Rollback, Uninstall](#5-manage-releases---upgrade-rollback-uninstall)
6. [Explore a Chart's Internal Structure](#6-explore-a-charts-structure)

## 1. Understand Helm Concepts
### What is Helm?
**Ans.** Helm is a package manager for Kubernetes - the equivalent of `apt` for Ubuntu or `yum` for RHEL, but for Kubernetes applications. It packages Kubernetes manifests into reusable, versioned units called **charts** and supports templating so one chart can serve multiple environments.
 
### Core Concepts
| Concept | Description |
|---|---|
| **Chart** | A collection of files describing a set of Kubernetes resources - Deployment, Service, ConfigMap, Secret - bundled as one installable unit |
| **Release** | A running instance of a chart in a cluster. The same chart can be installed multiple times under different release names |
| **Repository** | A hosted collection of charts, analogous to DockerHub for container images |
| **Values** | Configuration that customises a chart per deployment - replicas, image tag, resource limits, feature flags |
 
### Why Helm Over Raw Manifests?
**Ans.** The AI-BankApp's `k8s/` directory contains 12 separate YAML files. Changing the image tag means editing `bankapp-deployment.yml`. Switching environments means manually updating ConfigMaps and Secrets across every file. Helm addresses this across four dimensions:
 
| Problem | Helm Solution |
|---|---|
| Environment-specific config scattered across files | Templating - one chart, environment-specific `values.yaml` per deployment |
| No version history on applied manifests | Charts are versioned; every install/upgrade creates a tracked revision |
| Manually wiring app + database + storage configs | Dependencies - a chart can declare other charts as dependencies |
| Reinventing common infrastructure from scratch | Community repositories with thousands of pre-built charts |

## 2. Install Helm and Explore the AI-BankApp
### Prepare Your Cluster
Choose one option for a local Kubernetes cluster:
- **Kind** (recommended for this block)
- **Minikube**
- **Docker Desktop Kubernetes**

For Kind (using AI‑BankApp config):
```bash
git clone -b feat/gitops https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
cd AI-BankApp-DevOps

kind create cluster --config setup-k8s/kind-config.yml
```
- This creates **1 control plane + 2 worker nodes**.

Verify cluster:
```bash
kubectl cluster-info
kubectl get nodes
```

### Install Helm
**Linux**:
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
Verify installation:
```bash
helm version
```

### Confirm Helm ↔ Cluster Connectivity
Check cluster info:
```bash
kubectl cluster-info
```
List Helm releases (should be empty initially):
```bash
helm list
```

### Explore Raw Manifests
Look at the AI‑BankApp’s `k8s/` directory:
```bash
ls k8s/
```
The `k8s/` directory contains 12 hardcoded YAML files:
| File | Resource Type |
|---|---|
| `bankapp-deployment.yml` | Deployment |
| `mysql-deployment.yml` | Deployment |
| `ollama-deployment.yml` | Deployment |
| `service.yml` | Service |
| `gateway.yml` | Gateway |
| `configmap.yml` | ConfigMap |
| `secrets.yml` | Secret |
| `pv.yml` | PersistentVolume |
| `pvc.yml` | PersistentVolumeClaim |
| `namespace.yml` | Namespace |
| `hpa.yml` | HorizontalPodAutoscaler |
| `cert-manager.yml` | Certificate configuration |
 
Every value - image tags, passwords, resource limits, replica counts - is hardcoded. On Day 79, we convert these into a Helm chart.

## 3. Deploy MySQL Using a Helm Chart
The AI-BankApp depends on MySQL. Rather than applying the four raw files (`mysql-deployment.yml`, `secrets.yml`, `pvc.yml`, `service.yml`), deploy MySQL through Helm in a single command.

### Add the Bitnami Repository
Helm needs chart repositories (like DockerHub for images). Add Bitnami:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```
- `helm repo update` refreshes our local cache so we get the latest chart versions.

### Search for MySQL Chart
Confirm the chart exists:
```bash
helm search repo bitnami/mysql
```
- We’ll see available versions of the MySQL chart.

### Install MySQL with Helm
Deploy MySQL with the configuration expected by AI‑BankApp:
```bash
helm install bankapp-mysql bitnami/mysql \
  --set auth.rootPassword=Test@123 \
  --set auth.database=bankappdb \
  --set primary.resources.requests.memory=256Mi \
  --set primary.resources.requests.cpu=250m \
  --set primary.resources.limits.memory=512Mi \
  --set primary.resources.limits.cpu=500m \
  --set primary.persistence.size=5Gi
```
Compare this single command to the raw manifest approach: we’d need `mysql-deployment.yml`, `secrets.yml`, `pvc.yml`, `pv.yml`, and `service.yml`. Helm bundles all of it.

### Verify What Was Created
Check Helm releases:
```bash
helm list
```

Check Kubernetes resources created by this release:
```bash
# All Kubernetes resources for this release
kubectl get all -l app.kubernetes.io/instance=bankapp-mysql

# Persistent volume claims
kubectl get pvc -l app.kubernetes.io/instance=bankapp-mysql

# Secrets (auto-generated by Helm — no manual base64 encoding)
kubectl get secret -l app.kubernetes.io/instance=bankapp-mysql
```
Helm automatically created the Deployment, Service, Secret, and PVC - resources that would have required four separate files with raw manifests.

### Verify MySQL is Running
Exec into the pod and check databases:
```bash
kubectl exec -it bankapp-mysql-0 -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
```

Expected output includes:
```Code
information_schema
bankappdb
```

## 4. Customize a Deployment with Values Files
`--set` flags work for quick one-off overrides but become unwieldy with multiple settings. Values files are the production-grade approach - a single file that captures all configuration, is version-controllable, and can be swapped per environment.

### Create a Values File
Instead of typing multiple `--set` flags, create a file named `mysql-values.yaml`:
```yaml
auth:
  rootPassword: Test@123
  database: bankappdb
primary:
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi
  persistence:
    size: 5Gi
    storageClass: ""
metrics:
  enabled: true
  serviceMonitor:
    enabled: false
```
- This file captures all configuration in one place. It’s easier to version control and reuse across environments.

### Deploy Using the Values File
```bash
helm install bankapp-mysql-v2 bitnami/mysql -f mysql-values.yaml
```
- This installs a second release (`bankapp-mysql-v2`) using the values file instead of `--set` flags.

### Explore Configurable Options
To see all possible values supported by the chart:
```bash
helm show values bitnami/mysql | head -80
```
- This shows the first 80 lines of defaults.

For a full view, you can run without `head`:
```bash
helm show values bitnami/mysql
```
This is our reference for every knob we can turn. Notice how the chart supports metrics, replication, custom init scripts, and dozens more options -- all through values.

### Verify the Deployment
Check Helm releases:
```bash
helm list
```
Check resources created:
```bash
kubectl get all -l app.kubernetes.io/instance=bankapp-mysql-v2
```
Check PVCs:
```bash
kubectl get pvc -l app.kubernetes.io/instance=bankapp-mysql-v2
```
Check Secrets:
```bash
kubectl get secret -l app.kubernetes.io/instance=bankapp-mysql-v2
```

### Clean Up
Once we’ve verified, uninstall the second release:
```bash
helm uninstall bankapp-mysql-v2
```
- This removes all resources created by that release.

## 5. Manage Releases - Upgrade, Rollback, Uninstall
Helm tracks every install and upgrade as a numbered **revision**, enabling safe upgrades and one-command rollbacks - something raw `kubectl apply` workflows cannot provide.

### Upgrade the Release
Enable the metrics exporter on the existing `bankapp-mysql` release:
```bash
helm upgrade bankapp-mysql bitnami/mysql \
  --set auth.rootPassword=Test@123 \
  --set auth.database=bankappdb \
  --set metrics.enabled=true
```
- This updates the **existing release** (`bankapp-mysql`) instead of creating a new one. Helm tracks this as a new **revision**.

### Check Revision History
View the release history:
```bash
helm history bankapp-mysql
```
Expected output:
| Revision | Status | Description |
|---|---|---|
| 1 | superseded | Initial install - metrics disabled |
| 2 | deployed | Upgrade - metrics enabled |

### Rollback to Previous Version
Revert to the original configuration:
```bash
helm rollback bankapp-mysql 1
```
Helm does not overwrite revision 2. Instead it creates **revision 3**, which represents the state of revision 1. Full history is always preserved.

### Verify Rollback
Check history again:
```bash
helm history bankapp-mysql
```
Expected output:
| Revision | Status | Description |
|---|---|---|
| 1 | superseded | Initial install |
| 2 | superseded | Upgrade - metrics enabled |
| 3 | deployed | Rollback to revision 1 |
 
**Comparison with raw manifests:** `kubectl apply` has no revision history. Rolling back requires a `git revert` or manually re-applying old YAML files - neither of which is tracked by Kubernetes. `helm rollback` is atomic, tracked, and takes a single command.

## 6. Explore a Chart's Structure
Before building a chart for the AI-BankApp, understand what is inside a Helm chart by inspecting the Bitnami MySQL chart locally.

### Pull the MySQL Chart Locally
Download and untar the Bitnami MySQL chart:
```bash
helm pull bitnami/mysql --untar
ls mysql/
```
We’ll see a directory structure like:
```
mysql/
├── Chart.yaml                      # Chart metadata: name, version, appVersion
├── values.yaml                     # Default configuration values
├── charts/                         # Packaged subchart dependencies
└── templates/
    ├── primary/
    │   ├── statefulset.yaml        # StatefulSet with Go template expressions
    │   └── svc.yaml                # Service template
    ├── _helpers.tpl                # Reusable named template definitions
    ├── NOTES.txt                   # Post-install message shown to the user
    └── secrets.yaml                # Secret template
```

### Explore Template Syntax
Open the StatefulSet template:
```bash
vim mysql/templates/primary/statefulset.yaml
```
Look for Go template expressions:
```yaml
replicas: {{ .Values.primary.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```
`{{ .Values.primary.replicaCount }}` reads from `values.yaml` at render time. When you pass `--set primary.replicaCount=3`, that value is injected here — the template itself never changes.

### Inspect Chart.yaml
Open `Chart.yaml`:
```yaml
apiVersion: v2
name: mysql
description: A Helm chart for MySQL
version: 12.2.1      # Chart version (chart structure changes)
appVersion: "8.0.40"  # Version of MySQL inside the chart
```

### Compare the Helm chart approach to the AI-BankApp's raw manifests:
| Aspect | AI-BankApp `k8s/mysql-deployment.yml` | Bitnami MySQL Helm Chart |
|---|---|---|
| **Secrets** | Hardcoded base64 in `secrets.yml` | Auto-generated and managed by Helm |
| **Storage** | Manual StorageClass + separate PVC file | Configured via `persistence.size` value |
| **Replicas** | Hardcoded integer in YAML | `primary.replicaCount` value — overridable |
| **Metrics** | Not included | `metrics.enabled: true` in values |
| **Rollback** | Manual `git revert` + `kubectl apply` | `helm rollback <release> <revision>` |
| **Multi-env config** | Duplicate files per environment | One chart, one `values-<env>.yaml` per env |

### What is the difference between `version` and `appVersion` in Chart.yaml?
**Ans.** `version` tracks changes to the chart itself (templates, defaults, structure). `appVersion` tracks the version of the application the chart deploys. A chart can ship a new `version` (bug fix in a template) without changing the `appVersion` (still deploying MySQL 8.0.40).

### Clean Up
Remove the release and chart directory:
```bash
helm uninstall bankapp-mysql
rm -rf mysql/
```