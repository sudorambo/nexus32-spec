# Reference — NEXUS-32 Specification

This directory holds **consolidated reference documents** derived from [NEXUS32_Specification_v1.0.md](../NEXUS32_Specification_v1.0.md). They are for implementer convenience only. **The specification is authoritative**; on any discrepancy between a reference document and the spec, the spec wins.

These references enable implementers to verify memory-mapped I/O and system/register layout against the spec (§3, §8). Use them for conformance checking; the spec remains the single source of truth.

## Contents

- **[io-and-system-registers.md](io-and-system-registers.md)** — Memory-mapped I/O regions (§3) and system/timer/interrupt registers (§8). Use for base addresses and register layouts when implementing emulator or SDK.
