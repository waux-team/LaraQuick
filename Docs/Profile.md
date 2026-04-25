# Profile Information System — Database Design Report

## Design Philosophy

The goal is a **normalized, secure, and extensible** schema. Sensitive data is isolated into dedicated tables to enforce the principle of least privilege — a service that only needs to send emails should never touch session or credential records. All tables are in **3NF** (Third Normal Form) to eliminate redundancy and keep updates clean.

---

## Table Overview

```
users
  ├── user_contacts       (1 : 1)
  ├── user_credentials    (1 : 1)
  ├── user_addresses      (1 : many)
  ├── user_sessions       (1 : many)
  └── user_locations      (1 : many)
```

---

## Core Tables

### 1. users

The anchor table. Every other table links back here.

| Column | Type | Notes |
|---|---|---|
| user_id | UUID | Primary key, non-guessable |
| full_name | VARCHAR(120) | Required |
| username | VARCHAR(60) | Unique, indexed |
| date_of_birth | DATE | Nullable; used for age-gating |
| created_at | TIMESTAMP | Default now() |
| updated_at | TIMESTAMP | Auto-updated on change |
| is_active | BOOLEAN | Soft-delete flag |

---

### 2. user_contacts

Contact data separated from identity so it can be updated independently and versioned.

| Column | Type | Notes |
|---|---|---|
| contact_id | UUID | Primary key |
| user_id | UUID | FK → users |
| email | VARCHAR(255) | Unique, indexed, lowercased |
| phone_number | VARCHAR(20) | E.164 format (+66812345678) |
| email_verified | BOOLEAN | Default false |
| phone_verified | BOOLEAN | Default false |

---

### 3. user_credentials

Completely isolated from identity. Only the auth service touches this table.

| Column | Type | Notes |
|---|---|---|
| credential_id | UUID | Primary key |
| user_id | UUID | FK → users |
| password_hash | VARCHAR(255) | bcrypt / argon2 hash only |
| last_password_changed | TIMESTAMP | For expiry policies |
| mfa_enabled | BOOLEAN | Two-factor flag |
| mfa_secret | VARCHAR(100) | Encrypted at rest |

---

### 4. user_addresses

Supports multiple addresses per user (home, billing, shipping).

| Column | Type | Notes |
|---|---|---|
| address_id | UUID | Primary key |
| user_id | UUID | FK → users |
| address_type | ENUM | 'home', 'billing', 'shipping' |
| street_line_1 | VARCHAR(200) | — |
| street_line_2 | VARCHAR(200) | Nullable |
| city | VARCHAR(100) | — |
| state_province | VARCHAR(100) | — |
| postal_code | VARCHAR(20) | — |
| country_code | CHAR(2) | ISO 3166-1 alpha-2 |
| is_primary | BOOLEAN | Default false |

---

### 5. user_sessions

Tracks active login sessions and device info.

| Column | Type | Notes |
|---|---|---|
| session_id | UUID | Primary key |
| user_id | UUID | FK → users |
| ip_address | INET | IPv4 or IPv6 |
| device_type | VARCHAR(50) | 'mobile', 'desktop', 'tablet' |
| device_os | VARCHAR(50) | 'iOS', 'Android', 'Windows' |
| browser | VARCHAR(100) | User-agent parsed |
| created_at | TIMESTAMP | Login time |
| expires_at | TIMESTAMP | TTL-based expiry |
| revoked | BOOLEAN | For forced logout |

---

### 6. user_locations

Stores coarse or precise location as runtime/GPS data — separate from postal address.

| Column | Type | Notes |
|---|---|---|
| location_id | UUID | Primary key |
| user_id | UUID | FK → users |
| latitude | DECIMAL(9,6) | — |
| longitude | DECIMAL(9,6) | — |
| accuracy_meters | INTEGER | Nullable |
| recorded_at | TIMESTAMP | When captured |
| source | ENUM | 'gps', 'ip', 'manual' |

---

## Key Design Decisions

**UUIDs over auto-increment integers** — prevents enumeration attacks (a bad actor cannot guess `user_id=1001` to probe records).

**Soft deletes on users** — `is_active = false` instead of hard deletes preserves audit trails while hiding the account. Hard deletes cascade when legally required (GDPR right to erasure).

**Contacts separated from users** — email and phone can change independently of identity, and this structure allows verification state to be tracked per-contact without polluting the users table.

**Credentials fully isolated** — only the authentication service holds permission to query `user_credentials`. No other service needs to touch it.

**Location as a log, not a state** — `user_locations` is append-only. The current location is simply the most recent row. This gives a history for fraud detection without requiring updates.

**No payment data** — payment functionality belongs in a dedicated payments service with its own isolated database. Keeping it out of this schema ensures this system stays fully out of PCI-DSS scope.

---

## Recommended Indexes

| Index | Table | Purpose |
|---|---|---|
| `(email)` UNIQUE | user_contacts | Used on every login |
| `(user_id)` | user_contacts | FK lookup |
| `(user_id, expires_at)` composite | user_sessions | Active session checks |
| `(ip_address)` | user_sessions | Fraud and abuse lookups |
| `(user_id, recorded_at DESC)` | user_locations | Latest location query |

---

## Security Layers

| Concern | Solution |
|---|---|
| Password storage | Argon2id hash, never plaintext |
| MFA secret | Encrypted column (AES-256) at rest |
| Session tokens | Hashed before storage; short TTL + revocation flag |
| Location data | Access restricted to fraud/logistics services only |
| GDPR erasure | ON DELETE CASCADE from users covers all child records |
| Enumeration attacks | UUID primary keys across all tables |

---

## Recommended Database Engine

**PostgreSQL** is the right choice here for the following reasons:

- The `INET` type handles IPv4 and IPv6 addresses natively with built-in operators
- `UUID` generation is built-in via `gen_random_uuid()`
- `JSONB` is available if device metadata becomes more complex over time
- Row-Level Security (RLS) can enforce table-level access restrictions directly in the database layer
- Streaming replication and logical replication are mature and well-supported
- Fully open source under the PostgreSQL License

---

## Normalization Summary

| Table | Normal Form | Notes |
|---|---|---|
| users | 3NF | No transitive dependencies |
| user_contacts | 3NF | Contact fields depend only on contact_id |
| user_credentials | 3NF | Auth fields fully isolated |
| user_addresses | 3NF | Each address row is atomic |
| user_sessions | 3NF | Session data depends only on session_id |
| user_locations | 3NF | Each location reading is independent |

---

## Foreign Key Cascade Rules

All child tables use `ON DELETE CASCADE` — deleting a user from the `users` table automatically removes all associated contacts, credentials, addresses, sessions, and location records. This satisfies GDPR Article 17 (right to erasure) at the database level without requiring application-layer orchestration.

---

*Designed for PostgreSQL 15+. Schema version 1.0.*