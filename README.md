# DRAM Memory Simulator

## Overview

This project is a cycle-accurate DRAM simulator built for educational purposes, modeling the full memory hierarchy from the Memory Controller down to the subarray matrix level. The goal is to reinforce understanding of how real DRAM systems work — including scheduling, bus contention, row buffer management, and address decomposition — while practicing object-oriented design.

---

## Motivation & Learning Goals

Modern CPUs can execute instructions in ~1 cycle, but a DRAM access costs ~500 CPU cycles. Understanding *why* requires understanding every layer of the memory hierarchy. This simulator traces a memory request from the moment the LLC issues it, through the memory controller, down the channel to the DIMM, through the rank and bank, all the way to the subarray matrix where the actual capacitor cells live.

This is also a hands-on OOP exercise. Each layer of the hierarchy maps naturally to a class with well-defined responsibilities and interfaces.

---

## System Architecture

### Conceptual Hierarchy

```
Memory Controller
    └── Channel (x2)
            └── DIMMs (x2 per channel)
                    └── Rank (x2 per DIMM)
                            └── Banks (x16 per rank)
                                    └── Subarrays (x512 per bank)
                                            └── Matrix (1024 rows × 512 bit-columns)
```

### Physical Implementation

In practice, a rank is composed of **9 x8 DRAM chips** placed side by side:

- **8 data chips** — each has 8 data pins (x8), contributing 8 bits per transfer cycle
- **1 parity chip** — used for ECC (Error Correcting Code); stores syndrome bits for single-bit error detection. **Note: ECC behavior is architecturally present but not functionally simulated in this implementation.** The 9th chip is noted for correctness but does not affect scheduling or latency modeling.

Each chip internally contains a **logical (partial) bank** and a **logical row buffer** that are fractions of the conceptual bank and row buffer sizes. The rank coordinates all 9 chips together so that a single logical request is serviced in parallel across all of them.

#### Why x8 and Why 64 Bytes?

The LLC cache line size is **64 bytes (512 bits)**. To move one full cache line out of DRAM in a single burst:

- Each x8 chip sends 8 bits per beat
- Over an **8-beat burst**, one chip sends 64 bits = 8 bytes
- All 8 data chips together send: 8 chips × 8 bytes = **64 bytes per burst** ✓

This is how DRAM achieves high bandwidth — all chips fire in parallel, each contributing their slice of the cache line.

---

## Capacity & Address Space

| Parameter | Value |
|---|---|
| Total addressable space | 4 GB (32-bit addressing) |
| Memory channels | 2 → 2 GB per channel |
| DIMMs per channel | 2 → 1 GB per DIMM |
| Ranks per DIMM | 2 → 512 MB per rank |
| Banks per rank | 16 → 32 MB per bank |
| Data chips per rank | 8 (+ 1 ECC) |
| Capacity per chip | 64 MB |
| Subarrays per bank | 512 → 64 KB per subarray |
| Matrix dimensions | 1024 rows × 512 bit-columns (= 64 KB) |

### Subarray & Matrix Design

The subarray abstraction exists for a physical reason: if a bitline (column) is too long, the capacitance of the DRAM cells overwhelms the sense amplifier and the signal cannot be reliably read. Subarrays keep bitline lengths short. This detail is **invisible to the memory controller** but is vital for cycle-accurate latency modeling.

Each subarray is a **1024-row × 512-column** bit matrix:
- 512 bits per row = 64 bytes = one full cache line ✓
- 1024 rows per subarray
- 512 subarrays per bank → 512 × 1024 × 64 bytes = **32 MB per bank** ✓

When a row is activated, the entire 64-byte row is loaded into the **row buffer** (sense amplifiers). Subsequent column reads from the same open row are fast (row buffer hits). A different row requires a **precharge** followed by a new **activate** (row buffer miss/conflict).

---

## Address Decomposition (32-bit)

The memory controller decodes a 32-bit physical address as follows:

```
Bits  31–23  |  22–18  |  17–13  |  12–9  |  8   |  7   |  6   |  5–0
  Subarray   |   Row   |  Column |  Bank  | Rank | DIMM | Chan | Offset
   (9 bits)  | (5 bits)| (5 bits)|(4 bits)|(1 bit)|(1bit)|(1bit)|(6 bits)
```

