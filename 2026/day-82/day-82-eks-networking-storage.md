# EKS Networking with Gateway API and Persistent Storage
>Our EKS cluster is running and the AI-BankApp deployed with raw manifests. But production needs proper ingress, HTTPS, session persistence, and reliable storage. The AI-BankApp project uses the Kubernetes Gateway API with Envoy Gateway instead of traditional Ingress - the next generation of Kubernetes traffic management.

Today we set up the Gateway API, configure TLS with cert-manager, understand EBS storage in action, and explore the AI-BankApp's production networking setup.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch `feat/gitops`) -- `k8s/gateway.yml`, `k8s/cert-manager.yml`, `k8s/pv.yml`, `k8s/pvc.yml`

## Table of Contents
1. [Gateway API vs. Ingress](#1-understand-gateway-api-vs-ingress)
2. [Install Envoy Gateway](#2-install-envoy-gateway)
3. [Deploy the AI-BankApp with Gateway API](#3-deploy-the-ai-bankapp-with-gateway-api)
4. [Set Up TLS with cert-manager](#4-set-up-tls-with-cert-manager)
5. [EBS Persistent Storage in Action](#5-understand-ebs-persistent-storage-in-action)
6. [HPA and Node Capacity](#6-explore-hpa-and-node-capacity)

## 1. Understand Gateway API vs Ingress
The AI-BankApp uses the Gateway API instead of the traditional Ingress resource. The differences are significant in production:
 
| Feature | Ingress | Gateway API |
|---|---|---|
| **API maturity** | Stable but feature-limited | GA since Kubernetes 1.26 |
| **Traffic splitting** | Not supported | Built-in weighted backends |
| **Header matching** | Annotation-dependent, controller-specific | Native `HTTPRoute` match rules |
| **Role separation** | Single monolithic resource | `GatewayClass` (infra team) → `Gateway` (ops team) → `HTTPRoute` (dev team) |
| **TLS management** | Annotation-based, non-standard | Native TLS config on Gateway listeners |
| **Session affinity** | Not standardised | `BackendTrafficPolicy` (Envoy extension) |
 
### The AI-BankApp Gateway Architecture
```
[Internet]
    │
    ▼
[AWS NLB]  ← provisioned automatically by Envoy Gateway
    │
    ▼
[Gateway: bankapp-gateway]
    ├── Listener: HTTP  (port 80)
    └── Listener: HTTPS (port 443, TLS terminated)
            │
            ▼
    [HTTPRoute: bankapp-route]
            │
            ▼
    [Service: bankapp-service:8080]
            │
            ▼
    [Pods: bankapp ×2–4]
    (session affinity via BANKAPP_AFFINITY cookie)
```

**Role separation in practice:**
- The infrastructure team owns the `GatewayClass` — it defines which controller handles Gateways
- The ops team owns the `Gateway` — it controls ports, protocols, and TLS
- The dev team owns the `HTTPRoute` — it defines routing rules without needing cluster-admin access

## 2. Install Envoy Gateway
Envoy Gateway is the Gateway API implementation the AI-BankApp uses.

### Install Envoy Gateway via Helm
Run this command to install Envoy Gateway into its own namespace:
```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 \
  -n envoy-gateway-system --create-namespace \
  --wait
```
- `oci://docker.io/envoyproxy/gateway-helm` → Helm chart location.
- `--version v1.4.0` → specific stable release.
- `-n envoy-gateway-system` → installs into a dedicated namespace.
- `--create-namespace` → creates the namespace if it doesn’t exist.
- `--wait` → waits until pods are ready before finishing.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2166).png)

### Verify Installation
Check that the Envoy Gateway pods are running:
```bash
kubectl get pods -n envoy-gateway-system
```
- We should see pods like `envoy-gateway-controller-xxxx` in Running state.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2170).png)

Check that the GatewayClass has been registered:
```bash
kubectl get gatewayclass
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2173).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2174).png)

- This confirms Envoy Gateway is active and ready to manage Gateway resources.

###  Install Gateway API CRDs
The Gateway API requires CRDs (Custom Resource Definitions). Check if they’re already installed:
```bash
kubectl get crd gateways.gateway.networking.k8s.io
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2178).png)

