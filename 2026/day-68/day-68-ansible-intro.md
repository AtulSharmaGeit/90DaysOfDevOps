# Introduction to Ansible and Inventory Setup
A hands-on introduction to Ansible - covering configuration management concepts, architecture, lab setup with Terraform, inventory files, ad-hoc commands, and inventory group patterns.
 
## Table of Contents
- [Understand Ansible](#1-understand-ansible)
- [Set Up Your Lab Environment](#2-set-up-your-lab-environment)
- [Install Ansible](#3-install-ansible)
- [Create Your Inventory File](#4-create-your-inventory-file)
- [Run Ad-Hoc Commands](#5-run-ad-hoc-commands)
- [Explore Inventory Groups and Patterns](#6-explore-inventory-groups-and-patterns)

## 1. Understand Ansible
### What is Configuration Management?
Configuration management (CM) is the practice of defining, tracking, and enforcing the desired state of your infrastructure — which packages are installed, which files exist, which services are running — through code rather than manual intervention.
 
**Without CM:**
- Servers get configured by hand → each one ends up slightly different ("snowflake servers")
- No record of what changed, when, or why
- Rebuilding a failed server takes hours of guesswork
- Scaling to 100 servers means 100× the manual work
**With CM:**
 
| Benefit | What it means in practice |
|---|---|
| **Reproducibility** | Define a server once; spin up 100 identical copies |
| **Idempotency** | Run the tool 10 times — same result, no side effects |
| **Version control** | Infrastructure changes are code-reviewed via Git |
| **Auditability** | Every change is logged and traceable |
| **Disaster recovery** | Rebuild a failed server in minutes from code |
 
### Ansible vs other CM tools
| Dimension | Ansible | Chef | Puppet | Salt |
|---|---|---|---|---|
| Agent | Agentless ✅ | Required | Required | Optional |
| Language | YAML | Ruby DSL | Puppet DSL | YAML + Jinja2 |
| Approach | Procedural | Procedural | Declarative | Both |
| Transport | SSH / WinRM | HTTPS | HTTPS | ZeroMQ / SSH |
| Push / Pull | Push (default) | Pull | Pull | Push + Pull |
| Learning curve | Low | High | Medium–High | Medium |
| Best for | Ad-hoc tasks, simple automation | Complex app deployment | Large-scale enterprise | High-speed execution at scale |
 
Ansible's biggest differentiator is being **agentless + YAML-based**. If SSH works, Ansible works — no new language to learn, nothing to install on managed nodes.
 
### What does "agentless" mean?
Most CM tools require a daemon (agent) installed on every managed server — a background process that polls the master and applies configuration. Agents need installation, upgrades, and troubleshooting of their own.
 
Ansible requires nothing on the managed node beyond what is already there:
 
```
Control Node ──SSH──► Managed Node
                       (no agent — just Python + sshd)
```
 
**How an Ansible run works:**
1. You run a command on the **control node** (your laptop or a jump server)
2. Ansible connects to the **managed node** over SSH (Linux) or WinRM (Windows)
3. It pushes a small Python script to the target and executes it
4. The script is **cleaned up** — nothing persists after the run

**Requirements on managed nodes:** 
| Requirement | Notes |
|---|---|
| Python 2.6+ | Almost always pre-installed on Linux |
| SSH daemon (`sshd`) | Running and accessible |
| SSH authentication | Key-based or password |
 
### Ansible Architecture
 
```
┌─────────────────────────────────────────────────┐
│                  CONTROL NODE                   │
│           (your laptop / jump server)           │
│                                                 │
│  ┌─────────────┐   ┌───────────┐  ┌──────────┐  │
│  │  Playbook   │   │ Inventory │  │ Modules  │  │
│  │  (YAML)     │   │ (hosts)   │  │ (tasks)  │  │
│  └──────┬──────┘   └─────┬─────┘  └────┬─────┘  │
│         └────────────────┴─────────────┘        │
│                          │                      │
│               ┌──────────▼──────────┐           │
│               │   Ansible Engine    │           │
│               └──────────┬──────────┘           │
└──────────────────────────│──────────────────────┘
                           │ SSH / WinRM
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   web-01    │  │   web-02    │  │    db-01    │
   │  (EC2)      │  │  (EC2)      │  │  (EC2)      │
   │ Python+sshd │  │ Python+sshd │  │ Python+sshd │
   └─────────────┘  └─────────────┘  └─────────────┘
              MANAGED NODES (no agent installed)
```
 
### Core components
 
**Control Node** — the machine where Ansible is installed and all commands originate. This can be your laptop, a bastion server, or a CI/CD runner (GitHub Actions, Jenkins). Ansible runs only here.
 
**Managed Nodes** — the servers Ansible configures. No Ansible installation required — only Python and SSH.
 
**Inventory** — a file (or dynamic script) listing your managed nodes, optionally organized into groups.
 
```ini
# Static inventory example (hosts.ini)
[webservers]
web-01 ansible_host=10.0.1.10
web-02 ansible_host=10.0.1.11
 
[databases]
db-01 ansible_host=10.0.2.5
 
[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```
 
**Dynamic inventory** — a plugin or script that queries AWS, GCP, or Azure in real time to build the host list automatically, instead of maintaining a static file.
 
**Modules** — self-contained units of work that Ansible ships to managed nodes and executes. There are 3,000+ built-in modules, and every module is idempotent.
 
| Module | What it does |
|---|---|
| `apt` / `yum` | Install or remove packages |
| `copy` | Copy a file to the remote host |
| `template` | Render a Jinja2 template and deploy it |
| `service` | Start, stop, enable, or disable a service |
| `user` | Create or manage Linux users |
| `file` | Create directories, set permissions |
| `git` | Clone or update a Git repository |
| `command` / `shell` | Run arbitrary commands (use sparingly) |
 
**Playbooks** — YAML files that declare: *"on these hosts, in this order, run these tasks."*
 
```yaml
- name: Configure web servers
  hosts: webservers
  become: true        # run as sudo
 
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present    # idempotent: only installs if not already present
 
    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: true
 
    - name: Deploy config from template
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx
 
  handlers:
    - name: Restart nginx    # only runs if the template task changed
      service:
        name: nginx
        state: restarted
```
 
**Playbook structure:**
 
```
Playbook
 └── Play (targets a group of hosts)
      ├── vars       — variables for this play
      ├── tasks      — ordered list of module calls
      ├── handlers   — tasks triggered by notify
      └── roles      — reusable bundles of tasks/files/templates
```
 
### Quick reference cheat sheet
 
```bash
ansible --version                                        # check version
ansible all -i hosts.ini -m ping                         # ping all hosts
ansible-playbook -i hosts.ini site.yml                   # run a playbook
ansible-playbook -i hosts.ini site.yml -v                # verbose output
ansible-playbook -i hosts.ini site.yml --check           # dry run
ansible-playbook -i hosts.ini site.yml --limit web-01    # limit to one host
```

## 2. Set Up Your Lab Environment
Provision three EC2 instances with Terraform to use as Ansible managed nodes.
### Project setup
Create a new folder:
```bash
mkdir day-68-ansible-lab && cd day-68-ansible-lab
```
Initialize files:
```bash
touch main.tf variables.tf outputs.tf provider.tf
```

### Configure Provider
In `provider.tf`:
```hcl
provider "aws" {
  region = "ap-south-1"   # Mumbai region
}
```

### Define Variables
In `variables.tf`:
```hcl
variable "ami_id" {
  description = "AMI for Amazon Linux 2 or Ubuntu 22.04"
  default     = "ami-xxxxxxxx"   # Replace with valid AMI ID
}

variable "key_name" {
  description = "SSH key pair name"
  default     = "your-key"
}
```
> Find valid AMI IDs in the AWS Console → EC2 → AMIs. Use Amazon Linux 2 (`ec2-user`) or Ubuntu 22.04 (`ubuntu`).

### Create Security Group
In `main.tf`:
```hcl
resource "aws_security_group" "ssh" {
  name        = "ansible-ssh-sg"
  description = "Allow SSH inbound traffic"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_PUBLIC_IP/32"]  # Replace with your IP
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Provision EC2 Instances
Still in `main.tf`:
```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"
  key_name      = var.key_name
  vpc_security_group_ids = [aws_security_group.ssh.id]
  tags = { Name = "web-server" }
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t2.micro"
  key_name      = var.key_name
  vpc_security_group_ids = [aws_security_group.ssh.id]
  tags = { Name = "app-server" }
}

resource "aws_instance" "db" {
  ami           = var.ami_id
  instance_type = "t2.micro"
  key_name      = var.key_name
  vpc_security_group_ids = [aws_security_group.ssh.id]
  tags = { Name = "db-server" }
}
```

### Outputs for Easy Access
In `outputs.tf`:
```hcl
output "web_public_ip" {
  value = aws_instance.web.public_ip
}

output "app_public_ip" {
  value = aws_instance.app.public_ip
}

output "db_public_ip" {
  value = aws_instance.db.public_ip
}
```

### Deploy Infrastructure
Run:
```bash
terraform init
terraform plan
terraform apply -auto-approve
```
- After apply, Terraform will print the public IPs of your 3 servers.

### Verify SSH Access
From your control node:
```bash
ssh -i ~/your-key.pem ec2-user@<web_public_ip>
ssh -i ~/your-key.pem ec2-user@<app_public_ip>
ssh -i ~/your-key.pem ec2-user@<db_public_ip>
```
- Use `ec2-user` for Amazon Linux 2
- Use `ubuntu` for Ubuntu 22.04

#### Expected Result
- 3 EC2 instances (web-server, app-server, db-server)
- Security group allowing SSH from your IP
- Verified SSH connectivity from control node

## 3. Install Ansible
### Decide Your Control Node
- The **control node** is the machine where Ansible is installed and commands are run.
- You can use:
  - Your **local laptop** (macOS/Linux/Windows with WSL).
  - Or a **dedicated EC2 instance** (often a jump server).
- You only need Ansible on the control node because it connects to managed nodes via SSH - no agent is installed remotely.

### Install Ansible
Ubuntu/Debian
```bash
sudo apt update
sudo apt install ansible -y
```

### Verify Installation
Run:
```bash
ansible --version
```
Expected output includes:
- Ansible version (e.g., `ansible [core 2.16.3]`)
- Config file path (e.g., `/etc/ansible/ansible.cfg`)
- Python version used

### Why is Ansible only needed on the control node?
- Ansible is **agentless**.
- It connects via SSH and executes modules remotely.
- Managed nodes don’t need Ansible installed - only Python + SSH access.

## 4. Create Your Inventory File
The inventory tells Ansible which servers to manage and how to connect to them.
### Create Project Directory
```bash
mkdir ansible-practice && cd ansible-practice
```
- This keeps your Ansible files organized.

### Create Inventory File `inventory.ini`:
```ini
[web]
web-server ansible_host=<PUBLIC_IP_1>

[app]
app-server ansible_host=<PUBLIC_IP_2>

[db]
db-server ansible_host=<PUBLIC_IP_3>

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/your-key.pem
```
- Replace `<PUBLIC_IP_1>` etc. with the IPs Terraform printed.
- Use `ec2-user` for Amazon Linux 2, or `ubuntu` for Ubuntu 22.04.
- The `ansible_ssh_private_key_file` points to your `.pem` key.

### Verify Connectivity
Run:
```bash
ansible all -i inventory.ini -m ping
```
Expected output:
```json
web-server | SUCCESS => { "ping": "pong" }
app-server | SUCCESS => { "ping": "pong" }
db-server  | SUCCESS => { "ping": "pong" }
```

### Troubleshooting if Ping Fails
**SSH key permissions**
```bash
chmod 400 ~/your-key.pem
```
**Security group**  
- Ensure port 22 is open for your IP.

**Correct user**
- Check the `ansible_user` matches our AMI (`ec2-user` for Amazon Linux, `ubuntu` for Ubuntu)

## 5. Run Ad-Hoc Commands
Ad-hoc commands let you run quick one-off tasks across hosts without writing a playbook. Useful for checks, installs, and file operations.

### Command syntax 
```
ansible <target> -i <inventory> -m <module> -a "<arguments>" [options]
```

### Check Uptime on All Servers
```bash
ansible all -i inventory.ini -m command -a "uptime"
```
- `all` → runs on every host in your inventory.
- `-m command` → uses the `command` module (runs simple commands).
- `-a "uptime"` → passes the command to execute.<br>
Output shows how long each server has been running.

### Check Free Memory on Web Servers Only
```bash
ansible web -i inventory.ini -m command -a "free -h"
```
- `web` → runs only on the `[web]` group.
- `free -h` → shows memory usage in human‑readable format.

### Check Disk Space on All Servers
```bash
ansible all -i inventory.ini -m command -a "df -h"
```
- `df -h` → shows disk usage in human‑readable format.<br>
Useful for monitoring storage across all nodes.

### Install a Package on Web Group
```bash
ansible web -i inventory.ini -m yum -a "name=git state=present" --become
```
- `yum` **module** → installs packages (use `apt` if Ubuntu).
- `name=git state=present` → ensures Git is installed.
- `--become` → escalates privileges (like `sudo`).<br>
Needed because installing packages requires root access.

### Copy a File to All Servers
First, create a file:
```bash
echo "Hello from Ansible" > hello.txt
```
Then copy it:
```bash
ansible all -i inventory.ini -m copy -a "src=hello.txt dest=/tmp/hello.txt"
```
Verify:
```bash
ansible all -i inventory.ini -m command -a "cat /tmp/hello.txt"
```
Each server should print: `Hello from Ansible`.

### What does `--become` do?  
> `--become` escalates privileges to sudo. Required for any task that installs packages, manages services, or edits system files.

## 6. Explore Inventory Groups and Patterns
### Extend `inventory.ini` with group-of-groups
```ini
[web]
web-server ansible_host=<PUBLIC_IP_1>
 
[app]
app-server ansible_host=<PUBLIC_IP_2>
 
[db]
db-server ansible_host=<PUBLIC_IP_3>
 
[application:children]
web
app
 
[all_servers:children]
application
db
 
[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/your-key.pem
```
 
`[application:children]` combines the `web` and `app` groups. `[all_servers:children]` combines everything. This lets you target multiple groups with a single name.

### Run Commands Against Groups
Test connectivity:
```bash
ansible application  -i inventory.ini -m ping    # web + app servers
ansible db           -i inventory.ini -m ping    # db server only
ansible all_servers  -i inventory.ini -m ping    # all three servers
```
Expected output: each host responds with `"ping": "pong"`.

### Use patterns for flexible targeting hosts
| Pattern | Targets |
|---|---|
| `all` | Every host in inventory |
| `web` | Hosts in the `[web]` group |
| `'web:app'` | Hosts in web **or** app (union) |
| `'all:!db'` | All hosts **except** the db group |
| `web-server` | A specific named host |
 
```bash
ansible 'web:app'  -i inventory.ini -m ping    # web or app
ansible 'all:!db'  -i inventory.ini -m ping    # everyone except db
```


### Create `ansible.cfg`
To avoid typing `-i inventory.ini` every time, create `ansible.cfg` in the same directory:
```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ec2-user
private_key_file = ~/your-key.pem
```
| Setting | Effect |
|---|---|
| `inventory` | Default inventory file — no need for `-i inventory.ini` |
| `host_key_checking = False` | Skips SSH fingerprint prompts for new hosts |
| `remote_user` | Default login user for all connections |
| `private_key_file` | Path to your `.pem` key |
 
With `ansible.cfg` in place, commands simplify to:
```bash
ansible all -m ping
ansible web -m command -a "uptime"
ansible-playbook site.yml
```
If configured correctly, we’ll see SUCCESS without needing `-i inventory.ini`.
