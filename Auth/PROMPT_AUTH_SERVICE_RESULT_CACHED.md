# Prompt — Build Auth Service (Laravel + Postgres, Repository-Service Architecture) CACHED

Copy everything below the `---` line into a fresh Claude Code / LLM session inside a Laravel 13 project that already has the **Profile service** built. The output should match the original implementation 1:1.

---

You are working in a Laravel 13 project with an existing **Profile service** (users, contacts, credentials, sessions tables already exist). PHP 8.3+, PostgreSQL 15+, `pdo_pgsql` available. Build an **Auth service** using **Repository–Service architecture** and **Laravel Sanctum** for token-based API authentication. Do not introduce extra abstractions, packages, or features beyond what is specified.

Sanctum is already installed (`laravel/sanctum ^4.0` in `composer.json`) and the `personal_access_tokens` migration already exists.

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

## Prerequisites

This service depends on the Profile service. The following must already exist:

- `users`, `user_contacts`, `user_credentials` tables and their models
- `UserRepositoryInterface` with `find()`, `create()`, `findWithProfile()`
- `UserContactRepositoryInterface` with `findByEmail()`, `upsertForUser()`
- `UserCredentialRepositoryInterface` with `findByUserId()`, `rotatePassword()`

## 1. Database schema

### `password_reset_tokens`

| col        | type      | notes                |
|------------|-----------|----------------------|
| email      | string    | PK                   |
| token      | string    | Hashed reset token   |
| created_at | timestamp | Nullable             |

## 2. Migration

```php
// database/migrations/2026_05_03_200000_create_password_reset_tokens_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('password_reset_tokens', function (Blueprint $table) {
            $table->string('email')->primary();
            $table->string('token');
            $table->timestamp('created_at')->nullable();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('password_reset_tokens');
    }
};
```

## 3. Model update — `app/Models/User.php`

The User model must be updated to extend `Authenticatable` instead of `Model`, and use `HasApiTokens` + `Notifiable` traits. Two helper methods are added for password reset email routing since email lives in `user_contacts`.

```php
// app/Models/User.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasUuids, HasApiTokens, Notifiable;
    protected $table = 'users';
    protected $primaryKey = 'user_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['user_id']; }
    protected $fillable = ['user_id','full_name','username','date_of_birth','is_active'];
    protected $casts = ['date_of_birth' => 'date', 'is_active' => 'boolean'];

    public function getEmailForPasswordReset(): string { return $this->contact?->email ?? ''; }
    public function routeNotificationForMail(): string { return $this->getEmailForPasswordReset(); }

    public function contact(): HasOne { return $this->hasOne(UserContact::class, 'user_id', 'user_id'); }
    public function credential(): HasOne { return $this->hasOne(UserCredential::class, 'user_id', 'user_id'); }
    public function addresses(): HasMany { return $this->hasMany(UserAddress::class, 'user_id', 'user_id'); }
    public function sessions(): HasMany { return $this->hasMany(UserSession::class, 'user_id', 'user_id'); }
    public function locations(): HasMany { return $this->hasMany(UserLocation::class, 'user_id', 'user_id'); }
}
```

**Key changes from Profile service version:**
- `extends Model` → `extends Authenticatable`
- Added `HasApiTokens`, `Notifiable` traits
- Added `getEmailForPasswordReset()` and `routeNotificationForMail()` (email lives in `user_contacts`, not on user directly)

## 4. Service — `app/Services/AuthService.php`