- If we see `Error from server (NotFound)` or no output, install them:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```
- This installs all standard Gateway API CRDs (GatewayClass, Gateway, HTTPRoute, etc.).

### Confirm CRDs
Verify that the CRDs are present:
```bash
kubectl get crds | grep gateway.networking.k8s.io
```
We should see entries like:
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `gatewayclasses.gateway.networking.k8s.io`

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2182).png)

### What We’ve Achieved
- Envoy Gateway installed and running in `envoy-gateway-system`.
- GatewayClass (`envoy-gateway`) registered.
- Gateway API CRDs installed and ready.

This sets the foundation for **Task 3**, where we’ll deploy the AI-BankApp with Gateway resources (Gateway, HTTPRoute, BackendTrafficPolicy).

## 3. Deploy the AI-BankApp with Gateway API
### Verify the App is Running
Make sure the app is deployed (from Day 81):
```bash
kubectl get pods -n bankapp
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2187).png)

- If we see pods for **bankapp**, **mysql**, and **ollama** in `Running` state → good.
- If not → redeploy the manifests.
    ```bash
    cd AI-BankApp-DevOps
    kubectl apply -f k8s/namespace.yml
    kubectl apply -f k8s/pv.yml
    kubectl apply -f k8s/pvc.yml
    kubectl apply -f k8s/configmap.yml
    kubectl apply -f k8s/secrets.yml
    kubectl apply -f k8s/mysql-deployment.yml
    kubectl apply -f k8s/service.yml
    kubectl apply -f k8s/ollama-deployment.yml
    kubectl apply -f k8s/bankapp-deployment.yml
    kubectl apply -f k8s/hpa.yml
    kubectl get pods -n bankapp
    ```

### Study Gateway Configuration
Open `k8s/gateway.yml` and understand the four resources:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2190).png)

**1. GatewayClass** -- defines which controller handles Gateways:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**2. Gateway** -- creates the actual load balancer with listeners:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bankapp-gateway
  namespace: bankapp
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      hostname: <your-ip>.nip.io
      tls:
        mode: Terminate
        certificateRefs:
          - name: bankapp-tls
```

When this is applied, Envoy Gateway creates an AWS NLB (Network Load Balancer) automatically.

**3. HTTPRoute** -- routes traffic to the BankApp service:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bankapp-route
  namespace: bankapp
spec:
  parentRefs:
    - name: bankapp-gateway
      sectionName: https
    - name: bankapp-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: bankapp-service
          port: 8080
```

**4. BackendTrafficPolicy** -- session persistence via cookies:
```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: bankapp-session
  namespace: bankapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: bankapp-route
  loadBalancer:
    type: ConsistentHash
    consistentHash:
      type: Cookie
      cookie:
        name: BANKAPP_AFFINITY
        ttl: 3600s
```

**Why cookie-based session affinity?**<br>
**Ans.** The AI-BankApp uses Spring Security with form-based login. HTTP sessions are stored in-memory per pod. Without session affinity, subsequent requests from the same user can hit a different pod, invalidating the session and logging the user out. The `BANKAPP_AFFINITY` cookie ensures all requests from a given user are consistently routed to the same pod.

### Apply the Gateway configuration
```bash
kubectl apply -f k8s/gateway.yml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2193).png)

Watch the Gateway until the AWS NLB is provisioned:
```bash
kubectl get gateway -n bankapp -w
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2195).png)

