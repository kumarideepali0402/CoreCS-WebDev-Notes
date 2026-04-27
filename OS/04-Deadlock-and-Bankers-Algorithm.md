# OS 04 — Deadlock & Banker's Algorithm



## Table of Contents

1. [What is Deadlock?](#1-what-is-deadlock)
2. [Coffman's Four Necessary Conditions](#2-coffmans-four-necessary-conditions)
3. [Resource Allocation Graph (RAG)](#3-resource-allocation-graph-rag)
4. [Deadlock Handling Strategies](#4-deadlock-handling-strategies)
5. [Deadlock Prevention](#5-deadlock-prevention)
6. [Deadlock Avoidance — Banker's Algorithm](#6-deadlock-avoidance--bankers-algorithm)
7. [Deadlock Detection & Recovery](#7-deadlock-detection--recovery)
8. [FAANG / MAANG Interview Questions](#8-faang--maang-interview-questions)

---

## 1. What is Deadlock?

> **Deadlock:** A set of processes is **permanently blocked** — each process is waiting for a resource held by another process in the set, and no process can ever proceed.

```
P1 holds R1, waits for R2
P2 holds R2, waits for R1
→ Neither can proceed — DEADLOCK
```

### Deadlock vs Related Problems:

| Concept | What happens |
|---|---|
| **Deadlock** | Processes are **blocked forever** (circular dependency) |
| **Starvation** | A process waits **indefinitely** but others can still run (unfair scheduling) |
| **Livelock** | Processes are **active but making no progress** (keep reacting to each other) |

---

## 2. Coffman's Four Necessary Conditions

Deadlock can occur **only if ALL four** conditions hold simultaneously. Removing any one prevents deadlock.

### Condition 1 — Mutual Exclusion
> At least one resource is **non-shareable** — only one process can use it at a time.

```
Example: A printer can only be used by one process at a time.
```

### Condition 2 — Hold and Wait
> A process **holds at least one resource** while waiting to acquire additional resources held by other processes.

```
P1 holds R1, requests R2 (held by P2) → P1 holds AND waits
```

### Condition 3 — No Preemption
> A resource can only be **released voluntarily** by the process holding it — the OS cannot forcibly take it away.

```
P1 holds a mutex lock → OS cannot force P1 to release it
```

### Condition 4 — Circular Wait
> A circular chain of processes exists where each process waits for a resource held by the **next process** in the chain.

```
P1 → waits for R2 (held by P2)
P2 → waits for R3 (held by P3)
P3 → waits for R1 (held by P1)
→ Circular Wait!
```

> **GATE Key Point:** These 4 conditions are **necessary but not sufficient** for deadlock. All 4 must hold for deadlock to occur. Eliminating even one prevents deadlock.

---

## 3. Resource Allocation Graph (RAG)

The RAG is a directed graph used to represent the state of resource allocation and **detect deadlock** visually.

### Graph Elements:

```
Vertices:
  P = {P1, P2, P3, ...}    → Processes (circles)
  R = {R1, R2, R3, ...}    → Resources (rectangles, dots = instances)

Edges:
  Pi → Rj   (Request edge)   — Pi is requesting an instance of Rj
  Rj → Pi   (Assignment edge) — An instance of Rj is assigned to Pi
```

### RAG Example:

```
P1 ──request──→ R1
R1 ──assigned──→ P2
P2 ──request──→ R2
R2 ──assigned──→ P1
         ↑  CYCLE → DEADLOCK (single instance per resource)
```

### Deadlock Rules for RAG:

| Situation | Deadlock? |
|---|---|
| **No cycle** in RAG | No deadlock |
| **Cycle exists**, each resource has **one instance** | Deadlock **guaranteed** |
| **Cycle exists**, resources have **multiple instances** | Deadlock **possible** (not guaranteed) |

### RAG with Multiple Instances — Example (No Deadlock):

```
R1 has 2 instances: assigned to P1 and P2
P3 requests R1 → P3 waits
But P1 or P2 will eventually release → P3 will proceed → No deadlock
```

---

## 4. Deadlock Handling Strategies

| Strategy | Approach |
|---|---|
| **Prevention** | Design the system so at least one Coffman condition can never hold |
| **Avoidance** | At runtime, grant requests only if the system stays in a "safe state" |
| **Detection + Recovery** | Allow deadlock to occur, detect it, then recover |
| **Ignore (Ostrich Algorithm)** | Assume deadlock is rare; let the user restart (used by most OSes including Windows/Linux) |

---

## 5. Deadlock Prevention

Eliminate at least one Coffman condition **at system design time**.

### Eliminate Mutual Exclusion
- Make resources **shareable** where possible (e.g., read-only files can be shared)
- **Not always possible** — some resources are inherently non-shareable (printers, mutex locks)

### Eliminate Hold and Wait

**Strategy 1 — Request all resources at once (at start):**
```
Process must declare all needed resources upfront.
If all are available → allocate all and proceed.
If not → process waits without holding anything.
```

**Strategy 2 — Release before requesting more:**
```
Before requesting a new resource, a process must release all currently held resources.
```

**Problems:**
- Low resource utilization (resources held idle)
- Starvation possible (process may never get all resources simultaneously)

### Eliminate No Preemption

**Strategy — Preempt resources forcibly:**
```
If P1 holds R1 and requests R2 (unavailable):
  → Preempt all resources from P1
  → Add them to P1's needed list
  → P1 restarts when all needed resources are available
```

- Works well for resources whose state can be **saved and restored** (CPU registers, memory pages)
- **Not suitable** for resources like printers or mutex locks

### Eliminate Circular Wait (Most Practical!)

**Strategy — Total ordering of resources:**
```
Assign a unique number to each resource type.
A process can only request resources in increasing order of their numbers.

R1=1, R2=2, R3=3, R4=4

Process P:
  acquire(R1) → OK
  acquire(R3) → OK (3 > 1)
  acquire(R2) → NOT ALLOWED (2 < 3)
```

**Why it prevents circular wait:**
- If all processes request resources in the same order → a circular chain is impossible

> **GATE Point:** Ordering resources (eliminating circular wait) is the **most commonly used prevention technique** in practice.

---

## 6. Deadlock Avoidance — Banker's Algorithm

Rather than preventing deadlock at design time, **avoidance** dynamically checks whether granting a request keeps the system in a **safe state**.

### Key Concepts:

**Safe State:**
> A state where there exists at least one **safe sequence** — an ordering of all processes such that each process can complete using currently available resources plus resources released by processes that finish before it.

```
Safe state → No deadlock (deadlock is possible to avoid)
Unsafe state → Deadlock MAY occur (not guaranteed)
```

**Safe Sequence:** P1, P2, P3, ..., Pn is safe if:
- Resources needed by P1 ≤ Available resources → P1 finishes, releases its resources
- Resources needed by P2 ≤ Available + what P1 released → P2 finishes
- ... and so on for all processes

```
Safe state ⊃ Unsafe state ⊃ Deadlock
(every safe state is fine; not every unsafe state leads to deadlock, but avoidance treats unsafe = danger)
```

### Banker's Algorithm — Data Structures:

```
n = number of processes
m = number of resource types

Available[m]        — number of available instances of each resource type
Max[n][m]           — maximum demand of each process for each resource type
Allocation[n][m]    — resources currently allocated to each process
Need[n][m]          — remaining resource need of each process

Need[i][j] = Max[i][j] - Allocation[i][j]
```

### Safety Algorithm (Check if current state is safe):

```
Step 1: Let Work = Available (copy)
        Let Finish[i] = false for all i

Step 2: Find index i such that:
          Finish[i] == false  AND  Need[i] ≤ Work
        If no such i exists → go to Step 4

Step 3: Work = Work + Allocation[i]  (process i finishes, releases its resources)
        Finish[i] = true
        Go to Step 2

Step 4: If Finish[i] == true for all i → system is in SAFE STATE
        Otherwise → system is in UNSAFE STATE
```

### Resource Request Algorithm (Should we grant a request?):

```
Process Pi requests Request[i] units of resources.

Step 1: If Request[i] ≤ Need[i] → continue
        Else → error (process exceeded its maximum claim)

Step 2: If Request[i] ≤ Available → continue
        Else → Pi must wait (resources not available)

Step 3: Pretend to allocate (temporarily):
          Available   = Available   - Request[i]
          Allocation[i] = Allocation[i] + Request[i]
          Need[i]     = Need[i]     - Request[i]

Step 4: Run Safety Algorithm on the new state
        If SAFE   → grant the request (make allocation permanent)
        If UNSAFE → undo the pretend allocation, Pi must wait
```

### Banker's Algorithm — Full Example:

**Setup:** 5 processes (P0–P4), 3 resource types (A, B, C)

```
           Allocation    Max       Available
           A  B  C      A  B  C   A  B  C
P0         0  1  0      7  5  3   3  3  2
P1         2  0  0      3  2  2
P2         3  0  2      9  0  2
P3         2  1  1      2  2  2
P4         0  0  2      4  3  3
```

**Calculate Need = Max − Allocation:**

```
           Need
           A  B  C
P0         7  4  3
P1         1  2  2
P2         6  0  0
P3         0  1  1
P4         4  3  1
```

**Safety Check (Available = [3, 3, 2]):**

```
Step 1: Work = [3, 3, 2], Finish = [F, F, F, F, F]

  Find i where Need[i] ≤ Work:
    P0: Need=[7,4,3] ≤ [3,3,2]? NO
    P1: Need=[1,2,2] ≤ [3,3,2]? YES → run P1
      Work = [3,3,2] + [2,0,0] = [5,3,2], Finish[1] = T

  Find next i:
    P0: Need=[7,4,3] ≤ [5,3,2]? NO
    P2: Need=[6,0,0] ≤ [5,3,2]? NO
    P3: Need=[0,1,1] ≤ [5,3,2]? YES → run P3
      Work = [5,3,2] + [2,1,1] = [7,4,3], Finish[3] = T

  Find next i:
    P0: Need=[7,4,3] ≤ [7,4,3]? YES → run P0
      Work = [7,4,3] + [0,1,0] = [7,5,3], Finish[0] = T

    P2: Need=[6,0,0] ≤ [7,5,3]? YES → run P2
      Work = [7,5,3] + [3,0,2] = [10,5,5], Finish[2] = T

    P4: Need=[4,3,1] ≤ [10,5,5]? YES → run P4
      Finish[4] = T

All Finish = T → SAFE STATE
Safe sequence: P1 → P3 → P0 → P2 → P4
```

### Limitations of Banker's Algorithm:

| Limitation | Why it's a problem |
|---|---|
| Must know **maximum resource needs** in advance | Processes often don't know this |
| Number of processes and resources must be **fixed** | Dynamic systems don't fit |
| Low resource utilization | Overly conservative — refuses safe-but-usable states |
| **High overhead** per request | Safety check runs every time a resource is requested |
| Not practical for **general-purpose OS** | Used in specialized/embedded systems |

---

## 7. Deadlock Detection & Recovery

Allow deadlock to occur, then detect and recover from it.

### Detection — Single Instance Resources (Wait-For Graph):

Reduce the RAG by removing resource nodes — only keep process nodes.

```
RAG:  P1 → R1 → P2 → R2 → P1
WFG:  P1 → P2 → P1  ← CYCLE = DEADLOCK
```

**Rule:** Deadlock exists if and only if the **Wait-For Graph contains a cycle**.

### Detection — Multiple Instance Resources:

Use an algorithm similar to the Safety Algorithm:

```
Data structures:
  Available[m]      — available instances per resource type
  Allocation[n][m]  — current allocations
  Request[n][m]     — current pending requests (not Need — actual current requests)

Algorithm:
Step 1: Work = Available
        For all i: if Allocation[i] = 0 → Finish[i] = true, else false

Step 2: Find i such that Finish[i] = false AND Request[i] ≤ Work
        If no such i → go to Step 4

Step 3: Work = Work + Allocation[i]
        Finish[i] = true
        Go to Step 2

Step 4: If Finish[i] = false for any i → Pi is DEADLOCKED
```

### When to Run Detection?

| Approach | Trade-off |
|---|---|
| Run **after every request** | Expensive; finds deadlock immediately |
| Run at **fixed intervals** (e.g., every hour) | Cheaper; may let deadlock linger |
| Run when **CPU utilization drops below threshold** | Practical heuristic |

### Deadlock Recovery Methods:

#### Method 1 — Process Termination

**Option A — Kill all deadlocked processes:**
- Guaranteed to break deadlock
- Expensive — all work lost

**Option B — Kill processes one at a time (iterative):**
- Kill one, re-run detection algorithm, repeat if deadlock persists
- Expensive (multiple detections), but less work lost

**Criteria for choosing which process to kill:**
- Process priority (kill lowest-priority first)
- How long the process has computed and how much is left
- Resources used (kill the one holding the most)
- Resources needed to complete
- Number of processes that need to be terminated

#### Method 2 — Resource Preemption

Forcibly take resources from some processes and give them to others.

**Key issues:**
1. **Victim selection** — Which process to preempt? (minimize cost)
2. **Rollback** — Preempted process must be rolled back to a safe state (restart or checkpoint)
3. **Starvation** — Same process may always be chosen as victim → add a "number of rollbacks" factor to victim selection cost

---

## 8. FAANG / MAANG Interview Questions

> Questions asked at Google, Amazon, Meta, Microsoft, Apple, and other top companies.

---

### Deadlock Fundamentals

**Q1. What is deadlock? What are the four necessary conditions?**
> Deadlock = a state where a set of processes are **permanently blocked**, each waiting for a resource held by another. The four Coffman conditions (ALL must hold): (1) **Mutual Exclusion** — resource non-shareable; (2) **Hold & Wait** — process holds one resource while waiting for another; (3) **No Preemption** — resources released only voluntarily; (4) **Circular Wait** — P1 waits for P2, P2 waits for P3, ..., Pn waits for P1.

**Q2. How is deadlock different from livelock and starvation?**
> **Deadlock** — processes are **blocked, doing nothing**, waiting in a circular dependency forever. **Livelock** — processes are **actively running but making no progress** (e.g., two threads each detect the other is waiting and both back off repeatedly). **Starvation** — a process is ready to run but **never gets scheduled** — the system isn't stuck, just unfair. Fix for starvation: aging (gradually increase the priority of waiting processes).

**Q3. Can you have deadlock with a single resource and multiple processes?**
> No. Deadlock requires circular wait, which needs at least two resources and two processes (P1 holds R1 waits for R2, P2 holds R2 waits for R1). With a single resource, only one process can hold it — others wait but the holder will eventually release it (assuming no hold-and-wait).

---

### Resource Allocation Graph

**Q4. What is a Resource Allocation Graph? How do you detect deadlock from it?**
> A RAG is a directed graph with two types of nodes — processes (circles) and resources (rectangles, with dots for instances). Request edges go from process to resource; assignment edges go from resource to process. **Detection rule:** If all resources have single instances, deadlock exists iff the RAG has a **cycle**. With multiple instances, a cycle is necessary but not sufficient — you need the detection algorithm (similar to Banker's safety check using actual current requests, not max need).

**Q5. Draw a RAG with 3 processes and 2 resources that is deadlocked. Now draw one with a cycle but no deadlock.**
> **Deadlocked (single instance):**
> ```
> P1 → R1 → P2 → R2 → P1   (cycle, 1 instance each → deadlock)
> ```
> **Cycle but no deadlock (2 instances of R1):**
> ```
> P1 → R1 → P2    P3 → R1
> R1 has 2 instances: one assigned to P2, one assigned to P3
> P1 waits for R1, but P2 or P3 will eventually finish → no deadlock
> ```

---

### Prevention vs Avoidance vs Detection

**Q6. What is the difference between deadlock prevention, avoidance, and detection?**
> **Prevention** — eliminate at least one Coffman condition at **design time** (e.g., impose lock ordering to break circular wait). Simplest but most restrictive — low resource utilization. **Avoidance** — at **runtime**, only grant requests that keep the system in a safe state (Banker's Algorithm). More flexible but requires knowing max needs in advance. **Detection + Recovery** — don't restrict at all; let deadlock occur, detect it periodically, then recover (kill a process or preempt resources). Most flexible but recovery is costly.

**Q7. How does eliminating circular wait prevent deadlock? Give a code example.**
> Assign each lock type a unique number. Require that threads **always acquire locks in increasing numeric order**.
> ```c
> // Lock IDs: mutex_A = 1, mutex_B = 2
> 
> // Thread 1:                 Thread 2 (WRONG - causes deadlock):
> lock(mutex_A);               lock(mutex_B);
> lock(mutex_B);               lock(mutex_A);
> 
> // Thread 2 (CORRECT - same order):
> lock(mutex_A);  // acquire lower ID first
> lock(mutex_B);
> ```
> With consistent ordering, no thread can form a circular wait — you'd need T1 waiting on something T2 holds while T2 waits on something T1 holds, which the ordering rule prevents.

---

### Banker's Algorithm Deep Dive

**Q8. Explain the Banker's Algorithm. What does "safe state" mean?**
> The Banker's Algorithm is a **deadlock avoidance** algorithm. Before granting a resource request, it simulates the allocation and checks if the resulting state is **safe**. A safe state is one where there exists at least one **safe sequence** — an order in which every process can complete by using currently available resources + what previously completed processes release. If the state after granting would be safe → grant it; otherwise → make the process wait.

**Q9. What is the time complexity of the Banker's Algorithm?**
> The **Safety Algorithm** runs in **O(n² × m)** time — for each of n processes (outer), we scan n processes to find one with Need ≤ Work (inner), and compare m resource types. The **Resource Request Algorithm** runs the Safety Algorithm once per request, so it's also O(n² × m) per request. For a system with many processes and frequent requests, this overhead is significant.

**Q10. Why is the Banker's Algorithm not used in general-purpose operating systems?**
> Three main reasons: (1) **Maximum resource needs must be declared in advance** — most processes don't know this (e.g., a user's browser doesn't know how many files it'll open). (2) **Number of processes and resources must be fixed** — real systems have dynamic process creation. (3) **Low resource utilization** — the algorithm refuses requests that might be safe, to stay conservative. Linux and Windows use the "Ostrich Algorithm" — ignore deadlocks and let users deal with them (deadlocks are rare enough that detection overhead isn't worth it for general use).

**Q11. Given a state, determine if it's safe and find the safe sequence.**
> **Steps:** (1) Calculate Need = Max − Allocation. (2) Start with Work = Available, all Finish[i] = false. (3) Repeatedly find a process where Finish[i] = false AND Need[i] ≤ Work → add it to sequence, set Work += Allocation[i], Finish[i] = true. (4) If all Finish[i] = true → safe (output the sequence). If you get stuck with some Finish[i] = false → unsafe state.

---

### Detection and Recovery

**Q12. How would you detect a deadlock in a production system?**
> (1) **Timeout-based:** If a thread holds a lock for > N seconds, log an alert — use `jstack` (Java) or `/proc/<pid>/stack` (Linux) to inspect. (2) **Thread sanitizers:** `ThreadSanitizer` (C++), Helgrind/Valgrind detect lock-order violations at runtime before deadlock occurs. (3) **Wait-for graph monitoring:** Periodically build a wait-for graph from lock acquisition data and check for cycles. (4) **Watchdog threads:** Critical threads send heartbeats; if a heartbeat stops, the watchdog triggers an alert.

**Q13. What is the priority inversion problem? How does priority inheritance solve it?**
> **Priority Inversion:** A high-priority task (H) is blocked waiting for a resource held by a low-priority task (L). A medium-priority task (M) preempts L (since M > L) → L can't release the resource → H is stuck waiting behind a lower-priority process. **Example:** NASA Mars Pathfinder (1997) — a high-priority meteorological thread was starved by a medium-priority communications thread, because a low-priority data bus thread held a shared semaphore. Fixed by enabling **Priority Inheritance** in VxWorks: the mutex holder (L) temporarily inherits the priority of the highest-priority waiter (H) until it releases the resource.

**Q14. How do you recover from a deadlock?**
> Two approaches: (1) **Process Termination** — either kill all deadlocked processes at once (simple but costly) or kill them one by one with detection re-run after each kill (less wasteful but slow). Choose victim by priority, resource usage, computation time remaining. (2) **Resource Preemption** — forcibly take resources from victim processes and give them to blocked ones. Issues: must save/restore state (rollback to checkpoint), and must prevent the same process from always being chosen (starvation — fix by counting number of times preempted).

---

### System Design / Scenario Questions

**Q15. How does a database handle deadlocks?**
> Databases (MySQL InnoDB, PostgreSQL) use **deadlock detection** continuously. They maintain a lock wait graph and check for cycles after every lock request. When a cycle is detected, the database picks a **victim transaction** (usually the cheapest to abort — based on rows modified, locks held) and rolls it back, releasing its locks. The victim transaction gets an error and the application can retry. This is why database code always wraps transactions in retry loops.

**Q16. How does the Linux kernel avoid deadlocks in its locking code?**
> Linux uses several strategies: (1) **Lock ordering** — all kernel subsystems must acquire locks in a defined order (enforced by the `lockdep` validator at runtime in debug builds). (2) **`lockdep` (Lock Dependency Engine)** — detects potential deadlocks at development time by tracking lock acquisition order and flagging circular dependencies. (3) **RCU (Read-Copy-Update)** — a lock-free synchronization mechanism for read-heavy data structures that eliminates reader locks entirely. (4) **Spinlocks with IRQ disabling** — avoids interrupt-handler vs process deadlocks by disabling interrupts while holding certain locks.

**Q17. You have a microservices system where Service A calls Service B and Service B calls Service A simultaneously. How do you handle this distributed deadlock?**
> This is a **distributed deadlock** — the circular dependency spans network calls. Solutions: (1) **Timeout + Retry with backoff** — each service call has a deadline; if exceeded, return an error and the caller retries with exponential backoff + jitter. (2) **Saga pattern** — break the operation into steps with compensating transactions; if a step fails (timeout), run the compensating action to undo previous steps. (3) **Async communication** — replace synchronous REST calls with message queues (Kafka, SQS); A publishes a message, B consumes it and publishes a response — no blocking call chain. (4) **Circuit breaker** — if Service B is unresponsive, the circuit breaker trips and Service A gets an immediate failure instead of blocking.

---

### Quick Reference

**Coffman Conditions (all 4 must hold for deadlock):**

```
1. Mutual Exclusion    — resource is non-shareable
2. Hold & Wait         — holds one, waits for another
3. No Preemption       — resources released only voluntarily
4. Circular Wait       — P1→P2→...→Pn→P1
```

**Banker's Algorithm Variables:**

```
Available[m]      — free instances per resource
Max[n][m]         — max demand per process
Allocation[n][m]  — currently allocated
Need[n][m]        — Max - Allocation (remaining need)
```

**RAG Deadlock Rules:**

```
No cycle              → No deadlock
Cycle + single instance resources → Deadlock guaranteed
Cycle + multi instance resources  → Deadlock possible (run detection)
```

**Deadlock Strategies Summary:**

| Strategy | When | Trade-off |
|---|---|---|
| **Prevention** | Design time | Restrictive, low utilization |
| **Avoidance (Banker's)** | Runtime, per request | Needs max-need info, overhead |
| **Detection + Recovery** | Periodic or on-demand | Most flexible, recovery is costly |
| **Ostrich (Ignore)** | Deployed OS | Practical for rare deadlocks |

---