| Field | Bits | Width | Notes |
|---|---|---|---|
| Cache line offset | 0–5 | 6 bits | 2^6 = 64 byte cache line |
| Channel interleave | 6 | 1 bit | Alternates between channel A and B every 64 bytes |
| DIMM | 7 | 1 bit | Selects DIMM within channel |
| Rank | 8 | 1 bit | Selects rank within DIMM |
| Bank | 9–12 | 4 bits | 2^4 = 16 banks |
| Column | 13–17 | 5 bits | Column within subarray row |
| Row | 18–22 | 5 bits | Row within subarray |
| Subarray | 23–31 | 9 bits | 2^9 = 512 subarrays |

**Ordering rationale:** Subarray bits are placed highest so that sequential cache lines walk through columns and rows *before* crossing subarray boundaries. This maximizes row buffer locality for sequential access patterns, which is the most common workload.

**Channel interleave:** The LLC decides which memory controller to route to using bit 6, so the first 64-byte block goes to channel A, the next to channel B, alternating. This is handled before the address reaches the memory controller and is not re-decoded inside it.

---

## Memory Controller Design

### Queues

The memory controller maintains two separate queues:

- **Load Queue (Read Queue):** Holds incoming read requests
- **Store Queue (Write Queue):** Holds incoming write requests

**Write drain policy:** The store queue is only drained when it reaches a **high watermark**. At that point, writes are issued until the queue reaches the **low watermark (0)**. This batches writes together to minimize bus turnaround penalties and avoids polluting read scheduling.

**Read priority:** Reads are always prioritized over writes. Writes are not performance-critical (they can be buffered), while reads block the CPU until completed.

**Store-to-load forwarding:** When a read request arrives, the store queue is checked first. If the load address matches a pending store, the data is forwarded directly from the store queue without going to DRAM. This is critical for correctness — without forwarding, a read could return stale data that has been logically overwritten.

### FR-FCFS Scheduling

The memory controller implements **First-Ready, First-Come-First-Served (FR-FCFS)** scheduling. When ordering a batch of requests, the priority is:

1. **Row buffer hits first** — requests that target an already-open row are served before others, avoiding costly precharge + activate sequences
2. **Older requests first** — among requests with the same row buffer status, older requests are served first (FCFS ordering)

This policy maximizes row buffer hit rate and thus overall throughput.

### DRAM Command Sequence

Every memory access issues a sequence of DRAM commands, each with an associated latency cost (to be specified):

| Command | Description |
|---|---|
| `ACTIVATE` (ACT) | Opens a row — copies the row from the cell array into the row buffer |
| `READ` (RD) | Issues a column read from the open row buffer |
| `WRITE` (WR) | Issues a column write to the open row buffer |
| `PRECHARGE` (PRE) | Closes the open row — restores the row buffer for the next activate |

Latency parameters (tRCD, tCL, tRP, tWR, bus turnaround) will be defined separately and plugged into the simulator.

### Bus Batching

The memory channel is **bidirectional and shared** across both DIMMs. Switching the bus direction from write to read (or vice versa) incurs a **bus turnaround penalty**. The controller batches all reads together and all writes together within a scheduling window to minimize this overhead.

---

## I/O Format

### Input File

A plain text file of memory requests, one per line:

```
R: 0x1A2B3C4D
W: 0xDEADBEEF
R: 0x00000040
```

### Output File

For each request, the simulator logs the decoded destination and total simulated latency:

```
R: 0x1A2B3C4D -> Memory Controller: [Channel=0, DIMM=1, Rank=0, Bank=3, Subarray=12, Row=5, Col=22]; Latency: [142 cycles]
W: 0xDEADBEEF -> Memory Controller: [Channel=1, DIMM=0, Rank=1, Bank=7, Subarray=255, Row=31, Col=15]; Latency: [0 cycles (buffered)]
```

Writes that are buffered in the store queue and not yet drained report their latency at drain time.

---

## Class Structure (Planned)

```
MemoryController
    ├── LoadQueue
    ├── StoreQueue
    ├── FRFCFSScheduler
    └── Channel
            └── DIMM
                    └── Rank
                            ├── DRAMChip x9 (8 data + 1 ECC)
                            └── Bank x16
                                    └── Subarray x512
                                            └── Matrix (1024 × 512 bits)
```

Each class is responsible for one layer of the hierarchy. The `MemoryController` drives the simulation loop, issues commands, and tracks latency. Lower layers are passive — they model state (open row, row buffer contents) and report timing costs back up the chain.

---

## Known Simplifications

- ECC correction is not functionally simulated; the 9th chip is present architecturally only
- The channel interleave bit (bit 6) is resolved before the request reaches the memory controller
- Actual electrical behavior (capacitor charge, sense amplifier dynamics) is abstracted into fixed latency parameters
- Multi-controller parallelism is modeled by address striping but controllers do not interfere with each other
