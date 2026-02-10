# Linux Fundamentals with Real Commands.
## 1. Process Checks

### Command 1: List running processes
```bash
ps aux | head -5
```

![image alt](Screenshot)

### Command 2: List system resource usage
```bash
top -n 1 | head -10
```

![image alt](Screenshot)

### Command 3: Find processes by name
```bash
pgrep -l ssh
```

![image alt](Screenshot)

---

## 2. Service Checks

### Command 1: Check status of a service
```bash
systemctl status ssh
```

![image alt](Screenshot)

### Command 2: List active services
```bash
systemctl list-units --type=service | head -5
```

![image alt](Screenshot)

---

## 3. Log Checks

### Command 1: View logs for ssh service
```bash
journalctl -u ssh -n 10
```

![image alt](Screenshot)

### Command 2: Tail syslog
```bash
tail -n 10 /var/log/syslog
```

![image alt](Screenshot)

---

## 4. Mini troubleshooting steps
    
Step 1: Checked if `ssh` process is running with `pgrep -l ssh`.

![image alt](Screenshot)

Step 2: Verified service status using `systemctl status ssh`.

![image alt](Screenshot)

Step 3: Inspected logs with `journalctl -u ssh -n 10`.

![image alt](Screenshot)

Step 4: Confirmed no recent errors in `/var/log/syslog`.

![image alt](Screenshot)

Conclusion: SSH service is active and running correctly. Logs show normal startup messages, no errors detected.