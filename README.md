# LaraQuick

A scaffold/template repository for Laravel 13 APIs. Write features here first, review, then copy into the target project.

**Stack:** Laravel 13 · PHP 8.3+ · PostgreSQL 15+ · Laravel Sanctum

**Architecture:** Repository–Service pattern. Controllers handle HTTP only; Services own business logic; Repositories own all Eloquent queries; APIs wrap external HTTP calls; Models are thin data structures.

---

## Features

### Auth
Stateless token-based authentication via Laravel Sanctum.

- Register, login, logout
- Forgot password / reset password (hashed tokens, 60-minute expiry)
- Email enumeration prevention on all flows
- Credentials isolated in `user_credentials` — no other service touches it

### Profile
Normalized user schema in 3NF. Sensitive data split into dedicated tables so services only access what they need.

- `users` — identity anchor (UUID PK, `is_active` soft-delete)
- `user_contacts` — email + phone, verification state
- `user_credentials` — password hash, MFA secret (auth service only)
- `user_addresses` — multiple addresses per user (home / billing / shipping)
- `user_sessions` — active login session tracking
- `user_locations` — append-only GPS/IP location log

### RBAC
Scoped, inheritance-aware roles and permissions. No third-party RBAC packages — all resolution logic is owned by this codebase.

- Scope hierarchy: Global → Organisation → Team
- Users hold different roles per scope simultaneously
- Role inheritance (DAG): `super_admin` → `admin` → `moderator` → `member` → `viewer`
- Direct per-user permission grants (additive on top of roles)
- `super_admin` short-circuits all permission checks globally
- Time-bounded access via `expires_at` on role and permission assignments
- Immutable `permission_audit_log` — every assign/revoke writes a row

---

## Rules

See [RULES.md](RULES.md) for full code style, layer constraints, and generation instructions.
