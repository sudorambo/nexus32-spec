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

---

# Research: Round Four (I/O and System Register Reference)

**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

## Scope

Round four adds a **consolidated reference** for memory-mapped I/O regions and system/timer/interrupt registers so that emulator and SDK implementers can look up base addresses and register layouts in one place without re-scanning the full spec. Content is derived solely from NEXUS32_Specification_v1.0.md §3 (memory map) and §8 (timer, system, interrupt registers).

## Decisions

### 1. Reference Document Location and Format

**Decision**: Add a `reference/` directory at repository root with a single Markdown file `io-and-system-registers.md`. Content: (1) table of I/O regions (base address, end address or size, name, spec section); (2) tables for system registers (§8.3), timer/frame (§8.2), and interrupt control (§8.4) with offset, name, access, brief description, spec ref. Optional: `reference/README.md` stating that the spec is authoritative and this is a convenience reference.

**Rationale**: One directory keeps “reference” docs separate from encoding-tables (instruction/command/opcode data) and diagrams (architecture/memory map visuals). Markdown tables are diff-friendly and cite spec sections explicitly.

**Alternatives considered**: CSV for registers (rejected: prose descriptions and access R/W/R fit better in Markdown); adding to diagrams/ (rejected: diagrams are visual; this is a register table); putting under encoding-tables/ (rejected: encoding-tables is for instruction/command encodings).

---

### 2. Scope of Register Content

**Decision**: Include (a) all I/O regions from spec §3 (GPU, DMA, DMA table, Audio, Input, Timer, System, Interrupt control) with base, size, name; (b) system registers at 0x0B006000 (§8.3); (c) frame counter at 0x0B005020 (§8.2); (d) interrupt control at 0x0B007000 (§8.4). Do not duplicate full DMA, APU, or Input register layouts in round four—those remain in the main spec; round four focuses on “where things are” and the system/timer/IRQ register set that every implementer needs.

**Rationale**: Keeps round four deliverable small and immediately useful; DMA/APU/Input detail can be added in a later round if needed.

**Alternatives considered**: Full DMA, APU, Input register tables (deferred to avoid scope creep).

---

### 3. Contract or README

**Decision**: No new contract. Add a short `reference/README.md` (or an intro paragraph in `io-and-system-registers.md`) stating that this reference is derived from the spec, the spec is authoritative, and the document is for convenience only.

**Rationale**: Matches round three approach (README extension, no new contract). Reference format is simple (tables with spec refs).

**Alternatives considered**: New contract for “reference document format” (rejected: overkill for one document).

---

### 4. Round Four Scope Boundary

**Decision**: Round four does not add CONTRIBUTING.md, implementation checklists for other repos, or asset-type reference tables. Those can be a later round. Focus is I/O regions + system/timer/interrupt registers only.

**Rationale**: Single clear deliverable; CONTRIBUTING and checklists are orthogonal and can follow in round five if desired.

**Alternatives considered**: Adding CONTRIBUTING (deferred); asset type enum table (deferred).

---

# Research: Round Five (Contributing and Spec-Change Workflow)

**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

## Scope

Round five adds **contributing and spec-change workflow** documentation so that maintainers and contributors know how to propose changes to the specification, when to bump the spec version, and how to keep CHANGELOG in sync. Optionally, an **implementation checklist** for other repos (emulator, SDK, romtools) to self-check conformance against the spec and contracts. This was deferred from round four (“can follow in round five if desired”).

## Decisions

### 1. CONTRIBUTING.md Location and Content

**Decision**: Add CONTRIBUTING.md at repository root. Content: (1) brief purpose of the repo (spec as single source of truth); (2) how to propose spec changes (e.g. issue or PR, with reference to the spec document and contracts); (3) when to bump the spec version (observable behavior or interface changes) and link to CHANGELOG.md and contracts/spec-version.md; (4) CHANGELOG hygiene (entry per release/version); (5) link to .specify/memory/constitution.md for principles. No mandatory PR template or code of conduct unless already present.

**Rationale**: Standard open-source practice; aligns with constitution “Spec-first changes” and “Constitution check.” Gives contributors a single entry point for process.

**Alternatives considered**: Placing workflow only in .specify/ (rejected: CONTRIBUTING.md is discoverable at root); full code of conduct (out of scope for round five).

---

### 2. Implementation Checklist (Optional)

**Decision**: Optionally add a one- or two-page **implementation checklist** (e.g. docs/implementation-checklist.md) that lists conformance items for emulator, SDK, and romtools: e.g. ROM format contract implemented, SYS_VERSION reported, spec version vs. ROM format version handling, links to contracts and reference/encoding-tables. Not a test suite; a human- or tool-checkable list for self-assessment.

**Rationale**: Supports constitution V (testability and spec compliance) by giving other repos a clear checklist; reduces ambiguity on “what must we implement?”

**Alternatives considered**: No checklist (CONTRIBUTING only); full test spec (out of scope—tests live in other repos).

---

### 3. No New Contracts

**Decision**: Round five does not add a new contract. CONTRIBUTING and the optional checklist reference existing contracts (rom-format, spec-version, encoding-table-format) and the constitution.

**Rationale**: Process and checklist are documentation, not new interfaces. Existing contracts remain the normative references.

**Alternatives considered**: “Contributing contract” (rejected: CONTRIBUTING is process, not an API).

---

### 4. Round Five Scope Boundary

**Decision**: Round five does not add asset-type reference tables, DMA/APU/Input register tables, or automation (e.g. CI that enforces CHANGELOG). Focus is CONTRIBUTING.md and optional implementation checklist only.

**Rationale**: Keeps round five deliverable and governance-focused. Asset/register tables can be a later round.

**Alternatives considered**: Asset type table (deferred); CI for CHANGELOG (deferred).
