# Tasks: NEXUS-32 Ecosystem Specification Baseline (Round Two)

**Input**: Design documents from `specs/001-nexus32-spec-baseline/`  
**Prerequisites**: plan.md (Round Two), spec.md, research.md, data-model.md, contracts/

**Tests**: Not requested in the feature specification; no test tasks included.

**Organization**: Tasks are grouped by user story. Round Two adds encoding tables (spec-derived SSOT support) and diagrams (implementer reference). This repository is documentation-only; all tasks create or update docs and data files.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story (US1, US2, US3) for round-two deliverables
- Include exact file paths in descriptions

## Path Conventions

- **Spec repo root**: `/` = repository root (encoding-tables/, diagrams/, README.md)
- **Feature docs**: `specs/001-nexus32-spec-baseline/` (plan.md, quickstart.md, contracts/)

---

## Phase 1: Setup (Round Two)

**Purpose**: Prepare encoding-tables and contract reference so CSV files can be added consistently.

- [x] T001 Update encoding-tables/README.md at repository root to describe CSV column format (mnemonic, format, opcode_hex, func_hex, cycles, spec_ref for integer; mnemonic, vfunc_hex, cycles, spec_ref for vector) and add link to specs/001-nexus32-spec-baseline/contracts/encoding-table-format.md

**Checkpoint**: encoding-tables/README.md documents format and contract link

---

## Phase 2: Foundational (Round Two)

**Purpose**: Ensure encoding-table contract is in place and plan references are correct.

- [x] T002 Verify specs/001-nexus32-spec-baseline/contracts/encoding-table-format.md exists and specifies integer-instructions.csv and vector-instructions.csv columns; confirm plan.md references encoding-tables/ and diagrams/ paths

**Checkpoint**: Contract and plan aligned; ready to add CSV and diagram files

---

## Phase 3: User Story 1 — Spec as Single Source of Truth (Priority: P1) 🎯 MVP

**Goal**: Machine-readable encoding tables derived from the spec so that instruction encodings are traceable to NEXUS32_Specification_v1.0.md §2; no new encodings; spec remains authoritative.

**Independent Test**: Every row in integer-instructions.csv and vector-instructions.csv corresponds to a defined instruction in spec §2.3 and §2.4; column format matches contracts/encoding-table-format.md.

- [x] T003 [P] [US1] Create encoding-tables/integer-instructions.csv at repository root with header row and one row per integer instruction from NEXUS32_Specification_v1.0.md §2.3 (mnemonic, format, opcode_hex, func_hex, cycles, spec_ref; add encoding-field columns as needed)
- [x] T004 [P] [US1] Create encoding-tables/vector-instructions.csv at repository root with header row and one row per vector instruction from NEXUS32_Specification_v1.0.md §2.4 (mnemonic, vfunc_hex, cycles, spec_ref; add vs/vt/vd columns as needed per contract)

**Checkpoint**: Both CSVs exist and are populated from spec §2; US1 deliverable complete

---

## Phase 4: User Story 2 — Game Developers Can Build and Run ROMs (Priority: P2)

**Goal**: Diagrams give implementers a visual reference for memory map and architecture so that SDK/emulator work is easier; supports the build-and-run workflow defined in quickstart.

**Independent Test**: diagrams/memory-map.md and diagrams/architecture.md exist and reference spec §3 and §1; content reflects the spec (address ranges, component names).

- [x] T005 [P] [US2] Create diagrams/memory-map.md at repository root with memory map (address ranges, region names) from NEXUS32_Specification_v1.0.md §3; use Mermaid or ASCII; cite spec §3
- [x] T006 [P] [US2] Create diagrams/architecture.md at repository root with system bus and component layout from NEXUS32_Specification_v1.0.md §1 (CPU, VU, Memory, DMA, VGP, APU, Input, Timer); use Mermaid or ASCII; cite spec §1

**Checkpoint**: Memory map and architecture diagrams exist; US2 deliverable complete

---

