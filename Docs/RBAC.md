# Scoped RBAC System — Database Design Report

## Design Philosophy

The goal is a **fully scoped, auditable, and inheritance-aware** Role & Permission system. Roles and permissions are not flat — they exist within a scope hierarchy (Global → Organisation → Team) so that a user can hold different roles in different contexts simultaneously. No third-party RBAC packages are used; every resolution rule is owned by this codebase and is therefore fully auditable and adaptable.

---

## Scope Hierarchy

```
global
  └── organisations                (1 : many)
        └── teams                  (1 : many per org)

users
  ├── org_members                  (many : many via organisations)
  ├── team_members                 (many : many via teams)
  ├── user_roles                   (scoped: global | org | team)
  └── user_permissions             (scoped: global | org | team)

roles
  ├── role_permissions             (many : many via permissions)
  └── role_inheritance             (self-referential many : many)

permission_audit_log               (append-only event log)
```

---

## Core Tables

### 1. organisations

The top-level scope unit. Teams are always children of an organisation.

| Column | Type | Notes |
|---|---|---|
| org_id | UUID | Primary key |
| name | VARCHAR(120) | Required |
| slug | VARCHAR(80) | Unique, URL-safe identifier |
| description | TEXT | Nullable |
| is_active | BOOLEAN | Default true |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

Index `slug`.

---

### 2. teams

A sub-unit within an organisation. Cascades deleted when its parent org is deleted.

| Column | Type | Notes |
|---|---|---|
| team_id | UUID | Primary key |
| org_id | UUID | FK → organisations.org_id ON DELETE CASCADE |
| name | VARCHAR(120) | Required |
| slug | VARCHAR(80) | Unique within its org |
| description | TEXT | Nullable |
| is_active | BOOLEAN | Default true |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

Unique index on `(org_id, slug)`. Index `org_id`.

---

### 3. org_members

Tracks which users belong to an organisation, independent of role assignment.

| Column | Type | Notes |
|---|---|---|
| org_member_id | UUID | Primary key |
| org_id | UUID | FK → organisations.org_id ON DELETE CASCADE |
| user_id | UUID | FK → users.user_id ON DELETE CASCADE |
| invited_by | UUID | Nullable, FK → users.user_id (no cascade) |
| joined_at | TIMESTAMP | Nullable |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

Unique index on `(org_id, user_id)`. Index `user_id`.

---

### 4. team_members

Tracks which users belong to a team, independent of role assignment.

| Column | Type | Notes |
|---|---|---|
| team_member_id | UUID | Primary key |
| team_id | UUID | FK → teams.team_id ON DELETE CASCADE |
| user_id | UUID | FK → users.user_id ON DELETE CASCADE |
| invited_by | UUID | Nullable, FK → users.user_id (no cascade) |
| joined_at | TIMESTAMP | Nullable |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

Unique index on `(team_id, user_id)`. Index `user_id`.

---

### 5. roles

Named roles that can be assigned to users at any scope level. System roles (`is_system = true`) cannot be deleted via the API.

| Column | Type | Notes |
|---|---|---|
| role_id | UUID | Primary key |
| name | VARCHAR(80) | Unique, lowercase snake_case |
| display_name | VARCHAR(120) | Human-readable label |
| description | TEXT | Nullable |
| is_system | BOOLEAN | Default false — system roles are protected |
| scope_hint | ENUM('global','org','team') | Nullable, advisory only — not enforced at DB level |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

---

### 6. role_inheritance

Self-referential pivot that defines parent-child relationships between roles. A child role inherits all permissions of its parent, recursively. Cycles are prevented at the application layer.

| Column | Type | Notes |
|---|---|---|
| parent_role_id | UUID | FK → roles.role_id ON DELETE CASCADE |
| child_role_id | UUID | FK → roles.role_id ON DELETE CASCADE |

Composite PK on `(parent_role_id, child_role_id)`. Index each FK column individually. No timestamps.

---

### 7. permissions

Atomic capabilities expressed as `resource:action` strings (e.g. `profiles:delete`). Each combination is unique.

| Column | Type | Notes |
|---|---|---|
| permission_id | UUID | Primary key |
| resource | VARCHAR(80) | e.g. `profiles`, `teams` |
| action | VARCHAR(80) | e.g. `read`, `delete` |
| name | VARCHAR(160) | Unique — always `resource:action` |
| display_name | VARCHAR(200) | Nullable, human-readable |
| description | TEXT | Nullable |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

Unique index on `(resource, action)`.

