---
name: nds-localization-qa
description: Use when validating text overflow limits, checking script control code syntax, calculating string pixel widths, or generating xdelta release patches for NDS translation projects
---

# NDS Localization Quality Assurance (QA) & Patching

## Overview
Procedures and automated tools for validating translated dialogue strings against game UI constraints, checking control code syntax, detecting text overflows, and creating distribution release patches (`.xdelta`).

## When to Use
- Validating that translated strings do not exceed line-count or pixel-width limits of game dialogue boxes.
- Verifying that all control code tags (`{COLOR}`, `{PAGE}`, `{HERO_NAME}`) are properly matched, balanced, and uncorrupted.
- Running automated QA checks on dumped translation files (JSON/PO).
- Generating clean `.xdelta` / `.bps` patches comparing modified ROMs with original clean dumps.

When NOT to use:
- Editing raw graphics or palettes (use `nds-font-graphics`).
- Reverse engineering bytecode logic (use `nds-text-script-engine`).

## Core Patterns

### 1. Proportional Pixel-Width Validator (Dynamic Hero Tokens Preserved)
Russian/localized text is typically 20–35% longer than English. Never insert translations into a ROM without verifying pixel dimensions against dialogue box bounds.
Crucially, **dynamic hero tokens (`{CRONO}`, `{MARLE}`, `{HERO_NAME}`, etc.) must NOT be stripped** during pixel width calculation — they represent variable visual text on screen and must be credited with an estimated width (`DEFAULT_HERO_NAME_WIDTH_PX = 30`):

```python
import re
from typing import Dict, List, Tuple

# Dynamic hero tokens that represent variable visual text in-game
HERO_TOKENS = frozenset({
    "{CRONO}", "{MARLE}", "{LUCCA}", "{ROBO}",
    "{FROG}", "{AYLA}", "{MAGUS}", "{EPOCH}", "{HERO_NAME}"
})

HERO_PATTERN = re.compile(r"\{(?:CRONO|MARLE|LUCCA|ROBO|FROG|AYLA|MAGUS|EPOCH|HERO_NAME)\}")
TAG_REGEX = re.compile(r"\{[^{}]+\}")

DEFAULT_CHAR_WIDTH_PX = 8
DEFAULT_HERO_NAME_WIDTH_PX = 30

def strip_control_tags(text: str) -> str:
    """Strips non-visual control tags while preserving visual hero tokens."""
    def _replace(match: re.Match) -> str:
        token = match.group(0)
        if token in HERO_TOKENS:
            return token
        return ""  # Strip invisible control tags like {WAIT_KEY}, {COLOR:1}, etc.

    return TAG_REGEX.sub(_replace, text)

def calculate_string_pixel_width(
    line: str,
    glyph_widths: Dict[str, int],
    hero_name_width_px: int = DEFAULT_HERO_NAME_WIDTH_PX,
) -> int:
    """Calculates total pixel width of a single text line, preserving hero tokens."""
    clean_line = strip_control_tags(line)
    if not clean_line:
        return 0

    total_width = 0
    last_end = 0

    for match in HERO_PATTERN.finditer(clean_line):
        prefix = clean_line[last_end : match.start()]
        for ch in prefix:
            if ch in ('\r', '\n'):
                continue
            total_width += glyph_widths.get(ch, DEFAULT_CHAR_WIDTH_PX)
        total_width += hero_name_width_px
        last_end = match.end()

    suffix = clean_line[last_end:]
    for ch in suffix:
        if ch in ('\r', '\n'):
            continue
        total_width += glyph_widths.get(ch, DEFAULT_CHAR_WIDTH_PX)

    return total_width
```

---

### 2. Multi-Preset Architecture (`TextWindowPreset`) & Granular Entry Overrides
Different text contexts in NDS games have strictly different physical text boxes (dialogues, item descriptions, names, battle menus, config screens). Validating all strings against a single generic limit causes overflows or overly aggressive truncation:

