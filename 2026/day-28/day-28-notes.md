# Day 28 - Linux, Shell Scripting & Git Review

---

## Task 1: Self-Assessment Checklist

---

### Linux

---

#### ✅ Navigate the file system, create/move/delete files and directories

```bash
pwd                        # where am I?
ls -la                     # list all files with permissions
cd /etc                    # change directory
cd ~                       # go home
cd -                       # go back to previous directory

mkdir -p projects/api      # create nested directories
touch app.py               # create empty file
cp file.txt backup.txt     # copy
mv file.txt /tmp/          # move (also used to rename)
rm file.txt                # delete file
rm -rf old-project/        # delete directory recursively ⚠️

find /var/log -name "*.log" -mtime +7   # find logs older than 7 days
```

---

#### ✅ Manage processes — list, kill, background/foreground

```bash
ps aux                     # all running processes
ps aux | grep nginx        # find a specific process
top                        # live process monitor
htop                       # nicer live monitor (if installed)

kill 1234                  # graceful kill (SIGTERM)
kill -9 1234               # force kill (SIGKILL) ⚠️ last resort
killall nginx              # kill all processes by name
pkill -f "python app.py"   # kill by matching command string

sleep 60 &                 # run in background
jobs                       # list background jobs
fg %1                      # bring job 1 to foreground
bg %1                      # push stopped job to background
Ctrl+Z                     # suspend current foreground process
Ctrl+C                     # interrupt and terminate foreground process

nohup python app.py &      # run immune to hangup signals (survives logout)
```

---

#### ✅ Work with systemd — start, stop, enable, check status of services

```bash
systemctl start nginx          # start a service now
systemctl stop nginx           # stop a service now
systemctl restart nginx        # stop then start
systemctl reload nginx         # reload config without full restart
systemctl status nginx         # show status, last logs, PID

systemctl enable nginx         # start automatically at boot
systemctl disable nginx        # remove from boot sequence
systemctl is-enabled nginx     # check if it's set to start at boot
systemctl is-active nginx      # check if currently running

systemctl list-units --type=service --state=running   # list all running services

journalctl -u nginx            # all logs for nginx
journalctl -u nginx -f         # follow logs live

journalctl -u nginx --since "1 hour ago"

journalctl -xe                 # system-wide error log with context
```

---

#### ✅ Read and edit text files using vim or nano

**nano** (beginner-friendly):
```bash
nano file.txt        # open file
# Ctrl+O → save
# Ctrl+X → exit
# Ctrl+W → search
# Ctrl+K → cut line
# Ctrl+U → paste line
```

**vim** (powerful, worth learning):
```bash
vim file.txt         # open file

# MODES:
# Normal mode  → default, navigate and run commands
# Insert mode  → press i to enter, Esc to exit
# Visual mode  → press v to select text

# Normal mode commands:
:w           # save
:q           # quit
:wq          # save and quit
:q!          # quit without saving
dd           # delete current line
yy           # yank (copy) current line
p            # paste below
u            # undo
Ctrl+r       # redo
/word        # search forward
n            # next search result
:%s/old/new/g  # find and replace all occurrences
gg           # go to top of file
G            # go to bottom of file
:42          # go to line 42
```

---

#### ✅ Troubleshoot CPU, memory, and disk issues

```bash
# CPU
top                        # live — press P to sort by CPU
htop                       # visual top
uptime                     # load averages (1, 5, 15 min)
mpstat 1 5                 # CPU stats per second

# Memory
free -h                    # total, used, free RAM + swap
cat /proc/meminfo          # detailed memory breakdown
vmstat 1                   # virtual memory stats

# Disk
df -h                      # disk usage per filesystem
du -sh /var/log/*          # size of each item in /var/log
du -sh * | sort -rh        # largest directories, sorted
lsblk                      # block device layout
iostat 1                   # disk I/O stats per second

# Combined
sar -u 1 5                 # CPU, memory, I/O history (needs sysstat)
dmesg | tail -50           # kernel ring buffer — disk/hardware errors here
```

---

#### ✅ Explain the Linux file system hierarchy

