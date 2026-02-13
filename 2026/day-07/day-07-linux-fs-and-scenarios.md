# Linux File System Hierarchy (the most important directories).

1. `/` (root)

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(451).png)

- Contains the entire file system hierarchy.
- Example: ls -l / → shows bin, etc, home, var  
- I would use this when I need to navigate to the starting point of the system.

2. `/home` 

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(453').png)

- User home directories.
- Example: ls -l /home → shows atul, guest  
- I would use this when accessing user-specific files and configurations.

3. `/root`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(453).png)

- Home directory for the root user.
- Example: ls -l /root → shows .ssh, .bashrc  
- I would use this when logged in as root to manage system-level tasks.

4. `/etc`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(454).png)

- System configuration files.
- Example: ls -l /etc → shows hostname, hosts  
- I would use this when editing system configs like networking or hostname.

5. `/var/log`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(455).png)

- Log files for services and applications.
- Example: ls -l /var/log → shows syslog, auth.log  
- I would use this when troubleshooting service failures or errors.

6. `/tmp`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(456).png)

- Temporary files, cleared on reboot.
- Example: ls -l /tmp → shows tmp1234, socketXYZ  
- I would use this when storing short-lived files or testing scripts.

7. `/bin`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(457).png)

- Essential command binaries.
- Example: ls -l /bin → shows ls, cat, bash  
- I would use this when running basic commands required for system operation.

8. `/usr/bin` 

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(458).png)

- User command binaries.
- Example: ls -l /usr/bin → shows python3, git, vim  
- I would use this when running user-installed applications.

9. `/opt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/7b6aa3a44d096213cb0fe477df5b56430e7e5cf8/2026/day-07/Images/Screenshot%20(459).png)

- Optional/third-party applications.
- Example: ls -l /opt → shows google, custom-app  
- I would use this when installing external software packages.

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

Step 4: `kill -9 <PID> (if necessary)`
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