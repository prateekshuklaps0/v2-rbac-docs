# Audit Logs — Backend API Contract (for the super-admin audit page)

> Scope of this doc: the **backend APIs** for the super-admin route
> `/organizations/:orgSlug/audit-logs` (and its `/user/:userParam` sub-page).
> API → UI binding is to be done by the frontend dev. This document is the
> contract + the exact frontend changes required.
>
> Author: backend work completed 2026-06-11. Verified against the dev DB.

---

## 0. TL;DR

- The super-admin audit page is currently **100% mock data** (`super-admin/src/features/organizations/ui/mockAuditLogs.js`). Its shape (`status` success/warning/failure, `cluster`, action categories like Lead / Payment / Email) is **fictional** — none of it exists in the real `audit_logs` table.
- The real `audit_logs` table only records **RBAC + org-config + platform-catalog** events. For an org page (Masters Union = **org_id 12**) the events are: user create/update/delete, role create/update/delete, role permission sync, and org module/permission access updates.
- **No new table or migration is needed.** The read endpoint already existed; it has been **extended (backward-compatibly)** and a second endpoint added for the user sub-page.
- Decision taken with product: **map honestly to real data** (drop status/cluster, derive category + summary + before→after diff), **this org only** (`org_id = :orgId`), and a **real actor timeline** for the user sub-page.

---

## 1. What changed in the backend

| File | Change |
|---|---|
| `be-anandi/src/v2/utils/auditLogPresenter.js` | **NEW** — pure helpers: category map, action labels, one-line summary, field-level diff, row presenter. |
| `be-anandi/src/v2/controllers/auditLogController.js` | `listAuditLogs` extended (new optional filters + richer rows). Added `getActorActivity`. |
| `be-anandi/src/v2/routes/rbacAudit.routes.js` | Added `GET /audit-logs/actor-activity` (specific route declared before the list route). |

Nothing else was touched. No DB migration. The two endpoints below are mounted under `/api/v2/org/:orgId/rbac`.

### Backward compatibility (fe-anandi)
`GET /api/v2/org/:orgId/rbac/audit-logs` is **also consumed by fe-anandi**
(`fe-anandi/src/pages/UserAccessControl/AuditLog/AuditLog.jsx`). The change is a
**strict superset**:
- Every original row field is preserved: `id`, `actor{id,name,email}`, `targetType`, `targetId`, `action`, `before`, `after`, `ip`, `createdAt`.
- Every original query param behaves identically: `page`, `limit`, `targetType` (exact), `action` (contains), `actorId`.
- New params are all optional; new row fields are additive. fe-anandi ignores them and keeps working unchanged.

---

## 2. Authentication & Authorization

- All v2 routes require a JWT (`Authorization: Bearer <token>`) — `jwtValidator`.
- The endpoints are gated by RBAC action **`audit-logs.audit-logs.view`**, BUT the
  RBAC middleware **bypasses for `role === 'super_admin'`**
  (`be-anandi/src/v2/middleware/rbacMiddleware.js`). So a super-admin token is
  automatically authorized — no extra wiring needed in the super-admin app.
- The super-admin app already sends the bearer token via its RTK base query
  (`super-admin/src/infrastructure/api/baseApi.js`, `prepareHeaders`).

---

## 3. Endpoint 1 — List org audit logs

```
GET /api/v2/org/:orgId/rbac/audit-logs
```

Org-scoped (always `org_id = :orgId`), paginated, newest first.
For Masters Union, `:orgId` = **12** (resolve the slug → id via the existing
`useOrgIdFromSlug()` hook, which already maps `internalId` → `id`).

### Query params (all optional)

| Param | Type | Default | Notes |
|---|---|---|---|
| `page` | int | 1 | |
| `limit` | int | 50 | capped at 200 |
| `category` | string | — | Friendly group → expands to target_type(s). See §5. |
| `targetType` | string | — | Exact match. Takes precedence over `category`. |
| `action` | string | — | Case-insensitive **contains** on the raw action key. |
| `actorId` | int | — | Filter by acting user id. |
| `actorEmail` | string | — | Case-insensitive **contains** on actor email. |
| `search` | string | — | Free text; OR across `actor_email`, `action`, `target_id`. |
| `from` | ISO date | — | Inclusive lower bound on `created_at`. Invalid → **400**. |
| `to` | ISO date | — | Inclusive upper bound on `created_at`. Invalid → **400**. |

### Response `200`