```python
from dataclasses import dataclass
import fnmatch

@dataclass(frozen=True)
class TextWindowPreset:
    """Configuration preset for dialogue, hint, or UI text windows."""
    name: str
    max_width_px: int
    max_lines: int
    reflow: bool
    font_type: str  # "big" or "small"
    patterns: Tuple[str, ...]

WINDOW_PRESETS: Dict[str, TextWindowPreset] = {
    "dialogue": TextWindowPreset("dialogue", 230, 3, True, "big", ("msg*.json", "cmes*.json")),
    "tutorial": TextWindowPreset("tutorial", 230, 6, True, "big", ("tutorial.json", "start.json")),
    "encyclopedia": TextWindowPreset("encyclopedia", 215, 6, True, "big", ("player.json", "ex_mon*.json")),
    "item_desc": TextWindowPreset("item_desc", 195, 2, True, "big", ("item.json",)),
    "item_name": TextWindowPreset("item_name", 90, 1, False, "big", ("item_name.json",)),
    "tech_name": TextWindowPreset("tech_name", 85, 1, False, "big", ("tech_name.json",)),
    "tech_desc": TextWindowPreset("tech_desc", 200, 2, True, "big", ("tech.json",)),
    "monster_name": TextWindowPreset("monster_name", 80, 1, False, "big", ("mon_name.json",)),
    "map_location": TextWindowPreset("map_location", 140, 1, False, "big", ("map_loc.json",)),
    "bgm_name": TextWindowPreset("bgm_name", 160, 1, False, "big", ("bgm_title.json",)),
    "credits": TextWindowPreset("credits", 240, 4, False, "big", ("credits.json",)),
    "battle": TextWindowPreset("battle", 220, 2, False, "big", ("bt_*.json",)),
    "menu": TextWindowPreset("menu", 180, 1, False, "small", ("menu_*.json",)),
    "system_big": TextWindowPreset("system_big", 210, 2, False, "big", ("sys_big*.json",)),
    "small_system": TextWindowPreset("small_system", 160, 2, False, "small", ("system_*.json",)),
}

def resolve_entry_preset(file_stem: str, entry_id: int) -> TextWindowPreset:
    """Resolves the effective TextWindowPreset with fine-grained entry_id overrides."""
    # 1. Match base preset by file pattern
    base_preset = WINDOW_PRESETS["dialogue"]
    for preset in WINDOW_PRESETS.values():
        if any(fnmatch.fnmatch(file_stem + ".json", pat) for pat in preset.patterns):
            base_preset = preset
            break

    # 2. Fine-grained overrides for specific entry_ids (e.g. 2-column config, popup alerts)
    if file_stem == "system_config":
        if 5 <= entry_id <= 12:
            return TextWindowPreset("config_option", 85, 1, False, "small", ())
    elif file_stem == "menu_battle":
        if entry_id == 0:
            return TextWindowPreset("battle_prompt", 140, 1, False, "big", ())
        elif 10 <= entry_id <= 25:
            return TextWindowPreset("battle_command", 75, 1, False, "small", ())

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

### 3. Automated VWF-Reflow (`wrap_text_block`) & Auto-Pagination
When translating into Russian, manual line breaks from English source text cause ugly ragged boxes or overflows. The automated VWF-reflow engine collapses ragged lines within a page, performs proportional word-wrapping, and splits excess lines across dialog pages using `{PAGE}`:

```python
def wrap_text_block(
    text: str,
    glyph_widths: Dict[str, int],
    max_width_px: int = 230,
    max_lines: int = 3,
    auto_paginate: bool = False,
    hero_name_width_px: int = DEFAULT_HERO_NAME_WIDTH_PX,
    reflow: bool = True,
) -> Tuple[str, List[str]]:
    """Word-wraps dialogue text and optionally paginates across dialog boxes.

    Preserves {PAGE} tags, collapses ragged lines into spaces if reflow=True,
    applies proportional wrapping, and paginates if auto_paginate=True.
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
            # Collapse single line breaks within page into spaces, preserving double breaks
            collapsed = re.sub(r'(?<!\n)\n(?!\n)', ' ', raw_page)
        else:
            collapsed = raw_page

        input_lines = collapsed.split("\n")
        wrapped_lines: List[str] = []

        for line in input_lines:
            words = line.split(" ")
            current_line = ""
            for word in words:
                if not current_line:
                    current_line = word
                else:
                    candidate = f"{current_line} {word}"
                    if calculate_string_pixel_width(candidate, glyph_widths, hero_name_width_px) <= max_width_px:
                        current_line = candidate
                    else:
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


### 4. Control Tag Syntax Checker
Detects broken or unclosed tokens before bytecode assembly:

```python
def check_tag_integrity(text: str) -> List[str]:
    errors = []
    # Check for unclosed '{' or unopened '}'
    if text.count("{") != text.count("}"):
        errors.append("Mismatched curly bracket count for control tags")
        
    valid_tags = re.compile(r'^\{([A-Z_]+)(?::\d+)?\}$')
    tags = re.findall(r'\{[^{}]*\}', text)
    for tag in tags:
        if not valid_tags.match(tag):
            errors.append(f"Invalid or corrupted control tag format: '{tag}'")
            
    return errors
```

### 5. Generating Release Patches (`xdelta3`)
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
- [ ] No unescaped control tags or missing parameters.
- [ ] All dialogue lines within pixel limits of text boxes.
- [ ] Dialogue choices (Yes/No, battle choices) align correctly with cursors.
- [ ] Game tested past the Millennial Fair telepod (AP bypass verified).
- [ ] `.xdelta` patch tested against clean CRC32/SHA-1 of the original ROM dump.
