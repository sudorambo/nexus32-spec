# NEXUS-32 System Architecture

**Source**: NEXUS32_Specification_v1.0.md §1 (System Overview & Architecture Summary)

High-level system bus and components. The NEXUS-32 is a fantasy game console with a virtual machine comprising CPU, Vector Unit, Memory Controller, DMA, Virtual GPU, APU, Input, Timer, and System Registers.

## System Bus (spec §1.1)

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

## Components (spec §1.2)

| Component | Description |
|-----------|-------------|
| NX-RISC CPU Core | 32-bit RISC, r0–r31, program counter; central processor |
| Vector Unit (VU) | 16×128-bit SIMD registers (v0–v15); inline with CPU instruction stream |
| Memory Controller | Main RAM (32 MB), VRAM (16 MB), Audio RAM (8 MB), EEPROM (64 KB); bus arbitration |
| DMA Controller | 8 channels; bulk transfer between regions; concurrent with CPU |
| Virtual GPU (VGP) | Command buffer (2 MB/frame); translated to Vulkan by emulator |
| APU | 48 voices; PCM/ADPCM from Audio RAM; stereo ring buffer at 48 kHz |
| Input Controller | 4 controllers; memory-mapped state; updated once per frame |
| Timer | 2 programmable timers + frame counter |
| System Registers | RNG, version, cycle counters, etc. |

For full semantics and interfaces, see NEXUS32_Specification_v1.0.md §1–§8.
