# Phase 2 — Full Vivaldi-Level Feature Set

## Scope

Full Vivaldi-level feature set minus mail/calendar/feeds — advanced tab management (mute, stacking, hibernation), notes, themes, mouse gestures, keyboard shortcut editor, session management, advanced proxy, certificate handling, full extension host, multi-window support, integrated tools.

---

## Component Breakdown

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
| 20 | **DevTools Additions (Phase 2)** | Storage viewer (cookies, localStorage, IndexedDB, Cache Storage), security panel (certificate info, mixed content, CSP violations) |
| 21 | **Download Manager (Phase 2)** | Pause/resume, download categories, search, batch operations, speed limit, queue management |
| 22 | **Multi-Profile Support** | Multiple independent profiles, profile switching UI |
| 23 | **Settings Sync** | Cloud sync for settings across devices |

---

## Implementation Order (Phase 2)

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

---

## Validation Milestones

- **Milestone 7 (Phase 2):** Advanced proxy rules, PAC support, certificate handling UI, full WebExtensions API subset, URL autocomplete, search engine support, security indicators, bfcache integration, IDN/punycode, URL sanitization, multi-window support, tab mute/unmute, DevTools additions (storage viewer, security panel), download manager additions (pause/resume, categories, search, batch ops, speed limit, queue).
- **Milestone 8 (Phase 2):** Multi-profile support, settings export/import, settings sync.

---

## Risks

1. **WebKitGTK packaging/distribution:** Flatpak is the easiest path; deb/rpm packaging requires dependency management.
2. **Boa maturity:** Boa is not yet spec-complete. Side-panel scripts must use a restricted feature set or fail gracefully.
3. **Extension system scope:** Balancing a minimal Phase 1 extension API with a path to full WebExtensions compatibility requires careful interface design; too restrictive and Phase 2 migration is painful, too permissive and security suffers.
