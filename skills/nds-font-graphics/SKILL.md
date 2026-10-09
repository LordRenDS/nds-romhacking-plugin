---
name: nds-font-graphics
description: Use when editing Nintendo DS font files (NFTR), generating Cyrillic glyphs, adjusting VWF character widths, or translating tile graphics (NCGR, NCLR, NSCR, NCER)
---

# NDS Font & Graphics Localization

## Overview
Procedures and tool patterns for inspecting, editing, and generating Nintendo DS font resources (`.NFTR` files or raw 1bpp/2bpp/4bpp tile fonts), character width tables (Variable Width Fonts - VWF), localized 2D graphical assets (NCGR/NCLR/NSCR/NCER), and 12-shape hardware OAM partitioning.

## When to Use
- Adding Cyrillic (or other target language) glyphs to Nintendo standard `.NFTR` fonts or raw font sheets.
- Modifying character width tables (`CWDH` chunks) so localized letters render naturally without overlaps or abnormal gaps.
- Extracting and translating UI graphical banners, logos, chapter cards, battle commands, and world map graphics.
- Converting 4bpp/8bpp tiled graphics (`.NCGR`), palettes (`.NCLR`), and tilemaps (`.NSCR`) to PNG for editing and converting back to NDS binary format.
- Partitioning arbitrary rectangular graphics into valid 12-shape hardware OAM cell structures (`.NCER`).

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

Many first-party and third-party NDS games use proprietary 2bpp font formats rather than standard Nitro NFTR.

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
To preserve original English/Japanese glyphs and script opcodes, localized Cyrillic glyphs are typically injected into reserved space at higher indices (e.g. $\ge 450$):
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
from PIL import Image

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

When building backgrounds or sprite sheets, duplicate tiles must be eliminated to prevent exceeding hardware tile memory limits.

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

---

## 2D Greedy Rectangle Partitioner

When importing or localizing custom sprite rectangles (e.g. translated UI buttons, banners, dialogue plaques), the graphic must be sliced into valid hardware OAM pieces. The 2D Greedy Rectangle Partitioner covers any rectangular region using the minimum count of valid hardware shapes:

```python
from typing import Any, Dict, List, Tuple

# All 12 hardware NDS OAM dimensions sorted by area in descending order
# (w, h, shape, size)
HARDWARE_OAM_SHAPES: List[Tuple[int, int, int, int]] = [
    (64, 64, 0, 3),  # 4096 px (64 tiles)
    (64, 32, 1, 3),  # 2048 px (32 tiles)
    (32, 64, 2, 3),  # 2048 px (32 tiles)
    (32, 32, 0, 2),  # 1024 px (16 tiles)
    (32, 16, 1, 2),  #  512 px (8 tiles)
    (16, 32, 2, 2),  #  512 px (8 tiles)
    (16, 16, 0, 1),  #  256 px (4 tiles)
    (32,  8, 1, 1),  #  256 px (4 tiles)
    (8,  32, 2, 1),  #  256 px (4 tiles)
    (16,  8, 1, 0),  #  128 px (2 tiles)
    (8,  16, 2, 0),  #  128 px (2 tiles)
    (8,   8, 0, 0),  #   64 px (1 tile)
]

def partition_rect_to_oam(
    width: int,
    height: int,
    offset_x: int = 0,
    offset_y: int = 0,
) -> List[Dict[str, Any]]:
    """Greedily partitions an arbitrary bounding box into the minimum number of
    valid 2D hardware NDS OAM pieces.

    Snaps width and height to 8-pixel tile boundaries.

    Returns:
        List of dicts:
        [{
            "x": int,
            "y": int,
            "width": int,
            "height": int,
            "shape": int,        # Attr0 shape: 0=Square, 1=Wide, 2=Tall
            "size": int,         # Attr1 size: 0..3
            "tile_count": int,   # Number of 8x8 tiles (w * h // 64)
        }, ...]
    """
    grid_w = (width + 7) // 8
    grid_h = (height + 7) // 8

    covered = [[False] * grid_w for _ in range(grid_h)]
    pieces: List[Dict[str, Any]] = []

    for gy in range(grid_h):
        for gx in range(grid_w):
            if covered[gy][gx]:
                continue

            best_shape = None
            for pw, ph, shape, size in HARDWARE_OAM_SHAPES:
                gw_tiles = pw // 8
                gh_tiles = ph // 8

                if gx + gw_tiles > grid_w or gy + gh_tiles > grid_h:
                    continue

                can_fit = True
                for cy in range(gy, gy + gh_tiles):
                    for cx in range(gx, gx + gw_tiles):
                        if covered[cy][cx]:
                            can_fit = False
                            break
                    if not can_fit:
                        break

                if can_fit:
                    best_shape = (pw, ph, shape, size, gw_tiles, gh_tiles)
                    break

            if best_shape is None:
                best_shape = (8, 8, 0, 0, 1, 1)

            pw, ph, shape, size, gw_tiles, gh_tiles = best_shape

            for cy in range(gy, gy + gh_tiles):
                for cx in range(gx, gx + gw_tiles):
                    covered[cy][cx] = True

            pieces.append({
                "x": offset_x + gx * 8,
                "y": offset_y + gy * 8,
                "width": pw,
                "height": ph,
                "shape": shape,
                "size": size,
                "tile_count": (pw * ph) // 64,
            })

    return pieces
```

