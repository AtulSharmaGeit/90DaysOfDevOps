# Terraform State Management and Remote Backends
A hands-on guide to inspecting and understanding the state file, migrating to a remote S3 backend with DynamoDB locking, importing existing resources, performing state surgery, and detecting and fixing state drift.
 
 
## Table of Contents
- [Inspect Your Current State](#1-inspect-your-current-state)
- [Set Up S3 Remote Backend](#2-set-up-s3-remote-backend)
- [Test State Locking](#3-test-state-locking)
- [Import an Existing Resource](#4-import-an-existing-resource)
- [State Surgery — mv and rm](#5-state-surgery--mv-and-rm)
- [Simulate and Fix State Drift](#6-simulate-and-fix-state-drift)

## 1. Inspect Your Current State
### Apply your config
Make sure our infrastructure is created and Terraform has a state file.
```bash
terraform apply
```
Confirm all resources are provisioned in AWS before exploring the state.

### View the full state
Run:
```bash
terraform show
```
- This prints the entire state in human-readable format.
- We’ll see every resource Terraform knows about, with all attributes (IDs, tags, networking details, etc.).

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2388).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2389).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2390).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2391).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2393).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2394).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2395).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2396).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2397).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2398).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2399).png)

### List all resources tracked
Run:
```bash
terraform state list
```
- This shows a concise list of resource addresses Terraform is managing.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2400).png)

- Answer: Count the lines — that’s how many resources Terraform tracks.

### Inspect a specific resource
Run:
```bash
terraform state show aws_instance.web
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2403).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2404).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2406).png)

This prints every attribute Terraform knows about the EC2 instance:
- Instance ID, AMI, instance type
- Private/public IPs
- Subnet ID, VPC ID
- Security groups
- Tags
- Block device mappings
- Monitoring, tenancy, etc.

Notice: We only defined a few attributes in `.tf` (AMI, instance type, tags), but Terraform stores dozens more returned by AWS.

Do the same for the VPC:
```bash
terraform state show aws_vpc.main
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2407).png)

- We’ll see CIDR block, DNS settings, tags, etc.

### Inspect the raw state file
Open `terraform.tfstate` in a text editor:
```bash
cat terraform.tfstate   # or open in VS Code
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2409).png)

Look for:
```json
"serial": 12,
```
The `serial` is an incrementing counter — it increases every time Terraform updates the state. It acts as a version stamp: when two processes try to update state simultaneously, Terraform compares serials to detect conflicts and reject the stale write.

### How many resources does Terraform track?  
Terraform tracks every resource defined in our `.tf` files plus any dependencies created automatically. For a simple config with a VPC and EC2 instance, we’ll typically see:
- `aws_vpc.main`
- `aws_instance.web`
- `aws_security_group` (if defined)
- `aws_internet_gateway` / `aws_subnet` (if included)<br>
So usually **2–5 resources** depending on our config.

### What attributes does the state store for an EC2 instance?  
We only defined a handful of attributes in our `.tf` files (AMI, instance type, tags), but Terraform stores every attribute returned by the AWS API:
| Category | Examples |
|---|---|
| Identity | Instance ID, ARN |
| Compute | AMI, instance type, key pair |
| Networking | Private/public IPs, subnet ID, VPC ID, security groups |
| Storage | Block device mappings, EBS volume details |
| Metadata | Tenancy, monitoring, lifecycle state, tags |

Essentially, Terraform records **all attributes returned by AWS**, not just the ones we declared.

### Local state — why it is dangerous
| Risk | Consequence |
|---|---|
| File accidentally deleted | All state is lost — Terraform loses track of everything it created |
| No concurrent access control | Two people running `apply` simultaneously corrupts the state |
| Sensitive data in plaintext | Resource IDs, IPs, and sometimes secrets stored unencrypted |
| No version history | Cannot roll back to a previous known-good state |
 
This is why remote backends exist — and why no production team uses local state.

## 2. Set Up S3 Remote Backend
Storing state locally is dangerous -- one deleted file and we lose everything. Time to move it to S3.

The standard production backend for AWS is **S3 + DynamoDB**: S3 stores the state file with versioning, DynamoDB provides distributed locking to prevent concurrent writes.

### Create S3 Bucket
Provision a dedicated bucket to store Terraform state.
```bash
aws s3api create-bucket \
  --bucket terraweek-state-<yourname> \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2410).png)

- Confirm bucket creation in AWS Console → S3

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2416).png)