```jsonc
{
  "success": true,
  "data": [
    {
      "id": "336",
      "actor": { "id": 4441057, "name": "Test Admin forty six", "email": "test@admin46.com" },
      "actorName": "Test Admin forty six",   // convenience mirror of actor.name
      "actorEmail": "test@admin46.com",       // convenience mirror of actor.email
      "targetType": "user",
      "targetId": "4542573",
      "action": "user.create",
      "actionLabel": "User Created",          // human label (derived)
      "category": "user",                     // canonical category key (derived)
      "categoryLabel": "User",                // human category label (derived)
      "summary": "Created user sunny.singh@mastersunion.org",  // one-line "resource" (derived)
      "before": null,                          // raw JSONB
      "after": { "email": "…", "roleIds": [39], "...": "…" },   // raw JSONB
      "changes": [                             // field-level diff (derived)
        { "field": "email", "before": null, "after": "sunny.singh@mastersunion.org", "type": "added" }
        // type ∈ "added" | "removed" | "changed"; only changed fields included
      ],
      "ip": "::ffff:127.0.0.1",
      "userAgent": "Mozilla/5.0 …",            // now returned (was dropped before)
      "createdAt": "2026-06-10T11:05:21.095Z"
    }
  ],
  "pagination": { "page": 1, "limit": 50, "total": 120, "pages": 3 }
}
```

- `actor.name`/`actor.email` are `null` when the acting user no longer exists; the
  email snapshot stored on the row is still surfaced via `actorEmail` when present.
