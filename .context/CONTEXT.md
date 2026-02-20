# arcitect

## what this is

a standalone local application for interacting with and saving records of runs done in the extraction shooter ARC Raiders. tracks loot, loadouts, run outcomes, and session history. map drawing/annotation is the primary feature.

## architecture

- **app shell:** tauri v2 (rust backend + web frontend in system webview)
- **targets:** macOS + windows (no linux)
- **frontend:** TBD (svelte, react, or solid) + canvas library (fabric.js or konva)
- **backend:** rust (tauri commands for data/storage/filesystem)
- **storage:** TBD (sqlite, json, embedded db)
- **canvas:** HTML5 Canvas2D via fabric.js or konva — map annotation, drawing, overlays

## active decisions

### decided

- **app framework: tauri v2** — lightweight, full web canvas ecosystem, rust backend. evaluated against electron (too heavy), pyside6/qt (aesthetics burden), flutter (drawing ecosystem gaps), avalonia, compose multiplatform. canvas2d works correctly in WKWebView (macOS) — caido's migration was about CSS/UI fragmentation, not canvas issues.

### open

- frontend framework (svelte? react? solid?)
- canvas library (fabric.js vs konva)
- storage approach
- game data integration
- UI design system

## patterns

[none yet — patterns emerge from practice]

## research

research conducted on framework selection:
- tauri v2 maturity + canvas ecosystem
- flutter desktop drawing capabilities
- native framework options (qt, avalonia, compose)
- qt styling/aesthetics feasibility
- tauri-specific failure modes (caido investigation)

all artifacts in `.thinking/research/`

---

*context established: 2026-02-19*
*last updated: 2026-02-19*
