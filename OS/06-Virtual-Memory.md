# OS 06 — Virtual Memory

> GATE 2025 Crash Course | Lecture 06

---

## Table of Contents

1. [What is Virtual Memory?](#1-what-is-virtual-memory)
2. [Demand Paging](#2-demand-paging)
3. [Page Fault Handling (Deep Dive)](#3-page-fault-handling-deep-dive)
4. [Effective Access Time (EAT)](#4-effective-access-time-eat)
5. [Copy-on-Write (COW)](#5-copy-on-write-cow)
6. [Page Replacement Algorithms](#6-page-replacement-algorithms)
7. [Frame Allocation](#7-frame-allocation)
8. [Thrashing](#8-thrashing)
9. [Working Set Model](#9-working-set-model)
10. [Page Fault Frequency (PFF)](#10-page-fault-frequency-pff)
11. [Memory-Mapped Files](#11-memory-mapped-files)
12. [Kernel Memory Allocation](#12-kernel-memory-allocation)
13. [Other Virtual Memory Considerations](#13-other-virtual-memory-considerations)
14. [GATE PYQs & Key Formulas](#14-gate-pyqs--key-formulas)
15. [FAANG / MAANG Interview Questions](#15-faang--maang-interview-questions)

---

## 1. What is Virtual Memory?

> **Virtual Memory:** A technique that allows execution of processes that are **not completely in memory**. The logical address space can be larger than physical memory.

### Key Idea:

```
Physical Memory (RAM):  limited — e.g., 8 GB
Virtual Address Space:  per process — e.g., 4 GB (32-bit) or 128 TB (64-bit)

→ A process sees a large, contiguous address space
→ Only the parts it currently needs are loaded into RAM
→ Rest lives on disk (swap space)
```

### Benefits:

| Benefit | Explanation |
|---|---|
| **Programs larger than RAM** | Only active portions need to be in memory |
| **More processes in memory** | Each uses less physical RAM at any moment |
| **Efficient process creation** | fork() and exec() are fast via COW |
| **Shared memory** | Multiple processes can map the same physical pages |

### Virtual vs Physical Address Space:

```
Process A:  Virtual 0x0000 → 0xFFFF FFFF  (4 GB)
Process B:  Virtual 0x0000 → 0xFFFF FFFF  (4 GB)
            ↓ MMU / Page Table
Physical:   0x0000 → some subset of frames actually in RAM (e.g., 2 GB)
```

Each page in the virtual address space has a **valid bit** in the page table:
- `valid = 1` → page is currently in a physical frame
- `valid = 0` → page is on disk (or not allocated yet)

---

## 2. Demand Paging

> **Demand Paging:** Pages are loaded into memory **only when they are accessed** (demanded), not all at once when a process starts.

### Pure Demand Paging:

```
Process starts → NO pages loaded into memory initially
First instruction → PAGE FAULT → OS loads page 0
Second instruction → may or may not fault
...
Only accessed pages ever get loaded
```

### Advantages:
- Less I/O needed at start → faster process startup
- Less memory used → more processes can run
- No unnecessary pages loaded

### Locality of Reference:

Programs tend to access a **small subset** of their pages at any given time — the **working set** — due to:
- **Temporal locality** — recently accessed addresses likely accessed again soon (loops, variables)
- **Spatial locality** — addresses near recently accessed ones likely accessed soon (arrays, sequential code)

This is why demand paging works well in practice despite the overhead of page faults.

### Hardware Support Needed:

| Component | Role |
|---|---|
| **Page table with valid/invalid bit** | Marks which pages are in memory |
| **Secondary storage (swap)** | Holds pages not currently in RAM |
| **Restart instruction** | CPU must re-execute the faulting instruction after the page is loaded |

### Restart Instruction:

After a page fault, the **entire faulting instruction is restarted** from scratch. This is tricky for instructions that modify multiple memory locations (e.g., block move) — the CPU must either:
- Use a **microcode restart** mechanism, or
- Ensure all fault checks happen before any side effects

---

## 3. Page Fault Handling (Deep Dive)

### Step-by-Step:

```
1. CPU generates virtual address  →  MMU looks up page table
2. Page table entry: valid bit = 0  →  PAGE FAULT trap
3. CPU saves state (PC, registers) and switches to kernel mode
4. OS page fault handler runs:
   a. Check: is the address in the process's valid virtual address space?
      → NO: illegal access → send SIGSEGV → process killed
      → YES: page is just not in memory → proceed
   b. Find a free frame in physical memory
      → If no free frame: run PAGE REPLACEMENT algorithm to select victim
      → If victim page is dirty (modified): write it to swap first (double I/O)
   c. Schedule disk I/O to read the needed page from swap space
   d. Process blocked during disk I/O
5. I/O completes → OS updates page table (frame #, valid = 1)
6. OS puts process in ready queue
7. CPU resumes the faulting instruction from the beginning
```

### Page Table Entry Structure:

```
┌──────┬──────┬──────┬──────────┬───────────────┐
│Valid │Dirty │Ref   │Protection│  Frame Number  │
│  bit │  bit │  bit │   bits   │                │
└──────┴──────┴──────┴──────────┴───────────────┘

Valid (present) bit:  1 = in RAM,  0 = on disk
Dirty (modified) bit: 1 = page was written to (must be saved on eviction)
Reference bit:        1 = recently accessed (used by replacement algorithms)
Protection bits:      read / write / execute permissions
```

### Restart Instruction Problem:

Some architectures have instructions that span multiple pages or modify memory atomically. The OS must handle this carefully:
- e.g., `MOVS` (x86) copies a string — may fault mid-way
- Solution: check all potentially faulting addresses **before** beginning execution

---

## 4. Effective Access Time (EAT)

### Formula:

```
p = page fault rate  (0 ≤ p ≤ 1)
    p = 0 → no faults; p = 1 → every access faults

m = memory access time (e.g., 200 ns)
s = page fault service time (e.g., 8 ms = 8,000,000 ns)
    includes: OS overhead + disk I/O read + restart

EAT = (1 − p) × m  +  p × (s + m)
    ≈ (1 − p) × m  +  p × s       [since s >> m]
```

### Example:

```
m = 200 ns,  s = 8,000,000 ns (8 ms),  p = 0.001

EAT = (1 − 0.001) × 200 + 0.001 × 8,000,000
    = 0.999 × 200 + 0.001 × 8,000,000
    = 199.8 + 8000
    = 8199.8 ns  ≈  8.2 μs

Slowdown = EAT / m = 8199.8 / 200 ≈ 41×
```

> **Key Takeaway:** Even a page fault rate of 0.1% causes a 41× slowdown. Keeping `p` extremely small is critical.

### EAT with swap-out (dirty page):

```
If page replacement involves writing a dirty page first:
  s = (disk latency for write) + (disk latency for read) + OS overhead
  → s is roughly doubled
```

---

## 5. Copy-on-Write (COW)

> **COW:** After `fork()`, parent and child share the **same physical pages**, marked **read-only**. A private copy is made only when a page is **written to**.

### Without COW:

```
fork() → copy ALL parent pages → child has duplicate physical memory
→ Slow for large processes; wasteful if child immediately calls exec()
```

### With COW:

```
fork() called:
  Parent page table → Frame X (read-only)
  Child  page table → Frame X (read-only)   ← same physical frame!

Child writes to Frame X:
  MMU: write to read-only page → protection fault
  OS: allocate new frame Y, copy Frame X → Frame Y
  Child page table now → Frame Y (read-write)
  Parent still → Frame X (read-write, once no longer shared)
```

```
Memory on fork():
  Before write:  Parent[A] ──→ Frame 5 ←── Child[A]   (shared)
  After write:   Parent[A] ──→ Frame 5
                 Child[A]  ──→ Frame 7  (copy created)
```

### Benefits:
- `fork()` is O(1) — only copies page tables, not memory pages
- Processes that `exec()` immediately never copy any pages
- Shared libraries naturally share pages across processes

### vfork():

A variant of `fork()` where the child uses the **parent's address space** (no COW, no copy at all) until `exec()` is called. Even faster, but dangerous — child must not modify the address space.

---

## 6. Page Replacement Algorithms

When there are no free frames, the OS must **evict** a page to make room.

### Terminology:

- **Reference string:** Sequence of page numbers accessed (e.g., `7 0 1 2 0 3 0 4 2 3 0 3 2 1 2 0 1 7 0 1`)
- **Page fault:** Requested page is not in memory
- **Victim page:** Page chosen for eviction
- **Dirty bit:** If dirty = 1, victim must be written to disk before eviction

---

### Algorithm 1 — FIFO (First-In, First-Out)

Evict the page that has been in memory the **longest** (the oldest).

```
Reference: 1 2 3 4 1 2 5 1 2 3 4 5    Frames: 3

Step  Page  Frames         Fault
  1    1    [1, -, -]       F
  2    2    [1, 2, -]       F
  3    3    [1, 2, 3]       F
  4    4    [4, 2, 3]       F  ← evict 1 (oldest)
  5    1    [4, 1, 3]       F  ← evict 2 (oldest)
  6    2    [4, 1, 2]       F  ← evict 3 (oldest)
  7    5    [5, 1, 2]       F  ← evict 4 (oldest)
  8    1    [5, 1, 2]       -
  9    2    [5, 1, 2]       -
 10    3    [5, 3, 2]       F  ← evict 1
 11    4    [5, 3, 4]       F  ← evict 2
 12    5    [5, 3, 4]       -

Page faults = 9
```

**Belady's Anomaly — same reference string with 4 frames:**

```
Same reference: 1 2 3 4 1 2 5 1 2 3 4 5    Frames: 4

Page faults = 10   ← MORE faults with MORE frames!
```

> FIFO is the only common algorithm that suffers **Belady's Anomaly**.

---

### Algorithm 2 — Optimal (OPT / MIN / Belady's Algorithm)

Evict the page that will **not be used for the longest time in the future**.

```
Reference: 7 0 1 2 0 3 0 4 2 3 0 3 2 1 2 0 1 7 0 1    Frames: 3

At step 4 (page 2, frames = [7,0,1]):
  Next use: 7 → step 18, 0 → step 5, 1 → step 14
  Evict 7 (farthest future use) → [2, 0, 1]

Total faults ≈ 9  ← theoretical minimum
```

- **Theoretical only** — requires knowledge of future
- Used as benchmark to measure how close other algorithms are

---

### Algorithm 3 — LRU (Least Recently Used)

Evict the page that was used **least recently** (furthest in the past).

```
Reference: 7 0 1 2 0 3 0 4 2 3 0 3 2 1 2 0 1 7 0 1    Frames: 3

Step  Page   Frames (MRU→LRU)       Fault
  1    7     [7]                      F
  2    0     [0,7]                    F
  3    1     [1,0,7]                  F
  4    2     [2,1,0]  evict 7         F
  5    0     [0,2,1]  (move 0 to top) -
  6    3     [3,0,2]  evict 1         F
  7    0     [0,3,2]                  -
  8    4     [4,0,3]  evict 2         F
  ...

Total faults ≈ 12
```

**Properties:**
- Does NOT suffer Belady's Anomaly (has the **stack property**)
- Optimal approximation in practice

**Stack Property:**

```
Set of pages in memory with n frames ⊆ set with n+1 frames
→ More frames can only help, never hurt
```

**Implementation Options:**

| Method | Mechanism | Cost |
|---|---|---|
| **Counter / Timestamp** | Hardware timestamps each access; evict minimum | O(n) scan on fault |
| **Doubly-Linked List (Stack)** | Move accessed page to MRU end; evict from LRU end | O(1) but needs lock |
| **Reference bits (hardware approx)** | Bit set on access, periodically shifted; evict lowest | Cheap but approximate |

---

### Algorithm 4 — LRU Approximation: Clock (Second Chance)

Hardware sets a **reference bit** on each page access. The OS sweeps a "clock hand" in circular order.

```
Pages in frames arranged in a circle:
  [P1, R=1] → [P2, R=0] → [P3, R=1] → [P4, R=0] → ...
                            ↑ clock hand

On page fault:
  While current page's R == 1:
    Set R = 0  (give it a "second chance")
    Advance hand
  Evict current page (R == 0)
  Load new page here, set R = 1
  Advance hand
```

- If R is always 1 (all recently used): degenerates to FIFO
- Good approximation of LRU; much cheaper to implement

**Enhanced Clock (NRU):** Uses both R (reference) and M (modified/dirty) bits.

```
Priority for eviction (lowest priority = evict last):
  Class 0: R=0, M=0  ← best victim (not used, not dirty)
  Class 1: R=0, M=1  ← not recently used, but dirty (must write to disk)
  Class 2: R=1, M=0  ← recently used, clean
  Class 3: R=1, M=1  ← recently used, dirty (worst victim)

Evict the page in the lowest non-empty class.
```

---

### Algorithm 5 — LFU (Least Frequently Used)

Evict the page with the **lowest access count**.

```
Problem: Page used heavily early on keeps a high count → never evicted
Fix: Aging — periodically right-shift all counts (exponential decay)
     Recent accesses count more than old ones
```

---

### Algorithm Comparison:

| Algorithm | Faults (approx, 3 frames) | Belady's Anomaly | Notes |
|---|---|---|---|
| **FIFO** | ~15 | YES | Simplest; worst in practice |
| **Optimal** | ~9 | No | Theoretical lower bound |
| **LRU** | ~12 | No | Best practical; costly hardware |
| **Clock** | ~13–14 | No | LRU approximation; used in Linux |
| **LFU** | Varies | No | Frequency bias problem |

> **GATE Formula:** OPT ≤ LRU ≤ Clock ≤ FIFO  (in terms of page faults)

---

## 7. Frame Allocation

How do we decide **how many frames to give each process**?

### Minimum Frames:

Each process must have **enough frames** to hold the pages needed by a single instruction.
- If an instruction needs to access 3 memory locations → needs at least 3 frames
- x86 with 2-level addressing: instruction can reference up to 6 pages → minimum 6 frames

### Allocation Strategies:

#### Equal Allocation:
```
Total frames = F,  processes = n
Each process gets: F / n frames (ignoring remainder)

Problem: unfair — a 10 KB process gets the same as a 500 KB process
```

#### Proportional Allocation:
```
Process i has virtual size s_i,   total = S = Σ s_i
Frames for process i: a_i = (s_i / S) × F

Example:
  F = 64 frames,  Process 1: 10 pages,  Process 2: 127 pages
  S = 137
  a_1 = (10/137) × 64 ≈ 5 frames
  a_2 = (127/137) × 64 ≈ 59 frames
```

#### Priority Allocation:
- Allocate frames proportional to process **priority** (not size)
- High-priority processes get more frames → fewer page faults → run faster

---

### Global vs Local Replacement:

| Type | Description | Pros | Cons |
|---|---|---|---|
| **Local Replacement** | Process can only replace its own pages | Predictable; isolates processes | Can't use idle frames from other processes |
| **Global Replacement** | Process can replace any page in memory (from any process) | Better throughput | One process can steal frames from another; less predictable |

> **GATE Key Point:** Global replacement generally gives better throughput; local is more predictable. Linux uses global replacement.

---

## 8. Thrashing

> **Thrashing:** A process spends more time **handling page faults** than executing instructions. CPU utilization drops drastically.

### Cause:

```
Multiprogramming degree increases
→ Each process gets fewer frames
→ More page faults
→ Processes spend more time waiting for I/O (blocked)
→ CPU appears idle → OS thinks it can load MORE processes
→ Even more page faults → CPU utilization collapses
```

### CPU Utilization vs Multiprogramming Degree:

```
CPU Util
  │          ●●●
  │        ●●   ●●
  │      ●●       ●●
  │    ●●           ●●●●●●●
  └────────────────────────── # processes
              ↑
         Optimal point
         Beyond this = THRASHING
```

### Why It Happens:

```
Process's working set > frames allocated to it
→ Every working set page evicted before it can be reused
→ Constant page faults

Total working sets of all processes > total physical frames
→ System-wide thrashing
```

### Consequences:
- CPU utilization near 0% (all processes waiting on disk I/O)
- Disk utilization near 100% (constant page I/O)
- System appears "frozen"

---

## 9. Working Set Model

> **Working Set W(t, Δ):** The set of pages accessed during the last **Δ** time units (the working set window).

```
Reference string: ... 1 2 3 4 5 3 4 2 4 4 3 4 ...
                                              ↑ current time t

With Δ = 5: working set = {2, 3, 4} (pages accessed in last 5 refs)
```

### Key Properties:

```
Working set window Δ:
  Too small → misses needed pages → excessive faults
  Too large → includes unused pages → wastes frames
  Δ = ∞    → entire program's pages

Ideal: Δ should capture the current locality of reference
```

### Using Working Set to Prevent Thrashing:

```
For each process i:  compute |W_i(t, Δ)|  = working set size

Total demand D = Σ |W_i|

If D > total available frames:
    THRASH → suspend one process (preferably lowest priority or largest)
    → freed frames distributed to remaining processes

If D < total frames:
    Can bring in more processes (increase multiprogramming)
```

### Working Set vs Page Table:

The OS approximates the working set using **reference bits** and **interval timers** (since exact working sets require tracking every memory access).

---

## 10. Page Fault Frequency (PFF)

An alternative to the Working Set Model for controlling thrashing.

```
For each process:
  Monitor its page fault rate (faults per second)

If rate > upper bound threshold:
    Give the process MORE frames
    (or the process is thrashing → suspend it)

If rate < lower bound threshold:
    Take frames away (give to other processes)

Keep all processes' fault rates in the acceptable range [lower, upper]
```

```
Page Fault Rate
     │         ● ← too high (give more frames)
     │       ●
upper│─────●────────── Upper threshold
     │   ●      ●
lower│──────────────●── Lower threshold
     │              ●  ← too low (reclaim frames)
     └──────────────────── # frames allocated
```

---

## 11. Memory-Mapped Files

> **Memory-Mapped File:** A file mapped directly into the **virtual address space** of a process. File I/O becomes memory accesses — no separate read/write syscalls needed.

### How it works:

```
mmap(addr, length, prot, flags, fd, offset)

1. OS maps pages of the file into virtual address space
2. Initially, all mapped pages are invalid (not in RAM)
3. First access to a mapped address → page fault → OS reads the file page into a frame
4. Subsequent accesses: direct memory access (no syscall)
5. On msync() or unmap(): dirty pages written back to file
```

### Advantages:

| Advantage | Explanation |
|---|---|
| **No copy overhead** | No separate user-space buffer; kernel page cache used directly |
| **Demand loading** | Only accessed parts of file read from disk |
| **Shared mappings** | Multiple processes can map the same file → automatic shared memory |
| **Simpler code** | Work with files like memory arrays |

### Private vs Shared Mapping:

```
MAP_SHARED:  Writes visible to other processes and written back to file
MAP_PRIVATE: Copy-on-write — writes create private copies, not written to file
```

### Example — Shared Libraries:

```
libc.so mapped into every process's address space with MAP_SHARED
→ Physical frames for libc are shared by all processes
→ Only one copy of the library in RAM regardless of number of processes
```

---

## 12. Kernel Memory Allocation

User processes use demand paging. The **kernel** needs a different approach — kernel structures must be contiguous (no page faults allowed in kernel).

### Buddy System:

Allocates memory in **power-of-2 sized blocks**.

```
Total memory: 256 KB

Request for 25 KB:
  Need 32 KB (next power of 2)
  Split 256 → [128 | 128]
  Split 128 → [64 | 64]  (take left half)
  Split 64  → [32 | 32]  (take left half)
  Allocate 32 KB block

Free 32 KB block:
  Its buddy (adjacent 32 KB) is also free?
  → Merge into 64 KB block
  Its buddy (adjacent 64 KB) is also free?
  → Merge into 128 KB block
  ... (coalescing)
```

**Pros:** Fast allocation and deallocation, coalescing prevents fragmentation
**Cons:** Internal fragmentation (25 KB request → 32 KB allocated → 7 KB wasted)

---

### Slab Allocator:

Designed for **frequently allocated/freed kernel objects** of the same type (e.g., `struct task_struct`, `struct inode`).

```
Slab = one or more contiguous pages pre-allocated for objects of one type

Slab states:
  Full:    all objects allocated
  Partial: some objects allocated, some free
  Empty:   no objects allocated

Allocation: take an object from a partial/empty slab
Deallocation: return object to slab (mark as free — do NOT zero or free the page)
```

**Advantages:**
- No fragmentation (objects fit perfectly in slab)
- Fast alloc/free (no searching, just mark free/used)
- Objects can be pre-initialized (constructor called once, not on every alloc)

---

## 13. Other Virtual Memory Considerations

### Prepaging:

Bring in multiple pages at once, not just the faulting page, anticipating future use.

```
Prepaging: load page p AND its neighbors p-1, p+1, p+2, ...
           based on spatial locality

Tradeoff: fewer future faults vs. wasted I/O if prefetched pages never used
```

### Page Size Considerations:

| Smaller Pages | Larger Pages |
|---|---|
| Less internal fragmentation | More internal fragmentation |
| More pages needed → larger page table | Fewer entries in page table |
| Better resolution for locality | Coarser locality tracking |
| More page faults (less transferred per fault) | Fewer page faults (more data per transfer) |
| Disk I/O less efficient | Disk I/O more efficient (sequential) |

Modern trend: **huge pages** (2 MB or 1 GB on x86-64) for TLB coverage of large datasets.

### I/O Interlock:

```
Problem:
  Process issues I/O to read into a buffer at virtual address V
  OS starts I/O and switches to another process
  Other process's page replacement evicts the frame holding V's buffer
  I/O completes → writes into wrong frame (just evicted!) → DATA CORRUPTION

Solution:
  LOCK (pin) the I/O buffer pages in memory for the duration of the I/O
  They cannot be selected as victims by the page replacement algorithm
```

### Swapping vs Paging:

| | Swapping | Paging |
|---|---|---|
| **Unit** | Entire process | Individual pages |
| **Granularity** | Coarse | Fine |
| **Overhead** | Very high (swap entire process) | Low (swap individual pages) |
| **Usage** | Context of early OS; now rarely used alone | Used universally |

---

## 14. GATE PYQs & Key Formulas

### Formula Sheet:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EAT (with page faults, no TLB):
  EAT = (1 − p) × m + p × s
  where p = page fault rate, m = memory time, s = fault service time

EAT (with TLB, no page faults):
  EMAT = α(t + m) + (1−α)(t + 2m) = t + (2−α)m
  where α = TLB hit ratio, t = TLB time, m = memory time

EAT (with TLB AND page faults):
  EAT = α(t + m) + (1−α)[t + (1−p)·2m + p·(2m + s)]

Proportional frame allocation:
  a_i = (s_i / S) × F

Working set total demand:
  D = Σ |W_i(t, Δ)|
  If D > F (total frames) → thrash → suspend a process
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Classic GATE Question Types:

**Type 1 — Count page faults for a reference string:**

```
Reference: 1 2 3 4 2 1 5 6 2 1 2 3 7 6 3 2 1 2 3 6
Frames: 4, Algorithm: LRU

Trace through step by step:
  1: [1,-,-,-]              FAULT
  2: [1,2,-,-]              FAULT
  3: [1,2,3,-]              FAULT
  4: [1,2,3,4]              FAULT
  2: [1,2,3,4]              HIT  (2 is present)
  1: [1,2,3,4]              HIT
  5: [5,2,3,4]  evict 1     FAULT  (LRU: 1 used at step 6)
  6: [5,2,6,4]  evict 3     FAULT  (LRU: 3 used at step 3)
  ...
```

**Type 2 — EAT calculation:**

```
Given: TLB hit ratio = 90%, TLB time = 10 ns, Memory time = 100 ns,
       Page fault rate = 0.5%, Page fault service time = 10 ms

Step 1: EAT without page faults:
  EMAT = 0.9(10+100) + 0.1(10+200) = 99 + 21 = 120 ns

Step 2: Include page fault:
  EAT = (1 − 0.005) × 120 + 0.005 × 10,000,000
      = 0.995 × 120 + 50,000
      ≈ 119.4 + 50,000 ≈ 50,119 ns ≈ 50 μs
```

**Type 3 — Belady's Anomaly:**

```
Q: Which page replacement algorithm suffers from Belady's Anomaly?
A: FIFO only. LRU and OPT have the stack property → immune.
```

**Type 4 — Working Set:**

```
Q: Reference string: 2 3 2 1 3 4 3 5 4 3 4 3 2 3 5 3
   Δ = 4 (window size). What is the working set at t=10?
   
At t=10, last 4 references are: 5 4 3 4
Working set = {3, 4, 5}
```

### Common GATE Traps:

| Trap | Correct Answer |
|---|---|
| "OPT has fewest page faults" | TRUE — it's the theoretical minimum |
| "LRU never has more faults than FIFO" | TRUE (LRU ≤ FIFO approximately) but not strictly guaranteed for all strings |
| "More frames always means fewer faults" | TRUE for LRU/OPT; FALSE for FIFO (Belady's) |
| "Dirty page eviction needs one disk write" | TRUE — must write back to swap before frame is reused |
| "Thrashing means 100% CPU" | FALSE — CPU utilization drops near 0% (all processes blocked on I/O) |

---

## 15. FAANG / MAANG Interview Questions

**Q1. What is virtual memory and why is it useful?**
> Virtual memory creates an abstraction where each process sees a private, large address space independent of physical RAM. It enables: (1) programs larger than RAM by loading only needed pages; (2) memory isolation between processes; (3) efficient process creation via COW; (4) shared memory through shared page mappings. The OS + MMU together maintain the illusion that every process has all the memory it needs.

**Q2. Explain Copy-on-Write. Why does it make fork() fast?**
> After `fork()`, the child and parent share the **same physical pages**, all marked read-only. No pages are copied at fork time — only page tables are duplicated. When either process writes to a shared page, a **protection fault** occurs, and the OS creates a private copy of just that page. Since most child processes immediately call `exec()` (replacing their address space), they never write to the parent's pages, so no copying ever happens. This makes `fork()` O(1) regardless of address space size.

**Q3. How does the OS handle a page fault?**
> (1) MMU detects valid bit = 0 → raises page fault exception → CPU switches to kernel mode. (2) OS validates the address — if invalid, sends SIGSEGV to process. (3) OS finds a free frame; if none, runs page replacement (writing dirty victim to swap if needed). (4) OS schedules disk I/O to load the needed page; process blocks. (5) After I/O completes, OS updates the page table entry (frame number, valid = 1). (6) Process is moved to ready queue; CPU restarts the faulting instruction from scratch.

**Q4. What is thrashing and how do you prevent it?**
> Thrashing occurs when the total working sets of all active processes exceed physical memory — each process page-faults constantly, spending more time in I/O than executing. CPU utilization collapses. Prevention: (1) **Working Set Model** — allocate each process enough frames to hold its working set; suspend processes when total demand exceeds available frames. (2) **Page Fault Frequency (PFF)** — monitor fault rate per process; increase frames if too high, reclaim if too low. (3) **Reduce multiprogramming** — suspend some processes. (4) Add more RAM.

**Q5. Global vs local page replacement — which is better?**
> **Global replacement** allows a process to evict pages from any other process in memory. It generally achieves better system-wide throughput because idle processes' frames can be reallocated to active ones. **Local replacement** restricts a process to evicting only its own pages — more predictable per-process behavior but can't capitalize on uneven load. Linux and most modern OSes use **global replacement** (via a global page reclaim pool) with priority adjustments to prevent high-priority processes from being starved.

**Q6. What is a memory-mapped file? How is it different from read()/write()?**
> `mmap()` maps a file's contents directly into the process's virtual address space. Reads/writes go through the kernel's **page cache** without a separate user-space buffer — saving one memory copy vs `read()`. The file is demand-paged: only accessed portions are loaded. Shared mappings let multiple processes access the same file pages — changes visible immediately (vs buffered I/O). `read()`/`write()` are simpler but copy data kernel→user buffer; `mmap` is preferred for large sequential files or shared access.

**Q7. What is the Buddy System and Slab allocator used for in the kernel?**
> The **Buddy System** allocates kernel memory in power-of-2 blocks, splitting and merging "buddy" blocks to satisfy requests with low fragmentation — used for page-granule allocation (`alloc_pages`). The **Slab allocator** sits on top: it pre-allocates pages for fixed-size kernel objects (e.g., inodes, socket buffers, task structs), keeps a cache of free objects, and satisfies alloc/free in O(1) without fragmentation. Slab also allows objects to stay initialized between uses (constructor called once), reducing overhead on hot allocation paths.

---

*Sources used for supplementary detail:*
- [Operating Systems: Virtual Memory — UIC Course Notes](https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/9_VirtualMemory.html)
- [Page Replacement Algorithms — GeeksforGeeks](https://www.geeksforgeeks.org/page-replacement-algorithms-in-operating-systems/)
- [GATE CSE Memory Management PYQs — ExamSIDE](https://questions.examside.com/past-years/gate/gate-cse/operating-systems/memory-management)
