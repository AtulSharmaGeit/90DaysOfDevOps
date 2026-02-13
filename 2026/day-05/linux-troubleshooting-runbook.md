# LINUX TROUBLESHOOTING RUNBOOK

## TARGET SERVICE / PROCESS
Service chosen: `sshd` (OpenSSH server)

---

## ENVIRONMENT BASICS
**Command:** `uname -a`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(425).png)

**Observation:** Kernel version and architecture confirmed (Ubuntu 24.04 on AWS).

**Command:** `cat /etc/os-release`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(426).png)

**Observation:** Distribution identified as Ubuntu 24.04 LTS.

---

## FILESYSTEM SANITY
**Command:** `mkdir /tmp/runbook-demo` 

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(427).png)

**Observation:** Directory and file operations successful, permissions intact.

**Command:** `cp /etc/hosts /tmp/runbook-demo/hosts-copy && ls -l /tmp/runbook-demo`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(428).png)

**Observation:** Directory and file operations successful, permissions intact.



---

## CPU & MEMORY SNAPSHOT
**Command:** `top -p <sshd_pid>` [ Get <sshd_pid> from `ps -ef | grep sshd`]

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(430).png)

**Observation:** `sshd` listener is idle, consuming ~1.7% of system memory and negligible CPU. System load averages are near zero, swap is unused.

**Command:** `free -h`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(434).png)

**Observation:** System has 454Mi total memory, ~186Mi in use, ~36Mi free. Swap is disabled (0B). Buff/cache is healthy at 265Mi, leaving ~267Mi available.

---

## DISK / IO SNAPSHOT
**Command:** `df -h`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(435).png)

**Observation:** Root filesystem is at 27% usage (healthy). Boot partitions (`/boot` and `/boot/efi`) are lightly used. No immediate disk pressure across mounted filesystems.

**Command:** `du -sh /var/log`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(436).png)

**Observation:** Total log directory size is ~17MB, which is small and manageable. Some subdirectories are restricted (permission denied), but overall log volume is healthy.

**Command:** `iostat`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(437).png)

**Observation:** CPU is mostly idle (~81%), with ~18% steal time (expected in cloud/virtualized environments). Disk activity on `xvda` shows moderate reads (~145 kB/s) and writes (~95 kB/s), but overall utilization is low and stable.

---

## NETWORK SNAPSHOT
**Command:** `ss -tulpn | grep sshd`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(438).png)

**Observation:** sshd is listening on port 22 for both IPv4 and IPv6, bound to all interfaces. The socket is managed jointly by `sshd` and `systemd`, confirming the service is healthy and reachable.

**Command:** `curl -I localhost:22`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(439).png)

**Observation:** Connection attempt returned an error (expected, since port 22 is not an HTTP endpoint). This confirms the port is reachable and responding, validating sshd’s presence on the socket.

---

## LOGS REVIEWED
**Command:** `journalctl -u ssh -n 50`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(440).png)

- `sshd` started cleanly, listening on 0.0.0.0 and :: port 22  
- Accepted publickey logins for user ubuntu  

**Observation:** Service startup successful, recent logins normal, no errors.

**Command:** `tail -n 50 /var/log/auth.log`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4be49df3272fc4367cfa36870a429daa7ca209b8/2026/day-05/Images/Screenshot%20(441).png)

- User `ubuntu` added to groups, password changed by root  
- Multiple successful publickey logins for `ubuntu`  
- Normal CRON sessions for root  
- One invalid banner exchange from localhost  

**Observation:** Authentication events show normal activity, successful logins, no brute-force attempts. Minor invalid banner exchange noted but not critical.

---

## QUICK FINDINGS
- `sshd` is healthy: low CPU/memory usage, disk and IO stable.  
- Network socket open and functioning.  
- Logs show normal authentication activity, no anomalies.  
- System resources are well within safe thresholds.

---

## IF THIS WORSENS (NEXT Steps)
1. **Restart strategy:** Restart sshd service (`sudo systemctl restart ssh`) and confirm socket rebind. 

2. **Increase log verbosity:** Enable `LogLevel DEBUG` in `/etc/ssh/sshd_config` for deeper trace. 

3. **Collect diagnostics:** Use `strace -p <pid>` or `tcpdump -i any port 22` to capture anomalies.

---