### Enable Versioning
Turn on versioning as it allows us to recover a previous state file if the current one is corrupted or a bad apply needs to be rolled back.
```bash
aws s3api put-bucket-versioning \
  --bucket terraweek-state-<yourname> \
  --versioning-configuration Status=Enabled
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2419).png)

- Check in AWS Console → S3 → Properties → Versioning

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2421).png)

### Create DynamoDB Table
Set up a table for state locking to prevent concurrent writes.
```bash
aws dynamodb create-table \
  --table-name terraweek-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```
| Field | Value |
|---|---|
| Table name | `terraweek-state-lock` |
| Primary key | `LockID` (string) |
| Billing mode | PAY_PER_REQUEST (no capacity planning needed) |

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2424).png)

### Add Backend Block
Configure Terraform to use **S3 + DynamoDB** backend.

In `main.tf` add:
```hcl
terraform {
  backend "s3" {
    bucket         = "terraweek-state-atulsh"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}
```
| Field | Purpose |
|---|---|
| `bucket` | The S3 bucket that stores the state file. Ensure bucket name matches our created bucket |
| `key` | The path inside the bucket (enables multi-environment layout) |
| `dynamodb_table` | The table used for distributed locking |
| `encrypt` | Encrypts state at rest using AWS-managed keys |

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
  instance_type               = var.environment == "prod" ? "t3.small" : "t2.micro"
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

# Terraform Backend
terraform {
  backend "s3" {
    bucket         = "terraweek-state-atulsh"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}
```
### Initialize Backend
Migrate local state to remote backend.
```bash
terraform init
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2426).png)

- Terraform will ask: Do you want to copy existing state to the new backend? --> Answer **yes**
- This moves our local state file into S3

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2499).png)

### Verify the migration
- **S3 console** → bucket → object at `dev/terraform.tfstate`
- **Local** → `terraform.tfstate` should be empty or gone
- **Confirm** → `terraform plan` should show `No changes`

### Local vs Remote Backend — Comparison
| | Local state | S3 + DynamoDB |
|---|---|---|
| Location | `terraform.tfstate` on disk | S3 object |
| Team access | Single user only | Shared across the team |
| Concurrent writes | No protection | Locked via DynamoDB |
| Version history | None | S3 versioning |
| Encryption | None | AES-256 at rest |
| Suitable for production | ✗ No | ✓ Yes |

## 3. Test State Locking
State locking prevents two simultaneous `terraform apply` runs from corrupting the state. DynamoDB acts as a distributed mutex — only one process can hold the lock at a time.

### Open Two Terminals
- Navigate to the same Terraform project directory in **Terminal 1** and **Terminal 2**.
- Both terminals must point to the same working directory where our `main.tf` and backend config are located.

### Run `terraform apply` in Terminal 1
```bash
terraform apply
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2429).png)

- Terraform will refresh state and wait for our confirmation (yes).
- At this point, Terraform acquires a lock in DynamoDB to prevent other processes from modifying the state.

### Run `terraform plan` in Terminal 2
```bash
terraform plan
```
- Since Terminal 1 is holding the lock, Terminal 2 will fail with an error message.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2432).png)

### What is the error message? Why is locking critical for team environments?
- **Error Message:** Terraform reports `Error acquiring the state lock` with details including `Lock ID`, operation type, and timestamp.

**Why Locking Is Critical:**
| Without locking | With DynamoDB locking |
|---|---|
| Two applies run simultaneously | Second apply blocked until first completes |
| State file gets corrupted | State remains consistent |
| Resources may be double-created or half-deleted | Operations are serialized safely |
| No visibility into who is running what | Lock info shows who holds the lock and when |

### Handle Stale Locks (if needed)
If something crashes and the lock isn’t released:
```bash
terraform force-unlock <LOCK_ID>
```
- Replace <LOCK_ID> with the value shown in the error message.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2436).png)

> Use `force-unlock` only when we are certain no other Terraform process is actively running. Forcing an unlock while an apply is in progress will lead to the corruption we were trying to prevent.

## 4. Import an Existing Resource
Not everything starts with Terraform. Sometimes resources already exist in AWS and we need to bring them under Terraform management.

### Manually Create the Resource
- Go to the **AWS Console → S3 → Create bucket**.
- Name it: `terraweek-import-test-<yourname>`.
- Leave defaults for now — just ensure the bucket exists.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2438).png)

### Add Resource Block in Terraform
In our Terraform config (`main.tf` or a new file), add:
```hcl
resource "aws_s3_bucket" "imported" {
  bucket = "terraweek-import-test-<yourname>"
}
```
- Keep it minimal — only the bucket name. Don’t add ACLs, versioning, or tags yet.

### Import the Resource
Run:
```bash
terraform import aws_s3_bucket.imported terraweek-import-test-<yourname>
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2441).png)

- Terraform will connect to AWS, fetch the bucket’s attributes, and add it to our **state file**.
- This does **not** change the bucket — it only updates Terraform’s state.

### Verify with `terraform plan`
Run:
```bash
terraform plan
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2444).png)

- If we see **“No changes”** → Import was perfect.
- If we see **changes** → Our config doesn’t match reality. For example, AWS may have default settings (like private ACL or no versioning).
    - Update our `.tf` file to match those defaults.
    - Run `terraform plan` again until it shows “No changes.”

### Confirm in State
Run:
```bash
terraform state list
```
We should now see:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2450).png)

The imported bucket is now tracked alongside our other resources.

