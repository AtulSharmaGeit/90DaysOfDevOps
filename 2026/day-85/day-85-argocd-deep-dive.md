# ArgoCD Deep Dive: Sync Strategies, Rollbacks, and Multi-App Management
>Yesterday we deployed the AI-BankApp via ArgoCD and tested self-healing. Today we go deeper -- understanding sync waves, resource hooks, manual vs automated sync strategies, rollbacks, the App of Apps pattern for managing multiple applications, and ArgoCD notifications. These are the patterns teams use when managing production clusters with dozens of applications.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

## Table of Contents
1. [Sync Strategies — Manual vs. Automated](#1-understand-sync-strategies)
2. [Sync Waves and Resource Ordering](#2-sync-waves-and-resource-ordering)
3. [ArgoCD Rollbacks](#3-argocd-rollbacks)
4. [App of Apps Pattern](#4-app-of-apps-pattern)
5. [ArgoCD Notifications](#5-argocd-notifications)
6. [Projects and RBAC](#6-argocd-projects-and-rbac)

## 1. Understand Sync Strategies
### Check Current Sync Policy
```bash
argocd app get bankapp
```
Look for `Sync Policy`. The AI-BankApp currently uses automated sync:
```yaml
syncPolicy:
automated:
    prune: true
    selfHeal: true
```

### ArgoCD offers multiple ways to sync:

#### 1. Automated sync (what the AI-BankApp uses):
```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources removed from Git
    selfHeal: true   # Revert manual cluster changes
```
- Every Git change syncs automatically within 3 minutes
- No human approval needed
- Good for dev/staging environments

#### 2. Manual sync (for production):
```yaml
syncPolicy: {}   # No automated section
```
- ArgoCD detects drift but does NOT auto-correct
- A human must click "Sync" or run `argocd app sync`
- Good for production where you want a review gate

### Switch to Manual Sync
```bash
argocd app set bankapp --sync-policy none
```
- This removes the automated section.
- Verify again:
    ```bash
    argocd app get bankapp
    ```
    Expected: `Sync Policy: Manual`.

### Test Manual Sync
Edit your fork’s `k8s/configmap.yml`:
```yaml
data:
  APP_NAME: "BankApp-ManualMode"
  NEW_KEY: "manual-sync-test"
```
Commit and push:
```bash
git add k8s/configmap.yml
git commit -m "Test manual sync change"
git push
```
Wait ~3 minutes (ArgoCD’s default resync interval). Then:
```bash
argocd app get bankapp
```
Status will show `OutOfSync` but ArgoCD will NOT apply the change. You can see exactly what differs:
```bash
argocd app diff bankapp
```
We’ll see the new key and APP_NAME change listed.

### Preview Before Syncing:
```bash
# Dry run -- show what would change but doesn’t apply
argocd app sync bankapp --dry-run

# Sync for real -- applies the Git changes to the cluster
argocd app sync bankapp
```
In the UI, clicking **Sync** shows a preview dialog listing every resource that will be created, modified, or deleted — equivalent to `--dry-run` before committing.

Verify:
```bash
kubectl get configmap -n bankapp
```
We should see the updated APP_NAME and NEW_KEY.

### Switch Back to Automated
When done testing, restore automated sync:
```bash
argocd app set bankapp --sync-policy automated --self-heal --auto-prune
```
Verify:
```bash
argocd app get bankapp
```
Expected: Automated sync with prune + selfHeal enabled.

## 2. Sync Waves and Resource Ordering
The AI-BankApp has dependencies: MySQL must be running before the BankApp starts. ArgoCD handles this with **sync waves** -- annotations that control the order of resource creation.

### Sync Waves - Understand the Concept
- ArgoCD syncs resources in **waves**.
- Negative numbers run first, then zero, then positive.
- Resources in the same wave sync **in parallel**.
- ArgoCD waits for each wave to be **healthy** before moving to the next.

### Edit Manifests with Sync-Wave Annotations
Add sync wave annotations to the AI-BankApp manifests in your fork:

#### Wave -2 (Infrastructure)
```yaml
#Edit k8s/namespace.yml:
apiVersion: v1
kind: Namespace
metadata:
  name: bankapp
  annotations:
    argocd.argoproj.io/sync-wave: "-2"

#Edit k8s/pv.yml (StorageClass):
metadata:
  name: gp3
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

#### Wave -1 (Configuration)
```yaml
#Edit k8s/pvc.yml (both PVCs):
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

#Edit k8s/configmap.yml and k8s/secrets.yml:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

#### Wave 0 (Databases & Networking)
```yaml
#Edit k8s/mysql-deployment.yml:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"

#Edit k8s/ollama-deployment.yml:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"

#Edit k8s/service.yml (all three services):
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

#### Wave 1 (Application)
```yaml
#Edit k8s/bankapp-deployment.yml:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

#### Wave 2 (Scaling)
```yaml
#Edit k8s/hpa.yml:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

### The sync order becomes:
```
Wave -2: Namespace, StorageClass          (infrastructure)
Wave -1: PVCs, ConfigMap, Secret          (configuration)
Wave  0: MySQL, Ollama, Services          (databases and networking)
Wave  1: BankApp Deployment               (application)
Wave  2: HPA                              (scaling)
```

ArgoCD processes each wave in order. Resources in the same wave sync in parallel. ArgoCD waits for each wave to be healthy before moving to the next.

### Commit & Push Changes
```bash
git add k8s/*.yml
git commit -m "Add ArgoCD sync-wave annotations"
git push
```

### Trigger ArgoCD Sync
ArgoCD will detect changes and re-sync automatically (within ~3 minutes).<br>
We can also force sync:
```bash
argocd app sync bankapp
```

### Verify in ArgoCD UI
- Open the ArgoCD dashboard.
- Watch the deployment order:
    - Wave -2: Namespace + StorageClass
    - Wave -1: PVCs, ConfigMap, Secrets
    - Wave 0: MySQL, Ollama, Services
    - Wave 1: BankApp Deployment
    - Wave 2: HPA

Resources in the same wave will appear together, but ArgoCD won’t move to the next wave until the current one is healthy.

### CLI Verification
```bash
argocd app get bankapp
```
Look for `Sync Status: Synced` and check resource health.<br>
You can also confirm order by watching pods:
```bash
kubectl get pods -n bankapp -w
```
Expected startup sequence:
- Namespace + StorageClass
- PVCs, ConfigMap, Secrets
- MySQL + Ollama + Services
- BankApp Deployment
- HPA attaches scaling rules

## 3. ArgoCD Rollbacks
ArgoCD tracks every sync as a revision. You can rollback to any previous state.

### Check Sync History:
```bash
argocd app history bankapp
```

Output:
```
ID  DATE                 REVISION
1   2026-04-10 10:00:00  abc1234
2   2026-04-10 10:15:00  def5678   (sync wave annotations)
```
- Each **ID** is a sync revision ArgoCD has applied.
- Each **REVISION** corresponds to a Git commit hash.

### Rollback to a Previous Revision:
Via CLI:
```bash
argocd app rollback bankapp 1
```
- This reverts the cluster state to revision 1.
- It does **not** change Git — only the cluster.

Via UI:
- Open ArgoCD dashboard.
- Click **bankapp** → **History**.
- Select a revision → click **Rollback**.
- The cluster reverts to that revision.

### Verify Rollback
```bash
argocd app get bankapp
```

The status will show `OutOfSync` because the cluster now matches an older Git commit, not the latest Git state.

Check resources:
```bash
kubectl get pods -n bankapp
```
- We’ll see pods/configs from the older revision.

### GitOps-Correct Rollback
Rollback in ArgoCD is a temporary fix. It does not change Git. The proper GitOps rollback is:
```bash
# In your fork
git revert HEAD
git push
```

This creates a new commit that undoes the last change. ArgoCD syncs the revert and the cluster is updated. The Git history shows the full audit trail: deploy, then revert.

### What is the difference between ArgoCD rollback and `git revert`? Which is the GitOps-correct approach?

- **ArgoCD rollback** → Changes cluster state only. Temporary fix.

- **Git revert** → Creates a new commit, keeps Git as the single source of truth. Permanent, auditable, GitOps-correct approach.

## 4. App of Apps Pattern
In production, you do not manage one application -- you manage dozens. The **App of Apps** pattern uses one parent ArgoCD Application that creates child Applications.

### Create Directory for Child Applications
In our forked repo:
```bash
mkdir -p argocd-apps/
```
This directory will hold YAML definitions for each child application.

### Define the BankApp Application
`argocd-apps/bankapp.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
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
```
This is the same BankApp we’ve been working with, now defined as a child app.

### Define Monitoring (Prometheus + Grafana)
`argocd-apps/monitoring.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: "65.*"
    helm:
      values: |
        grafana:
          adminPassword: admin123
        prometheus:
          prometheusSpec:
            retention: 3d
            resources:
              requests:
                memory: 256Mi
                cpu: 100m
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```
This installs monitoring stack via Helm.

### Define Envoy Gateway
`argocd-apps/envoy-gateway.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: docker.io/envoyproxy
    chart: gateway-helm
    targetRevision: "v1.4.*"
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Define the Parent Application - Manages All Child Apps:
`argocd-apps/root-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: argocd-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
This parent app points to the `argocd-apps/` directory, which contains all child apps.

### Commit & Push
```bash
git add argocd-apps/*.yaml
git commit -m "Implement App of Apps pattern"
git push
```

### Apply the Root App
```bash
kubectl apply -f argocd-apps/root-app.yaml
```

ArgoCD will:
1. Read the `argocd-apps/` directory from Git
2. Find `bankapp.yaml`, `monitoring.yaml`, and `envoy-gateway.yaml`
3. Create three child Applications
4. Each child Application syncs independently

### Verify in ArgoCD UI
- Open the ArgoCD dashboard.
- We should now see **4 applications**:
    - `root-app` (parent)
    - `bankapp`
    - `monitoring`
    - `envoy-gateway`

Each child app syncs independently. Adding a new app to the cluster is as simple as adding a new YAML file to the `argocd-apps/` directory.

### CLI Verification
```bash
argocd app list
```
Expected output: All four apps listed with their sync status.

## 5. ArgoCD Notifications
Get notified when deployments succeed, fail, or drift.

### Verify Notifications Controller
```bash
# Check if notifications controller is running
kubectl get pods -n argocd -l app.kubernetes.io/component=notifications-controller
```
- We should see a pod like `argocd-notifications-controller-xxxxx` in **Running** state.
- If not present, ensure your ArgoCD installation includes notifications (modern versions do by default).

**Configure a Slack or webhook notification** (using a generic webhook example):

### Create Notification Config
Apply a ConfigMap with triggers and templates:
```bash
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]
  template.app-sync-succeeded: |
    message: "Application {{.app.metadata.name}} sync succeeded. Revision: {{.app.status.sync.revision}}"
  template.app-sync-failed: |
    message: "Application {{.app.metadata.name}} sync FAILED! Check ArgoCD for details."
  template.app-health-degraded: |
    message: "Application {{.app.metadata.name}} health is DEGRADED. Investigate immediately."
EOF
```
- **Triggers**: Define when notifications fire (success, failure, degraded).
- **Templates**: Define the message format.

### Subscribe an Application to Notifications
Annotate our app to subscribe:
```bash
kubectl annotate application bankapp -n argocd \
  notifications.argoproj.io/subscribe.on-sync-succeeded.webhook="" \
  notifications.argoproj.io/subscribe.on-sync-failed.webhook="" \
  notifications.argoproj.io/subscribe.on-health-degraded.webhook=""
```
- This tells ArgoCD to send notifications for `bankapp` events.
- Replace `webhook` with `slack` if you configure Slack integration.

### Slack Integration (Optional)
To send alerts to Slack:
- Add a Slack service to the ConfigMap with your webhook URL:
    ```yaml
    data:
    service.slack: |
        token: <your-slack-webhook-token>
    ```
- Then subscribe with:
    ```bash
    kubectl annotate application bankapp -n argocd \
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack="" \
    notifications.argoproj.io/subscribe.on-sync-failed.slack="" \
    notifications.argoproj.io/subscribe.on-health-degraded.slack=""
    ```

### Verify Notifications
Check notification history:
```bash
kubectl get applications bankapp -n argocd -o jsonpath='{.status.operationState.message}'
```
We’ll see messages like:
- `"Application bankapp sync succeeded. Revision: def5678"`
- `"Application bankapp sync FAILED! Check ArgoCD for details."`

## 6. ArgoCD Projects and RBAC
In production, you do not give every team access to every application. ArgoCD **Projects** provide multi-tenancy.

### Create a Project for BankApp Team
Run:
```bash
argocd proj create bankapp-team \
  --description "AI-BankApp team project" \
  --src "https://github.com/<your-username>/AI-BankApp-DevOps.git" \
  --dest "https://kubernetes.default.svc,bankapp" \
  --dest "https://kubernetes.default.svc,monitoring"
```
This project:
- Can only source from the AI-BankApp repo
- Can only deploy to the `bankapp` and `monitoring` namespaces
- Cannot deploy to `kube-system`, `argocd`, or other namespaces

### Move the Bankapp Application to this Project:
```bash
argocd app set bankapp --project bankapp-team
```
- Now the BankApp is governed by the project’s restrictions.

### Verify Restrictions Work
Try adding a forbidden destination:
```bash
# This should fail -- cert-manager namespace is not allowed
argocd proj add-destination bankapp-team https://kubernetes.default.svc kube-system 2>&1 || echo "Restricted!"
```
Expected: **Restricted!** → confirms the project prevents deployments outside allowed namespaces.

### Configure RBAC Policies
Edit the `argocd-rbac-cm` ConfigMap and add:
```yaml
policy.csv: |
  p, role:bankapp-dev, applications, get, bankapp-team/*, allow
  p, role:bankapp-dev, applications, sync, bankapp-team/*, allow
  p, role:bankapp-dev, applications, rollback, bankapp-team/*, deny
  g, bankapp-developers, role:bankapp-dev
```
This gives the `bankapp-developers` group permission to view and sync but NOT rollback. Rollback requires a senior team member.

Apply changes:
```bash
kubectl apply -n argocd -f argocd-rbac-cm.yaml
```

### Verify RBAC
- Log in as a developer user.
- Try syncing → allowed.
- Try rollback → denied.
- This enforces **least privilege**.

### How do Projects and RBAC prevent one team from accidentally affecting another team's applications?
- **Projects** restrict which repos and namespaces a team can deploy to.
- **RBAC** enforces fine-grained permissions (e.g., devs can sync but not rollback).
- Together, they ensure one team cannot accidentally deploy into another team’s namespace or override critical apps.

