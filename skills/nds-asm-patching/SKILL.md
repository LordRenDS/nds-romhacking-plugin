---
name: nds-asm-patching
description: Use when disassembling, patching, injecting ARM9/Thumb assembly code, bypassing Anti-Piracy checks, or using armips for NDS ROM hacking
---

# NDS ARM9 Assembly Patching & Anti-Piracy

## Overview
Techniques for analyzing, modifying, hooking, and injecting ARM9 (ARMv5TE) and Thumb assembly code in Nintendo DS binaries using `armips`, Ghidra, and direct byte patching.

## When to Use
- Implementing custom game engine hacks (e.g., Variable Width Font (VWF) render loops, expanding string length limits).
- Bypassing Anti-Piracy (AP) checks and copy protection routines in ARM9/overlays.
- Hooking text-rendering or script-interpreting functions to support 2-byte character encodings.
- Writing clean, relocatable ASM patches with `armips`.

When NOT to use:
- Extracting or repacking file archives without code modification (use `nds-nitrofs-tools`).
- Dumping standard dialogue text (use `nds-text-script-engine`).

## Memory Architecture & Offsets
- **NDS ARM9 RAM Base**: `0x02000000` (Main 4MB RAM, `0x02000000..0x023FFFFF`).
- **ARM9 Entry Point**: Typically `0x02000800` or defined in NDS ROM header at offset `0x024`.
- **Instruction Modes**:
  - **ARM Mode** (32-bit instructions): 4-byte aligned.
  - **Thumb Mode** (16-bit instructions): 2-byte aligned, bit 0 of jump target set to 1.

## Writing Assembly Patches with `armips`

`armips` is the standard assembler for NDS ROM hacking:

```armips
.nds
.open "clean_game.nds", "patched_game.nds", 0x02000000

// Target ARM9 binary in memory
.arm
.org 0x020045A0
    // Example: Hook into text draw routine
    bl      MyCustomVwfHandler

// Code cave for new routine (in unused padding area)
.org 0x020E8000
MyCustomVwfHandler:
    push    {r4-r7, lr}
    // Custom Cyrillic width calculation logic...
    pop     {r4-r7, pc}

.close
```

## NDS Anti-Piracy (AP) Bypass Patterns

Commercial NDS titles frequently implement multi-stage AP checks:
1. **Common Failure Modes**: If AP flags are tripped, games trigger intentional softlocks (infinite warp loops, disabled event triggers, freezes during map transitions, unresponsive touchscreen/buttons, or corrupted save games).
2. **Detection Vectors**: Integrity checks inspecting cartridge bus control registers (`0x040001A4` / `REG_ROMCTRL`, `0x04100010` / `REG_EXMEMCNT`), bus response timings, or RAM code checksums in `arm9.bin` and overlay routines.
3. **Branch Patching**: Replacing conditional branch instructions (`BNE` / `BEQ`) with unconditional branches (`B`) or `NOP` instructions (`0x00, 0x00, 0xA0, 0xE1` in ARM / `0x00, 0x46` in Thumb) at check locations.

```python
def apply_ap_branch_patch(
    binary_bytes: bytearray,
    check_signature: bytes,
    bypass_patch: bytes,
) -> bool:
    """Applies generic AP bypass signature patches to arm9.bin or overlay binaries."""
    idx = binary_bytes.find(check_signature)
    if idx != -1:
        binary_bytes[idx : idx + len(bypass_patch)] = bypass_patch
        return True
    return False
```

## Ghidra Workflow for NDS ARM9
When importing `arm9.bin` into Ghidra:
1. **Language**: `ARM:LE:32:v5t` or `ARM:LE:32:v5te` (Little Endian).
2. **Base Address**: Set Image Base to `0x02000000` (or `0x02004000` depending on header load address).
3. **Disassembly**: Toggle between ARM and Thumb modes using Set Register Values (`TMode = 1` for Thumb, `TMode = 0` for ARM).

## Common Mistakes
- **Branch target misalignment**: In Thumb mode, jumping to an odd address (e.g., `0x02001001`) sets the CPU to Thumb execution. Jumping to an even address without setting TMode will cause an Undefined Instruction crash.
- **Overwriting active code**: Always verify with Ghidra or memory search that your code cave (`.org 0x...`) contains only zeroes or dead bytes across all game scenes.
