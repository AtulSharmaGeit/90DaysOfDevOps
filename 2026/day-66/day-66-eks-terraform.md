# Provision an EKS Cluster with Terraform Modules
A hands-on guide to provisioning a production-style Amazon EKS cluster using official Terraform registry modules, connecting `kubectl`, deploying a workload with a cloud load balancer, and cleaning up safely.
 
## Table of Contents
- [Project Setup](#1-project-setup)
- [Create the VPC with Registry Module](#2-create-the-vpc-with-registry-module)
- [Create the EKS Cluster with Registry Module](#3-create-the-eks-cluster-with-registry-module)
- [Apply and Connect kubectl](#4-apply-and-connect-kubectl)
- [Deploy a Workload on the Cluster](#5-deploy-a-workload-on-the-cluster)
- [Destroy Everything](#6-destroy-everything)

## 1. Project Setup
### Directory structure
```
terraform-eks/
├── providers.tf        # Provider and backend config
├── vpc.tf              # VPC module call
├── eks.tf              # EKS module call
├── variables.tf        # All input variables
├── outputs.tf          # Cluster outputs
├── terraform.tfvars    # Variable values
└── k8s/                # Kubernetes manifests
```
```bash
mkdir terraform-eks && cd terraform-eks
touch providers.tf vpc.tf eks.tf variables.tf outputs.tf terraform.tfvars
mkdir k8s
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2652).png)

### Configure Providers (`providers.tf`)
This file tells Terraform which cloud provider and plugins to use.
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```
| Provider | Version | Purpose |
|---|---|---|
| `hashicorp/aws` | `~> 5.0` | Provisions all AWS infrastructure |
| `hashicorp/kubernetes` | `~> 2.0` | Manages Kubernetes workloads after cluster creation |
- **AWS provider pinned to ~> 5.0** → ensures compatibility and avoids breaking changes.
- **Kubernetes provider pinned** → we’ll use it later to manage workloads.
- **region** → pulled from a variable so we can change it easily.

### Define Input Variables (`variables.tf`)
This file declares all configurable inputs.
```hcl
variable "region" {
  description = "AWS region to deploy resources"
  type        = string
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "terraweek-eks"
}

variable "cluster_version" {
  description = "EKS cluster version"
  type        = string
  default     = "1.31"
}

variable "node_instance_type" {
  description = "EC2 instance type for worker nodes"
  type        = string
  default     = "t3.medium"
}

variable "node_desired_count" {
  description = "Desired number of worker nodes"
  type        = number
  default     = 2
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}
```
- Each variable has a **type** (string/number).
- Defaults make it easy to run without extra config.
- We can override values in `terraform.tfvars`.

### Set Variable Values (`terraform.tfvars`)
This file holds actual values we want to use.
```hcl
region           = "ap-south-1"
cluster_name     = "terraweek-eks"
cluster_version  = "1.31"
node_instance_type = "t3.medium"
node_desired_count = 2
vpc_cidr         = "10.0.0.0/16"
```
- `ap-south-1` = Mumbai region (closest).
- We can change these later without touching `variables.tf`.

### Verify Setup
Run:
```bash
terraform init
terraform validate
```
If everything is correct, Terraform will initialize the AWS provider and confirm our files are valid.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2661).png)

## 2. Create the VPC with Registry Module
EKS requires a VPC with both public and private subnets spread across multiple availability zones.

### Open/Create `vpc.tf`
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraweek-vpc"
  cidr = var.vpc_cidr

  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}
```
- **source** → pulls the official VPC module from Terraform Registry.
- **azs** → at least 2 availability zones for high availability.
- **public_subnets** → for LoadBalancers and internet‑facing services.
- **private_subnets** → for worker nodes and internal services.
- **NAT gateway** → allows private nodes to reach the internet (e.g., to pull container images).
- **DNS hostnames** → required for EKS to resolve service names.
- **tags** → tell EKS which subnets to use for external vs internal ELBs.

### Initialize Terraform
Run:
```bash
terraform init
```
This downloads the VPC module and AWS provider.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2665).png)

### Plan the VPC
Run:
```bash
terraform plan
```
- We should see a plan with ~20 resources (VPC, subnets, route tables, NAT gateway, etc.).

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2668).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2671).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2673).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2675).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2677).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2679).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2681).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2683).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2685).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2687).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2689).png)

### Why does EKS need both public and private subnets?
- **Public subnets** → expose Kubernetes `LoadBalancer` services (like Nginx) to the internet.
- **Private subnets** → host worker nodes securely, away from direct internet access.
- This separation follows AWS best practices for production workloads.

### What do the subnet tags do?
- `"kubernetes.io/role/elb"` → marks public subnets for external ELBs (internet‑facing).
- `"kubernetes.io/role/internal-elb"` → marks private subnets for internal ELBs (inside the VPC).
- EKS uses these tags to automatically place services in the correct subnet type.

## 3. Create the EKS Cluster with Registry Module
### Create `eks.tf`
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    terraweek_nodes = {
      ami_type       = "AL2_x86_64"
      instance_types = [var.node_instance_type]

      min_size     = 1
      max_size     = 3
      desired_size = var.node_desired_count
    }
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```
- **source** → pulls the official EKS module from Terraform Registry.
- **cluster_name / cluster_version** → controlled by your variables.
- **vpc_id / subnet_ids** → wires the cluster into the VPC you created in Task 2.
- **cluster_endpoint_public_access = true** → allows you to connect `kubectl` from our local machine.
- **eks_managed_node_groups** → defines worker nodes (AMI type, instance type, scaling).
- **tags** → useful for cost tracking and AWS console clarity.

### What the EKS module provisions automatically
The registry module creates **30+ resources** — you do not write any of these manually:
| Resource type | Examples |
|---|---|
| Control plane | EKS cluster, API endpoint, CloudWatch logs |
| IAM | Cluster role, node role, policy attachments |
| Networking | Security groups, ENIs |
| Compute | Managed node group, autoscaling group, EC2 instances |

### Initialize Terraform
Run:
```bash
terraform init
```
This downloads the EKS module and its dependencies.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2691).png)

### Plan the Cluster
Run:
```bash
terraform plan
```
Expect to see **30+ resources** in the plan:
- EKS cluster
- IAM roles and policies
- Security groups
- Node group (EC2 instances)
- Autoscaling group
- CloudWatch log groups
- ENIs (Elastic Network Interfaces)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2694).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2696).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2698).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2700).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2702).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2705).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2707).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2709).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2711).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2713).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2715).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2717).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2719).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2722).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2724).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2726).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2728).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2730).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2732).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2734).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2736).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2738).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2740).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2743).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2745).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2747).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2749).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2751).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2754).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2755).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2757).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2759).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2761).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2763).png)

