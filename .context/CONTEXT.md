# arcitect

## what this is

a standalone local-first companion app for the extraction shooter ARC Raiders. tracks runs, loot, loadouts, and session history. map drawing/annotation is the primary feature. local-first alternative to ARCTracker and Overwolf companions — users own their data, no accounts, no cloud, no telemetry.

## architecture

| layer | choice |
|-------|--------|
| app shell | tauri v2 (rust backend + web frontend) |
| frontend | react (imperative canvas integration) |
| canvas | fabric.js — full object model (select, drag, resize, group, freehand, text, serialization) |
| UI components | shadcn/ui + tailwind CSS |
| theme | dark/tactical — muted colors, sharp edges, ARC Raiders aesthetic |
| storage (data) | tauri-plugin-sql (SQLite) |
| storage (files) | tauri-plugin-fs + $APPDATA |
| game data | RaidTheory/arcraiders-data (MIT) — items, missions, skills, traders |
| run tracking | manual entry (no automated source exists) |
| window chrome | frameless + custom title bar |
| targets | macOS + windows |

## active decisions

### decided

- **app framework: tauri v2** — evaluated against electron, pyside6/qt, flutter, avalonia, compose multiplatform
- **frontend: react** — developer comfort, ecosystem size
- **canvas: fabric.js** — best object model for annotation UX (selection, grouping, freehand, serialization)
- **storage: sqlite + filesystem** — structured data in sqlite, file assets on disk
- **game data: community JSON repos** — RaidTheory/arcraiders-data as primary source
- **UI: shadcn/ui + tailwind, dark tactical theme**

### open

- data model schema (runs, loadouts, annotations)
- map image sourcing and licensing
- feature spec for v0.1

## patterns

[none yet — patterns emerge from practice]

## research

extensive framework research conducted (6 documents in .thinking/research/):
- tauri v2 canvas maturity and webview analysis
- flutter desktop drawing evaluation
- native framework options (qt, avalonia, compose)
- qt styling/aesthetics feasibility
- tauri-specific failure investigation (caido migration debunked for canvas)
- full stack comparison (frontend + canvas + storage + UI)
- ARC Raiders data ecosystem mapping

---

*context established: 2026-02-19*
*last updated: 2026-02-19*
