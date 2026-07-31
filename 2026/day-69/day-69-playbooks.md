# Ansible Playbooks and Modules
A hands-on guide to writing your first playbook, understanding playbook structure, mastering essential modules, using handlers, previewing changes safely, and orchestrating multiple server groups in one playbook.
 
## Table of Contents
- [Your First Playbook](#1-your-first-playbook)
- [Understand the Playbook Structure](#2-understand-the-playbook-structure)
- [Learn the Essential Modules](#3-learn-the-essential-modules)
- [Handlers — Restart Services Only When Needed](#4-handlers--restart-services-only-when-needed)
- [Dry Run, Diff, and Verbosity](#5-dry-run-diff-and-verbosity)
- [Multiple Plays in One Playbook](#6-multiple-plays-in-one-playbook)

## 1. Your First Playbook
### Create Playbook File
We need a YAML file to define automation tasks.
```bash
#On our control node
vim install-nginx.yml
```

```yaml
---
- name: Install and start Nginx on web servers
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      yum:                                    #use apt if running Ubuntu
        name: nginx
        state: present

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: true

    - name: Create a custom index page
      copy:
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1>"
        dest: /usr/share/nginx/html/index.html
```

### Run Playbook
```bash
#Execute the playbook to apply changes.
ansible-playbook install-nginx.yml
```
- Watch the output — tasks show `changed`, `ok`, or `failed`.
 
### Run it a **second time** without changing anything:
```bash
ansible-playbook install-nginx.yml
```
 
Tasks now show `ok` instead of `changed`. This is **idempotency** - Ansible compares the current state of each resource against the desired state and only makes changes when there is a difference.
 
| Run | Expected task status | Why |
|---|---|---|
| First | `changed` | Nginx wasn't installed; file didn't exist |
| Second | `ok` | Nginx is already installed and running; file already in place |
 
### Verify Deployment
Confirm Nginx is serving your custom page.
```bash
curl <web-server-public-ip>
```
- We should see: `<h1>Deployed by Ansible - TerraWeek Server</h1>`
- If the page doesn't load, check that `port 80` is open in the EC2 security group.

## 2. Understand the Playbook Structure
### Anatomy of a playbook
```yaml
---
- name: Install and start Nginx on web servers
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: true

    - name: Create a custom index page
      copy:
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1>"
        dest: /usr/share/nginx/html/index.html
```

Answer:
1. What is the difference between a play and a task?
    - A **play** maps a group of hosts to a set of tasks. Think of it as the big picture: “On these servers, do these things.”
    - A **task** is a single unit of work inside a play, executed by a module (e.g., install a package, copy a file).

2. Can you have multiple plays in one playbook?
    - Yes, one playbook can contain multiple plays. Example: one play for web servers, another for database servers, all in the same YAML file.

3. What does `become: true` do at the play level vs the task level?
    - At the **play level**: applies to all tasks in that play.
    - At the **task level**: applies only to that specific task. Useful if most tasks don’t need root, but one does.

4. What happens if a task fails -- do remaining tasks still run?
    - By default, if a task fails, Ansible stops executing further tasks on that host.
    - You can override with `ignore_errors: true` if you want the play to continue despite failure.

## 3. Learn the Essential Modules
### Create supporting files
```bash
mkdir files
```
 
`files/app.conf`:
```ini
[app]
setting1 = true
setting2 = 42
```
 
### `essential-modules.yml`
Practice each of these modules by writing a playbook in our Ansible project directory with multiple tasks:
```yaml
---
- name: Practice essential Ansible modules
  hosts: all
  become: true
 
  tasks:
    # 1. Package management
    - name: Install multiple packages
      yum:          # use apt if Ubuntu
        name:
          - git
          - curl
          - wget
          - tree
        state: present
 
    # 2. Service management
    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true
 
    # 3. Copy files from control node to managed nodes
    - name: Copy config file
      copy:
        src: files/app.conf
        dest: /etc/app.conf
        owner: root
        group: root
        mode: '0644'
 
    # 4. Create directories and manage permissions
    - name: Create application directory
      file:
        path: /opt/myapp
        state: directory
        owner: ec2-user
        mode: '0755'
 
    # 5. Run a command (no shell features — safer)
    - name: Check disk space
      command: df -h
      register: disk_output
 
    - name: Print disk space
      debug:
        var: disk_output.stdout_lines
 
    # 6. Run a shell command (pipes, redirects allowed)
    - name: Count running processes
      shell: ps aux | wc -l
      register: process_count
 
    - name: Show process count
      debug:
        msg: "Total processes: {{ process_count.stdout }}"
 
    # 7. Add or modify a single line in a file
    - name: Set timezone in environment
      lineinfile:
        path: /etc/environment
        line: 'TZ=Asia/Kolkata'
        create: true
```
### Essential module reference
| Module | What it does | Key arguments |
|---|---|---|
| `yum` / `apt` | Install or remove packages | `name`, `state: present/absent` |
| `service` | Start, stop, enable, or disable a service | `name`, `state`, `enabled` |
| `copy` | Push a file from the control node to managed nodes | `src`, `dest`, `owner`, `mode` |
| `file` | Create directories, set permissions, manage symlinks | `path`, `state: directory/file`, `mode` |
| `command` | Run a binary directly — no shell features | Command string, `register` |
| `shell` | Run a command with pipes, redirects, variables | Shell string, `register` |
| `lineinfile` | Ensure a specific line exists (or doesn't) in a file | `path`, `line`, `regexp`, `create` |
| `debug` | Print a variable or message to the terminal | `var`, `msg` |

### Run the Playbook
Execute:
```bash
ansible-playbook essential-modules.yml
```
Watch the output:
- **changed** → something was modified (package installed, file copied).
- **ok** → already in desired state.
- **failed** → error occurred.

### What is the difference between `command` and `shell`? When should you use each?
- **command**: Runs a binary directly, no shell features. Safer, avoids injection risks. Example: `command: df -h`.
- **shell**: Runs inside a shell, allows pipes (`|`), redirects (`>`), variables. Example: `shell: ps aux | wc -l`.
- **Rule of thumb**: Use `command` whenever possible. Use `shell` only when you need shell features.

This task shows you the **core modules** that cover 80% of automation needs:
- Package management
- Service control
- File copy and directory creation
- Running commands safely
- Editing configuration lines

## 4. Handlers -- Restart Services Only When Needed
A handler is a task that runs **only when notified** by another task. This prevents unnecessary service restarts - Nginx only restarts if the config file actually changed.

### Create the Playbook File `nginx-config.yml`:
```yaml
---
- name: Configure Nginx with a custom config
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Deploy Nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
      notify: Restart Nginx

    - name: Deploy custom index page
      copy:
        content: "<h1>Managed by Ansible</h1><p>Server: {{ inventory_hostname }}</p>"
        dest: /usr/share/nginx/html/index.html

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

### Create Supporting Config File
Create `files/nginx.conf` with a basic Nginx configuration, for example:
```nginx
events { }

http {
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
```

### Run the Playbook Twice
Execute:
```bash
ansible-playbook nginx-config.yml
```
### Handler behavior across runs
| Run | Config file changed? | Handler triggered? | Result |
|---|---|---|---|
| First | ✓ Yes (new file deployed) | ✓ Yes | Nginx restarts |
| Second | ✗ No (file identical) | ✗ No | Nginx stays running — no restart |
 
### Key handler rules
- Handlers run **at the end of the play**, not immediately when notified
- If the same handler is notified multiple times in one play, it runs **only once**
- If no task notifies a handler, the handler **never runs**
- Handlers are defined at the same level as `tasks:`, under a `handlers:` key
```bash
# Verify Nginx is active after both runs
systemctl status nginx
```

## 5. Dry Run, Diff, and Verbosity
Before running playbooks on production, always preview changes first.

### Dry Run (Check Mode)
Run your playbook in **check mode**:
```bash
ansible-playbook install-nginx.yml --check
```
- This simulates the run.
- Output shows what would change (marked as `changed`) but doesn’t actually apply changes.
- Useful for previewing effects without risk.

### Diff Mode
Combine check mode with diff:
```bash
ansible-playbook nginx-config.yml --check --
```
- Shows **file differences** line by line.
- Example: if `nginx.conf` changes, you’ll see old vs new content.
- Critical for config files - you know exactly what will be replaced.

### Verbosity Levels
Increase output detail for debugging:
```bash
ansible-playbook install-nginx.yml -v    # basic verbose
ansible-playbook install-nginx.yml -vv   # more detail
ansible-playbook install-nginx.yml -vvv  # connection debugging
```
- Use `-v` when you want to see what each task is 
- Use `-vv` when you want to see module arguments and return values
- Use `-vvv` if troubleshooting SSH or connection issues.

### Limit Hosts
Target specific hosts only:
```bash
ansible-playbook install-nginx.yml --limit web-server
```
- Runs playbook only on `web-server` host.
- Useful for testing changes on one machine before rolling out cluster‑wide.

### List Hosts and Tasks
Preview what would run:
```bash
ansible-playbook install-nginx.yml --list-hosts
ansible-playbook install-nginx.yml --list-tasks
```
- `--list-hosts`: shows which servers will be affected.
- `--list-tasks`: shows which tasks will run.
- No changes are applied - pure preview.

### Why is `--check --diff` the most important flag combination for production use?
- **Check mode** ensures no changes are applied - safe preview.
- **Diff mode** shows exactly what files/configs would change.
- Together, they give you a **risk‑free, detailed preview** of changes before touching production.
- Prevents accidental overwrites, service downtime, or misconfigurations.

## 6. Multiple Plays in One Playbook
A single playbook can contain multiple plays, each targeting a different inventory group. Plays execute in order - Play 1 completes on all its hosts before Play 2 begins.

Write `multi-play.yml` with separate plays for each server group:
```yaml
---
# Play 1: Web servers
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      yum:   # use apt if Ubuntu
        name: nginx
        state: present
    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: true

# Play 2: App servers
- name: Configure app servers
  hosts: app
  become: true
  tasks:
    - name: Install Node.js dependencies
      yum:
        name:
          - gcc
          - make
        state: present
    - name: Create app directory
      file:
        path: /opt/app
        state: directory
        mode: '0755'

# Play 3: Database servers
- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Install MySQL client
      yum:
        name: mysql
        state: present
    - name: Create data directory
      file:
        path: /var/lib/appdata
        state: directory
        mode: '0700'
```

### Run the Playbook
Execute:
```bash
ansible-playbook multi-play.yml
```
- Ansible will run **Play 1** on all hosts in the `web` group.
- Then **Play 2** on all hosts in the `app` group.
- Finally **Play 3** on all hosts in the `db` group.

Each play runs independently, targeting only its inventory group.

### Watch the Output
- You’ll see three separate sections in the output, one per play.
- Tasks show `changed` or `ok` depending on whether something was modified.
- Example: On web servers, you’ll see Nginx installed and started. On db servers, you’ll see MySQL client installed.

### Verify Results
On each group of servers:
- **Web servers:**
    ```bash
    systemctl status nginx
    ```
    - Nginx should be active.

- **App servers:**
    ```bash
    ls -ld /opt/app
    ```
    - Directory exists with mode `0755`.

- **Database servers:**
    ```bash
    mysql --version
    ls -ld /var/lib/appdata
    ```
    - MySQL client installed, directory exists with mode `0700`.

### Multi-play output structure
The output shows three clearly labelled sections — one `PLAY` block per play, each with its own `TASK` entries and a `PLAY RECAP` summary line. This makes it easy to see at a glance which changes were applied to which server group.
 
| Group | Expected changes |
|---|---|
| Web (`web`) | Nginx installed and started |
| App (`app`) | `gcc`, `make` installed; `/opt/app` created |
| DB (`db`) | MySQL client installed; `/var/lib/appdata` created with `0700` permissions |