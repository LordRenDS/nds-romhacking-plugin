---
name: nds-text-script-engine
description: Use when extracting, parsing, translating, formatting, and reinserting NDS game text, dialogue scripts, bytecode control codes, and pointer tables
---

# NDS Text & Script Engine

## Overview
Comprehensive methods for reverse engineering, dumping, and reinserting game text, story dialogues, menu strings, script VM opcodes, and pointer tables in Nintendo DS ROMs.

## When to Use
- Locating and extracting dialogue and UI text from binary archives (`.bin`, `.dat`, `.msg`).
- Parsing and rebuilding pointer tables (16-bit / 32-bit, absolute ROM, relative, or RAM offsets).
- Handling game-specific character encodings (Shift-JIS, UTF-16LE, custom TBL, DTE/MTE dictionaries).
- Parsing script VM control codes (color tags, line breaks, page transitions, character name variables, choice branches).
- Formatting dumps into clean, editable JSON or PO formats for translation and reinserting translated text without data corruption.

When NOT to use:
- Editing font glyph textures or palette data (use `nds-font-graphics`).
- Patching ARM9 assembly routines (use `nds-asm-patching`).

## Core Patterns

### 1. Multilingual Containers with 32-Bit Offsets & Dynamic Delimiter
Many NDS titles (e.g. *Chrono Trigger DS*) store script text in multilingual binary archives (`.msg`) identified by the `TEXT` magic signature.

#### Binary Header Layout (16 Bytes)
```text
[0x00..0x03]: 4-byte unknown/flags
[0x04..0x07]: Magic bytes b"TEXT"
[0x08]:       lang_count (uint8, e.g. 4 for multi-language, or 1)
[0x09..0x0A]: entry_count (uint16)
[0x0B]:       unknown (uint8)
[0x0C..0x0F]: file_size (uint32)
[0x10..EndPtrs]: 32-bit pointer table (entry_count * lang_count * 4 bytes, uint32 little-endian)
```

#### Dynamic Delimiter Detection & Parsing
Strings are delimited not necessarily by null (`\x00`), but dynamically by the first control byte pointed to by pointer 0 (typically `\x02` for dialogue strings or `\x00` for UI text):

```python
import struct
from typing import Any, Dict, List, Optional

def dump_msg(msg_bytes: bytes, font_type: str = "big") -> List[Dict[str, Any]]:
    """Parses binary .msg data into a list of structured entry dictionaries."""
    if len(msg_bytes) < 16:
        raise ValueError(f"Invalid .msg binary: length {len(msg_bytes)} < 16")

    magic = msg_bytes[4:8]
    if magic != b"TEXT":
        raise ValueError(f"Invalid .msg magic: expected b'TEXT', got {magic!r}")

    lang_count, entry_count, _unk, file_size = struct.unpack_from("<BHBI", msg_bytes, 8)
    if entry_count == 0:
        return []

    ptr_count = entry_count * lang_count
    ptrs = [
        struct.unpack_from("<I", msg_bytes, 16 + j * 4)[0]
        for j in range(ptr_count)
    ]

    # Dynamic delimiter: first byte of first string (usually 0x02 for dialogue, 0x00 for menus)
    delim = bytes([msg_bytes[ptrs[0]]]) if ptrs[0] < len(msg_bytes) else (b"\x02" if font_type == "big" else b"\x00")

    entries: List[Dict[str, Any]] = []

    for i in range(entry_count):
        if lang_count >= 4:
            p0, p1, p2, p3 = ptrs[i * lang_count : i * lang_count + 4]

            # English text at p1
            end1 = msg_bytes.find(delim, p1)
            if end1 == -1 or (p2 > p1 and end1 > p2):
                end1 = p2 - 1 if p2 > p1 else p1
            en_text = tokenize_bytes(msg_bytes[p1:end1], font_type=font_type)

            # French text at p3
            next_p0 = ptrs[(i + 1) * lang_count] if (i + 1) < entry_count else len(msg_bytes)
            end3 = msg_bytes.find(delim, p3)
            if end3 == -1 or (next_p0 > p3 and end3 > next_p0):
                end3 = next_p0 - 1 if next_p0 > p3 else p3
            fr_text = tokenize_bytes(msg_bytes[p3:end3], font_type=font_type)

            entries.append({
                "id": i,
                "original_en": en_text,
                "original_fr": fr_text,
                "translation": "",
            })
        else:
            p = ptrs[i * lang_count]
            next_p = ptrs[(i + 1) * lang_count] if (i + 1) < entry_count else len(msg_bytes)
            end = msg_bytes.find(delim, p)
            if end == -1 or end > next_p:
                end = next_p - 1 if next_p > p else p
            text = tokenize_bytes(msg_bytes[p:end], font_type=font_type)
            entries.append({
                "id": i,
                "original_en": text,
                "original_fr": "",
                "translation": "",
            })

    return entries
```

