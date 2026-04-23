# OS-Jackfruit: Custom Container Runtime

### 1. Team Information
**Member 1**  

**Name:** CHAITHANYA.N  

**SRN:** PES1UG24CS124

**Member 2**  

**Name:** CH YASHWITHA  

**SRN:** PES1UG24CS121
 
### 2. Build and Run Instructions
**Build:** \`make\`  
**Load Kernel Module:** \`sudo insmod monitor.ko\`  
**Verify Device:** \`ls -l /dev/container_monitor\`  
**Start Supervisor:** \`sudo ./engine supervisor ./rootfs-alpha\`  
**Clean Shutdown:** Press \`Ctrl+C\` in supervisor terminal; then \`sudo rmmod monitor\`.

### 3. Demo with Screenshots
 **SS1: Multi-container supervision:**  
 
  ![multi-container supervision](./screenshots/1.jpeg)  
  
 **SS2: Metadata tracking:** (Shows `engine ps` output)  
 
  ![metadata tracking](./screenshots/2.jpeg)  
  
 **SS3: Bounded-buffer logging:** (Shows `engine logs` output)  
 
  ![bounded buffer logging](./screenshots/3.jpeg)  
  
 **SS4: CLI and IPC:** (Shows start/stop commands)  
 
  ![cli and ipc](./screenshots/4.jpeg)  
  
 **SS5: Soft-limit warning:** (Shows dmesg output) and 
 **SS6: Hard-limit enforcement:** (Shows SIGKILL in dmesg)  
 
  ![soft limit warning and hard limit enforcement](./screenshots/5&6.jpeg)  
  
 **SS7: Scheduling experiment:** (Shows duration difference between hi/lo)  
 
  ![scheduling experiment](./screenshots/7.jpeg)  
  
 **SS8: Clean teardown:** (Shows "exited cleanly" and no zombies)  
 
  ![multi-container supervision](./screenshots/8A.jpeg)  
  
  ![multi-container supervision](./screenshots/8B.jpeg)

### 4. Engineering Analysis

#### Isolation Mechanisms
We achieve isolation using **Namespaces**. 
* **PID Namespace:** Ensures the container cannot see or kill host processes.
* **Mount/chroot:** Restricts the container to its own folder (e.g., `rootfs-alpha`). 
* **Sharing:** The container still shares the **Host Kernel**. Unlike a VM, there is no guest OS; the hardware is managed directly by the host kernel.

#### Supervisor and Process Lifecycle
The long-running supervisor acts as **Init (PID 1)** for the containers. It is responsible for **reaping** (cleaning up exit codes) so they don't become zombies.

#### IPC, Threads, and Synchronization
* **Mechanisms:** We use **Unix Domain Sockets** for CLI-to-Supervisor talk and **Shared Memory/Files** for logging.
* **Synchronization:** We use **Mutexes** on the container list. This prevents a "Race Condition" where two threads try to add a container at the exact same microsecond, which would corrupt the memory.

#### Memory Management and Enforcement
* **RSS:** Measures physical RAM currently held by the process. It does *not* measure swapped memory.
* **Limits:** Soft limits are for warnings; Hard limits are for system safety. Enforcement must be in **Kernel Space** because user-space programs can be paused or ignored; the Kernel has the absolute authority to trigger an immediate `SIGKILL`.

#### Scheduling Behavior
Our results showed that higher **Nice values** (lower priority) caused the `cpu_lo` task to wait longer for CPU time. This demonstrates that the Linux CFS (Completely Fair Scheduler) prioritizes responsiveness for tasks with lower nice values.

### 5. Scheduler Experiment Results
| Container | Nice Value | Result |
| :--- | :--- | :--- |
| cpu_hi | -5 | Finished duration 20 quickly |
| cpu_lo | 5 | First logs delayed (elapsed 23) |