### Review Carefully
Before applying:
- Check that the **cluster name** matches `terraweek-eks`.
- Verify **desired node count** is 2 (from our variables).
- Confirm **subnet IDs** are private subnets (from VPC module).
- Ensure IAM roles are being created automatically (we don’t need to write them manually).

## 4. Apply and Connect kubectl
### Apply the Terraform Config
Run:
```bash
terraform apply
```
- Terraform will show the plan again — type `yes` to confirm.
- This takes **10–15 minutes**. EKS cluster creation is slow because AWS provisions control plane, IAM roles, node groups, and networking.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2765).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2768).png)

### Add Outputs (`outputs.tf`)
```hcl
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_region" {
  value = var.region
}
```
- These outputs make it easy to reference cluster details later.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2772).png)

### Update kubeconfig
Once Terraform finishes, run:
```bash
aws eks update-kubeconfig --name terraweek-eks --region ap-south-1
```
- This command writes cluster connection info into your local `~/.kube/config`.
- It allows `kubectl` to talk to our new EKS cluster.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2776).png)

### Verify Cluster Connectivity
Run these commands:
```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2791).png)

- Nodes → 2 nodes in `Ready` state (from your `node_desired_count`).
- Pods → `kube-system` namespace pods like `coredns`, `aws-node`, `kube-proxy` should be running.
- Cluster info → shows API server endpoint and DNS.

### Debugging Tips
Check EKS authentication mode
```bash
aws eks describe-cluster --name terraweek-eks --region ap-south-1 --query "cluster.accessConfig.authenticationMode"
```
- This tells whether the cluster uses CONFIG_MAP, API, or API_AND_CONFIG_MAP. 
- Newer clusters (1.23+) support EKS Access Entries, which is the modern, easier way to grant access.

If mode is `API or API_AND_CONFIG_MAP`: 
```bash
#Add an access entry
aws eks create-access-entry --cluster-name terraweek-eks --region ap-south-1 --principal-arn arn:aws:iam::008971674349:user/Atul-IAM-Admin --type STANDARD

#Then associate an access policy
aws eks associate-access-policy --cluster-name terraweek-eks --region ap-south-1 --principal-arn arn:aws:iam::008971674349:user/Atul-IAM-Admin --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy --access-scope type=cluster
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2778).png)