| Path | Purpose |
|------|---------|
| `/` | Root — everything starts here |
| `/bin` | Essential user binaries (`ls`, `cp`, `bash`) — needed to boot |
| `/sbin` | System binaries for root (`fdisk`, `iptables`) |
| `/usr` | User programs and libraries (most installed software lives here) |
| `/usr/local` | Locally compiled/installed software — not managed by the package manager |
| `/etc` | System-wide configuration files (`/etc/nginx/`, `/etc/fstab`) |
| `/var` | Variable data — logs (`/var/log`), databases, spool, mail |
| `/home` | User home directories (`/home/alice`, `/home/bob`) |
| `/root` | Home directory for the root user |
| `/tmp` | Temporary files — cleared on reboot |
| `/proc` | Virtual filesystem — exposes kernel and process state as files |
| `/sys` | Virtual filesystem — hardware and driver information |
| `/dev` | Device files (`/dev/sda`, `/dev/null`, `/dev/random`) |
| `/opt` | Optional third-party applications |
| `/mnt` | Temporary mount points for external drives |
| `/boot` | Bootloader and kernel images |
| `/lib` | Shared libraries needed by `/bin` and `/sbin` |

---

#### ✅ Create users and groups, manage passwords

```bash
# Users
useradd alice                          # create user (no home dir by default)
useradd -m -s /bin/bash alice          # create with home dir and bash shell
usermod -aG docker alice               # add alice to docker group
usermod -s /bin/zsh alice              # change default shell
userdel alice                          # delete user
userdel -r alice                       # delete user and their home directory

# Passwords
passwd alice                           # set/change password for alice
passwd -l alice                        # lock account
passwd -u alice                        # unlock account
chage -l alice                         # show password expiry info
chage -M 90 alice                      # force password change every 90 days

# Groups
groupadd devteam                       # create group
groupdel devteam                       # delete group
groups alice                           # list groups alice belongs to
id alice                               # UID, GID, all groups

# User info
cat /etc/passwd                        # all users
cat /etc/group                         # all groups
who                                    # who is currently logged in
last                                   # login history

# Sudo
visudo                                 # safely edit /etc/sudoers
usermod -aG sudo alice                 # give alice sudo (Debian/Ubuntu)
usermod -aG wheel alice                # give alice sudo (RHEL/CentOS)
```

---

#### ✅ Set file permissions using chmod

**Permission structure:**
```
-rwxr-xr--
│└──┬──┘└──┬──┘└──┬──┘
│  owner  group  others
│
└─ type: - = file, d = directory, l = symlink
```

**Numeric (octal):**
```
4 = read (r)
2 = write (w)
1 = execute (x)

chmod 755 script.sh    # rwxr-xr-x → owner: full, group/others: read+execute
chmod 644 config.txt   # rw-r--r-- → owner: read+write, others: read only
chmod 600 id_rsa       # rw------- → owner only (correct for SSH private keys)
chmod 777 file         # rwxrwxrwx → everyone full access ⚠️ almost never correct
chmod 000 secret       # --------- → no one can access
```

**Symbolic:**
```bash
chmod +x script.sh         # add execute for everyone
chmod u+x script.sh        # add execute for owner only
chmod g-w file.txt         # remove write from group
chmod o=r file.txt         # set others to read-only exactly
chmod a+r file.txt         # add read for all (a = all)
chmod u=rwx,g=rx,o= file   # set each class explicitly
```

**Directories:**
- `r` on a directory = can list contents (`ls`)
- `w` on a directory = can create/delete files inside
- `x` on a directory = can `cd` into it and access files by name

---

#### ✅ Change file ownership with chown and chgrp

```bash
chown alice file.txt            # change owner to alice
chown alice:devteam file.txt    # change owner and group
chown :devteam file.txt         # change group only
chown -R alice:alice /app       # recursive — entire directory tree

chgrp devteam file.txt          # change group only (alternative to chown :group)
chgrp -R devteam /var/www       # recursive

ls -la                          # verify ownership
stat file.txt                   # detailed ownership + permissions info
```

---

#### ✅ Create and manage LVM volumes

