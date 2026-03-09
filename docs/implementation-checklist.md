# Implementation Checklist — NEXUS-32 Conformance

This checklist helps **emulator**, **SDK**, and **romtools** implementers self-check conformance against the NEXUS-32 specification and contracts. The [specification](NEXUS32_Specification_v1.0.md) is authoritative; this document is a convenience. Use the linked contracts and reference docs for normative details.

**Contracts**: [specs/001-nexus32-spec-baseline/contracts/](../specs/001-nexus32-spec-baseline/contracts/)  
**Reference (I/O and registers)**: [reference/io-and-system-registers.md](../reference/io-and-system-registers.md)  
**Encoding tables**: [encoding-tables/](../encoding-tables/) (CPU/VU §2, GPU/shaders §5)

---

## All Implementations

- [ ] **Spec as SSOT**: No behavior or interface is introduced that is not defined or permitted by [NEXUS32_Specification_v1.0.md](../NEXUS32_Specification_v1.0.md).
- [ ] **Version handling**: Understand [spec-version vs. ROM format version](specs/001-nexus32-spec-baseline/contracts/spec-version.md); reject or report unsupported format versions clearly.

---

## Romtools (ROM pack, validate, inspect)

- [ ] **ROM format contract**: Validation and packing follow [contracts/rom-format.md](../specs/001-nexus32-spec-baseline/contracts/rom-format.md): magic `NX32`, supported `format_version` (e.g. 0x0100), header layout, checksums, segment offsets/sizes within limits.
- [ ] **Accept/reject rules**: Valid ROMs (per contract) are accepted; invalid or unsupported-version ROMs are rejected with clear, actionable feedback.
- [ ] **No silent undefined behavior**: Unsupported format version must not be run with undefined behavior.

---

## Emulator

- [ ] **ROM loading**: Load ROMs per [contracts/rom-format.md](../specs/001-nexus32-spec-baseline/contracts/rom-format.md); reject unsupported `format_version`.
- [ ] **SYS_VERSION**: Report supported spec version via system register at 0x0B006000 per [contracts/spec-version.md](../specs/001-nexus32-spec-baseline/contracts/spec-version.md) and [reference/io-and-system-registers.md](../reference/io-and-system-registers.md).
- [ ] **Memory map and I/O**: Implement memory map and I/O regions per spec §3 and [reference/io-and-system-registers.md](../reference/io-and-system-registers.md).
- [ ] **CPU/VU semantics**: Instruction encodings and execution follow spec §2; use [encoding-tables/](../encoding-tables/) for decoder/assembler tests (spec is authoritative).
- [ ] **Exceptions**: ADD/ADDI trap on signed overflow (CAUSE_OVERFLOW); unaligned LW/SW/LH/SH/LHU raise CAUSE_ADDR_ALIGN; unknown opcodes raise CAUSE_ILLEGAL (not silently treated as NOP).
- [ ] **MUL/DIV/MOD**: Implement MUL, MULH, DIV, DIVU, MOD with correct cycle costs per spec §2.3.
- [ ] **Data segment bounds check**: Reject ROMs whose code + data segments exceed Main RAM capacity.
- [ ] **GPU command buffer**: Command interpretation follows spec §5; use [encoding-tables/gpu-commands.csv](../encoding-tables/gpu-commands.csv) and [encoding-tables/shader-opcodes.csv](../encoding-tables/shader-opcodes.csv) as reference (spec §5.2, §5.6 is authoritative).
- [ ] **Determinism**: Observable behavior (cycle budget, DMA, GPU, APU) matches spec; host-side optimizations only when game-observable behavior is correct.

### Debug Overlay (ClearUI)

The emulator uses [ClearUI](../../clearui-1.1.1/) as an immediate-mode debug overlay (registers, disassembly, memory) composited over the Vulkan framebuffer. For full build setup, Vulkan integration details, gotchas, and how to add widgets, see the integration guide:

> **[ClearUI Integration Guide](../../nexus32-emulator/docs/clearui-integration.md)**

---

## SDK (compiler, assembler, linker, tools)

- [ ] **ROM output**: Produced ROMs have valid header and segments per [contracts/rom-format.md](../specs/001-nexus32-spec-baseline/contracts/rom-format.md); target format_version (e.g. 0x0100) for spec 1.0.
- [ ] **Instruction encodings**: Assembler/compiler use spec §2; [encoding-tables/integer-instructions.csv](../encoding-tables/integer-instructions.csv) and [encoding-tables/vector-instructions.csv](../encoding-tables/vector-instructions.csv) for reference (spec is authoritative). Assembler must support the full integer ISA including MUL/DIV/MOD, variable shifts, all load/store widths, and comparison branches.
- [ ] **Linker**: `nxld` reports all undefined symbols as errors (does not silently link). Supports `-e <symbol>` for custom entry point.
- [ ] **libnx**: Standard library modules (crt0, sys, gfx, input, etc.) are implemented in assembly and assembled into `libnx.a` via `nxasm`.
- [ ] **Shader tooling**: If implementing shader compiler, use spec §5.6 and [encoding-tables/shader-opcodes.csv](../encoding-tables/shader-opcodes.csv) (spec is authoritative).

---

## Notes

- **Spec wins**: On any discrepancy between this checklist (or encoding tables or reference docs) and the specification, the spec wins. Update your implementation and tests against [NEXUS32_Specification_v1.0.md](../NEXUS32_Specification_v1.0.md).
- **Constitution**: Broader governance (spec-first changes, forward compatibility, testability) is in [.specify/memory/constitution.md](../.specify/memory/constitution.md).
