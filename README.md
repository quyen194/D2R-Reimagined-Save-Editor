# D2R Hero Editor — Reimagined

Browser-based editor for Diablo II: Resurrected `.d2s` save files using Reimagined mod data.

Maintained by Gildyboye, not the Reimagined team. 
Unofficial.

Reimagined Mod Links: https://www.nexusmods.com/diablo2resurrected/mods/503 or https://github.com/D2R-Reimagined/d2r-reimagined-mod

---

## Overview

This tool loads a `.d2s` save file, parses its binary structure, and exposes editable fields through a tabbed interface.  
All edits are applied to an in-memory copy and can be written back to disk as a new file.

The editor relies on external data from the D2R Reimagined repository (item data, strings, templates, etc.), with automatic fallback to bundled local data if remote fetch fails.

---

## Features

### Character Editing
- Name, class, level, and core stats
- Difficulty and progression flags
- Hardcore / Expansion / death flags
- Direct stat input with validation

### Skills
- Full skill tree layout per class
- Editable skill levels with limits enforced
- Unspent skill points control
- Reads and writes directly to save data

### Inventory & Equipment
- Grid-based inventory and stash system
- Paperdoll equipment slots
- Item rarity coloring and metadata display
- Tooltip inspection with modifiers and flags
- Item insertion via categorized modal (weapons, armor, misc, uniques, sets, runewords)

### Stash & Shared Stash
- Multi-tab stash support
- Specialty tab splitting (gems, runes, materials)
- Item search across tabs
- Grid rendering consistent with in-game layout
- Tab arrangement: reorder shared stash tabs by drag & drop, staged behind Apply / Revert
- Sort Items: per-category sorting (unique/crafted gear, rare gear, set items, rings & amulets,
  charms, jewels) into tabs you pick
- Auto Sort: one click, sorts every category into a fixed stash layout — crafted & unique gear
  (tabs 1-5), rings & amulets by rarity (6-8), charms (9-11), jewels (12-13), set items (14-19),
  rare gear (20+). Categories that overflow borrow empty tabs and push the rest of the layout
  along; ones that come up short keep their tabs blank, and a category you own nothing of is
  still handed its tabs once everything else is placed. Each step commits on its own and rolls
  itself back if it fails