---

### 2. Multi-Byte Variant Character Encoding (`0xC0 + (glyph >> 6)`)
NDS games with proprietary font engines map glyph indices into variable-length byte sequences:
- **Glyph indices $0..127$**: Stored as a single direct byte (`bytes([glyph_index])`).
- **Glyph indices $\ge 128$** (such as Cyrillic at 450..515 or accented Latin): Encoded as a 2-byte sequence where:
  $$\text{prefix} = 0\text{xC}0 + (\text{glyph\_index} \gg 6)$$
  $$\text{suffix} = 0\text{x}80 + (\text{glyph\_index} \ \& \ 0\text{x}3\text{F})$$

```python
def glyph_to_bytes(glyph_index: int) -> bytes:
    """Encodes a glyph index into variable-length bytes (1 or 2 bytes)."""
    if glyph_index < 128:
        return bytes([glyph_index])
    prefix = 0xC0 + (glyph_index >> 6)
    suffix = 0x80 + (glyph_index & 0x3F)
    return bytes([prefix, suffix])

def bytes_to_glyph(prefix: int, suffix: int) -> int:
    """Decodes a 2-byte sequence into its glyph index."""
    return ((prefix - 0xC0) << 6) | (suffix & 0x3F)
```

---

### 3. 5-Byte Event Opcodes & 2-Byte Script Tokens
Dialogue VM engines combine script opcodes with text streams:

#### 5-Byte Sequences
- `{EVENT_SYNC:XX}`: `0xC6 0x95 [XX] 0xC6 0x96` (synchronizes VM threads with map events).
- `{SOUND:XX}`: `0xC6 0x97 [XX] 0xC6 0x98` (triggers sound effects/music cues).

#### 2-Byte Sequences
- Dynamic Hero Tokens:
  - `0xC5 0xB7` $\rightarrow$ `{CRONO}`
  - `0xC5 0xB8` $\rightarrow$ `{MARLE}`
  - `0xC5 0xB9` $\rightarrow$ `{LUCCA}`
  - `0xC5 0xBA` $\rightarrow$ `{ROBO}`
  - `0xC5 0xBB` $\rightarrow$ `{FROG}`
  - `0xC5 0xBC` $\rightarrow$ `{AYLA}`
  - `0xC5 0xBD` $\rightarrow$ `{MAGUS}`
  - `0xC5 0xBE` $\rightarrow$ `{EPOCH}`
- Dialogue Control:
  - `0xC5 0xBF` $\rightarrow$ `{WAIT_KEY}` (pauses until user button press)

