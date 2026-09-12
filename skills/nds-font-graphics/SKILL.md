---
name: nds-font-graphics
description: Use when editing Nintendo DS font files (NFTR), generating Cyrillic glyphs, adjusting VWF character widths, or translating tile graphics (NCGR, NCLR, NSCR, NCER)
---

# NDS Font & Graphics Localization

## Overview
Procedures and tool patterns for inspecting, editing, and generating Nintendo DS font resources (`.NFTR` files or raw 1bpp/2bpp/4bpp tile fonts), character width tables (Variable Width Fonts - VWF), and localized 2D graphical assets (NCGR/NCLR/NSCR/NCER).

## When to Use
- Adding Cyrillic (or other target language) glyphs to Nintendo standard `.NFTR` fonts or raw font sheets.
- Modifying character width tables (`CWDH` chunks) so localized letters render naturally without overlaps or abnormal gaps.
- Extracting and translating UI graphical banners, logos, chapter cards, battle commands, and world map graphics.
- Converting 4bpp/8bpp tiled graphics (`.NCGR`), palettes (`.NCLR`), and tilemaps (`.NSCR`) to PNG for editing and converting back to NDS binary format.

When NOT to use:
- Modifying dialogue scripts and pointer tables (use `nds-text-script-engine`).
- Modifying ARM9 font rendering routines (use `nds-asm-patching`).

## NFTR Font Architecture
An `.NFTR` (Nitro Font Resource) file contains standard chunks:
1. **`RTFN` / `NFTR` (Header)**: Magic bytes, version, total file size.
2. **`FNTR` (Font Header)**: Default width, height, bit-depth (1bpp, 2bpp, 4bpp).
3. **`PLGC` (Glyph Data Chunk)**: Raw bitmap data for each character.
4. **`HDAT` / `CWDH` (Character Width Chunk)**: Left spacing, glyph width, advance width per glyph.
5. **`CMAP` (Character Mapping Chunk)**: Maps character codes (Unicode, Shift-JIS, or direct indices) to glyph indices in `PLGC`.

### Modifying / Generating NFTR Fonts with `ndspy`
```python
import ndspy.font

# Load an existing NFTR font
with open("font.nftr", "rb") as f:
    font = ndspy.font.NFTR(f.read())

print(f"Font dimensions: {font.glyphWidth}x{font.glyphHeight}, depth: {font.bitsPerPixel} bpp")

# Inspect character mapping and widths
for char_code, glyph_idx in font.characterMap.items():
    width_info = font.widths[glyph_idx]
    # width_info contains: leftMargin, glyphWidth, advanceWidth
```

### Adding Cyrillic Glyphs via Python & Pillow
```python
from PIL import Image, ImageDraw, ImageFont

def render_cyrillic_glyph_sheet(font_path: str, size: int = 12):
    """Renders a standard Russian Cyrillic alphabet tile sheet."""
    ttf_font = ImageFont.truetype(font_path, size)
    chars = "АБВГДЕЁЖЗИЙКЛМНОПРСТУФХЦЧШЩЪЫЬЭЮЯабвгдеёжзийклмнопрстуфхцчшщъыьэюя"
    
    img = Image.new("L", (16 * size, 8 * size), color=0)
    draw = ImageDraw.Draw(img)
    
    for idx, ch in enumerate(chars):
        col = idx % 16
        row = idx // 16
        draw.text((col * size, row * size), ch, fill=255, font=ttf_font)
        
    img.save("cyrillic_glyphs.png")
```

---

## Proprietary `.fnt` (2bpp) Font Architecture

Many first-party and major third-party NDS games (such as *Chrono Trigger DS*) use proprietary 2bpp font formats rather than Nitro NFTR.

