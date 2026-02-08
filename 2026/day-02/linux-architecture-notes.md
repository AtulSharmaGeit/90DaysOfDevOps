# Linux Architecture Notes

## Core Components
- **Kernel**: The heart of Linux. Manages hardware, memory, processes, and system calls.  
- **User Space**: Where applications and user processes run. Includes shells, utilities, and libraries.  
- **init/systemd**: The first process started by the kernel (PID 1). Initializes the system, launches services, and manages dependencies. Modern Linux uses **systemd**.

---

## Process Creation & Management
- Processes are created via **fork()** (copy of parent) and **exec()** (replace with new program).  
- Each process has a **PID** and is tracked by the kernel.  
- **Process States**:
  - **Running** – Actively executing on CPU.  
  - **Sleeping** – Waiting for an event (I/O, resource).  
  - **Zombie** – Finished execution but not yet cleaned up by parent.  
  - **Stopped** – Suspended, often by signals (e.g., Ctrl+Z).  

---

## Role of systemd
- **Service Manager**: Starts, stops, and monitors services.  
- **Parallel Startup**: Boots faster by running tasks simultaneously.  
- **Dependency Handling**: Ensures services start in the right order.  
- **Logging**: Integrates with `journalctl` for centralized logs.  
- **Targets**: Replace runlevels (e.g., `multi-user.target`, `graphical.target`).  

---

## Daily Commands
- `pwd` – Shows our current directory.
Useful when navigating file systems or writing scripts. 
- `ls -l` – Lists files with details (permissions, size, date).
Helps inspect file ownership and access rights.  
- `cd /path/to/folder` – Changes directory.
Essential for moving around the system.  
- `ps aux` – Displays all running processes.
Great for checking what’s consuming resources or debugging stuck services.  
- `systemctl status <service>` – Checks the status of a service.
Use this to verify if a service is active, failed, or inactive.  

---