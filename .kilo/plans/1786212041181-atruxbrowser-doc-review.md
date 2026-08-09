# ATRUXBrowser — Architecture & Implementation Plan

## What A Browser Must Have (Minimum Functional Layers)

Before scoping features, here is the irreducible set of components a desktop browser must have to be functional. Every item below is required; missing any of them means the application is not a browser.

| # | Layer | Responsibility | Implementation in ATRUXBrowser |
|---|-------|----------------|-------------------------------|
| 1 | **Networking** | HTTP/1.1, HTTP/2, HTTPS (TLS 1.2/1.3), cookies, redirects, caching, proxy support | libsoup via WebKitGTK (Phase 1). Basic proxy UI (Phase 1). Advanced proxy rules/PAC (Phase 2). Per-tab proxy routing (Phase 4). Request interception via extensions (Phase 1); core interception (Phase 3). Certificate handling UI (Phase 2). |
| 2 | **Rendering Engine** | HTML parsing, CSS box model/layout, painting, hit-testing | WebKitGTK 4.1 as fallback (Phase 1). Custom engine by combining best existing packages, or writing from scratch only if necessary (Phase 3). |
| 3 | **JavaScript Runtime** | ECMAScript execution, DOM bindings, event loop, GC | Page JS: QuickJS (Phase 1, primary). Boa as fallback for performance/security isolation needs (Phase 1). Extension/panel JS: Boa (Phase 1). QuickJS dev toggle (Phase 2). |
| 4 | **Navigation & URL Handling** | Address bar, autocomplete, history, back/forward, reload, stop | Basic URL bar with history back/forward/reload/stop (Phase 1). Protocol handling: use what we get, default missing protocol to https (Phase 1). Autocomplete (local history + bookmarks Phase 1; search engine suggestions Phase 2). Search vs URL detection (Phase 2). Navigation state model (Phase 2). Search engine support (TBD Phase 2/3). Security indicators (Phase 2). bfcache: use WebKitGTK bfcache in Phase 1; otherwise Phase 2. Session restore: current URL only in Phase 1; full navigation history in Phase 3. IDN/punycode (Phase 2). URL sanitization (Phase 2). HTTPS upgrade consideration (Phase 2+). |
| 5 | **Tab & Window Management** | Tab lifecycle (create, suspend, discard, restore), window frame, focus model | Basic tab bar (create/close/duplicate/pin, drag-reorder, detach) Phase 1. Tab tiling (Phase 3). Tab mute/unmute, stacking, hibernation, session tabs, tab search (Phase 2). Multi-window, window state (Phase 2). Keyboard nav/focus model, tab groups (Phase 3). Tab suspension (Phase 4). Advanced drag & drop (Phase 5). |
| 6 | **Storage & State** | Cookies, localStorage/sessionStorage, IndexedDB, Cache Storage, credentials | WebKitGTK provides web storage. Browser state: SQLite for bookmarks, history, settings, sessions (Phase 1). Credentials deferred to Phase 2. Encryption at rest via OS keyring (libsecret) in Phase 1. Import/export: bookmarks (HTML), settings/sessions (JSON) in Phase 1. Cookie viewer (Phase 3). |
| 7 | **DevTools Protocol** | At minimum: DOM inspector, console, network timeline | Phase 2: Reuse WebKit's built-in inspector via remote debugging protocol (console, basic DOM tree, basic network timeline). Phase 2 additions: storage viewer, security panel. Phase 3: Custom minimal DevTools for custom engine. |
| 8 | **Extension Host** | WebExtensions API (or subset): content scripts, background scripts, popup UI | Phase 1: Content scripts, popup UI, request interception (block+redirect), basic local storage, tabs API (query/create/close). Directory-based loading. Runtime prompts. Event-driven background only. File-based storage. Phase 2: Full WebExtensions API subset (sync storage, runtime messaging, bookmarks, cookies, history, downloads, notifications, context menus). Core request interception native (Phase 3). |
| 9 | **Downloads** | MIME sniffing, save-to-disk, progress, pause/resume, quarantine | Phase 1: Basic download, MIME sniffing, save dialog, progress indicator, notifications, download manager popover, quarantine, file type handling, retry failed. No pause/resume. Default to system Downloads folder; user-configurable custom folder. Phase 2: Pause/resume, download categories, search, batch operations, speed limit, queue management. |
| 10 | **Settings/Profile** | Preferences engine, user data directory, first-run experience | Phase 1: Single JSON file for preferences. Categories: General, Appearance, Privacy, Search, Proxy, Tabs, Language. Hybrid UI: main preferences window + quick toggles in chrome. First-run wizard: welcome → theme selection → import data → set defaults → finish. Single profile. File-based export/import. Reset to defaults + reset specific categories. Phase 2: Multi-profile support. Cloud sync deferred. |

**Vivaldi adds on top of this:** tab tiling/stacking, side panels, mouse gestures, keyboard shortcut editor, themes, session management, notes, integrated mail/calendar/feed reader, and extensive UI customization.

---

## Scope

- **Phase 1:** Functional desktop browser with Vivaldi-inspired UI touches — basic side panels, basic proxy, basic extension system, customizable UI chrome. Target: Linux desktop initially, but architecture is cross-platform Rust. Uses wry + WebKitGTK as the practical rendering fallback while the custom engine work begins. Basic tab bar without tiling.
- **Phase 2:** Full Vivaldi-level feature set minus mail/calendar/feeds — advanced tab management (mute, stacking, hibernation), notes, themes, mouse gestures, keyboard shortcut editor, session management, advanced proxy, certificate handling, full extension host, multi-window support, integrated tools. Cross-platform: Linux, Windows, macOS via wry abstraction.
- **Phase 3:** Custom engine integration — write our own engine or combine existing engines to replace the WebKitGTK fallback. Core request interception. Tab tiling (horizontal/vertical/grid, nested splits). Navigation state model. Search vs URL detection. Keyboard navigation/focus model. Tab groups.
- **Phase 4:** Per-tab proxy routing, proxy authentication, tab suspension.
- **Phase 5:** Advanced drag & drop (URL→tab, tab→tile).

## Architecture Decision: Rendering Engine

**Decision:** Use **wry as the cross-platform webview abstraction** in Phase 1/2, with WebKitGTK as the Linux backend. wry provides a unified Rust API while using platform-native webviews: WebKitGTK on Linux, WKWebView on macOS, WebView2 on Windows. This eliminates the need for platform-specific WebKit port work and aligns with the goal of a Rust-native, cross-platform browser.

**Decision:** Use **egui or iced for browser chrome/UI** instead of GTK4. Both are Rust-native, cross-platform GUI frameworks. egui is immediate-mode; iced is retained-mode. This keeps the entire stack in Rust and removes the GTK dependency.

**Rationale:**
- The user does not require GTK specifically. The priority is cross-platform and Rust.
- wry solves the cross-platform webview problem without writing platform-specific code.
- egui/iced provide native-looking Rust GUI toolkits that work on all three platforms.
- This architecture is simpler, more maintainable, and better aligned with the long-term goal of replacing the webview backend with a custom engine.
- When the custom engine is ready in Phase 3, we only need to replace wry's backend, not rewrite the entire UI layer.

**Phase 3 engine strategy:**
- **Primary approach:** Find and combine the best existing packages/components (HTML parser, CSS engine, layout/paint pipeline, JS runtime) and unify them behind the abstract wrapper.
- **Fallback:** Only write from scratch if no suitable existing packages exist or if licensing/compatibility requirements force it.
- **Evaluation criteria:** License compatibility, Rust-native preference, maintenance status, spec coverage, integration effort.

**Consequence for QuickJS/Boa:**
- In Phase 1, page JS runs via wry's webview backend (WebKitGTK's JavaScriptCore on Linux).
- QuickJS and Boa remain **not** the page JS runtime. They become the **extension/scripting engine** for:
  - Side panel widgets
  - User scripts / content script injection
  - DevTools panels
  - Internal automation/scripting
- This preserves the original architectural intent (custom JS engines) while accepting reality for web content rendering in the near term.

## Architecture Decision: UI Toolkit and Windowing

**Decision:** Use **egui or iced for browser chrome/UI** instead of GTK4. Both are Rust-native, cross-platform GUI frameworks. egui is immediate-mode; iced is retained-mode.

**Decision:** Use **wry for webview abstraction** instead of embedding WebKitGTK directly. wry provides a unified Rust API while using platform-native webviews: WebKitGTK on Linux, WKWebView on macOS, WebView2 on Windows.

