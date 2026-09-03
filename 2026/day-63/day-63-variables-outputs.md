# Terraform - Variables, Outputs, Data Sources and Expressions
A hands-on guide to parameterizing Terraform configs with variables, exposing resource attributes through outputs, reading existing infrastructure with data sources, and using locals and built-in functions for dynamic configuration.
 
## Table of Contents
- [Extract Variables](#1-extract-variables)
- [Variable Files and Precedence](#2-variable-files-and-precedence)
- [Add Outputs](#3-add-outputs)
- [Use Data Sources](#4-use-data-sources)
- [Use Locals for Dynamic Values](#5-use-locals-for-dynamic-values)
- [Built-in Functions and Conditional Expressions](#6-built-in-functions-and-conditional-expressions)

## 1. Extract Variables
Hardcoding values like region, CIDR blocks, and instance types in `main.tf` makes configs inflexible and hard to reuse. Variables externalize these values so the same config can be applied to different environments without editing the source.
 
### `variables.tf`
```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}
 
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}
 
variable "subnet_cidr" {
  type    = string
  default = "10.0.1.0/24"
}
 
variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
  default     = "ami-01a18c38ece67e620"   # Remove before Task-04
}

variable "project_name" {
  type        = string
  description = "Project name — no default, always required"
}
 
variable "environment" {
  type    = string
  default = "dev"
}
 
variable "allowed_ports" {
  type    = list(number)
  default = [22, 80, 443]
}
 
variable "extra_tags" {
  type    = map(string)
  default = {}
}
```
> `project_name` has no default, so Terraform will interactively prompt for it during `terraform plan` unless it is supplied via a `.tfvars` file, `-var` flag, or environment variable.
 
### The five Terraform variable types
| Type | Description | Example |
|---|---|---|
| `string` | Text value | `"ap-south-1"` |
| `number` | Numeric value | `22` |
| `bool` | True or false flag | `true` |
| `list` | Ordered collection | `[22, 80, 443]` |
| `map` | Key-value pairs | `{ Name = "web", Env = "dev" }` |
 
### Refactor `main.tf` to use variables
Replace every hardcoded value with a `var.*` reference:
```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnet_cidr
  map_public_ip_on_launch = true
  tags = {
    Name = "${var.project_name}-subnet"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Route Table
resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "${var.project_name}-rt"
  }
}

# Route Table Association
resource "aws_route_table_association" "assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.rt.id
}

# Security Group
resource "aws_security_group" "sg" {
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.allowed_ports
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
    Name = "${var.project_name}-sg"
  }
}

# EC2 Instance
resource "aws_instance" "main" {
  ami                         = var.ami_id   # will later be replaced with data source
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.sg.id]
  associate_public_ip_address = true

  tags = {
    Name = "${var.project_name}-server"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Random suffix to ensure unique bucket name
resource "random_id" "suffix" {
  byte_length = 4
}

# S3 Bucket for logs
resource "aws_s3_bucket" "logs" {
  bucket = "${var.project_name}-logs-${random_id.suffix.hex}"

  depends_on = [aws_instance.main]

  tags = {
    Name = "${var.project_name}-logs"
  }
}
```
 
```bash
terraform plan
# Prompts for project_name (no default set)
# All other values use their defaults
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2265).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2267).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2268).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2270).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2272).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2274).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2276).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2278).png)

## 2. Variable Files and Precedence
### Create `terraform.tfvars`
This file is automatically loaded by Terraform if present. Create it in our project root:
```hcl
project_name = "terraweek"
environment  = "dev"
instance_type = "t2.micro"
```
- This sets defaults for our dev environment.

### Create `prod.tfvars`
This file is for production overrides. Create it alongside `terraform.tfvars`:
```hcl
project_name = "terraweek"
environment  = "prod"
instance_type = "t3.small"
vpc_cidr     = "10.1.0.0/16"
subnet_cidr  = "10.1.1.0/24"
```
- Notice how CIDR blocks and instance type differ from dev.

### Apply with Default File
```bash
terraform plan
```
- Terraform automatically loads `terraform.tfvars`.
- We’ll see a plan with `environment = dev` and `instance_type = t2.micro`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2280).png)

### Apply with Prod File
```bash
terraform plan -var-file="prod.tfvars"
```
- This overrides defaults with production values.
- We’ll see `environment = prod`, `instance_type = t3.small`, and new CIDRs.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2283).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2286).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2288).png)

### Override with CLI
```bash
terraform plan -var="instance_type=t2.nano"
```
- CLI flags always win.
- Even if `prod.tfvars` says `t3.small`, this forces `t2.nano`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2290).png)

### Override with Environment Variable
```bash
export TF_VAR_environment="staging"
terraform plan
```
- Environment variables override defaults, but **not tfvars**.
- If `terraform.tfvars` sets `dev`, it stays `dev`.
- If no tfvars file sets `environment`, then `staging` is used.

### Variable precedence — lowest to highest
| Priority | Source | Notes |
|---|---|---|
| 1 (lowest) | Default values in `variables.tf` | Used when nothing else is set |
| 2 | `terraform.tfvars` | Auto-loaded if present in the working directory |
| 3 | `*.auto.tfvars` | Auto-loaded, alphabetical order |
| 4 | `-var-file` flag | Explicitly named file |
| 5 | `-var` flag | CLI override |
| 6 (highest) | `TF_VAR_*` environment variables | Overrides defaults, but **not** tfvars files |
> A common gotcha: `TF_VAR_*` environment variables do **not** override `.tfvars` files. If `terraform.tfvars` sets `environment = "dev"`, that takes precedence over `export TF_VAR_environment="staging"`.

## 3. Add Outputs
Outputs expose resource attributes after `terraform apply` — making values like public IPs and resource IDs available for display, scripting, and inter-module references.

### Create `outputs.tf`
In our project root, add a new file called `outputs.tf` with the following:
```hcl
output "vpc_id" {
  description = "The VPC ID"
  value       = aws_vpc.main.id
}

output "subnet_id" {
  description = "The public subnet ID"
  value       = aws_subnet.public.id
}

output "instance_id" {
  description = "The EC2 instance ID"
  value       = aws_instance.main.id
}

output "instance_public_ip" {
  description = "The public IP of the EC2 instance"
  value       = aws_instance.main.public_ip
}

output "instance_public_dns" {
  description = "The public DNS name of the EC2 instance"
  value       = aws_instance.main.public_dns
}

output "security_group_id" {
  description = "The security group ID"
  value       = aws_security_group.sg.id
}
```
- Each output block references the resource created in our `main.tf`. Adjust names (`aws_vpc.main`, `aws_subnet.public`, etc.) to match our actual resource names.

### Apply Your Config
```bash
terraform apply
```
- Terraform will provision our resources.
- At the end of the apply, we’ll see the outputs printed automatically.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2294).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2295).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2296).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2297).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2299).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2301).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2302).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2303).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2304).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2305).png)

### Verify Outputs
After apply, test the following commands:
```bash
terraform output
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2307).png)

Shows all outputs.
```bash
terraform output instance_public_ip
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2309).png)

Shows only the public IP.
```bash
terraform output -json
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2312).png)

Shows outputs in JSON format (useful for scripting or CI/CD pipelines).

### When outputs are useful
| Use case | Example |
|---|---|
| Display after deploy | Print the public IP so we can SSH immediately |
| Pass to scripts | `IP=$(terraform output -raw instance_public_ip)` |
| Inter-module references | A networking module exposes `vpc_id` for other modules to consume |
| CI/CD pipelines | Parse JSON output to drive downstream steps |

## 4. Use Data Sources
A **data source** reads existing information from AWS (or another provider) without creating anything. This keeps our config dynamic and region-portable — no more hardcoded AMI IDs that break when we change regions.
 
### Data source vs resource
| | `resource` | `data` source |
|---|---|---|
| Creates infrastructure | ✓ Yes | ✗ No |
| Modifies infrastructure | ✓ Yes | ✗ No |
| Read-only | ✗ No | ✓ Yes |
| Tracked in state | ✓ Yes | Cached only |
| Syntax | `resource "type" "name"` | `data "type" "name"` |

### Add AMI Data Source `data.tf`
Instead of hardcoding an AMI ID, fetch the latest Amazon Linux 2 image dynamically:
```hcl
# Fetch latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}
```
- This ensures we always get the latest Amazon Linux 2 AMI in any region.

### Replace Hardcoded AMI
Update our EC2 resource in `main.tf`:
```hcl
# EC2 Instance
resource "aws_instance" "main" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.sg.id]
  associate_public_ip_address = true

  tags = {
    Name = "${var.project_name}-server"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### Add Availability Zones Data Source `data.tf`
Fetch available AZs in our region:
```hcl
# Fetch available AZs
data "aws_availability_zones" "available" {}
```
Use the first AZ for our subnet:
```hcl
# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnet_cidr
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "${var.project_name}-subnet"
  }
}
```
- Now our subnet automatically adapts to whichever region we deploy in.

### Apply and Verify
Run:
```bash
terraform apply
```
- Terraform will fetch the latest AMI and AZ dynamically.
- No more broken configs when we change regions.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2315).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2317).png)

Verify:
```bash
terraform output instance_public_ip
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2320).png)

