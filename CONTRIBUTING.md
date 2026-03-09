# Contributing to NEXUS-32 Specification

This repository (**nexus32-spec**) is the **single source of truth** for the NEXUS-32 fantasy game console: hardware, ROM format, SDK surface, and ecosystem layout. All implementations (emulator, SDK, romtools, examples) must conform to the specification.

## Purpose of This Repo

- **Specification document**: [NEXUS32_Specification_v1.0.md](NEXUS32_Specification_v1.0.md) (and versioned successors) defines all observable behavior and interfaces.
- **Contracts**: [specs/001-nexus32-spec-baseline/contracts/](specs/001-nexus32-spec-baseline/contracts/) formalize ROM format, spec/version reporting, and encoding-table format for implementers.
- **Principles**: [.specify/memory/constitution.md](.specify/memory/constitution.md) describes governance (spec as SSOT, determinism, forward compatibility, component boundaries, testability, version reporting).

## How to Propose Spec Changes

1. **Open an issue or pull request** that clearly references:
   - The relevant section(s) of [NEXUS32_Specification_v1.0.md](NEXUS32_Specification_v1.0.md)
   - Any affected [contracts](specs/001-nexus32-spec-baseline/contracts/) (e.g. rom-format, spec-version)
2. **Describe the change** in terms of observable behavior or interfaces (memory map, instructions, GPU commands, ROM layout, etc.).
3. **Ensure spec-first**: Changes to observable behavior or interfaces must be specified first and reflected in the spec (and CHANGELOG) before implementation in other repos.

## When to Bump the Spec Version

- **Bump the spec version** when any **observable behavior** or **interface** in the specification changes (e.g. new instructions, memory map change, ROM header layout, GPU command semantics).
- **Version and changelog**: Update [CHANGELOG.md](CHANGELOG.md) with an entry for the new version. See [specs/001-nexus32-spec-baseline/contracts/spec-version.md](specs/001-nexus32-spec-baseline/contracts/spec-version.md) for how spec version and ROM format version relate and for compatibility rules.
- **Forward compatibility**: New features must use reserved or additive definitions; older ROMs must remain runnable on newer emulators per the constitution.

## CHANGELOG Hygiene

- **One entry per version**: Each spec version bump must have a corresponding entry in [CHANGELOG.md](CHANGELOG.md).
- **Describe what changed**: Briefly list changes to behavior or interfaces so implementers and tool authors can assess impact.

## Governance and Principles

All changes must align with the [NEXUS-32 Constitution](.specify/memory/constitution.md): specification as single source of truth, determinism and VM contract, forward compatibility and additive changes, component boundaries, testability and spec compliance, and version/capability reporting.

## Implementation Checklist for Other Repos

If you implement the emulator, SDK, or romtools, use [docs/implementation-checklist.md](docs/implementation-checklist.md) to self-check conformance against the spec and contracts.