```bash
# Physical Volume (PV) — raw disk/partition prepared for LVM
pvcreate /dev/sdb               # initialise disk for LVM
pvs                             # list physical volumes
pvdisplay /dev/sdb              # detailed info

# Volume Group (VG) — pool of physical volumes
vgcreate datavg /dev/sdb        # create VG named datavg
vgextend datavg /dev/sdc        # add another disk to the pool
vgs                             # list volume groups

# Logical Volume (LV) — flexible "partition" carved from the VG
lvcreate -L 20G -n applv datavg       # create 20GB LV named applv
lvcreate -l 100%FREE -n datalv datavg # use all remaining space
lvs                                    # list logical volumes
lvdisplay /dev/datavg/applv

# Format and mount
mkfs.ext4 /dev/datavg/applv           # format
mount /dev/datavg/applv /mnt/app      # mount
echo "/dev/datavg/applv /mnt/app ext4 defaults 0 0" >> /etc/fstab  # persist

# Extend (online — no unmounting needed for ext4/xfs)
lvextend -L +10G /dev/datavg/applv    # add 10GB to LV
resize2fs /dev/datavg/applv           # resize ext4 to fill new space
xfs_growfs /mnt/app                   # resize xfs to fill new space

# Snapshot
lvcreate -L 5G -s -n appsnap /dev/datavg/applv  # take a snapshot
```

---

#### ✅ Check network connectivity

```bash
# Basic connectivity
ping google.com                    # ICMP echo — is the host reachable?
ping -c 4 google.com               # send exactly 4 pings
traceroute google.com              # hop-by-hop path to host
mtr google.com                     # live traceroute (if installed)

# HTTP / service testing
curl -I https://example.com        # HTTP headers only
curl -v https://example.com        # verbose — full request/response
curl -o /dev/null -w "%{http_code}" https://example.com  # just the status code
wget https://example.com/file.tar.gz   # download a file

# Ports and sockets
ss -tulnp                          # listening TCP/UDP ports + process names
ss -tnp                            # all TCP connections + process
netstat -tulnp                     # older equivalent (may need net-tools)
lsof -i :8080                      # what's using port 8080?

# DNS
dig google.com                     # full DNS query output
dig google.com +short              # just the IP
dig MX gmail.com                   # query a specific record type
nslookup google.com                # simpler DNS lookup
host google.com                    # quick hostname → IP

# Network interfaces
ip a                               # show all interfaces and IPs
ip r                               # routing table
ifconfig                           # older equivalent
```

---

#### ✅ Explain DNS, IP addressing, subnets, and common ports