### Get External IP
Once provisioned, extract the IP:
```bash
export GATEWAY_IP=$(kubectl get gateway bankapp-gateway -n bankapp -o jsonpath='{.status.addresses[0].value}')
echo "App URL: http://$GATEWAY_IP"
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2198).png)

### Test Access
Try reaching the app:
```bash
curl http://$GATEWAY_IP
```
Expected: HTML response from the BankApp login page.

![image alt]()

### What We’ve Achieved
- BankApp deployed with Gateway API.
- Envoy Gateway created AWS NLB automatically.
- HTTPRoute mapped traffic to service.
- Session persistence enabled via cookie.

This is the **production-ready ingress setup** - far more robust than traditional Ingress.

## 4. Set Up TLS with cert-manager
The AI-BankApp uses cert-manager with Let's Encrypt for automatic HTTPS certificates.

### Install cert-manager
Add the Jetstack Helm repo and update:
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2203).png)

Install cert-manager with CRDs enabled:
```bash
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true \
  --wait
```
- `--set crds.enabled=true` ensures all required CRDs are installed.
- `--wait` makes Helm wait until pods are ready.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2205).png)

Verify Installation
```bash
kubectl get pods -n cert-manager
```
- We should see pods like `cert-manager`, `cert-manager-webhook`, and `cert-manager-cainjector` in **Running** state.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2208).png)

### Apply ClusterIssuer
Open `k8s/cert-manager.yml`. It defines a **ClusterIssuer** for Let’s Encrypt:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2211).png)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - group: gateway.networking.k8s.io
                kind: Gateway
                name: bankapp-gateway
                namespace: bankapp
```

Apply it:
```bash
kubectl apply -f k8s/cert-manager.yml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2213).png)

### How Certificate Issuance Works
```
1. cert-manager requests a certificate from Let's Encrypt
2. Let's Encrypt issues an HTTP-01 challenge — a specific path must return a specific token
3. cert-manager creates a temporary HTTPRoute to serve the challenge response
4. Let's Encrypt verifies the response and issues the signed certificate
5. cert-manager stores the certificate in the 'bankapp-tls' Secret
6. The Gateway references this Secret for HTTPS termination
```

### Configure Hostname
We need a hostname pointing to our NLB IP. Use **nip.io** for quick DNS:
```bash
export HOSTNAME="${GATEWAY_IP}.nip.io"
echo "HTTPS URL: https://$HOSTNAME"
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2217).png)

Update the Gateway resource (`k8s/gateway.yml`) to use this hostname under the HTTPS listener:
```yaml
hostname: ${HOSTNAME}
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2221).png)

Reapply:
```bash
kubectl apply -f k8s/gateway.yml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2220).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2223).png)

### Verify HTTPS
Wait for cert-manager to issue the certificate:
```bash
kubectl describe certificate -n bankapp
```

![image alt]()

Test HTTPS access:
```bash
curl -k https://$HOSTNAME
```
- Expected: HTML response from the BankApp login page, now served over HTTPS.

![image alt]()

### What We’ve Achieved
- cert-manager installed and running.
- ClusterIssuer configured for Let’s Encrypt.
- Gateway updated with hostname and TLS termination.
- BankApp now accessible securely via HTTPS.

## 5. Understand EBS Persistent Storage in Action
The AI-BankApp uses EBS volumes for MySQL (5 Gi) and Ollama (10 Gi). This section validates that the EBS CSI driver is working correctly and demonstrates the persistence guarantee.

### Check StorageClass
```bash
kubectl get storageclass gp3
```
- We should see `gp3` as the default StorageClass.
- **gp3** is AWS’s latest SSD class: 3000 IOPS baseline, cheaper than gp2, supports expansion.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2226).png)

### Verify PVCs
Check PersistentVolumeClaims in the `bankapp` namespace:
```bash
kubectl get pvc -n bankapp
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2228).png)

- This means both MySQL and Ollama have bound volumes.

### Verify PVs
Check PersistentVolumes:
```bash
kubectl get pv
```
- We should see dynamically provisioned volumes backing those PVCs.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2230).png)

### Confirm EBS Volumes in AWS
Run this AWS CLI command:
```bash
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-by,Values=ebs.csi.aws.com" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,AZ:AvailabilityZone,State:State}" \
  --output table \
  --region us-west-2