---

### 8. role_permissions

Pivot assigning permissions to roles. No expiry — permissions belong to a role until explicitly synced.

| Column | Type | Notes |
|---|---|---|
| role_id | UUID | FK → roles.role_id ON DELETE CASCADE |
| permission_id | UUID | FK → permissions.permission_id ON DELETE CASCADE |

Composite PK on `(role_id, permission_id)`. No timestamps.

---

### 9. user_roles

Assigns a role to a user within a specific scope. The same user can hold different roles at different scope levels simultaneously.

| Column | Type | Notes |
|---|---|---|
| user_role_id | UUID | Primary key |
| user_id | UUID | FK → users.user_id ON DELETE CASCADE |
| role_id | UUID | FK → roles.role_id ON DELETE CASCADE |
| scope_type | ENUM('global','org','team') | Default `global` |
| scope_id | UUID | Nullable — org_id or team_id depending on scope_type |
| assigned_by | UUID | Nullable, FK → users.user_id (no cascade) |
| assigned_at | TIMESTAMP | Default now() |
| expires_at | TIMESTAMP | Nullable — role lapses when this passes |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

Unique index on `(user_id, role_id, scope_type, scope_id)`. Index `(user_id, scope_type, scope_id)`.

---

### 10. user_permissions

Direct (non-role) permission grants to a specific user within a scope. These are additive on top of role-based permissions.

| Column | Type | Notes |
|---|---|---|
| user_permission_id | UUID | Primary key |
| user_id | UUID | FK → users.user_id ON DELETE CASCADE |
| permission_id | UUID | FK → permissions.permission_id ON DELETE CASCADE |
| scope_type | ENUM('global','org','team') | Default `global` |
| scope_id | UUID | Nullable |
| granted_by | UUID | Nullable, FK → users.user_id (no cascade) |
| granted_at | TIMESTAMP | Default now() |
| expires_at | TIMESTAMP | Nullable |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |

Unique index on `(user_id, permission_id, scope_type, scope_id)`. Index `(user_id, scope_type, scope_id)`.

---

### 11. permission_audit_log

Immutable event log. Every role assignment, revocation, and permission grant produces one row. Never updated or deleted.

| Column | Type | Notes |
|---|---|---|
| audit_id | UUID | Primary key |
| actor_id | UUID | Nullable — who performed the action |
| target_user_id | UUID | Nullable — who was affected |
| action | ENUM | `role_assigned`, `role_revoked`, `permission_granted`, `permission_revoked`, `role_created`, `role_deleted`, `permission_created`, `permission_deleted`, `member_added`, `member_removed` |
| subject_type | VARCHAR(50) | e.g. `role`, `permission` |
| subject_id | UUID | Nullable — the role_id or permission_id involved |
| subject_name | VARCHAR(200) | Snapshot of the name at time of action |
| scope_type | VARCHAR(20) | Nullable |
| scope_id | UUID | Nullable |
| metadata | JSON | Nullable — arbitrary extra context |
| created_at | TIMESTAMP | Write-once, set on insert |

Index `(target_user_id)`, `(actor_id)`, `(created_at)`, `(scope_type, scope_id)`. No `updated_at`.

---

## Key Design Decisions

**Membership and role assignment are separate concerns** — a user can be a member of an organisation (`org_members`) without holding any role there, and can hold a role at a scope without being listed as a member. This separation allows read-only observers and invited but unprivileged users.

**Permission resolution uses union logic (most permissive wins)** — the system collects roles from all applicable scopes (global, the requested scope, and any org-level cascade for team scopes), then unions their permissions together with any direct `user_permissions`. The first matching hit returns `true`.

**Org-to-team cascade during resolution** — when checking a team-scope permission, the resolver automatically also loads the user's org-level roles for the team's parent organisation. An admin at the org level is therefore implicitly an admin in every team under that org, without needing explicit team-role assignments.

**`super_admin` bypasses all checks** — a user with an active `super_admin` role (at any scope) short-circuits the entire resolution chain and returns `true` for every permission check globally. This is intentional and is evaluated first.

**Role inheritance is a DAG, not a flat list** — roles inherit permissions from parent roles recursively. A `member` role inherits from `viewer`; `moderator` inherits from `member`; `admin` from `moderator`; `super_admin` from `admin`. Cycle detection is the application's responsibility.

**`expires_at` on both `user_roles` and `user_permissions`** — time-bounded access is a first-class feature, not an afterthought. Expired records are kept in the table for audit purposes but are excluded from all resolution queries.

