# Ansible Project: Automate Docker and Nginx Deployment
> Five days of Ansible culminate here - one command to go from a fresh server to a fully running, production-style setup.
 
This project automates a complete deployment pipeline: install Docker, pull and run a containerized application, configure Nginx as a reverse proxy, and manage everything through Ansible roles. It draws on every concept covered across the week - inventory, playbooks, modules, handlers, variables, facts, conditionals, loops, roles, templates, Galaxy, and Vault.
 
## Table of Contents
1. [Project Structure](#1-plan-the-project-structure)
2. [Common Role - Baseline Setup](#2-build-the-common-role)
3. [Docker Role - Container Management](#3-build-the-docker-role)
4. [Nginx Role - Reverse Proxy](#4-build-the-nginx-role)
5. [Vault - Encrypt Docker Hub Credentials](#5-encrypt-docker-hub-credentials-with-vault)
6. [Master Playbook & Full Deployment](#6-write-the-master-playbook-and-deploy)
7. [Bonus - Deploy a Different App](#7-bonus---deploy-a-different-app-and-re-run)


## 1. Plan the Project Structure
### Create the complete project layout:
```
ansible-docker-project/
├── ansible.cfg
├── inventory.ini
├── site.yml                          # Master playbook
├── group_vars/
│   ├── all.yml                       # Common variables
│   └── web/
│       ├── vars.yml                  # Nginx-specific variables
│       └── vault.yml                 # Encrypted Docker Hub credentials
└── roles/
    ├── common/                       # Baseline setup for all servers
    │   └── tasks/main.yml
    ├── docker/                       # Docker install and container management
    │   ├── tasks/main.yml
    │   ├── templates/
    │   │   └── docker-compose.yml.j2
    │   ├── handlers/main.yml
    │   └── defaults/main.yml
    └── nginx/                        # Nginx reverse proxy
        ├── tasks/main.yml
        ├── templates/
        │   ├── nginx.conf.j2
        │   └── app-proxy.conf.j2
        ├── handlers/main.yml
        └── defaults/main.yml
```

### Create the Project Root
```bash
mkdir ansible-docker-project
cd ansible-docker-project
```

### Create Top‑Level Files
Inside `ansible-docker-project/`, create the following empty files:
```bash
touch ansible.cfg inventory.ini site.yml
```
- **ansible.cfg** → global settings (inventory, vault password file, SSH options).
- **inventory.ini** → list of managed nodes (web, app, db).
- **site.yml** → master playbook that calls roles.

### Create Group Variables Directories & Files
```bash
mkdir -p group_vars/web
touch group_vars/all.yml group_vars/web/vars.yml group_vars/web/vault.yml
```
- **all.yml** → common variables (timezone, project name, packages).
- **vars.yml** → Nginx‑specific variables (server_name, ports).
- **vault.yml** → encrypted Docker Hub credentials (created later with `ansible-vault create`).

### Generate Role Skeletons
```bash
mkdir -p roles
ansible-galaxy init roles/common
ansible-galaxy init roles/docker
ansible-galaxy init roles/nginx
```
- This generates the standard role layout (`tasks/`, `handlers/`, `templates/`, `defaults/`, etc.) for each role.

### Add Required Templates
Inside each role, create the templates we’ll need:
```bash
touch roles/docker/templates/docker-compose.yml.j2
touch roles/nginx/templates/nginx.conf.j2
touch roles/nginx/templates/app-proxy.conf.j2
```

### Populate `ansible.cfg`
Based on our Day‑68 setup:
```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ec2-user        # or ubuntu if using Ubuntu AMI
private_key_file = ~/your-key.pem
vault_password_file = .vault_pass
```
- This avoids typing `-i inventory.ini` or `--ask-vault-pass` on every command.

### Populate `inventory.ini`
Use the IPs from our Day‑68 Terraform outputs:
```ini
[web]
web-server ansible_host=<WEB_PUBLIC_IP>

[app]
app-server ansible_host=<APP_PUBLIC_IP>

[db]
db-server ansible_host=<DB_PUBLIC_IP>

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/your-key.pem
```
- Replace `<WEB_PUBLIC_IP>` etc. with actual values.
- If Ubuntu AMIs: change `ansible_user` to `ubuntu`.

### Verify Connectivity
```bash
ansible all -m ping
```

Expected output:
```Code
web-server | SUCCESS => { "ping": "pong" }
app-server | SUCCESS => { "ping": "pong" }
db-server  | SUCCESS => { "ping": "pong" }
```

## 2. Build the Common Role
The `common` role runs on every server and establishes a consistent baseline: updated packages, correct timezone, hostname, and a dedicated deploy user.

### Navigate to the Common Role
```bash
cd ansible-docker-project/roles/common/tasks
```

### Create `roles/common/tasks/main.yml`:
```yaml
---
- name: Update package cache
  yum:                   # use apt if Ubuntu
    update_cache: true
  tags: common

- name: Install common packages
  yum:                   # or apt
    name: "{{ common_packages }}"
    state: present
  tags: common

- name: Set hostname
  hostname:
    name: "{{ inventory_hostname }}"
  tags: common

- name: Set timezone
  timezone:
    name: "{{ timezone }}"
  tags: common

- name: Create deploy user
  user:
    name: deploy
    groups: wheel
    shell: /bin/bash
    state: present
  tags: common
```

### Define Global Variables
Edit `group_vars/all.yml`:
```yaml
---
timezone: Asia/Kolkata
project_name: devops-app
app_env: development
common_packages:
  - vim
  - curl
  - wget
  - git
  - htop
  -
   tree
  - jq
  - unzip
```
These variables are referenced in our tasks:
- `timezone` → used by the timezone module
- `common_packages` → list of baseline packages
- `project_name` and `app_env` → useful later in Nginx templates

### Verify Role Syntax
Run a quick syntax check:
```bash
ansible-playbook site.yml --syntax-check
```

### Dry Run the Common Role
Target all servers with just the common role:
```bash
ansible-playbook site.yml --tags common --check --diff
```
- This shows what would change without actually applying it.

### Full Apply
```bash
ansible-playbook site.yml --tags common
```

## 3. Build the Docker Role
This role installs Docker CE, starts the service, authenticates with Docker Hub, pulls the application image, and runs it as a container.

### Define Defaults
`roles/docker/defaults/main.yml`
```yaml
---
docker_app_image: nginx
docker_app_tag: latest
docker_app_name: myapp
docker_app_port: 8080
docker_container_port: 80
```
- These are our baseline variables. Later we can override them in `group_vars/all.yml` or at runtime via `-e`.

### Write Tasks
`roles/docker/tasks/main.yml` 
```yaml
---
# Install Docker dependencies
- name: Install Docker dependencies
  yum:                      # use apt if Ubuntu
    name:
      - yum-utils
      - device-mapper-persistent-data
      - lvm2
    state: present
  tags: docker

# Add Docker CE repository
- name: Add Docker CE repo
  get_url:
    url: https://download.docker.com/linux/centos/docker-ce.repo
    dest: /etc/yum.repos.d/docker-ce.repo
  when: ansible_os_family == "RedHat"
  tags: docker

# Install Docker CE
- name: Install Docker CE
  yum:                      # or apt
    name: docker-ce
    state: present
  tags: docker

# Start and enable Docker service
- name: Enable Docker service
  service:
    name: docker
    state: started
    enabled: true
  tags: docker

# Add deploy user to docker group
- name: Add deploy user to docker group
  user:
    name: deploy
    groups: docker
    append: true
  tags: docker

# Install Docker Compose
- name: Install Docker Compose
  pip:
    name: docker-compose
  tags: docker

# Log in to Docker Hub
- name: Log in to Docker Hub
  community.docker.docker_login:
    username: "{{ vault_docker_username }}"
    password: "{{ vault_docker_password }}"
  become_user: deploy
  when: vault_docker_username is defined
  tags: docker

# Pull application image
- name: Pull application image
  community.docker.docker_image:
    name: "{{ docker_app_image }}"
    tag: "{{ docker_app_tag }}"
    source: pull
  tags: docker

# Run application container
- name: Run application container
  community.docker.docker_container:
    name: "{{ docker_app_name }}"
    image: "{{ docker_app_image }}:{{ docker_app_tag }}"
    state: started
    restart_policy: always
    ports:
      - "{{ docker_app_port }}:{{ docker_container_port }}"
  tags: docker

# Verify container health
- name: Wait for container to be healthy
  uri:
    url: "http://localhost:{{ docker_app_port }}"
    status_code: 200
  retries: 5
  delay: 3
  register: health_check
  until: health_check.status == 200
  tags: docker
```

### Add Handler
`roles/docker/handlers/main.yml`:
```yaml
---
- name: Restart Docker
  service:
    name: docker
    state: restarted
```

### Install Required Collection
On our control node:
```bash
ansible-galaxy collection install community.docker
```
- This enables the `community.docker` modules used above.

### Dry Run
Run only the docker role:
```bash
ansible-playbook site.yml --tags docker --check --diff
```

### Full Apply
```bash
ansible-playbook site.yml --tags docker
```

## 4. Build the Nginx Role
This role installs Nginx and configures it to proxy incoming HTTP traffic to the running Docker container.

### Define Defaults
`roles/nginx/defaults/main.yml`:
```yaml
---
nginx_http_port: 80
nginx_upstream_port: 8080
nginx_server_name: "_"
```
- These variables control the proxy port, upstream container port, and server name.

### Write Tasks
`roles/nginx/tasks/main.yml`:
```yaml
---
# Install Nginx
- name: Install Nginx
  yum:                   # use apt if Ubuntu
    name: nginx
    state: present
  tags: nginx

# Remove default site config
- name: Remove default Nginx site
  file:
    path: /etc/nginx/conf.d/default.conf
    state: absent
  tags: nginx

# Deploy main Nginx config
- name: Deploy main Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Reload Nginx
  tags: nginx

# Deploy reverse proxy config
- name: Deploy reverse proxy config
  template:
    src: app-proxy.conf.j2
    dest: /etc/nginx/conf.d/app-proxy.conf
  notify: Reload Nginx
  tags: nginx

# Test Nginx configuration
- name: Test Nginx configuration
  command: nginx -t
  changed_when: false
  tags: nginx

# Start and enable Nginx
- name: Enable Nginx service
  service:
    name: nginx
    state: started
    enabled: true
  tags: nginx
```

### Add Templates
`roles/nginx/templates/app-proxy.conf.j2`:
```jinja
# Reverse Proxy to Docker Container -- Managed by Ansible
upstream docker_app {
    server 127.0.0.1:{{ nginx_upstream_port }};
}

server {
    listen {{ nginx_http_port }};
    server_name {{ nginx_server_name }};

    location / {
        proxy_pass http://docker_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }

{% if app_env == 'production' %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log;
{% else %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log debug;
{% endif %}
}
```
- We’ll need a main Nginx config template (`roles/nginx/templates/nginx.conf.j2`) to replace the default `/etc/nginx/nginx.conf`. 
  ```jinja
  # Managed by Ansible - Main Nginx Config

  user  nginx;
  worker_processes  auto;

  error_log  /var/log/nginx/error.log warn;
  pid        /var/run/nginx.pid;

  events {
      worker_connections 1024;
  }

  http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;

      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

      access_log  /var/log/nginx/access.log  main;

      sendfile        on;
      keepalive_timeout  65;

      # Include all site configs (like app-proxy.conf)
      include /etc/nginx/conf.d/*.conf;
  }

  ```

### Add Handlers
`roles/nginx/handlers/main.yml`:
```yaml
---
- name: Reload Nginx
  service:
    name: nginx
    state: reloaded

- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

### Dry Run
Run only the nginx role:
```bash
ansible-playbook site.yml --tags nginx --check --diff
```

### Full Apply
```bash
ansible-playbook site.yml --tags nginx
```

## 5. Encrypt Docker Hub Credentials with Vault
Never store Docker Hub tokens or passwords in plaintext. Ansible Vault encrypts them at rest while making them transparently available to playbooks at runtime.

### Create the Encrypted Vault File
```bash
ansible-vault create group_vars/web/vault.yml
```

It will open our editor. Paste in:
```yaml
vault_docker_username: your-dockerhub-username
vault_docker_password: your-dockerhub-token
```
- **Save and exit**. The file is now AES-256 encrypted - `cat group_vars/web/vault.yml` will show only ciphertext.

### Create a Vault Password File
Instead of typing the vault password every time, store it locally:
```bash
echo "YourVaultPassword" > .vault_pass
chmod 600 .vault_pass
echo ".vault_pass" >> .gitignore
```
- `.vault_pass` → contains our vault password.
- `chmod 600` → ensures only we can read it.
- `.gitignore` → prevents committing the password file to GitHub.

With `vault_password_file = .vault_pass` already set in `ansible.cfg`, Ansible decrypts vault files automatically on every run - no interactive prompt needed.
```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ec2-user
private_key_file = ~/your-key.pem
vault_password_file = .vault_pass
```
- Now Ansible will automatically use `.vault_pass` when decrypting vault files.

### Verify Vault Setup
```bash
ansible-vault view group_vars/web/vault.yml
```
- It should prompt for the password (or use `.vault_pass`) and then show our encrypted values.

### Test Docker Login Task
When we run the **docker role**, the login task will use:
```yaml
community.docker.docker_login:
  username: "{{ vault_docker_username }}"
  password: "{{ vault_docker_password }}"
```
- If the vault is set up correctly, Ansible will decrypt the credentials and log in to Docker Hub.

## 6. Write the Master Playbook and Deploy
### Master Playbook
`site.yml`
```yaml
---
- name: Apply common configuration
  hosts: all
  become: true
  roles:
    - common
  tags: common

- name: Install Docker and run containers
  hosts: web
  become: true
  roles:
    - docker
  tags: docker

- name: Configure Nginx reverse proxy
  hosts: web
  become: true
  roles:
    - nginx
  tags: nginx
```
This ensures:
- **common** runs on all servers.
- **docker** runs only on web servers.
- **nginx** runs only on web servers.

### Dry Run First
Always check before applying:
```bash
ansible-playbook site.yml --check --diff
```
- This shows what would change without touching the servers.

### Full Deploy
Once dry run looks good:
```bash
ansible-playbook site.yml
```

### Selective Runs with Tags
```bash
# Run only the Docker role
ansible-playbook site.yml --tags docker
 
# Run only the Nginx role
ansible-playbook site.yml --tags nginx
 
# Skip the common role
ansible-playbook site.yml --skip-tags common
```

### Verification Checklist
After deployment, confirm everything is working:
| Component | Verification Command | Expected Output |
| --- | --- | --- |
| Docker container | `docker ps` |	Container `myapp` running, port `8080:80` mapped |
| Direct app access |	`curl http://localhost:8080` | HTTP 200 response from container |
| Nginx reverse proxy |	`curl http://localhost:80` |	Same response, proxied through Nginx |
| Health endpoint |	`curl http://localhost:80/health` |	Returns `OK` |


### Idempotency Check
Run the playbook again:
```bash
ansible-playbook site.yml
```
Almost all tasks should report `ok` with minimal `changed` - confirming the automation is fully idempotent.

## 7. Bonus - Deploy a Different App and Re-Run
### Override Docker Variables
Instead of editing defaults, we can override at runtime:
```bash
ansible-playbook site.yml --tags docker \
  -e "docker_app_image=httpd docker_app_tag=latest docker_app_name=apache-app"
```
- `docker_app_image=httpd` → switch from Nginx to Apache HTTPD.
- `docker_app_name=apache-app` → container name changes.
- **Nginx config** → unchanged, still proxies port 8080 → 80.

### Verify Replacement
```bash
docker ps
```
Expected:
- Old `myapp` container gone.
- New `apache-app` container running on port `8080:80`.

### Full Playbook Re‑Run
Now run everything:
```bash
ansible-playbook site.yml
```
- Output should show mostly **ok** with minimal **changed** - proving idempotency.

### Reflection & Documentation
#### Task Count
- Count tasks across common, docker, nginx roles.
- First run → many **changed**.
- Second run → mostly **ok**.

#### Concept Mapping
| Day | Concept Used |
|-----|-------------|
| 68 | Inventory, ad-hoc commands, SSH setup |
| 69 | Playbooks, modules, handlers |
| 70 | Variables, facts, conditionals, loops |
| 71 | Roles, templates, Galaxy, Vault |
| 72 | Everything combined in one project |


#### What would you add for production enhancement? 
- **SSL/TLS** — automate certificate provisioning with Certbot
- **Monitoring** — deploy Prometheus and Grafana alongside the app
- **Log rotation** — configure `logrotate` for Nginx and application logs
- **Multi-container setup** — extend with Docker Compose (app + DB + cache)
- **CI/CD integration** — trigger `ansible-playbook` from GitHub Actions or Jenkins

### Clean Up
Terraform:
```bash
terraform destroy
```
Manual EC2: terminate from AWS console.

