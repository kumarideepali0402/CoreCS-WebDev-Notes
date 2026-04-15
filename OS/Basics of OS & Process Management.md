#  Basics of Operating Systems & Process Management



---

## Table of Contents

1. [What is an Operating System?](#1-what-is-an-operating-system)
2. [Goals and Functions of OS](#2-goals-and-functions-of-os)
3. [Types of Operating Systems](#3-types-of-operating-systems)
4. [System Calls](#4-system-calls)
5. [What is a Process?](#5-what-is-a-process)
6. [Process in Memory](#6-process-in-memory)
7. [Process States](#7-process-states)
8. [Process Control Block (PCB)](#8-process-control-block-pcb)
9. [Context Switching](#9-context-switching)
10. [Process Scheduling & Schedulers](#10-process-scheduling--schedulers)
11. [CPU Scheduling Algorithms](#11-cpu-scheduling-algorithms)
12. [GATE PYQ Tips & Key Points](#12-gate-pyq-tips--key-points)

---

## 1. What is an Operating System?

An **Operating System (OS)** is **system software** that acts as an **intermediary between the user and the hardware**.

```
User / Applications
        ↕
  Operating System       ← manages resources
        ↕
     Hardware
```

> **Simple Definition:** OS = Resource Manager + Interface Provider

### Why do we need an OS?
- Hardware alone is too complex to program directly
- Multiple users/programs need to share CPU, memory, and I/O fairly
- OS provides **abstraction** — hides hardware complexity from users

---

## 2. Goals and Functions of OS

### Goals:
| Goal | Description |
|---|---|
| **Convenience** | Makes the computer easier to use |
| **Efficiency** | Manages hardware resources optimally |
| **Ability to Evolve** | Allows new features without disrupting existing service |

### Functions:
- **Process Management** — create, schedule, terminate processes
- **Memory Management** — allocate/deallocate RAM, virtual memory
- **File System Management** — organize data on disk
- **I/O Management** — control I/O devices via drivers
- **Security & Protection** — prevent unauthorized access
- **Networking** — manage network communication

---

## 3. Types of Operating Systems

### 3.1 Batch Operating System
- Jobs are collected in **batches** and submitted all at once
- No direct interaction between user and job during execution
- CPU may remain **idle** when waiting for I/O

```
Jobs → Batch → CPU executes one by one
```

| Pros | Cons |
|---|---|
| Good throughput for similar jobs | No user interaction |
| Simple to implement | Starvation of long jobs possible |

---

### 3.2 Multiprogramming OS
- **Multiple programs** are kept in memory simultaneously
- When one process waits for I/O, CPU switches to another → **no CPU idle time**
- Goal: **maximize CPU utilization**

```
P1 (running) → P1 waits for I/O → CPU switches to P2
P2 (running) → P2 waits for I/O → CPU switches to P3
```

> **Key Point:** Multiprogramming = multiple programs in memory, CPU does NOT time-share — it switches only on I/O wait.

---

### 3.3 Multitasking / Time-Sharing OS
- Extension of multiprogramming with **time slices (quantum)**
- CPU switches between processes so rapidly that each user feels they have **dedicated CPU**
- Enables **interactive** computing

| Feature | Multiprogramming | Multitasking |
|---|---|---|
| Switching trigger | I/O wait | Timer (quantum) expiry OR I/O |
| User interaction | No | Yes |
| Response time | Not a priority | Priority |
| Example | Batch systems | UNIX, Windows |

---

### 3.4 Real-Time OS (RTOS)
- Must respond within a **guaranteed time deadline**
- Two types:

| Type | Description | Example |
|---|---|---|
| **Hard RTOS** | Missing deadline = system failure | Pacemaker, Air traffic control |
| **Soft RTOS** | Missing deadline = degraded performance | Multimedia, Gaming |

---

### 3.5 Distributed OS
- Manages a **group of networked computers** to appear as a single system
- Resources (CPU, memory) are distributed across nodes
- Example: Google's data center systems

---

### 3.6 Multi-Processor OS
- **Multiple CPUs** share the same memory and bus
- Two types:
  - **Symmetric Multiprocessing (SMP)** — all CPUs are equal; OS runs on any
  - **Asymmetric Multiprocessing** — master-slave; one CPU controls the rest

---

## 4. System Calls

A **system call** is the mechanism by which a user program requests a **service from the OS kernel**.

```
User Program → System Call → Kernel (OS) → Hardware
```

### Privilege Modes:
| Mode | Also Called | Who runs here |
|---|---|---|
| **User Mode** | Ring 3 | User applications |
| **Kernel Mode** | Ring 0 / Supervisor Mode | OS kernel |

> A system call **switches from user mode to kernel mode** — this is called a **mode switch** (or trap).

### Types of System Calls:

| Category | Examples |
|---|---|
| **Process Control** | `fork()`, `exec()`, `exit()`, `wait()` |
| **File Management** | `open()`, `read()`, `write()`, `close()` |
| **Device Management** | `ioctl()`, `read()`, `write()` |
| **Information Maintenance** | `getpid()`, `time()` |
| **Communication** | `pipe()`, `socket()`, `send()`, `recv()` |

---

## 5. What is a Process?

> **Process = Program in Execution**

| Concept | Definition |
|---|---|
| **Program** | Passive entity — a file stored on disk (`.exe`, `.out`) |
| **Process** | Active entity — program loaded into memory and executing |

A single program can create **multiple processes** (e.g., opening Chrome twice = two processes).

### What does a process need?
- **CPU time** — to execute instructions
- **Memory** — to hold its code, data, and stack
- **Files** — I/O resources
- **I/O devices** — for input/output operations

---

## 6. Process in Memory

When a process is created, the OS allocates memory divided into 4 sections:

```
High Address ┌─────────────┐
             │    Stack    │  ← grows downward (local vars, function calls)
             ├─────────────┤
             │     ↓       │
             │    (gap)    │
             │     ↑       │
             ├─────────────┤
             │    Heap     │  ← grows upward (dynamic memory: malloc/new)
             ├─────────────┤
             │    Data     │  ← global & static variables
             ├─────────────┤
Low Address  │    Text     │  ← program code (read-only)
             └─────────────┘
```

| Section | Contents | Size |
|---|---|---|
| **Text** | Compiled machine code | Fixed |
| **Data** | Initialized global/static variables | Fixed |
| **Heap** | Dynamic memory (`malloc`, `new`) | Variable (grows up) |
| **Stack** | Local variables, return addresses, function frames | Variable (grows down) |

> **GATE Point:** Stack and Heap grow toward each other. Stack overflow occurs when they collide.

---

## 7. Process States

A process moves through **5 states** during its lifetime:

```
        admit              dispatch
New ──────────→ Ready ──────────────→ Running
                  ↑                      │
                  │   interrupt           │  I/O or event wait
                  └──────────────────────┘
                                         │
                                         ↓
                                      Waiting ──→ (I/O complete) ──→ Ready
                                         
Running ──────────────────────────────→ Terminated
               exit
```

### State Descriptions:

| State | Description |
|---|---|
| **New** | Process is being created |
| **Ready** | Loaded in memory, waiting for CPU allocation |
| **Running** | Currently executing on CPU |
| **Waiting / Blocked** | Waiting for I/O or an event to complete |
| **Terminated** | Process has finished execution |

### State Transitions:

| Transition | Trigger |
|---|---|
| New → Ready | Process admitted to ready queue |
| Ready → Running | Short-term scheduler dispatches (CPU assigned) |
| Running → Ready | **Preempted** by interrupt or time quantum expiry |
| Running → Waiting | Process requests I/O or waits for event |
| Waiting → Ready | I/O completes or event occurs |
| Running → Terminated | Process calls `exit()` or is killed |

> **GATE Point:** A process can go from Running → Ready (preemption) but NEVER from Waiting → Running directly. It must go through Ready first.

---

## 8. Process Control Block (PCB)

The **PCB** (also called **Task Control Block**) is a **data structure** maintained by the OS for every process. It stores all information about a process.

```
┌─────────────────────────────┐
│       Process State         │  (New, Ready, Running, Waiting, Terminated)
├─────────────────────────────┤
│       Process ID (PID)      │  (unique identifier)
├─────────────────────────────┤
│      Program Counter (PC)   │  (address of next instruction)
├─────────────────────────────┤
│        CPU Registers        │  (all register values saved here)
├─────────────────────────────┤
│    CPU Scheduling Info      │  (priority, scheduling queue pointers)
├─────────────────────────────┤
│   Memory Management Info    │  (base/limit registers, page tables)
├─────────────────────────────┤
│     Accounting Information  │  (CPU time used, time limits)
├─────────────────────────────┤
│      I/O Status Info        │  (open files, I/O devices allocated)
└─────────────────────────────┘
```

> **PCB = "snapshot" of a process** — everything needed to restart it from where it stopped.

In Linux, the PCB is represented by the `task_struct` structure in the kernel.

---

## 9. Context Switching

**Context switching** is the process of **saving the state of one process and restoring the state of another** when the CPU switches between processes.

### Steps in Context Switch:

```
Process P1 running
        ↓
1. Save CPU state of P1 → into P1's PCB
2. Update P1's state (Running → Ready or Waiting)
3. Select next process P2 (by scheduler)
4. Restore CPU state of P2 ← from P2's PCB
5. Update P2's state (Ready → Running)
        ↓
Process P2 running
```

### Why is Context Switching Necessary?
- Prevents data leakage between processes (security)
- Ensures each process resumes correctly from where it paused
- Enables concurrency on a single CPU

### Cost of Context Switching:
- Context switching is **pure overhead** — no useful work is done during the switch
- Involves saving/restoring registers, updating PCBs, flushing cache (TLB flush)
- Modern OS minimizes context switches for performance

> **GATE Point:** Context switch time is **overhead** — it is wasted CPU time. Systems try to minimize it.

---

## 10. Process Scheduling & Schedulers

### What is Scheduling?
The OS decides **which process gets the CPU** next — this is called **CPU scheduling**.

### Three Types of Schedulers:

| Scheduler | Also Called | Function | Frequency |
|---|---|---|---|
| **Long-term** | Job Scheduler | Selects which jobs from disk enter memory (ready queue) | Infrequent |
| **Short-term** | CPU Scheduler | Selects which ready process gets the CPU next | Very frequent (ms) |
| **Medium-term** | Swapper | Swaps processes in/out of memory (swapping) | Moderate |

```
Disk → [Long-term scheduler] → Ready Queue → [Short-term scheduler] → CPU
                                    ↑                    ↓
                               [Medium-term]         Running
                               (swapping)                ↓
                                                     Waiting Queue
```

### Degree of Multiprogramming:
- Controlled by the **long-term scheduler**
- = number of processes simultaneously in memory

### Scheduling Criteria (what we want to optimize):

| Criterion | Goal | Notes |
|---|---|---|
| **CPU Utilization** | Maximize | Keep CPU busy |
| **Throughput** | Maximize | Processes completed per unit time |
| **Turnaround Time** | Minimize | Finish time − Arrival time |
| **Waiting Time** | Minimize | Time spent in ready queue |
| **Response Time** | Minimize | First response − Arrival time |

> **Key Formulas:**
> ```
> Turnaround Time (TAT) = Completion Time − Arrival Time
> Waiting Time (WT)     = TAT − Burst Time
> Response Time         = First CPU Time − Arrival Time
> ```

---

## 11. CPU Scheduling Algorithms

### 11.1 FCFS — First Come First Served
- **Non-preemptive** — process runs until it finishes or blocks
- Jobs are executed in **arrival order**
- Simple queue (FIFO)

**Example:**

| Process | Arrival | Burst |
|---|---|---|
| P1 | 0 | 5 |
| P2 | 1 | 3 |
| P3 | 2 | 8 |

Gantt Chart: `P1(0-5) | P2(5-8) | P3(8-16)`

| Process | TAT | WT |
|---|---|---|
| P1 | 5−0=5 | 0 |
| P2 | 8−1=7 | 4 |
| P3 | 16−2=14 | 6 |

> **Disadvantage:** **Convoy Effect** — short processes stuck behind a long process.

---

### 11.2 SJF — Shortest Job First
- **Non-preemptive** — selects the process with the **shortest burst time** from the ready queue
- Optimal for minimizing **average waiting time** (proven)

> **Disadvantage:** **Starvation** of long processes if short jobs keep arriving.

---

### 11.3 SRTF — Shortest Remaining Time First
- **Preemptive** version of SJF
- If a new process arrives with a **shorter remaining time** than the current process → preempt
- Also optimal for minimizing average waiting time

---

### 11.4 Round Robin (RR)
- Each process gets a **fixed time slice (quantum = q)**
- **Preemptive** — if process doesn't finish in q units, it goes back to end of ready queue
- Designed for **time-sharing** systems

```
q = 2 units, processes: P1(5), P2(3), P3(8)
Gantt: P1(0-2) P2(2-4) P3(4-6) P1(6-8) P2(8-9) P3(9-11) P1(11-12) P3(12-14) P3(14-16)
```

| q value | Effect |
|---|---|
| Very small | High context switch overhead |
| Very large | Degenerates to FCFS |
| q → ∞ | Same as FCFS |

> **GATE Point:** RR gives the **best response time** among all algorithms.  
> Average waiting time is NOT always minimum for RR.

---

### 11.5 Priority Scheduling
- Each process is assigned a **priority number**
- CPU given to the process with the **highest priority** (smallest number = highest priority in most systems)
- Can be **preemptive** or **non-preemptive**

> **Problem:** **Indefinite Blocking (Starvation)** — low priority process may never execute  
> **Solution:** **Aging** — gradually increase the priority of waiting processes over time

---

### Summary Table — Scheduling Algorithms

| Algorithm | Preemptive? | Criteria | Problem |
|---|---|---|---|
| FCFS | No | Arrival order | Convoy effect |
| SJF | No | Shortest burst | Starvation, needs future knowledge |
| SRTF | Yes | Shortest remaining | Starvation |
| Round Robin | Yes | Time quantum | High overhead for small q |
| Priority | Both | Priority value | Starvation (fixed by aging) |

---

## 12. GATE PYQ Tips & Key Points

### Must-Remember Facts:

1. **Process = Program in Execution.** A program is passive; a process is active.

2. **PCB stores everything** about a process: PID, state, PC, registers, memory info, I/O info.

3. **Context Switch is pure overhead** — no useful work done during the switch.

4. **Process states:** New → Ready → Running → Waiting → Terminated  
   - Waiting → Running is **NOT** a direct transition (must go through Ready).

5. **Short-term scheduler** (CPU scheduler) runs most frequently — every few milliseconds.  
   **Long-term scheduler** runs infrequently — controls degree of multiprogramming.

6. **SJF is optimal** — gives minimum average waiting time among non-preemptive algorithms.

7. **Round Robin** gives best **response time** / **interactive performance**.

8. **Aging** solves starvation in Priority Scheduling.

9. **FCFS Convoy Effect** — one long process blocks all short processes behind it.

10. **Stack grows downward**, Heap grows upward. They grow toward each other.

11. **Multiprogramming** — switches on I/O wait only (non-preemptive in nature).  
    **Multitasking** — switches on timer quantum expiry too (preemptive).

---

### Quick Revision — Key Formulas:

| Formula | Expression |
|---|---|
| Turnaround Time | `Completion Time − Arrival Time` |
| Waiting Time | `Turnaround Time − Burst Time` |
| Response Time | `First CPU Time − Arrival Time` |
| CPU Utilization | `CPU busy time / Total time × 100%` |
| Throughput | `No. of processes / Total time` |

---

### Frequently Asked GATE Question Types:

- Calculate TAT, WT, response time for given scheduling algorithm
- Draw Gantt chart for FCFS, SJF, SRTF, RR, Priority
- Identify which scheduling algorithm is being described
- Which algorithm gives minimum average waiting time?
- What happens to RR when quantum → ∞?
- Which algorithm suffers from convoy effect / starvation?
- Identify PCB fields or what gets saved during context switch
- Process state transition questions (can a process go directly from Waiting → Running?)

---
