# arcitect — requirements specification v0.1

*established: 2026-02-19*

## product positioning

local-first alternative to ARCTracker and Overwolf companions for ARC Raiders. emphasis on data ownership, map annotation, and offline operation. target user: players who want to own their run data and annotate maps without browser/cloud dependencies.

---

## user stories

### US-1: map switching
> as a user, I would like to be able to open and swap between all existing ARC Maps.

**acceptance criteria:**
- sidebar or tab bar shows all available maps (currently 6 + practice range)
- switching maps preserves state on the previous map (annotations, visibility toggles)
- map loads with correct base image, centered and fitted to viewport
- map list updates when new maps are added to game data

**maps (as of feb 2026):**
1. Dam Battlegrounds (starter)
2. Buried City
3. The Spaceport
4. The Blue Gate
5. Stella Montis
6. Practice Range
7. Riven Tides (coming april 2026)

**data source:** map metadata from RaidTheory/arcraiders-data, images TBD (community composites or FModel extraction)

---

### US-2: pre-annotated toggleable layers
> as a user, I would like each map to come pre-annotated with toggleable zones, fields, tokens which correspond to known entities and regions of interest.

**acceptance criteria:**
- each map ships with default POI layers derived from community game data
- layers are toggled via a layer panel (checkbox per layer)
- layer types include at minimum:
  - extraction points
  - loot zones / crate locations
  - enemy spawn regions
  - trader locations
  - key/locked door locations
  - points of interest (named areas)
- toggling a layer shows/hides all its markers without affecting user annotations
- layer data is read-only (users cannot edit pre-annotated data)
- layer data can be updated when game data is refreshed

**implementation notes:**
- POI data stored as JSON in sqlite, loaded as fabric.js objects with `selectable: false, evented: false`
- each layer is a logical group with a `layerId` metadata field
- show/hide via `object.visible = true/false` filtered by layerId
- visual style: semi-transparent fills for zones, icon markers for points, dashed outlines for fields

---

### US-3: run path drawings
> as a user, I would like to be able to draw in unique colors my own path mappings from a run across the map, or see drawings from previous runs.

**acceptance criteria:**
- user can enter "draw mode" and freehand draw a path on the map
- color picker allows selecting the path color before drawing
- each path is associated with a specific run (via run selector or "current run")
- previous run paths are visible as overlays on the map
- user can toggle visibility of individual run paths (or all run paths)
- paths are labeled or color-coded to distinguish between runs
- paths persist across sessions (saved to storage)

**implementation notes:**
- fabric.js `isDrawingMode = true` with `PencilBrush`
- path color set via `canvas.freeDrawingBrush.color`
- completed paths stored as serialized fabric.js Path objects in sqlite
- run selector dropdown determines which run_id the path attaches to

---

### US-4: custom annotations
> as a user, I would like to embed custom annotations to the map in general, and be able to adjust their visibility and sizing levels.

**acceptance criteria:**
- user can add text, shapes (rectangles, circles, arrows), and freehand drawings to any map
- annotations are NOT tied to a specific run — they persist on the map generally
- each annotation has individual visibility toggle
- annotations can be resized via selection handles
- annotations can be moved, rotated, and deleted
- annotation opacity/visibility can be adjusted (per-annotation or bulk)
- annotations persist across sessions

**implementation notes:**
- fabric.js objects with `type: 'annotation'` metadata
- full object model: select, drag, resize, rotate, group, delete
- stored as serialized fabric.js JSON in sqlite (per-map annotation set)
- visibility managed via object.visible + opacity slider in properties panel

---

### US-5: run value tracking
> as a user, I would like the amount of value I earned on a run to be recorded in some way if I provide it.

**acceptance criteria:**
- when logging a run, user can optionally enter the value earned (currency/loot value)
- value is displayed in run history alongside map, date, duration, outcome
- value field is optional — runs can be logged without it
- aggregate stats available: total value, average per run, value by map
- value is a numeric field (supports decimals for currency)

**implementation notes:**
- `runs.value_earned REAL` column, nullable
- run entry form: map (required), outcome (required), value (optional), duration (optional), notes (optional)
- stats computed as sqlite aggregates

---

### US-6: custom tokens (loot, enemies, deaths, PVP)
> as a user, I would like to be able to annotate the map with my own tokens representing where I have found types of loot, enemies, died, or had PVP encounters.

