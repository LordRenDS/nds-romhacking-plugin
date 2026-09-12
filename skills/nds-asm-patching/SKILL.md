---
name: nds-asm-patching
description: Use when disassembling, patching, injecting ARM9/Thumb assembly code, bypassing Anti-Piracy checks, or using armips for NDS ROM hacking
---

# NDS ARM9 Assembly Patching & Anti-Piracy

## Overview
Techniques for analyzing, modifying, hooking, and injecting ARM9 (ARMv5TE) and Thumb assembly code in Nintendo DS binaries using `armips`, Ghidra, and direct byte patching.

## When to Use
- Implementing custom game engine hacks (e.g., Variable Width Font (VWF) render loops, expanding string length limits).
- Bypassing Anti-Piracy (AP) checks (such as the famous Chrono Trigger DS Millennial Fair telepod infinite time-warp lock).
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

`armips` is the gold standard assembler for NDS ROM hacking:

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

## Chrono Trigger (NDS) Anti-Piracy (AP) Bypass Patterns

Chrono Trigger DS implements multi-stage AP checks:
1. **Telepod Portal Lock**: If anti-piracy flags are tripped, entering the telepod at the Millennial Fair locks the player in an infinite hyperspace animation.
2. **Detection Mechanism**: The game performs integrity checks and reads specific hardware register values in `arm9.bin` or overlay routines.

### Common AP Fix Byte Signatures (USA / EUR)
For Chrono Trigger (USA - `YCTE`):
- AP checks often test a register return value or compare RAM checksums.
- Typical patch: Replacing conditional branch instructions (`BNE` / `BEQ`) with unconditional branches (`B`) or `NOP` (`0x00, 0x00, 0xA0, 0xE1` in ARM / `0x00, 0x46` in Thumb) at the check locations.

```python
def apply_ct_ap_patch(arm9_bytes: bytearray) -> bool:
    """Applies known AP bypass patches to Chrono Trigger USA arm9.bin."""
    # Example signature pattern search
    pattern = b"\x00\x00\x50\xE3\x02\x00\x00\x0A"  # CMP r0, #0; BEQ target
    replacement = b"\x00\x00\x50\xE3\x02\x00\x00\xEA"  # CMP r0, #0; B target
    
    idx = arm9_bytes.find(pattern)
    if idx != -1:
        arm9_bytes[idx:idx+len(replacement)] = replacement
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
- **Overwriting active code**: Always verify with Ghidra or memory search that your code cave (`.org 0x...`) contains only zeroes or dead bytes in all game scenes.
