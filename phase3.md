# Phase 3 — Custom Engine Integration

## Scope

Custom engine integration — write our own engine or combine existing engines to replace the WebKitGTK fallback. Core request interception. Tab tiling (horizontal/vertical/grid, nested splits). Navigation state model. Search vs URL detection. Keyboard navigation/focus model. Tab groups.

---

## Component Breakdown

| # | Component | Responsibility |
|---|-----------|----------------|
| 1 | **Custom Engine Integration** | Abstract rendering interface already implemented in Phase 1. Engine switching layer. WebKitGTK → custom engine migration path. |
| 2 | **Engine Selection** | Find and combine best existing packages (HTML parser, CSS engine, layout/paint, JS runtime). Write from scratch only if no suitable packages exist. Evaluate by license, Rust-native preference, maintenance, spec coverage, and integration effort. |
| 3 | **Custom DevTools** | Minimal custom inspector from scratch for the custom engine. DOM inspector, console, network timeline. |
| 4 | **Core Request Interception** | Native request/response interception for ad blocking, privacy features, request modification — not limited to extension API |
| 5 | **Custom Engine Prototype Phase 0** | Load basic HTML+CSS pages and pass ACID test |
| 6 | **Custom Engine Prototype Phase 1** | Load real-world sites (Wikipedia, GitHub, etc.) |
| 7 | **Search vs URL Detection** | TBD — determine Phase 3 vs Phase 2 |
| 8 | **Session Restore Navigation** | Full navigation history (back/forward stack) in session restore |
| 9 | **Tab Tiling** | Horizontal/vertical/grid tiling, nested splits, drag tabs between tiles, tile collapse on close |
| 10 | **Navigation State Model** | Explicit state machine for loading/loaded/failed/redirecting — design and implementation |
| 11 | **Keyboard Navigation / Focus Model** | Ctrl+Tab, Ctrl+1-8, Ctrl+9, focus-follows-mouse, tiled focus behavior |
| 12 | **Tab Groups** | Color-coded tab groups, group collapse/expand |
| 13 | **Cookie Viewer** | View/search/delete cookies, cookie inspection UI |

---

## Implementation Order (Phase 3)

1. **Abstract rendering interface** — define and implement against WebKitGTK (if not done in Phase 1).
2. **Engine selection** — evaluate and select custom engine packages.
3. **Custom engine prototype Phase 0** — HTML+CSS + ACID test.
4. **Custom engine prototype Phase 1** — real-world sites.
5. **Tab tiling** — horizontal/vertical/grid, nested splits, drag between tiles, tile collapse.
6. **Navigation state model** — explicit state machine.
7. **Search vs URL detection** — implementation.
8. **Session restore navigation** — full back/forward stack.
9. **Keyboard navigation / focus model** — Ctrl+Tab, Ctrl+1-8, tiled focus.
10. **Tab groups** — color-coded groups.
11. **Core request interception** — native, not extension-limited.

---

## Validation Milestones

- **Milestone 9 (Phase 3):** Abstract rendering interface defined and implemented against WebKitGTK. Custom engine prototype Phase 0: load a basic HTML+CSS page and pass ACID test. Core request interception implemented natively. Tab tiling implemented (horizontal/vertical/grid, nested splits, drag between tiles). Navigation state model. Search vs URL detection. Keyboard navigation/focus model. Tab groups. Cookie viewer.
- **Milestone 10 (Phase 3):** Custom engine prototype Phase 1: load real-world sites (Wikipedia, GitHub, etc.). Session restore with full navigation history.

---

## Risks

1. **Custom engine effort:** Writing or combining a production-ready rendering engine is a multi-year undertaking; scope and sequencing must be managed carefully.
2. **Engine selection:** Finding suitable existing packages with compatible licenses and sufficient spec coverage is uncertain.
3. **Integration complexity:** Combining separate engine components into a unified system requires deep systems programming expertise.
