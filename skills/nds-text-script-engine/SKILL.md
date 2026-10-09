---
name: nds-text-script-engine
description: Use when extracting, parsing, translating, formatting, and reinserting NDS game text, dialogue scripts, bytecode control codes, and pointer tables
---

# NDS Text & Script Engine

## Overview
Comprehensive methods for reverse engineering, dumping, and reinserting game text, story dialogues, menu strings, script VM opcodes, and pointer tables across Nintendo DS titles.

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
Many NDS titles store script text in structured binary archives (`.msg`, `.dat`) identified by standard signatures such as `TEXT`.

#### Binary Header Layout (16 Bytes)
```text
[0x00..0x03]: 4-byte flags or CRC
[0x04..0x07]: Magic bytes b"TEXT"
[0x08]:       lang_count (uint8, e.g. 1 for single-language, or 4..6 for multi-language)
[0x09..0x0A]: entry_count (uint16)
[0x0B]:       reserved / unknown (uint8)
[0x0C..0x0F]: file_size (uint32)
[0x10..EndPtrs]: 32-bit pointer table (entry_count * lang_count * 4 bytes, uint32 little-endian)
```

#### Dynamic Delimiter Detection & Parsing
Strings are delimited dynamically by the first control byte pointed to by pointer 0 (typically `\x02` for dialogue strings or `\x00` for UI text):

```python
import struct
from typing import Any, Dict, List, Optional

def dump_msg(
    msg_bytes: bytes,
    font_type: str = "big",
    lang_labels: Optional[List[str]] = None,
) -> List[Dict[str, Any]]:
    """Parses binary .msg data into a list of structured entry dictionaries.

    Supports arbitrary lang_count (1, 2, 4, 6, etc.) without game-specific hardcoding.
    """
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

    # Dynamic delimiter: first byte of first string (typically 0x02 for dialogue, 0x00 for menus)
    delim = bytes([msg_bytes[ptrs[0]]]) if ptrs[0] < len(msg_bytes) else (b"\x02" if font_type == "big" else b"\x00")

    if not lang_labels:
        if lang_count == 1:
            lang_labels = ["source_text"]
        else:
            lang_labels = [f"lang_{l}" for l in range(lang_count)]

    entries: List[Dict[str, Any]] = []

    for i in range(entry_count):
        entry_dict: Dict[str, Any] = {"id": i}
        base_ptr_idx = i * lang_count

        for l_idx in range(lang_count):
            label = lang_labels[l_idx] if l_idx < len(lang_labels) else f"lang_{l_idx}"
            p = ptrs[base_ptr_idx + l_idx]

            next_ptr_idx = base_ptr_idx + l_idx + 1
            next_p = ptrs[next_ptr_idx] if next_ptr_idx < ptr_count else len(msg_bytes)

            end = msg_bytes.find(delim, p)
            if end == -1 or end > next_p:
                end = next_p - 1 if next_p > p else p

            raw_str = msg_bytes[p:end]
            entry_dict[label] = tokenize_bytes(raw_str, font_type=font_type)

        entry_dict["translation"] = ""
        entries.append(entry_dict)

    return entries
```

---

### 2. Multi-Byte Variant Character Encoding (`0xC0 + (glyph >> 6)`)
NDS games with proprietary font engines frequently map glyph indices into variable-length byte sequences:
- **Glyph indices $0..127$**: Stored as a single direct byte (`bytes([glyph_index])`).
- **Glyph indices $\ge 128$** (e.g., localized Cyrillic at $450..515$ or accented Latin): Encoded as a 2-byte sequence:
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

### 3. Script Opcodes, VM Events & Token Parsing
Dialogue VM engines combine script opcodes with text streams:

- **Multi-Byte Control Sequences**: e.g., `{EVENT_SYNC:XX}` (`0xC6 0x95 [XX] 0xC6 0x96`) or sound cues `{SOUND:XX}` (`0xC6 0x97 [XX] 0xC6 0x98`).
- **Dynamic Variable Tokens**: games substitute runtime party/actor names via dedicated bytecode tokens (e.g. `{PLAYER_NAME}`, `{TARGET}`, `{VARIABLE}`).
- **Interactive Prompts**: `{WAIT_KEY}` (`0xC5 0xBF`), line formatting, and page breaks (`{PAGE}`).

