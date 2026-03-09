# Quickstart: NEXUS-32 Spec Repository and Round-One Deliverables

**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

## Round One Deliverables (Index)

| Document | Description |
|----------|-------------|
| [plan.md](plan.md) | Implementation plan: tech context, constitution check, project structure |
| [spec.md](spec.md) | Feature specification: user stories, requirements, success criteria |
| [research.md](research.md) | Research and decisions: spec-as-SSOT, versioning, contracts |
| [data-model.md](data-model.md) | Entities and relationships: Specification, ROM, versioning, etc. |
| [contracts/](contracts/) | ROM format and spec/version contracts for implementers |
| [quickstart.md](quickstart.md) | This document: how to use the spec repo and contracts |
| [tasks.md](tasks.md) | Task list for round-one implementation |

This repository is the **NEXUS-32 specification repository** (nexus32-spec). It holds the single source of truth for the NEXUS-32 fantasy game console and the contracts that emulator, SDK, and romtools implement against.

## What’s in This Repo

- **[NEXUS32_Specification_v1.0.md](../../NEXUS32_Specification_v1.0.md)** — Full hardware and ecosystem specification (CPU, memory, DMA, GPU, APU, input, ROM format, SDK, toolchain, multi-repo layout).
- **CHANGELOG.md** — Spec version history.
- **diagrams/** — Architecture and memory map diagrams.
- **encoding-tables/** — Instruction encoding exports, e.g. CSV.
- **specs/** — Feature specs and implementation plans (e.g. this baseline and plan).
- **.specify/** — Speckit workflow (constitution, templates, scripts).

## Interface Index (Traceability)

Key interfaces and where they are defined in the main specification:

| Interface | Spec section | Description |
|-----------|--------------|-------------|
| Memory map | §3 | Main RAM, VRAM, Audio RAM, EEPROM, GPU command buffer, I/O registers |
| ROM header & format | §9 | Magic, format_version, segments, checksums, validation |
| GPU commands | §5 | Command buffer layout, CMD_* types, vertex/sprite formats |
| SYS_VERSION / version reporting | §8.3, §13.3 | System registers at 0x0B006000; spec vs. ROM format version |
| CPU/VU instruction set | §2 | Integer and vector encodings, cycle costs |
| DMA descriptors & registers | §4 | Descriptor format, channels, control registers |
| APU / audio registers | §6 | Voice layout, ring buffer, sample formats |
| Input controller state | §7 | Per-controller layout at 0x0B004000 |
| Timer & system registers | §8 | Timers, frame counter, RNG, interrupt control |

When assessing conformance, every defined interface above MUST be covered by the specification.

**Conformance review**: When assessing an implementation (emulator, SDK, or tool), every defined interface (memory map, registers, GPU commands, ROM layout) MUST be covered by the specification. Use the Interface index above to trace each interface to its spec section; no behavior may be introduced that is not defined or permitted by the spec.

## Round-One Deliverables

1. **Spec layout** — Ensure `CHANGELOG.md`, `diagrams/`, and `encoding-tables/` exist and are referenced from the main spec.
2. **Contracts** — ROM format and version reporting:
   - [contracts/rom-format.md](contracts/rom-format.md): ROM header layout, validation rules, forward compatibility.
   - [contracts/spec-version.md](contracts/spec-version.md): Spec version, ROM format version, compatibility rules, SYS_VERSION.
3. **Quickstart** — This document.

## Developer Workflow (Build and Run)

The standard workflow for game developers is:

1. **Develop** — Write game code and prepare assets; use the SDK (compiler, assembler, linker, standard library) to produce object files and link them.
2. **Build ROM** — Use the SDK build orchestrator and romtools to pack code, data, and assets into a ROM image (`.nxrom`) with a valid header per [contracts/rom-format.md](contracts/rom-format.md).
3. **Validate** — Run romtools validator (e.g. `romcheck`) to verify the ROM: magic "NX32", supported format_version (e.g. 0x0100), checksums, and segment layout per [contracts/rom-format.md](contracts/rom-format.md) and [contracts/spec-version.md](contracts/spec-version.md).
4. **Run** — Load the ROM in the emulator; the emulator reports its supported spec version via SYS_VERSION; the game runs with deterministic behavior per the spec.

**Build-and-run checklist**: Produce a ROM with valid header (magic `NX32`, format_version `0x0100` for spec 1.0); validate with romtools per the ROM format contract; run on the emulator. All steps are defined in the main spec §9 and the contracts above.

## How to Use the Spec

- **Implementing the emulator**: Read NEXUS32_Specification_v1.0.md; implement CPU, memory, DMA, GPU, APU, input, timers per spec; load ROMs per §9 and [contracts/rom-format.md](contracts/rom-format.md); report version per [contracts/spec-version.md](contracts/spec-version.md).
- **Implementing romtools**: Build ROM packer/validator per §9 and [contracts/rom-format.md](contracts/rom-format.md); accept/reject ROMs per validation rules; report unsupported format_version clearly.
- **Implementing the SDK**: Produce ROMs with valid header and segments per spec §9; use format_version 0x0100 for spec 1.0.
- **Checking compatibility**: Emulator reports spec version via SYS_VERSION; ROM declares format_version in header; tools compare against supported versions per [contracts/spec-version.md](contracts/spec-version.md).

## Other Repositories

The ecosystem has five repos (see spec §13):

| Repo              | Purpose                          |
|-------------------|-----------------------------------|
| nexus32-spec      | This repo — spec and contracts    |
| nexus32-emulator  | Vulkan emulator (C)               |
| nexus32-sdk       | Compiler, assembler, linker, lib  |
| nexus32-romtools  | ROM pack/validate/inspect         |
| nexus32-examples  | Sample games and demos            |

Dependencies: emulator (Vulkan, SDL2); SDK (none for core tools); romtools (LZ4); examples (SDK, romtools).

## Conformance Verification

- **ROM validation (romtools)**: Use [contracts/rom-format.md](contracts/rom-format.md) as the reference for accept/reject rules. Validators MUST accept ROMs that satisfy the contract (magic, supported format_version, checksums, offsets/sizes within file and within limits) and MUST reject invalid or unsupported-version ROMs with clear feedback. See main spec §9.
- **Emulator behavior**: The emulator MUST match the specification for CPU, GPU, DMA, and APU semantics (instruction execution, memory map, command buffer processing, audio mixing). Observable behavior (registers, memory, display, audio) is defined in the main spec; see §2 (CPU/VU), §5 (GPU), §4 (DMA), §6 (APU), and §12 (emulator architecture notes).

## Constitution and Workflow

- All implementations MUST conform to the spec; no behavior outside the spec.
- Spec changes that affect observable behavior require a spec version bump and a CHANGELOG entry.
- ROM format is forward-compatible; new features use reserved bits and additive definitions.
- See [.specify/memory/constitution.md](../../.specify/memory/constitution.md) for full governance.

## End User Experience

When running a valid NEXUS-32 ROM in the emulator, end users can expect:

- **Frame rate and cycle budget**: The game runs at the specified frame rate (e.g. 60 FPS) with a fixed cycle budget per frame (default 3M cycles; configurable per ROM in the header). See main spec §2.5 and §9 (cycle_budget).
- **Input**: Up to 4 controllers; state is memory-mapped and read once per frame per spec §7. The emulator maps host input (keyboard, gamepad) to the virtual controller layout.
- **Save data (EEPROM)**: A 64 KB EEPROM region is memory-mapped at 0x08000000–0x0800FFFF. The emulator persists this region to a host save file (e.g. `<rom_name>.sav`) and loads it at startup; see main spec §9.9 and §12.7. End users can expect save data to persist between sessions when the game writes to the EEPROM region.

## Save Data (EEPROM) and .sav File

Per NEXUS32_Specification_v1.0.md §9.9 and §12.7: the 64 KB EEPROM is mapped to a file on the host (e.g. `<rom_name>.sav`). The file is loaded into the EEPROM at emulator startup if it exists; on each PRESENT (or periodically), if the EEPROM has been modified, the emulator writes the 64 KB to disk. On shutdown, a final flush occurs. This defines end-user persistence expectations for save games.

## Next Steps After Round One

- Implement or extend **nexus32-romtools** to validate ROMs against [contracts/rom-format.md](contracts/rom-format.md).
- Implement or extend **nexus32-emulator** to load ROMs per the contract and report SYS_VERSION per [contracts/spec-version.md](contracts/spec-version.md).
- Use **nexus32-sdk** and **nexus32-romtools** to build a minimal reference ROM and run it on the emulator to satisfy success criteria SC-002 and SC-004.
