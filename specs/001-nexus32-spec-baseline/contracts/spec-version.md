# Contract: Spec and Format Version Reporting

**Source of truth**: NEXUS32_Specification_v1.0.md §8.3, §13.3; Constitution III, VI  
**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

This contract defines how spec version and ROM format version are reported and how compatibility is determined.

## Spec Version

- **Definition**: Version of the NEXUS-32 specification document (e.g. 1.0). Bumped when any observable behavior or interface in the spec changes.
- **Location in spec repo**: Document title/header and CHANGELOG.md.
- **Reporting by emulator**: The emulator MUST report its supported spec version so that games and tools can check compatibility. Per spec §8.3, **SYS_VERSION** (system register at 0x0B006000) is read-only: high 16 bits = major, low 16 bits = minor (e.g. 0x00010000 for spec 1.0).

## ROM Format Version

- **Definition**: 16-bit value in ROM header at offset 4 (`format_version`). Indicates which ROM layout and validation rules the ROM was built for (e.g. 0x0100 = 1.0).
- **Location**: First 128 bytes of ROM file; see [rom-format.md](rom-format.md).

## Compatibility Rules

1. **Forward compatibility (ROM format)**: An emulator that supports spec version N MUST run ROMs whose `format_version` is less than or equal to the maximum format version the emulator supports. Current spec 1.0 supports ROM format 0x0100; future spec versions may support additional format versions while still accepting 0x0100.
2. **Spec version vs. format version**: Spec version describes the full specification; format version describes the ROM file layout. A ROM built for format 0x0100 is valid for spec 1.0; when the spec is amended, new ROM format versions may be introduced (using reserved fields), but 0x0100 ROMs remain valid for emulators that support spec 1.x.
3. **Rejection**: If an emulator or tool does not support a ROM’s format_version, it MUST reject the ROM and report the unsupported format version (e.g. to the user or caller). It MUST NOT silently run with undefined behavior.

## Capability Queries

- **At runtime (game)**: Read SYS_VERSION (0x0B006000) to obtain the emulator’s supported spec version (major/minor). Games may use this to enable or disable features or to show a compatibility message.
- **At load time (emulator/tools)**: Read ROM header `format_version`; if supported, proceed with load/validation; otherwise reject with clear message.

## Changelog and Spec Repo

- **CHANGELOG.md** in the spec repository MUST record spec version changes and, when applicable, ROM format version changes. Round-one deliverables include ensuring CHANGELOG exists and is updated on each spec version bump.
