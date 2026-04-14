# CS220: 32-bit MIPS Processor (Labs 7–9)

> **Course:** CS220 – Computer Organization — IIT Kanpur, Spring 2026  
> **Instructor:** Prof. Mainak Chaudhuri  
> **Target Hardware:** PYNQ-Z2 FPGA Board (Zynq Z-7000 series)  
> **HDL:** Verilog, synthesized via Vivado/Vitis IDE

---

## Overview

This repository contains the Verilog implementation of a **32-bit MIPS processor** developed across three lab assignments (Labs 7, 8, and 9). The processor is progressively built up — starting from a basic single-cycle core supporting arithmetic/logic instructions, and growing into a full three-cycle pipelined design with branch/jump support, memory access (load/store), and keyboard I/O — all implemented and demonstrated on the PYNQ-Z2 FPGA board.

---

## Architecture

The processor follows a **three-cycle execution model**:

| Cycle | Operations |
|-------|-----------|
| **Cycle 1** | Instruction Fetch (IF) + Instruction Decode (ID) |
| **Cycle 2** | Execute (EX) — ALU computation, branch target calculation |
| **Cycle 3** | Write-Back (WB) — Register file update / Memory access |

---

## Repository Structure

```
.
├── src/
│   ├── Processor.v        # Top-level MIPS processor module
│   ├── ALU.v              # Arithmetic Logic Unit
│   ├── RegisterFile.v     # 32-register file (2 read ports, 1 write port)
│   ├── Memory.v           # 4KB byte-addressable data memory (4 × 1KB blocks)
│   ├── Computer.v         # Wrapper connecting Processor + Instruction Memory
│   └── defs.vh            # Shared macro/parameter definitions
├── testbench/
│   └── testbench.v        # Simulation testbench
├── constraints/
│   └── PYNQ-Z2_v1.0.xdc  # FPGA pin constraints file
└── README.md
```

---

## Lab 7 — Arithmetic & Logic Instructions

**Goal:** Implement a 3-cycle MIPS processor supporting R-type and I-type arithmetic/logic instructions.

### Supported Instructions

| Category | Instructions |
|----------|-------------|
| **R-type Arithmetic/Logic** | `add`, `sub`, `and`, `or`, `xor`, `nor` |
| **I-type Immediate** | `addi`, `andi`, `ori`, `xori` |
| **Shifts** | `sll`, `srl`, `sra`, `sllv`, `srlv`, `srav` |
| **System** | `syscall` (halt/exit) |

### Milestones

1. **Milestone 1:** Single-cycle implementation — all stages in one clock cycle.
2. **Milestone 2:** Two-cycle — fetch/decode/execute in Cycle 1, write-back in Cycle 2.
3. **Milestone 3:** Three-cycle — fetch/decode in Cycle 1, execute in Cycle 2, write-back in Cycle 3.

### Key Module Interfaces

```verilog
// Top-level Processor
module Processor(
    input         clk,
    input         reset,
    input  [31:0] ins,           // Instruction from Instruction Memory
    output        halt,          // High when syscall (exit) is reached
    output reg [7:0] pc,         // Program Counter (byte address)
    output [31:0] io_reg1,       // I/O output registers (for print syscalls)
    output [31:0] io_reg2,
    output [31:0] io_reg3,
    output [31:0] io_reg4
);

// ALU
module ALU(
    input  [31:0] src1,
    input  [31:0] src2,
    input  [5:0]  opcode,
    input  [5:0]  func,
    output [31:0] dest_data,
    output        dest_data_valid
);
```

### Hardware
- **Instruction Memory:** 4KB (1024 × 32-bit words), word-aligned.
- **Program Counter:** 8-bit, byte-addressed (increments by 4 each fetch cycle).
- **Register File:** 32 × 32-bit registers, with 2 read ports and 1 write port.

---

## Lab 8 — Branch, Jump & Comparison Instructions

**Goal:** Extend the three-cycle processor to support control flow and comparison instructions, plus unlimited print syscall support via stalling.

### New Instructions

| Category | Instructions | Opcode / Funct |
|----------|-------------|----------------|
| **Conditional Branch** | `beq`, `bne` | op = 4, 5 |
| **Unconditional Jump** | `j` | op = 2 |
| **Comparison (R-type)** | `slt`, `sltu` | funct = 0x2A, 0x2B |
| **Comparison (I-type)** | `slti` | op = 0xA |

Branch target address and comparison results are computed in **Cycle 3**.

### Instruction Formats

```
beq/bne :  [op:6][rs:5][rt:5][offset:16]
j       :  [op:6][address:26]
slt/sltu:  [op:6][rs:5][rt:5][rd:5][sa:5][funct:6]
slti    :  [op:6][rs:5][rt:5][immediate:16]
```

