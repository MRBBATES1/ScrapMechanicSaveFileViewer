# Scrap Mechanic Save File Explorer

A single self-contained HTML file that reads a Scrap Mechanic **survival save file** (`.db`) directly in your browser and lets you browse your creations, harvestables, scriptable objects, and the raw database — entirely offline, with nothing uploaded anywhere.

Open `Scrap_Mechanic_Save_File_Explorer.html` in any modern browser (double-click it, no server or install needed) and load your save.

## ⚠️ Compatibility: Scrap Mechanic 1.0+

This tool was built and tested against saves from **Scrap Mechanic 1.0 and later**. The 1.0 update changed the save format in several ways (see [Discoveries](#discoveries--things-that-werent-documented-anywhere) below), and this tool relies on those changes — most notably an SQLite `rtree` (spatial index) table introduced in 1.0. **It will very likely fail to load, or fail to parse correctly, on saves from before the 1.0 update.**

## Where to find your save

```
%AppData%\Axolot Games\Scrap Mechanic\User\User_<yourSteamID>\Save\Survival\<world>\
```

Close the game before copying the file so it isn't mid-write.

## Privacy

This is a static HTML file with no server component. The save file (and any `.lua` files you load) are read with your browser's local `FileReader` API and never leave your machine. SQLite parsing happens via [sql.js](https://github.com/sql-js/sql.js) (SQLite compiled to WebAssembly), running entirely in your browser tab.

---

## Tabs

### Overview

The landing page after loading a save. Shows:

- **Save file summary** — quick counts of creations, blocks, harvestables, scriptable objects, and total tables.
- **Worlds in this save** — see [Worlds](#worlds) below.
- **Unresolved UUIDs** — every part/harvestable UUID that couldn't be matched to a name, sorted by how often it appears, with a "Copy all as list" button. High-count entries are usually common terrain resources (see [Discoveries](#discoveries--things-that-werent-documented-anywhere)); low-count ones are often one-off or dynamically-generated content that may never resolve to a name.
- **Tables found in this save** — every table in the database with row counts and column types, including whether a table is a regular table or an SQLite virtual table (e.g. the `rtree` index).
- Any parsing warnings the tool ran into (a table missing, an unreadable column, etc.) show here rather than failing silently.

### Map

A pan/zoom canvas plotting creations and harvestables by their decoded world position.

- **Creations** are colored by a heuristic guess at whether they're something you built or a pre-placed world structure (a crashed ship, a warehouse interior, etc.) — see [Player creations vs. world structures](#player-creations-vs-world-structures-heuristic) below. Marker size scales with block count.
- **Harvestables** are shown in green.
- Use the **World** selector (top of the page) to isolate a single world — this matters a lot, since the Overworld and every warehouse/dungeon/underground instance you've visited each have their own separate coordinate space. Viewing "All worlds combined" will overlap unrelated locations on top of each other.
- Scroll to zoom, drag to pan, hover a marker for details, search by part name or UUID to filter what's shown.
- Grid lines and axis labels are shown for orientation; coordinates are approximate — cross-check anything that looks wrong against the Creations/Harvestables tabs.

### Creations

Every `RigidBody` in the save (i.e. everything with a position and a block/part list) — **this includes both your own builds and pre-placed world structures**, since the game stores them the same way. Columns:

- Position (X/Y/Z, approximate), block count, distinct part count.
- **Type** — a heuristic guess: *"Likely your creation"*, *"Likely a world structure (POI)"*, *"Mixed / unclear"*, or *"Unknown"* if no parts could be identified. Filter by this using the "Show" dropdown.
- **Parts ▾** — expand any row to see its full part breakdown *and* the category evidence behind the Type guess (see below).

Search filters by part name/UUID or by body ID.

#### Player creations vs. world structures (heuristic)

`survival_items.lua` groups every part under a comment marking its source asset file (`-- blocks.json`, `-- interactive.json`, `-- spaceship.json`, `-- warehouse.json`, etc.). Some of those categories are things available in the CraftBot building menu; others are clearly fixed scenery/structure sets for specific locations. The tool classifies a creation by which kind of category most of its blocks come from.

**This is a heuristic, not a certainty** — expand "Parts" on any row to see the exact category breakdown behind the guess. It reliably flags pure structures (e.g. the starting crashed ship comes back 100% `spaceship.json`, correctly classified as a world structure). It is **not** reliable at telling apart a real player build from a world structure when the player build legitimately uses parts from a "scenery" category — a build using some decorative parts can be misclassified. Treat it as a starting point for filtering, not ground truth.

### Harvestables

Every `Harvestable` row (crops, ore/stone deposits, trees, loot, etc.) with its decoded position and, where known, a resolved name. Search filters by UUID or name.

### Scriptable Objects

Lists every `ScriptableObject` row, grouped by its 16-byte type UUID (with a per-type occurrence count and which world(s) it appears in), plus the full row-by-row list.

**Important limitation:** the `ScriptableObject` binary record is fixed at 27 bytes and has **no room for position data** — this table can tell you *that* something exists and *which world* it's in, but not *where*. One type UUID is confirmed (it matches a hardcoded constant found directly in the game's `SurvivalGame.lua`); the rest are labeled as informed guesses based on occurrence patterns, not confirmed mappings — verify in-game where you can.

### Raw Table Browser

The reliable fallback and the tool used to reverse-engineer everything else: browse any table's raw rows and columns exactly as SQLite stores them (BLOB columns shown as hex), or run a custom SQL query. Use this to independently verify anything shown in the other tabs, or to explore tables this tool doesn't have a dedicated view for.

### Loading part names

Click **"Update part names (.lua)"** to load `.lua` files (e.g. `survival_items.lua`, `survival_harvestable.lua`, or any file with `= sm.uuid.new("...")` definitions) straight from your game install. A large baked-in dictionary (~1,700 names, pulled from a full scan of the game's `Survival\Scripts\game` folder) is already included, so this is mainly useful for catching anything new added by a future game update. Loaded names apply for the current session only.

---

## Worlds

Scrap Mechanic saves contain many separate "worlds" — the Overworld, plus one instance per Warehouse/Dungeon/Underground area you've entered — each with its **own local coordinate space**. Mixing them together on one map makes positions meaningless and inflates occurrence counts (a "common" harvestable might just be one appearing once in twenty different warehouses, not twenty times in one place).

The world list itself is decoded from a table called `GenericData` — see [Discoveries](#discoveries--things-that-werent-documented-anywhere) below for how that works. Where possible, a friendly name is derived from the world's own data (e.g. a `DungeonWorld` pointing at `growlab_01.world` is labeled "Growlab 01").

---

## Discoveries / things that weren't documented anywhere

Some of what this tool relies on wasn't available in any existing community documentation and had to be reverse-engineered from scratch against real save data:

- **World data lives inside a compressed, double-wrapped binary envelope.** The `GenericData` table (columns: `uid`, `channel`, `key`, `worldId`, `flags`, `data`) is a generic key-value store. World definitions are found by filtering on a fixed marker UUID, and the `data` blob is itself another full binary envelope — LZ4-compressed — that has to be decompressed and parsed again to get the world's seed, script filename, and classname. A minimal pure-JS LZ4 block decoder is embedded in this tool and was verified byte-for-byte against a known-correct test case before being trusted.
- **`ScriptableObject`'s binary format** (tag byte, version, ID, world ID, a 16-byte type UUID, world ID repeated) was decoded from scratch and confirmed correct by matching one type UUID to a hardcoded constant in the game's own `SurvivalGame.lua`.
- **Common "unknown" harvestable UUIDs are likely runtime-generated, not missing data.** A number of very frequently-occurring harvestable UUIDs (generic rocks, trees, terrain resources) don't appear as literal text anywhere in the game's scripts. The most likely explanation is that they're created with the engine's named/version-5 UUID generator (a deterministic hash of a name string) at world-generation time, rather than declared as a constant — meaning there's genuinely no file to find them in, not just one this tool missed.
- **`RigidBody`, `ChildShape`, and `Harvestable`'s binary layouts** build on prior community reverse-engineering but were independently re-verified against real save data from this project (including a byte-for-byte check against a specific UUID that also appears in the game's Lua files), since the 1.0 update was suspected to have changed some details from older documentation.

## Known limitations

- Position decoding for `RigidBody`/`Harvestable`/`Unit`-style tables is reverse-engineered, not from official documentation — always cross-check anything that looks wrong.
- The player-vs-world-structure classification on the Creations tab is a heuristic (see above) and can be wrong for creations that mix real building with decorative/scenery-category parts.
- `ScriptableObject` has no position data at all — this is a structural limitation of that table, not something a better offset would fix.
- A meaningful number of harvestable UUIDs may never resolve to a name (see the runtime-generated-UUID point above).
