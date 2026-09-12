# NDS ROM Hacking & Localization Guidelines

When working with Nintendo DS (NDS) ROM files, NitroFS, binary scripts, fonts, graphics, and ARM9 assembly, you must strictly follow these rules and engineering standards.

---

## 1. ROM & NitroFS Integrity

- **Strict Non-Destructive Source Policy**: Always keep a pristine, untouched backup of the original `.nds` clean dump. All modifications, unpacks, and patches must be performed on working copies or generated output files.
- **Base ROM Preservation (`base_nds_path`)**: When repacking or synthesizing a modified ROM, always initialize from a clean original base ROM via `base_nds_path` (e.g. `ndspy.rom.NintendoDSRom.fromFile(base_nds_path)`). Assembling a ROM from scratch without a base template drops critical execution parameters, ARM9/ARM7 hardware bus timings, banner metadata, and the encrypted Secure Area (0x4000..0x7FFF), resulting in immediate boot crashes or anti-tamper triggers on real hardware.
- **4-Byte Alignment Rule**: ARM architectures and the NDS BIOS memory controller expect data structures, file chunks, and code sections to be 4-byte (word) aligned. Always pad inserted binary tables, archives, and files to 4-byte boundaries (`pad_to(4, b'\x00')` or `0xFF` depending on target container).
- **FAT & FNT Consistency**: When files inside NitroFS are added, deleted, or resized, always update the File Allocation Table (FAT) and File Name Table (FNT) coherently. Prefer using high-level libraries (`ndspy.rom.NintendoDSRom`) that automatically recalculate offsets and table sizes upon saving.

---

## 2. Pointer Tables & Script Math

- **Pointer Type Identification**: Always verify whether pointer tables use:
  - **Absolute ROM Offsets** (rare inside sub-files, common in `arm9.bin`).
  - **Relative Offsets** (relative to the start of the text block, archive header, or entry count table).
  - **RAM Memory Addresses** (e.g., mapped into ARM9 main memory `0x02000000..0x023FFFFF`).
- **Atomic Pointer Updates**: Never modify a text string without simultaneously adjusting its corresponding pointer in the pointer table.
- **Repoint Overflows**: If localized text exceeds original section boundaries, allocate space at the end of the data chunk and safely repoint all references.

---

## 3. Script Engine & Control Code Preservation

- **Strict Control Code Isolation**: Dialogues in NDS RPGs contain embedded VM opcodes (`{PAGE}`, `{LINE}`, `{COLOR N}`, `{NAME N}`, `{WAIT N}`, `{CHOICE}`, etc.). Never strip, reorder, or alter opcode parameters unless intentionally hacking the game's script engine logic.
- **Escape Sequence Standard**: Always represent control codes using clear bracketed syntax (`{TAG_NAME:PARAM}`) in human-readable dumps, ensuring robust bidirectional round-tripping between text format (PO/JSON) and raw bytecode.

---

## 4. Fonts, Glyphs & Width (VWF)

- **Variable Width Font (VWF) Metrics**: When introducing localized (e.g., Cyrillic) glyphs into NFTR or custom bitmap fonts, always define individual character width and glyph advance metrics.
- **Dynamic Hero Token Preservation**: When calculating string pixel widths for pre-insertion validation, never strip dynamic hero tokens (`{HERO_NAME}`, `{CRONO}`, `{MARLE}`, etc.) like non-visual formatting tags. Assign hero tokens a nominal width (`DEFAULT_HERO_NAME_WIDTH_PX = 30`) to prevent dialogue box overflow when in-game names are substituted.
- **ARM9 Hardcoded Font Offsets**: Before expanding font headers, charmaps, or glyph pools, inspect the ARM9 binary and relevant overlay modules for hardcoded memory offsets or struct assumptions. Modifying a font header layout without patching referencing code can cause memory corruption or character misalignment.
- **Pixel-Art Priority over TTF for 256x192**: For native Nintendo DS resolution (256x192), hand-crafted pixel-art glyphs tailored to the exact 8x8/10x10 cell bounds and 1bpp/2bpp anti-aliasing palettes must strictly take priority over automated TrueType (TTF) rasterization. Downscaled TTF fonts cause muddy anti-aliasing and illegible sub-pixel artifacts on DS LCD screens.
- **Line Length & Overflow Pre-Checks**: Calculate string pixel widths against target text box bounding limits prior to ROM insertion. Russian/localized translations are typically 20–35% wider than English/Japanese source text.

---

## 5. Anti-Piracy (AP) & Code Injection

- **Anti-Piracy (AP) Awareness**: Games like *Chrono Trigger (NDS)* include anti-tampering triggers in `arm9.bin` or overlay modules. Before and during testing, verify whether the target game has AP checks and apply verified AP bypass patches where necessary to avoid softlocks (such as infinite warp loops).
- **Safe Code Cave Allocation**: When injecting ARM9/Thumb ASM hooks via `armips`, place new code strictly in verified unused padding/caves (e.g., end of `arm9.bin` or unused RAM regions) without overwriting active heap/stack memory.

---

## 6. OAM & NCER Hardware Architecture

- **12 Hardware OAM Dimensions**: The NDS 2D Graphics Engine (2D OAM) only supports 12 fixed hardware sprite dimensions across 3 shape categories (`OBJ_SHAPE` in Attribute 0, `OBJ_SIZE` in Attribute 1):
  - **Square (Shape 0)**: `8×8` (1 tile), `16×16` (4 tiles), `32×32` (16 tiles), `64×64` (64 tiles).
  - **Wide (Shape 1)**: `16×8` (2 tiles), `32×8` (4 tiles), `32×16` (8 tiles), `64×32` (32 tiles).
  - **Tall (Shape 2)**: `8×16` (2 tiles), `8×32` (4 tiles), `16×32` (8 tiles), `32×64` (32 tiles).
- **1D Mapping Modes (16K–256K)**: In 1D OBJ tile mapping, the tile index stored in OAM Attribute 2 is multiplied by the mapping mode step:
  $$\text{tile\_multiplier} = 1 \ll \text{mapping\_mode}$$
  Modes 0..4 represent boundary increments: 0 (16KB, multiplier=1), 1 (32KB, multiplier=2), 2 (64KB, multiplier=4), 3 (128KB, multiplier=8), 4 (256KB, multiplier=16). Always decode raw NCGR tile offset as `raw_tile = (attr2 & 0x3FF) * tile_multiplier` and re-encode as `(raw_tile // tile_multiplier) & 0x3FF`.
- **Tile Occlusion & Coverage Accounting**: When composing cell sheets (NCER), OAM pieces may overlap or leave gaps. Tools must track both covered and uncovered tiles (`uncovered` blocks) to guarantee that every tile in NCGR is accounted for without corrupting shared graphics memory or producing seams.
- **Inclusive Pixel Coordinate Formula**: In raster editors (Aseprite, Photoshop, GIMP) and canvas calculations, coordinates are 0-indexed discrete points. The ending boundary must always subtract 1:
  $$X_{\text{end}} = X_{\text{start}} + W - 1, \quad Y_{\text{end}} = Y_{\text{start}} + H - 1$$
  Failing to subtract 1 introduces a 1-pixel drift that shifts adjacent OAM blocks and destroys tile alignment.

