# arcitect — data model

*established: 2026-02-19*

## sqlite schema

```sql
-- game maps metadata
-- populated from bundled game data, read-only to user
CREATE TABLE maps (
    id TEXT PRIMARY KEY,           -- e.g., 'dam-battlegrounds'
    name TEXT NOT NULL,            -- e.g., 'Dam Battlegrounds'
    image_filename TEXT NOT NULL,  -- e.g., 'dam-battlegrounds.png'
    width INTEGER NOT NULL,
    height INTEGER NOT NULL,
    unlock_requirement TEXT,       -- e.g., '8 rounds completed'
    round_duration TEXT,           -- e.g., '30 min'
    sort_order INTEGER NOT NULL DEFAULT 0
);

-- pre-annotated POI layers per map (from game data)
-- read-only to user, toggleable visibility
CREATE TABLE map_layers (
    id TEXT PRIMARY KEY,
    map_id TEXT NOT NULL REFERENCES maps(id),
    name TEXT NOT NULL,             -- e.g., 'Extraction Points'
    layer_type TEXT NOT NULL,       -- 'zone' | 'field' | 'token'
    icon TEXT,                      -- icon identifier for token-type layers
    color TEXT,                     -- hex color for visual styling
    visible_default INTEGER NOT NULL DEFAULT 1,
    sort_order INTEGER NOT NULL DEFAULT 0
);

-- individual POIs within a layer
CREATE TABLE map_pois (
    id TEXT PRIMARY KEY,
    layer_id TEXT NOT NULL REFERENCES map_layers(id),
    map_id TEXT NOT NULL REFERENCES maps(id),
    label TEXT,
    x REAL NOT NULL,
    y REAL NOT NULL,
    -- for zones: polygon/rect bounds as JSON array of points
    -- for fields: polygon bounds
    -- for tokens: single point (x, y above)
    bounds TEXT,                    -- JSON, nullable
    metadata TEXT                   -- JSON, extra game data
);

-- user-logged runs
CREATE TABLE runs (
    id TEXT PRIMARY KEY,            -- UUID
    map_id TEXT NOT NULL REFERENCES maps(id),
    started_at TEXT,                -- ISO 8601, nullable
    duration_minutes INTEGER,       -- nullable
    value_earned REAL,              -- nullable (US-5)
    outcome TEXT NOT NULL DEFAULT 'unknown',  -- 'extracted' | 'died' | 'abandoned' | 'unknown'
    notes TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- freehand path drawings per run (US-3)
CREATE TABLE run_paths (
    id TEXT PRIMARY KEY,            -- UUID
    run_id TEXT NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    map_id TEXT NOT NULL REFERENCES maps(id),
    color TEXT NOT NULL DEFAULT '#ff0000',
    stroke_width REAL NOT NULL DEFAULT 2.0,
    path_data TEXT NOT NULL,        -- fabric.js serialized Path JSON
    visible INTEGER NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- user custom annotations, not tied to a run (US-4)
CREATE TABLE annotations (
    id TEXT PRIMARY KEY,            -- UUID
    map_id TEXT NOT NULL REFERENCES maps(id),
    annotation_type TEXT NOT NULL,  -- 'text' | 'rect' | 'circle' | 'arrow' | 'freehand' | 'image'
    fabric_data TEXT NOT NULL,      -- fabric.js toJSON() for this object
    visible INTEGER NOT NULL DEFAULT 1,
    opacity REAL NOT NULL DEFAULT 1.0,
    locked INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- user-placed tokens on maps (US-6)
CREATE TABLE tokens (
    id TEXT PRIMARY KEY,            -- UUID
    map_id TEXT NOT NULL REFERENCES maps(id),
    run_id TEXT REFERENCES runs(id) ON DELETE SET NULL,  -- nullable, can be general
    token_type TEXT NOT NULL,       -- 'loot' | 'enemy' | 'death' | 'pvp' | 'custom'
    subtype TEXT,                   -- e.g., specific item name, enemy type
    icon TEXT,                      -- icon override, nullable
    x REAL NOT NULL,
    y REAL NOT NULL,
    notes TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- user settings
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

-- sharing format version tracking
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY,
    applied_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- indexes for common queries
CREATE INDEX idx_run_paths_run ON run_paths(run_id);
CREATE INDEX idx_run_paths_map ON run_paths(map_id);
CREATE INDEX idx_annotations_map ON annotations(map_id);
CREATE INDEX idx_tokens_map ON tokens(map_id);
CREATE INDEX idx_tokens_run ON tokens(run_id);
CREATE INDEX idx_tokens_type ON tokens(map_id, token_type);
CREATE INDEX idx_runs_map ON runs(map_id);
CREATE INDEX idx_map_pois_layer ON map_pois(layer_id);
CREATE INDEX idx_map_pois_map ON map_pois(map_id);
```

## fabric.js layer architecture

fabric.js doesn't have native layers like konva. layers are managed via object metadata and visibility filtering.

```
canvas object stack (bottom to top):
├── map image (locked, non-selectable, non-evented)
├── POI layer objects (grouped by layer_id, toggleable)
│   ├── extraction points
│   ├── loot zones
│   ├── enemy spawns
│   └── ...
├── run path objects (grouped by run_id, toggleable)
│   ├── run 2026-02-19 path (blue)
│   ├── run 2026-02-18 path (red)
│   └── ...
├── custom annotations (individually selectable)
│   ├── text labels
│   ├── shapes
│   └── freehand drawings
└── custom tokens (selectable, filterable by type)
    ├── loot markers
    ├── death markers
    ├── enemy markers
    └── pvp markers
```

each object carries metadata:
```javascript
{
  layerGroup: 'poi' | 'runPath' | 'annotation' | 'token',
  layerId: string,   // for POI layers
  runId: string,     // for run paths and run-linked tokens
  tokenType: string, // for tokens
  arcitect_id: string // maps to sqlite row id
}
```

show/hide layer:
```javascript
canvas.getObjects()
  .filter(obj => obj.layerGroup === 'poi' && obj.layerId === layerId)
  .forEach(obj => obj.visible = !obj.visible);
canvas.requestRenderAll();
```

## sharing format (US-7)

```json
{
  "version": 1,
  "app": "arcitect",
  "map_id": "dam-battlegrounds",
  "exported_at": "2026-02-19T10:30:00Z",
  "includes": {
    "annotations": true,
    "tokens": true,
    "paths": true
  },
  "annotations": [
    {
      "id": "uuid",
      "type": "text",
      "fabric": { /* fabric.js toObject() */ }
    }
  ],
  "tokens": [
    {
      "id": "uuid",
      "type": "loot",
      "subtype": "Blue Keycard",
      "x": 1234.5,
      "y": 678.9,
      "notes": "spawns in basement"
    }
  ],
  "paths": [
    {
      "id": "uuid",
      "color": "#ff4444",
      "stroke_width": 2.0,
      "fabric": { /* fabric.js toObject() for path */ }
    }
  ]
}
```

clipboard: JSON → gzip → base64
file: JSON → gzip → `.arcmap` extension

## state location

| platform | path |
|----------|------|
| macOS | `~/Library/Application Support/arcitect/` |
| windows | `%APPDATA%/arcitect/` |

contents:
```
arcitect/
├── arcitect.db          # sqlite database
├── maps/                # base map images
│   ├── dam-battlegrounds.png
│   ├── buried-city.png
│   └── ...
└── exports/             # user exports (optional cache)
```