#### Tokenizer Implementation
```python
DEFAULT_SCRIPT_TOKENS: Dict[bytes, str] = {
    b"\xC5\xBF": "{WAIT_KEY}",
    b"\xC5\xB7": "{VAR_0}",
    b"\xC5\xB8": "{VAR_1}",
    b"\xC5\xB9": "{VAR_2}",
    b"\xC5\xBA": "{VAR_3}",
}

def tokenize_bytes(
    raw_bytes: bytes,
    font_type: str = "big",
    custom_tokens: Optional[Dict[bytes, str]] = None,
    glyph_map: Optional[Dict[int, str]] = None,
) -> str:
    """Converts raw binary bytecode into a human-readable text string with control tags.

    Supports custom game opcodes, variable-length glyphs, and dynamic delimiters.
    """
    token_map = dict(DEFAULT_SCRIPT_TOKENS)
    if custom_tokens:
        token_map.update(custom_tokens)

    res = []
    i = 0
    while i < len(raw_bytes):
        b = raw_bytes[i]

        # 5-byte sequences: e.g. {EVENT_SYNC:XX} and {SOUND:XX}
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

        # 2-byte script tokens & variable-length glyphs
        if b >= 0xC0 and i + 1 < len(raw_bytes):
            two = raw_bytes[i : i + 2]
            if two in token_map:
                res.append(token_map[two])
                i += 2
                continue

            b2 = raw_bytes[i + 1]
            if 0xC2 <= b <= 0xC8 and 0x80 <= b2 <= 0xBF:
                g_idx = bytes_to_glyph(b, b2)
                ch = glyph_map.get(g_idx) if glyph_map else None
                res.append(ch if ch else f"{{GLYPH:{g_idx}}}")
                i += 2
                continue

        # Single byte control tags & standard characters
        if b == 0:
            res.append("{NULL}" if font_type != "big" else "")
            i += 1
        elif b == 2 and font_type == "big":
            res.append("{PAGE}")
            i += 1
        else:
            ch = glyph_map.get(b) if glyph_map else None
            if ch:
                res.append(ch)
            elif 32 <= b <= 126:
                res.append(chr(b))
            else:
                res.append(f"{{TAG:{b:02X}}}")
            i += 1

    return "".join(res)
```

---

### 4. Safe Rebuilding & 32-Bit Pointer Recalculation
```python
def build_msg(
    entries: List[Dict[str, Any]],
    lang_count: int = 1,
    target_lang_key: str = "translation",
    fallback_lang_key: str = "source_text",
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
        trans = entry.get(target_lang_key) or entry.get(fallback_lang_key, "")
        raw_bytes = trans.encode("utf-8") + delim  # Or game-specific detokenizer

        for _ in range(lang_count):
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
    "id": 101,
    "context": "Dialogue_Elder_House_01",
    "source_text": "Welcome to our village, {PLAYER_NAME}!{PAGE}\nPlease take your rest.",
    "translation": "Добро пожаловать в нашу деревню, {PLAYER_NAME}!{PAGE}\nОтдохните с дороги.",
    "max_lines": 2,
    "max_width_px": 210
  }
]
```

## Common Mistakes & Edge Cases
- **Stripping unknown byte sequences**: Unknown bytes are often VM branch flags or sound triggers (`\xC6\x95..`). Keep them as `{TAG:XX}` or `{EVENT_SYNC:XX}` tokens.
- **Ignoring 32-bit pointer alignment**: Always pack pointer tables using `<I` (uint32) and pad containers to 4-byte boundaries.
- **Hardcoding null delimiters**: Check whether the target `.msg` uses `\x02` (common in RPG dialogue engines) or `\x00` (common in system menus) as the string terminator.
