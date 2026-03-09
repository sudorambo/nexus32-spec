# NEXUS-32 Specification Repository

This repository is **nexus32-spec**: the single source of truth for the NEXUS-32 fantasy game console — hardware, ROM format, SDK surface, and ecosystem layout.

## Specification Document

- **[NEXUS32_Specification_v1.0.md](NEXUS32_Specification_v1.0.md)** — Full hardware and ecosystem specification (CPU, memory, DMA, Virtual GPU, APU, input, ROM format, SDK, toolchain, multi-repo structure).
- **[CHANGELOG.md](CHANGELOG.md)** — Spec version history.

## Quickstart and Contracts

- **[specs/001-nexus32-spec-baseline/quickstart.md](specs/001-nexus32-spec-baseline/quickstart.md)** — How to use this repo, round-one deliverables, developer workflow, and links to contracts.
- **Contracts** (ROM format and version reporting): [specs/001-nexus32-spec-baseline/contracts/](specs/001-nexus32-spec-baseline/contracts/).

## Supporting Directories

- **diagrams/** — Architecture diagrams and memory maps (see [diagrams/README.md](diagrams/README.md)).
- **encoding-tables/** — CSV exports of instruction encodings (see [encoding-tables/README.md](encoding-tables/README.md)).

## Ecosystem Repositories

| Repository        | Purpose                    |
|-------------------|----------------------------|
| **nexus32-spec**  | This repo — spec & contracts |
| **nexus32-emulator** | Vulkan-based emulator (C) |
| **nexus32-sdk**   | Compiler, assembler, linker, standard library |
| **nexus32-romtools** | ROM pack, validate, inspect |
| **nexus32-examples** | Sample games and demos   |

See NEXUS32_Specification_v1.0.md §13 for layout and build dependencies.
