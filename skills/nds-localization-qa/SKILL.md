---
name: nds-localization-qa
description: Use when validating text overflow limits, checking script control code syntax, calculating string pixel widths, or generating xdelta release patches for NDS translation projects
---

# NDS Localization Quality Assurance (QA) & Patching

## Overview
Procedures and automated tools for validating translated dialogue strings against game UI constraints, checking control code syntax, detecting text overflows, handling hyphenation carry, and creating distribution release patches (`.xdelta`).

## When to Use
- Validating that translated strings do not exceed line-count or pixel-width limits of game dialogue boxes.
- Verifying that all control code tags (`{COLOR:N}`, `{PAGE}`, `{HERO_NAME}`, `{GLYPH:N}`) are properly matched, balanced, and uncorrupted.
- Running automated QA checks on dumped translation files (JSON/PO).
- Applying VWF reflow with syllable hyphenation carry and compound word preservation.
- Generating clean `.xdelta` / `.bps` patches comparing modified ROMs with original clean dumps.

When NOT to use:
- Editing raw graphics or palettes (use `nds-font-graphics`).
- Reverse engineering bytecode logic (use `nds-text-script-engine`).

## Core Patterns

### 1. Proportional Pixel-Width Validator (Dynamic Variable Tokens Preserved)
Russian and other localized texts are typically 20–35% longer than English. Never insert translations into a ROM without verifying pixel dimensions against dialogue box bounds.

Crucially, **dynamic variable tokens (`{HERO_NAME}`, `{PLAYER_NAME}`, `{TARGET}`, `{VARIABLE}`) must NOT be stripped** as zero-width formatting codes — they represent runtime text on screen and must be credited with an estimated width (`DEFAULT_HERO_NAME_WIDTH_PX = 30`).

```python
import re
from typing import Dict, List, Set, Tuple

# Default generic dynamic variable tokens representing variable visual text in-game.
# Projects can override or extend this set in their project-specific configuration or AGENTS.md.
DEFAULT_VARIABLE_TOKENS: Set[str] = {
    "{HERO_NAME}", "{PLAYER_NAME}", "{TARGET}", "{VARIABLE}",
    "{ITEM_NAME}", "{LOCATION}", "{VALUE}", "{NUMBER}"
}

DEFAULT_CHAR_WIDTH_PX = 8
DEFAULT_HERO_NAME_WIDTH_PX = 30

# Control tag pattern: matches standard {TAG}, parameterized {OPCODE:N}, {GLYPH:N}, and hex tags {TAG:02}
CONTROL_TAG_PATTERN = re.compile(r"\{[A-Za-z0-9_]+(?::[A-Za-z0-9_]+)?\}")

def strip_control_tags(text: str, preserve_tokens: Set[str] = DEFAULT_VARIABLE_TOKENS) -> str:
    """Strips non-visual control tags (e.g. {COLOR:1}, {GLYPH:450}, {WAIT_KEY}, {TAG:02})
    while preserving visual variable tokens (e.g. {HERO_NAME}, {TARGET}).
    """
    def _replace(match: re.Match) -> str:
        token = match.group(0)
        if token in preserve_tokens:
            return token
        return ""  # Strip invisible control tags and script opcodes

    return CONTROL_TAG_PATTERN.sub(_replace, text)

def calculate_string_pixel_width(
    line: str,
    glyph_widths: Dict[str, int],
    variable_width_px: int = DEFAULT_HERO_NAME_WIDTH_PX,
    variable_tokens: Set[str] = DEFAULT_VARIABLE_TOKENS,
) -> int:
    """Calculates total pixel width of a single text line, preserving variable tokens."""
    clean_line = strip_control_tags(line, preserve_tokens=variable_tokens)
    if not clean_line:
        return 0

    escaped_tokens = [re.escape(tok) for tok in variable_tokens]
    token_pattern = re.compile("|".join(escaped_tokens)) if escaped_tokens else None

    total_width = 0
    last_end = 0

    if token_pattern:
        for match in token_pattern.finditer(clean_line):
            prefix = clean_line[last_end : match.start()]
            for ch in prefix:
                if ch in ('\r', '\n'):
                    continue
                total_width += glyph_widths.get(ch, DEFAULT_CHAR_WIDTH_PX)
            total_width += variable_width_px
            last_end = match.end()

    suffix = clean_line[last_end:]
    for ch in suffix:
        if ch in ('\r', '\n'):
            continue
        total_width += glyph_widths.get(ch, DEFAULT_CHAR_WIDTH_PX)

    return total_width
```

