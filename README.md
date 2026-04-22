# OS-Jackfruit: Custom Container Runtime

### 1. Team Information
* **Name:** [Your Name]
* **SRN:** [Your SRN]

### 2. Build and Run Instructions
**Build the project:**
\`\`\`bash
make
\`\`\`
**Load and Verify:**
\`\`\`bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
\`\`\`
**Run Supervisor:**
\`\`\`bash
sudo ./engine supervisor ./rootfs-alpha
\`\`\`
**CLI Operations (New Terminal):**
\`\`\`bash
sudo ./engine start alpha ./rootfs-alpha "/memory_hog 10 5" --soft-mib 5
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
\`\`\`
**Cleanup:**
\`\`\`bash
sudo rmmod monitor
make clean
\`\`\`

### 3. Demo with Screenshots
* **SS1: Multi-container supervision:**
  ![multi-container supervision](./screenshots/1.jpeg)
* **SS2: Metadata tracking:** (Shows `engine ps` output)
  ![metadata tracking](./screenshots/2.jpeg)
* **SS3: Bounded-buffer logging:** (Shows `engine logs` output)
  ![bounded buffer logging](./screenshots/3.jpeg)
* **SS4: CLI and IPC:** (Shows start/stop commands)
  ![cli and ipc](./screenshots/4.jpeg)
* **SS5: Soft-limit warning:** (Shows dmesg output) and 
* **SS6: Hard-limit enforcement:** (Shows SIGKILL in dmesg)
  ![soft limit warning and hard limit enforcement](./screenshots/5&6.jpeg)
* **SS7: Scheduling experiment:** (Shows duration difference between hi/lo)
  ![scheduling experiment](./screenshots/7.jpeg)
* **SS8: Clean teardown:** (Shows "exited cleanly" and no zombies)
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
