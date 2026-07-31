# Kubernetes Services
A hands-on guide to exposing applications inside and outside a Kubernetes cluster using ClusterIP, NodePort, and LoadBalancer service types.
 
## Table of Contents
- [Why Services?](#why-services)
- [Deploy the Application](#1-deploy-the-application)
- [ClusterIP - Internal Access](#2-clusterip-service-internal-access)
- [Service Discovery with DNS](#3-discover-services-with-dns)
- [NodePort - External Access via Node](#4-nodeport-service-external-access-via-node)
- [LoadBalancer - Cloud External Access](#5-loadbalancer-service-cloud-external-access)
- [Service Types Side by Side](#6-service-types-side-by-side)
- [Clean Up](#7-clean-up)
 
## Why Services?
Every Pod gets its own IP address, but two problems make Pod IPs unreliable:
 
1. **Pod IPs are not stable** - when a Pod restarts or is replaced, it gets a new IP
2. **Deployments run multiple Pods** - there is no single IP to connect to.

A Service solves both problems by providing:
- A **stable IP and DNS name** that never changes, regardless of Pod restarts
- **Automatic load balancing** across all healthy Pods that match its selector
```
[Client] ──► [Service (stable IP)] ──► [Pod 1]
                                   ──► [Pod 2]
                                   ──► [Pod 3]
```
## 1. Deploy the Application
First create a Deployment that we will expose with Services.
### Manifest - `app-deployment.yaml`
```yaml
apiVersion: apps/v1                 #tells Kubernetes which API group/version to use. apps/v1 is stable API for Deployments, ReplicaSets, StatefulSets, etc.
kind: Deployment                    #defines resource type. A Deployment manages Pods via ReplicaSets, ensuring scaling and self‑healing.
metadata:
  name: web-app                     #unique name of Deployment.
  labels:                           #key‑value label used for grouping and Service selectors.
    app: web-app
spec:                               #desired state of Deployment.
  replicas: 3
  selector:                         #tells Deployment which Pods it manages. Must match Pod template labels.
    matchLabels:
      app: web-app
  template:                         #pod blueprint used by Deployment.
    metadata:                       #must match selector above (app: web-app).
      labels:
        app: web-app
    spec:
      containers:                   #defines workload inside each Pod.
      - name: nginx                 #logical name for container.
        image: nginx:1.25           #docker image pulled from registry (Nginx version 1.25).
        ports:                      #declares container listens on port 80 (HTTP). This helps Services target Pod correctly.
        - containerPort: 80
```
### Apply and Check Pods
```bash
kubectl apply -f app-deployment.yaml
kubectl get pods -o wide
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(662).png)

Note the individual Pod IPs shown in the output. These will change whenever a Pod is restarted or replaced - this is exactly the problem Services are designed to fix.

## 2. ClusterIP Service (Internal Access)
ClusterIP is the **default** Service type. It assigns a stable internal IP that is reachable only from within the cluster - pods can talk to each other through it, but it is not accessible from outside.
### Manifest - `clusterip-service.yaml`
```yaml
apiVersion: v1
kind: Service                 #defines resource type. A Service provides stable networking for Pods, abstracting away their dynamic IPs.
metadata:
  name: web-app-clusterip     #unique name of Service in namespace.
spec:                         #desired state of Service.
  type: ClusterIP             #default Service type. Exposes app internally within cluster using a stable IP.
  selector:                   #matches Pods with label app: web-app. This ties Service to Pods created by your Deployment.
    app: web-app
  ports:                      #defines how traffic is routed.
  - port: 80                  #port exposed by Service.
    targetPort: 80            #port inside Pod’s container that traffic is forwarded to.
```
| Field | Purpose |
|---|---|
| `selector.app: web-app` | Routes traffic to all Pods with this label |
| `port: 80` | The port the Service listens on |
| `targetPort: 80` | The port on each Pod to forward traffic to |
### Apply and Check Service
```bash
kubectl apply -f clusterip-service.yaml
kubectl get services
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(663).png)

The `CLUSTER-IP` shown is stable - it will not change even if every Pod behind it is replaced.

### Now test it from inside the cluster:
```bash
# Spin up a temporary pod for testing (auto-deleted on exit)
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh
# Inside the test pod, run:
wget -qO- http://web-app-clusterip
exit
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(665).png)

We should see the Nginx welcome page HTML. The Service load-balanced our request across the 3 running Pods. Run `wget` multiple times - each request may be routed to a different Pod.

## 3. Discover Services with DNS
Kubernetes includes a built-in DNS server. Every Service automatically receives a DNS entry in the format:
```
<service-name>.<namespace>.svc.cluster.local
```
### Test DNS resolution
```bash
kubectl run dns-test --image=busybox:latest --rm -it --restart=Never -- sh
# Inside the pod:
# Short name (works within the same namespace)
wget -qO- http://web-app-clusterip
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(670).png)

```bash
# Full DNS name
wget -qO- http://web-app-clusterip.default.svc.cluster.local
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(667).png)

```bash
# Look up the DNS entry
nslookup web-app-clusterip
exit
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(669).png)

Both names resolve to the same ClusterIP address.
| DNS form | When to use |
|---|---|
| Short name (`web-app-clusterip`) | Communicating within the same namespace |
| Full FQDN (`web-app-clusterip.default.svc.cluster.local`) | Reaching a service across namespaces |
 
The IP returned by `nslookup` should match the `CLUSTER-IP` shown in `kubectl get services`.

## 4. NodePort Service (External Access via Node)
A NodePort Service opens a specific port on **every node** in the cluster, making the application reachable from outside the cluster via `<NodeIP>:<NodePort>`.
### Manifest — `nodeport-service.yaml`
```yaml
apiVersion: v1                  #uses core Kubernetes API group (v1) for basic resources like Services, Pods, ConfigMaps.
kind: Service                   #defines resource type. A Service provides stable networking for Pods.
metadata:
  name: web-app-nodeport        #unique name of Service.
spec:                           #desired state of Service.
  type: NodePort                #exposes Service on each Node’s IP at a static port (between 30000–32767). This allows external access without a LoadBalancer.
  selector:                     #matches Pods with label app: web-app. This ties Service to our Deployment Pods.
    app: web-app
  ports:
  - port: 80                    #port exposed by Service inside cluster.
    targetPort: 80              #port inside Pod’s container that traffic is forwarded to.
    nodePort: 30080             #external port on each Node where Service is accessible.
```
> NodePort values must be in the range **30000–32767**.
 
**Traffic flow:** `<NodeIP>:30080` → Service → `Pod:80`

```bash
kubectl apply -f nodeport-service.yaml
kubectl get services
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(672).png)

### Access the service:
```bash
# If using Kind, get the node IP first
kubectl get nodes -o wide
# Then curl <node-internal-ip>:30080

# If using Docker Desktop
curl http://localhost:30080
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(673).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(676).png)

## 5. LoadBalancer Service (Cloud External Access)
In a cloud environment (AWS, GCP, Azure), a LoadBalancer Service provisions a real external load balancer that routes public traffic to your cluster nodes.
### Manifest — `loadbalancer-service.yaml`
```yaml
apiVersion: v1                      #uses core Kubernetes API group (v1) for basic resources like Services, Pods, ConfigMaps.
kind: Service
metadata:
  name: web-app-loadbalancer
spec:
  type: LoadBalancer                #Requests an external load balancer from the cloud provider. On managed clusters (EKS, GKE, AKS), this provisions a cloud load balancer and assigns a public IP. On bare EC2 or local clusters, the EXTERNAL-IP stays <pending> unless you install something like MetalLB.
  selector:                         #matches Pods with label app: web-app. This ties Service to our Deployment Pods.
    app: web-app
  ports:
  - port: 80                        #port exposed by Service externally.
    targetPort: 80                  #port inside Pod’s container that traffic is forwarded to.
```

```bash
kubectl apply -f loadbalancer-service.yaml
kubectl get services
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(677).png)

On a local cluster (Minikube, Kind, Docker Desktop), the EXTERNAL-IP will show `<pending>` because there is no cloud provider to create a real load balancer. This is expected.

In a real cloud cluster, the EXTERNAL-IP would be a public IP address or hostname provisioned by the cloud provider.

## 6. Service Types Side by Side
### Check all three services:
```bash
kubectl get services -o wide
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(678).png)

### Compare them:
| Type | Accessible From | Use Case |
|---|---|---|
| **ClusterIP** | Inside the cluster only | Internal communication between services |
| **NodePort** | Outside via `<NodeIP>:<NodePort>` | Development, testing, direct node access |
| **LoadBalancer** | Outside via cloud load balancer | Production traffic in cloud environments |
 
### Service types are cumulative
Each type builds on the one before it:
```
LoadBalancer
  └─► creates a NodePort
        └─► creates a ClusterIP
```
A LoadBalancer Service therefore also has a ClusterIP and a NodePort automatically assigned. Verify this:
```bash
kubectl describe service web-app-loadbalancer
```
 
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(679).png)

We should see all three: a `ClusterIP`, a `NodePort`, and the `LoadBalancer` configuration in the output.

## 7. Clean Up
```bash
kubectl delete -f app-deployment.yaml
kubectl delete -f clusterip-service.yaml
kubectl delete -f nodeport-service.yaml
kubectl delete -f loadbalancer-service.yaml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(680).png)

```bash
kubectl get pods
kubectl get services
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0f94421d250bdde5786498a0c59a729f44765c28/2026/day-53/Screenshots/Screenshot%20(681).png)

Only the built-in `kubernetes` service in the default namespace should remain.
