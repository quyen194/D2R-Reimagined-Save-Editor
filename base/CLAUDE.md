# CLAUDE.md — base/

Local fallback copy of **D2R Reimagined** game data. See the root
[../CLAUDE.md](../CLAUDE.md) for how this fits into the app.

## Purpose

The editor normally fetches this data live from
`raw.githubusercontent.com/D2R-Reimagined/d2r-reimagined-mod` (and, for the
editor-specific bits, from `raw.githubusercontent.com/GildyBoye/D2R-Reimagined-Save-Editor/main/base/`).
Everything under this folder is a **mirror of that remote data**, used only
when the remote fetch fails (`confirmFallback()` / `EDITOR_DATA_URL` in
[../index.html](../index.html)). Don't treat these files as hand-authored
config — they're generated/exported from the Reimagined mod and should mirror
its schema exactly.

## Layout

- **`*.txt`** (57 files) — tab-separated Diablo II data tables (`armor.txt`,
  `itemstatcost.txt`, `skills.txt`, `magicprefix.txt`, `setitems.txt`,
  `properties.txt`, etc). Parsed via `parseTSV()` in `index.html`. Column
  names matter — the `build*Map`/`build*Table` functions in `index.html`
  index rows by specific header names (case varies, e.g. `row['*Id']` vs
  `row['Id']`), so preserve headers exactly if a file is regenerated.
- **`*.json`** (20 files) — structured data: `items.json`, `sets.json`,
  `categories.json`, `item_meta.json`, `globaldatahd.json`, etc.
- **`basechars/*.d2s`** — template save files, one per class (`BaseBarb`,
  `BaseSorc`, ...), used as a starting point (e.g. for new-character
  creation/reset flows).
- **`strings/*.json`** — localization tables (`ui.json`, `skills.json`,
  `monsters.json`, `item-names.json`, `item-modifiers.json`,
  `item-nameaffixes.json`, `item-runes.json`, `levels.json`,
  `mercenaries.json`) plus `GildLocalization.json` (editor-specific strings
  not present in the base mod).
- **`images/*.jpg`** — quest/level artwork shown in the Quests tab.
- **`sprites/`** (`armor/`, `weapon/`, `misc/`) — `.lowend.sprite` files used
  for item icon rendering; mirrors the mod's `data/hd/global/ui/items/`
  layout under `REIMAGINED_SPRITE_URL`.

## Working here

- Treat this directory as **data, not code**. If behavior looks wrong, first
  check whether it's a parsing issue in `index.html` vs. stale/incorrect data
  here.
- If you update a file here to match a newer Reimagined mod release, keep the
  exact column/key names the `index.html` parsers expect — grep `index.html`
  for the filename (e.g. `itemstatcost.txt`) to find every place that reads
  it before changing its shape.
- This is a fallback path, exercised only when the network fetch fails or is
  disabled — test changes here by simulating that (e.g. blocking the remote
  host) rather than assuming the happy path exercises it.
