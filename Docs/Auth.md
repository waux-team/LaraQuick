# Authentication System — Database Design Report

## Design Philosophy

The goal is a **stateless, token-based API authentication** system built on Laravel Sanctum. Rather than storing passwords directly on users, the existing normalized schema is reused — email lives in `user_contacts`, password hash in `user_credentials`. This keeps the security isolation enforced by the Profile service intact. The only new table is `password_reset_tokens` for time-limited reset flows.

---

## Table Overview

```
users                        (existing — anchor table)
  ├── user_contacts          (existing — email lookup for login)
  ├── user_credentials       (existing — password hash verification)
  └── personal_access_tokens (Sanctum — API token storage)

password_reset_tokens        (new — time-limited reset tokens)
```

---

## New Table

### password_reset_tokens

Stores hashed reset tokens with a time-based expiry. One row per email — subsequent requests overwrite the previous token.

| Column | Type | Notes |
|---|---|---|
| email | VARCHAR(255) | Primary key, matches `user_contacts.email` |
| token | VARCHAR(255) | Bcrypt-hashed reset token |
| created_at | TIMESTAMP | Nullable, used for 60-minute expiry check |

---

## Existing Tables Used

### personal_access_tokens (Sanctum)

Managed by Laravel Sanctum. Stores hashed API tokens linked to the `users` table via polymorphic `tokenable_type` + `tokenable_id`.

| Column | Type | Notes |
|---|---|---|
| id | BIGINT | Auto-increment PK |
| tokenable_type | VARCHAR | Polymorphic type (`App\Models\User`) |
| tokenable_id | UUID | FK → users.user_id |
| name | VARCHAR | Token label (e.g. `auth`) |
| token | VARCHAR(64) | SHA-256 hash of the plaintext token |
| abilities | TEXT | JSON array of abilities (default `["*"]`) |
| last_used_at | TIMESTAMP | Nullable |
| expires_at | TIMESTAMP | Nullable |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

Unique index on `token`.

---

## Authentication Flows

### Register

```
Client → POST /api/auth/register
  → AuthController → AuthService::register()
    → UserRepository::create()           (create user row)
    → UserContactRepository::upsertForUser()  (store email)
    → UserCredentialRepository::rotatePassword() (hash + store password)
    → User::createToken('auth')          (Sanctum token)
  ← { user: ProfileResource, token: "1|abc..." }
```

All wrapped in `DB::transaction`.

### Login

```
Client → POST /api/auth/login
  → AuthController → AuthService::login()
    → UserContactRepository::findByEmail()       (lookup user by email)
    → UserRepository::find()                     (load user, check is_active)
    → UserCredentialRepository::findByUserId()   (load credential)
    → Hash::check(password, password_hash)       (verify)
    → User::createToken('auth')                  (Sanctum token)
  ← { user: ProfileResource, token: "2|def..." }
```

### Logout

```
Client → POST /api/auth/logout  [Authorization: Bearer {token}]
  → auth:sanctum middleware resolves user
  → AuthController → AuthService::logout()
    → currentAccessToken()->delete()     (revoke current token only)
  ← { message: "Logged out" }
```

### Forgot Password

```
Client → POST /api/auth/forgot-password
  → AuthController → AuthService::forgotPassword()
    → UserContactRepository::findByEmail()
    → Generate random 64-char token
    → Hash + store in password_reset_tokens (updateOrInsert)
    → (Production: send email with plaintext token)
  ← { message: "If the email exists, a reset link has been sent" }
```

Always returns 200 — prevents email enumeration.

### Reset Password

```
Client → POST /api/auth/reset-password
  → AuthController → AuthService::resetPassword()
    → Lookup hashed token in password_reset_tokens
    → Hash::check(submitted_token, stored_hash)
    → Check created_at + 60 minutes not past
    → UserCredentialRepository::rotatePassword()  (new hash)
    → Delete token row from password_reset_tokens
    → User::tokens()->delete()  (revoke all Sanctum tokens)
  ← { message: "Password has been reset" }
```

---

## Key Design Decisions

**Reuses existing Profile schema** — no new `users.email` or `users.password` columns. Email lookup goes through `user_contacts`, password verification through `user_credentials`. This preserves the normalized 3NF design and the principle that only the auth service touches credentials.

**Sanctum for stateless API auth** — no sessions, no cookies. Each login generates a bearer token stored in `personal_access_tokens`. Tokens are SHA-256 hashed before storage — the plaintext is returned only once at creation.

**Token-per-login, not token-per-user** — multiple concurrent tokens are supported (e.g. mobile + web). `logout` revokes only the current token; `resetPassword` revokes all tokens as a security measure.

**Hashed reset tokens** — the plaintext token is never stored. `password_reset_tokens.token` holds a bcrypt hash. This means even database access doesn't expose usable reset tokens.

**60-minute token expiry** — reset tokens are valid for 60 minutes from `created_at`. Expired tokens fail silently. Configurable via `config('auth.passwords.users.expire')`.

**Email enumeration prevention** — `forgotPassword` always returns the same success response regardless of whether the email exists. `login` returns a generic "Invalid credentials" message without distinguishing between "email not found" and "wrong password".

**User model extends Authenticatable** — the Profile service originally used `Illuminate\Database\Eloquent\Model`. The Auth service changes the base class to `Illuminate\Foundation\Auth\User` (aliased as `Authenticatable`) to support Sanctum's `HasApiTokens` and Laravel's `Notifiable` trait for password reset emails.

**No new repositories** — `password_reset_tokens` is a simple key-value table accessed via `DB::table()` directly. Creating a full repository + interface for a single-table CRUD with no complex queries would be over-engineering.

---

## Security Layers

| Concern | Solution |
|---|---|
| Password storage | Bcrypt/Argon2 hash in `user_credentials.password_hash` |
| Token storage | SHA-256 hash in `personal_access_tokens.token` |
| Reset token storage | Bcrypt hash in `password_reset_tokens.token` |
| Email enumeration | Identical responses for existing/non-existing emails |
| Token expiry | 60-minute window on reset tokens; Sanctum tokens optionally expire |
| Credential stuffing | Deactivated users (`is_active = false`) cannot login |
| Session hijacking | Stateless tokens — no server-side session to hijack |
| Password reset | Revokes all Sanctum tokens after password change |
| GDPR erasure | `ON DELETE CASCADE` from users covers `personal_access_tokens` via polymorphic cleanup |

---

## API Endpoints

| Method | URI | Auth | Status | Response |
|---|---|---|---|---|
| POST | `/api/auth/register` | Public | 201 | `{ user, token }` |
| POST | `/api/auth/login` | Public | 200 / 401 | `{ user, token }` \| `{ message }` |
| POST | `/api/auth/logout` | Bearer | 200 | `{ message }` |
| POST | `/api/auth/forgot-password` | Public | 200 | `{ message }` |
| POST | `/api/auth/reset-password` | Public | 200 / 422 | `{ message }` |

---

*Designed for PostgreSQL 15+, Laravel 13, PHP 8.3+. Uses Laravel Sanctum ^4.0. Schema version 1.0.*