**Rationale:**
- GTK4 is Linux-only. The project requires cross-platform support (Linux, Windows, macOS) from Phase 2.
- wry solves the cross-platform webview problem without writing platform-specific code.
- egui/iced provide native-looking Rust GUI toolkits that work on all three platforms.
- This keeps the entire stack in Rust, which aligns with the memory-safety and code-scannability goals.
- When the custom engine is ready in Phase 3, we only need to replace wry's backend, not rewrite the entire UI layer.
- The browser chrome (tabs, address bar, side panels, menus) is rendered in egui/iced.
- Web content is rendered in wry's webview, which uses the platform's native webview engine.

## Architecture Decision: JS Engine Strategy (Phase 1)

**Decision:** Use **QuickJS as the primary scripting engine** for Phase 1. Use **Boa as the fallback** for cases requiring performance isolation or Rust-native safety guarantees.

**Rationale:**
- QuickJS is more mature and spec-complete than Boa, making it safer for Phase 1 extension and panel scripts.
- QuickJS has a small footprint and fast startup, aligning with the minimalism principle.
- Boa serves as a fallback for specific use cases where its Rust-native safety or future JIT capabilities are beneficial.
- This reverses the earlier Boa-primary decision based on current engine maturity.

**Page JS in Phase 1 (via WebKitGTK):**
- Page scripts run via WebKitGTK's built-in JavaScriptCore. These are the scripts that web pages themselves load and execute (e.g., React apps, analytics, inline event handlers).
- Extension/panel scripts run via QuickJS (or Boa fallback). These are scripts the browser or user injects (content scripts, side panel logic, user scripts).
- The distinction matters because page scripts need maximum web compatibility, while extension scripts can tolerate some spec gaps.

**Page JS in Phase 3 (custom engine):**
- When WebKitGTK is replaced, JavaScriptCore disappears with it.
- Options: embed JSC standalone, promote QuickJS to page runtime, promote Boa to page runtime, or evaluate a fourth engine.
- Decision deferred until Phase 3 engine strategy is selected.

**Option A — Embed JavaScriptCore standalone:**
- JavaScriptCore (JSC) is the mature, spec-complete engine powering Safari and WebKit.
- Embedding JSC standalone means including it as a separate C/C++ library, independent of WebKitGTK.
- Pros: Maximum web compatibility, proven performance, identical behavior to Phase 1 page scripts.
- Cons: Large C++ codebase, adds build complexity, memory overhead of running two engines (JSC + custom engine), licensing (LGPL).
- This is the safest path for Phase 3 if web compatibility is the top priority.

**Pluggable JS engine performance impact:**
- The trait/vtable dispatch overhead is negligible (one pointer indirection per call, nanoseconds).
- Actual JS execution (parsing, compiling, bytecode) happens entirely inside the engine. The wrapper only orchestrates.
- Memory overhead is minimal: each engine instance is already isolated. The wrapper just holds a pointer.
- The real cost is marshaling data across engine boundaries (e.g., Rust → QuickJS → Rust), but this cost exists regardless of whether the engine is pluggable or hardcoded.
- **Conclusion:** Make it pluggable. The performance impact does not justify tight coupling.

**DOM bindings recommendation (3.4):**
- Use a **shared bridge layer** between the rendering engine and the JS engine.
- Neither side should own DOM bindings exclusively:
  - If rendering engine owns bindings → tightly coupled to every JS engine's API
  - If JS engine owns bindings → needs intimate knowledge of rendering engine internals
- The bridge layer translates JS-side DOM operations (e.g., `document.getElementById`) to rendering-engine operations, and vice versa for events/callbacks.
- This mirrors real browser architecture (WebKit's bindings layer, Servo's bridge).
- Critical for Phase 3 because it insulates combined packages from each other's internals.

## Phase 1 — Component Breakdown

| # | Component | Responsibility | Priority |
|---|-----------|----------------|----------|
| 1 | **wry Webview Embed (Cross-Platform)** | Window, webview lifecycle, navigation, process model — cross-platform via wry abstraction (WebKitGTK on Linux, WKWebView on macOS, WebView2 on Windows) | P0 |
| 2 | **egui/iced Browser Chrome** | Tab bar, navigation bar, side panels, menus — Rust-native cross-platform UI | P0 |
| 3 | **Proxy System (Basic)** | System proxy passthrough, manual HTTP/HTTPS/SOCKS proxy configuration | P0 |
| 4 | **Tab Bar (Basic)** | Tab lifecycle: create, close, duplicate, pin/unpin. Drag-to-reorder, detach to new window. | P0 |
| 5 | **Navigation Bar** | URL bar with back/forward/reload/stop. Protocol handling: use what we get, default missing protocol to https. Security indicators deferred. | P0 |
| 6 | **Side Panels** | Dynamic web-panel system, editable headers, resize handles, state persistence | P1 |
| 7 | **Bookmarks & History** | CRUD, folders, import/export (HTML/Netscape), search. Stored in SQLite. | P1 |
| 8 | **Downloads** | Basic download, MIME sniffing, save dialog, progress indicator, notifications, download manager popover, quarantine, file type handling, retry failed. No pause/resume in Phase 1. | P1 |
| 9 | **Settings Engine** | Single JSON file for preferences. Categories: General, Appearance, Privacy, Search, Proxy, Tabs, Language. Hybrid UI: main preferences window + quick toggles in chrome. First-run wizard: welcome → theme selection → import data → set defaults → finish. | P1 |
| 10 | **Basic Extension Host** | Minimal extension system: content scripts, popup UI, request interception (block+redirect), basic local storage, tabs API (query/create/close). Directory-based loading. Runtime prompts. Event-driven background only. File-based storage. | P1 |
| 11 | **JS Engine Wrapper (Pluggable)** | Unified trait for JS engines (QuickJS primary, Boa fallback). Exposes eval, bind, call, GC. Negligible overhead. | P1 |
| 12 | **Boa Scripting Host** | Embed Boa for side-panel scripts and user automation as fallback engine; expose safe API surface | P2 |
| 13 | **Session Management** | Save/restore window + tab layout states | P2 |
| 14 | **Basic Themes** | Light/dark mode, accent colors, CSS-injected theming for chrome | P2 |
| 15 | **Custom Engine Integration** | Abstract rendering interface design and wry implementation (Phase 1). Engine switching layer (Phase 3). | P2 |

**Out of scope for Phase 1:**
- Full WebExtensions API (only basic extension system in Phase 1)
- Mouse gestures
- Advanced keyboard shortcut editor
- Notes, mail, calendar, feeds
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

## Why wry + egui/iced in Phase 1

The table below explains why we chose wry + egui/iced over GTK4 and platform-specific WebKit ports.

| Approach | Cross-Platform | Rust-Native | GTK Dependency | Phase 1 Effort | Phase 3 Migration |
|----------|---------------|-------------|----------------|-----------------|-------------------|
| **GTK4 + WebKitGTK** | Linux only | Partial (GTK is C) | Yes | Low | Medium — replace WebKitGTK |
| **Own abstraction + 3 WebKit ports** | Yes | Partial | Yes (Linux) | High — write abstraction + 3 backends | Low — swap backend |
| **wry + egui/iced** | Yes | Yes | No | Low — use existing crates | Low — replace wry backend |
| **wry + Slint** | Yes | Yes | No | Low — use existing crates | Low — replace wry backend |

**Conclusion:** wry + egui/iced/Slint gives us cross-platform, Rust-native, low-effort Phase 1 with a clean migration path to the custom engine in Phase 3. GTK4 adds Linux-only dependency without corresponding benefit.

## UI Toolkit Deep-Dive: GTK4 vs egui vs iced vs wry

This section documents the detailed analysis of why we chose wry + egui/iced over GTK4.

### The Open Question Explained

**Original question:** Should we use GTK4 for the browser shell?

**Why GTK4 was the original choice:**
- Mature, production-ready GUI toolkit
- Deep Linux integration
- Large widget set out of the box
- WebKitGTK is already a GTK widget, so integration is natural

**Why GTK4 is no longer the choice:**
- **Linux-only.** The project requires cross-platform support (Linux, Windows, macOS) from Phase 2.
- **C-based.** GTK4 is written in C with Rust bindings (`gtk4-rs`). This adds FFI boundary complexity.
- **Not needed.** The browser chrome (tabs, address bar, side panels) doesn't need a full desktop GUI toolkit. A lightweight Rust-native UI is sufficient.
- **Phase 3 incompatibility.** When the custom engine arrives, GTK4's GTK-specific integration code becomes dead weight.

