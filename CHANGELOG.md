# Changelog — NEXUS-32 Specification

All notable changes to the NEXUS-32 specification document are recorded here. The specification itself is **NEXUS32_Specification_v1.0.md** (and its versioned successors) in this repository.

## Versioning

- **When to bump spec version**: Bump the specification version when any *observable behavior* or *defined interface* changes (e.g. new or modified instructions, memory map, register layout, GPU commands, ROM format, or public APIs). Clarifications and typo fixes that do not change behavior do not require a version bump.
- **How to add entries**: For each release, add a dated section below (e.g. `## [1.1] — YYYY-MM-DD`) with a short summary of changes. Keep the format: version in brackets, date, then bullet list of changes. Update the main spec document header (version and date) to match.

---

## [1.0] — 2026-03-08

- Initial specification release: NEXUS-32 Fantasy Game Console — Hardware & Ecosystem Specification.
- Defines CPU/VU ISA, memory map, DMA, Virtual GPU, APU, input, timers, ROM format (§9), SDK, toolchain, emulator architecture, and multi-repository structure (§13).
- Single source of truth for the nexus32-spec, nexus32-emulator, nexus32-sdk, nexus32-romtools, and nexus32-examples ecosystem.
