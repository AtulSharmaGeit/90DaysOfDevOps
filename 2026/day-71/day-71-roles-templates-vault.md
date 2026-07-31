# Ansible - Roles, Galaxy, Templates and Vault
> Organizing, reusing, and securing automation at scale.
 
As playbooks grow, managing tasks, variables, handlers, and files in a single YAML file becomes unsustainable. In real-world projects, we orchestrate dozens of servers - web servers, databases, monitoring agents, load balancers — each with distinct responsibilities. This guide covers four pillars that bring structure and security to production Ansible workflows:
 
| Topic | Purpose |
|---|---|
| **Jinja2 Templates** | Generate dynamic config files from variables and facts |
| **Ansible Roles** | Standardize and reuse automation across projects |
| **Ansible Galaxy** | Consume and share community-built roles |
| **Ansible Vault** | Encrypt secrets - passwords, keys, tokens |
 
## Table of Contents
1. [Jinja2 Templates](#1-jinja2-templates)
2. [Understanding Role Structure](#2-understand-the-role-structure)
3. [Building a Custom Webserver Role](#3-build-a-custom-webserver-role)
4. [Ansible Galaxy - Use Community Roles](#4-ansible-galaxy---use-community-roles)
5. [Ansible Vault - Encrypt Secrets](#5-ansible-vault---encrypt-secrets)
6. [Combine Roles, Templates, and Vault](#6-combine-roles-templates-and-vault)

## 1. Jinja2 Templates
Templates let us generate configuration files dynamically by injecting variables and host facts at runtime, eliminating hardcoded values and enabling reuse across environments.

### Create the Template
`templates/nginx-vhost.conf.j2`
```jinja2
# Managed by Ansible -- do not edit manually
server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

### Create the Playbook
 `template-demo.yml`
```yaml
---
- name: Deploy Nginx with template
  hosts: web
  become: true
  vars:
    app_name: terraweek-app
    http_port: 80

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Create web root
      file:
        path: "/var/www/{{ app_name }}"
        state: directory
        mode: '0755'

    - name: Deploy vhost config from template
      template:
        src: templates/nginx-vhost.conf.j2
        dest: "/etc/nginx/conf.d/{{ app_name }}.conf"
        owner: root
        mode: '0644'
      notify: Restart Nginx

    - name: Deploy index page
      copy:
        content: "<h1>{{ app_name }}</h1><p>Host: {{ ansible_hostname }} | IP: {{ ansible_default_ipv4.address }}</p>"
        dest: "/var/www/{{ app_name }}/index.html"

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```
This playbook installs Nginx, creates directories, and deploys the template.

### Run the Playbook
```bash
#Execute the playbook to apply changes and render the template.
ansible-playbook template-demo.yml --diff
```
The `--diff` flag shows line-by-line config changes. On the first run, Nginx is installed, the config is created, and the handler triggers a restart. On subsequent runs, unchanged tasks report `ok`, demonstrating **idempotency**.

### Verify the Rendered Config
Check that variables were replaced with actual values on the server.
```bash
##SSH into web server
cat /etc/nginx/conf.d/terraweek-app.conf
```
Confirm that:
- `listen 80;` and the correct `server_name` are present
- Root path resolves to `/var/www/terraweek-app`
- Logs are named `terraweek-app_access.log` and `terraweek-app_error.log`
- Curl the server IP → custom index page shows app name, host, IP

## 2. Understand the Role Structure
A **role** is a self-contained, reusable unit of automation with a standardized directory layout that Ansible recognizes automatically.
 
```
roles/
└── webserver/
    ├── tasks/
    │   └── main.yml        # Primary task list
    ├── handlers/
    │   └── main.yml        # Service restarts and notifications
    ├── templates/
    │   └── nginx.conf.j2   # Jinja2 templates
    ├── files/
    │   └── index.html      # Static files for direct copy
    ├── vars/
    │   └── main.yml        # High-priority variables
    ├── defaults/
    │   └── main.yml        # Low-priority overridable defaults
    └── meta/
        └── main.yml        # Role metadata and dependencies
```
 
Every subdirectory contains a `main.yml` that Ansible loads automatically. You only need to create the directories relevant to your role.

### Generate a Role Skeleton
```bash
ansible-galaxy init roles/webserver
```
This command scaffolds a **standardized directory structure** under `roles/webserver/`. We’ll see folders like `tasks/`, `handlers/`, `templates/`, `files/`, `vars/`, `defaults/`, and `meta/`.

### Explore the Structure
Here’s what each directory is for:
- **tasks** → Contains `main.yml` with the role’s primary task list.
- **handlers** → Defines service restarts or notifications triggered by tasks.
- **templates** → Holds Jinja2 templates (`*.j2`) for dynamic config files.
- **files** → Static files copied directly to hosts.
- **vars** → High‑priority variables that are hard to override.
- **defaults** → Low‑priority defaults that can easily be overridden.
- **meta** → Metadata like role dependencies, author info, supported platforms.

### Read the README
Inside `roles/webserver/README.md`, Galaxy generates a starter doc explaining how to use the role. This is a good place to document what your role does, required variables, and example usage.

### What is the difference between `vars/main.yml` and `defaults/main.yml`?
| Directory	| Priority | Use Case |
| --- | --- | --- |
| **defaults/main.yml** | Lowest priority | Safe defaults. Caller can override easily in playbooks or inventory. Example: `http_port: 80`. |
| **vars/main.yml**	| High priority | Hard to override. Use for values that must remain fixed. Example: internal paths or OS‑specific settings. |

Think of `defaults` as “suggestions” and `vars` as “enforced values.”

### Verification
- Run `tree roles/webserver` to confirm the skeleton.
- Open `roles/webserver/defaults/main.yml` and `roles/webserver/vars/main.yml`.
- (Optional) Try overriding a default in your playbook — it should work.
- (Optional) Try overriding a var — Ansible will ignore your override, keeping the role’s value.

## 3. Build a Custom Webserver Role
Build a complete `webserver` role from scratch:

### Define Defaults
Edit `roles/webserver/defaults/main.yml`:
```yaml
---
http_port: 80
app_name: myapp
max_connections: 512
```
These are safe defaults. We can override them in playbooks.

### Write Tasks
Edit `roles/webserver/tasks/main.yml`:
```yaml
---
- name: Install Nginx
  yum:
    name: nginx
    state: present

- name: Deploy Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
  notify: Restart Nginx

- name: Deploy vhost config
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/conf.d/{{ app_name }}.conf"
    owner: root
    mode: '0644'
  notify: Restart Nginx

- name: Create web root
  file:
    path: "/var/www/{{ app_name }}"
    state: directory
    mode: '0755'

- name: Deploy index page
  template:
    src: index.html.j2
    dest: "/var/www/{{ app_name }}/index.html"
    mode: '0644'

- name: Start and enable Nginx
  service:
    name: nginx
    state: started
    enabled: true
```

### Define Handlers
Edit `roles/webserver/handlers/main.yml`:
```yaml
---
- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

### Create Templates
`roles/webserver/templates/index.html.j2`:
```html
<h1>{{ app_name }}</h1>
<p>Server: {{ ansible_hostname }}</p>
<p>IP: {{ ansible_default_ipv4.address }}</p>
<p>Environment: {{ app_env | default('development') }}</p>
<p>Managed by Ansible</p>
```

`roles/webserver/templates/vhost.conf.j2`:
```jinja
# Managed by Ansible
server {
    listen {{ http_port }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

`roles/webserver/templates/nginx.conf.j2`:
```jinja
# Global Nginx Configuration
worker_processes auto;
events {
    worker_connections {{ max_connections }};
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```

### Call Role in Playbook
Create `site.yml`:
```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80
```

Run it:
```bash
ansible-playbook site.yml
```

### Verification
Check Nginx service:
```bash
systemctl status nginx
```
- Should be `active (running)`.

Curl the webserver:
```bash
curl http://<web-server-ip>
```
- Should return our custom index page with hostname, IP, and environment.

Inspect config:
```bash
cat /etc/nginx/conf.d/terraweek.conf
```
- Variables (`app_name`, `http_port`) should be rendered.

## 4. Ansible Galaxy - Use Community Roles
[Ansible Galaxy](https://galaxy.ansible.com) is the official marketplace for community-built and vendor-maintained roles.

### Search for roles:
```bash
ansible-galaxy search nginx --platforms EL
ansible-galaxy search mysql
```
This queries Galaxy for roles tagged for **Enterprise Linux (EL)** and MySQL. We’ll see community roles with descriptions, author names, and download counts.

### Install a role from Galaxy:
```bash
ansible-galaxy install geerlingguy.docker
```
Example: install Docker role by Jeff Geerling.

### Check where it was installed:
```bash
ansible-galaxy list
```
Confirms the role is installed under `~/.ansible/roles/`.

### Use the installed role
Create `docker-setup.yml`:
```yaml
---
- name: Install Docker using Galaxy role
  hosts: app
  become: true
  roles:
    - geerlingguy.docker
```
Run:
```bash
ansible-playbook docker-setup.yml
```
Docker will be installed on our `app` servers with a single role call.

### Manage Multiple Roles with Requirements File
Rather than installing roles ad hoc, declare all dependencies in a single file - the Ansible equivalent of `package.json` or `requirements.txt`.

Create `requirements.yml`:
```yaml
---
roles:
  - name: geerlingguy.docker
    version: "7.4.1"
  - name: geerlingguy.ntp
```

Install all at once:
```bash
ansible-galaxy install -r requirements.yml
```

### Verification
- Run `ansible-galaxy list` → confirm roles installed.
- Run `ansible-playbook docker-setup.yml` → check Docker service:
    ```bash
    systemctl status docker
    ```
    - Should be `active (running)`.
- Run `docker --version` → confirms installation.

### Why use a `requirements.yml` instead of installing roles manually?

| Manual install | Requirements.yml |
| --- | --- |
| Ad‑hoc, one‑off installs | Declarative, version‑controlled installs|
| No version pinning | Explicit version pinning (e.g., `7.4.1`) |
| Hard to reproduce across environments | Easy to reproduce in CI/CD pipelines |
| No single source of truth | Acts as a manifest for all required roles |

Think of `requirements.yml` as your **package.json** or **requirements.txt** for Ansible roles - reproducibility and automation are the key benefits.

## 5. Ansible Vault - Encrypt Secrets
**Never store passwords, API keys, or tokens in plain text.** Ansible Vault provides AES-256 encryption for sensitive data within your repository.

### Create an encrypted file:
```bash
ansible-vault create group_vars/db/vault.yml
```
- It will prompt us for a **vault password**.
- After entering, it opens our editor. Add:
    ```yaml
    vault_db_password: SuperSecretP@ssw0rd
    vault_db_root_password: R00tP@ssw0rd123
    vault_api_key: sk-abc123xyz789
    ```
- **Save and exit**.

If we run `cat group_vars/db/vault.yml`, we’ll see encrypted gibberish - not plain text.

### Manage Vault Files
Edit an encrypted file:
```bash
ansible-vault edit group_vars/db/vault.yml
```

View without editing:
```bash
ansible-vault view group_vars/db/vault.yml
```

Encrypt an existing file:
```bash
ansible-vault encrypt group_vars/db/secrets.yml
```

### Use Vault Variables in Playbook
Create `db-setup.yml`:
```yaml
---
- name: Configure database
  hosts: db
  become: true

  tasks:
    - name: Show DB password (never do this in production)
      debug:
        msg: "DB password is set: {{ vault_db_password | length > 0 }}"
```

Run with vault password prompt:
```bash
ansible-playbook db-setup.yml --ask-vault-pass
```

### Automate Vault Password Handling (better for CI/CD):
Instead of typing the password every run:

- Create password file:
    ```bash
    echo "YourVaultPassword" > .vault_pass
    chmod 600 .vault_pass
    echo ".vault_pass" >> .gitignore  # Exclude from version control
    ```
    - Ensures the file is secure and not committed to Git.

Run with file:
```bash
ansible-playbook db-setup.yml --vault-password-file .vault_pass
```
- Output should show:
    ```Code
    TASK [Show DB password] 
    ok: [db-server] => {
        "msg": "DB password is set: True"
    }
    ```
- Check `/etc/db-config.env` later when we integrate with templates - secrets should render correctly.

Or configure in `ansible.cfg`:
```ini
[defaults]
vault_password_file = .vault_pass
```

### Why is `--vault-password-file` better than `--ask-vault-pass` for automated pipelines?
| --ask-vault-pass | --vault-password-file |
| --- | --- |
| Requires interactive input | Non‑interactive, works in CI/CD |
| Blocks automation	| Enables seamless pipeline runs |
| Risk of human error |	Secure, consistent, reproducible |
| Good for manual testing |	Best for production pipelines |

In CI/CD, we need **non‑interactive execution**. Password files (secured and ignored in Git) make automation possible.

## 6. Combine Roles, Templates, and Vault
This section ties everything together into a single unified playbook that configures web servers, app servers, and database servers.

### Prepare Our Role
Make sure our **custom webserver role** from Task 3 is in place under `roles/webserver/`. It should have:
- `defaults/main.yml` with safe defaults (`http_port`, `app_name`, etc.)
- `tasks/main.yml` with Nginx install/config tasks
- `handlers/main.yml` with restart logic
- `templates/` containing `nginx.conf.j2`, `vhost.conf.j2`, `index.html.j2`

### Create Database Template
Add `templates/db-config.j2` at the project root:
```jinja
# Database Configuration -- Managed by Ansible
DB_HOST={{ ansible_default_ipv4.address }}
DB_PORT={{ db_port | default(3306) }}
DB_PASSWORD={{ vault_db_password }}
DB_ROOT_PASSWORD={{ vault_db_root_password }}
```
Secrets are injected from the Vault-encrypted `group_vars/db/vault.yml` at runtime - never stored in plaintext.

### Write the Unified Playbook
Create `site.yml`:
```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80

- name: Configure app servers with Docker
  hosts: app
  become: true
  roles:
    - geerlingguy.docker

- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Create DB config with secrets
      template:
        src: templates/db-config.j2
        dest: /etc/db-config.env
        owner: root
        mode: '0600'
```

### Run the Playbook
```bash
ansible-playbook site.yml --vault-password-file .vault_pass
```
Using the password file ensures non‑interactive runs (better for CI/CD).

### Verification
#### Web servers:
```bash
curl http://<web-server-ip>
```
- Custom index page loads with hostname, IP, and environment.

#### App servers:
```bash
systemctl status docker
docker --version
```
- Docker installed and running.

#### Database servers
```bash
ls -l /etc/db-config.env
# Expected: -rw------- (mode 600)
 
cat /etc/db-config.env
# Expected: secrets rendered correctly, file owned by root
```

## Summary 
| Concept | What it Solves |
|---|---|
| **Jinja2 Templates** | Eliminates hardcoded config values; enables environment-aware files |
| **Roles** | Enforces structure, promotes reuse, and keeps playbooks clean |
| **Galaxy** | Avoids reinventing the wheel; vetted community automation |
| **Vault** | Keeps secrets out of version control while keeping automation intact |
 
Together, these tools form the foundation of **production-grade Ansible** - automation that is organized, reproducible, and secure.
