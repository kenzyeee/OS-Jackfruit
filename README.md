# Multi-Container Runtime

A lightweight Linux container runtime in C with a long-running supervisor process and a kernel-space memory monitor.

---

## 1. Team Information


ADITYA KUMAR PES2UG24CS032
AADI KORE PES2UG24CS006
---

## 2. Build, Load, and Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot OFF. No WSL.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build everything

```bash
cd boilerplate
make
```

This produces:
- `engine` — user-space supervisor and CLI binary
- `monitor.ko` — kernel module
- `cpu_hog`, `io_pulse`, `memory_hog` — workload binaries

To build with kernel monitor integration enabled:

```bash
gcc -O2 -Wall -DHAVE_MONITOR -o engine engine.c -lpthread
```

### Prepare root filesystems

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-minirootfs-3.20.3-aarch64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-aarch64.tar.gz -C rootfs-base

sudo cp -a ./rootfs-base ./rootfs-alpha
sudo cp -a ./rootfs-base ./rootfs-beta
```

> Note: use the `aarch64` tarball on ARM64 VMs and `x86_64` on x86 VMs.

Copy workload binaries (must be statically linked) into rootfs before launch:

```bash
gcc -O2 -static -o cpu_hog cpu_hog.c
gcc -O2 -static -o memory_hog memory_hog.c
gcc -O2 -static -o io_pulse io_pulse.c

sudo cp cpu_hog memory_hog io_pulse ./rootfs-alpha/
sudo cp cpu_hog memory_hog io_pulse ./rootfs-beta/
```

### Load the kernel module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
sudo dmesg | tail -3
```

Expected:
```
[container_monitor] Module loaded. Device: /dev/container_monitor
```

### Start the supervisor

In a dedicated terminal (Terminal 1):

```bash
sudo ./engine supervisor ./rootfs-base
```

### Launch containers

In a second terminal (Terminal 2):

```bash
# Start two background containers
sudo ./engine start alpha ./rootfs-alpha "while true; do echo hello from alpha; sleep 2; done"
sudo ./engine start beta  ./rootfs-beta  "while true; do echo hello from beta;  sleep 3; done"

# List running containers
sudo ./engine ps

# Inspect logs
sudo ./engine logs alpha

# Run a foreground container (blocks until exit)
sudo ./engine run test1 ./rootfs-alpha "echo hello; sleep 1; echo done"

# Stop a container
sudo ./engine stop alpha
```

### Run memory limit test

```bash
sudo ./engine start memtest ./rootfs-alpha "/memory_hog 8 1000" --soft-mib 20 --hard-mib 40
sudo dmesg | grep container_monitor
sudo ./engine ps
```

### Run scheduling experiment

Terminal 2:
```bash
time sudo ./engine run cpu-hi ./rootfs-alpha "/cpu_hog 15" --nice -5
```

Terminal 3 (immediately):
```bash
time sudo ./engine run cpu-lo ./rootfs-beta "/cpu_hog 15" --nice 10
```

### Clean up

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Ctrl+C the supervisor in Terminal 1
sudo rmmod monitor
sudo dmesg | tail -5
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-container supervision
![alt text](<1.png>)
![alt text](2.png)
*Two containers (alpha and beta) running simultaneously under one supervisor process. Terminal 1 shows the supervisor output with both `started container` lines. Terminal 2 shows the `start` commands and their responses.*

---

### Screenshot 2 — Metadata tracking
![alt text](<3.png>)

*Output of `sudo ./engine ps` showing both containers in `running` state with their PIDs, start times, and exit codes.*

---

### Screenshot 3 — Bounded-buffer logging
![alt text](<4.png>)
![alt text](AlphaLog.png)
*Contents of `logs/alpha.log` captured through the producer-consumer logging pipeline, showing repeated `hello from alpha` lines. `ls -lh logs/` confirms both log files exist with nonzero sizes.*

---

### Screenshot 4 — CLI and IPC
![alt text](<4(CLI and IPC).png>)
*The `run` command issued from the CLI client, blocking until the container exits, then printing the supervisor's response. Demonstrates the UNIX domain socket control channel (Path B) between the CLI process and the supervisor daemon.*

---

### Screenshot 5 — Soft-limit warning
![alt text](<5(SOFT HARD Limit).png>)

*`dmesg` output showing the kernel module emitting a soft-limit warning for container `memtest` when its RSS exceeded 20 MiB. The warning includes the container ID, PID, actual RSS, and configured limit.*

---

### Screenshot 6 — Hard-limit enforcement
![alt text](<5(Hard Limit).png>)

*`dmesg` output showing the kernel module sending SIGKILL to `memtest` after its RSS exceeded 40 MiB, followed by `sudo ./engine ps` output showing the container state as `hard_limit_killed` with exit code 137 (128 + SIGKILL).*

---

### Screenshot 7 — Scheduling experiment
![alt text](7(Scheduling)-1.png)
![alt text](7(Scheduling)-2.png)
*Two CPU-bound containers running simultaneously with different nice values (`-5` vs `+10`). The high-priority container (cpu-hi) completed in 14.748s while the low-priority container (cpu-lo) took 16.329s on the same 15-second workload, demonstrating the CFS scheduler allocating more CPU time to the lower-nice process.*

---