- Check that the instance boots with Amazon Linux 2 and is placed in a valid AZ.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2322).png)

## 5. Use Locals for Dynamic Values
Locals are computed values derived from variables or expressions. They are evaluated once and reused throughout the config — ideal for consistent naming and tagging patterns.

### Add a `locals` Block
Create a new block in our config (we can put it in `main.tf` or a separate `locals.tf`):
```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```
- `name_prefix` automatically combines project + environment (e.g., `terraweek-dev`).
- `common_tags` ensures every resource has consistent metadata.

### Replace Hardcoded Name Tags
Update our resources to use `local.name_prefix` instead of hardcoded strings:
```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnet_cidr
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[0]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-subnet"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-igw"
  })
}

# Route Table
resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-rt"
  })
}

# Route Table Association
resource "aws_route_table_association" "assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.rt.id
}

# Security Group
resource "aws_security_group" "sg" {
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.allowed_ports
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

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-sg"
  })
}

# EC2 Instance
resource "aws_instance" "main" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.sg.id]
  associate_public_ip_address = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-server"
  })

  lifecycle {
    create_before_destroy = true
  }
}

# Random suffix to ensure unique bucket name
resource "random_id" "suffix" {
  byte_length = 4
}

# S3 Bucket for logs
resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs-${random_id.suffix.hex}"

  depends_on = [aws_instance.main]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-logs"
  })
}
```
- Notice how every resource now has consistent naming (`terraweek-dev-vpc`, `terraweek-dev-subnet`, `terraweek-dev-server`) and shared tags.
- `merge()` combines `common_tags` with the resource-specific `Name` tag. Every resource automatically gets consistent tagging with a single change point.

