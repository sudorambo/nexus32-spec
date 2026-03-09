# NEXUS-32 Fantasy Game Console — Hardware & Ecosystem Specification

**Version:** 1.0 Draft  
**Date:** March 8, 2026  
**Classification:** Engineering Specification — Single Source of Truth

---

## Table of Contents

1. [System Overview & Architecture Summary](#1-system-overview--architecture-summary)
2. [CPU — Instruction Set Architecture](#2-cpu--instruction-set-architecture)
3. [Memory Architecture](#3-memory-architecture)
4. [DMA Controller](#4-dma-controller)
5. [Virtual GPU — Command Model & Rendering](#5-virtual-gpu--command-model--rendering)
6. [Audio Processing Unit](#6-audio-processing-unit)
7. [Input System](#7-input-system)
8. [Timer & System Registers](#8-timer--system-registers)
9. [ROM Image Format](#9-rom-image-format)
10. [Game SDK Specification](#10-game-sdk-specification)
11. [ROM Packaging Toolchain](#11-rom-packaging-toolchain)
12. [Emulator Architecture Notes](#12-emulator-architecture-notes)
13. [Multi-Repository Project Structure](#13-multi-repository-project-structure)

---

## 1. System Overview & Architecture Summary

### 1.1 High-Level Architecture

The NEXUS-32 is a fantasy game console whose architecture is inspired by the complexity and developer experience of the PS2 era, but designed from scratch with clean interfaces and a modern host rendering backend (Vulkan). The console defines a virtual machine with the following major components:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        NEXUS-32 System Bus                         │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │   NX-RISC    │  │   Vector     │  │    Memory Controller     │  │
│  │   CPU Core   │──│   Unit (VU)  │──│  ┌────────┐ ┌────────┐  │  │
│  │  32-bit RISC │  │ 16×128-bit   │  │  │Main RAM│ │  VRAM  │  │  │
│  │  r0–r31      │  │ v0–v15       │  │  │ 32 MB  │ │ 16 MB  │  │  │
│  └──────┬───────┘  └──────────────┘  │  └────────┘ └────────┘  │  │
│         │                            │  ┌────────┐ ┌────────┐  │  │
│  ┌──────┴───────┐  ┌──────────────┐  │  │AudioRAM│ │ EEPROM │  │  │
│  │   Interrupt   │  │     DMA      │  │  │  8 MB  │ │ 64 KB  │  │  │
│  │   Controller  │  │  Controller  │──│  └────────┘ └────────┘  │  │
│  │  8 IRQ lines  │  │  8 Channels  │  └──────────────────────────┘  │
│  └──────────────┘  └──────────────┘                                 │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │   Virtual     │  │   Audio      │  │    Input Controller      │  │
│  │   GPU (VGP)   │  │   Processing │  │    4 controllers         │  │
│  │   Cmd Buffer  │  │   Unit (APU) │  │    Memory-mapped         │  │
│  │   2 MB/frame  │  │  48 voices   │  │    state                 │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐                                 │
│  │   Timer       │  │   System     │                                │
│  │   2 hw timers │  │   Registers  │                                │
│  │   Frame ctr   │  │   RNG, info  │                                │
│  └──────────────┘  └──────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Interconnection

The **NX-RISC CPU Core** is the central processing unit, a custom 32-bit RISC processor with 32 general-purpose integer registers and a program counter. It is tightly coupled with the **Vector Unit (VU)**, which provides 16 128-bit SIMD registers for floating-point math (geometry transforms, physics, interpolation). The CPU and VU share the same instruction stream; vector instructions are issued inline alongside integer instructions.

The **Memory Controller** arbitrates access to four distinct memory regions: Main RAM (32 MB), VRAM (16 MB), Audio RAM (8 MB), and Save EEPROM (64 KB). All regions are memory-mapped into a unified 32-bit address space. The controller handles bus arbitration between the CPU, DMA controller, and GPU.

The **DMA Controller** provides 8 independent channels for bulk data transfer between any memory regions. Its primary role is moving texture data from Main RAM into VRAM, streaming audio samples into Audio RAM, and performing VRAM-to-VRAM copies. DMA transfers run concurrently with CPU execution; bus contention is modeled as stall cycles on the CPU when both access the same memory bank simultaneously.

The **Virtual GPU (VGP)** does not execute in parallel with the CPU. Instead, the game fills a command buffer (up to 2 MB) at a fixed memory-mapped address during each frame. Commands describe the full rendering workload: clear operations, texture binds, render state changes, draw calls (triangles, indexed geometry, sprites), shader binds, uniform uploads, and a final PRESENT command. When the emulator processes the frame, it reads this buffer sequentially and translates each command into Vulkan API calls. The game code never calls Vulkan; it only writes binary command structs to the buffer.

The **Audio Processing Unit (APU)** is a 48-voice sample-based synthesizer. Each voice reads PCM or ADPCM sample data from Audio RAM and mixes into a stereo ring buffer in Main RAM at 48 kHz. The APU runs asynchronously; it raises an interrupt when the ring buffer is half-consumed, signaling the game to update voice parameters or trigger new sounds.

The **Input Controller** maintains a memory-mapped read-only region containing the current state of up to 4 game controllers. The emulator writes this region once per frame (before VBlank) by mapping host input devices (keyboard, gamepad) to the virtual controller layout.

The **Timer system** provides 2 programmable countdown timers and a monotonic frame counter. Timers can generate interrupts on overflow and are used for gameplay timing, animation, and profiling.

### 1.3 Typical Frame Data Flow

A typical frame proceeds as follows:

1. **VBlank interrupt fires.** The frame counter increments. Input state is updated. The CPU jumps to the VBlank handler, which typically signals the game loop to begin a new frame.
2. **Game logic (CPU).** The game reads input, updates AI, physics, and world state. This is pure integer and vector computation against Main RAM.
3. **Scene submission.** The game iterates over visible objects and writes GPU commands to the command buffer: bind textures, set render state, issue draw calls. Vertex data and textures have been pre-loaded into VRAM (via DMA, usually during loading screens or streaming).
4. **PRESENT.** The game writes the PRESENT command and writes to the GPU control register to signal frame completion.
5. **Emulator picks up.** The host emulator reads the command buffer, translates each command to Vulkan calls (creating/caching VkImage handles for textures, compiling mini-shader programs to SPIR-V, issuing vkCmdDraw calls), renders to a Vulkan swapchain image, and presents.
6. **Audio mixing.** Concurrently, the APU fills the audio ring buffer. The emulator drains it into the host audio system.

### 1.4 The Contract: Fantasy Hardware vs. Host Emulator

The game sees a deterministic virtual machine: a fixed memory map, a fixed CPU cycle budget per frame, predictable DMA transfer times, and a GPU that faithfully executes every command in the buffer. There are no race conditions, no variable clock speeds, and no cache hierarchies to worry about.

The host emulator provides the illusion. It interprets CPU instructions (or JIT-compiles them), processes DMA descriptors, translates the GPU command buffer to Vulkan, and mixes audio. The emulator is free to use any host-side optimization (multithreading, resolution scaling, texture filtering upgrades, MSAA) as long as the game observes correct behavior. The contract is: the game writes to defined memory regions and registers; the emulator faithfully implements the observable semantics.

---

## 2. CPU — Instruction Set Architecture

### 2.1 Register File

**Integer Registers:** 32 general-purpose 32-bit registers.

| Register | Name  | Convention                                             |
| -------- | ----- | ------------------------------------------------------ |
| r0       | zero  | Hardwired to 0. Writes are discarded.                  |
| r1       | at    | Assembler temporary. Reserved for pseudo-instructions. |
| r2–r3    | v0–v1 | Function return values.                                |
| r4–r7    | a0–a3 | Function arguments (first 4).                          |
| r8–r15   | t0–t7 | Temporary registers. Caller-saved.                     |
| r16–r23  | s0–s7 | Saved registers. Callee-saved.                         |
| r24–r25  | t8–t9 | Additional temporaries. Caller-saved.                  |
| r26–r27  | k0–k1 | Reserved for interrupt handlers.                       |
| r28      | gp    | Global pointer (for accessing global data).            |
| r29      | fp    | Frame pointer. Callee-saved.                           |
| r30      | sp    | Stack pointer. Callee-saved.                           |
| r31      | ra    | Return address / link register.                        |

**Vector Registers:** 16 registers, each 128 bits wide (four 32-bit IEEE 754 single-precision floats). Named v0–v15. Components are accessed as .x, .y, .z, .w (indices 0–3).

**Special Registers:**

| Register | Width  | Description                                         |
| -------- | ------ | --------------------------------------------------- |
| PC       | 32-bit | Program counter.                                    |
| SR       | 32-bit | Status register.                                    |
| EPC      | 32-bit | Exception program counter (saved PC on interrupt).  |
| CAUSE    | 32-bit | Interrupt/exception cause register.                 |
| VFCC     | 16-bit | Vector flag/condition code register (1 bit per VR). |

**Status Register (SR) bit layout:**

| Bits  | Name | Description                               |
| ----- | ---- | ----------------------------------------- |
| 0     | Z    | Zero flag                                 |
| 1     | N    | Negative flag                             |
| 2     | C    | Carry flag                                |
| 3     | V    | Overflow flag                             |
| 4     | IE   | Interrupt enable (global)                 |
| 5–12  | IM   | Interrupt mask (8 bits, one per IRQ line) |
| 13–15 | —    | Reserved                                  |
| 16–31 | —    | Reserved                                  |

### 2.2 Instruction Encoding

All instructions are fixed-width 32 bits. Five encoding formats:

**R-Type (Register-Register):**

```
 31    26 25   21 20   16 15   11 10    6 5      0
┌────────┬───────┬───────┬───────┬───────┬────────┐
│ opcode │  rs   │  rt   │  rd   │ shamt │  func  │
│  6 bit │ 5 bit │ 5 bit │ 5 bit │ 5 bit │  6 bit │
└────────┴───────┴───────┴───────┴───────┴────────┘
```

**I-Type (Immediate):**

```
 31    26 25   21 20   16 15                     0
┌────────┬───────┬───────┬───────────────────────┐
│ opcode │  rs   │  rt   │     immediate (16)     │
│  6 bit │ 5 bit │ 5 bit │        16 bit          │
└────────┴───────┴───────┴───────────────────────┘
```

**J-Type (Jump):**

```
 31    26 25                                     0
┌────────┬───────────────────────────────────────┐
│ opcode │          target address (26)           │
│  6 bit │              26 bit                    │
└────────┴───────────────────────────────────────┘
```

The effective jump target is `{ PC[31:28], target[25:0], 2'b00 }` — the upper 4 bits of the current PC concatenated with the 26-bit target shifted left by 2 (word-aligned).

**V-Type (Vector):**

```
 31    26 25   22 21   18 17   14 13   10 9     4 3    0
┌────────┬───────┬───────┬───────┬───────┬───────┬──────┐
│ opcode │  vs   │  vt   │  vd   │ elem  │ vfunc │ flag │
│  6 bit │ 4 bit │ 4 bit │ 4 bit │ 4 bit │ 6 bit │ 4 bit│
└────────┴───────┴───────┴───────┴───────┴───────┴──────┘
```

- `vs`, `vt`, `vd`: source and destination vector registers (4-bit, 0–15).
- `elem`: element selector for swizzle/broadcast operations.
- `vfunc`: vector function code.
- `flag`: modifier flags (e.g., write mask per component).

**S-Type (System):**

```
 31    26 25   21 20                              0
┌────────┬───────┬───────────────────────────────┐
│ opcode │  func │         operand (21)           │
│  6 bit │ 5 bit │           21 bit               │
└────────┴───────┴───────────────────────────────┘
```

### 2.3 Integer Instruction Set

Primary opcode `0x00` indicates R-type; the `func` field selects the operation.

**Arithmetic (R-Type, opcode 0x00):**

| Mnemonic | func | Format     | Description                     | Cycles |
| -------- | ---- | ---------- | ------------------------------- | ------ |
| ADD      | 0x20 | rd, rs, rt | rd = rs + rt (trap on overflow) | 1      |
| ADDU     | 0x21 | rd, rs, rt | rd = rs + rt (no trap)          | 1      |
| SUB      | 0x22 | rd, rs, rt | rd = rs - rt (trap on overflow) | 1      |
| SUBU     | 0x23 | rd, rs, rt | rd = rs - rt (no trap)          | 1      |
| MUL      | 0x18 | rd, rs, rt | rd = (rs × rt)[31:0]            | 2      |
| MULH     | 0x19 | rd, rs, rt | rd = (rs × rt)[63:32] (signed)  | 2      |
| DIV      | 0x1A | rd, rs, rt | rd = rs / rt (signed)           | 8      |
| DIVU     | 0x1B | rd, rs, rt | rd = rs / rt (unsigned)         | 8      |
| MOD      | 0x1C | rd, rs, rt | rd = rs % rt (signed)           | 8      |

**Multi-cycle instruction behavior:** DIV, DIVU, MOD, MUL, and MULH are **fully interlocked stalling instructions**. The CPU pipeline stalls for the full cycle cost; no subsequent instruction executes until the result is written to `rd`. This differs from real MIPS hardware (where HI/LO are written asynchronously). The NEXUS-32 trades theoretical throughput for simplicity — there are no separate HI/LO registers, no MFHI/MFLO hazards, and no need for the compiler to schedule instructions around pending divides. The result is directly available in `rd` when the stall completes. Division by zero writes `0xFFFFFFFF` to `rd` and sets the overflow flag (V) in the status register; it does not trap.

**Logical (R-Type, opcode 0x00):**

| Mnemonic | func | Format        | Description                     | Cycles |
| -------- | ---- | ------------- | ------------------------------- | ------ |
| AND      | 0x24 | rd, rs, rt    | rd = rs & rt                    | 1      |
| OR       | 0x25 | rd, rs, rt    | rd = rs \| rt                   | 1      |
| XOR      | 0x26 | rd, rs, rt    | rd = rs ^ rt                    | 1      |
| NOR      | 0x27 | rd, rs, rt    | rd = ~(rs \| rt)                | 1      |
| SLL      | 0x00 | rd, rt, shamt | rd = rt << shamt                | 1      |
| SRL      | 0x02 | rd, rt, shamt | rd = rt >> shamt (logical)      | 1      |
| SRA      | 0x03 | rd, rt, shamt | rd = rt >> shamt (arithmetic)   | 1      |
| SLLV     | 0x04 | rd, rt, rs    | rd = rt << rs[4:0]              | 1      |
| SRLV     | 0x06 | rd, rt, rs    | rd = rt >> rs[4:0] (logical)    | 1      |
| SRAV     | 0x07 | rd, rt, rs    | rd = rt >> rs[4:0] (arithmetic) | 1      |

**Comparison (R-Type, opcode 0x00):**

| Mnemonic | func | Format     | Description                       | Cycles |
| -------- | ---- | ---------- | --------------------------------- | ------ |
| SLT      | 0x2A | rd, rs, rt | rd = (rs < rt) ? 1 : 0 (signed)   | 1      |
| SLTU     | 0x2B | rd, rs, rt | rd = (rs < rt) ? 1 : 0 (unsigned) | 1      |

**Immediate Arithmetic/Logical (I-Type):**

| Mnemonic | opcode | Format        | Description                         | Cycles |
| -------- | ------ | ------------- | ----------------------------------- | ------ |
| ADDI     | 0x08   | rt, rs, imm16 | rt = rs + sign_ext(imm16)           | 1      |
| ADDIU    | 0x09   | rt, rs, imm16 | rt = rs + sign_ext(imm16) (no trap) | 1      |
| ANDI     | 0x0C   | rt, rs, imm16 | rt = rs & zero_ext(imm16)           | 1      |
| ORI      | 0x0D   | rt, rs, imm16 | rt = rs \| zero_ext(imm16)          | 1      |
| XORI     | 0x0E   | rt, rs, imm16 | rt = rs ^ zero_ext(imm16)           | 1      |
| LUI      | 0x0F   | rt, imm16     | rt = imm16 << 16                    | 1      |
| SLTI     | 0x0A   | rt, rs, imm16 | rt = (rs < sign_ext(imm16)) ? 1:0   | 1      |
| SLTIU    | 0x0B   | rt, rs, imm16 | rt = unsigned compare               | 1      |

**Branch (I-Type):**

| Mnemonic | opcode | Format         | Description                 | Cycles         |
| -------- | ------ | -------------- | --------------------------- | -------------- |
| BEQ      | 0x04   | rs, rt, offset | Branch if rs == rt          | 2 taken, 1 not |
| BNE      | 0x05   | rs, rt, offset | Branch if rs != rt          | 2 taken, 1 not |
| BLT      | 0x14   | rs, rt, offset | Branch if rs < rt (signed)  | 2 taken, 1 not |
| BGT      | 0x15   | rs, rt, offset | Branch if rs > rt (signed)  | 2 taken, 1 not |
| BLE      | 0x16   | rs, rt, offset | Branch if rs <= rt (signed) | 2 taken, 1 not |
| BGE      | 0x17   | rs, rt, offset | Branch if rs >= rt (signed) | 2 taken, 1 not |

Branch target: `PC + 4 + (sign_ext(offset) << 2)`.

**Branch delay slots:** The NEXUS-32 CPU does **not** utilize branch delay slots. When a branch is taken, the instruction immediately following the branch in memory is **not** executed. The PC is updated to the branch target before the next fetch. This simplifies compiler code generation and hand-written assembly. The 2-cycle cost for taken branches accounts for the pipeline flush.

**Jump (J-Type):**

| Mnemonic | opcode | Format | Description                                | Cycles |
| -------- | ------ | ------ | ------------------------------------------ | ------ |
| J        | 0x02   | target | PC = {PC[31:28], target, 00}               | 2      |
| JAL      | 0x03   | target | r31 = PC + 4; PC = {PC[31:28], target, 00} | 2      |

**Jump Register (R-Type, opcode 0x00):**

| Mnemonic | func | Format | Description          | Cycles |
| -------- | ---- | ------ | -------------------- | ------ |
| JR       | 0x08 | rs     | PC = rs              | 2      |
| JALR     | 0x09 | rd, rs | rd = PC + 4; PC = rs | 2      |

**Load/Store (I-Type):**

| Mnemonic | opcode | Format         | Description                     | Cycles |
| -------- | ------ | -------------- | ------------------------------- | ------ |
| LB       | 0x20   | rt, offset(rs) | rt = sign_ext(mem8[rs+offset])  | 3      |
| LBU      | 0x24   | rt, offset(rs) | rt = zero_ext(mem8[rs+offset])  | 3      |
| LH       | 0x21   | rt, offset(rs) | rt = sign_ext(mem16[rs+offset]) | 3      |
| LHU      | 0x25   | rt, offset(rs) | rt = zero_ext(mem16[rs+offset]) | 3      |
| LW       | 0x23   | rt, offset(rs) | rt = mem32[rs+offset]           | 3      |
| SB       | 0x28   | rt, offset(rs) | mem8[rs+offset] = rt[7:0]       | 3      |
| SH       | 0x29   | rt, offset(rs) | mem16[rs+offset] = rt[15:0]     | 3      |
| SW       | 0x2B   | rt, offset(rs) | mem32[rs+offset] = rt           | 3      |

All load/store addresses are computed as `rs + sign_ext(offset)`. Word loads (LW) and stores (SW) require 4-byte alignment. Half-word (LH/SH) require 2-byte alignment. Unaligned access triggers an address alignment exception.

**System (S-Type, opcode 0x3F):**

| Mnemonic | func | Description                                    | Cycles |
| -------- | ---- | ---------------------------------------------- | ------ |
| SYSCALL  | 0x00 | Trigger system call exception.                 | 4      |
| BREAK    | 0x01 | Trigger breakpoint exception (debug).          | 1      |
| NOP      | 0x02 | No operation. (Encoded as `SLL r0, r0, 0`.)    | 1      |
| HALT     | 0x03 | Halt CPU until next interrupt.                 | 1      |
| ERET     | 0x04 | Return from exception (PC = EPC, restore SR).  | 2      |
| MFC0     | 0x10 | rt = special_reg[rd] (move from coprocessor 0) | 1      |
| MTC0     | 0x11 | special_reg[rd] = rt (move to coprocessor 0)   | 1      |

### 2.4 Vector Instruction Set

All vector instructions use V-Type encoding. Primary opcode: `0x3E`.

| Mnemonic | vfunc | Format         | Description                                        | Cycles |
| -------- | ----- | -------------- | -------------------------------------------------- | ------ |
| VADD     | 0x00  | vd, vs, vt     | vd = vs + vt (component-wise)                      | 2      |
| VSUB     | 0x01  | vd, vs, vt     | vd = vs - vt (component-wise)                      | 2      |
| VMUL     | 0x02  | vd, vs, vt     | vd = vs × vt (component-wise)                      | 3      |
| VDIV     | 0x03  | vd, vs, vt     | vd = vs / vt (component-wise)                      | 8      |
| VDOT     | 0x04  | vd, vs, vt     | vd.x = dot(vs, vt), vd.yzw = 0                     | 3      |
| VCROSS   | 0x05  | vd, vs, vt     | vd.xyz = cross(vs.xyz, vt.xyz), vd.w = 0           | 4      |
| VNORM    | 0x06  | vd, vs         | vd = normalize(vs) (vs / length(vs))               | 6      |
| VLERP    | 0x07  | vd, vs, vt     | vd = lerp(vs, vt, vd.x) — vd.x is the factor       | 4      |
| VSWIZZLE | 0x08  | vd, vs, elem   | vd = swizzle vs by elem pattern                    | 2      |
| VMOV     | 0x09  | vd, vs         | vd = vs                                            | 1      |
| VLW      | 0x0A  | vd, offset(rs) | vd = mem128[rs + sign_ext(offset)]                 | 4      |
| VSW      | 0x0B  | vs, offset(rs) | mem128[rs + sign_ext(offset)] = vs                 | 4      |
| VCMP     | 0x0C  | vs, vt         | VFCC = component-wise comparison flags             | 2      |
| VMIN     | 0x0D  | vd, vs, vt     | vd = min(vs, vt) (component-wise)                  | 2      |
| VMAX     | 0x0E  | vd, vs, vt     | vd = max(vs, vt) (component-wise)                  | 2      |
| VMAD     | 0x0F  | vd, vs, vt     | vd = vd + (vs × vt) (multiply-add, component-wise) | 3      |
| VRSQ     | 0x10  | vd, vs         | vd = 1/sqrt(vs) (component-wise)                   | 4      |
| VRCP     | 0x11  | vd, vs         | vd = 1/vs (component-wise)                         | 4      |
| VABS     | 0x12  | vd, vs         | vd = abs(vs) (component-wise)                      | 2      |
| VNEG     | 0x13  | vd, vs         | vd = -vs (component-wise)                          | 2      |
| VCLAMP   | 0x14  | vd, vs, vt     | vd = clamp(vd, vs, vt) (min=vs, max=vt)            | 2      |
| VITOF    | 0x15  | vd, rs         | vd = float4(rs, rs, rs, rs) — broadcast int to vec | 2      |
| VFTOI    | 0x16  | rd, vs         | rd = int(vs.x) — truncate first component          | 2      |

**VSWIZZLE encoding:** The `elem` field (4 bits) encodes common swizzle patterns:

| elem    | Pattern | Description           |
| ------- | ------- | --------------------- |
| 0x0     | xyzw    | Identity (no swizzle) |
| 0x1     | xxxx    | Broadcast X           |
| 0x2     | yyyy    | Broadcast Y           |
| 0x3     | zzzz    | Broadcast Z           |
| 0x4     | wwww    | Broadcast W           |
| 0x5     | wzyx    | Reverse               |
| 0x6     | xxyy    | Paired broadcast      |
| 0x7     | zzww    | Paired broadcast      |
| 0x8     | xyxy    | XY repeat             |
| 0x9     | zwzw    | ZW repeat             |
| 0xA     | yxwz    | Swap pairs            |
| 0xB–0xF | —       | Reserved              |

**Vector load/store:** VLW and VSW use a hybrid encoding. The base address register `rs` comes from the integer register file (5 bits borrowed from the `vs` and `elem` fields). The offset is encoded in the `flag` and lower `vfunc` bits, providing a 7-bit signed offset scaled by 16 (the vector size), giving a range of ±1024 bytes from the base.

### 2.5 Cycle Budget Model

The NEXUS-32 does not emulate a real clock frequency. Instead, it defines a **cycle budget per frame**. At 60 FPS, the default budget is **3,000,000 cycles per frame** (3M). This is configurable per-ROM in the header between 2,000,000 (2M) and 4,000,000 (4M).

**Cycle costs by instruction class:**

| Category           | Instructions                   | Cycles |
| ------------------ | ------------------------------ | ------ |
| Integer ALU        | ADD, SUB, AND, OR, XOR, shifts | 1      |
| Integer Multiply   | MUL, MULH                      | 2      |
| Integer Divide     | DIV, DIVU, MOD                 | 8      |
| Comparison         | SLT, SLTU, SLTI                | 1      |
| Branch (not taken) | BEQ, BNE, etc.                 | 1      |
| Branch (taken)     | BEQ, BNE, etc.                 | 2      |
| Jump               | J, JAL, JR, JALR               | 2      |
| Load/Store         | LB, LH, LW, SB, SH, SW         | 3      |
| Vector ALU         | VADD, VSUB, VMIN, VMAX, etc.   | 2      |
| Vector Multiply    | VMUL, VMAD                     | 3      |
| Vector Complex     | VCROSS, VLERP, VRSQ, VRCP      | 4      |
| Vector Normalize   | VNORM                          | 6      |
| Vector Divide      | VDIV                           | 8      |
| Vector Load/Store  | VLW, VSW                       | 4      |
| System             | SYSCALL                        | 4      |
| System             | ERET                           | 2      |

**Bus contention:** When the CPU issues a load/store to a memory bank that the DMA controller is also accessing, an additional 2-cycle stall is added. Consecutive loads to the same bank incur no extra penalty (pipelined).

**Practical implications:** At 3M cycles/frame, assuming an average CPI of ~2.0 (mix of ALU, loads, branches), a developer can execute roughly 1.5 million instructions per frame. This is sufficient for:

- A 3D game with ~200 objects doing basic AI and physics (~300K cycles for game logic)
- Transform and lighting for ~10,000 vertices (~600K cycles for vector math)
- Command buffer construction for ~500 draw calls (~200K cycles)
- Audio management and misc overhead (~100K cycles)
- Leaving ~1.8M cycles of headroom for complex effects, particle systems, etc.

### 2.6 Interrupt Model

The NEXUS-32 supports 8 interrupt request lines (IRQ 0–7) with fixed priorities. Higher IRQ numbers have higher priority.

| IRQ | Source           | Priority | Description                                  |
| --- | ---------------- | -------- | -------------------------------------------- |
| 0   | Timer 0 overflow | Lowest   | Programmable timer 0 expired.                |
| 1   | Timer 1 overflow |          | Programmable timer 1 expired.                |
| 2   | Input poll       |          | Controller state updated (once per frame).   |
| 3   | DMA completion   |          | Any DMA channel completed transfer.          |
| 4   | Audio buffer     |          | APU ring buffer half-empty, needs refill.    |
| 5   | HBlank           |          | Horizontal blank (per-scanline, if enabled). |
| 6   | VBlank           |          | Vertical blank (once per frame).             |
| 7   | System/Debug     | Highest  | Breakpoint hit or system exception.          |

**Interrupt Vector Table (IVT):** Located at address `0x00000000` in Main RAM. Each entry is 4 bytes (an address to the handler). The table occupies the first 32 bytes of memory (8 entries × 4 bytes).

| Address    | Content               |
| ---------- | --------------------- |
| 0x00000000 | IRQ 0 handler address |
| 0x00000004 | IRQ 1 handler address |
| 0x00000008 | IRQ 2 handler address |
| 0x0000000C | IRQ 3 handler address |
| 0x00000010 | IRQ 4 handler address |
| 0x00000014 | IRQ 5 handler address |
| 0x00000018 | IRQ 6 handler address |
| 0x0000001C | IRQ 7 handler address |

When an interrupt is taken: the current PC is saved to EPC, the cause is written to CAUSE, the IE bit in SR is cleared (disabling further interrupts), and the CPU jumps to the handler address from the IVT. The handler must execute ERET to restore PC from EPC and re-enable interrupts.

Interrupts are masked per-line via the IM field in SR. Setting bit `5+N` masks IRQ N. An interrupt is delivered only if IE is set and the corresponding IM bit is clear.

---

## 3. Memory Architecture

### 3.1 Memory Map

The NEXUS-32 has a 32-bit address space (4 GB theoretical). Only the following regions are backed by physical memory or I/O:

| Start Address | End Address | Size  | Region                  | Access      |
| ------------- | ----------- | ----- | ----------------------- | ----------- |
| 0x00000000    | 0x01FFFFFF  | 32 MB | Main RAM                | Read/Write  |
| 0x04000000    | 0x04FFFFFF  | 16 MB | VRAM                    | Read/Write* |
| 0x06000000    | 0x067FFFFF  | 8 MB  | Audio RAM               | Read/Write* |
| 0x08000000    | 0x0800FFFF  | 64 KB | Save EEPROM             | Read/Write  |
| 0x0A000000    | 0x0A1FFFFF  | 2 MB  | GPU Command Buffer      | Write-only  |
| 0x0B000000    | 0x0B0000FF  | 256 B | GPU Control/Status Regs | Read/Write  |
| 0x0B001000    | 0x0B0010FF  | 256 B | DMA Control Registers   | Read/Write  |
| 0x0B002000    | 0x0B002FFF  | 4 KB  | DMA Descriptor Table    | Read/Write  |
| 0x0B003000    | 0x0B003FFF  | 4 KB  | Audio Registers         | Read/Write  |
| 0x0B004000    | 0x0B00407F  | 128 B | Input Controller State  | Read-only   |
| 0x0B005000    | 0x0B00503F  | 64 B  | Timer Registers         | Read/Write  |
| 0x0B006000    | 0x0B00603F  | 64 B  | System Registers        | Read/Write  |
| 0x0B007000    | 0x0B00701F  | 32 B  | Interrupt Control       | Read/Write  |

All other addresses return 0 on read and silently discard writes.

*VRAM and Audio RAM are accessible by the CPU for initialization and debugging, but primary data transfer should use DMA channels. Direct CPU writes to VRAM are throttled: each write costs 4 cycles instead of the normal 3. This penalty is an intentional architectural constraint — not a model of real-time bus contention (since the Virtual GPU processes the command buffer only after PRESENT, it does not compete with the CPU for VRAM access during the frame). The extra cycle cost exists to discourage developers from using the CPU as a bulk data path into VRAM and to reward proper use of DMA channels, which transfer at 64 bytes/cycle without consuming CPU cycles. Reads from VRAM by the CPU are similarly penalized at 4 cycles.

### 3.2 Main RAM Layout

The 32 MB of Main RAM (0x00000000–0x01FFFFFF) is partitioned by convention (enforced by the SDK linker, not hardware):

| Address Range           | Size  | Purpose                             |
| ----------------------- | ----- | ----------------------------------- |
| 0x00000000 – 0x0000001F | 32 B  | Interrupt Vector Table              |
| 0x00000020 – 0x000003FF | ~1 KB | System reserved (boot params)       |
| 0x00000400 – 0x003FFFFF | ~4 MB | Code segment (ROM code loaded here) |
| 0x00400000 – 0x007FFFFF | 4 MB  | Data segment (initialized data)     |
| 0x00800000 – 0x017FFFFF | 16 MB | Heap (dynamic allocation)           |
| 0x01800000 – 0x01FEFFFF | ~8 MB | Audio ring buffer + scratch         |
| 0x01FF0000 – 0x01FFFFFF | 64 KB | Stack (grows downward from top)     |

Default stack pointer (SP): `0x01FFFFFF` (top of Main RAM, aligned to 16 bytes: `0x01FFFFF0`).

Audio ring buffer base: `0x01800000`, size: 64 KB (sufficient for ~170 ms of 48 kHz stereo 16-bit audio). The APU writes to this buffer and the emulator reads from it.

### 3.3 VRAM Layout

VRAM is 16 MB at `0x04000000–0x04FFFFFF`. It stores textures, framebuffers, vertex buffers, index buffers, and shader programs. The layout is not fixed by the hardware; the game and SDK manage VRAM allocation. Recommended layout:

| Offset Range (from 0x04000000) | Size    | Suggested Purpose                                                                                                            |
| ------------------------------ | ------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 0x000000 – 0x4AFFFF            | ~4.7 MB | Framebuffer 0 (640×480 RGBA8, 1.2MB) + Framebuffer 1 (double buffer) + Depth buffer (640×480 16-bit, 600KB) + Render targets |
| 0x4B0000 – 0xBFFFFF            | ~7.3 MB | Texture storage                                                                                                              |
| 0xC00000 – 0xEFFFFF            | 3 MB    | Vertex and index buffers                                                                                                     |
| 0xF00000 – 0xF7FFFF            | 512 KB  | Shader program storage                                                                                                       |
| 0xF80000 – 0xFFFFFF            | 512 KB  | Reserved / scratch                                                                                                           |

**Framebuffer:** The default framebuffer is 640×480 at RGBA8888 (4 bytes/pixel) = 1,228,800 bytes (~1.2 MB). The depth buffer at 16-bit = 614,400 bytes (~600 KB). Double buffering doubles the framebuffer cost. Total default FB usage: ~3 MB.

### 3.4 Memory-Mapped I/O Details

**GPU Control/Status Registers (0x0B000000–0x0B0000FF):**

| Offset | Name           | R/W | Description                                                        |
| ------ | -------------- | --- | ------------------------------------------------------------------ |
| 0x00   | GPU_STATUS     | R   | Bit 0: busy (1 = processing command buffer). Bit 1: VBlank active. |
| 0x04   | GPU_CONTROL    | W   | Bit 0: submit frame (write 1 to signal PRESENT). Bit 1: reset GPU. |
| 0x08   | GPU_CB_ADDR    | R/W | Command buffer start address (default: 0x0A000000).                |
| 0x0C   | GPU_CB_SIZE    | R/W | Number of bytes written to command buffer this frame.              |
| 0x10   | GPU_FB_ADDR    | R/W | Framebuffer 0 base address in VRAM.                                |
| 0x14   | GPU_FB1_ADDR   | R/W | Framebuffer 1 base address (for double buffering).                 |
| 0x18   | GPU_DB_ADDR    | R/W | Depth buffer base address in VRAM.                                 |
| 0x1C   | GPU_WIDTH      | R/W | Framebuffer width (default: 640).                                  |
| 0x20   | GPU_HEIGHT     | R/W | Framebuffer height (default: 480).                                 |
| 0x24   | GPU_DRAW_CALLS | R   | Number of draw calls processed last frame (profiling).             |
| 0x28   | GPU_VRAM_USED  | R   | VRAM bytes used for textures+shaders (profiling).                  |

### 3.5 Endianness and Alignment

The NEXUS-32 is **little-endian**. Multi-byte values are stored least-significant byte first.

**Alignment requirements:**

| Access Type | Required Alignment | Penalty for Unaligned    |
| ----------- | ------------------ | ------------------------ |
| LB / SB     | None               | N/A                      |
| LH / SH     | 2-byte             | Address exception (trap) |
| LW / SW     | 4-byte             | Address exception (trap) |
| VLW / VSW   | 16-byte            | Address exception (trap) |

Unaligned access is a hard fault, not a performance penalty. The SDK library provides unaligned access helper functions for the rare cases where they are needed (using byte loads and shifts).

---

## 4. DMA Controller

### 4.1 Overview

The DMA controller provides 8 independent channels for bulk memory transfers. DMA runs concurrently with CPU execution, enabling the game to load textures into VRAM, stream audio into Audio RAM, or perform VRAM-to-VRAM copies without stalling the CPU (except for bus contention on shared memory banks).

### 4.2 DMA Channels

| Channel | Intended Purpose                       | Source Region | Dest Region |
| ------- | -------------------------------------- | ------------- | ----------- |
| 0       | Main RAM → VRAM (texture upload)       | Main RAM      | VRAM        |
| 1       | Main RAM → Audio RAM (sample upload)   | Main RAM      | Audio RAM   |
| 2       | VRAM → VRAM (blit / copy)              | VRAM          | VRAM        |
| 3       | Main RAM → Main RAM (memcpy offload)   | Main RAM      | Main RAM    |
| 4       | VRAM → Main RAM (readback)             | VRAM          | Main RAM    |
| 5–7     | General purpose (any valid src → dest) | Any           | Any         |

Channels are not hard-restricted to these uses; the channel assignment is a convention. Any channel can transfer between any valid memory regions.

### 4.3 DMA Descriptor Format

Each DMA transfer is described by a 32-byte descriptor:

```c
typedef struct {
    uint32_t src_addr;      // Source address (byte-aligned)
    uint32_t dst_addr;      // Destination address (byte-aligned)
    uint32_t transfer_size; // Transfer size in bytes (max 16 MB)
    uint32_t stride_src;    // Source stride (for interleaved mode, 0 = contiguous)
    uint32_t stride_dst;    // Dest stride (for interleaved mode, 0 = contiguous)
    uint16_t block_size;    // Block size for strided transfers
    uint16_t flags;         // See flags below
    uint32_t next_desc;     // Address of next descriptor (for chain mode, 0 = end)
} dma_descriptor_t;         // 32 bytes total
```

**Flags field:**

| Bit  | Name         | Description                                  |
| ---- | ------------ | -------------------------------------------- |
| 0    | ENABLE       | Set to 1 to enable this descriptor.          |
| 1    | IRQ_COMPLETE | Raise DMA completion interrupt when done.    |
| 2    | CHAIN        | Follow `next_desc` pointer after completion. |
| 3    | INTERLEAVED  | Use stride_src/stride_dst/block_size.        |
| 4–15 | Reserved     | Must be 0.                                   |

### 4.4 Transfer Modes

**Block Mode (default):** Contiguous copy from `src_addr` to `dst_addr` for `transfer_size` bytes. This is the simplest and most common mode.

**Chain Mode (bit 2 set):** After completing the current transfer, the DMA controller reads the next descriptor from the address in `next_desc` and begins that transfer. This forms a linked list of transfers. A chain terminates when `next_desc` is 0 or the CHAIN flag is not set. This is useful for scatter-gather operations or multi-stage loading.

**Interleaved/Strided Mode (bit 3 set):** Transfers `block_size` bytes, then advances the source pointer by `stride_src` and the destination pointer by `stride_dst`. Repeats until `transfer_size` total bytes have been transferred. This is essential for format conversions (e.g., de-interleaving packed vertex data) and rectangular VRAM blits (copy a sub-rectangle of a texture).

### 4.5 DMA Control Registers (0x0B001000)

| Offset        | Name               | R/W | Description                                                                |
| ------------- | ------------------ | --- | -------------------------------------------------------------------------- |
| 0x00          | DMA_GLOBAL_STATUS  | R   | Bit N = 1 if channel N is busy. 8 bits used.                               |
| 0x04          | DMA_GLOBAL_CTRL    | W   | Bit N = 1 to abort channel N. Write 0xFF to reset all.                     |
| 0x10 + N×0x10 | DMA_CHN_DESC_ADDR  | W   | Write the address of the first descriptor for channel N to start transfer. |
| 0x14 + N×0x10 | DMA_CHN_STATUS     | R   | Channel N status: 0=idle, 1=running, 2=complete, 3=error.                  |
| 0x18 + N×0x10 | DMA_CHN_BYTES_LEFT | R   | Remaining bytes in current transfer.                                       |

To initiate a transfer: write a DMA descriptor to the descriptor table region (0x0B002000), then write its address to `DMA_CHN_DESC_ADDR` for the desired channel. The transfer begins immediately.

### 4.6 DMA Timing

DMA transfer rate: **64 bytes per CPU cycle** (4 bytes per DMA clock at 16:1 ratio). A 1 MB texture upload takes approximately 16,384 CPU cycles (~0.5% of the frame budget). Bus contention: if the CPU attempts to read/write the same memory bank that DMA is accessing, the CPU stalls for 2 additional cycles per access. The DMA controller has higher bus priority than the CPU.

---

## 5. Virtual GPU — Command Model & Rendering

### 5.1 Rendering Architecture

The NEXUS-32's Virtual Graphics Processor (VGP) is a command-buffer-driven renderer. The game does not call rendering functions directly; it writes binary GPU command structs sequentially into the GPU Command Buffer (at 0x0A000000, up to 2 MB). After all commands for a frame have been written, the game writes the PRESENT command and signals the GPU by writing to GPU_CONTROL.

The host emulator reads the command buffer from beginning to the PRESENT command and translates each command to equivalent Vulkan API calls. This abstraction keeps the fantasy hardware simple while allowing the emulator full freedom to optimize rendering on the host.

### 5.2 GPU Command Format

Every GPU command begins with a 4-byte command header:

```c
typedef struct {
    uint16_t cmd_type;    // Command type identifier
    uint16_t cmd_size;    // Total command size in bytes (including this header)
} gpu_cmd_header_t;
```

Command types:

| cmd_type | Name                  | Size (bytes) | Description                           |
| -------- | --------------------- | ------------ | ------------------------------------- |
| 0x0001   | CMD_CLEAR             | 16           | Clear framebuffer and/or depth buffer |
| 0x0002   | CMD_BIND_TEXTURE      | 24           | Bind a texture to a slot              |
| 0x0003   | CMD_SET_STATE         | 16           | Set render state                      |
| 0x0004   | CMD_DRAW_TRIS         | 20           | Draw triangle list                    |
| 0x0005   | CMD_DRAW_INDEXED      | 24           | Draw indexed triangles                |
| 0x0006   | CMD_DRAW_SPRITES      | 16           | Draw sprite batch                     |
| 0x0007   | CMD_SET_VIEWPORT      | 20           | Set viewport rectangle                |
| 0x0008   | CMD_SET_SHADER        | 12           | Bind vertex/fragment shader program   |
| 0x0009   | CMD_SET_UNIFORM       | Variable     | Upload uniform data                   |
| 0x000A   | CMD_COPY_REGION       | 28           | VRAM-to-VRAM blit                     |
| 0x000B   | CMD_SET_SCISSOR       | 20           | Set scissor rectangle                 |
| 0x000C   | CMD_SET_RENDER_TARGET | 16           | Set render target (for multi-pass)    |
| 0x00FF   | CMD_PRESENT           | 4            | Signal frame complete                 |

### 5.3 Command Definitions

**CMD_CLEAR (0x0001):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0001, 16 }
    uint8_t  r, g, b, a;     // Clear color (0–255 per channel)
    float    depth;           // Depth clear value (0.0–1.0)
    uint8_t  flags;           // Bit 0: clear color. Bit 1: clear depth.
    uint8_t  _pad[3];
} cmd_clear_t;
```

**CMD_BIND_TEXTURE (0x0002):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0002, 24 }
    uint8_t  slot;            // Texture slot (0–15)
    uint8_t  format;          // Pixel format (see texture formats)
    uint16_t width;           // Texture width in pixels
    uint16_t height;          // Texture height in pixels
    uint16_t _pad;
    uint32_t vram_offset;     // Byte offset from VRAM base (0x04000000)
    uint32_t flags;           // Bit 0: has mipmaps. Bits 1–2: wrap mode (0=repeat, 1=clamp, 2=mirror). Bits 3–4: filter (0=nearest, 1=bilinear, 2=trilinear).
} cmd_bind_texture_t;
```

**Texture Formats:**

| format | Name     | Bytes/pixel | Description                      |
| ------ | -------- | ----------- | -------------------------------- |
| 0x00   | RGBA8888 | 4           | 8 bits per channel, uncompressed |
| 0x01   | RGB565   | 2           | 5-6-5 packed, no alpha           |
| 0x02   | RGBA4444 | 2           | 4 bits per channel               |
| 0x03   | A8       | 1           | Alpha only (for font atlases)    |
| 0x04   | DXT1/BC1 | 0.5         | Block compressed, 1-bit alpha    |
| 0x05   | DXT5/BC3 | 1           | Block compressed, full alpha     |
| 0x06   | L8       | 1           | Luminance only (grayscale)       |
| 0x07   | LA88     | 2           | Luminance + alpha                |

Maximum texture size: 2048×2048. Mipmaps are stored contiguously in VRAM after the base level. Maximum 16 texture slots bound simultaneously.

**CMD_SET_STATE (0x0003):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0003, 16 }
    uint8_t  depth_test;      // 0=off, 1=less, 2=lequal, 3=equal, 4=gequal, 5=greater, 6=always
    uint8_t  depth_write;     // 0=off, 1=on
    uint8_t  blend_mode;      // 0=opaque, 1=alpha, 2=additive, 3=multiply
    uint8_t  cull_mode;       // 0=none, 1=back, 2=front
    uint8_t  polygon_mode;    // 0=fill, 1=wireframe
    uint8_t  _pad[7];
} cmd_set_state_t;
```

**Blend mode equations:**

| Mode     | RGB Formula                            | Alpha Formula            |
| -------- | -------------------------------------- | ------------------------ |
| Opaque   | output = src                           | output.a = 1.0           |
| Alpha    | output = src.a × src + (1-src.a) × dst | output.a = src.a         |
| Additive | output = src + dst                     | output.a = src.a         |
| Multiply | output = src × dst                     | output.a = src.a × dst.a |

**CMD_DRAW_TRIS (0x0004):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0004, 20 }
    uint32_t vram_offset;     // Vertex data offset in VRAM
    uint32_t vertex_count;    // Number of vertices (must be multiple of 3)
    uint32_t vertex_format;   // Vertex format descriptor (see below)
    uint32_t _pad;
} cmd_draw_tris_t;
```

**CMD_DRAW_INDEXED (0x0005):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0005, 24 }
    uint32_t vertex_offset;   // Vertex data offset in VRAM
    uint32_t index_offset;    // Index data offset in VRAM
    uint32_t index_count;     // Number of indices
    uint16_t vertex_format;   // Vertex format descriptor
    uint8_t  index_type;      // 0 = uint16, 1 = uint32
    uint8_t  _pad;
} cmd_draw_indexed_t;
```

**CMD_DRAW_SPRITES (0x0006):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0006, 16 }
    uint32_t sprite_offset;   // Sprite batch data offset in VRAM
    uint32_t sprite_count;    // Number of sprites in batch
    uint32_t _pad;
} cmd_draw_sprites_t;
```

**CMD_SET_VIEWPORT (0x0007):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0007, 20 }
    uint16_t x, y;            // Top-left corner
    uint16_t width, height;   // Viewport dimensions
    float    depth_min;       // Min depth (0.0)
    float    depth_max;       // Max depth (1.0)
} cmd_set_viewport_t;
```

**CMD_SET_SHADER (0x0008):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0008, 12 }
    uint16_t vertex_shader;   // Shader program index (0xFFFF = default/fixed function)
    uint16_t fragment_shader; // Shader program index (0xFFFF = default/fixed function)
    uint32_t _pad;
} cmd_set_shader_t;
```

Shader programs are stored in the shader region of VRAM and referenced by index (0–255). Index 0xFFFF selects the built-in fixed-function pipeline: vertex shader does model-view-projection transform using uniforms u0–u3 as a 4×4 matrix; fragment shader samples texture slot 0 and multiplies by vertex color.

**CMD_SET_UNIFORM (0x0009):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x0009, 8 + data_size }
    uint8_t  index;           // Uniform register index (0–15)
    uint8_t  type;            // 0=float (4B), 1=vec4 (16B), 2=mat4 (64B)
    uint16_t _pad;
    // Followed by data bytes (4, 16, or 64 bytes)
} cmd_set_uniform_t;
```

**CMD_COPY_REGION (0x000A):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x000A, 28 }
    uint32_t src_offset;      // Source VRAM offset
    uint32_t dst_offset;      // Destination VRAM offset
    uint16_t src_x, src_y;    // Source rectangle origin
    uint16_t dst_x, dst_y;    // Destination rectangle origin
    uint16_t width, height;   // Rectangle dimensions
    uint16_t src_stride;      // Source image width (for offset calculation)
    uint16_t dst_stride;      // Dest image width
} cmd_copy_region_t;
```

**CMD_PRESENT (0x00FF):**

```c
typedef struct {
    gpu_cmd_header_t header;  // { 0x00FF, 4 }
} cmd_present_t;
```

### 5.4 Vertex Formats

The vertex format descriptor is a bitmask specifying which attributes are present:

| Bit  | Attribute | Type   | Size | Description              |
| ---- | --------- | ------ | ---- | ------------------------ |
| 0    | POSITION  | float3 | 12 B | XYZ position (required)  |
| 1    | COLOR     | RGBA8  | 4 B  | Vertex color             |
| 2    | TEXCOORD0 | float2 | 8 B  | Primary UV coordinates   |
| 3    | TEXCOORD1 | float2 | 8 B  | Secondary UVs (lightmap) |
| 4    | NORMAL    | float3 | 12 B | Surface normal           |
| 5–15 | Reserved  | —      | —    | Must be 0                |

Attributes are packed in order of bit position. The vertex stride is the sum of all present attribute sizes. Example: format `0x0007` (POSITION + COLOR + TEXCOORD0) = 12 + 4 + 8 = 24 bytes per vertex.

### 5.5 Sprite System

Each sprite in a sprite batch is defined by a 32-byte entry:

```c
typedef struct {
    float    x, y;            // Screen position (top-left corner)
    float    w, h;            // Display size in pixels
    uint16_t tex_u, tex_v;    // Texture region origin (in texels)
    uint16_t tex_w, tex_h;    // Texture region size (in texels)
    uint8_t  r, g, b, a;     // Color tint / alpha
    uint16_t rotation;        // Rotation in 1/256th degrees (0–65535 → 0°–~256°)
    uint8_t  flags;           // Bit 0: flip horizontal. Bit 1: flip vertical.
    uint8_t  texture_slot;    // Which bound texture to use (0–15)
} sprite_entry_t;             // 32 bytes
```

Sprites are rendered as textured quads in submission order (painter's algorithm). The emulator converts each sprite to 2 triangles. Sprites always use alpha blending and are rendered after all 3D geometry unless the game explicitly interleaves 3D and sprite draw commands.

### 5.6 Mini Shader Language

The NEXUS-32 supports programmable vertex and fragment shaders via a simple instruction set. Shaders are compiled offline by the `shaderc` tool and uploaded to VRAM as binary programs.

**Shader Resources:**

| Resource         | Count | Description                                 |
| ---------------- | ----- | ------------------------------------------- |
| Temp registers   | 8     | t0–t7, read/write, 4-component float (vec4) |
| Uniform regs     | 16    | u0–u15, read-only, set via CMD_SET_UNIFORM  |
| Input registers  | 4     | i0–i3, vertex attributes (see below)        |
| Output registers | 2     | o0–o1, written by shader (see below)        |
| Sampler          | 4     | s0–s3, bound texture samplers               |
| Max instructions | 64    | Per program                                 |

**Vertex shader inputs/outputs:**

| Register | Input     | Output                        |
| -------- | --------- | ----------------------------- |
| i0       | position  | —                             |
| i1       | color     | —                             |
| i2       | texcoord0 | —                             |
| i3       | normal    | —                             |
| o0       | —         | position (clip space)         |
| o1       | —         | color/texcoord (interpolated) |

**Fragment shader inputs/outputs:**

| Register | Input                       | Output             |
| -------- | --------------------------- | ------------------ |
| i0       | interpolated position       | —                  |
| i1       | interpolated color/texcoord | —                  |
| o0       | —                           | final color (RGBA) |

**Shader Instruction Set:**

Each instruction is 8 bytes:

```c
typedef struct {
    uint8_t  opcode;        // Instruction opcode
    uint8_t  dst;           // Destination register + write mask
    uint8_t  src0;          // Source 0 register + swizzle
    uint8_t  src1;          // Source 1 register + swizzle
    uint8_t  src2;          // Source 2 register (for MAD/LERP/CMP)
    uint8_t  modifier;      // Modifier flags (saturate, negate src)
    uint16_t _pad;
} shader_instruction_t;     // 8 bytes
```

Register encoding (in dst, src0, src1, src2):

| Bits | Field | Description                                     |
| ---- | ----- | ----------------------------------------------- |
| 7–5  | Bank  | 0=temp, 1=uniform, 2=input, 3=output, 4=sampler |
| 4–0  | Index | Register index within bank                      |

Write mask (modifier bits for dst):

| Bits of modifier | Mask |
| ---------------- | ---- |
| 0                | .x   |
| 1                | .y   |
| 2                | .z   |
| 3                | .w   |

Swizzle (encoded in high nibble of src fields): 2-bit per component selects .x/.y/.z/.w.

**Opcodes:**

| Opcode | Mnemonic | Operation                                      |
| ------ | -------- | ---------------------------------------------- |
| 0x00   | MOV      | dst = src0                                     |
| 0x01   | ADD      | dst = src0 + src1                              |
| 0x02   | SUB      | dst = src0 - src1                              |
| 0x03   | MUL      | dst = src0 × src1                              |
| 0x04   | MAD      | dst = src0 × src1 + src2                       |
| 0x05   | DP3      | dst = dot(src0.xyz, src1.xyz)                  |
| 0x06   | DP4      | dst = dot(src0, src1)                          |
| 0x07   | RSQ      | dst = 1 / sqrt(src0)                           |
| 0x08   | RCP      | dst = 1 / src0                                 |
| 0x09   | MIN      | dst = min(src0, src1)                          |
| 0x0A   | MAX      | dst = max(src0, src1)                          |
| 0x0B   | CLAMP    | dst = clamp(src0, src1, src2)                  |
| 0x0C   | LERP     | dst = src0 + src2 × (src1 - src0)              |
| 0x0D   | TEX      | dst = texture_sample(sampler[src1.x], src0.xy) |
| 0x0E   | CMP      | dst = (src0 >= 0) ? src1 : src2                |
| 0x0F   | ABS      | dst = abs(src0)                                |
| 0x10   | NEG      | dst = -src0                                    |
| 0x11   | FRC      | dst = fract(src0)                              |
| 0x12   | FLR      | dst = floor(src0)                              |
| 0x13   | EXP      | dst = 2^src0 (approximate)                     |
| 0x14   | LOG      | dst = log2(src0) (approximate)                 |
| 0x15   | NOP      | No operation                                   |

There is no dynamic branching. All conditional logic must use CMP (conditional move). This simplifies the shader pipeline and makes shader execution fully predictable.

### 5.7 Framebuffer and Resolution

**Native resolution:** 640×480 (default). The console also supports a half-resolution mode of 320×240 for performance-intensive scenes (the emulator upscales).

Resolution is configured via GPU control registers `GPU_WIDTH` and `GPU_HEIGHT`. Supported modes:

| Mode | Width | Height | FB Size (RGBA8) | + Depth (16-bit) | Total (double buffered) |
| ---- | ----- | ------ | --------------- | ---------------- | ----------------------- |
| Full | 640   | 480    | 1.2 MB          | 600 KB           | 3.0 MB                  |
| Half | 320   | 240    | 300 KB          | 150 KB           | 750 KB                  |
| Wide | 640   | 360    | 900 KB          | 450 KB           | 2.25 MB                 |

**Render targets:** Up to 3 auxiliary render targets can be configured by writing VRAM addresses to GPU registers at offsets 0x30–0x3B. These enable multi-pass rendering (shadow maps, reflection probes, post-processing). The game switches the active render target via CMD_SET_RENDER_TARGET.

### 5.8 Command Buffer Limits

The command buffer is 2 MB per frame. Typical command sizes range from 4 bytes (PRESENT) to 72 bytes (SET_UNIFORM with mat4). A frame with 500 draw calls, each requiring a state change, texture bind, and draw command (~60 bytes total) uses approximately 30 KB — well within budget. The 2 MB limit permits extremely complex scenes (theoretically ~25,000+ draw calls) while still imposing a natural upper bound that rewards efficient batching.

**Command buffer overflow:** If the game writes past the 2 MB boundary (i.e., writes to addresses at or beyond `0x0A200000`), the write triggers a **System/Debug exception (IRQ 7)** and the offending write is discarded. The `CAUSE` register is set to `0x00000010` (GPU command buffer overflow). The emulator logs a diagnostic message with the current PC and the number of bytes written. The GPU will still render all commands written before the overflow point when PRESENT is eventually issued. This hardware trap prevents silent data corruption and gives developers immediate, visible feedback when their rendering workload exceeds the buffer. The SDK's `gfx_present()` function also checks `GPU_CB_SIZE` against the buffer limit before submitting and will call `debug_print()` with a warning if usage exceeds 90%.

---

## 6. Audio Processing Unit

### 6.1 Voice Model

The APU provides **48 simultaneous voices**, each capable of independent sample playback. This is sufficient for rich soundscapes: 16 voices for music, 24 for sound effects, and 8 reserved for ambient/UI sounds (by convention; the hardware does not enforce partitioning).

Each voice reads sample data from Audio RAM and produces a mono output that is mixed into the stereo output bus. Voice parameters are fully memory-mapped and can be updated at any time.

### 6.2 Audio Registers (0x0B003000)

Each voice occupies a 64-byte register block. Total: 48 voices × 64 bytes = 3,072 bytes.

**Per-Voice Register Block (64 bytes, at base + voice_index × 0x40):**

| Offset    | Name        | Type   | Description                                               |
| --------- | ----------- | ------ | --------------------------------------------------------- |
| 0x00      | SAMPLE_ADDR | uint32 | Sample data address in Audio RAM (relative to 0x06000000) |
| 0x04      | SAMPLE_LEN  | uint32 | Sample length in frames (samples, not bytes)              |
| 0x08      | LOOP_ADDR   | uint32 | Loop start address (in Audio RAM)                         |
| 0x0C      | LOOP_LEN    | uint32 | Loop length in frames (0 = no loop)                       |
| 0x10      | PITCH       | uint16 | Playback rate (fixed-point 8.8: 0x0100 = 1.0×)            |
| 0x12      | FORMAT      | uint8  | Sample format (see below)                                 |
| 0x13      | STATUS      | uint8  | R: 0=idle, 1=playing, 2=releasing                         |
| 0x14      | VOL_L       | uint8  | Left volume (0–255)                                       |
| 0x15      | VOL_R       | uint8  | Right volume (0–255)                                      |
| 0x16      | KEY_ON      | uint8  | W: write 1 to start playback                              |
| 0x17      | KEY_OFF     | uint8  | W: write 1 to enter release phase                         |
| 0x18      | ENV_ATTACK  | uint16 | Attack time in milliseconds (0–10000)                     |
| 0x1A      | ENV_DECAY   | uint16 | Decay time in milliseconds (0–10000)                      |
| 0x1C      | ENV_SUSTAIN | uint8  | Sustain level (0–255)                                     |
| 0x1D      | ENV_RELEASE | uint16 | Release time in milliseconds (0–10000)                    |
| 0x1F      | _pad        | uint8  | Reserved                                                  |
| 0x20      | CURRENT_POS | uint32 | R: current playback position (for visualization)          |
| 0x24–0x3F | _reserved   | —      | Reserved for future use                                   |

**Sample formats:**

| format | Name  | Bytes/frame | Description                        |
| ------ | ----- | ----------- | ---------------------------------- |
| 0x00   | PCM8  | 1           | 8-bit signed PCM                   |
| 0x01   | PCM16 | 2           | 16-bit signed PCM (native quality) |
| 0x02   | ADPCM | 0.5         | IMA ADPCM, 4-bit (2:1 compression) |

### 6.3 Global Audio Registers (at 0x0B003000 + 0xC00)

| Offset | Name            | Type   | Description                                                                 |
| ------ | --------------- | ------ | --------------------------------------------------------------------------- |
| 0x00   | MASTER_VOL      | uint8  | Master volume (0–255)                                                       |
| 0x01   | REVERB_ENABLE   | uint8  | 0=off, 1=on                                                                 |
| 0x02   | REVERB_SIZE     | uint8  | Room size (0=small, 128=medium, 255=large)                                  |
| 0x03   | REVERB_DAMP     | uint8  | High-frequency damping (0–255)                                              |
| 0x04   | REVERB_WET      | uint8  | Wet mix level (0–255)                                                       |
| 0x05   | REVERB_DRY      | uint8  | Dry mix level (0–255)                                                       |
| 0x06   | EQ_LOW          | int8   | Low-band EQ gain (-12 to +12 dB, mapped 0–255)                              |
| 0x07   | EQ_MID          | int8   | Mid-band EQ gain                                                            |
| 0x08   | EQ_HIGH         | int8   | High-band EQ gain                                                           |
| 0x09   | RING_BUF_ADDR   | uint32 | Audio ring buffer address in Main RAM                                       |
| 0x0D   | RING_BUF_SIZE   | uint16 | Ring buffer size in bytes                                                   |
| 0x0F   | RING_BUF_POS    | uint16 | R: current write position in ring buffer                                    |
| 0x11   | RING_BUF_STATUS | uint8  | R: Bit 0: underrun occurred. Auto-clears when buffer recovers to ≥25% fill. |

### 6.4 Audio Output Pipeline

The APU mixes all active voices into a stereo ring buffer at **48 kHz, 16-bit signed, interleaved stereo** (4 bytes per sample frame). The ring buffer resides in Main RAM at the address specified by `RING_BUF_ADDR` (default: 0x01800000, size: 65,536 bytes).

The APU writes continuously to the ring buffer. When the write position crosses the halfway point, it fires **IRQ 4** (audio buffer interrupt), signaling the game that the first half of the buffer has been consumed and can be safely updated (e.g., trigger new voices, update parameters). This double-buffered approach ensures glitch-free playback.

**Audio buffer underrun:** If the APU's write pointer catches up to the emulator's read pointer (because the CPU failed to update voices or the interrupt was missed due to heavy processing or disabled interrupts), the APU enters an underrun state. In this state, the APU outputs silence (zeros) rather than looping over stale data. A status flag is set in the global audio registers: `RING_BUF_STATUS` (offset 0x11 in the global audio block, read-only) bit 0 = underrun occurred. The game can poll this flag to detect audio glitches. The flag is cleared automatically when the ring buffer returns to a healthy fill level (write pointer leads read pointer by at least 25% of the buffer size). The SDK's `audio_init()` function registers a default IRQ 4 handler that calls the APU mixer, but developers can override it for custom audio logic.

The host emulator reads from the ring buffer and feeds samples to the host audio system (via SDL2 audio callback or similar). If the host audio sample rate differs from 48 kHz, the emulator performs sample rate conversion.

### 6.5 Sequencer

The NEXUS-32 includes a **software sequencer** implemented in the SDK standard library (`audio` module), not in hardware. The sequencer supports a tracker-style format:

**Sequence format (.nxseq):**

```c
typedef struct {
    uint16_t tempo;           // BPM (20–300)
    uint8_t  speed;           // Ticks per row (1–32, default 6)
    uint8_t  num_channels;    // Number of channels (1–32)
    uint8_t  num_patterns;    // Number of patterns
    uint8_t  num_instruments; // Number of instruments
    uint16_t sequence_length; // Length of pattern order table
    // Followed by: instrument table, pattern order table, pattern data
} nxseq_header_t;
```

Each row in a pattern specifies: note (0–127), instrument index, volume (0–64), and an effect command (vibrato, portamento, volume slide, etc.). The SDK's `audio_play_sequence()` function interprets this data and drives the hardware voices. This keeps the hardware simple while providing rich music playback through software.

---

## 7. Input System

### 7.1 Controller Model

The NEXUS-32 supports up to **4 controllers**. Each controller provides:

- **D-pad:** 4 directional buttons (up, down, left, right)
- **Face buttons:** A, B, X, Y
- **Shoulder buttons:** L, R (digital)
- **Triggers:** LT, RT (analog, 8-bit, 0–255)
- **Analog sticks:** Left stick (X, Y), Right stick (X, Y) — each axis is signed 8-bit (-128 to +127)
- **System buttons:** Start, Select

### 7.2 Memory-Mapped Input Layout (0x0B004000)

Each controller occupies 32 bytes. Four controllers: 128 bytes total.

**Per-Controller Layout (32 bytes, at base + controller_index × 0x20):**

```c
typedef struct {
    uint16_t buttons;       // Button bitmask (see below)
    int8_t   left_x;        // Left stick X (-128 to +127)
    int8_t   left_y;        // Left stick Y (-128 to +127)
    int8_t   right_x;       // Right stick X
    int8_t   right_y;       // Right stick Y
    uint8_t  trigger_l;     // Left trigger (0–255)
    uint8_t  trigger_r;     // Right trigger (0–255)
    uint16_t buttons_prev;  // Button state from previous frame
    uint8_t  connected;     // 0=disconnected, 1=connected
    uint8_t  _pad[21];      // Reserved (total 32 bytes)
} controller_state_t;
```

**Button bitmask:**

| Bit   | Button        |
| ----- | ------------- |
| 0     | A             |
| 1     | B             |
| 2     | X             |
| 3     | Y             |
| 4     | L             |
| 5     | R             |
| 6     | Start         |
| 7     | Select        |
| 8     | D-Up          |
| 9     | D-Down        |
| 10    | D-Left        |
| 11    | D-Right       |
| 12    | L-Stick Click |
| 13    | R-Stick Click |
| 14–15 | Reserved      |

The emulator updates all controller state once per frame, before the VBlank interrupt. The input region is read-only from the CPU; writes are silently discarded.

The `buttons_prev` field is maintained by the emulator (it copies `buttons` to `buttons_prev` at the start of each frame before updating `buttons`). This allows the SDK to implement "just pressed" detection as `(buttons & ~buttons_prev)` and "just released" as `(~buttons & buttons_prev)`.

### 7.3 No Networking

The NEXUS-32 intentionally provides no networking hardware. All multiplayer is local (up to 4 players via controllers). This design constraint pushes developers toward focused, complete, offline game experiences — the spirit of the console.

---

## 8. Timer & System Registers

### 8.1 Programmable Timers (0x0B005000)

Two independent countdown timers are available. Each timer decrements at a configurable rate and can fire an interrupt on reaching zero.

**Timer Register Layout (per timer, 16 bytes each):**

| Offset | Name        | R/W | Description                                           |
| ------ | ----------- | --- | ----------------------------------------------------- |
| 0x00   | TMR_COUNT   | R/W | Current counter value (32-bit). Decrements each tick. |
| 0x04   | TMR_RELOAD  | R/W | Reload value (written to COUNT on overflow/reset).    |
| 0x08   | TMR_CONTROL | R/W | See control bits below.                               |
| 0x0C   | TMR_STATUS  | R   | Bit 0: overflow flag (write 1 to clear).              |

Timer 0: offset 0x00–0x0F. Timer 1: offset 0x10–0x1F.

**TMR_CONTROL bits:**

| Bit | Name        | Description                                       |
| --- | ----------- | ------------------------------------------------- |
| 0   | ENABLE      | 1=timer running, 0=stopped                        |
| 1   | IRQ_EN      | 1=fire interrupt on overflow                      |
| 2   | AUTO_RELOAD | 1=reload from TMR_RELOAD on overflow, 0=stop      |
| 3–5 | PRESCALE    | Clock divider: 0=1×, 1=8×, 2=64×, 3=256×, 4=1024× |

The timer's effective tick rate is the CPU's cycle rate divided by the prescaler. At prescale 0, the timer decrements once per CPU cycle. At prescale 4, it decrements once every 1024 CPU cycles.

### 8.2 Frame Counter (0x0B005020)

| Offset | Name        | R   | Description                              |
| ------ | ----------- | --- | ---------------------------------------- |
| 0x20   | FRAME_COUNT | R   | 32-bit counter, increments every VBlank. |

Read-only. Overflows after ~2.27 years at 60 FPS. Useful for timing, animation, and seeding RNG.

### 8.3 System Registers (0x0B006000)

| Offset | Name          | R/W | Description                                                             |
| ------ | ------------- | --- | ----------------------------------------------------------------------- |
| 0x00   | SYS_VERSION   | R   | Console version: major (high 16) / minor (low 16). Current: 0x00010000. |
| 0x04   | SYS_RAM_SIZE  | R   | Main RAM size in bytes (0x02000000 = 32 MB).                            |
| 0x08   | SYS_VRAM_SIZE | R   | VRAM size in bytes (0x01000000 = 16 MB).                                |
| 0x0C   | SYS_ARAM_SIZE | R   | Audio RAM size in bytes (0x00800000 = 8 MB).                            |
| 0x10   | SYS_RNG       | R   | Hardware RNG. Each read returns a new pseudo-random 32-bit value.       |
| 0x14   | SYS_CYCLES    | R   | Cycles consumed this frame (for profiling).                             |
| 0x18   | SYS_CYCLE_MAX | R   | Cycle budget for this frame.                                            |

The RNG uses a hardware xorshift128+ generator seeded from the host's entropy source at boot. Each read advances the state and returns the next value.

### 8.4 Interrupt Control Registers (0x0B007000)

| Offset | Name        | R/W | Description                                      |
| ------ | ----------- | --- | ------------------------------------------------ |
| 0x00   | IRQ_PENDING | R   | Bitmask of pending interrupts (8 bits).          |
| 0x04   | IRQ_ENABLE  | R/W | Bitmask of enabled interrupts (mirrors SR.IM).   |
| 0x08   | IRQ_ACK     | W   | Write bitmask to acknowledge/clear pending IRQs. |

---

## 9. ROM Image Format

### 9.1 File Extension and Magic

ROM files use the extension `.nxrom`. The magic bytes are `NX32` (ASCII: 0x4E583332).

### 9.2 ROM Header (128 bytes)

```c
typedef struct {
    uint8_t  magic[4];          // "NX32" (0x4E, 0x58, 0x33, 0x32)
    uint16_t format_version;    // ROM format version (current: 0x0100 = 1.0)
    uint16_t flags;             // Bit 0: compressed assets. Bit 1: save data required.
    uint32_t entry_point;       // Entry point address in Main RAM
    uint32_t code_offset;       // Offset of code segment in ROM file
    uint32_t code_size;         // Code segment size in bytes
    uint32_t data_offset;       // Offset of data segment in ROM file
    uint32_t data_size;         // Data segment size in bytes
    uint32_t asset_table_offset;// Offset of asset directory in ROM file
    uint32_t asset_table_size;  // Asset directory size in bytes
    uint32_t total_rom_size;    // Total ROM file size in bytes
    uint32_t cycle_budget;      // Per-frame cycle budget (0 = use default 3M)
    uint16_t screen_width;      // Preferred screen width (0 = default 640)
    uint16_t screen_height;     // Preferred screen height (0 = default 480)
    char     title[32];         // Game title (UTF-8, null-terminated)
    char     author[32];        // Author/studio (UTF-8, null-terminated)
    uint32_t checksum;          // CRC32 of entire ROM excluding this field and header_checksum
    uint32_t header_checksum;   // CRC32 of header bytes 0–119
    uint8_t  _reserved[8];      // Reserved (must be 0)
} nxrom_header_t;               // 128 bytes total
```

### 9.3 Code Segment

The code segment contains raw NEXUS-32 machine code. At load time, the emulator copies this segment to Main RAM starting at the `entry_point` address (typically 0x00000400). The code segment is not compressed; it is always loaded verbatim.

Maximum code segment size: 4 MB (must fit within the code area of Main RAM).

### 9.4 Data Segment

The data segment contains initialized global data, lookup tables, constant arrays, and other non-code data the program requires at startup. Loaded to Main RAM immediately after the code segment.

Maximum data segment size: 4 MB.

### 9.5 Asset Directory

The asset directory is an index of all packed assets:

```c
typedef struct {
    uint32_t num_assets;        // Number of assets in directory
    // Followed by num_assets × asset_entry_t
} asset_directory_t;

typedef struct {
    char     name[32];          // Asset name (e.g., "player_sprite.tex"), null-terminated
    uint32_t asset_type;        // Type: 0=texture, 1=mesh, 2=audio, 3=tilemap, 4=shader, 5=generic
    uint32_t format;            // Type-specific format code
    uint32_t rom_offset;        // Offset of asset data within ROM file
    uint32_t compressed_size;   // Size in ROM (compressed)
    uint32_t uncompressed_size; // Size when decompressed
    uint32_t target_region;     // Target: 0=Main RAM, 1=VRAM, 2=Audio RAM
    uint32_t target_address;    // Suggested load address (0 = auto-allocate)
} asset_entry_t;                // 64 bytes
```

### 9.6 Asset Segments

Asset data follows the asset directory. Each asset is stored contiguously at the offset specified in its directory entry. If the ROM's `flags` bit 0 (compressed assets) is set, asset data is LZ4-compressed. The SDK runtime (or emulator) decompresses assets on load using the LZ4 fast decompression algorithm.

Asset types and their target memory:

| Type    | ID  | Target    | Typical Formats                              |
| ------- | --- | --------- | -------------------------------------------- |
| Texture | 0   | VRAM      | RGBA8888, RGB565, RGBA4444, DXT1, DXT5       |
| Mesh    | 1   | VRAM      | Vertex + index data in console vertex format |
| Audio   | 2   | Audio RAM | PCM16, ADPCM                                 |
| Tilemap | 3   | Main RAM  | Console tilemap binary format                |
| Shader  | 4   | VRAM      | Compiled shader binary                       |
| Generic | 5   | Main RAM  | Arbitrary data (scripts, configs, etc.)      |

### 9.7 Compression

LZ4 is the sole supported compression algorithm. LZ4 was chosen for its decompression speed (typically >2 GB/s on modern hosts, and the NEXUS-32 SDK runtime includes an optimized software decoder). Compression is applied per-asset, not to the ROM as a whole. The code and data segments are never compressed.

### 9.8 ROM Size Limit

Maximum ROM size: **256 MB**. This is large enough for substantial games with rich art and audio assets, but small enough to prevent lazy mega-asset bloat and encourage thoughtful asset budgeting.

### 9.9 Save Data

Save data uses a **64 KB virtual EEPROM** memory-mapped at `0x08000000–0x0800FFFF`. The game reads and writes this region like normal memory. The emulator persists this region to a host file (`<rom_name>.sav`) and loads it at startup.

Writes to the EEPROM region are buffered and flushed to disk on PRESENT (once per frame) or when the emulator is closed. The SDK provides `save_read()` and `save_write()` helper functions that abstract the raw memory access and add a small header with a validity check.

---

## 10. Game SDK Specification

### 10.1 C Compiler

The SDK ships a **C99-compliant cross-compiler** (`nxcc`) targeting the NEXUS-32 ISA. It accepts standard C source files and produces object files in the NEXUS-32 object format (`.nxo`).

**Calling convention:**

| Register        | Usage                                     |
| --------------- | ----------------------------------------- |
| r4–r7           | Arguments 1–4 (excess on stack)           |
| r2–r3           | Return values (r2 primary, r3 for 64-bit) |
| r8–r15, r24–r25 | Caller-saved temporaries                  |
| r16–r23, r29    | Callee-saved                              |
| r30 (sp)        | Stack pointer (16-byte aligned)           |
| r31 (ra)        | Return address                            |
| r28 (gp)        | Global pointer                            |

Stack frames grow downward. The caller pushes arguments beyond the first 4 onto the stack right-to-left. The callee is responsible for saving and restoring any callee-saved registers it modifies.

**Vector intrinsics:** The compiler provides built-in functions for vector operations:

```c
#include <nx_vec.h>

typedef float __attribute__((vector_size(16))) vec4_t;

vec4_t __nx_vadd(vec4_t a, vec4_t b);       // Maps to VADD
vec4_t __nx_vsub(vec4_t a, vec4_t b);       // Maps to VSUB
vec4_t __nx_vmul(vec4_t a, vec4_t b);       // Maps to VMUL
float  __nx_vdot(vec4_t a, vec4_t b);       // Maps to VDOT
vec4_t __nx_vcross(vec4_t a, vec4_t b);     // Maps to VCROSS
vec4_t __nx_vnorm(vec4_t a);                // Maps to VNORM
vec4_t __nx_vlerp(vec4_t a, vec4_t b, float t); // Maps to VLERP
vec4_t __nx_vmin(vec4_t a, vec4_t b);       // Maps to VMIN
vec4_t __nx_vmax(vec4_t a, vec4_t b);       // Maps to VMAX
vec4_t __nx_vec4(float x, float y, float z, float w); // Construct vec4
```

Inline assembly is supported via `__asm__` blocks with GCC-compatible syntax.

### 10.2 Assembler

The standalone assembler (`nxasm`) accepts `.asm` source files using AT&T-style syntax and produces `.nxo` object files. It supports labels, `.section` directives (`.text`, `.data`, `.rodata`, `.bss`), `.global` for symbol export, and all NEXUS-32 integer and vector mnemonics.

### 10.3 Linker

The linker (`nxld`) takes one or more `.nxo` object files and the SDK standard library archive (`libnx.a`) and produces a `.nxbin` linked binary. It resolves all symbols, assigns final addresses according to the memory map layout (Section 3.2), and generates a symbol table for debugging.

### 10.4 Standard Library Modules

**`sys` — System Control:**

```c
void sys_init(void);                    // Initialize console (called once at boot)
void sys_vsync(void);                   // Wait for VBlank interrupt (frame sync)
void sys_set_irq_handler(int irq, void (*handler)(void)); // Register interrupt handler
uint32_t sys_frame_count(void);         // Read frame counter
uint32_t sys_cycles_used(void);         // Cycles consumed this frame
void sys_halt(void);                    // Halt until next interrupt
```

**`gfx` — Graphics:**

```c
void gfx_init(void);                    // Initialize GPU command buffer
void gfx_clear(uint8_t r, uint8_t g, uint8_t b, float depth);
void gfx_bind_texture(int slot, uint32_t vram_offset, int w, int h, int format);
void gfx_set_blend(int mode);           // BLEND_OPAQUE, BLEND_ALPHA, etc.
void gfx_set_depth(int test, int write);
void gfx_set_cull(int mode);
void gfx_draw_triangles(uint32_t vram_offset, int count, int format);
void gfx_draw_indexed(uint32_t vert_off, uint32_t idx_off, int count, int format, int idx_type);
void gfx_draw_sprites(sprite_entry_t* sprites, int count); // Uploads to VRAM then issues CMD
void gfx_set_shader(int vert_idx, int frag_idx);
void gfx_set_uniform_float(int idx, float val);
void gfx_set_uniform_vec4(int idx, float x, float y, float z, float w);
void gfx_set_uniform_mat4(int idx, const float* mat);  // 16 floats, column-major
void gfx_set_viewport(int x, int y, int w, int h);
void gfx_present(void);                 // Write PRESENT command and signal GPU
// High-level helpers:
void gfx_draw_tilemap(const tilemap_t* map, int scroll_x, int scroll_y);
void gfx_draw_text(const char* str, int x, int y, uint8_t r, uint8_t g, uint8_t b);
```

**`input` — Input:**

```c
uint16_t input_buttons(int controller);         // Current button bitmask
uint16_t input_buttons_pressed(int controller);  // Just pressed this frame
uint16_t input_buttons_released(int controller); // Just released this frame
int8_t   input_stick_lx(int controller);         // Left stick X
int8_t   input_stick_ly(int controller);         // Left stick Y
int8_t   input_stick_rx(int controller);         // Right stick X
int8_t   input_stick_ry(int controller);         // Right stick Y
uint8_t  input_trigger_l(int controller);        // Left trigger
uint8_t  input_trigger_r(int controller);        // Right trigger
int      input_connected(int controller);        // Is controller connected?

// Convenience: returns stick with deadzone applied (default deadzone: 16)
int8_t   input_stick_lx_dz(int controller, int deadzone);
```

**`audio` — Audio:**

```c
void audio_init(void);                    // Initialize APU and ring buffer
void audio_play_sample(int voice, uint32_t addr, uint32_t len, int format, int loop);
void audio_set_pitch(int voice, uint16_t pitch);   // Fixed-point 8.8
void audio_set_volume(int voice, uint8_t left, uint8_t right);
void audio_set_adsr(int voice, uint16_t a, uint16_t d, uint8_t s, uint16_t r);
void audio_key_on(int voice);
void audio_key_off(int voice);
void audio_set_reverb(int enable, uint8_t size, uint8_t damp, uint8_t wet, uint8_t dry);
void audio_set_master_volume(uint8_t vol);
void audio_play_sequence(const void* seq_data);  // Play tracker sequence
void audio_stop_sequence(void);
```

**`math` — Math Library:**

```c
// Trig (lookup table + linear interpolation, fast)
float nx_sinf(float radians);
float nx_cosf(float radians);
float nx_tanf(float radians);
float nx_atan2f(float y, float x);

// Vector math (wraps intrinsics)
vec4_t vec4_add(vec4_t a, vec4_t b);
vec4_t vec4_sub(vec4_t a, vec4_t b);
vec4_t vec4_scale(vec4_t v, float s);
float  vec4_dot(vec4_t a, vec4_t b);
vec4_t vec4_cross(vec4_t a, vec4_t b);
vec4_t vec4_normalize(vec4_t v);
float  vec4_length(vec4_t v);
vec4_t vec4_lerp(vec4_t a, vec4_t b, float t);

// Matrix math (4x4, column-major float[16])
void mat4_identity(float* out);
void mat4_multiply(float* out, const float* a, const float* b);
void mat4_translate(float* out, float x, float y, float z);
void mat4_rotate_y(float* out, float radians);
void mat4_rotate_x(float* out, float radians);
void mat4_rotate_z(float* out, float radians);
void mat4_scale(float* out, float x, float y, float z);
void mat4_perspective(float* out, float fov, float aspect, float near, float far);
void mat4_ortho(float* out, float left, float right, float bottom, float top, float near, float far);
void mat4_inverse(float* out, const float* m);
void mat4_look_at(float* out, vec4_t eye, vec4_t target, vec4_t up);

// Random
uint32_t nx_rand(void);               // Read hardware RNG
int      nx_rand_range(int min, int max); // Random int in [min, max]
float    nx_randf(void);              // Random float in [0.0, 1.0)
```

**`mem` — Memory Management:**

```c
void  mem_init(void);                   // Initialize heap allocator
void* mem_alloc(uint32_t size);         // Arena-style allocation (fast, no individual free)
void  mem_free_all(void);              // Free entire heap (call once per level/scene)
void* mem_pool_create(uint32_t obj_size, uint32_t count); // Pool allocator for fixed-size objects
void* mem_pool_alloc(void* pool);
void  mem_pool_free(void* pool, void* ptr);
void  mem_copy(void* dst, const void* src, uint32_t size); // CPU memcpy
void  mem_set(void* dst, uint8_t val, uint32_t size);      // CPU memset
void  mem_dma_copy(int channel, uint32_t dst, uint32_t src, uint32_t size); // DMA transfer
void  mem_dma_wait(int channel);        // Block until DMA channel completes
```

**`asset` — Asset Loading:**

```c
void    asset_init(const void* rom_base);  // Initialize from ROM asset directory
void*   asset_load(const char* name);       // Load asset by name, decompress, return pointer
void*   asset_load_to(const char* name, uint32_t target_addr); // Load to specific address
int     asset_count(void);                  // Number of assets in ROM
const char* asset_name(int index);          // Get asset name by index
```

**`save` — Save Data:**

```c
int  save_read(void* buf, uint32_t offset, uint32_t size);   // Read from EEPROM
int  save_write(const void* buf, uint32_t offset, uint32_t size); // Write to EEPROM
int  save_valid(void);    // Check if save data has valid header
void save_clear(void);    // Erase all save data
```

**`debug` — Debugging:**

```c
void debug_print(const char* fmt, ...);    // Printf to emulator debug console
void debug_log_cycles(const char* label);  // Log current cycle count with label
void debug_break(void);                    // Trigger breakpoint (BREAK instruction)
uint32_t debug_perf_start(void);           // Read cycle counter
uint32_t debug_perf_end(uint32_t start);   // Return elapsed cycles since start
```

### 10.5 Asset Pipeline Tools

All tools run on the host machine (not on the console). They convert standard file formats to NEXUS-32 native binary formats.

**`img2tex`** — Image to texture converter.

```
Usage: img2tex input.png -o output.tex [-format rgba8888|rgb565|rgba4444|a8|dxt1|dxt5|l8|la88] [-mipmap] [-resize WxH]
```

Reads PNG or BMP. Outputs `.tex` file containing: a small header (format, width, height, mip count) followed by pixel data. Generates mipmaps if requested (box filter downsampling). Default format: RGBA8888.

**`obj2mesh`** — Model to mesh converter.

```
Usage: obj2mesh input.obj -o output.mesh [-format pos|pos_col|pos_col_uv|pos_col_uv_nrm]
```

Reads OBJ or glTF. Triangulates all faces. Outputs `.mesh` file: header (vertex count, index count, vertex format) followed by vertex data and 16-bit index data. Vertex attributes are interleaved.

**`wav2smp`** — Audio sample converter.

```
Usage: wav2smp input.wav -o output.smp [-format pcm16|pcm8|adpcm] [-rate 48000] [-mono]
```

Reads WAV. Resamples to target rate. Encodes to specified format. Outputs `.smp` file: header (format, sample rate, length, loop point) + sample data.

**`map2lvl`** — Tilemap converter.

```
Usage: map2lvl input.json -o output.lvl [-tiled|-ldtk]
```

Reads Tiled (.json) or LDTK exports. Outputs `.lvl` file containing: tilemap dimensions, tile size, layer count, tile data (uint16 indices), collision layer, and object/entity spawn points.

**`shaderc`** — Shader compiler.

```
Usage: shaderc input.shader -o output.shd [-type vertex|fragment]
```

Compiles the NEXUS-32 mini shader language from text source to binary format. The source language uses a simple assembly-like syntax:

```
# Example vertex shader
mov t0, i0              # position
dp4 o0.x, t0, u0       # transform by MVP matrix row 0
dp4 o0.y, t0, u1       # row 1
dp4 o0.z, t0, u2       # row 2
dp4 o0.w, t0, u3       # row 3
mov o1, i1              # pass through color
```

The compiler validates instruction count (max 64), register usage, and outputs the binary `shader_instruction_t` array.

### 10.6 Build System

The SDK provides `nxbuild`, a build orchestrator that wraps the entire pipeline:

```
nxbuild [--config build.toml] [--release] [--verbose]
```

**Build pipeline:**

1. **Compile:** `nxcc` compiles all `.c` files to `.nxo` object files.
2. **Assemble:** `nxasm` assembles any `.asm` files to `.nxo`.
3. **Link:** `nxld` links all `.nxo` files with `libnx.a` to produce `game.nxbin`.
4. **Convert assets:** `img2tex`, `obj2mesh`, `wav2smp`, `map2lvl`, `shaderc` run on all assets listed in `build.toml`.
5. **Pack ROM:** `rompack` combines `game.nxbin` and all asset files into `game.nxrom`.

**`build.toml` example:**

```toml
[project]
name = "My Game"
author = "Studio Name"
entry = "src/main.c"

[build]
sources = ["src/*.c"]
includes = ["include/"]
libs = ["libnx"]
optimize = 2            # -O2

[screen]
width = 640
height = 480
cycle_budget = 3000000

[assets]
textures = [
    { src = "art/player.png", name = "player.tex", format = "rgba8888", mipmap = true },
    { src = "art/tileset.png", name = "tileset.tex", format = "rgb565" },
]
meshes = [
    { src = "models/level.obj", name = "level.mesh", format = "pos_col_uv_nrm" },
]
audio = [
    { src = "sfx/jump.wav", name = "jump.smp", format = "adpcm" },
    { src = "music/theme.wav", name = "theme.smp", format = "pcm16" },
]
shaders = [
    { src = "shaders/phong.shader", name = "phong_vert.shd", type = "vertex" },
    { src = "shaders/phong_frag.shader", name = "phong_frag.shd", type = "fragment" },
]
tilemaps = [
    { src = "levels/world1.json", name = "world1.lvl", parser = "tiled" },
]
```

---

## 11. ROM Packaging Toolchain

### 11.1 rompack — ROM Packer

```
Usage: rompack game.nxbin -o game.nxrom -c build.toml [--compress] [--no-validate]
```

`rompack` is the final stage of the build pipeline. It takes the linked binary (`.nxbin`), all converted asset files, and the `build.toml` manifest, and produces a complete `.nxrom` ROM image.

**Process:**

1. Reads the `.nxbin` and extracts the code and data segments.
2. Reads all asset files listed in `build.toml`.
3. Optionally LZ4-compresses each asset (if `--compress` is set).
4. Constructs the asset directory table.
5. Assembles the ROM: header → code segment → data segment → asset directory → asset data.
6. Computes CRC32 checksums for the header and full ROM body.
7. Validates the final ROM against console constraints:
   - Code segment ≤ 4 MB.
   - Data segment ≤ 4 MB.
   - Total ROM ≤ 256 MB.
   - Entry point falls within code segment range.
   - All asset target addresses are within valid memory regions.
8. Writes the `.nxrom` file.

### 11.2 romcheck — ROM Validator

```
Usage: romcheck game.nxrom [--verbose]
```

A standalone validation tool that verifies an existing ROM without building it:

- Checks magic bytes and format version.
- Verifies header checksum and full-ROM checksum.
- Validates segment offsets and sizes (no overlaps, within ROM bounds).
- Confirms all asset directory entries point to valid offsets with correct sizes.
- Verifies that the entry point address is within the code segment's loaded range.
- Disassembles the first 16 instructions at the entry point as a sanity check (looking for valid opcodes, not trap/halt as first instruction).
- Reports any errors or warnings.

Exit code 0 = valid ROM. Exit code 1 = errors found.

### 11.3 rominspect — ROM Inspector

```
Usage: rominspect game.nxrom [--assets] [--header] [--disasm N]
```

A developer tool for examining ROM contents:

- `--header`: Dumps all header fields in human-readable format.
- `--assets`: Lists all assets with name, type, format, compressed/uncompressed size, target region.
- `--disasm N`: Disassembles N instructions starting from the entry point.
- Default (no flags): Shows header summary and memory usage breakdown.

**Example output:**

```
NEXUS-32 ROM Inspector — game.nxrom
═══════════════════════════════════
Title:       My Game
Author:      Studio Name
Format:      v1.0
Total Size:  12,345,678 bytes (11.8 MB)
Checksum:    OK

Memory Usage:
  Code:      312,400 bytes (305 KB)
  Data:      48,200 bytes (47 KB)
  Assets:    11,984,950 bytes (11.4 MB)
    Textures:  8,200,000 bytes (24 files)
    Meshes:    2,100,000 bytes (8 files)
    Audio:     1,600,000 bytes (12 files)
    Shaders:   84,950 bytes (6 files)

VRAM Budget:  9.2 MB used / 16 MB available
Audio RAM:    1.5 MB used / 8 MB available
```

---

## 12. Emulator Architecture Notes

### 12.1 Core Emulation Loop

The emulator's main loop runs once per frame (targeting 60 Hz):

```
while (running) {
    1. Poll host input → write to virtual controller memory (0x0B004000)
    2. Fire input poll interrupt (IRQ 2)
    3. Run CPU for cycle_budget cycles:
       - Fetch/decode/execute instructions
       - Process DMA transfers concurrently (advance DMA state each cycle)
       - Handle interrupts as they fire
    4. When CPU writes PRESENT to GPU_CONTROL:
       - Read GPU command buffer (0x0A000000, GPU_CB_SIZE bytes)
       - Translate each command to Vulkan draw calls
       - Submit Vulkan command buffer and present to swapchain
    5. Mix audio: read APU ring buffer, feed to host audio callback
    6. Fire VBlank interrupt (IRQ 6), increment frame counter
    7. Regulate to 60 FPS (sleep or busy-wait as needed)
}
```

If the CPU exceeds its cycle budget before writing PRESENT, the emulator forces a frame boundary: the current command buffer state is rendered as-is (potentially an incomplete frame), a warning is logged, and VBlank fires. This prevents infinite loops from hanging the emulator while giving the developer visible feedback that they've exceeded budget.

### 12.2 Vulkan Rendering Strategy

The emulator initializes a Vulkan device, swapchain (double-buffered, FIFO present mode), and a primary render pass with color and depth attachments.

**Command translation:**

| Virtual GPU Command | Vulkan Translation                                                                                                     |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| CMD_CLEAR           | vkCmdClearAttachments (or set loadOp = CLEAR on render pass)                                                           |
| CMD_BIND_TEXTURE    | Create/cache VkImage + VkImageView from VRAM data. Create VkSampler matching filter/wrap. Update descriptor set.       |
| CMD_SET_STATE       | Configure pipeline state: depth test/write, blend mode, cull mode. May require pipeline rebind (cached by state hash). |
| CMD_DRAW_TRIS       | Copy vertex data from VRAM to host vertex buffer. vkCmdBindVertexBuffers + vkCmdDraw.                                  |
| CMD_DRAW_INDEXED    | Copy vertex + index data. vkCmdBindVertexBuffers + vkCmdBindIndexBuffer + vkCmdDrawIndexed.                            |
| CMD_DRAW_SPRITES    | Convert sprites to 2 triangles each, write to vertex buffer. Draw as textured quads with alpha blend.                  |
| CMD_SET_SHADER      | Compile mini shader to SPIR-V (cached by shader index). Create/bind VkPipeline.                                        |
| CMD_SET_UNIFORM     | Write to push constants or uniform buffer.                                                                             |
| CMD_COPY_REGION     | vkCmdCopyImage or vkCmdBlitImage between VRAM-backed images.                                                           |
| CMD_PRESENT         | vkEndRenderPass, vkQueueSubmit, vkQueuePresentKHR.                                                                     |

**Texture caching:** The emulator maintains a texture cache keyed by (VRAM offset, width, height, format). On CMD_BIND_TEXTURE, it checks if a VkImage already exists for that key. If the VRAM data has been modified since the last cache entry (tracked via dirty flags on VRAM pages), the texture is re-uploaded. Cache eviction uses LRU when VRAM page count exceeds a configurable host-side limit.

**Pipeline caching:** Vulkan pipelines are expensive to create. The emulator caches pipelines by a hash of: vertex format, shader indices, blend mode, depth state, and cull mode. Pipeline creation happens on first use and is reused thereafter.

### 12.3 Mini Shader → SPIR-V Compilation

The emulator translates NEXUS-32 shader binary programs to SPIR-V at first use. The translation is straightforward because the mini shader ISA maps closely to SPIR-V operations:

| Mini Shader Op  | SPIR-V Equivalent                                     |
| --------------- | ----------------------------------------------------- |
| MOV             | OpCopyObject                                          |
| ADD             | OpFAdd                                                |
| SUB             | OpFSub                                                |
| MUL             | OpFMul                                                |
| MAD             | OpExtInst (Fma) or MUL+ADD                            |
| DP3             | OpDot (vec3)                                          |
| DP4             | OpDot (vec4)                                          |
| RSQ             | OpExtInst (InverseSqrt)                               |
| RCP             | OpFDiv (1.0 / src)                                    |
| MIN             | OpExtInst (FMin)                                      |
| MAX             | OpExtInst (FMax)                                      |
| CLAMP           | OpExtInst (FClamp)                                    |
| LERP            | OpExtInst (FMix)                                      |
| TEX             | OpImageSampleImplicitLod                              |
| CMP             | OpSelect with OpFOrdGreaterThanEqual                  |
| ABS/NEG/FRC/FLR | Corresponding OpExtInst (FAbs, FNegate, Fract, Floor) |
| EXP/LOG         | OpExtInst (Exp2, Log2)                                |

The compiled SPIR-V modules are cached by shader index. If a shader is updated in VRAM, the cache entry is invalidated and recompiled on next use.

### 12.4 Host-Side Enhancements

The emulator optionally provides visual enhancements that are invisible to the game:

- **Resolution scaling:** The render pass can target a larger backbuffer (2×, 4×, or native host resolution). The framebuffer dimensions reported to the game remain 640×480; only the Vulkan render target resolution changes.
- **Texture filtering:** Even if the game requests nearest-neighbor filtering, the emulator can optionally apply bilinear or anisotropic filtering for smoother textures.
- **Anti-aliasing:** MSAA (2×, 4×, 8×) configured on the Vulkan render pass.
- **Post-processing:** Optional passes for CRT scanline simulation, bloom, color grading, or vignette. These are emulator settings, not part of the console spec.

All enhancements are toggle-able in the emulator's settings menu and default to off (pixel-perfect rendering by default).

### 12.5 Audio Backend

The emulator reads the APU ring buffer from Main RAM and feeds it to the host audio system via an SDL2 audio callback (or equivalent). The callback requests 48 kHz stereo 16-bit samples. If the host audio device runs at a different rate (e.g., 44.1 kHz), the emulator performs linear interpolation resampling.

The emulator also simulates the APU's voice mixing: for each audio tick, it advances each active voice's playback position by the voice's pitch rate, reads the sample (applying format decoding for ADPCM), applies the ADSR envelope, multiplies by voice volume (L/R), and sums into the ring buffer. Reverb is implemented as a simple Schroeder reverb with configurable parameters.

### 12.6 Input Mapping

The emulator maps host input to virtual controllers via a configurable mapping table:

**Default keyboard mapping (Controller 1):**

| Host Key          | Virtual Button    |
| ----------------- | ----------------- |
| Arrow keys        | D-pad             |
| Z                 | A                 |
| X                 | B                 |
| A                 | X                 |
| S                 | Y                 |
| Q                 | L                 |
| W                 | R                 |
| Enter             | Start             |
| Backspace         | Select            |
| IJKL              | Right stick       |
| WASD (hold Shift) | Left stick analog |

**Gamepad mapping:** SDL2 game controller API. Standard Xbox/PS layout mapped directly. Multiple gamepads map to controllers 1–4 in connection order.

### 12.7 Save Data Persistence

The emulator maps the 64 KB EEPROM region (0x08000000–0x0800FFFF) to a file on the host filesystem: `<rom_name>.sav`. The file is loaded into the EEPROM memory region at startup (if it exists). On each PRESENT, if the EEPROM region has been modified (tracked via dirty flag), the emulator writes the entire 64 KB to disk. On emulator shutdown, a final flush occurs.

### 12.8 Debug Features

The emulator includes a built-in debugger accessible via a hotkey (F12 by default):

- **CPU Debugger:** Step through instructions one at a time. View all integer and vector register values. View disassembly around the current PC. Set breakpoints by address or symbol name. Watch memory addresses for reads/writes.
- **Memory Viewer:** Hex dump of any memory region. Search for byte patterns. View memory as different data types (uint8, uint16, uint32, float).
- **GPU Command Buffer Inspector:** View the current frame's command buffer as decoded commands. Highlight individual draw calls. Show vertex data and texture bindings for selected draw calls.
- **Performance Overlay:** Toggle an on-screen overlay showing: cycle usage (bar graph), draw call count, triangle count, VRAM usage, frame time, and audio voice utilization.
- **Debug Console:** Displays output from `debug_print()` calls in the game code. Supports emulator commands for modifying state at runtime.

---

## 13. Multi-Repository Project Structure

### 13.1 Repository Layout

The NEXUS-32 ecosystem is organized into five repositories:

**`nexus32-spec/`** — This specification document and supplementary materials.

```
nexus32-spec/
├── NEXUS32_Specification_v1.0.md     # This document
├── diagrams/                          # Architecture diagrams, memory maps
├── encoding-tables/                   # CSV exports of instruction encodings
└── CHANGELOG.md                       # Spec version history
```

**`nexus32-emulator/`** — The Vulkan-based emulator, written in C.

```
nexus32-emulator/
├── src/
│   ├── main.c                # Entry point, main loop, window management
│   ├── cpu/
│   │   ├── cpu.c             # CPU core: fetch, decode, execute
│   │   ├── cpu.h             # Register state, public API
│   │   ├── vector.c          # Vector unit implementation
│   │   └── disasm.c          # Disassembler for debug output
│   ├── gpu/
│   │   ├── gpu.c             # Command buffer parser, Vulkan rendering
│   │   ├── gpu.h
│   │   ├── shader_compiler.c # Mini shader → SPIR-V translation
│   │   ├── texture_cache.c   # VRAM texture caching
│   │   └── pipeline_cache.c  # Vulkan pipeline cache
│   ├── apu/
│   │   ├── apu.c             # Voice mixer, ADSR, reverb
│   │   └── apu.h
│   ├── dma/
│   │   ├── dma.c             # DMA controller simulation
│   │   └── dma.h
│   ├── input/
│   │   ├── input.c           # SDL2 input handling, mapping
│   │   └── input.h
│   ├── mem/
│   │   ├── memory.c          # Memory controller, address decoding, bus arbitration
│   │   └── memory.h
│   ├── debug/
│   │   ├── debugger.c        # Interactive debugger
│   │   ├── profiler.c        # Performance overlay
│   │   └── debug.h
│   └── platform/
│       ├── vulkan_init.c     # Vulkan boilerplate: instance, device, swapchain
│       └── audio_backend.c   # SDL2 audio callback
├── shaders/                   # GLSL shaders for emulator's own rendering (overlay, post-FX)
├── CMakeLists.txt
└── README.md
```

**`nexus32-sdk/`** — The game SDK.

```
nexus32-sdk/
├── compiler/
│   ├── nxcc/                  # C compiler source
│   └── nxasm/                 # Assembler source
├── linker/
│   └── nxld/                  # Linker source
├── lib/
│   ├── src/                   # Standard library source (sys.c, gfx.c, input.c, etc.)
│   ├── include/               # Public headers (nx_sys.h, nx_gfx.h, etc.)
│   └── libnx.a               # Pre-built library archive
├── tools/
│   ├── img2tex/               # Image → texture converter
│   ├── obj2mesh/              # Model → mesh converter
│   ├── wav2smp/               # Audio → sample converter
│   ├── map2lvl/               # Tilemap converter
│   ├── shaderc/               # Shader compiler
│   └── nxbuild/               # Build orchestrator
├── CMakeLists.txt
└── README.md
```

**`nexus32-romtools/`** — ROM packaging tools (separate for modularity).

```
nexus32-romtools/
├── src/
│   ├── rompack.c              # ROM packer
│   ├── romcheck.c             # ROM validator
│   ├── rominspect.c           # ROM inspector
│   └── lz4.c                  # LZ4 compression/decompression
├── CMakeLists.txt
└── README.md
```

**`nexus32-examples/`** — Sample games and demos.

```
nexus32-examples/
├── hello-world/               # Minimal: character in a room (~50 lines of C)
├── platformer/                # Simple 2D platformer with tilemap
├── 3d-scene/                  # 3D rotating model with lighting
├── audio-test/                # Audio playback and sequencer demo
├── sprite-stress/             # Sprite rendering benchmark (10,000 sprites)
├── shader-demo/               # Custom shader showcase
└── README.md
```

### 13.2 Build Dependencies

| Repository       | Dependencies                                                                |
| ---------------- | --------------------------------------------------------------------------- |
| nexus32-emulator | Vulkan SDK (1.3+), SDL2 (2.28+), C17 compiler (GCC/Clang)                   |
| nexus32-sdk      | C17 compiler for host tools, no external libs for compiler/assembler/linker |
| nexus32-romtools | C17 compiler, LZ4 library (bundled)                                         |
| nexus32-examples | nexus32-sdk, nexus32-romtools                                               |

### 13.3 Versioning Strategy

The specification has a version number (currently 1.0). The ROM format includes the `format_version` field. The emulator and SDK each target a specific spec version.

Compatibility rules:

- **ROM format versions are forward-compatible.** An emulator supporting spec 1.2 must be able to run ROMs built for spec 1.0 and 1.1.
- **New features are additive.** New GPU commands, new instruction encodings, and new I/O registers use reserved/unused bits and address ranges.
- **The spec version is bumped** when any observable behavior changes (new instructions, changed memory map, new GPU features).
- **The emulator reports its supported spec version** via SYS_VERSION register (0x0B006000), so games can query capabilities at runtime.

---

*End of specification.*
