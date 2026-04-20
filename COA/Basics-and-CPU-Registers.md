# COA 01 — Basics of COA & CPU Registers


## Table of Contents

1. [What is COA?](#1-what-is-coa)
2. [Basic Computer Organization](#2-basic-computer-organization)
3. [Von Neumann Architecture](#3-von-neumann-architecture)
4. [Functional Units of a Computer](#4-functional-units-of-a-computer)
5. [CPU — Central Processing Unit](#5-cpu--central-processing-unit)
6. [CPU Registers — Complete List](#6-cpu-registers--complete-list)
7. [Register Transfer Language (RTL)](#7-register-transfer-language-rtl)
8. [Instruction Cycle (Fetch–Decode–Execute)](#8-instruction-cycle-fetchdecodeexecute)
9. [Bus Organization](#9-bus-organization)
10. [Memory Organization Basics](#10-memory-organization-basics)
11. [FAANG / MAANG Interview Questions](#11-faang--maang-interview-questions)

---

## 1. What is COA?

**Computer Organization** deals with the **operational units** and their interconnections that realize the architectural specifications.

**Computer Architecture** refers to those attributes of a system visible to the programmer — instruction set, number of bits used to represent data, I/O mechanisms, addressing techniques.

| Aspect | Computer Organization | Computer Architecture |
|---|---|---|
| Deals with | How hardware components work | What the hardware does (programmer view) |
| Example | Control signals, memory tech | Instruction set, addressing modes |
| Focus | Implementation | Design / Structure |

> **Simple Definition:** COA = How a computer is built + How it executes instructions.

---

## 2. Basic Computer Organization

A computer system has these main physical components:

```
INPUT → CPU (ALU + Control Unit + Registers) ↔ MEMORY → OUTPUT
```

- **Input Unit** — Keyboard, mouse, scanner
- **CPU** — Brain of the computer
- **Memory Unit** — RAM (primary), HDD/SSD (secondary)
- **Output Unit** — Monitor, printer

---

## 3. Von Neumann Architecture

Proposed by **John Von Neumann** (1945). Also called **Princeton Architecture**.

### Key Concepts:
- A **single shared memory** holds both **instructions and data**
- Instructions are fetched and executed **sequentially**
- A **CPU** fetches instructions from memory, decodes them, and executes them

### Von Neumann Bottleneck:
> Since data and instructions share the **same bus** and **same memory**, only one can be accessed at a time → creates a performance bottleneck.

### Von Neumann vs Harvard Architecture:

| Feature | Von Neumann | Harvard |
|---|---|---|
| Memory | Single (shared) | Separate for data & instructions |
| Bus | Single bus | Separate buses |
| Speed | Slower (bottleneck) | Faster (parallel access) |
| Example | General PCs | Microcontrollers, DSP |

---

## 4. Functional Units of a Computer

### 4.1 ALU — Arithmetic Logic Unit
- Performs **arithmetic operations**: Add, Subtract, Multiply, Divide
- Performs **logical operations**: AND, OR, NOT, XOR, Shift
- Operates on data stored in **registers**

### 4.2 Control Unit (CU)
- Acts as the **director/manager** of the CPU
- Fetches instructions from memory
- Decodes instructions
- Generates **control signals** to coordinate all units
- Does **NOT** perform actual data processing

### 4.3 Memory Unit
- **Primary (Main) Memory** — RAM, ROM (volatile / non-volatile)
- **Cache Memory** — Fastest, closest to CPU
- **Secondary Memory** — HDD, SSD (non-volatile, large capacity)

### Memory Hierarchy (fastest → slowest):
```
Registers → Cache → RAM → Secondary Memory → Tertiary (Optical)
```

---

## 5. CPU — Central Processing Unit

The CPU contains:
1. **ALU** — processes data
2. **Control Unit** — directs operations
3. **Register Set** — high-speed temporary storage

### Key CPU Characteristics:
- **Word Size** — number of bits the CPU can process at once (8-bit, 16-bit, 32-bit, 64-bit)
- **Clock Speed** — measured in Hz (GHz today); determines how fast instructions execute
- **Registers** — fastest memory; directly accessible by the CPU

---

## 6. CPU Registers — Complete List

Registers are **small, high-speed storage locations** inside the CPU.

> **Key Property:** Registers are the **fastest** memory in the memory hierarchy.

---

### 6.1 Program Counter (PC)

| Property | Detail |
|---|---|
| Also called | Instruction Pointer (IP) in x86 |
| Holds | Address of the **next instruction** to be fetched |
| Size | Equal to memory address size |
| Auto-increment | Yes — incremented after each fetch |

```
Before fetch : PC = 100   → fetches instruction at address 100
After fetch  : PC = 101   → ready to fetch next instruction
```

> **GATE Point:** PC holds the address of the NEXT instruction, not the current one being executed.

---

### 6.2 Instruction Register (IR)

| Property | Detail |
|---|---|
| Holds | The **currently executing instruction** |
| Updated by | Control Unit after instruction is fetched |
| Purpose | Instruction is decoded from IR |

```
Fetch phase: Memory[PC] → IR
Decode phase: IR → Control Unit interprets opcode
```

---

### 6.3 Memory Address Register (MAR)

| Property | Detail |
|---|---|
| Connected to | Address bus |
| Holds | Address of the memory location to be accessed |
| Direction | CPU → Memory |

```
To READ from memory:
  MAR ← address
  MDR ← Memory[MAR]   ← data comes into MDR

To WRITE to memory:
  MAR ← address
  MDR ← data
  Memory[MAR] ← MDR
```

---

### 6.4 Memory Data Register (MDR) / Memory Buffer Register (MBR)

| Property | Detail |
|---|---|
| Also called | Memory Buffer Register (MBR) |
| Connected to | Data bus |
| Holds | Data being read from / written to memory |
| Acts as | Buffer between CPU and memory |

---

### 6.5 Accumulator (AC / ACC)

| Property | Detail |
|---|---|
| Purpose | Stores **intermediate results** of ALU operations |
| Used in | **Accumulator-based architecture** (single accumulator) |
| Example | `ADD R1` → AC ← AC + R1 |

> In **accumulator architecture**, one operand is always the AC, and the result goes back to AC.

---

### 6.6 Stack Pointer (SP)

| Property | Detail |
|---|---|
| Holds | Address of the **top of the stack** |
| Used for | Subroutine calls, interrupt handling, local variables |
| Operation | Auto-increments / decrements on PUSH/POP |

```
PUSH: SP ← SP - 1 ; Memory[SP] ← data   (stack grows downward)
POP:  data ← Memory[SP] ; SP ← SP + 1
```

---

### 6.7 Status Register / Flag Register / PSW

| Property | Detail |
|---|---|
| Also called | Program Status Word (PSW) / Condition Code Register (CCR) |
| Holds | Individual **flag bits** set after ALU operations |

#### Common Flags:

| Flag | Full Name | Set When |
|---|---|---|
| **Z** | Zero Flag | Result is zero |
| **N / S** | Negative/Sign Flag | Result is negative (MSB = 1) |
| **C** | Carry Flag | Unsigned overflow (carry out of MSB) |
| **V / OV** | Overflow Flag | Signed overflow |
| **P** | Parity Flag | Even number of 1-bits in result |
| **I** | Interrupt Enable Flag | Interrupts are enabled |

---

### 6.8 General Purpose Registers (GPRs)

| Property | Detail |
|---|---|
| Examples | R0–R7, AX, BX, CX, DX (x86) |
| Purpose | Temporary storage for operands and results |
| Flexibility | Can hold data, addresses, or intermediate values |

> In **RISC** architecture → large number of GPRs (32 typically)  
> In **CISC** architecture → fewer dedicated registers

---

### 6.9 Index Register

| Property | Detail |
|---|---|
| Purpose | Used in **indexed addressing mode** |
| Operation | Base address + Index = Effective Address |
| Used for | Array traversal, loop operations |

```
Effective Address = Base Address + Index Register value
```

---

### 6.10 Base Register

| Property | Detail |
|---|---|
| Purpose | Holds the **base address** of a segment/program |
| Used in | **Base register addressing** |
| Advantage | Enables program relocation in memory |

---

### 6.11 Temporary Register (TEMP)

- Used internally by the CPU to hold **intermediate values** during instruction execution
- Not directly accessible by the programmer
- Used by the **Control Unit** during multi-step micro-operations

---

### Summary Table — All CPU Registers

| Register | Full Name | Purpose |
|---|---|---|
| PC | Program Counter | Address of next instruction |
| IR | Instruction Register | Currently executing instruction |
| MAR | Memory Address Register | Address to access in memory |
| MDR/MBR | Memory Data/Buffer Register | Data to/from memory |
| AC | Accumulator | ALU result storage |
| SP | Stack Pointer | Top of stack address |
| PSW/SR | Program Status Word | CPU flag bits |
| GPR | General Purpose Register | General data/address storage |
| IR | Index Register | Indexed addressing offset |
| BR | Base Register | Base address for relocation |

---

## 7. Register Transfer Language (RTL)

RTL is a symbolic notation used to describe **data transfer between registers and memory**.

### Notation:

| Symbol | Meaning |
|---|---|
| `R1` | Register R1 |
| `M[R1]` | Memory location pointed to by R1 |
| `R1 ← R2` | Transfer contents of R2 into R1 |
| `R1 ← R1 + R2` | Add R2 to R1, store in R1 |
| `PC ← PC + 1` | Increment Program Counter |
| `MAR ← PC` | Copy PC value into MAR |

### Example — Fetch Cycle in RTL:
```
Step 1: MAR ← PC          (copy instruction address to MAR)
Step 2: MDR ← M[MAR]      (fetch instruction from memory into MDR)
        PC  ← PC + 1      (increment PC to point to next instruction)
Step 3: IR  ← MDR         (move instruction to Instruction Register)
```

---

## 8. Instruction Cycle (Fetch–Decode–Execute)

Every instruction goes through these phases:

```
FETCH → DECODE → (MEMORY ACCESS) → EXECUTE → (WRITE BACK)
```

### Phase 1 — FETCH
- PC → MAR
- Memory[MAR] → MDR
- MDR → IR
- PC ← PC + 1

### Phase 2 — DECODE
- Control unit reads opcode from IR
- Determines what operation to perform
- Identifies source/destination registers or memory addresses

### Phase 3 — OPERAND FETCH (if needed)
- If instruction needs memory operand:  
  - Effective Address → MAR  
  - Memory[MAR] → MDR

### Phase 4 — EXECUTE
- ALU performs the operation
- Result stored in register (AC / GPR) or memory

### Phase 5 — WRITE BACK (if needed)
- Result written back to register file or memory
- Flags updated in PSW/Status Register

---

## 9. Bus Organization

A **bus** is a shared communication path (set of wires) connecting CPU, memory, and I/O devices.

### Three Types of Buses:

| Bus | Direction | Carries |
|---|---|---|
| **Address Bus** | CPU → Memory/IO (unidirectional) | Memory addresses |
| **Data Bus** | Bidirectional | Actual data |
| **Control Bus** | Both directions | Control signals (Read, Write, Interrupt) |

### Key Relationships:
```
Address Bus width (n bits) → can address 2ⁿ memory locations
Data Bus width              → determines word size transferred per cycle
```

> **GATE Point:**  
> - Address bus = 32 bits → can address **2³² = 4 GB** memory  
> - Data bus = 64 bits → transfers 8 bytes per cycle

### Bus Types by Architecture:

| Type | Description |
|---|---|
| Single Bus | All units share one bus (simple, but slow) |
| Double Bus | Separate buses for memory and I/O |
| Triple Bus | Separate address, data, control buses |

---

## 10. Memory Organization Basics

### Memory Cell
- Smallest unit = **1 bit**
- Grouped into **words** (typically 1 byte = 8 bits)

### Addressability
- Each unique **address** points to one **memory word**
- If address bus = n bits → **2ⁿ addressable locations**
- Total memory = 2ⁿ × word_size (in bits)

### Example:
```
Address bus = 16 bits → 2¹⁶ = 65,536 locations
Word size   = 8 bits (1 byte)
Total Memory = 65,536 × 1 byte = 64 KB
```

### Memory Types:

| Type | Full Name | Volatile? | Writable? |
|---|---|---|---|
| RAM | Random Access Memory | Yes | Yes |
| ROM | Read Only Memory | No | No (factory) |
| PROM | Programmable ROM | No | Once |
| EPROM | Erasable PROM | No | UV erasable |
| EEPROM | Electrically Erasable PROM | No | Electrically |
| Cache | — | Yes | Yes |

---

## 11. FAANG / MAANG Interview Questions

> Questions commonly asked at Google, Amazon, Microsoft, Meta, Apple, Flipkart and other top tech companies.

---

### Conceptual / Definition Level

**Q1. What is the Von Neumann bottleneck? How do modern CPUs address it?**
> The shared bus between CPU and memory limits throughput — only one access (instruction fetch OR data read/write) at a time. Modern CPUs solve it with **multi-level caches**, **prefetching**, **out-of-order execution**, and **wide memory buses**.

**Q2. Why do we need a memory hierarchy? Why can't everything be in registers?**
> Registers are fast but extremely expensive and small. The hierarchy (Registers → Cache → RAM → Disk) balances cost vs speed — frequently used data stays in fast memory, the rest in slower cheap memory.

**Q3. What is the difference between MAR and MDR?**
> MAR (Memory Address Register) holds the **address** of the memory location to access — connected to the address bus. MDR (Memory Data Register) holds the actual **data** being read from or written to memory — connected to the data bus.

**Q4. What does the Program Counter hold? What happens to it after an instruction is fetched?**
> PC holds the address of the **next instruction** to execute. After a fetch, PC is automatically incremented to point to the following instruction. On a branch/jump, PC is explicitly set to the target address.

**Q5. What is the difference between RISC and CISC? Which approach do modern processors use?**
> RISC has simple fixed-length instructions, many registers, and relies on the compiler. CISC has complex variable-length instructions and fewer registers. Modern processors (x86) use a **CISC ISA externally but RISC-like micro-ops internally**.

**Q6. What is the difference between the Carry flag and the Overflow flag?**
> **Carry flag** — set on unsigned arithmetic overflow (carry out of the MSB). **Overflow flag** — set on signed arithmetic overflow (result doesn't fit in signed range). For example: 127 + 1 in 8-bit signed sets overflow (not carry); 255 + 1 in 8-bit unsigned sets carry (not overflow).

**Q7. Why does the stack grow downward in memory?**
> It's a hardware convention — the stack starts at a high address and grows toward lower addresses, while the heap grows upward from low addresses. This allows both to grow toward each other without a fixed partition, maximizing memory utilization.

**Q8. What is the instruction cycle? Walk me through fetch-decode-execute.**
> **Fetch:** PC → MAR, Memory[MAR] → MDR → IR, PC++. **Decode:** Control unit reads opcode from IR. **Execute:** ALU performs the operation. For memory ops, an additional memory access step occurs between decode and execute.

---

### Memory & Cache (Frequently Asked at Google / AMD / Intel roles)

**Q9. What are the three types of cache misses?**
> **Cold miss (Compulsory)** — first access to a block, unavoidable. **Capacity miss** — cache too small to hold all needed blocks. **Conflict miss** — multiple blocks map to the same cache set (direct-mapped caches).

**Q10. What is the difference between direct-mapped, set-associative, and fully associative cache?**
> **Direct-mapped:** each memory block maps to exactly one cache line — fast but high conflict misses. **Fully associative:** a block can go anywhere — fewest misses but slowest lookup. **Set-associative (N-way):** compromise — blocks map to a set of N lines.

**Q11. What is cache coherence? How is it maintained in multi-core processors?**
> When multiple cores have their own caches, the same memory location can have different cached values — cache coherence ensures all cores see a consistent view. Protocols like **MESI** (Modified, Exclusive, Shared, Invalid) track cache line states and broadcast invalidations on writes.

**Q12. What is a TLB? What happens on a TLB miss?**
> TLB (Translation Lookaside Buffer) is a cache for page table entries — maps virtual addresses to physical addresses without walking the full page table. On a miss, hardware (or software on some architectures) **walks the page table**, loads the mapping into TLB, then retries the access.

---

### Pipeline & Advanced (Asked at hardware/systems roles)

**Q13. What are pipeline hazards? How is a data hazard resolved?**
> **Structural hazard** — two instructions need the same hardware unit. **Data hazard (RAW/WAR/WAW)** — instruction depends on result of a previous one not yet written back. **Control hazard** — branch target unknown. Data hazards are resolved via **forwarding (bypassing)**, **pipeline stalls (bubbles)**, or **out-of-order execution**.

**Q14. Why is context switching expensive at the hardware level?**
> Context switching saves/restores all CPU registers and PCB state — but also **flushes the TLB** (since virtual address mappings change) and **pollutes the CPU caches** (new process accesses different memory). These cache and TLB cold-start effects are the dominant cost, not the register save/restore itself.

**Q15. What is register renaming and why is it used?**
> Register renaming maps architectural registers to a larger set of physical registers, eliminating **WAR (write-after-read) and WAW (write-after-write) false dependencies**. This allows more instructions to execute out of order in superscalar processors.

**Q16. What is out-of-order execution? What problem does it solve?**
> The CPU executes instructions in a different order than the program specifies, based on data availability rather than program order. It solves the problem of CPU stalls waiting for slow memory accesses — other independent instructions can execute in the meantime.

---

### Register Quick Reference:

| Register | One-liner |
|---|---|
| PC | "Where is the next instruction?" |
| IR | "What instruction am I executing NOW?" |
| MAR | "Which memory address do I want?" |
| MDR | "What data came from / going to memory?" |
| ACC | "Where does ALU put its answer?" |
| SP | "Where is the top of the stack?" |
| PSW | "What happened after the last operation?" |

---


