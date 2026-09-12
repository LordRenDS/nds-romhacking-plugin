---
name: nds-nitrofs-tools
description: Use when extracting, inspecting, modifying, compressing, or repacking Nintendo DS (NDS) ROM files, NitroFS filesystems, overlays, and archives
---

# NDS NitroFS & ROM Tools

## Overview
Provides procedures and code patterns for unpacking, navigating, modifying, and rebuilding Nintendo DS ROMs, NitroFS directory trees, overlays, and standard compression formats (LZ77/LZ10/LZ11/Huffman).

## When to Use
- Unpacking a `.nds` ROM into filesystem components (`data/`, `arm9.bin`, `arm7.bin`, overlays, banners).
- Decompressing and compressing sub-archives or stream data using LZ77 (types 0x10, 0x11), Huffman, or RLE.
- Rebuilding a modified filesystem back into a bootable `.nds` ROM with updated FAT/FNT tables.
- Managing ARM9 overlays (`overlay9_XXXX.bin`) and overlay table entries.

When NOT to use:
- Analyzing ARM9 assembly logic (use `nds-asm-patching`).
- Parsing game dialog bytecode/text formats (use `nds-text-script-engine`).

## Quick Reference (Python `ndspy`)

### 1. Unpacking & Exploring NitroFS
```python
import ndspy.rom
import os

# Load clean NDS ROM
rom = ndspy.rom.NintendoDSRom.fromFile("game.nds")

print(f"Game Title: {rom.name.decode('ascii', errors='ignore')}")
print(f"Game Code: {rom.idCode.decode('ascii', errors='ignore')}")
print(f"ARM9 size: {len(rom.arm9)} bytes, ARM7 size: {len(rom.arm7)} bytes")

# Iterate files in NitroFS
for path, file_id in rom.filenames.items():
    data = rom.getFileByName(path)
    print(f"File: {path} ({len(data)} bytes)")

# Save a file from NitroFS
os.makedirs("extracted/data", exist_ok=True)
for path in rom.filenames:
    full_path = os.path.join("extracted", path)
    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    with open(full_path, "wb") as f:
        f.write(rom.getFileByName(path))
```

### 2. Rebuilding ROM with Base ROM Preservation (`base_nds_path`)
Never rebuild an NDS ROM from scratch when modifying game assets. Always load from a clean base ROM (`base_nds_path`) to retain original hardware execution parameters, bus timings, banner CRC, and encrypted Secure Area:

```python
import ndspy.rom
import os
from typing import Optional

def build_rom(
    extracted_dir: str,
    output_nds_path: str,
    base_nds_path: Optional[str] = None,
) -> None:
    """Builds an NDS ROM from extracted files, preserving base ROM metadata."""
    if base_nds_path and os.path.isfile(base_nds_path):
        # Retains exact headers, execution parameters, and Secure Area (0x4000..0x7FFF)
        rom = ndspy.rom.NintendoDSRom.fromFile(base_nds_path)
    else:
        rom = ndspy.rom.NintendoDSRom()

    # 1. Update ARM9 and ARM7 binaries if present
    arm9_path = os.path.join(extracted_dir, "arm9.bin")
    if os.path.isfile(arm9_path):
        with open(arm9_path, "rb") as f:
            rom.arm9 = f.read()

    arm7_path = os.path.join(extracted_dir, "arm7.bin")
    if os.path.isfile(arm7_path):
        with open(arm7_path, "rb") as f:
            rom.arm7 = f.read()

    # 2. Update NitroFS files
    data_dir = os.path.join(extracted_dir, "data")
    if os.path.isdir(data_dir):
        for root, _, files in os.walk(data_dir):
            for file in files:
                full_path = os.path.join(root, file)
                rel_path = os.path.relpath(full_path, extracted_dir).replace("\\", "/")
                with open(full_path, "rb") as f:
                    rom.setFileByName(rel_path, f.read())

    # 3. Save modified ROM (FAT/FNT automatically recalculated)
    os.makedirs(os.path.dirname(os.path.abspath(output_nds_path)), exist_ok=True)
    rom.saveToFile(output_nds_path)
```

---

### 3. Multi-Layer LZ10 Stream Decompression & Compression
NDS background graphics and sub-containers frequently employ multiple nested layers of LZ10 compression:

```python
import ndspy.lz10
from typing import Tuple

def decompress_stream(raw_bytes: bytes) -> Tuple[bytes, int]:
    """Unwraps single- or multi-layer LZ10 compression until raw binary data is reached.

    Returns:
        tuple[bytes, int]: (unwrapped_bytes, layer_count)
    """
    current_data = raw_bytes
    layer_count = 0

    while len(current_data) >= 4 and current_data[0] == 0x10:
        # Stop unwrapping if a known uncompressed Nitro/game magic is already exposed
        if len(current_data) >= 8 and current_data[:4] in (
            b"NCG\x00", b"NCL\x00", b"NSC\x00", b"RECN", b"TEXT", b"FONT"
        ):
            break

        try:
            decomp = ndspy.lz10.decompress(current_data)
            layer_count += 1
            current_data = decomp
        except Exception:
            # Not a valid LZ10 stream or corrupt; stop unwrapping
            break

    return current_data, layer_count

def compress_stream(data: bytes, layers: int = 1) -> bytes:
    """Applies `layers` passes of standard LZ10 compression to data."""
    result = data
    for _ in range(layers):
        result = ndspy.lz10.compress(result)
    return result
```

---

### 4. ARM9 Overlays & Compression Flag Management
ARM9 overlays (`overlay9_XXXX.bin`) are loaded dynamically into main memory. Each overlay's metadata is defined in the ARM9 overlay table (`rom.arm9OverlayTable`):

```python
def update_arm9_overlay(
    rom: ndspy.rom.NintendoDSRom,
    overlay_id: int,
    modified_data: bytes,
    compress: bool = False,
) -> None:
    """Updates an ARM9 overlay and ensures its compression flag matches."""
    if overlay_id not in rom.arm9OverlayTable:
        raise KeyError(f"Overlay ID {overlay_id} not in ARM9 overlay table")

    entry = rom.arm9OverlayTable[overlay_id]

    if compress:
        compressed_data = ndspy.lz10.compress(modified_data)
        # Set compression flag: bit 24 or flag bit 0
        entry.flags |= 0x01000000
        data_to_store = compressed_data
    else:
        # Clear compression flag
        entry.flags &= ~0x01000000
        data_to_store = modified_data

    # Store overlay binary and commit updated table
    rom.files[entry.fileID] = data_to_store
    rom.arm9OverlayTable[overlay_id] = entry
```

---

## Common Mistakes & Edge Cases
| Issue | Cause | Solution |
| :--- | :--- | :--- |
| Game crashes immediately on boot | Rebuilt ROM without `base_nds_path` | Always pass `base_nds_path` referencing the original clean dump to preserve secure area and hardware timings. |
| Modified overlay executes garbage or crashes | Overlay table compression flag mismatch | Ensure `entry.flags` bit 24 matches whether overlay binary is compressed with LZ10. |
| Nested LZ10 graphics corrupted after rebuild | Compressed with wrong layer count | Use `decompress_stream` to record `layer_count` upon extraction, and `compress_stream` with the identical count upon packing. |
| Decompressed file truncated | File starts with custom container header before LZ header | Inspect file header; slice offset to the actual `0x10` magic byte before decompression. |

