# Phase 4 — Per-Tab Proxy & Tab Suspension

## Scope

Per-tab proxy routing, proxy authentication, tab suspension.

---

## Component Breakdown

| # | Component | Responsibility |
|---|-----------|----------------|
| 1 | **Per-Tab Proxy Routing** | Proxy toggle per-window/per-tab, profile-based proxy settings |
| 2 | **Proxy Auth** | Proxy authentication support (Basic, NTLM, Digest) |
| 3 | **Tab Suspension** | Manual and automatic tab suspension for memory management |

---

## Implementation Order (Phase 4)

1. **Per-tab proxy routing** — proxy toggle per-window/per-tab.
2. **Proxy authentication** — Basic, NTLM, Digest.
3. **Tab suspension** — manual and automatic.

---

## Validation Milestones

- **Milestone 11 (Phase 4):** Per-tab proxy routing, proxy authentication, tab suspension.

---

## Risks

1. **Proxy complexity:** Advanced proxy features (PAC scripts, upstream chains, per-context routing) interact deeply with the networking stack and can introduce subtle security/correctness bugs.
2. **Tab suspension:** Suspending tabs requires careful state management to avoid data loss and ensure smooth restoration.