### Why wry + egui/iced

| Factor | GTK4 + WebKitGTK | wry + egui/iced |
|--------|-----------------|-----------------|
| **Cross-platform** | Linux only | Yes — Linux, Windows, macOS |
| **Language** | C + Rust bindings | Pure Rust |
| **Webview backend** | WebKitGTK only | Platform-native (WebKitGTK, WKWebView, WebView2) |
| **UI chrome** | GTK widgets | egui/iced widgets |
| **Phase 3 migration** | Replace WebKitGTK + rewrite GTK chrome | Replace wry backend only |
| **Maintenance burden** | High — 3 platform ports if cross-platform | Low — wry handles platform differences |
| **Community** | Mature but declining | Growing rapidly in Rust ecosystem |

### egui vs iced vs Slint

| Factor | egui | iced | Slint |
|--------|------|------|-------|
| **Paradigm** | Immediate-mode | Retained-mode | Declarative retained-mode |
| **Language** | Rust | Rust | Rust (.slint DSL + Rust) |
| **Learning curve** | Low | Medium | Medium — need to learn .slint DSL |
| **Customization** | High — you control every frame | Medium — framework manages state | High — declarative UI with full styling |
| **Performance** | Fast for simple UIs, can struggle with complex layouts | Generally better for complex, stateful UIs | Excellent — compiled to native code, GPU-accelerated |
| **Maturity** | High — used in production by many projects | Medium — promising but newer | Medium-High — backed by company, used in embedded and desktop |
| **Widget set** | Basic but growing | More complete out of the box | Rich — buttons, tabs, scroll areas, list views, etc. |
| **Styling** | Manual per-widget | Theming support | CSS-like styling, themes, animations |
| **Best for** | Tools, debug panels, custom UIs | Full applications with standard widgets | Polished desktop applications with native look |
| **License** | MIT/Apache-2.0 | MIT/Apache-2.0 | GPL/Commercial — requires careful license review |

**Recommendation:** Evaluate egui, iced, and Slint with a prototype. For a browser chrome with tabs, address bar, and side panels, all three could work. egui gives maximum control; iced gives structure; Slint gives the most polished native look with CSS-like styling. The choice depends on which feels more natural for the team and license compatibility.

**License note on Slint:** Slint uses GPL for the open-source version, with commercial licenses available. For a proprietary browser, this requires either: (1) purchasing a commercial license, (2) releasing the browser under GPL, or (3) using Slint only for internal tools while keeping the browser itself proprietary. This must be resolved before selecting Slint.

### What wry Gives Us

wry is a cross-platform webview library that abstracts the platform's native webview:

| Platform | Backend | Webview Engine |
|----------|---------|----------------|
| **Linux** | WebKitGTK | WebKit |
| **macOS** | WKWebView | WebKit |
| **Windows** | WebView2 | Blink (Edge) |

This means:
- **No platform-specific code** in our application
- **Native webview performance** on each platform
- **Automatic updates** as platforms update their webviews
- **Smaller attack surface** than embedding a full browser engine

### What This Means for Phase 3

When the custom engine is ready:
1. We implement a new wry backend that uses our custom engine instead of the platform's native webview
2. The browser chrome (egui/iced) does not change
3. The extension system, JS engine wrapper, and all other Phase 1/2 code continues to work

This is why wry is the right choice: it's a **replacement boundary**, not a permanent dependency.

## Phase 2 — Component Breakdown

| # | Component | Responsibility |
|---|-----------|----------------|
| 1 | **Extension Host (Full)** | Full WebExtensions API subset: content scripts, background scripts, popup UI, storage (sync + local), runtime messaging, bookmarks, cookies, history, downloads, notifications, context menus |
| 2 | **Mouse Gestures** | Configurable gesture bindings for navigation and UI actions |
| 3 | **Keyboard Shortcut Editor** | Full shortcut customization with conflict detection |
| 4 | **Advanced Tab Management** | Tab stacking, tab hibernation, session tabs, tab search, tab mute/unmute |
| 5 | **Notes** | Integrated note-taking with web-page association |
| 6 | **Themes & Skins** | Full theming engine, CSS variable injection, community theme support |
| 7 | **QuickJS Integration** | Re-introduce as optional developer/profiling engine behind a flag |
| 8 | **Cross-Platform** | Windows and macOS support |
| 9 | **Advanced Proxy System** | Proxy rules, PAC scripts, proxy auth, upstream proxy chains |
| 10 | **Certificate Handling** | Certificate pinning UI, certificate error overrides, HSTS management |
| 11 | **URL Autocomplete** | Autocomplete from history + bookmarks + search engine suggestions |
| 12 | **Search Engine Support** | Search engine management, keyword search, omnibox behavior |
| 13 | **Security Indicators** | Lock icon, certificate info popover, mixed content warnings, HSTS status |
| 14 | **Navigation State Model** | Explicit state machine for loading/loaded/failed/redirecting |
| 15 | **bfcache Integration** | WebKitGTK back/forward cache configuration and session restore integration |
| 16 | **IDN/Punycode Handling** | Internationalized domain name support |
| 17 | **URL Sanitization** | Strip tracking parameters, normalize URLs |
| 18 | **HTTPS Upgrade Consideration** | Consider changing http:// to https:// automatically — needs more discussion |
| 19 | **Multi-Window Support** | Multiple windows, window state save/restore (size, position, maximized) |
| 20 | **DevTools (Phase 2)** | Reuse WebKit's built-in inspector via remote debugging protocol. Console, basic DOM tree, basic network timeline. Storage viewer, security panel. |
| 21 | **Download Manager (Phase 2)** | Pause/resume, download categories, search, batch operations, speed limit, queue management |
| 22 | **Multi-Profile Support** | Multiple independent profiles, profile switching UI |
| 23 | **Settings Sync** | Cloud sync for settings across devices |

## Phase 3 — Component Breakdown

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

## Phase 4 — Component Breakdown

| # | Component | Responsibility |
|---|-----------|----------------|
| 1 | **Per-Tab Proxy Routing** | Proxy toggle per-window/per-tab, profile-based proxy settings |
| 2 | **Proxy Auth** | Proxy authentication support (Basic, NTLM, Digest) |
| 3 | **Tab Suspension** | Manual and automatic tab suspension for memory management |

## Phase 5 — Component Breakdown

| # | Component | Responsibility |
|---|-----------|----------------|
| 1 | **Advanced Drag & Drop** | Drag URL from outside browser to create tab, drag tab into specific tile in tiling view |

## Proposed Implementation Order (Phase 1)

1. **Scaffold Rust app + wry webview** — prove cross-platform window + webview works on Linux. Load test sites, prove navigation, cookies, localStorage work.
2. **Choose and scaffold UI framework** — evaluate egui vs iced vs Slint. Implement basic window chrome (title bar, close/minimize/maximize).
3. **Proxy system (basic)** — system proxy passthrough, manual HTTP/HTTPS/SOCKS proxy configuration.
4. **Basic extension host** — content scripts, basic popup UI, safe API surface for request interception.
5. **JS Engine Wrapper (pluggable)** — define trait, implement QuickJS primary + Boa fallback. Prove with simple scripts.
6. **Tab bar (basic)** — create, close, duplicate, pin/unpin, drag-to-reorder, detach.
7. **Navigation bar** — URL bar with back/forward/reload/stop, protocol handling (default missing to https).
8. **Side panels** — dynamic web-panel system with editable headers and persistence.
9. **Bookmarks + History** — standard CRUD with import/export.
10. **Downloads** — OS integration and progress UI.
11. **Settings engine** — preferences, first-run, user data directory.
12. **Session management** — save/restore current URLs.
13. **Basic Themes** — light/dark + accent colors.
14. **DevTools bridge** — reuse wry/webview remote debugging protocol wiring.

## Proposed Implementation Order (Phase 2)

1. **Advanced tab management** — mute/unmute, stacking, hibernation, session tabs, tab search.
2. **Multi-window support** — multiple windows, window state save/restore.
3. **Full extension host** — WebExtensions API subset.
4. **URL autocomplete** — history + bookmarks + search engine suggestions.
5. **Search engine support** — management, keyword search, omnibox.
6. **Security indicators** — lock icon, cert popover, mixed content warnings.
7. **bfcache integration** — session restore with navigation history.
8. **IDN/punycode handling** — internationalized domain names.
9. **URL sanitization** — strip tracking parameters.
10. **Certificate handling** — pinning UI, error overrides, HSTS.
11. **Advanced proxy** — rules, PAC scripts.
12. **Keyboard shortcuts** — full customization editor.

