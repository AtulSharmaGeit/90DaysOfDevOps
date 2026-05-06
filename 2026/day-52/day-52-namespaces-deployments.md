# Kubernetes Namespaces and Deployments
A hands-on guide to organizing workloads with namespaces, managing replicated applications with Deployments, and practicing self-healing, scaling, and rolling updates.
 
## Table of Contents
 
- [Explore Default Namespaces](#1-explore-default-namespaces)
- [Create and Use Custom Namespaces](#2-create-and-use-custom-namespaces)
- [Create Our First Deployment](#3-create-our-first-deployment)
- [Self-Healing (Delete a Pod and Watch It Come Back)](#4-self-healing-delete-a-pod-and-watch-it-come-back)
- [Scale the Deployment](#5-scale-the-deployment)
- [Rolling Updates and Rollbacks](#6-rolling-updates-and-rollbacks)
- [Clean Up](#7-clean-up)
## 1. Explore Default Namespaces
### Kubernetes comes with built-in namespaces.
Every Kubernetes cluster starts with four built-in namespaces:
```bash
kubectl get namespaces
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(635).png)

We should see at least:
| Namespace | Purpose |
|---|---|
| `default` | Where our resources land if no namespace is specified |
| `kube-system` | Internal Kubernetes components (API server, scheduler, etc.) |
| `kube-public` | Publicly readable cluster resources |
| `kube-node-lease` | Node heartbeat tracking for availability detection |

Check what is running inside `kube-system`:
```bash
kubectl get pods -n kube-system
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(636).png)

These are the control plane components keeping our cluster alive:
| Pod | Role |
|---|---|
| `etcd-*` | Cluster state store |
| `kube-apiserver-*` | API server - the cluster's front door |
| `kube-scheduler-*` | Assigns pods to nodes |
| `kube-controller-manager-*` | Ensures actual state matches desired state |
| `coredns-*` | DNS-based service discovery |
| `kube-proxy-*` | Manages pod networking rules |
 
> **Do not modify or delete pods in `kube-system`.** These are critical system components.
 

## 2. Create and Use Custom Namespaces
Create two namespaces - one for a development environment and one for staging:
```bash
kubectl create namespace dev
kubectl create namespace staging
```
Verify they exist:
```bash
kubectl get namespaces
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(638).png)

We can also create a namespace from a manifest:
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```
```bash
kubectl apply -f namespace.yaml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(639).png)

Now run a pod in a specific namespace:
```bash
kubectl run nginx-dev --image=nginx:latest -n dev
kubectl run nginx-staging --image=nginx:latest -n staging
```
List pods across all namespaces:
```bash
kubectl get pods -A
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(641).png)

| Command | Shows |
|---|---|
| `kubectl get pods` | `default` namespace only |
| `kubectl get pods -n dev` | `dev` namespace only |
| `kubectl get pods -A` | Every namespace in the cluster |

## 3. Create Our First Deployment
A Deployment tells Kubernetes: "I want X replicas of this Pod running at all times." If a Pod crashes, the Deployment controller recreates it automatically.

Create a file `nginx-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
```
### Key differences from a standalone Pod
| Field | Standalone Pod | Deployment |
|---|---|---|
| `apiVersion` | `v1` | `apps/v1` |
| `kind` | `Pod` | `Deployment` |
| `replicas` | N/A | Desired number of identical pods |
| `selector.matchLabels` | N/A | Links the Deployment to the pods it manages |
| `template` | N/A | Pod blueprint - Deployment creates pods from this |

Apply it:
```bash
kubectl apply -f nginx-deployment.yaml
```
Check the result:
```bash
kubectl get deployments -n dev
kubectl get pods -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(644).png)

- We should see 3 pods with names like `nginx-deployment-xxxxx-yyyyy`.

### Understanding the Deployment output columns

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(643).png)

| Column | Meaning |
|---|---|
| `READY` | Running pods vs desired (e.g. `2/3` means 1 is still starting) |
| `UP-TO-DATE` | Pods updated to the latest Deployment spec |
| `AVAILABLE` | Pods ready to serve traffic - the real measure of health |

Together, these columns confirm whether our Deployment is healthy, fully rolled out, and serving traffic.

## 4. Self-Healing (Delete a Pod and Watch It Come Back)
This is the key difference between a Deployment and a standalone Pod.

### List current Pods in dev namespace
```bash
kubectl get pods -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(645).png)

### Delete one of the deployment's pods (use an actual pod name from the output)
```bash
kubectl delete pod <pod-name> -n dev
```

### Check again immediately
```bash
kubectl get pods -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(649).png)

The Deployment controller detects that only 2 of 3 desired replicas exist and immediately creates a new one. The deleted pod is replaced within seconds.

### What happens to the pod name?
 
The replacement pod gets a **new, unique name** - it does not reuse the deleted pod's name. Pod names are dynamically generated with random suffixes (e.g., `nginx-deployment-7f9d8c9b4f-abcde`).
 
| Behavior | Standalone Pod | Deployment-managed Pod |
|---|---|---|
| Pod deleted | Gone permanently | Recreated automatically |
| Pod name reused | N/A | No - new suffix every time |
| Replica count maintained | No | Yes |

This is the critical difference: standalone Pods vanish forever, but Deployments recreate them automatically.

## 5. Scale the Deployment
Change the number of replicas:

### Scale up to 5 replicas
```bash
kubectl scale deployment nginx-deployment --replicas=5 -n dev
kubectl get pods -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(651).png)

- Kubernetes will create 2 new Pods, bringing the total to 5.

### Scale down to 2 replicas
```bash
kubectl scale deployment nginx-deployment --replicas=2 -n dev
kubectl get pods -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(652).png)

- Kubernetes terminates 3 Pods, leaving only 2 running.

We can also scale by editing the manifest:
```yaml
replicas: 4
```
in `nginx-deployment.yaml`, then re‑apply:
```bash
kubectl apply -f nginx-deployment.yaml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(654).png)

### What happens during scaling?
 
| Direction | Kubernetes action |
|---|---|
| Scale **up** | Creates new pods until desired count is reached |
| Scale **down** | Terminates excess pods gracefully |

## 6. Rolling Updates and Rollbacks
Kubernetes updates Deployments using a rolling strategy by default - new pods are brought up before old ones are terminated, ensuring **zero downtime**.
### Update the Nginx image version to trigger a rolling update:
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev
```

### Watch the rollout in real time:
```bash
kubectl rollout status deployment/nginx-deployment -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(655).png)

- Kubernetes replaces pods one by one - old pods are terminated only after new ones are healthy. This means zero downtime.

### Check the rollout history:
```bash
kubectl rollout history deployment/nginx-deployment -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(656).png)

### Now roll back to the previous version:
```bash
kubectl rollout undo deployment/nginx-deployment -n dev
kubectl rollout status deployment/nginx-deployment -n dev
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(658).png)

### Verify the image is back to the previous version:
```bash
kubectl describe deployment nginx-deployment -n dev | grep Image
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(659).png)

### Rolling update behavior summary
 
| Phase | What Kubernetes does |
|---|---|
| Update triggered | Starts creating new pods with the updated image |
| New pods healthy | Begins terminating old pods |
| Rollout complete | All replicas running the new version |
| Rollback triggered | Reverses to the previous recorded revision |

## 7. Clean Up
### Run these in order:
```bash
kubectl delete deployment nginx-deployment -n dev
kubectl delete pod nginx-dev -n dev
kubectl delete pod nginx-staging -n staging
kubectl delete namespace dev staging production
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(660).png)

### Verify everything is gone
```bash
kubectl get namespaces
kubectl get pods -A
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b9434a1800589e50bffb8b20721c2557ecb57c14/2026/day-52/Screenshots/Screenshot%20(661).png)

After deletion, only the built-in namespaces (`default`, `kube-system`, `kube-public`, `kube-node-lease`) and their system pods remain.
 
> **Warning:** Deleting a namespace removes every resource inside it - pods, deployments, services, and configmaps. Be extremely careful with this command in production.
