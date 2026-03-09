# Encoding Tables — NEXUS-32 Specification

This directory holds **CSV exports of instruction encodings** for the NEXUS-32 CPU and vector unit, as described in the main specification §13.1 (Repository Layout).

**Format contract**: Column layout and validation rules are defined in [specs/001-nexus32-spec-baseline/contracts/encoding-table-format.md](specs/001-nexus32-spec-baseline/contracts/encoding-table-format.md). The main spec is authoritative; tables are a convenience export for tools.

## CSV column format

- **Integer instructions** (`integer-instructions.csv`): `mnemonic`, `format` (R|I|J|S), `opcode_hex` (6-bit), `func_hex` (6-bit for R-type), `cycles`, `spec_ref` (e.g. 2.3). Additional columns for encoding fields (rs, rt, rd, shamt, immediate) may be included.
- **Vector instructions** (`vector-instructions.csv`): `mnemonic`, `vfunc_hex` (vector function code), `cycles`, `spec_ref` (e.g. 2.4). Additional columns for vs, vt, vd, elem, flag as needed.
- **GPU commands** (`gpu-commands.csv`): `cmd_type_hex` (16-bit), `name`, `size_bytes` (fixed size or "variable"), `spec_ref` (e.g. 5.2). One row per command from spec §5.2.
- **Shader opcodes** (`shader-opcodes.csv`): `opcode_hex` (8-bit), `mnemonic`, `operation` (short description), `spec_ref` (e.g. 5.6). One row per opcode from spec §5.6.

All numeric encoding fields are in hex. See the contract for full details.

## Contents

- Integer instruction set (R-type, I-type, J-type, S-type) — see spec §2.3
- Vector instruction set (V-type) — see spec §2.4
- GPU command types — see spec §5.2 (`gpu-commands.csv`)
- Mini-shader opcodes — see spec §5.6 (`shader-opcodes.csv`)

Add CSV (or other machine-readable) encoding tables here as they are exported; keep format documented and aligned with [NEXUS32_Specification_v1.0.md](../NEXUS32_Specification_v1.0.md).

**Conformance**: These encoding tables enable decoder and assembler conformance tests per NEXUS32_Specification_v1.0.md §2 (CPU/VU), and command-buffer and shader-compiler conformance tests per §5 (GPU commands, mini-shader opcodes). The specification is authoritative; on any discrepancy between a table and the spec, the spec wins. Update the tables when the spec changes (with a corresponding CHANGELOG entry).