#### Project-Specific Token Overrides
Individual game projects should extend the variable set in their project-level configuration:
```python
# In game-specific pipeline scripts:
# GAME_VARIABLE_TOKENS = DEFAULT_VARIABLE_TOKENS | {"{PARTY_LEAD}", "{PET_NAME}", "{FACTION}"}
```

---

### 2. Syllable Hyphenation & Compound Word Preservation

In narrow Nintendo DS dialogue boxes, long localized words (especially in Russian) require hyphenation carry. The engine must distinguish soft wrapping hyphens from genuine hyphenated compound words.

```python
GENUINE_COMPOUND_PREFIXES = frozenset({"кое", "по", "из", "во"})
GENUINE_COMPOUND_SUFFIXES = frozenset({"то", "либо", "нибудь", "ка", "де", "таки"})

def _is_genuine_compound(left_part: str, right_part: str) -> bool:
    """Determines whether a hyphen between two word parts is a genuine compound word
    (e.g. Russian 'что-то', 'из-за', 'кое-как', 'кто-нибудь') rather than a soft wrap hyphen.
    """
    l = re.sub(r'^[^\w]+|[^\w]+$', '', left_part.lower())
    r = re.sub(r'^[^\w]+|[^\w]+$', '', right_part.lower())
    if not l or not r:
        return False
    if r in GENUINE_COMPOUND_SUFFIXES:
        return True
    if l in GENUINE_COMPOUND_PREFIXES:
        return True
    if l == "из" and r in ("за", "под"):
        return True
    if l == "друг" and r == "друга":
        return True
    return False

def collapse_hyphenated_breaks(text: str) -> str:
    """Collapses soft line breaks while preserving genuine hyphenated compound words.

    Example:
        'сло-\\nво' -> 'слово' (soft wrap hyphen collapsed)
        'что-\\nто' -> 'что-то' (genuine compound word hyphen preserved)
    """
    def _repl(match: re.Match) -> str:
        left = match.group(1)
        right = match.group(2)
        if _is_genuine_compound(left, right):
            return f"{left}-{right}"
        return f"{left}{right}"

    pattern = re.compile(r'(\b\w+)-[ \t]*\r?\n[ \t]*(\w+\b)', re.UNICODE)
    return pattern.sub(_repl, text)

def find_syllable_hyphen_points(word: str) -> List[int]:
    """Finds candidate syllable hyphenation points (character indices) in a word."""
    vowels = set("аеёиоуыэюяaeiouyАЕЁИОУЫЭЮЯAEIOUY")
    clean_word = re.sub(r'^[^\w]+|[^\w]+$', '', word)
    if len(clean_word) < 4:
        return []

    vowel_indices = [i for i, ch in enumerate(clean_word) if ch in vowels]
    if len(vowel_indices) < 2:
        return []  # Monosyllabic words cannot be split

    split_points = []
    for k in range(len(vowel_indices) - 1):
        v1 = vowel_indices[k]
        v2 = vowel_indices[k + 1]
        if v2 - v1 == 1:
            split = v1 + 1  # Hiatus
        elif v2 - v1 == 2:
            split = v1 + 1  # Single consonant between vowels
        else:
            split = v1 + 2  # Consonant cluster: split after first consonant

        if 2 <= split <= len(clean_word) - 2:
            split_points.append(split)

    return sorted(set(split_points))

def split_word_carry(
    word: str,
    available_width_px: int,
    glyph_widths: Dict[str, int],
    hyphen_char: str = "-",
    min_prefix_len: int = 2,
    min_suffix_len: int = 2,
) -> Tuple[str, str]:
    """Splits a long word to fit available pixel width on the current line.

    Attempts syllable-based hyphenation first. If no syllable cut fits,
    falls back to geometric character-by-character fitting.

    Returns:
        tuple[str, str]: (fitted_part_with_hyphen, carried_remainder)
        If the word cannot be split within available width, returns ("", word).
    """
    clean = word.rstrip(".,!?:;\"'»")
    trailing_punct = word[len(clean):]

    # 1. Syllable-based splitting
    syllable_points = find_syllable_hyphen_points(clean)
    best_split = None

    for pt in reversed(syllable_points):
        prefix = clean[:pt] + hyphen_char
        prefix_width = calculate_string_pixel_width(prefix, glyph_widths)
        if prefix_width <= available_width_px:
            best_split = pt
            break

    if best_split is not None:
        left = clean[:best_split] + hyphen_char
        right = clean[best_split:] + trailing_punct
        return left, right

    # 2. Geometric fallback
    for i in range(len(clean) - min_suffix_len, min_prefix_len - 1, -1):
        prefix = clean[:i] + hyphen_char
        prefix_width = calculate_string_pixel_width(prefix, glyph_widths)
        if prefix_width <= available_width_px:
            return prefix, clean[i:] + trailing_punct

    return "", word
```

