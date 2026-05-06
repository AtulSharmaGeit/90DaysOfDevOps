# Kubernetes Architecture and Cluster Setup
 
A deep-dive into Kubernetes concepts, architecture, and hands-on local cluster setup using `kind` or `minikube`.
 
---
 
## Table of Contents
 
- [Background](#background)
  - [Why Kubernetes?](#why-kubernetes)
  - [Origin and Inspiration](#origin-and-inspiration)
  - [What Does the Name Mean?](#what-does-the-name-mean)
- [Architecture](#architecture)
  - [Diagram](#diagram)
  - [Control Plane Components](#control-plane-components)
  - [Worker Node Components](#worker-node-components)
- [Request Lifecycle](#request-lifecycle)
  - [What Happens When You Run kubectl apply?](#what-happens-when-you-run-kubectl-apply)
  - [What Happens If the API Server Goes Down?](#what-happens-if-the-api-server-goes-down)
  - [What Happens If a Worker Node Goes Down?](#what-happens-if-a-worker-node-goes-down)
- [Local Cluster Setup](#local-cluster-setup)
  - [Install kubectl](#install-kubectl)
  - [Option A: kind](#option-a-kind-kubernetes-in-docker)
  - [Option B: minikube](#option-b-minikube)
- [Explore Our Cluster](#explore-our-cluster)
- [Practice Cluster Lifecycle ](#practice-cluster-lifecycle)
---
 
## Background
 
### Why Kubernetes?
 
Docker packages applications into containers, ensuring they run consistently across environments. However, Docker alone does not manage:
 
- **Where** containers run across multiple servers
- **How** they scale up or down based on demand
- **What** happens when a server fails
When applications grow to hundreds or thousands of containers distributed across many machines, we need a system that can schedule workloads, balance resources, and recover from failures automatically.
 
Kubernetes was designed as an open-source orchestrator to automate the **deployment, scaling, and management** of containerized applications across clusters of machines.
 
---
 
### Origin and Inspiration
 
Kubernetes was created at **Google in 2014** by engineers **Joe Beda, Brendan Burns, and Craig McLuckie**. It was donated to the **Cloud Native Computing Foundation (CNCF)** in 2015, which now oversees its development.
 
Kubernetes drew from three key sources of inspiration:
 
| Influence | Contribution |
|---|---|
| **Borg** (Google internal) | Introduced declarative configuration, workload scheduling, and self-healing - all core to Kubernetes |
| **Omega** (Google internal) | Influenced Kubernetes' approach to scalability and scheduler flexibility |
| **Promise Theory** | A distributed systems model where components make autonomous promises about their behavior, rather than relying on strict centralized control |
 
---
 
### What Does the Name Mean?
 
The name **Kubernetes** comes from the Ancient Greek word **κυβερνήτης** (*kubernḗtēs*), meaning **helmsman** or **ship's pilot**.
 
It was chosen to symbolize Kubernetes' role in steering and managing containerized applications across distributed systems - much like a helmsman guiding a ship through open water.
 
> This is also why the Kubernetes logo is a ship's wheel, and why related projects often use nautical terminology (Helm, Harbor, Fleet, etc.)
 
---
 
## Architecture
 
### Diagram
 
```
┌──────────────────────────────────────────────────────┐
│                    CONTROL PLANE                     │
│                                                      │
│   ┌──────────────┐         ┌───────────────────┐     │
│   │  API Server  │◄───────►│       etcd        │     │
│   │ (front door) │         │  (cluster state)  │     │
│   └──────┬───────┘         └───────────────────┘     │
│          │                                           │
│   ┌──────┴───────┐         ┌───────────────────┐     │
│   │  Scheduler   │         │Controller Manager │     │
│   │(node assign) │         │ (reconcile state) │     │
│   └──────────────┘         └───────────────────┘     │
└───────────────────────┬──────────────────────────────┘
                        │  watches / manages
┌───────────────────────▼──────────────────────────────┐
│                    WORKER NODE                       │
│                                                      │
│   ┌──────────────┐         ┌───────────────────┐     │
│   │   kubelet    │         │   kube-proxy      │     │
│   │  (pod agent) │         │  (networking)     │     │
│   └──────────────┘         └───────────────────┘     │
│                                                      │
│   ┌────────────────────────────────────────────┐     │
│   │           Container Runtime                │     │
│   │       (containerd / CRI-O)                 │     │
│   │   ┌──────────┐   ┌──────────┐              │     │
│   │   │  Pod A   │   │  Pod B   │   ...        │     │
│   │   └──────────┘   └──────────┘              │     │
│   └────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```
 
---
 
### Control Plane Components
 
| Component | Role |
|---|---|
| **API Server** | The front door to the cluster. Every `kubectl` command and every internal component communicates through it. Validates requests and persists cluster state to `etcd`. |
| **etcd** | A distributed key-value store and the single source of truth for all cluster state - what pods exist, which nodes are available, and what the desired configuration is. |
| **Scheduler** | Watches for newly created pods with no assigned node. Selects the best node based on resource availability, affinity rules, and constraints. |
| **Controller Manager** | Runs a collection of control loops. Each controller watches current state and drives it toward desired state - for example, the ReplicaSet controller ensures the correct number of pod replicas are always running. |
 
---
 
### Worker Node Components
 
| Component | Role |
|---|---|
| **kubelet** | An agent running on every node. Receives pod specs from the API server and instructs the container runtime to start or stop containers. Reports node and pod status back to the control plane. |
| **kube-proxy** | Maintains network rules on each node so pods can communicate with each other and with Services, both inside and outside the cluster. |
| **Container Runtime** | The engine that actually runs containers. Kubernetes supports any CRI-compatible runtime - most commonly `containerd` or `CRI-O`. Docker Engine is no longer directly supported. |
 
---
 
## Request Lifecycle
 
### What Happens When We Run `kubectl apply`?
 
Tracing `kubectl apply -f pod.yaml` through each component:
 
```
kubectl
  └─► API Server        (validates and persists the pod spec to etcd)
        └─► etcd        (stores desired state)
        └─► Scheduler   (detects unscheduled pod, selects a node, writes binding to etcd)
              └─► kubelet (on the assigned node, reads the pod spec)
                    └─► Container Runtime
                              └─► Pod starts
```
 
---
 
### What Happens If the API Server Goes Down?
 
The API server is the cluster's central communication hub. Its failure has a significant but **not total** impact.
 
**What breaks:**
 
| Capability | Impact |
|---|---|
| `kubectl` commands | All fail — `kubectl get`, `apply`, `delete`, etc. all require the API server |
| Controller reconciliation | Controller Manager and Scheduler cannot read/write state, so they stop functioning |
| Scaling and self-healing | If a pod crashes, no replacements can be created |
| New deployments | No new workloads can be scheduled |
| Cluster visibility | Monitoring dashboards and tools that query the API server stop working |
 
**What continues working:**
 
| Capability | Why |
|---|---|
| Existing pods keep running | `kubelet` and the container runtime manage containers locally and don't depend on the API server for ongoing execution |
| Pod-to-pod networking | `kube-proxy` rules are already in place and continue to function |
 
> **Key insight:** The control plane going down freezes the cluster's desired state management, but it does not immediately kill running workloads.
 
---
 
### What Happens If a Worker Node Goes Down?
 
| Capability | Impact |
|---|---|
| Pods on the failed node | Stop running immediately - kubelet and runtime are gone |
| Node status | Marked `NotReady` by the API server and Controller Manager |
| Cluster capacity | Reduced - fewer CPU and memory resources available for scheduling |
| Other nodes | Unaffected - they continue operating normally |
| Control plane | Fully operational - cluster management continues |
 
**Recovery (automatic):**
 
1. Controller Manager detects missing pod replicas on the failed node
2. Pods are rescheduled onto healthy nodes (if resources are available)
3. Deployments and ReplicaSets restore the desired replica count
4. When the node is repaired or replaced, it rejoins the cluster and becomes schedulable again
> **Key insight:** Kubernetes' self-healing is scoped to workloads managed by controllers (Deployments, ReplicaSets, etc.). Standalone pods are **not** rescheduled automatically.
 

# Local Cluster Setup
## 1. Install kubectl
`kubectl` is the CLI tool that lets us talk to our Kubernetes cluster. 
### For Linux (amd64)
```bash
uname -m
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
### Verify Installation
- Run:
    ```bash
    kubectl version --client
    ```
- We should see output showing the client version of kubectl installed on our machine. That confirms it’s ready to connect to clusters.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(573).png)

## 2. Set Up Our Local Cluster
For our **local Kubernetes cluster setup**, we have two solid options:

### Option A: kind (Kubernetes in Docker)
`kind` runs Kubernetes cluster nodes as Docker containers.
```bash
# Install kind
# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create a cluster
kind create cluster --name devops-cluster

# Verify
kubectl cluster-info
kubectl get nodes
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(577).png)

| Pros | Cons |
|---|---|
| Very lightweight and fast to spin up | Requires Docker to be running |
| Ideal for CI/CD pipelines and quick testing | Less suited for simulating VM-based environments |
| Easy to delete and recreate clusters | |

### Option B: minikube
`minikube` creates a single-node cluster using a VM or Docker driver.
```bash
# Install minikube
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start a cluster
minikube start

# Verify
kubectl cluster-info
kubectl get nodes
```
| Pros | Cons |
|---|---|
| Flexible drivers (Docker, VirtualBox, Hyper-V) | Slightly heavier than kind |
| Rich addon ecosystem (ingress, metrics-server) | Startup can be slower |
| Closer to a real cluster feel | |

### Which one did I choose and why?
**Ans.** I am starting with `kind` because:
- It’s lightweight and integrates seamlessly with Docker (which you’ve already been using).
- Perfect for practicing cluster lifecycle commands (`create`, `delete`, `get nodes`).
- Easy to reset if something breaks.

Later, we can experiment with `minikube` when we want to explore addons, simulate VM drivers, or practice multi-node setups.

## Explore Our Cluster
### See cluster info
```bash
kubectl cluster-info
```
- Shows the control plane endpoints (API server, DNS).
### List all nodes
```bash
kubectl get nodes
```
- Confirms our cluster nodes are Ready.
### Get detailed info about node
```bash
kubectl describe node <node-name>
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(580).png)

- Displays capacity, labels, taints, and running pods.
### List all namespaces
```bash
kubectl get namespaces
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(582).png)

- We’ll see defaults like `default`, `kube-system`, `kube-public`, `kube-node-lease`.
### See ALL pods running in the cluster (across all namespaces)
```bash
kubectl get pods -A
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(584).png)

### Look at the pods running in the `kube-system` namespace:
```bash
kubectl get pods -n kube-system
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(587).png)

- This is where the core Kubernetes components live.
- We should see pods like `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `coredns`, and `kube-proxy`. These are the architecture components we drew in Task 2 - running as pods inside the cluster.

### Matching `kube-system` pods to architecture components
 
The pods running in `kube-system` are the exact components from the architecture diagram above - now visible as containers inside the cluster:
 
| Pod | Architecture Component |
|---|---|
| `etcd-*` | Distributed key-value store for cluster state |
| `kube-apiserver-*` | API Server - the cluster's front door |
| `kube-scheduler-*` | Assigns new pods to nodes |
| `kube-controller-manager-*` | Reconciles desired state with actual state |
| `coredns-*` | DNS-based service discovery inside the cluster |
| `kube-proxy-*` | Manages networking rules for pod communication |

## Practice Cluster Lifecycle
### Delete cluster
```bash
kind delete cluster --name devops-cluster
# (or: minikube delete)
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(589).png)

### Recreate it
```bash
kind create cluster --name devops-cluster
# (or: minikube start)
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(590).png)

### Verify it is back
```bash
kubectl get nodes
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(595).png)

### Check which cluster kubectl is connected to
```bash
kubectl config current-context
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(596).png)

### List all available contexts (clusters)
```bash
kubectl config get-contexts
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(597).png)

### See the full kubeconfig
```bash
kubectl config view
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b0157be9e68ef09b5361a9288a529bd990d84b7f/2026/day-50/Screenshots/Screenshot%20(599).png)

### What is a kubeconfig?
 A **kubeconfig** is a YAML file that tells `kubectl` how to connect to a cluster. It contains:
 
- **Cluster** - API server address and CA certificate
- **User** - credentials (certificate, token, or username/password)
- **Context** - a named binding of cluster + user + namespace

**Default location:**
```
~/.kube/config
```
 When we run any `kubectl` command, it reads this file to determine which cluster to talk to and how to authenticate. When we create a cluster with `kind` or `minikube`, they automatically write a new context into this file and set it as the active context.
