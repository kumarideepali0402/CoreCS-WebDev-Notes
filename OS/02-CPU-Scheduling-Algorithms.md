# OS 02 — CPU Scheduling Algorithms

> GATE 2025 Crash Course | CS & IT

---

## Table of Contents

1. [CPU Scheduling — Introduction](#1-cpu-scheduling--introduction)
2. [Key Terminologies & Formulas](#2-key-terminologies--formulas)
3. [Scheduling Criteria](#3-scheduling-criteria)
4. [FCFS — First Come First Served](#4-fcfs--first-come-first-served)
5. [SJF — Shortest Job First (Non-Preemptive)](#5-sjf--shortest-job-first-non-preemptive)
6. [SRTF — Shortest Remaining Time First (Preemptive SJF)](#6-srtf--shortest-remaining-time-first-preemptive-sjf)
7. [Round Robin (RR)](#7-round-robin-rr)
8. [Priority Scheduling](#8-priority-scheduling)
9. [MLQ — Multilevel Queue Scheduling](#9-mlq--multilevel-queue-scheduling)
10. [MLFQ — Multilevel Feedback Queue Scheduling](#10-mlfq--multilevel-feedback-queue-scheduling)
11. [Algorithm Comparison Table](#11-algorithm-comparison-table)
12. [FAANG / MAANG Interview Questions](#12-faang--maang-interview-questions)

---

## 1. CPU Scheduling — Introduction

**CPU Scheduling** is the process by which the OS decides **which process in the ready queue gets the CPU next**.

### Why is CPU Scheduling Needed?
- Processes frequently wait for I/O → CPU would remain idle without scheduling
- Multiple processes compete for a single CPU
- Goal: keep CPU as busy as possible and serve processes fairly

### Preemptive vs Non-Preemptive:

| Type | Description | When CPU can be taken away |
|---|---|---|
| **Non-Preemptive** | Once a process gets CPU, it runs until it finishes or blocks | Only on: completion or I/O wait |
| **Preemptive** | OS can forcibly take CPU away from a running process | On: timer expiry, higher priority arrival, I/O wait, completion |

> **GATE Point:** Preemptive scheduling requires saving/restoring process state → involves **context switch overhead**.

### Scheduling Decision Points:
Scheduling happens when a process:
1. Switches from **Running → Waiting** (I/O request)
2. Switches from **Running → Ready** (timer interrupt)
3. Switches from **Waiting → Ready** (I/O completes)
4. **Terminates**

Cases 1 & 4 → only non-preemptive choice available  
Cases 2 & 3 → choice of preemptive or non-preemptive

---

## 2. Key Terminologies & Formulas

| Term | Symbol | Definition |
|---|---|---|
| **Arrival Time** | AT | Time at which process enters the ready queue |
| **Burst Time** | BT | Total CPU time the process needs |
| **Completion Time** | CT | Time at which process finishes execution |
| **Turnaround Time** | TAT | Total time from arrival to completion |
| **Waiting Time** | WT | Total time spent waiting in the ready queue |
| **Response Time** | RT | Time from arrival until process first gets the CPU |

### Core Formulas:

```
TAT = CT − AT

WT  = TAT − BT  =  CT − AT − BT

RT  = First CPU Time − AT

Avg TAT = Σ(TAT) / n

Avg WT  = Σ(WT) / n

CPU Utilization (%) = (CPU busy time / Total time) × 100

Throughput = Number of processes completed / Total time
```

> **Tip:** Always draw a Gantt chart first — it makes CT trivial to read off.

---

## 3. Scheduling Criteria

| Criterion | Goal | Description |
|---|---|---|
| **CPU Utilization** | Maximize | Keep CPU busy as much as possible |
| **Throughput** | Maximize | More processes completed per unit time |
| **Turnaround Time** | Minimize | Less total time per process |
| **Waiting Time** | Minimize | Less time sitting idle in ready queue |
| **Response Time** | Minimize | Quick first response (critical for interactive systems) |

> **GATE Point:** No single algorithm optimizes all criteria simultaneously — trade-offs always exist.

---

## 4. FCFS — First Come First Served

- **Type:** Non-Preemptive
- **Policy:** Process that arrives first gets CPU first (FIFO queue)
- **No starvation** — every process eventually gets the CPU

### Example:

| Process | AT | BT |
|---|---|---|
| P1 | 0 | 5 |
| P2 | 1 | 3 |
| P3 | 2 | 8 |

**Gantt Chart:**
```
| P1  |  P2  |       P3        |
0     5      8                16
```

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P1 | 0 | 5 | 5 | 5 | 0 |
| P2 | 1 | 3 | 8 | 7 | 4 |
| P3 | 2 | 8 | 16 | 14 | 6 |

- **Avg TAT** = (5 + 7 + 14) / 3 = **8.67**
- **Avg WT** = (0 + 4 + 6) / 3 = **3.33**

### Convoy Effect:
> A long process at the head of the queue forces all short processes behind it to wait — called the **Convoy Effect**.

```
P_long(BT=100) → P_short1(BT=1) → P_short2(BT=1)
Both short processes wait 100 units unnecessarily
```

### FCFS Properties:
| Property | Value |
|---|---|
| Preemptive? | No |
| Starvation? | No |
| Optimal for WT? | No |
| Main Problem | **Convoy Effect** |

---

## 5. SJF — Shortest Job First (Non-Preemptive)

- **Type:** Non-Preemptive
- **Policy:** Among processes in the ready queue, pick the one with the **smallest burst time**
- **Proven optimal** — gives **minimum average waiting time** among non-preemptive algorithms

### Example:

| Process | AT | BT |
|---|---|---|
| P1 | 0 | 6 |
| P2 | 1 | 4 |
| P3 | 2 | 2 |
| P4 | 3 | 8 |

At t=0 only P1 available → P1 runs (non-preemptive, can't switch mid-burst).  
At t=6: Ready = {P2(4), P3(2), P4(8)} → pick **P3** (shortest).  
At t=8: Ready = {P2(4), P4(8)} → pick **P2**.  
At t=12: Only P4 → runs.

**Gantt Chart:**
```
|   P1   |  P3 |   P2   |       P4        |
0        6     8       12                 20
```

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P1 | 0 | 6 | 6 | 6 | 0 |
| P2 | 1 | 4 | 12 | 11 | 7 |
| P3 | 2 | 2 | 8 | 6 | 4 |
| P4 | 3 | 8 | 20 | 17 | 9 |

- **Avg WT** = (0 + 7 + 4 + 9) / 4 = **5.0**

### Problems with SJF:
1. **Starvation** — long processes may wait indefinitely if short processes keep arriving
2. **Burst time not known in advance** — must be estimated
3. Estimation via **exponential averaging:**
   ```
   next_burst = α × actual_burst + (1 − α) × previous_estimate
   ```

### SJF Properties:
| Property | Value |
|---|---|
| Preemptive? | No |
| Starvation? | **Yes** |
| Optimal? | **Yes — minimum Avg WT (non-preemptive)** |
| Main Problem | Starvation, burst time must be known/estimated |

---

## 6. SRTF — Shortest Remaining Time First (Preemptive SJF)

- **Type:** Preemptive
- **Policy:** At every new arrival, compare its BT with the **remaining time** of the current process; if new process is shorter → preempt
- Also optimal — gives **minimum average waiting time overall**

### Example:

| Process | AT | BT |
|---|---|---|
| P1 | 0 | 8 |
| P2 | 1 | 4 |
| P3 | 2 | 9 |
| P4 | 3 | 5 |

**Trace:**
```
t=0: Only P1 → P1 runs. P1 remaining = 8
t=1: P2 arrives (BT=4). P1 rem=7 → 4 < 7 → preempt P1, run P2
t=2: P3 arrives (BT=9). P2 rem=3 → 3 < 9 → P2 continues
t=3: P4 arrives (BT=5). P2 rem=2 → 2 < 5 → P2 continues
t=5: P2 done. Ready = {P1(rem=7), P3(9), P4(5)} → run P4
t=10: P4 done. Ready = {P1(rem=7), P3(9)} → run P1
t=17: P1 done. Only P3 left → run P3
t=26: P3 done.
```

**Gantt Chart:**
```
|P1| P2  |   P4   |    P1    |         P3          |
0  1     5       10         17                     26
```

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P1 | 0 | 8 | 17 | 17 | 9 |
| P2 | 1 | 4 | 5 | 4 | 0 |
| P3 | 2 | 9 | 26 | 24 | 15 |
| P4 | 3 | 5 | 10 | 7 | 2 |

- **Avg WT** = (9 + 0 + 15 + 2) / 4 = **6.5**

### SRTF Properties:
| Property | Value |
|---|---|
| Preemptive? | **Yes** |
| Starvation? | **Yes** |
| Optimal? | **Yes — minimum Avg WT (overall)** |
| Why not practical? | Burst time unknown; high preemption overhead |

---

## 7. Round Robin (RR)

- **Type:** Preemptive
- **Policy:** Each process gets a fixed **time quantum (q)**; processes cycle through in FIFO order
- Designed for **time-sharing / interactive** systems
- **No starvation** — every process gets CPU within bounded time

### Algorithm:
```
1. Ready queue is a circular FIFO queue
2. Pick next process, give it CPU for q time units
3. If process finishes within q → CPU released voluntarily
4. If process does NOT finish → preempt, move to END of ready queue
5. Repeat
```

### Example (q = 2):

| Process | AT | BT |
|---|---|---|
| P1 | 0 | 5 |
| P2 | 1 | 3 |
| P3 | 2 | 1 |
| P4 | 3 | 2 |

**Trace:**
```
t=0:  Queue={P1}. Run P1 (q=2). t=2, P1 rem=3
t=2:  P2,P3 arrived. Queue={P2,P3,P1(rem=3)}. Run P2 (q=2). t=4, P2 rem=1
t=4:  P4 arrived. Queue={P3,P1(rem=3),P4,P2(rem=1)}. Run P3 (BT=1). t=5 (P3 done)
t=5:  Queue={P1(rem=3),P4,P2(rem=1)}. Run P1 (q=2). t=7, P1 rem=1
t=7:  Queue={P4,P2(rem=1),P1(rem=1)}. Run P4 (BT=2,q=2). t=9 (P4 done)
t=9:  Queue={P2(rem=1),P1(rem=1)}. Run P2 (1 unit). t=10 (P2 done)
t=10: Queue={P1(rem=1)}. Run P1 (1 unit). t=11 (P1 done)
```

**Gantt Chart:**
```
| P1 | P2 |P3| P1 | P4 |P2|P1|
0    2    4  5    7    9 10 11
```

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P1 | 0 | 5 | 11 | 11 | 6 |
| P2 | 1 | 3 | 10 | 9 | 6 |
| P3 | 2 | 1 | 5 | 3 | 2 |
| P4 | 3 | 2 | 9 | 6 | 4 |

### Effect of Quantum Size:

| Quantum | Effect |
|---|---|
| Very **small** (q → 0) | Near-ideal concurrency but huge context-switch overhead |
| Very **large** (q → ∞) | Degenerates to **FCFS** |
| **Rule of thumb** | 80% of processes should complete within one quantum |

### Round Robin Properties:
| Property | Value |
|---|---|
| Preemptive? | **Yes** |
| Starvation? | **No** |
| Best metric | **Response Time** (best among all algorithms) |
| Avg WT | Not minimum (worse than SJF/SRTF) |
| q → ∞ | **= FCFS** |

---

## 8. Priority Scheduling

- **Policy:** Each process has a **priority number**; CPU goes to highest priority process
- Convention: **smaller number = higher priority** (P0 is highest)
- Can be **preemptive** or **non-preemptive**

### Preemptive Priority:
If a higher-priority process arrives while a lower-priority process is running → **preempt immediately**

### Non-Preemptive Priority:
Current process runs to completion; on next scheduling decision pick highest priority from ready queue

### Example (Non-Preemptive, lower number = higher priority):

| Process | AT | BT | Priority |
|---|---|---|---|
| P1 | 0 | 4 | 3 |
| P2 | 1 | 3 | 1 |
| P3 | 2 | 1 | 4 |
| P4 | 3 | 5 | 2 |

At t=0: only P1 → runs (non-preemptive, won't be interrupted).  
At t=4: Ready = {P2(pri=1), P3(pri=4), P4(pri=2)} → **P2** (priority 1 = highest).  
At t=7: Ready = {P3(pri=4), P4(pri=2)} → **P4**.  
At t=12: **P3** runs.

**Gantt Chart:**
```
|   P1  |  P2  |     P4     |P3|
0       4      7           12 13
```

| Process | AT | BT | CT | TAT | WT |
|---|---|---|---|---|---|
| P1 | 0 | 4 | 4 | 4 | 0 |
| P2 | 1 | 3 | 7 | 6 | 3 |
| P3 | 2 | 1 | 13 | 11 | 10 |
| P4 | 3 | 5 | 12 | 9 | 4 |

### Starvation & Aging:

> **Starvation:** A low-priority process may wait **indefinitely** as high-priority processes keep arriving.

> **Aging (Fix):** Gradually increase the priority of waiting processes over time.  
> Example: Every 15 minutes waiting → increase priority by 1.

### Key Relationship:
> **SJF is a special case of Priority Scheduling** where priority = 1 / Burst Time  
> (shorter burst = higher priority)

### Priority Scheduling Properties:
| Property | Value |
|---|---|
| Preemptive version? | Yes |
| Non-preemptive version? | Yes |
| Starvation? | **Yes** |
| Fix for starvation | **Aging** |

---

## 9. MLQ — Multilevel Queue Scheduling

Ready queue is split into **multiple permanent queues** based on process type/category.

### Typical Structure:

```
┌─────────────────────────────────────┐  Priority 1 (Highest)
│       System Processes              │  → Round Robin
├─────────────────────────────────────┤  Priority 2
│     Interactive Processes           │  → Round Robin
├─────────────────────────────────────┤  Priority 3
│   Interactive Editing Processes     │  → Round Robin
├─────────────────────────────────────┤  Priority 4
│         Batch Processes             │  → FCFS
├─────────────────────────────────────┤  Priority 5 (Lowest)
│        Student Processes            │  → FCFS
└─────────────────────────────────────┘
```

### Rules:
- Each process is **permanently assigned** to a queue (no movement)
- Higher-priority queue always has **absolute priority** over lower queues
- A lower queue runs only when **all higher queues are empty**
- Each queue has its **own scheduling algorithm**

### MLQ Properties:
| Property | Value |
|---|---|
| Process movement between queues? | **No (fixed)** |
| Starvation? | **Yes** (low-priority queues can starve) |
| Preemptive between queues? | Yes |
| Limitation | Rigid — processes permanently categorized |

---

## 10. MLFQ — Multilevel Feedback Queue Scheduling

Extension of MLQ — processes **can move between queues** based on CPU behavior.  
Most **complex** but most **flexible and fair** — used in real operating systems.

### Key Idea:
- New processes start at the **highest priority queue**
- Process uses entire quantum → **demote** (CPU-bound → lower priority)
- Process gives up CPU before quantum ends → **stay** (I/O-bound → stays high priority)
- Process waits too long → **promote** (aging → prevents starvation)

### Typical 3-Queue Structure:

```
Queue 0 ── RR, q=8ms    ← new processes enter here (highest priority)
Queue 1 ── RR, q=16ms   ← demoted from Queue 0
Queue 2 ── FCFS         ← demoted from Queue 1 (lowest priority)
```

**Flow:**
```
New process → Q0 (q=8ms)
    ├── Finishes in ≤8ms → Done  (treated as interactive/short)
    └── Uses full 8ms → Demoted to Q1 (q=16ms)
            ├── Finishes in ≤16ms → Done
            └── Uses full 16ms → Demoted to Q2 (FCFS, long job)
```

### MLFQ Parameters (configurable):
- Number of queues
- Scheduling algorithm per queue
- Time quantum per queue
- When to promote (boost) / demote a process

### MLFQ Properties:
| Property | Value |
|---|---|
| Starvation? | **No** (boosting/aging prevents it) |
| Preemptive? | Yes |
| Complexity | **Highest** among all algorithms |
| Used in | Linux (CFS), Windows, macOS |
| Advantage | Adapts to process behavior automatically |

---

## 11. Algorithm Comparison Table

| Algorithm | Preemptive | Starvation | Avg WT | Response Time | Best For |
|---|---|---|---|---|---|
| **FCFS** | No | No | High | High | Simple batch |
| **SJF** | No | **Yes** | **Min (non-pre)** | Medium | Batch (known BT) |
| **SRTF** | **Yes** | **Yes** | **Min (overall)** | Low | Optimal WT |
| **Round Robin** | **Yes** | No | Medium | **Best** | Time-sharing, interactive |
| **Priority (NP)** | No | **Yes** | Medium | Medium | Batch with priorities |
| **Priority (P)** | **Yes** | **Yes** | Low | Low | Real-time |
| **MLQ** | Between queues | **Yes** | Low | Low | Mixed workloads |
| **MLFQ** | **Yes** | **No** | Lowest | Best | General-purpose OS |

---

## 12. FAANG / MAANG Interview Questions

> Questions commonly asked at Google, Amazon, Microsoft, Meta, Apple, Flipkart and other top tech companies.

---

### Core Concepts

**Q1. Why is SJF optimal but almost never used in production OS?**
> SJF gives provably minimum average waiting time, but it requires knowing the **burst time in advance** — which is impossible in a general-purpose OS. Real systems estimate it using exponential averaging, but the estimate can be wildly wrong. Additionally, it causes starvation for long jobs.

**Q2. What scheduling algorithm does Linux use?**
> Linux uses the **CFS (Completely Fair Scheduler)** since kernel 2.6.23. It models CPU as an ideal processor shared equally among all runnable processes, tracked via **virtual runtime (vruntime)**. The process with the lowest vruntime runs next. CFS uses a red-black tree for O(log n) scheduling. It's inspired by MLFQ principles but fairer.

**Q3. What scheduling algorithm does Windows use?**
> Windows uses a **priority-based preemptive scheduler** with 32 priority levels (0–31). The highest-priority ready thread always runs. It uses **priority boosting** — temporarily raises priority of threads coming off I/O waits to improve responsiveness — similar to aging.

**Q4. What is priority inversion? Give a real-world example.**
> Priority inversion: a **high-priority task** is blocked waiting for a resource held by a **low-priority task**, while a **medium-priority task** preempts the low-priority one — the high-priority task is now indirectly blocked by the medium one. **NASA Mars Pathfinder (1997):** a high-priority meteorological thread was starved because a low-priority data bus task held a shared mutex, and a medium-priority thread kept running. The spacecraft kept resetting. Fixed by enabling **Priority Inheritance** in the VxWorks OS config.

**Q5. How does the Priority Inheritance Protocol solve priority inversion?**
> When a high-priority task blocks on a mutex held by a low-priority task, the OS **temporarily raises** the low-priority task's priority to match the high-priority blocker. This prevents medium-priority tasks from preempting it, so the low-priority task finishes quickly and releases the mutex.

---

### Design & Trade-off Questions

**Q6. When would you choose non-preemptive scheduling over preemptive?**
> Non-preemptive is better for **batch processing** with long CPU-bound jobs — fewer context switches means less overhead and better throughput. It's also simpler to implement and avoids race conditions in the scheduler itself. Preemptive is essential for interactive and real-time systems where responsiveness matters.

**Q7. What happens if the Round Robin time quantum is too small? Too large?**
> **Too small:** excessive context switching — overhead dominates, effective throughput collapses. **Too large:** degenerates to FCFS — long jobs monopolize CPU, response time suffers. Rule of thumb: 80% of CPU bursts should be shorter than one quantum (typically 10–100ms).

**Q8. How does MLFQ determine whether a process is CPU-bound or I/O-bound?**
> MLFQ doesn't explicitly classify — it **infers behavior**. If a process uses its entire time quantum (CPU-bound behavior) → demote to lower queue. If it voluntarily gives up CPU before the quantum expires (I/O-bound behavior) → stays at current priority or gets boosted. Over time, CPU-bound processes sink to lower queues, I/O-bound stay high.

**Q9. What is the thundering herd problem in the context of scheduling?**
> When a shared event (like a new connection arriving) wakes up all waiting threads/processes, only one can handle it — the rest do useless work before going back to sleep, burning CPU. Solved in Linux with `EPOLLEXCLUSIVE` (wake only one) and `accept4()` with `SOCK_NONBLOCK`.

**Q10. What is cooperative vs preemptive multitasking? What are the risks of cooperative?**
> **Cooperative:** processes voluntarily yield the CPU — simpler, no race conditions in the scheduler, but a buggy or malicious process can hog the CPU indefinitely. **Preemptive:** OS forces preemption via timer interrupts — guarantees fairness and responsiveness. All modern general-purpose OS use preemptive multitasking.

---

### System Design Angle

**Q11. How would you design a scheduler for a web server handling thousands of connections?**
> Use an **event-driven model** (like nginx) with a small number of worker threads (≈ CPU cores), each running an **epoll/kqueue event loop**. This avoids the context-switching overhead of a thread-per-connection model. Within threads, use cooperative scheduling via async I/O. For CPU-heavy tasks, offload to a separate thread pool.

**Q12. What is process starvation? How does aging prevent it?**
> Starvation: a low-priority process waits indefinitely because high-priority processes keep arriving. **Aging** gradually increases a waiting process's priority over time (e.g., +1 every 15 seconds), so even the lowest-priority process eventually gets a high enough priority to run.

**Q13. What is the difference between MLQ and MLFQ? Why is MLFQ preferred?**
> **MLQ:** processes are permanently assigned to a fixed queue — no movement. Inflexible, causes starvation of lower queues. **MLFQ:** processes move between queues based on behavior. I/O-bound processes stay high priority (good response time). CPU-bound sink lower. Aging prevents starvation. MLFQ is used in real OS because it adapts to actual workload.

---

### Quick Formula Reference:

```
TAT  = CT − AT
WT   = TAT − BT
RT   = First CPU time − AT
```

---
