# Tasks: NEXUS-32 Ecosystem Specification Baseline (Round Four)

**Input**: Design documents from `specs/001-nexus32-spec-baseline/`  
**Prerequisites**: plan.md (Round Four), spec.md, research.md, data-model.md, contracts/

**Tests**: Not requested in the feature specification; no test tasks included.

**Organization**: Tasks are grouped by user story. Round Four adds the I/O and system register reference (spec-derived SSOT support and implementer conformance). This repository is documentation-only; all tasks create or update docs.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story (US1, US3) for round-four deliverables
- Include exact file paths in descriptions

## Path Conventions

- **Spec repo root**: `/` = repository root (reference/, README.md)
- **Feature docs**: `specs/001-nexus32-spec-baseline/` (plan.md, quickstart.md, contracts/)

---

## Phase 1: Setup (Round Four)

**Purpose**: Create reference directory and README so the I/O/system register document can be added consistently.

- [x] T001 Create reference/ directory at repository root and add reference/README.md stating that reference documents are derived from NEXUS32_Specification_v1.0.md, the spec is authoritative, and this reference is for implementer convenience only

**Checkpoint**: reference/ exists with README; ready to add io-and-system-registers.md

---

## Phase 2: Foundational (Round Four)

**Purpose**: Confirm plan and research reference spec §3 and §8 and the path reference/io-and-system-registers.md.

- [x] T002 Verify specs/001-nexus32-spec-baseline/plan.md and research.md reference NEXUS32_Specification_v1.0.md §3 (memory map) and §8 (timer, system, interrupt); confirm plan references reference/io-and-system-registers.md

**Checkpoint**: Plan and research aligned; ready to create reference content

---

## Phase 3: User Story 1 — Spec as Single Source of Truth (Priority: P1) 🎯 MVP

**Goal**: Consolidated I/O and system register reference derived from the spec so that base addresses and register layouts are traceable to NEXUS32_Specification_v1.0.md §3 and §8; no new addresses or behavior; spec remains authoritative.

**Independent Test**: Every table in reference/io-and-system-registers.md corresponds to spec §3 (I/O regions) and §8 (system, timer, interrupt registers); each section cites the spec.

- [x] T003 [P] [US1] Create reference/io-and-system-registers.md at repository root with (1) I/O regions table: base address, end address or size, region name, spec section (§3) for GPU, DMA, DMA table, Audio, Input, Timer, System, Interrupt control from NEXUS32_Specification_v1.0.md §3; (2) System registers table (0x0B006000, §8.3): offset, name, access, brief description, spec ref; (3) Frame counter (0x0B005020, §8.2); (4) Interrupt control (0x0B007000, §8.4). Cite NEXUS32_Specification_v1.0.md in each section.

**Checkpoint**: reference/io-and-system-registers.md exists and is populated from spec §3 and §8; US1 deliverable complete

---

## Phase 4: User Story 3 — Emulator and Tools Conform to the Spec (Priority: P3)

**Goal**: Document that the reference supports conformance verification (register layout, addresses); spec remains authoritative.

**Independent Test**: Reference README or doc intro states that the reference enables implementers to verify I/O and register layout against the spec and that the spec wins on any discrepancy; content validated against spec.

- [x] T004 Validate reference/io-and-system-registers.md against NEXUS32_Specification_v1.0.md §3 and §8; fix any missing or incorrect entries (spec is authoritative)

- [x] T005 [US3] Add to reference/README.md at repository root: this reference enables implementers to verify memory-mapped I/O and system/register layout against NEXUS32_Specification_v1.0.md §3 and §8; the spec is authoritative and wins on any discrepancy

**Checkpoint**: Conformance use of reference is documented; US3 deliverable complete

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Link reference from root README and quickstart; finalize Round Four deliverables.

- [x] T006 [P] Update README.md at repository root to add one-line description and link for reference/ (I/O and system register reference from spec §3, §8)

- [x] T007 Update specs/001-nexus32-spec-baseline/quickstart.md Round-Four Deliverables section to state that reference/io-and-system-registers.md is in place with path and spec references (§3, §8); mark Round-Four Deliverables as Complete

**Checkpoint**: README and quickstart updated; Round Four complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001); quick check of plan and research
- **US1 (Phase 3)**: Depends on Foundational; T003 creates the reference document
- **US3 (Phase 4)**: Depends on US1 (reference doc must exist to validate and document); T004 then T005
- **Polish (Phase 5)**: Depends on Phase 4; T006 and T007 can run in parallel

### User Story Dependencies

- **User Story 1 (P1)**: I/O and system register reference as spec-derived data — can start after Foundational
- **User Story 3 (P3)**: Conformance documentation and validation — depends on US1 (reference created)

### Parallel Opportunities

- T006 and T007 (README and quickstart updates) can run in parallel after T005

---

## Parallel Example: Polish

```text
# After Phase 4, update docs in parallel:
Task T006: Update README.md with reference/ link
Task T007: Update quickstart.md Round-Four section to Complete
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001)
2. Complete Phase 2: Foundational (T002)
3. Complete Phase 3: User Story 1 (T003)
4. **STOP and VALIDATE**: Confirm reference content matches spec §3 and §8

### Full Round Four

1. Complete Setup + Foundational → ready to add reference doc
2. Add US1 reference (T003) → validate (T004)
3. Add US3 conformance note (T005) → Polish (T006, T007)

---

## Notes

- [P] tasks = different files, no dependencies
- [US1] / [US3] map tasks to user stories for traceability
- Round Four does not add encoding tables or diagrams; it adds a single reference document under reference/
- Spec is authoritative: on any discrepancy between the reference and NEXUS32_Specification_v1.0.md, the spec wins
- Commit after each task or logical group