```php
<?php
namespace App\Services;

use App\Models\User;
use App\Repositories\Contracts\UserContactRepositoryInterface;
use App\Repositories\Contracts\UserCredentialRepositoryInterface;
use App\Repositories\Contracts\UserRepositoryInterface;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Carbon\Carbon;

class AuthService
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
        private readonly UserContactRepositoryInterface $contacts,
        private readonly UserCredentialRepositoryInterface $credentials,
    ) {}

    public function register(array $data): array
    {
        return DB::transaction(function () use ($data) {
            $user = $this->users->create([
                'full_name' => $data['full_name'],
                'username'  => $data['username'],
                'is_active' => true,
            ]);

            $this->contacts->upsertForUser($user->user_id, [
                'email' => strtolower($data['email']),
            ]);

            $this->credentials->rotatePassword($user->user_id, Hash::make($data['password']));

            $user = $this->users->findWithProfile($user->user_id);
            $token = $user->createToken('auth')->plainTextToken;

            return ['user' => $user, 'token' => $token];
        });
    }

    public function login(string $email, string $password): ?array
    {
        $contact = $this->contacts->findByEmail(strtolower($email));
        if (!$contact) return null;

        $user = $this->users->find($contact->user_id);
        if (!$user || !$user->is_active) return null;

        $credential = $this->credentials->findByUserId($user->user_id);
        if (!$credential || !Hash::check($password, $credential->password_hash)) return null;

        $token = $user->createToken('auth')->plainTextToken;

        return ['user' => $this->users->findWithProfile($user->user_id), 'token' => $token];
    }

    public function logout(User $user): bool
    {
        $user->currentAccessToken()->delete();
        return true;
    }

    public function forgotPassword(string $email): bool
    {
        $contact = $this->contacts->findByEmail(strtolower($email));
        if (!$contact) return false;

        $token = Str::random(64);

        DB::table('password_reset_tokens')->updateOrInsert(
            ['email' => strtolower($email)],
            ['token' => Hash::make($token), 'created_at' => Carbon::now()]
        );

        // In production, send email with the token.
        // For now, return true to indicate success.
        // The token would be sent via: Mail::to($email)->send(new ResetPasswordMail($token));

        return true;
    }

    public function resetPassword(string $email, string $token, string $newPassword): bool
    {
        $record = DB::table('password_reset_tokens')
            ->where('email', strtolower($email))
            ->first();

        if (!$record) return false;

        if (!Hash::check($token, $record->token)) return false;

        // Check if token is expired (60 minutes)
        if (Carbon::parse($record->created_at)->addMinutes(60)->isPast()) return false;

        $contact = $this->contacts->findByEmail(strtolower($email));
        if (!$contact) return false;

        $this->credentials->rotatePassword($contact->user_id, Hash::make($newPassword));

        DB::table('password_reset_tokens')->where('email', strtolower($email))->delete();

        // Revoke all existing Sanctum tokens
        $user = $this->users->find($contact->user_id);
        if ($user) $user->tokens()->delete();

        return true;
    }
}
```

**Design notes:**
- `forgotPassword` uses `DB::table()` directly for `password_reset_tokens` — this is a simple key-value table that doesn't warrant a full repository
- `resetPassword` verifies the hashed token, checks 60-minute expiry, rotates the password, deletes the token, and revokes all Sanctum tokens
- `login` looks up email in `user_contacts` then verifies password from `user_credentials` — matches the isolated credential pattern
- `register` wraps user + contact + credential creation in a transaction and returns a Sanctum token
- `logout` deletes only the current access token, not all tokens

## 5. HTTP layer

### Form requests — `app/Http/Requests/Auth/`

```php
// RegisterRequest.php
<?php
namespace App\Http\Requests\Auth;

use Illuminate\Foundation\Http\FormRequest;

class RegisterRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'full_name' => ['required', 'string', 'max:120'],
            'username'  => ['required', 'string', 'max:60', 'unique:users,username'],
            'email'     => ['required', 'email:rfc', 'unique:user_contacts,email'],
            'password'  => ['required', 'string', 'min:8', 'confirmed'],
        ];
    }
}
```

```php
// LoginRequest.php
<?php
namespace App\Http\Requests\Auth;

use Illuminate\Foundation\Http\FormRequest;

class LoginRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'email'    => ['required', 'email'],
            'password' => ['required', 'string'],
        ];
    }
}
```

```php
// ForgotPasswordRequest.php
<?php
namespace App\Http\Requests\Auth;

use Illuminate\Foundation\Http\FormRequest;

class ForgotPasswordRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'email' => ['required', 'email'],
        ];
    }
}
```

```php
// ResetPasswordRequest.php
<?php
namespace App\Http\Requests\Auth;

use Illuminate\Foundation\Http\FormRequest;

class ResetPasswordRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'email'    => ['required', 'email'],
            'token'    => ['required', 'string'],
            'password' => ['required', 'string', 'min:8', 'confirmed'],
        ];
    }
}
```

### Controller — `app/Http/Controllers/Api/AuthController.php`

