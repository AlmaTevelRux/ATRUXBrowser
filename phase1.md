# Phase 1 — Functional Desktop Browser

## Scope

Functional desktop browser with Vivaldi-inspired UI touches — basic side panels, basic proxy, basic extension system, customizable UI chrome. Target: Linux desktop. Uses WebKitGTK as the practical rendering fallback while the custom engine work begins. Basic tab bar without tiling.

---

## Architecture Decisions

- **Rendering backend:** WebKitGTK 4.1 (latest stable, better security) — used as fallback while custom engine is developed
- **Abstract rendering interface:** Define and implement in Phase 1 against WebKitGTK; makes Phase 3 migration tractable
- **Custom engine strategy:** Combine best existing packages first; write from scratch only if no suitable packages exist. Evaluate by license, Rust-native preference, maintenance, spec coverage, integration effort.
- **Custom engine prototype scope:** Phase 0: HTML+CSS + ACID. Phase 1: real-world sites (Wikipedia, GitHub, etc.)
- **JS engine (Phase 1):** QuickJS primary (mature, spec-complete, small footprint). Boa fallback for performance isolation or Rust-native safety needs.
- **JS engine architecture:** Pluggable via unified trait. Overhead is negligible (vtable dispatch); actual JS execution happens inside the engine.
- **Shell language:** Rust with `gtk4-rs` — matches Boa/QuickJS integration and memory-safety goal
- **UI toolkit:** GTK4 for maximum customization; libadwaita constrains chrome styling
- **Process model:** Start single-process for simplicity; migrate to WebKit's multi-process model when stable
- **Boa API surface:** Restricted sandbox with explicit allow-list for side-panel scripts
- **Proxy system (Phase 1):** Custom proxy UI in Phase 1 (basic proxy); advanced rules/PAC in Phase 2
- **Extension system (Phase 1):** Minimal custom extension API in Phase 1; full WebExtensions subset in Phase 2
- **Autocomplete (Phase 1):** Basic URL bar only in Phase 1. Autocomplete, search engine suggestions, omnibox — Phase 2/3.
- **Search vs URL detection:** TBD — defer to Phase 2 or Phase 3
- **Protocol handling (Phase 1):** Default missing protocol to https in Phase 1. Consider http→https upgrade in Phase 2 (needs more discussion).
- **Security indicators (Phase 1):** Lock icon + certificate info popover in Phase 2. Mixed content, HSTS, dangerous site warnings — Phase 2+.
- **bfcache (Phase 1):** If using WebKitGTK: use its bfcache in Phase 1. Otherwise: Phase 2.
- **Session restore nav state (Phase 1):** Current URL only in Phase 1. Full navigation history — Phase 3.
- **IDN/punycode:** Phase 2
- **URL sanitization:** Phase 2
- **Navigation state model:** Phase 2
- **Search engine support:** TBD — determine based on Phase 1 user feedback
- **Cookie viewer:** Phase 3
- **DevTools Phase 1 approach:** Reuse WebKit's built-in inspector via remote debugging protocol in Phase 1. Custom minimal DevTools for custom engine in Phase 3.
- **DevTools Phase 1 scope:** Minimal: console, basic DOM tree, basic network timeline. No source viewer or profiler in Phase 1.
- **Extension Phase 1 APIs:** Content scripts, popup UI, request interception (block+redirect), basic local storage, tabs API (query/create/close only).
- **Extension packaging (Phase 1):** Directory-based loading in Phase 1. No CRX/XPI complexity.
- **Extension permissions (Phase 1):** Runtime prompts for sensitive APIs.
- **Content script injection (Phase 1):** Both manifest-based and programmatic.
- **Request interception scope (Phase 1):** Block + redirect. No body modification in Phase 1.
- **Background scripts (Phase 1):** Event-driven background only. No persistent background in Phase 1.
- **Extension storage (Phase 1):** File-based in Phase 1. SQLite in Phase 2.
- **Download pause/resume (Phase 1):** Defer to Phase 2. WebKitGTK support is limited.
- **Download manager UI (Phase 1):** Simple list in popover in Phase 1.
- **Download folder (Phase 1):** Default to system Downloads folder. User-configurable custom folder in settings.
- **Preferences format (Phase 1):** Single JSON file for Phase 1. Easier to debug, no dependencies.
- **Settings categories (Phase 1):** General, Appearance, Privacy, Search, Proxy, Tabs, Language in Phase 1.
- **Settings UI (Phase 1):** Hybrid: main preferences window with categories + quick toggles in chrome.
- **First-run wizard (Phase 1):** Welcome → theme selection → import data → set defaults → finish.
- **Profile support (Phase 1):** Single profile in Phase 1. Multi-profile in Phase 2.
- **Settings sync (Phase 1):** File-based export/import in Phase 1. Cloud sync deferred.

---

## Component Breakdown

