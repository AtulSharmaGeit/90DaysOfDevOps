# Introduction to GitOps and ArgoCD
>We have deployed the AI-BankApp on EKS using `kubectl apply`. That works, but who ran the command? From which machine? Was the YAML they applied the same as what is in Git? If someone manually edits a Deployment in the cluster, how do we know?

**GitOps** solves all of this. Git becomes the single source of truth. A tool watches your Git repository and continuously ensures the cluster matches what is committed. That tool is **ArgoCD**.

The AI-BankApp project (https://github.com/TrainWithShubham/AI-BankApp-DevOps, branch `feat/gitops`) already has ArgoCD installed via Terraform and an Application manifest ready to go. Today we understand GitOps principles, explore ArgoCD, and deploy the AI-BankApp through ArgoCD for the first time.

## Table of Contents
1. [Understanding GitOps](#1-understand-gitops)
2. [Access ArgoCD on Your EKS Cluster](#2-access-argocd-on-your-eks-cluster)
3. [Study the ArgoCD Application Manifest](#3-study-the-ai-bankapps-argocd-application-manifest)
4. [Deploy the AI-BankApp via ArgoCD](#4-deploy-the-ai-bankapp-via-argocd)
5. [Explore ArgoCD's Live View](#5-explore-argocds-live-view)
6. [Test Self-Healing](#6-test-self-healing)

## 1. Understand GitOps
### What is GitOps?
**Ans.** GitOps is a deployment methodology where Git is the single source of truth for both application and infrastructure state. An operator (ArgoCD) watches Git and continuously reconciles the live cluster to match what is committed. If anything in the cluster diverges from Git — whether from a manual `kubectl` command, a failed partial update, or any other cause — the operator detects and corrects it automatically.
 
All changes flow through Git: pull requests, code review, and merge. The cluster is never a destination that humans reach into directly.
 
### GitOps vs. Traditional CI/CD
| Dimension | Traditional CI/CD | GitOps |
|---|---|---|
| **Deployment trigger** | CI pipeline runs `kubectl apply` | Git commit triggers ArgoCD sync |
| **Source of truth** | Pipeline scripts and configuration | Git repository |
| **Drift detection** | None — cluster state is opaque | Continuous reconciliation loop |
| **Rollback** | Re-run pipeline or manual kubectl | `git revert` → ArgoCD syncs the old state |
| **Audit trail** | Pipeline logs (often ephemeral) | Git commit history (permanent, attributable) |
| **Cluster access** | CI server holds broad cluster credentials | Only ArgoCD has cluster access — developers push to Git |
| **Security posture** | Large attack surface (CI server = cluster admin) | Developers never touch the cluster directly |
 
The security dimension is particularly important: in a GitOps model, a compromised CI server cannot directly alter cluster state because it has no cluster credentials — only the ability to push to Git, which ArgoCD then validates and applies.
 
### The AI-BankApp GitOps Flow
```
Developer pushes code to feat/gitops
        │
        ▼
[GitHub Actions CI]
  - Build Maven project and run tests
  - Build Docker image, push to DockerHub (tagged with git SHA)
  - Update image tag in k8s/bankapp-deployment.yml
  - Commit the change back to Git
        │
        ▼
[ArgoCD — continuous reconciliation loop]
  - Detects the new commit on feat/gitops
  - Compares k8s/ manifests with live cluster state
  - Syncs the change → rolling update of BankApp pods
  - Pods restart with the new image
        │
        ▼
[Zero human intervention after git push]
```
 
### Four GitOps Principles (OpenGitOps)
| Principle | What It Means |
|---|---|
| **Declarative** | Desired state is expressed as Kubernetes YAML - not imperative scripts |
| **Versioned and immutable** | Desired state is stored in Git - every change is versioned and auditable |
| **Pulled automatically** | ArgoCD pulls from Git rather than having CI push to the cluster |
| **Continuously reconciled** | ArgoCD compares desired vs. actual state on a continuous loop and corrects drift |
 

## 2. Access ArgoCD on Your EKS Cluster
ArgoCD was installed by Terraform on Day 81 (via `terraform/argocd.tf`).

### Verify ArgoCD is Running
```bash
kubectl get pods -n argocd
```
All of the following pods should be in `Running` state:
| Pod | Role |
|---|---|
| `argocd-server` | API server and UI |
| `argocd-repo-server` | Clones and renders Git repositories |
| `argocd-application-controller` | Reconciliation loop — compares Git vs. cluster |
| `argocd-applicationset-controller` | Manages ApplicationSet resources |
| `argocd-redis` | Caching layer for the controller |
| `argocd-dex-server` | SSO / identity provider integration |

### Get the ArgoCD Admin Password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
Save this password — you will need it for both the UI and CLI login.

### Access the ArgoCD UI:
Option A: LoadBalancer (preferred if Terraform exposed it)
```bash
export ARGOCD_URL=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ArgoCD URL: http://$ARGOCD_URL"
```
- Open the printed URL in your browser.

Option B: Port-forward (fallback)
```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

Open `https://localhost:8443` (accept the self-signed certificate).<br>
Log in with:
- Username: `admin`
- Password: the value from the command above

### Install the ArgoCD CLI:
```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Verify
argocd version --client
```

### Log in via CLI:
```bash
# If using LoadBalancer:
argocd login $ARGOCD_URL --username admin --password <your-password> --insecure

# Or if using port-forward:
argocd login localhost:8443 --username admin --password <your-password> --insecure
```

### Explore the ArgoCD UI:
- **Applications** -- shows all managed applications (empty for now)
- **Settings > Repositories** -- Git repos ArgoCD can access
- **Settings > Clusters** -- Kubernetes clusters ArgoCD manages (your EKS cluster is the default `in-cluster`)

## 3. Study the AI-BankApp's ArgoCD Application Manifest
The AI-BankApp repository (`feat/gitops` branch) includes `argocd/application.yml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Break down every field:
| Field | Value | Purpose |
|-------|-------|---------|
| `source.repoURL` | The AI-BankApp GitHub repo | Where ArgoCD fetches manifests from |
| `source.targetRevision` | `feat/gitops` | Which Git branch to watch |
| `source.path` | `k8s` | The directory containing Kubernetes manifests |
| `destination.server` | `kubernetes.default.svc` | Deploy to the local cluster (in-cluster) |
| `destination.namespace` | `bankapp` | Target namespace for resources |
| `syncPolicy.automated` | enabled | ArgoCD syncs automatically on Git changes |
| `prune: true` | enabled | Delete resources removed from Git |
| `selfHeal: true` | enabled | Revert manual changes made directly to the cluster |
| `CreateNamespace=true` | enabled | Create the `bankapp` namespace if it does not exist |
| `ServerSideApply=true` | enabled | Use server-side apply for better conflict handling |

### Understand the Flow
- ArgoCD **watches your repo** (`feat/gitops` branch).
- It **compares manifests** in `k8s/` with the live cluster.
- It **syncs automatically** (thanks to `automated` policy).
- If you delete or edit something manually, **selfHeal** restores Git’s version.
- If you remove a resource from Git, **prune** deletes it from the cluster.

>Git is always authoritative. The cluster is always a reflection of Git.

## 4. Deploy the AI-BankApp via ArgoCD
### Ensure a Clean Slate
First, make sure the BankApp is NOT already deployed:
```bash
kubectl delete namespace bankapp 2>/dev/null
```
- Expected: Either “namespace deleted” or no output (if it didn’t exist).

### Fork the AI-BankApp Repo
We need your own copy to push changes later:
1. Go to https://github.com/TrainWithShubham/AI-BankApp-DevOps
2. Click "Fork" and create your fork
3. Note your fork URL: `https://github.com/<your-username>/AI-BankApp-DevOps.git`

### Create ArgoCD Application
Apply the manifest, replacing `repoURL` with your fork:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF
```
-  Expected: `application.argoproj.io/bankapp created`.

### Watch ArgoCD Deploy the App
UI method:
- In the ArgoCD UI, click on the `bankapp` application
- You will see a visual tree of all Kubernetes resources being created
- Each resource shows its sync and health status (green = healthy, yellow = progressing, red = degraded)

Or CLI method:
```bash
# Full status overview
argocd app get bankapp

# Block until the application reaches Healthy+Synced
argocd app wait bankapp
```

### Monitor Pods Coming Up
```bash
kubectl get pods -n bankapp -w
```

The deployment order is automatic -- ArgoCD applies all manifests from the `k8s/` directory. MySQL and Ollama start first, then the BankApp's init containers wait for dependencies.

### Verify Health
After everything is healthy (5-10 minutes):
```bash
argocd app get bankapp
```
Status should show:
- `Health: Healthy`
- `Sync: Synced`.

## 5. Explore ArgoCD's Live View
### Open the BankApp Application in ArgoCD UI
- In the ArgoCD dashboard, click on `bankapp`.
- You’ll see a **resource tree** showing all Kubernetes objects deployed:
```
bankapp (Application)
├── Namespace: bankapp
├── StorageClass: gp3
├── PVC: mysql-pvc (Bound)
├── PVC: ollama-pvc (Bound)
├── ConfigMap: bankapp-config
├── Secret: bankapp-secret
├── Deployment: mysql → ReplicaSet → Pod
├── Deployment: ollama → ReplicaSet → Pod
├── Deployment: bankapp → ReplicaSet → Pod (×4)
├── Service: mysql-service
├── Service: ollama-service
├── Service: bankapp-service
└── HPA: bankapp-hpa
```
- This tree is a **visual map** of your application — every resource is linked back to Git.

### Inspect Resources
Click on any resource (e.g., `bankapp-config` or `bankapp-service`) to view:
- **Pod logs** (live streaming)
- **Events** (scheduling, scaling, errors)
- **YAML manifest** (as applied to the cluster)
- **Diff view** (shows drift between Git and cluster state)

This is powerful for debugging and verifying GitOps enforcement.

### Explore App Details Tab
The **App Details** tab shows:
- **Source repo & path** (your fork, `feat/gitops`, `k8s/`)
- **Last sync time & commit SHA** (traceable to Git history)
- **Sync status & health status** (Healthy / Synced)
- **History of all syncs** (every commit applied to cluster)

### Check Sync History via CLI
```bash
argocd app history bankapp
```
Expected output: A list of revisions, each with:
- Commit SHA
- Sync time
- Status (Synced/Healthy)

This outputs a timeline of every deployment: commit SHA, timestamp, and sync status. This is your deployment audit trail.

## 6. Test Self-Healing
`selfHeal: true` means ArgoCD reverts any manual cluster change back to the Git-defined state within its reconciliation interval (default: 3 minutes).

### Test 1 -- Manually Scale the BankApp:
```bash
kubectl scale deployment bankapp -n bankapp --replicas=1
```
Watch what happens:
```bash
kubectl get pods -n bankapp -w
```
Within 3-5 minutes, ArgoCD detects the drift and scales it back to the value defined in Git (4 replicas, or whatever the HPA decides). Check the ArgoCD UI → you’ll see a sync event triggered automatically.

### Test 2 -- Manually Delete a ConfigMap:
```bash
kubectl delete configmap bankapp-config -n bankapp
kubectl get configmap -n bankapp
```
ArgoCD detects the absence of a Git-defined resource and recreates the ConfigMap from the repository. UI shows: `OutOfSync → Syncing → Synced`.

### Test 3 -- Manually Change an Environment Variable:
```bash
kubectl edit configmap bankapp-config -n bankapp
# Change MYSQL_DATABASE to something wrong
```
ArgoCD will overwrite your change with the value from Git. UI Diff view shows your manual edit vs Git state.

### Self-Healing Summary
| Test | Manual Change | ArgoCD Response | Time to Revert |
|---|---|---|---|
| Scale Deployment | `replicas: 1` | Scales back to Git-defined count | ~3 minutes |
| Delete ConfigMap | Resource missing | Recreates from Git | ~3 minutes |
| Edit ConfigMap | Wrong env var value | Overwrites with Git version | ~3 minutes |
 
**This is the core GitOps guarantee:** the cluster always reflects Git. Manual changes do not survive the next reconciliation cycle. Every change must be made through a Git commit - giving you code review, rollback via `git revert`, and a permanent, attributable audit trail for every cluster state change.


