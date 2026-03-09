<!--
Sync Impact Report
==================
Version change: (template/placeholder) → 1.0.0
Modified principles: N/A (initial population from NEXUS32_Specification_v1.0.md)
Added sections: Core Principles (6), Technology & Compatibility, Development Workflow, Governance
Removed sections: None (replaced all placeholders)
Templates requiring updates:
  - .specify/templates/plan-template.md ✅ (Constitution Check gate aligns with principles)
  - .specify/templates/spec-template.md ✅ (scope/requirements compatible)
  - .specify/templates/tasks-template.md ✅ (task categorization compatible)
  - .specify/templates/checklist-template.md ✅ (no conflict)
Follow-up TODOs: None
-->

# NEXUS-32 Constitution

## Core Principles

### I. Specification as Single Source of Truth

All implementations in the NEXUS-32 ecosystem (emulator, SDK, romtools, examples) MUST conform to the official NEXUS32 specification. No behavior may be introduced that is not defined or permitted by the spec. The specification document is the authoritative reference for instruction encodings, memory map, register semantics, GPU command format, ROM layout, and API contracts. Rationale: a single source of truth prevents implementation drift and ensures that ROMs run identically across compliant emulators and toolchains.

### II. Determinism and Virtual Machine Contract

The NEXUS-32 virtual machine is defined to be deterministic: fixed memory map, fixed cycle budget per frame, predictable DMA transfer times, and a GPU that executes every command in the buffer in order. Implementations MUST preserve the observable semantics defined in the spec. Host-side optimizations (e.g., resolution scaling, texture filtering, multithreading) are allowed only when the game observes correct behavior. Rationale: games must be able to rely on deterministic timing and behavior for gameplay, replay, and debugging.

### III. Forward Compatibility and Additive Changes

ROM format versions MUST remain forward-compatible: an emulator supporting a newer spec version MUST run ROMs built for older spec versions. New features (GPU commands, instruction encodings, I/O registers) MUST use reserved or previously unused bits and address ranges. The spec version MUST be bumped when any observable behavior changes. Rationale: protects existing ROMs and tooling when the platform evolves.

### IV. Component Boundaries and Interfaces

The system is defined by discrete components (NX-RISC CPU, Vector Unit, Memory Controller, DMA, Virtual GPU, APU, Input, Timer, System Registers). Implementations MUST respect these boundaries and the specified interfaces: memory map, register layout, command buffer format, and interrupt model. Cross-component behavior (e.g., DMA vs. CPU bus arbitration) MUST follow the spec. Rationale: clear boundaries enable independent development and testing of each subsystem and keep the contract understandable.

### V. Testability and Spec Compliance

Emulator, SDK, and tooling MUST be testable against the specification. Critical areas include: ROM validation (format, headers, segments), instruction encoding and execution semantics, memory map and register behavior, DMA transfer correctness, GPU command interpretation, and APU mixing. Tests MUST be able to assert spec-defined outcomes (e.g., cycle counts, register values, memory state). Rationale: automated compliance checks prevent regressions and document intended behavior.

### VI. Version and Capability Reporting

The emulator MUST report its supported spec version (e.g., via SYS_VERSION register at 0x0B006000) so that games can query capabilities at runtime. ROMs and tools MUST declare target format version and/or spec version for compatibility checks. Rationale: enables version-aware behavior and clear compatibility guarantees across the ecosystem.

## Technology & Compatibility

- **Specification**: NEXUS32_Specification_v1.0.md (and successors) in this repository; version and changelog in CHANGELOG.md.
- **Emulator**: C17, Vulkan 1.3+, SDL2 2.28+; target platforms as defined in the spec.
- **SDK**: C17 compiler (nxcc, nxasm, nxld); host tools with no external libs for core compiler/assembler/linker.
- **ROM tools**: C17, LZ4 (bundled); ROM format version in header; validator must reject invalid or unsupported versions per spec.
- **Multi-repo layout**: nexus32-spec (this repo), nexus32-emulator, nexus32-sdk, nexus32-romtools, nexus32-examples; dependencies between repos follow the spec’s build dependency table.

## Development Workflow

- **Spec-first changes**: Any change to observable behavior (instructions, memory map, GPU commands, ROM format) MUST be specified first in the specification and reflected in CHANGELOG.md with a version bump.
- **Constitution check**: Plans and feature work MUST verify alignment with these principles (spec compliance, determinism, forward compatibility, component boundaries, testability, version reporting).
- **Reviews**: PRs that affect emulator, SDK, or romtools MUST confirm that behavior matches the spec and that new features are additive and use reserved ranges where required.

## Governance

This constitution supersedes ad-hoc practices for the NEXUS-32 project. All PRs and reviews MUST verify compliance with the core principles. Amendments to this document require: (1) documentation of the change and rationale, (2) update to the Sync Impact Report at the top of this file, (3) semantic version bump (MAJOR for backward-incompatible principle removals or redefinitions, MINOR for new principles or materially expanded guidance, PATCH for clarifications and non-semantic refinements). Complexity or exceptions to principles MUST be justified and recorded. For implementation guidance, use the NEXUS32 specification and repository-specific READMEs.

**Version**: 1.0.0 | **Ratified**: 2026-03-08 | **Last Amended**: 2026-03-08