- Auto Sort settings (⚙ next to the button): the layout, the sort order within each
  category, and the run's behaviour come from [`base/autosort.json`](base/autosort.json) and
  can be overridden per browser. Edit the JSON in place, switch between named profiles, or
  Import / Export it as a file. See [Auto Sort settings](#auto-sort-settings) below

### Vault System
- Load and save `.d2i` vault files
- Search items across all pages
- Move items between sources and vault
- Multi-page and quad-view layouts

### Grail Tracking
- Visual item completion grid
- Displays found vs missing items
- Page-based navigation

### Quests
- Per-act quest tracking
- Complete / clear all controls
- Difficulty-specific state editing
- Reward handling tied to completion flags

### Waypoints
- Toggle waypoint unlocks per difficulty
- Organized by act

### Mercenary
- View and edit mercenary data
- Equipment and stats where applicable

---

## Authentication

This tool uses Supabase for authentication, with Discord as the OAuth provider.

### How it works
- Users sign in via Discord OAuth
- Supabase manages authentication, sessions, and tokens
- A valid session may be required for features that rely on remote data or persistence

### Notes
- No standalone account system is implemented
- Discord is used only as an identity provider
- Authentication is handled entirely through Supabase

---

## File Handling

### Input
- Accepts `.d2s` character save files
- Drag-and-drop or file picker supported

### Output
- Writes modified data to a new file
- Original file is not overwritten by default

### Additional Files
- `.d2i` vault files (optional)
- Shared stash and grail data handled separately

---

## Data Loading

On startup, the editor performs:

1. Fetch base game data (items, stats, templates)
2. Fetch string tables for localization
3. Load editor-specific JSON data
4. Fallback to local copies if remote fetch fails

Failure to load required data prevents the editor from initializing.

---

## Auto Sort settings

The shared stash Auto Sort reads its layout from [`base/autosort.json`](base/autosort.json).
Three layers apply, each overriding the one before it:

1. A built-in copy of the same layout, so Auto Sort still works if nothing loads
2. `base/autosort.json` — the default this repo ships
3. Your own override, saved in this browser via the ⚙ button next to Auto Sort

The ⚙ dialog edits the effective config as JSON. **Apply** saves it to this browser only,
**Reset** drops it and goes back to the bundled default, and **Export** / **Import** move it
around as a file. The dialog never sorts anything — close it and press Auto Sort when ready.

A field that isn't valid falls back to the layer below and is reported; a broken config can
make the layout ugly, but it can't make the sort do something unsafe.

### Global options

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `version` | number | `1` | Schema version |
| `startTab` | integer ≥ 0 | `0` | Leave the first N tabs completely alone — not sorted, not used as a source, not borrowed. Your junk/mule tabs |
| `compactEmptyTabsFirst` | boolean | `true` | Slide empty tabs to the end before sorting. Turning it off strands empty tabs mid-stash, which makes "not enough room" failures far more likely |
| `padding` | `roundRobin` \| `inOrder` \| `off` | `roundRobin` | How blank tabs are handed to categories that came up short of `minTabs`. `roundRobin` shares a shortage out; `inOrder` fills the top of the list first; `off` hands out nothing |
| `addEmptyPageWhen` | `"never"` \| `"always"` \| 0–100 | `30` | Give a category one extra empty tab when the **last tab of its run** is more than this percent full. `"always"` is the same as `0`. Independent of `padding` |
| `continueOnFailure` | boolean | `false` | Keep going past a category that fails instead of stopping the run there |
| `skipConfirm` | boolean | `false` | Skip the confirmation dialog |
| `gearGroups` | array of bucket keys | mode default | Pull equipment buckets to the front of the slot order. Anything left out keeps its built-in place behind them |
| `profiles` | object | two examples | Named sets of settings — see below |
| `activeProfile` | string \| `null` | `null` | Which profile is in force |

### Per-category options

The order of the `categories` array **is** the tab order — swap two entries to swap two
categories.

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `modeId` | string | required | One of the sort modes listed below |
| `minTabs` | integer ≥ 1 | `1` | Tabs the category reserves before it has to borrow more |
| `enabled` | boolean | `true` | `false` skips the category entirely — its items are left untouched and it claims no tabs |
| `addEmptyPageWhen` | as above | inherits the global | Per-category override |
| `sortKeys` | array of key names | mode default | The order items sort in. Prefix a name with `-` to reverse it |
| `fill` | object of stream → direction | mode default | Which way each of the mode's streams fills a tab: `row`, `colL` or `colR` |

Valid `modeId` values: `set`, `uniqueGear`, `rareGear`, `uniqueCharm`, `magicCharm`,
`uniqueJewel`, `rareMagicJewel`, `uniqueRingAmu`, `rareRingAmu`, `magicRingAmu`.

### Sort keys

`sortKeys` replaces a category's ordering with your own. Keys compare in the order listed;
a `-` prefix reverses that key.

| Key | Meaning |
| --- | --- |
| `handTier` | Armor, then one-handed weapons, then two-handed |
| `gearGroup` | Equipment bucket (helm, body armor, belt, …), per `gearGroups` |
| `quality` | Crafted before unique |
| `reqLevel` | Required level to equip |
| `ilvl` | Item level |
| `height` | Grid height — grand, large, then small charms |
| `baseName` | Base item name |

The shipped defaults, written out:

```
uniqueGear      handTier, gearGroup, quality, reqLevel, -ilvl, baseName
rareGear        handTier, gearGroup, reqLevel, -ilvl, baseName
uniqueCharm     -height, quality, reqLevel, -ilvl, baseName
magicCharm      -height, reqLevel, -ilvl, baseName
uniqueJewel     quality, reqLevel, -ilvl, baseName
rareMagicJewel  reqLevel, -ilvl, baseName
uniqueRingAmu   quality, reqLevel, -ilvl, baseName
rareRingAmu     reqLevel, -ilvl, baseName
magicRingAmu    reqLevel, -ilvl, baseName
```

`set` takes no `sortKeys`: its order comes from keeping whole sets together, not from a
per-item key.

`sortKeys` and `fill` apply to the **Sort Items** modal as well as to Auto Sort — one
category, one definition of how it sorts.

**`gearGroups` interacts with `sortKeys`.** `handTier` runs ahead of `gearGroup`, so
reordering buckets only has an effect *within* a hand tier. To put weapons before armor,
drop `handTier` from that category's `sortKeys` as well.

Bucket keys: `helm`, `tors`, `belt`, `glov`, `boot`, `shld`, `axe`, `swor`, `knif`,
`blun`, `scep`, `wand`, `staf`, `orb`, `spea`, `pole`, `jave`, `bow`, `bowq`, `xbow`,
`xboq`, `h2h`, `weap`.

### Profiles

A profile is a partial config applied last, after every other layer — so whichever layer
names it, its settings win. It cannot select another profile or redefine the set of them,
so there is no chain to follow. The gear dialog shows a dropdown whenever any profile
exists; picking one edits `activeProfile` in the box, and Apply commits it like any other
change. The box always shows your settings *without* the profile applied, so switching
away restores them.

```json
"activeProfile": "compact",
"profiles": {
  "compact":  { "addEmptyPageWhen": "never",  "padding": "off" },
  "spacious": { "addEmptyPageWhen": "always", "padding": "roundRobin" }
}
```

### Example

```json
{
  "version": 1,
  "startTab": 0,
  "compactEmptyTabsFirst": true,
  "padding": "roundRobin",
  "addEmptyPageWhen": 30,
  "continueOnFailure": false,
  "skipConfirm": false,

  "activeProfile": null,
  "profiles": {
    "compact":  { "addEmptyPageWhen": "never",  "padding": "off" },
    "spacious": { "addEmptyPageWhen": "always", "padding": "roundRobin" }
  },

  "categories": [
    { "modeId": "uniqueGear",     "minTabs": 5, "enabled": true },
    { "modeId": "uniqueRingAmu",  "minTabs": 1, "enabled": true },
    { "modeId": "rareRingAmu",    "minTabs": 1, "enabled": true },
    { "modeId": "magicRingAmu",   "minTabs": 1, "enabled": true },
    { "modeId": "uniqueCharm",    "minTabs": 1, "enabled": true },
    { "modeId": "magicCharm",     "minTabs": 2, "enabled": true },
    { "modeId": "uniqueJewel",    "minTabs": 1, "enabled": true },
    { "modeId": "rareMagicJewel", "minTabs": 1, "enabled": true },
    { "modeId": "set",            "minTabs": 6, "enabled": true, "addEmptyPageWhen": "always" },
    { "modeId": "rareGear",       "minTabs": 1, "enabled": true, "addEmptyPageWhen": "never" }
  ]
}
```

---

## Localization

- Supports multiple languages via string tables
- Dynamically updates UI labels and item names
- Cleans raw strings (removes formatting and color codes)

---

## UI Structure

### Views
- Loading View – Fetches required data
- Error View – Displays critical load failures
- Upload View – File selection interface
- Editor View – Main editing interface

### Tabs
- Character
- Skills
- Inventory
- Stash
- Shared Stash
- Quests
- Waypoints
- Mercenary
- Grail
- Vault

Some tabs (inventory-related) require confirmation due to potential data corruption risks.

---

## Editing Model

- Save file is parsed into structured data (`parsedChar`)
- Changes are applied to a working byte array
- Dirty state tracks unsaved changes
- Undo snapshot supports single-level rollback

---

## Auto-Save

- Optional auto-save using the File System Access API
- Writes are debounced (~2.5 seconds after changes)
- Tracks file modification timestamps
- Detects external file changes
- Disables after repeated write failures

---

## Item Editing

### Capabilities
- Insert new items via selection modal
- Adjust item level (ilvl) within valid bounds
- Handle special item types (runes, misc, runewords)
- Supports multi-slot item placement

### Limitations
- Some item operations are unstable
- Inventory editing may produce invalid states if misused
- Warnings are shown before enabling editing

---

## Search

- Search across inventory, stash, and vault
- Supports:
  - Item name
  - Type code
- Results highlight and navigate to item location

---

## Visual Systems

- Canvas-based item rendering
- Grid-based layout for inventory and stash
- Background visual effects (pixel-based animation)

---

## Dependencies

### Remote Data Sources
- D2R Reimagined repository (GitHub)
- Localization string files
- Editor-specific JSON datasets

### Browser Requirements
- Modern Chromium-based browser recommended
- File System Access API required for auto-save

---

## Safety Notes

- Always back up original save files before editing
- Inventory modifications can corrupt saves if used incorrectly
- Not all item structures are fully validated

---

## Credits

- Developed by GildyBoye
- Uses data from the D2R Reimagined project

---