### Screenshot 8 — Clean teardown
![alt text](<8(Clean Teardown)-1.png>)
![alt text](<8(Clean Teardown)-2.png>)

*`sudo ./engine ps` showing both containers in `stopped` state after `stop` commands. `ps aux | grep defunct` returns no zombie processes. Terminal 1 shows the supervisor printing `[supervisor] shutting down`, `[logger] consumer thread exiting, flushing done`, and `[supervisor] exited cleanly` on Ctrl+C.*

---

# 4. Engineering Analysis

## 4.1 Isolation Mechanisms

The runtime uses Linux namespaces to isolate containers by calling `clone()` with:
`CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`.

- **PID namespace:** Each container has its own process tree and sees itself as PID 1.
- **UTS namespace:** Each container can have its own hostname.
- **Mount namespace:** Filesystem changes inside a container do not affect the host.

For filesystem isolation:
- The process changes into the container root directory using `chdir()`
- Then applies `chroot(".")`, making that directory appear as `/`

The `/proc` filesystem is mounted inside the container so tools like `ps` work correctly.

The kernel is still shared across all containers. Namespaces only isolate the view of resources, not the kernel itself. The host can still see container processes using their host PIDs.

---

## 4.2 Supervisor and Process Lifecycle

A persistent supervisor process manages all containers.

- Containers are created using `clone()` instead of `fork()` so namespace flags can be applied directly.
- A `SIGCHLD` handler is installed to clean up child processes using `waitpid()`.

The handler:
- Retrieves exit status
- Updates container state (`stopped`, `exited`, or `hard_limit_killed`)

A `stop_requested` flag is used to differentiate between:
- User-triggered stops
- Kernel-enforced terminations

Keeping the supervisor alive ensures proper tracking, logging, and cleanup of containers.

---

## 4.3 IPC, Threads, and Synchronization

Two IPC mechanisms are used:

### Logging (Pipes)
- Container output (`stdout`, `stderr`) is redirected using `dup2()`
- A producer thread reads from the pipe
- A consumer thread writes logs to files using a bounded buffer

### Control (UNIX Domain Socket)
- CLI communicates with the supervisor through `/tmp/mini_runtime.sock`
- Requests and responses use structured messages

### Synchronization

The bounded buffer uses:
- `pthread_mutex_t` for locking
- `pthread_cond_t` (`not_full`, `not_empty`) for coordination

This prevents race conditions and avoids busy-waiting.

A separate mutex protects container metadata to avoid deadlocks.

---

## 4.4 Memory Management and Enforcement

Memory usage is tracked using **RSS (Resident Set Size)**:
- Represents actual physical memory usage
- Does not include swapped-out or unused virtual memory

Two limits are enforced:

- **Soft limit:** Generates a warning
- **Hard limit:** Terminates the process using `SIGKILL`

Kernel-level enforcement is used because:
- It avoids delays from user-space scheduling
- It directly accesses memory structures efficiently

User-space monitoring would be slower and less reliable.

---

## 4.5 Scheduling Behavior

The runtime uses the Linux Completely Fair Scheduler (CFS).

- CPU allocation is based on nice values
- Lower nice value = higher priority = more CPU time

### Experiment Results

Two CPU-bound containers were run:

| Container | Nice | Priority | Time |
|----------|------|----------|------|
| cpu-hi   | -5   | High     | 14.748s |
| cpu-lo   | +10  | Low      | 16.329s |

### Observations

- The high-priority container finished about 1.6 seconds faster
- Both containers completed successfully (no starvation)
- CPU time was distributed proportionally

### Conclusion

CFS ensures fair scheduling:
- Higher priority gets more CPU share
- Lower priority progresses more slowly

The `--nice` flag successfully affects scheduling behavior.

---

# 5. Design Decisions and Tradeoffs

## Namespace Isolation
**Choice:** Namespaces with `chroot()`  
**Tradeoff:** Less secure than `pivot_root()`  
**Reason:** Simpler implementation suitable for this project  

---

## Supervisor Design
**Choice:** Single-threaded loop with blocking `accept()`  
**Tradeoff:** One request can block others  
**Reason:** Simpler design with low expected usage  

---

## IPC and Logging
**Choice:** Pipes + UNIX socket + bounded buffer  
**Tradeoff:** Limited buffer size may block producers  
**Reason:** Prevents excessive memory usage  

---

## Kernel Monitor
**Choice:** Mutex for protecting kernel data  
**Tradeoff:** Spinlock would be safer in some contexts  
**Reason:** Simpler and works reliably in this case  

---

## Scheduling Control
**Choice:** Use of `nice()` values  
**Tradeoff:** No strict guarantees like real-time scheduling  
**Reason:** Portable and easy to implement  

---

# 6. Scheduler Experiment Results

## CPU-bound Workload Test

Both containers ran a CPU-intensive task (`/cpu_hog 15`).

| Container | Nice | Priority | Time |
|----------|------|----------|------|
| cpu-hi   | -5   | High     | 14.748s |
| cpu-lo   | +10  | Low      | 16.329s |

### Key Findings

- The high-priority container completed faster
- No starvation occurred
- CPU allocation was proportional:
  - cpu-hi ≈ 60%
  - cpu-lo ≈ 40%

### Final Insight

The experiment confirms:
- Correct behavior of the CFS scheduler
- Effective control of priority using nice values