## Proposed Implementation Order (Phase 3)

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

## Proposed Implementation Order (Phase 4)

1. **Per-tab proxy routing** — proxy toggle per-window/per-tab.
2. **Proxy authentication** — Basic, NTLM, Digest.
3. **Tab suspension** — manual and automatic.

## Proposed Implementation Order (Phase 5)

1. **Advanced drag & drop** — URL→tab, tab→tile.

## Key Technical Decisions To Resolve

| # | Decision | Options | Recommended |
|---|----------|---------|-------------|
| 1 | **Rendering backend (Phase 1)** | WebKitGTK 4.0 vs 4.1 | WebKitGTK 4.1 (latest stable, better security) — used as fallback while custom engine is developed |
| 2 | **Abstract rendering interface** | Define in Phase 1 vs defer to Phase 3 | Define and implement in Phase 1 against WebKitGTK; makes Phase 3 migration tractable |
| 3 | **Custom engine strategy (Phase 3)** | Combine existing packages vs write from scratch | Combine best existing packages first; write from scratch only if no suitable packages exist. Evaluate by license, Rust-native preference, maintenance, spec coverage, integration effort. |
| 4 | **Custom engine prototype scope** | ACID test only vs real-world sites | Phase 0: HTML+CSS + ACID. Phase 1: real-world sites (Wikipedia, GitHub, etc.) |
| 5 | **JS engine (Phase 1)** | Boa primary vs QuickJS primary | QuickJS primary (mature, spec-complete, small footprint). Boa fallback for performance isolation or Rust-native safety needs. |
| 6 | **JS engine architecture** | Pluggable vs tightly coupled | Pluggable via unified trait. Overhead is negligible (vtable dispatch); actual JS execution happens inside the engine. |
| 7 | **Shell language** | Rust (gtk-rs) vs C/GTK | Rust — matches Boa/QuickJS integration and memory-safety goal |
| 8 | **UI toolkit (Phase 1)** | GTK4 vs libadwaita vs egui vs iced vs Slint vs wry-style | **Cross-platform Rust GUI:** egui, iced, or Slint for browser chrome + wry for webview abstraction. GTK4 is Linux-only and not needed. Evaluate all three; Slint requires GPL/commercial license review. |
| 9 | **Cross-platform webview (Phase 1/2)** | GTK-only vs own abstraction vs wry-style | **wry** — cross-platform webview abstraction. Uses WebKitGTK on Linux, WKWebView on macOS, WebView2 on Windows. Eliminates need for platform-specific WebKit port work. |
| 10 | **Process model** | Single-process vs multi-process (WebKit site per process) | Start single-process for simplicity; migrate to WebKit's multi-process model when stable |
| 10 | **Boa API surface** | Full ECMAScript vs restricted sandbox | Restricted sandbox with explicit allow-list for side-panel scripts |
| 11 | **Proxy system (Phase 1)** | System proxy only vs custom proxy UI | Custom proxy UI in Phase 1 (basic proxy); advanced rules/PAC in Phase 2 |
| 12 | **Extension system (Phase 1)** | Full WebExtensions vs minimal custom API | Minimal custom extension API in Phase 1; full WebExtensions subset in Phase 2 |
| 13 | **Autocomplete (Phase 1)** | Local history + bookmarks vs full omnibox | Basic URL bar only in Phase 1. Autocomplete, search engine suggestions, omnibox — Phase 2/3. |
| 14 | **Search vs URL detection** | Smart detection vs explicit URL entry | TBD — defer to Phase 2 or Phase 3 |
| 15 | **Protocol handling (Phase 1)** | What to do with missing protocol | Default missing protocol to https in Phase 1. Consider http→https upgrade in Phase 2 (needs more discussion). |
| 16 | **Security indicators (Phase 1)** | Lock icon only vs full indicator set | Lock icon + certificate info popover in Phase 2. Mixed content, HSTS, dangerous site warnings — Phase 2+. |
| 17 | **bfcache (Phase 1)** | Use WebKitGTK bfcache vs custom | If using WebKitGTK: use its bfcache in Phase 1. Otherwise: Phase 2. |
| 18 | **Browser state format (Phase 1)** | JSON vs SQLite | SQLite for bookmarks, history, settings, sessions. Human-readable JSON for settings export/import. |
| 19 | **Credentials storage (Phase 1)** | Plaintext vs OS keyring vs custom encryption | Defer credentials to Phase 2. Use OS keyring (libsecret) for any sensitive data in Phase 1. |
| 20 | **Cookie viewer** | Phase 1 vs later | Phase 3 |
| 21 | **DevTools approach** | Reuse WebKit inspector vs custom | Reuse WebKit's built-in inspector via remote debugging protocol in Phase 2. Custom minimal DevTools for custom engine in Phase 3. |
| 22 | **DevTools scope (Phase 2)** | Full inspector vs minimal | Minimal: console, basic DOM tree, basic network timeline, storage viewer, security panel. No source viewer or profiler in Phase 2. |
| 23 | **Extension Phase 1 APIs** | Full WebExtensions vs minimal custom | Content scripts, popup UI, request interception (block+redirect), basic local storage, tabs API (query/create/close only). |
| 24 | **Extension packaging (Phase 1)** | CRX/XPI vs custom vs directory-based | Directory-based loading in Phase 1. No CRX/XPI complexity. |
| 25 | **Extension permissions (Phase 1)** | All-or-nothing vs runtime prompts vs sandboxed | Runtime prompts for sensitive APIs. |
| 26 | **Content script injection (Phase 1)** | Manifest-based vs programmatic vs both | Both manifest-based and programmatic. |
| 27 | **Request interception scope (Phase 1)** | Block only vs block+redirect vs modify headers/body | Block + redirect. No body modification in Phase 1. |
| 28 | **Background scripts (Phase 1)** | None vs event-driven vs persistent | Event-driven background only. No persistent background in Phase 1. |
| 29 | **Extension storage (Phase 1)** | File-based vs SQLite vs in-memory | File-based in Phase 1. SQLite in Phase 2. |
| 30 | **Download pause/resume (Phase 1)** | Include vs defer | Defer to Phase 2. WebKitGTK support is limited. |
| 31 | **Download manager UI (Phase 1)** | Popover vs full window vs both | Simple list in popover in Phase 1. |
| 32 | **Download folder (Phase 1)** | System default vs custom vs per-file | Default to system Downloads folder. User-configurable custom folder in settings. |
| 33 | **Preferences format (Phase 1)** | Single JSON vs multiple JSON vs SQLite | Single JSON file for Phase 1. Easier to debug, no dependencies. |
| 34 | **Settings categories (Phase 1)** | Which categories to include | General, Appearance, Privacy, Search, Proxy, Tabs, Language in Phase 1. |
| 35 | **Settings UI (Phase 1)** | Single window vs per-feature vs hybrid | Hybrid: main preferences window with categories + quick toggles in chrome. |
| 36 | **First-run wizard (Phase 1)** | Steps included | Welcome → theme selection → import data → set defaults → finish. |
| 37 | **Profile support (Phase 1)** | Single vs multiple profiles | Single profile in Phase 1. Multi-profile in Phase 2. |
| 38 | **Settings sync (Phase 1)** | Local only vs file-based vs cloud | File-based export/import in Phase 1. Cloud sync deferred. |

## Decisions Log

