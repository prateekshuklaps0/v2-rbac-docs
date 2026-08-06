# Audit Logs — Frontend Development Map, Design Architecture & UI/UX
> Companion to `README.md` | Scope: **frontend only** (fe-anandi + super-admin) | No backend/code changes in this doc
> Last updated: 2026-06-03

---

## 0. TL;DR

- The audit-log **backend + the org-admin page already exist**. This is a *fix + extend + add-the-other-half* effort, not a greenfield build.
- **fe-anandi** has a working but **buggy** page (`/admin/audit-logs`). Two of its four filters silently return zero rows; platform events can never appear; no date filter; raw-JSON change display.
- **super-admin** has **nothing** — yet it is the *only* place the platform-level events (module/action catalog changes, org bootstrap) can ever be shown, because those are written with `org_id = NULL`.
- This document defines: information architecture, page layout, component architecture (respecting each app's existing design system), filter bar, table-vs-timeline decision, action color system, before/after diff UX, all empty/loading/error/denied states, audit-specific UX rules, and a phased frontend roadmap.
- A final appendix lists the **data-contract gaps** the frontend will need from the backend — to be actioned *later*, per your instruction.

---

## 1. What already exists (the starting line)

### Backend (do not change yet — listed for contract reference)
- Table `audit_logs`: `actor_id, actor_email, target_type, target_id, action, before(JSONB), after(JSONB), ip, user_agent, org_id, created_at`.
- Read API: `GET /api/v2/org/:orgId/rbac/audit-logs`, gated by RBAC action `audit-logs.audit-logs.view`.
  - Params: `page`, `limit` (default 50), `targetType` (exact), `action` (iLike contains), `actorId`.
  - Returns per row: `{ id, actor:{id,name,email}, targetType, targetId, action, before, after, ip, createdAt }`. **Does not return** `user_agent` or `org_id`.
  - **Hard-filters** `where.org_id = Number(orgId)` → platform (`org_id = NULL`) events are unreachable through this endpoint.

### Event vocabulary (what is actually logged)
| Scope | `target_type` | `action` values |
|---|---|---|
| **Org** (→ fe-anandi) | `user` | `user.create`, `user.attachToOrg`, `user.update`, `user.delete` |
| **Org** | `role` | `role.create`, `role.update`, `role.delete` |
| **Org** | `role_actions` | `role.syncActions` *(permission grant/revoke — most security-relevant)* |
| **Org** | `organization`, `organization_modules` | `organization.create.bootstrap`, `org.modules.update` |
| **Platform** (→ super-admin, `org_id = NULL`) | `module`, `module_tree`, `module_group` | `module.create(.submodule)`, `module.update`, `module.delete`, `module.restore(.submodule)`, `module.reorder` |
| **Platform** | `action`, `action_group` | `action.create`, `action.update`, `action.delete`, `action.reorder`, `action.ensure.default_sidebar` |

> Separate `email_config_audit_logs` table exists (different schema). **Out of scope** for this plan; flag only.

### fe-anandi (`src/pages/UserAccessControl/AuditLog/AuditLog.jsx`)
Working page: target-type dropdown + action search + 6-col table (Date · Actor · Action · Target Type · Target ID · Details) + prev/next paging. Action keys mapped to friendly labels.

**Confirmed bugs to fix:**
1. Filter sends `role_action` / `org`; backend stores `role_actions` / `organization` → "Permission" & "Organization" filters return **0 rows**. Badge map has the same mismatch.
2. Maps `module.*` / `action.*` labels that **can never appear** (those rows are `org_id = NULL`).
3. No date-range filter (despite the `(org_id, created_at)` index existing for exactly this).
4. `before/after` rendered as raw `JSON.stringify` (shows `roleIds:[3,5]`, not names).
5. `orgId` falls back to `|| 1` (hardcoded-org class of bug).
6. `ip`/`user_agent` captured but never shown; `actorId` filter supported by API but no UI.

### super-admin
No page, route, API slice, or nav item. The TableShell/MotionBody/RowActionsMenu pattern (Framer Motion) is the convention to follow.

---

## 2. Information Architecture & Navigation

### Two distinct surfaces, by data ownership
| | **fe-anandi — "Activity Log"** | **super-admin — "Platform Audit"** |
|---|---|---|
| Audience | Org admins (level-6 role; `audit-logs.audit-logs.view`) | Super admins only (no RBAC gating in that app) |
| Data shown | This org's events (`org_id = <org>`) | Platform catalog events (`org_id = NULL`) + cross-org bootstrap feed |
| Nav location | Existing **User Access Control** group, item "Audit Logs" (already seeded) | New top-level **nav item** in `DashboardLayout/navItems.js` → "Audit" / "Activity" |
| Route | `/admin/audit-logs` (exists) | new e.g. `/audit` (or `/activity`) |

### Why the split (not one shared page)
The data is already partitioned by `org_id` at write time. Org admins must not see other orgs' or platform internals; super admins are the only ones who *can* see `NULL`-org events. Building one page that serves both would require role-branching + a backend change that lets one endpoint return both scopes — more complex and leakier than two purpose-built surfaces.

### Cross-links (deep-linking; UX rule `deep-linking`)
- From **fe-anandi → ManageUsers / ManageRoles**: row action "View activity" → audit page pre-filtered by `targetType`+`targetId` (and/or `actorId`). URL-encode filters as query params so the view is shareable and back-restorable (`state-preservation`).
- From **super-admin → Modules / Organizations** detail pages: same "Recent activity" entry point.

---

## 3. Page Layout (shared skeleton, per-app styling)

```
┌───────────────────────────────────────────────────────────────────────┐
│  HEADER                                                                  │
│  Audit Log                                   [ 1,248 entries ]           │
│  Read-only trail of every change. Entries cannot be edited or deleted.   │  ← immutability cue
├───────────────────────────────────────────────────────────────────────┤
│  FILTER BAR (sticky)                                                     │
│  [Date range ▾] [Category ▾] [Target type ▾] [Actor ▾]  🔎 search action │
│                                              [ Clear all ]  [ Export ⤓ ] │
│  active chips:  • Last 7 days ✕   • Role changes ✕                        │
├───────────────────────────────────────────────────────────────────────┤
│  TABLE                                                                    │
│  When ▾    Actor          Action            Target            Changes     │
│  ───────────────────────────────────────────────────────────────────────│
│  2m ago    Asha Mehta     ● Role updated    Role · Counsellor  ▸ 3 fields │
│            asha@x.com        (role.update)                                 │
│  1h ago    System          ● User created   User · #482        ▸ view     │
│  …                                                                         │
├───────────────────────────────────────────────────────────────────────┤
│  FOOTER:   Showing 1–50 of 1,248      ‹ Prev   Page 1 / 25   Next ›       │
└───────────────────────────────────────────────────────────────────────┘
```

- **Header**: title + live total + one-line subtitle stating immutability (sets the mental model that this is a record, not a workspace).
- **Filter bar**: sticky on scroll; active filters shown as removable **chips** below the controls (`progressive-disclosure`, `state-preservation`).
- **Row**: click anywhere → opens **detail drawer** (see §7). The "Changes" cell is a summary affordance, not the full diff.
- **Footer**: result range + pager. fe-anandi keeps its current prev/next; super-admin uses its `TableFooter`/`PagerRow`.

### Density
Comfortable rows (~56px), 2-line actor cell (name + muted email). Tabular figures for time/ids (`number-tabular`). 8px spacing rhythm.

---

## 4. Component Architecture

### 4.1 fe-anandi (custom styled-components; reuse existing primitives)
Keep the existing folder; refactor the monolithic page into composable pieces:

```
src/pages/UserAccessControl/AuditLog/
├── AuditLog.jsx                 ← page shell: state, RTK query, layout, RBAC guard
├── components/
│   ├── AuditFilterBar.jsx       ← date range + category + target + actor + search + chips
│   ├── AuditTable.jsx           ← <WidgetTable> rows; maps logs → AuditRow
│   ├── AuditRow.jsx             ← one <tr>; renders ActionBadge + TargetBadge + summary
│   ├── AuditDetailDrawer.jsx    ← side panel: full metadata + ChangeDiff
│   ├── ChangeDiff.jsx           ← before/after field-level diff renderer
│   ├── ActionBadge.jsx          ← verb-category color + icon + label
│   ├── TargetBadge.jsx          ← target_type badge (CORRECT values)
│   └── states/
│       ├── AuditEmptyState.jsx
│       ├── AuditSkeleton.jsx    ← shimmer rows (not a single spinner)
│       └── AuditErrorState.jsx  ← failure + Retry
├── constants/
│   ├── actionLabels.js          ← ACTION_LABELS + humanizeAction (moved out of page)
│   ├── actionCategories.js      ← action → {category, color, icon}
│   └── targetTypes.js           ← canonical target_type list (fixes the mismatch)
└── hooks/
    └── useAuditFilters.js       ← filter state ↔ URL query-param sync
```
Reused primitives: `WidgetTable`, `CommonHead`, `Flexbox`, `Badge`, `Button`, `SearchBox`, `Dropdown` (Theme/GlobalStyles), `Theme/Typography`, `Theme/Icons`, `ButtonLoader`, `useRBACPermissions`, `useV2ListAuditLogsQuery`.

### 4.2 super-admin (feature-based; styled-components + Framer Motion)
New feature module mirroring `schools`/`organizations`:

```
src/features/auditLogs/
├── application/
│   └── auditLogsApi.js          ← RTK Query slice on baseApi (new endpoint(s))
├── ui/
│   ├── AuditPage.jsx            ← page shell + layout
│   ├── AuditFilterBar.jsx
│   ├── AuditTable.jsx           ← TableShell/Scroller/Table/THead/MotionBody/MotionRow
│   ├── AuditRow.jsx
│   ├── AuditDetailDrawer.jsx
│   ├── ChangeDiff.jsx
│   ├── ActionBadge.jsx
│   └── states/ (Empty / Skeleton / Error)
└── model/
    ├── actionCategories.js      ← platform-event categories (module/action/org)
    └── targetTypes.js
```
Wire-up: add route in `app/routes/AppRoutes.jsx` (under `ProtectedRoute`/`DashboardLayout`), add nav item in `navItems.js`, register `auditLogsApi` tag types in `store.js`.

> **Shared logic, not shared components.** The two apps have different design systems, so components are *not* shared. But `actionLabels`, `actionCategories`, `targetTypes`, and the diff-flattening helper are pure functions — keep them structurally identical in both apps (copy, or extract to a tiny shared package later) so labels/colors stay consistent across portals.

---

## 5. Filter Bar Design

Controls left-to-right (priority order), with active-filter chips beneath:

| Control | Type | Notes |
|---|---|---|
| **Date range** | preset dropdown (Today / 7d / 30d / Custom) | Most-used audit filter. Custom → two date inputs. *(Needs backend `from`/`to` params — see appendix.)* |
| **Category** | dropdown | Friendly grouping over raw actions: All / User changes / Role changes / Permission changes / (super-admin: Module changes / Permission catalog / Org setup). Maps to one-or-more `action` values. |
| **Target type** | dropdown | **Canonical values** — fe-anandi: `user`, `role`, `role_actions`, `organization`, `organization_modules`. super-admin: `module`, `action`, `module_tree`, `module_group`, `action_group`. |
| **Actor** | searchable dropdown | Populated from a "distinct actors in audit logs" source. *(Needs endpoint or reuse users list — appendix.)* |
| **Search action** | text input (Enter to submit, debounced) | Keep existing free-text `action` iLike search as power-user escape hatch. |
| **Clear all** | text button | Resets to defaults; clears chips + URL params. |
| **Export** | button (Phase 3) | CSV of current filter result. |

UX rules applied: `empty-states`, `inline-validation` (date range validity), `debounce-throttle` (search), `state-preservation` (filters in URL), `color-not-only` (chips have text, not just color).

---

## 6. Table vs Timeline

**Decision: data table is primary; timeline is an optional view toggle (Phase 3).**

- **Table** is correct for audit: dense, scannable, sortable, comparable across rows, export-friendly, accessible (`sortable-table`, `data-table`). It matches both apps' existing list conventions.
- **Timeline** (grouped by day, vertical rail) reads nicely for "what happened recently to X" but is poor for scanning thousands of rows. Offer it later as a toggle on a *single-target* filtered view (e.g. "all activity for Role: Counsellor").
- **Columns (table):** When (relative + absolute tooltip) · Actor (name + email) · Action (category-colored badge) · Target (type badge + id/name) · Changes (summary affordance → drawer). Optional secondary columns behind a column toggle: IP, Device.
- **Sorting:** default `created_at DESC`. Allow sort on When (cheap, indexed). Indicate with `aria-sort`.
- **Virtualization:** if a page can exceed ~50 rows in DOM (or you raise the page size / add infinite scroll), virtualize (`virtualize-lists`). At the current 50/page it's optional.

---

## 7. Row Detail Drawer & Before/After Diff

### Detail drawer (side panel, slides from right)
Opens on row click. Sections:
1. **Summary**: action label, category badge, timestamp (absolute + relative).
2. **Actor**: name, email, id; IP; device/user-agent (parsed to "Chrome on Windows" when available).
3. **Target**: type badge + id, resolved name + deep-link to that entity's page when resolvable.
4. **Changes**: the `ChangeDiff` component.
5. **Raw**: collapsible raw JSON (power users / debugging) — keep the current behavior as a fallback, not the default.

Motion: super-admin uses Framer Motion slide-in from trigger source (`modal-motion`, `interruptible`, exit faster than enter). fe-anandi: simple CSS transform/opacity transition 200ms (`duration-timing`, `transform-performance`).

### ChangeDiff — field-level, not raw JSON
Flatten `before`/`after` to a shared key set and render per-field rows:

```
Field            Before              After
──────────────────────────────────────────────
status           active        →     disabled        (changed — amber)
roleIds          [Counsellor]  →     [Counsellor, Admin]   (added — green)
name             —             →     "Asha Mehta"    (set — green)
phone            "9985…"             "9985…"          (unchanged — muted, optionally hidden)
```

Rules:
- **create** (before = null): show all `after` fields as "set/added" (green left-border).
- **delete / disable** (status flips): emphasize the status change (red).
- **update**: show only changed fields by default; "show unchanged" toggle (`progressive-disclosure`).
- **id → name resolution**: `roleIds`/`schoolIds`/`managerIds` etc. shown as names where resolvable; raw id otherwise. *(Resolution source = appendix item.)*
- **Color is never the only signal** (`color-not-only`): pair with a word/icon — "Added", "Removed", "Changed", arrow glyph.
- **Reorder events** (`module.reorder`, `action.reorder`): render as an ordered before→after list, not a field diff.

---

## 8. Action Category & Color System

Two visual dimensions, kept distinct so the eye isn't overloaded:

### A. Action **verb category** → drives the Action badge (semantic, `color-semantic`, `contrast-feedback` ≥4.5:1)
| Category | Actions | Color intent | Icon |
|---|---|---|---|
| **Create** | `*.create`, `*.create.submodule`, `organization.create.bootstrap` | green (success) | plus |
| **Update** | `*.update` | blue (info) | pencil |
| **Delete** | `*.delete` | red (danger) | trash |
| **Permission/Security** | `role.syncActions`, `org.modules.update`, `action.ensure.default_sidebar` | purple (highlight — most audit-sensitive) | shield |
| **Restore** | `*.restore`, `*.restore.submodule` | teal | rotate |
| **Reorder** | `*.reorder` | neutral grey | sort |

### B. **Target type** → subtle outline badge (identity, not severity)
fe-anandi (fixed canonical values): `user` (blue), `role` (orange), `role_actions` (green→"Permission"), `organization` (red→"Org"), `organization_modules` (grey→"Org Modules").
super-admin: `module`, `action`, `module_tree`, `module_group`, `action_group` — neutral palette, labels humanized.

> Keep category colors muted/desaturated for an enterprise admin feel (the design-rule recommendation for admin/dashboard surfaces) — not saturated marketing colors. Verify each pairing ≥4.5:1 in the app's actual theme; define as semantic tokens, no raw hex in components.

---

## 9. States (every one designed, not just the happy path)

| State | fe-anandi | super-admin | Rule |
|---|---|---|---|
| **Loading (initial)** | Shimmer **skeleton rows** (5–8 placeholder rows), not one centered spinner | same via Motion shimmer | `progressive-loading`, `loading-states` |
| **Loading (refetch/page change)** | keep current rows, disable pager, subtle top progress | same | `content-jumping` (reserve space, no jump) |
| **Empty (no data at all)** | icon + "No activity recorded yet" | "No platform activity yet" | `empty-states` |
| **Empty (filters too narrow)** | "No entries match these filters" + **Clear filters** button | same | `empty-states`, `error-recovery` |
| **Access denied** (fe-anandi only) | existing centered "Access Denied" (keep) | N/A (super-admin gated by app) | — |
| **Error / network** | message + **Retry** (RTK refetch) | same | `error-state-chart`/`timeout-feedback`, `error-recovery` |
| **Partial (actor deleted)** | fall back to `actor_email` snapshot, label "(removed user)" | same | graceful degradation |

---

## 10. Audit-specific UX Guidelines (the "feel" of a trail)

1. **Read-only & immutable** — no edit/delete row actions ever. State it in the subtitle. The only row action is "View details" / "View activity for this target".
2. **Time is first-class** — relative time in the cell ("2m ago"), absolute on hover/in drawer; tabular figures; respect locale.
3. **Actor accountability** — always show who; never collapse to just an id. Show "System" explicitly when `actor_id` is null.
4. **Newest first, always** — default `created_at DESC`; don't surprise with other default sorts.
5. **Filters are shareable & restorable** — encode in URL; back button restores them (`state-preservation`).
6. **Security events stand out** — permission/role-grant changes (`role.syncActions`, `org.modules.update`) use the purple "Security" category so they're scannable.
7. **No destructive affordances, no bulk mutate** — bulk *export* is fine; bulk delete/edit is conceptually wrong for an audit trail.
8. **Don't leak across tenants/scope** — fe-anandi shows only its org; super-admin owns platform/cross-org. Enforced by which endpoint each calls.
9. **Accessibility** — keyboard-navigable rows + drawer, `aria-sort` on sortable header, focus moves into drawer on open and returns on close, color never the sole signal, contrast ≥4.5:1, `prefers-reduced-motion` respected on the slide-in.
10. **Performance** — debounce search, paginate (or virtualize), skeletons over spinners, reserve space to avoid layout shift.

---

## 11. Frontend Development Roadmap (no backend dependency until Phase noted)

**Phase 0 — Refactor & correctness (fe-anandi)** *(no backend change)*
- Extract page into the component tree in §4.1.
- Fix target-type values (`role_actions`, `organization`, add `organization_modules`); fix badge map.
- Remove platform-only labels from the org page (or guard them out).
- Replace `orgId || 1` with proper org from auth state.
- Skeleton + proper empty/error/denied states.

**Phase 1 — Diff & detail (fe-anandi)** *(no backend change)*
- `ChangeDiff` field-level renderer + detail drawer.
- Action category color system + `ActionBadge`/`TargetBadge`.

**Phase 2 — super-admin Platform Audit (new)** *(needs backend: platform/cross-org read — appendix #1)*
- New `features/auditLogs` module, route, nav item, RTK slice.
- Table (Motion), filter bar, diff, drawer, states.

**Phase 3 — Filters & polish (both)** *(needs backend: date range, actor list, name resolution — appendix #2–#4)*
- Date-range filter + category dropdown + actor dropdown + filter chips + URL sync.
- CSV export.
- Optional timeline toggle on single-target views; deep-links from Users/Roles/Modules pages.

---

## 12. Appendix — Data-contract gaps to raise with backend (LATER, per your note)

The frontend design above assumes these backend capabilities. None block Phase 0–1; they gate Phase 2–3.

1. **Platform / cross-org read for super-admin** — current endpoint hard-filters `org_id = Number(orgId)`, so `NULL`-org (platform) and cross-org feeds are unreachable. Need a super-admin endpoint that returns `org_id IS NULL` (and/or all orgs), e.g. `GET /api/v2/platform/audit-logs` or an `orgId=all` / `scope=platform` mode.
2. **Date-range params** — add `from`/`to` (or `dateFrom`/`dateTo`) to the list endpoint; the `(org_id, created_at)` index already supports it efficiently.
3. **Actor list** — endpoint returning distinct actors present in audit logs (or reuse the users list) to populate the actor filter.
4. **Target name resolution** — either backend returns a resolved `targetName` / nested ids→names, or frontend resolves via existing user/role/module queries. Decide ownership.
5. **Return `user_agent`** (and `org_id` for super-admin) in the row payload — captured but currently dropped by the controller mapper.
6. **Filter-value contract** — agree the canonical `target_type` strings and expose the distinct `action`/`target_type` enums (endpoint or shared constant) so dropdowns never drift from DB reality again (this is the root cause of the current filter bug).
7. **(Optional) email-config audit** — `email_config_audit_logs` is a separate silo; decide whether it ever surfaces in either portal.

---

## 13. Open Design Decisions (need your call before/where noted)

- **D1 — super-admin nav label & route**: "Audit", "Activity", or "Platform Audit"? Route `/audit` vs `/activity`?
- **D2 — Timeline view**: build the optional timeline toggle (Phase 3) or table-only?
- **D3 — Category taxonomy**: confirm the friendly category groupings in §5/§8 match how your team talks about these changes.
- **D4 — Export**: CSV only, or also a printable/scoped report?
- **D5 — Name resolution ownership** (appendix #4): backend-resolved vs frontend-resolved.
```
<!--  -->