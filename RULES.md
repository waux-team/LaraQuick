# Prompt — LaraQuick Scaffold Rules (Laravel + Postgres, Repository-Service Architecture)

Copy everything below the `---` line into a fresh Claude Code / LLM session when building a new feature in this scaffold repo. The output must follow these rules exactly.

---

You are working in **LaraQuick** — a scaffold/template Laravel 13 repo. PHP 8.3+, PostgreSQL 15+, `pdo_pgsql` available. All features use **Repository–Service architecture**. Code is written here first, reviewed, then copied into the actual project repo. Do not introduce extra abstractions, packages, or features beyond what is specified.

Assume DB connection (`pgsql`) is already configured in `.env`. Set `SESSION_DRIVER=file` if not already, to avoid clashing with session tables.

## Style rules (apply everywhere)

- Single-line compact style for short methods: `public function foo(): Bar { return $this->bar; }`
- Properties on one line: `protected $fillable = ['id','col1','col2'];`
- `$casts` on one line: `protected $casts = ['flag' => 'boolean'];`
- No space after `!` in conditions: `if (!$record)`
- Controller property name: `$service`
- All controller methods return `JsonResponse`; wrap every return in `response()->json()`
- 404 pattern: `if (!$profile) return response()->json(['message' => 'Not found'], 404);`
- Service input param name: `$data`
- `use Illuminate\Support\Collection;` (not `Illuminate\Database\Eloquent\Collection`)
- Concrete repository constructors: `public function __construct(ModelClass $model) { parent::__construct($model); }`
- Repository methods open brace on same line: `public function foo(string $id) {`
- Use `$this->model->newQuery()` inside repositories — never hold stateful query builders

## Structure

Each feature = one folder named after the service + one cache file inside it.

```
Profile/
└── PROMPT_PROFILE_SERVICE_RESULT_CACHED.md

Order/
└── PROMPT_ORDER_SERVICE_RESULT_CACHED.md
```

## Dependency direction

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

> **Rule:** No layer may import from a layer above it. Controllers handle HTTP only and delegate everything to Services.

---

## 1. `app/Models/`

**Purpose:** Eloquent model classes — one file per database table.

### Rules
- Named **PascalCase singular** matching the table (e.g., `Profile.php` → `profiles` table)
- Define `$fillable`, `$casts`, relationships, and scopes here
- No business logic — thin data representations only
- Accessors/mutators allowed but keep them simple
- Namespace: `App\Models`
- Use `Profile` — **never** `User` — as the main account entity
- UUID PKs: use `HasUuids`, set `$primaryKey`, `public $incrementing = false`, `$keyType = 'string'`, and `uniqueIds()`

### Example

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Profile extends Model
{
    use HasUuids;
    protected $table = 'profiles';
    protected $primaryKey = 'profile_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['profile_id']; }
    protected $fillable = ['profile_id','display_name','bio','avatar_url','status'];
    protected $casts = ['is_active' => 'boolean'];

    public function orders(): HasMany { return $this->hasMany(Order::class, 'profile_id', 'profile_id'); }
}
```

### ❌ Do NOT
- Call services, repositories, or APIs from a model
- Put HTTP or request logic inside models

---

## 2. `app/Repos/`

**Purpose:** All Eloquent queries and database interaction — one interface + one implementation per model.

### Rules
- Folders: `app/Repos/Contracts` (interfaces), `app/Repos/Eloquent` (implementations)
- Only interacts with Models/Eloquent — no HTTP, no business rules
- Methods return Model instances or Collections, never raw query builders
- Register bindings in `RepositoryServiceProvider`
- Use `with()` eager loading here to prevent N+1
- All write paths touching multiple tables: wrap in `DB::transaction`

### Example

```php
// app/Repos/Contracts/ProfileRepositoryInterface.php
<?php
namespace App\Repos\Contracts;
interface ProfileRepositoryInterface extends BaseRepositoryInterface
{
    public function findBySlug(string $slug);
    public function findWithRelations(string $id);
    public function deactivate(string $id): bool;
}

// app/Repos/Eloquent/ProfileRepository.php
<?php
namespace App\Repos\Eloquent;
use App\Models\Profile;
use App\Repos\Contracts\ProfileRepositoryInterface;

class ProfileRepository extends BaseRepository implements ProfileRepositoryInterface
{
    public function __construct(Profile $model) { parent::__construct($model); }

    public function findBySlug(string $slug) {
        return $this->model->newQuery()->where('slug', $slug)->first();
    }

    public function findWithRelations(string $id) {
        return $this->model->newQuery()->with(['orders','contact'])->find($id);
    }

    public function deactivate(string $id): bool {
        return (bool) $this->update($id, ['is_active' => false]);
    }
}
```

### ❌ Do NOT
- Use `DB::` raw queries unless absolutely necessary
- Put validation or business rules inside a repository
- Return raw query builders to callers

---

## 3. `app/Services/`

**Purpose:** Business logic and orchestration — the heart of the application.

### Rules
- One service class per domain area (e.g., `ProfileService.php`, `OrderService.php`)
- Inject Repositories and API clients via constructor — **never** use Eloquent or `DB::` directly
- All validation logic, calculations, event dispatching, and notifications go here
- Services may call other Services — avoid circular dependencies
- Return plain values, DTOs, or Model instances — **never** responses or redirects
- Multi-table write paths wrap in `DB::transaction`

### Example

```php
<?php
namespace App\Services;
use App\Repos\Contracts\ProfileRepositoryInterface;
use App\Apis\TwilioApi;
use Illuminate\Database\Eloquent\ModelNotFoundException;

