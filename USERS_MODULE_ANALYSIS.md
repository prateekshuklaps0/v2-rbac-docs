# Users / Roles / Permissions Module — Deep Analysis
> Maintained by AI Agent | Last Updated: 2026-09-13 (Implementation In Progress)
> Branch: `users` | All new work → `v2` folder (backend) and `pages/Admin/v2/` (frontend)

---

## TABLE OF CONTENTS

1. [Project Overview & Full Architecture](#1-project-overview--full-architecture)
2. [Backend: Complete System Map](#2-backend-complete-system-map)
3. [Frontend: Complete System Map](#3-frontend-complete-system-map)
4. [Frontend ↔ Backend Data Flow](#4-frontend--backend-data-flow)
5. [Current Module: How It Actually Works](#5-current-module-how-it-actually-works)
6. [Full Bug & Weakness Inventory](#6-full-bug--weakness-inventory)
7. [Reusable vs Rebuild Decision](#7-reusable-vs-rebuild-decision)
8. [Research: How Top CRMs Handle This](#8-research-how-top-crms-handle-this)
9. [Proposed Feature List for v2](#9-proposed-feature-list-for-v2)
10. [Recommended Architecture for v2](#10-recommended-architecture-for-v2)
11. [Roadmap: Step-by-Step Rebuild Plan](#11-roadmap-step-by-step-rebuild-plan)
12. [Open Questions for Prateek](#12-open-questions-for-prateek)
13. [Progress Log](#13-progress-log)
14. [Stage & Sub-Stage Permissions (Per-Role Instance Scope)](#14-stage--sub-stage-permissions-per-role-instance-scope--2026-06-17)
15. [Data Visibility — Whose Records You See](#15-data-visibility--whose-records-you-see--2026-08-06)
16. [Auth Storage & RBAC Request Performance](#16-auth-storage--rbac-request-performance--2026-08-12)
17. [`users.role` — Why It Is Neither an Authority Nor an Index](#17-usersrole--why-it-is-neither-an-authority-nor-an-index--2026-08-28)
18. [One Email, Several `users` Rows — Identity, Duplicates & Super-Admin Accounts](#18-one-email-several-users-rows--identity-duplicates--super-admin-accounts--2026-09-13)

---

## 1. Project Overview & Full Architecture

### Tech Stack
| Layer | Technology |
|---|---|
| Backend | Node.js + Express (ES Modules), PostgreSQL via Sequelize v6 |
| Frontend | React 18 + Vite, Redux Toolkit + RTK Query, Ant Design + MUI |
| Auth | JWT (Bearer token) + httpOnly cookie fallback, OTP login (Phone/Email/WhatsApp) |
| Queues | BullMQ + Redis |
| Email | SendGrid |
| SMS/WhatsApp | MSG91, Karix |
| File Storage | AWS S3 |
| Infra | AWS RDS (PostgreSQL), staging + production DB |

### Overall Project Structure
```
Anandi/
├── be-anandi/src/
│   ├── server.js               ← entry point (Express + Socket.IO)
│   ├── config/
│   │   ├── env.js              ← environment variables with defaults
│   │   └── db.js               ← Sequelize connection (PostgreSQL, SSL on RDS)
│   ├── models/                 ← 80+ Sequelize models
│   ├── controllers/
│   │   ├── rbac/               ← current RBAC module (to be rebuilt in v2)
│   │   ├── users/              ← auth controller (login, OTP, password reset)
│   │   ├── org/                ← org-level controllers
│   │   └── public/             ← public (no-auth) controllers
│   ├── routes/
│   │   ├── org/
│   │   │   ├── rbac.routes.js  ← main RBAC routes (mounted at /:orgId/rbac)
│   │   │   └── role.routes.js  ← feature-based role routes (SEPARATE system)
│   │   └── _index.route.js     ← main API router
│   ├── middleware/             ← auth, feature access, rate limit, etc.
│   ├── v2/                     ← NEW v2 API (leads, orgs, schools — NO RBAC YET)
│   ├── migrations/             ← .cjs format, YYYYMMDDHHMMSS-desc.cjs
│   ├── services/               ← email, notification services
│   └── utils/                  ← crypto, responses, errors, getModels
│
└── fe-anandi/src/
    ├── Routes/MainRoutes.jsx   ← THE ONLY active routes file
    ├── App.jsx                 ← layout wrapper, login check
    ├── Layout/
    │   ├── NewSidemenuV2.jsx   ← ACTIVE sidebar (RBAC-driven)
    │   ├── Sidemenu.jsx        ← DEPRECATED (not used)
    │   └── SidebarMenus.jsx    ← DEPRECATED static menu config
    ├── Redux/
    │   ├── store.js            ← 23 API slices registered
    │   ├── Slices/
    │   │   ├── authSlice.js    ← login state (persisted to localStorage)
    │   │   └── rbacSlice.js    ← RBAC state (NOT persisted, fresh each session)
    │   └── Services/
    │       └── rbacService.js  ← RTK Query RBAC endpoints
    ├── hooks/
    │   ├── useRBACPermissions.js ← NEW permission hook (used in 24 files)
    │   └── usePermissions.js    ← OLD permission hook (legacy, still coexisting)
    ├── pages/Admin/UserManagement/ ← all user/role/permission pages
    └── SideFilter/             ← side panel forms (CreateUserV2, CreateRoleV2)
```

### Multi-Tenant Design
- `Organization` is the top-level tenant
- `OrgUser` bridges User ↔ Organization with org-specific role/feature metadata
- `x-org-id` header or `:orgId` URL param identifies tenant in every request
- `School` is a sub-entity inside an Organization (education-domain specific)
- Routes mounted at `/api/org/:orgId/rbac/...` and `/api/v2/org/:orgId/...`

### Implementation Conventions
- For this RBAC rebuild, **DB column names and Sequelize model field names must use the same case/style**.
- For this RBAC rebuild, **new DB table names will follow the current implemented `snake_case` convention**.
- Sequelize model class names may stay in `PascalCase`, but **table names must match the actual implemented migration/code names**.
- Column names and model field names should also match directly in case/style for new RBAC work.
- Do **not** introduce new case-mapping layers for fresh work.
- Existing legacy mismatches may remain only where backward compatibility requires them, but they are not the standard for new RBAC code.
- When this document describes the **current live/legacy system**, it may still reference existing `snake_case` table names because that is the reality of the current codebase/DB.
- When this document describes the **new RBAC rebuild / forward-looking design**, it should use the same `snake_case` table names that exist in code/migrations.
- The repo-wide Sequelize default still uses `underscored: true`, and the current RBAC Phase 1 implementation also follows that actual table naming direction.
- Progress logs, file names, and already-created migration names must remain **factual** and aligned with the implemented code.
- For service and middleware modules in this rebuild, define functions as local `const` functions and **export a single default object** containing those functions.
- Import those modules via the default object pattern, for example: `import rbacService from '../services/rbacService.js'` and then call `rbacService.getUserRbacContext(...)`.
- Avoid mixing this pattern with scattered named exports in new RBAC service/middleware files unless there is a strong reason.
- **Documentation discipline for this file:** Only record changes here if they materially affect architecture, API/frontend contract, data model, authorization flow, routing, migration strategy, handoff state, or testing expectations. Do **not** add every small UI tweak or low-impact bug fix. This file is a continuation spec and decision log, not a full changelog.
- **Developer FAQ discipline:** Keep the RBAC developer FAQ `.docx` in sync with this document. If a change modifies a documented flow, contract, permission formula, login/role-selection behavior, debugging path, or developer-facing workflow, update the matching FAQ answer. If a new flow is introduced and future developers may reasonably ask about it, add a new FAQ question with the answer, code snippet, or visual flow needed to understand it.

---

## 2. Backend: Complete System Map

### server.js
- Express + HTTP server + Socket.IO (namespace: `/karix`)
- Middleware order: Helmet → CORS → Morgan → body-parser → cookie-parser
- **Rate limiting DISABLED** (`generalLimiter` commented out)
- **Request logging DISABLED** (`requestLogger` commented out)
- CORS allows ALL origins (no whitelist)
- Bull Dashboard mounted at `/bull`
- Health check: `GET /health`

### Route Mounts (server.js)
```
/api/v2/public/*    → v2 public routes (no auth)
/api/v2/org/*       → v2 org routes (jwt required globally)
/api/               → v1 routes (_index.route.js)
  /org/:orgId/rbac  → rbac.routes.js   (main RBAC)
  /org/:orgId/roles → role.routes.js   (feature-access roles — SEPARATE system)
  /users            → user auth routes (login, OTP, forgot/reset password)
  /super_admin      → super admin routes
  /keys             → API key management
  /public           → public routes
  ...and 15 more
```

### Authentication System (controllers/users/authController.js)
- **adminLoginV2**: Main login flow. Returns user, JWT, userRoles, userOrganization
  - Checks `users.role` ENUM (not UserRoles junction)
  - **MASTER_PASSWORD bypass**: If `env.MASTER_PASSWORD` set, any password works for ANY user — critical backdoor
  - Sets httpOnly JWT cookie (24h, secure+sameSite:none in production)
  - JWT payload: `{ userId, name, email, phone, role, organizationId }`
  - JWT default expiry: **7 days** (no refresh token mechanism)
- **studentLogin**: Restricted to role='student', fetches Lead + Facebook CAPI event
- **forgotPassword / resetPassword**: Token-based flow (exists and works)
- **OTP Flows**: Phone OTP, Email OTP, WhatsApp OTP (complete flows exist)

### JWT Generation (utils/crypto.js)
```javascript
generateToken(payload)  // jwt.sign, default 7d expiry
verifyToken(token)      // jwt.verify, throws on invalid
hashPassword(password)  // bcrypt with env.BCRYPT_ROUNDS (default 10)
comparePassword(p, h)   // bcrypt.compare
```

### Environment Config (config/env.js) — KEY OBSERVATIONS
```
JWT_SECRET       → defaults to 'default-dev-secret-change-me' (insecure)
SESSION_SECRET   → defaults to 'default-session-secret-change-me' (insecure)
MASTER_PASSWORD  → backdoor: if set, skips bcrypt and authenticates anyone
BCRYPT_ROUNDS    → default 10
DB_HOST/USER/NAME/PASSWORD → dev DB
PROD_DB_*        → production DB (fallbacks to dev if not set)
```

### Database Config (config/db.js)
- Sequelize dialect: PostgreSQL
- SSL: auto-enabled for AWS RDS hostnames
- Connection pool: 10 max (prod), 5 max (dev)
- Global repo convention: `underscored: true` (legacy/default Sequelize behavior across much of the existing codebase)
- Current RBAC rebuild implementation also follows this actual `snake_case` table naming direction in the DB/migrations
- Timestamps: enabled everywhere (created_at, updated_at)

---

## 3. Database Models (Complete)

### Core RBAC Models

#### User (models/User.js)
```javascript
id, email (unique), passwordHash, 
role: ENUM('admin','super_admin','student','counsellor','user'),  // LEGACY
subRole: STRING,       // purpose unclear
status: ENUM('active','disabled','invited'),
name, phone, countryCode, countryIso, image,
organizationId (FK),   // direct org link — legacy, before OrgUser existed
schoolId (FK),         // direct school link — legacy
superAdminFeatures: ARRAY,
loginCount, lastLoggedInAt,
tempId                 // purpose unclear
// ASSOCIATIONS:
BelongsToMany: Role (through UserRoles), School (through UserSchools)
BelongsTo: Organization, School
HasMany: [30+ associations]
```

#### Role (models/Role.js)
```javascript
id, name, internalId (generated from name, lowercase+underscores),
level: INTEGER,         // 1-6, 6=highest org-level role
features: ARRAY,        // role-level feature flags
description: TEXT,
isActive: BOOLEAN,
orgId (FK),
createdBy (FK to User),
// UNIQUE CONSTRAINT: (orgId, internalId)
// ASSOCIATIONS:
BelongsTo: Organization, User (creator)
BelongsToMany: User (through UserRoles), Actions (through RoleActions)
```

#### OrgUser (models/OrgUser.js)
```javascript
id, orgId (FK), userId (FK),
orgRole: STRING,        // plain string, NOT FK to roles table — legacy dual system
features: ARRAY,        // per-user per-org feature override
invitedBy (FK to User),
status: ENUM('invited','active','invite_rejected','removed'),
invitationLink,         // invite flow partially built here
displayName, displayEmail,
isPrimary: BOOLEAN      // identifies primary org for user
// UNIQUE CONSTRAINT: (orgId, userId)
```

#### UserRoles (models/userRoles.js)
```javascript
id (BIGINT), userId (BIGINT FK), roleId (BIGINT FK),
isActive: BOOLEAN
// UNIQUE CONSTRAINT: (userId, roleId)
```

#### Modules (models/modules.js)
```javascript
id (BIGINT), parentId (FK self),  // tree structure
internalId (unique), key (unique),
name, label, description, icon, isBeta, isActive, version,
config: JSONB,    // stores route(s) used by sidebar
sortOrder: INTEGER
// BelongsTo: parent Module
// HasMany: submodule Modules, Actions
```

#### Actions (models/actions.js)
```javascript
id (BIGINT), moduleId (FK),
key (unique),            // e.g., "manage-leads.manageLeads.edit"
name, label, description, icon,
actionType: ENUM('page','table','row','custom'),
group: STRING,
config: JSONB,           // optional action route metadata: { route, routeParams }
sortOrder: INTEGER, isActive: BOOLEAN
// BelongsTo: Modules
// BelongsToMany: Role (through RoleActions)
```

#### RoleActions (models/roleActions.js)
```javascript
id (BIGINT), roleId (BIGINT FK), actionId (BIGINT FK),
isActive: BOOLEAN, updatedBy (BIGINT FK to User)
// UNIQUE CONSTRAINT: (roleId, actionId)
// This is the ACTIVE permission store
```

#### permissions + rolePermissions (models/permissions.js, rolePermissions.js)
```
Status: APPEAR UNUSED — superseded by RoleActions
permissions: { id, moduleId, actionId, key ("leads.edit"), isActive }
rolePermissions: { id, roleId, permissionId, isActive }
These are NOT being used by any controller actively.
```

#### Organization (models/Organization.js)
```javascript
id, name, internalId (unique slug),
features: ARRAY,           // base feature flags
featuresOverride: JSONB,   // { toAdd: [], toRemove: [] } — dynamic override
limitations: JSONB,         // per-org usage limits (mostly unused)
contactEmail, contactPhone, logo, registeredAt, registeredBy
```

#### School (models/schools.js)
```javascript
id, orgId (FK), name, address (JSONB),
contactEmail, contactPhone, description,
isActive: BOOLEAN, createdBy, updatedBy
```

---

## 4. Backend: RBAC Controllers (Complete)

### usersController.js
| Function | What it does | Key issues |
|---|---|---|
| `createUser` | Creates user + UserRoles + UserSchools, sends welcome email | No transaction, password in response, countryIso hardcoded 'IN', school required |
| `getAllUsers` | Paginated user list filtered by org | Uses req.user.organizationId (not URL param), Op.like not iLike (case-sensitive), sortBy/sortOrder ignored |
| `getUser` | Single user with roles + schools | Minimal formatting issues |
| `updateUser` | Transactional update of user + roles + schools | Hard-deletes existing UserRoles/UserSchools instead of soft-delete |
| `deleteUser` | Soft delete (status='disabled') + deactivates UserRoles | Works correctly |
| `getSchools` | List active schools with org filter | Works correctly |

### rolesController.js
| Function | What it does | Key issues |
|---|---|---|
| `createRole` | Creates role (schoolId required, orgId from school) | schoolId mandatory, level check missing |
| `getAllRoles` | All active roles | **No org scoping at all** — returns all orgs' roles |
| `getRole` | Role + all its actions | Works |
| `updateRole` | Updates role + optional action sync | Level-based auth (only here, not in create). schoolId required even for simple renames |
| `deleteRole` | Soft delete + deactivate UserRoles | Works |
| `getRoleActions` | Returns actionIds array for UI | Works |
| `syncRoleActions` | Full action replacement | No transaction |
| `migrateUserRoles` | Migrates users.role enum → user_roles table | **JWT commented out — fully unprotected** |

### majorRbacController.js (the critical one)
| Function | What it does | Used by |
|---|---|---|
| `getAllControls` | Builds full module+action tree for a user → frontend stores in Redux | NewSidemenuV2, every hasAction() call |
| `getAllModulesWithActions` | All modules+actions (no user filter) | CRUDPermissions admin tool |
| `getUserModulesWithActions` | Same as getAllControls but without user/roles in response | Some admin pages |
| `getRoleModulesWithActions` | All modules+actions for a specific role | ModuleWisePermissionV2 |
| `getRolesOverview` | Summary: userCount, actionCount, moduleCount per role | ManagePermissionsV2 |

**getAllControls response shape (CRITICAL — frontend depends on this exactly):**
```javascript
{
  user: { id, email, name },
  roles: [{ roleId, name, internalId, is_active, assignedAt }],
  roleCount: number,
  modulesTree: [              // hierarchical tree — drives the entire sidebar
    {
      moduleId, key, name, label, icon, config,
      actions: [{ actionId, key, actionType, ... }],
      children: [...]         // submodules
    }
  ],
  flatModules: [...],         // flat list of all modules
  flatActions: [...]          // flat list of all actions user has
}
```

### basicRbacController.js
- `getAllModules()` — hierarchical module tree (parent → submodule → actions)
- `getAllActions()` — all active actions with module info
- `getActionsByModuleId()` — actions for a module + submodules

### devRbacController.js
- Protected by `devOnlyMiddleware` (`env.rbacMode === "crud"`)
- CRUD for modules and actions (create, update, delete)
- Auto-fixes PostgreSQL sequence errors on insert
- Should be disabled in production

---

## 5. Backend: Middleware (Complete)

### jwtValidator.js
**Token extraction:** Authorization header → cookie → skips if x-api-key present
**User loading:** User table → fallback to Parent table (for OTP login, marks isParent=true)
**Organization resolution (in order):**
1. x-org-id header or :orgId param
2. OrgUser primary membership (if no org context provided)
3. Legacy user.organizationId fallback
**Membership validation:** For non-super_admins, active OrgUser record required (legacy fallback: user.organizationId)
**Counsellor name:** Extra DB query to Counsellor table to fetch display name
**req.user set to:** `{ id, email, name, role, superAdminFeatures, status, isParent, organizationId, userId }`

### featureAccessValidator.js
**Two-level check:**
1. `org.features` (with toAdd/toRemove override) must include required features
2. `orgUser.features` (user-level) must include required features (unless wildcard `*`)
**Attaches to req:** `req.org`, `req.orgUser`, `req.effectiveFeatures`
**Note:** This is the OLD authorization system. NOT connected to module/action RBAC.

### roleValidator.js
Simple check: `req.user.role` must be in allowedRoles array
Checks the legacy `users.role` ENUM column

### Other middleware
- `apiKeyValidator.js` — x-api-key header
- `rateLimit.js` — exists but disabled
- `error.js` — catches ApiError and unhandled errors
- `cronAuth.js` — x-cron-secret for cron jobs

---

## 6. Frontend: Complete System Map

### App.jsx
- Renders Header + NewSidemenuV2 + MainRoutes when logged in
- Static route exclusions: `/`, `/reset-password`, etc.
- Token-based guard (isLoggedIn from authSlice)

### MainRoutes.jsx
Key user management routes:
```
/admin/manage-users                     → ManageUsersV2 (ACTIVE)
/admin/manage-role-and-permissions      → ManagePermissionsV2 (ACTIVE)
/admin/manage-role-and-permissions/:id  → ModuleWisePermissionV2 (ACTIVE)
/admin/manage-role                      → ManageRoles
/admin/crud-permssions                  → CRUDPermissions (dev tool)
/admin/create-user                      → CreateUser (LEGACY, not used)
/admin/edit-user                        → EditUserDetails (LEGACY, not used)
```

### Redux Store (store.js)
- 23 RTK Query API slices registered
- **Only `authSlice` is persisted** to localStorage (via redux-persist)
- `rbacSlice` is NOT persisted — fetched fresh on every session
- **unauthorizedMiddleware**: Any 401 response → full logout (clears all state, cookies, localStorage, sessionStorage, IndexedDB) → redirect to `/`

### authSlice.js — State Shape
```javascript
{
  token, isLoggedIn, isAdmin, isCounsellor, isPublisher,
  studentId, userName, userEmail, userMobileNumber, image, IP,
  organizationId,    // primary org identifier (extracted from login response)
  organizationName, orgInternalId, organization,
  schoolDetails, selectedSchoolData,
  userRoles, roles: { policies: {}, permissions: [] },
  paymentsData: { isPaymentSuccess, currency, amount, method, transactionID }
}
```

**Organization extraction priority (SetLoginData):**
1. `payload.organization`
2. `payload.organizationId + payload.orgInternalId`
3. `userRoles[0].organisation`
4. `payload.user.organizationId`

### rbacSlice.js — State Shape
```javascript
{
  raw: null,
  modules: [],
  allActions: [],
  fullModuleTree: [],
  userPermissions: {
    modulesTree: [],      // drives NewSidemenuV2 and all hasAction() checks
    flatModules: [],
    flatActions: [],
    user: null,
    roles: []
  },
  actionKeyMap: {}        // { "module.sub.action": actionObject }
}
```
Built by: `setUserPermissions(data)` reducer — called automatically when `getControls` query resolves (in rbacService.js `onQueryStarted`)

### Two Permission Systems (BOTH ACTIVE)
| System | Hook | Data Source | Format | Used In |
|---|---|---|---|---|
| Legacy | `usePermissions()` | `authSlice.roles.policies` (hashed keys) | `CAN_EDIT_USERS: true` | Some pages (being phased out) |
| New RBAC v2 | `useRBACPermissions()` | `rbacSlice.actionKeyMap` | `hasAction("module.sub.action")` | 24 files across entire app |

### useRBACPermissions.js
```javascript
const { hasAction, canAccess, getModuleActions, getActionsByType } = useRBACPermissions();

// Most common usage:
hasAction("manage-leads.manage-leads.view-lead-detail")  // checks actionKeyMap
canAccess("manage-leads", "row")                          // checks modulesTree recursively
```

### NewSidemenuV2.jsx (ACTIVE SIDEBAR)
- Calls `useGetControlsQuery({ orgId: organizationId || 4, userId })`
- Transforms `modulesTree` → sidebar menu items
- Only shows modules that have `page`, `custom`, or `table` type actions
- Routes come from `module.config.route` (stored in modules DB table)
- **Hardcoded:** `checklist-manager` and `section-manager` get special icons
- **Hardcoded:** 'Enrolled Applicants' menu item is manually injected after 'manage-applicant'
- If RBAC API fails → sidebar shows skeleton indefinitely (no error state)
- orgId fallback: uses `organizationId || 4` (hardcoded default)

### rbacService.js — Endpoint Map

| Endpoint | URL Pattern | orgId Handling |
|---|---|---|
| `getControls` | `org/{orgId}/rbac/controls/{userId}` | Dynamic ✅ |
| `getAllRoles` | `org/{orgId}/rbac/roles` | Dynamic ✅ |
| `getAllRolesOverview` | `org/{orgId}/rbac/allRolesOverview` | Dynamic ✅ |
| `getRoleByOrgId` | `org/{orgId}/rbac/roles/{roleId}` | Dynamic ✅ |
| `createRole` | `org/{payload.orgId}/rbac/createRole` | From payload ✅ |
| `updateRole` | `org/{orgId}/rbac/updateRoles/{roleId}` | Dynamic ✅ |
| `getRoleModulesWithActions` | `org/{orgId}/rbac/roles/{roleId}/getRoleModulesWithActions` | Dynamic ✅ |
| `getRole` | `rbac/roles/{roleId}` | **Missing orgId** ⚠️ |
| `deleteRole` | `org/4/rbac/roles/{roleId}` | **HARDCODED 4** ❌ |
| `createUser` | `org/4/rbac/createUser` | **HARDCODED 4** ❌ |
| `getUser` | `org/4/rbac/users/{userId}` | **HARDCODED 4** ❌ |
| `updateUser` | `org/4/rbac/users/{userId}` | **HARDCODED 4** ❌ |
| `getAllUsers` | `org/4/rbac/getAllUsers` | **HARDCODED 4** ❌ |
| `deleteUser` | `org/4/rbac/users/{userId}` | **HARDCODED 4** ❌ |
| `getAllModulesActionsPermissions` | `org/4/rbac/allModulesActionsPermissions` | **HARDCODED 4** ❌ |
| `getAllUserModulesActionsPermissions` | `org/4/rbac/users/{userId}/getUserModulesWithActions` | **HARDCODED 4** ❌ |
| `getAllSchools` | `org/4/rbac/schools` | **HARDCODED 4** ❌ |

### User Management Pages (Current Active vs Legacy)

| Page | File | Status | Role |
|---|---|---|---|
| User list | ManageUsersV2.jsx | **ACTIVE** | Paginated list + search + CRUD |
| Role list | ManagePermissionsV2.jsx | **ACTIVE** | Lists roles + overview stats |
| Permission editor | ModuleWisePermissionV2.jsx | **ACTIVE** | Module/action checkbox tree |
| Role CRUD form | CreateRoleV2.jsx (SideFilter) | **ACTIVE** | Sidebar form |
| User CRUD form | CreateUserV2.jsx (SideFilter) | **ACTIVE** | Sidebar form |
| Module/Action CRUD | CRUDPermissions.jsx | **ACTIVE** | Dev tool for managing modules/actions |
| ManageRoles | ManageRoles.jsx | **ACTIVE** | Wrapper for ManageRoleTab1 |
| Create user (page) | CreateUser.jsx | **LEGACY** | Old multi-step form, not used |
| Edit user (page) | EditUserDetails.jsx | **LEGACY** | Old form, not used |
| ManageUsers | ManageUsers.jsx | **LEGACY** | Replaced by V2 |
| ManagePermissions | ManagePermissions.jsx | **LEGACY** | Replaced by V2 |
| ModuleWisePermission | ModuleWisePermission.jsx | **LEGACY** | Replaced by V2 |

---

## 5. Current Module: How It Actually Works

### Authorization System — The Two Parallel Systems

**System A: Feature-Array (Legacy — controls backend middleware)**
```
org.features[] + OrgUser.features[]
checked by featureAccessValidator middleware
used on role.routes.js (a SEPARATE routes file from rbac.routes.js)
```

**System B: Module-Action RBAC (New — controls frontend UI only)**
```
modules → actions → role_actions → user_roles
checked by: NO BACKEND MIDDLEWARE
used by: frontend only (useRBACPermissions hook + NewSidemenuV2)
```

**Critical gap:** System B exists in DB and drives the ENTIRE frontend UI (sidebar, all buttons, all actions in 24 files), but **backend routes do not enforce it at all**. There is no middleware that says "check if this user has action X before allowing this API call."

### Complete Permission Flow (Current)
```
1. User logs in → adminLoginV2
   → JWT contains: { userId, role (ENUM), organizationId }
   → Response includes: userRoles (UserRoles → Role → School → Org)

2. Frontend: NewSidemenuV2 mounts
   → getControls({ orgId, userId })
   → Backend: majorRbacController.getAllControls()
     → UserRoles → Role → RoleActions → Actions → Modules (tree built)
   → Response: { user, roles, modulesTree, flatModules, flatActions }
   → setUserPermissions dispatched → rbacSlice.actionKeyMap built

3. Frontend navigation
   → NewSidemenuV2: hides modules with no page actions
   → Pages: hasAction("x.y.z") → checks actionKeyMap → show/hide UI element

4. Backend API calls
   → jwtValidator extracts user (from users.role, not UserRoles)
   → role.routes.js: featureAccessValidator checks org.features + orgUser.features
   → rbac.routes.js: ONLY jwtValidator — no role/permission check at all
```

### Role Level Hierarchy (Current)
- Level 1–5: standard hierarchy
- Level 6: super admin (only level 6 can edit other level 6 roles)
- Level check ONLY exists in `rolesController.updateRole()` — not on create, not on delete, not at middleware level

---

## 6. Full Bug & Weakness Inventory

### 🔴 Critical Security Issues

**CS1 — MASTER_PASSWORD backdoor in authController.js**
If `env.MASTER_PASSWORD` is set, this password bypasses bcrypt for ANY user's login (adminLogin, adminLoginV2, studentLogin). A single leaked env var gives full access to every account.

**CS2 — Insecure default JWT secret**
`JWT_SECRET` defaults to `'default-dev-secret-change-me'`. If production is somehow started without this env var set, tokens are trivially forgeable.

**CS3 — migrateUserRoles has no authentication**
`/api/org/:orgId/rbac/migrateUserRoles` — jwtValidator is commented out on BOTH GET and POST. Anyone on the internet can trigger a data migration.

**CS4 — No authorization middleware on any RBAC route**
`createRole`, `createUser`, `updateUser`, `deleteUser`, `deleteRole`, `syncRoleActions` — all only check `jwtValidator`. Any logged-in user (even a counsellor) can call these.

**CS5 — Generated password returned in API response**
`createUser` returns `generatedPassword` in the response body. If any log/monitoring tool captures API responses, passwords are exposed.

**CS6 — `POST /api/users/` creates a `super_admin` with no authentication** *(found 2026-09-13, open)*
`routes/users.routes.js:47` mounts `createUser` with no `jwtValidator` and no rate limit, and the controller accepts `role: "super_admin"` from the body. Confirmed live on `api-v2` (201, id 4895048). No client calls this route. Remove it. See §18.4.

### 🔴 Critical Data/Logic Bugs

**DB1 — getAllRoles has zero org scoping**
`Role.findAll({ where: { isActive: true } })` — no orgId filter. Every org sees every other org's roles.

**DB2 — getAllUsers uses req.user.organizationId, not URL param**
A super admin querying `GET /org/5/rbac/getAllUsers` always gets their own org's users, not org 5's users.

**DB3 — createUser: no database transaction**
User is created first, then UserRoles inserted (via raw queryInterface), then UserSchools bulkCreated. If step 2 or 3 fails, a ghost user exists with no roles and no schools — orphaned record.

**DB4 — Hardcoded orgId = 4 in 9 frontend service endpoints**
`createUser`, `getUser`, `updateUser`, `getAllUsers`, `deleteUser`, `deleteRole`, `getAllSchools`, `getAllModulesActionsPermissions`, `getAllUserModulesActionsPermissions` — all hardcoded to org 4 in rbacService.js. Also hardcoded in ManagePermissionsV2.jsx (`orgId: 4`). This breaks multi-org support entirely.

**DB5 — Dual role system out of sync**
`users.role` ENUM and `user_roles` junction table both exist. `migrateUserRoles` endpoint exists specifically to sync them, but the migration is not enforced. Login uses `users.role` for JWT, RBAC uses `user_roles` for permissions — they can diverge.

**DB6 — updateUser hard-deletes UserRoles and UserSchools**
`UserRoles.destroy()` and `UserSchools.destroy()` — permanent deletion, not soft delete (inconsistent with `deleteUser` which soft-deletes).

**DB7 — No pagination on getAllRoles**
Returns all roles at once with no limit. At scale this will time out or OOM.

**DB8 — syncRoleActions: no transaction**
Multiple DB operations (soft-delete → reactivate → bulk-create) without transaction. Race conditions possible.

### 🟠 Design Weaknesses

**W1 — schoolId mandatory for roles**
Every role MUST have a `schoolId`. Org-level roles (not tied to any school) are impossible in the current design.

**W2 — createRole: no level hierarchy check**
A user can create a role with `level: 6` (super admin) without any check. Level guard only exists in `updateRole`.

**W3 — countryIso hardcoded to 'IN'**
Line 123 in usersController: `countryIso: 'IN'`. Not scalable.

**W4 — featureAccessValidator uses old feature-array system**
Backend authorization middleware is disconnected from the module/action RBAC system. The entire frontend RBAC system has no backend enforcement.

**W5 — Op.like for search (case-sensitive in PostgreSQL)**
`getAllUsers` uses `Op.like` instead of `Op.iLike`. PostgreSQL LIKE is case-sensitive, so searching "john" won't find "John".

**W6 — sortBy/sortOrder query params accepted but ignored**
`getAllUsers` accepts `sortBy` and `sortOrder` query params but the query always uses `[["id", "DESC"]]` hardcoded.

**W7 — getAllControls: N+1 query problem**
`majorRbacController.getAllControls` makes separate DB queries for user info, userRoles, roleActions, modules, etc. Potentially 20+ queries per request. No batching, no caching.

**W8 — CORS too permissive**
`origin: callback(null, true)` — allows all origins including hostile sites. Credentials are enabled too.

**W9 — Role internalId can silently change on rename**
`updateRole` regenerates `internalId` from name if name changes. Any code referencing `role.internalId` will break silently.

**W10 — OrgUser.orgRole is a plain string**
`OrgUser.orgRole` exists but is NOT a FK to the `roles` table. It's a separate concept entirely, creating two parallel "role" representations for org members.

**W11 — NewSidemenuV2: orgId fallback to 4 hardcoded**
`getControls({ orgId: organizationId || 4, userId })` — if organizationId is not in Redux, falls back to org 4. So sidebar silently shows org 4's modules.

**W12 — Hardcoded menu item in sidebar**
'Enrolled Applicants' is manually spliced into the sidebar array in NewSidemenuV2 — it bypasses the RBAC system entirely.

**W13 — Debug console.logs left in production code**
`getAllUsers` has `console.log("page", page)` and `console.log("limit", limit)` written twice (lines 271–274).

**W14 — permissions and rolePermissions tables unused**
These models/tables exist but no controller uses them. They add confusion about which table is canonical for permissions.

### 🟡 Missing Features

**M1 — No invite flow**
Users created as immediately active with a generated password. No invitation email → set-own-password flow.

**M2 — No audit log**
Zero record of who changed what, when. No trail of permission changes, user creation/deletion, role modifications.

**M3 — No session management**
No way to see active sessions, invalidate a specific session, or force logout.

**M4 — No password policy**
No minimum length, no complexity requirements, no forced change on first login.

**M5 — No email verification**
Users created without verifying their email address.

**M6 — No user status management UI**
Soft delete exists (`status='disabled'`) but no UI to suspend/reactivate a user.

**M7 — No role cloning**
No way to duplicate an existing role as a starting point.

**M8 — No bulk operations**
Cannot assign a role to multiple users at once, cannot deactivate multiple users.

**M9 — No super admin management UI**
No panel for platform-level super admin to manage organizations, override features, impersonate users.

**M10 — No RBAC middleware for v2 routes**
New v2 API (`/api/v2/org/...`) has no action-based authorization — only jwtValidator.

---

## 7. Reusable vs Rebuild Decision

### Keep As-Is (Reuse)
| Item | Why |
|---|---|
| `modules` table + seeded data | Good hierarchical structure, all action keys already in production use |
| `actions` table + data | actionType enum works well, all keys tied to live frontend code |
| `role_actions` junction | Schema is correct and in active use |
| `jwtValidator` middleware | Works well, handles all token sources correctly |
| `featureAccessValidator` | Keep for org-level feature gating (supplement, don't replace) |
| `getModels()` utility | Clean pattern, use everywhere |
| `sendSuccess` / `sendPaginated` / `ApiError` | Well-designed, reuse in all v2 controllers |
| `catchAsync` wrapper | Use on all v2 routes |
| `getAllControls` response shape | **MUST preserve exactly** — 24 frontend files depend on this contract |
| `rbacSlice` + `useRBACPermissions` | Frontend state shape is solid, extend don't replace |
| RTK Query structure in rbacService | Good foundation, add v2 endpoints |
| `unauthorizedMiddleware` in store | Good logout flow, keep |
| OTP and password reset flows | Already built and working |

### Partially Reuse (Needs Fixes)
| Item | What needs to change |
|---|---|
| `majorRbacController.getAllControls` | Extract to service, add caching, fix N+1 queries |
| `Role` model | Make `schoolId` optional (nullable FK with default null) |
| `UserRoles` model | Schema fine, fix the raw queryInterface workaround |
| ManageUsersV2.jsx | Fix orgId handling, add invite flow, status management |
| ManagePermissionsV2.jsx | Fix hardcoded orgId=4, improve role editor UX |
| ModuleWisePermissionV2.jsx | Solid permission editor, fix toggle logic edge cases |
| NewSidemenuV2.jsx | Remove hardcoded menu items, add error state for RBAC failure |
| rbacService.js | Fix all hardcoded org 4 endpoints, add v2 endpoints |

### Must Rebuild (Do Not Reuse)
| Item | Why |
|---|---|
| `usersController.js` | Multiple critical bugs, no transaction, wrong org scoping |
| `rolesController.getAllRoles` | No org scoping |
| `rbac.routes.js` | No authorization middleware — just JWT check on everything |
| `roleValidator` middleware | Uses legacy `users.role` ENUM |
| Password generation approach | Replace with invite flow |
| `users.role` ENUM as source of truth | Should be fully migrated to `user_roles` |
| `OrgUser.orgRole` string field | Confusing parallel system, consolidate |
| `permissions` + `rolePermissions` tables | Unused, should be dropped or formally adopted |

---

## 8. Research: How Top CRMs Handle This

### Salesforce
- **Profiles** = base permission set per user (like roles)
- **Permission Sets** = additive permissions layered on top of profiles
- **Role Hierarchy** = used for DATA visibility (not permissions): who can see whose records
- Principle of Least Privilege enforced strictly — default = no access
- **Field-level security**: per-field read/edit permissions per profile
- **IP restriction + Login Hours** per profile
- Full audit trail: every permission change logged with actor + timestamp + before/after
- **Password policies** configurable per org: minimum length, complexity, expiry, no-reuse count
- Session settings: timeout, concurrent sessions, MFA requirements

### HubSpot
- Super Admins have unrestricted access, manage other admins
- Roles define what can be viewed / edited / deleted per module (3 levels per module)
- **Team-based data visibility**: assign records to teams, control who sees what
- Seat-based licensing affects what roles can be assigned
- **User invite flow**: email invite → user sets own password → verifies email
- All role changes audited

### Zoho CRM
- **Profiles** with per-module CRUD permissions (4 checkboxes per module)
- **Roles** for org hierarchy (data visibility, like Salesforce)
- Profiles assigned to roles
- Login history, IP allowlist, password policy, session timeout — all org-configurable
- Audit log filterable by user, action, date range, module
- **User import/export** via CSV

### Pipedrive
- Clean Admin / Regular User split (simple, effective for SMBs)
- Role-based visibility: own records vs team vs all
- No role hierarchy — simplicity over complexity

### Modern Best Practices (OWASP / Industry Standard)
1. **Backend enforcement always**: Never trust frontend-only permission checks
2. **Invite flow, not password generation**: Server never generates passwords
3. **Email verification**: New users must verify before active
4. **Principle of Least Privilege**: Default = zero access, explicit grants
5. **Immutable audit logs**: Append-only, store actor + target + action + ip + timestamp + before/after payload
6. **Separation of duties**: Creating a super admin requires separate confirmation step
7. **Session management**: View active sessions, invalidate specific token, force logout
8. **Rate limiting on auth endpoints**: Per-IP, per-user
9. **Role level hierarchy on ALL operations** (create, read, update, delete) — not just update
10. **Permission inheritance** (optional): Child roles inherit parent permissions with ability to restrict
11. **Temporal access**: Grant elevated role for X hours/days with auto-expiry
12. **Field-level access** (advanced): Different roles see/edit different fields on same record

---

## 9. Proposed Feature List for v2

### Core (Must Have — Phase 1-4)
- [ ] RBAC authorization middleware that enforces action keys at backend route level
- [ ] User CRUD with full transaction safety (invite flow)
- [ ] Role CRUD with org scoping
- [ ] Module/action permission assignment to roles (preserve existing data + API contract)
- [ ] Role level hierarchy enforcement on ALL operations (create + delete, not just update)
- [ ] Audit log: every mutation recorded (actor, target, action, before, after, ip, timestamp)
- [ ] Fix all hardcoded orgId=4 in frontend service endpoints
- [ ] getAllRoles with org scoping
- [ ] Pagination + search + filter on users AND roles
- [ ] Op.iLike for case-insensitive search

### Important (Should Have — Phase 5-6)
- [ ] Invite flow (email invite → tokenized link → user sets own password)
- [ ] Email verification for new users
- [ ] Password policy (min length, complexity, forced change on first login)
- [ ] User status management UI (suspend/reactivate without delete)
- [ ] Role cloning
- [ ] Bulk operations (invite multiple, assign role to multiple users)
- [ ] Super admin panel (manage orgs, override features, impersonate)
- [ ] Audit log UI (filterable by user/action/date)
- [ ] Fix sidebar hardcoded menu item (Enrolled Applicants should be in RBAC)
- [ ] Fix sidebar error state (show message if RBAC API fails)

### Advanced (Nice to Have — Phase 7+)
- [ ] Session management (view active sessions, invalidate specific)
- [ ] Temporary role elevation (time-limited access)
- [ ] Permission diff view (compare two roles)
- [ ] Login history per user
- [ ] IP allowlist per org
- [ ] Export user/role data (CSV)
- [ ] Field-level permissions (per module per role)
- [ ] Remove or formally adopt `permissions` + `rolePermissions` tables

---

## 10. Recommended Architecture for v2

### Backend: be-anandi/src/v2/

```
src/v2/
├── controllers/
│   ├── users/
│   │   ├── usersController.js       ← user CRUD (transactional, org-scoped)
│   │   └── inviteController.js      ← invite flow
│   ├── roles/
│   │   └── rolesController.js       ← role CRUD (org-scoped, level check everywhere)
│   ├── permissions/
│   │   └── permissionsController.js ← getAllControls, module tree, controls
│   └── auditLog/
│       └── auditLogController.js    ← audit log read
├── services/
│   ├── rbacService.js               ← core: getModulesTree, buildControlsResponse
│   ├── inviteService.js             ← token generation, invite email
│   └── auditLogService.js           ← write audit event (used by all v2 mutating controllers)
├── middleware/
│   └── rbacMiddleware.js            ← action-key-based auth middleware (NEW)
├── routes/
│   ├── users.routes.js
│   ├── roles.routes.js
│   ├── permissions.routes.js
│   └── audit.routes.js
└── validators/                      ← Joi/Zod schemas
    ├── userValidator.js
    └── roleValidator.js
```

### rbacMiddleware.js (the critical missing piece)
```javascript
// Usage on any route:
router.post('/users', requireAction('users.create'), catchAsync(createUser))
router.delete('/users/:id', requireAction('users.delete'), catchAsync(deleteUser))
router.put('/roles/:id', requireAction('roles.edit'), catchAsync(updateRole))

// Implementation:
export const requireAction = (actionKey) => async (req, res, next) => {
  // 1. Get userId from req.user
  // 2. Fetch user's role_actions via UserRoles → RoleActions
  // 3. Check if actionKey is in the set
  // 4. If not, return 403
  // 5. If yes, next()
  // (with Redis caching for performance)
}
```

### New DB Tables Needed (migration files only)
1. **`audit_logs`** — `{ id, actor_id, actor_email, target_type, target_id, action, before, after, ip, user_agent, org_id, created_at }`

### Frontend: fe-anandi/src/pages/Admin/v2/
```
pages/Admin/v2/
├── UserManagement/
│   ├── UsersList.jsx           ← paginated + search + filter + bulk ops
│   ├── InviteUser.jsx          ← invite flow (replaces CreateUserV2)
│   └── UserDetail.jsx         ← user profile + roles + activity
├── RoleManagement/
│   ├── RolesList.jsx           ← roles with usage stats (replaces ManagePermissionsV2)
│   └── RoleEditor.jsx          ← module/action tree editor (replaces ModuleWisePermissionV2)
└── AuditLog/
    └── AuditLog.jsx            ← filterable audit trail
```

### rbacService.js v2 additions
All new endpoints must use dynamic orgId from Redux auth state. Zero hardcoded org IDs.

---

## 11. Roadmap: Step-by-Step Rebuild Plan

> Move to next step ONLY when Prateek says `next`

### Phase 0 — Clarifications (CURRENT)
- Answer 12 open questions below
- Decide on schoolId optionality, invite flow preference, legacy cleanup

### Phase 1 — DB Foundation
- Step 1.1: Migration — `org_modules`, `user_programs`, `user_application_forms`, `user_reporting_managers`, `audit_logs`
- Step 1.2: Create Sequelize models for the new Phase 1 tables
- Step 1.3: Build `rbacService.js` + `rbacMiddleware.js` foundation with Redis-backed caching
- Step 1.4: Keep legacy `roles.school_id` column for backward compatibility, but ignore it in v2 role design

### Phase 2 — Users v2 Backend
- Step 2.1: `usersController.js` v2 (full CRUD, transactional, org-scoped, iLike search)
- Step 2.2: Wire to v2 routes with rbacMiddleware
- Step 2.3: Audit log on all mutations

### Phase 3 — Roles v2 Backend
- Step 3.1: `rolesController.js` v2 (org-scoped, level check on all ops)
- Step 3.2: Transactional syncRoleActions
- Step 3.3: Wire to v2 routes
- Step 3.4: Audit log

### Phase 4 — Permissions v2 Backend
- Step 4.1: `permissionsController.js` (getAllControls optimized, role/user trees)
- Step 4.2: Optimize getAllControls (Redis cache, batch queries)

### Phase 5 — Audit Log
- Step 5.1: `auditLogController.js` (list + filter)
- Step 5.2: Wire routes

### Phase 6 — Frontend v2
- Step 6.1: Fix all hardcoded orgId=4 in rbacService.js
- Step 6.2: New RTK Query endpoints for v2 APIs
- Step 6.3: UsersList with invite flow
- Step 6.4: RoleEditor improvements
- Step 6.5: AuditLog page

### Phase 7 — Integration & Testing
- Step 7.1: End-to-end on staging DB
- Step 7.2: Super admin review
- Step 7.3: Staging sign-off → production plan

---

## 12. Open Questions for Prateek

**Q1 — School dependency on Roles:**
Must every role belong to a school? Or should school be optional (org-level roles with no school)?

**Q2 — Invite flow vs immediate creation:**
Should v2 always use invite (user sets own password via email link), or support both modes?

**Q3 — Legacy users.role ENUM:**
Should v2 fully deprecate it and migrate all users to `user_roles` only? Or keep it in sync?

**Q4 — Role level scale:**
Is 1–6 fixed? What does each level mean in your business context? Should the scale be configurable?

**Q5 — Super Admin scope:**
Is super admin platform-wide (manages all orgs) or per-org (each org has their own super admin)?

**Q6 — permissions + rolePermissions tables:**
Should v2 use these tables (formally adopt), or drop them (clean up)? Currently unused.

**Q7 — Audit log visibility:**
Should audit logs be accessible in the admin UI, or just stored in DB?

**Q8 — Frontend routing for v2:**
Should v2 pages replace existing routes (`/admin/users`) or live separately during transition?

**Q9 — Multi-school roles:**
Can a user have different roles in different schools, or is it one role set across all their schools?

**Q10 — Email verification:**
Should new users verify email before becoming active?

**Q11 — MASTER_PASSWORD backdoor:**
Should this be removed entirely, or kept as a controlled super admin override (with audit logging)?

**Q12 — Enrolled Applicants hardcoded in sidebar:**
This menu item is manually injected in NewSidemenuV2. Should it be added as a proper module/action in the RBAC DB so it follows the system?

---

## 13. Progress Log

| Date | Step | Status | Notes |
|---|---|---|---|
| 2026-04-06 | Initial codebase analysis | ✅ Complete | |
| 2026-04-06 | Full deep research (both repos) | ✅ Complete | Both backend and frontend read exhaustively |
| 2026-04-06 | Bug inventory | ✅ Complete | 5 critical security, 8 critical data/logic, 14 design weaknesses, 10 missing features |
| 2026-04-06 | Research: top CRM practices | ✅ Complete | Salesforce, HubSpot, Zoho, Pipedrive |
| 2026-04-06 | Architecture proposal | ✅ Complete | v2 folder structure, rbacMiddleware design |
| 2026-04-06 | Feature list | ✅ Complete | Core / Important / Advanced tiers |
| 2026-04-06 | Open questions documented | ✅ Complete | 12 questions for Prateek |
| 2026-04-06 | All 12 questions answered by Prateek | ✅ Complete | See decisions below |
| — | Phase 0: Finalize flow design | ✅ Complete | |
| 2026-04-08 | Phase 1 foundation started | ⏳ In Progress | Schema + RBAC enforcement foundation underway |
| 2026-04-08 | Phase 1 schema foundation | ✅ Complete | Added migration for `org_modules`, `user_programs`, `user_application_forms`, `user_reporting_managers`, `audit_logs` |
| 2026-04-08 | Phase 1 model foundation | ✅ Complete | Added Sequelize models and associations for the new tables |
| 2026-04-08 | Phase 1 RBAC enforcement foundation | ✅ Complete | Added `src/v2/services/rbacService.js` and `src/v2/middleware/rbacMiddleware.js` with org module ceiling enforcement |
| 2026-04-09 | Super admin repo review | ✅ Complete | Reviewed `/organizations` flow and confirmed current create-org path in `super-admin` and `be-anandi` |
| 2026-04-09 | Org creation RBAC bootstrap | ✅ Complete | Create-org flow now bootstraps first admin user, default level-6 Admin role, `org_modules`, `user_roles`, `org_users`, matching `role_actions`, and audit log entry |
| 2026-04-09 | Super admin module catalog | ✅ Complete | Added live module catalog API and Modules page so org creation allocates from the real RBAC tree |
| 2026-04-09 | Super admin module definition flow | ✅ Complete | Added v2 endpoints and super-admin UI flow to create top-level modules, submodules, and action routes |
| 2026-04-09 | Super admin module edit flow | ✅ Complete | Added module, submodule, and action edit/update flow in the `/modules` page |
| 2026-04-09 | Route template support | ✅ Complete | Added `config.route` + `config.routeParams` support for module/submodule routes including dynamic params like `:orgSlug` |
| 2026-04-09 | Drag-and-drop reorder | ✅ Complete | Added same-group drag/drop reorder for top-level modules, submodules, and routes with persisted `sortOrder` rewrite |
| 2026-04-09 | Reorder loading UX | ✅ Complete | Added full-screen pending overlay and centered loader while reorder APIs are in flight |
| 2026-04-09 | Super admin drawer focus fix | ✅ Complete | Fixed shared drawer focus-reset issue that blurred inputs while typing in module forms |
| 2026-04-17 | v2 RBAC continuation alignment | ✅ Complete | Synced backend behavior with the working handoff: user form payload shape, helper endpoint response shape, and full org-module ancestry enforcement |
| 2026-06-17 | Per-role stage/sub-stage permissions | ✅ Complete | New instance-level scope layer on top of action-key RBAC. See §14. Backend enforcement on change-stage (bulk + profile PATCH) and settings save; role-editor "Stage Access" tab. Migrations pending `npm run migrate`. |
| 2026-08-06 | Data visibility keyed to role level | ✅ Complete | Third scope layer — WHOSE records you see. `users.role` ENUM removed as the org-wide test (it made every level-2 Team Lead an org-wide admin); replaced by `userService.isOrgWideActor` (level 6 = org owner). Lead list, applicants, archive, export, dashboard-v2, calendar, user-dashboard all aligned. See §15. |
| 2026-08-12 | Auth storage consolidated to redux | ✅ Complete | `cookieService` deleted; the `auth` slice is the only client-side store. Fixed logout leaving the backend httpOnly cookie alive (up to 30 days) and 15 services authenticating on that cookie instead of the `Bearer` header. See §16. |
| 2026-08-12 | RBAC endpoint performance | ✅ Complete | Sequelize multi-include cartesian (159,840 rows for ~81 rows of data): `/rbac/me/context` 8.5s → 0.35s, `/rbac/users/:id` 6.4s → 0.40s. Dashboard no longer blocks on `me/context`. See §16. |
| 2026-08-13 | Form-scoped visibility (`roles.visibility_scope`) | ✅ Complete | Third scope axis: a role can see every lead on its members' allocated application forms, OR'd with the reporting downline. Reporting managers inherit a form-scoped report's view. All lead surfaces + dashboard tiles resolve through one function (`resolveActorVisibility`). Role editor toggle added; profile page now shows the user's reporting line. See §15. |
| 2026-08-28 | `getSystemUser()` cached — 91% DB load | ✅ Complete | Uncached `users WHERE role = 'super_admin'` seq scan (657 MB, 3 CPUs) ran per WhatsApp/SMS delivery event. Cached per process; deliberately **not** indexed (would change the `LIMIT 1` row and re-attribute system records). See §17. |
| 2026-09-13 | Staff duplicate-email rule; super-admin create fixed | ✅ Complete | `users.email` is not unique; student rows are a separate identity. `createSuperAdmin` now ignores student rows and compares case-insensitively. `updateMe` still has the old check (open). See §18.1–18.2. |
| 2026-09-13 | Super-admin accounts card | ✅ Complete | Fixed list emptied by the search debounce after data arrived; create-drawer errors now inline only (`INLINE_ERROR_ENDPOINTS`). See §18.5. |
| 2026-09-13 | Test-email 403 mislabelled as a permission error | ✅ Complete | Not RBAC: sender verification. Preview drawers now resolve the org sender (`utils/orgSender.js`). See §18.6. |
| 2026-09-13 | Unauthenticated `POST /api/users/` creates `super_admin` | 🔴 Open | Unused route, live on api-v2. Recommend removal. See CS6 / §18.4. |
| 2026-09-13 | Mixed-case staff twin rows break login (kapish) | 🔴 Open | Reset and login resolve different rows. UPDATE proposed, not run. See §18.3. |

## FINAL DECISIONS (from Prateek's answers — Round 1)

| Question | Decision |
|---|---|
| School on roles | Removed from role design. Roles are organization-scoped only |
| Multi-school roles | Not applicable at role level. School/program/form allocation is per-user for data visibility |
| permissions/rolePermissions tables | Not used in v2. Marked for future drop migration |
| Legacy users.role ENUM | Fully abandoned in v2. System B (user_roles) is canonical. Old column stays in DB but v2 ignores it |
| Password flow | Backend generates random password → sends via email only. NOT returned in API response |
| Email verification | Not required. Users created as active immediately |
| Super Admin scope | Platform-wide (Option A). One super admin manages all orgs. Separate from org-level admins |
| Audit log | Visible in admin UI (Option A) — filterable page showing history of all actions |
| Frontend routing | Keep old routes, replace components behind them with v2 versions |
| MASTER_PASSWORD | Keep but controlled — audit log must record every use |
| Enrolled Applicants in sidebar | Must be added as proper module/action in RBAC DB |
| No patching | Full rebuild in v2. No patches to old code |

## REVISED REQUIREMENTS (from Prateek — 2026-04-08)

> **Role levels are BACK** — previous decision to remove them is overridden.

### Entity Hierarchy
```
Super Admin (platform-wide, manages all orgs)
  └── Organization
        └── School(s)
              └── Program(s)
                    └── Application Form(s)
```

### Organization Creation Flow
1. Super admin creates an Organization
2. An **"Admin" role** is auto-created for the org with full module access
3. Super admin provides user credentials for the org's first user → that user gets the Admin role
4. Super admin can configure **which permissions** the Admin role actually has
5. This Admin user is the **top-level owner** of the organization
6. Users added to the "Admin" role also act as top-level owners

### Super Admin
- Super admin is a **user in the `users` table** with a special flag (e.g., `role='super_admin'` or a boolean)
- Super admin can **only login to the super admin frontend module** (separate repo)
- Super admin is **outside the org-level level system** — they manage all orgs from the platform level
- Super admin creates orgs, configures which modules/permissions the org's Admin role has

### Role Level System (REVISED — levels ARE enforced)
- Available levels: **1, 2, 3, 4, 5, 6** (all six — level 5 was a typo in original list)
- Higher number = more privileged
- The **auto-created "Admin" role** gets the highest level (6) since it's the default/main role of the first org user
- **Level rules for a user with effective level N:**
  - ✅ Can **see and edit** users/roles at level N and below (own level included)
  - ❌ Cannot see or edit roles/users at levels above N
  - ✅ Can **create** roles at levels **strictly below** N (< N, NOT ≤ N)
  - ✅ Can assign modules/actions to roles **only if their own role(s) have those modules**
- Permissions flow **downward only** — you can never grant what you don't have
- **Multi-role users:** If a user has roles at different levels (e.g., level 3 + level 5), the **highest level** is their effective level. They also get access to resources (modules/actions) from ALL their roles combined
- **rbacMiddleware must also check `org_modules`** — if a module isn't allocated to the org, no role in that org can access it

### Module/Permission Configuration
- Super admin configures which modules, sub-modules, and routes (actions) are available per org
- After a role is created by any non-super-admin, the creator specifies permissions from their own allowed set

### User Creation Requirements (EXPANDED)
1. User can have **multiple roles**
2. Must allocate user to **one or more schools** (or all schools) in the org — **per-user** (not per-role)
3. Based on allocated schools → list available **programs** → allocate programs to user
4. Based on programs → list available **application forms** → allocate forms to user
5. Assign **reporting manager(s)** = users from roles with level **≥** the user's selected role level
6. Allocations are **per-user**, not per-role (a user's schools/programs/forms are independent of which role they have)
7. **School/Program/Form allocations are for DATA VISIBILITY only** — they control which leads/data the user can see and act on. CRM feature access is solely determined by the module/action permissions of the user's role(s)

### Reporting Managers
- When creating a user, one or more reporting managers can be assigned
- Reporting managers must be users within the **same organization**
- **Purpose:** If User X (counsellor) has reporting managers User A and User B, then A and B can see X's assigned leads, activity, and performance in the dashboard alongside their own data
- **Design decision:** Many-to-many relationship via a `user_reporting_managers` junction table:
  ```
  user_reporting_managers { id, user_id (FK), manager_id (FK → users.id), org_id (FK), is_active, created_at }
  ```

### Existing DB Models Confirmed
- `programs` table exists — linked to org + school
- `applicationForms` table exists — linked to org + program + batch + round
- `schools` table exists — linked to org
- Schema details in `v2_architecture_db_diagram.md`
- **New junction tables needed:**
  - `org_modules` (NEW — `org_id`, `module_id`, `is_active`, `granted_by`) — links modules to organizations at platform level
  - `user_schools` (already exists as `UserSchools`)
  - `user_programs` (NEW — `user_id`, `program_id`, `org_id`)
  - `user_application_forms` (NEW — `user_id`, `form_id`, `org_id`)
  - `user_reporting_managers` (NEW — `user_id`, `manager_id`, `org_id`, `is_active`)
- **Note:** `Organization.features` / `featuresOverride` is the OLD feature-flag system — NOT connected to RBAC modules. The new `org_modules` table replaces this for v2

### Super Admin Frontend (Separate Repo)
- A **separate repo** now exists in this working directory for the super admin frontend module: `super-admin/`
- Org creation → School → Program → Application Form flow is **partially built** there
- It uses **v2 APIs** from `be-anandi`
- Current org creation UI entry point:
  - `super-admin/src/features/organizations/ui/OrganizationsPage.jsx`
  - `super-admin/src/features/organizations/ui/CreateOrganizationDrawer.jsx`
- Current backend API used by the drawer:
  - `POST /api/v2/org/orgs`
- Current super-admin module management UI entry point:
  - `super-admin/src/features/modules/ui/ModulesPage.jsx`
- Current backend platform-module service/controller entry points:
  - `be-anandi/src/v2/services/platformModuleService.js`
  - `be-anandi/src/v2/controllers/platformModuleController.js`
- Initial review findings:
  - org creation UI previously only collected organization fields (name, internalId, contact, logo)
  - backend org creation previously only created the `organizations` row
  - no RBAC bootstrap existed for default Admin role, first admin user, `org_modules`, or `role_actions`
- Status: **Repo access received and initial org-creation/RBAC integration started**

### Open Questions (Round 2 — 2026-04-08) — ANSWERED

| # | Question | Answer |
|---|---|---|
| Q1 | Level 5 missing from list (1,2,3,4,6) — intentional or typo? | **Typo** — all 6 levels exist (1-6) |
| Q2 | Is level 6 reserved for super admin only? | **No.** Super admin is separate (platform-level, own frontend). Level 6 = org's Admin role, the highest within an org |
| Q3 | Level 3 user: can they edit level 3 users, or only below? | **REVISED:** Can edit at own level AND below (see Round 3 Q11 update) |
| Q4 | Level 3 user: can they SEE level 3 roles, or only 1 & 2? | **REVISED:** Can see own level AND below (see Round 3 Q11 update) |
| Q5 | Permission assignment: creator can only assign modules they have? | **Yes, confirmed** |
| Q6 | Programs and Application Forms — existing DB models? | **Yes** — `programs` and `applicationForms` tables exist (see v2_architecture_db_diagram.md) |
| Q7 | Reporting manager — many-to-many? New junction table? | **Prateek deferred to AI.** Decision: many-to-many via `user_reporting_managers` junction table |
| Q8 | User-School-Program-Form allocation: per-user or per-role? | **Per-user** |
| Q9 | Super admin: user in `users` table with special flag, or separate entity? | **User in `users` table with special flag** |
| Q10 | Org creation: Admin user must be new, or can assign existing user? | **Must be new** |

### Open Questions (Round 3 — 2026-04-08) — ANSWERED

| # | Question | Answer |
|---|---|---|
| Q11 | Who edits level 6 users? Only super admin? Or can level 6 users edit each other? | **Level 6 users can edit each other.** Original "strictly below" rule is dropped. New rule: **own level and below** |
| Q12 | The auto-created "Admin" role gets level 6 — confirmed? | **Yes, confirmed** |
| Q13 | Can a user have roles at different levels? If so, which level applies? | **Highest level wins.** User gets combined resources from all roles. Example: level 3 + level 5 → effective level 5, with modules from both roles |

### Open Questions (Round 4 — 2026-04-08) — ANSWERED

| # | Question | Answer |
|---|---|---|
| Q14 | rbacMiddleware should also check org_modules (module allocated to org)? | **Yes, agreed** |
| Q15 | Can a level N user create roles at their own level (≤ N) or strictly below (< N)? | **Strictly below (< N).** A level 4 user can create level 1, 2, 3 roles only |
| Q16 | User-School-Program-Form allocations: data visibility or CRM feature access? | **Data visibility only.** Controls which leads/data user sees. CRM features are solely from role module/action permissions |

### Module Configuration System (NEW — 2026-04-08)

**Correct platform model for sidebar + permissions:**
1. **Module** = top-level sidebar item or expandable sidebar group
2. **Submodule** = child sidebar item under a parent module
3. **Action** = permission under a module or submodule

**Important distinction:**
- A **module/submodule route** is for sidebar navigation
- An **action** is a permission
- An action may:
  - stay on the same page and only control a button/modal/API capability
  - or optionally carry its own route if clicking that action opens another page

**Examples from the live CRM:**
- `Manage Lead`
  - module route: `/admin/manage-leads`
  - same-page actions: `communicate`, `change-lead-stage`, `assign-counsellor`
  - action with its own route: `view-lead-detail` → `/admin/view-leads/:leadId`
- `Template Manager`
  - parent module with no direct route
  - submodules:
    - `Email Templates` → `/admin/email-templates`
    - `WhatsApp Templates` → `/admin/whatsapp-templates`
    - `Sms Templates` → `/admin/sms-templates`
  - submodule actions:
    - `preview` → no route
    - `edit` → may have its own route if edit opens a separate page

**Super-admin configuration rule:**
- Create **Module** or **Submodule** when you are defining sidebar structure/navigation
- Create **Action** when you are defining permission
- Add an action route only if that action truly redirects to another page
- Do **not** model a route-bearing action as a submodule unless it should also appear as a sidebar item

**Important implementation correction (2026-04-09):**
- The super-admin UI originally used the word `Route` in places where the system was actually creating an **Action**
- This caused confusion and was corrected
- Current intended language:
  - `Module` / `Submodule` = navigation structure
  - `Action` = permission
  - `Action Route` = optional route metadata only when that permission opens another page

**Current super-admin `/modules` UX rules (must preserve):**
- There is **no global `Create Action` button** anymore
- Actions should be created only from the specific module/submodule 3-dot menu
- Submodules should be created only from the parent module 3-dot menu
- In contextual create flows:
  - creating a submodule from a module menu must **not** ask the user to choose `Parent Module` again
  - creating an action from a module/submodule menu must **not** ask the user to choose `Attach To` again
- Those parent/owner fields should appear only as locked context in create flows, not as redundant selectors
- Manual `sortOrder` input was removed from the create/edit drawer because ordering is managed via drag-and-drop on the list page
- New modules/submodules/actions are appended automatically to the end of their sibling group by backend default ordering logic

**Modules are configured at the platform level by super admin:**
1. Super admin defines all modules, sub-modules, and their routes/actions in the system
2. When creating an organization, super admin selects **which modules** to grant to that org
3. Upon org creation: the Admin role (level 6) + the first admin user are created, and the selected modules' permissions are granted to the Admin role
4. This gives platform-level control over what each org can access

**Why this matters:**
- Different orgs can have different module sets (some orgs get custom-built modules only they can access)
- Super admin controls the ceiling — no org-level user can grant modules that weren't allocated to the org
- Permission hierarchy: **Platform modules → Org-allocated modules → Admin role permissions → Sub-role permissions → User's combined role permissions**

**The full permission chain:**
```
Super Admin defines: [Module A, B, C, D, E, F, G]
    ↓
Org "Acme" gets: [A, B, C, D]          (selected during org creation)
    ↓
Admin role (level 6) gets: [A, B, C, D]  (all org modules by default)
    ↓
Admin (level 6) creates "Manager" role (level 5): [A, B, C]  (strictly below 6, modules from what Admin has)
    ↓
Manager (level 5) creates "Counsellor" role (level 3): [A, B]  (strictly below 5, modules from what Manager has)
    ↓
A user with roles "Manager" + "Counsellor": effective level 4, modules [A, B, C] (union of both roles)
```

## CONVERSATION UPDATES

### 2026-04-08 — Scope correction before implementation
- Roles are **organization-scoped only**. They do not belong to a school.
- School, program, and application form assignments remain **per-user allocations** used only for data visibility.
- This file remains the running source of truth for decisions, progress, and handoff context until implementation is complete.

### 2026-04-08 — First implementation pass completed
- Added backend migration files:
  - `be-anandi/src/migrations/20260408090000-create-org-modules.cjs`
  - `be-anandi/src/migrations/20260408090001-create-user-programs.cjs`
  - `be-anandi/src/migrations/20260408090002-create-user-application-forms.cjs`
  - `be-anandi/src/migrations/20260408090003-create-user-reporting-managers.cjs`
  - `be-anandi/src/migrations/20260408090004-create-audit-logs.cjs`
- Added backend models: `OrgModule`, `UserProgram`, `UserApplicationForm`, `UserReportingManager`, `AuditLog`
- Added RBAC enforcement foundation:
  - `be-anandi/src/v2/services/rbacService.js`
  - `be-anandi/src/v2/middleware/rbacMiddleware.js`
- `rbacService` currently computes:
  - user's active org-scoped roles
  - effective highest level
  - union of allowed action keys
  - org-level module ceiling via `org_modules`
- Migration strategy updated: **one table per migration file** so each table can be rolled back independently in the future.
- Legacy `roles.school_id` is **not dropped yet**. It is retained only to avoid breaking old code; v2 logic must ignore it.
- `InviteToken` is **not part of the active RBAC module build**. If invite-link onboarding is ever needed later, it should be implemented only after the core module is complete.

### 2026-04-09 — Super admin repo integrated into current workspace
- `super-admin/` repo is now available for direct implementation work.
- Confirmed active route under review: `http://localhost:3000/organizations`
- Confirmed org creation flow currently opens `CreateOrganizationDrawer.jsx` from `OrganizationsPage.jsx`
- Confirmed drawer previously posted only basic org fields to `POST /api/v2/org/orgs`
- Confirmed backend `createOrg` previously created only the `organizations` row and did not bootstrap RBAC
- Current implementation direction:
  - extend org creation to also create first admin user + default Admin role
  - allocate selected modules into `org_modules`
  - grant matching actions to the Admin role
  - expose a separate super-admin Modules page for module/submodule/admin-route management

### 2026-04-09 — Super admin module-definition flow started
- Added backend v2 endpoints for super-admin platform-module definition:
  - `GET /api/v2/org/module-catalog`
  - `POST /api/v2/org/modules`
  - `POST /api/v2/org/actions`
- Added backend service/controller files:
  - `be-anandi/src/v2/services/platformModuleService.js`
  - `be-anandi/src/v2/controllers/platformModuleController.js`
- Added super-admin module definition UI:
  - `/modules` route
  - create top-level module flow
  - create submodule flow
  - create action flow
- Build verification:
  - `super-admin` build passes
  - existing ESLint warning remains in `src/ui-kit/Drawer/DrawerHeader.jsx` and is unrelated to this RBAC work

### 2026-04-09 — Super admin module-management flow expanded
- Added backend update endpoints for:
  - module edit
  - submodule edit
  - action edit
- Added route-template support in module config:
  - `config.route`
  - `config.routeParams`
- Added optional route metadata support in action config:
  - `actions.config.route`
  - `actions.config.routeParams`
- Dynamic route templates are now supported in module definitions, for example:
  - `/organizations/:orgSlug`
  - `/organizations/:orgSlug/schools/:schoolId/programs`
- Dynamic action routes are now supported for actions that open another page, for example:
  - `/admin/view-leads/:leadId`
- Added same-group drag-and-drop reorder support for:
  - top-level modules
  - submodules within the same parent
  - actions within the same module
- Reorder persistence now rewrites sibling `sortOrder` values sequentially through dedicated v2 reorder endpoints
- Added backend delete endpoints:
  - `DELETE /api/v2/org/modules/:moduleId`
  - `DELETE /api/v2/org/actions/:actionId`
- Delete behavior is soft-delete:
  - deleting a module/submodule deactivates that module tree and all descendant actions
  - deleting an action deactivates only that action

### 2026-04-09 — Super admin UX fixes
- Fixed shared drawer focus behavior so module form inputs no longer lose focus while typing
- Replaced the small reorder-pending text state with a full-screen overlay and centered loader during reorder saves
- Simplified the explanatory note on the `/modules` page so it reads as admin guidance rather than developer documentation
- Renamed misleading UI language from `Route` to `Action` where the user is actually defining permissions
- Replaced text-heavy module/submodule/action controls with consistent 3-dot action menus
- Moved module and submodule 3-dot menus into the same visual row as their target names so ownership is obvious
- Action rows also now use the same 3-dot menu pattern for consistency
- Removed the global `Create Action` button from the page header because actions must be created in context
- Removed manual ordering input from the drawer because drag-and-drop is the canonical ordering UX
- In contextual create flows, redundant `Parent Module` / `Attach To` selectors were removed and replaced with locked context display

### 2026-04-09 — Current implementation status
- **Backend completed for super-admin platform module management:**
  - create module
  - edit module
  - delete module tree
  - create action
  - edit action
  - delete action
  - reorder modules
  - reorder actions
  - optional `actions.config.route` + `actions.config.routeParams`
  - automatic default ordering for newly created sibling items
- **Frontend completed for super-admin `/modules`:**
  - module cards render current live RBAC catalog
  - module/submodule/action 3-dot menus exist
  - create/edit/delete flows exist
  - drag/drop reorder exists
  - full-screen reorder loader exists
  - contextual create flows are cleaned up
- **Remaining work:**
  - end-to-end testing by actually creating/editing/deleting/reordering modules and actions against the local DB
  - apply and run the migration `20260409153000-add-config-to-actions.cjs`
  - verify org creation bootstrap against real selected modules/actions
  - continue into org-level RBAC user/role management build after super-admin module system is validated
- **Known pre-existing unrelated warning:**
  - `super-admin/src/ui-kit/Drawer/DrawerHeader.jsx`
  - ESLint duplicate props warning remains and is not part of RBAC implementation

### 2026-04-09 — Super admin code review and fixes (by Claude Code, replacing Codex)
- **Issues found in Codex's `/modules` page:**
  1. Submodule actions were invisible — only showed a count, no way to see/edit/delete individual actions under submodules
  2. `actionType` dropdown (Page/Table/Row/Custom) had no guidance on what each type means or how it affects sidebar visibility
  3. `routeParams` was a separate redundant field — params were already visible in the route template (e.g. `:leadId`)
  4. `buildModuleTree` in `platformModuleService.js` dropped `description` — edit forms couldn't pre-fill descriptions
  5. Error handling in `platformModuleController.js` used fragile exact-string matching on error messages
- **Fixes applied:**
  - `ModulesPage.jsx`: submodule actions now render inline under each submodule with full drag/drop reorder and 3-dot edit/delete menus (same as top-level module actions)
  - `ModuleComposerDrawer.jsx`: actionType options now include descriptive labels explaining sidebar visibility; `routeParams` field removed entirely — params auto-extracted from route template via regex; default actionType changed from `page` to `custom`
  - `platformModuleService.js`: `buildModuleTree` now includes `description` for both modules and actions; all `throw new Error()` replaced with `throw new ApiError(statusCode, message)` for proper HTTP status codes
  - `platformModuleController.js`: replaced ~100 lines of string-matching error handling with a single `handleServiceError` that reads `error.statusCode` from `ApiError`
  - `ModulesPage.styles.js`: added `SubmoduleActions` styled component for indented action lists under submodules

### 2026-04-09 — Create Organization drawer cleanup
- **Removed fields from org creation form:**
  - `Internal ID` — now auto-generated from org name via `slugifyInternalId()`, no longer shown as a separate field
  - `Contact Email` — redundant with the admin user's email
  - `Contact Phone` — redundant with the admin user's phone
  - These fields existed from the pre-RBAC era when org creation didn't create an admin user
- **Admin Phone is now required** (was optional)
- **Country code dropdown added** for admin phone:
  - Uses existing `RawCountriesList` from `infrastructure/api/countries.js`
  - Format: `+91 (IN)`, `+1 (US)`, `+1 (CA)` etc.
  - Default: `+91 (IN)`
  - `countryCode` is now sent in the `adminUser` payload to the backend
- **Module allocation UX improvements:**
  - Added "Select All Modules" checkbox with selection count badge (`5 / 12 selected`)
  - Checking a parent module auto-selects all its children; unchecking deselects all children
  - Submodule checkboxes disabled when parent is unchecked (prevents orphaned selections)
  - Selected parent modules get a highlighted border for visual scanning
  - Child modules indented with a left border line showing hierarchy
  - Each module's metadata shows route and submodule count inline
- **Backend:** `orgBootstrapService.js` updated to accept `countryCode` from admin user payload and store both `countryCode` and `countryIso` on the user record. Uses `RawCountriesList` from `be-anandi/src/utils/countries.js` for lookup.
- **Phone validation added** (frontend + backend): validates against the selected country's `mobileValidation` rules (min/max length + regex pattern) from `RawCountriesList`

### 2026-04-10 — Phase 2: Users v2 Backend (by Claude Code)
- **New files created:**
  - `be-anandi/src/v2/services/userService.js` — core user CRUD business logic
  - `be-anandi/src/v2/controllers/userController.js` — HTTP handlers
  - `be-anandi/src/v2/routes/rbacUsers.routes.js` — routes with rbacMiddleware
- **Routes mounted at** `/api/v2/org/:orgId/rbac/...`:
  - `GET /users` — paginated user list (search by name/email/phone, iLike, level-filtered)
  - `GET /users/:userId` — single user with roles, schools, programs, forms, reporting managers
  - `POST /users` — create user (transactional: User + OrgUser + UserRoles + UserSchools + UserPrograms + UserApplicationForms + UserReportingManagers + audit log)
  - `PATCH /users/:userId` — update user (transactional sync of all allocations)
  - `DELETE /users/:userId` — soft delete (status='disabled', OrgUser='removed', UserRoles deactivated)
  - `GET /schools` — org schools for form dropdown
  - `GET /programs?schoolIds=1,2` — programs filtered by schools
  - `GET /application-forms?programIds=1,2` — forms filtered by programs
  - `GET /potential-managers?minLevel=3` — users with effective level >= minLevel
- **All routes protected by `rbacMiddleware.requireAction()`** with action keys:
  - `manage-users.manage-users.view` (list, get, helpers)
  - `manage-users.manage-users.create`
  - `manage-users.manage-users.edit`
  - `manage-users.manage-users.delete`
- **Key design decisions in userService:**
  - Level hierarchy enforced on all operations (actor can only see/edit/delete users at own level or below)
  - Cannot delete your own account
  - Roles validated: must exist in the org, level must not exceed actor's level
  - Password generated server-side, emailed via `emailService.sendTemplate()`, never in API response
  - Phone validation uses `RawCountriesList.mobileValidation` (same as org creation)
  - RBAC cache invalidated after update/delete via `rbacService.invalidateUserRbacContext()`
  - Audit log entry on every create/update/delete
  - v2 models (UserProgram, UserApplicationForm, UserReportingManager) use snake_case field names; old models (User, OrgUser, UserRoles, UserSchools) use camelCase
- **Action keys needed in the module system:** Before these APIs can be used, the super-admin must create a `manage-users` module with actions keyed as `manage-users.view`, `manage-users.create`, `manage-users.edit`, `manage-users.delete` — the middleware will check the composite key `manage-users.manage-users.{action}`

### 2026-04-10 — Phase 3: Roles v2 Backend (by Claude Code)
- **New files created:**
  - `be-anandi/src/v2/services/roleService.js` — core role CRUD + permission sync business logic
  - `be-anandi/src/v2/controllers/roleController.js` — HTTP handlers
  - `be-anandi/src/v2/routes/rbacRoles.routes.js` — routes with rbacMiddleware
- **Routes mounted at** `/api/v2/org/:orgId/rbac/...`:
  - `GET /roles` — paginated role list (search by name, iLike, level-filtered, includes userCount + actionCount)
  - `GET /roles/:roleId` — single role with all granted actions
  - `POST /roles` — create role (level must be strictly below actor's level)
  - `PATCH /roles/:roleId` — update role name/level/description (level change also enforced)
  - `DELETE /roles/:roleId` — soft delete (blocked if users still assigned)
  - `GET /roles/:roleId/permissions` — full module/action tree showing granted vs available, with `assignable` flag per action (based on what the actor has)
  - `PUT /roles/:roleId/permissions` — transactional sync of role_actions (deactivate all → upsert new); validates actor has all actions being assigned
- **All routes protected by `rbacMiddleware.requireAction()`** with action keys:
  - `manage-roles.manage-roles.view` (list, get, get permissions)
  - `manage-roles.manage-roles.create`
  - `manage-roles.manage-roles.edit` (update role + sync permissions)
  - `manage-roles.manage-roles.delete`
- **Key design decisions in roleService:**
  - Level hierarchy enforced on ALL operations (create, read, update, delete, permission sync)
  - Create: level must be **strictly below** actor's level (< N, not ≤ N)
  - Edit/View/Delete: can operate on own level and below (≤ N)
  - Delete blocked if users are still assigned to the role (must reassign first)
  - syncRoleActions validates actor has every action being assigned (downward-only permission flow)
  - getRoleModulesWithActions returns tree filtered by org_modules ceiling, with `granted` (role has it) and `assignable` (actor has it) flags per action — drives the permission editor checkbox UI
  - Audit log on every create/update/delete/syncActions
- **Action keys needed:** Super-admin must create a `manage-roles` module with actions: `manage-roles.view`, `manage-roles.create`, `manage-roles.edit`, `manage-roles.delete`

### 2026-04-10 — Phase 4: Permissions v2 Backend (by Claude Code)
- **New files created:**
  - `be-anandi/src/v2/services/permissionsService.js` — v2 `getAllControls` with `org_modules` ceiling
  - `be-anandi/src/v2/controllers/permissionsController.js` — HTTP handler
  - `be-anandi/src/v2/routes/rbacPermissions.routes.js` — route (no rbacMiddleware — this IS the permission-loading endpoint)
- **Route:** `GET /api/v2/org/:orgId/rbac/controls/:userId`
- **Key differences from v1 `getAllControls`:**
  - Filters actions through `org_modules` — if a module isn't allocated to the org, its actions are excluded
  - Walks parent chain for modules to include parent modules in the tree even if only submodules are allocated
  - Same exact response shape as v1 (modulesTree, flatModules, flatActions, user, roles) — frontend works without changes
  - Action key format preserved: `${module.key}.${action.key}`
  - `flatModules` and `flatActions` use snake_case field names (matching v1 raw DB format)
  - `config` field included on modules (needed by sidebar for route navigation)
- **No rbacMiddleware on this route** — this is the bootstrap endpoint that loads the user's permissions; requiring permissions to load permissions would be circular

### 2026-04-10 — Phase 5: Audit Log Backend (by Claude Code)
- **New files created:**
  - `be-anandi/src/v2/controllers/auditLogController.js` — paginated list with filters
  - `be-anandi/src/v2/routes/rbacAudit.routes.js` — route with rbacMiddleware
- **Route:** `GET /api/v2/org/:orgId/rbac/audit-logs?page=1&limit=50&targetType=user&action=create&actorId=5`
- **Filters:** `targetType` (exact match), `action` (iLike partial), `actorId` (exact)
- **Includes actor user info** (name, email) via join
- **Protected by** `rbacMiddleware.requireAction('audit-logs.audit-logs.view')`
- **Action key needed:** Super-admin must create an `audit-logs` module with action: `audit-logs.view`

### 2026-04-10 — Phase 6: Frontend v2 RBAC Rebuild (by Antigravity)

#### Step 6.1 — Fixed Hardcoded `orgId=4` in rbacService.js
- **Fixed 9 endpoints** that had hardcoded `org/4/rbac/...` URLs:
  - `deleteRole`, `createUser`, `getUser`, `updateUser`, `getAllUsers`, `deleteUser`, `getAllModulesActionsPermissions`, `getAllUserModulesActionsPermissions`, `getAllSchools`
- All now accept `{ orgId, ... }` as query/mutation args and use dynamic `org/${orgId}/rbac/...`

#### Step 6.2 — Added v2 RTK Query Endpoints
- **17 new v2 endpoints** added to `rbacService.js` with proper tag types for cache invalidation:
  - **Users:** `v2ListUsers`, `v2GetUser`, `v2CreateUser`, `v2UpdateUser`, `v2DeleteUser`
  - **Helpers:** `v2ListSchools`, `v2ListPrograms`, `v2ListApplicationForms`, `v2ListPotentialManagers`
  - **Roles:** `v2ListRoles`, `v2GetRole`, `v2CreateRole`, `v2UpdateRole`, `v2DeleteRole`, `v2GetRolePermissions`, `v2SyncRolePermissions`
  - **Controls:** `v2GetControls` (mirrors v1 behavior — calls `setUserPermissions` on success)
  - **Audit:** `v2ListAuditLogs`
- All use `v2/org/${orgId}/rbac/...` URL prefix
- Tag types: `v2Users`, `v2Roles`, `v2AuditLogs` for RTK Query cache auto-invalidation
- All hooks exported (both generate and lazy variants where needed)

#### Step 6.3 — Rewired ManageUsersV2 + CreateUserV2
- **ManageUsersV2:**
  - Switched from `useLazyGetAllUsersQuery` → `useLazyV2ListUsersQuery`
  - Switched from `useDeleteUserMutation` → `useV2DeleteUserMutation`
  - Added RBAC permission guards (`hasAction`) on Create, Edit, Delete buttons
  - Added **Status column** with color-coded badges (active/disabled/invited)
  - Handles both v2 response shapes for backward compatibility
- **CreateUserV2:**
  - Switched all API hooks to v2 versions
  - Added **cascading dropdowns**: Schools → Programs → Application Forms
    - Programs auto-fetch when schools are selected (`v2ListPrograms?schoolIds=1,2`)
    - Application Forms auto-fetch when programs are selected (`v2ListApplicationForms?programIds=5`)
    - Downstream selections clear when upstream changes
  - Added **Reporting Managers** multi-select dropdown (`v2ListPotentialManagers?minLevel=N`)
  - **v2 payload shape:**
    ```json
    {
      "name": "...", "email": "...", "phone": "...", "countryCode": "91",
      "roleIds": [1, 2],
      "schoolIds": [3, 4],
      "programIds": [5],
      "applicationFormIds": [10, 11],
      "reportingManagerIds": [7, 8]
    }
    ```
  - Renamed `mobileNumber` → `phone` and `school` → `schools` to match v2 backend contract
  - Uses `useV2GetUserQuery` for edit mode (fetches full user details with all allocations)

#### Step 6.4 — Rewired ManagePermissionsV2 + CreateRoleV2
- **ManagePermissionsV2:**
  - Replaced `useGetAllRolesOverviewQuery({ orgId: 4 })` → `useV2ListRolesQuery({ orgId })`
  - Replaced `useDeleteRoleMutation` → `useV2DeleteRoleMutation`
  - Added RBAC guards on Create/Edit/Delete
  - Added search-on-submit pattern (no more instant client-side filter)
  - Added Level 6 to hierarchy list (matching 6-level RBAC system)
  - Simplified table columns: Name, Level, Description, Users, Actions, Action
- **CreateRoleV2:**
  - **Removed school dropdown entirely** — roles are org-scoped in v2 (no school binding)
  - Switched to `useV2CreateRoleMutation` / `useV2UpdateRoleMutation`
  - Simplified payload to `{ name, description, level }` only
  - Removed `console.log` debug statements
  - Removed `useRef` mapping complexity (no longer needed without school)

#### Step 6.5 — Rewired ModuleWisePermissionV2
- Switched from two separate queries (`useGetAllModulesActionsPermissionsQuery` + `useGetRoleModulesWithActionsQuery`) → **single** `useV2GetRolePermissionsQuery`
  - v2 returns the full module tree with `granted` and `assignable` flags per action in one call
- Switched from `useUpdateRoleMutation` → `useV2SyncRolePermissionsMutation`
  - v2 uses `PUT /roles/:roleId/permissions` with body `{ actionIds: [...] }` (dedicated endpoint)
- **Assignable flag enforcement:** Actions the current user doesn't have are shown but **disabled** (greyed out, non-clickable) — enforces downward-only permission flow in the UI
- Module toggles only affect assignable actions (non-assignable ones are skipped)
- Removed `getUser()` cookie dependency — no longer needed (v2 validates on server side)

#### Step 6.6 — New Audit Log Page
- **New file:** `fe-anandi/src/pages/Admin/UserManagement/AuditLog/AuditLog.jsx`
  - Paginated table with 50 entries per page
  - **Filters:** Target Type dropdown (User/Role/Permission/Org), Action text search
  - **Columns:** Date, Actor (name + email), Action, Target Type (color badge), Target ID, Details
  - **Expandable details:** `<details>` element showing before/after JSON diff
  - Previous/Next pagination buttons
  - RBAC guard: `hasAction('audit-logs.audit-logs.view')` — shows "Access Denied" if missing
- **Route added:** `/admin/audit-logs` in MainRoutes.jsx

#### Key Design Decisions (Phase 6)
- **No sidebar switch to v2:** Sidebar still uses v1 `getControls` endpoint — v2 `getControls` is available but not wired to sidebar yet (risk: blank sidebar if `org_modules` not seeded)
- **OrgId fallback preserved:** `|| 1` fallback kept (was `|| 4` in some places) during transition
- **v1 hooks kept:** All v1 exported hooks preserved for backward compatibility — other parts of the app may still use them
- **Build status:** All modified files compile. Pre-existing `dompurify` import error in EmailPreviewFilter.jsx is unrelated

### 2026-04-10 — Phase 6 Addendum: Directory Restructuring (by Antigravity)

Moved all v2 RBAC page components out of `pages/Admin/UserManagement/` into a new top-level `pages/UserAccessControl/` directory to give the RBAC module its own clean namespace, separate from legacy Admin pages.

#### New Directory Structure
```
fe-anandi/src/pages/UserAccessControl/
├── ManageUsers/
│   └── ManageUsersV2.jsx        ← was pages/Admin/UserManagement/ManageUser/
├── ManageRoles/
│   └── ManagePermissionsV2.jsx  ← was pages/Admin/UserManagement/ManagePermission/
├── ModulePermissions/
│   ├── ModuleWisePermissionV2.jsx  ← was pages/Admin/UserManagement/ModuleWisePermission/
│   └── ModuleWisePermissionStyle.jsx
└── AuditLog/
    └── AuditLog.jsx             ← was pages/Admin/UserManagement/AuditLog/
```

#### What Changed
- **4 page components + 1 style file** created in new `pages/UserAccessControl/` directory
- All relative imports updated from `../../../../` (4 levels) to `../../../` (3 levels)
- `MainRoutes.jsx` imports updated to point to new locations
- **Old v2 files deleted** from `pages/Admin/UserManagement/` — only legacy v1 components remain there (`ManageUsers.jsx`, `ManagePermissions.jsx`, `ModuleWisePermission.jsx`, etc.)
- SideFilter components (`CreateUserV2`, `CreateRoleV2`) stay in `SideFilter/` — they're shared side-panel forms, not pages
- Routes remain unchanged (`/admin/manage-users`, `/admin/manage-role-and-permissions`, `/admin/audit-logs`)

---

## Current Status (as of 2026-04-13, end of session)

> **READ THIS SECTION FIRST** — it is the complete handoff for the next agent/session.

### Completed Work Summary

| Phase | Status | What was built |
|---|---|---|
| 0 — Decisions | ✅ Done | All 16 questions answered, architecture decided |
| 1 — DB Foundation | ✅ Done | 5 migrations, 5 models, rbacService.js, rbacMiddleware.js |
| Super-admin Modules | ✅ Done | Module/submodule/action CRUD + drag-drop reorder (backend + frontend) |
| Super-admin Org Creation | ✅ Done | Full RBAC bootstrap: org + admin user + role + org_modules + role_actions |
| 2 — Users v2 Backend | ✅ Done | User CRUD + helpers (schools, programs, forms, managers) |
| 3 — Roles v2 Backend | ✅ Done | Role CRUD + transactional syncRoleActions + permission tree |
| 4 — Permissions v2 Backend | ✅ Done | v2 getAllControls with org_modules ceiling |
| 5 — Audit Log Backend | ✅ Done | Paginated, filterable audit log read endpoint |
| 6 — Frontend v2 | ✅ Done | All pages rewired to v2 APIs, new audit log page, directory restructured |
| Sidebar v2 | ✅ Done | NewSidemenuV2.jsx switched to v2 getControls |
| Org module backfill | ✅ Done | Migration + super-admin UI for managing module allocations on existing orgs |

### All v2 Backend Files (new code only)

```
be-anandi/src/v2/
├── services/
│   ├── rbacService.js              ← RBAC context builder (Redis + local cache)
│   ├── platformModuleService.js     ← Super-admin module CRUD
│   ├── orgBootstrapService.js       ← Org creation with full RBAC bootstrap
│   ├── userService.js               ← User CRUD (Phase 2)
│   ├── roleService.js               ← Role CRUD + permission sync (Phase 3)
│   └── permissionsService.js        ← v2 getAllControls (Phase 4)
├── controllers/
│   ├── platformModuleController.js  ← Super-admin module endpoints
│   ├── orgController.js             ← Org CRUD
│   ├── userController.js            ← User CRUD (Phase 2)
│   ├── roleController.js            ← Role CRUD (Phase 3)
│   ├── permissionsController.js     ← getAllControls (Phase 4)
│   └── auditLogController.js        ← Audit log list (Phase 5)
├── middleware/
│   └── rbacMiddleware.js            ← requireAction() middleware
└── routes/
    ├── rbacUsers.routes.js          ← /rbac/users/*, /rbac/schools, /rbac/programs, etc.
    ├── rbacRoles.routes.js          ← /rbac/roles/*, /rbac/roles/:id/permissions
    ├── rbacPermissions.routes.js    ← /rbac/controls/:userId
    └── rbacAudit.routes.js          ← /rbac/audit-logs
```

All routes mounted at `/api/v2/org/:orgId/rbac/...` via `be-anandi/src/v2/routes/index.js`.

### All v2 Frontend Files

```
fe-anandi/src/
├── Layout/
│   └── NewSidemenuV2.jsx           ← NOW uses v2 getControls (useV2GetControlsQuery)
├── Redux/Services/
│   └── rbacService.js              ← 17 new v2 RTK Query endpoints added, 9 hardcoded orgId=4 fixed
├── pages/UserAccessControl/         ← NEW directory for all v2 RBAC pages
│   ├── ManageUsers/ManageUsersV2.jsx
│   ├── ManageRoles/ManagePermissionsV2.jsx
│   ├── ModulePermissions/ModuleWisePermissionV2.jsx
│   └── AuditLog/AuditLog.jsx
├── SideFilter/
│   ├── CreateUser/CreateUserV2.jsx  ← Rewired to v2 APIs, cascading dropdowns
│   └── CreateRole/CreateRoleV2.jsx  ← Rewired to v2 APIs, no school field
└── Routes/MainRoutes.jsx            ← Imports from pages/UserAccessControl/

super-admin/src/
├── features/modules/ui/
│   ├── ModulesPage.jsx              ← Module management (fixed: submodule actions visible)
│   └── ModuleComposerDrawer.jsx     ← Create/edit form (fixed: actionType labels, no routeParams)
└── features/organizations/ui/
    └── CreateOrganizationDrawer.jsx ← Org creation (country code dropdown, phone validation, module allocation)
```

### v2 API Surface — Complete Endpoint Map

**Super-admin (no orgId, requires `role='super_admin'`):**
| Method | Path | Handler |
|---|---|---|
| POST | `/v2/org/orgs` | Create org with RBAC bootstrap |
| GET | `/v2/org/orgs` | List all orgs |
| GET | `/v2/org/module-catalog` | Full module tree |
| POST/PATCH/DELETE | `/v2/org/modules/*` | Module CRUD + reorder |
| POST/PATCH/DELETE | `/v2/org/actions/*` | Action CRUD + reorder |

**Org-level RBAC (requires JWT + rbacMiddleware):**
| Method | Path | Action Key | Handler |
|---|---|---|---|
| GET | `/:orgId/rbac/controls/:userId` | *(none — bootstrap)* | getAllControls |
| GET | `/:orgId/rbac/users` | `manage-users.manage-users.view` | List users |
| GET | `/:orgId/rbac/users/:userId` | `manage-users.manage-users.view` | Get user |
| POST | `/:orgId/rbac/users` | `manage-users.manage-users.create` | Create user |
| PATCH | `/:orgId/rbac/users/:userId` | `manage-users.manage-users.edit` | Update user |
| DELETE | `/:orgId/rbac/users/:userId` | `manage-users.manage-users.delete` | Delete user |
| GET | `/:orgId/rbac/schools` | `manage-users.manage-users.view` | List schools |
| GET | `/:orgId/rbac/programs` | `manage-users.manage-users.view` | List programs |
| GET | `/:orgId/rbac/application-forms` | `manage-users.manage-users.view` | List forms |
| GET | `/:orgId/rbac/potential-managers` | `manage-users.manage-users.view` | List managers |
| GET | `/:orgId/rbac/roles` | `manage-roles.manage-roles.view` | List roles |
| GET | `/:orgId/rbac/roles/:roleId` | `manage-roles.manage-roles.view` | Get role |
| POST | `/:orgId/rbac/roles` | `manage-roles.manage-roles.create` | Create role |
| PATCH | `/:orgId/rbac/roles/:roleId` | `manage-roles.manage-roles.edit` | Update role |
| DELETE | `/:orgId/rbac/roles/:roleId` | `manage-roles.manage-roles.delete` | Delete role |
| GET | `/:orgId/rbac/roles/:roleId/permissions` | `manage-roles.manage-roles.view` | Get role permission tree |
| PUT | `/:orgId/rbac/roles/:roleId/permissions` | `manage-roles.manage-roles.edit` | Sync role actions |
| GET | `/:orgId/rbac/audit-logs` | `audit-logs.audit-logs.view` | List audit logs |

### What MUST be done next (in order of priority)

**1. Create action keys in super-admin (PREREQUISITE for everything else)**
The v2 RBAC middleware blocks access unless these action keys exist in the DB and are granted to the user's role. Use the super-admin `/modules` page to create:

| Module | Key | Actions to create (key → actionType) |
|---|---|---|
| Manage Users | `manage-users` | `manage-users.view` → custom, `manage-users.create` → custom, `manage-users.edit` → custom, `manage-users.delete` → custom |
| Manage Roles | `manage-roles` | `manage-roles.view` → custom, `manage-roles.create` → custom, `manage-roles.edit` → custom, `manage-roles.delete` → custom |
| Audit Logs | `audit-logs` | `audit-logs.view` → custom |

After creating these modules, when you create a new org and select these modules, the Admin role will automatically get all the action keys. The admin user can then access the user/role management pages.

**2. ✅ Backfill `org_modules` for legacy orgs — DONE**
Both solutions implemented:
- **Migration** `20260413000000-backfill-org-modules.cjs`: inserts all active modules for all existing orgs (run once on staging/production)
- **Super-admin UI**: `/organizations/:slug/modules` page — super admin can view and edit module allocations for any org at any time
  - `GET /api/v2/org/:orgId/org/modules` — returns catalog + current allocatedModuleIds
  - `PUT /api/v2/org/:orgId/org/modules` — atomically replaces allocation, auto-grants new actions to Admin role

**3. End-to-end testing**
Test the full flow:
1. Super-admin: create modules + action keys (step 1 above)
2. Super-admin: create an org (select modules, fill admin user)
3. Admin: log into admin portal → verify sidebar shows only allocated modules
4. Admin: create a role (level < 6) → assign permissions
5. Admin: create a user → assign role, schools, programs, forms, managers
6. Admin: view audit log
7. Admin: edit/delete user and role → verify level hierarchy enforcement
8. Super-admin: use `/organizations/:slug/modules` page to add/remove modules for a legacy org → verify sidebar updates after re-login

**4. Legacy v1 cleanup (low priority)**
- Old components in `pages/Admin/UserManagement/` (v1 ManageUsers, ManagePermissions, etc.) can be deleted once v2 is validated
- Old v1 `getControls` endpoint and related code can be deprecated

### Key Architecture Notes for Next Agent

- **Convention:** New RBAC service/middleware/controller files use `const` functions + `export default { ... }` pattern. Do NOT use `export const`.
- **Convention:** DB column names and Sequelize field names use same case. v2 models (UserProgram, UserApplicationForm, UserReportingManager, AuditLog) use snake_case fields directly. Old models (User, OrgUser, Role, UserRoles, UserSchools) use camelCase with `field:` mappings.
- **Role hierarchy:** 6 levels (1–6). Higher = more privileged. Level 6 = org Admin. Create roles strictly below your level. Edit/view/delete at your level or below.
- **Permission flow:** Downward-only. A user can only grant permissions they themselves possess. `rbacService.assertAssignableActionKeys()` enforces this.
- **Data visibility vs feature access:** School/Program/Form allocations are per-user (controls which leads/data they see). CRM feature access is role-based (module/action permissions).
- **Action key format:** `${module.key}.${action.key}` — the module key is prepended by `getAllControls` when building the response. So in the DB, an action's key might be `manage-users.view`, and the frontend checks `hasAction('manage-users.manage-users.view')`.
- **Frontend RBAC:** `useRBACPermissions()` hook → `hasAction('module.submodule.action')` for permission checks. Driven by `rbacSlice.actionKeyMap`.
- **Redux state:** `rbacSlice.actionKeyMap` is the source of truth for permissions; `authSlice.organizationId` is the dynamic orgId.
- **v2 API base:** All v2 endpoints use `/api/v2/org/${orgId}/rbac/...` URL prefix.
- **Super-admin panel:** Runs on separate port (typically `localhost:3000`). Repo: `super-admin/`.
- **Admin portal:** Runs on `localhost:7005`. Repo: `fe-anandi/`.
- **Backend:** Runs on the port configured in `be-anandi`. Repo: `be-anandi/`.
- **Pre-existing build issue:** `dompurify` import error in `EmailPreviewFilter.jsx` blocks production build — unrelated to RBAC work.

### 2026-04-13 — Bug fixes: migration + updateOrgModules + Org Users page

**Bugs fixed:**

1. **Migration silent no-op** (`20260413000000-backfill-org-modules.cjs`)
   - Root cause: `const [[modules], [orgs]]` with `QueryTypes.SELECT` — Sequelize returns bare row arrays (not `[rows, meta]`) with SELECT type, so `Promise.all` gives `[rowsArray1, rowsArray2]` and the destructuring pulled out the first element of each (a single `{id: X}` object). `!{id:X}.length = !undefined = true` → early return. Nothing inserted.
   - Fix: `const [modules, orgs]`

2. **Migration didn't bootstrap role_actions** — Even with org_modules populated, legacy orgs whose Admin role never had `role_actions` rows still get a blank sidebar (`getAllControls` finds zero role actions). The migration now also runs:
   ```sql
   INSERT INTO role_actions (role_id, action_id, is_active, ...)
   SELECT r.id, a.id, true, ...
   FROM roles r CROSS JOIN actions a
   WHERE r.level = 6 AND r.is_active = true AND a.is_active = true
   ON CONFLICT (role_id, action_id) DO UPDATE SET is_active = true
   ```
   This grants all active actions to all level-6 Admin roles, idempotently.

3. **`updateOrgModules` used `updateOnDuplicate`** — Sequelize v6 on PostgreSQL has unreliable behavior with `updateOnDuplicate` on compound unique indexes. Changed `OrgModule.update + bulkCreate(updateOnDuplicate)` → `OrgModule.destroy + bulkCreate`. Changed `RoleActions.bulkCreate(updateOnDuplicate)` → `RoleActions.update(reactivate existing) + bulkCreate(ignoreDuplicates)`.

4. **v2 `getAllControls` crashed for legacy-style timestamp fields** — The service selected `UserRoles.createdAt` and mixed camelCase model fields with snake_case/raw result access while building the module tree. On PostgreSQL this caused `column UserRoles.createdAt does not exist`, which meant the admin portal sidebar stayed blank even though `org_modules` and `role_actions` were present.
   - Fix: use `created_at` in the `UserRoles` query, normalize `orgId` / `userId`, and consistently read module/action fields using the Sequelize model field names before shaping the final snake_case response.

5. **Super-admin Org Modules page showed saved modules as unchecked after refresh** — The backend returned `allocatedModuleIds` as numbers while the module catalog tree IDs were strings, so `selectedIds.includes(module.id)` failed in the UI and the page looked like nothing had persisted.
   - Fix: normalize catalog IDs and `allocatedModuleIds` to numbers in `OrgModulesPage.jsx`.

6. **Super-admin Org Users page failed to load** — `getOrgUsers` selected `user.createdAt` and ordered by `OrgUser.createdAt`, which caused PostgreSQL errors (`column user.createdAt does not exist` / `column OrgUser.createdAt does not exist`) and surfaced in the UI as “Failed to load users.”
   - Fix: switch the query to `created_at` for both the included `User` attributes and the `OrgUser` ordering, then map the response back to `createdAt` for the frontend.

**Re-run steps for test1 org:**
1. `sequelize db:migrate:undo --name 20260413000000-backfill-org-modules.cjs` (if run previously)
2. `sequelize db:migrate --name 20260413000000-backfill-org-modules.cjs`
3. Restart be-anandi
4. Log in as admin — sidebar should now show all modules

**New feature — Org Users page:**

### 2026-04-13 — Org Module Management (backfill + super-admin UI)

**Problem solved:** v2 `getAllControls` enforces an `org_modules` ceiling — orgs created before v2 had no rows and got a blank sidebar.

**Two solutions shipped:**

1. **Migration** `be-anandi/src/migrations/20260413000000-backfill-org-modules.cjs`
   - Inserts an `org_modules` row for every (org × active module) pair that does not already exist
   - Gives all legacy orgs unrestricted access to all platform modules (equivalent to pre-v2 behaviour)
   - Roll-back: `bulkDelete` clears the table

2. **Super-admin org module management UI** (`/organizations/:slug/modules`)
   - **Backend:** `GET /api/v2/org/:orgId/org/modules` returns full module catalog + `allocatedModuleIds`
   - **Backend:** `PUT /api/v2/org/:orgId/org/modules` atomically replaces the org's module allocation
     - Uses same expansion logic as org creation (`expandSelectedModuleIds`)
     - Auto-grants all actions from newly added modules to the org's level-6 Admin role
     - Does NOT revoke Admin role actions when modules are removed (the RBAC ceiling check blocks them silently)
     - Writes audit log entry (`org.modules.update`)
   - **Frontend:** `OrgModulesPage.jsx` at `super-admin/src/features/organizations/ui/OrgModulesPage.jsx`
     - Checkbox tree with Select All, parent → child cascade (same UX as org creation drawer)
     - Pre-populates from current `allocatedModuleIds`
     - Save button disabled until user makes changes
   - **Route:** `/organizations/:orgSlug/modules` added to `AppRoutes.jsx`
   - **Hub card:** "Module Access" card added to `OrganizationDetailPage` hub as the first card
   - **API:** `useGetOrgModulesQuery` and `useUpdateOrgModulesMutation` added to `organizationsApi.js`; `OrgModules` tag type added to `baseApi.js`

**New backend files modified:**
- `be-anandi/src/v2/controllers/orgController.js` — added `getOrgModules`, `updateOrgModules`, and private `_buildModuleMaps` / `_expandSelectedModuleIds` helpers
- `be-anandi/src/v2/routes/org.routes.js` — added `GET /modules` and `PUT /modules`
- `be-anandi/src/migrations/20260413000000-backfill-org-modules.cjs` — new backfill migration
- `be-anandi/src/v2/services/permissionsService.js` — fixed v2 controls query field mapping / timestamp access so sidebar loads correctly

**New feature — Org Users page** (`/organizations/:slug/users`):
- Backend: `GET /api/v2/org/:orgId/org/users` (requires `role='super_admin'`) — lists all non-removed users in org with their roles + isPrimary flag (the Primary Admin is the user bootstrapped at org creation)
- Frontend: `OrgUsersPage.jsx` — table showing Name, Email, Phone, Roles (with level badges), Status
- "Primary Admin" badge highlights the user created during org bootstrap
- "Users" hub card added as the first card on the org detail page
- Route: `/organizations/:orgSlug/users` in AppRoutes.jsx

**New super-admin files modified:**
- `super-admin/src/features/organizations/ui/OrgModulesPage.jsx` — new page
- `super-admin/src/features/organizations/index.js` — exported `OrgModulesPage`
- `super-admin/src/features/organizations/application/organizationsApi.js` — added two endpoints + hooks
- `super-admin/src/features/organizations/ui/OrgUsersPage.jsx` — new page
- `super-admin/src/features/organizations/ui/OrganizationDetailPage.jsx` — added Users + Module Access hub cards
- `super-admin/src/features/organizations/ui/sections/ManagementHub.jsx` — added `modules` route resolver
- `super-admin/src/app/routes/AppRoutes.jsx` — added `/organizations/:orgSlug/modules` route
- `super-admin/src/infrastructure/api/baseApi.js` — added `OrgModules` to tagTypes

### 2026-04-13 — Post-handoff stabilization fixes

- Confirmed on live DB that org `test1` already had:
  - `org_modules` rows populated
  - a level-6 Admin role
  - active `role_actions`
  - admin user `test@admin46.com` mapped to that role
- Root cause of the still-blank admin sidebar was therefore not missing seed data, but a crashing v2 controls query in `permissionsService.js`.
- Confirmed after the fix that `permissionsService.getAllControls({ orgId: 14, userId: 4441057 })` returns a non-empty response for the test admin:
  - `roleCount: 1`
  - `moduleCount: 44`
  - `flatActionsCount: 185`
- Confirmed after the Org Users fix that `GET /api/v2/org/14/org/users` returns the expected user row for `test@admin46.com` with `Admin (L6)` and `isPrimary: true`.

### 2026-04-17 — Continuation fixes to keep code aligned with this document

- This document must be treated as the active continuation spec for the v2 Users / Roles / Permissions rebuild. The following fixes were made specifically to bring the current code back into line with the documented design and frontend expectations.

**Backend / frontend contract fixes completed:**

1. **v2 user create/update payload mismatch fixed**
   - `fe-anandi/src/SideFilter/CreateUser/CreateUserV2.jsx` sends:
     - `applicationFormIds`
     - `reportingManagerIds`
   - But `be-anandi/src/v2/services/userService.js` only read:
     - `formIds`
     - `managerIds`
   - Result: application-form and reporting-manager assignments from the v2 user drawer were silently dropped.
   - Fix: `userService.createUser()` and `userService.updateUser()` now accept the frontend payload names and map them into the backend sync flow.

2. **v2 helper endpoint response shape flattened to match frontend usage**
   - `CreateUserV2.jsx` expects raw arrays from:
     - `GET /rbac/schools`
     - `GET /rbac/programs`
     - `GET /rbac/application-forms`
     - `GET /rbac/potential-managers`
     - `GET /rbac/users/:userId`
   - But `be-anandi/src/v2/controllers/userController.js` was wrapping these as `{ schools }`, `{ programs }`, `{ forms }`, `{ managers }`, and `{ user }`.
   - Result: dropdowns and edit-prefill logic could appear empty even when service data existed.
   - Fix: controller responses were flattened so `sendSuccess()` returns the raw arrays/object shapes the v2 frontend already consumes.

3. **Application form lookup fixed for real model fields**
   - `ApplicationForm` uses `title`, `applicationFormName`, and `organizationId`.
   - `userService.listApplicationForms()` was querying `orgId` and selecting `name`, which does not match the actual model definition.
   - Result: form dropdown data could be empty or shaped incorrectly.
   - Fix:
     - query now filters on `organizationId`
     - selected attributes now use `title` + `applicationFormName`
     - response now exposes a normalized `name` field for the existing frontend dropdown code

4. **v2 getUser application-form payload normalized**
   - `userService.getUser()` previously returned `applicationForms: [{ id, name }]`, but the actual form model does not provide a `name` column.
   - Fix: the response now returns:
     - `id`
     - `title`
     - `applicationFormName`
     - normalized `name`
   - This preserves compatibility with the current edit drawer while staying faithful to the real DB/model structure.

**RBAC ceiling / module-tree fixes completed:**

5. **Org-module ceiling enforcement now uses full ancestor traversal, not only immediate parent checks**
   - The architecture in this document assumes `org_modules` acts as a ceiling across the entire module tree.
   - Parts of the v2 implementation were only checking:
     - action module itself
     - immediate parent module
   - That is insufficient for deeper nesting and can incorrectly hide valid actions or allow inconsistent permission-tree behavior.
   - Fix:
     - `be-anandi/src/v2/services/permissionsService.js` now computes ancestry through the full module chain
     - `be-anandi/src/v2/services/roleService.js` now uses the same ancestor-based logic for role-permission editing

6. **Role permission sync now rejects actions outside the org's allocated module tree**
   - `syncRoleActions()` previously validated existence and actor ownership of action keys, but did not robustly enforce the full `org_modules` ceiling for nested module ancestry.
   - Fix: requested actions are now rejected if none of their module ancestors are granted to the org.

7. **Role permission tree visibility now follows ancestor-based org allocation**
   - `getRoleModulesWithActions()` now includes modules based on full ancestor allocation, matching the intended architecture and the behavior expected by the permission editor UI.

**Files updated in this continuation:**
- `be-anandi/src/v2/services/userService.js`
- `be-anandi/src/v2/controllers/userController.js`
- `be-anandi/src/v2/services/roleService.js`
- `be-anandi/src/v2/services/permissionsService.js`

**Verification completed:**
- Ran `node --check` successfully on:
  - `be-anandi/src/v2/services/userService.js`
  - `be-anandi/src/v2/services/roleService.js`
  - `be-anandi/src/v2/services/permissionsService.js`
  - `be-anandi/src/v2/controllers/userController.js`

**What still needs validation after these fixes:**
1. Open the admin portal and test the full Create User flow:
   - select schools → programs → forms
   - assign reporting managers
   - save user
   - reopen in edit mode and verify values prefill correctly
2. Open Module Wise Permission page for a role and verify:
   - nested modules appear correctly
   - assignable/granted actions behave correctly
   - saving permissions works for orgs using nested module trees
3. Re-test sidebar output after login for an org whose module allocation relies on parent-level grants covering nested descendants

### 2026-04-17 — Super-admin organization card cleanup

- Updated the `/organizations` card UI in `super-admin/src/features/organizations/ui/sections/OrganizationsGrid.jsx` so the 3-dot actions menu sits in the top identity row beside the organization name instead of at the bottom of the card.
- Removed low-signal card metadata text from the list view:
  - `Email`
  - `Features`
  - `Created Recently`
  - `Active`
- Added organization-card menu actions for:
  - `Edit`
  - `View Modules`
  - `View Primary Role`
  - `View Primary User`
- Fixed the organization-card dropdown clipping issue by allowing the card container to render the menu outside its border bounds.
- Build verification: `super-admin` `npm run build` passes after the card update.

### 2026-04-20 — Legacy RBAC key compatibility expanded

- The legacy/v2 permission bridge was expanded again for orgs whose stored action keys use module-prefixed legacy forms such as:
  - `manage-users.users.create`
  - `manage-users.users.view`
  - `manage-users.manage-users.edit-user`
  - `manage-users.manage-users.delete-user`
  - `manage-roles.roles.view`
  - `manage-roles.roles.assignPermissions`
- This update was applied in both:
  - `be-anandi/src/v2/services/rbacService.js`
  - `fe-anandi/src/hooks/useRBACPermissions.js`
- Purpose: prevent valid legacy permissions from being hidden in the admin UI or rejected by v2 RBAC checks solely because the stored key format differs from the new canonical key shape.

### 2026-04-20 — v2 created-user password generation aligned with bootstrap rule

- `be-anandi/src/v2/services/userService.js` now passes the created user's name into `generatePassword(...)` during v2 user creation.
- Result: users created from the admin portal now follow the same password rule already used for org-bootstrap admins:
  - `first word of name + @123`
- Before this fix, v2-created users could incorrectly fall back to the default generator output because the name argument was not being passed.

### 2026-04-20 — Direct-reports helper made org-optional and include-driven

- `userService.listUsersManagedByUser(...)` no longer depends on `orgId`; if `orgId` is absent or invalid, it still returns the manager's direct reports across all org relationships found in `user_reporting_managers`.
- The helper/API is now include-driven for heavier relationship payloads. Base user fields are returned by default, and the following are opt-in booleans:
  - `organization`
  - `membership`
  - `roles`
  - `schools`
  - `programs`
  - `applicationForms`
- `userController.listUsersManagedByUser` now reads those flags from request body or query params and forwards them into the service.
- Purpose: keep direct-reports queries fast by default and load heavier relationship data only when explicitly requested.
- The API route for direct reports is now non-org-scoped to match the helper contract:
  - `POST /api/v2/org/users/:userId/direct-reports`
  - It is no longer mounted under `/:orgId/rbac/...`

### 2026-04-17 — Org Users page narrowed to the primary-role cohort

- `GET /api/v2/org/:orgId/org/users` and the super-admin route `/organizations/:orgSlug/users` now represent the organization's bootstrap ownership layer rather than the full org user list.
- The backend resolves the target cohort as users assigned to the org's earliest active level-6 `admin` role.
- The page/API now expose additional operational context for those users:
  - country name derived from `countryIso`
  - `lastActiveAt`
  - `loginCount`
  - joined timestamp

### 2026-04-17 — Reporting-manager direct reports helper added

- Added `userService.listUsersManagedByUser({ managerUserId, ...includeFlags })` in `be-anandi/src/v2/services/userService.js`.
- It returns active direct-report users for a reporting-manager user id without requiring an `orgId`.
- Default response is intentionally lean for reuse inside heavier query/computation flows.
- Extra relationship payloads are opt-in through boolean flags:
  - `organization`
  - `membership`
  - `roles`
  - `schools`
  - `programs`
  - `applicationForms`
- Current debug/consumption route in code:
  - `POST /api/v2/org/users/:userId/direct-reports`
  - controller method: `userController.listUsersManagedByUser`
  - no `orgId` path parameter because the helper is intentionally manager-id scoped

### 2026-04-17 — Admin login contract simplified and sidebar root cause fixed

- The current admin login flow no longer relies on the incoming request payload `role` array for user lookup.
- Backend login now resolves by email and blocks admin-portal access for:
  - `super_admin`
  - `student`
- When `rememberMe=true` is sent to the admin login flow, JWT validity and auth-cookie lifetime are now extended to `30d` instead of the default `1d`.
- This keeps counsellor/admin access possible while removing the previous client-driven role filter from the login contract.
- Organization-bootstrap admin password generation is now deterministic from the provided admin name:
  - format: `first_word_of_admin_name + @123`
  - example: `Test Admin 2` → `Test@123`
- Root cause of the org-admin blank sidebar for org `14` was not missing RBAC seed data; the org had:
  - active admin user
  - active primary org membership
  - level-6 Admin role
  - active `user_roles`
  - active `org_modules`
  - active `role_actions`
- The actual failure was in `be-anandi/src/v2/services/permissionsService.js`, where the module query still selected `createdAt` / `updatedAt` instead of the real `created_at` / `updated_at` DB columns.
- After fixing that query, `permissionsService.getAllControls({ orgId: 14, userId: 4441057 })` returns a non-empty result again:
  - `roleCount: 1`
  - `modulesTreeCount: 20`
  - `flatModulesCount: 44`
  - `flatActionsCount: 185`

### 2026-04-17 — Org module allocation semantics corrected

- The org-module save path previously re-expanded any selected parent module into all descendant submodules on the backend.
- Result: unchecking a submodule in the super-admin org module screen could be silently undone during save, and that submodule could still appear in the admin sidebar after re-login.
- Fixed in:
  - `be-anandi/src/v2/controllers/orgController.js`
  - `be-anandi/src/v2/services/orgBootstrapService.js`
  - `be-anandi/src/v2/services/permissionsService.js`
  - `be-anandi/src/v2/services/roleService.js`
- New rule:
  - explicitly selected modules grant their own actions
  - ancestor modules are still persisted for structural/tree continuity
  - ancestor allocation alone does **not** grant descendant submodule actions
- Practical note for orgs saved before this fix:
  - previously over-expanded submodule rows may already exist in `org_modules`
  - re-save the org’s Module Access once after this patch so the persisted allocation matches the intended checkbox state

### 2026-04-20 — Legacy role-permission compatibility bridge added

- Existing orgs in the dev DB still carry the older role-management action family:
  - `roles.view`
  - `roles.assignPermissions`
  - `manage-roles-permissions.edit-role`
  - `manage-roles-permissions.delete-role`
  - `manage-roles-permissions.edit-permissions`
- But the v2 roles page and v2 RBAC routes were checking the newer permission family:
  - `manage-roles.manage-roles.view/create/edit/delete`
- Result: `/admin/manage-role-and-permissions` could hide the Create Role button and legacy org admins could fail v2 role-route authorization even though they still had the old role-management permissions.
- Fixed via a compatibility bridge in:
  - `be-anandi/src/v2/services/rbacService.js`
  - `fe-anandi/src/hooks/useRBACPermissions.js`
- Current behavior:
  - v2 frontend guards and backend `rbacMiddleware` accept both the new role-management keys and the legacy role-management keys for existing orgs
  - this keeps legacy orgs working without forcing an immediate permission reseed before role management can be used

### 2026-04-20 — Legacy user-management permission compatibility added

- Existing orgs in the dev DB still use the older user-management action family:
  - `users.view`
  - `users.create`
  - `manage-users.edit-user`
  - `manage-users.delete-user`
- But the v2 users page and v2 RBAC routes check the newer permission family:
  - `manage-users.manage-users.view/create/edit/delete`
- Result: `/admin/manage-users` could fail with `Missing required permission: manage-users.manage-users.view` for legacy org admins even though they still had the older user-management permissions.
- Fixed via the same compatibility bridge pattern in:
  - `be-anandi/src/v2/services/rbacService.js`
  - `fe-anandi/src/hooks/useRBACPermissions.js`
- Current behavior:
  - v2 frontend guards and backend `rbacMiddleware` accept both the new user-management keys and the legacy user-management keys for existing orgs

### 2026-04-20 — Org-level action ceiling added for super-admin module allocation

- The earlier org-allocation system only persisted `org_modules`, so super-admin could check/uncheck modules and submodules but had no canonical way to allow/deny individual actions inside those modules.
- Added a new org-level action ceiling via:
  - migration: `be-anandi/src/migrations/20260420090000-create-org-actions.cjs`
  - model: `be-anandi/src/models/OrgAction.js`
- Backend contract changes:
  - `GET /api/v2/org/:orgId/org/modules` now returns:
    - `catalog`
    - `allocatedModuleIds`
    - `allocatedActionIds`
  - `PUT /api/v2/org/:orgId/org/modules` now accepts:
    - `moduleIds`
    - `actionIds`
- Enforcement changes:
  - `permissionsService`, `rbacService`, and `roleService` now respect an org-level action ceiling when `org_actions` rows exist
  - backward compatibility is preserved: if an org has no `org_actions` rows yet, the system still treats all active actions under allocated modules as allowed
- Bootstrap/default behavior:
  - new org creation now seeds `org_actions` with all active actions under the selected modules
  - the super-admin org Module Access page now renders action checkboxes beneath modules/submodules and saves both module and action allocation

### 2026-04-21 — Admin login response user contract cleaned up

- `POST /api/users/auth/adminLoginV2` no longer returns the legacy `role` field inside `data.user`.
- `data.user` also no longer carries `organizationId`; org identity belongs to `data.organization`.
- `data.organization` is intentionally lightweight and currently carries:
  - `id`
  - `name`
- The response now includes user profile fields needed by the admin frontend header/profile state:
  - `countryIsoCode`
  - `image`
- The JWT payload may still include the legacy role for middleware/backward compatibility, but frontend login state should not depend on `data.user.role`.
- `fe-anandi/src/pages/Auth/LoginComponents/Signin.jsx` now derives legacy Redux flags without reading `user.role`:
  - counsellor is inferred from the presence of the login response `counsellor` object
  - admin-portal login defaults to admin-style legacy flags when a token exists and the user is not a counsellor
- `authSlice` now persists `countryIsoCode` from the login response alongside existing user display fields.
- Admin frontend org resolution now reads organization id from the organization object instead of `user.organizationId`.
- RBAC action keys remain the source of truth for actual access control.

### 2026-04-21 — Admin login organization/role selection flow added

- `POST /api/users/auth/adminLoginV2` now supports a staged login flow for users who belong to multiple organizations and/or have multiple roles.
- Stage 1 verifies email/password. If selection is needed, it returns no final JWT and instead returns:
  - `selectionRequired: true`
  - `selectionStep: "organization"` or `"role"`
  - short-lived `selectionToken` valid for login continuation
  - lightweight organization or role options
- Stage 2 sends the `selectionToken` plus selected `organizationId` and, when needed, selected `roleId`.
- Final successful login now represents exactly one organization and exactly one role:
  - JWT contains `organizationId` and `selectedRoleId`
  - response `organization` is `{ id, name }`
  - response `userRoles` is a single role object `{ id, name, level }`, not an array
- Backend RBAC now respects selected role context:
  - `jwtValidator` exposes `req.user.selectedRoleId`
  - `rbacMiddleware` builds RBAC context for the selected role only
  - `permissionsService.getAllControls()` filters controls to the selected role
  - RBAC context cache keys include selected role id
- Admin frontend changes:
  - `Signin.jsx` shows org and role selection popups when required
  - `authSlice` stores `userRole` and `selectedRoleId`
  - `NewSidemenuV2` includes selected role in the controls query so sidebar permissions match the chosen login role

### 2026-04-21 — Super-admin login separated from admin portal login

- Added dedicated backend endpoint:
  - `POST /api/users/auth/superAdminLogin`
- Purpose:
  - only users with `users.role = 'super_admin'` can log into the separate `super-admin/` frontend
  - admin-portal users, counsellors, students, and normal org users are rejected from this endpoint
- Existing admin portal endpoint remains intentionally org/role scoped:
  - `POST /api/users/auth/adminLoginV2`
  - this endpoint rejects `super_admin` and `student` users
- `super-admin/src/features/auth/application/authApi.js` now uses `/api/users/auth/superAdminLogin` and no longer sends the old role-array payload.
- This keeps platform-level super admin access separate from org-level admin portal access.

### 2026-04-21 — Admin org context compatibility after lightweight login response

- Since admin login now returns `organization: { id, name }` only, the admin frontend can no longer rely on `organization.internalId` being present after login.
- `Signin.jsx` now writes the selected numeric organization id into the existing `crm_org_id` compatibility cookie when no internal id/slug is available.
- `dashboardService.js` was updated so dashboard stats use the active route `dashboard/counts` and send selected org context through `x-org-id`.
- This prevents legacy frontend services that still call `getOrgId()` from failing immediately after the lighter login response, while v2 pages continue to use Redux `organizationId` directly.

### 2026-04-21 — Current user data-scope context added

- Login remains lightweight; school/program/application-form allocations are not added to `adminLoginV2`.
- Added a dedicated current-user context endpoint:
  - `GET /api/v2/org/:orgId/rbac/me/context`
- The endpoint returns only the logged-in user's data-visibility scope:
  - `schools`
  - `programs` with `schoolId`
  - `applicationForms` with `programId` and `schoolId`
- Added frontend slice:
  - `fe-anandi/src/Redux/Slices/currentUserContextSlice.js`
- `App.jsx` fetches this context after login and stores it in `currentUserContextState`.
- Future data-visibility logic should read assigned schools/programs/forms from `currentUserContextState`, while sidebar/action access should continue to come from `rbacSlice`.

### 2026-04-21 — User profile moved into v2 route-controller-service flow

- `/admin/profile` now uses v2 profile APIs:
  - `GET /api/v2/org/:orgId/profile/me`
  - `PATCH /api/v2/org/:orgId/profile/me`
- Profile business logic lives in `be-anandi/src/v2/services/userProfileService.js`; future admin, super-admin, and student profile work should be added through v2 route → controller → service files, not the legacy `controllers/userProfile` controller.
- Legacy admin profile endpoints under `/api/userProfile/*` are no longer mounted for this flow. Admin profile read/update must use the v2 profile routes only.
- The v2 profile response includes profile fields, selected organization, selected role(s), assigned schools, assigned programs with `schoolId`, and assigned application forms with `programId` and `schoolId`.
- Profile `roles` now represents all active roles assigned to the user in the active organization. The current login/session role is exposed separately as `selectedRole` and must not be used to hide other assigned roles on the profile screen.
- The profile page must not render persisted Redux/auth values as profile content on first load. It shows a loader until `GET /api/v2/org/:orgId/profile/me` completes for the active org, then renders the API response.
- Profile updates support name, phone, country/mobile code, and profile image. The v2 update path runs inside a Sequelize transaction and reads the composed profile response back from the same transaction. Filestack remains the image uploader, and saved profile changes sync back into persisted auth Redux so the header avatar updates immediately.

### 2026-04-22 — Org module allocation now supports legacy Admin roles

- Super-admin Module Access can now grant selected org actions to the org Admin role even when a legacy org's Admin role is `internalId='admin'` but not level 6.
- `updateOrgModules()` now resolves the Admin role by `internalId='admin'` first, with level-6 as the v2 fallback, then invalidates RBAC context cache for users assigned to that role.
- `permissionsService.getAllControls()` now applies the org-module ceiling through full module ancestry instead of only the action module's direct id, keeping sidebar visibility aligned with nested module allocation.
- Masters Union (`org_id=12`) had `org_actions` populated but `0` active `role_actions`; the existing Admin role was repaired by granting its currently selected org actions so admin sidebar controls now return non-empty modules/actions.

### 2026-04-23 — Org module allocation now mirrors Admin role actions exactly

- Super-admin Module Access saves now treat selected actions as the exact active action set for the organization's Admin role.
- `updateOrgModules()` still grants/reactivates selected actions, but now also deactivates Admin-role `role_actions` that were removed from the org allocation.
- This keeps the platform permission chain consistent: selected org modules/actions → org Admin role → admin portal sidebar/API permissions, without stale Admin-role actions lingering after super-admin unchecks access.

### 2026-04-23 — Organization bootstrap Admin credential email finalized

- When super-admin creates an organization, the bootstrapped primary Admin user is sent an onboarding email after the DB transaction commits.
- The email includes the admin portal URL, login email, generated password, assigned Admin role/level, and support contact; internal organization id/slug are intentionally not shown to the user.
- The bootstrap mail points at the admin portal and no longer uses the student frontend URL.
- The bootstrap Admin credential email explicitly sends as `Lead Matrix <no-reply@mastersunion.org>` by passing sender overrides through `emailService.sendTemplate()`.
- Super-admin org creation now includes a `sendAdminEmail` option. It defaults to `true`; when false, the Admin user is still created but the credential email is skipped.
- The email remains non-blocking: organization creation succeeds even if email delivery fails, and failures are logged for follow-up.

### 2026-04-24 — v2 action policies for backend data scoping

- The old CRM-style `user.userPolicies["policy:key"]` data-scoping pattern is now supported through the existing v2 RBAC action system instead of a separate `policies` / `role_policies` table.
- Source of truth remains:
  - `actions`
  - `org_actions`
  - `role_actions`
  - `user_roles`
- `rbacService` now exposes helper functions for service-layer checks:
  - `rbacService.userHasAction(userOrContext, actionKey)`
  - `rbacService.userHasAnyAction(userOrContext, actionKeys)`
  - `rbacService.userHasAllActions(userOrContext, actionKeys)`
  - `rbacService.buildActionPolicyMap(rbacContext)`
  - `rbacService.attachActionPoliciesToUser(user, rbacContext)`
- `rbacMiddleware.requireAction()` now attaches a policy-like map to `req.user` after building RBAC context:
  - `req.user.userPolicies`
  - `req.user.actionPolicies`
  - `req.user.allowedActionKeys`
- This gives old-CRM developers a familiar usage style while keeping one RBAC source of truth:
  ```javascript
  const canViewAllLeads = rbacService.userHasAction(
    actor,
    'manage-leads.manage-leads.view-all',
  );
  ```
- The attached `userPolicies` map includes compatible action-key aliases, so direct old-style checks also work for full composite keys:
  ```javascript
  const canViewAllLeads = Boolean(
    actor?.userPolicies?.['manage-leads.manage-leads.view-all'],
  );
  ```
- Route guards and data scoping are intentionally separate:
  - route guard: `rbacMiddleware.requireAction('manage-leads.manage-leads.view')`
  - service query scope: check actions such as `manage-leads.manage-leads.view-all`, `view-team`, or `view-own`
- Default data access should remain conservative. If the actor does not have the explicit broad-scope action, services should apply own/team/reporting-chain filters instead of returning all organization records.
- **Action keys for data scoping (e.g. `view-all`) are NOT seeded via migrations.** Developers create them dynamically via the super-admin `/modules` page like any other action key. No code change or migration is needed to introduce a new scoping capability — just add the action in super-admin, assign it to the relevant roles, and check it in the service.

### 2026-04-24 — v2 canonical RBAC action keys migration

- Migration `be-anandi/src/migrations/20260424090000-seed-v2-rbac-action-keys.cjs` added.
- Creates two new submodules under `uac` (id: 9):
  - `manage-roles` (key: `manage-roles`, route: `/admin/manage-role-and-permissions`, sort_order: 3)
  - `audit-logs` (key: `audit-logs`, route: `/admin/audit-logs`, sort_order: 4)
- Adds 9 v2 canonical action keys:
  - `manage-users.view`, `manage-users.create`, `manage-users.edit`, `manage-users.delete` → on module `manage-users` (id: 10)
  - `manage-roles.view`, `manage-roles.create`, `manage-roles.edit`, `manage-roles.delete` → on new `manage-roles` module
  - `audit-logs.view` → on new `audit-logs` module
- These are the keys the v2 `rbacMiddleware` checks (composite form: `manage-users.manage-users.view` etc.)
- The legacy seeded keys (`users.view`, `users.create`, `roles.view`, `roles.assignPermissions`) remain in place — covered by the existing RBAC compatibility bridge.
- Migration is fully idempotent (`ON CONFLICT … DO NOTHING` on all inserts).
- After inserting, the migration backfills:
  - `org_modules` for the two new modules on all existing orgs
  - `org_actions` for the 9 new actions on orgs that already have `org_actions` rows
  - `role_actions` granting all 9 actions to every active level-6 Admin role

### 2026-04-24 — listUsersManagedByUser made fully recursive (BFS)

- `be-anandi/src/v2/services/userService.js` — `listUsersManagedByUser` was previously a single-level query: `WHERE manager_id = X`. It only returned direct reports of the given manager.
- Changed to a **BFS traversal** through the full reporting chain:
  - Round 1: fetch direct reports of `managerUserId`
  - Round 2: fetch direct reports of those users
  - Continues until no new users are found
- A `visitedIds` set prevents infinite loops if a data cycle exists.
- Function signature, return shape, and all include flags are unchanged. All existing callers work without modification.
- **Developer pattern for data scoping** — when a developer needs to scope a query to a user's reporting chain (e.g. for leads, applicants, or any assigned-record filter):

  ```javascript
  import rbacService from '../v2/services/rbacService.js';

  // 1. Check if the user has the broad "view all" capability.
  //    The action key is created via super-admin /modules — no migration needed.
  //    Use rbacService.hasActionAccess() — NOT req.rbac.allowedActionKeys.includes()
  //    directly — because hasActionAccess runs the legacy-key compatibility bridge.
  //    req.rbac?.bypassed is true for super_admin users — treat as full access.
  const canViewAll =
    req.rbac?.bypassed ||
    rbacService.hasActionAccess(req.rbac, 'manage-leads.manage-leads.view-all');

  if (canViewAll) {
    // No data filter — user sees all org records
  } else {
    // 2. Get the full reporting chain (manager + all direct and indirect reports)
    const chainUsers = await userService.listUsersManagedByUser({
      managerUserId: req.user.id,
      includeExistingUser: true,   // includes the manager themselves
    });
    const scopedUserIds = chainUsers.map(u => u.id);

    // 3. Pass scopedUserIds into the service WHERE clause:
    //    WHERE assigned_to IN (:scopedUserIds)
  }
  ```

  - `req.rbac` is the RBAC context attached by `rbacMiddleware.requireAction()` — the property is `req.rbac`, NOT `req.rbacContext`.
  - `rbacService.hasActionAccess(req.rbac, key)` is the correct check — it handles legacy key aliases via the compatibility bridge. Plain `req.rbac.allowedActionKeys.includes(key)` bypasses this.
  - `req.rbac` is only present on routes that have a `requireAction()` guard. Routes without it will have `req.rbac` as undefined.
  - **`req.rbac?.bypassed`** — for super_admin users the middleware sets `req.rbac = { bypassed: true, reason: 'super_admin' }`. This object has no `allowedActionKeys`, so `rbacService.hasActionAccess()` returns `false` for it. Always check `req.rbac?.bypassed` first for any capability gate — a bypassed context means unconditional access.
  - `includeExistingUser: true` ensures the manager's own assigned records are included alongside their reports'.
  - If `chainUsers` is empty and `includeExistingUser: false`, the user has no team and sees nothing — this is the expected conservative default.

### 2026-04-27 — Routable modules now get a default sidebar action

- Investigation on org `abcd` showed the primary Admin user and Admin role were valid, but newly created routable modules (`manage-lead-v2`, `manage-applications`, `manage-archives`) had zero active actions and were not sidebar-eligible.
- Important RBAC rule confirmed: the admin sidebar is **action-driven**, not module-only. A module appears only when `getAllControls()` returns at least one granted action under that module whose `actionType` is `page`, `custom`, or `table`, and the module has a route in `config.route`.
- `platformModuleService` now ensures that any active module with `config.route` has a default `page` action when the module is created/updated or when org module access is saved.
- `updateOrgModules()` now includes newly ensured default sidebar actions in the org/Admin role action grants during the same save, so a selected routable module can become visible without a second manual action-creation step.
- Super-admin guidance: if a module should appear in the admin portal sidebar, it must have:
  - a valid module `config.route`
  - at least one granted action with `actionType` `page`, `custom`, or `table`
  - org allocation in `org_modules` / `org_actions`
  - active Admin-role grant in `role_actions`

### 2026-04-27 — Org Module Access supports drag/drop ordering

- Super-admin org Module Access page (`/organizations/:slug/modules`) now supports drag/drop ordering for:
  - top-level modules in the left module list
  - submodules inside the selected module detail pane
- Reordering uses the existing platform module reorder endpoint (`PATCH /api/v2/org/modules/reorder`) and updates `modules.sort_order`.
- The admin sidebar already sorts from `modules.sort_order` through `permissionsService.getAllControls()`, so the changed order is reflected in the admin portal after RBAC controls refetch/re-login.
- Current limitation: order is platform-catalog level, not org-specific, because there is no separate `org_modules.sort_order` / org-specific ordering table yet.

### 2026-04-27 — Admin role permission editor ordering + soft-delete recreate fix

- Admin portal role permission editor (`/admin/manage-role-and-permissions/:roleId`) now supports drag/drop ordering for:
  - top-level modules in the left permission module list
  - submodules in the selected module permission panel
- Added v2 RBAC-protected endpoint:
  - `PATCH /api/v2/org/:orgId/rbac/roles/modules/reorder`
  - guarded by `manage-roles.manage-roles.edit`
  - delegates to the same platform reorder service and updates `modules.sort_order`
- Because `permissionsService.getAllControls()` and `roleService.getRoleModulesWithActions()` both sort by `modules.sort_order`, the order now stays consistent between:
  - super-admin module catalog
  - super-admin org Module Access
  - admin role permission editor
  - admin portal sidebar
- Fixed soft-deleted platform modules blocking recreation:
  - `deleteModule()` soft-deactivates modules (`is_active=false`)
  - `createModule()` now treats inactive matching `key` / `internalId` rows as restorable instead of returning "already exists"
  - restoring updates the old row with the new payload, reactivates it, writes audit log action `module.restore` / `module.restore.submodule`, and ensures default sidebar action if the module has `config.route`

### 2026-05-11 — User drawer allocation scoping + single-call helper endpoint

- The Create/Update User drawer in the admin portal previously fired three cascading helper requests:
  - `GET /v2/org/:orgId/rbac/schools` (all org schools)
  - `GET /v2/org/:orgId/rbac/programs?schoolIds=…` (after schools picked)
  - `GET /v2/org/:orgId/rbac/application-forms?programIds=…` (after programs picked)
- Those endpoints returned the **entire org's** schools/programs/forms regardless of who the actor was, which let a non-admin assign data scopes they did not themselves cover.
- New endpoint replaces the cascade with a single round trip:
  - `GET /v2/org/:orgId/rbac/user-allocations` → `{ schools, programs (with schoolId), applicationForms (with programId) }`
  - guarded by `manage-users.manage-users.view`
  - returns the **full org catalog** when the actor is an org Admin (`req.rbac.bypassed === true` or `req.rbac.effectiveLevel === 6`)
  - otherwise scoped to the actor's reporting chain via `userService.listUsersManagedByUser({ managerUserId: actor.id, includeExistingUser: true, schools: true, programs: true, applicationForms: true })`, then deduped by id
- Frontend now drives the school → program → form cascade entirely client-side using the included `schoolId` / `programId` fields. Selected programs filter to those whose `schoolId` is in the picked schools; selected forms filter to those whose `programId` is in the picked programs.
- The legacy `/schools`, `/programs`, `/application-forms` endpoints are still mounted for backward compatibility but should not be used for new actor-scoped flows.

### 2026-05-11 — Org-Admin identification widened across roleService

- The "is this the org Admin" check used by `roleService` and `roleController` flows was previously `effectiveLevel === 6` only. That check fails for legacy orgs whose Admin role was created at level 1 (Masters Union and other pre-v2 tenants), leaving their admins unable to do things only the org Admin should be allowed to do.
- The same widened rule that the 2026-04-22 org-module sync adopted is now applied across `roleService`:
  ```javascript
  const isOrgAdmin =
    Number(actorContext?.effectiveLevel) === 6
    || (Array.isArray(actorContext?.roles)
        && actorContext.roles.some((r) => r?.internalId === 'admin'));
  ```
- Either signal qualifies. `internalId='admin'` covers legacy orgs; `effectiveLevel=6` covers v2-bootstrapped orgs.
- This rule is now used by `getRoleModulesWithActions` (assignable flag), `syncRoleActions` (downward-only check), and `createRole` (level guard).

### 2026-05-11 — Permission editor org-Admin grant override

- The permission editor's "downward-only" rule said *the actor can only grant actions they themselves possess* — enforced both on the read side (`getRoleModulesWithActions.assignable`) and the save side (`syncRoleActions`).
- That rule created a chicken-and-egg for org Admins: when super-admin adds a brand-new module to an org via Module Access, the action is added to `org_actions` for the org but is only added to the org Admin's `role_actions` if super-admin re-saves the page. Until then, even the org Admin had `assignable: false` for the new action and could not grant it to anyone (including themselves).
- The strict rule has been relaxed for org Admin only: the `org_modules` / `org_actions` ceiling already filters the catalog returned to the editor, so any action that survives that filter is by definition allocated to the org. The org Admin (per the widened identification above) can grant any action within that ceiling regardless of whether their own `role_actions` row exists.
- Non-admin actors keep the original strict downward-only rule, routed through `rbacService.hasActionAccess` so legacy key aliases (e.g. `roles.view` ↔ `manage-roles.view`) match correctly via the compatibility bridge.

### 2026-05-11 — Role-create level rule changed to "at or below" own level

- **Rule change** (overrides the Q15 Round 4 decision documented earlier): a level-N actor can now create roles at levels `1..N` (own level included), not strictly below.
- Previous rule: strictly below own level → a level-3 actor could create level 1 and 2.
- New rule: at or below own level → a level-3 actor can create levels 1, 2, 3.
- Org Admin override applies on top: an org Admin (`internalId='admin'` or `effectiveLevel=6`) can create roles at any level 1–6.
- Enforced in `roleService.createRole`. The "Create Role" drawer in `ManagePermissionsV2` filters its level dropdown to match this rule (no disallowed options appear in the picker at all).
- Error message on rejection: `"Cannot create a role at level N — you may only create roles up to level X"` or `"Your role does not allow creating roles"` if the actor's effective level is 0 / unresolvable.

### 2026-05-11 — `/controls` response now includes role `level`

- `permissionsService.getAllControls()` previously selected only `['id', 'name', 'internalId']` for roles and mapped out `{ roleId, name, internalId, is_active, assignedAt }`. The role `level` was not exposed to the admin frontend.
- Added `level` to both the SQL select and the mapped output so:
  - the "Create Role" drawer can compute the actor's effective level locally and filter the level dropdown without an extra round trip
  - any frontend that needs to hide buttons / table rows for higher-level targets can do so from `state.rbacState.userPermissions.roles` directly
- New shape per role entry: `{ roleId, name, internalId, level, is_active, assignedAt }`
- Frontend contract additive only — existing consumers are unaffected.

### 2026-05-11 — userController error responses surface real DB messages

- `userController.handleServiceError` previously returned the generic fallback (e.g. `"Failed to update user"`) for any non-`ApiError` exception, masking the underlying Sequelize / PG error.
- Now reads `error.original?.message || error.parent?.message || error.message` before falling back to the generic message, so the real cause (e.g. `"duplicate key value violates unique constraint …"`) reaches the client.
- The full stack remains in the dev-mode response and in `notifyError` telemetry — this change only affects the `message` field of the JSON response.

### 2026-05-19 — Manage Roles & Permissions page overhaul

**Header polish:**
- `fe-anandi/src/Layout/Header.jsx` — settings icon was rendering at `size="25px"` and looked visibly shrunken next to the 32×32 user avatar. Bumped to `size="32px"` for visual parity. No behavioral change.

**Roles list — backend contract additions (`be-anandi/src/v2/services/roleService.js`, `controllers/roleController.js`, `routes/rbacRoles.routes.js`):**
- `listRoles` now also returns:
  - `createdBy: { id, name, email } | null` — joined via `Role.belongsTo(User, as: 'creator')`.
  - `moduleCount: integer` — distinct active modules behind the role's currently-active `role_actions`. Computed via a small `GROUP BY ra.role_id COUNT(DISTINCT a.module_id)` raw query, keyed on the page's `roleIds`. The old `actionCount` field is preserved (still returned) but the UI no longer renders it.
- `listRoles` query params extended (all optional, all backward-compatible):
  - `levels` — comma-separated or array of integers (e.g. `?levels=1,2,3`). Narrows visibility within the actor's level ceiling; cannot widen it.
  - `createdFrom` / `createdTo` — ISO timestamps, inclusive range on `roles.created_at`.
  - `createdByIds` — comma-separated or array of user ids.
- Frontend forwards arrays as comma-joined single params; backend `toIdArray` / `toIntArray` accept both `levels=1,2,3` and repeated `levels[]=...` forms.
- New endpoint: `GET /api/v2/org/:orgId/rbac/roles/creators` → distinct users who have created at least one active role in this org, scoped by the actor's level ceiling. Powers the "Created By" multi-select in the filter drawer.
  - Route declared **before** `/roles/:roleId` so Express doesn't treat `creators` as a `roleId` param.
  - Same `manage-roles.manage-roles.view` guard as the list endpoint.

**Frontend RTK Query (`fe-anandi/src/Redux/Services/rbacService.js`):**
- `v2ListRoles` query args extended to `{ orgId, page, limit, search, levels, createdFrom, createdTo, createdByIds }`. Arrays comma-joined into `URLSearchParams`. Empty arrays are omitted from the URL.
- New `v2ListRoleCreators` query — provides `v2Roles` tag, so creator list refreshes automatically after any role mutation.

**Manage Roles & Permissions page (`fe-anandi/src/pages/UserAccessControl/ManageRoles/ManagePermissionsV2.jsx`) — full UI overhaul:**
- Sets `document.title = 'Roles & Permissions • Lead Matrix'` on mount, restores previous title on unmount.
- Sub-heading description rendered below the "Manage Roles & Permissions" heading.
- Level column header has a small `InfoCircle` icon (from `@untitled-ui/icons-react`) with a tooltip explaining what role levels mean.
- Description column max-width 240px, single-line ellipsis, full text in `ToolTip` on hover (tooltip suppressed when description is ≤60 chars to avoid empty tooltips).
- Column changes:
  - Removed: old `Actions` column (formerly showed `actionCount`).
  - Added: `Modules` column showing the new `moduleCount`.
  - Added: `Created At` — formatted as `25 July 2025, Saturday` via `moment.tz(iso, moment.tz.guess()).format('D MMMM YYYY, dddd')`.
  - Added: `Created By` — shows `createdBy.name`, falls back to email, falls back to `-`. Hover shows email tooltip.
- Search box removed (per Q3 = option b). Name search now lives inside the Filter drawer.
- Filter icon next to "Create Role" opens a new drawer `fe-anandi/src/SideFilter/FilterRoles/FilterRoles.jsx` with Name (contains), Levels (multi-select 1–6), Created Between (`DateRangePicker` from `rsuite`), Created By (multi-select fed by `v2ListRoleCreators`).
- Badge on filter icon shows count of currently-applied filters.
- Pagination model switched to **Load More**. Page-size selector (10 / 25 / 50 / 100, default 10). Any change to filters or page size **resets the accumulated list and refetches page 1**. Load More appends the next page; dedup-by-id prevents double-append on refetch.
- Loaders:
  - Initial fetch shows N skeleton rows where N = current `pageSize`.
  - Load-more in flight shows an inline `<ButtonLoader />` row at the bottom of the table plus a spinner inside the Load More button.
- Empty state copy now differentiates "no roles yet" vs "no roles match your filters".

**Files modified:**
- `be-anandi/src/v2/services/roleService.js`
- `be-anandi/src/v2/controllers/roleController.js`
- `be-anandi/src/v2/routes/rbacRoles.routes.js`
- `fe-anandi/src/Redux/Services/rbacService.js`
- `fe-anandi/src/Layout/Header.jsx`
- `fe-anandi/src/pages/UserAccessControl/ManageRoles/ManagePermissionsV2.jsx`

**Files added:**
- `fe-anandi/src/SideFilter/FilterRoles/FilterRoles.jsx`

**Verification done:**
- `node --check` clean on the 3 modified backend files.
- `esbuild` syntax-check clean on the 3 modified/added JSX files (`Header.jsx`, `FilterRoles.jsx`, `ManagePermissionsV2.jsx`).
- Full `vite build` runs through all 14,911 modules and only fails at a pre-existing missing-import in `pages/Admin/StudentProfile/Components/StudentCommLog.jsx` (referencing a non-existent `CommunicationLogsService` module) — unrelated to this work.

**Manual smoke-test still required:**
- Open `/admin/manage-role-and-permissions` and confirm: document title, sub-heading, level tooltip, description ellipsis, Created At formatted in user's tz, Created By name + tooltip, Modules column count.
- Click filter icon → drawer opens, all four filters work, badge count is correct, "Clear filters" resets.
- Change page size to 25, then to 100, confirm list resets and refetches.
- Click "Load More" — verify rows append (no duplicates), pagination counter updates.
- Open the Header — settings icon should now look the same visual height as the user avatar next to it.

### 2026-05-19 — `users.role` ENUM locked down to canonical actor types

**Policy decision:** going forward, every runtime user-creation path emits one of only three values for `users.role`:

- `admin` — any platform/org user managing the system from the admin portal
- `super_admin` — platform super admin (separate portal)
- `student` — anyone on the student/applicant side

The legacy `'counsellor'` and `'user'` values are no longer produced by any creation path. The real role identity continues to live in `user_roles` (the junction table); the ENUM column is just a vestigial actor-type tag retained for backward compatibility with code that still reads it.

**Why:**
- The original ENUM was being used inconsistently — v2 was writing `'user'`, v1 was deriving `'counsellor'` from role internalId, bulk import was passing through raw CSV — yielding 158 `user` rows and 20 `counsellor` rows on the dev DB while the v2 RBAC junction already captured the true role assignment.
- Tightening creation to three canonical actor types keeps the ENUM honest as a coarse classifier (admin-portal user vs super-admin vs student) without conflating it with the granular role system in `user_roles`.

**Runtime creation paths changed:**

| File | Before | After |
|---|---|---|
| `be-anandi/src/v2/services/userService.js` (v2 admin-portal create) | hardcoded `role: 'user'` | hardcoded `role: 'admin'` |
| `be-anandi/src/controllers/rbac/usersController.js` (legacy v1 RBAC create) | derived ENUM from role `internalId`; allowed `'counsellor'`, fell back to `'user'` | only `admin/super_admin/student` pass through; everything else (including `counsellor`) → `'admin'` |
| `be-anandi/src/controllers/users/userController.js` (legacy public `/users` create) | `role: role || 'student'` (raw passthrough) | normalizes — admin/super_admin/student pass through; anything else (incl. counsellor/user) → `'admin'`; still defaults to `'student'` when no role provided |
| `be-anandi/src/controllers/org/bulkUserController.js` (CSV bulk import) | accepted `counsellor` from CSV and stored it as-is | still accepts `'counsellor'` and `'user'` at the CSV gate (so old templates import cleanly), but the user row is stored with `role = 'admin'`. The `Counsellor` table side-effect at line ~178 still triggers on the *original* CSV value so legacy `getCounsellors` endpoints continue to surface those rows |

**Paths intentionally not changed (already correct):**

- `be-anandi/src/services/orgService.js:88` — `role: 'admin'`
- `be-anandi/src/services/inviteService.js:111` — `role: 'admin'`
- `be-anandi/src/controllers/public/campaignLeadController.js:223` — `role: 'student'` (lead/applicant signup)
- `be-anandi/src/utils/getSystemUser.js:28` — `role: 'admin'` (system actor)
- Seeders (`seeders/*.cjs`) — these are one-off DB seed scripts and not runtime paths. They still emit `'counsellor'` rows for historical/dev fixture purposes; leave them as-is.

**Read-side compatibility notes for the next agent:**

- The legacy ENUM is still queried by:
  - `controllers/users/userController.js:93` (`getAllCounsellors`)
  - `v2/controllers/counsellorController.js:44, 59` (`getCounsellors`)
  - `services/studentQuery.service.js:170`
- After this policy change, those queries will continue to find historical rows (the 20 existing `counsellor` rows + anything seeders inserted) but will return zero new counsellor users going forward.
- The v2 `getCounsellors` endpoint also looks at `OrgUser.orgRole IN ('counsellor', 'manager', 'admin')` (line 30-38), so org-level counsellor membership remains queryable independent of the user-level ENUM.
- **No data migration was run.** Existing 158 `user` rows and 20 `counsellor` rows stay as they are. If a future cleanup wants to backfill them to `'admin'`, that's a separate migration and should be paired with a sweep of every read-side `WHERE role = 'counsellor'` / `WHERE role = 'user'` query.

**Verification:**
- `node --check` clean on all four modified files.
- ENUM constraint not changed — the DB still permits `'counsellor'` and `'user'`, so no migration was needed. Historical rows continue to validate against the schema.

### 2026-05-19 — `org_users` scoped to admin-portal membership only

**Policy decision:** `org_users` is now treated as the admin-portal **membership** table — it tracks which org-level users (admin / counsellor / manager / etc.) belong to which organizations. **Students no longer get an `org_users` row.** Their org link lives solely on `users.organization_id`.

**Why:**

- A `SELECT org_role, count(*) FROM org_users GROUP BY org_role` on the dev DB returned 3580 `student` rows alongside 28 `user`, 16 `counsellor`, and 10 `admin`. The student rows were created by the lead-intake flow (every public lead/applicant create wrote an `org_users` row), bloating the table for no downstream benefit.
- Every consumer of `org_users` on the admin-portal side already filtered to non-student roles (`getCounsellors` filters `orgRole IN ('counsellor', 'manager', 'admin')`; the new v2 `listUsers` filters `users.role = 'admin'`). The student rows were therefore dead weight.
- `org_users` is going to keep growing with admin-portal users over the years; mixing student records into it ruins index efficiency and turns every membership query into a wider scan.

**Code paths changed:**

| File | Change |
|---|---|
| `be-anandi/src/v2/services/dynamicLeadService.js` | Removed the `OrgUser.create({ orgRole: 'student' })` block from the lead-intake `createLead`. Student users are still created with `users.organization_id` and a `user_roles` row for the org's `student` Role. Also removed the now-unused `OrgUser` import. |
| `be-anandi/src/services/studentQuery.service.js` (`createQuery`) | Membership check now accepts membership via **either** `users.organization_id` (students) **or** an active `org_users` row (admin-portal members). Previously it required an `org_users` row, which would have broken every student raising a query after the policy change. Also dropped the bogus `schoolId` clause on the OrgUser query — `org_users` has no `school_id` column, so that part was a silent no-op. |

**Read paths audited as part of this change (sweep of 25 files referencing OrgUser):**

| Layer | File | Verdict |
|---|---|---|
| Auth middleware | `middleware/jwtValidator.js` | Safe. Membership check at line ~141 explicitly skips `user.role === 'student'`. Fallback "primary org" lookups at lines ~116, ~174 only fire when JWT lacks `organizationId`; student JWTs always carry it. |
| Auth controller | `controllers/users/authController.js` | Safe. `studentLogin` doesn't touch `org_users`. The `OrgUser.findAll` blocks at lines 216 (admin login options) and 629 (organizations list in login response) are gated by `user.role !== 'student'`. |
| v2 user listing | `v2/services/userService.js` | Safe. Admin-portal flow; filters to `users.role='admin'`. |
| v2 user profile | `v2/services/userProfileService.js` | Safe. Handles null `orgUser` gracefully; admin-only paths blocked at line 45. |
| v2 counsellor listing | `v2/controllers/counsellorController.js` | Safe. Filters `orgRole IN ('counsellor','manager','admin')`. Students excluded. |
| v2 org controller | `v2/controllers/orgController.js` | Safe. Super-admin only. |
| Org bootstrap | `v2/services/orgBootstrapService.js` | Safe. Writes `'admin'` rows only. |
| Bulk user import | `controllers/org/bulkUserController.js` | Safe. CSV import is admin-only. |
| Legacy invite | `services/inviteService.js` | Safe. Writes whatever `orgRole` the invite record specifies; invites are admin-portal. |
| Legacy org service | `services/orgService.js` | Safe. Writes `'admin'` only. |
| Legacy feature middleware | `middleware/featureAccessValidator.js` | Safe. Used only by `role.routes.js` (legacy admin-only routes). |
| Legacy feature service | `services/featureService.js` | Safe. Legacy admin-only authorization. |
| Super-admin controllers | `controllers/super_admin/*` | Safe. Counts will drop by ~3580 after cleanup (dashboard metric), but no functional breakage. |
| Membership controller | `controllers/org/membershipController.js` | Safe. Org-membership management is admin-only. |
| `services/studentQuery.service.js` | `createQuery` | **PATCHED** (only consumer that previously required student `org_users` rows). |

**One-off cleanup query** (run once, not a migration — review counts first):

```sql
-- Verify before deleting
SELECT COUNT(*) FROM org_users WHERE org_role = 'student';
-- Expected at the time of policy change: 3580 (will drift if intake ran since)

-- Delete student membership rows
DELETE FROM org_users WHERE org_role = 'student';

-- Confirm after
SELECT org_role, COUNT(*) FROM org_users GROUP BY org_role ORDER BY org_role;
-- Expected: admin / user / counsellor remain; no student rows.
```

**Verification done:**
- `node --check` clean on both modified backend files.
- Full grep sweep of all 25 files that reference `OrgUser` to confirm no student-side consumer relies on the rows beyond `studentQuery.service.js` (now patched).

**Risks the next agent should know about:**
- Any **future** code that filters `OrgUser` by user without a `role` precondition will silently start returning fewer rows after cleanup (because student rows are gone). Search for the pattern before adding new `OrgUser` queries.
- The super-admin dashboard's "active members" count metric drops by ~3580 after cleanup. Cosmetic; not a bug.
- The `org_role = 'student'` value is still a valid string in the column (column is plain VARCHAR, not ENUM), but no runtime path emits it any more. A future migration could narrow this further if desired.

### 2026-05-19 — `user_roles` scoped to admin-portal RBAC only (parallel to `org_users`)

**Policy decision:** `user_roles` is the **v2 RBAC permission junction** for admin-portal users. It exists so that `getUserRbacContext` can resolve a user's allowed action keys through `user_roles → role_actions → actions`. **Students no longer get a `user_roles` row** — they don't traverse v2 RBAC at all, so their entry was dead weight.

**Why this mirrors the `org_users` decision:**

- `rbacMiddleware` only guards admin-portal routes. `rbacService.getUserRbacContext` is never called for students.
- A student's identity is fully captured by `users.role = 'student'`. Their org link is `users.organization_id`. There is no v2 Role / action permission that a student needs.
- The lead-intake flow was looking up an org's `Role` with `internalId='student'` (when one existed) and inserting a `user_roles` row pointing at it. Nothing reads these rows.
- Same scale problem as `org_users` — historically ~3580 student `user_roles` rows on the dev DB, bloating the junction table that should only carry admin-portal RBAC assignments.

**Read paths audited** (every `UserRoles.findOne / findAll / count` in the codebase):

| Layer | Reader | Fires for students? |
|---|---|---|
| Admin login flow | `authController.getAdminLoginAccessOptions` (line 233) | No — admin login only |
| Admin login flow | `authController` line 658 (login response `schoolDetails`) | No — wrapped in `if (user.role !== 'student')` (line 626) |
| v2 admin user CRUD | `v2/services/userService.js:893` | No — admin endpoint |
| v2 role CRUD | `v2/services/roleService.js:172, 282, 480` | No — admin endpoint |
| v2 RBAC context | `v2/services/rbacService.js:204` (`getUserRbacContext`) | No — `rbacMiddleware` only runs on admin routes |
| v2 controls bootstrap | `v2/services/permissionsService.js:56` (`getAllControls`) | No — admin endpoint |
| v2 profile | `v2/services/userProfileService.js:65` | Could fire if a student hits `/profile/me`, but an empty `user_roles` result returns `roles: []` gracefully — correct |
| Super-admin | `v2/controllers/orgController.js:169, 274, 325, 602` | No — super-admin only |
| Legacy v1 RBAC | `controllers/rbac/usersController.js`, `majorRbacController.js`, `rolesController.js` | No — admin-portal only |
| `services/` (student-portal logic) | grep returns **zero** `UserRoles.findOne / findAll` results | No reader exists |
| `controllers/public/` (student-portal endpoints) | grep returns **zero** `UserRoles` references | No reader exists |

Conclusion: removing the student `user_roles` row creates zero behavior change. The student profile endpoint will return an empty `roles` array — which is semantically correct.

**Code change applied:**

| File | Change |
|---|---|
| `be-anandi/src/v2/services/dynamicLeadService.js` | Removed the `Role.findOne({ internalId: 'student' })` + `UserRoles.create(...)` block. Replaced with a single policy comment explaining why. Also dropped the now-unused `Role` import. The lead intake still creates the `User` row (`role='student'`, `organization_id=orgId`) — that's all a student needs. |

**One-off cleanup query for historical student `user_roles` rows** (run on the same DB you ran the `org_users` cleanup against):

```sql
-- Step 1: snapshot current counts. Join to roles to see which Role
-- rows the student user_roles entries point at.
SELECT r.internal_id, COUNT(*) AS user_role_rows
FROM user_roles ur
JOIN roles r ON r.id = ur.role_id
WHERE r.internal_id = 'student'
GROUP BY r.internal_id;

-- Step 2: delete the student user_roles rows in a transaction.
BEGIN;

DELETE FROM user_roles
WHERE role_id IN (SELECT id FROM roles WHERE internal_id = 'student');

-- Confirm before commit.
SELECT r.internal_id, COUNT(*) AS user_role_rows
FROM user_roles ur
JOIN roles r ON r.id = ur.role_id
WHERE r.internal_id = 'student'
GROUP BY r.internal_id;
-- Expected: zero rows (or empty result set).

COMMIT;
-- (ROLLBACK if anything looks wrong.)

-- Step 3 (optional): the `roles` rows with internal_id='student' are
-- now unused by user_roles, but DON'T delete them yet — they're still
-- referenced by historical role_actions, and the lead-intake code may
-- have been creating them through other admin flows. Drop these only
-- after a separate audit confirms no admin role inherits permissions
-- from the student Role.
```

**Risks the next agent should know about:**

- Same caveat as `org_users` — any future code that queries `user_roles` by user without a `users.role !== 'student'` precondition will silently see different counts. The `v2/services/userProfileService.js:65` read returns `roles: []` for students after cleanup, which is correct but worth knowing if you add new profile fields that depend on it.
- The `roles` table still contains org-scoped `internal_id='student'` rows from before this change. They're harmless but technically dead data — clean them up in a separate pass if you want a tidy `roles` table.

---

## 14. Stage & Sub-Stage Permissions (Per-Role Instance Scope) — 2026-06-17

### Why this is a new mechanism
The action-key RBAC (`requireAction` → `rbacContext.allowedActionKeys`) is **binary per action**; it cannot express "this specific stage row". Lead/application stages are **dynamic, per-org data** (`leadStage`, `leadSubStage`, `applicationStage`, `applicationSubStage`), so they cannot live in the platform action catalog. This feature adds an **instance-level scope layer** that sits *on top of* the existing engine — the action-key system still answers "can this role touch stages at all"; the new layer answers "WHICH stages/sub-stages".

### Model & rules
New tables (migrations `20260617120000`, `20260617120100`; models `RoleLeadStagePermission`, `RoleApplicationStagePermission`):
```
role_lead_stage_permissions        (role_id, lead_stage_id, lead_sub_stage_id NULLABLE, is_active, created_by, updated_by)
role_application_stage_permissions (role_id, application_stage_id, application_sub_stage_id NULLABLE, ...)
```
- Row with `sub_stage_id = NULL` → grants the **stage**. Row with both → grants that **sub-stage**.
- **A role with NO rows for a stage-type is UNRESTRICTED (all stages).** Backward-compatible: existing roles keep working untouched; restriction is opt-in.
- **super_admin, level-6 roles, and `internalId='admin'` are always unrestricted** (top of the org hierarchy — they configure everyone else). Matches the org-admin detection used in `roleService`.
- A stage is "allowed" if it has a stage-level row **or** any of its sub-stages is granted (you can't sit in a sub-stage without its parent). A sub-stage is allowed only via its own row.
- Move to (stage, no sub) ⇒ stage allowed. Move to (stage, sub) ⇒ stage allowed **and** sub allowed.

### Core service — `src/v2/services/stagePermissionService.js`
`resolveStageScope({ userId, orgId, selectedRoleId, userRole })` → `{ bypassed, roleIds, lead:{unrestricted,stageIds,subStageIds,stageUuids,subStageUuids}, application:{…} }`. Plus `assertLeadStageAllowed` / `assertApplicationStageAllowed` (throw 403), `annotateLeadStages` / `annotateApplicationStages` (add `permitted` per stage/sub), `assertCanEditExistingStage` + `autoGrantCreatedStage` (settings flow), and `getRoleStagePermissions` / `syncRoleStagePermissions` (role editor). Uses the selected role; falls back to the union of active roles when none is selected.

### Enforcement points (existing, in-use APIs — no new stage APIs were created)
| Flow | File | Behaviour |
|---|---|---|
| Bulk change lead stage | `controllers/org/leadController.js` `changeLeadStage` | `assertLeadStageAllowed` after stage/sub lookup |
| Bulk change application stage | `services/applicationStageManagerService.js` `changeApplicationStage` | `assertApplicationStageAllowed` after lookup |
| Profile/list single-lead change (`PATCH /v2/:orgId/leads/:id`) | `v2/services/dynamicLeadService.js` `updateLead` | scope check before the stage write (controller now passes `_selectedRoleId`/`_userRole`) |
| Settings save (lead) | `services/leadStageManagerService.js` `postLeadStagesAndSubStages` | restricted roles edit only in-scope stages; may create new ones (auto-granted) |
| Settings save (application) | `services/applicationStageManagerService.js` `postApplicationStagesAndSubStages` | same |
| Stage reads (`getLeadStagesWithSubStages` / `getApplicationStagesWithSubStages`) | their controllers | now behind `jwtValidator`; each stage/sub annotated with `permitted` (list NOT filtered). The automation caller hits the service fn directly and is unaffected. |

### Config surface (v2, reuses existing `manage-roles` gates — no new action keys)
- `GET  /api/v2/org/:orgId/rbac/roles/:roleId/stage-permissions` → `roleController.getRoleStagePermissions`
- `PUT  /api/v2/org/:orgId/rbac/roles/:roleId/stage-permissions` → `roleController.syncRoleStagePermissions`
- Downward-only: non-admin actors can only grant stages they can use; org-admin/level-6/super_admin can grant anything in the org. Same level-hierarchy guard as `syncRoleActions`.

### Frontend (fe-anandi)
- `Redux/Services/rbacService.js`: `v2GetRoleStagePermissions` / `v2SyncRoleStagePermissions` (+ hooks).
- `pages/UserAccessControl/ModulePermissions/StageAccessPermissionV2.jsx`: new "Stage Access" tab in the role permission editor (`ModuleWisePermissionV2.jsx`).
- `SideFilter/ChangeStage/ChangeLeadStage.jsx`: stage & sub-stage dropdowns filtered by `permitted !== false`.
- `pages/Admin/Settings/ManageCRM/LeadStageItem.jsx` & `ApplicationStageItem.jsx`: non-permitted stages/sub-stages rendered read-only (`permitted === false` ⇒ locked).

### Notes / decisions
- **No new `requireAction` route gates** were added on the change/manage routes — a hard capability gate would 403 existing non-admin users on a tested, live flow. The per-stage scope check is the requirement.
- `permitted` annotation is additive (`!== false`), so a missing flag never hides/locks anything — safe if the annotation is ever skipped.
- A restricted role that **creates** a new stage is auto-granted that stage (and any new sub-stages) for its selected role, so it can immediately use what it created. No-op for unrestricted/admin roles.
- **Run `cd be-anandi && npm run migrate`** to create the two tables before this is exercised.

### 2026-07-16 — User drawer allocation picker restricted to actor's OWN scopes

- **Bug:** a non-admin (e.g. a level-2 Counsellor) opening the Create/Edit User drawer saw the **entire org's** schools/programs/forms, not just their own. Root cause: `userService.listActorAllocations` built the non-admin pool from `listUsersManagedByUser({ includeExistingUser: true })`, i.e. the union of the actor's own allocations **plus their whole reporting downline's**. In a wide reporting tree that union equals the full org catalog.
- **Fix (revises the 2026-05-11 "reporting chain" rule):** the non-top-level branch of `listActorAllocations` now returns **only the actor's own** `user_schools` / `user_programs` / `user_application_forms` for the org — no downline. Output shape unchanged (still includes `schoolId`/`programId` for the client-side cascade).
- **Rule (confirmed with Prateek):** top-level actors only — `req.rbac.effectiveLevel === 6` or super_admin `bypassed` — get the full org catalog. Every other role gets **own-assigned only**. If the actor has no own allocations, the pickers are **empty** (strict; no downline fallback).
- Also touches `widget.ctrl.js listApplicationForms`, the other caller of `listActorAllocations({ unrestricted:false })` — same own-only rule now applies to the widget form picker (comment updated).
- Files: `be-anandi/src/v2/services/userService.js` (`listActorAllocations`), `be-anandi/src/v2/controllers/widget.ctrl.js` (comment only).

### 2026-07-16 — Reporting-manager picker scoped to actor's level

- **Bug:** the Reporting Managers dropdown (Create/Edit User drawer **and** the bulk-create page — both call `GET /v2/org/:orgId/rbac/potential-managers`) listed **every** active user in the org regardless of who the actor was. `listPotentialManagers` filtered `maxLevel >= minLevel` with `minLevel` hardcoded to `1` on the frontend, i.e. no effective filter.
- **Fix:** `userService.listPotentialManagers` now takes `actor`, resolves `actorContext.effectiveLevel`, and returns only users whose highest role level is **≤ the actor's level** (`actorLevel === null || maxLevel <= actorLevel`) — the same visibility rule the manage-users list already applies. Top-level actors (effectiveLevel `null`/`6`) still see everyone. The `minLevel` query param is now ignored (frontend still sends it harmlessly).
- Files: `be-anandi/src/v2/services/userService.js` (`listPotentialManagers`), `be-anandi/src/v2/controllers/userController.js` (passes `buildActor(req)`).
- **Confirmed already-correct (no change needed):** `roleService.listRoles` already filters roles to `level <= actorLevel` (per-row `userCount` included); `userService.listUsers` already hides users above the actor's level. The bulk-create page reuses the drawer's `user-allocations` / `roles` / `potential-managers` endpoints, so the school/program/form own-only scoping (2026-07-16 above) and this manager fix apply there automatically — no separate bulk-create code.
- **Known follow-up (not changed):** `listUsers` applies the level filter in-memory *after* `findAndCountAll`, so `pagination.total` reflects the pre-filter count and pages can come back short. Display is correct (no higher-level users leak); only the total/paging math is off.

### 2026-07-16 — Manage-users list visibility: level → school → role-match (final: role-match)

Visibility of the User Management list (`listUsers`) went through three iterations on the same day; the **final** rule is role-match. History kept for context:
1. **Level scoping** (pre-existing): only users with max role level ≤ actor's level (applied in-memory after the query).
2. **School-overlap** (tester Bug 2, interim): also required sharing ≥1 assigned school.
3. **Role-match + reports (FINAL, confirmed with Prateek):** a non-top-level actor sees users who **share at least one of the actor's own assigned roles** (`actorContext.roleIds`) **UNION** any users the actor is the **reporting manager of** (direct `user_reporting_managers` rows where `manager_id = actor`, this org, active). The level filter **and** the school filter were both **dropped**. Level-6 (`effectiveLevel === 6`) and super-admins (`effectiveLevel === null`) stay **unscoped** (see all).
- **Implementation:** a `userWhere.id [Op.in]` constraint = (users with `UserRoles.roleId ∈ actor.roleIds`) ∪ (users with `UserReportingManager.manager_id = actor`), intersected with any explicit `?roleIds=` filter. Because it runs **in the query**, `findAndCountAll`'s `count` is exact — the old in-memory level filter (which left `pagination.total` inflated, e.g. "Showing 4 of 457") was removed, so the footer `Showing X of Y` is now correct. The `users.role='admin'` ENUM filter (admin-portal users only, per 2026-05-19) still applies to the union, so non-admin-enum reports are excluded.
- Verified live: bandana (Manager L3, role 44) → 14 users, `total=14`; a Sales Head (L5) with 170 direct reports + 4 role-mates → 174 users, `total=174`; level-6 admin → all 485.
- **Direct reports only** (not the full downline). If the whole reporting sub-tree is wanted later, `listUsersManagedByUser` already does the BFS.
- Files: `be-anandi/src/v2/services/userService.js` (`listUsers`).
- **On the "Executive" allocation bug (Bug 1) re-report:** the tester's `execv2` account holds the level-6 **Admin** role, so by the level-6 rule it correctly sees the whole org catalog in the Create-User pickers. The own-only allocation restriction (`listActorAllocations`) already applies to every non-level-6 user — no change. A proper non-level-6 role is required to observe the restriction.

### 2026-07-16 — RBAC context cache invalidation now clears all role variants

- **Bug:** users (e.g. a Manager) intermittently saw empty "No Roles yet" / empty user lists right after their role or permissions were set up. Root cause: the RBAC context is cached under two key shapes — `…:role:all` (list services, `getUserRbacContext` with no roleId) and `…:role:<selectedRoleId>` (the `requireAction` middleware, via `req.user.selectedRoleId`). `invalidateUserRbacContext({userId, orgId})` only deleted `role:all`, so the **middleware kept authorizing against a stale `role:<id>` context until the 5-min TTL** → 403 → empty UI.
- **Fix:** when no specific `roleId` is passed (every current caller), `invalidateUserRbacContext` now deletes the whole per-user prefix `cache:v2:rbac-context:org:<org>:user:<id>:role:*` across **both** the local in-memory cache and Redis (via `redisCache.delPattern`). No cache **key format** changed — only the breadth of deletion. Passing an explicit `roleId` still clears just that one variant.
- Manual cache flush (unchanged, still available): `POST /api/internal/cache/keys/clear?key=kms4000` with `{ "keys": ["cache:v2:rbac-context:org:<org>:user:<id>:role:"] }`.
- Files: `be-anandi/src/v2/services/rbacService.js` (`invalidateUserRbacContext` + new `deleteCacheByPrefix`).

### 2026-07-16 — Student OTP/password login scoped to the specific program (form)

- **Bug (tester):** on the student `-pgp` auth flow, a phone/email not registered in one program (the tester's "form") but registered in **another** program of the **same school** could still authenticate into the first program. Root cause: `findApplicantLead` / `ensureUserMatchesProgram` (in `studentAuth.service.js`) matched a lead by `userId + orgId + schoolId` only — never `programId` — and sibling programs share a school.
- **Fix:** `findApplicantLead` now accepts an optional `programId`; `ensureUserMatchesProgram` passes `program.id` when the context is a program (`login-pgp`/`send-otp-pgp`/`verify-otp-pgp`/`forgot-password-pgp` with a `programId`). No match → `ERROR_NO_PROGRAM_MATCH` ("No account found for this program. Please register first."). The school-only flow (`schoolId` given, no program) keeps the org+school match, since there is no specific program to bind to.
- Safe: of 104,512 org-12 leads only 5 have a null `program_id`, so requiring a program match doesn't strand legitimate registrations. Verified live: a student registered in program 8 (school 14) is now rejected from program 48 of the same school (send **and** verify), while program 8 still sends/validates OTP normally.
- Files: `be-anandi/src/v2/services/studentAuth.service.js`.
- The gate is shared by all four `-pgp` endpoints, so login/OTP/forgot-password are all corrected together.

### 2026-07-16 — Lead/Application stages made school-scoped (settings + Stage Access only)

- **Goal:** each school gets its own Lead/Application stages instead of one org-wide set. Scoped deliberately to the surfaces we own now — the Manage-CRM **settings** pages and the role-editor **Stage Access** tab. Other stage consumers (Change-Stage dropdown, dashboards, filters, exports, automation, campaigns) are intentionally left for their respective devs; this change must NOT disturb them.
- **Model:** both `leadStage` and `applicationStage` already had a dormant nullable `schoolId` FK→`schools.id`. Added the field to the Sequelize models and put it to use. Sub-stage tables unchanged (they inherit scope from the parent stage).
- **Semantics — `NULL = shared org default`:**
  - **Read (school given):** return that school's own stages if it has any, else the org-wide shared defaults (`schoolId IS NULL`).
  - **Read (no school given):** shared defaults only. This keeps every existing org-scoped caller (automation trigger picker, Change-Stage dropdown, etc.) seeing exactly today's stages, unaffected by any school-specific rows — the key non-disruption guarantee.
  - **Save (school given, first time):** the incoming rows are the shared defaults, so we FORK them into fresh school-owned rows (new uuids/ids), leaving the shared NULL rows — and the 100k+ leads FK'd to them — untouched. Later saves upsert the school's own rows by uuid. No school → legacy behaviour (writes shared NULL rows).
- **Files (backend):** `models/leadStage.js`, `models/applicationStage.js`; `services/leadStageManagerService.js` (`getLeadStagesWithSubStages` +`schoolId` param, `postLeadStagesAndSubStages` +fork), `services/applicationStageManagerService.js` (same two); `controllers/users/leadStageManagerController.js` + `applicationStageManagerController.js` (read `schoolId` from query/body); `v2/services/stagePermissionService.js` `getRoleStagePermissions` (+`schoolId`, same override on both stage lists) + `v2/controllers/roleController.js`.
- **Files (frontend):** `Redux/Services/Settings Apis/ManageCRM/leadStageService.js` (both get queries accept `schoolId`); `pages/Admin/Settings/ManageCRM/LeadStage.jsx` + `ApplicationStage.jsx` (fetch + save pass `selectedSchoolData.id`); `Redux/Services/rbacService.js` (`v2GetRoleStagePermissions` +`schoolId`); `pages/UserAccessControl/ModulePermissions/StageAccessPermissionV2.jsx` (new **School** dropdown fed by `useV2ListSchoolsQuery`; `All schools (shared defaults)` = no filter).
- **Verified live (org 4, school 21, with cleanup):** no-school read = 19 shared defaults; school read (no own) falls back to the 19; first save forks 19 into school-owned rows; school read then returns its own (marked) copy while the no-school read stays the untouched defaults (no leak); Stage-Access endpoint mirrors this with/without the school filter. All test rows deleted afterward.

### Doc discrepancy noted while implementing
Earlier sections of this file (written 2026-04) describe v2 RBAC as *proposed/in-progress* and mark `usersController`/`rolesController` v1 flows as "to be rebuilt". In reality the v2 RBAC system (`src/v2/`, `rbacMiddleware.requireAction`, `org_modules`/`org_actions`/`role_actions`, controls API, audit logs) is **implemented and live** — consistent with the RBAC Developer FAQ `.docx`. Treat §1–§13 as historical analysis; the FAQ `.docx` + this §14 reflect the current implemented system.

---

## 15. Data Visibility — Whose Records You See — 2026-08-06

### Where this sits
This is the **third** scope layer, and it answers a different question from the other two:

| Layer | Question | Mechanism |
|---|---|---|
| Action-key RBAC (§1–§13) | *May I use this feature at all?* | `role_actions` → `rbacMiddleware.requireAction` |
| Stage permissions (§14) | *Which stage rows may I touch?* | `role_lead_stage_permissions` |
| **Data visibility (this §)** | ***Whose* records do I see?** | role `level` + `user_reporting_managers` |

A user can hold `leads.view` and still see zero leads — the action grants the page, this layer fills it.

### The authority is role LEVEL — never `users.role`

`users.role` is a **v1 leftover** that only records which portal a user signs into (`admin` / `super_admin` → admin portals, `student` → student portal, `counsellor` / `user` → dead v1 values). v2 RBAC ignores it, as already stated in FINAL DECISIONS above. There is a permanent note on the column in `be-anandi/src/models/User.js`.

**Why this matters (the bug that forced this section).** Lead scoping used to read:

```js
const isAdmin = ['admin','super_admin'].includes(findUser?.role) && !isCounsellor;
```

Almost every admin-portal user carries `role = 'admin'` regardless of seniority, so a **level-2 Team Lead was indistinguishable from a level-6 org owner** and skipped scoping entirely. In org 12 that was 62 users (all Team Leads, Managers, GMs, Sales Heads) seeing all ~426k leads. Replaced by:

```js
// userService.js
const ORG_OWNER_LEVEL = 6;
const isOrgWideActor = async ({ userId, orgId, roleId = null }) => {
  const context = await rbacService.getUserRbacContext({ userId, orgId, roleId });
  return Number(context?.effectiveLevel) >= ORG_OWNER_LEVEL;
};
```

`effectiveLevel` is `null` for a user with no roles, and `Number(null) >= 6` is false — so an unknown actor **fails closed** (scoped), never open.

> `is_counsellor_role` is **not** a visibility flag. It marks a role as assignable-as-a-counsellor and drives the counsellor pickers (`counsellorController`, `advanceFilterService`, `userDashboard.service`) plus the post-login landing route. Do not flag roles to change what they can see — change their `level`.

### The visible set

Built by `userService.listUsersManagedByUser`. For an actor at level **N**, the union of:

1. **Themselves** — `includeExistingUser: true`
2. **Their reporting downline**, walked recursively through `user_reporting_managers` (BFS, `visitedIds` breaks cycles), with `capReportingByLevel: true` dropping anyone at level ≥ N *and* stopping the descent through them — otherwise a junior listed as a senior's reporting manager would inherit that senior's whole subtree
3. **Everyone in the org strictly below level N** — `includeRoleHierarchy: true`

Both level tests are **strictly below**, so **peers are never visible**: a Team Lead cannot see another Team Lead's records. This is the intended contract, confirmed with the POC and the reporting manager on 2026-08-06.

`includeRoleHierarchy` / `capReportingByLevel` both **require `orgId` and `roleId`** and throw `400` without them — levels are per-org, and see the multi-role note below.

> **UPDATE 2026-08-07 — `includeRoleHierarchy` is OFF at all 11 call sites** (commit
> `780c3d99` "rbac toggle"). Visibility in the running system is therefore **purely
> reporting-based**: self + capped reporting downline. Item 3 above is dormant.
> Consequence worth knowing: a level-5 user with one report who owns no leads sees
> **zero** records — correct per the toggle, but a common source of "why can't I see
> anything?".

### The form-scope axis — `roles.visibility_scope` (2026-08-13)

The reporting tree is no longer the only way to be granted records. A role now
carries `visibility_scope`:

| Value | Meaning |
|---|---|
| `hierarchy` *(default)* | self + capped reporting downline — everything above |
| `forms` | the above **UNION** every lead on the user's allocated application forms |

Resolved by **`userService.resolveActorVisibility`**, which returns
`{ unscoped, userIds, formIds }`. The two arms are **OR'd, never AND'ed**:

```sql
WHERE counsellor_id = ANY(userIds) OR form_id = ANY(formIds)
```

so a `forms` user sees every lead on their forms **regardless of owner**,
including unassigned ones. An intersection would make the role useless — its
members typically own no leads at all.

`formIds` is filled from two places:

1. **The actor's own** allocated forms, only when the role they *signed in with*
   is `forms`-scoped (same acting-role rule as level).
2. **Their reports' forms** — for any user in the capped downline holding a
   `forms` role. Without this, making someone the reporting manager of a
   form-scoped user grants nothing, because such users own no rows. This cascade
   follows the **reporting tree only**; it is not `includeRoleHierarchy`.

Two deliberate decisions:

- **School allocation does not gate forms.** A form allocated to a user whose
  school is not allocated still grants visibility ("form is the authority"), so
  nothing an admin ticks is silently ignored. School only drives dropdown options.
- **A client-supplied `formId` can only narrow.** It is AND'ed onto the clause
  above, so passing an unallocated form still returns only rows the OR already
  permits. Form ids are always derived server-side.

> **Blast radius warning:** the scope of a `forms` role is set by *which forms you
> tick per user*, not by the role. Allocating a large form (PGP TBM ≈ 150k leads)
> grants that whole population. Ticking every form makes the role equivalent to a
> level-6 admin.

**Every** lead surface resolves through `resolveActorVisibility` — lead list,
applicants, archive, export, and all dashboard tiles — precisely so a dashboard
count can never disagree with the page it links to.

### `roleId` is mandatory — the multi-role trap

`getUserRbacContext` without a `roleId` returns the **highest** level across *all* the user's roles (per FINAL DECISIONS, multi-role users take their max). For visibility that is wrong: a user holding *Counsellor (L1)* **and** *PGP ADMIN (L5)*, signed in as the counsellor, would be scoped as L5 and see everything below level 5.

Always pass the **acting** role — `req.user.selectedRoleId` — into anything that resolves visibility. Real case: `robert.johnson@mastersunion.org` holds Counsellor (L1) + GM (L4); acting as counsellor he correctly sees 122 leads, not 644.

### Enforcement points

| File | Function(s) | Notes |
|---|---|---|
| `v2/services/manageLeadService.js` | `buildLeadScopeWhere`, `fetchApplicationManager`, `fetchArchiveLeads`, `fetchV2LeadsForExport` | lead list, applicants, archive, export. `bulkLeadResolver` + bulk workers inherit via `buildLeadScopeWhere` |
| `v2/services/adminDashboardV2Service.js` | `resolveVisibleCounsellorIds` → `applyCounsellorScope` / `buildScopeSql` | deliberately mirrors the lead list so a tile and the listing can never disagree |
| `v2/services/calendar.service.js` | `getCalendarEventsForUserAndReporting` | |
| `services/userDashboard.service.js` | 2 sites | |

Dashboard cache keys include `roleKey` + `userKey`, so two roles can never share a cached scope. **Purge after any change here** (`GET /api/internal/cache/purge-lm?key=kms4000&prefix=lmapicache:dashboard:v2:`); `rbacService` also holds a 5-minute in-process cache that no endpoint can reach.

**Changing a role's `visibility_scope` needs BOTH caches purged** — the RBAC
context (`prefix=cache:v2:rbac-context:`) and the dashboard — or the old scope
survives for up to 5 minutes.

#### The two dashboard totals must mirror the two list pages

Fixed 2026-08-13. `Total Leads` and `Total Applications` are clickthroughs into
`/admin/v2/manage-leads` and `/admin/v2/manage-applicants`, so both now build on
`buildLeadsListingWhere` — the same predicate those pages use (Applications adds
`type: 'applicant'`).

They previously diverged: the applicant tiles used `buildScopedLeadWhere`, which
additionally auto-filters to the user's allocated forms via `resolveScope`, while
the listing pages do not. A form-restricted user therefore saw one number on the
tile and a different one on the page behind it.

The **paid / unpaid** tiles still use `buildScopedLeadWhere` on purpose — they are
payment analytics with no clickthrough, so nothing has to reconcile against them.

> When adding a dashboard tile: if it links to a list, build its `where` the same
> way that list does. Otherwise it will drift, and users report it as a bug.

### Known gaps — deliberately NOT scoped

- **`v2/services/campaignRecipientResolver.js`** — email-campaign recipient resolution takes only `orgId` + `filterJson`; no `userId`/`roleId`. A Team Lead building a campaign can target every lead in the org. Its comment claims to mirror `buildLeadScopeWhere` but only mirrors the status/type filters, **not** counsellor visibility. Pre-existing, never scoped.
- **Dashboard Queries + Communication tiles** — remain org-wide; `queries` has no `v2_lead_id`/`counsellor_id` and communication logs have neither, so there is no join path yet.
- **People-pickers** (`getAllCounsellorsByUserOrganization`, `fetchCounsellorsByUser`) — intentionally unscoped; they populate assignment dropdowns, not record lists.

### Level is breadth, actions are power

A frequent modelling mistake: giving a read-only role level 1 to "limit" it. Level controls **how much data** is visible; **actions** control what can be done. Org 12 had `Admin View Only` and `Executive View` at level 1 — nothing below level 1, no leads assigned, so after this change they correctly see nothing. If such a role should see everything read-only, give it a **high level** and **view-only actions**.

### Verified live (org 12, 2026-08-06)

Dashboard total vs manage-leads total, per acting role — all exact matches against SQL:

| User | Acting role | Visible leads |
|---|---|---|
| Naman | ADMIN 2 (L5) | 201,216 — was 425,995 before the fix |
| Shreya | PGP TBM Counsellors (L3) | 83,645 |
| Anushka | Counsellor sales (L1) | 10 |
| Robert | Counsellor (L1), also holds GM (L4) | 122 — was 644 |

Per-role peer check (one real user per role): every level sees itself, **0 peers**, **0 above**, and everyone below.

---

## 16. Auth Storage & RBAC Request Performance — 2026-08-12

Frontend (`fe-anandi`) plus two backend query fixes. Nothing here changes *who*
can see what — §15 remains the authority on that.

### 16.1 One place for auth data

The token used to live in **four** places: the redux `auth` slice, and three
cookies — `crm_token`, `crm_user` (both JS-readable) and the backend's httpOnly
`token`. Every RTK service read the cookie *in preference to* redux
(`const cookieToken = getToken(); ... cookieToken || token`), so three copies of
the same credential drifted independently.

Now: **the `auth` slice in redux-persist is the only client-side store.**

| Need auth in… | Use |
|---|---|
| a React component | `useSelector(AuthSelector)` |
| an RTK `prepareHeaders` | `getState().persistedReducer[AuthSliceName]` |
| a plain module (no React, no `getState`) | `Redux/authAccess.js` |

`services/cookieService.js` is **deleted**. `Redux/authAccess.js` replaces it —
it stores nothing, it only reads live redux state, and `store.js` registers the
store with it so it can never form an import cycle.

> `getAuthOrgId()` returns `orgInternalId || organizationId` — i.e.
> `"masters-union"`, **not** `"12"`. That is what the old `crm_org_id` cookie
> held, and ~20 files interpolate it straight into request URLs. Returning the
> numeric id instead breaks them silently.

### 16.2 Two real bugs this fixed

**Logout did not log you out.** The frontend never called `/auth/logout`, and JS
cannot delete an httpOnly cookie — so the backend's `token` cookie survived
"logout" for its full `maxAge` (**1 day, or 30 days with Remember Me**) and kept
authenticating requests, because `jwtValidator` accepts it as a fallback.

**15 services authenticated on that cookie, not the header.** `jwtValidator` only
reads the header when it starts with `Bearer `, but 15 service files sent the raw
JWT with no prefix — including `rbacService` and `authService`. Their requests
were silently falling through to the cookie. All 15 now send `Bearer`.

There is now exactly one teardown, `authAccess.clearAuthEverywhere()`, used by
the logout button, the 401 interceptor and `globalFunctions`. It does all three
things that must happen together:

1. dispatch `LogOut` (empties the slice)
2. clear `localStorage` / `sessionStorage` (its persisted copy)
3. `POST /auth/logout` so the server drops the httpOnly cookie

`SetLoginData` also resets to a clean state before applying the payload — it
previously assigned only 28 of 35 fields, so `IP`, `paymentsData`, `roles` and
`organization` leaked across sessions.

### 16.3 The Sequelize cartesian trap (read this before adding an include)

`getUser` eager-loaded **five** `belongsToMany` associations in one `findByPk`.
Sequelize emits a single join, so Postgres materialises their cartesian product
before de-duplicating in JS. For an admin with 2 roles x 6 schools x 37 programs
x 36 forms that is **159,840 rows** to return ~81 rows of data.

| Endpoint | Before | After |
|---|---|---|
| `/rbac/me/context` | **8.5s** | **0.33–0.36s** |
| `/rbac/users/:id` | **6.4s** | **0.40s** |

Both now resolve the lists as two waves of parallel queries — junction ids
first, then the entities — so no list multiplies against another.

- `userService.getUserAllocations` is the lean path for `/me/context`, which
  only ever needed schools/programs/forms.
- `getUser` keeps its full contract (roles, reportingManagers, status …) and was
  verified byte-identical against captured baselines for four users.

> Both use an explicit `order: [['id','ASC']]`. The old join *happened* to come
> back id-ascending; `WHERE id IN (...)` guarantees nothing. `Header.jsx`
> auto-selects `schools[0]`, so a non-deterministic order could silently change
> which school a user lands on.

**Rule of thumb:** more than one `belongsToMany` include in a single query is a
cartesian product. Split them.

### 16.4 The `me/context` waterfall

Every query on the Admin Dashboard was gated on `isUserContextReady`, which came
from `/rbac/me/context` — so a cold load blocked ~8.5s before any dashboard
request was even issued. The `currentUserContext` slice is now persisted
(its own redux-persist key, so `store.currentUserContextState` and its 16
consumers are untouched), and the gate opens from the rehydrated copy while the
network refreshes behind it.

The slice resets on `LogOut` and `SetLoginData`, and stamps `orgId`/`userId` so a
persisted copy can never be shown to a different user.

### 16.5 Verified

Playwright, real logins against the live v2 DB: 38 routes with real API traffic
(no 401/403/5xx/JS errors), 20 Bearer / 0 raw headers, logout destroying the
httpOnly cookie with no JWT left in `localStorage`, cross-user isolation across a
logout→login, expired-session self-clear, and a full user **create → edit →
delete** round trip confirmed in both the DB and the UI.

---

## 17. `users.role` — Why It Is Neither an Authority Nor an Index — 2026-08-28

`users.role` has now caused two production problems for two completely unrelated
reasons, and the answer to both is "don't touch it". §15 covers why it is not a
permission authority. This section covers why it must not be *queried* — and, the
non-obvious half, why the fix is **not** to index it.

### 17.1 The incident

CRM-wide slowness, user complaints arriving in waves every ~15 minutes. RDS
Performance Insights:

| # | Load (AAS) | % | Query |
|---|---|---|---|
| 1 | 3.104 | **90.8%** | `SELECT id, email, password_hash, role, … FROM users` |
| 2 | 0.121 | 3.5% | `INSERT INTO timelines …` |
| 3 | 0.052 | 1.5% | `UPDATE communicationAudiences SET code…` |
| 4 | 0.029 | 0.9% | `SELECT … communicationAudiences JOIN communicationLogs` |
| 5 | 0.024 | 0.7% | `UPDATE communicationAudiences SET status…` |

Total 3.42 AAS on a 4-vCPU instance — roughly 85% of the box's parallelism, almost
all of it one query.

The query is `getSystemUser()` in `src/utils/queueUtils.js`:

```js
db.User.findOne({ where: { role: "super_admin" } })
```

`EXPLAIN ANALYZE`:

```
Limit  (actual time=0.599..56.036 rows=1)
  ->  Gather   Workers Planned: 2   Workers Launched: 2
        ->  Parallel Seq Scan on users
              Filter: (role = 'super_admin')
              Rows Removed by Filter: 140622
              Buffers: shared hit=84102
```

84,102 buffers ≈ **657 MB read, 56 ms, and 3 CPUs** (parent plus 2 parallel workers)
**per call**. `users` is 931 MB across 423,493 live rows, and had accumulated 4.3M
sequential scans reading 555 billion rows cumulatively.

### 17.2 It was not the frontend

No page caused this. The origin is an inbound vendor webhook:

```
Karix (WhatsApp vendor)
  → POST /webhooks/karix   (+ /karix-ug, /karix-pgprise, /karix-pgptbm)
  → handleKarixWebhook.controller → karixWhatsappQueue.add("deliveryEventWebhook")
  → karixWhatsapp.worker          → getSystemUser() → 657 MB scan
```

Every delivery event (sent / delivered / read / failed) becomes one queue job, and
each job calls `getSystemUser()` one to three times — `karixWhatsapp.worker:249`,
`:670`, and `deliveryStatus.helper:149` reached via `:134`. At ~460,000 messages/day
that is millions of scans.

Note the whole top-5 is a *single pipeline*: #2 is the timeline write, #3/#4/#5 the
delivery status write-back. **97.4% of database load was the WhatsApp delivery
flow.** The only frontend endpoint that touches `getSystemUser()` is
`PATCH /waba/chat/:chatUUID/status`, once per agent click — irrelevant by volume.

> **Lesson.** When the CRM is slow "everywhere at once" and no single page is at
> fault, look at queue workers and vendor webhooks before looking at the UI. Shared
> RDS saturation makes every page slow regardless of what the user clicked.

### 17.3 The fix: cache, don't index

`getSystemUser()` now resolves once per process, with an in-flight promise guard to
collapse the cold-start stampede (without it, every job starting before the first
lookup resolves fires its own scan). Measured: 8 concurrent calls → **1 query**;
2,000 sequential calls → **1 query total, 1 ms**.

The query text itself was left byte-identical, deliberately — see below.

### 17.4 Why NOT an index on `users.role` — the part that matters

Adding `CREATE INDEX ON users (role)` is the obvious fix and it is **wrong**. There
are **4** rows with `role = 'super_admin'`, and the query has **no `ORDER BY`**:

```sql
SELECT … FROM users WHERE role = 'super_admin' LIMIT 1;
```

`LIMIT 1` without `ORDER BY` returns an *arbitrary* row — whichever the chosen plan
reaches first. Under a sequential scan that is physical heap order, which today
yields id 5913. An index scan walks the index instead and can return a different one
of the four.

That id is written as the actor on system-authored records — `timelines.created_by`,
WhatsApp chat messages, stage logs. So the index would silently re-attribute every
future system-authored record to a different user, with no error raised and no
migration to point at. **The table would get faster and the data would get wrong.**

> **Rule.** Never add an index to change the plan of a `LIMIT`-without-`ORDER BY`
> query. Fix the query first — a deterministic `ORDER BY`, or a lookup by unique key
> — or remove the need for it, as here, by caching. Only then consider the index.

Related trap: `src/utils/getSystemUser.js` is a second, already-cached implementation
that resolves a **different** user (id 4441913, looked up by email, which uses
`users_email`). The two are not interchangeable — repointing callers from one to the
other also changes attribution.

### 17.5 Standing rules for `users.role`

1. **Not a permission authority.** It only records which portal a user belongs to —
   admin, super admin, or student. Use the acting role's level and action keys
   (§15), never this column.
2. **Do not filter by it.** Unindexed on a 931 MB table; every such query is a
   parallel sequential scan occupying 3 CPUs.
3. `roleValidator('super_admin', 'admin', 'manager')` in `lead.routes.js`
   (`bulk-archive`, `bulk-unarchive`, `bulk-delete`) still gates on this column.
   Legacy — do not copy it into new routes; use `rbacMiddleware.requireAction`.
4. **A system actor must come from a cached helper.** Never look one up per message,
   per job, or per webhook.

### 17.6 Verified

`EXPLAIN ANALYZE` captured before the change. After: 2,008 calls produced 1 query,
and `getSystemUser()` returned id 5913 — identical to the pre-change row, so
attribution is unchanged. Diff is **+34 / −0** lines in one file; eslint output
identical before and after (3 pre-existing errors, 1 warning, none in the edited
region). No schema change and no migration — the fix takes effect on restart, and
the cached value is held per process until it restarts.

### 17.7 Open, not fixed

The row this resolves to is **id 5913, `superadminstaging@test.com`, status
`disabled`** — a staging test account currently recorded as the actor on production
WhatsApp and SMS timeline entries. Pre-existing, and deliberately left alone: it is
an attribution decision, not a performance fix. Worth deciding whether these should
point at the real automation user (id 4441913).

---

## 18. One Email, Several `users` Rows — Identity, Duplicates & Super-Admin Accounts — 2026-09-13

`users.email` has **no unique constraint**. There are only three plain indexes on it
(`users_email`, `users_email_status`, `users_email_lower_idx`). The same address
legitimately sits on many rows — `rohit.yadav@mastersunion.org` has 4 student rows.
Read this before writing any "does this email already exist?" check.

### 18.1 The identity rule

- **Student rows are a separate identity.** One person can be a student (often on
  several rows) and also be staff. A student row never blocks a staff account.
- **Staff rows (`admin`, `super_admin`) must be unique per email, ignoring case, in
  every status.** Every staff auth lookup silently assumes this:

| Lookup | Where | Filter |
|---|---|---|
| `adminLoginV2` | `controllers/users/authController.js:344` | `LOWER(email) = x AND role <> 'student'` |
| `superAdminLogin` | `controllers/users/authController.js:159` | `LOWER(email) = x AND role = 'super_admin'` (no status filter) |
| `forgotPassword` | `controllers/users/authController.js:744` | exact `email = x`, then `ILIKE`; `role <> 'student'` |
| v2 `createUser` | `v2/services/userService.js:789` | `email = x AND role <> 'student'` |

None of them has an `ORDER BY`. If two staff rows share an email, which one you get
is arbitrary — the same trap as §17.4.

> **Rule.** A duplicate-email check for a staff account must (1) ignore student rows,
> (2) compare with `LOWER(email)`, and (3) include every status, `disabled` too —
> `superAdminLogin` does not filter status, so a disabled twin can shadow the live one.

### 18.2 Fixed — "A user with this email already exists" for a brand-new super admin

**Symptom.** Super-admin portal → `/profile` → *Create super admin* with
`rohit.yadav@mastersunion.org` → `409 A user with this email already exists`, while
`SELECT … WHERE role = 'super_admin'` returned 0 rows.

**Cause.** `createSuperAdmin` (`controllers/super_admin/userController.js:175`,
route `POST /api/super_admin/users/accounts`) checked `where: { email }` — any role,
any status. It matched Rohit's 4 student rows.

**Fix** (`userController.js:204`). The check is now `LOWER(email) = x AND role <> 'student'`,
and the message says what it clashed with:

| Existing staff row | Message |
|---|---|
| `super_admin`, active | A super admin with this email already exists |
| `super_admin`, disabled | A deleted super admin account with this email already exists |
| `admin` | An admin account with this email already exists |

**Verified** (read-only, v2 DB): `rohit.yadav@…` → no conflict (the old rule matched 4 rows);
existing super admin id 4899981 → still blocked; existing admin id 6001 → still blocked.

**Same bug, not fixed:** `updateMe` (`userController.js:107`) still checks
`email = x AND id <> me` — any role, exact case. A super admin changing their own email
to an address that has student rows is wrongly rejected.

### 18.3 Open — mixed-case staff twins break login (the kapish case)

`kapish.sabharwal@mastersunion.org` has two staff rows whose emails differ only by case.

1. `forgotPassword` tries an exact match first → it resets the **lowercase** row.
2. `adminLoginV2` uses `LOWER(email)` with no `ORDER BY` → it can pick the **capital-K** row.
3. Result: "Incorrect Password" straight after a successful reset.

Two more users have the same shape. Proposed, **not run** (awaiting approval):

```sql
-- park the orphan rows so each staff email resolves to exactly one row
UPDATE users SET email = 'dup-' || id || '.' || email, updated_at = NOW()
 WHERE id IN (4627803, 4749008, 4644995);
```

Code follow-up: one shared `findAuthUserByEmail()` (exact match first, then
case-insensitive) used by both login and password reset, so they can never disagree.

### 18.4 Security — `POST /api/users/` creates a super admin with no login

See **CS6** in §6. Summary:

- `routes/users.routes.js:47` — `router.post('/', createUser)`: no `jwtValidator`,
  no rate limit.
- `controllers/users/userController.js` `createUser` takes `role` from the body and
  accepts `super_admin`.
- **Confirmed live** on `api-v2.mastersunion.org` (`201`, user id 4895048,
  `role: super_admin`). That account can sign in to the super-admin portal through
  `superAdminLogin`.
- **Nothing calls it.** No caller in `fe-anandi`, `super-admin`, `widget-v2`, or the
  backend. All real user creation goes through `admin/users`,
  `org/:orgId/rbac/createUser`, `v2/org/:orgId/rbac/users` or its bulk upload.
- **Recommended:** delete the route line. **Status: open, not changed.**

To review super admins created outside the portal (they have no `super_admin.create`
audit row). Older bootstrap accounts will also appear, so treat it as a list to check,
not proof:

```sql
SELECT u.id, u.email, u.status, u.created_at
  FROM users u
 WHERE u.role = 'super_admin'
   AND NOT EXISTS (SELECT 1 FROM audit_logs a
                    WHERE a.action = 'super_admin.create'
                      AND a.target_id = u.id::text)
 ORDER BY u.created_at DESC;
```

### 18.5 Super-admin portal — *Super admin accounts* card and *Create* drawer

File: `super-admin/src/features/superAdminProfile/ui/SuperAdminProfilePage.jsx`.

**a) Empty list while the API returned rows — fixed.**
The card showed "No super admin accounts match this search." *and* "Load more", while
`GET /accounts?page=1&limit=4` returned users. The page copies API rows into local
`accounts` state (so "Load more" can append). The 350 ms search debounce ran on mount
and **always** emptied that list:

| Time | Event | `accounts` |
|---|---|---|
| 0 ms | page mounts, request starts, 350 ms timer starts | `[]` |
| ~250 ms | response arrives, sync effect copies rows | 4 rows |
| 350 ms | timer fires, empties the list; search is still empty | `[]` |
| after | query arg unchanged → no new data → sync effect never re-runs | stays `[]` |

It hit whenever the API answered in under 350 ms, and every time on in-app navigation
back (cached data). Fix (line 44): the debounce resets page/list **only when the
trimmed search actually changed**.

> **Rule.** Never clear state that is filled from an RTK Query result unless the query
> argument changes in the same update. If the argument stays the same, no new data
> arrives to refill it.

**b) Create errors are shown inline only — changed.**
The backend error used to appear three times: two toasts (the page's own `pushToast`
plus the global `rtkErrorMiddleware`) and the red `DrawerError`.

- `infrastructure/api/rtkErrorMiddleware.js:17` — new `INLINE_ERROR_ENDPOINTS` set
  (`createSuperAdmin`). Endpoints in it get no global toast. A 401 still shows
  "Session expired" and logs out.
- The drawer's `catch` no longer calls `pushToast`. The message stays in `DrawerError`.
- `updateCreateForm` (line 30) wraps every drawer field change and clears that error.

To make another screen inline-only: add its RTK `endpointName` to the set and do not
`pushToast` in its `catch`.

### 18.6 Said "no permission", was not RBAC — test-email 403

"No permission to send test emails" for `admin-gurgaon@anandi.org`, with no such
permission configured anywhere.

- `/sendTestEmail` has **no `requireAction`**. The 403 comes from **sender
  verification** (`controllers/templateManager/communicationEmail.ctrl.js:124`): the
  `fromEmail` was not one of the org's configured senders.
- The frontend labelled every 403 as a permission error.
- **Fixed in `fe-anandi`:** new `src/utils/orgSender.js` (`resolveOrgSender`) picks
  the org's transactional sender. `EmailPreviewFilter.jsx` (used the logged-in user's
  email) and `SideFilter/PreviewFilter/PreviewFilter.jsx` (hardcoded
  `admissions@anandi.org`) now use it. The false "permission" label is removed there
  and in `EmailBuilder.jsx` / `EmailBuilderV2.jsx`; the backend message is shown instead.
- **Open:** the backend should arguably return 400/422 for this, not 403.

> **Lesson.** Before debugging RBAC for a "no permission" message, check that the route
> actually has a `requireAction`.

### 18.7 Diagnoses with no code change

- **Login role picker ≠ `user_roles` rows.** The picker lists only `user_roles` rows
  with `is_active = true`. For `kishan.soni@mastersunion.org` the Admin row had
  `is_active = false`, so it was not offered. `audit_logs` shows who deactivated it
  and when. Note there are three different "removed" flags:
  `user_roles.is_active` (role assignment), `org_users.status = 'removed'`
  (membership), `v2_leads.is_deleted` (data).
- **Allocated forms, but no leads visible.** `rahul1+ug2@…` and `chandana.jaiswal+1@…`
  hold `visibility_scope = 'hierarchy'` roles, own 0 leads and have 0 reports.
  Their form allocations (45,899 leads) are ignored by a hierarchy role — working as
  designed (§15). Pending decision: set role 58 "UG ADMIN" (2 holders) to `'forms'`;
  do **not** flip role 46 (564 holders, ~24 forms each — it would widen hundreds of
  users). Move Chandana to an existing forms role (40 "Admin View Only" / 56
  "Executive View") instead.

### 18.8 Open, not fixed

| # | Item | See |
|---|---|---|
| 1 | Remove unauthenticated `POST /api/users/`; review super admins it may have created | CS6, §18.4 |
| 2 | Park the 3 mixed-case staff twin rows; add shared `findAuthUserByEmail()` | §18.3 |
| 3 | `updateMe` duplicate-email check: same bug as §18.2 | §18.2 |
| 4 | Test-email sender failure should not be HTTP 403 | §18.6 |
| 5 | Role 58 / Chandana visibility-scope decision | §18.7 |
| 6 | No DB unique index for staff emails — a partial `UNIQUE (LOWER(email)) WHERE role <> 'student'` would enforce §18.1, but only after #2 is cleaned up | §18.1 |
