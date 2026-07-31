# Terraform Modules: Build Reusable Infrastructure
A hands-on guide to understanding module structure, authoring custom EC2 and security group modules, calling them from a root module, using a public registry module, and applying module versioning best practices.
 
## Table of Contents
- [Understand Module Structure](#1-understand-module-structure)
- [Build a Custom EC2 Module](#2-build-a-custom-ec2-module)
- [Build a Custom Security Group Module](#3-build-a-custom-security-group-module)
- [Call Your Modules from Root](#4-call-your-modules-from-root)
- [Use a Public Registry Module](#5-use-a-public-registry-module)
- [Module Versioning and Best Practices](#6-module-versioning-and-best-practices)

## 1. Understand Module Structure
A Terraform module is simply a directory containing `.tf` files. Every Terraform project has at least one module - the **root module**. Child modules live in subdirectories and are called from the root.
 
### Standard project layout
```
terraform-modules/
├── main.tf               # Root module — calls child modules
├── variables.tf          # Root input variables
├── outputs.tf            # Root outputs
├── providers.tf          # Provider configuration
└── modules/
    ├── ec2-instance/
    │   ├── main.tf       # EC2 resource definition
    │   ├── variables.tf
    │   └── outputs.tf
    └── security-group/
        ├── main.tf       # Security group resource definition
        ├── variables.tf
        └── outputs.tf
```
This is the standard layout every Terraform project follows.

### Create Root Directory
Start by creating the main Terraform project folder.
- Run `mkdir terraform-modules`
- Navigate inside with `cd terraform-modules`
- This will hold your root module files and the `modules/` subfolder.

### Add Root Files
These files define the entry point and provider configuration.
- Create empty files: `touch main.tf variables.tf outputs.tf providers.tf`
- `main.tf` will call child modules
- `variables.tf` defines inputs for the root
- `outputs.tf` exposes values
- `providers.tf` configures AWS provider

### Create Modules Subfolder
This folder stores reusable child modules.
- Run `mkdir modules`
- Inside, create two subfolders: `ec2-instance` and `security-group`
- Each subfolder will contain its own `main.tf`, `variables.tf`, and `outputs.tf`

### Scaffold EC2 Module
Prepare the EC2 instance module structure.
- Navigate to `modules/ec2-instance`
- Run `touch main.tf variables.tf outputs.tf`
- These files will later define EC2 resources, inputs, and outputs

### Scaffold Security Group Module
Prepare the Security Group module structure.
- Navigate to `modules/security-group`
- Run `touch main.tf variables.tf outputs.tf`
- These files will later define SG resources, inputs, and outputs

### Verify Layout
Check that your directory tree matches the standard Terraform module layout.
- Run `tree terraform-modules/`
- Confirm you see root files and two child module folders
- This structure is reusable across projects

### Root module vs child module
| | Root Module | Child Module |
|---|---|---|
| Location | Project root (`terraform-modules/`) | `modules/<name>/` subdirectory |
| Entry point | ✓ Yes — `terraform init/plan/apply` runs here | ✗ No — called from root |
| Contains provider config | ✓ Yes | ✗ No |
| Reusable across projects | Generally no | ✓ Yes |
| How it's invoked | Directly by Terraform CLI | `module "name" { source = "./modules/..." }` |
 
The key benefit of this separation: the same EC2 module can create a web server, an API server, and a database host - just by passing different variable values. No copy-pasting.

## 2. Build a Custom EC2 Module
### Create the Directory
Inside your project root (`terraform-modules/`), make a new folder:
```bash
mkdir -p terraform-modules/modules/ec2-instance
cd terraform-modules/modules/ec2-instance
touch main.tf variables.tf outputs.tf
```

### Define Inputs (`variables.tf`)
This file declares what values the module expects when someone calls it.
```hcl
variable "ami_id" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_id" {
  description = "Subnet ID to launch the instance in"
  type        = string
}

variable "security_group_ids" {
  description = "List of security group IDs to attach"
  type        = list(string)
}

variable "instance_name" {
  description = "Name tag for the instance"
  type        = string
}

variable "tags" {
  description = "Additional tags for the instance"
  type        = map(string)
  default     = {}
}
```
- This ensures flexibility: you can reuse the same module for different AMIs, subnets, and SGs.

### Define Resource (`main.tf`)
This is where the EC2 instance is created. Notice how we merge the `Name` tag with any extra tags passed in.
```hcl
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = merge(
    {
      Name = var.instance_name
    },
    var.tags
  )
}
```
`merge()` combines the mandatory `Name` tag with any additional tags passed in by the caller — every instance gets consistent tagging without the caller needing to repeat the Name key.

### Define Outputs (`outputs.tf`)
Outputs expose useful values back to the root module.
```hcl
output "instance_id" {
  description = "The ID of the EC2 instance"
  value       = aws_instance.this.id
}

output "public_ip" {
  description = "The public IP of the EC2 instance"
  value       = aws_instance.this.public_ip
}

output "private_ip" {
  description = "The private IP of the EC2 instance"
  value       = aws_instance.this.private_ip
}
```
> Do not apply yet — write all module files first, then call them from root in Task 4.

## 3. Build a Custom Security Group Module
### Create the Directory
Inside your project root, make the folder and files:
```bash
mkdir -p terraform-modules/modules/security-group
cd terraform-modules/modules/security-group
touch main.tf variables.tf outputs.tf
```

### Define Inputs (`variables.tf`)
These inputs make the module flexible:
```hcl
variable "vpc_id" {
  description = "VPC ID where the security group will be created"
  type        = string
}

variable "sg_name" {
  description = "Name of the security group"
  type        = string
}

variable "ingress_ports" {
  description = "List of ingress ports to allow"
  type        = list(number)
  default     = [22, 80]
}

variable "tags" {
  description = "Additional tags for the security group"
  type        = map(string)
  default     = {}
}
```

### Define Resource (`main.tf`)
Here’s the magic: the dynamic block loops over each port in `ingress_ports` and generates an ingress rule.
```hcl
resource "aws_security_group" "this" {
  name        = var.sg_name
  description = "Managed by Terraform"
  vpc_id      = var.vpc_id

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

  tags = merge(
    {
      Name = var.sg_name
    },
    var.tags
  )
}
```
### How the `dynamic` block works
This is our first time using a `dynamic` block -- it loops over a list to generate repeated nested blocks.

Instead of writing a separate `ingress` block for every port, the `dynamic` block generates them programmatically:
| Expression | Meaning |
|---|---|
| `dynamic "ingress"` | Tells Terraform to generate multiple `ingress` blocks |
| `for_each = var.ingress_ports` | Loops over the list `[22, 80, 443]` |
| `ingress.value` | The current port number in the loop |
 
Calling with `ingress_ports = [22, 80, 443]` generates three ingress rules automatically. Calling with `ingress_ports = [5432]` generates one rule for PostgreSQL. Same module, completely different behavior.

### Define Outputs (`outputs.tf`)
Expose the SG ID so the EC2 module can use it.
```hcl
output "sg_id" {
  description = "The ID of the security group"
  value       = aws_security_group.this.id
}
```

## 4. Call Your Modules from Root
### Root Directory Setup
Make sure you’re inside your root project folder (`terraform-modules/`). You should already have:
```Code
terraform-modules/
  main.tf
  variables.tf
  outputs.tf
  providers.tf
  modules/
    ec2-instance/
    security-group/
```

### Provider Config (`providers.tf`)
Define AWS provider (adjust region if needed):
```hcl
provider "aws" {
  region = "ap-south-1"
}
```

### Create VPC + Subnet (simplified) - `main.tf`
For now, let’s create a basic VPC and subnet directly (later you’ll replace with registry module in Task 5):
```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = {
    Name = "terraweek-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "terraweek-public-subnet"
  }
}
```

### Call Security Group Module
Wire in your SG module:
```hcl
module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = aws_vpc.main.id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = {
    Project = "Terraweek"
    Env     = "Dev"
  }
}
```

### Call EC2 Module Twice
Deploy two EC2 instances using the same module:
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = {
    Project = "Terraweek"
    Env     = "Dev"
  }
}

module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = {
    Project = "Terraweek"
    Env     = "Dev"
  }
}
```

### Root Outputs (`outputs.tf`)
Expose the public IPs:
```hcl
output "web_server_ip" {
  value = module.web_server.public_ip
}

output "api_server_ip" {
  value = module.api_server.public_ip
}
```
### How data flows between modules
```
data.aws_ami.amazon_linux.id
        │
        ▼
module "web_sg"              ← receives vpc_id from aws_vpc.main.id
        │ sg_id
        ▼
module "web_server"          ← receives subnet_id and sg_id
module "api_server"          ← receives subnet_id and sg_id
        │ public_ip
        ▼
root outputs.tf
```

### Step 7: Run & Verify
Initialize modules:
```bash
terraform init
```

Preview resources:
```bash
terraform plan
```

Apply changes:
```bash
terraform apply -auto-approve
```

Verify in AWS Console:
- Two EC2 instances: `terraweek-web` and `terraweek-api`
- Both attached to the same security group
- Both in the public subnet

## 5. Use a Public Registry Module
Instead of building your own VPC from scratch, use the official module from the Terraform Registry.

The [Terraform Registry](https://registry.terraform.io) hosts community and official modules for common infrastructure patterns. Using a registry module replaces your manual VPC/subnet resources with a battle-tested, feature-complete implementation.

### Replace VPC Resource with Registry Module
In your root `main.tf`, remove the manual `aws_vpc` and `aws_subnet` resources. Add this block instead:
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraweek-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = false
  enable_dns_hostnames = true

  tags = {
    Project = "Terraweek"
    Env     = "Dev"
  }
}
```

### Update Module Calls
Now point your EC2 and SG modules to the VPC module outputs:
```hcl
module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = module.vpc.vpc_id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = {
    Project = "Terraweek"
    Env     = "Dev"
  }
}

module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[0]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = {
    Project = "Terraweek"
    Env     = "Dev"
  }
}

module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[0]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = {
    Project = "Terraweek"
    Env     = "Dev"
  }
}
```

### Run Terraform
Initialize (downloads registry module):
```bash
terraform init
```

Preview resources:
```bash
terraform plan
```
Apply changes:
```bash
terraform apply -auto-approve
```

### Manual VPC vs registry VPC module
| | Manual (Task 4) | Registry module |
|---|---|---|
| Resources created | ~2–3 (VPC, subnet) | 15+ (VPC, subnets, route tables, IGW, associations, DHCP options, etc.) |
| Lines of HCL | ~20 | ~15 (the module call) |
| Multi-AZ support | Manual | Built-in |
| NAT Gateway support | Manual | Built-in flag |
| Maintained by | You | Community + HashiCorp |

Run `terraform plan` and check the resource count difference. You’ll see the registry module builds a complete networking setup far beyond your manual config.

Registry modules encode years of best-practice infrastructure patterns. The VPC module alone handles routing, DHCP, DNS, and multi-AZ layout that would take hundreds of lines to write manually.
 
Registry modules are downloaded to `.terraform/modules/` and pinned to the version you specified — ensuring every team member and CI/CD run uses the same code.

## 6. Module Versioning and Best Practices
### Pin Registry Module Versions
When you call a registry module, always **pin the version**. You have three options:
```hcl
# Exact version
version = "5.1.0"

# Any 5.x version
version = "~> 5.0"

# Range
version = ">= 5.0, < 6.0"
```
Without a version pin, `terraform init` may silently upgrade to a newer major version that contains breaking changes — and your infrastructure will behave differently without any change to your code.

### Upgrade Modules
Run this to check for newer versions:
```bash
terraform init -upgrade
```
Re-downloads all modules and providers, respecting your version constraints. Run this intentionally when you want to pick up new features or patches.

### Inspect State
List all resources in state:
```bash
terraform state list
```
Resources created by modules appear with their module path as a prefix:
```
module.vpc.aws_vpc.this
module.vpc.aws_subnet.public[0]
module.web_sg.aws_security_group.this
module.web_server.aws_instance.this
module.api_server.aws_instance.this
```
This makes it easy to identify which module owns each resource when debugging or performing state surgery.

### Destroy Everything
When you’re done, clean up:
```bash
terraform destroy -auto-approve
```
Destroys all resources managed by your root and child modules in reverse dependency order.
 
### Five module best practices
| Practice | Why it matters |
|---|---|
| **Pin versions for registry modules** | Prevents silent upgrades from breaking your infrastructure |
| **Keep modules focused** | One concern per module (EC2, SG, VPC) — easier to test, reuse, and reason about |
| **Use variables for all configurable values** | Never hardcode AMI IDs, CIDRs, or instance types inside a module |
| **Define outputs for every useful attribute** | Callers need IDs and IPs to wire modules together and build outputs |
| **Add a `README.md` to every custom module** | Document inputs, outputs, usage examples, and any required permissions |
