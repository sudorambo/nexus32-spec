# Tasks: NEXUS-32 Ecosystem Specification Baseline (Round One)

**Input**: Design documents from `specs/001-nexus32-spec-baseline/`  
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: Not requested in the feature specification; no test tasks included.

**Organization**: Tasks are grouped by user story to enable independent implementation and validation of each story. This repository is documentation-only (spec repo); all tasks create or update docs and contracts.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

- **Spec repo root**: `/` = repository root (NEXUS32_Specification_v1.0.md, CHANGELOG.md, diagrams/, encoding-tables/, README.md)
- **Feature docs**: `specs/001-nexus32-spec-baseline/` (plan.md, spec.md, research.md, data-model.md, quickstart.md, contracts/)

---

## Phase 1: Setup (Spec Repository Structure)

**Purpose**: Initialize spec repo layout per plan so the main spec is the single source of truth with version history and supporting directories.

- [x] T001 Create CHANGELOG.md at repository root with initial entry for spec version 1.0 and date (reference NEXUS32_Specification_v1.0.md)
- [x] T002 Create diagrams/ directory at repository root and add README.md in diagrams/ describing it holds architecture diagrams and memory maps per spec §13.1
- [x] T003 Create encoding-tables/ directory at repository root and add README.md in encoding-tables/ describing it holds CSV exports of instruction encodings per spec §13.1
- [x] T004 [P] Create or update README.md at repository root with project title, link to NEXUS32_Specification_v1.0.md, link to specs/001-nexus32-spec-baseline/quickstart.md, and list of ecosystem repos (nexus32-emulator, nexus32-sdk, nexus32-romtools, nexus32-examples) per spec §13

**Checkpoint**: Spec repo has CHANGELOG.md, diagrams/, encoding-tables/, and root README.md

---

## Phase 2: Foundational (Contracts and Traceability)

**Purpose**: Ensure contracts and spec references are complete so implementers can validate ROMs and conformance.

**⚠️ CRITICAL**: User story documentation and refinements depend on these foundations.

- [x] T005 Verify specs/001-nexus32-spec-baseline/contracts/rom-format.md cites NEXUS32_Specification_v1.0.md §9 for every validation rule and header field
- [x] T006 Verify specs/001-nexus32-spec-baseline/contracts/spec-version.md cites spec §8.3 and §13.3 and constitution principles III and VI
- [x] T007 Add a short "Interface index" to specs/001-nexus32-spec-baseline/quickstart.md or data-model.md mapping key interfaces (memory map, ROM header, SYS_VERSION, GPU commands) to main spec section numbers for traceability

**Checkpoint**: Contracts are verified and traceability from interfaces to spec sections is documented

---

## Phase 3: User Story 1 — Spec as Single Source of Truth (Priority: P1) 🎯 MVP

**Goal**: Maintainers and implementers can treat the spec document as the single authoritative reference; every behavioral requirement traces to a spec section; changes to observable behavior are versioned in CHANGELOG.

**Independent Test**: Review any proposed behavior change and confirm it is documented in the specification and reflected in CHANGELOG before implementation; confirm interface index covers memory map, registers, command formats, ROM layout.

- [x] T008 [US1] In CHANGELOG.md at repository root, add a "Versioning" subsection describing when to bump spec version (observable behavior change) and how to add entries (date, version, summary)
- [x] T009 [US1] Ensure NEXUS32_Specification_v1.0.md header (top of file) contains document version (1.0) and date; add if missing
- [x] T010 [US1] Add "Conformance review" note to specs/001-nexus32-spec-baseline/quickstart.md: when assessing an implementation, every defined interface (memory map, registers, GPU commands, ROM layout) must be covered by the specification

**Checkpoint**: Spec versioning and conformance traceability are documented; US1 acceptance scenarios can be checked via CHANGELOG and interface index

---

## Phase 4: User Story 2 — Game Developers Can Build and Run ROMs (Priority: P2)

**Goal**: The workflow "develop with SDK → build ROM → validate with romtools → run on emulator" is clearly documented so game developers know how to produce and run ROMs; contracts support this workflow.

**Independent Test**: A reader can follow quickstart and contracts to understand how to build and run a ROM once SDK, romtools, and emulator exist; no ambiguity on ROM format or version checks.

- [x] T011 [US2] In specs/001-nexus32-spec-baseline/quickstart.md, add or expand "Developer workflow" subsection: SDK builds ROM → romtools pack/validate → emulator loads and runs; link to contracts/rom-format.md and contracts/spec-version.md
- [x] T012 [US2] In specs/001-nexus32-spec-baseline/quickstart.md, add "Build and run" checklist: produce ROM with valid header (magic, format_version 0x0100), validate with romtools per contract, run on emulator; reference spec §9 and contracts

**Checkpoint**: Build-and-run workflow and checklist are in quickstart; US2 is document-complete for round one

---

## Phase 5: User Story 3 — Emulator and Tools Conform to the Spec (Priority: P3)

**Goal**: Implementers can verify ROM validation (format, checksums) and emulator behavior against the spec; contracts and quickstart describe how to check conformance.

**Independent Test**: ROM validation rules in contracts match spec §9; quickstart explains how to verify romtools accept/reject and how emulator behavior is checked against the spec.

