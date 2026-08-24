# Introduction to Amazon EKS with Terraform
>We have been running Kubernetes locally with Kind. That works for learning, but the AI-BankApp needs a production environment - managed control plane, auto-scaling nodes, persistent EBS storage, and IAM integration.

Amazon EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes offering. The AI-BankApp project (https://github.com/TrainWithShubham/AI-BankApp-DevOps, branch `feat/gitops`) already has a complete Terraform configuration in its `terraform/` directory that provisions a production-grade EKS cluster. Today we understand EKS architecture, study the Terraform configs, provision the cluster, and connect to it.

## Table of Contents
1. [Understanding EKS Architecture](#1-understand-eks-architecture)
2. [Study the Terraform Configuration](#2-study-the-ai-bankapp-terraform-configuration)
3. [Provision the EKS Cluster](#3-provision-the-eks-cluster)
4. [Connect to the Cluster](#4-connect-to-your-cluster)
5. [Deploy the AI-BankApp Manually](#5-deploy-the-ai-bankapp-manually-before-argocd)
6. [EKS Costs and Clean-Up Strategy](#6-understand-eks-costs-and-clean-up-strategy)

## 1. Understand EKS Architecture
### Managed vs. Self-Managed Kubernetes
EKS is AWS's managed Kubernetes offering. The key distinction from self-managed Kubernetes is which parts of the cluster we are responsible for:
| Layer | Managed by |
|---|---|
| API server, etcd, scheduler, controller manager | **AWS** - patched, upgraded, and HA across multiple AZs automatically |
| Worker nodes (EC2 instances running our pods) | **We** - provisioned via node groups |
 
### EKS Node Group Types
| Type | Description | When to Use |
|---|---|---|
| **Managed Node Groups** | AWS handles provisioning, scaling, AMI updates, and draining | Most production workloads |
| **Self-Managed Nodes** | We own the EC2 lifecycle | Custom AMIs or deep OS customisation |
| **Fargate Profiles** | Serverless - no nodes to manage at all | Bursty, lightweight workloads |
 
The AI-BankApp uses **Managed Node Groups** on AL2023 AMI.
 
### EKS Add-ons Used by the AI-BankApp
| Add-on | Purpose |
|---|---|
| `coredns` | DNS resolution for services and pods inside the cluster |
| `kube-proxy` | Network routing for Kubernetes Services |
| `vpc-cni` | AWS VPC CNI - assigns real VPC IPs directly to pods |
| `eks-pod-identity-agent` | Enables pod-level IAM roles without node-level credentials |
| `aws-ebs-csi-driver` | Allows pods to provision and mount EBS volumes - required for MySQL and Ollama storage |
| `metrics-server` | Provides resource metrics for `kubectl top` and HPA autoscaling |

## 2. Study the AI-BankApp Terraform Configuration
### Clone the Repo and Examine the `terraform/` Directory:
```bash
sudo apt update && sudo apt upgrade -y
git clone -b feat/gitops https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
cd AI-BankApp-DevOps/terraform
ls
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2043).png)

### File Inventory
| File | What It Provisions |
|---|---|
| `variables.tf` | Input variable declarations (cluster name, region, instance type, node counts) |
| `terraform.tfvars` | Default variable values |
| `provider.tf` | AWS, Helm, and Kubernetes providers; local values |
| `vpc.tf` | VPC with 9 subnets across 3 AZs, NAT Gateway, Internet Gateway |
| `eks.tf` | EKS cluster, managed node group, 6 add-ons, IRSA for EBS CSI driver |
| `argocd.tf` | ArgoCD Helm release, LoadBalancer service, cluster dependency |
| `outputs.tf` | `aws eks update-kubeconfig` command, ArgoCD password retrieval command |

### Default Variable Values (`terraform.tfvars`)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2047).png)
 
### VPC Design (`vpc.tf`)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2048).png)

Uses the `terraform-aws-modules/vpc/aws` module across 3 Availability Zones:
| Subnet Type | CIDR Range | Purpose | Load Balancer Tag |
|---|---|---|---|
| Public | 10.0.1–3.0/24 | Internet-facing load balancers, NAT Gateway egress point | `kubernetes.io/role/elb` |
| Private | 10.0.4–6.0/24 | Worker nodes and pods — no direct internet access | `kubernetes.io/role/internal-elb` |
| Intra | 10.0.7–9.0/24 | EKS control plane ENIs — isolated from internet | None |
 
The subnet tags are required for the AWS Load Balancer Controller to automatically discover which subnets to use when creating load balancers from Kubernetes Service and Ingress objects.
 
### EKS Cluster (`eks.tf`)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2050).png)

- Module: `terraform-aws-modules/eks/aws` (~> 21.0)
- Node AMI: AL2023 (Amazon Linux 2023)
- Node type: `t3.medium`, desired 3 / max 5
- All 6 add-ons installed as managed cluster add-ons
- IRSA (IAM Roles for Service Accounts) configured for the EBS CSI driver — grants the driver permission to create and attach EBS volumes without using node-level IAM credentials
- Both public and private API endpoint access enabled

### ArgoCD (`argocd.tf`)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2052).png)

- Installed via the `argo-cd` Helm chart
- Exposed as a `LoadBalancer` Service — provisions an AWS Classic or NLB automatically
- Declared with a `depends_on` the EKS module, ensuring it is installed only after the cluster and node group are fully ready

### Architecture Overview
```
AWS Region (us-west-2)
└── VPC (10.0.0.0/16)
    ├── Public Subnets (10.0.1-3.0/24)  — Load Balancers, NAT Gateway egress
    │   └── tagged: kubernetes.io/role/elb
    ├── Private Subnets (10.0.4-6.0/24) — Worker Nodes, Pods
    │   └── tagged: kubernetes.io/role/internal-elb
    └── Intra Subnets (10.0.7-9.0/24)   — EKS Control Plane ENIs
        │
        ├── EKS Control Plane (AWS-managed, multi-AZ)
        │   └── API Server · etcd · Scheduler · Controller Manager
        │
        └── Managed Node Group (3× t3.medium, min 3 / max 5)
            └── Pods: BankApp · MySQL · Ollama · ArgoCD · Add-ons
```

## 3. Provision the EKS Cluster
### Make sure we have the required Tools:
```bash
terraform --version    # >= 1.0
aws --version          # AWS CLI v2
kubectl version --client
helm version
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2053).png)

### Configure AWS Credentials:
```bash
aws configure
# Enter: Access Key ID, Secret Access Key, Region (us-west-2), Output (json)

# Verify
aws sts get-caller-identity
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2057).png)

Expected output includes our AWS account ID, user ARN, and user ID. If this fails, our credentials are not configured correctly — do not proceed to `terraform apply`.

### Initialize and Apply:
```bash
cd terraform

terraform init -upgrade
```
Downloads the AWS, Helm, and Kubernetes provider plugins and the referenced modules.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2068).png)

### Review the Plan
```bash
terraform plan
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2070).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2071).png)

Review the plan carefully. It will create:
| Resource | Count |
|---|---|
| VPC with 9 subnets, NAT Gateway, Internet Gateway | 1 |
| EKS cluster (control plane) | 1 |
| Managed node group (3× t3.medium) | 1 |
| EKS add-ons | 6 |
| IAM roles and policies (cluster, nodes, EBS CSI) | Several |
| ArgoCD Helm release | 1 |
 
Review the plan output carefully before applying - in particular, confirm the region, instance type, and node count match our expectations.

### Apply the Configuration
```bash
terraform apply
```
> This takes **15–20 minutes**. The longest steps are EKS cluster creation (~10 min) and the ArgoCD Helm release waiting for the node group to be ready.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2076).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2079).png)

