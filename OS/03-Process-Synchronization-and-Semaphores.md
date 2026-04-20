# OS 03 — Process Synchronization & Semaphores

> GATE 2025 Crash Course | CS & IT

---

## Table of Contents

1. [Why Synchronization?](#1-why-synchronization)
2. [Race Condition](#2-race-condition)
3. [Critical Section Problem](#3-critical-section-problem)
4. [Requirements for a Valid Solution](#4-requirements-for-a-valid-solution)
5. [Software Solutions](#5-software-solutions)
6. [Hardware Solutions](#6-hardware-solutions)
7. [Mutex Locks (Spinlock)](#7-mutex-locks-spinlock)
8. [Semaphores](#8-semaphores)
9. [Classical Synchronization Problems](#9-classical-synchronization-problems)
10. [Monitors](#10-monitors)
11. [FAANG / MAANG Interview Questions](#11-faang--maang-interview-questions)

---

## 1. Why Synchronization?

When multiple processes run **concurrently** and share data (shared memory, files, variables), their execution can interleave in unpredictable ways.

```
Without sync:
  P1 reads x=5, P2 reads x=5
  P1 writes x=6, P2 writes x=6
  Final x = 6 — but should be 7 if both incremented!
```

**Process Synchronization** = set of techniques to ensure **correct and consistent** access to shared resources among concurrent processes.

---

## 2. Race Condition

> **Race Condition:** When the final result of concurrent execution **depends on the order/timing** of process scheduling — leading to incorrect or unpredictable output.

### Classic Example — Bank Balance:

```
Shared variable: balance = 1000

Process P1 (Deposit 500):      Process P2 (Withdraw 200):
  temp1 = balance  (= 1000)      temp2 = balance  (= 1000)
  temp1 = temp1 + 500            temp2 = temp2 - 200
  balance = temp1  (= 1500)      balance = temp2  (= 800)

Final balance = 800 ❌ (should be 1300!)
```

The problem: both processes read the old value of `balance` before either writes back.

### Why It Happens:
- `balance = balance + 500` is **not atomic** — it's 3 machine instructions (read, add, write)
- A context switch can happen between any two instructions

> **Fix:** Ensure that reading and updating shared data happens as an **atomic (indivisible) operation**.

---

## 3. Critical Section Problem

The **Critical Section (CS)** is the part of code that accesses shared resources (variables, files, hardware).

### Process Structure:

```
do {
    /* --- ENTRY SECTION ---  (request permission to enter CS) */

        CRITICAL SECTION
        (access shared resource here)

    /* --- EXIT SECTION ---   (signal that CS is done) */

        REMAINDER SECTION
        (rest of the process)

} while (true);
```

| Section | Purpose |
|---|---|
| **Entry Section** | Code that requests permission to enter CS |
| **Critical Section** | Code that accesses shared resources |
| **Exit Section** | Code that signals CS is released |
| **Remainder Section** | All other code (no shared resource access) |

---

## 4. Requirements for a Valid Solution

Any correct solution to the Critical Section problem **must satisfy all 3 conditions**:

### 1. Mutual Exclusion
> Only **one process** can be in the critical section at a time.

```
If P_i is in CS → no other P_j can be in CS simultaneously.
```

### 2. Progress
> If no process is in CS and some processes want to enter, **a decision must be made in finite time** — the selection cannot be postponed indefinitely.

```
Processes in the remainder section do NOT participate in this decision.
The system must NOT deadlock at the entry section.
```

### 3. Bounded Waiting
> There is a **limit on the number of times** other processes can enter CS after a process has requested entry and before that request is granted — **no starvation**.

```
If P_i is waiting, P_j can enter CS at most N-1 times before P_i gets its turn.
```

> **GATE Point:** A solution that satisfies only Mutual Exclusion but not Progress = **deadlock**.  
> A solution that satisfies only Mutual Exclusion but not Bounded Waiting = **starvation**.

---

## 5. Software Solutions

### 5.1 Peterson's Solution (2-Process Software Solution)

Uses two shared variables:
- `flag[2]` — `flag[i] = true` means process i **wants** to enter CS
- `turn` — indicates whose **turn** it is to enter CS

```c
// Shared variables
boolean flag[2] = {false, false};
int turn;

// Code for Process P_i (the other process is P_j)
do {
    flag[i] = true;        // "I want to enter CS"
    turn = j;              // "You go first if you want"

    while (flag[j] && turn == j);  // Wait if Pj wants AND it's Pj's turn

    /* CRITICAL SECTION */

    flag[i] = false;       // "I'm done with CS"

    /* REMAINDER SECTION */

} while (true);
```

### Why Peterson's Works:

| Condition | How it's satisfied |
|---|---|
| **Mutual Exclusion** | `turn` can only be `i` or `j`, never both → only one enters |
| **Progress** | If Pj doesn't want CS (`flag[j]=false`), Pi enters immediately |
| **Bounded Waiting** | After Pi sets `turn=j`, Pj gets at most ONE more chance before Pi enters |

### Limitations:
- Works for **only 2 processes** (can be generalized but complex)
- May **not work on modern hardware** without memory barriers (due to instruction reordering)
- Software-only — no hardware support needed but slower

---

## 6. Hardware Solutions

Hardware provides **atomic instructions** that read and write in a single uninterruptible step.

### 6.1 Test-and-Set (TAS)

```c
// Atomic hardware instruction
boolean TestAndSet(boolean *lock) {
    boolean rv = *lock;   // Read current value
    *lock = true;         // Set lock to true
    return rv;            // Return old value
}

// Usage:
boolean lock = false;     // Shared lock variable

do {
    while (TestAndSet(&lock));  // Busy wait until lock = false

    /* CRITICAL SECTION */

    lock = false;          // Release lock

    /* REMAINDER SECTION */
} while (true);
```

**How it works:**
- If `lock = false` → TAS returns `false` (exits while loop) AND sets lock to `true` → process enters CS
- If `lock = true` → TAS returns `true` (stays in while loop) → process waits (busy waits)

| Property | Satisfied? |
|---|---|
| Mutual Exclusion | ✅ Yes |
| Progress | ✅ Yes |
| Bounded Waiting | ❌ **No** — a process may starve |

### 6.2 Compare-and-Swap (CAS)

```c
// Atomic hardware instruction
int CompareAndSwap(int *value, int expected, int new_value) {
    int temp = *value;
    if (*value == expected)
        *value = new_value;
    return temp;
}

// Usage:
int lock = 0;   // 0 = free, 1 = locked

do {
    while (CompareAndSwap(&lock, 0, 1) != 0);  // Wait until lock = 0, then set to 1

    /* CRITICAL SECTION */

    lock = 0;  // Release

    /* REMAINDER SECTION */
} while (true);
```

> **GATE Point:** TAS and CAS both ensure mutual exclusion but **do NOT guarantee bounded waiting** by default. A modified version with a waiting array can add bounded waiting.

---

## 7. Mutex Locks (Spinlock)

A **mutex lock** (mutual exclusion lock) is a high-level software tool built using atomic hardware instructions.

```c
// Two operations:
acquire() {
    while (!available);   // Busy wait (spin)
    available = false;
}

release() {
    available = true;
}

// Usage:
do {
    acquire();
    /* CRITICAL SECTION */
    release();
    /* REMAINDER SECTION */
} while (true);
```

### Busy Waiting Problem:
- Process keeps **looping** (spinning) on the lock → **wastes CPU cycles**
- Called a **spinlock** — acceptable only when CS is very short
- On a single-CPU system, busy waiting is especially wasteful

> **Busy Waiting Fix → Semaphores with blocking** (process sleeps instead of spinning)

---

## 8. Semaphores

A **semaphore** is an integer variable accessed only through two **atomic operations**: `wait()` and `signal()`.

```
Semaphore S;     // Integer variable
```

### Operations:

```c
wait(S):           // Also called P(S), down(S)
    S = S - 1;
    if (S < 0) {
        block this process;   // Add to semaphore's waiting queue
    }

signal(S):         // Also called V(S), up(S)
    S = S + 1;
    if (S <= 0) {
        wake up one waiting process;  // Remove from queue, move to Ready
    }
```

> **GATE Point:** `wait` and `signal` must be **atomic** — no two processes execute them simultaneously on the same semaphore.

### 8.1 Binary Semaphore (Mutex Semaphore)

- Value restricted to **0 or 1** only
- Used for **mutual exclusion** (one process in CS at a time)
- Behaves exactly like a mutex lock

```
Initial value: S = 1

Process uses CS:
    wait(S)    → S becomes 0 (CS locked)
    [critical section]
    signal(S)  → S becomes 1 (CS unlocked)
```

### 8.2 Counting Semaphore

- Value can be **any non-negative integer**
- Used to control access to a resource with **multiple instances**
- Initial value = number of available resource instances

```
Example: 3 printers available
Initial: S = 3

Process acquires printer:   wait(S)   → S = 2
Another process:            wait(S)   → S = 1
Another:                    wait(S)   → S = 0
Next process:               wait(S)   → S = -1 (blocked, no printer)

Process frees printer:      signal(S) → S = 0 (wakes up blocked process)
```

### Semaphore Value Interpretation:

| S value | Meaning |
|---|---|
| S > 0 | Number of **available** resources |
| S = 0 | No resources available; no process waiting |
| S < 0 | \|S\| = number of **blocked/waiting** processes |

### Semaphore vs Mutex:

| Feature | Mutex | Binary Semaphore |
|---|---|---|
| Ownership | Only owner can unlock | Any process can signal |
| Used for | Mutual exclusion | Mutual exclusion + signaling |
| Value | 0 or 1 | 0 or 1 |

### Semaphore for Ordering / Synchronization:

Semaphores can enforce **execution order** between processes:

```c
// Ensure S2 in P2 executes AFTER S1 in P1
Semaphore synch = 0;

Process P1:              Process P2:
    S1;                      wait(synch);
    signal(synch);           S2;
```

---

## 9. Classical Synchronization Problems

### 9.1 Producer-Consumer Problem (Bounded Buffer)

**Scenario:** Producer creates items and puts them in a buffer. Consumer removes items from the buffer. Buffer has finite size N.

**Constraints:**
- Producer must wait if buffer is **full**
- Consumer must wait if buffer is **empty**
- Only one process accesses buffer at a time (mutual exclusion)

```c
Semaphore mutex = 1;    // Mutual exclusion for buffer access
Semaphore empty = N;    // Counts empty slots (initially N)
Semaphore full  = 0;    // Counts full slots (initially 0)

// PRODUCER:
do {
    produce item;
    wait(empty);        // Wait if no empty slot
    wait(mutex);        // Lock buffer
    add item to buffer;
    signal(mutex);      // Unlock buffer
    signal(full);       // One more full slot
} while (true);

// CONSUMER:
do {
    wait(full);         // Wait if no item to consume
    wait(mutex);        // Lock buffer
    remove item from buffer;
    signal(mutex);      // Unlock buffer
    signal(empty);      // One more empty slot
    consume item;
} while (true);
```

> **GATE Point:** Order of `wait()` calls matters! `wait(mutex)` must come **after** `wait(empty)` / `wait(full)` — reversing causes **deadlock**.

---

### 9.2 Readers-Writers Problem

**Scenario:** Multiple readers can read simultaneously (no conflict). A writer needs **exclusive access** — no reader or writer can access while writing.

**Two variants:**
- **Reader-priority:** Readers are never kept waiting unless a writer holds access → writers may starve
- **Writer-priority:** Once a writer is waiting, no new readers allowed → readers may starve

```c
Semaphore mutex     = 1;   // Protects read_count
Semaphore writeBlock = 1;  // Blocks writers (and first reader)
int read_count       = 0;  // Number of active readers

// READER:
do {
    wait(mutex);
    read_count++;
    if (read_count == 1)
        wait(writeBlock);   // First reader blocks writers
    signal(mutex);

    /* READ DATA */

    wait(mutex);
    read_count--;
    if (read_count == 0)
        signal(writeBlock); // Last reader unblocks writers
    signal(mutex);
} while (true);

// WRITER:
do {
    wait(writeBlock);       // Exclusive access
    /* WRITE DATA */
    signal(writeBlock);
} while (true);
```

> **GATE Point:** Multiple readers can be in CS simultaneously. Writers always need exclusive access.

---

### 9.3 Dining Philosophers Problem

**Scenario:** 5 philosophers sit around a table. Between each pair is one fork (5 forks total). A philosopher needs **both left and right fork** to eat. After eating, they think and put forks down.

```
         P0
      F4    F0
   P4          P1
      F3    F1
         P3
          F2
         P2
```

**Naive Solution (causes deadlock):**

```c
Semaphore fork[5] = {1,1,1,1,1};  // One semaphore per fork

Philosopher i:
do {
    wait(fork[i]);           // Pick left fork
    wait(fork[(i+1) % 5]);  // Pick right fork
    EAT;
    signal(fork[i]);
    signal(fork[(i+1) % 5]);
    THINK;
} while (true);
```

**Problem:** If all 5 philosophers pick their left fork simultaneously → **deadlock** (each waits for right fork held by the neighbor).

**Solutions to Deadlock:**

| Solution | How |
|---|---|
| **Allow only 4 philosophers** to sit | Add `room` semaphore initialized to 4 |
| **Asymmetric solution** | Even philosophers: left then right; Odd philosophers: right then left |
| **Pick up both forks atomically** | Use a mutex to check both forks together |

```c
// Solution 1: Room semaphore (max 4 seated)
Semaphore room = 4;

Philosopher i:
    wait(room);
    wait(fork[i]);
    wait(fork[(i+1)%5]);
    EAT;
    signal(fork[(i+1)%5]);
    signal(fork[i]);
    signal(room);
```

---

## 10. Monitors

A **Monitor** is a high-level synchronization construct — a programming language feature that automatically enforces mutual exclusion.

```
Monitor MonitorName {
    // Shared variables
    condition x, y;   // Condition variables

    procedure P1 (...) { ... }
    procedure P2 (...) { ... }

    initialization code (...) { ... }
}
```

### Key Properties:
- Only **one process** can be active inside a monitor at a time (automatic mutual exclusion)
- No need to explicitly call `wait(mutex)` / `signal(mutex)` — the compiler handles it
- Uses **condition variables** for additional synchronization

### Condition Variables:

```c
condition x;

x.wait()    // Suspends the calling process; releases monitor
x.signal()  // Resumes exactly ONE suspended process (if any)
            // If no process is waiting → signal has no effect
```

| Feature | Semaphore | Monitor |
|---|---|---|
| Level | Low-level | High-level |
| Mutual exclusion | Manual | Automatic |
| Ease of use | Error-prone | Safer |
| Language support | Any (OS-level) | Built-in (Java `synchronized`, etc.) |

> **GATE Point:** Monitors are safer than semaphores — less chance of programmer error. Java's `synchronized` keyword implements monitor behavior.

---

## 11. FAANG / MAANG Interview Questions

> Questions commonly asked at Google, Amazon, Microsoft, Meta, Apple, Flipkart and other top tech companies.

---

### Race Conditions & Mutual Exclusion

**Q1. What is a race condition? Give a code-level example. How do you fix it?**
> A race condition occurs when the result depends on the **timing/order** of concurrent operations.
> ```c
> // Thread 1 & Thread 2 both run: counter++
> // counter++ = READ counter → ADD 1 → WRITE counter (3 steps, non-atomic)
> // Both threads read 5 → both write 6 → result is 6, should be 7
> ```
> Fix: use a **mutex lock** around the read-modify-write, or use an **atomic operation** (`std::atomic<int>`, `AtomicInteger`).

**Q2. What is the difference between a mutex and a semaphore?**
> | | Mutex | Semaphore |
> |---|---|---|
> | Value | Binary (locked/unlocked) | Integer (0 to N) |
> | Ownership | Only the **locking thread** can unlock | **Any thread** can signal |
> | Use case | Mutual exclusion | Counting resources, signaling |
> | Extra | Has ownership semantics | No ownership |
>
> A mutex is a **lock with ownership**. A semaphore is a **signaling mechanism**. Using a semaphore for mutual exclusion is possible but loses the ownership guarantee.

**Q3. When would you use a spinlock instead of a mutex?**
> Use a spinlock when: (1) the **critical section is very short** (nanoseconds), (2) you're on a **multi-core system**, and (3) you can't afford the overhead of a context switch. On a single-core system, spinning is always wasteful — the current thread is burning CPU waiting for something that can only progress if it yields. In Linux kernel code, spinlocks are used extensively for short critical sections.

---

### Deadlock (Most Frequently Asked)

**Q4. What is deadlock? What are the 4 necessary conditions (Coffman conditions)?**
> Deadlock = a set of processes are **permanently blocked**, each waiting for a resource held by another.
> 4 conditions (ALL must hold simultaneously):
> 1. **Mutual Exclusion** — at least one resource is non-shareable
> 2. **Hold and Wait** — a process holds a resource while waiting for another
> 3. **No Preemption** — resources can only be released voluntarily
> 4. **Circular Wait** — P1 waits for P2, P2 waits for P3, ..., Pn waits for P1

**Q5. What is the difference between deadlock prevention, avoidance, and detection?**
> **Prevention** — eliminate at least one of the 4 Coffman conditions at design time (e.g., always acquire locks in a fixed order → eliminates circular wait). **Avoidance** — at runtime, decide whether to grant a resource request based on whether it keeps the system in a "safe state" (Banker's Algorithm). **Detection + Recovery** — allow deadlock, periodically check for it (resource allocation graph), then recover by preempting or killing a process.

**Q6. What is the Banker's Algorithm? What is its limitation?**
> The Banker's Algorithm checks if granting a resource request leaves the system in a **safe state** — a state where all processes can eventually complete even in worst-case scenarios. It simulates granting the request and checks if a safe sequence exists. Limitation: requires knowing **maximum resource needs in advance** — impractical for general-purpose OS.

**Q7. How would you detect a deadlock in a production system?**
> Practical approaches: (1) **Timeout-based detection** — if a thread holds a lock for more than N seconds, log a warning (jstack in Java, `/proc/<pid>/stack` in Linux). (2) **Lock ordering analysis** — tools like ThreadSanitizer, Helgrind detect lock-order violations at runtime. (3) **Resource allocation graph** — check for cycles in the wait-for graph. (4) **Watchdog threads** — monitor heartbeats from critical threads.

---

### Priority Inversion & Livelock

**Q8. What is priority inversion? Describe the NASA Mars Pathfinder incident.**
> In 1997, the Mars Pathfinder kept resetting. Root cause: a **high-priority meteorological thread** was starved because a **low-priority data bus thread** held a shared mutex, and a **medium-priority communications thread** kept preempting the low-priority thread. The high-priority thread could never get the mutex. Fixed by enabling **Priority Inheritance** in VxWorks — the mutex holder temporarily inherits the waiter's priority.

**Q9. What is a livelock? How is it different from deadlock?**
> In a **deadlock**, all involved processes are blocked and doing nothing. In a **livelock**, processes are **actively running** but making no progress — they keep responding to each other's actions and changing state without doing useful work. Example: two people in a corridor, each steps the same direction to let the other pass, both keep mirroring each other indefinitely.

**Q10. What is starvation? How is it different from deadlock?**
> **Starvation** — a process waits indefinitely because other processes always get priority (the system is not stuck, just unfair). **Deadlock** — a circular dependency where no process can ever proceed (the system IS stuck). A starved process *could* run if others stop arriving; a deadlocked process cannot run regardless.

---

### Semaphores & Synchronization Primitives

**Q11. What is spurious wakeup? How do you handle it?**
> A thread waiting on a condition variable can wake up even when the condition hasn't been signaled — this is a spurious wakeup (allowed by POSIX for implementation efficiency). **Always use a `while` loop, never `if`, when checking the condition:**
> ```c
> // WRONG:
> if (!buffer_full) wait(cv, mutex);
>
> // CORRECT:
> while (!buffer_full) wait(cv, mutex);  // re-check after every wakeup
> ```

**Q12. What is the difference between a condition variable and a semaphore?**
> **Semaphore** — remembers signals (if `signal()` fires before `wait()`, the count is incremented — the next `wait()` won't block). **Condition variable** — signal is **lost if nobody is waiting** (no memory). Condition variables are always used with a mutex and a predicate; semaphores are standalone. CV is better for state-based synchronization; semaphore for resource counting.

**Q13. What is a read-write lock? When would you use it over a regular mutex?**
> A read-write lock allows **multiple concurrent readers** OR **one exclusive writer** — never both. Use it when reads are far more frequent than writes (e.g., caches, configuration data, DNS lookup tables). With a regular mutex, all readers would serialize unnecessarily. Trade-off: if writers are frequent, writer starvation can occur with reader-preference implementations.

---

### Classical Problems (Coding/Design Interviews)

**Q14. Explain the Producer-Consumer problem. How would you implement it in Java?**
> Producers add items to a bounded buffer; consumers remove them. Buffer must handle full (producer waits) and empty (consumer waits) cases with thread safety.
> ```java
> // Java: Using wait/notify
> synchronized void produce(T item) throws InterruptedException {
>     while (buffer.size() == MAX) wait();   // wait if full
>     buffer.add(item);
>     notifyAll();
> }
> synchronized T consume() throws InterruptedException {
>     while (buffer.isEmpty()) wait();       // wait if empty
>     T item = buffer.remove(0);
>     notifyAll();
>     return item;
> }
> ```

**Q15. Explain the Dining Philosophers problem. Why does the naive solution deadlock? How do you fix it?**
> 5 philosophers, 5 forks, each needs 2. Naive: everyone picks left fork → all wait for right → circular wait → deadlock. Fixes: (1) **Allow only 4 philosophers** to sit (add a room semaphore = 4). (2) **Asymmetric:** even philosophers pick left then right, odd pick right then left — breaks circular wait. (3) **Atomic pickup:** grab both forks or none using a mutex.

**Q16. How would you implement a thread-safe singleton in Java?**
> ```java
> // Double-checked locking with volatile
> public class Singleton {
>     private static volatile Singleton instance;
>     private Singleton() {}
>     public static Singleton getInstance() {
>         if (instance == null) {
>             synchronized (Singleton.class) {
>                 if (instance == null)          // second check inside lock
>                     instance = new Singleton();
>             }
>         }
>         return instance;
>     }
> }
> // volatile ensures the instance write is visible across threads immediately
> ```

---

### Semaphore Quick Reference:

```
wait(S)  / P(S):    S--; if S < 0 → block process
signal(S)/ V(S):    S++; if S ≤ 0 → wake one waiting process

Binary Semaphore:   S ∈ {0, 1}       → mutual exclusion
Counting Semaphore: S ∈ {0..N}       → resource pool of N instances
S < 0 → |S| processes are currently waiting
```

---

### Classical Problems Summary:

| Problem | Key Primitives | Common Interview Trap |
|---|---|---|
| **Producer-Consumer** | `mutex=1`, `empty=N`, `full=0` | Reversing wait order → deadlock |
| **Readers-Writers** | `mutex=1`, `writeBlock=1`, `read_count` | Writer starvation in reader-priority |
| **Dining Philosophers** | `fork[5]={1}`, `room=4` | Naive solution always deadlocks |

---