### 1. Binary Structure of `FONT` (2bpp)
```text
[0x00..0x03]: 4-byte padding (0x00000000)
[0x04..0x07]: Magic bytes b"FONT"
[0x08..0x09]: def_width (uint8), height_words (uint8, cell_h = def_width)
[0x0A..0x0D]: glyph_count (uint16), reserved (uint16)
[0x0E..StartOffsets]: char_map (array of uint16 character codes to glyph indices)
[StartOffsets..GlyphData]: offsets (array of uint32 offsets to each glyph record)
[GlyphData..EOF]: Array of glyph records:
    - width (uint8): Actual proportional pixel width of glyph
    - bpr (uint8): Bytes per row (ceil(width * 2 / 8))
    - bitmap (bpr * cell_h bytes): 2-bit per pixel little-endian packed pixels
```

In 2bpp bitmap rows, 4 pixels are packed per byte:
```python
# Unpacking 4 pixels from 1 byte in 2bpp
px0 = b & 0x03
px1 = (b >> 2) & 0x03
px2 = (b >> 4) & 0x03
px3 = (b >> 6) & 0x03
```

### 2. Reserving Glyph Base for Localization (`>= 450`)
To preserve original English/Japanese glyphs and script opcodes, localized Cyrillic glyphs are injected into reserved space at indices $\ge 450$:
```python
CYRILLIC_UPPER = "АБВГДЕЖЗИЙКЛМНОПРСТУФХЦЧШЩЪЫЬЭЮЯ"  # 32 chars
CYRILLIC_LOWER = "абвгдежзийклмнопрстуфхцчшщъыьэюя"  # 32 chars
CYRILLIC_SPECIAL = [("Ё", 64), ("ё", 65)]             # 2 chars (total 66 chars)

CYRILLIC_BASE_GLYPH = 450  # Keep 0..449 untouched for game UI & ASCII

# Map characters starting at index 450
char_to_glyph = {}
for idx, char in enumerate(list(CYRILLIC_UPPER) + list(CYRILLIC_LOWER) + ["Ё", "ё"]):
    char_to_glyph[char] = CYRILLIC_BASE_GLYPH + idx
```

### 3. 2bpp Anti-Aliasing Palette & Visual Grid
When exporting `.fnt` to PNG for editing in Aseprite/Photoshop, use an indexed 2bpp palette expanded with UI guide colors:
```python
GRID_COLOR_CELL = 4   # (50, 80, 140)  - Muted slate blue for cell border
GRID_COLOR_WIDTH = 5  # (180, 70, 50)  - Muted coral red for glyph width boundary

PALETTE_2BPP = [
    0, 0, 0,        # Index 0: Transparent background / Black
    255, 255, 255,  # Index 1: Main font body / White
    180, 180, 180,  # Index 2: Anti-aliasing / Light Gray
    90, 90, 90,     # Index 3: Outline / Shadow / Dark Gray
    50, 80, 140,    # Index 4: Cell border guide
    180, 70, 50,    # Index 5: Glyph width guide
] + [0] * (256 * 3 - 18)
```
- **Cell Border Rule**: The top and left border lines belong to the current cell, whereas the rightmost and bottom guide pixels mark the start of the next cell.
- **Proportional Width Marker**: Pixels between column 0 and `GRID_COLOR_WIDTH` are read as the proportional font advance width.

---

## Graphics Conversion: NCGR / NCLR / NSCR

Nintendo DS 2D background graphics use indexed color palettes and 8x8 pixel tiles:
- **NCLR (Nitro Color Resource)**: Color palette (16 or 256 colors in BGR555 format).
- **NCGR (Nitro Character Graphics Resource)**: Tiled image bitmap data.
- **NSCR (Nitro Screen Resource)**: Screen map indexing which tiles are drawn where and with which palette bank.

### Converting NCGR/NCLR to PNG
```python
import ndspy.color
import ndspy.graphics

def export_tiles_to_png(ncgr_bytes: bytes, nclr_bytes: bytes, output_png: str):
    palette = ndspy.color.NCLR(nclr_bytes)
    graphics = ndspy.graphics.NCGR(ncgr_bytes)
    
    # Convert tiles + palette to a Pillow Image
    pil_image = graphics.toPIL(palette)
    pil_image.save(output_png)

def import_png_to_ncgr(input_png: str, original_ncgr_bytes: bytes) -> bytes:
    img = Image.open(input_png)
    graphics = ndspy.graphics.NCGR(original_ncgr_bytes)
    graphics.fromPIL(img)
    return graphics.save()
```

