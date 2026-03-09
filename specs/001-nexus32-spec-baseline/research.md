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

---

# Research: Round Two (Encoding Tables & Diagrams)

**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

## Scope

Round two populates `encoding-tables/` and `diagrams/` with spec-derived content so that emulator and SDK implementers have machine-readable encodings and visual reference without re-parsing the full spec.

## Decisions

### 1. Encoding Table Format (CSV)

**Decision**: Export CPU integer (§2.3) and vector (§2.4) instruction encodings as CSV files. Columns: mnemonic, opcode (hex), func/format type, encoding fields (e.g. rs, rt, rd, immediate), cycles, and spec section reference. One CSV per instruction set (integer, vector).

**Rationale**: CSV is tool-friendly (assemblers, disassemblers, tests), version-controllable, and unambiguous. Matches constitution V (testability).

**Alternatives considered**: JSON (more verbose); inline in spec only (harder for tools to consume).

---

### 2. Diagram Format and Content

**Decision**: Add at least one memory-map diagram and one architecture (system bus) diagram. Format: Markdown with Mermaid and/or ASCII art, or a short placeholder that references spec §1 and §3 figures. Files: `diagrams/memory-map.md`, `diagrams/architecture.md`.

**Rationale**: Spec §13.1 calls for "architecture diagrams, memory maps"; round one created empty directories. Round two adds content so the repo is self-contained for implementers.

**Alternatives considered**: Binary images (harder to diff); external diagram tool only (would not live in repo).

---

### 3. Optional Encoding-Table Contract

**Decision**: Optionally add `contracts/encoding-table-format.md` describing CSV column names, units (hex for opcodes), and that tables are authoritative only insofar as they match the spec. If omitted, README in encoding-tables/ suffices.

**Rationale**: Enables consistent consumption by multiple tools; contract is lightweight (one page).

**Alternatives considered**: No contract (README only); full schema (overkill for round two).

---

### 4. Round Two Scope Boundary

**Decision**: Round two does not add GPU command encodings or mini-shader opcode tables; those can be a later round. Focus is CPU/VU instruction encodings and memory map + architecture diagrams.

**Rationale**: Keeps round two deliverable and aligned with highest implementer need (CPU/decoder first).

**Alternatives considered**: Including GPU/shaders (deferred to avoid scope creep).

---

# Research: Round Three (GPU Commands & Mini-Shader Opcodes)

**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

## Scope

Round three adds machine-readable exports for **GPU command types** (spec §5.2) and **mini-shader opcodes** (spec §5.6) so that emulator command-buffer parsers and SDK shader compilers can consume spec-derived data without re-parsing the full spec. This was listed as “can be added in a later round” in round-one research.

## Decisions

### 1. GPU Command Table Format

**Decision**: Add `encoding-tables/gpu-commands.csv` with columns: `cmd_type_hex`, `name`, `size_bytes` (or “variable” for CMD_SET_UNIFORM), `spec_ref` (e.g. 5.2). One row per command type from spec §5.2 (CMD_CLEAR through CMD_PRESENT).

**Rationale**: Matches existing encoding-table pattern; enables emulator and tools to validate command buffers and build parsers from a single CSV. Spec remains authoritative.

**Alternatives considered**: JSON (more verbose); no table (rejected: round-two pattern already established for CPU/VU).

---

### 2. Mini-Shader Opcode Table Format

**Decision**: Add `encoding-tables/shader-opcodes.csv` with columns: `opcode_hex`, `mnemonic`, `operation` (short description), `spec_ref` (e.g. 5.6). One row per opcode from spec §5.6 (MOV through NOP).

**Rationale**: Supports SDK shaderc and emulator shader→SPIR-V translation; same CSV pattern as other encoding tables. Spec is authoritative.

**Alternatives considered**: Separate “ISA” document (rejected: single CSV keeps encoding-tables consistent).

---

### 3. Contract for GPU/Shader Tables

**Decision**: Do not add a new contract file. Extend `encoding-tables/README.md` (and optionally `contracts/encoding-table-format.md`) to describe the two new tables and state that they follow the same “spec authoritative” rule. Column names documented in README.

**Rationale**: Lightweight; avoids contract proliferation. Encoding-table contract already establishes “spec wins” and column conventions.

**Alternatives considered**: New `contracts/gpu-shader-tables.md` (rejected: README extension suffices for round three).

---

### 4. Round Three Scope Boundary

**Decision**: Round three includes only GPU command types and shader opcodes. It does not include vertex-format bitmasks, texture-format enums, or full command struct layouts—those remain in the main spec. Optional: one-sentence note in quickstart on how to propose spec changes (link to CHANGELOG/versioning).

**Rationale**: Keeps round three deliverable and consistent with round two (encoding tables only). Deeper GPU/shaders can be a later round if needed.

**Alternatives considered**: Full command struct CSV (deferred; spec §5.3+ has the detail).