## 5. Deploy a Workload on the Cluster
Our Terraform-provisioned cluster is live. Deploy something on it.

### Create the Deployment File - `k8s/nginx-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-terraweek
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
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```
- **Deployment** → runs 3 replicas of Nginx pods.
- **Service (LoadBalancer)** → exposes Nginx to the internet via AWS ELB.

### Apply the Manifest
Run:
```bash
kubectl apply -f k8s/nginx-deployment.yaml
```
We should see:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2793).png)

### Watch the LoadBalancer
Run:
```bash
kubectl get svc nginx-service -w
```
- Initially, `EXTERNAL-IP` will show `<pending>`.
- After a few minutes, AWS provisions an ELB and assigns a public IP/DNS name.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2796).png)

### Verify the Deployment
Run these commands:
```bash
kubectl get nodes
kubectl get deployments
kubectl get pods
kubectl get svc
```
- **Nodes** → 2 nodes in `Ready` state.
- **Deployments** → `nginx-terraweek` with 3 replicas.
- **Pods** → 3 pods running.
- **Service** → `nginx-service` with an external IP/DNS.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2802).png)

### Access the Nginx Page
Copy the external IP/DNS from the service and open it in browser:
```Code
http://<external-ip>
```
- We should see the **Nginx welcome page**.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2807).png)

### Debugging Tips
If pods are stuck in `Pending`:
```bash
kubectl describe pods
kubectl get events --sort-by=.metadata.creationTimestamp
```
- If service never gets an external IP:
  - Check AWS console → EC2 → Load Balancers.
  - Ensure VPC subnets have the correct tags (`elb` vs `internal-elb`).
- If page doesn’t load:
  - Confirm security groups allow inbound port 80.
  - Run `kubectl logs <pod-name>` to check Nginx startup.

## 6. Destroy Everything
> **Cost warning:** EKS clusters and NAT Gateways incur charges by the hour. Always clean up when done.

### Delete Kubernetes Workload
First remove the Nginx deployment and service:
```bash
kubectl delete -f k8s/nginx-deployment.yaml
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2812).png)

> We **must** delete the `LoadBalancer` Service before running `terraform destroy`. The Service creates an AWS ELB outside of Terraform's management. If it still exists when Terraform tries to delete the VPC, the destroy will fail — the ELB holds ENIs in the subnets that block VPC deletion.

### Wait for LoadBalancer Cleanup
Run:
```bash
kubectl get svc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2816).png)

- We should see no `nginx-service` anymore.
- Then check in AWS Console:
  - Go to **EC2 → Load Balancers**.
  - Verify the ELB created for Nginx is gone.

### Destroy Terraform Resources
Now destroy everything provisioned by Terraform:
```bash
terraform destroy
```
- Terraform will show the plan again — type `yes` to confirm.
- This takes **10–15 minutes** (similar to cluster creation). Terraform destroys resources in reverse dependency order:
 
```
Workload (manually deleted) → EKS node group → EKS cluster
  → NAT Gateway → Subnets → Route Tables → IGW → VPC
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2822).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/d25387ef402bad24902886f06f33c70c8ec11d7f/2026/day-66/Screenshots/Screenshot%20(2823).png)

### Verify in AWS Console
After destroy finishes, check:
- **EKS clusters** → empty
- **EC2 instances** → no node group instances
- **VPCs** → `terraweek-vpc` deleted
- **NAT Gateways** → deleted
- **Elastic IPs** → released

If Terraform gets stuck:
- Check for leftover **ENIs** (Elastic Network Interfaces) or **security groups** in the VPC.
- Delete them manually in the AWS console, then re‑run `terraform destroy`.

### Debugging Tips
If `terraform destroy` fails:
- Run `terraform plan -destroy` to see what’s blocking.
- Manually delete stuck resources (ENIs, SGs).

Always delete Kubernetes `LoadBalancer` services before destroying the VPC.

### What this project provisioned — full resource map
```
terraform.tfvars
      │
      ▼
module "vpc"                         ← ~20 resources (VPC, subnets, NAT, routes)
      │  vpc_id, private_subnets
      ▼
module "eks"                         ← ~30 resources (cluster, IAM, SGs, nodes)
      │  cluster_name, endpoint
      ▼
aws eks update-kubeconfig
      │
      ▼
kubectl apply nginx-deployment.yaml  ← Deployment + LoadBalancer Service
      │
      ▼
AWS ELB → Nginx welcome page
```