**acceptance criteria:**
- toolbar provides token stamps: loot, enemy, death, PVP encounter (+ custom)
- user clicks/taps a location on the map to place a token
- tokens are visually distinct by type (unique icon + color per type)
- tokens can optionally be associated with a run
- tokens can have a text note attached (e.g., "blue keycard found here")
- tokens can be filtered by type (show only deaths, show only loot, etc.)
- loot tokens support subtype (specific item name from game data)
- tokens persist across sessions
- heatmap-like density view would be nice-to-have (not v0.1)

**implementation notes:**
- predefined token types with icons: loot (gold), enemy (red), death (skull), pvp (crossed swords), custom (user-defined)
- placed as fabric.js Image or Group objects with metadata (tokenType, subtype, runId, notes)
- filter panel toggles visibility by tokenType
- subtype dropdown populated from game data items list

---

### US-7: map sharing
> as a user, I would like to be able to share these maps via a clipboard or file with other users.
> as a developer, I would like to have a rough semantic encoding / zip of the annotations, or failing that a small enough file/jsonblob that can be passed around with state.

**acceptance criteria:**
- "export" button produces a shareable representation of the current map's annotations
- export includes: custom annotations, tokens, run paths, and their metadata
- export does NOT include: base map image or pre-annotated POI data (recipient already has these)
- two export modes:
  1. **clipboard** — compact base64-encoded string that can be pasted in Discord, etc.
  2. **file** — `.arcmap` file (gzipped JSON) for file sharing
- "import" button accepts both clipboard paste and file selection
- imported annotations merge onto the recipient's map (additive, not destructive)
- sharing format is versioned for forward compatibility

**implementation notes:**
- export format:
  ```json
  {
    "version": 1,
    "app": "arcitect",
    "map_id": "dam-battlegrounds",
    "exported_at": "2026-02-19T10:30:00Z",
    "annotations": [...],  // fabric.js serialized objects
    "tokens": [...],       // token objects with positions + metadata
    "paths": [...]         // run paths with color + points
  }
  ```
- clipboard: `JSON.stringify()` → gzip → base64 → clipboard
- file: same JSON → gzip → save as `.arcmap`
- import: detect format (base64 string or file), decompress, validate version, merge objects onto canvas
- size estimate: 50-200 annotations ≈ 10-50KB JSON → ~2-10KB gzipped → ~3-14KB base64

---

### US-8: state storage
> as a developer, I would like state to be stored outside of the easy purview of users.

**acceptance criteria:**
- all persistent state (runs, annotations, tokens, settings) stored in sqlite database
- database file located in platform-specific app data directory:
  - macOS: `~/Library/Application Support/arcitect/arcitect.db`
  - windows: `%APPDATA%/arcitect/arcitect.db`
- map images stored alongside DB in the same directory
- no state stored in Documents, Desktop, or any user-visible location
- no plaintext config files in obvious locations

**implementation notes:**
- tauri's `$APPDATA` resolves to these platform paths automatically
- `tauri-plugin-sql` for sqlite operations
- `tauri-plugin-fs` scoped to `$APPDATA/**` for file operations
- backup/export functionality (nice-to-have) for users who want to manually back up their data

---

### US-9: intuitive map navigation
> as a user, I would like navigating the map to feel intuitive.

**acceptance criteria:**
- mouse wheel zooms in/out centered on cursor position
- click-drag on empty canvas pans the map
- trackpad pinch-to-zoom supported (macOS)
- keyboard shortcuts: +/- for zoom, arrow keys for pan, Home/0 to reset view
- zoom has min/max bounds (don't zoom so far you lose the map, don't zoom so close it's pixels)
- minimap in corner showing current viewport position (nice-to-have, not v0.1)
- double-click to zoom-to-fit
- smooth zoom animation (not jarring snaps)
- pan/zoom does NOT interfere with drawing mode — clear mode separation

**implementation notes:**
- fabric.js viewport transform: `canvas.zoomToPoint()`, `canvas.relativePan()`
- mouse wheel handler with smooth zoom interpolation
- mode manager: navigate mode (default) vs draw mode vs token placement mode
- cursor changes to indicate current mode (grab cursor for navigate, crosshair for draw, etc.)