class ProfileService
{
    public function __construct(
        private readonly ProfileRepositoryInterface $profiles,
        private readonly TwilioApi $twilio,
    ) {}

    public function updateBio(string $profileId, array $data): Profile
    {
        $profile = $this->profiles->findById($profileId);
        if (!$profile) throw new ModelNotFoundException('Profile not found.');
        return $this->profiles->update($profileId, ['bio' => $data['bio']]);
    }
}
```

### ❌ Do NOT
- Call `Profile::find()` or any Eloquent methods directly inside a Service
- Return `Response` or `redirect()` from a Service
- Put HTTP request handling inside a Service

---

## 4. `app/Apis/`

**Purpose:** External API integration classes — one file per third-party service.

### Rules
- Handles all HTTP calls to outside services (payment gateways, SMS, REST APIs)
- Wraps raw `Http::` calls in clean, named methods
- Returns clean data — **never** raw `Response` objects — back to the Service layer
- No business logic — only transport and response mapping
- Named clearly after the service (e.g., `StripeApi.php`, `TwilioApi.php`)
- Namespace: `App\Apis`

### Example

```php
<?php
namespace App\Apis;
use Illuminate\Support\Facades\Http;

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

## 5. `database/Migrations/`

**Purpose:** Laravel migration files for schema versioning — auto-managed by Artisan.

### Rules
- **Never hand-edit** a migration already run in production
- Generate via `php artisan make:migration <name>`
- Every migration must implement both `up()` and `down()` correctly
- Migration names: Laravel timestamp convention in descriptive `snake_case`
- Seed data goes in `database/seeders/`, **not** in migrations
- UUID PKs: `$table->uuid('profile_id')->primary()`; FKs: `->constrained('profiles','profile_id')->onDelete('cascade')`

### Naming convention

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
        $table->uuid('profile_id')->primary();
        $table->string('display_name', 120);
        $table->text('bio')->nullable();
        $table->string('avatar_url')->nullable();
        $table->boolean('is_active')->default(true);
        $table->timestamps();
    });
}

public function down(): void
{
    Schema::dropIfExists('profiles');
}
```

### ❌ Do NOT
- Hand-edit a migration after it has run in production
- Put seed data inside a migration

---

## 6. HTTP layer

### Form Requests — `app/Http/Requests/{Feature}/`

- One `Store{Feature}Request` + one `Update{Feature}Request` per resource
- `authorize()` returns `true`
- `Store*`: use `required` / `nullable` rules
- `Update*`: use `sometimes` on all fields; unique rules ignore current record via `Rule::unique(...)->ignore($this->route('id'), 'pk_col')`

### Resources — `app/Http/Resources/{Feature}/`

- One resource class per model; map all public columns including PK and FK
- `ProfileResource` (or equivalent) nests relations using `whenLoaded()`
- Cast date fields: `$this->date_of_birth?->toDateString()`

### Controller — `app/Http/Controllers/Api/{Feature}Controller.php`

- Inject Service via constructor as `$service`
- Every method returns `JsonResponse`
- 404 check: `if (!$record) return response()->json(['message' => 'Not found'], 404);`
- `store()` returns 201; all others return 200

### Routes — `routes/api.php`

- Run `php artisan install:api` first
- Group under `Route::prefix('{feature}s')->group(...)`
- RESTful: `get /`, `post /`, `get /{id}`, `patch /{id}`, `delete /{id}`, plus sub-resource routes

---

## 7. Service provider

Register repository bindings in `app/Providers/RepositoryServiceProvider.php` using a `$repositoryBindings` array. Register the provider in `bootstrap/providers.php`.

```php
<?php
namespace App\Providers;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    private array $repositoryBindings = [
        ProfileRepositoryInterface::class => ProfileRepository::class,
    ];

    public function register(): void
    {
        foreach ($this->repositoryBindings as $interface => $implementation) {
            $this->app->bind($interface, $implementation);
        }
    }
}
```

---

## 8. Verification checklist

After building a feature:

1. `php artisan migrate` — all new migrations run clean
2. `php artisan route:list --path={feature}` — expected routes present
3. Boot server (`php artisan serve`) and smoke-test each endpoint (create → read → update → delete)
4. Confirm nested relations load in GET response
5. Confirm 404 returns `{"message":"Not found"}` for missing IDs

---

## Constraints

- No extra packages (no spatie, no laravel-data, etc.)
- No factories/seeders unless explicitly asked
- No auth middleware on routes unless specified
- Don't write multi-paragraph docblocks — one-line comments only when truly needed
- No layer imports from a layer above it
- `$this->model->newQuery()` in repositories — never stateful query builders
- All multi-table write paths wrap in `DB::transaction`
- UUID primary keys on all tables; FKs with `ON DELETE CASCADE`

---

## Quick reference

| Layer                | File Example                        | Namespace                        | Depends On  |
|----------------------|-------------------------------------|----------------------------------|-------------|
| Controller           | `ProfileController.php`             | `App\Http\Controllers\Api`       | Services    |
| Service                  | `ProfileService.php`                | `App\Services`                   | Repos + Apis     |
| Repository Interface     | `ProfileRepositoryInterface.php`    | `App\Repos\Contracts`            | —                |
| Repository Impl          | `ProfileRepository.php`             | `App\Repos\Eloquent`             | Models           |
| Api Client               | `StripeApi.php`                     | `App\Apis`                       | —                |
| Model                    | `Profile.php`                       | `App\Models`                     | —                |
| Migration                | `..._create_profiles_table.php`     | —                                | — (Artisan only) |