#### Tokenizer & Detokenizer Implementation
```python
PARTY_CONTROL_TOKENS = {
    b"\xC5\xB7": "{CRONO}",
    b"\xC5\xB8": "{MARLE}",
    b"\xC5\xB9": "{LUCCA}",
    b"\xC5\xBA": "{ROBO}",
    b"\xC5\xBB": "{FROG}",
    b"\xC5\xBC": "{AYLA}",
    b"\xC5\xBD": "{MAGUS}",
    b"\xC5\xBE": "{EPOCH}",
}
REV_PARTY_CONTROL_TOKENS = {v: k for k, v in PARTY_CONTROL_TOKENS.items()}

def tokenize_bytes(raw_bytes: bytes, font_type: str = "big") -> str:
    """Converts raw binary bytecode into a human-readable text string with control tags."""
    res = []
    i = 0
    while i < len(raw_bytes):
        b = raw_bytes[i]

        # 5-byte sequences: {EVENT_SYNC:XX} and {SOUND:XX}
        if b == 0xC6 and i + 4 < len(raw_bytes):
            b2, b3, b4, b5 = raw_bytes[i+1 : i+5]
            if b2 == 0x95 and b4 == 0xC6 and b5 == 0x96:
                res.append(f"{{EVENT_SYNC:{b3:02X}}}")
                i += 5
                continue
            elif b2 == 0x97 and b4 == 0xC6 and b5 == 0x98:
                res.append(f"{{SOUND:{b3:02X}}}")
                i += 5
                continue

        # 2-byte sequences: party tokens, wait_key, or multi-byte glyphs
        if b >= 0xC0 and i + 1 < len(raw_bytes):
            two = raw_bytes[i : i + 2]
            if two in PARTY_CONTROL_TOKENS:
                res.append(PARTY_CONTROL_TOKENS[two])
                i += 2
                continue
            if two == b"\xC5\xBF":
                res.append("{WAIT_KEY}")
                i += 2
                continue
            b2 = raw_bytes[i + 1]
            if 0xC2 <= b <= 0xC8 and 0x80 <= b2 <= 0xBF:
                g_idx = bytes_to_glyph(b, b2)
                res.append(GLYPH_TO_CHAR.get(g_idx, f"{{GLYPH:{g_idx}}}"))
                i += 2
                continue

        if b == 0:
            res.append("e" if font_type == "big" else "{NULL}")
            i += 1
        elif b == 2 and font_type == "big":
            res.append("{PAGE}")
            i += 1
        else:
            res.append(GLYPH_TO_CHAR.get(b, f"{{TAG:{b:02X}}}"))
            i += 1

    return "".join(res)
```

---

### 4. Safe Rebuilding & 32-Bit Pointer Recalculation
```python
def build_msg(
    entries: List[Dict[str, Any]],
    lang_count: int = 4,
    font_type: str = "big",
) -> bytes:
    """Serializes translated entries back into a binary .msg file with 32-bit pointers."""
    entry_count = len(entries)
    ptr_count = entry_count * lang_count
    header_size = 16 + (ptr_count * 4)

    # Pad header to 4-byte boundary
    header_size = (header_size + 3) & ~3
    current_offset = header_size

    ptrs: List[int] = []
    text_payload = bytearray()
    delim = b"\x02" if font_type == "big" else b"\x00"

    for entry in entries:
        trans = entry.get("translation") or entry.get("original_en", "")
        raw_bytes = detokenize_string(trans, font_type=font_type) + delim

        for lang in range(lang_count):
            ptrs.append(current_offset)

        text_payload += raw_bytes
        current_offset += len(raw_bytes)

    total_size = header_size + len(text_payload)
    header = bytearray(16)
    header[4:8] = b"TEXT"
    struct.pack_into("<BHBI", header, 8, lang_count, entry_count, 0, total_size)

    ptr_bytes = bytearray()
    for ptr in ptrs:
        ptr_bytes += struct.pack("<I", ptr)

    # Pad pointers section
    while len(header) + len(ptr_bytes) < header_size:
        ptr_bytes += b"\x00"

    return bytes(header + ptr_bytes + text_payload)
```

---

## Translation File Formats (JSON / PO)
Store dumped strings in structured JSON with metadata:
```json
[
  {
    "id": 142,
    "context": "Dialogue_Millennial_Fair_Marle_01",
    "original_en": "Watch where you're going!{PAGE}\nAre you all right?",
    "original_fr": "Fais attention!{PAGE}\nTu vas bien?",
    "translation": "Смотри, куда идешь!{PAGE}\nТы в порядке?",
    "max_lines": 2,
    "max_width_px": 210
  }
]
```

## Common Mistakes & Edge Cases
- **Stripping unknown byte sequences**: Unknown bytes are often VM branch flags or sound triggers (`\xC6\x95..`). Keep them as `{TAG:XX}` or `{EVENT_SYNC:XX}` tokens.
- **Ignoring 32-bit pointer alignment**: Always pack pointer tables using `<I` (uint32) and pad containers to 4-byte boundaries.
- **Hardcoding null delimiters**: Check whether the target `.msg` uses `\x02` (common in RPG dialogue engines) or `\x00` (common in system menus) as the string terminator.

