# Tasks: NEXUS-32 Ecosystem Specification Baseline (Round Five)

**Input**: Design documents from `specs/001-nexus32-spec-baseline/`  
**Prerequisites**: plan.md (Round Five), spec.md, research.md, data-model.md, contracts/

**Tests**: Not requested in the feature specification; no test tasks included.

**Organization**: Tasks are grouped by user story. Round Five adds contributing and spec-change workflow (CONTRIBUTING.md) and optional implementation checklist for other repos. This repository is documentation-only; all tasks create or update docs.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story (US1, US3) for round-five deliverables
- Include exact file paths in descriptions

## Path Conventions

- **Spec repo root**: `/` = repository root (CONTRIBUTING.md, README.md, docs/)
- **Feature docs**: `specs/001-nexus32-spec-baseline/` (plan.md, quickstart.md, contracts/)

---

## Phase 1: Setup (Round Five)

**Purpose**: Ensure docs/ directory exists for optional implementation checklist.

- [x] T001 Create docs/ directory at repository root (for implementation checklist in T004; may be skipped if checklist is omitted)

**Checkpoint**: docs/ exists; ready to add CONTRIBUTING and optional checklist

---

## Phase 2: Foundational (Round Five)

**Purpose**: Confirm plan and research reference CONTRIBUTING.md and optional implementation checklist.

- [x] T002 Verify specs/001-nexus32-spec-baseline/plan.md and research.md reference CONTRIBUTING.md at repository root and optional docs/implementation-checklist.md; confirm alignment with constitution and contracts

**Checkpoint**: Plan and research aligned; ready to create CONTRIBUTING and checklist

---

## Phase 3: User Story 1 — Spec as Single Source of Truth (Priority: P1) 🎯 MVP

**Goal**: CONTRIBUTING.md that reinforces spec-first changes, when to bump spec version, and CHANGELOG hygiene; links to constitution and contracts.

**Independent Test**: CONTRIBUTING.md exists at repository root, describes how to propose spec changes and when to bump version, and links to .specify/memory/constitution.md and specs/001-nexus32-spec-baseline/contracts/.

- [x] T003 [US1] Create CONTRIBUTING.md at repository root with: (1) brief purpose of this repo (spec as single source of truth for NEXUS-32); (2) how to propose spec changes (e.g. issue or PR referencing NEXUS32_Specification_v1.0.md and contracts); (3) when to bump the spec version (observable behavior or interface changes) and reference to CHANGELOG.md and specs/001-nexus32-spec-baseline/contracts/spec-version.md; (4) CHANGELOG hygiene (entry per version); (5) link to .specify/memory/constitution.md for principles

**Checkpoint**: CONTRIBUTING.md in place; US1 deliverable complete

---

## Phase 4: User Story 3 — Emulator and Tools Conform to the Spec (Priority: P3)

**Goal**: Optional implementation checklist so emulator, SDK, and romtools can self-check conformance; spec and contracts remain authoritative.

**Independent Test**: docs/implementation-checklist.md exists (if implemented) with conformance items and links to contracts, reference/, encoding-tables; CONTRIBUTING or quickstart mentions the checklist.

- [x] T004 [US3] (Optional) Create docs/implementation-checklist.md at repository root with conformance self-check items for emulator, SDK, and romtools (e.g. ROM format contract, SYS_VERSION reporting, spec vs. ROM format version handling), with links to specs/001-nexus32-spec-baseline/contracts/, reference/, and encoding-tables/

**Checkpoint**: Optional checklist in place; US3 deliverable complete

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Link CONTRIBUTING and optional checklist from README and quickstart; finalize Round Five deliverables.

- [x] T005 [P] Update README.md at repository root to add one-line description and link for CONTRIBUTING.md (how to propose spec changes, versioning, CHANGELOG); if docs/implementation-checklist.md exists, add link to it

- [x] T006 Update specs/001-nexus32-spec-baseline/quickstart.md Round-Five Deliverables section to state that CONTRIBUTING.md is in place with path; if implementation checklist was added, state docs/implementation-checklist.md with path; mark Round-Five Deliverables as Complete

**Checkpoint**: README and quickstart updated; Round Five complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001); quick check of plan and research
- **US1 (Phase 3)**: Depends on Foundational; T003 creates CONTRIBUTING.md
- **US3 (Phase 4)**: Depends on US1 (CONTRIBUTING can reference checklist); T004 is optional
- **Polish (Phase 5)**: Depends on Phase 3 (and Phase 4 if checklist added); T005 and T006 can run in parallel

### User Story Dependencies

- **User Story 1 (P1)**: CONTRIBUTING.md — can start after Foundational
- **User Story 3 (P3)**: Optional implementation checklist — can follow US1; may be skipped

### Parallel Opportunities

- T005 and T006 (README and quickstart updates) can run in parallel after T003 (and T004 if done)

---

## Parallel Example: Polish

```text
# After Phase 3 (and optional Phase 4), update docs in parallel:
Task T005: Update README.md with CONTRIBUTING and optional checklist link
Task T006: Update quickstart.md Round-Five section to Complete
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001)
2. Complete Phase 2: Foundational (T002)
3. Complete Phase 3: User Story 1 (T003)
4. **STOP and VALIDATE**: CONTRIBUTING.md reflects constitution and contracts
5. Optional: Add Polish (T005, T006) to link CONTRIBUTING from README and quickstart

### Full Round Five

1. Complete Setup + Foundational → ready for CONTRIBUTING
2. Add US1 CONTRIBUTING (T003) → optionally add US3 checklist (T004)
3. Polish (T005, T006) → link CONTRIBUTING and optional checklist; mark Round Five Complete

---

## Notes

- [P] tasks = different files, no dependencies
- [US1] / [US3] map tasks to user stories for traceability
- T004 is optional: round five is complete with CONTRIBUTING.md only; checklist is additive
- Content in CONTRIBUTING and checklist MUST align with constitution and existing contracts; no new behavioral requirements
- Commit after each task or logical group
