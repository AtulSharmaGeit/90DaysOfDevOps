# Networking Basics & Troubleshooting

## Quick Concepts
- **OSI vs TCP/IP**  
  - OSI: 7 layers (Physical → Application).  
  - TCP/IP: 4 layers (Link, Internet, Transport, Application).  
- **Protocol placement**  
  - IP → Internet layer  
  - TCP/UDP → Transport layer  
  - HTTP/HTTPS, DNS → Application layer  
- **Example**  
  - `curl https://example.com` = Application (HTTP) over Transport (TCP) over Internet (IP).

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/518098880bc95aa453ab1a6c360158239bf54cfb/2026/day-14/Screenshots/Screenshot%20(510).png)

---

## Hands-on Checklist (Target: google.com)

- **Identity**  
  `hostname -I` → `172.31.35.165` (local IP noted).

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/518098880bc95aa453ab1a6c360158239bf54cfb/2026/day-14/Screenshots/Screenshot%20(511).png) 

- **Reachability**  
  `ping google.com` → ~25 ms latency, 0% packet loss.

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/518098880bc95aa453ab1a6c360158239bf54cfb/2026/day-14/Screenshots/Screenshot%20(512).png)  

- **Path**  
  `traceroute google.com` → 12 hops, one long delay at hop 8.  

- **Ports**  
  `ss -tulpn` → `tcp LISTEN 0 4096 0.0.0.0:22` (SSH service).

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/518098880bc95aa453ab1a6c360158239bf54cfb/2026/day-14/Screenshots/Screenshot%20(513).png) 

- **Name resolution**  
  `dig google.com` → `142.250.77.78` resolved.

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/518098880bc95aa453ab1a6c360158239bf54cfb/2026/day-14/Screenshots/Screenshot%20(514).png) 

- **HTTP check**  
  `curl -I https://google.com` → `HTTP/2 301`.

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/518098880bc95aa453ab1a6c360158239bf54cfb/2026/day-14/Screenshots/Screenshot%20(515).png) 

- **Connections snapshot**  
  `netstat -an | head` → 5 `ESTABLISHED`, 2 `LISTEN`.

---

## Mini Task: Port Probe
- Found: SSH on port 22.  
- Test: `nc -zv localhost 22` → `Connection succeeded`.  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/518098880bc95aa453ab1a6c360158239bf54cfb/2026/day-14/Screenshots/Screenshot%20(516).png)

- Reachable ✅. Next check if failed: `systemctl status sshd` or firewall rules.

---

## Reflection
- **Fastest signal when broken:** `ping` (reachability check).  
- **If DNS fails:** Inspect Internet layer (IP routing, DNS resolver).  
- **If HTTP 500:** Application layer (web server logs, app code).  
- **Two follow-ups in real incident:**  
  1. Check `journalctl -u <service>` for logs.  
  2. Verify firewall/security group rules.