After completion, note the outputs:
```bash
terraform output
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2081).png)

We’ll get:
- `aws eks update-kubeconfig` command → use this to connect `kubectl`.
- ArgoCD admin password retrieval command.

### At this stage:
- Our **EKS cluster** is live.
- Terraform has provisioned VPC, subnets, NAT, node group, IAM roles, add-ons, and ArgoCD.
- We have helper outputs ready for connecting `kubectl`.

## 4. Connect to Your Cluster
### Update kubeconfig
Terraform gave us the command in its outputs. Run:
```bash
aws eks update-kubeconfig --name bankapp-eks --region us-west-2
```
- This updates our local `~/.kube/config` so `kubectl` knows how to reach the cluster.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2084).png)

### Verify the Connection:
```bash
# Check context
kubectl config current-context

# Cluster info
kubectl cluster-info

# List nodes
kubectl get nodes -o wide
```
We should see 3 nodes with status `Ready`, instance type `t3.medium`, spread across 3 AZs.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2087).png)

### Explore the Cluster:
```bash
# All system pods (DNS, networking, storage, metrics)
kubectl get pods -n kube-system
 
# DaemonSets — one pod per node for kube-proxy, vpc-cni, etc.
kubectl get daemonsets -n kube-system
 
# EBS CSI driver pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
 
# Confirm metrics-server is functioning
kubectl top nodes
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2100).png)

### Check ArgoCD is Running:
```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2106).png)

### Retrieve ArgoCD Credentials
Get the ArgoCD admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2109).png)

Get the ArgoCD LoadBalancer URL:
```bash
kubectl get svc -n argocd argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2116).png)