---

## Smart Tile Deduplication (`deduplicate_tiles`)

When building backgrounds or sprite sheets, duplicate tiles must be eliminated to prevent exceeding the hardware tile memory limits.

```python
from collections import Counter
from typing import Dict, List, Tuple

class TilePoolOverflowError(Exception):
    """Raised when unique tile count exceeds Nintendo DS hardware limit (1024 tiles)."""
    pass

def deduplicate_tiles(
    tiles_8x8: List[List[int]],
    is_8bpp: bool,
    allow_flips: bool = True,
) -> Tuple[List[List[int]], List[Tuple[int, bool, bool, int]]]:
    """Deduplicates 8x8 pixel tiles with flip detection and 4bpp subpalette distribution.
    
    Returns:
        tuple: (unique_tiles, tile_placements)
        where each placement is (tile_index, h_flip, v_flip, palette_num)
    """
    unique_tiles: List[List[int]] = []
    tile_lookup: Dict[Tuple[int, ...], Tuple[int, bool, bool]] = {}
    placements: List[Tuple[int, bool, bool, int]] = []

    for tile_px in tiles_8x8:
        if is_8bpp:
            pal_num = 0
            norm_px = list(tile_px)
        else:
            # 4bpp: Identify palette bank (0..15) based on most frequent non-zero color
            non_zero = [p for p in tile_px if p != 0]
            if not non_zero:
                pal_num = 0
                norm_px = [0] * 64
            else:
                banks = [p // 16 for p in non_zero]
                pal_num = Counter(banks).most_common(1)[0][0]
                norm_px = []
                for p in tile_px:
                    if p == 0:
                        norm_px.append(0)
                    elif p // 16 == pal_num:
                        norm_px.append(p % 16)
                    else:
                        norm_px.append(0)  # Drop colors from other subpalettes

        t_norm = tuple(norm_px)
        if t_norm in tile_lookup:
            idx, hf, vf = tile_lookup[t_norm]
        else:
            idx = len(unique_tiles)
            max_tiles = 1024 if allow_flips else 256
            if idx >= max_tiles:
                raise TilePoolOverflowError(
                    f"Screen exceeded maximum allowable unique tiles ({max_tiles}). Got {idx + 1} unique tiles."
                )
            unique_tiles.append(norm_px)

            tile_lookup[t_norm] = (idx, False, False)
            if allow_flips:
                # Precompute H-flip, V-flip, and HV-flip representations
                tile_2d = [norm_px[i * 8 : (i + 1) * 8] for i in range(8)]
                t_h = tuple(p for row in tile_2d for p in reversed(row))
                t_v = tuple(p for row in reversed(tile_2d) for p in row)
                t_hv = tuple(p for row in reversed(tile_2d) for p in reversed(row))

                if t_h not in tile_lookup:
                    tile_lookup[t_h] = (idx, True, False)
                if t_v not in tile_lookup:
                    tile_lookup[t_v] = (idx, False, True)
                if t_hv not in tile_lookup:
                    tile_lookup[t_hv] = (idx, True, True)

            hf, vf = False, False

        placements.append((idx, hf, vf, pal_num))

    return unique_tiles, placements
```

---

## NCER Cell Bank Architecture & Hardware OAM

Nintendo DS sprite graphics are composed of NCGR (tiles) mapped via **NCER (Cell Bank)** containers into hardware OAM structures.

### 1. NCER Binary Block Structure
An NCER file uses the standard Nitro container format (`RECN` header) with 3 mandatory blocks:
```text
[Header 16B]: Magic b"RECN", Endian 0xFEFF, Version 0x0100, total_size, header_size (16), num_blocks (3)
├── Block 0: b"KBEC" (CEBK - Cell Bank)
│   ├── Payload Header (24B): num_cells, cell_bank_type, cell_data_offset, mapping_mode, vram_offset, unk1, unk2
│   ├── Cell Entries Table: N * 8 bytes [n_oam uint16, cell_attr uint16, oam_offset uint32]
│   └── OAM Attribute Pool: Total_OAMs * 6 bytes [attr0 uint16, attr1 uint16, attr2 uint16]
├── Block 1: b"LBAL" (LABL - Labels)
│   ├── String Offset Table: num_cells * 4 bytes
│   └── Null-terminated ASCII Strings: "CellAnime0\x00", etc.
└── Block 2: b"TXEU" (UEXT - Extended User Data)
    └── 4 bytes payload (0x00000000)
```