**DNS Resolution (what happens when you type google.com):**
1. Browser checks its own cache.
2. OS checks `/etc/hosts`.
3. OS asks the **recursive resolver** (usually your router or ISP's DNS, e.g. 8.8.8.8).
4. Resolver asks a **root nameserver** → directed to the `.com` TLD server.
5. TLD server → directed to Google's **authoritative nameserver**.
6. Authoritative nameserver returns the A record (IP address).
7. Resolver caches it (per TTL) and returns it to the OS.

**IP Addressing:**
- IPv4: 32-bit, written as four octets: `192.168.1.10`
- Private ranges (not routed on public internet): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- IPv6: 128-bit, written as eight hex groups: `2001:0db8::1`
- Loopback: `127.0.0.1` (always "this machine")

**Subnets (CIDR notation):**
- `/24` = 255.255.255.0 → 254 usable hosts (e.g. `192.168.1.0/24`)
- `/16` = 255.255.0.0 → 65,534 usable hosts
- `/32` = single host (one specific IP)
- The number after `/` is how many bits are the network prefix — the rest are host bits.

**Common Ports:**

| Port | Protocol | Service |
|------|---------|---------|
| 22 | TCP | SSH |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 27017 | TCP | MongoDB |
| 8080 | TCP | HTTP alt / dev servers |

---

### Shell Scripting

---

#### ✅ Write a script with variables, arguments, and user input

```bash
#!/usr/bin/env bash
set -euo pipefail

# Variables
APP_NAME="myapp"
VERSION="1.0"

# Arguments ($1, $2 ... $N)
# $0 = script name, $# = argument count, $@ = all arguments
if [[ $# -lt 1 ]]; then
  echo "Usage: $0 <environment>"
  exit 1
fi
ENV=$1

# User input
read -p "Enter your name: " USERNAME
read -sp "Enter password: " PASSWORD   # -s = silent (no echo)
echo

echo "Deploying $APP_NAME v$VERSION to $ENV as $USERNAME"
```

---

#### ✅ Use if/elif/else and case statements

```bash
# if/elif/else
if [[ $ENV == "production" ]]; then
  echo "Deploying to prod — extra care!"
elif [[ $ENV == "staging" ]]; then
  echo "Deploying to staging"
else
  echo "Unknown environment: $ENV"
  exit 1
fi

# Common test operators
# -z "$var"   → var is empty
# -n "$var"   → var is not empty
# -f "$file"  → file exists and is a regular file
# -d "$dir"   → directory exists
# -eq, -ne, -lt, -gt  → numeric comparison
# ==, !=       → string comparison

# case statement
case $ENV in
  production)  echo "Running prod checks" ;;
  staging)     echo "Running staging checks" ;;
  dev|local)   echo "Dev mode — skipping checks" ;;
  *)           echo "Unknown: $ENV"; exit 1 ;;
esac
```

---

#### ✅ Write for, while, and until loops

```bash
# for loop — iterate a list
for server in web1 web2 web3; do
  echo "Deploying to $server"
  ssh "$server" "docker pull myapp:latest"
done

# for loop — C-style
for ((i=1; i<=5; i++)); do
  echo "Attempt $i"
done

# for loop — iterate files
for file in /var/log/*.log; do
  echo "Processing $file"
done

# while loop — condition-based
RETRIES=0
while [[ $RETRIES -lt 5 ]]; do
  curl -sf http://localhost/health && break
  RETRIES=$((RETRIES + 1))
  echo "Waiting... attempt $RETRIES"
  sleep 2
done

# until loop — opposite of while (runs until condition is true)
until curl -sf http://localhost/health; do
  echo "Service not ready yet, waiting..."
  sleep 3
done
```

---

#### ✅ Define and call functions with arguments and return values

```bash
#!/usr/bin/env bash
set -euo pipefail

# Define
deploy() {
  local env=$1          # local = scoped to this function
  local version=${2:-"latest"}   # default to "latest" if not passed

  echo "Deploying version $version to $env"

  if [[ $env == "production" ]]; then
    return 0   # success
  else
    return 1   # failure (non-zero = error)
  fi
}

# Call
deploy "staging" "1.2.3"

if deploy "production"; then
  echo "Prod deploy succeeded"
fi

# Capture output
get_container_id() {
  docker ps -qf "name=myapp"
}

CONTAINER_ID=$(get_container_id)
echo "Container: $CONTAINER_ID"
```

---

#### ✅ Use grep, awk, sed, sort, uniq for text processing

```bash
# grep — search for patterns
grep "ERROR" app.log                    # lines containing ERROR
grep -i "error" app.log                 # case-insensitive
grep -n "error" app.log                 # with line numbers
grep -r "TODO" ./src/                   # recursive
grep -v "DEBUG" app.log                 # invert — exclude matches
grep -E "ERROR|WARN" app.log            # regex — either pattern

# awk — column-based processing
awk '{print $1}' access.log             # print first column
awk '{print $1, $7}' access.log         # columns 1 and 7
awk -F: '{print $1}' /etc/passwd        # use : as delimiter → usernames
awk '$9 == 404 {print $7}' access.log   # print URLs that returned 404
awk '{sum += $5} END {print sum}' file  # sum a column

# sed — stream editor / find-replace
sed 's/foo/bar/' file.txt               # replace first occurrence per line
sed 's/foo/bar/g' file.txt             # replace all occurrences
sed -i 's/foo/bar/g' file.txt          # in-place edit (modifies file)
sed -i.bak 's/foo/bar/g' file.txt      # in-place with .bak backup
sed -n '10,20p' file.txt               # print lines 10–20
sed '/^#/d' config.txt                 # delete comment lines

# sort & uniq
sort file.txt                          # alphabetical sort
sort -n numbers.txt                    # numeric sort
sort -rn numbers.txt                   # reverse numeric
sort -k2 file.txt                      # sort by second column
sort -t: -k3 -n /etc/passwd           # sort passwd by UID

sort access.log | uniq                 # remove duplicate lines (must sort first)
sort access.log | uniq -c             # count occurrences
sort access.log | uniq -c | sort -rn  # most common lines first

# Pipeline examples
# Top 10 IP addresses in access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# All unique HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Extract ERROR lines, get 3rd field, count occurrences
grep "ERROR" app.log | awk '{print $3}' | sort | uniq -c | sort -rn
```

---

#### ✅ Handle errors with set -e, set -u, set -o pipefail, trap

```bash
#!/usr/bin/env bash
set -euo pipefail
# -e  → exit immediately if any command returns non-zero
# -u  → treat unset variables as errors (catches typos like $USERNMAE)
# -o pipefail → fail if any command in a pipeline fails
#              (without this: false | true exits 0)

# trap — cleanup on exit or error
TMPFILE=$(mktemp)

cleanup() {
  echo "Cleaning up..."
  rm -f "$TMPFILE"
}
trap cleanup EXIT         # runs on any exit (normal or error)
trap cleanup INT TERM     # also runs on Ctrl+C or kill

# Intentional error handling (override set -e locally)
if ! docker pull myapp:latest; then
  echo "Pull failed, using cached image"
fi

# Check exit code explicitly
curl -sf http://localhost/health
STATUS=$?
if [[ $STATUS -ne 0 ]]; then
  echo "Health check failed with exit code $STATUS"
  exit 1
fi
```

---

#### ✅ Schedule scripts with crontab

```bash
crontab -e       # edit current user's crontab
crontab -l       # list current user's crontab
crontab -r       # remove all cron jobs ⚠️

# Format: minute  hour  day  month  weekday  command
# *       *      *     *      *
# 0-59   0-23   1-31  1-12   0-7 (0 and 7 = Sunday)

# Every day at 3 AM
0 3 * * * /opt/scripts/backup.sh

# Every 15 minutes
*/15 * * * * /opt/scripts/healthcheck.sh

# Monday–Friday at 8 AM
0 8 * * 1-5 /opt/scripts/report.sh

# First day of every month at midnight
0 0 1 * * /opt/scripts/monthly-cleanup.sh

# Every hour
0 * * * * /opt/scripts/sync.sh

# Redirect output to a log file
0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# System-wide crontab (has an extra user field)
# /etc/cron.d/myapp:
# 0 3 * * * root /opt/scripts/backup.sh
```

**Test cron commands before scheduling:**
```bash
run-parts --test /etc/cron.daily     # test which scripts would run
systemctl status cron                # make sure cron is running
grep CRON /var/log/syslog            # cron execution log
```

---

### Git & GitHub

---

#### ✅ Initialize, stage, commit, view history

```bash
git init                          # new repo
git add .                         # stage everything
git add -p                        # interactively stage hunks
git commit -m "feat: add login"   # commit
git log --oneline --graph --all   # visual history
git show abc1234                  # show a specific commit
```

---

#### ✅ Branches, push, pull

```bash
git switch -c feature/auth        # create + switch
git push -u origin feature/auth   # push + set upstream
git pull --rebase                 # pull without merge commits
```

---

#### ✅ Clone vs Fork

- **Clone:** Downloads any repo to your local machine. You can push if you have write access. Used when you're a direct collaborator.
- **Fork:** Creates your own copy of someone else's repo on GitHub. You have full control of the fork. You contribute back via pull requests. Used for open-source projects where you don't have write access to the original.

---

#### ✅ Fast-forward vs merge commit

```bash
# Fast-forward (no divergence — branch is ahead in a straight line)
git merge feature/login              # moves main pointer forward, no commit created
git merge --no-ff feature/login      # force a merge commit even if FF is possible

# Three-way merge commit (branches diverged — Git creates a new commit)
# main:    A → B → C
# feature: A → B → D → E
# result:  A → B → C → D → E → M (M = merge commit)
```

Use `--no-ff` when you want branch history to be visible in `git log --graph`. Fast-forward gives a cleaner linear history but hides that a feature branch existed.

---

#### ✅ Rebase vs merge

```bash
git rebase main             # replay your commits on top of main's latest

# Merge: preserves history exactly — creates a merge commit
# A → B → C → M
#         ↗
#     D → E

# Rebase: rewrites history — your commits appear as if they started from the current tip
# A → B → C → D' → E'
# (D and E become D' and E' — new SHAs, same changes)
```

**When to use rebase:** Cleaning up local feature branches before merging. Keeping a linear history. Syncing a feature branch with main before opening a PR.

**When NOT to rebase:** On shared/public branches. Never rebase commits that others have already pulled — it rewrites SHAs and breaks their history.

---

#### ✅ git stash

```bash
git stash                          # save dirty working tree
git stash push -m "wip: login form"  # stash with a name
git stash list                     # list stashes
git stash pop                      # apply latest stash + remove it
git stash apply stash@{2}          # apply specific stash, keep it
git stash drop stash@{0}           # delete a stash
git stash branch feature/resume    # create a branch from a stash
```

---

#### ✅ Cherry-pick

```bash
git cherry-pick abc1234            # apply a single commit to current branch
git cherry-pick abc1234..def5678   # apply a range of commits
git cherry-pick --no-commit abc1234  # apply changes without committing (review first)
```

**Use case:** A bug was fixed on `main` but your release branch needs the same fix without pulling everything else from `main`.

---

#### ✅ Squash merge vs regular merge

| | Regular merge | Squash merge |
|---|---|---|
| History | All commits preserved | All commits collapsed into one |
| `git log` | Shows every WIP commit | Clean single commit per feature |
| Attribution | Individual authors visible | Lost in the squash |
| When to use | Long-lived branches | Short feature branches with messy history |

```bash
git merge --squash feature/login   # squash all commits into staging area
git commit -m "feat: add login"    # write a clean commit message
```

---

#### ✅ git reset vs git revert

| | `git reset` | `git revert` |
|---|---|---|
| What it does | Moves HEAD backward, rewrites history | Creates a new commit that undoes changes |
| Safe on shared branches? | ❌ No — changes SHAs others have pulled | ✅ Yes — adds to history, doesn't rewrite it |
| Recoverable? | Only via `reflog` within ~90 days | Yes — always, it's just another commit |

```bash
git reset --soft HEAD~1    # undo commit, keep changes staged
git reset --mixed HEAD~1   # undo commit, keep changes unstaged (default)
git reset --hard HEAD~1    # undo commit, discard all changes ⚠️ destructive

git revert abc1234         # create a new "undo" commit — safe for shared branches
```

---

#### ✅ GitFlow, GitHub Flow, and Trunk-Based Development

**GitFlow:**
Long-lived branches: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`. Structured and predictable. Good for software with versioned releases (mobile apps, packaged software). Complex — many branches to manage.

**GitHub Flow:**
One main branch. Feature branches off main, short-lived. Merge via PR, deploy immediately. Simple. Works well for web apps deploying continuously. No dedicated develop or release branches.

**Trunk-Based Development:**
Everyone commits directly to `main` (or very short-lived branches, merged in hours). Feature flags control what users see. Requires strong CI/CD and disciplined testing. Used by Google, Facebook. Enables true Continuous Deployment.

---

#### ✅ GitHub CLI

```bash
gh auth login                                    # authenticate
gh repo create my-project --public --clone       # create + clone repo
gh repo clone owner/repo                         # clone a repo

gh pr create --title "feat: login" --body "Adds login flow"
gh pr list                                       # list open PRs
gh pr view 42                                    # view PR #42
gh pr merge 42 --squash                          # merge PR with squash
gh pr checkout 42                                # check out a PR branch locally

gh issue create --title "Bug: login fails" --label bug
gh issue list                                    # list open issues
gh issue close 15                                # close issue

gh workflow list                                 # list GitHub Actions workflows
gh run list                                      # list recent pipeline runs
gh run watch                                     # watch current run live
```

---

## Task 2: Quick-Fire Questions

---

**1. What does `chmod 755 script.sh` do?**

**Ans.** Sets the owner to `rwx` (read, write, execute), and group and others to `r-x` (read and execute - no write).

The owner can edit and run it; everyone else can run it but not modify it. The standard permission for executable scripts and web server directories.

---

**2. What is the difference between a process and a service?**<br>
**Ans.** <br>
A **process** is any running program - a single instance of an executable, with its own PID, memory space, and CPU time. It exists for the duration of its execution and then exits.

A **service** is a process designed to run continuously in the background, managed by systemd (or another init system). Services start automatically at boot, restart on failure, and have structured management commands.

All services are processes, but not all processes are services.

---

**3. How do you find which process is using port 8080?**

```bash
ss -tulnp | grep :8080
# or
lsof -i :8080
# or
fuser 8080/tcp
```

All three show the PID and process name. `ss` is the modern standard; `lsof` gives more detail; `fuser` is the most minimal.

---

**4. What does `set -euo pipefail` do in a shell script?**<br>
**Ans.**
Three independent flags combined:
- `-e` — the script exits immediately when any command returns a non-zero exit code, instead of silently continuing on failure.
- `-u` — treats unset variables as errors. `echo $USERNMAE` (typo) would crash the script rather than silently expand to an empty string.
- `-o pipefail` — makes a pipeline fail if *any* command in it fails, not just the last one. Without this, `false | true` exits 0 because `true` is the last command.

Together they make shell scripts behave like real programming languages with error handling rather than silently doing the wrong thing.

---

**5. What is the difference between `git reset --hard` and `git revert`?**<br>
**Ans.**<br>
`git reset --hard` moves the branch pointer backward and destroys any commits after that point. It rewrites history - if anyone else has pulled those commits, their repo now diverges from yours. Destructive, fast, only safe on local work you haven't shared.

`git revert` creates a *new commit* that applies the inverse of the commit you want to undo. History is preserved - the mistake and the fix are both visible. Safe to use on any branch, including shared ones, because it only adds to history rather than rewriting it.

---

**6. What branching strategy would you recommend for a team of 5 developers shipping weekly?**<br>
**Ans.** **GitHub Flow.**<br>
Short-lived feature branches off `main`, merged via pull request with CI checks. A simple strategy that keeps everyone close to the main branch, avoids long-running merge conflicts, and maps naturally onto weekly releases. GitFlow's added complexity (`develop`, `release/*`) isn't worth it at this team size. Trunk-based development works too, but requires strong discipline and feature flag infrastructure that a small team might not have yet.

---

**7. What does `git stash` do and when would you use it?**<br>
**Ans.**<br>
`git stash` saves your uncommitted changes (both staged and unstaged) onto a temporary stack and restores the working tree to a clean state matching HEAD. Your changes aren't lost — they're stashed and can be reapplied with `git stash pop`.

**Use it when:** You're mid-feature and need to quickly switch branches to fix an urgent bug. Or when you want to pull the latest changes from remote but your uncommitted work would conflict with the merge.

---

**8. How do you schedule a script to run every day at 3 AM?**<br>
**Ans.**
```bash
crontab -e
# Add the line:
0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

The cron format is `minute hour day month weekday`. `0 3 * * *` means: at minute 0 of hour 3, every day, every month, every day of the week. The `>> /var/log/backup.log 2>&1` captures both stdout and stderr so you can check whether it ran successfully.

---

**9. What is the difference between `git fetch` and `git pull`?**<br>
**Ans.**<br>
`git fetch` downloads new commits, branches, and tags from the remote but does **not** touch your working directory or current branch. It updates your remote-tracking references (`origin/main`) so you can see what changed without affecting your local work.

`git pull` is `git fetch` followed immediately by a merge (or rebase, with `--rebase`) into your current branch. It downloads *and* integrates in one step.

The habit: `git fetch` when you want to see what's changed without risk. `git pull --rebase` when you're ready to sync and want to keep a clean linear history.

---

**10. What is LVM and why would you use it instead of regular partitions?**<br>
**Ans.**
LVM (Logical Volume Manager) is an abstraction layer between physical disks and filesystems. Instead of partitioning disks directly and being locked into fixed sizes, LVM lets you:

- **Pool multiple disks** into a single Volume Group, then carve out Logical Volumes of any size from that pool.
- **Resize volumes online** — extend a filesystem while it's mounted and in use, without unmounting or reformatting.
- **Add more storage non-destructively** — add a new physical disk to the pool and immediately make that space available to any volume.
- **Take snapshots** — point-in-time snapshots of a volume for backups or testing.

With regular partitions, running out of disk space on `/var` means complex repartitioning, data migration, or downtime. With LVM it's two commands: `lvextend` and `resize2fs`.

---

## Task 3: Teach It Back — File Permissions for a New Linux User

Imagine every file on your computer has a padlock with three separate keyholes: one for **you** (the owner), one for your **team** (a group), and one for **everyone else**. Linux file permissions are exactly that — three sets of rules controlling who can do what with each file.

For each keyhole, there are three possible actions: **read** (look at the file), **write** (change the file), and **execute** (run the file as a program). You can grant or deny each action independently.

When you run `ls -la` and see `-rwxr-xr--`, you're reading those three keyholes left to right: the owner can read, write, and execute; the group can read and execute but not write; everyone else can only read.

The `chmod` command changes these rules. `chmod 755 script.sh` is a shorthand: 7 means `rwx` (4+2+1), 5 means `r-x` (4+0+1), so owner gets full control and everyone else can read and run it. `chmod 600 id_rsa` (your SSH private key) gives only you read and write access — anyone else gets nothing, which is why SSH refuses to use keys with looser permissions.

The most important habit: never set `777` (everyone can do everything) unless you genuinely mean it and understand why. Most files need `644` (owner writes, world reads) and most scripts need `755` (everyone can run it, only you can change it).

---
