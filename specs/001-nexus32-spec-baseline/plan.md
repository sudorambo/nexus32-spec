# Implementation Plan: NEXUS-32 Ecosystem Specification Baseline (Round Five)

**Branch**: `001-nexus32-spec-baseline` | **Date**: 2026-03-08 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `specs/001-nexus32-spec-baseline/spec.md`

**Previous rounds**: Round one: layout, ROM/spec-version contracts, quickstart. Round two: CPU/VU encoding tables, diagrams, encoding-table contract. Round three: GPU commands and shader opcodes CSVs. Round four: reference/ with I/O and system register reference (§3, §8).

## Summary

Round five adds **contributing and spec-change workflow** documentation: a CONTRIBUTING.md (or equivalent) at repository root that describes how to propose changes to the specification, when to bump the spec version, CHANGELOG hygiene, and links to the constitution and contracts. Optionally, a short **implementation checklist** (e.g. in docs/ or .specify/) that other repos (emulator, SDK, romtools) can use to self-check conformance (ROM format, SYS_VERSION, contracts). No new contracts; documentation only. Aligns with constitution “Spec-first changes” and “Constitution check.”

## Technical Context

**Language/Version**: N/A (spec repo). Markdown only.  
**Primary Dependencies**: None.  
**Storage**: CONTRIBUTING.md at repository root; optional checklist in docs/ or .specify/ (e.g. docs/implementation-checklist.md).  
**Testing**: Manual review that CONTRIBUTING reflects constitution and existing contracts.  
**Target Platform**: This repo: any (docs); consumers: maintainers, contributors, other-repo implementers.  
**Project Type**: Documentation / specification repository; round five adds governance and process docs.  
**Performance Goals**: N/A.  
**Constraints**: Content MUST align with constitution and contracts; no new behavioral requirements.  
**Scale/Scope**: Round five: CONTRIBUTING.md (spec-change workflow, versioning, CHANGELOG); optional implementation checklist for other repos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|--------|
| I. Specification as Single Source of Truth | Pass | CONTRIBUTING reinforces spec-first changes and versioning. |
| II. Determinism and Virtual Machine Contract | Pass | No change to VM contract. |
| III. Forward Compatibility and Additive Changes | Pass | Workflow doc references version and CHANGELOG rules. |
| IV. Component Boundaries and Interfaces | Pass | Optional checklist references contracts and interfaces. |
| V. Testability and Spec Compliance | Pass | Implementation checklist supports conformance self-check. |
| VI. Version and Capability Reporting | Pass | CONTRIBUTING documents when/how to bump version. |

No violations; no complexity tracking required.

## Project Structure

### Documentation (this feature)

```text
specs/001-nexus32-spec-baseline/
├── plan.md              # This file (Round Five)
├── research.md          # Phase 0 (Round Five decisions appended)
├── data-model.md        # Phase 1 (optional: Contributing / Checklist entity)
├── quickstart.md        # Phase 1 (Round Five section added)
├── contracts/
├── checklists/
└── tasks.md             # From /speckit.tasks for Round Five
```

### Repository root (Round Five additions)

```text
/
├── NEXUS32_Specification_v1.0.md
├── CHANGELOG.md
├── CONTRIBUTING.md          # Round five: spec-change workflow, versioning, CHANGELOG
├── diagrams/
├── encoding-tables/
├── reference/
├── docs/                    # Optional: implementation checklist
│   └── implementation-checklist.md
├── .specify/
├── specs/
└── README.md
```

**Structure Decision**: CONTRIBUTING.md at root is standard for open-source and contributor-facing repos. Optional implementation checklist can live in docs/ to keep root minimal, or in .specify/ if it is part of the Speckit workflow; research will decide.

## Complexity Tracking

Not applicable; no constitution violations.