| Date | Decision | Context |
|------|----------|---------|
| 2026-08-09 | **Phase numbering:** Phases renumbered from A/B to 1/2/3/4/5 | User request for numeric phases |
| 2026-08-09 | **Rendering engine:** WebKitGTK 4.1 as Phase 1 fallback | Only production-ready option for HTML/CSS/JS in Phase 1 |
| 2026-08-09 | **Custom engine strategy:** Combine existing packages first, write from scratch only if necessary | User choice; evaluate by license, Rust-native preference, maintenance, spec coverage |
| 2026-08-09 | **Custom engine prototype scope:** Phase 0 = HTML+CSS + ACID test; Phase 1 = real-world sites | User decision |
| 2026-08-09 | **Abstract rendering interface:** Define and implement in Phase 1 against WebKitGTK | Makes Phase 3 migration tractable |
| 2026-08-09 | **JS engine (Phase 1):** QuickJS primary, Boa fallback | User preference; QuickJS is more mature |
| 2026-08-09 | **JS engine architecture:** Pluggable via unified trait | Performance impact negligible |
| 2026-08-09 | **DOM bindings:** Shared bridge layer between rendering engine and JS engine | Prevents tight coupling |
| 2026-08-09 | **Proxy system:** Basic proxy UI in Phase 1, advanced rules/PAC in Phase 2, per-tab routing in Phase 4 | User request |
| 2026-08-09 | **Extension system:** Basic extension system in Phase 1, full WebExtensions in Phase 2 | User request; request interception via extensions in Phase 1, core interception in Phase 3 |
| 2026-08-09 | **Tab lifecycle (Phase 1):** create, close, duplicate, pin/unpin, drag-reorder, detach | User request |
| 2026-08-09 | **Tab tiling:** Phase 3 only | User request |
| 2026-08-09 | **Multi-window support:** Phase 2 | User request |
| 2026-08-09 | **Keyboard navigation/focus model:** Phase 3 | User request |
| 2026-08-09 | **Tab suspension:** Phase 4 | User request |
| 2026-08-09 | **Advanced drag & drop:** Phase 5 | User request |
| 2026-08-09 | **Browser state format:** SQLite for bookmarks/history/settings/sessions | User accepted recommendation |
| 2026-08-09 | **Credentials storage:** Deferred to Phase 2; use OS keyring (libsecret) in Phase 1 | User accepted recommendation |
| 2026-08-09 | **Cookie viewer:** Phase 3 | User request |
| 2026-08-09 | **DevTools Phase 1:** Reuse WebKit's built-in inspector via remote debugging protocol | User accepted recommendation |
| 2026-08-09 | **DevTools Phase 1 scope:** Console, basic DOM tree, basic network timeline only | User accepted recommendation |
| 2026-08-09 | **DevTools moved to Phase 2** | User requested move from Phase 1 to Phase 2 |
| 2026-08-09 | **Extension Phase 1 APIs:** Content scripts, popup UI, request interception (block+redirect), basic local storage, tabs API (query/create/close only) | User accepted recommendation |
| 2026-08-09 | **Extension packaging (Phase 1):** Directory-based loading | User accepted recommendation |
| 2026-08-09 | **Extension permissions (Phase 1):** Runtime prompts for sensitive APIs | User accepted recommendation |
| 2026-08-09 | **Request interception scope (Phase 1):** Block + redirect, no body modification | User accepted recommendation |
| 2026-08-09 | **Background scripts (Phase 1):** Event-driven only, no persistent | User accepted recommendation |
| 2026-08-09 | **Extension storage (Phase 1):** File-based; SQLite in Phase 2 | User accepted recommendation |
| 2026-08-09 | **Downloads Phase 1:** Basic download, no pause/resume | User accepted recommendation |
| 2026-08-09 | **Download manager UI (Phase 1):** Popover only | User accepted recommendation |
| 2026-08-09 | **Download folder:** Default system Downloads; user-configurable custom folder | User accepted recommendation |
| 2026-08-09 | **Preferences format:** Single JSON file | User accepted recommendation |
| 2026-08-09 | **Settings categories:** General, Appearance, Privacy, Search, Proxy, Tabs, Language | User accepted recommendation |
| 2026-08-09 | **Settings UI:** Hybrid — main preferences window + quick toggles in chrome | User accepted recommendation |
| 2026-08-09 | **First-run wizard:** Welcome → theme selection → import data → set defaults → finish | User accepted recommendation |
| 2026-08-09 | **Profile support:** Single profile in Phase 1; multi-profile in Phase 2 | User accepted recommendation |
| 2026-08-09 | **Settings sync:** File-based export/import in Phase 1; cloud sync deferred | User accepted recommendation |
| 2026-08-09 | **Navigation bar Phase 1:** Basic URL bar, back/forward/reload/stop, protocol handling (default missing to https) | User accepted recommendation |
| 2026-08-09 | **Security indicators:** Phase 2 | User request |
| 2026-08-09 | **bfcache:** Use WebKitGTK bfcache in Phase 1 | Conditional on using WebKitGTK |
| 2026-08-09 | **Session restore:** Current URL only in Phase 1; full navigation history in Phase 3 | User accepted recommendation |
| 2026-08-09 | **IDN/punycode:** Phase 2 | User request |
| 2026-08-09 | **URL sanitization:** Phase 2 | User request |
| 2026-08-09 | **Search vs URL detection:** TBD Phase 2/3 | User request |
| 2026-08-09 | **Navigation state model:** Phase 2 | User request |
| 2026-08-09 | **Autocomplete:** Local history + bookmarks Phase 1; search engine suggestions Phase 2 | User accepted recommendation |
| 2026-08-09 | **Custom engine reality:** Five major gaps in Rust ecosystem — CSS layout, rendering, IndexedDB, bfcache, DevTools protocol | Engineering assessment |
| 2026-08-09 | **Phase 3 engine path:** Combine existing packages first; write from scratch only if no suitable packages exist | User decision |
| 2026-08-09 | **DevTools moved to Phase 2** | User requested move from Phase 1 to Phase 2 |
| 2026-08-09 | **Project identity:** Goal is to replace old engines, using a Vivaldi-grade browser for users as the vehicle | User clarified |
| 2026-08-09 | **Custom engine path decision:** The goal is to replace all old engines. Use WebKitGTK as Phase 1 fallback, but Phase 3+ is about owning the rendering stack | User clarified |
| 2026-08-09 | **IndexedDB feasibility:** No mature Rust crate exists. Would need to build mini-database engine inside browser. 3–6 months minimum for basic implementation | Engineering assessment |
| 2026-08-09 | **CSS layout feasibility:** No production-ready Rust CSS layout engine. Servo `style` is research-grade. 6–18 months basic, 2–4 years full compliance | Engineering assessment |
| 2026-08-09 | **Rendering/painting feasibility:** No production-ready Rust browser renderer. `webrender` is production-grade GPU only, needs compositor + software fallback. 3–6 months basic, 1+ year production-quality | Engineering assessment |
| 2026-08-09 | **bfcache feasibility:** No Rust implementation exists. Requires deep integration between layout, JS, and networking. 3–6 months minimum | Engineering assessment |
| 2026-08-09 | **DevTools protocol feasibility:** No mature Rust implementation. Building from scratch is 6–12 months minimum | Engineering assessment |
| 2026-08-09 | **Text rendering approach:** FreeType is the standard choice for font rasterization. HarfBuzz for shaping. Both are mature C libraries with Rust bindings | Engineering recommendation |
| 2026-08-09 | **Spec compliance vs real-world compatibility:** Passing the spec is necessary but not sufficient. Real-world sites depend on bug-for-bug compatibility with Chrome/Firefox/Safari. Expect 60–80% compatibility with a spec-compliant engine, 95%+ only after years of site-specific fixes | Engineering assessment |
| 2026-08-09 | **WebKitGTK cross-platform status:** WebKitGTK is Linux-only (GTK port). Not available on Windows/macOS. For cross-platform Phase 2, we need either platform-specific WebKit ports or a cross-platform abstraction layer | Engineering assessment |
| 2026-08-09 | **Cross-platform rendering strategy for Phase 2:** Options: (1) Abstract rendering backend and use WebKitGTK on Linux, WKWebView on macOS, WebKit2 on Windows; (2) Use wry-style webview abstraction; (3) Accelerate custom engine to be truly cross-platform | **Selected: wry-style webview abstraction (Option B)** — use existing cross-platform solution rather than building own abstraction. GTK4 is Linux-only and not required. |
| 2026-08-09 | **UI toolkit preference:** User prefers egui or iced over GTK4. egui is immediate-mode, iced is retained-mode. Both are Rust-native and cross-platform. Slint also added as option. Decision: evaluate all three, pick based on browser chrome requirements and license compatibility | User preference |

## Risks