```
- This confirms the actual EBS volumes backing the PVCs, including their AZ placement. The volume AZ should match the AZ of the node running the MySQL or Ollama pod - this is enforced by `WaitForFirstConsumer` binding mode.

### Key EBS concepts on EKS
- `WaitForFirstConsumer` -- the volume is created in the same AZ as the pod that claims it
- `ReadWriteOnce` -- EBS can only attach to one node at a time (MySQL and Ollama use Recreate strategy because of this)
- `gp3` -- latest generation SSD, 3000 IOPS baseline, cheaper than gp2
- `allowVolumeExpansion: true` -- you can grow volumes without recreating them

### Test Persistence
Delete the MySQL pod and watch it come back with data intact:
```bash
# Check current MySQL data
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"

# Delete the pod
kubectl delete pod -n bankapp -l app=mysql

# Watch it recreate
kubectl get pods -n bankapp -l app=mysql -w

# Verify data survived
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2235).png)

The database is intact because the EBS volume exists independently of the pod lifecycle. The replacement pod mounts the same volume with all data preserved.

### What We’ve Achieved
- Confirmed PVCs and PVs are bound to EBS volumes.
- Verified actual AWS EBS volumes exist.
- Demonstrated persistence: MySQL data survives pod deletion.

This is the **proof of reliability** we’ll want in our portfolio — showing that stateful workloads are safe on EKS.

## 6. Explore HPA and Node Capacity
The AI-BankApp's HPA scales pods between 2 and 4 based on CPU.

### Check HPA Status
```bash
kubectl get hpa -n bankapp
```
- We should see the **HorizontalPodAutoscaler** resource for `bankapp`.
- It will show **minReplicas=2**, **maxReplicas=4**, and **target CPU utilization=50%**.
- The **CURRENT/TARGETS** column tells us if scaling is happening.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2238).png)

### Check Node Resource Usage
```bash
kubectl top nodes
```
This shows CPU and memory usage across all nodes.
- With 3× **t3.medium** nodes, we have ~6000m CPU (6 cores) and 12Gi memory total.
- Compare usage against requests to see if we’re close to limits.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2239).png)

### Check Pod Resource Usage
```bash
kubectl top pods -n bankapp
```
This shows CPU/memory usage per pod.
- Expect **Ollama** to be the heaviest consumer (~900m CPU, 2Gi memory).
- BankApp pods request 250m CPU, 256Mi memory each.
- MySQL requests 250m CPU, 256Mi memory.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2242).png)

### Resource Budget Analysis
Resource budget for the **AI-BankApp** on 3x `t3.medium` nodes:
| Component | CPU Request | Memory Request | Instances |
|-----------|-----------|---------------|-----------|
| BankApp | 250m | 256Mi | 2-4 pods |
| MySQL | 250m | 256Mi | 1 pod |
| Ollama | 900m | 2Gi | 1 pod |
| Init containers | 50m | 32Mi | temporary |
| System pods | ~500m | ~500Mi | per node |
| **Total available** | **6000m (3 nodes)** | **12Gi (3 nodes)** | |

Ollama is by far the heaviest consumer. With 4 BankApp pods at peak scaling, total CPU requests reach approximately 2,900m plus system overhead - leaving headroom on a 3-node cluster but not a large margin.

### Observe Scaling
Generate load on BankApp (e.g., using `kubectl run` with `curl` loops or Apache Bench).
Watch HPA react:
```bash
kubectl get hpa -n bankapp -w
```
Replica count should increment from 2 → 3 → 4 as CPU crosses the 70% threshold. After stopping the load generator, the HPA scales back down after the stabilisation window (300 seconds by default for scale-down).

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2244).png)

### Clean Up Workload
Keep the cluster for Day 83, but remove the BankApp resources:
```bash
kubectl delete -f k8s/gateway.yml 2>/dev/null
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

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/e4f087db13fb9fb0d6599b6d98040661a453274f/2026/day-82/Screenshots/Screenshot%20(2246).png)

The EKS cluster, node group, and ArgoCD remain running for Day 83.

### What We’ve Achieved
- Verified HPA scaling behavior.
- Checked node and pod resource usage.
- Understood Ollama’s heavy footprint.
- Cleaned up workloads while keeping the cluster ready for Day 83.