# NEXUS-32 Specification Repository

This repository is **nexus32-spec**: the single source of truth for the NEXUS-32 fantasy game console — hardware, ROM format, SDK surface, and ecosystem layout.

## Specification Document

- **[NEXUS32_Specification_v1.0.md](NEXUS32_Specification_v1.0.md)** — Full hardware and ecosystem specification (CPU, memory, DMA, Virtual GPU, APU, input, ROM format, SDK, toolchain, multi-repo structure).
- **[CHANGELOG.md](CHANGELOG.md)** — Spec version history.

## Quickstart and Contracts

- **[specs/001-nexus32-spec-baseline/quickstart.md](specs/001-nexus32-spec-baseline/quickstart.md)** — How to use this repo, round-one deliverables, developer workflow, and links to contracts.
- **Contracts** (ROM format and version reporting): [specs/001-nexus32-spec-baseline/contracts/](specs/001-nexus32-spec-baseline/contracts/).
- **[CONTRIBUTING.md](CONTRIBUTING.md)** — How to propose spec changes, versioning, and CHANGELOG. [docs/implementation-checklist.md](docs/implementation-checklist.md) — Conformance self-check for emulator, SDK, and romtools.

## Supporting Directories

- **[diagrams/](diagrams/)** — Architecture (§1) and memory map (§3): [memory-map.md](diagrams/memory-map.md), [architecture.md](diagrams/architecture.md). See [diagrams/README.md](diagrams/README.md).
- **[encoding-tables/](encoding-tables/)** — CSV encodings from spec §2 (CPU/VU) and §5 (GPU/shaders): [integer-instructions.csv](encoding-tables/integer-instructions.csv), [vector-instructions.csv](encoding-tables/vector-instructions.csv), [gpu-commands.csv](encoding-tables/gpu-commands.csv) (GPU command types §5.2), [shader-opcodes.csv](encoding-tables/shader-opcodes.csv) (mini-shader opcodes §5.6). See [encoding-tables/README.md](encoding-tables/README.md).
- **[reference/](reference/)** — I/O and system register reference from spec §3, §8: [io-and-system-registers.md](reference/io-and-system-registers.md). See [reference/README.md](reference/README.md).

## Ecosystem Repositories

| Repository        | Purpose                    |
|-------------------|----------------------------|
| **nexus32-spec**  | This repo — spec & contracts |
| **nexus32-emulator** | Vulkan-based emulator (C) |
| **nexus32-sdk**   | Compiler, assembler, linker, standard library |
| **nexus32-romtools** | ROM pack, validate, inspect |
| **nexus32-examples** | Sample games and demos   |

See NEXUS32_Specification_v1.0.md §13 for layout and build dependencies.