---

## Tile Auto-Numbering and Allocation

In NCER/NCGR graphics pipelines, each OAM piece must be assigned a starting tile offset in graphics memory that respects 1D mapping mode alignment:

```python
def calculate_max_used_tile(oam_pieces: List[Dict[str, Any]]) -> int:
    """Calculates the highest NCGR tile index used by an existing list of OAM pieces."""
    max_tile = 0
    for piece in oam_pieces:
        tile_offset = piece.get("tile_offset", 0)
        tile_count = piece.get("tile_count", (piece.get("width", 8) * piece.get("height", 8)) // 64)
        end_tile = tile_offset + tile_count
        if end_tile > max_tile:
            max_tile = end_tile
    return max_tile

def calculate_next_free_tile(
    oam_pieces: List[Dict[str, Any]],
    mapping_mode: int = 0,
    alignment: int = 1,
) -> int:
    """Calculates the next available tile index, aligned to the 1D mapping mode step."""
    max_used = calculate_max_used_tile(oam_pieces)
    step = max(alignment, 1 << mapping_mode)
    return ((max_used + step - 1) // step) * step

def autonumber_oam_pieces(
    oam_pieces: List[Dict[str, Any]],
    start_tile: int = 0,
    mapping_mode: int = 0,
) -> List[Dict[str, Any]]:
    """Assigns sequential tile offsets to OAM pieces starting from start_tile.

    Calculates attr2_tile for NCER encoding:
        attr2_tile = (tile_offset // tile_multiplier) & 0x3FF
    """
    multiplier = 1 << mapping_mode
    current_tile = ((start_tile + multiplier - 1) // multiplier) * multiplier

    numbered_pieces = []
    for piece in oam_pieces:
        p = dict(piece)
        tile_count = p.get("tile_count", (p["width"] * p["height"]) // 64)
        p["tile_offset"] = current_tile
        p["attr2_tile"] = (current_tile // multiplier) & 0x3FF
        current_tile += tile_count
        current_tile = ((current_tile + multiplier - 1) // multiplier) * multiplier
        numbered_pieces.append(p)

    return numbered_pieces
```

---

## Visual Tools & Inclusive Pixel Coordinate Formula

When creating or inspecting sprite sheets and cell banks in image editors (Aseprite, Photoshop, GIMP) or canvas coordinate systems:
- Pixel coordinates are 0-indexed discrete points $[0, \text{dimension} - 1]$.
- **Inclusive Boundary Formula**: The bottom-right boundary pixel must always be calculated by subtracting 1:
  $$X_{\text{end}} = X_{\text{start}} + W - 1, \quad Y_{\text{end}} = Y_{\text{start}} + H - 1$$
- For example, a 32×16 OAM piece placed at $(0, 0)$ spans pixels $X \in [0, 31]$ and $Y \in [0, 15]$. Calculating $X_{\text{end}} = 0 + 32 = 32$ introduces a 1-pixel drift and overlaps the adjacent tile at $X = 32$.
- **Indexed Palette Mode (`P`)**: Always export and import sprite sheets in 8-bit Indexed Palette mode (`P`) matching the target NCLR palette. Editing sprites as 24-bit RGB or 32-bit RGBA without controlled quantization disrupts palette bank indexing.

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