**`scope_hint` is advisory only** — it documents the intended scope for a role (`global`, `org`, or `team`) but is not enforced as a database constraint. A role marked `scope_hint = 'team'` can technically be assigned at global scope; the application layer decides whether to allow that.

**`is_system` roles are soft-protected** — `deleteRole()` returns `false` silently for system roles rather than throwing. This prevents accidental deletion of foundational roles (`viewer`, `admin`, `super_admin`) without requiring a hard DB constraint.

**No Spatie or third-party RBAC packages** — all resolution logic lives in `ScopedPermissionService`. This keeps the permission model fully inspectable, testable, and free from external dependency drift.

---

## Recommended Indexes

| Index | Table | Purpose |
|---|---|---|
| `(slug)` UNIQUE | organisations | Slug-based org lookup |
| `(org_id, slug)` UNIQUE | teams | Slug-based team lookup within org |
| `(org_id, user_id)` UNIQUE | org_members | Membership checks and deduplication |
| `(team_id, user_id)` UNIQUE | team_members | Membership checks and deduplication |
| `(resource, action)` UNIQUE | permissions | `firstOrCreate` lookups by resource+action |
| `(user_id, scope_type, scope_id)` | user_roles | Core resolution query — called on every `userCan()` |
| `(user_id, scope_type, scope_id)` | user_permissions | Direct permission resolution |
| `(target_user_id)` | permission_audit_log | Per-user audit history |
| `(scope_type, scope_id)` | permission_audit_log | Scope-filtered audit queries |
| `(created_at)` | permission_audit_log | Time-ordered log reads |

---

## Security Layers

| Concern | Solution |
|---|---|
| Permission escalation | No user can exceed the permissions held by their assigning actor (enforced at application layer) |
| Scope leakage | Resolution always requires an explicit `scope_type` + `scope_id` — no implicit cross-scope bleed except the documented org→team cascade |
| Expired access | `expires_at` checked on every resolution query; expired rows are excluded, not deleted |
| Audit trail | Every mutation (assign, revoke, grant) writes to `permission_audit_log` inside a `DB::transaction` |
| System role protection | `is_system = true` roles cannot be deleted; their permissions are synced only via the seeder or explicit admin API |
| Enumeration resistance | UUID primary keys on all tables prevent sequential ID scanning |
| Middleware guard | `spermission` middleware reads `{org}` and `{team}` route params — routes must use those exact parameter names for scoped gates to activate |

---

## Normalization Summary

| Table | Normal Form | Notes |
|---|---|---|
| organisations | 3NF | No transitive dependencies |
| teams | 3NF | All non-key columns depend only on team_id |
| org_members | 3NF | Membership fact table; no derived columns |
| team_members | 3NF | Membership fact table; no derived columns |
| roles | 3NF | Display fields depend only on role_id |
| role_inheritance | 3NF | Pure relationship pivot; no payload columns |
| permissions | 3NF | `name` is a deterministic derivative of resource+action |
| role_permissions | 3NF | Pure relationship pivot |
| user_roles | 3NF | All columns depend on user_role_id; scope columns are not transitive |
| user_permissions | 3NF | Same structure as user_roles |
| permission_audit_log | 3NF | Append-only snapshot; `subject_name` is intentionally denormalized for historical accuracy |

---

## Foreign Key Cascade Rules

| Child Table | On Delete Parent |
|---|---|
| teams | CASCADE from organisations |
| org_members | CASCADE from organisations and users |
| team_members | CASCADE from teams and users |
| role_permissions | CASCADE from roles and permissions |
| role_inheritance | CASCADE from roles (both parent and child) |
| user_roles | CASCADE from users; role row stays (audit value) |
| user_permissions | CASCADE from users; permission row stays |

`invited_by` and `assigned_by` / `granted_by` / `actor_id` columns use no cascade — the actor record is preserved for audit purposes even if the actor's own account is removed.

---

## Seeded Roles

The `RbacSeeder` creates the following system roles with inheritance:

```
super_admin  (all permissions)
  └── admin  (full scope management)
        └── moderator  (profile and address management)
              └── member  (standard create/update)
                    └── viewer  (read-only)
```

| Role | Scope Hint | Inherits From |
|---|---|---|
| viewer | — | — |
| member | team | viewer |
| moderator | org | member |
| admin | org | moderator |
| super_admin | global | admin |

---

*Designed for PostgreSQL 15+, Laravel 13, PHP 8.3+. Schema version 1.0.*