### 2. 12 Hardware OAM Dimensions
The NDS 2D engine does not support arbitrary sprite rectangles. Sprites must be assembled from 12 hardware OAM configurations:

| Shape Code (`Attr0[15:14]`) | Size Code (`Attr1[15:14]`) | Dimensions ($W \times H$) | Tile Count (8×8) | Tile Grid ($X \times Y$) |
| :--- | :--- | :--- | :--- | :--- |
| **0 (Square)** | **0** | **8×8** | 1 | 1×1 |
| 0 | **1** | **16×16** | 4 | 2×2 |
| 0 | **2** | **32×32** | 16 | 4×4 |
| 0 | **3** | **64×64** | 64 | 8×8 |
| **1 (Wide)** | **0** | **16×8** | 2 | 2×1 |
| 1 | **1** | **32×8** | 4 | 4×1 |
| 1 | **2** | **32×16** | 8 | 4×2 |
| 1 | **3** | **64×32** | 32 | 8×4 |
| **2 (Tall)** | **0** | **8×16** | 2 | 1×2 |
| 2 | **1** | **8×32** | 4 | 1×4 |
| 2 | **2** | **16×32** | 8 | 2×4 |
| 2 | **3** | **32×64** | 32 | 4×8 |

### 3. 1D Mapping Modes & Tile Addressing
In 1D OBJ tile mapping modes (Modes 0..4):
$$\text{tile\_multiplier} = 1 \ll \text{mapping\_mode}$$
- When reading from NCER:
  $$\text{raw\_tile} = (\text{attr2} \ \& \ 0\text{x}3\text{FF}) \times \text{tile\_multiplier}$$
- When packing into NCER:
  $$\text{attr2\_tile} = (\text{raw\_tile} // \text{tile\_multiplier}) \ \& \ 0\text{x}3\text{FF}$$

### 4. 2D Shelf Packing for Cell Sheets
To organize disjoint Cell graphics onto a single editable PNG atlas, use a 2D Shelf Packer:
1. Sort Cell components by height in descending order.
2. Maintain a running shelf with `shelf_x`, `shelf_y`, and `shelf_height`.
3. If a Cell exceeds `canvas_width` (e.g. 256px), open a new shelf at `shelf_y += shelf_height`.
4. Store `canvas_x`, `canvas_y`, `width`, `height`, and `min_x`, `min_y` in metadata JSON to enable exact 1:1 rebuilding back to binary NCER/NCGR.

---

## Common Mistakes & Troubleshooting
| Problem | Cause | Solution |
| :--- | :--- | :--- |
| Cyrillic letters render as '?' or boxes | Missing `CMAP` entries in `.NFTR` | Add mappings for Unicode range `0x0400..0x04FF` pointing to your new glyph indices. |
| Russian letters touch each other or overlap | Advance width in `CWDH` is too small | Set `advanceWidth = glyphWidth + 1` for proportional rendering. |
| Exported PNG has scrambled colors when re-imported | Palette order or color count changed | Preserve the exact 16-color or 256-color palette indexing when saving PNGs in image editors. |
| OAM tiles distorted or horizontally scrambled | Incorrect mapping_mode multiplier | Multiply/divide `attr2` tile field by `1 << mapping_mode` (1D mapping mode multiplier). |
| Sprite has 1px gap or seam in graphics editor | Did not subtract 1 from end coordinate | Always use inclusive boundary formula: $X_{\text{end}} = X_{\text{start}} + W - 1$. |
| Rebuilding NCER crashes game VM | Block padding or magic mismatch | Pad blocks KBEC and LBAL to 4-byte boundaries with `b'\x00'`; preserve block order RECN -> KBEC -> LBAL -> TXEU. |

