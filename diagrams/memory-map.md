# NEXUS-32 Memory Map

**Source**: NEXUS32_Specification_v1.0.md §3 (Memory Architecture)

32-bit address space. Only the following regions are backed by physical memory or I/O. All other addresses read as 0 and discard writes.

## Memory Map Table (spec §3.1)

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

\* VRAM and Audio RAM are CPU-accessible; primary data path should use DMA. Direct CPU access to VRAM incurs a 4-cycle penalty per access (spec §3.1).

## Main RAM Layout (spec §3.2)

| Address Range           | Size  | Purpose                |
| ----------------------- | ----- | ---------------------- |
| 0x00000000 – 0x0000001F | 32 B  | Interrupt Vector Table |
| 0x00000020 – 0x000003FF | ~1 KB | System reserved        |
| 0x00000400 – 0x003FFFFF | ~4 MB | Code segment           |
| 0x00400000 – 0x007FFFFF | 4 MB  | Data segment           |
| 0x00800000 – 0x017FFFFF | 16 MB | Heap                   |
| 0x01800000 – 0x01FEFFFF | ~8 MB | Audio ring buffer etc. |
| 0x01FF0000 – 0x01FFFFFF | 64 KB | Stack (grows down)      |

Default stack pointer: 0x01FFFFF0 (16-byte aligned). Audio ring buffer base: 0x01800000, 64 KB.

For register layouts and alignment rules, see NEXUS32_Specification_v1.0.md §3.