- `actor.id` / `actorEmail` are `null` for **system** events (no actor) — render as "System".
- `createdAt` is UTC ISO. Format client-side (the mock's `timestamp` string is replaced by this).

---

## 4. Endpoint 2 — Actor activity (the `/user/:userParam` sub-page)

```
GET /api/v2/org/:orgId/rbac/audit-logs/actor-activity?actorId=<id>
GET /api/v2/org/:orgId/rbac/audit-logs/actor-activity?email=<email>
```

Returns one actor's profile + stats + paged timeline **within this org**.
Provide **either** `actorId` (preferred) **or** `email` (the super-admin UI
currently uses the email as its user key, so pass `email`). Also supports
`page` (1) and `limit` (50, max 200) for the timeline.

### Response `200`

```jsonc
{
  "success": true,
  "data": {
    "actor": {
      "resolved": true,              // false when the user row no longer exists
      "id": 4441049,
      "name": "MU Admin",
      "email": "admin@mastersunion.org",
      "role": "admin",               // AUTH role enum (admin/super_admin/...), NOT the RBAC role name
      "status": "active",
      "memberSince": "2026-04-03T07:09:54.406Z",  // users.created_at
      "lastActive": "2026-05-20T...Z"             // users.last_logged_in_at (may be null)
    },
    "stats": {
      "total": 12,
      "byCategory": [
        { "category": "user", "label": "User", "count": 7 },
        { "category": "permission", "label": "Permission", "count": 3 }
      ],
      "firstActivity": "2026-05-11T07:02:50.703Z",
      "lastActivity":  "2026-05-22T07:43:35.557Z"
    },
    "timeline": [ /* same row shape as Endpoint 1's `data[]` */ ],
    "pagination": { "page": 1, "limit": 50, "total": 12, "pages": 1 }
  }
}
```

### Error cases
- Neither `actorId` nor `email` → **400** `"actorId or email is required"`.
- No matching user **and** no events in this org → **404**.

> Honest-data note: there is **no department, "joined" month label, or
> success/warning/failure stats** in the audit data. `department` does not exist
> (render "—" or remove). The status-count cards in the mock profile
> (`Successful / Warnings / Failed`) have no backing data — replace them with the
> `byCategory` breakdown or with `total` only.

---

## 5. Event vocabulary (the real data)

### Categories (`category` param ↔ `target_type`)

| `category` | Label | DB `target_type`(s) | Org page? |
|---|---|---|---|
| `user` | User | `user` | ✅ |
| `role` | Role | `role` | ✅ |
| `permission` | Permission | `role_actions` | ✅ |
| `module_access` | Module Access | `organization_modules` | ✅ |
| `organization` | Organization | `organization` | ✅ |
| `module_catalog` | Module Catalog | `module`, `module_tree`, `module_group` | platform only (org_id NULL) |
| `permission_catalog` | Permission Catalog | `action`, `action_group` | platform only (org_id NULL) |

For an org page you will only ever see the first five. (The platform categories
have `org_id = NULL` and are out of scope for this per-org page — see §7.)

### Action → label (also returned as `actionLabel`)

`user.create` → User Created · `user.update` → User Updated · `user.delete` → User Deleted ·
`user.attachToOrg` → User Attached to Organization · `role.create/update/delete` → Role … ·
`role.syncActions` → Role Permissions Updated · `org.modules.update` → Organization Modules Updated ·
`organization.create.bootstrap` → Organization Created.
Unknown keys fall back to a Title-Cased humanization.

---

## 6. Frontend changes required (super-admin) — for the binding dev

All in `super-admin/src/features/organizations/`.

1. **Add an RTK Query slice** (e.g. `application/auditLogsApi.js`) on `baseApi`:
   - `listAuditLogs({ orgId, page, limit, category, targetType, action, actorEmail, search, from, to })`
     → `GET /api/v2/org/${orgId}/rbac/audit-logs`, `transformResponse: r => ({ rows: r.data, pagination: r.pagination })`.
   - `getActorActivity({ orgId, email /* or actorId */, page, limit })`
     → `GET /api/v2/org/${orgId}/rbac/audit-logs/actor-activity`, `transformResponse: r => r.data`.
   - Register a new tag type `'AuditLog'` in `baseApi.js` `tagTypes` and `providesTags` on both.
   - Pass params via the `params:` key so `baseQueryWithCleanedParams` strips the empty ones.

2. **`OrgAuditLogsPage.jsx`** — replace `mockAuditLogs` with the query. Field mapping:

   | Mock field (delete) | Real source |
   |---|---|
   | `entry.timestamp` (string) | `row.createdAt` (ISO → format client-side) |
   | `entry.user` | `row.actorEmail` (fallback `"System"` when null) |
   | `entry.actionType` / `entry.actionLabel` | `row.category` / `row.actionLabel` |
   | `entry.resource` | `row.summary` |
   | `entry.ip` | `row.ip` |
   | `entry.status` (success/warning/failure) | **does not exist — remove the Status column & chip** |
   | `entry.cluster` | **does not exist — remove "Environment" from the drawer** |
   | `entry.payload` | `row.before` / `row.after` / `row.changes` (use the diff; keep raw JSON as a collapsible) |
   | `MOCK_TOTAL_ENTRIES` | `pagination.total` |

   - Wire the existing **User** text input → `search` (or `actorEmail`) param; the
     **Action type** select → `category` param (re-point its options to §5 keys); the
     **Date range** trigger → `from`/`to`. Move filtering/pagination server-side
     (it is currently client-side over the mock array).

3. **`AuditEntryDrawer.jsx`** — remove the **Environment** (cluster) and **Status**
   fields. Render `row.changes` as a before→after diff; keep `before`/`after` raw
   JSON as a collapsible "Raw payload". `Event ID` → `row.id`.

4. **`OrgAuditUserProfilePage.jsx` / `AuditUserProfile.jsx`** — replace
   `getAuditUserProfile` / `getAuditEntriesForUser` with `getActorActivity`.
   Map `actor.name/email/role/status/memberSince/lastActive`; drop `department`
   ("—"); replace the success/warning/failure stat cards with `stats.byCategory`
   (or `stats.total`). The timeline list maps from `data.timeline` using the same
   field mapping as the table.

5. **Delete `mockAuditLogs.js`** once the above is wired (keep the small pure
   helpers `encode/decodeAuditUserParam`, `getAuditUserProfilePath`,
   `isAuditUserClickable`, `PAGE_SIZE` — move them to a non-mock module).

---

## 7. Notes / future (not built — out of scope per product decision)

- **Platform feed** (the 128 `org_id = NULL` catalog events) is intentionally NOT
  surfaced on the per-org page. If a super-admin "Platform Audit" page is wanted
  later, it needs a separate endpoint that filters `org_id IS NULL` (the current
  endpoint hard-scopes to the path's `:orgId`). The presenter/category map already
  supports the `module_catalog` / `permission_catalog` categories for that.
- **Distinct-actor dropdown**: not built (the UI's actor filter is free-text →
  use `search`/`actorEmail`). Add a `GET …/audit-logs/actors` endpoint later if a
  populated dropdown is desired.
- **CSV export**: not built.
- `role` in actor profile is the auth-role enum, not the friendly RBAC role name.
  Resolving RBAC role names per actor would need an extra join — add later if needed.
```