### Log in to ArgoCD
- Open the LoadBalancer URL in browser.
- Username: `admin`
- Password: (from the secret above).

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2118).png)

- We’re now inside ArgoCD — we’ll use this in Days 84–86 for GitOps.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2120).png)

### At this stage:
- `kubectl` is connected to our EKS cluster.
- Nodes are healthy and spread across AZs.
- System add-ons (DNS, networking, storage, metrics) are running.
- ArgoCD is installed and accessible.

## 5. Deploy the AI-BankApp Manually (Before ArgoCD)
Before introducing GitOps, validate the cluster works by deploying the AI-BankApp from raw manifests. This confirms EBS volumes provision correctly, init containers work as expected, and the app is reachable.

### Apply the Raw Manifests from the `k8s/` Directory:
Move to Repo Root
```bash
cd ../   # from terraform/ back to repo root
```
Apply Raw Manifests in Order
```bash
#Run these one by one
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
```
- This sets up namespace, storage, configs, secrets, MySQL, Ollama, BankApp, and autoscaler.
- Order matters: storage and config must exist before Deployments that reference them.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2123).png)

### Watch the Pods Start:
```bash
kubectl get pods -n bankapp -w
```
The startup order is:
1. **MySQL** starts and becomes healthy (15-30 seconds)
2. **Ollama** starts and pulls the TinyLlama model (2-5 minutes)
3. **BankApp** init containers wait for both, then the app starts (30-60 seconds after dependencies)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2126).png)

All pods should reach `Running`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2129).png)

### Check PVCs are Bound to EBS Volumes:
```bash
kubectl get pvc -n bankapp
kubectl get pv
```
- We should see 5Gi (MySQL) and 10Gi (Ollama) EBS volumes in the **correct AZs**.
- Both PVCs should show `STATUS = Bound`. The backing EBS volumes are provisioned in the same AZ as the node the pod is scheduled on - confirm this matches our nodes from `kubectl get nodes -o wide`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2135).png)

### Access the App
Once all pods are running, access the app:
```bash
kubectl port-forward svc/bankapp-service -n bankapp 8080:8080 --address 0.0.0.0
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2143).png)

Open in browser:
```Code
http://localhost:8080
```
Expected:
- AI-BankApp login page.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2151).png)

- Register account, log in, test AI Chatbot to confirm all three services are communicating correctly.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/36eb1231db23f7a8a211e8a1e8cdd27c44c85ca3/2026/day-81/Screenshots/Screenshot%20(2153).png)

### Verify HPA
```bash
kubectl get hpa -n bankapp
```
Expected:
- HPA object present.
- Target CPU utilization configured.
- Current/desired replicas shown.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2158).png)

### At this stage:
- BankApp, MySQL, and Ollama are running on EKS.
- PVCs are bound to EBS volumes.
- We can access the app locally via port-forward.
- HPA is active and monitoring CPU.

## 6. Understand EKS Costs and Clean Up Strategy
EKS is billed continuously while resources are running. The AI-BankApp cluster costs approximately:

| Component | Cost (approximate) |
|-----------|-------------------|
| EKS Control Plane | $0.10/hr (~$73/month) |
| t3.medium nodes (3x) | ~$0.042/hr each (~$91/month total) |
| NAT Gateway | ~$0.045/hr + data transfer (~$33/month) |
| EBS volumes (15Gi total) | ~$1.50/month |
| LoadBalancer (ArgoCD) | ~$0.025/hr (~$18/month) |
| **Total for this lab** | **~$220/month (~$7/day)** |

>**Do not leave the cluster running when we are not actively using it.**


### Delete the BankApp workload (keep the cluster for Days 82-83):
```bash
#Run these in order
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
- This removes all application resources while keeping the EKS cluster, node group, and ArgoCD alive for the next sessions.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2245).png)

### Destroy Everything (do this at the end of Day 83 or if taking a break):
```bash
cd terraform
terraform destroy
```
- Type `yes` when prompted.
- Takes ~10–15 minutes.
- Deletes VPC, subnets, NAT, EKS cluster, node group, IAM roles, EBS volumes,LoadBalancer, ArgoCD.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2249).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f6cad694f43be7ecb14720593c51801e3ec73e2a/2026/day-81/Screenshots/Screenshot%20(2250).png)

After this, our AWS account should be completely clean - no leftover resources.

### What are the cost components of the AI-BankApp EKS setup? Why is the NAT Gateway surprisingly expensive?
**Ans.** The NAT Gateway is surprisingly expensive because it charges both per hour and per gigabyte of outbound data. Every pod in the private subnets that pulls a container image, downloads a package, or calls an external API routes through the NAT Gateway. Image pulls for MySQL, Ollama (a large model), and the BankApp add up quickly.
