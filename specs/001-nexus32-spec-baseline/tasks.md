# Tasks: NEXUS-32 Ecosystem Specification Baseline (Round Three)

**Input**: Design documents from `specs/001-nexus32-spec-baseline/`  
**Prerequisites**: plan.md (Round Three), spec.md, research.md, data-model.md, contracts/

**Tests**: Not requested in the feature specification; no test tasks included.

**Organization**: Tasks are grouped by user story. Round Three adds GPU command and mini-shader opcode encoding tables (spec-derived SSOT support and tool/emulator conformance). This repository is documentation-only; all tasks create or update docs and data files.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story (US1, US3) for round-three deliverables
- Include exact file paths in descriptions

## Path Conventions

- **Spec repo root**: `/` = repository root (encoding-tables/, README.md)
- **Feature docs**: `specs/001-nexus32-spec-baseline/` (plan.md, quickstart.md, contracts/)

---

## Phase 1: Setup (Round Three)

**Purpose**: Ensure encoding-tables README and contract describe the two new tables so CSVs can be added consistently.

- [x] T001 Extend encoding-tables/README.md at repository root to describe gpu-commands.csv (cmd_type_hex, name, size_bytes, spec_ref per spec §5.2) and shader-opcodes.csv (opcode_hex, mnemonic, operation, spec_ref per spec §5.6), and link to specs/001-nexus32-spec-baseline/contracts/encoding-table-format.md for column details

**Checkpoint**: encoding-tables/README documents GPU and shader table format; contract already defines columns (Round Three plan)

---

## Phase 2: Foundational (Round Three)

**Purpose**: Confirm encoding-table contract and plan reference the new files and source sections.

- [x] T002 Verify specs/001-nexus32-spec-baseline/contracts/encoding-table-format.md defines gpu-commands.csv and shader-opcodes.csv columns; confirm plan.md references encoding-tables/gpu-commands.csv and encoding-tables/shader-opcodes.csv and NEXUS32_Specification_v1.0.md §5.2 and §5.6

**Checkpoint**: Contract and plan aligned; ready to add GPU and shader CSV files

---

## Phase 3: User Story 1 — Spec as Single Source of Truth (Priority: P1) 🎯 MVP

**Goal**: Machine-readable GPU command and shader opcode tables derived from the spec so that command types and shader opcodes are traceable to NEXUS32_Specification_v1.0.md §5.2 and §5.6; no new encodings; spec remains authoritative.

**Independent Test**: Every row in gpu-commands.csv and shader-opcodes.csv corresponds to a defined command or opcode in spec §5.2 and §5.6; column format matches contracts/encoding-table-format.md.

- [x] T003 [P] [US1] Create encoding-tables/gpu-commands.csv at repository root with header row (cmd_type_hex,name,size_bytes,spec_ref) and one row per GPU command from NEXUS32_Specification_v1.0.md §5.2 (CMD_CLEAR 0x0001 through CMD_PRESENT 0x00FF); use "variable" for CMD_SET_UNIFORM size

- [x] T004 [P] [US1] Create encoding-tables/shader-opcodes.csv at repository root with header row (opcode_hex,mnemonic,operation,spec_ref) and one row per mini-shader opcode from NEXUS32_Specification_v1.0.md §5.6 (MOV 0x00 through NOP 0x15)

**Checkpoint**: Both CSVs exist and are populated from spec §5.2 and §5.6; US1 deliverable complete

---

## Phase 4: User Story 3 — Emulator and Tools Conform to the Spec (Priority: P3)

**Goal**: Document that GPU and shader tables support conformance verification (command-buffer parser, shader compiler); spec remains authoritative.

**Independent Test**: README or quickstart states that gpu-commands and shader-opcodes tables enable tests against spec §5 and that the spec wins on any conflict; tables validate against spec.

- [x] T005 Validate encoding-tables/gpu-commands.csv and encoding-tables/shader-opcodes.csv against NEXUS32_Specification_v1.0.md §5.2 and §5.6; fix any missing or incorrect rows (spec is authoritative)

- [x] T006 [US3] Add to encoding-tables/README.md at repository root: GPU and shader encoding tables enable command-buffer and shader-compiler conformance tests per NEXUS32_Specification_v1.0.md §5; the spec is authoritative and wins over the tables on any discrepancy

**Checkpoint**: Conformance use of GPU/shader tables is documented; US3 deliverable complete

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Link new artifacts from root README and quickstart; finalize Round Three deliverables.

- [x] T007 [P] Update README.md at repository root to add one-line descriptions and links for encoding-tables/gpu-commands.csv (GPU command types from spec §5.2) and encoding-tables/shader-opcodes.csv (mini-shader opcodes from spec §5.6)

- [x] T008 Update specs/001-nexus32-spec-baseline/quickstart.md Round-Three Deliverables section to state that encoding tables (gpu-commands.csv, shader-opcodes.csv) are in place with paths and spec references (§5.2, §5.6); mark Round-Three Deliverables as Complete

**Checkpoint**: README and quickstart updated; Round Three complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001); quick check of contract and plan
- **US1 (Phase 3)**: Depends on Foundational; T003 and T004 can run in parallel
- **US3 (Phase 4)**: Depends on US1 (CSVs must exist to validate and document); T005 then T006
- **Polish (Phase 5)**: Depends on Phase 4; T007 and T008 can run in parallel

### User Story Dependencies

- **User Story 1 (P1)**: GPU and shader tables as spec-derived data — can start after Foundational
- **User Story 3 (P3)**: Conformance documentation and validation — depends on US1 (tables created)

### Parallel Opportunities

- T003 and T004 (create gpu-commands.csv and shader-opcodes.csv) can run in parallel
- T007 and T008 (README and quickstart updates) can run in parallel after T006

---

## Parallel Example: User Story 1

```text
# Create both Round Three encoding tables in parallel:
Task T003: Create encoding-tables/gpu-commands.csv from spec §5.2
Task T004: Create encoding-tables/shader-opcodes.csv from spec §5.6
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001)
2. Complete Phase 2: Foundational (T002)
3. Complete Phase 3: User Story 1 (T003, T004)
4. **STOP and VALIDATE**: Confirm both CSVs match spec §5.2 and §5.6

### Full Round Three

1. Complete Setup + Foundational → ready to add CSVs
2. Add US1 tables (T003, T004) → validate (T005)
3. Add US3 conformance note (T006) → Polish (T007, T008)

---

## Notes

- [P] tasks = different files, no dependencies
- [US1] / [US3] map tasks to user stories for traceability
- Round Three does not add new diagrams (US2); Round Two already delivered memory-map and architecture
- Spec is authoritative: on any discrepancy between a table and NEXUS32_Specification_v1.0.md, the spec wins
- Commit after each task or logical group
