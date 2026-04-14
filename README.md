# Multi-Container Runtime

## Team Information
- Name: Yaseen | SRN: (your SRN here)
- Name: (teammate name) | SRN: (teammate SRN here)

## Build, Load, and Run Instructions

### Build
```bash
cd boilerplate
make
```

### Load kernel module
```bash
sudo insmod monitor.ko
ls -la /dev/container_monitor
```

### Prepare rootfs
```bash
cd ..
cp -a rootfs rootfs-base
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

### Start supervisor
```bash
cd boilerplate
sudo ./engine supervisor ../rootfs-base
```

### In a second terminal — start containers
```bash
sudo ./engine start alpha ../rootfs-alpha "echo hello from alpha"
sudo ./engine start beta ../rootfs-beta "echo hello from beta"
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
```

### Unload module
```bash
sudo rmmod monitor
```

## Engineering Analysis

### 1. Isolation Mechanisms
Each container is created using `clone()` with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS` flags. This gives each container its own PID namespace (so processes inside see themselves as PID 1), UTS namespace (own hostname), and mount namespace (own filesystem view). `chroot()` restricts the container to its assigned rootfs directory. The host kernel is still shared — containers share the same kernel, network stack, and hardware resources.

### 2. Supervisor and Process Lifecycle
The supervisor is a long-running parent process that uses `clone()` to create container children. It installs a `SIGCHLD` handler to reap exited children with `waitpid(WNOHANG)` to avoid zombies. Container metadata is maintained in a linked list protected by a mutex. The supervisor distinguishes between normal exit, manual stop, and hard-limit kill using the `stop_requested` flag.

### 3. IPC, Threads, and Synchronization
Two IPC mechanisms are used. Path A uses pipes to capture container stdout/stderr into the supervisor logging pipeline. Path B uses a UNIX domain socket for CLI-to-supervisor control commands. The bounded buffer uses a mutex + two condition variables (not_full, not_empty). Without the mutex, producers and consumers could corrupt the buffer simultaneously. The condition variables prevent busy-waiting.

### 4. Memory Management and Enforcement
RSS (Resident Set Size) measures physical memory pages currently mapped and in RAM. It does not measure virtual memory or shared libraries fully. Soft limits warn early before memory becomes critical. Hard limits kill the process to protect system stability. Enforcement belongs in kernel space because a user-space process cannot reliably monitor and kill another process — the kernel has direct access to mm_struct and can act atomically.

### 5. Scheduling Behavior
Experiments showed that a container with nice=-5 completed in 0.062s while a container with nice=10 took 3.023s running the same cpu_hog workload simultaneously. This demonstrates that the Linux CFS scheduler allocates more CPU time to higher-priority processes, directly affecting throughput.

## Design Decisions and Tradeoffs

| Subsystem | Choice | Tradeoff | Justification |
|---|---|---|---|
| Namespace isolation | CLONE_NEWPID + CLONE_NEWNS + CLONE_NEWUTS | No network isolation | Sufficient for process and filesystem isolation |
| Supervisor architecture | Single process + threads | Single point of failure | Simpler to implement and debug |
| IPC/logging | Pipes + bounded buffer | Buffer size limits throughput | Prevents log loss under high output |
| Kernel monitor | Mutex + linked list | Mutex slower than spinlock | Timer callback can sleep, spinlock unsafe |
| Scheduling experiments | nice values | Limited to CFS tuning | Simple and demonstrable on any Linux VM |

## Scheduler Experiment Results

| Container | Nice Value | Completion Time |
|---|---|---|
| highpri3 | -5 | 0.062s |
| lowpri3 | +10 | 3.023s |

Both containers ran the same `cpu_hog` workload simultaneously. The high priority container finished ~48x faster, demonstrating that Linux CFS scheduler heavily favors lower nice values when CPU resources are contested.