### Unlimited Print Stalling

The processor supports an arbitrary number of `SYS_print` syscalls by stalling when all 4 I/O registers are full.

**New ports added to `Processor` and `Computer`:**

```verilog
output        io_stall,         // Processor needs to stall (I/O regs full)
input         copied_io_regs,   // Environment signals I/O regs have been read
input  [1:0]  io_reg_index      // Selects which I/O register to read after halt
```

### Test Program (Lab 8)

```c
int i, x = 10, N = 30;
for (i = 0; i < N; i++) {
    if ((i & 0x1) == 0) { x += f(x, i); print x; }
    else                 { x -= f(x, i); print x; }
}
exit();
int f(int a, int b) { return a + b; }
```

---

## Lab 9 — Load/Store, `lui`, and Keyboard Input

**Goal:** Add data memory (load/store) support, the `lui` instruction, and keyboard I/O via a new system call.

### New Instructions

| Category | Instructions | Opcode |
|----------|-------------|--------|
| **Upper Immediate** | `lui` | 0xF |
| **Word Load/Store** | `lw`, `sw` | 0x23, 0x2B |
| **Byte Load/Store** | `lb`, `lbu`, `sb` | 0x20, 0x24, 0x28 |
| **Half-word Load/Store** | `lh`, `lhu`, `sh` | 0x21, 0x25, 0x29 |

> Memory is **big-endian**. Stores to `sb` and `sh` use **Read-Modify-Write** to update only the target bytes.

### Data Memory

- **Size:** 4KB byte-addressable RAM, implemented as four 1KB blocks.
- **Access:** 32-bit word-aligned reads/writes; byte and half-word accesses use bit masking.

**New ports added to `Processor`:**

```verilog
output [11:0] dmem_addr,         // Data memory address (12-bit, byte-addressed)
output [31:0] dmem_write_data,   // Data to write to memory
output [3:0]  dmem_write_enable, // Byte-level write enable (one bit per byte)
output        dmem_read_enable,  // High when a load instruction is executing
input  [31:0] dmem_read_data,    // Data read from memory
```

### Keyboard Input (`SYS_read`)

A new syscall (`syscall number = 32'h0003` / `32'd1003`) enables reading a 32-bit value from the keyboard.

- The processor asserts `waiting_for_input` and **stalls in Cycle 2**.
- When the environment sets `input_value_valid = 1`, the processor accepts `input_value`, writes it to `regfile[rd]`, and continues in Cycle 3.

**New ports:**

```verilog
output        waiting_for_input,  // Processor is waiting for keyboard input
input  [31:0] input_value,        // 32-bit value from keyboard/environment
input         input_value_valid   // High when input_value is ready
```

### Test Programs (Lab 9)

| # | Description |
|---|-------------|
| 1 | Sum loop — takes inputs `x` and `N`, prints intermediate sums. |
| 2 | String assembly — writes "Hello World!\n" byte-by-byte to memory, prints with `lb`. |
| 3 | Array manipulation — uses `sh`/`lh` for short integers; includes bounds and alignment checks. |
| 4 | Recursive `sum(n)` — tests stack operations; initialize `$sp = 1024`. |

---

## Module Summary

| Module | File | Description |
|--------|------|-------------|
| `Processor` | `Processor.v` | Top-level 3-cycle MIPS processor |
| `ALU` | `ALU.v` | All arithmetic, logic, shift, compare operations |
| `RegisterFile` | `RegisterFile.v` | 32 × 32-bit general-purpose registers |
| `Memory` | `Memory.v` | 4KB byte-addressable data memory |
| `Computer` | `Computer.v` | Wrapper: Processor + Instruction Memory |

---

## Deliverables

- Complete Vivado/Vitis project directory (**excluding the `runs/` folder**).
- This `README.md`.
- Testbenches demonstrating simulation results for all test programs.

---

## References

- D. A. Patterson and J. L. Hennessy, *Computer Organization and Design: The Hardware/Software Interface*, 6th Ed., Morgan Kaufmann.
- S. Palnitkar, *Verilog HDL: A Guide to Digital Design and Synthesis*, Pearson India.
- [PYNQ-Z2 User Manual](https://www.cse.iitk.ac.in/users/mainakc/2026Spring/lec220/pynqz2_user_manual_v1_0.pdf)
- [Zynq Z-7000 Datasheet](https://www.cse.iitk.ac.in/users/mainakc/2026Spring/lec220/ds190-Zynq-7000-Overview.pdf)
- [CS220 Course Page — IIT Kanpur](https://www.cse.iitk.ac.in/users/mainakc/2026Spring/lectures220.html)
