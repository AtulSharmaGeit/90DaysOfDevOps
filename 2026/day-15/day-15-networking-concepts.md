## NETWORKING CONCEPTS

# Task 1: DNS – How Names Become IPs

1. What happens when we type `google.com` in a browser?

    When we type google.com in a browser, our system queries a DNS resolver. The resolver checks caches or contacts root servers, then TLD servers, then authoritative servers to resolve the domain into an IP address. Finally, our browser connects to that IP.

2. Record Types

    - A: Maps a domain to an IPv4 address.

    - AAAA: Maps a domain to an IPv6 address.

    - CNAME: Alias record pointing one domain to another.

    - MX: Mail exchange record, routes email.

    - NS: Nameserver record, defines authoritative DNS servers.

3. Command Output

    `dig google.com`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6dcf07cdef179c4cda91ce8b249a9cc65b0eb6/2026/day-15/Screenshots/Screenshot%20(517).png)

    A Record: 142.250.195.78

    TTL: 58 seconds

## Task 2: IP Addressing

1. What is an IPv4 address? How is it structured?

    An IPv4 address is a 32-bit number written in dotted decimal format (e.g., 192.168.1.10). It consists of four octets, each ranging from 0–255.

    - Public IP example: 8.8.8.8 (Google DNS).

    - Private IP example: 192.168.1.10 (home LAN).

2. Private IP Ranges

    - 10.0.0.0 – 10.255.255.255

    - 172.16.0.0 – 172.31.255.255

    - 192.168.0.0 – 192.168.255.255

3. Command Output

`ip addr show`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6dcf07cdef179c4cda91ce8b249a9cc65b0eb6/2026/day-15/Screenshots/Screenshot%20(518).png)

Private IP detected: 172.31.35.165

## Task 3: CIDR & Subnetting
1. What does /24 mean in 192.168.1.0/24?

    192.168.1.0/24 means the first 24 bits are the network portion, leaving 8 bits for hosts.

2. How many usable hosts in a /24? A /16? A /28?
    Usable hosts:

    - /24: 254

    - /16: 65,534

    - /28: 14

3. Why do we subnet?  
    
    Subnetting divides a large network into smaller segments to improve efficiency, security, and reduce broadcast traffic.

4. CIDR Table

| CIDR | Subnet Mask     | Total IPs | Usable Hosts |
|------|-----------------|-----------|--------------|
| /24  | 255.255.255.0   | 256       | 254          |
| /16  | 255.255.0.0     | 65,536    | 65,534       |
| /28  | 255.255.255.240 | 16        | 14           |

## Task 4: Ports – The Doors to Services
1. What is a port? Why do we need them?

A port is a logical endpoint for communication. They allow multiple services to run on the same IP by differentiating traffic.

2. Common Ports

| Port | Service |
|------|---------|
| 22   | SSH     |
| 80   | HTTP    |
| 443  | HTTPS   |
| 53   | DNS     |
| 3306 | MySQL   |
| 6379 | Redis   |
| 27017| MongoDB |

3. Command Output

`ss -tulpn`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6dcf07cdef179c4cda91ce8b249a9cc65b0eb6/2026/day-15/Screenshots/Screenshot%20(519).png)

Port 22 → SSH

Port 3306 → MySQL

## Task 5: Putting It Together

`curl http://myapp.com:8080` 

Networking concepts involved are -
- DNS resolves myapp.com to an IP.
- TCP connects to port 8080.
- HTTP protocol handles the request.

App can’t reach DB at `10.0.1.50:3306`
- First check connectivity (ping), then firewall rules, then whether MySQL is listening on port 3306.

## Key Learnings
1. DNS is the backbone of name resolution, translating human-readable domains into IPs.

2. IP addressing and subnetting define how devices are organized and communicate in networks.

3. Ports are critical for service differentiation, and knowing common ones is essential for troubleshooting.