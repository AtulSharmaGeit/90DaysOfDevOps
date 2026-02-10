# Linux Fundamentals with Real Commands.
## 1. Process Checks

### Command 1: List running processes
```bash
ps aux | head -5
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/43e2a9823dd1697d3589eeac39a7fd9b98cca47d/2026/day-04/images/Screenshot%20(417).png)

### Command 2: List system resource usage
```bash
top -n 1 | head -10
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/43e2a9823dd1697d3589eeac39a7fd9b98cca47d/2026/day-04/images/Screenshot%20(418).png)

### Command 3: Find processes by name
```bash
pgrep -l ssh
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/43e2a9823dd1697d3589eeac39a7fd9b98cca47d/2026/day-04/images/Screenshot%20(419).png)

---

## 2. Service Checks

### Command 1: Check status of a service
```bash
systemctl status ssh
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/43e2a9823dd1697d3589eeac39a7fd9b98cca47d/2026/day-04/images/Screenshot%20(420).png)

### Command 2: List active services
```bash
systemctl list-units --type=service | head -5
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/43e2a9823dd1697d3589eeac39a7fd9b98cca47d/2026/day-04/images/Screenshot%20(421).png)

---

## 3. Log Checks

### Command 1: View logs for ssh service
```bash
journalctl -u ssh -n 10
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/43e2a9823dd1697d3589eeac39a7fd9b98cca47d/2026/day-04/images/Screenshot%20(422).png)

### Command 2: Tail syslog
```bash
tail -n 10 /var/log/syslog
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/43e2a9823dd1697d3589eeac39a7fd9b98cca47d/2026/day-04/images/Screenshot%20(424).png)

---

## 4. Mini troubleshooting steps
    
Step 1: Checked if `ssh` process is running with `pgrep -l ssh`.

Step 2: Verified service status using `systemctl status ssh`.

Step 3: Inspected logs with `journalctl -u ssh -n 10`.

Step 4: Confirmed no recent errors in `/var/log/syslog`.


Conclusion: SSH service is active and running correctly. Logs show normal startup messages, no errors detected.