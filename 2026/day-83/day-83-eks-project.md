# EKS Project: Production Deployment of AI-BankApp
Three days of EKS - Cluster Provisioning with Terraform, Gateway API networking, EBS storage, and TLS. Today we put it all together and deploy the AI-BankApp as a production-grade application on EKS. 
>Full stack: Spring Boot app with MySQL and Ollama AI, persistent storage, autoscaling, monitoring, and the complete end-to-end validation.

This is the kind of deployment we would do on the job.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

## Table of Contents
1. [Deploy the Complete AI-BankApp Stack](#1-deploy-the-complete-ai-bankapp-stack)
2. [Set Up Gateway API and Access the App](#2-set-up-gateway-api-and-access-the-app)
3. [Deploy the Monitoring Stack](#3-deploy-the-monitoring-stack)
4. [End-to-End Validation](#4-end-to-end-validation-checklist)
5. [Three-Day EKS Journey — Reflection](#5-reflect-on-the-full-eks-journey)
6. [Complete Teardown](#6-complete-teardown)

## 1. Deploy the Complete AI-BankApp Stack
### Verify EKS Cluster is Running:
```bash
kubectl get nodes
```
- You should see 2–3 nodes in `Ready` state
- If you destroyed the cluster, re-provision with Terraform:
    ```bash
    cd AI-BankApp-DevOps/terraform
    terraform apply
    aws eks update-kubeconfig --name bankapp-eks --region us-west-2
    ```
    - Terraform provisions VPC, subnets, node groups, IAM roles
    - `update-kubeconfig` ensures kubectl points to the new cluster

>Apply resources in the order their dependencies require - storage before workloads, configs before pods that reference them.

### Deploy Namespace and Storage
Create the namespace and persistent volumes for MySQL and Ollama.
```bash
cd AI-BankApp-DevOps
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/pv.yml
kubectl apply -f k8s/pvc.yml

# Verify PVCs are bound before proceeding
kubectl get pvc -n bankapp
# Expected: STATUS = Bound for both mysql-pvc and ollama-pvc
```

### Apply Configurations
Load ConfigMaps and Secrets for application settings.
```bash
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml
```
- ConfigMap stores app configs
- Secrets store sensitive values like MySQL password

### Deploy Database and AI Service
Start MySQL and Ollama pods with services.
```bash
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ollama-deployment.yml
```
- MySQL pod should start in ~30s
- Ollama pod pulls TinyLlama model (2–5 min)

### Wait for Dependencies
Ensure MySQL and Ollama are ready before deploying BankApp.
```bash
echo "Waiting for MySQL..."
kubectl wait --for=condition=ready pod -l app=mysql -n bankapp --timeout=120s

echo "Waiting for Ollama (this takes 2-5 minutes for model pull)..."
kubectl wait --for=condition=ready pod -l app=ollama -n bankapp --timeout=600s
```
Do not proceed until both conditions report `condition met`. The BankApp's init containers poll MySQL and Ollama before the main container starts — if either is not ready, the BankApp pods will remain in `Init` state indefinitely.

### Deploy BankApp and HPA
Launch the main application and autoscaler.
```bash
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml
```
- BankApp pods scale between 2–4 replicas
- HPA monitors CPU utilization

### Wait for BankApp Readiness
Confirm BankApp pods are running.
```bash
echo "Waiting for BankApp..."
kubectl wait --for=condition=ready pod -l app=bankapp -n bankapp --timeout=300s
```
- Expect pods to reach `Running`
- Dependencies must be healthy first

### Verify Deployment
Check all resources in the namespace
```bash
kubectl get all -n bankapp
kubectl get pvc -n bankapp
```
Expected state:
| Resource | Count | State |
|---|---|---|
| MySQL pod | 1 | Running |
| Ollama pod | 1 | Running |
| BankApp pods | 2–4 | Running (managed by HPA) |
| ClusterIP Services | 3 | — |
| mysql-pvc | 1 | Bound (5 Gi, gp3) |
| ollama-pvc | 1 | Bound (10 Gi, gp3) |

### By the end:
- Your **EKS cluster** is live.
- **MySQL** and **Ollama** are running with persistent storage.
- **BankApp** pods are autoscaling.
- All services are exposed internally via ClusterIP.

## 2. Set Up Gateway API and Access the App
### Install Envoy Gateway (if not done on Day 82):
```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 \
  -n envoy-gateway-system --create-namespace \
  --wait 2>/dev/null || echo "Already installed"
```
- Installs Envoy Gateway into its own namespace.
- **Expected output**: Either a successful Helm release or the message “Already installed” if you did it on Day 82.
- Verify:
    ```bash
    kubectl get pods -n envoy-gateway-system
    ```
    - We should see Envoy pods running.

### Apply Gateway Configuration:
```bash
kubectl apply -f k8s/gateway.yml
```
- Creates the Gateway resource (`bankapp-gateway`) and links it to your BankApp service.
- Verify:
    ```bash
    kubectl get gateway -n bankapp
    ```
    - Status should show **Accepted** and eventually an external address.

### Wait for the AWS NLB:
```bash
kubectl get gateway -n bankapp -w
```
- Gateway status will transition to `Accepted` once the NLB is ready. This typically takes 60–90 seconds.
- **Expected output**: An external IP/DNS name appears under `.status.addresses`.

### Get the external address:
```bash
export APP_URL=$(kubectl get gateway bankapp-gateway -n bankapp -o jsonpath='{.status.addresses[0].value}')
echo "AI-BankApp URL: http://$APP_URL"
```
- Saves the external URL into an environment variable for easy reuse.
- **Expected output**: Prints something like `http://a1b2c3d4e5f6.us-west-2.elb.amazonaws.com`.

### Test the Application:
```bash
# Health check (Spring Boot Actuator)
curl -s http://$APP_URL/actuator/health | python3 -m json.tool
#Expected: {"status":"UP"}.

# Load the home page
curl -s -o /dev/null -w "%{http_code}" http://$APP_URL
#Expected: 200.
```

### Access in Browser
Open `http://$APP_URL` and work through each feature:
1. **Register** — create a new account
2. **Log in** — confirm session persistence (requests should stay on the same pod via the `BANKAPP_AFFINITY` cookie)
3. **Banking operations** — deposit, withdraw, transfer between accounts
4. **AI chatbot** — ask a financial question to confirm Ollama's TinyLlama model is responding
5. **Dark/light mode toggle** — confirms the frontend is fully rendered

>At this point the full stack is live on EKS: Spring Boot serves the UI, MySQL persists accounts and transactions, and Ollama's TinyLlama model powers the AI chatbot — all on managed Kubernetes with persistent storage and autoscaling.

## 3. Deploy the Monitoring Stack
Deploy Prometheus and Grafana to monitor the AI-BankApp on EKS.
### Add Prometheus Helm Repo
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
- Adds the official Prometheus community charts.
- Verify: We should see `prometheus-community` in the repo list:
    ```bash
    helm repo list
    ```

### Install kube‑prometheus‑stack
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=3d \
  --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.requests.cpu=100m \
  --wait --timeout 600s
```
- This single Helm release deploys Prometheus, Grafana, Alertmanager, and all required CRDs.
- **Expected output**: Helm release installed successfully.
- **Verify pods**:
    ```bash
    kubectl get pods -n monitoring
    ```
    - We should see Prometheus, Grafana, and Alertmanager pods running.

### Access Grafana
```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```
- Forwards Grafana service to our local machine.
- **Access**: Open http://localhost:3000.
- **Login**: `admin / admin123`.
- **Expected outcome**: Grafana dashboard loads with pre‑built Kubernetes dashboards.

>**The AI-BankApp exposes Prometheus metrics natively.** The Spring Boot Actuator endpoint at `/actuator/prometheus` provides JVM metrics, HTTP request metrics, and more.

### Create a ServiceMonitor to Scrape the BankApp
The BankApp exposes Prometheus metrics natively via Spring Boot Actuator at `/actuator/prometheus`. Create a `ServiceMonitor` to tell Prometheus to scrape it:
`bankapp-servicemonitor.yaml`
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bankapp-monitor
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames:
      - bankapp
  selector:
    matchLabels:
      app: bankapp
  endpoints:
    - port: "8080"
      path: /actuator/prometheus
      interval: 15s
```
Apply it:
```bash
kubectl apply -f bankapp-servicemonitor.yaml
```
- Tells Prometheus to scrape BankApp’s `/actuator/prometheus` endpoint.
- Verify:
    ```bash
    kubectl get servicemonitor -n monitoring
    ```

### Access Prometheus
```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```
- Open `http://localhost:9090`
- Query AI-BankApp metrics:
    ```promql
    # JVM memory usage
    jvm_memory_used_bytes{namespace="bankapp"}

    # HTTP request rate
    rate(http_server_requests_seconds_count{namespace="bankapp"}[5m])

    # HTTP request latency (95th percentile)
    histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{namespace="bankapp"}[5m]))
    ```

### Explore the Pre-Built Grafana Dashboard
Inside Grafana, check:
- **Kubernetes / Compute Resources / Namespace (Pods)** → select `bankapp` namespace.
- **Kubernetes / Compute Resources / Pod** → drill into individual pods.
- **Node Exporter / Nodes** → see EKS worker node health.

## 4. End-to-End Validation Checklist
Run through each layer of the stack systematically.

### Application Layer
```bash
# All pods running and ready
kubectl get pods -n bankapp
echo "---"

# App responds on health endpoint
curl -s http://$APP_URL/actuator/health
echo "---"

# HPA is active and monitoring CPU
kubectl get hpa -n bankapp
echo "---"

# Prometheus metrics endpoint works
curl -s http://$APP_URL/actuator/prometheus | head -10
```
Expected outcomes:
- All pods show `Running` and `Ready`.
- Health endpoint returns `{"status":"UP"}`.
- HPA shows CPU target and current utilization.
- Prometheus metrics output begins with JVM/HTTP metrics lines.

### Data Layer
```bash
# MySQL is healthy with persistent storage
kubectl exec -n bankapp deploy/mysql -- mysqladmin ping -h localhost -uroot -pTest@123
echo "---"

# PVCs are bound to EBS volumes
kubectl get pvc -n bankapp
echo "---"

# Ollama has the model loaded
kubectl exec -n bankapp deploy/ollama -- ollama list
```
Expected outcomes:
- MySQL responds with `mysqld is alive`.
- PVCs show `Bound` with 5Gi (MySQL) and 10Gi (Ollama).
- Ollama lists `tinyllama` model as loaded.

### Infrastructure Layer
```bash
# Nodes are healthy
kubectl get nodes
kubectl top nodes
echo "---"

# Gateway is serving traffic
kubectl get gateway -n bankapp
echo "---"

# Monitoring is running
kubectl get pods -n monitoring | head -5
```
Expected outcomes:
- Nodes show `Ready` and resource usage via `top`.
- Gateway status shows `Accepted` with external address.
- Monitoring pods (Prometheus, Grafana) are running.

### Security Layer
```bash
# BankApp runs as non-root (devsecops user)
kubectl exec -n bankapp deploy/bankapp -- whoami

# Secrets are not exposed in environment
kubectl get secret bankapp-secret -n bankapp -o yaml | grep -c "MYSQL_ROOT_PASSWORD"
```
Expected outcomes:
- `whoami` returns `devsecops`.
- Secret shows encoded values, not plaintext (grep count > 0 confirms presence but not exposure).

### Final Validation
If all checks pass:
- Application pods are healthy and responding.
- Data layer is persistent and AI model is loaded.
- Infrastructure is stable and Gateway routes traffic.
- Monitoring stack is active.
- Security posture is enforced (non‑root, secrets safe).

## 5. Reflect on the Full EKS Journey
### Map each concept to the day you learned it:
| Day | What You Built | AI-BankApp Connection |
|-----|---------------|----------------------|
| 81 | EKS cluster via Terraform, kubectl connection, manual deploy | Used the project's `terraform/` configs to provision infra |
| 82 | Gateway API, Envoy, TLS, EBS storage, session persistence | Used `k8s/gateway.yml`, `k8s/cert-manager.yml`, `k8s/pv.yml` |
| 83 | Full production deployment, monitoring, validation | Complete stack: app + DB + AI + networking + observability |

### What the AI-BankApp's EKS setup includes that you have now seen:
The full production stack you have now seen and deployed:
- Terraform-provisioned VPC with 3-AZ networking and separate public/private/intra subnets
- Managed node group with autoscaling (min 3, max 5)
- 6 EKS add-ons: CoreDNS, VPC CNI, kube-proxy, Pod Identity Agent, EBS CSI Driver, Metrics Server
- ArgoCD pre-installed (used on Days 84–86 for GitOps)
- Gateway API with Envoy for traffic management — replacing traditional Ingress
- cert-manager with Let's Encrypt for automated HTTPS
- Cookie-based session persistence for the stateful Spring Security session
- EBS persistent storage (gp3) for MySQL and Ollama
- HPA with scale-up and scale-down stabilisation policies
- Spring Boot Actuator metrics endpoint for Prometheus scraping
- Init containers for dependency ordering at pod startup
- `postStart` lifecycle hook for Ollama model pull

### What a Real Production Deployment Would Add
| Enhancement | Purpose |
|---|---|
| Route 53 + ExternalDNS | Automatic DNS records for Gateway addresses — no manual IP management |
| Network Policies | Pod-to-pod traffic isolation — MySQL only reachable from BankApp pods |
| Pod Disruption Budgets | Guarantees minimum availability during node drain (rolling upgrades, spot interruptions) |
| External Secrets Operator | Pulls credentials from AWS Secrets Manager at runtime — nothing sensitive in Git |
| Automated MySQL backups | Scheduled CronJob dumping to S3 with retention policy |
| Loki log aggregation | Centralised logs from all pods (you built this on Day 75) |
| Multi-environment clusters | Separate dev and prod clusters with environment-specific Helm values |

## 6. Complete Teardown
Work through teardown in reverse dependency order to avoid orphaned AWS resources (particularly LoadBalancers and EBS volumes, which Terraform does not always clean up if Kubernetes created them).

### Delete Monitoring Stack
```bash
helm uninstall monitoring -n monitoring
```
- Removes Prometheus + Grafana.
- Verify:
    ```bash
    kubectl get pods -n monitoring
    ```
    - Should return **No resources found**.

### Delete Gateway Resources
```bash
kubectl delete -f k8s/gateway.yml 2>/dev/null
```
- Releases the AWS NLB created for Envoy Gateway.
- Verify:
    ```bash
    kubectl get gateway -n bankapp
    ```
    - Should show **Not Found**.

### Delete BankApp Stack
Run in order:
```bash
kubectl delete -f k8s/hpa.yml
kubectl delete -f k8s/bankapp-deployment.yml
kubectl delete -f k8s/ollama-deployment.yml
kubectl delete -f k8s/mysql-deployment.yml
kubectl delete -f k8s/service.yml
kubectl delete -f k8s/secrets.yml
kubectl delete -f k8s/configmap.yml
kubectl delete -f k8s/pvc.yml
kubectl delete -f k8s/pv.yml
kubectl delete -f k8s/namespace.yml
```
- Cleans up all workloads, configs, and storage.
- Verify:
    ```bash
    kubectl get all -n bankapp
    ```
    - Should return **No resources found**.

### Delete Envoy Gateway + cert-manager
```bash
helm uninstall envoy-gateway -n envoy-gateway-system 2>/dev/null
helm uninstall cert-manager -n cert-manager 2>/dev/null
```
Verify:
```bash
kubectl get pods -n envoy-gateway-system
kubectl get pods -n cert-manager
```
- Both should return **No resources found**.

### Delete Namespaces
```bash
kubectl delete namespace monitoring envoy-gateway-system cert-manager 2>/dev/null
```
Verify:
```bash
kubectl get ns
```
- These namespaces should be gone.

### Check for Lingering Resources
Before running `terraform destroy`, verify Kubernetes has released all AWS-provisioned resources:
```bash
# Check for LoadBalancers
kubectl get svc -A | grep LoadBalancer

# Check for PVCs
kubectl get pvc -A
```
Both commands should return empty output. If a LoadBalancer service still exists, delete it before running `terraform destroy` — otherwise the VPC teardown will fail because the NLB's ENIs still reference the subnets.

### Destroy Infrastructure with Terraform
```bash
cd terraform
terraform destroy
```
- Deletes: EKS cluster, node group, VPC, subnets, NAT Gateway, IAM roles, ArgoCD, and all associated resources.
- **Expected runtime**: 10–15 minutes.

### Post-Teardown Verification in AWS Console 
| Service | What to Check |
|---|---|
| EKS | No clusters listed |
| EC2 → Instances | No running instances |
| EC2 → Load Balancers | No load balancers |
| EC2 → Volumes | No EBS volumes with `kubernetes.io` tags |
| VPC | `bankapp-eks` VPC deleted |
| IAM | No `bankapp-eks-*` roles remaining |
 
> **Billing:** Charges stop within approximately one hour of teardown. Expected cost for the full 3-day lab: **$15–25**, depending on total runtime. Verify in the AWS Billing Dashboard that no unexpected services are still accumulating charges.