# Contract: Encoding Table Format

**Source of truth**: NEXUS32_Specification_v1.0.md §2 (CPU, Vector Unit)  
**Feature**: 001-nexus32-spec-baseline (Round Two)  
**Date**: 2026-03-08

This contract describes the format of instruction encoding CSV files in `encoding-tables/`. The main spec is authoritative; these tables are a convenience export for tools (assemblers, disassemblers, tests).

## File Format

- **Encoding**: UTF-8 CSV.
- **Header row**: First row lists column names.
- **Separator**: Comma (`,`). No comma inside quoted fields.
- **Spec reference**: Tables MUST match NEXUS32_Specification_v1.0.md §2.3 (integer) and §2.4 (vector). Any discrepancy is resolved in favor of the spec.

## Integer Instruction Table (`integer-instructions.csv`)

Suggested columns (names may vary; document in encoding-tables/README.md):

| Column    | Description                    | Example |
|-----------|--------------------------------|---------|
| mnemonic  | Instruction name               | ADD     |
| format    | R / I / J / S                  | R       |
| opcode_hex| Primary opcode (6-bit) hex     | 00      |
| func_hex  | Func field for R-type (6-bit)  | 20      |
| cycles    | Cycle cost per spec            | 1       |
| spec_ref  | Spec section (e.g. §2.3)       | 2.3     |

Additional columns for encoding fields (rs, rt, rd, immediate, shamt, etc.) may be included as needed. All numeric encoding fields in hex unless noted.

## Vector Instruction Table (`vector-instructions.csv`)

Suggested columns:

| Column     | Description                    | Example |
|------------|--------------------------------|---------|
| mnemonic   | Instruction name               | VADD    |
| vfunc_hex  | Vector function code (vfunc)   | 00      |
| cycles     | Cycle cost per spec            | 2       |
| spec_ref   | Spec section (e.g. §2.4)       | 2.4     |

Additional columns for vs, vt, vd, elem, flag, etc. as needed. All numeric in hex unless noted.

## Validation

- Tools that consume these tables MUST treat the spec as authoritative. If a table row conflicts with the spec, the spec wins.
- New instructions or encoding changes require a spec update and CHANGELOG; tables are updated in the same or a follow-up change.
- Round two deliverables: at least `integer-instructions.csv` and `vector-instructions.csv` populated from spec §2.3 and §2.4.
