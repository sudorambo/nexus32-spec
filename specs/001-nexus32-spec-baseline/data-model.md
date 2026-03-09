# Data Model: NEXUS-32 Ecosystem Specification Baseline

**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08  
**Source**: NEXUS32_Specification_v1.0.md (repository root)

This document describes the key entities and their relationships as defined by the specification. It is authoritative only insofar as it reflects the main spec; the spec remains the single source of truth.

## Entities

### Specification

- **Description**: The official NEXUS-32 hardware and ecosystem document (e.g. `NEXUS32_Specification_v1.0.md`). Defines CPU/VU ISA, memory map, DMA, Virtual GPU, APU, input, timers, ROM format, SDK surface, and toolchain.
- **Attributes**: Version (e.g. 1.0), ratification/amendment dates, changelog (CHANGELOG.md).
- **Relationships**: Referenced by all other entities; changes to observable behavior require a spec version bump.
- **Validation**: Version and changelog must be kept in sync with content changes.

### Spec Version / ROM Format Version

- **Description**: Two related version concepts. (1) **Spec version**: semantic version of the specification document (e.g. 1.0). (2) **ROM format version**: 16-bit value in ROM header (e.g. 0x0100 = 1.0); indicates which ROM layout and rules the ROM was built for.
- **Attributes**: Spec version (major.minor); ROM format_version (uint16 in header). Forward-compatibility rule: emulator supporting spec N must run ROMs with format_version ≤ supported format.
- **Relationships**: Emulator reports supported spec version (e.g. SYS_VERSION); ROM declares format_version in header; tools use both for compatibility checks.
- **Validation**: New features must be additive; reserved bits and ranges used for extensions.

### ROM

- **Description**: A packaged game image (`.nxrom`). Comprises header, code segment, data segment, asset directory, and asset segments.
- **Attributes**: File extension `.nxrom`; magic `NX32`; format_version; entry_point; code/data/asset layout; checksums; title; author; cycle_budget; screen dimensions.
- **Relationships**: Built by SDK + romtools; loaded and executed by emulator; validated by romtools against ROM format contract.
- **Validation**: See [contracts/rom-format.md](contracts/rom-format.md); magic, format_version, offsets, sizes, checksums per spec §9.

### ROM Header (nxrom_header_t)

- **Description**: First 128 bytes of a ROM file. Defines layout and metadata.
- **Attributes**: magic[4], format_version, flags, entry_point, code_offset/size, data_offset/size, asset_table_offset/size, total_rom_size, cycle_budget, screen_width/height, title[32], author[32], checksum, header_checksum, _reserved[8].
- **Validation**: Magic must be `NX32`; format_version must be supported by validator/emulator; header_checksum = CRC32(header bytes 0–119); checksum = CRC32(ROM excluding checksum and header_checksum fields).

### Asset Directory & Asset Entry

- **Description**: Index of packed assets (textures, meshes, audio, tilemaps, shaders, generic). Follows header and segment layout in spec §9.5–9.6.
- **Attributes**: num_assets; per entry: name[32], asset_type, format, rom_offset, compressed_size, uncompressed_size, target_region, target_address.
- **Relationships**: Referenced by ROM; asset data at rom_offset; target_region indicates Main RAM, VRAM, or Audio RAM.
- **Validation**: Offsets and sizes must lie within ROM; asset_type and target_region per spec.

### Emulator (conceptual)

- **Description**: Host application that implements the NEXUS-32 virtual machine. Not stored in this repo; behavior is defined by the spec.
- **Attributes**: Supported spec version (reported via SYS_VERSION); load/run/display/save behavior per spec.
- **Relationships**: Loads ROM; conforms to spec; reports version for compatibility.

### SDK / romtools (conceptual)

- **Description**: Toolchain that produces and validates ROMs. Not stored in this repo; output and validation rules are defined by the spec and contracts.
- **Attributes**: Produces ROMs with valid header and segments; validates format_version, checksums, and layout.
- **Relationships**: SDK + romtools produce ROMs; romtools validate against ROM format contract; both must conform to spec.

## State Transitions

- **Spec**: Draft → Versioned (1.0) → Amended (version bump + CHANGELOG). No formal state machine; version and CHANGELOG reflect current state.
- **ROM**: Source + assets → Built (SDK) → Packed (romtools) → Valid (romcheck) → Loadable by emulator. Invalid ROMs are rejected at validation or load with clear feedback per spec.

### Encoding Table (Round Two)