## Phase 5: User Story 3 — Emulator and Tools Conform to the Spec (Priority: P3)

**Goal**: Document that encoding tables support conformance verification (decoder/assembler tests); spec remains authoritative.

**Independent Test**: README or quickstart states that tables enable tests against spec §2 and that the spec wins on any conflict.

- [x] T007 [US3] Add to encoding-tables/README.md at repository root: encoding tables enable decoder and assembler conformance tests per NEXUS32_Specification_v1.0.md §2; the spec is authoritative and wins over the tables on any discrepancy

**Checkpoint**: Conformance use of tables is documented; US3 deliverable complete

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Link new artifacts from README and quickstart; validate tables and diagrams against the spec.

- [x] T008 [P] Update README.md at repository root to add one-line descriptions and links for encoding-tables/ (CSV instruction encodings from spec §2) and diagrams/ (memory map §3, architecture §1)
- [x] T009 Validate encoding-tables/integer-instructions.csv and encoding-tables/vector-instructions.csv against NEXUS32_Specification_v1.0.md §2.3 and §2.4; fix any missing or incorrect rows (spec is authoritative)
- [x] T010 Update specs/001-nexus32-spec-baseline/quickstart.md Round-Two Deliverables section to state that encoding tables (integer-instructions.csv, vector-instructions.csv) and diagrams (memory-map.md, architecture.md) are in place with paths and spec references

**Checkpoint**: README and quickstart updated; tables validated; Round Two complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001); quick check of contract and plan
- **US1 (Phase 3)**: Depends on Foundational; T003 and T004 can run in parallel
- **US2 (Phase 4)**: Depends on Foundational; T005 and T006 can run in parallel; can run in parallel with Phase 3
- **US3 (Phase 5)**: Depends on Phase 3 (encoding tables exist); T007 updates README
- **Polish (Phase 6)**: Depends on Phases 3, 4, 5 so that files exist to link and validate

### User Story Dependencies

- **US1 (encoding tables)**: After Phase 2; no dependency on US2
- **US2 (diagrams)**: After Phase 2; no dependency on US1
- **US3 (conformance note)**: After US1 (tables exist)

### Parallel Opportunities

- T003 and T004 can run in parallel (different CSV files)
- T005 and T006 can run in parallel (different diagram files)
- After Phase 2, Phase 3 (T003, T004) and Phase 4 (T005, T006) can run in parallel
- T008 can run in parallel with T009 (different files)

---

## Parallel Example: Phase 3 and Phase 4

```text
# After Phase 2:
T003: Create integer-instructions.csv
T004: Create vector-instructions.csv
T005: Create diagrams/memory-map.md
T006: Create diagrams/architecture.md
(T003–T006 can be done in parallel by different owners)
```

---

## Implementation Strategy

### MVP First (User Story 1 — Encoding Tables)

1. Complete Phase 1: Setup (T001)
2. Complete Phase 2: Foundational (T002)
3. Complete Phase 3: US1 (T003, T004) — encoding tables
4. **STOP and VALIDATE**: Confirm CSVs match spec §2 and contract format
5. Optional: Run T007 (conformance note) and T009 (validation)

### Incremental Delivery

1. Setup + Foundational → Ready for data files
2. Add US1 (encoding tables) → Spec-derived CSVs for tools
3. Add US2 (diagrams) → Memory map and architecture reference
4. Add US3 (conformance note) → Document testability
5. Polish → README, quickstart, validation

### Parallel Team Strategy

- After Phase 2: Owner A — integer-instructions.csv (T003); Owner B — vector-instructions.csv (T004); Owner C — memory-map.md (T005); Owner D — architecture.md (T006). Then one person does T007, T008–T010.

---

## Notes

- [P] tasks use different files; no same-file conflicts
- [USn] labels map Round Two deliverables to spec user stories for traceability
- This repo contains no application code; every task produces or updates documentation or CSV/diagram content
- Suggested MVP scope: Phases 1–3 (Setup, Foundational, US1) so encoding tables are complete and spec-derived