| # | Component | Responsibility | Priority |
|---|-----------|----------------|----------|
| 1 | **WebKitGTK Embed (Fallback)** | Window, webview lifecycle, navigation, process model — fallback until custom engine is ready | P0 |
| 2 | **Proxy System (Basic)** | System proxy passthrough, manual HTTP/HTTPS/SOCKS proxy configuration | P0 |
| 3 | **Tab Bar (Basic)** | Tab lifecycle: create, close, duplicate, pin/unpin. Drag-to-reorder, detach to new window. | P0 |
| 4 | **Navigation Bar** | URL bar with back/forward/reload/stop. Protocol handling: use what we get, default missing protocol to https. Security indicators deferred. | P0 |
| 5 | **Side Panels** | Dynamic web-panel system, editable headers, resize handles, state persistence | P1 |
| 6 | **Bookmarks & History** | CRUD, folders, import/export (HTML/Netscape), search. Stored in SQLite. | P1 |
| 7 | **Downloads** | Basic download, MIME sniffing, save dialog, progress indicator, notifications, download manager popover, quarantine, file type handling, retry failed. No pause/resume in Phase 1. | P1 |
| 8 | **Settings Engine** | Single JSON file for preferences. Categories: General, Appearance, Privacy, Search, Proxy, Tabs, Language. Hybrid UI: main preferences window + quick toggles in chrome. First-run wizard: welcome → theme selection → import data → set defaults → finish. | P1 |
| 9 | **Basic Extension Host** | Minimal extension system: content scripts, popup UI, request interception (block+redirect), basic local storage, tabs API (query/create/close). Directory-based loading. Runtime prompts. Event-driven background only. File-based storage. | P1 |
| 10 | **JS Engine Wrapper (Pluggable)** | Unified trait for JS engines (QuickJS primary, Boa fallback). Exposes eval, bind, call, GC. Negligible overhead. | P1 |
| 11 | **Boa Scripting Host** | Embed Boa for side-panel scripts and user automation as fallback engine; expose safe API surface | P2 |
| 12 | **Session Management** | Save/restore window + tab layout states | P2 |
| 13 | **Basic Themes** | Light/dark mode, accent colors, CSS-injected theming for chrome | P2 |
| 14 | **DevTools Protocol Bridge** | Reuse WebKit's built-in inspector via remote debugging protocol. Console, basic DOM tree, basic network timeline. | P2 |
| 15 | **Custom Engine Integration** | Abstract rendering interface design and WebKitGTK implementation (Phase 1). Engine switching layer (Phase 3). | P2 |

**Out of scope for Phase 1:**
- Full WebExtensions API (only basic extension system in Phase 1)
- Mouse gestures
- Advanced keyboard shortcut editor
- Notes, mail, calendar, feeds
- Cross-platform (Linux-only for now)
- Multiple profiles
- Advanced proxy rules/PAC/scripts (basic proxy in Phase 1; advanced in Phase 2)
- Per-tab proxy routing (Phase 4)
- Proxy authentication (Phase 4)
- Certificate handling UI (Phase 2)
- Tab tiling (Phase 3)
- Tab mute/unmute (Phase 2)
- Tab groups (Phase 3)
- Multi-window support (Phase 2)
- Window state save/restore (Phase 2)
- Keyboard navigation / focus model (Phase 3)
- Tab suspension (Phase 4)
- Advanced drag & drop: URL→tab, tab→tile (Phase 5)
- Cookie viewer (Phase 3)

---

## Implementation Order (Phase 1)

1. **Scaffold GTK app + WebKitGTK webview** — prove navigation, cookies, localStorage work.
2. **Proxy system (basic)** — system proxy passthrough, manual HTTP/HTTPS/SOCKS proxy configuration.
3. **Basic extension host** — content scripts, basic popup UI, safe API surface for request interception.
4. **JS Engine Wrapper (pluggable)** — define trait, implement QuickJS primary + Boa fallback. Prove with simple scripts.
5. **Tab bar (basic)** — create, close, duplicate, pin/unpin, drag-to-reorder, detach.
6. **Navigation bar** — URL bar with back/forward/reload/stop, protocol handling (default missing to https).
7. **Side panels** — dynamic web-panel system with editable headers and persistence.
8. **Bookmarks + History** — standard CRUD with import/export.
9. **Downloads** — OS integration and progress UI.
10. **Settings engine** — preferences, first-run, user data directory.
11. **Session management** — save/restore current URLs.
12. **Basic Themes** — light/dark + accent colors.
13. **DevTools bridge** — remote debugging protocol wiring.

---

## Validation Milestones

- **Milestone 1 (Weeks 1–2):** GTK window with a single WebKitGTK webview loads 5 test sites (Wikipedia, GitHub, Reddit, a SPA, and a site with heavy JS). Basic proxy UI works (system proxy + manual HTTP/HTTPS/SOCKS).
- **Milestone 2 (Weeks 3–4):** JS Engine Wrapper implemented: QuickJS primary, Boa fallback. Basic extension host loads a QuickJS-based content script and popup. Tab bar (basic) with create/close/duplicate/pin, drag-to-reorder, detach. Navigation bar with basic URL input, back/forward/reload/stop, and protocol handling (default missing protocol to https).
- **Milestone 3 (Weeks 5–6):** Side panels with editable headers and persistent state. Request interception via basic extension API (ad blocking demo).
- **Milestone 4 (Weeks 7–8):** Bookmarks, history, downloads, and settings.
- **Milestone 5 (Weeks 9–10):** Session save/restore (current URLs only).
- **Milestone 6 (Weeks 11–12):** Polish, basic themes, DevTools bridge (WebKit built-in inspector via remote debugging protocol: console, basic DOM, basic network), beta release.

---

## Risks

1. **WebKitGTK packaging/distribution:** Flatpak is the easiest path; deb/rpm packaging requires dependency management.
2. **Boa maturity:** Boa is not yet spec-complete. Side-panel scripts must use a restricted feature set or fail gracefully.
3. **Extension system scope:** Balancing a minimal Phase 1 extension API with a path to full WebExtensions compatibility requires careful interface design; too restrictive and Phase 2 migration is painful, too permissive and security suffers.
