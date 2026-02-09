# Linux Commands Cheat Sheet (Beginner-Friendly)

## Process Management
- `whoami` – Display the current logged‑in user.
- `uptime` – Show how long the system has been running and load averages.
- `ps` – Show running processes.
- `ps -ef` – Show all processes in full format (with parent/child relationships).
- `pgrep <name>` – Find process IDs by name (e.g., pgrep nginx).
- `pkill <name>` – Kill processes by name (e.g., pkill python).
- `nice -n 10 <command>` – Start a process with adjusted priority.
- `renice <PID>` -n 5 – Change the priority of a running process.
- `top` – Live view of CPU/memory usage.
- `uname -a` – Show system information (kernel, architecture).

---

## File System
- `pwd` – Show current directory.
- `ls` – List files in a directory.
- `ls -a` – List all files, including hidden ones.
- `ls -lh` – List files with human‑readable sizes.
- `tree` – Display directory structure as a tree (install if missing).
- `cd <dir>` – Change directory.
- `mkdir <dir>` – Create a new folder.
- `rm <file>` – Delete a file.
- `cp <src> <dest>` – Copy a file.
- `mv <src> <dest>` – Move or rename a file.
- `cat <file>` – Show file contents.
- `touch <file>` – Create an empty file.
- `stat <file>` – Show detailed info about a file (size, timestamps, permissions).
- `head <file>` – Show the first 10 lines of a file.
- `tail <file>` – Show the last 10 lines of a file.
- `tail -f <file>` – Continuously watch a file (great for logs).
- `less <file>` – View file contents page by page.
- `wc <file>` – Count lines, words, and characters in a file.
- `chmod 755 <file>` – Change file permissions.
- `chown user:group <file>` – Change file ownership.
- `du -h` – Show disk usage in human-readable format.
- `df -h` – Show disk space usage in human‑readable format.

---

## Networking Troubleshooting
- `ping google.com` – Test if a host is reachable.
- `curl http://example.com` – Fetch a webpage or API response.
- `wget http://example.com/file.zip` – Download a file from the web.
- `ip addr` – Show your system’s IP addresses.
- `dig example.com` – Look up DNS records.
- `nslookup example.com` – Query DNS records (similar to dig).

---
