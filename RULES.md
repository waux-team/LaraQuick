# 📁 Laravel Project Folder Rules

## Overview

This project follows a **Repository + Service pattern** in Laravel.
All layers have a single, well-defined responsibility.

This is a **scaffold/template repo** — code is written here first, then copied into the actual project repository.

---

## Folder Structure

```
app/
├── Apis/          → External API clients (third-party integrations)
├── Models/        → Eloquent models (data representation)
├── Repos/         → Database queries (data access layer)
├── Services/      → Business logic (orchestration layer)
database/
└── Migrations/    → Schema versioning (Artisan-managed)

QuickStartCommands.txt  → Artisan commands to bootstrap new features
```

---

## `QuickStartCommands.txt`

**Purpose:** A reference list of `php artisan` commands to quickly scaffold models, migrations, repos, services, and API clients for new features.

### Rules
- Always add new commands here before running them, so the team has a record
- Group commands by feature/domain (e.g., `# Profile`, `# Order`)
- After running commands, move/edit generated files to match the correct folder structure
- Once code is finalized and reviewed, copy it to the **actual project repo**
- Do **not** copy unreviewed boilerplate — only working, tested code gets copied across

### Example Format

```
# Profile
php artisan make:model Profile -m
php artisan make:migration create_profiles_table

# Order
php artisan make:model Order -m
php artisan make:migration create_orders_table
```

### Copy-to-Repo Checklist
- [ ] Code reviewed and tested locally
- [ ] Migration runs cleanly (`php artisan migrate`)
- [ ] All `down()` methods implemented
- [ ] No hardcoded credentials or `.env` values in code
- [ ] Files copied to actual project repo in the correct folder structure

---

## `Apis/`

**Purpose:** External API integration classes — one file per third-party service.

### Rules
- Handles all HTTP calls to outside services (e.g., payment gateways, SMS, REST APIs)
- Wraps raw HTTP calls (e.g., Laravel `Http::`) in clean, named methods
- Returns clean data — **never** raw `Response` objects — back to the Service layer
- No business logic here — only transport and response mapping
- Named clearly after the service (e.g., `StripeApi.php`, `TwilioApi.php`)
- Belongs in namespace `App\Apis`

### Example

```php
// ✅ Good
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

### ❌ Do NOT
- Put business rules or validation inside an API class
- Return raw `Illuminate\Http\Client\Response` to callers
- Call API classes directly from Controllers — always go through a Service

---

## `Models/`

**Purpose:** Eloquent model classes only — one file per database table.

### Rules
- Named in **PascalCase singular** matching the table (e.g., `Profile.php` → `profiles` table)
- Define `$fillable` or `$guarded`, casts, relationships, and scopes here
- No business logic — keep models as thin data representations
- Accessors/mutators are allowed but keep them simple
- Belongs in namespace `App\Models`
- Use `Profile` — **never** `User` — as the main account entity

### Example

```php
// ✅ Good
class Profile extends Model
{
    protected $fillable = ['display_name', 'bio', 'avatar_url', 'status'];

    public function orders(): HasMany
    {
        return $this->hasMany(Order::class);
    }
}
```

### ❌ Do NOT
- Call services, repositories, or APIs from a model
- Put HTTP or request logic inside models

---

## `Repos/`

**Purpose:** All Eloquent queries and database interaction — keeps queries out of Services and Controllers.

### Rules
- One **interface** + one **implementation** per model
  - e.g., `ProfileRepositoryInterface.php` + `ProfileRepository.php`
- Only interacts with Models/Eloquent — no HTTP, no business rules
- Methods return Model instances or Collections, never raw query builders
- Register interface bindings in `AppServiceProvider` or a dedicated `RepositoryServiceProvider`
- Use `with()` eager loading here to prevent N+1 issues

### Example

```php
// Interface
interface ProfileRepositoryInterface
{
    public function findById(int $id): ?Profile;
    public function findActiveProfiles(): Collection;
}

// Implementation
class ProfileRepository implements ProfileRepositoryInterface
{
    public function findById(int $id): ?Profile
    {
        return Profile::with('orders')->find($id);
    }

    public function findActiveProfiles(): Collection
    {
        return Profile::where('status', 'active')->get();
    }
}
```

### ❌ Do NOT
- Use `DB::` raw queries unless absolutely necessary
- Put validation or business rules inside a repository
- Return raw query builders to callers

---

## `Services/`

**Purpose:** Business logic and orchestration — the heart of the application.

### Rules
- One service class per domain area (e.g., `ProfileService.php`, `OrderService.php`)
- Inject Repositories and API clients via constructor — **never** use Eloquent or `DB::` directly
- All validation logic, calculations, event dispatching, and notifications go here
- Services may call other Services, but avoid circular dependencies
- Return plain values, DTOs, or Model instances — **never** responses or redirects

### Example

```php
class ProfileService
{
    public function __construct(
        private ProfileRepositoryInterface $profiles,
        private TwilioApi $twilio
    ) {}

    public function updateBio(int $profileId, string $bio): Profile
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

### ❌ Do NOT
- Call `Profile::find()` or any Eloquent methods directly inside a Service
- Return `Response` or `redirect()` from a Service
- Put HTTP request handling inside a Service

---

## `Migrations/`

**Purpose:** Laravel migration files for schema versioning — auto-managed by Artisan.

### Rules
- **Never hand-edit** a migration that has already been run in production
- Generate via `QuickStartCommands.txt` using `php artisan make:migration <name>`
- Every migration must implement both `up()` and `down()` correctly
- Migration names follow Laravel's timestamp convention in descriptive `snake_case`
- Seed data goes in `database/seeders/`, **not** in migrations

### Naming Convention

```
✅ Good
2024_04_25_120000_create_profiles_table.php
2024_04_26_083000_add_avatar_url_to_profiles_table.php

❌ Bad
2024_04_25_120000_update.php
2024_04_25_120000_fix.php
```

### Example

```php
public function up(): void
{
    Schema::create('profiles', function (Blueprint $table) {
        $table->id();
        $table->string('display_name');
        $table->text('bio')->nullable();
        $table->string('avatar_url')->nullable();
        $table->string('status')->default('active');
        $table->timestamps();
    });
}

public function down(): void
{
    Schema::dropIfExists('profiles');
}
```

---

## Dependency Direction

```
Controllers
    │
    ▼
Services          ← Business logic & orchestration
  │     │
  ▼     ▼
Repos  Apis       ← Data access / External HTTP
  │
  ▼
Models            ← Data structure only
  │
  ▼
Migrations        ← Schema only (Artisan-managed)
```

> **Rule:** No layer may import from a layer above it.
> Controllers handle HTTP only and delegate everything to Services.

---

## Quick Reference

| Layer                    | File Example                       | Depends On         |
|--------------------------|------------------------------------|--------------------|
| `QuickStartCommands.txt` | —                                  | — (scaffold ref)   |
| Controller               | `ProfileController.php`            | Services           |
| Service                  | `ProfileService.php`               | Repos + Apis       |
| Repository               | `ProfileRepository.php`            | Models             |
| Api Client               | `StripeApi.php`                    | —                  |
| Model                    | `Profile.php`                      | —                  |
| Migration                | `..._create_profiles_table.php`    | — (Artisan only)   |