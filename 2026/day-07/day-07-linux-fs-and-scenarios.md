# Linux File System Hierarchy (the most important directories).

1. `/` (root)

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(451).png)

- Contains the entire file system hierarchy.
-   `/bin` → essential binaries like ls, cat, bash.

    `/etc` → configuration files.

    `/home` → user home directories.

    `/root` → root user’s home.

    `/var` → logs, spool files, caches.

    `/tmp` → temporary files.

    `/usr` → user applications and libraries. 
- I would use this when I need to navigate to the starting point of the system.

2. `/home` 

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(453').png)

- User home directories.
- Each subdirectory under `/home` corresponds to a user account.
- Inside each user’s home directory, you’ll find files like `.bashrc`, `.ssh/`, `Documents/`, `Downloads/`. 
- I would use this when accessing user-specific files and configurations.

3. `/root`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(453).png)

- `/root` is the home directory for the root user, separate from /home.
-   `.bashrc` → shell configuration file for root’s interactive sessions.

    `.profile` → login shell configuration.

    `.ssh/` → SSH keys and configuration for root.
- I would use this when performing administrative tasks that require root privileges, like editing system-wide configs or managing secure keys.

4. `/etc`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(454).png)

- `/etc` holds configuration files for services, applications, and the system itself.
- `/etc` is the control center for system behavior.
-   `hostname` → defines the system’s hostname.

    `hosts` → maps hostnames to IP addresses.

    `ssh/` → configuration for the SSH service.

    `apache2/` → configuration for Apache web server.

    `network/` → networking configuration files. 
- I would use this when I need to configure services, networking, or system identity (like hostname).

5. `/var/log`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(455).png)

-  It contains logs for services, applications, and the operating system itself.
- `/var/log` is your first stop for troubleshooting.
-   `auth.log` → authentication attempts (logins, sudo usage).

    `syslog` → general system messages.

    `kern.log` → kernel-related messages.

    `dpkg.log` → package installation/removal logs (Debian/Ubuntu).

- I would use this when diagnosing why a service failed, checking authentication attempts, or monitoring system health.

6. `/tmp`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(456).png)

- `/tmp` is used for temporary storage by applications, installers, and scripts.
- `/tmp` is used by applications and the system to store short-lived files. It’s world-writable (any user can write here), and its contents are usually cleared on reboot.
- I would use this when testing scripts that need scratch space or when checking if an application created temporary files or sockets.

7. `/bin`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(457).png)

- `/bin` contains the fundamental programs needed for the system to boot and run in single-user mode. 
- `/bin` is critical because it contains the basic tools needed to interact with the system.
- I would use this when running fundamental commands like ls, cat, or bash that are required even if other filesystems aren’t mounted.

8. `/usr/bin` 

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(458).png)

- `/usr/bin` is one of the largest directories on a Linux system because it contains most of the executables that regular users and administrators run day-to-day. 
- `/usr/bin` is where most user-level applications and utilities live.
- I would use this when running programming languages, editors, or tools installed via package managers.

9. `/opt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(459).png)

- `/opt` is used for installing third-party or add-on software that doesn’t come with the base operating system. It’s often empty unless you’ve installed external packages manually or via certain installers.
- `/opt` is the playground for external software.
- I would use this when installing or managing applications that aren’t part of the default Linux distribution, like Chrome, proprietary tools, or enterprise apps.

---
# Practice solving real-world scenarios.

## Scenario 1: Service Not Starting
Problem: myapp service failed after reboot.

Step 1: `systemctl status myapp`  
Why: Check if service is active, failed, or inactive.

Step 2: `journalctl -u myapp -n 50`  
Why: View last 50 log entries for error messages.

Step 3: `systemctl is-enabled myapp`    
Why: Verify if service is enabled to start on boot.

Step 4: `systemctl list-units --type=service | grep myapp`  
Why: Confirm if the service unit exists and is loaded.

## Scenario 2: High CPU Usage
Problem: Server is slow.

Step 1: `top`  
Why: Shows live CPU usage and top processes.

Step 2: `htop`  
Why: Provides interactive view with CPU/memory breakdown.

Step 3: `ps aux --sort=-%cpu | head -10`    
Why: Lists top 10 processes sorted by CPU usage.

Step 4: `kill -9 <PID>` (if necessary)   
Why: Terminate runaway process after identifying PID.

## Scenario 3: Finding Service Logs
Problem: Developer asks for Docker service logs.

Step 1: `systemctl status docker`  
Why: Check current status and recent log snippets.

Step 2: `journalctl -u docker -n 50`  
Why: View last 50 log lines for Docker service.

Step 3: `journalctl -u docker -f`  
Why: Follow logs in real-time, similar to tail -f.

## Scenario 4: File Permissions Issue
Problem: /home/user/backup.sh not executing.

Step 1: `ls -l /home/user/backup.sh`  
Why: Check current permissions.

Step 2: `chmod +x /home/user/backup.sh`  
Why: Add execute permission.

Step 3: `ls -l /home/user/backup.sh`  
Why: Verify permissions now include x.

Step 4: `./backup.sh`   
Why: Run the script after fixing permissions.