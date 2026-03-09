# Research: NEXUS-32 Ecosystem Specification Baseline (Round One)

**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

## Scope

Round one focuses on the **spec repository** (this repo) as the single source of truth. No implementation of emulator, SDK, or romtools in this repo; only specification content, versioning, and machine-readable contracts so that other repositories can implement against a stable, testable definition.

## Decisions

### 1. Specification as Single Source of Truth

**Decision**: The document `NEXUS32_Specification_v1.0.md` (and its versioned successors) is the sole authority for observable behavior (CPU/VU ISA, memory map, DMA, GPU, APU, input, timers, ROM format, SDK surface). All other repos (emulator, SDK, romtools, examples) MUST conform and MUST NOT introduce behavior outside the spec.

**Rationale**: Prevents implementation drift, ensures ROMs run identically across compliant implementations, and gives a single place to resolve ambiguities.

**Alternatives considered**: Multiple “spec” docs per component (rejected: would fragment authority and complicate versioning).

---

### 2. Versioning Strategy (Spec vs. ROM Format)

**Decision**: Two version concepts: (1) **Spec version** (e.g., 1.0) — bumped when any observable behavior or interface in the spec changes; (2) **ROM format version** (e.g., 0x0100) — in ROM header, forward-compatible so older-format ROMs run on newer emulators. New features use reserved bits and additive definitions.

**Rationale**: Aligns with NEXUS32_Specification_v1.0.md §13.3 and constitution III/VI. Enables safe evolution without breaking existing ROMs.

**Alternatives considered**: Single version for both (rejected: spec and format can evolve at different rates; format is a subset of spec).

---

### 3. Multi-Repository Layout

**Decision**: Five repos — nexus32-spec (this repo), nexus32-emulator, nexus32-sdk, nexus32-romtools, nexus32-examples — with dependencies as in spec §13.2. This repo contains only the spec, diagrams, encoding-tables, CHANGELOG, and contracts.

**Rationale**: Keeps spec and tooling separate; implementers clone only what they need; spec repo stays minimal and reviewable.

**Alternatives considered**: Monorepo (rejected: spec would be tied to one codebase’s release cycle and tooling).

---

### 4. Contracts for Round One

**Decision**: Publish two contract artifacts in this repo: (1) **ROM format contract** — header layout, magic, format_version, validation rules (accept/reject) from spec §9; (2) **Spec/version contract** — how spec version and ROM format version are reported and compared (e.g., SYS_VERSION, header format_version), and forward-compatibility rules.

**Rationale**: Gives romtools and emulator a concise, testable definition for validation and compatibility checks without re-parsing the full spec.

**Alternatives considered**: No formal contracts, only prose (rejected: would slow down romtools/emulator implementation and increase inconsistency risk).

---

### 5. Spec Repo Structure (Round One)

**Decision**: Ensure repo root has: main spec document, CHANGELOG.md for spec version history, `diagrams/` for architecture and memory maps, `encoding-tables/` for instruction encoding CSV exports. Add README.md pointing to the spec and ecosystem repos if missing.

**Rationale**: Matches spec §13.1 and supports implementers with diagrams and machine-readable encoding data.

**Alternatives considered**: Keeping only the single .md file (rejected: diagrams and encoding tables reduce errors when implementing CPU/decoders).

## Unresolved / Out of Scope for Round One

- Implementation of emulator, SDK, or romtools (other repos).
- Concrete test suites (lived in emulator/romtools repos; contracts in this repo enable them).
- Mini-shader or GPU command binary format exports (can be added in a later round).
