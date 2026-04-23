# Operating System Fundamentals
### A Complete Reference for DevOps Engineers

> **How to use this document:** Read it top to bottom the first time. Every section builds on the previous one. Revisit individual sections when you encounter them in real work. By the end, you should be able to explain every concept here without saying *"I don't know."*

---

## Table of Contents

1. [What is an Operating System?](#1-what-is-an-operating-system)
2. [How a Computer Boots](#2-how-a-computer-boots)
3. [Kernel — The Heart of the OS](#3-kernel--the-heart-of-the-os)
4. [User Space vs Kernel Space](#4-user-space-vs-kernel-space)
5. [System Calls](#5-system-calls)
6. [Processes](#6-processes)
7. [Threads](#7-threads)
8. [Process vs Thread — The Real Difference](#8-process-vs-thread--the-real-difference)
9. [Process Lifecycle & States](#9-process-lifecycle--states)
10. [Scheduling](#10-scheduling)
11. [Memory Management](#11-memory-management)
12. [Virtual Memory & Swap](#12-virtual-memory--swap)
13. [File System](#13-file-system)
14. [Inodes, File Descriptors & Links](#14-inodes-file-descriptors--links)
15. [Linux File System Hierarchy](#15-linux-file-system-hierarchy)
16. [Permissions & Ownership](#16-permissions--ownership)
17. [Signals](#17-signals)
18. [Inter-Process Communication (IPC)](#18-inter-process-communication-ipc)
19. [Init Systems & systemd](#19-init-systems--systemd)
20. [Users, Groups & the Root User](#20-users-groups--the-root-user)
21. [The /proc and /sys Filesystems](#21-the-proc-and-sys-filesystems)
22. [CPU Concepts Every DevOps Engineer Must Know](#22-cpu-concepts-every-devops-engineer-must-know)
23. [I/O & Storage Concepts](#23-io--storage-concepts)
24. [Linux Namespaces & cgroups (Foundation of Containers)](#24-linux-namespaces--cgroups-foundation-of-containers)
25. [Quick Reference Cheat Sheet](#25-quick-reference-cheat-sheet)

---

## 1. What is an Operating System?

An **Operating System (OS)** is the software that sits between your hardware (CPU, RAM, disk, network card) and all the applications you run. It manages hardware resources and provides a consistent interface for programs to use those resources.

Think of the OS as a **hotel manager**:
- The hardware is the hotel (rooms = RAM, staff = CPU, storage = warehouse)
- Applications are the guests
- The OS is the manager — it decides who gets which room, when, and for how long

### What an OS Actually Does

| Responsibility | What it means in practice |
|---|---|
| **Process Management** | Starts, pauses, resumes, and kills programs |
| **Memory Management** | Allocates RAM to programs, takes it back when done |
| **File System Management** | Organizes how data is stored and retrieved on disk |
| **Device Management** | Provides drivers so programs can use hardware |
| **Security & Access Control** | Decides who can do what |
| **Networking** | Manages network interfaces and connections |

### Types of Operating Systems

- **General Purpose:** Linux, Windows, macOS — designed for a wide range of tasks
- **Real-Time OS (RTOS):** Used in embedded systems where timing guarantees are critical (aircraft, medical devices)
- **Distributed OS:** Manages a group of machines as one system

> **As a DevOps engineer,** you work with Linux daily. Everything in this document is rooted in how Linux works.

---

## 2. How a Computer Boots

This is the journey from pressing the power button to seeing a login prompt. Understanding this means you'll never be confused when a server fails to boot.

### The Full Boot Sequence

```
Power ON
   ↓
BIOS / UEFI (firmware)
   ↓
POST (Power-On Self Test)
   ↓
Bootloader (GRUB)
   ↓
Kernel loads into RAM
   ↓
Kernel initializes hardware
   ↓
Init system starts (systemd)
   ↓
Services & daemons start
   ↓
Login prompt
```

### Step-by-Step Breakdown

#### Step 1: BIOS / UEFI
- **BIOS** (Basic Input/Output System) is firmware stored on a chip on the motherboard
- **UEFI** (Unified Extensible Firmware Interface) is the modern replacement for BIOS — faster, supports larger disks, has a graphical interface
- The firmware's job: do a basic hardware check (POST), then find a bootable device

#### Step 2: POST (Power-On Self Test)
- The firmware checks that RAM, CPU, and essential hardware are functioning
- If something fails, you might hear beep codes or see an error before anything loads

#### Step 3: Bootloader (GRUB)
- **GRUB** (Grand Unified Bootloader) is the most common Linux bootloader
- It lives in the MBR (Master Boot Record) or the EFI partition
- GRUB's job: find the Linux kernel file on disk and load it into RAM
- GRUB also lets you choose which OS to boot (if you have multiple) and pass parameters to the kernel

#### Step 4: Kernel Loads
- The kernel file (e.g., `/boot/vmlinuz-5.15.0`) is loaded into RAM
- The kernel decompresses itself and starts running
- It detects and initializes hardware: CPU, memory, PCI devices, storage controllers
- It mounts the **initramfs** (a temporary root filesystem in RAM) to access disk drivers before the real disk is accessible
- Once the real disk is accessible, it mounts the real root filesystem (`/`)

#### Step 5: Init System (systemd)
- The kernel starts the very first process: **PID 1** — this is `systemd` on modern Linux
- systemd is the "mother of all processes" — everything else is started by it
- systemd reads its configuration and starts all services in the correct order

#### Step 6: Services Start
- Network interfaces come up
- SSH daemon starts
- Logging starts
- Your application services start
- Eventually, a login shell or display manager appears

> **Why this matters for DevOps:** When a server doesn't boot, you troubleshoot in reverse. Is it POST? Is GRUB not finding the kernel? Did the kernel panic? Did systemd fail to start a service? Knowing the sequence tells you where to look.

---

## 3. Kernel — The Heart of the OS

The **kernel** is the core of the operating system. It is the only piece of software that has complete, unrestricted access to the hardware.

### What the Kernel Does

- **Process scheduling** — decides which process runs on the CPU at any given time
- **Memory management** — tracks who owns which piece of RAM
- **Hardware abstraction** — provides uniform interfaces to diverse hardware via drivers
- **File system management** — handles reading/writing to disk
- **Network stack** — manages TCP/IP networking
- **Security enforcement** — enforces permissions and isolation between processes

### The Linux Kernel is Monolithic (with Modules)

A **monolithic kernel** means all core OS services run in a single large program in privileged memory. This is fast because there's no overhead switching between components.

However, Linux uses **loadable kernel modules (LKMs)** — pieces of kernel code that can be loaded or unloaded at runtime without rebooting.

```bash
lsmod          # List currently loaded kernel modules
modprobe ext4  # Load a kernel module
rmmod ext4     # Remove a kernel module
```

Common modules: filesystem drivers (ext4, xfs), network drivers, USB drivers, GPU drivers.

### Kernel Version

```bash
uname -r       # Shows kernel version, e.g.: 5.15.0-91-generic
#              Major.Minor.Patch-BuildInfo
```

---

## 4. User Space vs Kernel Space

This is one of the most important concepts in OS design.

### The Two Worlds

```
┌────────────────────────────────────────────────────┐
│                    USER SPACE                       │
│                                                     │
│   Your App    Web Server    Database    Shell       │
│                                                     │
│        (Limited privileges, isolated)               │
├─────────────────── System Call Interface ──────────┤
│                   KERNEL SPACE                      │
│                                                     │
│   Process Mgmt   Memory Mgmt   File System         │
│   Network Stack  Device Drivers   Security         │
│                                                     │
│        (Full hardware access, trusted)              │
├────────────────────────────────────────────────────┤
│                    HARDWARE                         │
│         CPU     RAM     Disk     Network            │
└────────────────────────────────────────────────────┘
```

### User Space
- Where **all regular programs run** — your application, bash, python, nginx, everything
- Programs here **cannot directly access hardware**
- Each program gets its own isolated memory space — they can't read each other's memory
- If a user-space program crashes, only that program dies — the OS and other programs continue running
- Programs run with limited CPU privileges (protection rings)

### Kernel Space
- Where the **kernel runs** — only the kernel and kernel modules
- Has **direct hardware access** — can read/write any memory, any device
- If kernel space code crashes, the **entire system crashes** (kernel panic)
- There's only one kernel space — all hardware is managed here centrally

### Why This Separation Exists

**Stability:** A buggy application crashing cannot take down the entire system.

**Security:** An application cannot read another application's memory or directly access hardware.

**Portability:** Applications don't need to know what specific hardware they're running on — the kernel handles that.

### Protection Rings (CPU Concept)
Modern CPUs have privilege levels called rings:
- **Ring 0 (Kernel mode):** Full access — the kernel runs here
- **Ring 3 (User mode):** Restricted access — all regular applications run here
- Rings 1 and 2 exist but are rarely used on modern systems

When a program needs something from the kernel, it crosses the boundary via a **system call**.

---

## 5. System Calls

A **system call (syscall)** is the mechanism a user-space program uses to request a service from the kernel.

Since user-space programs cannot directly access hardware, they ask the kernel: *"Please read this file for me," "Please allocate me memory," "Please open a network connection."*

### How a System Call Works

```
User Program calls read()
         ↓
C library (glibc) wraps it
         ↓
CPU switches to kernel mode (Ring 0)
         ↓
Kernel executes the read operation
         ↓
Result is returned to user space
         ↓
CPU switches back to user mode (Ring 3)
         ↓
Program continues
```

This context switch (user → kernel → user) has overhead. That's why minimizing syscalls is important for performance.

### Common System Calls

| Category | System Call | What it does |
|---|---|---|
| **File I/O** | `open()` | Open a file |
| **File I/O** | `read()` | Read data from a file/socket |
| **File I/O** | `write()` | Write data to a file/socket |
| **File I/O** | `close()` | Close a file descriptor |
| **Process** | `fork()` | Create a new process (clone current) |
| **Process** | `exec()` | Replace current process with a new program |
| **Process** | `exit()` | Terminate the current process |
| **Process** | `wait()` | Wait for a child process to finish |
| **Memory** | `mmap()` | Map memory or files into address space |
| **Memory** | `brk()` | Expand the heap (used by malloc internally) |
| **Network** | `socket()` | Create a network socket |
| **Network** | `connect()` | Connect to a remote address |
| **Network** | `bind()` | Bind socket to a local address/port |
| **Info** | `getpid()` | Get current process ID |
| **Info** | `getuid()` | Get current user ID |

### Watching System Calls in Real Time

```bash
strace ls          # Trace all syscalls made by the 'ls' command
strace -p 1234     # Trace syscalls of a running process with PID 1234
strace -e openat ls  # Only trace 'openat' syscalls
```

`strace` is invaluable for debugging — if an app fails mysteriously, strace shows exactly what it was asking the kernel for.

---

## 6. Processes

A **process** is a running instance of a program.

When you run `nginx`, the OS creates a process. If you run `nginx` again, a second independent process is created. The program (the binary file on disk) is static. A process is that program in action — with its own memory, state, and identity.

### What a Process Contains

Every process has:

| Component | Description |
|---|---|
| **PID** | Process ID — unique number assigned by the OS |
| **PPID** | Parent Process ID — who created this process |
| **Memory space** | Its own private virtual address space |
| **File descriptors** | Open files, sockets, pipes |
| **Environment variables** | Key-value pairs like `PATH`, `HOME` |
| **CPU registers** | Current state of execution |
| **Execution state** | Running, sleeping, stopped, zombie |
| **Priority/niceness** | How much CPU time it's entitled to |
| **Credentials** | UID, GID — who owns this process |

### How Processes Are Created: fork() and exec()

In Linux, every process (except PID 1) is created by another process. The mechanism is two system calls:

#### fork()
- Creates an **exact copy** of the calling process
- The new copy is called the **child process**
- The original is the **parent process**
- After fork(), both processes run the same code, but independently
- The child gets a new PID

```
Parent Process (PID 100)
        │
        │ calls fork()
        │
   ┌────┴────┐
   │         │
Parent     Child
(PID 100)  (PID 101)  ← exact copy of parent
```

#### exec()
- Replaces the current process image with a new program
- The process keeps its PID, but now runs different code
- Used after fork() to start a different program

**The full pattern:**
```
Shell (bash) runs a command
      ↓
bash calls fork() → creates child process
      ↓
child calls exec("ls") → child becomes the 'ls' program
      ↓
'ls' runs, then exits
      ↓
bash (parent) waits for child, then shows prompt again
```

This fork-exec pattern is how every program on Linux is launched.

### Viewing Processes

```bash
ps aux              # Snapshot of all running processes
ps aux | grep nginx # Find a specific process
top                 # Live view of processes (sorted by CPU)
htop                # Better version of top (may need install)
pstree              # Show processes as a tree (shows parent-child relationships)

# Key columns in ps aux:
# USER   PID  %CPU  %MEM  VSZ    RSS   TTY  STAT  START  TIME  COMMAND
# root   1     0.0   0.1  ...    ...   ?    Ss    Jan01  0:01  /sbin/init
```

### Process Identifiers

```bash
echo $$        # PID of the current shell
echo $PPID     # PPID of the current shell
pgrep nginx    # Find PID of processes named nginx
pidof nginx    # Same — find PID by name
```

---

## 7. Threads

A **thread** is the smallest unit of execution within a process.

A process can have one thread (single-threaded) or many threads (multi-threaded). All threads inside a process **share the same memory space** but each thread has its own:
- Stack (local variables, function call history)
- Registers (current execution point)
- Thread ID

### Visualizing Threads vs Processes

```
PROCESS A                          PROCESS B
┌──────────────────────────┐      ┌──────────────────────────┐
│  Shared Memory Space      │      │  Separate Memory Space    │
│                           │      │                           │
│  Thread 1 | Thread 2      │      │  Thread 1 | Thread 2      │
│  (own stack, own regs)    │      │  (own stack, own regs)    │
│                           │      │                           │
│  Shared: heap, globals,   │      │  Shared: heap, globals,   │
│  open files               │      │  open files               │
└──────────────────────────┘      └──────────────────────────┘
          ↑                                    ↑
  Threads share memory                Processes have
  within the process                  ISOLATED memory
```

### Why Use Threads?

- **Parallelism:** Multiple threads can run simultaneously on multi-core CPUs, doing work faster
- **Responsiveness:** A web server handles each request in a thread — one slow request doesn't block others
- **Efficiency:** Threads are cheaper to create than processes (no need to copy memory)

### Thread Challenges

- **Race conditions:** Two threads modify the same data simultaneously → unpredictable results
- **Deadlocks:** Thread A waits for Thread B's lock, Thread B waits for Thread A's lock → both stuck forever
- **Debugging complexity:** Multi-threaded bugs are notoriously hard to reproduce

---

## 8. Process vs Thread — The Real Difference

This is one of the most commonly asked concepts. Here it is clearly:

| Aspect | Process | Thread |
|---|---|---|
| **Memory** | Own isolated memory space | Shares process memory |
| **Creation cost** | Expensive (fork copies memory) | Cheap (just a new stack) |
| **Communication** | Complex (IPC: pipes, sockets, signals) | Easy (shared memory, but needs locks) |
| **Crash impact** | Only that process dies | Can crash the whole process |
| **Isolation** | Strong — processes can't see each other | Weak — all threads share everything |
| **Switching overhead** | Higher (full context switch) | Lower (lighter context switch) |
| **Example** | Each tab in Chrome (isolated) | Worker threads inside one tab |

### The Real-World Analogy

Think of a **company**:
- Each **department** is a **process** — they have their own budget (memory), resources, and can't rifle through another department's files
- **Employees within a department** are **threads** — they share the same office, resources, and files, but each works independently

### How Applications Use Them

- **Web servers like Apache:** Multi-process (one process per connection — very isolated)
- **Web servers like Nginx:** Multi-threaded/event-loop (efficient, handles many connections in fewer threads)
- **Python programs:** Limited by the GIL (Global Interpreter Lock) — only one thread runs Python code at a time (CPU-bound tasks use multiprocessing instead)
- **Java applications:** Heavily multi-threaded (JVM manages a thread pool)

---

## 9. Process Lifecycle & States

A process doesn't just "run" — it moves through different states during its life.

### Process States

```
                      fork()
                        ↓
              ┌─────[CREATED]─────┐
              ↓                   
           [READY] ←──────────────────┐
              ↓  (scheduler picks it) │
          [RUNNING]                   │
           ↙      ↘                   │
    [WAITING]    [TERMINATED]         │
    (I/O done) ──────────────────────┘
```

| State | Meaning |
|---|---|
| **Created/New** | Process has just been forked, not yet scheduled |
| **Ready** | Process is ready to run, waiting for CPU time |
| **Running** | Currently executing on a CPU core |
| **Waiting/Sleeping** | Blocked — waiting for I/O, a signal, or a timer |
| **Stopped** | Paused (e.g., Ctrl+Z in terminal) |
| **Zombie** | Process has finished but parent hasn't called wait() yet |

### What is a Zombie Process?

When a process exits, it doesn't immediately disappear. It enters a **zombie state** — the OS keeps a small record of it (exit code, resource usage stats) so the parent can retrieve this information via `wait()`.

Once the parent calls `wait()`, the zombie is fully cleaned up.

**Problem:** If the parent never calls `wait()` (buggy code), zombies accumulate. They waste process table entries. A system can only have a limited number of PIDs — too many zombies can prevent new processes from starting.

```bash
# Zombie processes show as 'Z' in the STAT column
ps aux | grep 'Z'
```

### What is an Orphan Process?

If a parent process dies before its children, the children become **orphans**. Linux automatically re-parents orphans to PID 1 (systemd/init), which will properly call `wait()` for them.

---

## 10. Scheduling

The **scheduler** is the part of the kernel that decides which process runs on the CPU and for how long.

### Why is Scheduling Needed?

A modern server might have hundreds of processes but only 8 CPU cores. The scheduler creates the illusion that all processes run simultaneously by rapidly switching between them — this is called **time-sharing** or **multitasking**.

### Key Concepts

#### Context Switch
When the CPU switches from running process A to process B:
1. Save process A's state (registers, instruction pointer) to memory
2. Load process B's state from memory
3. Resume process B

This is a **context switch**. It takes time — too many context switches hurt performance.

#### Time Slice / Quantum
The maximum amount of time a process can run before the scheduler preempts it and gives the CPU to another process. Typically a few milliseconds.

#### Preemptive vs Cooperative Multitasking
- **Preemptive (modern Linux):** The kernel forcibly takes the CPU away from a process after its time slice
- **Cooperative (old systems):** Programs voluntarily yield the CPU — a misbehaving program could hog everything

### Linux Scheduler: CFS (Completely Fair Scheduler)

Linux uses CFS since kernel 2.6.23. Its goal: give every process a **fair** share of CPU time.

- It tracks how much CPU time each process has used
- The process with the **least CPU time** gets to run next (most "needy")
- Higher priority processes accumulate CPU time slower (so they get more turns)

### Priority and Niceness

Every process has a **nice value** ranging from -20 to +19:
- **-20 = highest priority** (very "un-nice" to other processes — takes more CPU)
- **+19 = lowest priority** (very "nice" — yields CPU to others)
- Default is 0

```bash
nice -n 10 my_script.sh    # Start a program with niceness +10 (lower priority)
renice -n -5 -p 1234       # Change niceness of running process PID 1234
ps -el | head              # Shows PRI and NI (priority and nice) columns
```

> **Rule of thumb:** Run batch/background jobs with high niceness (+10 to +19) so they don't compete with interactive or critical processes.

---

## 11. Memory Management

Memory management is how the OS allocates and tracks RAM usage across all running processes.

### Types of Memory

```
┌─────────────────────────────────────────────┐
│          Process Virtual Address Space       │
│                                              │
│  ┌─────────┐  High addresses                 │
│  │  Stack  │  ← grows downward               │
│  │         │  (local variables, call frames) │
│  ├─────────┤                                 │
│  │   ...   │  (unmapped gap)                 │
│  ├─────────┤                                 │
│  │  Heap   │  ← grows upward                 │
│  │         │  (dynamic memory: malloc/new)   │
│  ├─────────┤                                 │
│  │   BSS   │  (uninitialized global vars)    │
│  ├─────────┤                                 │
│  │  Data   │  (initialized global vars)      │
│  ├─────────┤                                 │
│  │  Text   │  (program code / instructions)  │
│  └─────────┘  Low addresses                  │
└─────────────────────────────────────────────┘
```

| Region | What lives here | Who manages it |
|---|---|---|
| **Text / Code** | The compiled program instructions | Static |
| **Data** | Global/static variables with initial values | Static |
| **BSS** | Global/static variables without initial values | Static |
| **Heap** | Memory allocated at runtime (`malloc`, `new`) | Developer/GC |
| **Stack** | Function calls, local variables, return addresses | OS/CPU |

### Stack vs Heap — The Key Difference

**Stack:**
- Automatically managed — memory is allocated when a function is called, freed when it returns
- Very fast — just move a pointer
- Limited size (typically 8MB per thread on Linux)
- Stack overflow = recursion too deep, or very large local variables

**Heap:**
- Manually managed by the developer (in C/C++) or by a garbage collector (Java, Python, Go)
- Allocated with `malloc()`/`free()` in C, `new`/`delete` in C++
- Much larger than stack
- Memory leak = allocated memory that's never freed

### Memory Leak
When a program allocates heap memory but never frees it. Over time the process consumes more and more RAM until the system runs out. Common in long-running services.

```bash
# Check memory usage of a specific process
cat /proc/1234/status | grep VmRSS    # Resident Set Size (actual RAM used)
valgrind --leak-check=full ./program  # Detect memory leaks (C/C++ programs)
```

### What RSS and VSZ Mean (You See These in ps)

```
USER   PID  %CPU  %MEM    VSZ    RSS
root   1234  0.5   2.1  512000  43000
```

- **VSZ (Virtual Memory Size):** Total virtual address space the process has mapped — includes shared libraries, memory-mapped files. This number is often huge and misleading.
- **RSS (Resident Set Size):** The actual RAM currently in physical memory. This is the number you care about when checking real memory usage.

---

## 12. Virtual Memory & Swap

### What is Virtual Memory?

Every process thinks it has its own private, contiguous block of memory — this is **virtual memory**. It's an illusion created by the OS.

In reality, the physical RAM is shared among all processes, fragmented, and partially stored on disk (swap). The **Memory Management Unit (MMU)** in the CPU transparently translates virtual addresses to physical addresses.

```
Process A thinks:          Physical RAM Reality:
0x0000 → 0x8000           0x0000-0x2000 = Process A's code
                           0x2000-0x4000 = Process B's data
Process B thinks:          0x4000-0x6000 = Process A's heap
0x0000 → 0x8000           0x6000-0x8000 = OS kernel
                           ... some of it is actually on swap disk
```

### Why Virtual Memory Exists

1. **Isolation:** Each process thinks it has the whole address space — it can't accidentally (or maliciously) access another process's memory
2. **Flexibility:** Processes can use more memory than physically exists (some pages swapped to disk)
3. **Efficiency:** Multiple processes can share the same physical pages for read-only data (like shared libraries — `libc` is only loaded into RAM once even if 100 processes use it)

### Pages and Page Tables

Memory is divided into fixed-size chunks called **pages** (typically 4KB on Linux).

The **page table** is a map maintained by the kernel that says "virtual page X of process A lives at physical page Y."

When a process accesses a virtual address:
1. The MMU looks up the page table
2. Finds the physical address
3. Returns the data

If the page isn't in RAM (it's on swap or hasn't been loaded yet) → **page fault** → kernel handles it, loads the page.

### Swap Space

**Swap** is disk space used as overflow when RAM is full.

When RAM fills up, the kernel selects the least recently used pages and writes them to the swap partition or swap file on disk. This **swapping out** frees RAM. When that memory is needed again, it's **swapped in** (read back from disk).

**The tradeoff:** Disk is 100–1000x slower than RAM. A system that heavily uses swap is **thrashing** — spending more time moving data between disk and RAM than actually doing work.

```bash
free -h           # Show RAM and swap usage
swapon --show     # Show active swap spaces
vmstat 1          # Live view of memory stats (si/so = swap in/out)
cat /proc/meminfo # Detailed memory breakdown
```

### Understanding free -h Output

```
              total   used    free   shared  buff/cache  available
Mem:          15Gi    4.2Gi   8.3Gi  234Mi   2.7Gi       10Gi
Swap:         2.0Gi   0B      2.0Gi
```

- **total:** Total installed RAM
- **used:** RAM currently used by processes
- **free:** RAM completely unused
- **buff/cache:** RAM used by kernel for disk cache (can be reclaimed if needed!)
- **available:** Free + reclaimable cache — this is the real answer to "how much memory can new programs use"

> **Common mistake:** New DevOps engineers panic when they see "free" is low. The `available` column is what actually matters. Linux intentionally uses all RAM for disk cache — idle RAM is wasted RAM.

### OOM Killer (Out of Memory Killer)

When the system runs out of both RAM and swap, the kernel activates the **OOM Killer**. It selects a process to kill to free up memory. The selection is based on an **oom_score** — processes using the most memory with the least "importance" get killed first.

```bash
# Check OOM score of a process
cat /proc/1234/oom_score

# Make a process less likely to be OOM killed (lower = less likely)
echo -500 > /proc/1234/oom_score_adj

# Find OOM kill events in logs
dmesg | grep -i "killed process"
journalctl -k | grep -i oom
```

---

## 13. File System

A **file system** is the method and structure the OS uses to organize and store files on a storage device.

Without a file system, a disk is just billions of raw bits with no organization — you couldn't find anything.

### What a File System Does

- Organizes data into **files** and **directories**
- Tracks **where each file's data is** on the physical disk
- Manages **free space** (knows which disk blocks are unused)
- Stores **metadata** about files (name, size, permissions, timestamps)
- Handles **concurrent access** (what happens when two programs write to the same file)

### Common Linux File Systems

| File System | Description | Best For |
|---|---|---|
| **ext4** | Most common Linux FS. Mature, reliable, journaled | General purpose, most Linux installs |
| **xfs** | High performance, great for large files and parallel I/O | Databases, large files, RHEL default |
| **btrfs** | Modern FS with snapshotting, RAID, compression built-in | NAS, systems needing snapshots |
| **tmpfs** | File system stored entirely in RAM — lost on reboot | `/tmp`, `/run` — temporary data |
| **ext3** | Older than ext4, still works, no extent-based storage | Legacy systems |
| **ZFS** | Enterprise-grade, data integrity, compression | NAS, storage servers (not in main Linux kernel) |
| **FAT32/exFAT** | Universal compatibility | USB drives, cross-platform |
| **NTFS** | Windows file system | Windows partitions |

### Journaling

**Journaling** is a crash-safety mechanism in modern file systems (ext4, xfs, btrfs).

Before making changes, the file system writes what it's about to do to a **journal** (a log area). If the system crashes mid-write, on reboot the FS checks the journal and either completes or rolls back the incomplete operation. Without journaling (like old ext2), a crash could corrupt the filesystem.

### Mounting

In Linux, storage devices don't appear as `C:\` or `D:\` like in Windows. Instead, they are **mounted** to a directory in the single unified directory tree.

```bash
mount /dev/sdb1 /mnt/data    # Mount partition sdb1 to /mnt/data
umount /mnt/data             # Unmount it
mount | grep sdb             # Show what's mounted
df -h                        # Show all mounted filesystems and disk usage
lsblk                        # Show block devices and their mount points
```

The file `/etc/fstab` defines what gets mounted automatically at boot:
```
# Device         Mount Point   FS Type  Options     Dump  Pass
UUID=abc123...   /             ext4     defaults     0     1
UUID=def456...   /boot/efi     vfat     defaults     0     2
UUID=ghi789...   /home         ext4     defaults     0     2
/dev/sdb1        /mnt/data     xfs      defaults     0     0
```

---

## 14. Inodes, File Descriptors & Links

### Inodes

An **inode** (index node) is a data structure that stores **metadata about a file** — everything except the file's name and actual data.

Every file and directory has exactly one inode. The inode contains:

| Inode Field | Description |
|---|---|
| **Inode number** | Unique identifier within the filesystem |
| **File type** | Regular file, directory, symlink, socket, etc. |
| **Permissions** | Read/write/execute for owner, group, others |
| **Owner UID/GID** | User and group who own this file |
| **Size** | File size in bytes |
| **Timestamps** | atime (last accessed), mtime (last modified), ctime (last changed) |
| **Link count** | Number of hard links pointing to this inode |
| **Block pointers** | Pointers to disk blocks where file data actually lives |

Notice: **the filename is NOT in the inode**. The filename lives in the directory entry, which maps name → inode number.

```bash
ls -li            # Show inode numbers (first column)
stat filename     # Show complete inode information for a file
df -i             # Show inode usage (you can run out of inodes even with disk space!)
```

### How Filenames Map to Inodes

A **directory** is actually a special file that contains a list of entries:
```
(filename) → (inode number)
"readme.txt" → 12345
"config.json" → 67890
"logs" → 11111
```

When you access `/home/user/readme.txt`:
1. OS reads inode of `/` (always inode 2)
2. Finds `home` in its directory entries → inode X
3. Reads inode X, finds `user` → inode Y
4. Reads inode Y, finds `readme.txt` → inode 12345
5. Reads inode 12345 to find file permissions, size, and data block locations
6. Reads the actual data

### Hard Links vs Symbolic Links

#### Hard Link
A hard link is an **additional directory entry pointing to the same inode**.

```bash
ln original.txt hardlink.txt
```

```
hardlink.txt ──┐
               ├──→ Inode 12345 ──→ Actual data on disk
original.txt ──┘
```

- Both filenames are equal — deleting one doesn't delete the data (inode link count goes from 2 to 1)
- Data is only deleted when link count reaches 0
- Hard links **cannot span filesystems** (inode numbers are filesystem-local)
- Hard links **cannot point to directories**

#### Symbolic Link (Symlink)
A symlink is a **special file that contains a path string** pointing to another file.

```bash
ln -s /etc/nginx/nginx.conf nginx.conf
```

```
nginx.conf (symlink) ──→ "/etc/nginx/nginx.conf" (path string)
                                  ↓
                         /etc/nginx/nginx.conf ──→ Inode 67890 ──→ Data
```

- Delete the original → symlink is broken (dangling symlink)
- Can span filesystems and point to directories
- `ls -la` shows symlinks with `l` type and `→` pointing to target

```bash
ls -la /etc/alternatives/python  # Common place to see symlinks
readlink -f symlink_name         # Follow symlink to find real file
file filename                    # Shows if it's a symlink
```

### File Descriptors

A **file descriptor (fd)** is an integer that represents an open file (or socket, pipe, device) for a process.

When a process opens a file, the kernel:
1. Validates permissions
2. Creates an entry in the process's file descriptor table
3. Returns a small integer (the file descriptor) to the process

Every process starts with 3 file descriptors:

| FD | Name | Default connection |
|---|---|---|
| **0** | stdin | Keyboard input |
| **1** | stdout | Terminal output |
| **2** | stderr | Terminal error output |

```bash
# See all file descriptors for a process
ls -la /proc/1234/fd

# Check the open file limit
ulimit -n              # Per-process limit (default 1024 or 65536)
cat /proc/sys/fs/file-max  # System-wide limit

# See how many files are currently open system-wide
lsof | wc -l

# Files opened by a specific process
lsof -p 1234
```

> **Why this matters:** If a service crashes with "Too many open files," you need to raise the `ulimit`. Common issue with high-traffic servers.

---

## 15. Linux File System Hierarchy

Linux follows the **FHS (Filesystem Hierarchy Standard)** — a standardized directory structure.

```
/
├── bin/      → Essential user binaries (ls, cp, bash) — available before /usr mounts
├── sbin/     → Essential system binaries (mount, fdisk) — for root/admin
├── usr/      → Secondary hierarchy for user programs
│   ├── bin/  → Most user commands (git, python, nginx)
│   ├── lib/  → Shared libraries for /usr/bin programs
│   └── local/→ Locally installed software (not from package manager)
├── etc/      → Configuration files (nginx.conf, fstab, passwd)
├── home/     → User home directories (/home/alice, /home/bob)
├── root/     → Home directory for the root user
├── var/      → Variable data (logs, caches, databases, spools)
│   ├── log/  → Log files (syslog, auth.log, nginx/access.log)
│   ├── lib/  → Application state (docker images, package databases)
│   └── spool/→ Mail queues, print queues
├── tmp/      → Temporary files — cleared on reboot
├── dev/      → Device files (sda, null, zero, random, tty)
├── proc/     → Virtual FS — live kernel and process info
├── sys/      → Virtual FS — live hardware/kernel parameters
├── run/      → Runtime data since last boot (PID files, sockets)
├── boot/     → Kernel, initramfs, bootloader files
├── lib/      → Shared libraries for /bin and /sbin
├── mnt/      → Temporary mount points (manual mounts go here)
├── media/    → Auto-mounted removable devices (USB, CD)
└── opt/      → Optional third-party software
```

### Key Directories You Touch as DevOps

| Directory | Why you care |
|---|---|
| `/etc/` | All configuration lives here — nginx, ssh, systemd, cron |
| `/var/log/` | All logs — first place to check when something breaks |
| `/tmp/` | Temp files, often used by installers and scripts |
| `/proc/` | Live system information — incredibly useful for debugging |
| `/dev/null` | The black hole — redirect output here to discard it |
| `/dev/sda` | First disk — partitions are `/dev/sda1`, `/dev/sda2`, etc. |
| `/run/` | PID files, Unix sockets for running services |

---

## 16. Permissions & Ownership

Linux permissions control who can read, write, or execute a file.

### The Permission Model

Every file has:
- **One owner** (a user)
- **One group** (a group of users)
- **Three sets of permissions:** owner, group, others

### Reading Permissions

```bash
ls -la /etc/passwd
-rw-r--r-- 1 root root 2847 Jan 10 08:21 /etc/passwd
```

Breaking it down:
```
- rw- r-- r--
│ │   │   │
│ │   │   └── Others:  r-- = read only
│ │   └────── Group:   r-- = read only
│ └────────── Owner:   rw- = read and write
└──────────── File type: - = regular file
```

**File types:**
- `-` = regular file
- `d` = directory
- `l` = symbolic link
- `c` = character device
- `b` = block device
- `s` = socket
- `p` = pipe

**Permission bits:**
- `r` = read (4)
- `w` = write (2)
- `x` = execute (1)
- `-` = permission not set (0)

### Numeric (Octal) Notation

Each permission set can be represented as a number:
```
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0
```

Common permission patterns:
| Octal | Symbolic | Meaning |
|---|---|---|
| `755` | `rwxr-xr-x` | Owner can do anything; group/others can read & execute |
| `644` | `rw-r--r--` | Owner can read/write; group/others read only |
| `600` | `rw-------` | Owner only — good for private keys |
| `777` | `rwxrwxrwx` | Everyone can do everything — **avoid this** |
| `700` | `rwx------` | Only owner can access |

### Changing Permissions

```bash
chmod 755 script.sh          # Set permissions by octal
chmod u+x script.sh          # Add execute for owner (u=user, g=group, o=others)
chmod go-w sensitive.txt     # Remove write from group and others
chmod -R 755 /var/www/       # Recursive — apply to directory and all contents

chown alice file.txt         # Change owner to alice
chown alice:devteam file.txt # Change owner to alice and group to devteam
chown -R www-data:www-data /var/www/  # Recursive ownership change
```

### Special Permission Bits

#### setuid (SUID) — bit value 4
- On an executable: runs with the **file owner's privileges**, not the caller's
- Example: `/usr/bin/passwd` is owned by root with SUID set → any user can change their password because it runs as root briefly
- Shown as `s` in the owner execute position: `-rwsr-xr-x`

#### setgid (SGID) — bit value 2
- On an executable: runs with the **file group's privileges**
- On a directory: new files created inside inherit the directory's group
- Useful for shared project directories

#### Sticky Bit — bit value 1
- On a directory: **only the file owner can delete their own files**, even if others have write permission
- Used on `/tmp/` so users can't delete each other's temp files
- Shown as `t` in the others execute position: `drwxrwxrwt`

```bash
ls -la /tmp
drwxrwxrwt 1 root root ... /tmp    # See the 't' at the end
ls -la /usr/bin/passwd
-rwsr-xr-x 1 root root ... /usr/bin/passwd   # See the 's'
```

### umask

**umask** is a mask that determines the default permissions for newly created files.

Default file permissions = `666` minus umask
Default directory permissions = `777` minus umask

```bash
umask         # Show current umask (typically 022)
# 666 - 022 = 644 → new files get rw-r--r--
# 777 - 022 = 755 → new directories get rwxr-xr-x
```

---

## 17. Signals

**Signals** are the OS's mechanism for sending notifications to processes. Think of them as software interrupts — the process stops what it's doing and handles the signal.

### Common Signals

| Signal | Number | Default Action | When it's used |
|---|---|---|---|
| **SIGHUP** | 1 | Terminate | Terminal closed; also used to reload config |
| **SIGINT** | 2 | Terminate | User presses Ctrl+C |
| **SIGQUIT** | 3 | Core dump | User presses Ctrl+\ |
| **SIGKILL** | 9 | Terminate (cannot be caught) | Force kill — cannot be ignored or handled |
| **SIGSEGV** | 11 | Core dump | Segmentation fault — invalid memory access |
| **SIGTERM** | 15 | Terminate | Polite termination request — can be caught |
| **SIGSTOP** | 19 | Stop (cannot be caught) | Pause process — cannot be ignored |
| **SIGCONT** | 18 | Continue | Resume a stopped process |
| **SIGCHLD** | 17 | Ignore | Sent to parent when child terminates |
| **SIGUSR1/2** | 10/12 | Terminate | User-defined — used by apps for custom actions |

### The Critical Difference: SIGTERM vs SIGKILL

This is asked constantly in interviews and matters every day.

**SIGTERM (15)** — *"Please stop"*
- Can be **caught, handled, or ignored** by the process
- A well-written application catches SIGTERM and performs **graceful shutdown**: saves state, closes database connections, finishes in-flight requests, releases locks
- `kill 1234` sends SIGTERM by default

**SIGKILL (9)** — *"You're dead"*
- **Cannot be caught, handled, or ignored** — ever
- The kernel immediately destroys the process
- No cleanup happens — open files might be corrupted, connections are dropped ungracefully
- Only use this when SIGTERM doesn't work

**Best practice — the graceful kill sequence:**
```bash
kill 1234           # Send SIGTERM — wait 10-30 seconds
kill -9 1234        # Only if still running after SIGTERM — send SIGKILL
```

### SIGHUP — More Than Just "Hangup"

Originally sent when a terminal connection was lost. Modern usage: **reload configuration**.

Many daemons (nginx, sshd, rsyslog) catch SIGHUP and reload their config files without restarting:
```bash
kill -HUP $(cat /var/run/nginx.pid)    # Tell nginx to reload config
kill -1 1234                            # Same — send SIGHUP by number
```

### Sending Signals

```bash
kill -SIGTERM 1234     # Send SIGTERM to PID 1234
kill -15 1234          # Same by number
kill -9 1234           # SIGKILL
kill -HUP 1234         # SIGHUP
killall nginx          # Send SIGTERM to all processes named nginx
pkill -9 nginx         # Send SIGKILL to all processes matching "nginx"

# From keyboard:
Ctrl+C   # SIGINT  — stop the foreground process
Ctrl+Z   # SIGSTOP — pause the foreground process (resume with 'fg' or 'bg')
Ctrl+\   # SIGQUIT — quit with core dump
```

### Signal Handlers in Code

Applications can register **signal handlers** — functions that run when a specific signal is received:

```python
import signal
import sys

def graceful_shutdown(signum, frame):
    print("SIGTERM received, cleaning up...")
    # Close DB connections, save state, etc.
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

---

## 18. Inter-Process Communication (IPC)

Processes are isolated — they can't access each other's memory. IPC is how they communicate.

### IPC Mechanisms

#### Pipes
A unidirectional communication channel. Data flows one way.

```bash
ls -la | grep ".conf"   # Pipe output of ls into grep
# ls writes to its stdout → pipe → grep reads from its stdin
```

**Named pipes (FIFOs):** Pipes that exist as filesystem entries, allowing unrelated processes to communicate.

#### Unix Domain Sockets
A socket file on the filesystem. Two processes on the same machine communicate through it like a network socket, but faster (no TCP overhead).

Common examples:
- MySQL listens on `/var/run/mysqld/mysqld.sock`
- Docker daemon on `/var/run/docker.sock`
- nginx can talk to PHP-FPM via `/run/php/php8.1-fpm.sock`

```bash
ls -la /var/run/ | grep sock    # See Unix socket files
ss -lx                          # List Unix domain sockets in use
```

#### Shared Memory
The fastest IPC method — two processes map the same physical memory page into their address spaces and communicate by reading/writing to it. Requires careful synchronization to avoid race conditions.

#### Message Queues
Processes send and receive structured messages via a kernel-managed queue. Messages persist until consumed.

#### Signals
As covered above — the simplest IPC, but only for notifications (no data payload).

#### Network Sockets
Communication via TCP/UDP — even between processes on the same machine. More overhead than Unix sockets but works across machines.

---

## 19. Init Systems & systemd

### What is an Init System?

The **init system** is the first process started by the kernel (PID 1). Its job:
1. Start all required system services in the correct order
2. Manage service dependencies
3. Keep services running (restart on crash)
4. Handle system shutdown/reboot

### Evolution of Init Systems

- **SysV init** (old): Shell scripts in `/etc/init.d/`, run level system, sequential startup (slow)
- **Upstart** (Ubuntu 2006-2014): Event-driven, faster than SysV
- **systemd** (modern standard): Parallel startup, declarative unit files, socket activation

### systemd

**systemd** is the init system used by almost all modern Linux distributions (RHEL, CentOS, Ubuntu, Debian, Fedora, Arch, etc.).

Key advantages:
- **Parallel service startup** — massively faster boot times
- **Dependency management** — declares what must be started before/after
- **On-demand activation** — start services only when first needed
- **Automatic restart** — restart failed services automatically
- **Journal** — built-in logging with `journalctl`
- **Cgroups integration** — resource control per service

### systemd Units

systemd manages **units** — configuration files describing resources. Types:

| Unit Type | Extension | Purpose |
|---|---|---|
| **Service** | `.service` | A process/daemon |
| **Socket** | `.socket` | A socket that can activate a service |
| **Timer** | `.timer` | Cron-like scheduled tasks |
| **Mount** | `.mount` | A filesystem mount point |
| **Target** | `.target` | A group of units (like runlevels) |
| **Path** | `.path` | Trigger on filesystem changes |

Unit files live in:
- `/lib/systemd/system/` — system defaults (don't edit these)
- `/etc/systemd/system/` — your overrides (edit these)

### A Service Unit File — Explained

```ini
[Unit]
Description=My Web Application          # Human-readable name
Documentation=https://example.com/docs  # Where to find docs
After=network.target postgresql.service # Start AFTER network and postgres are up
Requires=postgresql.service             # If postgres fails, this fails too

[Service]
Type=simple                  # Process stays in foreground (not daemonizing itself)
User=appuser                 # Run as this user (not root!)
Group=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start.sh       # Command to start the service
ExecReload=/bin/kill -HUP $MAINPID      # Command to reload config
ExecStop=/bin/kill -TERM $MAINPID       # Command to stop gracefully
Restart=on-failure           # Restart if process exits with non-zero code
RestartSec=5                 # Wait 5 seconds before restarting
StandardOutput=journal       # Send stdout to systemd journal
StandardError=journal        # Send stderr to systemd journal
Environment=NODE_ENV=production  # Set env variables

[Install]
WantedBy=multi-user.target   # Start this service when system reaches multi-user mode
```

### Essential systemctl Commands

```bash
# Service management
systemctl start nginx          # Start a service now
systemctl stop nginx           # Stop a service
systemctl restart nginx        # Stop and start
systemctl reload nginx         # Reload config without restart (sends SIGHUP)
systemctl status nginx         # Show detailed status, recent logs, PID

# Boot-time management
systemctl enable nginx         # Start automatically on boot (creates symlink)
systemctl disable nginx        # Don't start on boot
systemctl is-enabled nginx     # Check if enabled
systemctl is-active nginx      # Check if currently running

# Inspecting
systemctl list-units            # All active units
systemctl list-units --failed   # Failed units — first thing to check after boot
systemctl list-unit-files       # All installed units and their state
systemctl cat nginx             # Show the unit file content
systemctl show nginx            # Show all properties of the unit

# System state
systemctl get-default           # Show default target (boot target)
systemctl isolate rescue.target # Switch to rescue mode (single-user)
systemctl reboot                # Reboot
systemctl poweroff              # Shutdown
```

### journalctl — Reading systemd Logs

```bash
journalctl -u nginx                  # Logs for nginx service
journalctl -u nginx -f               # Follow logs in real time (like tail -f)
journalctl -u nginx --since "1 hour ago"  # Logs from last hour
journalctl -u nginx -n 100           # Last 100 lines
journalctl -p err                    # Only error priority and above
journalctl -k                        # Kernel messages (like dmesg)
journalctl --disk-usage              # How much space logs are using
journalctl --vacuum-time=7d          # Delete logs older than 7 days
```

---

## 20. Users, Groups & the Root User

### Users

Every process and file is owned by a user. Users are identified by a **UID (User ID)** — an integer.

```bash
id                     # Show current user's UID, GID, and group memberships
id alice               # Show info for another user
cat /etc/passwd        # User database — one line per user
```

`/etc/passwd` format:
```
username:x:UID:GID:comment:home_directory:shell
root:x:0:0:root:/root:/bin/bash
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
nginx:x:998:998:nginx user:/var/cache/nginx:/sbin/nologin
```

- The `x` in the second field means password is stored in `/etc/shadow` (hashed)
- `/sbin/nologin` as shell means this user can't log in (used for service accounts)
- UIDs 0–99 are reserved for system accounts; UIDs 1000+ are regular users

### Groups

Groups let you grant the same permissions to multiple users. Every user has a **primary group** (set in `/etc/passwd`) and can belong to multiple **supplementary groups**.

```bash
cat /etc/group         # Group database
groups alice           # Show groups alice belongs to
usermod -aG docker alice  # Add alice to the docker group (needs re-login)
newgrp docker          # Switch to docker group in current session
```

### Root — The Superuser

**Root (UID 0)** is the superuser. Root bypasses all permission checks — it can read any file, kill any process, bind to any port.

```bash
sudo command           # Run a single command as root
sudo -i                # Switch to root shell
su - alice             # Switch to alice's account
whoami                 # Who am I right now?
```

**sudo** is configured in `/etc/sudoers`:
```
# Allow alice to run all commands as root
alice ALL=(ALL:ALL) ALL

# Allow the 'deploy' user to restart nginx without a password
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
```

> **Security principle:** Never run applications as root. If they're compromised, the attacker has root access. Always use a dedicated service user with minimal permissions (principle of least privilege).

---

## 21. The /proc and /sys Filesystems

These are **virtual filesystems** — they don't exist on disk. The kernel generates their content on-the-fly when you read them. They are your window into the live state of the kernel.

### /proc — Process and Kernel Info

`/proc` contains a directory for every running process, named by PID, plus system-wide kernel information.

```bash
/proc/
├── 1/                  # Directory for PID 1 (systemd)
│   ├── cmdline         # The command that started this process
│   ├── status          # Memory usage, state, parent PID
│   ├── fd/             # All open file descriptors (symlinks to actual files)
│   ├── maps            # Memory map — what's mapped where
│   ├── environ         # Environment variables
│   └── net/            # Network state for this process's namespace
├── cpuinfo             # CPU model, cores, features
├── meminfo             # Detailed memory usage
├── loadavg             # System load averages (1, 5, 15 min)
├── uptime              # System uptime
├── sys/                # Kernel tunable parameters
├── net/                # Network stats (open connections, interfaces)
└── version             # Kernel version string
```

```bash
cat /proc/cpuinfo                    # CPU details
cat /proc/meminfo                    # Memory details
cat /proc/loadavg                    # Load averages
cat /proc/$(pgrep nginx)/cmdline     # How nginx was started
ls -la /proc/$(pgrep nginx)/fd       # Open files for nginx
cat /proc/sys/net/ipv4/ip_forward    # Check if IP forwarding is enabled
```

### /sys — Hardware and Kernel Parameters

`/sys` exposes the kernel's device model — the hierarchy of hardware devices and drivers.

```bash
/sys/
├── block/          # Block devices (disks)
├── class/          # Device classes (net, input, etc.)
├── devices/        # All devices in the system
└── kernel/         # Kernel internal state
```

### Kernel Tuning with sysctl

`sysctl` reads and writes kernel parameters in `/proc/sys/`:

```bash
sysctl -a                              # Show all kernel parameters
sysctl net.ipv4.ip_forward             # Check IP forwarding
sysctl -w net.ipv4.ip_forward=1        # Enable IP forwarding (temporary)

# Permanent changes go in /etc/sysctl.conf or /etc/sysctl.d/
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/99-custom.conf
sysctl --system                        # Reload all sysctl settings
```

Common sysctl parameters DevOps engineers use:
```bash
# Network performance
net.core.somaxconn = 65535           # Max pending connection queue
net.ipv4.tcp_max_syn_backlog = 65535 # SYN queue size
net.ipv4.ip_local_port_range = 1024 65535  # Ephemeral port range

# Virtual memory
vm.swappiness = 10                   # Prefer RAM over swap (0-100, lower = prefer RAM)
vm.overcommit_memory = 1             # Allow memory overcommit (needed for some apps)

# File system
fs.file-max = 2097152                # System-wide open file limit
fs.inotify.max_user_watches = 524288 # For file watchers (Prometheus, IDEs)
```

---

## 22. CPU Concepts Every DevOps Engineer Must Know

### CPU Cores and Hyperthreading

- **Core:** A physical processing unit — can run one thread at a time
- **Hyperthreading (Intel) / SMT (AMD):** Each physical core presents itself as 2 logical CPUs to the OS — lets the OS fill "waiting" slots in a core's pipeline with work from a second thread
- A 4-core CPU with hyperthreading = 8 logical CPUs to the OS

```bash
lscpu                           # Detailed CPU topology
nproc                           # Number of logical CPU cores
cat /proc/cpuinfo | grep processor | wc -l  # Same thing
```

### Load Average

Load average is one of the most important metrics. It measures how many processes are:
- **Running** (currently on a CPU core)
- **Runnable** (ready to run but waiting for a CPU core)
- **Waiting for disk I/O** (uninterruptible sleep — in some cases)

```bash
uptime
# 14:23:01 up 5 days, 4:12, 2 users, load average: 1.23, 0.88, 0.72
#                                                    1min  5min  15min
```

**How to interpret load average:**
- Load of `1.0` on a single-core machine = 100% utilized
- On a 4-core machine, `4.0` = 100% utilized
- **Rule:** Load average / number of cores = utilization ratio
  - Below 1.0 per core = healthy
  - Above 1.0 per core = processes are queuing → potential bottleneck

```bash
# If load average is 3.5 on a 4-core machine:
# 3.5 / 4 = 0.875 → 87.5% utilization → acceptable
# If load average is 8.0 on a 4-core machine:
# 8.0 / 4 = 2.0 → Processes are waiting for CPU, investigate
```

### Context Switch Cost

Every time the CPU switches between processes, it incurs overhead (saving/loading registers, TLB flushes). High context switch rates (thousands per second) degrade performance.

```bash
vmstat 1              # cs column = context switches per second
pidstat -w 1          # Context switches per process
```

### CPU Affinity

You can pin a process to specific CPU cores, preventing context switches between cores and improving cache performance:

```bash
taskset -c 0,1 ./my_app         # Run my_app on cores 0 and 1 only
taskset -pc 2 1234              # Pin running process 1234 to core 2
```

---

## 23. I/O & Storage Concepts

### How Disk I/O Works

```
Application calls write()
      ↓
Kernel copies data to Page Cache (RAM buffer)
      ↓
Returns to application immediately (non-blocking in most cases)
      ↓
Kernel writes dirty pages to disk asynchronously
      ↓
fsync() forces immediate disk write (critical for databases)
```

### The Page Cache

The kernel uses unused RAM as a **page cache** — a buffer for disk data. Frequently read files are cached in RAM, making subsequent reads much faster.

This is why `free -h` shows high "buff/cache" usage — that's the OS doing its job.

```bash
# Force the OS to drop the page cache (useful for testing or freeing memory)
echo 3 > /proc/sys/vm/drop_caches   # DON'T do this in production casually
```

### I/O Schedulers

The I/O scheduler determines the order in which disk requests are processed. Optimizing for throughput vs latency.

```bash
cat /sys/block/sda/queue/scheduler  # See current scheduler
# [mq-deadline] kyber bfq none

# For SSDs and NVMe: use 'none' or 'mq-deadline'
# For HDDs: 'bfq' or 'mq-deadline'
echo mq-deadline > /sys/block/sda/queue/scheduler
```

### Key I/O Monitoring Tools

```bash
iostat -xz 1           # Detailed I/O stats per disk, per second
#  %util — how busy the disk is (100% = saturated)
#  await — average time for I/O requests (ms) — higher = slower
#  r/s, w/s — read/write operations per second

iotop                   # Which processes are doing the most I/O (like top for disk)

dstat                   # Combined view of CPU, disk, network, memory

# Find which files are being read/written by a process
lsof -p 1234 | grep REG
```

### RAID Concepts

**RAID (Redundant Array of Independent Disks)** combines multiple disks for performance or redundancy:

| RAID Level | Description | Min Disks | Redundancy | Use Case |
|---|---|---|---|---|
| **RAID 0** | Striping — data split across disks | 2 | None | Max performance, no safety |
| **RAID 1** | Mirroring — exact copy on 2 disks | 2 | 1 disk failure | OS disks, simple redundancy |
| **RAID 5** | Striping + parity | 3 | 1 disk failure | Common for NAS/servers |
| **RAID 6** | Striping + double parity | 4 | 2 disk failures | Critical data |
| **RAID 10** | RAID 1 + RAID 0 (mirrored stripes) | 4 | 1 per mirror pair | Databases |

---

## 24. Linux Namespaces & cgroups (Foundation of Containers)

This is where OS fundamentals connect directly to Docker and Kubernetes.

### Linux Namespaces

A **namespace** wraps a system resource in an abstraction layer so that processes inside the namespace see their own isolated instance of that resource.

There are 7 namespaces in Linux:

| Namespace | Isolates | What container gets |
|---|---|---|
| **pid** | Process IDs | Containers have their own PID 1 |
| **net** | Network interfaces, IPs, routing | Container has its own virtual network interface |
| **mnt** | Filesystem mount points | Container sees its own root filesystem |
| **uts** | Hostname and domain name | Container has its own hostname |
| **ipc** | Inter-process communication | Isolated message queues, shared memory |
| **user** | User and group IDs | Container's root can map to an unprivileged host user |
| **cgroup** | cgroup root directory | Container sees its own cgroup hierarchy |

**A container is simply a process with these namespaces applied** — it thinks it's alone on the machine.

```bash
# See what namespaces a process belongs to
ls -la /proc/1234/ns

# Run a command in a new namespace
unshare --pid --mount --fork bash    # Start bash in its own PID and mount namespaces
nsenter -t 1234 -n                   # Enter the network namespace of process 1234
```

### cgroups (Control Groups)

**cgroups** limit and account for resource usage of groups of processes.

While namespaces provide *isolation*, cgroups provide *resource control*:

- **CPU:** Limit how much CPU a process group can use (`cpu.max = 50% of 1 core`)
- **Memory:** Limit RAM and trigger OOM killer if exceeded (`memory.max = 512M`)
- **I/O:** Limit disk read/write speed
- **Network:** (via tc/traffic control)
- **PIDs:** Limit the number of processes

Docker/Kubernetes use cgroups to enforce container resource limits:

```bash
# Docker behind the scenes creates cgroups
docker run --memory=512m --cpus=1 nginx

# Browse cgroups hierarchy
ls /sys/fs/cgroup/

# See cgroup limits for a Docker container
cat /sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes
```

**cgroups v1 vs v2:**
- cgroups v1: Multiple hierarchies, one per resource type — complex
- cgroups v2 (modern): Unified hierarchy — all resource controllers in one tree. Default on modern Linux, required by newer Kubernetes

### How Docker Uses Namespaces + cgroups

When you run `docker run nginx`:

```
Docker daemon:
1. Downloads nginx image (filesystem layers)
2. Creates a new mount namespace → unpack image as new root filesystem (/)
3. Creates a new pid namespace → nginx's first process gets PID 1 in its namespace
4. Creates a new net namespace → assigns a virtual ethernet interface (veth pair)
5. Creates a new uts namespace → assigns container hostname
6. Creates cgroup → limits memory to --memory value, CPU to --cpus value
7. Runs nginx init process inside all these namespaces
```

The nginx process thinks it's alone on a machine with its own filesystem, network, and PID 1. The host kernel is still running everything — it's one process to the host OS.

---

## 25. Quick Reference Cheat Sheet

### Process Commands
```bash
ps aux                    # All processes
ps aux | grep <name>      # Find a process
top / htop                # Live process viewer
pgrep <name>              # Get PID by name
kill <PID>                # Send SIGTERM
kill -9 <PID>             # Send SIGKILL (force)
kill -HUP <PID>           # Send SIGHUP (reload config)
strace -p <PID>           # Trace system calls
lsof -p <PID>             # Open files for a process
nice -n 10 <cmd>          # Run with lower priority
```

### Memory Commands
```bash
free -h                   # RAM and swap usage
vmstat 1                  # Virtual memory stats every second
cat /proc/meminfo         # Detailed memory breakdown
cat /proc/<PID>/status    # Memory usage of a specific process
dmesg | grep -i oom       # Find OOM kill events
```

### Disk & Filesystem Commands
```bash
df -h                     # Disk space by filesystem
df -i                     # Inode usage
lsblk                     # Block devices
mount                     # Show all mounts
iostat -xz 1              # Disk I/O statistics
du -sh /var/log/*         # Disk usage by directory
stat <file>               # Full inode info for a file
ls -li                    # Filenames with inode numbers
```

### System Info Commands
```bash
uname -r                  # Kernel version
uname -a                  # All kernel info
uptime                    # Uptime + load averages
lscpu                     # CPU details
lsmod                     # Loaded kernel modules
dmesg | tail -20          # Recent kernel messages
cat /proc/cpuinfo         # CPU info
cat /proc/version         # Kernel version string
hostnamectl               # Hostname and OS info
```

### systemd / Service Commands
```bash
systemctl status <svc>    # Status of a service
systemctl start/stop/restart <svc>
systemctl enable/disable <svc>
systemctl list-units --failed
journalctl -u <svc> -f    # Follow service logs
journalctl -k             # Kernel messages
```

### Permissions Quick Reference
```bash
chmod 755 file            # rwxr-xr-x
chmod 644 file            # rw-r--r--
chmod 600 file            # rw------- (private keys)
chown user:group file     # Change owner and group
sudo command              # Run as root
id                        # Show current user/group IDs
```

---

## Concept Summary Map

```
HARDWARE (CPU, RAM, Disk, Network)
         ↑ managed by
      KERNEL (Kernel Space)
    /    |     |     \
Processes  Memory  FS  Network
    |        |
 Threads   Virtual Memory / Swap
    |
Scheduler (CFS)
    |
System Calls (boundary between user/kernel space)
    |
USER SPACE PROGRAMS (your app, nginx, bash...)
    |
Signals ←→ IPC (pipes, sockets, shared mem)
    |
Init System (systemd → PID 1 → all services)
    |
Namespaces + cgroups → CONTAINERS (Docker/K8s)
```

---

*Document version 1.0 — covers Linux OS fundamentals for DevOps engineers with 6+ years experience looking to solidify theoretical foundations.*
