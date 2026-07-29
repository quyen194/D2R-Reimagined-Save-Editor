# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this project is

A browser-based editor for **Diablo II: Resurrected** `.d2s` character save files,
targeting the **D2R Reimagined** mod
(https://github.com/D2R-Reimagined/d2r-reimagined-mod). It parses the binary
save format client-side, exposes the data through a tabbed UI, and writes
edits back out as a new `.d2s` file. Unofficial, maintained by GildyBoye, not
affiliated with the Reimagined team. See [README.md](README.md) for the full
user-facing feature list.

## Repository shape

This is **not** a typical Node/npm project — there is no `package.json`, no
build step, and no bundler.

- **[index.html](index.html)** — the entire application. ~20,000 lines of
  inline `<style>` CSS and vanilla JS in a single `<script>` block. This is
  where almost all real code changes happen.
- **[README.md](README.md)** — user-facing feature documentation.
- **[base/](base/)** — bundled fallback game data (see
  [base/CLAUDE.md](base/CLAUDE.md)). Static data, not application logic.

There is no test suite, linter config, or CI pipeline in this repo. Verify
changes by opening `index.html` directly in a browser (or via a static file
server) and exercising the relevant tab.

## How the app is organized (inside index.html)

Roughly top-to-bottom by line number:

1. **`<head>` / `<style>`** — D2-themed CSS (custom properties for the gold/red
   palette, gradients, panel/grid styling).
2. **Canvas/background effects** (`stepGrid`, `drawGrid`, `render`) — the
   ambient pixel animation on the loading/upload screens.
3. **Localization** — `stringTable`, `L()` / `Lfmt()` lookup helpers,
   `rebuildStringsForLang()`, `_updateStaticUIStrings()`. Strings are loaded
   from `base/strings/*.json` and the Reimagined string tables.
4. **Data loading & fallback** — `fetchTimed()`, `confirmFallback()`,
   `baseData`. On startup the app fetches Reimagined `.txt`/`.json` data from
   GitHub (`BASE_URL`, `STRINGS_BASE`, `REIMAGINED_HD_URL`,
   `REIMAGINED_SPRITE_URL` — all `raw.githubusercontent.com`) and falls back to
   the bundled copies under `base/` (`EDITOR_DATA_URL`) if the remote fetch
   fails. Startup is driven by `startBaseFileLoad()` on `DOMContentLoaded`.
5. **Lookup table builders** — `buildSkillNameMap`, `buildMagicAffixMaps`,
   `buildRareAffixMaps`, `buildSetNameMap`, `buildStatCostMap`,
   `buildMagicAffixModTables`, `buildUniqueItemModTable`,
   `buildSetItemModTable`, `buildSetBonusTable`, `buildPropDisplayEngine`,
   `buildPropGroups`, `buildStatByNameMap`, `buildStatRangesMap`,
   `buildStatDescriptors`, `buildItemTypeSets`. These turn the raw
   tab-separated `.txt` data into indexed JS objects used everywhere else
   (`STAT_META`, `MAGIC_PREFIX_MODS`/`MAGIC_SUFFIX_MODS`, `SET_BONUSES`,
   `PROP_ENGINE`, `PROP_GROUPS`, `STAT_BY_NAME`, `STAT_RANGES`,
   `STAT_DESCRIPTORS`, etc).
6. **Item/stat resolution** — `propCodeToStatEntries`, `buildStatListFromMods`,
   `propToDisplay`, `resolveRunewordMods`, `resolveItemMods`,
   `resolveMagicMods`, `resolveUniqueMods`, `resolveSetMods`, `resolveRareMods`
   — turn raw item property codes into human-readable stat lines for tooltips.
7. **Save parsing** — `parseD2S(buf)` (~line 8173) parses the `.d2s` binary
   format into `parsedChar`. Downstream editing works against this structure
   and a working byte array; a dirty-state flag and single-level undo
   snapshot track changes before they're written back to a new file.
8. **UI tabs / views** — Character, Skills, Inventory, Stash, Shared Stash,
   Quests, Waypoints, Mercenary, Grail, Vault. Inventory-related tabs gate
   behind a confirmation warning because malformed edits can corrupt a save.
9. **Auto-save** — File System Access API based, debounced (~2.5s), tracks
   file mtimes and disables itself after repeated write failures
   (`_autoSaveTimers`, `_fileHandles`, `_lastSeenMtime`, `_autoSaveWriting`,
   `_autoSaveFailures`, `_editGen`).
10. **Auth** (near end of file, ~line 20072) — Supabase client
    (`CLOUD_SUPABASE_URL`), loaded lazily from `cdn.jsdelivr.net`, with Discord
    as the OAuth provider. Used to gate features that need remote
    data/persistence — there's no standalone account system.

## Working in this codebase

- **Everything is one file.** Use `Grep`/line numbers to navigate
  `index.html`; there's no module system to `import` from. When editing,
  match the surrounding style — vanilla ES6+, no framework, no build/transpile
  step, so whatever you write runs as-is in the browser.
- **Data-driven, not hardcoded.** Item/skill/stat behavior comes from the
  Reimagined `.txt`/`.json` tables (fetched remotely, mirrored in `base/`).
  Prefer adding/adjusting lookups in the `build*Map`/`build*Table` functions
  over hardcoding values in UI code.
- **Two data sources must stay in sync.** Remote (`BASE_URL` /
  `raw.githubusercontent.com/D2R-Reimagined/...`) is the primary source; local
  `base/` is the offline fallback. If you change how a data file is parsed,
  check whether the local copy under `base/` has the same shape as what's
  fetched remotely.
- **Inventory/save editing is fragile by design.** The README explicitly
  warns that some item operations are unstable and can corrupt saves — be
  conservative and preserve existing validation/bounds-checking when touching
  `parseD2S`, item insertion, or the byte-array write path.
- **No package manager, no bundler.** Don't introduce `npm`/build tooling
  unless the user explicitly asks for it; the project's whole value
  proposition is "open `index.html` in a browser."