1. **wry maturity:** wry is less mature than GTK4/WebKitGTK. Cross-platform webview abstraction may have platform-specific bugs or limitations.
2. **egui/iced selection:** Choosing between egui (immediate-mode) and iced (retained-mode) affects the entire browser chrome architecture. Need to evaluate both before committing.
3. **Boa maturity:** Boa is not yet spec-complete. Side-panel scripts must use a restricted feature set or fail gracefully.
4. **Tiling complexity:** Drag-to-resize with persistent layouts requires careful state serialization.
5. **Performance with tiling:** Multiple webviews in a single window are memory-heavy; consider suspending non-visible tiles.
6. **Custom engine effort:** Writing or combining a production-ready rendering engine is a multi-year undertaking; scope and sequencing must be managed carefully.
7. **Proxy complexity:** Advanced proxy features (PAC scripts, upstream chains, per-context routing) interact deeply with the networking stack and can introduce subtle security/correctness bugs.
8. **Extension system scope:** Balancing a minimal Phase 1 extension API with a path to full WebExtensions compatibility requires careful interface design; too restrictive and Phase 2 migration is painful, too permissive and security suffers.

## Why wry + WebKitGTK in Phase 1

The table below breaks down what WebKitGTK provides versus building each piece in Rust. Without WebKitGTK, Phase 1 would take **2–4 years** longer because the major gaps have no production-ready Rust equivalents today.

| # | What WebKitGTK Provides | Rust Equivalent / Crate | Maturity | Time to Build in Rust |
|---|-------------------------|------------------------|----------|----------------------|
| 1 | **HTML5 parser + DOM tree** | `html5ever` (Servo) | High — parsing is solid, but full tree construction and error recovery still need work | **3–6 months** for parser + DOM tree that handles real-world pages |
| 2 | **CSS parser + selector matching** | `cssparser`, `selectors` (Servo) | High — used in production by Firefox/Servo for parsing | **2–4 months** for parsing; matching is straightforward |
| 3 | **CSS box model + layout engine** | None complete. `style` (Servo) is research-quality only | Low — no production-ready Rust CSS layout engine exists | **6–18 months** for basic layout; **2–4 years** for full CSS2.1 + CSS3 compliance |
| 4 | **Rendering / painting** | `webrender` (Mozilla, GPU), `pixels` (software), `skia-safe` (bindings) | Medium — `webrender` is production-grade but is a renderer, not a browser engine. `skia-safe` bindings exist but are complex | **3–6 months** basic; **1+ year** for production-quality |
| 5 | **JavaScriptCore (page JS)** | `rquickjs` (QuickJS bindings, mature), `boa` (Rust-native, incomplete), `deno_core` (V8 bindings, heavy) | Medium — QuickJS via rquickjs is usable. Boa is not spec-complete. V8 via deno_core is production-grade but massive | **Already solved** with rquickjs; Boa needs **6–12 months** more |
| 6 | **Networking: HTTP/1.1, HTTP/2, HTTPS** | `reqwest`/`hyper` + `rustls` | High — production-ready, widely used | **Already solved** — weeks to integrate |
| 7 | **Cookie handling** | `cookie_store` + `reqwest` | High — mature | **Already solved** — days to integrate |
| 8 | **Request caching** | `cached` crate, or custom with `sled`/`rocksdb` | Medium — basic caching exists, HTTP cache semantics need work | **1–3 months** for spec-compliant HTTP cache |
| 9 | **Navigation lifecycle** (load, redirect, stop) | Custom code on top of webview/networking | N/A | **1–2 months** glue code |
| 10 | **Back/forward cache (bfcache)** | None direct equivalent | Low | **3–6 months** to implement properly |
| 11 | **Downloads** (MIME, save-to-disk) | `mime_guess` + std::fs | High — trivial | **Already solved** — days |
| 12 | **Web storage: localStorage/sessionStorage** | `sled`/`rocksdb` + custom API | Medium — storage backend exists, API semantics need work | **1–2 months** |
| 13 | **Web storage: IndexedDB** | None mature | Low | **3–6 months** minimum |
| 14 | **Web storage: Cache Storage** | None mature | Low | **2–4 months** |
| 15 | **DevTools Protocol** (remote debugging) | None mature | Low | **6–12 months** for basic protocol |
| 16 | **Accessibility (ATK/AT-SPI)** | `atk` crate | Low — barely used | **2–4 months** for basic support |
| 17 | **Process model / sandbox** | Custom + OS primitives | N/A | **3–6 months** for single-process; **6–12 months** for multi-process |
| 18 | **Hit-testing / input events** | Custom code | N/A | **1–2 months** |

**Summary:**

| Category | WebKitGTK | Rust-native path |
|----------|-----------|------------------|
| **Already solved in Rust** | Networking, cookies, basic downloads | `reqwest`, `rustls`, `cookie_store` — production-ready |
| **Partially solved** | HTML parsing, CSS parsing, JS (QuickJS) | `html5ever`, `cssparser`, `rquickjs` — usable but need integration |
| **Major gap** | CSS layout, rendering, IndexedDB, bfcache, DevTools protocol | No production-ready crates. Would be multi-year effort |
| **Phase 1 critical path** | Rendering engine + layout + JS for pages | **Not feasible to replace in Phase 1** |

**Conclusion:** WebKitGTK in Phase 1 gives you a production-ready, spec-compliant browser engine for **free**. Replacing it with Rust crates would add **2–4 years** of engine work before you even start building Vivaldi features. The plan correctly treats WebKitGTK as the fallback while the abstract interface is built, and defers custom engine work to Phase 3.

The only parts you could realistically replace earlier are:
- **Networking** — already solved in Rust
- **Extension JS** — QuickJS via rquickjs is mature
- **Storage backends** — SQLite/sled are mature, but web storage APIs still need work

Everything else (CSS layout, rendering, bfcache, DevTools protocol) has no mature Rust equivalent today.

## Technical Deep-Dive: The Five Gaps

This section documents the detailed analysis of why CSS layout, rendering, IndexedDB, bfcache, and DevTools protocol are the hard wall preventing a production-ready browser engine in Rust today.

### 1. CSS Layout

CSS layout is not one algorithm. It's a stack of interacting layout systems:

| Layer | Spec Complexity | What It Does |
|-------|-----------------|--------------|
| CSS2.1 box model | ~300 pages | Normal flow, floats, clearance, inline formatting, line breaking |
| CSS Flexbox | ~200 pages | One-dimensional layout, flex lines, min/max sizing, flex factors |
| CSS Grid | ~400 pages | Two-dimensional layout, named grid lines/areas, implicit grids, spanning |
| CSS Subgrid | ~100 pages | Nested grids that inherit parent track sizing |
| CSS Container Queries | ~150 pages | Layout-aware responsive design |
| CSS Writing Modes | ~200 pages | Vertical text, right-to-left, bidirectional text |
| CSS Box Alignment | ~100 pages | Align-self, justify-content, stretch — applies across flex/grid/block |

**The core problem is not the algorithms.** Flexbox and grid layout are mathematically well-understood.