- [x] T013 [US3] In specs/001-nexus32-spec-baseline/quickstart.md, add "Conformance verification" subsection: ROM validation (romtools) uses contracts/rom-format.md; emulator must match spec for CPU, GPU, DMA, APU semantics; link to contracts and spec §9, §12
- [x] T014 [US3] Confirm specs/001-nexus32-spec-baseline/contracts/rom-format.md validation rules (accept/reject) are complete per NEXUS32_Specification_v1.0.md §9; add any missing rule (e.g. code_size/data_size limits, reserved bytes) and note spec section

**Checkpoint**: Conformance verification is documented; ROM contract validation rules are complete; US3 is document-complete for round one

---

## Phase 6: User Story 4 — End Users Run ROMs with Correct Experience (Priority: P4)

**Goal**: End-user expectations (frame rate, cycle budget, input, save persistence) are documented so that implementers and users know what "correct" behavior means per the spec.

**Independent Test**: Quickstart or a dedicated subsection describes what end users can expect when running a valid ROM (timing, input, save data persistence) with spec references.

- [x] T015 [US4] In specs/001-nexus32-spec-baseline/quickstart.md, add "End user experience" subsection: valid ROM runs at specified frame rate and cycle budget, input per spec §7, save data (EEPROM) persisted per spec and emulator persistence rules; link to main spec sections
- [x] T016 [US4] Document in quickstart.md or README: EEPROM region and save file behavior (e.g. .sav file) per NEXUS32_Specification_v1.0.md so end-user persistence expectations are clear

**Checkpoint**: End-user experience and save persistence are documented; US4 is document-complete for round one

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Validate links, cross-check artifacts, and ensure the feature folder is navigable.

- [x] T017 [P] Run through specs/001-nexus32-spec-baseline/quickstart.md manually; fix any broken links, missing paths, or outdated repo names
- [x] T018 [P] Cross-check specs/001-nexus32-spec-baseline/data-model.md entities (Specification, ROM, ROM Header, etc.) against specs/001-nexus32-spec-baseline/contracts/ and main spec; add missing references or fix inconsistencies
- [x] T019 Add a short index or "Round one deliverables" section at the top of specs/001-nexus32-spec-baseline/plan.md or quickstart.md listing plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md, and tasks.md with one-line descriptions

**Checkpoint**: All links and references are valid; data model and contracts are consistent; round one is complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001–T004) so CHANGELOG and directories exist; blocks refinement of user story docs
- **User Story 1 (Phase 3)**: Depends on Foundational (T005–T007); defines versioning and traceability
- **User Story 2 (Phase 4)**: Depends on Foundational; can be done in parallel with US1 after Phase 2
- **User Story 3 (Phase 5)**: Depends on Foundational; can be done in parallel with US1/US2 after Phase 2
- **User Story 4 (Phase 6)**: Depends on Foundational; can be done in parallel with US1–US3 after Phase 2
- **Polish (Phase 7)**: Depends on all user story phases (T008–T016) so all docs exist to validate

### User Story Dependencies

- **US1 (P1)**: After Phase 2 — no dependency on US2–US4
- **US2 (P2)**: After Phase 2 — independent; references contracts from Phase 2
- **US3 (P3)**: After Phase 2 — independent; references contracts from Phase 2
- **US4 (P4)**: After Phase 2 — independent; references main spec only

### Parallel Opportunities

- T002, T003, T004 can run in parallel (different directories/files)
- After Phase 2, T008–T016 (US1–US4) can be split across owners and run in parallel
- T017 and T018 can run in parallel
- T019 can run after T017–T018

---

## Parallel Example: User Story 1

```text
# After Phase 2, US1 tasks in sequence (T008 → T009 → T010):
T008: Update CHANGELOG.md versioning subsection
T009: Ensure main spec header has version and date
T010: Add conformance review note to quickstart.md
```

## Parallel Example: Setup

```text
# Phase 1 parallel:
T002: Create diagrams/ and README
T003: Create encoding-tables/ and README
T004: Create or update root README.md
(T001 should complete first so CHANGELOG exists; T002–T004 can then run in parallel)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T004)
2. Complete Phase 2: Foundational (T005–T007)
3. Complete Phase 3: User Story 1 (T008–T010)
4. **STOP and VALIDATE**: Confirm CHANGELOG versioning and interface traceability are in place; spec header has version/date
5. Optional: Run Phase 7 T017 (quickstart validation) early

### Incremental Delivery

1. Setup + Foundational → Spec repo structure and contracts verified
2. Add US1 → Versioning and conformance docs → MVP for "spec as SSOT"
3. Add US2 → Build-and-run workflow docs
4. Add US3 → Conformance verification docs
5. Add US4 → End-user experience docs
6. Polish → Links and cross-checks

### Parallel Team Strategy

- One owner: Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7
- Multiple owners after Phase 2: Owner A (US1), Owner B (US2), Owner C (US3), Owner D (US4); then one person runs Phase 7

---

## Notes

- [P] tasks use different files or sections; no same-file conflicts
- [USn] labels map tasks to user stories for traceability
- This repo contains no application code; every task produces or updates documentation or contracts
- Commit after each task or logical group; validate quickstart links at the end
- Suggested MVP scope: Phases 1–3 (Setup, Foundational, US1) so "Spec as Single Source of Truth" is fully documented and versioned