---

### 3. Multi-Preset Architecture (`TextWindowPreset`) & Calibration
Different text contexts in NDS games have strictly different physical text boxes (dialogues, item descriptions, names, battle menus, config screens).

> **Note**: The table below provides recommended reference presets for standard 256x192 Nintendo DS displays. Each game project **MUST calibrate and define its own presets** in its project-specific `AGENTS.md`.

```python
from dataclasses import dataclass
import fnmatch

@dataclass(frozen=True)
class TextWindowPreset:
    """Configuration preset for dialogue, hint, or UI text windows on Nintendo DS."""
    name: str
    max_width_px: int
    max_lines: int
    reflow: bool
    font_type: str  # "big" or "small"
    patterns: Tuple[str, ...]

# Typical reference presets for standard 256x192 Nintendo DS displays:
WINDOW_PRESETS: Dict[str, TextWindowPreset] = {
    "dialogue_standard": TextWindowPreset("dialogue_standard", 220, 3, True, "big", ("dialogue*.json", "msg*.json", "story*.json")),
    "dialogue_wide": TextWindowPreset("dialogue_wide", 240, 3, True, "big", ("scenario*.json", "event*.json")),
    "tutorial_box": TextWindowPreset("tutorial_box", 220, 5, True, "big", ("tutorial*.json", "help*.json")),
    "item_desc": TextWindowPreset("item_desc", 190, 2, True, "small", ("item_desc*.json", "equip_desc*.json")),
    "item_name": TextWindowPreset("item_name", 90, 1, False, "big", ("item_name*.json", "skill_name*.json")),
    "monster_name": TextWindowPreset("monster_name", 85, 1, False, "big", ("enemy_name*.json", "monster*.json")),
    "map_location": TextWindowPreset("map_location", 140, 1, False, "big", ("location*.json", "map_name*.json")),
    "system_popup": TextWindowPreset("system_popup", 200, 2, False, "small", ("sys_msg*.json", "notice*.json")),
    "menu_option": TextWindowPreset("menu_option", 120, 1, False, "small", ("menu*.json", "config*.json")),
    "battle_message": TextWindowPreset("battle_message", 210, 2, False, "big", ("battle_msg*.json", "bt_*.json")),
    "credits": TextWindowPreset("credits", 240, 4, False, "big", ("credits*.json", "staff*.json")),
}

def resolve_entry_preset(file_stem: str, entry_id: int) -> TextWindowPreset:
    """Resolves the effective TextWindowPreset with fine-grained entry_id overrides."""
    base_preset = WINDOW_PRESETS["dialogue_standard"]
    for preset in WINDOW_PRESETS.values():
        if any(fnmatch.fnmatch(file_stem + ".json", pat) for pat in preset.patterns):
            base_preset = preset
            break
    return base_preset

def validate_translation_entry(
    entry: dict,
    file_stem: str,
    glyph_widths: Dict[str, int],
) -> List[str]:
    """Validates an entry from translation JSON against granular window presets."""
    errors = []
    entry_id = entry.get("id", 0)
    preset = resolve_entry_preset(file_stem, entry_id)
    text = entry.get("translation", "")

    pages = text.split("{PAGE}")
    for page_num, page in enumerate(pages, 1):
        lines = page.strip().split("\n")
        if len(lines) > preset.max_lines:
            errors.append(
                f"[{preset.name}] Entry {entry_id}, Page {page_num}: {len(lines)} lines "
                f"exceeds max allowed ({preset.max_lines})"
            )

        for line_num, line in enumerate(lines, 1):
            px = calculate_string_pixel_width(line, glyph_widths)
            if px > preset.max_width_px:
                errors.append(
                    f"[{preset.name}] Entry {entry_id}, Page {page_num}, Line {line_num}: "
                    f"width {px}px exceeds box {preset.max_width_px}px: '{line}'"
                )

    return errors
```

---

### 4. Automated VWF-Reflow (`wrap_text_block`) & Auto-Pagination

Combines hyphenation collapse, proportional wrapping, and optional auto-pagination:

