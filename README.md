# LaraQuick

Laravel scaffold/template repo for building modular, maintainable applications using the **Repository + Service pattern**. Code is written and reviewed here first, then copied into the actual project repository.

## Architecture

```
Controllers  →  Services  →  Repos  →  Models  →  Migrations
                         ↘  Apis
```

No layer imports from a layer above it. Controllers handle HTTP only.

### Folder Structure

```
Profile/                     ← Feature module (one per domain)
├── Apis/                    → External API clients (third-party integrations)
├── Migrations/              → Schema versioning (Artisan-managed)
├── Models/                  → Eloquent models (data representation only)
├── Repos/                   → Database queries (data access layer)
├── Services/                → Business logic (orchestration layer)
└── QuickStartCommands.txt   → Artisan commands to scaffold this feature
```

Directories containing only `DELETE_IF_FILE_EXISTS.txt` are empty placeholders — delete that file when adding real code.

See [RULES.md](RULES.md) for full conventions per layer.

## Scaffold Workflow

Each module has a `QuickStartCommands.txt` — a step-by-step guide to generate, move, implement, and ship code for that feature.

**Step 1 — Generate skeletons**
```bash
php artisan make:controller ProfileController
php artisan make:model Profile
php artisan make:migration create_profiles_table
```

**Step 2 — Move generated files** into the correct module subfolders (e.g. `Profile/Models/`, `Profile/Repos/`)

**Step 3 — Implement** each layer following [RULES.md](RULES.md)

**Step 4 — Run migrations**
```bash
php artisan migrate
php artisan db:seed --class=ProfileSeeder   # optional
```

**Step 5 — Copy to project repo** (reviewed, working code only)

### Copy-to-Repo Checklist

- [ ] Code reviewed and tested locally
- [ ] Migration runs cleanly (`php artisan migrate`)
- [ ] All `down()` methods implemented
- [ ] No hardcoded credentials or `.env` values in code
- [ ] Files placed in correct folder structure in target repo

## Key Features

- **Repository + Service pattern** — enforced dependency direction
- **Apis layer** — external HTTP clients isolated from business logic
- **Normalized schema (3NF)** — eliminates redundancy, keeps updates clean
- **Security-first DB design** — sensitive data isolated; principle of least privilege
- **UUID primary keys** — prevents enumeration attacks across all tables
- **PostgreSQL-optimized** — leverages `INET`, `UUID`, JSONB, and Row-Level Security

## Layer Quick Reference

| Layer      | Example file                    | Depends on        |
|------------|---------------------------------|-------------------|
| Controller | `ProfileController.php`         | Services          |
| Service    | `ProfileService.php`            | Repos + Apis      |
| Repository | `ProfileRepository.php`         | Models            |
| Api Client | `StripeApi.php`                 | —                 |
| Model      | `Profile.php`                   | —                 |
| Migration  | `..._create_profiles_table.php` | — (Artisan only)  |

## Code Examples

### Api Client

```php
// Profile/Apis/StripeApi.php
class StripeApi
{
    public function charge(int $amount, string $token): array
    {
        return Http::withToken(config('services.stripe.key'))
            ->post('https://api.stripe.com/v1/charges', [
                'amount' => $amount,
                'source' => $token,
            ])->json();
    }
}
```

### Repository

```php
// Bind in AppServiceProvider
$this->app->bind(ProfileRepositoryInterface::class, ProfileRepository::class);

// Profile/Repos/ProfileRepository.php
class ProfileRepository implements ProfileRepositoryInterface
{
    public function findById(string $id): ?Profile
    {
        return Profile::with('orders')->find($id);
    }
}
```

### Service

```php
// Profile/Services/ProfileService.php
class ProfileService
{
    public function __construct(
        private ProfileRepositoryInterface $profiles,
        private StripeApi $stripe
    ) {}

    public function updateBio(string $profileId, string $bio): Profile
    {
        $profile = $this->profiles->findById($profileId);

        if (!$profile) {
            throw new ModelNotFoundException("Profile not found.");
        }

        $profile->update(['bio' => $bio]);

        return $profile;
    }
}
```

### Migration naming

```
✅ 2024_04_25_120000_create_profiles_table.php
✅ 2024_04_26_083000_add_avatar_url_to_profiles_table.php
❌ 2024_04_25_120000_update.php
❌ 2024_04_25_120000_fix.php
```

## Database Design

User module ships with a fully normalized schema. See [Docs/User.md](Docs/User.md) for full column specs, indexes, and cascade rules.

```
users
  ├── user_contacts       (1:1)    — email, phone, verification state
  ├── user_credentials    (1:1)    — password hash, MFA (auth service only)
  ├── user_addresses      (1:many) — home, billing, shipping
  ├── user_sessions       (1:many) — device, IP, TTL, revocation
  └── user_locations      (1:many) — GPS/IP location log (append-only)
```

**Key decisions:**
- Credentials fully isolated — only the auth service queries `user_credentials`
- Soft deletes via `is_active` flag; hard delete cascades for GDPR erasure
- Location is a log, not state — most recent row = current location
- No payment data — keeps schema out of PCI-DSS scope

## Requirements

- PHP 8.2+
- Laravel 11+
- PostgreSQL 15+

## Installation

```bash
composer require laraquick/laraquick
php artisan vendor:publish --tag=laraquick
php artisan migrate
```
