# 📁 Laravel Project Folder Rules

## Overview

This project follows a **Repository + Service pattern** in Laravel.
All layers have a single, well-defined responsibility.

---

## Folder Structure

```
app/
├── Models/        → Eloquent models (data representation)
├── Repos/         → Database queries (data access layer)
├── Services/      → Business logic (orchestration layer)
database/
└── Migrations/    → Schema versioning (Artisan-managed)
```

---

## `Models/`

**Purpose:** Eloquent model classes only — one file per database table.

### Rules
- Named in **PascalCase singular** matching the table (e.g., `Profile.php` → `profiles` table)
- Define `$fillable` or `$guarded`, casts, relationships, and scopes here
- No business logic — keep models as thin data representations
- Accessors/mutators are allowed but keep them simple
- Belongs in namespace `App\Models`

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
- Call services or repositories from a model
- Put HTTP or request logic inside models
- Use `User` — always use `Profile` for the main account entity

---

## `Repos/`

**Purpose:** All Eloquent queries and database interaction — keeps queries out of Services and Controllers.

### Rules
- One **interface** + one **implementation** per model
  - e.g., `ProfileRepositoryInterface.php` + `ProfileRepository.php`
- Only interacts with Models/Eloquent — no HTTP, no business rules
- Methods return Model instances or collections, never raw query builders to callers
- Register bindings in `AppServiceProvider` or a dedicated `RepositoryServiceProvider`
- Use `with()` eager loading here to prevent N+1 issues
- All methods should be `async`-friendly — prefer returning `Collection` or `Model`

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
- Return raw query builders (`->query()`) to callers

---

## `Services/`

**Purpose:** Business logic and orchestration — the heart of the application.

### Rules
- One service class per domain area (e.g., `ProfileService.php`, `OrderService.php`)
- Inject Repositories via constructor — **never** use Eloquent or `DB::` directly
- All validation logic, calculations, event dispatching, and notifications go here
- Services may call other Services, but avoid circular dependencies
- Return plain values, DTOs, or Model instances — **never** responses or redirects

### Example

```php
class ProfileService
{
    public function __construct(
        private ProfileRepositoryInterface $profiles
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
- Generate with `php artisan make:migration <name>` — always use descriptive `snake_case` names
- Every migration must implement both `up()` and `down()` correctly
- Migration names follow Laravel's timestamp convention
- Seed data goes in `database/seeders/`, **not** in migrations

### Naming Convention

```
✅ Good
2024_04_25_120000_create_profiles_table.php
2024_04_26_083000_add_avatar_url_to_profiles_table.php
2024_04_27_140000_add_status_index_to_profiles_table.php

❌ Bad
2024_04_25_120000_update.php
2024_04_25_120000_fix_profiles.php
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
Services          ← Business logic lives here
    │
    ▼
Repos             ← All DB queries live here
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

| Layer      | File Example                    | Depends On         |
|------------|---------------------------------|--------------------|
| Controller | `ProfileController.php`         | Services           |
| Service    | `ProfileService.php`            | Repos (interface)  |
| Repository | `ProfileRepository.php`         | Models             |
| Model      | `Profile.php`                   | —                  |
| Migration  | `..._create_profiles_table.php` | — (Artisan only)   |