**The real problems are:**
1. **Spec compliance is endless.** CSS has ~10,000+ test cases in WPT (Web Platform Tests). Getting 90% pass requires handling thousands of edge cases.
2. **Intrinsic sizing is a maze.** `min-content`, `max-content`, `fit-content`, `minmax()`, `auto` — these interact with floats, overflow, writing modes, and replaced elements in complex ways.
3. **Line breaking is a research problem.** CSS text layout requires font shaping (HarfBuzz), bidi algorithm (Unicode UAX#9), line breaking (UAX#14), and then box layout for each line. This alone is months of work.
4. **Everything interacts with everything.** A float in one flex item affects the height of a grid item that contains a block with `writing-mode: vertical-rl`.

**Servo's `style` crate** has done the most work here, but it's research-grade. Making it production-ready would be 6–18 months for basic layout, 2–4 years for full compliance.

**Rust ecosystem status:** No production-ready CSS layout engine exists. `style` (Servo) is the closest but research-grade.

### 2. Rendering / Painting

Rendering is not "draw pixels." It's a multi-stage pipeline with strict correctness requirements:

| Stage | What It Must Do | Complexity |
|-------|-----------------|------------|
| Layer building | Decide which DOM subtrees become compositing layers | Medium |
| Paint ordering | Traverse layers in correct z-order, respecting stacking contexts, z-index, isolation | High |
| Text rendering | Sub-pixel positioning, font hinting, fallback fonts, OpenType features, color fonts | High |
| Image decoding | JPEG, PNG, WebP, AVIF, SVG, GIF — progressive decoding, color space conversion | Medium |
| Color management | sRGB, Display P3, wide gamut — CSS `color()` function, HDR | Medium |
| GPU acceleration | Texture atlases, tile-based rendering, texture memory limits, driver bugs | High |
| Software fallback | Must produce identical output to GPU path | High |
| Layer compositing | Blend modes, filters, masks, backdrop-filter, clip-path | High |
| Printing | Pagination, page breaks, CMYK color space, print-specific CSS | Medium |
| Accessibility tree | Every painted element must expose accessible role, name, state, value | Medium |

**The real problems:**
1. **WebKit's rendering pipeline is ~20 years of accumulated complexity.** It handles edge cases from IE5-era sites to modern CSS effects.
2. **GPU programming is hard.** A compositor must handle texture atlasing, tile-based rendering, synchronization between CPU paint and GPU display, memory pressure, and driver-specific bugs.
3. **Text is the hardest part.** Font rendering requires font fallback chains, OpenType features, sub-pixel anti-aliasing, color fonts, and variable fonts.
4. **FreeType + HarfBuzz are the standard choices for text rendering in a custom engine.** FreeType handles font rasterization. HarfBuzz handles text shaping (ligatures, kerning, bidi). Both are mature C libraries with Rust bindings (`freetype-rs`, `rustybuzz`). This is one area where existing tools are solid.

**Rust ecosystem status:** No production-ready browser renderer exists. `webrender` (Mozilla) is production-grade GPU rendering but is not a complete browser renderer — it's the paint step only. `skia-safe` bindings exist but are complex. A complete renderer would be 3–6 months basic, 1+ year production-quality.

### 3. IndexedDB

IndexedDB is a transactional, schema-aware, indexed NoSQL database:

| Feature | What It Means |
|---------|---------------|
| Transactions | Atomic read/write within transaction scope. Rollback on failure. |
| Schema management | Database versions. Upgrade schemas while page is running. |
| Indexes | B-tree structures over serialized JS values. Cursor-based queries. |
| Value serialization | Structured clone algorithm for JS objects, arrays, blobs, dates. |
| Concurrency | Multiple tabs/windows accessing same database. Locking, version conflicts. |
| Crash recovery | Write-ahead logging or similar to prevent corruption. |
| Query planner | Choose which index to use, merge cursors, handle compound keys. |

**Where it meets us:**
- **Phase 1:** WebKitGTK provides IndexedDB. You never touch it.
- **Phase 3:** When replacing WebKitGTK, you must implement IndexedDB yourself. No mature Rust crate exists. `heed` (LMDB wrapper) and `rocksdb` are possible backends, but you'd still need to map all IDB semantics onto them.

**Reality:** Building a mini-database engine inside the browser. 3–6 months minimum for basic implementation passing conformance tests, 1+ year for production quality.

**Rust ecosystem status:** No mature IndexedDB implementation exists.

### 4. bfcache (Back/Forward Cache)

**What it's for:** Instant back/forward navigation.

Without bfcache, clicking Back requires re-fetching, re-parsing, re-layouting, re-painting, and re-executing JavaScript — taking 200ms–2s. With bfcache, the entire page state is snapshotted in memory and restored instantly.

**Why you need it:** Modern browsers are expected to navigate back/forward instantly. Without it, the browser feels broken on any site with non-trivial JS.

**What it must snapshot:**
- Complete DOM tree
- JavaScript heap state
- Scroll position
- Form data / input values
- Canvas content
- Pending network requests
- CSS computed styles
- Event listeners

**What it must restore:**
- All of the above, instantly
- Pending requests must resume
- Timers and intervals must restart at correct times
- Scroll position must be pixel-perfect
- Form validation state must be preserved

**Rust ecosystem status:** No Rust implementation exists. Requires deep integration between layout, JS, and networking. 3–6 months minimum to implement properly.

### 5. DevTools Protocol

Chrome DevTools Protocol (CDP) is ~100+ method pairs with bidirectional streaming. Firefox uses a different protocol (Remote Agent).

**What DevTools are tied to:**
- DOM inspector ↔ engine's DOM tree nodes
- CSS panel ↔ engine's computed style / CSSOM
- Network panel ↔ engine's networking stack
- Memory/JS profiler ↔ engine's JS heap and GC
- Sources/debugger ↔ engine's JS execution pipeline
- Console ↔ engine's JS runtime and console API

**Can you use Firefox DevTools with WebKitGTK?** No. The protocols are different, and the backend actors are tightly coupled to each engine's internal object model.

**Can you build a custom DevTools frontend?** The UI is the easy part. The protocol backend is everything. You'd need to implement:
- A WebSocket/pipe server
- JSON-RPC message dispatch
- DOM change observation
- CSS style tracking
- Network request interception and logging
- JS heap profiling
- Debugger protocol (breakpoints, step, call stack)

**Rust ecosystem status:** No mature Rust DevTools protocol implementation exists. Building from scratch is 6–12 months minimum for basic functionality.

---

## Crate and Library Inventory

This section documents every crate and C/C++ library we expect to use, organized by phase. License compatibility must be verified before inclusion.

### Phase 1 — Production Dependencies

| Component | Crate / Library | Language | License | Maturity | Notes |
|-----------|-----------------|----------|---------|----------|-------|
| **Webview abstraction** | `wry` | Rust | MIT/Apache-2.0 | Medium | Cross-platform webview. Uses WebKitGTK on Linux, WKWebView on macOS, WebView2 on Windows. |
| **Windowing** | `tao` | Rust | MIT/Apache-2.0 | Medium | Windowing library used by wry. Provides window creation and event loop. |
| **Browser chrome UI** | `egui` or `iced` | Rust | MIT/Apache-2.0 | Medium-High | Rust-native cross-platform GUI for tabs, address bar, side panels. egui is immediate-mode; iced is retained-mode. |
| **Rendering backend (via wry)** | WebKitGTK 4.1 | C/C++ | LGPL-2.1+ | High | Linux backend for wry. Provides HTML/CSS/JS for web content. |
| **Networking (via wry/WebKitGTK)** | libsoup | C | LGPL-2.1+ | High | HTTP/1.1, HTTP/2, HTTPS, cookies, caching. |
| **TLS (via wry/WebKitGTK)** | GIO/TLS / GnuTLS or OpenSSL | C | LGPL/GPL | High | HTTPS support through WebKitGTK stack. |
| **JavaScript (extensions)** | `rquickjs` | Rust + C | MIT | Medium-High | QuickJS bindings. Primary extension JS engine. |
| **JavaScript (fallback)** | `boa` | Rust | MIT | Low-Medium | Rust-native JS engine. Fallback for specific use cases. Not spec-complete. |
| **Database (browser state)** | `rusqlite` | Rust | MIT | High | SQLite for bookmarks, history, settings, sessions. |
| **Secrets / credentials** | `libsecret` / `keyring` | C / Rust | LGPL-2.1+ / MIT | Medium | OS keyring integration for sensitive data. |
| **Downloads** | `mime_guess` | Rust | MIT | High | MIME type detection for downloads. |
| **File dialogs** | Native OS dialogs via wry/tao | Rust | MIT/Apache-2.0 | Medium | Save/load file choosers. Platform-native. |
| **JSON config** | `serde_json` | Rust | MIT/Apache-2.0 | High | Preferences file read/write. |
| **Async runtime** | `tokio` or wry's built-in | Rust | MIT/Apache-2.0 | High | Async runtime if needed. wry may have its own event loop integration. |
| **Logging** | `tracing` / `env_logger` | Rust | MIT/Apache-2.0 | High | Structured logging. |
| **Error handling** | `thiserror` / `anyhow` | Rust | MIT/Apache-2.0 | High | Error types and propagation. |
| **Text processing** | `url` / `percent-encoding` | Rust | MIT/Apache-2.0 | High | URL parsing and normalization. |
| **Time/date** | `chrono` | Rust | MIT/Apache-2.0 | High | Timestamps for history, downloads, sessions. |

### Phase 1 — Optional / Dev Dependencies

| Component | Crate / Library | Language | License | Maturity | Notes |
|-----------|-----------------|----------|---------|----------|-------|
| **Async runtime (alternative)** | `tokio` | Rust | MIT/Apache-2.0 | High | Only if wry integration requires it. |
| **CLI parsing** | `clap` | Rust | MIT/Apache-2.0 | High | Command-line argument parsing for debugging. |
| **Image handling** | `image` | Rust | MIT/Apache-2.0 | High | Favicon decoding, screenshot handling. |
| **Compression** | `flate2` / `brotli` | Rust | MIT/Apache-2.0 | High | HTTP content decoding if needed beyond what wry provides. |

### Phase 3+ — Research / Custom Engine Dependencies

| Component | Crate / Library | Language | License | Maturity | Notes |
|-----------|-----------------|----------|---------|----------|-------|
| **HTML parser** | `html5ever` | Rust | MIT/Apache-2.0 | Medium-High | Servo's HTML5 parser. Solid parsing, needs DOM tree integration. |
| **CSS parser** | `cssparser` | Rust | MIT/Apache-2.0 | High | Servo's CSS parser. Production-grade for parsing. |
| **CSS selector matching** | `selectors` | Rust | MIT/Apache-2.0 | High | Servo's selector engine. Production-grade. |
| **CSS layout** | `style` (Servo) | Rust | MIT/Apache-2.0 | Low | Research-grade CSS layout engine. Not production-ready. |
| **JS engine** | `rquickjs` | Rust + C | MIT | Medium-High | QuickJS bindings. Could be promoted to page JS runtime. |
| **JS engine** | `boa` | Rust | MIT | Low-Medium | Rust-native JS engine. Needs more spec compliance work. |
| **JS engine** | `deno_core` | Rust | MIT/Apache-2.0 | Medium | V8 bindings via Deno. Heavy but production-grade. |
| **2D rendering** | `webrender` | Rust | MPL-2.0 | High | Mozilla's GPU renderer. Paint step only, not a full browser renderer. |
| **2D rendering** | `pixels` | Rust | MIT | Medium | Software renderer. Useful for fallback/testing. |
| **2D rendering** | `skia-safe` | Rust | MIT/Apache-2.0 | Medium | Skia bindings. Complex but production-grade. |
| **Font rasterization** | `freetype-rs` | Rust (FreeType C) | FreeType license | High | Font rasterization. Standard choice for custom engines. |
| **Text shaping** | `rustybuzz` | Rust (HarfBuzz C) | MIT | Medium | HarfBuzz bindings. Text shaping, ligatures, kerning, bidi. |
| **Font fallback** | `fontconfig` | C | MIT | High | Font discovery and fallback. Mature, widely used. |
| **Database (IndexedDB backend)** | `heed` | Rust | MIT/Apache-2.0 | Medium | LMDB wrapper. Possible IndexedDB storage backend. |
| **Database (IndexedDB backend)** | `rocksdb` | C++ | Apache-2.0 / GPL | High | Possible IndexedDB storage backend. |
| **Networking** | `reqwest` / `hyper` | Rust | MIT/Apache-2.0 | High | HTTP client. Production-ready alternative to WebKitGTK networking. |
| **TLS** | `rustls` | Rust | MIT/Apache-2.0 | High | TLS implementation. Pure Rust alternative to system TLS. |
| **Cookie handling** | `cookie_store` | Rust | MIT/Apache-2.0 | Medium | Cookie jar for reqwest. |
| **Caching** | `cached` | Rust | MIT/Apache-2.0 | Medium | HTTP cache semantics. Needs work for spec compliance. |
| **Storage** | `sled` | Rust | MIT/Apache-2.0 | Medium | Embedded database. Possible localStorage/sessionStorage backend. |
| **Accessibility** | `atk` | C | LGPL-2.1+ | Low | ATK/AT-SPI bindings. Barely used in Rust ecosystem. |

### C/C++ Libraries (Used via FFI or System Integration)

| Library | Purpose | License | Notes |
|---------|---------|---------|-------|
| **WebKitGTK** | Rendering engine (via wry on Linux), networking, JS (JSC), web storage | LGPL-2.1+ | Phase 1 Linux backend via wry. Large dependency but essential. |
| **GTK4** | Not used — replaced by egui/iced + wry/tao | LGPL-2.1+ | Removed from dependencies. Not needed for cross-platform Rust stack. |
| **FreeType** | Font rasterization | FreeType license | Standard for text rendering in custom engines. |
| **HarfBuzz** | Text shaping | MIT | Standard for text shaping in custom engines. |
| **Fontconfig** | Font discovery and fallback | MIT | Standard for font management on Linux. |
| **libsoup** | HTTP networking (via wry/WebKitGTK on Linux) | LGPL-2.1+ | |
| **GIO/TLS** | TLS support (via wry/WebKitGTK on Linux) | LGPL-2.1+ | |
| **libsecret** | OS keyring integration | LGPL-2.1+ | Credential storage. |
| **SQLite** | Database engine | Public domain | Browser state storage. |

### License Compatibility Matrix

| License | Commercial Use | Modification | Distribution | Notes |
|---------|---------------|--------------|--------------|-------|
| MIT | Yes | Yes | Yes | Permissive. Most Rust crates use this. |
| Apache-2.0 | Yes | Yes | Yes | Permissive. Patent grant included. |
| LGPL-2.1+ | Yes | Yes | Yes | Dynamic linking allowed. Static linking requires object files. |
| MPL-2.0 | Yes | Yes | Yes | File-level copyleft. Changes to MPL files must be shared. |
| FreeType license | Yes | Yes | Yes | Permissive with attribution. |
| MIT (HarfBuzz) | Yes | Yes | Yes | Permissive. |
| GPL | Yes | Yes | Yes | Copyleft. Avoid for libraries if possible. |

**Key licensing considerations:**
- **WebKitGTK is LGPL-2.1+.** This means you can dynamically link to it in a proprietary browser. Static linking would require releasing object files. This is acceptable for Phase 1 as a wry backend.
- **wry is MIT/Apache-2.0.** Permissive. Safe to use.
- **tao is MIT/Apache-2.0.** Permissive. Safe to use.
- **egui is MIT/Apache-2.0.** Permissive. Safe to use.
- **iced is MIT/Apache-2.0.** Permissive. Safe to use.
- **Servo crates (`html5ever`, `cssparser`, `selectors`) are MIT/Apache-2.0.** Permissive. Safe to use.
- **`webrender` is MPL-2.0.** File-level copyleft. Safe for most use cases, but changes to webrender itself must be released.
- **`skia-safe` is MIT/Apache-2.0.** Permissive. Skia itself is BSD-licensed.

---

## Validation Plan

- **Milestone 1 (Weeks 1–2):** Cross-platform Rust window with wry webview loads 5 test sites (Wikipedia, GitHub, Reddit, a SPA, and a site with heavy JS) on Linux. Basic proxy UI works (system proxy + manual HTTP/HTTPS/SOCKS). egui/iced chrome renders tabs and address bar.
- **Milestone 2 (Weeks 3–4):** JS Engine Wrapper implemented: QuickJS primary, Boa fallback. Basic extension host loads a QuickJS-based content script and popup. Tab bar (basic) with create/close/duplicate/pin, drag-to-reorder, detach. Navigation bar with basic URL input, back/forward/reload/stop, and protocol handling (default missing protocol to https).
- **Milestone 3 (Weeks 5–6):** Side panels with editable headers and persistent state. Request interception via basic extension API (ad blocking demo).
- **Milestone 4 (Weeks 7–8):** Bookmarks, history, downloads, and settings.
- **Milestone 5 (Weeks 9–10):** Session save/restore (current URLs only).
- **Milestone 6 (Weeks 11–12):** Polish, basic themes, beta release.
- **Milestone 7 (Phase 2):** Advanced proxy rules, PAC support, certificate handling UI, full WebExtensions API subset, URL autocomplete, search engine support, security indicators, bfcache integration, IDN/punycode, URL sanitization, multi-window support, tab mute/unmute, DevTools (WebKit built-in inspector via remote debugging protocol: console, basic DOM, basic network; storage viewer, security panel), download manager additions (pause/resume, categories, search, batch ops, speed limit, queue), cross-platform support (Windows, macOS via wry).
- **Milestone 8 (Phase 2):** Multi-profile support, settings export/import, settings sync.
- **Milestone 9 (Phase 3):** Abstract rendering interface defined and implemented against wry. Custom engine prototype Phase 0: load a basic HTML+CSS page and pass ACID test. Core request interception implemented natively. Tab tiling implemented (horizontal/vertical/grid, nested splits, drag between tiles). Navigation state model. Search vs URL detection. Keyboard navigation/focus model. Tab groups. Cookie viewer.
- **Milestone 10 (Phase 3):** Custom engine prototype Phase 1: load real-world sites (Wikipedia, GitHub, etc.). Session restore with full navigation history.
- **Milestone 11 (Phase 4):** Per-tab proxy routing, proxy authentication, tab suspension.
- **Milestone 12 (Phase 5):** Advanced drag & drop (URL→tab, tab→tile).
