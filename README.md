# Multi-Container Runtime

## 1. Team Information

| Name | SRN |
|------|-----|
| Rohith S | PES2UG24CS409 |
| Rajath N | PES2UG24CS394 |

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 2. Build, Load, and Run Instructions

### Prerequisites

- Ubuntu 22.04 or 24.04 (bare metal or VM)  
- Secure Boot **OFF** (required for kernel module loading)  
- Linux kernel headers installed  

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build
```bash
cd boilerplate

# Build all user-space binaries
make ci

# Build kernel module (requires kernel headers)
sudo make module

# Verify binaries exist
ls engine cpu_hog io_pulse memory_hog monitor.ko

```

### Set Up Root Filesystem

```bash
cd boilerplate

mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
sudo tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Create per-container writable copies
sudo cp -a rootfs-base rootfs-alpha
sudo cp -a rootfs-base rootfs-beta

# Copy workload binaries into rootfs copies
sudo cp cpu_hog memory_hog io_pulse rootfs-alpha/
sudo cp cpu_hog memory_hog io_pulse rootfs-beta/
```

### Load Kernel Module
```bash
sudo insmod monitor.ko

# Verify device was created
ls -la /dev/container_monitor

# Check kernel log
sudo dmesg | tail -3
```
### Start Supervisor

```bash
sudo rm -f /tmp/mini_runtime.sock
sudo ./engine supervisor ./rootfs-base
```

### Launch Containers

```bash
# In a second terminal:

# Start a container in background
sudo ./engine start alpha ./rootfs-alpha /cpu_hog

# Start a container and wait for it to finish
sudo ./engine run beta ./rootfs-beta /io_pulse

# Start with custom memory limits and priority
sudo ./engine start memtest ./rootfs-alpha /memory_hog \
    --soft-mib 10 --hard-mib 20 --nice 5
```
### 3. Demo Screenshots


### Screenshot 1 — Multi-container supervision

Two containers running simultaneously under one supervisor process with unique PIDs.
![Screenshot 1](screenshots/1.png)

### Screenshot 2 — Metadata tracking

Output of engine ps showing container ID, PID, state, and resource limits.
![Screenshot 2](screenshots/2.png)

### Screenshot 3 — Bounded-buffer logging

Output from engine logs showing data captured via the producer-consumer pipeline.
![Screenshot 3](screenshots/3.png)

### Screenshot 4 — CLI and IPC

Demonstration of engine run blocking until the container exits via UNIX domain socket IPC.
![Screenshot 4](screenshots/4.png)

### Screenshot 5 — Soft-limit warning

dmesg output showing the kernel module detecting a soft limit breach.
![Screenshot 5](screenshots/5.png)

### Screenshot 6 — Hard-limit enforcement

Kernel log showing a container being SIGKILLed for exceeding hard memory limits.
![Screenshot 6](screenshots/6.png)

### Screenshot 7 — Scheduling experiment

Comparison of CPU-bound vs I/O-bound processes and different nice values.
![Screenshot 7](screenshots/7.png)

### Screenshot 8 — Clean teardown

Evidence of zero zombie processes and successful kernel module unloading.
![Screenshot 8](screenshots/8.png)

### 4. Engineering Analysis

### 4.1 Isolation Mechanisms

We utilize Linux Namespaces to create isolated environments:

PID Namespace: Prevents containers from seeing or signaling host processes
UTS Namespace: Allows each container to have a unique hostname
Mount Namespace: Combined with chroot() to restrict filesystem access

### 4.2 Supervisor and Process Lifecycle

The supervisor acts as the init process for containers:

Prevents orphan processes
Handles SIGCHLD to immediately reap exited processes
Avoids zombie accumulation

### 4.3 IPC, Threads, and Synchronization

Path A (Logging):

Uses pipes and a bounded buffer
Implemented using pthread_mutex_t and pthread_cond_t
Prevents race conditions between producer and consumer threads

Path B (Control):

Uses a UNIX domain socket
Enables bidirectional communication between CLI and supervisor

### 4.4 Memory Management and Enforcement
Kernel module monitors RSS (Resident Set Size)
Soft limit: Logs warning in dmesg
Hard limit: Immediately kills process (SIGKILL)

Kernel-space enforcement ensures:

Low latency response
Protection from memory exhaustion

### 5. Design Decisions and Tradeoffs
Namespace isolation — chroot vs pivot_root

Choice: chroot()
Tradeoff: Less secure than pivot_root()
Justification: Simpler and sufficient for educational use

IPC mechanism — UNIX domain socket

Choice: UNIX domain socket
Tradeoff: Requires cleanup of .sock file
Justification: Supports bidirectional communication

Kernel monitor — Mutex vs Spinlock

Choice: Mutex
Tradeoff: Slight overhead vs spinlock
Justification: Required because memory allocation may sleep

### 6. Scheduler Experiment Results
Experiment 1 — CPU-bound priorities
Container	nice value	Wall-clock time
alpha	0	9.72s
beta	15	9.73s

### Analysis:
On multi-core systems, both tasks receive sufficient CPU time. The CFS scheduler ensures fairness unless the system is saturated.

### Experiment 2 — CPU-bound vs I/O-bound
Container	Workload	Behaviour
cpuwork	cpu_hog	Continuous execution
iowork	io_pulse	Frequent yielding (sleep)

### Analysis:
I/O-bound tasks are prioritized when waking from sleep, ensuring responsiveness. CPU-bound tasks utilize remaining CPU cycles
