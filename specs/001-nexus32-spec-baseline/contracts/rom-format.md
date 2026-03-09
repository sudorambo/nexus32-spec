# Contract: ROM Format

**Source of truth**: NEXUS32_Specification_v1.0.md §9  
**Feature**: 001-nexus32-spec-baseline  
**Date**: 2026-03-08

This contract summarizes the ROM image format for implementers (romtools, emulator). The main spec is authoritative; this document is a stable reference for validation and loading.

## File and Magic

- **File extension**: `.nxrom`
- **Magic bytes**: `NX32` (ASCII), 4 bytes: `0x4E, 0x58, 0x33, 0x32`
- **Byte order**: Little-endian (NEXUS-32 is little-endian)

## Header (128 bytes)

| Offset | Size  | Field               | Type    | Description |
|--------|-------|---------------------|---------|-------------|
| 0      | 4     | magic               | uint8[4]| Must be "NX32" |
| 4      | 2     | format_version      | uint16  | ROM format version (current: 0x0100 = 1.0) |
| 6      | 2     | flags               | uint16  | Bit 0: compressed assets; Bit 1: save data required |
| 8      | 4     | entry_point         | uint32  | Entry point address in Main RAM |
| 12     | 4     | code_offset         | uint32  | Offset of code segment in ROM file |
| 16     | 4     | code_size           | uint32  | Code segment size in bytes |
| 20     | 4     | data_offset         | uint32  | Offset of data segment in ROM file |
| 24     | 4     | data_size           | uint32  | Data segment size in bytes |
| 28     | 4     | asset_table_offset  | uint32  | Offset of asset directory in ROM file |
| 32     | 4     | asset_table_size    | uint32  | Asset directory size in bytes |
| 36     | 4     | total_rom_size      | uint32  | Total ROM file size in bytes |
| 40     | 4     | cycle_budget        | uint32  | Per-frame cycle budget (0 = default 3M) |
| 44     | 2     | screen_width        | uint16  | Preferred width (0 = default 640) |
| 46     | 2     | screen_height       | uint16  | Preferred height (0 = default 480) |
| 48     | 32    | title               | char[32]| Game title (UTF-8, null-terminated) |
| 80     | 32    | author              | char[32]| Author/studio (UTF-8, null-terminated) |
| 112    | 4     | checksum            | uint32  | CRC32 of entire ROM excluding this field and header_checksum |
| 116    | 4     | header_checksum     | uint32  | CRC32 of header bytes 0–119 |
| 120    | 8     | _reserved           | uint8[8]| Must be 0 |

## Validation Rules (Accept / Reject)

All header fields and validation rules in this section are defined in **NEXUS32_Specification_v1.0.md §9** (ROM Image Format).

Implementers (e.g. romcheck) MUST:

1. **Accept** if: magic is "NX32"; format_version is supported (e.g. 0x0100 for spec 1.0); header_checksum equals CRC32(header bytes 0–119); checksum equals CRC32(ROM excluding checksum and header_checksum); all offsets and sizes are within file and consistent with total_rom_size; code_size ≤ 4 MB (spec §9.3); data_size ≤ 4 MB (spec §9.4); total_rom_size ≤ 256 MB (spec §9.8); header _reserved bytes (offset 120–127) are zero.
2. **Reject** with clear feedback if: magic wrong; format_version not supported; checksum mismatch; offset/size out of range or inconsistent; reserved bytes (120–127) non-zero; code_size or data_size exceeds 4 MB; total_rom_size exceeds 256 MB.

## Forward Compatibility

- ROM format versions are forward-compatible: an emulator supporting a newer spec version MUST run ROMs built for older format versions (e.g. 0x0100).
- New format versions MUST use additive or reserved fields; validators MAY reject format_versions they do not support and MUST report the unsupported version to the user.

## Segments and Asset Directory

- Code segment: at `code_offset`, length `code_size`; loaded to Main RAM at entry_point.
- Data segment: at `data_offset`, length `data_size`; loaded after code per spec.
- Asset directory: at `asset_table_offset`, length `asset_table_size`; layout per spec §9.5–9.6.

Full segment and asset layout are defined in the main specification §9.
