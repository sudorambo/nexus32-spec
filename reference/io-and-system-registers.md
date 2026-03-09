# I/O and System Register Reference

**Source**: [NEXUS32_Specification_v1.0.md](../NEXUS32_Specification_v1.0.md) — this document is a convenience reference only. The specification is authoritative; on any discrepancy, the spec wins.

---

## 1. Memory-mapped I/O regions (§3)

From NEXUS32_Specification_v1.0.md §3.1 (Memory Map). Only these regions are backed by physical memory or I/O; all other addresses return 0 on read and discard writes.

| Start Address | End Address | Size  | Region                  | Access     | Spec |
| ------------- | ----------- | ----- | ----------------------- | ---------- | ---- |
| 0x00000000    | 0x01FFFFFF  | 32 MB | Main RAM                | Read/Write | §3   |
| 0x04000000    | 0x04FFFFFF  | 16 MB | VRAM                    | Read/Write | §3   |
| 0x06000000    | 0x067FFFFF  | 8 MB  | Audio RAM               | Read/Write | §3   |
| 0x08000000    | 0x0800FFFF  | 64 KB | Save EEPROM             | Read/Write | §3   |
| 0x0A000000    | 0x0A1FFFFF  | 2 MB  | GPU Command Buffer      | Write-only | §3   |
| 0x0B000000    | 0x0B0000FF  | 256 B | GPU Control/Status Regs | Read/Write | §3   |
| 0x0B001000    | 0x0B0010FF  | 256 B | DMA Control Registers   | Read/Write | §3   |
| 0x0B002000    | 0x0B002FFF  | 4 KB  | DMA Descriptor Table    | Read/Write | §3   |
| 0x0B003000    | 0x0B003FFF  | 4 KB  | Audio Registers         | Read/Write | §3   |
| 0x0B004000    | 0x0B00407F  | 128 B | Input Controller State  | Read-only  | §3   |
| 0x0B005000    | 0x0B00503F  | 64 B  | Timer Registers         | Read/Write | §3   |
| 0x0B006000    | 0x0B00603F  | 64 B  | System Registers        | Read/Write | §3   |
| 0x0B007000    | 0x0B00701F  | 32 B  | Interrupt Control       | Read/Write | §3   |

---

## 2. GPU control registers (0x0B000000, §3.4)

From NEXUS32_Specification_v1.0.md §3.4. Base address: **0x0B000000**.

| Offset | Name           | Access | Description                                                        | Spec |
| ------ | -------------- | ------ | ------------------------------------------------------------------ | ---- |
| 0x00   | GPU_STATUS     | R      | Bit 0: busy (processing CB). Bit 1: VBlank active.                | §3.4 |
| 0x04   | GPU_CONTROL    | W      | Bit 0: submit frame (PRESENT). Bit 1: reset GPU.                  | §3.4 |
| 0x08   | GPU_CB_ADDR    | R/W    | Command buffer start address (default: 0x0A000000).                | §3.4 |
| 0x0C   | GPU_CB_SIZE    | R/W    | Bytes written to command buffer this frame.                        | §3.4 |
| 0x10   | GPU_FB_ADDR    | R/W    | Framebuffer 0 base address in VRAM.                                | §3.4 |
| 0x14   | GPU_FB1_ADDR   | R/W    | Framebuffer 1 base address (double buffering).                     | §3.4 |
| 0x18   | GPU_DB_ADDR    | R/W    | Depth buffer base address in VRAM.                                 | §3.4 |
| 0x1C   | GPU_WIDTH      | R/W    | Framebuffer width (default: 640).                                  | §3.4 |
| 0x20   | GPU_HEIGHT     | R/W    | Framebuffer height (default: 480).                                 | §3.4 |
| 0x24   | GPU_DRAW_CALLS | R      | Draw calls processed last frame (profiling).                       | §3.4 |
| 0x28   | GPU_VRAM_USED  | R      | VRAM bytes used for textures+shaders (profiling).                  | §3.4 |
| 0x30   | GPU_AUX_RT0    | R/W    | Auxiliary render target 0 VRAM address.                            | §5.7 |
| 0x34   | GPU_AUX_RT1    | R/W    | Auxiliary render target 1 VRAM address.                            | §5.7 |
| 0x38   | GPU_AUX_RT2    | R/W    | Auxiliary render target 2 VRAM address.                            | §5.7 |

Auxiliary render targets (0x30–0x3B) enable multi-pass rendering. Write a VRAM address to configure; switch active target via CMD_SET_RENDER_TARGET.

---

## 3. System registers (0x0B006000, §8.3)

From NEXUS32_Specification_v1.0.md §8.3. Base address: **0x0B006000**.

| Offset | Name          | Access | Description                                      | Spec  |
| ------ | ------------- | ------ | ------------------------------------------------ | ----- |
| 0x00   | SYS_VERSION   | R      | Console version: major (high 16) / minor (low 16). Current: 0x00010000. | §8.3  |
| 0x04   | SYS_RAM_SIZE  | R      | Main RAM size in bytes (0x02000000 = 32 MB).     | §8.3  |
| 0x08   | SYS_VRAM_SIZE | R      | VRAM size in bytes (0x01000000 = 16 MB).          | §8.3  |
| 0x0C   | SYS_ARAM_SIZE | R      | Audio RAM size in bytes (0x00800000 = 8 MB).     | §8.3  |
| 0x10   | SYS_RNG       | R      | Hardware RNG. Each read returns a new pseudo-random 32-bit value. | §8.3  |
| 0x14   | SYS_CYCLES    | R      | Cycles consumed this frame (for profiling).      | §8.3  |
| 0x18   | SYS_CYCLE_MAX | R      | Cycle budget for this frame.                     | §8.3  |

---

## 4. Frame counter (0x0B005020, §8.2)

From NEXUS32_Specification_v1.0.md §8.2. Base address: **0x0B005000**; frame counter at offset 0x20.

| Offset | Name        | Access | Description                              | Spec  |
| ------ | ----------- | ------ | ---------------------------------------- | ----- |
| 0x20   | FRAME_COUNT  | R      | 32-bit counter, increments every VBlank. | §8.2  |

Read-only. Overflows after ~2.27 years at 60 FPS.

---

## 5. Interrupt control registers (0x0B007000, §8.4)

From NEXUS32_Specification_v1.0.md §8.4. Base address: **0x0B007000**.

| Offset | Name        | Access | Description                                    | Spec  |
| ------ | ----------- | ------ | ---------------------------------------------- | ----- |
| 0x00   | IRQ_PENDING | R      | Bitmask of pending interrupts (8 bits).         | §8.4  |
| 0x04   | IRQ_ENABLE  | R/W    | Bitmask of enabled interrupts (mirrors SR.IM).  | §8.4  |
| 0x08   | IRQ_ACK     | W      | Write bitmask to acknowledge/clear pending IRQs.| §8.4  |