- **Description**: Machine-readable export of instruction encodings from the spec (e.g. CSV). One file per instruction set: integer (§2.3), vector (§2.4). Used by assemblers, disassemblers, and tests.
- **Attributes**: Mnemonic, opcode (hex), format type, encoding fields, cycles, spec section. Format defined in [contracts/encoding-table-format.md](contracts/encoding-table-format.md) or encoding-tables/README.md.
- **Relationships**: Derived from Specification; MUST match spec; no new encodings.
- **Validation**: Each row MUST correspond to a defined instruction in NEXUS32_Specification_v1.0.md §2.

### Diagram (Round Two)

- **Description**: Visual reference for architecture or memory map, stored in `diagrams/` (e.g. memory-map.md, architecture.md). Content is Markdown with Mermaid or ASCII; references spec §1, §3.
- **Attributes**: Title, spec section reference, format (Mermaid/ASCII). No binary images required.
- **Relationships**: Derived from Specification; for implementer reference only.
- **Validation**: Content MUST reflect the spec; no new interfaces or addresses.

### GPU Command Table (Round Three)

- **Description**: Machine-readable export of GPU command types from spec §5.2. Stored as `encoding-tables/gpu-commands.csv`. Used by emulator command-buffer parser and tools.
- **Attributes**: cmd_type_hex, name, size_bytes (or “variable”), spec_ref. One row per command (CMD_CLEAR through CMD_PRESENT).
- **Relationships**: Derived from Specification; MUST match spec §5.2; no new command types.
- **Validation**: Each row MUST correspond to a defined command in NEXUS32_Specification_v1.0.md §5.2.

### Shader Opcode Table (Round Three)

- **Description**: Machine-readable export of mini-shader opcodes from spec §5.6. Stored as `encoding-tables/shader-opcodes.csv`. Used by SDK shaderc and emulator shader→SPIR-V path.
- **Attributes**: opcode_hex, mnemonic, operation (short description), spec_ref. One row per opcode (MOV through NOP).
- **Relationships**: Derived from Specification; MUST match spec §5.6; no new opcodes.
- **Validation**: Each row MUST correspond to a defined opcode in NEXUS32_Specification_v1.0.md §5.6.

### I/O and System Register Reference (Round Four)

- **Description**: Consolidated reference document for memory-mapped I/O regions and system/timer/interrupt registers. Stored as `reference/io-and-system-registers.md`. Derived from spec §3 (memory map) and §8 (timer, system, interrupt). For implementer convenience; spec remains authoritative.
- **Attributes**: I/O region table (base, size, name, spec_ref); system registers (offset, name, access, description, spec_ref); optional reference/README.md.
- **Relationships**: Derived from Specification; MUST match spec §3 and §8; no new addresses or behavior.
- **Validation**: Every table row or section MUST cite the spec section it comes from; content MUST match NEXUS32_Specification_v1.0.md.

### Contributing and Spec-Change Workflow (Round Five)

- **Description**: Process documentation for proposing spec changes and versioning. Stored as CONTRIBUTING.md at repository root. References constitution and contracts; describes when to bump spec version and CHANGELOG hygiene.
- **Attributes**: How to propose changes; when to bump version; CHANGELOG rules; link to constitution and contracts.
- **Relationships**: Aligns with Constitution (Spec-first changes, Constitution check); references contracts/spec-version.md, rom-format.md; no new interfaces.
- **Validation**: Content MUST align with constitution and existing contracts; no new behavioral requirements.

### Implementation Checklist (Round Five, optional)

- **Description**: Optional checklist for emulator, SDK, and romtools to self-check conformance (e.g. ROM format contract, SYS_VERSION, version handling). Stored as docs/implementation-checklist.md (or equivalent). Not a test suite; human- or tool-checkable list.
- **Attributes**: Conformance items per repo type; links to contracts, reference/, encoding-tables.
- **Relationships**: Supports Constitution V (testability); references existing contracts and spec; no new contracts.
- **Validation**: Checklist items MUST trace to spec or contracts; no new requirements.

## Notes

- Implementation details (C structs, file formats) are in the main spec; this data model summarizes entities and rules for planning and contracts only.
- Round one documents and contracts the existing spec (§9, §13); round two adds Encoding Table and Diagram artifacts; round three adds GPU command and shader opcode tables; round four adds I/O and system register reference; round five adds contributing workflow and optional implementation checklist.
- For validation rules and header layout, this data model aligns with [contracts/rom-format.md](contracts/rom-format.md) and [contracts/spec-version.md](contracts/spec-version.md); the main spec (NEXUS32_Specification_v1.0.md) is the single source of truth.
