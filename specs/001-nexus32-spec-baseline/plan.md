# Implementation Plan: NEXUS-32 Ecosystem Specification Baseline (Round Two)

**Branch**: `001-nexus32-spec-baseline` | **Date**: 2026-03-08 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `specs/001-nexus32-spec-baseline/spec.md`

**Previous round**: Round one delivered spec repo layout (CHANGELOG, diagrams/, encoding-tables/, README), ROM format and spec-version contracts, quickstart, interface index, and conformance/versioning docs. All tasks in tasks.md were completed.

## Summary

Round two extends the spec repository with **machine-readable encoding tables** (CSV exports of CPU/VU instruction encodings from spec §2) and **diagram placeholders or reference diagrams** (architecture and memory map per spec §1, §3) so that emulator and SDK implementers have a single, spec-derived source for decoders and visual reference. Optional: a small contract for encoding-table format. No application code in this repo; documentation and data only.

## Technical Context

**Language/Version**: N/A (spec repo). Encoding tables: CSV (UTF-8); diagrams: Markdown with Mermaid or ASCII art, or placeholder READMEs referencing spec figures.  
**Primary Dependencies**: None; CSV and Markdown are human- and tool-readable.  
**Storage**: Files in `encoding-tables/` and `diagrams/` at repository root.  
**Testing**: Manual check that encoding table rows match NEXUS32_Specification_v1.0.md §2.3 and §2.4; diagram references match spec sections.  
**Target Platform**: This repo: any (docs/data); consumers: emulator, SDK (e.g. assembler, disassembler).  
**Project Type**: Documentation / specification repository; round two adds spec-derived data (encoding tables) and diagram assets.  
**Performance Goals**: N/A.  
**Constraints**: Encoding tables and diagrams MUST be derived from the spec only; no new behavior or encoding.  
**Scale/Scope**: Round two: populate encoding-tables/ with integer and vector instruction encoding CSVs; add at least one memory-map and one architecture diagram (or placeholders with spec § references); optional encoding-table contract; update quickstart.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|--------|
| I. Specification as Single Source of Truth | Pass | Encoding tables and diagrams are derived from the spec; no new encodings or interfaces. |
| II. Determinism and Virtual Machine Contract | Pass | No change to VM contract. |
| III. Forward Compatibility and Additive Changes | Pass | No change to version or ROM contracts. |
| IV. Component Boundaries and Interfaces | Pass | Encoding tables document CPU/VU interfaces per spec; optional contract for table format. |
| V. Testability and Spec Compliance | Pass | Tables enable automated decoder/assembler tests against spec. |
| VI. Version and Capability Reporting | Pass | Tables and diagrams are tied to spec version (1.0). |

No violations; no complexity tracking required.

## Project Structure

### Documentation (this feature)

```text
specs/001-nexus32-spec-baseline/
├── plan.md              # This file (Round Two)
├── research.md          # Phase 0 (Round Two decisions appended)
├── data-model.md        # Phase 1 (optional: encoding table entity)
├── quickstart.md        # Phase 1 (Round Two section added)
├── contracts/           # Optional: encoding-table-format.md
├── checklists/
│   └── requirements.md
└── tasks.md             # From /speckit.tasks for Round Two
```

### Source Code (repository root — spec repo, Round Two additions)

```text
/
├── NEXUS32_Specification_v1.0.md
├── CHANGELOG.md
├── diagrams/                    # Round two: add memory-map + architecture diagram or placeholders
│   ├── README.md                # (existing)
│   ├── memory-map.md            # Memory map per spec §3 (new or placeholder)
│   └── architecture.md          # System bus / components per spec §1 (new or placeholder)
├── encoding-tables/             # Round two: add CSV exports from spec §2
│   ├── README.md                # (existing)
│   ├── integer-instructions.csv # R/I/J/S-type encodings per spec §2.3 (new)
│   └── vector-instructions.csv  # V-type encodings per spec §2.4 (new)
├── .specify/
├── specs/
└── README.md
```

**Structure Decision**: Same spec-only repo. Round two adds concrete files under `encoding-tables/` and `diagrams/` so that implementers can consume spec-derived data without re-extracting from the main spec document.

## Complexity Tracking

Not applicable; no constitution violations.
