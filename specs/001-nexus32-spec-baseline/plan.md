# Implementation Plan: NEXUS-32 Ecosystem Specification Baseline (Round Three)

**Branch**: `001-nexus32-spec-baseline` | **Date**: 2026-03-08 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `specs/001-nexus32-spec-baseline/spec.md`

**Previous rounds**: Round one delivered spec repo layout, ROM/spec-version contracts, quickstart, interface index. Round two delivered CPU/VU encoding tables (integer-instructions.csv, vector-instructions.csv), diagrams (memory-map.md, architecture.md), and encoding-table contract.

## Summary

Round three adds **GPU command** and **mini-shader opcode** encoding references to the spec repo: CSV (or equivalent) exports of GPU command types (spec §5.2) and mini-shader instruction opcodes (spec §5.6) so that emulator and SDK shader/command-buffer tooling have a single, spec-derived source. No new behavior or interfaces; documentation and data only. Optional: short contract or README extension for GPU/shader table format.

## Technical Context

**Language/Version**: N/A (spec repo). Same as round two: CSV UTF-8 for encoding tables.  
**Primary Dependencies**: None.  
**Storage**: New files in `encoding-tables/` at repository root: `gpu-commands.csv`, `shader-opcodes.csv` (or equivalent).  
**Testing**: Manual check that table rows match NEXUS32_Specification_v1.0.md §5.2 and §5.6.  
**Target Platform**: This repo: any (docs/data); consumers: emulator (command buffer parser, shader compiler), SDK (shaderc, tools).  
**Project Type**: Documentation / specification repository; round three extends encoding-tables with GPU and shader data.  
**Performance Goals**: N/A.  
**Constraints**: Tables MUST be derived from the spec only; spec is authoritative.  
**Scale/Scope**: Round three: add gpu-commands.csv (cmd_type, name, size_byte, spec_ref) and shader-opcodes.csv (opcode_hex, mnemonic, operation, spec_ref); update encoding-tables README and quickstart.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|--------|
| I. Specification as Single Source of Truth | Pass | GPU and shader tables are derived from spec §5; no new encodings. |
| II. Determinism and Virtual Machine Contract | Pass | No change to VM contract. |
| III. Forward Compatibility and Additive Changes | Pass | No change to version or ROM contracts. |
| IV. Component Boundaries and Interfaces | Pass | Tables document VGP command and shader interfaces per spec. |
| V. Testability and Spec Compliance | Pass | Tables enable command-buffer and shader-compiler tests against spec. |
| VI. Version and Capability Reporting | Pass | Tables tied to spec version (1.0). |

No violations; no complexity tracking required.

## Project Structure

### Documentation (this feature)

```text
specs/001-nexus32-spec-baseline/
├── plan.md              # This file (Round Three)
├── research.md          # Phase 0 (Round Three decisions appended)
├── data-model.md        # Phase 1 (optional: GPU command / shader opcode entities)
├── quickstart.md        # Phase 1 (Round Three section added)
├── contracts/           # Optional: extend or reference encoding-table contract
├── checklists/
│   └── requirements.md
└── tasks.md             # From /speckit.tasks for Round Three
```

### Source Code (repository root — spec repo, Round Three additions)

```text
/
├── NEXUS32_Specification_v1.0.md
├── CHANGELOG.md
├── diagrams/
│   ├── README.md
│   ├── memory-map.md
│   └── architecture.md
├── encoding-tables/
│   ├── README.md
│   ├── integer-instructions.csv
│   ├── vector-instructions.csv
│   ├── gpu-commands.csv          # Round three: cmd_type, name, size, spec_ref (§5.2)
│   └── shader-opcodes.csv        # Round three: opcode, mnemonic, operation (§5.6)
├── .specify/
├── specs/
└── README.md
```

**Structure Decision**: Same spec-only repo. Round three adds two CSV files under `encoding-tables/` for GPU command types and mini-shader opcodes, following the same “spec-derived, spec authoritative” pattern as round two.

## Complexity Tracking

Not applicable; no constitution violations.