### What is the difference between `terraform import` and creating a resource from scratch?
- Creating from scratch: Terraform provisions a new resource in AWS based on our `.tf` config.
- Import: Terraform does not create anything — it simply brings an existing AWS resource under Terraform’s management by adding it to the state file.
- Import is essential when infrastructure already exists (e.g., manually created buckets, legacy resources) and we want Terraform to manage them going forward.

## 5. State Surgery -- mv and rm
State surgery lets us rename or remove resources from Terraform's tracking without touching the real infrastructure in AWS.

### List Current Resources
Run:
```bash
terraform state list
```
- This shows all resources Terraform currently tracks.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2450).png)

### Rename a Resource in State
Suppose we want to rename the imported bucket to `logs_bucket`.

Run:
```bash
terraform state mv aws_s3_bucket.imported aws_s3_bucket.logs_bucket
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2455).png)

- This updates the **state file only** — AWS resource itself is untouched.
- Now update our `.tf` file to match:
```hcl
resource "aws_s3_bucket" "logs_bucket" {
  bucket = "terraweek-import-test-atul"
}
```
Verify:
```bash
terraform plan
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2459).png)

- It should show **“No changes”** because the resource is still the same, only the name in state/config changed.

### Remove a Resource from State
Run:
```bash
terraform state rm aws_s3_bucket.logs_bucket
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2462).png)

- This removes the bucket from Terraform’s state file.
- The bucket **still exists in AWS**, but Terraform no longer manages it.

Verify:
```bash
terraform plan
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2466).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2468).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2469).png)

- Terraform will act as if the bucket doesn’t exist. No resource listed for it.

### Re‑Import the Resource
Bring it back under Terraform management:
```bash
terraform import aws_s3_bucket.logs_bucket terraweek-import-test-<yourname>
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2472).png)

Verify again:
```bash
terraform state list
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2474).png)

- The bucket reappears in the state.

### When would we use `state mv` in a real project? When would we use `state rm`?
When to use `state mv`:
- When we rename a resource in our `.tf` files (e.g., `imported → logs_bucket`) but don’t want Terraform to destroy and recreate it.
- Keeps infra intact while aligning state with config.
- Common in refactoring or reorganizing code.

When to use `state rm`:
- When we want Terraform to **stop managing** a resource but leave it running in AWS.
- Useful if a resource is now managed outside Terraform, or we want to decommission Terraform’s control without deleting the infra.
- Example: handing over a bucket to another team or tool.

## 6. Simulate and Fix State Drift
**State drift** occurs when someone modifies infrastructure outside of Terraform — through the AWS console, CLI, or another tool — causing the real state of AWS to diverge from what Terraform believes it to be.

### Ensure Config and State Are in Sync
Run:
```bash
terraform apply
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2476).png)

- This ensures Terraform state matches our `.tf` files and AWS resources.
- At this point, `terraform plan` should show **“No changes.”**

### Manually Change Infrastructure in AWS Console
Go to the AWS Management Console:
- Navigate to our EC2 instance.
- Change the **Name tag** to `ManuallyChanged`.
- If the instance is stopped, change the **instance type** (e.g., `t2.micro → t2.small`) or add another tag.

These manual changes create **state drift** — reality no longer matches Terraform’s desired state.

### Detect Drift
Run:
```bash
terraform plan
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2481).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2485).png)
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2487).png)

- Terraform will show a **diff**:
    - Tag mismatch (`Name` tag differs).
    - Instance type mismatch (if changed).
        ```
        ~ aws_instance.web
        ~ tags = {
            ~ "Name" = "TerraWeek-Server" -> "ManuallyChanged"
            }
        ```
- Terraform detects drift automatically on every `plan` by comparing our config against the live state of AWS resources.

### Choose How to Resolve
We have two options:
- **Option A (Reconcile):** Run `terraform apply` → Terraform forces AWS back to match our `.tf` config.
- **Option B (Accept Drift):** Update our `.tf` files to match the manual changes → Terraform accepts the new reality.

For this challenge, choose **Option A**:
```bash
terraform apply
```
- Confirm with `yes`.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2494).png)

- Terraform restores the EC2 instance tags and type to match our config.

### Verify Resolution
Run:
```bash
terraform plan
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/ac9059135f727738aa986f7d11e8956e864e3ea2/2026/day-64/Screenshots/Screenshot%20(2497).png)

- Output should show **“No changes.”**
- This confirms drift has been reconciled.

### How teams prevent state drift in production
| Practice | What it prevents |
|---|---|
| Restrict AWS console access via IAM | Stops unauthorized manual changes |
| Enforce CI/CD pipelines for all infra changes | Ensures all changes go through Terraform |
| Scheduled `terraform plan` in CI | Detects drift automatically on a cadence |
| Policy-as-code (OPA, Gatekeeper) | Enforces compliance rules before apply |
| `terraform refresh` | Syncs state with current AWS reality without applying changes |
 
The goal is to make Terraform the **only** path to infrastructure changes — so state drift never has a chance to occur.
