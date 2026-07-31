# TerraWeek Capstone: Multi-Environment Infrastructure with Workspaces and Modules
A capstone project that brings together Terraform workspaces, custom modules, environment-specific variable files, and workspace-aware locals to deploy isolated dev, staging, and production infrastructure from a single codebase.
 
## Table of Contents
- [Learn Terraform Workspaces](#1-learn-terraform-workspaces)
- [Set Up the Project Structure](#2-set-up-the-project-structure)
- [Build the Custom Modules](#3-build-the-custom-modules)
- [Wire It All Together with Workspace-Aware Config](#4-wire-it-all-together-with-workspace-aware-config)
- [Deploy All Three Environments](#5-deploy-all-three-environments)
- [Terraform Best Practices Guide](#6-terraform-best-practices-guide)
- [Destroy All Environments](#7-destroy-all-environments)

## 1. Learn Terraform Workspaces
A Terraform workspace is an isolated state environment within a single project directory. Instead of maintaining separate folders for dev, staging, and prod - each with duplicated code - workspaces let you use one codebase with independent state files per environment.

### Create Project Directory
Start by creating a new folder for your capstone project.
```bash
mkdir terraweek-capstone && cd terraweek-capstone
```
- This keeps all files organized in one place.

### Initialize Terraform
Initialize Terraform to set up the working directory.
```bash
terraform init
```
- Downloads provider plugins
- Creates `.terraform/` folder for state and configs

### Check Current Workspace
Verify which workspace you are currently in.
```bash
terraform workspace show
```
- Default workspace is always `default`
- This is where state is stored initially

### Create New Workspaces
Add separate workspaces for dev, staging, and prod.
```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
```
- Each creates its own isolated state file

### List All Workspaces
Confirm that all workspaces exist.
```bash
terraform workspace list
```
- We should see: `default`, `dev`, `staging`, `prod`
- Active workspace marked with `*`

### Switch Between Workspaces
Move between environments easily.
```bash
terraform workspace select dev
terraform workspace select staging
terraform workspace select prod
```
- Each switch changes the active state context

### What does `terraform.workspace` return inside a config?
Inside any `.tf` file, `terraform.workspace` returns the name of the active workspace as a string:
```hcl
local.environment = terraform.workspace   # "dev", "staging", or "prod"
``` 
This single value drives environment-specific naming, tagging, and sizing across every resource. We can use this in locals, tags, or resource names to make configs environment-aware.

### Where does each workspace store its state file?  
Locally: `.terraform/terraform.tfstate.d/<workspace>/terraform.tfstate`<br>
With remote backends (like S3): `env:/<workspace>/terraform.tfstate` — each workspace gets its own isolated state file.

### How is this different from using separate directories per environment?
**Workspaces vs Separate Directories**
| Dimension | Workspaces | Separate Directories |
|---|---|---|
| Code duplication | None - single codebase | High - code copied per env |
| State isolation | ✓ One state file per workspace | ✓ Separate states |
| Consistency | Easy to keep environments in sync | Risk of drift between copies |
| Maintenance | Single place to make changes | Changes must be applied to all dirs |
| Environment awareness | `terraform.workspace` variable | Hardcoded per directory |

## 2. Set Up the Project Structure
### Directory layout
```
terraweek-capstone/
├── main.tf             # Root module — calls child modules
├── variables.tf        # Root input variables
├── outputs.tf          # Root outputs
├── providers.tf        # AWS provider and backend
├── locals.tf           # Workspace-aware locals and common tags
├── dev.tfvars          # Dev environment values
├── staging.tfvars      # Staging environment values
├── prod.tfvars         # Prod environment values
├── .gitignore          # Ignore state, .terraform, secrets
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── security-group/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2-instance/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Create the root project folder
```bash
mkdir terraweek-capstone && cd terraweek-capstone
```
- This is your root directory where all Terraform configs will live.

### Create the root-level files
```bash
touch main.tf variables.tf outputs.tf providers.tf locals.tf
touch dev.tfvars staging.tfvars prod.tfvars
touch .gitignore
```
- **main.tf** → orchestrates modules (calls VPC, SG, EC2).
- **variables.tf** → defines input variables.
- **outputs.tf** → defines outputs (e.g., instance IP).
- **providers.tf** → configures AWS provider + backend.
- **locals.tf** → defines workspace-aware locals and tags.
- **tfvars files** → environment-specific values (dev, staging, prod).
- **.gitignore** → prevents committing sensitive files/state.

### Create the modules folder
```bash
mkdir -p modules/vpc modules/security-group modules/ec2-instance
touch modules/vpc/main.tf modules/vpc/variables.tf modules/vpc/outputs.tf
touch modules/security-group/main.tf modules/security-group/variables.tf modules/security-group/outputs.tf
touch modules/ec2-instance/main.tf modules/ec2-instance/variables.tf modules/ec2-instance/outputs.tf
```
Each module is self-contained:
- **VPC module** → networking (VPC, subnet, IGW, routes).
- **Security group module** → firewall rules.
- **EC2 module** → compute instances.

### Add the `.gitignore`
Open `.gitignore` and paste:
```gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
.terraform.lock.hcl
```
| Entry | Reason |
|---|---|
| `.terraform/` | Provider binaries — large, machine-specific, regenerated by `init` |
| `*.tfstate` | Contains sensitive resource IDs; managed by backend, not Git |
| `*.tfvars` | May contain secrets — never commit |
| `.terraform.lock.hcl` | Avoids version conflicts across team members |

### Why This File Structure Is Best Practice
| Principle | Implementation |
|---|---|
| Separation of concerns | Each file has one role (providers, vars, outputs, main, locals) |
| Reusability | Modules encapsulate one responsibility and can be reused across projects |
| Environment isolation | Workspaces + `.tfvars` files separate environments without duplicating code |
| Security | `.gitignore` prevents leaking state and secrets |
| Scalability | As infra grows, modules can be versioned, extended, or swapped with registry modules |
| Maintainability | Teams know exactly where to find providers, variables, and module calls |
| Professional polish | Mirrors how infra teams structure production repos |

## 3. Build the Custom Modules
Create three focused modules:
### Module 1: VPC
`modules/vpc/variables.tf`
```hcl
variable "cidr" {
  type = string
}

variable "public_subnet_cidr" {
  type = string
}

variable "environment" {
  type = string
}

variable "project_name" {
  type = string
}
```

`modules/vpc/main.tf`
```hcl
resource "aws_vpc" "this" {
  cidr_block = var.cidr
  tags = {
    Name        = "${var.project_name}-${var.environment}-vpc"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnet_cidr
  map_public_ip_on_launch = true
  tags = {
    Name        = "${var.project_name}-${var.environment}-subnet"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.this.id
  tags = {
    Name        = "${var.project_name}-${var.environment}-igw"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name        = "${var.project_name}-${var.environment}-rt"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

`modules/vpc/outputs.tf`
```hcl
output "vpc_id" {
  value = aws_vpc.this.id
}

output "subnet_id" {
  value = aws_subnet.public.id
}
```

### Module 2: Security Group

`modules/security-group/variables.tf`
```hcl
variable "vpc_id" {
  type = string
}

variable "ingress_ports" {
  type = list(number)
}

variable "environment" {
  type = string
}

variable "project_name" {
  type = string
}
```

`modules/security-group/main.tf`
```hcl
resource "aws_security_group" "this" {
  vpc_id = var.vpc_id
  name   = "${var.project_name}-${var.environment}-sg"

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-${var.environment}-sg"
    Environment = var.environment
    Project     = var.project_name
  }
}
```

`modules/security-group/outputs.tf`
```hcl
output "sg_id" {
  value = aws_security_group.this.id
}
```

### Module 3: EC2 Instance

`modules/ec2-instance/variables.tf`
```hcl
variable "ami_id" {
  type = string
}

variable "instance_type" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "security_group_ids" {
  type = list(string)
}

variable "environment" {
  type = string
}

variable "project_name" {
  type = string
}
```

`modules/ec2-instance/main.tf`
```hcl
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = {
    Name        = "${var.project_name}-${var.environment}-server"
    Environment = var.environment
    Project     = var.project_name
  }
}
```

`modules/ec2-instance/outputs.tf`
```hcl
output "instance_id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
```

### Validation
After writing each module, run:
```bash
terraform validate
```
- This checks syntax and ensures the modules are valid Terraform configs.

## 4. Wire It All Together with Workspace-Aware Config
In the root module, use `terraform.workspace` to drive environment-specific behavior.

### Define locals
`locals.tf`
```hcl
locals {
  environment = terraform.workspace
  name_prefix = "${var.project_name}-${local.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = local.environment
    ManagedBy   = "Terraform"
    Workspace   = terraform.workspace
  }
}
```
- `terraform.workspace` → automatically injects the current workspace (`dev`, `staging`, `prod`).
- `common_tags` → ensures every resource is tagged consistently.

### Define root variables
`variables.tf`
```hcl
variable "project_name" {
  type    = string
  default = "terraweek"
}

variable "vpc_cidr" {
  type = string
}

variable "subnet_cidr" {
  type = string
}

variable "instance_type" {
  type = string
}

variable "ingress_ports" {
  type    = list(number)
  default = [22, 80]
}
```

### Wire modules together
`main.tf`
```hcl
module "vpc" {
  source            = "./modules/vpc"
  cidr              = var.vpc_cidr
  public_subnet_cidr = var.subnet_cidr
  environment       = local.environment
  project_name      = var.project_name
}

module "security_group" {
  source       = "./modules/security-group"
  vpc_id       = module.vpc.vpc_id
  ingress_ports = var.ingress_ports
  environment  = local.environment
  project_name = var.project_name
}

module "ec2_instance" {
  source            = "./modules/ec2-instance"
  ami_id            = "ami-0c55b159cbfafe1f0" # Example Amazon Linux 2 AMI
  instance_type     = var.instance_type
  subnet_id         = module.vpc.subnet_id
  security_group_ids = [module.security_group.sg_id]
  environment       = local.environment
  project_name      = var.project_name
}
```

### Define outputs
`outputs.tf`
```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "subnet_id" {
  value = module.vpc.subnet_id
}

output "sg_id" {
  value = module.security_group.sg_id
}

output "instance_id" {
  value = module.ec2_instance.instance_id
}

output "public_ip" {
  value = module.ec2_instance.public_ip
}
```

### Environment-specific tfvars
`dev.tfvars`
```hcl
vpc_cidr      = "10.0.0.0/16"
subnet_cidr   = "10.0.1.0/24"
instance_type = "t2.micro"
ingress_ports = [22, 80]
```

`staging.tfvars`
```hcl
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
instance_type = "t2.small"
ingress_ports = [22, 80, 443]
```

`prod.tfvars`
```hcl
vpc_cidr      = "10.2.0.0/16"
subnet_cidr   = "10.2.1.0/24"
instance_type = "t3.small"
ingress_ports = [80, 443]
```

### Environment differences at a glance
| Setting | Dev | Staging | Prod |
|---|---|---|---|
| VPC CIDR | `10.0.0.0/16` | `10.1.0.0/16` | `10.2.0.0/16` |
| Instance type | `t2.micro` | `t2.small` | `t3.small` |
| Open ports | 22, 80 | 22, 80, 443 | 80, 443 |
| SSH access | ✓ Yes | ✓ Yes | ✗ No |
| EC2 Name tag | `terraweek-dev-server` | `terraweek-staging-server` | `terraweek-prod-server` |
 
Non-overlapping CIDRs ensure the three VPCs could even be peered in the future without address conflicts.

## 5. Deploy All Three Environments
Deploy each environment using its workspace and tfvars file:
### Deploy Dev Environment
```bash
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
```
- This provisions a **VPC (10.0.0.0/16)**, subnet, SG (ports 22 + 80), and a **t2.micro EC2 instance**.
- Tags: `terraweek-dev-server`.

### Deploy Staging Environment
```bash
terraform workspace select staging
terraform plan -var-file="staging.tfvars"
terraform apply -var-file="staging.tfvars"
```
- Provisions **VPC (10.1.0.0/16)**, subnet, SG (ports 22, 80, 443), and a **t2.small EC2 instance**.
- Tags: `terraweek-staging-server`.

### Deploy Prod Environment
```bash
terraform workspace select prod
terraform plan -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars"
```
- Provisions **VPC (10.2.0.0/16)**, subnet, SG (ports 80, 443 only — no SSH), and a **t3.small EC2 instance**.
- Tags: `terraweek-prod-server`.

### Verification
**Terraform Outputs**

Run for each workspace:
```bash
terraform workspace select dev && terraform output
terraform workspace select staging && terraform output
terraform workspace select prod && terraform output
```
We should see:
- VPC IDs
- Subnet IDs
- Security Group IDs
- Instance IDs
- Public IPs

### Expected state in AWS Console
| Resource | Dev | Staging | Prod |
|---|---|---|---|
| VPC | `10.0.0.0/16` | `10.1.0.0/16` | `10.2.0.0/16` |
| EC2 instance type | `t2.micro` | `t2.small` | `t3.small` |
| EC2 Name tag | `terraweek-dev-server` | `terraweek-staging-server` | `terraweek-prod-server` |
| SSH (port 22) | ✓ Open | ✓ Open | ✗ Closed |

### Are all three environments completely isolated from each other?
- Each environment has its **own VPC** → no overlap in CIDRs.
- Each EC2 instance is in its **own subnet** → no accidental cross-talk.
- Security groups differ per environment (dev allows SSH, prod does not).
- State files are isolated per workspace → no risk of overwriting resources.

Answer: Yes, all three environments are **completely isolated** from each other.

## 6. Terraform Best Practices Guide
### File Structure
- **Separate files** for providers, variables, outputs, main, and locals.
- Keeps code modular, readable, and easier to maintain.
- Modules encapsulate one responsibility each (networking, security, compute).

### State Management
- **Remote backend** (e.g., S3 + DynamoDB) for collaboration.
- Enable **locking** to prevent race conditions.
- Enable **versioning** to recover from accidental changes.

### Variables
- **Never hardcode** values.
- Use `.tfvars` per environment (dev, staging, prod).
- Add **validation blocks** to catch invalid inputs early.

### Modules
- **Single concern per module** (VPC, SG, EC2).
- Always define **inputs/outputs** clearly.
- Pin registry module versions for reproducibility.

### Workspaces
- Use **workspaces** for environment isolation.
- Reference `terraform.workspace` in configs for dynamic naming and tagging.
- Avoid duplicating code across directories.

### Security
- Add `.gitignore` to exclude state and tfvars.
- Encrypt state at rest (e.g., S3 bucket encryption).
- Restrict backend access with IAM policies.

### Commands
- Always run `terraform plan` before `apply`.
- Use `terraform fmt` for consistent formatting.
- Run `terraform validate` before committing.

### Tagging
- **Tag every resource** with:
    - Project
    - Environment
    - ManagedBy = Terraform
- Helps with cost allocation, auditing, and cleanup.

### Naming
- Consistent prefix pattern:<br>
`<project>-<environment>-<resource>`
- Example: `terraweek-dev-server`, `terraweek-prod-vpc`.

### Cleanup
- Always `terraform destroy` non-production environments when not in use.
- Prevents unnecessary costs and keeps AWS accounts clean.

This guide reflects the professional standards used by infrastructure teams at scale.

## 7. Destroy All Environments
Destroy in reverse order - prod first, then staging, then dev - to avoid any dependency confusion.

### Destroy Prod Environment
```bash
terraform workspace select prod
terraform destroy -var-file="prod.tfvars"
```
- Tears down the **prod VPC, subnet, SG, and EC2 instance**.
- Since prod is the most restrictive (no SSH), you remove it first to avoid confusion.

### Destroy Staging Environment
```bash
terraform workspace select staging
terraform destroy -var-file="staging.tfvars"
```
- Removes the **staging VPC, subnet, SG, and EC2 instance**.
- This was your middle-tier environment (`t2.small`).

### Destroy Dev Environment
```bash
terraform workspace select dev
terraform destroy -var-file="dev.tfvars"
```
- Cleans up the **dev VPC, subnet, SG, and EC2 instance**.
- This was your smallest environment (`t2.micro`).

### Verify in AWS Console
Check in the **AWS VPC console** and **EC2 console**:
- No custom VPCs remain (only the default VPC should exist).
- No EC2 instances running.
- No custom security groups or internet gateways left.

### Delete Workspaces
Switch back to default and delete the others:
```bash
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```
- You cannot delete the workspace you’re currently on, so always select `default` first.
- After deletion, only `default` remains.

### Final Verification
Run:
```bash
terraform workspace list
```
-  Should show only `default`.
- In AWS console, confirm **no leftover resources** (VPCs, EC2s, SGs, IGWs).
- Your AWS account is now completely clean.

### Key Reminders
- **Reverse order cleanup** prevents dependency conflicts.
- **Workspace deletion** ensures no stray state files.
- Always verify in AWS console — Terraform may miss resources if they were created manually outside Terraform.
- This step demonstrates professional discipline: infra teams always clean up non-prod environments to save costs.

### What this capstone covered — concept map
```
Workspaces (Task 1)
      │
      ▼
Project structure + .gitignore (Task 2)
      │
      ▼
Custom modules: VPC + SG + EC2 (Task 3)
      │
      ▼
Workspace-aware locals + tfvars wiring (Task 4)
      │
      ▼
terraform workspace select dev/staging/prod
terraform apply -var-file="<env>.tfvars"  (Task 5)
      │
      ▼
Best practices codified (Task 6)
      │
      ▼
terraform destroy + workspace delete (Task 7)
```

Every concept from Days 61–66 (variables, outputs, state, modules, remote backends, data sources) is applied here in a single production-style workflow.