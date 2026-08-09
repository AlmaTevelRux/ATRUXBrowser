# ATRUXBrowser — Project Hub

This is the main project document. It links to the detailed phase plans, architecture decisions, and technical analysis.

## Documentation

| Document | Description |
|----------|-------------|
| **[ATRUXBrowser.md](ATRUXBrowser.md)** | Original architectural guidelines and vision |
| **[phase1.md](phase1.md)** | Phase 1: Functional desktop browser with Vivaldi-inspired UI |
| **[phase2.md](phase2.md)** | Phase 2: Full Vivaldi-level feature set |
| **[phase3.md](phase3.md)** | Phase 3: Custom engine integration |
| **[phase4.md](phase4.md)** | Phase 4: Per-tab proxy routing, proxy auth, tab suspension |
| **[phase5.md](phase5.md)** | Phase 5: Advanced drag & drop |
| **Architecture Plan** | `.kilo/plans/1786212041181-atruxbrowser-doc-review.md` — Full architecture with all decisions, component breakdowns, validation milestones, crate inventory, and technical deep-dive |

## Quick Reference

- **Shell:** Rust + `gtk4-rs`
- **Rendering (Phase 1):** WebKitGTK 4.1
- **Rendering (Phase 3+):** Custom engine or combined packages
- **JS Engine (Phase 1):** QuickJS primary, Boa fallback
- **Browser State:** SQLite for bookmarks/history/settings/sessions
- **Preferences:** Single JSON file
- **Extensions (Phase 1):** Minimal custom API, directory-based loading
- **Phase 1 Target:** Linux desktop only
- **Phase 2 Target:** Cross-platform (Windows, macOS)

## Phase Summary

| Phase | Focus |
|-------|-------|
| **Phase 1** | Functional browser shell around WebKitGTK: tabs, navigation, proxy, extensions, bookmarks, downloads, settings |
| **Phase 2** | Vivaldi-level features: multi-window, tiling, advanced proxy, full extensions, security indicators, themes |
| **Phase 3** | Custom rendering engine: replace WebKitGTK with combined Rust packages or custom implementation |
| **Phase 4** | Per-tab proxy, proxy authentication, tab suspension |
| **Phase 5** | Advanced drag & drop (URL→tab, tab→tile) |

## Key Technical Decisions

- **Rendering engine:** WebKitGTK 4.1 as Phase 1 fallback; custom engine in Phase 3
- **Custom engine strategy:** Combine best existing packages first; write from scratch only if necessary
- **JS engine:** QuickJS primary for extensions, Boa fallback; page JS via WebKitGTK's JavaScriptCore in Phase 1
- **Browser state:** SQLite for structured data, JSON for preferences
- **Settings:** Single JSON file, hybrid UI, first-run wizard
- **Extensions:** Minimal custom API in Phase 1, full WebExtensions subset in Phase 2
- **DevTools:** Reuse WebKit's built-in inspector in Phase 2; custom DevTools for custom engine in Phase 3

## Critical Path Awareness

The five major gaps in the Rust ecosystem that prevent replacing WebKitGTK in Phase 1:

1. **CSS layout** — No production-ready Rust CSS layout engine exists
2. **Rendering / painting** — No production-ready Rust browser renderer exists
3. **IndexedDB** — No mature Rust implementation exists
4. **bfcache** — No Rust implementation exists
5. **DevTools protocol** — No mature Rust implementation exists

These are documented in detail in the Architecture Plan under "Technical Deep-Dive: The Five Gaps."
