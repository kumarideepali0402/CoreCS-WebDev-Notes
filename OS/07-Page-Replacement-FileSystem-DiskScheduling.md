# OS 07 — Page Replacement Algorithms, File System & Disk Scheduling

> GATE 2025 Crash Course | Lecture 07

---

## Table of Contents

1. [Page Replacement — Quick Recap & Numericals](#1-page-replacement--quick-recap--numericals)
2. [File System Concepts](#2-file-system-concepts)
3. [File Allocation Methods](#3-file-allocation-methods)
4. [Inode Structure & File Size Calculations](#4-inode-structure--file-size-calculations)
5. [Directory Structure](#5-directory-structure)
6. [Free Space Management](#6-free-space-management)
7. [Disk Structure & Terminology](#7-disk-structure--terminology)
8. [Disk Scheduling Algorithms](#8-disk-scheduling-algorithms)
9. [Algorithm Comparison & GATE Traps](#9-algorithm-comparison--gate-traps)
10. [GATE PYQs & Key Formulas](#10-gate-pyqs--key-formulas)
11. [FAANG / MAANG Interview Questions](#11-faang--maang-interview-questions)

---

## 1. Page Replacement — Quick Recap & Numericals

> Covered in depth in OS 05 & OS 06. This section focuses on **GATE-style numericals** and tricky cases.

### Reference String Walkthrough — All Algorithms Side by Side

```
Reference string:  3  2  1  0  3  2  4  3  2  1  0  4
Frames:  3
```

#### FIFO:

```
Step  Page  Frames (queue: oldest→newest)     Fault
  1    3    [3,-,-]                             F
  2    2    [3,2,-]                             F
  3    1    [3,2,1]                             F
  4    0    [0,2,1]  evict 3 (oldest)           F
  5    3    [0,3,1]  evict 2 (oldest)           F
  6    2    [0,3,2]  evict 1 (oldest)           F
  7    4    [4,3,2]  evict 0 (oldest)           F
  8    3    [4,3,2]  3 present                  -
  9    2    [4,3,2]  2 present                  -
 10    1    [4,1,2]  evict 3 (oldest)           F
 11    0    [4,1,0]  evict 2 (oldest)           F
 12    4    [4,1,0]  4 present                  -

Total FIFO faults = 9
```

#### LRU:

```
Step  Page  Frames (MRU→LRU)           Fault
  1    3    [3]                          F
  2    2    [2,3]                        F
  3    1    [1,2,3]                      F
  4    0    [0,1,2]   evict 3            F
  5    3    [3,0,1]   evict 2            F
  6    2    [2,3,0]   evict 1            F
  7    4    [4,2,3]   evict 0            F
  8    3    [3,4,2]   promote 3          -
  9    2    [2,3,4]   promote 2          -
 10    1    [1,2,3]   evict 4            F
 11    0    [0,1,2]   evict 3            F
 12    4    [4,0,1]   evict 2            F

Total LRU faults = 10
```

#### Optimal:

```
Step  Page  Frames          Evict (used farthest)      Fault
  1    3    [3,-,-]                                      F
  2    2    [3,2,-]                                      F
  3    1    [3,2,1]                                      F
  4    0    [0,2,1]  evict 3 (next use: step 5)          F
  5    3    [0,2,3]  ...wait, 0 next at step 11          F
               note: 1 next at step 10, 2 next at step 9, 0 next at step 11
               evict 0 (farthest) → [3,2,1]
  ...

Total OPT faults = 8  ← minimum
```

> **Important:** LRU can have MORE faults than FIFO on some reference strings (but generally LRU ≤ FIFO in practice).

---

### Belady's Anomaly — Numerical Proof

```
Reference string:  1  2  3  4  1  2  5  1  2  3  4  5

With 3 frames (FIFO):
  1: [1,-,-] F
  2: [1,2,-] F
  3: [1,2,3] F
  4: [4,2,3] F  evict 1
  1: [4,1,3] F  evict 2
  2: [4,1,2] F  evict 3
  5: [5,1,2] F  evict 4
  1:         - (1 present)
  2:         - (2 present)
  3: [5,3,2] F  evict 1
  4: [5,3,4] F  evict 2
  5:         - (5 present)
  
  Total = 9 faults

With 4 frames (FIFO):
  1: [1,-,-,-] F
  2: [1,2,-,-] F
  3: [1,2,3,-] F
  4: [1,2,3,4] F
  1:           - (hit)
  2:           - (hit)
  5: [5,2,3,4] F  evict 1
  1: [5,1,3,4] F  evict 2
  2: [5,1,2,4] F  evict 3
  3: [5,1,2,3] F  evict 4
  4: [4,1,2,3] F  evict 5
  5: [4,5,2,3] F  evict 1

  Total = 10 faults  ← MORE faults with MORE frames!
```

> Algorithms with the **stack property** (LRU, OPT) are **immune** to Belady's Anomaly.

---

## 2. File System Concepts

> **File System:** The OS subsystem that organizes and manages data stored on secondary storage (disk). It provides an abstract view of storage as named files and directories.

### What a File System Does:

| Function | Description |
|---|---|
| **Naming** | Allows files to be identified by name |
| **Organization** | Hierarchical directories |
| **Storage allocation** | Maps file data to disk blocks |
| **Metadata management** | Permissions, timestamps, ownership |
| **Access control** | Read/write/execute permissions per user/group |
| **Reliability** | Journaling, checksums to survive crashes |

### File Attributes (Metadata):

```
┌───────────────┬────────────────────────────────────────┐
│ Attribute     │ Description                            │
├───────────────┼────────────────────────────────────────┤
│ Name          │ Human-readable file name               │
│ Identifier    │ Unique inode number (not human-readable)│
│ Type          │ Regular, directory, symlink, device... │
│ Location      │ Pointer to file's data blocks on disk  │
│ Size          │ Current file size in bytes             │
│ Protection    │ rwxrwxrwx permissions (owner/group/other) │
│ Timestamps    │ Created, last accessed, last modified  │
│ Owner/Group   │ User ID and group ID                   │
└───────────────┴────────────────────────────────────────┘
```

### File Operations:

| Operation | Syscall (Unix) | Description |
|---|---|---|
| Create | `open(O_CREAT)` | Allocate inode + directory entry |
| Delete | `unlink()` | Remove directory entry; free inode + blocks when link count = 0 |
| Open | `open()` | Return file descriptor; load inode into memory |
| Read | `read()` | Copy data from disk blocks to user buffer |
| Write | `write()` | Copy data from user buffer to disk blocks |
| Seek | `lseek()` | Move file offset pointer |
| Close | `close()` | Release file descriptor |

### Open File Table:

```
Per-process open file table:
  fd 0 → stdin
  fd 1 → stdout
  fd 2 → stderr
  fd 3 → /etc/config (entry: file offset, access mode, pointer to ↓)

System-wide open file table:
  Entry: inode pointer, current file offset, reference count, flags

Inode table (in memory):
  Cached copy of on-disk inode for each open file
```

---

## 3. File Allocation Methods

How does the file system map file data to disk blocks?

### Method 1 — Contiguous Allocation

Each file occupies **consecutive disk blocks**.

```
Disk:
  Block:  0   1   2   3   4   5   6   7   8   9  10  11
          [—] [A] [A] [A] [B] [B] [—] [C] [C] [C] [C] [—]
                             ↑ File B: start=4, length=2

Directory entry stores: (file name, start block, length)
```

**Advantages:**
- Simple — only start + length needed
- Excellent sequential and random access (seek directly to block `start + i`)
- Minimum seek time (blocks are adjacent)

**Disadvantages:**
- **External fragmentation** — holes between files can't be used for larger files
- File cannot grow easily (next blocks may be occupied)
- Must know file size at creation time

```
Fragmentation example:
  After deleting file A (3 blocks) and C (4 blocks):
  Free: 0, 1, 2, 3, 7, 8, 9, 10  (8 free blocks total)
  But can only fit a file of ≤4 blocks (largest contiguous hole)
  A 6-block file CANNOT be stored despite 8 free blocks
```

---

### Method 2 — Linked Allocation

Each block contains a **pointer to the next block** of the file.

```
File "X" starts at block 5:
  Block 5 → [data | next=2]
  Block 2 → [data | next=8]
  Block 8 → [data | next=-1]  ← last block

Directory entry stores: (file name, start block, end block)
```

**Advantages:**
- No external fragmentation — any free block can be used
- File can grow dynamically (add any free block to the chain)

**Disadvantages:**
- **No random access** — must traverse chain to find block i (O(n))
- **Pointer overhead** — each block wastes some bytes for the pointer
  - e.g., 512-byte block with 4-byte pointer → 508 bytes of usable data
- **Reliability** — one broken pointer corrupts rest of file chain

---

### Method 3 — FAT (File Allocation Table)

FAT is an **improved linked allocation** — all next-pointers are stored in a **separate table** (the FAT) rather than inside each data block.

```
FAT (File Allocation Table):
  Index:  0    1    2    3    4    5    6    7    8    9   10
  Value: [—]  [7]  [—]  [9] [EOF] [2]  [—]  [4]  [—] [EOF] [—]

File "X" starts at block 1:
  FAT[1] = 7  → go to block 7
  FAT[7] = 4  → go to block 4
  FAT[4] = EOF → last block
  File X uses blocks: 1, 7, 4

File "Y" starts at block 5:
  FAT[5] = 2, FAT[2] = EOF
  File Y uses blocks: 5, 2
```

**Advantages over linked:**
- FAT can be **cached in memory** → random access possible (traverse FAT in memory, not disk)
- No pointer overhead in data blocks

**Disadvantages:**
- FAT itself can be large (needs an entry per disk block)
  - 200 GB disk with 512-byte blocks → 400M entries → FAT can be hundreds of MB
- FAT must be stored reliably (critical structure) — DOS/Windows keeps two copies

**Used by:** FAT12, FAT16, FAT32 (MS-DOS, Windows, USB drives)

---

### Method 4 — Indexed Allocation (Unix inode style)

Each file has an **index block** (inode) that contains an array of pointers to data blocks.

```
Inode:
  ┌──────────────────────┐
  │ Metadata             │
  │ Direct[0] → Block 5  │
  │ Direct[1] → Block 12 │
  │ Direct[2] → Block 3  │
  │ ...                  │
  │ Direct[11]→ Block 17 │
  │ Single Indirect → BI │
  │ Double Indirect → BII│
  │ Triple Indirect → BIII│
  └──────────────────────┘
```

**Advantages:**
- Supports random access (jump directly to data block via index)
- No external fragmentation
- No maximum file size limit (with multi-level indirect blocks)

**Disadvantages:**
- Small files waste index block space
- Large files need multiple levels of indirection (overhead)

---

## 4. Inode Structure & File Size Calculations

> This is a **major GATE topic** — expect at least one numerical.

### Unix Inode Block Pointer Structure:

```
Inode contains 15 pointers:
  12 Direct pointers       → point directly to data blocks
   1 Single Indirect (SI)  → points to a block of direct pointers
   1 Double Indirect (DI)  → points to a block of SI blocks
   1 Triple Indirect (TI)  → points to a block of DI blocks
```

### General Formulas:

```
Let:
  B  = block size (bytes)
  P  = pointer size (bytes)
  N  = B / P  = number of pointers per block  (block capacity)

Direct:         12 × B
Single Indirect: N × B
Double Indirect: N² × B
Triple Indirect: N³ × B

Maximum file size = 12B + NB + N²B + N³B
                  = B × (12 + N + N² + N³)
```

### Worked Example 1 — Classic GATE:

```
Block size B = 4 KB = 4096 bytes
Pointer size P = 4 bytes
N = 4096 / 4 = 1024 pointers per block

Direct blocks:            12  × 4 KB  =      48 KB
Single Indirect:        1024  × 4 KB  =       4 MB
Double Indirect:     1024²   × 4 KB  =       4 GB
Triple Indirect:     1024³   × 4 KB  =       4 TB
                                       ──────────────
Maximum file size ≈ 4 TB (dominated by triple indirect)
```

### Worked Example 2 — Smaller Pointers:

```
Block size B = 1 KB = 1024 bytes
Pointer size P = 4 bytes
N = 1024 / 4 = 256 pointers per block

Direct:           12 × 1 KB  =      12 KB
Single Indirect: 256 × 1 KB  =     256 KB
Double Indirect: 256² × 1KB  =      64 MB
Triple Indirect: 256³ × 1KB  =      16 GB

Max file size ≈ 16 GB
```

### Worked Example 3 — Given disk address size:

```
Q: Block size = 512 bytes, disk address = 32 bits = 4 bytes.
   How many blocks can a single-indirect block point to?

N = 512 / 4 = 128 blocks
Single indirect: 128 × 512 = 65,536 bytes = 64 KB
```

### Worked Example 4 — Finding which indirect level holds a given block:

```
Block size = 4 KB, pointer = 4 bytes, N = 1024

File byte offset 500 KB:
  Block number = 500 KB / 4 KB = 125

  Direct blocks:  block 0–11     (12 blocks)
  SI range:       block 12–1035  (1024 blocks)
  Block 125 is in SI range → needs 1 indirect block access
  
  Total memory accesses = 1 (read SI block) + 1 (read data block) = 2
  With page table: +1 for page table lookup
```

---

## 5. Directory Structure

### Types of Directory Organization:

#### Single-Level Directory:

```
Root:  [file1] [file2] [file3] [file4] ...

All files in one flat list.
Problem: No two files can have the same name; no grouping by user/project
```

#### Two-Level Directory:

```
Root:
  ├─ User1:  [fileA] [fileB]
  └─ User2:  [fileA] [fileC]

Users have separate directories. Same filename allowed across users.
Problem: No subdirectories within a user's space.
```

#### Tree-Structured Directory (Hierarchical):

```
Root (/)
  ├─ bin/
  │    ├─ ls
  │    └─ cp
  ├─ home/
  │    ├─ alice/
  │    │    ├─ docs/
  │    │    └─ notes.txt
  │    └─ bob/
  └─ etc/
       └─ config

Paths:
  Absolute path: /home/alice/notes.txt  (from root)
  Relative path: ../bob  (relative to current directory)
```

**Used by:** Unix, Windows, all modern OSes.

#### Acyclic-Graph Directory (Shared Files):

```
Allows files/directories to be shared via hard links or symbolic links.

alice/notes.txt ──\
                   ──→ [inode 42 : data blocks]
shared/notes.txt ─/

Hard link: both point to the same inode (same data, same inode number)
Soft link (symlink): a pointer file containing the path to the target
```

**Hard link vs Symbolic link:**

| | Hard Link | Symbolic Link |
|---|---|---|
| Points to | Inode directly | Path string |
| Inode number | Same as original | Different inode |
| Survives original deletion | YES (link count > 0) | NO (dangling link) |
| Can cross filesystems | NO | YES |
| Can link to directory | NO (usually) | YES |
| `ls -l` | same size | shows → target |

---

## 6. Free Space Management

How does the OS track which disk blocks are free?

### Method 1 — Bit Vector (Bitmap):

```
Each bit represents one block:
  0 = free,  1 = allocated

Disk with 16 blocks:
  Bitmap: 1 1 0 0 1 1 1 0 0 0 1 0 0 0 1 1
           0 1 2 3 4 5 6 7 8 9 ...

Free blocks: 2, 3, 7, 8, 9, 11, 12, 13

Find first free block: scan bitmap for 0 bit (fast with hardware bit-scan)
```

**Space for bitmap:**
```
Disk size = 40 GB,  Block size = 4 KB
Total blocks = 40 GB / 4 KB = 10,485,760 blocks
Bitmap size  = 10,485,760 bits = 1,310,720 bytes ≈ 1.25 MB
```

### Method 2 — Linked Free List:

```
Free blocks linked together:
  Block 3 → Block 7 → Block 8 → Block 9 → NULL

Allocate: take block 3 from head of list
Free: prepend to free list
```

- Simple but cannot get contiguous free blocks easily
- Must traverse list to find contiguous run

### Method 3 — Grouping:

- First free block stores addresses of N free blocks
- Last entry in that block points to another block of N free addresses
- Recursive structure

### Method 4 — Counting:

- Store (first free block, count of consecutive free blocks)
- e.g., (block 3, 5) means blocks 3, 4, 5, 6, 7 are free
- Efficient when free space tends to be contiguous

---

## 7. Disk Structure & Terminology

> Understanding disk geometry is essential for disk scheduling problems.

### Physical Disk Components:

```
┌─────────────────────────────────────────────────────┐
│                    Disk Platter                     │
│                                                     │
│    Track (concentric ring)                          │
│      └── Sector (smallest addressable unit, ~512B)  │
│                                                     │
│    Cylinder = all tracks at same radius across      │
│               all platters stacked vertically       │
│                                                     │
│    Read/Write Head (one per platter surface)        │
│    Disk Arm (moves all heads together)              │
└─────────────────────────────────────────────────────┘
```

### Key Timing Terms:

| Term | Definition | Typical Value |
|---|---|---|
| **Seek Time** | Time to move arm to the correct **cylinder/track** | 5–15 ms |
| **Rotational Latency** | Time for the correct **sector** to rotate under the head | 0–8 ms (avg = half a rotation) |
| **Transfer Time** | Time to read/write the actual **data** | ~1 ms for a few KB |
| **Disk Access Time** | Seek + Rotational Latency + Transfer Time | |
| **RPM** | Disk rotation speed (revolutions per minute) | 5400–15000 RPM |

### Rotation Speed and Latency:

```
RPM = 7200
Rotation period T = 60 / 7200 = 8.33 ms
Average rotational latency = T / 2 = 4.17 ms
```

### Modern Disk Addressing:

- **CHS (Cylinder-Head-Sector):** Old physical addressing — specify cylinder, head, sector
- **LBA (Logical Block Addressing):** Modern standard — each block has a sequential number 0, 1, 2, ...

> In GATE problems, disk requests are given as **cylinder numbers** (track numbers). Seek distance = |current_cylinder − target_cylinder|.

---

## 8. Disk Scheduling Algorithms

Goal: **minimize total seek time** (total head movement in cylinders).

### Setup for All Examples:

```
Disk cylinders:       0 to 199
Current head position: 53 (moving toward higher cylinders)
Request queue (order of arrival): 98, 183, 41, 122, 14, 124, 65, 67
```

---

### Algorithm 1 — FCFS (First Come, First Served)

Service requests **in the order they arrive**.

```
Head path: 53 → 98 → 183 → 41 → 122 → 14 → 124 → 65 → 67

Distances:
  |53  - 98|  = 45
  |98  - 183| = 85
  |183 - 41|  = 142
  |41  - 122| = 81
  |122 - 14|  = 108
  |14  - 124| = 110
  |124 - 65|  = 59
  |65  - 67|  = 2
  
Total = 45 + 85 + 142 + 81 + 108 + 110 + 59 + 2 = 632 cylinders
```

**Pros:** Simple, fair (no starvation)
**Cons:** Large head movements, poor performance under heavy load

---

### Algorithm 2 — SSTF (Shortest Seek Time First)

Service the request **closest to the current head position** (greedy).

```
Head at 53, Requests: {98, 183, 41, 122, 14, 124, 65, 67}

Step  Head   Distances to remaining requests     Chosen
  1    53    41→12, 65→12, 67→14, 14→39...       65 (tie: choose 65)
             actually: |53-41|=12, |53-65|=12 → tie → pick lower = 41? 
             In most GATE problems: pick the one in the current direction first, or pick lower number on tie.
             Let's pick 65 (ties go to lower cylinder number first, moving in either direction).

Standard SSTF trace:
  53 → 65 (dist 12) → 67 (dist 2) → 41 (dist 26) → 14 (dist 27)
     → 98 (dist 84) → 122 (dist 24) → 124 (dist 2) → 183 (dist 59)

Total = 12 + 2 + 26 + 27 + 84 + 24 + 2 + 59 = 236 cylinders
```

> Note: tie-breaking rule affects the answer. GATE problems usually have no ties, or specify a rule.

**Pros:** Much better average seek time than FCFS
**Cons:**
- **Starvation** — requests near the current head always served first; requests at far end may wait forever
- Not optimal (greedy, not globally optimal)

---

### Algorithm 3 — SCAN (Elevator Algorithm)

Head moves in **one direction**, services all requests in that direction, then **reverses** at the end of the disk.

```
Initial direction: toward higher cylinders
Head at 53, Requests: {98, 183, 41, 122, 14, 124, 65, 67}

Sort requests: 14, 41, 65, 67, 98, 122, 124, 183

Moving UP from 53:
  53 → 65 → 67 → 98 → 122 → 124 → 183 → 199  ← reaches end
  Then reverses (moves DOWN):
  199 → 41 → 14

Distances:
  53→65   = 12
  65→67   = 2
  67→98   = 31
  98→122  = 24
  122→124 = 2
  124→183 = 59
  183→199 = 16   (going to end of disk)
  199→41  = 158
  41→14   = 27

Total = 12+2+31+24+2+59+16+158+27 = 331 cylinders
```

**Pros:** No starvation (everything serviced in at most one full sweep)
**Cons:** Requests just behind the head (on the side it just came from) have to wait for a full sweep

---

### Algorithm 4 — C-SCAN (Circular SCAN)

Head moves in **one direction only**. When it reaches the end, it jumps back to the **beginning** (cylinder 0) without servicing on the return trip.

```
Initial direction: toward higher cylinders
Head at 53, Requests: {98, 183, 41, 122, 14, 124, 65, 67}

Moving UP from 53:
  53 → 65 → 67 → 98 → 122 → 124 → 183 → 199  (end)
  Jump to 0 (no servicing during jump)
  Continue UP from 0:
  0 → 14 → 41

Distances:
  53→65   = 12
  65→67   = 2
  67→98   = 31
  98→122  = 24
  122→124 = 2
  124→183 = 59
  183→199 = 16
  199→0   = 199  (jump — counted as movement)
  0→14    = 14
  14→41   = 27

Total = 12+2+31+24+2+59+16+199+14+27 = 386 cylinders
```

> Note: Some GATE problems count the 199→0 jump, some don't (consider it "resetting"). Check what the question asks.

**Pros:** More **uniform wait time** than SCAN — requests near just-serviced end wait no longer than those near the "about to service" end
**Cons:** Slightly more total movement than SCAN

---

### Algorithm 5 — LOOK

Like SCAN, but the head **does not go all the way to the end of the disk** — it reverses at the **last request** in that direction.

```
Initial direction: toward higher cylinders
Head at 53, Requests: {98, 183, 41, 122, 14, 124, 65, 67}

Moving UP from 53:
  53 → 65 → 67 → 98 → 122 → 124 → 183  ← reverses here (183 is last UP request)
  Reversing DOWN:
  183 → 41 → 14

Distances:
  53→65   = 12
  65→67   = 2
  67→98   = 31
  98→122  = 24
  122→124 = 2
  124→183 = 59
  183→41  = 142
  41→14   = 27

Total = 12+2+31+24+2+59+142+27 = 299 cylinders
```

**LOOK vs SCAN:** LOOK is SCAN without the unnecessary travel to disk ends → generally less movement.

---

### Algorithm 6 — C-LOOK (Circular LOOK)

Like C-SCAN, but jumps back to the **smallest pending request** (not cylinder 0) when it reaches the highest pending request.

```
Initial direction: toward higher cylinders
Head at 53, Requests: {98, 183, 41, 122, 14, 124, 65, 67}

Sorted: 14, 41, 65, 67, 98, 122, 124, 183

Moving UP from 53:
  53 → 65 → 67 → 98 → 122 → 124 → 183  (highest request — jump back)
  Jump to 14 (lowest request, no service during jump)
  Continue UP:
  14 → 41

Distances:
  53→65   = 12
  65→67   = 2
  67→98   = 31
  98→122  = 24
  122→124 = 2
  124→183 = 59
  183→14  = 169  (jump)
  14→41   = 27

Total = 12+2+31+24+2+59+169+27 = 326 cylinders
```

---

## 9. Algorithm Comparison & GATE Traps

### Seek Distance Comparison (same example):

| Algorithm | Total Seek Distance | Notes |
|---|---|---|
| **FCFS** | 632 | Worst; no optimization |
| **SSTF** | 236 | Best average; starvation risk |
| **SCAN** | 331 | No starvation; goes to disk end |
| **C-SCAN** | 386 | More uniform; goes to disk end + jump |
| **LOOK** | 299 | Like SCAN but no wasted travel to ends |
| **C-LOOK** | 326 | Like C-SCAN but jumps to lowest request |

### Algorithm Selection Guide:

| Scenario | Best Algorithm |
|---|---|
| Light load | FCFS (simple, fair) |
| Heavy load, maximize throughput | SSTF or LOOK |
| Real-time / fairness needed | SCAN or C-SCAN |
| SSDs | FCFS (no mechanical seek — all algorithms equivalent) |
| General purpose OS | LOOK or C-LOOK (Linux default: CFQ/deadline) |

### GATE Traps:

| Trap | Correct Answer |
|---|---|
| "SSTF always gives minimum seek time" | FALSE — it's greedy, not globally optimal |
| "SCAN goes to the last requested cylinder, then reverses" | FALSE — SCAN goes to the **end of disk**, then reverses (LOOK stops at last request) |
| "C-SCAN is always faster than SCAN" | FALSE — C-SCAN has more movement (the return jump) |
| "SSTF can cause starvation" | TRUE — requests far from head may never be served |
| "FCFS never causes starvation" | TRUE — FCFS is strictly ordered by arrival |
| "SSDs don't benefit from disk scheduling" | TRUE — SSDs have no mechanical parts; seek time is uniform |

---

## 10. GATE PYQs & Key Formulas

### Formula Sheet:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Disk Access Time:
  Access Time = Seek Time + Rotational Latency + Transfer Time
  Average Rotational Latency = (1 / RPM) × 60 × 1000 / 2  ms

Inode Max File Size:
  N = Block Size / Pointer Size
  Max = 12×B + N×B + N²×B + N³×B

FAT entry count:
  = Total disk blocks = Disk Size / Block Size

Bitmap size:
  = Total blocks / 8  bytes  (1 bit per block)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### GATE Type 1 — Disk Scheduling Seek Distance:

```
Q: Disk head at cylinder 100, moving toward lower numbers.
   Requests: 55, 58, 39, 18, 90, 160, 150, 38, 184
   Algorithm: SCAN

Sort: 18, 38, 39, 55, 58, 90, 100, 150, 160, 184

Moving DOWN from 100:
  100 → 90 → 58 → 55 → 39 → 38 → 18 → 0 (end) → reverse
  → 150 → 160 → 184

Total = (100-0) + (184-0) = 100 + 184 = 284 cylinders

(Tip: for SCAN going to 0 then back up: total = 2×max_distance - first_in_other_direction)
```

### GATE Type 2 — Inode Calculation:

```
Q: A file system uses 1 KB blocks and 4-byte disk addresses. 
   An inode has 12 direct, 1 single indirect, 1 double indirect block pointers.
   What is the maximum file size?

N = 1024 / 4 = 256

Direct:   12 × 1 KB  =  12 KB
SI:      256 × 1 KB  = 256 KB
DI:    256² × 1 KB   =  64 MB

Max = 12 KB + 256 KB + 64 MB ≈ 64.26 MB
```

### GATE Type 3 — Block access count for a given offset:

```
Q: Block size = 4 KB, pointer = 4 bytes (N=1024).
   File is 14,000 KB. How many disk accesses needed to read
   the byte at offset 13,000 KB? (assume inode is in memory, 
   NO TLB, NO cache)

Block containing 13,000 KB:
  Block number = 13,000 KB / 4 KB = 3250

  Direct blocks:  0–11      (12 blocks)
  SI range:       12–1035   (1024 blocks)
  DI range:       1036–1049611  (1024² blocks)

  Block 3250 is in DI range (1036 ≤ 3250 ≤ 1049611)
  
  Relative position in DI: 3250 - 1036 = 2214
  DI block index:   2214 / 1024 = 2  (row in DI)
  SI block index:   2214 % 1024 = 166 (column in SI)

  Accesses needed:
    1. Read DI block (outer)
    2. Read SI block (inner, at row 2)
    3. Read actual data block
    Total = 3 disk accesses
```

### GATE Type 4 — Rotational Latency:

```
Q: A disk rotates at 6000 RPM. What is the average rotational latency?

T = 60 / 6000 = 0.01 seconds = 10 ms  (full rotation)
Average latency = T / 2 = 5 ms
```

### GATE Type 5 — File Allocation:

```
Q: A file of 100 KB is stored on a disk with 1 KB blocks.
   How many blocks are needed?
   (a) Contiguous allocation
   (b) Linked allocation (4-byte pointer per block)
   (c) Indexed allocation with 4-byte pointers

(a) 100 blocks

(b) Each block holds 1024 - 4 = 1020 bytes of data
    Blocks = ceil(100 × 1024 / 1020) = ceil(100.39) = 101 blocks

(c) All 100 data blocks + 1 index block = 101 blocks
    (if all 100 pointers fit in the index block: 1024/4 = 256 slots ≥ 100 ✓)
```

---

## 11. FAANG / MAANG Interview Questions

**Q1. How does the Unix inode work? What information does it store?**
> An **inode** (index node) is the kernel's per-file metadata structure. It stores: file type, permissions, owner/group IDs, timestamps (ctime/mtime/atime), size, link count, and — crucially — an array of block pointers (12 direct + 1 single-indirect + 1 double-indirect + 1 triple-indirect). The inode does **not** store the file name — names are in directory entries that map a name to an inode number. Multiple directory entries can point to the same inode (hard links); the file is deleted only when the link count drops to zero and no process has it open.

**Q2. What is the difference between a hard link and a symbolic link?**
> A **hard link** is a new directory entry pointing to the same inode as the original file. Both names are equal — same inode, same data, same permissions. Deleting one doesn't affect the other (inode freed when last link removed). Hard links cannot span filesystems or link to directories. A **symbolic (soft) link** is a special file whose content is a pathname. It has its own inode. If the target is deleted, the symlink becomes a dangling pointer — any access fails with "No such file". Symlinks can cross filesystems and link to directories, making them more flexible but slightly slower (require an extra stat + open).

**Q3. Compare contiguous, linked, and indexed file allocation.**
> **Contiguous:** fast sequential and random access (start + offset), minimal overhead, but causes external fragmentation and cannot grow files easily. **Linked:** any free block usable (no fragmentation), files grow easily, but random access is O(n) (must follow chain) and one broken pointer corrupts the file. FAT improves on this by caching pointers in memory. **Indexed (inode):** supports both sequential and O(1) random access via multi-level block pointer tree, no external fragmentation, handles arbitrary file sizes via single/double/triple indirect blocks — at the cost of extra disk accesses for indirect blocks and index block overhead for small files. Unix/Linux use indexed allocation.

**Q4. Explain SSTF disk scheduling. What is its main drawback?**
> SSTF (Shortest Seek Time First) always picks the pending request whose cylinder is closest to the current head position. It minimizes seek distance greedily and significantly outperforms FCFS in throughput. However, its main drawback is **starvation**: if requests near the current head position keep arriving, far-away requests may never be serviced. It can also be hard to implement fairly. Linux's CFQ (Completely Fair Queuing) and deadline schedulers solve this by maintaining per-process queues and deadline timers to ensure no request waits too long.

**Q5. What is the difference between SCAN and LOOK?**
> **SCAN** moves the disk head to the **physical end of the disk** (cylinder 0 or max) before reversing direction, even if there are no pending requests near the ends — wasting movement. **LOOK** is the practical improvement: it only travels as far as the **last pending request** in the current direction, then reverses. LOOK avoids unnecessary travel to disk boundaries, resulting in less total head movement and lower latency. Most real OS implementations use LOOK-type algorithms (e.g., Linux's deadline scheduler uses a time-based LOOK variant).

**Q6. What happens when you delete a file in Unix? When is disk space freed?**
> `unlink()` removes the **directory entry** mapping the file name to its inode. The inode's **link count** is decremented. If the link count drops to 0 **and** no process has the file open (reference count = 0), the OS: (1) frees all data blocks listed in the inode (marks them free in the bitmap), (2) frees the inode itself. If a process still has the file open when `unlink()` is called, the file remains accessible via the file descriptor but the directory entry is gone — disk space is freed only when the last file descriptor is closed. This is why deleting a large log file doesn't free space if a daemon still has it open (`lsof | grep deleted`).

**Q7. How does a file system handle a crash mid-write?**
> A simple file system that writes data blocks and then updates metadata (inode, bitmap) can leave inconsistent state after a crash — e.g., data blocks allocated but inode not updated. The classic fix is **journaling**: before modifying any metadata, first write the intended changes to a **journal (log)** on disk. After journaling, apply the changes. On crash recovery, replay the journal to complete any interrupted operations. Most modern file systems (ext4, NTFS, XFS, APFS) use journaling. An alternative is **copy-on-write (CoW)** file systems (ZFS, Btrfs): never overwrite data in place — write new version to a free location, then atomically update the pointer. Old data remains valid until the new write commits, ensuring consistency by construction.

---

*Sources used for supplementary detail:*
- [Disk Scheduling Algorithms — GeeksforGeeks](https://www.geeksforgeeks.org/operating-systems/disk-scheduling-algorithms/)
- [Page Replacement Algorithms — GeeksforGeeks](https://www.geeksforgeeks.org/operating-systems/page-replacement-algorithms-in-operating-systems/)
- [Inode in Operating System — GeeksforGeeks](https://www.geeksforgeeks.org/operating-systems/inode-in-operating-system/)
- [File System IO & Protection — GATE PYQs ExamSIDE](https://questions.examside.com/past-years/gate/gate-cse/operating-systems/file-system-io-and-protection)
- [Disk Scheduling GATE PYQs — PracticePaper](https://practicepaper.in/gate-cse/disk-scheduling)