### Apply and Verify
Run:
```bash
terraform apply
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2327).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2329).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2331).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2332).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2333).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2335).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2341).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2338).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2342).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2343).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2345).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2347).png)

Verify in the AWS console:
| Resource | Expected Name Tag |
|---|---|
| VPC | `terraweek-dev-vpc` |
| Subnet | `terraweek-dev-subnet` |
| EC2 instance | `terraweek-dev-server` |
| All resources | `Project=terraweek`, `Environment=dev`, `ManagedBy=Terraform` |

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2349).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2351).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2354).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2356).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2358).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2360).png)

## 6. Built-in Functions and Conditional Expressions
### Open Terraform Console
Run inside our project directory:
```bash
terraform console
````
- This opens an interactive REPL where we can test expressions before using them in configs.

### Practice String Functions
Type these one by one:
```hcl
upper("terraweek")
# Output: "TERRAWEEK"

join("-", ["terra", "week", "2026"])
# Output: "terra-week-2026"

format("arn:aws:s3:::%s", "my-bucket")
# Output: "arn:aws:s3:::my-bucket"
```
- These help with string manipulation and formatting.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2375).png)

### Practice Collection Functions
```hcl
length(["a", "b", "c"])
# Output: 3

lookup({dev = "t2.micro", prod = "t3.small"}, "dev")
# Output: "t2.micro"

toset(["a", "b", "a"])
# Output: ["a", "b"]
```
- Useful for working with lists and maps.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2378).png)

### Practice Networking Function
```hcl
cidrsubnet("10.0.0.0/16", 8, 1)
# Output: "10.0.1.0/24"
```
- Useful for dynamically carving subnets from a parent CIDR block — increment the last argument to generate `10.0.2.0/24`, `10.0.3.0/24`, etc.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2381).png)

### Add Conditional Expression
In our EC2 resource (`main.tf`):
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.environment == "prod" ? "t3.small" : "t2.micro"
  subnet_id     = aws_subnet.public.id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-server"
  })
}
```
Syntax: `condition ? value_if_true : value_if_false`
| `environment` value | Resolved `instance_type` |
|---|---|
| `"prod"` | `t3.small` |
| anything else | `t2.micro` |

### Apply and Verify
Run:
```bash
# Test with prod values
terraform apply -var-file="prod.tfvars"
# Plan shows instance_type = t3.small
 
# Test with dev values (terraform.tfvars)
terraform apply
# Plan shows instance_type = t2.micro
```
- Check the plan output — the instance type should be `t3.small`.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2383).png)

- Switch back to `terraform.tfvars` (dev) and it should be `t2.micro`.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/87ed0508a7c81de5c9955fe84372235747fe99f6/2026/day-63/Screenshots/Screenshot%20(2386).png)

### Most useful functions - quick reference
| Function | What it does | Real-world use |
|---|---|---|
| `upper` | Converts string to uppercase | Enforce naming conventions |
| `join` | Concatenates list with delimiter | Build composite names or IDs |
| `format` | String template interpolation | Construct ARNs, URLs, policy paths |
| `lookup` | Safely reads a value from a map | Environment-specific config maps |
| `merge` | Combines two or more maps | Consistent tagging across resources |
| `cidrsubnet` | Derives a subnet CIDR from a parent | Dynamic multi-AZ subnet allocation |


### Putting it all together — config evolution across this project
```
Day 62: Hardcoded values in main.tf
      │
      ▼
Day 63 Task 1: Extract to variables.tf
      │
      ▼
Day 63 Task 2: Add tfvars files for env-specific values
      │
      ▼
Day 63 Task 3: Expose outputs for IPs and IDs
      │
      ▼
Day 63 Task 4: Replace hardcoded AMIs with data sources
      │
      ▼
Day 63 Task 5: Add locals for consistent naming/tagging
      │
      ▼
Day 63 Task 6: Add functions and conditional expressions
```
Each step makes the config more reusable, readable, and production-ready.