```python
def wrap_text_block(
    text: str,
    glyph_widths: Dict[str, int],
    max_width_px: int = 220,
    max_lines: int = 3,
    auto_paginate: bool = False,
    variable_width_px: int = DEFAULT_HERO_NAME_WIDTH_PX,
    reflow: bool = True,
    enable_hyphenation: bool = True,
) -> Tuple[str, List[str]]:
    """Word-wraps dialogue text and optionally paginates across dialog boxes.

    Preserves {PAGE} tags, collapses ragged lines with hyphen awareness if reflow=True,
    applies proportional wrapping with syllable/geometric word carry, and paginates if auto_paginate=True.
    """
    warnings: List[str] = []
    pages = text.split("{PAGE}")
    output_pages: List[str] = []

    for page_idx, page in enumerate(pages):
        raw_page = page.strip("\r\n")
        if not raw_page:
            output_pages.append("")
            continue

        if reflow:
            # 1. Collapse hyphenated line breaks (preserving genuine compound words)
            dehyphenated = collapse_hyphenated_breaks(raw_page)
            # 2. Collapse single line breaks within page into spaces, preserving double breaks
            collapsed = re.sub(r'(?<!\n)\n(?!\n)', ' ', dehyphenated)
        else:
            collapsed = raw_page

        input_lines = collapsed.split("\n")
        wrapped_lines: List[str] = []

        for line in input_lines:
            words = line.split(" ")
            current_line = ""
            for word in words:
                if not word:
                    continue
                if not current_line:
                    # Check if single word exceeds line limit
                    if calculate_string_pixel_width(word, glyph_widths, variable_width_px) > max_width_px:
                        if enable_hyphenation:
                            part1, part2 = split_word_carry(word, max_width_px, glyph_widths)
                            if part1:
                                wrapped_lines.append(part1)
                                current_line = part2
                                continue
                    current_line = word
                else:
                    candidate = f"{current_line} {word}"
                    if calculate_string_pixel_width(candidate, glyph_widths, variable_width_px) <= max_width_px:
                        current_line = candidate
                    else:
                        # Try syllable hyphen carry if word is long
                        avail = max_width_px - calculate_string_pixel_width(f"{current_line} ", glyph_widths, variable_width_px)
                        if enable_hyphenation and avail >= 24:
                            part1, part2 = split_word_carry(word, avail, glyph_widths)
                            if part1:
                                wrapped_lines.append(f"{current_line} {part1}")
                                current_line = part2
                                continue
                        wrapped_lines.append(current_line)
                        current_line = word
            if current_line:
                wrapped_lines.append(current_line)

        # Handle line overflow & auto-pagination
        if auto_paginate and len(wrapped_lines) > max_lines:
            sub_pages = []
            for i in range(0, len(wrapped_lines), max_lines):
                sub_pages.append("\n".join(wrapped_lines[i : i + max_lines]))
            output_pages.append("{PAGE}\n".join(sub_pages))
        else:
            if len(wrapped_lines) > max_lines:
                warnings.append(
                    f"Page {page_idx + 1} has {len(wrapped_lines)} lines (max allowed: {max_lines})"
                )
            output_pages.append("\n".join(wrapped_lines))

    return "{PAGE}\n".join(output_pages), warnings
```

---

### 5. Control Tag Syntax Checker
Detects broken or unclosed tokens before bytecode assembly:

```python
def check_tag_integrity(text: str) -> List[str]:
    errors = []
    # Check for unclosed '{' or unopened '}'
    if text.count("{") != text.count("}"):
        errors.append("Mismatched curly bracket count for control tags")

    valid_tags = re.compile(r'^\{([A-Za-z0-9_]+)(?::[A-Za-z0-9_]+)?\}$')
    tags = re.findall(r'\{[^{}]*\}', text)
    for tag in tags:
        if not valid_tags.match(tag):
            errors.append(f"Invalid or corrupted control tag format: '{tag}'")

    return errors
```

---

### 6. Generating Release Patches (`xdelta3`)
For sharing the translation without distributing copyrighted ROM data:

```python
import subprocess
import os

def create_xdelta_patch(original_nds: str, modified_nds: str, output_patch: str) -> bool:
    """Generates an xdelta3 patch file comparing clean and modified ROMs."""
    cmd = ["xdelta3", "-e", "-s", original_nds, modified_nds, output_patch]
    res = subprocess.run(cmd, capture_output=True, text=True)
    if res.returncode == 0:
        print(f"Patch created successfully: {output_patch} ({os.path.getsize(output_patch)} bytes)")
        return True
    else:
        print(f"Error creating patch: {res.stderr}")
        return False
```

## Quality Checklist
- [ ] All strings decoded and re-encoded in round-trip test without byte loss.
- [ ] No unescaped control tags or missing opcode parameters.
- [ ] All dialogue lines within pixel limits of calibrated text boxes.
- [ ] Dialogue choices (Yes/No, battle choices) align correctly with cursors.
- [ ] Game tested past initial copy protection / Anti-Piracy triggers (AP bypass verified).
- [ ] `.xdelta` patch tested against clean CRC32/SHA-1 of the original clean ROM dump.
