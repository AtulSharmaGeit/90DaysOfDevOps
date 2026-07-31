# GitOps Project: End-to-End CI/CD Pipeline with AI-BankApp
>Two days of ArgoCD — setup, self-healing, sync strategies, App of Apps, and RBAC. Today we wire the complete GitOps pipeline end to end. A developer pushes a code change, GitHub Actions builds and pushes the Docker image, updates the Kubernetes manifest in Git, and ArgoCD automatically deploys the new version to EKS. Zero human intervention from `git push` to production.

This is the AI-BankApp's actual production workflow from `.github/workflows/gitops-ci.yml`.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

## Table of Contents
1. [Study the GitOps CI Pipeline](#1-study-the-ai-bankapps-gitops-ci-pipeline)
2. [Set Up the Pipeline on Your Fork](#2-set-up-the-pipeline-on-your-fork)
3. [Trigger the Full Pipeline](#3-trigger-the-full-pipeline)
4. [Test Drift Detection and Recovery](#4-test-drift-detection-and-recovery)
5. [The Complete DevOps Pipeline — End-to-End](#5-reflect-on-the-complete-devops-pipeline)
6. [Complete Teardown](#6-complete-teardown)

## 1. Study the AI-BankApp's GitOps CI Pipeline
### Open the Workflow File
Locate the GitOps CI pipeline definition in the repository.
- Go to the **AI-BankApp** repo (branch: `feat/gitops`)
- Open the file `.github/workflows/gitops-ci.yml`
- This file defines the CI pipeline that connects GitHub Actions to ArgoCD
- This is a production-grade GitOps CI pipeline.

### Understand Workflow Triggers
The pipeline only runs when application code changes, not when manifests change.
```yaml
on:
  push:
    branches: [feat/gitops]
    paths:
      - 'src/**'
      - 'pom.xml'
      - 'Dockerfile'
  workflow_dispatch:
```
- Triggered on **push** to `feat/gitops` branch
-  Runs only when application code changes (`src/`, `pom.xml`, `Dockerfile`) -- not when Kubernetes manifests change
- Also supports manual runs via `workflow_dispatch`
- Prevents infinite loops since manifests are updated by the pipeline itself

The `paths` filter is critical: the workflow only runs when application code changes, not when Kubernetes manifests change. Without this, the manifest commit the pipeline produces would re-trigger the pipeline, which would produce another manifest commit — an infinite loop. `workflow_dispatch` allows manual runs for testing.

### Pipeline Steps
| Step | Action | Notes |
|---|---|---|
| Checkout code | Clones the repository | — |
| Set up JDK 21 | Installs Java 21 with Maven cache | Speeds up subsequent builds |
| Build with Maven | `./mvnw clean package -DskipTests -B` | Fast build without test execution |
| Run tests | `./mvnw test -B` | `continue-on-error: true` — non-blocking |
| Set image tag | `git rev-parse --short HEAD` | Produces a tag like `1c7cb0e` — immutable, traceable to commit |
| Login to DockerHub | Authenticates with repository secrets | Credentials never appear in logs |
| Build and push image | Pushes `:latest` and `:<sha>` tags | SHA tag is what the manifest uses |
| Update K8s manifest | `sed` replaces image tag in `bankapp-deployment.yml` | The GitOps handoff step |
| Commit updated manifest | Commits with `[skip ci]` flag | Prevents pipeline re-trigger |

### The Critical GitOps Step
```yaml
- name: Update Kubernetes deployment manifest
  run: |
    sed -i "s|image: ${{ env.DOCKERHUB_REPO }}:.*|image: ${{ env.DOCKERHUB_REPO }}:${{ steps.tag.outputs.sha_short }}|" k8s/bankapp-deployment.yml

- name: Commit updated manifest
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add k8s/bankapp-deployment.yml
    git diff --staged --quiet || git commit -m "ci: update bankapp image to ${{ steps.tag.outputs.sha_short }} [skip ci]"
    git push
```

The `[skip ci]` tag in the commit message is what prevents the infinite loop — GitHub Actions ignores commits containing this string. The `git diff --staged --quiet || git commit` pattern ensures no empty commits are created if the image tag didn't change.

### End-to-End Flow
```
Developer pushes code to feat/gitops
        │
        ▼
[GitHub Actions]
  Builds Maven project → runs tests
  Builds Docker image, pushes to DockerHub with SHA tag
  Updates image tag in k8s/bankapp-deployment.yml
  Commits with [skip ci] → pushes to Git
        │
        ▼
[ArgoCD — continuous reconciliation]
  Detects new commit within ~3 minutes
  Compares: cluster has old image tag, Git has new image tag
  Performs rolling update — new pods start, old pods terminate
        │
        ▼
[Zero downtime deployment complete]
[Every step is traceable: code commit → image SHA → manifest commit → ArgoCD sync]
```

## 2. Set Up the Pipeline on Your Fork
To run the full pipeline, you need your own fork with GitHub Secrets.

### Fork the Repo (if not done on Day 84):
```
https://github.com/TrainWithShubham/AI-BankApp-DevOps -> Fork
```

### Create a DockerHub Access Token
- Go to https://hub.docker.com/settings/security
- Click **New Access Token** → give it a name (e.g., `gitops-ci`).
- Select **Read/Write** permissions.
- Copy the token — you’ll need it for GitHub Secrets.

### Add GitHub Secrets to our Fork
Go to your fork → **Settings → Secrets and variables → Actions → New repository secret**:
| Secret Name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your DockerHub username |
| `DOCKERHUB_TOKEN` | The access token from the previous step |

These secrets are injected into the workflow at runtime so GitHub Actions can log in to DockerHub securely — they never appear in logs or workflow YAML.

### Update Workflow to Push to our DockerHub Repo
- Open `.github/workflows/gitops-ci.yml` in our fork.
- Find the `env:` section. Replace with our DockerHub repo:
```yaml
env:
  DOCKERHUB_REPO: <your-dockerhub-username>/ai-bankapp-eks
```
This ensures images are pushed to your DockerHub account.

### Point ArgoCD to Your Fork
In your cluster, run:
```bash
argocd app set bankapp --repo https://github.com/<your-username>/AI-BankApp-DevOps.git
```
This tells ArgoCD to watch your fork instead of the original repo.

### Update the Kubernetes Deployment to Pull from our DockerHub
- Edit `k8s/bankapp-deployment.yml` in your fork.
- Update the image line:
  ```yaml
  image: <your-dockerhub-username>/ai-bankapp-eks:latest
  ```
- Commit and push changes to your fork’s `feat/gitops` branch:
  ```bash
  git add k8s/bankapp-deployment.yml .github/workflows/gitops-ci.yml
  git commit -m "chore: configure pipeline for personal DockerHub and fork"
  git push origin feat/gitops
  ```

## 3. Trigger the Full Pipeline
### Make a Visible Code Change in the Application
Edit a source file so the resulting change is verifiable in the running app. For example, update the page title in `src/main/resources/templates/fragments/layout.html`:
```html
<!-- Find the title or footer and add your touch -->
<title>AI BankApp - Built by YourName</title>
```

Commit and push:
```bash
git add src/
git commit -m "feat: customize app title"
git push origin feat/gitops
```
This pushes your change to the `feat/gitops` branch, which triggers the GitHub Actions workflow.

### Watch GitHub Actions Run
- Go to your fork → **Actions tab**.
- Look for the workflow: **GitOps CI - Build & Push to DockerHub**.
- Watch each stage complete in sequence:
  - Checkout code
  - Set up JDK 21
  - Build with Maven
  - Run tests
  - Build & push Docker image
  - Update manifest with new image tag
  - Commit manifest with `[skip ci]`

After the workflow finishes, check the last commit on `feat/gitops` — you should see a commit from `github-actions[bot]`:
```
ci: update bankapp image to <sha> [skip ci]
```
Open `k8s/bankapp-deployment.yml` and confirm the image line now contains the new SHA tag.

### Watch ArgoCD Sync
```bash
argocd app get bankapp --refresh
argocd app wait bankapp
```
Or open the ArgoCD UI:
- You’ll see a new sync event with the updated revision.
- ArgoCD will detect the new commit, compare the image tag in Git against the running pods, and trigger a rolling update.

### Observe Pods Updating
```bash
kubectl get pods -n bankapp -w
```
You’ll see:
- Old pods terminating gracefully
- New pods starting with the updated image

This is a **zero downtime rolling update** driven entirely by a Git commit.

### Verify the Change Live
Forward the service:
```bash
kubectl port-forward svc/bankapp-service -n bankapp 8080:8080
```
- Open `http://localhost:8080` and confirm your title change is visible.

> **You just completed a full GitOps cycle:** code change → CI builds and pushes image → updates manifest in Git → ArgoCD deploys to production. Zero manual intervention from `git push` to live application.

## 4. Test Drift Detection and Recovery
GitOps guarantees the cluster always matches Git. Test what happens when someone makes unauthorised changes directly to the cluster.

### Scenario 1: Someone Scales Down the App Directly
Scale down manually:
```bash
kubectl scale deployment bankapp -n bankapp --replicas=1
```
Check ArgoCD status:
```bash
argocd app get bankapp
```
- Status should show `OutOfSync`. With `selfHeal: true`, ArgoCD will correct it within 3 minutes.

Monitor pods:
```bash
kubectl get pods -n bankapp -w
```
- Within ~3 minutes, ArgoCD restores replicas back to **4** (or whatever manifest specifies).

### Scenario 2: Someone Updates the Image Tag Directly
Tamper with image:
```bash
kubectl set image deployment/bankapp bankapp=nginx:latest -n bankapp
```
ArgoCD detects that the running image differs from the Git manifest and reverts the Deployment to the correct image on the next reconciliation cycle. The BankApp pods restart with the correct image.

### Scenario 3: Someone Deletes a Critical Resource
Delete service:
```bash
kubectl delete service bankapp-service -n bankapp
```
ArgoCD recreates the Service from Git. The application is briefly unreachable, but the resource is restored automatically.

### View All Drift Events
CLI:
```bash
argocd app history bankapp
```
In the ArgoCD UI → **bankapp** → **Events tab**: every self-heal action is logged with the before and after state, providing a full audit trail of every drift and correction.

### What would happen if `selfHeal` was disabled?
**Ans.** ArgoCD would detect drift and mark the application `OutOfSync`, but take no corrective action. Unauthorised changes would persist until a human manually triggered a sync — undermining the GitOps discipline.

## 5. Reflect on the Complete DevOps Pipeline
Step back and look at everything you have built across the entire 90-day challenge that connects to this GitOps pipeline:
```
[Developer writes code]
        │
        ▼
[Git push to GitHub]          ← Days 22–28: Git & GitHub
        │
        ▼
[GitHub Actions CI]           ← Days 40–49: GitHub Actions
  ├── Build with Maven
  ├── Run tests
  ├── Build Docker image       ← Days 29–37: Docker
  ├── Push to DockerHub
  ├── Update K8s manifest
  └── Commit back to Git
        │
        ▼
[ArgoCD detects change]       ← Days 84–86: GitOps & ArgoCD
        │
        ▼
[ArgoCD syncs to EKS]         ← Days 81–83: Amazon EKS
  ├── Rolling update (zero downtime)
  ├── Health checks pass
  └── HPA scales as needed    ← Days 78–80: Helm
        │
        ▼
[Prometheus scrapes metrics]  ← Days 73–77: Observability
  ├── Grafana dashboards
  └── Alerts if degraded
        │
        ▼
[Application live — fully automated, fully observable]
```
Every block in this series connects to the next. The GitOps pipeline is not a standalone feature — it is the integration point where infrastructure, containerisation, CI/CD, orchestration, and observability all converge.

## 6. Complete Teardown
>This is the end of the EKS and ArgoCD block. Delete everything to stop AWS charges.

### Delete ArgoCD Applications
Run these commands in sequence:
```bash
argocd app delete bankapp --cascade -y
argocd app delete monitoring --cascade -y 2>/dev/null
argocd app delete envoy-gateway --cascade -y 2>/dev/null
argocd app delete root-app --cascade -y 2>/dev/null
```
- The `--cascade` flag ensures **all Kubernetes resources managed by each app are deleted**.
- Without it, ArgoCD would only delete the Application object, leaving pods, services, and load balancers running (and billing).

### Wait for Cleanup
Check that namespaces are empty:
```bash
kubectl get all -n bankapp 2>/dev/null
kubectl get all -n monitoring 2>/dev/null
```
Both should return no resources. If anything lingers — particularly LoadBalancer Services or PVCs — delete them manually before running `terraform destroy`:
 
```bash
# Check for lingering LoadBalancers (would block VPC teardown)
kubectl get svc -A | grep LoadBalancer
 
# Check for lingering PVCs (would leave orphaned EBS volumes)
kubectl get pvc -A
```

### Destroy EKS Cluster with Terraform
Navigate to your Terraform directory:
```bash
cd AI-BankApp-DevOps/terraform
terraform destroy
```
- Confirm deletion (type `yes`). This takes 10-15 minutes.
- Deletes: EKS cluster, node groups, VPC, subnets, NAT Gateway, Elastic IPs, IAM roles, ArgoCD LoadBalancer, and all associated resources.

### Post-Teardown Verification in AWS Console
| Service | What to Confirm |
|---|---|
| EKS | No clusters listed |
| EC2 → Instances | No running instances |
| EC2 → Load Balancers | No load balancers |
| EC2 → Volumes | No EBS volumes with `kubernetes.io` tags |
| VPC | `bankapp-eks` VPC deleted |
| IAM | No `bankapp-eks-*` roles remaining |
 
> **Billing:** Charges stop within approximately one hour of teardown. Expected cost for the full 3-day lab: **$15–25**, depending on runtime. Verify in the AWS Billing Dashboard that no services are still accumulating charges.
 
### Three-Day ArgoCD Journey
| Day | What You Built |
|---|---|
| 84 | ArgoCD setup, first GitOps deploy, self-healing tests |
| 85 | Sync waves, rollbacks, App of Apps pattern, notifications, RBAC |
| 86 | Full CI/CD pipeline end-to-end, drift detection, complete teardown |