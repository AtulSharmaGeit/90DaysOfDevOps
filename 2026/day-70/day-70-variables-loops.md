# Ansible - Variables, Facts, Conditionals and Loops
Our playbooks work, but they are static -- same packages, same config, same behavior on every server. Real infrastructure is not like that. Web servers need Nginx, app servers need Node.js, production gets more memory than dev. Today we make your playbooks smart.

Variables, facts, conditionals, and loops turn a rigid script into flexible automation that adapts to each host, each group, and each environment.

## Table of Contents 
- [Variables in Playbooks](#1-variables-in-playbooks)
- [group_vars and host_vars](#2-group_vars-and-host_vars)
- [Ansible Facts — Gathering System Information](#3-ansible-facts--gathering-system-information)
- [Conditionals with when](#4-conditionals-with-when)
- [Loops](#5-loops)
- [Register, Debug, and Combine Everything](#6-register-debug-and-combine-everything)

## 1. Variables in Playbooks
### Create Playbook File
We need a YAML file that defines variables and tasks.

Create a file `variables-demo.yml`:
```bash
vim variables-demo.yml
```

```yaml
---
- name: Variable demo
  hosts: all
  become: true

  vars:
    app_name: terraweek-app
    app_port: 8080
    app_dir: "/opt/{{ app_name }}"
    packages:
      - git
      - curl
      - wget

  tasks:
    - name: Print app details
      debug:
        msg: "Deploying {{ app_name }} on port {{ app_port }} to {{ app_dir }}"

    - name: Create application directory
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: Install required packages
      yum:
        name: "{{ packages }}"
        state: present
```

### Run Playbook with Defaults
```bash
#Execute the playbook to see variables resolve.
ansible-playbook variables-demo.yml
```
- Watch the debug output → should print `Deploying terraweek-app on port 8080 to /opt/terraweek-app`
- Verify directory `/opt/terraweek-app` is created
- Confirm packages git, curl, wget are installed

### Override Variables via CLI
```bash
#Extra vars (-e) override playbook defaults.
ansible-playbook variables-demo.yml -e "app_name=my-custom-app app_port=9090"
```
- Debug output should now show `Deploying my-custom-app on port 9090 to /opt/my-custom-app`
- Directory `/opt/my-custom-app` created
- Packages installed remain the same

### Does the CLI variable override the playbook variable?
Check that CLI overrides beat playbook vars.
- Confirm `my-custom-app` directory exists
- Confirm debug message reflects overridden values

>Note: precedence order → role defaults < group_vars < host_vars < playbook vars < task vars < extra vars (-e)

## 2. `group_vars` and `host_vars`
Variables defined inline in playbooks become hard to manage at scale. The standard approach is to move them to dedicated files that Ansible loads automatically.

### Create the project structure
Inside our working directory, scaffold this layout:
```bash
mkdir -p ansible-practice/{group_vars,host_vars,playbooks}
touch ansible-practice/{inventory.ini,ansible.cfg}
touch ansible-practice/group_vars/{all.yml,web.yml,db.yml}
touch ansible-practice/host_vars/web-server.yml
touch ansible-practice/playbooks/site.yml
```
- This ensures Ansible automatically loads variables from `group_vars/` and `host_vars/`.

```
ansible-practice/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   ├── all.yml         # applied to every host
│   ├── web.yml         # applied to [web] group only
│   └── db.yml          # applied to [db] group only
├── host_vars/
│   └── web-server.yml  # applied to web-server only
└── playbooks/
    └── site.yml
```

### Define your inventory
`ansible-practice/inventory.ini`:
```ini
[web]
web-server ansible_host=1.2.3.4 ansible_user=ec2-user

[db]
db-server ansible_host=5.6.7.8 ansible_user=ec2-user
```
- Replace IPs with your actual servers.
- Groups: `web` and `db`.

### Configure Ansible defaults
`ansible-practice/ansible.cfg`:
```ini
[defaults]
inventory = ./inventory.ini
remote_user = ec2-user
host_key_checking = False
```
- This avoids typing `-i inventory.ini` every time.

### Create `group_vars` files
- `group_vars/all.yml` — applied to every host:
    ```yaml
    ---
    ntp_server: pool.ntp.org
    app_env: development
    common_packages:
    - vim
    - htop
    - tree
    ```

- `group_vars/web.yml` — applied to `[web]` group only:
    ```yaml
    ---
    http_port: 80
    max_connections: 1000
    web_packages:
    - nginx
    ```

- `group_vars/db.yml` — applied to `[db]` group only:
    ```yaml
    ---
    db_port: 3306
    db_packages:
    - mysql-server
    ```

### Create `host_vars` file
`host_vars/web-server.yml` — overrides for one specific host:
```yaml
---
max_connections: 2000
custom_message: "This is the primary web server"
```
> `host_vars` overrides `group_vars`. The `web-server` host will have `max_connections = 2000`, while all other web hosts have `max_connections = 1000`.

### Write the playbook
`playbooks/site.yml`:
```yaml
---
- name: Apply common config
  hosts: all
  become: true
  tasks:
    - name: Install common packages
      yum:
        name: "{{ common_packages }}"
        state: present
    - name: Show environment
      debug:
        msg: "Environment: {{ app_env }}"

- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Show web config
      debug:
        msg: "HTTP port: {{ http_port }}, Max connections: {{ max_connections }}"
    - name: Show host-specific message
      debug:
        msg: "{{ custom_message }}"
```

### Run the playbook
```bash
cd ansible-practice
ansible-playbook playbooks/site.yml
```
- On **all hosts** → installs `vim`, `htop`, `tree`.
- On **web group** → prints HTTP port and max connections.
- On **web-server host** → max_connections shows `2000` (host_vars override).
- On **db group** → only common tasks run, db vars available if referenced.

### Verify variable precedence
Run with CLI override:
```bash
ansible-playbook playbooks/site.yml -e "app_env=production"
```
- Output should show `Environment: production`.
- Confirms **extra vars (-e)** override everything.

### What each host receives 
| Host | `max_connections` | Source | `app_env` |
|---|---|---|---|
| `web-server` | 2000 | `host_vars/web-server.yml` | `development` |
| Other web hosts | 1000 | `group_vars/web.yml` | `development` |
| DB hosts | N/A | not defined | `development` |
| All hosts (with `-e`) | — | — | `production` |

### Variable precedence — lowest to highest
| Priority | Source | Example |
|---|---|---|
| 1 (lowest) | Role defaults | `defaults/main.yml` in a role |
| 2 | `group_vars/all` | Applies to every host |
| 3 | `group_vars/<group>` | Applies to a specific group |
| 4 | `host_vars/<host>` | Applies to one specific host |
| 5 | Playbook `vars:` block | Defined inline in the play |
| 6 | Task `vars:` | Defined on a single task |
| 7 (highest) | Extra vars (`-e`) | CLI override — beats everything |
 
The `-e` flag always wins. Use it to override defaults for a single run without editing any files.

## 3. Ansible Facts -- Gathering System Information
Facts are variables automatically gathered from managed nodes at the start of every playbook run. They describe the host's actual hardware, OS, and network configuration.

### See all facts for a host:
```bash
ansible web-server -m setup
```

### Filter specific facts:
```bash
ansible web-server -m setup -a "filter=ansible_os_family"
ansible web-server -m setup -a "filter=ansible_distribution*"
ansible web-server -m setup -a "filter=ansible_memtotal_mb"
ansible web-server -m setup -a "filter=ansible_default_ipv4"
```
- This shows us OS family, distribution, memory, and IP address.

### Use Facts in a Playbook - Create `facts-demo.yml`:
```yaml
---
- name: Facts demo
  hosts: all
  tasks:
    - name: Show OS info
      debug:
        msg: >
          Hostname: {{ ansible_hostname }},
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }},
          RAM: {{ ansible_memtotal_mb }}MB,
          IP: {{ ansible_default_ipv4.address }}

    - name: Show all network interfaces
      debug:
        var: ansible_interfaces
```

### Run the playbook
```bash
ansible-playbook facts-demo.yml
```
- Output prints hostname, OS, RAM, IP for each host.
- Second task dumps all interfaces (eth0, lo, etc.).

### Verify
Check that values match reality:
- Hostname → `hostname` command on server.
- OS → `cat /etc/os-release`.
- RAM → `free -m`.
- IP → `ip addr show`.

Interfaces list should include loopback + primary NIC.

### Name five facts you would use in real playbooks and why.
**Ans.** Five useful facts in real playbooks:
- **ansible_distribution** → Run OS‑specific tasks (e.g., `apt` vs `yum`).
- **ansible_memtotal_mb** → Apply memory‑based tuning (swap, JVM heap).
- **ansible_default_ipv4** → Bind services to correct IP.
- **ansible_hostname** → Use in naming conventions or reports.
- **ansible_interfaces** → Configure networking, firewalls, or monitoring.

These facts let us write **adaptive playbooks** that configure servers differently based on their actual hardware and OS.

## 4. Conditionals with when
Tasks should not always run on every host. Use `when` to control execution.

Use `when` to run a task only when a condition is true. Tasks that don't match the condition show `skipping` in the output — they are not run, not failed.

### Create the playbook
Inside our `playbooks/` directory, create `conditional-demo.yml`:
```yaml
---
- name: Conditional tasks demo
  hosts: all
  become: true

  tasks:
    - name: Install Nginx (only on web servers)
      yum:
        name: nginx
        state: present
      when: "'web' in group_names"

    - name: Install MySQL (only on db servers)
      yum:
        name: mysql-server
        state: present
      when: "'db' in group_names"

    - name: Show warning on low memory hosts
      debug:
        msg: "WARNING: This host has less than 1GB RAM"
      when: ansible_memtotal_mb < 1024

    - name: Run only on Amazon Linux
      debug:
        msg: "This is an Amazon Linux machine"
      when: ansible_distribution == "Amazon"

    - name: Run only on Ubuntu
      debug:
        msg: "This is an Ubuntu machine"
      when: ansible_distribution == "Ubuntu"

    - name: Run only in production
      debug:
        msg: "Production settings applied"
      when: app_env == "production"

    - name: Multiple conditions (AND)
      debug:
        msg: "Web server with enough memory"
      when:
        - "'web' in group_names"
        - ansible_memtotal_mb >= 512

    - name: OR condition
      debug:
        msg: "Either web or app server"
      when: "'web' in group_names or 'app' in group_names"
```

### Run the playbook
```bash
ansible-playbook playbooks/conditional-demo.yml
```

### Observe the output
On **web servers**:
- Nginx installed.
- “Web server with enough memory” runs if RAM ≥ 512MB.
- OR condition runs (web group).

On **db servers**:
- MySQL installed.
- OR condition runs if host is in `app` or `web`.

On **low‑memory hosts (<1GB)**:
- Warning message printed.

On **Amazon Linux**:
- “This is an Amazon Linux machine” printed.

On **Ubuntu**:
- “This is an Ubuntu machine” printed.

On **production environment (`app_env=production`)**:
- “Production settings applied” printed.

Tasks that don’t match conditions will show **“skipping”** in the output.

### Verify correctness
SSH into different hosts and confirm:
- Web servers have Nginx installed.
- DB servers have MySQL installed.
- Hosts with <1GB RAM show warning.
- OS‑specific messages match distribution.
- Production environment message only shows when `app_env` is set to `production`.

### Important syntax rules
- **Conditionals syntax:** `when:` followed by expression.
- **No Jinja braces needed** → write `when: app_env == "production"`, not `when: "{{ app_env }} == 'production'"`.
- **Built‑in variable:** `group_names` contains all groups the host belongs to.
- **Multiple conditions:** Use list for AND, use `or` for OR.

This is how you prevent **unnecessary installs** and tailor tasks per host.

## 5. Loops
Loops eliminate repetition. Instead of writing the same task 10 times with different values, define one task and iterate over a list.

### Create the playbook
Inside our `playbooks/` directory, create `loops-demo.yml`:

```yaml
---
- name: Loops demo
  hosts: all
  become: true

  vars:
    users:
      - name: deploy
        groups: wheel
      - name: monitor
        groups: wheel
      - name: appuser
        groups: users

    directories:
      - /opt/app/logs
      - /opt/app/config
      - /opt/app/data
      - /opt/app/tmp

  tasks:
    - name: Create multiple users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      loop: "{{ users }}"

    - name: Create multiple directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop: "{{ directories }}"

    - name: Install multiple packages
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - curl
        - unzip
        - jq

    - name: Print each user created
      debug:
        msg: "Created user {{ item.name }} in group {{ item.groups }}"
      loop: "{{ users }}"
```

### Run the playbook
```bash
ansible-playbook playbooks/loops-demo.yml
```

### Observe the output
- **User creation:** Each user (`deploy`, `monitor`, `appuser`) is created in its group.
- **Directories:** `/opt/app/logs`, `/opt/app/config`, `/opt/app/data`, `/opt/app/tmp` created with mode `0755`.
- **Packages:** `git`, `curl`, `unzip`, `jq` installed.
- **Debug messages:** Prints one line per user, e.g.
    ```Code
    Created user deploy in group wheel
    Created user monitor in group wheel
    Created user appuser in group users
    ```

Each loop iteration is shown separately in the output.

### Verify correctness
SSH into a host and check:
```bash
id deploy && id monitor && id appuser    # users created
ls -ld /opt/app/*                        # directories created with 0755
rpm -q git curl unzip jq                 # packages installed (RHEL/CentOS)
```

### What is the difference between `loop` and the older `with_items`?
- **Loop syntax:** `loop:` is the modern, unified way to iterate.
- **Older syntax:** `with_items:` was used in older versions.
- **Difference:**
    - `loop` supports complex data structures (dicts, lists, nested).
    - `with_items` was limited to simple lists.
    - `loop` is the **recommended syntax** going forward.

## 6. Register, Debug, and Combine Everything
This playbook combines everything from today — variables, facts, conditionals, register, and debug — into a practical server health report.

### Create the playbook
Inside our `playbooks/` directory, create `server-report.yml`:
```yaml
---
- name: Server Health Report
  hosts: all

  tasks:
    - name: Check disk space
      command: df -h /
      register: disk_result

    - name: Check memory
      command: free -m
      register: memory_result

    - name: Check running services
      shell: systemctl list-units --type=service --state=running | head -20
      register: services_result

    - name: Generate report
      debug:
        msg:
          - "========== {{ inventory_hostname }} =========="
          - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
          - "IP: {{ ansible_default_ipv4.address }}"
          - "RAM: {{ ansible_memtotal_mb }}MB"
          - "Disk: {{ disk_result.stdout_lines[1] }}"
          - "Running services (first 20): {{ services_result.stdout_lines | length }}"

    - name: Flag if disk is critically low
      debug:
        msg: "ALERT: Check disk space on {{ inventory_hostname }}"
      when: "'9[0-9]%' in disk_result.stdout or '100%' in disk_result.stdout"

    - name: Save report to file
      copy:
        content: |
          Server: {{ inventory_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address }}
          RAM: {{ ansible_memtotal_mb }}MB
          Disk: {{ disk_result.stdout }}
          Checked at: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/server-report-{{ inventory_hostname }}.txt"
      become: true
```

### Run the playbook
```bash
ansible-playbook playbooks/server-report.yml
```

### Observe the output
- **Disk check:** `disk_result` captures `df -h /`.
- **Memory check:** `memory_result` captures `free -m`.
- **Services check:** `services_result` captures first 20 running services.
- **Debug report:** Prints OS, IP, RAM, disk line, and service count.
- **Alert:** If disk usage ≥ 90%, prints warning.
- **File saved:** `/tmp/server-report-<hostname>.txt` created on each host.

### Verify correctness
SSH into a host:
```bash
cat /tmp/server-report-<hostname>.txt
```

Confirm:
- Hostname matches inventory.
- OS and version correct (`cat /etc/os-release`).
- IP matches `ip addr show`.
- RAM matches `free -m`.
- Disk usage matches `df -h /`.
- Timestamp is current.

### How the playbook combines every concept
- **register**: Stores command output in a variable (`stdout`, `stderr`, `rc`, `stdout_lines`).
- **debug**: Formats structured messages for readability.
- **facts**: Provide OS, IP, RAM automatically.
- **conditionals**: Alert only when disk usage is high.
- **copy**: Persists report to file per host.

This is the standard **adaptive health check** pattern - one playbook that behaves differently on every host based on its actual state, and saves evidence to disk.

