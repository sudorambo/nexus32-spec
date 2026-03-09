# Implementation Plan: NEXUS-32 Ecosystem Specification Baseline (Round Four)

**Branch**: `001-nexus32-spec-baseline` | **Date**: 2026-03-08 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `specs/001-nexus32-spec-baseline/spec.md`

**Previous rounds**: Round one: spec repo layout, ROM/spec-version contracts, quickstart. Round two: CPU/VU encoding tables, diagrams (memory-map, architecture), encoding-table contract. Round three: GPU commands CSV, shader opcodes CSV, README/quickstart updates.

## Summary

Round four adds a **memory-mapped I/O and system register reference** to the spec repo: a single consolidated reference document (Markdown) that lists I/O regions (base address, size, name, spec section) and system/timer/interrupt registers (offset, name, access, description, spec section) from NEXUS32_Specification_v1.0.md §3 and §8. This gives emulator and SDK implementers one place to look up base addresses and key register layouts without re-scanning the full spec. No new behavior or interfaces; documentation only. Optional: CONTRIBUTING or spec-change workflow note.

## Technical Context

**Language/Version**: N/A (spec repo). Markdown; no code.  
**Primary Dependencies**: None.  
**Storage**: New file(s) at repository root, e.g. `reference/io-registers.md` or `reference-tables/io-and-system-registers.md` (directory and name TBD in research).  
**Testing**: Manual check that every row/section cites the spec and matches spec §3 (memory map) and §8 (timer, system, interrupt registers).  
**Target Platform**: This repo: any (docs); consumers: emulator, SDK (register headers, debuggers).  
**Project Type**: Documentation / specification repository; round four adds reference documentation.  
**Performance Goals**: N/A.  
**Constraints**: Reference MUST be derived from the spec only; spec is authoritative.  
**Scale/Scope**: Round four: one consolidated I/O + system register reference; optional CONTRIBUTING or workflow note. No new CSVs in encoding-tables; this is a prose/table reference.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|--------|
| I. Specification as Single Source of Truth | Pass | Reference is derived from spec §3, §8; no new addresses or behavior. |
| II. Determinism and Virtual Machine Contract | Pass | No change to VM contract. |
| III. Forward Compatibility and Additive Changes | Pass | No change to version or ROM contracts. |
| IV. Component Boundaries and Interfaces | Pass | Reference documents existing I/O and system interfaces per spec. |
| V. Testability and Spec Compliance | Pass | Enables implementers to verify register layout and addresses against spec. |
| VI. Version and Capability Reporting | Pass | Reference tied to spec version (1.0). |

No violations; no complexity tracking required.

## Project Structure

### Documentation (this feature)

```text
specs/001-nexus32-spec-baseline/
├── plan.md              # This file (Round Four)
├── research.md          # Phase 0 (Round Four decisions appended)
├── data-model.md        # Phase 1 (optional: I/O register reference entity)
├── quickstart.md        # Phase 1 (Round Four section added)
├── contracts/
├── checklists/
└── tasks.md             # From /speckit.tasks for Round Four
```

### Repository root (Round Four additions)

```text
/
├── NEXUS32_Specification_v1.0.md
├── CHANGELOG.md
├── diagrams/
├── encoding-tables/
├── reference/                    # Round four: I/O and system register reference
│   ├── README.md                 # (optional) Purpose and spec ref
│   └── io-and-system-registers.md  # Consolidated table(s) from §3, §8
├── .specify/
├── specs/
└── README.md
```

**Structure Decision**: Add a `reference/` directory (or equivalent) for consolidated reference docs. Round four adds one file: I/O regions + system/timer/interrupt registers, all citing spec sections. No new contract unless we later add a “reference document format” contract; for round four, README in reference/ or a short intro in the doc suffices.

## Complexity Tracking

Not applicable; no constitution violations.