```php
<?php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\ForgotPasswordRequest;
use App\Http\Requests\Auth\LoginRequest;
use App\Http\Requests\Auth\RegisterRequest;
use App\Http\Requests\Auth\ResetPasswordRequest;
use App\Http\Resources\Profile\ProfileResource;
use App\Services\AuthService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class AuthController extends Controller
{
    public function __construct(private readonly AuthService $service) {}

    public function register(RegisterRequest $request): JsonResponse
    {
        $result = $this->service->register($request->validated());
        return response()->json([
            'user'  => new ProfileResource($result['user']),
            'token' => $result['token'],
        ], 201);
    }

    public function login(LoginRequest $request): JsonResponse
    {
        $result = $this->service->login($request->input('email'), $request->input('password'));
        if (!$result) {
            return response()->json(['message' => 'Invalid credentials'], 401);
        }
        return response()->json([
            'user'  => new ProfileResource($result['user']),
            'token' => $result['token'],
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        $this->service->logout($request->user());
        return response()->json(['message' => 'Logged out']);
    }

    public function forgotPassword(ForgotPasswordRequest $request): JsonResponse
    {
        $this->service->forgotPassword($request->input('email'));
        // Always return success to prevent email enumeration
        return response()->json(['message' => 'If the email exists, a reset link has been sent']);
    }

    public function resetPassword(ResetPasswordRequest $request): JsonResponse
    {
        $reset = $this->service->resetPassword(
            $request->input('email'),
            $request->input('token'),
            $request->input('password'),
        );

        if (!$reset) {
            return response()->json(['message' => 'Invalid or expired reset token'], 422);
        }

        return response()->json(['message' => 'Password has been reset']);
    }
}
```

**Notes:**
- Reuses `ProfileResource` from the Profile service for user data in register/login responses
- `forgotPassword` always returns 200 regardless of whether the email exists — prevents enumeration attacks
- `login` returns 401 for invalid credentials
- `register` returns 201
- `logout` requires `auth:sanctum` middleware (applied on the route, not in the controller)

### Routes — `routes/api.php`

Add **before** the existing profile routes:

```php
use App\Http\Controllers\Api\AuthController;

Route::prefix('auth')->group(function () {
    Route::post('/register',        [AuthController::class, 'register']);
    Route::post('/login',           [AuthController::class, 'login']);
    Route::post('/forgot-password', [AuthController::class, 'forgotPassword']);
    Route::post('/reset-password',  [AuthController::class, 'resetPassword']);
    Route::post('/logout',          [AuthController::class, 'logout'])->middleware('auth:sanctum');
});
```

## 6. Verification

1. `php artisan migrate` — the `password_reset_tokens` migration should run clean.
2. `php artisan route:list --path=auth` — should show 5 routes.
3. Boot server (`php artisan serve`) and hit:

**Register:**
```
curl -X POST http://127.0.0.1:8000/api/auth/register \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"full_name":"Auth User","username":"auth_user1","email":"auth@example.com","password":"supersecret","password_confirmation":"supersecret"}'
```

Expect 201 with `{ user: ProfileResource, token: "1|abc..." }`.

**Login:**
```
curl -X POST http://127.0.0.1:8000/api/auth/login \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"email":"auth@example.com","password":"supersecret"}'
```

Expect 200 with `{ user: ProfileResource, token: "2|def..." }`.

**Logout (use token from login):**
```
curl -X POST http://127.0.0.1:8000/api/auth/logout \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer 2|def...'
```

Expect 200 with `{ message: "Logged out" }`.

**Forgot Password:**
```
curl -X POST http://127.0.0.1:8000/api/auth/forgot-password \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"email":"auth@example.com"}'
```

Expect 200 with `{ message: "If the email exists, a reset link has been sent" }`.

## Constraints

- Only Sanctum is used for token auth — already in `composer.json`, no new packages.
- No factories/seeders unless asked.
- `auth:sanctum` middleware only on `/logout` — all other auth routes are public.
- `password_reset_tokens` is accessed via `DB::table()` directly — no model/repository needed for a simple key-value table.
- Don't write multi-paragraph docblocks. One-line comments only when truly needed.
- Use `$this->model->newQuery()` inside repositories rather than holding stateful query builders.
- All write paths that touch multiple tables wrap in `DB::transaction`.

When done, run the smoke test in section 6 and report the JSON response.
