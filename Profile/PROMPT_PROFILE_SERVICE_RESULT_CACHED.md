# Prompt — Build Profile Service (Laravel + Postgres, Repository-Service Architecture) CACHED

Copy everything below the `---` line into a fresh Claude Code / LLM session inside an empty Laravel 13 project (created with `composer create-project laravel/laravel .`). The output should match the original implementation 1:1.

---

You are working in a freshly scaffolded Laravel 13 project. PHP 8.3+, PostgreSQL 15+, `pdo_pgsql` available. Build a **Profile service** using **Repository–Service architecture**. Do not introduce extra abstractions, packages, or features beyond what is specified.

Assume DB connection (`pgsql`) is already configured in `.env`. Set `SESSION_DRIVER=file` if not already, to avoid clashing with the `user_sessions` table.

## Style rules (apply everywhere)

- Single-line compact style for short methods: `public function foo(): Bar { return $this->bar; }`
- Properties on one line: `protected $fillable = ['id','col1','col2'];`
- `$casts` on one line: `protected $casts = ['flag' => 'boolean'];`
- No space after `!` in conditions: `if (!$record)`
- No trailing space before `{` on single-line methods
- Controller property name: `$service`
- All controller methods return `JsonResponse`; wrap every return in `response()->json()`
- 404 pattern: `if (!$profile) return response()->json(['message' => 'Not found'], 404);`
- Service input param name: `$data`
- `use Illuminate\Support\Collection;` (not `Illuminate\Database\Eloquent\Collection`)
- Addresses loop: `foreach ($data['addresses'] ?? [] as $addr)`
- Concrete repository constructors: `public function __construct(ModelClass $model) { parent::__construct($model); }`
- Repository methods open brace on same line: `public function foo(string $id) {`

## 1. Database schema

Replace the default `create_users_table` migration. Create six tables. UUID primary keys, FK `ON DELETE CASCADE`, all child columns (except FK + PK) nullable as listed.

### `users`

| col           | type        | notes        |
| ------------- | ----------- | ------------ |
| user_id       | UUID        | PK           |
| full_name     | string(120) | required     |
| username      | string(60)  | unique       |
| date_of_birth | date        | nullable     |
| is_active     | boolean     | default true |
| timestamps    |             |              |

### `user_contacts` (1:1)

contact_id UUID PK, user_id UUID FK→users.user_id cascade, email string(255) unique nullable, phone_number string(20) nullable, email_verified bool default false nullable, phone_verified bool default false nullable, timestamps. Index user_id.

### `user_credentials` (1:1)

credential_id UUID PK, user_id UUID FK cascade, password_hash string(255) nullable, last_password_changed timestamp nullable, mfa_enabled bool default false nullable, mfa_secret string(100) nullable, timestamps. Index user_id.

### `user_addresses` (1:many)

address_id UUID PK, user_id UUID FK cascade, address_type enum('home','billing','shipping') nullable, street_line_1 string(200), street_line_2 string(200), city string(100), state_province string(100), postal_code string(20), country_code char(2), is_primary bool default false nullable, timestamps. Index user_id.

### `user_sessions` (1:many)

session_id UUID PK, user_id UUID FK cascade, ip_address `ipAddress` nullable, device_type string(50), device_os string(50), browser string(100), expires_at timestamp, revoked bool default false nullable, timestamps. Indexes: (user_id, expires_at), ip_address.

### `user_locations` (1:many)

location_id UUID PK, user_id UUID FK cascade, latitude decimal(9,6), longitude decimal(9,6), accuracy_meters integer, recorded_at timestamp, source enum('gps','ip','manual') nullable, timestamps. Index (user_id, recorded_at).

## 2. Models — `app/Models`

Write exactly these six files:

```php
// app/Models/User.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasOne;

class User extends Model
{
    use HasUuids;
    protected $table = 'users';
    protected $primaryKey = 'user_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['user_id']; }
    protected $fillable = ['user_id','full_name','username','date_of_birth','is_active'];
    protected $casts = ['date_of_birth' => 'date', 'is_active' => 'boolean'];

    public function contact(): HasOne { return $this->hasOne(UserContact::class, 'user_id', 'user_id'); }
    public function credential(): HasOne { return $this->hasOne(UserCredential::class, 'user_id', 'user_id'); }
    public function addresses(): HasMany { return $this->hasMany(UserAddress::class, 'user_id', 'user_id'); }
    public function sessions(): HasMany { return $this->hasMany(UserSession::class, 'user_id', 'user_id'); }
    public function locations(): HasMany { return $this->hasMany(UserLocation::class, 'user_id', 'user_id'); }
}
```

```php
// app/Models/UserContact.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UserContact extends Model
{
    use HasUuids;
    protected $table = 'user_contacts';
    protected $primaryKey = 'contact_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['contact_id']; }
    protected $fillable = ['contact_id','user_id','email','phone_number','email_verified','phone_verified'];
    protected $casts = ['email_verified' => 'boolean', 'phone_verified' => 'boolean'];

    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// app/Models/UserCredential.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UserCredential extends Model
{
    use HasUuids;
    protected $table = 'user_credentials';
    protected $primaryKey = 'credential_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['credential_id']; }
    protected $fillable = ['credential_id','user_id','password_hash','last_password_changed','mfa_enabled','mfa_secret'];
    protected $hidden = ['password_hash','mfa_secret'];
    protected $casts = ['mfa_enabled' => 'boolean', 'last_password_changed' => 'datetime'];

    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// app/Models/UserAddress.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UserAddress extends Model
{
    use HasUuids;
    protected $table = 'user_addresses';
    protected $primaryKey = 'address_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['address_id']; }
    protected $fillable = ['address_id','user_id','address_type','street_line_1','street_line_2','city','state_province','postal_code','country_code','is_primary'];
    protected $casts = ['is_primary' => 'boolean'];

    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// app/Models/UserSession.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UserSession extends Model
{
    use HasUuids;
    protected $table = 'user_sessions';
    protected $primaryKey = 'session_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['session_id']; }
    protected $fillable = ['session_id','user_id','ip_address','device_type','device_os','browser','expires_at','revoked'];
    protected $casts = ['revoked' => 'boolean', 'expires_at' => 'datetime'];

    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// app/Models/UserLocation.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UserLocation extends Model
{
    use HasUuids;
    protected $table = 'user_locations';
    protected $primaryKey = 'location_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['location_id']; }
    protected $fillable = ['location_id','user_id','latitude','longitude','accuracy_meters','recorded_at','source'];
    protected $casts = ['latitude' => 'decimal:6', 'longitude' => 'decimal:6', 'recorded_at' => 'datetime'];

    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

## 3. Repositories

Folders: `app/Repositories/Contracts`, `app/Repositories/Eloquent`.

### Interfaces (in Contracts)

```php
// app/Repositories/Contracts/BaseRepositoryInterface.php
<?php
namespace App\Repositories\Contracts;

interface BaseRepositoryInterface
{
    public function all();
    public function find(string $id);
    public function findOrFail(string $id);
    public function create(array $data);
    public function update(string $id, array $data);
    public function delete(string $id): bool;
}
```

Per-entity interfaces — each extends `BaseRepositoryInterface`, single-line method signatures:

- `UserRepositoryInterface`: `findByUsername(string $username)`, `findWithProfile(string $id)`, `deactivate(string $id): bool`
- `UserContactRepositoryInterface`: `findByUserId(string $userId)`, `findByEmail(string $email)`, `upsertForUser(string $userId, array $data)`
- `UserCredentialRepositoryInterface`: `findByUserId(string $userId)`, `upsertForUser(string $userId, array $data)`, `rotatePassword(string $userId, string $passwordHash)`
- `UserAddressRepositoryInterface`: `listByUser(string $userId)`, `setPrimary(string $userId, string $addressId): bool`
- `UserSessionRepositoryInterface`: `activeForUser(string $userId)`, `revoke(string $sessionId): bool`, `revokeAllForUser(string $userId): int`
- `UserLocationRepositoryInterface`: `latestForUser(string $userId)`, `historyForUser(string $userId, int $limit = 50)`

### Eloquent implementations

```php
// app/Repositories/Eloquent/BaseRepository.php
<?php
namespace App\Repositories\Eloquent;

use App\Repositories\Contracts\BaseRepositoryInterface;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\ModelNotFoundException;

abstract class BaseRepository implements BaseRepositoryInterface
{
    public function __construct(protected Model $model) {}

    public function all() { return $this->model->newQuery()->get(); }
    public function find(string $id) { return $this->model->newQuery()->find($id); }
    public function findOrFail(string $id) { return $this->model->newQuery()->findOrFail($id); }
    public function create(array $data) { return $this->model->newQuery()->create($data); }

    public function update(string $id, array $data)
    {
        $record = $this->findOrFail($id);
        $record->fill($data);
        $record->save();
        return $record->refresh();
    }

    public function delete(string $id): bool
    {
        $record = $this->find($id);
        if (!$record) return false;
        return (bool) $record->delete();
    }
}
```

```php
// app/Repositories/Eloquent/UserRepository.php
<?php
namespace App\Repositories\Eloquent;

use App\Models\User;
use App\Repositories\Contracts\UserRepositoryInterface;

class UserRepository extends BaseRepository implements UserRepositoryInterface
{
    public function __construct(User $model) { parent::__construct($model); }

    public function findByUsername(string $username) {
        return $this->model->newQuery()->where('username', $username)->first();
    }

    public function findWithProfile(string $id) {
        return $this->model->newQuery()
            ->with(['contact','credential','addresses','sessions','locations'])
            ->find($id);
    }

    public function deactivate(string $id): bool {
        return (bool) $this->update($id, ['is_active' => false]);
    }
}
```

```php
// app/Repositories/Eloquent/UserContactRepository.php
<?php
namespace App\Repositories\Eloquent;

use App\Models\UserContact;
use App\Repositories\Contracts\UserContactRepositoryInterface;

class UserContactRepository extends BaseRepository implements UserContactRepositoryInterface
{
    public function __construct(UserContact $model) { parent::__construct($model); }

    public function findByUserId(string $userId) {
        return $this->model->newQuery()->where('user_id', $userId)->first();
    }

    public function findByEmail(string $email) {
        return $this->model->newQuery()->where('email', $email)->first();
    }

    public function upsertForUser(string $userId, array $data) {
        if (isset($data['email'])) $data['email'] = strtolower($data['email']);
        $contact = $this->findByUserId($userId);
        if (!$contact) return $this->create(array_merge($data, ['user_id' => $userId]));
        $contact->fill($data);
        $contact->save();
        return $contact;
    }
}
```

```php
// app/Repositories/Eloquent/UserCredentialRepository.php
<?php
namespace App\Repositories\Eloquent;

use App\Models\UserCredential;
use App\Repositories\Contracts\UserCredentialRepositoryInterface;
use Carbon\Carbon;

class UserCredentialRepository extends BaseRepository implements UserCredentialRepositoryInterface
{
    public function __construct(UserCredential $model) { parent::__construct($model); }

    public function findByUserId(string $userId) {
        return $this->model->newQuery()->where('user_id', $userId)->first();
    }

    public function upsertForUser(string $userId, array $data) {
        $cred = $this->findByUserId($userId);
        if (!$cred) return $this->create(array_merge($data, ['user_id' => $userId]));
        $cred->fill($data);
        $cred->save();
        return $cred;
    }

    public function rotatePassword(string $userId, string $passwordHash) {
        return $this->upsertForUser($userId, [
            'password_hash' => $passwordHash,
            'last_password_changed' => Carbon::now(),
        ]);
    }
}
```

```php
// app/Repositories/Eloquent/UserAddressRepository.php
<?php
namespace App\Repositories\Eloquent;

use App\Models\UserAddress;
use App\Repositories\Contracts\UserAddressRepositoryInterface;
use Illuminate\Support\Facades\DB;

class UserAddressRepository extends BaseRepository implements UserAddressRepositoryInterface
{
    public function __construct(UserAddress $model) { parent::__construct($model); }

    public function listByUser(string $userId) {
        return $this->model->newQuery()->where('user_id', $userId)->get();
    }

    public function setPrimary(string $userId, string $addressId): bool {
        return DB::transaction(function () use ($userId, $addressId) {
            $this->model->newQuery()->where('user_id', $userId)->update(['is_primary' => false]);
            return (bool) $this->model->newQuery()
                ->where('address_id', $addressId)
                ->where('user_id', $userId)
                ->update(['is_primary' => true]);
        });
    }
}
```

```php
// app/Repositories/Eloquent/UserSessionRepository.php
<?php
namespace App\Repositories\Eloquent;

use App\Models\UserSession;
use App\Repositories\Contracts\UserSessionRepositoryInterface;
use Carbon\Carbon;

class UserSessionRepository extends BaseRepository implements UserSessionRepositoryInterface
{
    public function __construct(UserSession $model) { parent::__construct($model); }

    public function activeForUser(string $userId) {
        return $this->model->newQuery()
            ->where('user_id', $userId)
            ->where('revoked', false)
            ->where(fn($q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', Carbon::now()))
            ->get();
    }

    public function revoke(string $sessionId): bool {
        return (bool) $this->model->newQuery()
            ->where('session_id', $sessionId)
            ->update(['revoked' => true]);
    }

    public function revokeAllForUser(string $userId): int {
        return $this->model->newQuery()
            ->where('user_id', $userId)
            ->update(['revoked' => true]);
    }
}
```

```php
// app/Repositories/Eloquent/UserLocationRepository.php
<?php
namespace App\Repositories\Eloquent;

use App\Models\UserLocation;
use App\Repositories\Contracts\UserLocationRepositoryInterface;

class UserLocationRepository extends BaseRepository implements UserLocationRepositoryInterface
{
    public function __construct(UserLocation $model) { parent::__construct($model); }

    public function latestForUser(string $userId) {
        return $this->model->newQuery()
            ->where('user_id', $userId)
            ->orderBy('recorded_at', 'desc')
            ->first();
    }

    public function historyForUser(string $userId, int $limit = 50) {
        return $this->model->newQuery()
            ->where('user_id', $userId)
            ->orderBy('recorded_at', 'desc')
            ->limit($limit)
            ->get();
    }
}
```

## 4. Service — `app/Services/ProfileService.php`

```php
<?php
namespace App\Services;

use App\Models\User;
use App\Models\UserAddress;
use App\Models\UserLocation;
use App\Repositories\Contracts\UserAddressRepositoryInterface;
use App\Repositories\Contracts\UserContactRepositoryInterface;
use App\Repositories\Contracts\UserCredentialRepositoryInterface;
use App\Repositories\Contracts\UserLocationRepositoryInterface;
use App\Repositories\Contracts\UserRepositoryInterface;
use App\Repositories\Contracts\UserSessionRepositoryInterface;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;

class ProfileService
{
    public function __construct(
        private readonly UserRepositoryInterface $users,
        private readonly UserContactRepositoryInterface $contacts,
        private readonly UserCredentialRepositoryInterface $credentials,
        private readonly UserAddressRepositoryInterface $addresses,
        private readonly UserSessionRepositoryInterface $sessions,
        private readonly UserLocationRepositoryInterface $locations,
    ) {}

    public function listProfiles(): Collection { return $this->users->all(); }

    public function getProfile(string $id): ?User { return $this->users->findWithProfile($id); }

    public function registerProfile(array $data): User
    {
        return DB::transaction(function () use ($data) {
            $user = $this->users->create([
                'full_name' => $data['full_name'],
                'username' => $data['username'],
                'date_of_birth' => $data['date_of_birth'] ?? null,
                'is_active' => $data['is_active'] ?? true,
            ]);

            if (!empty($data['contact'])) {
                $this->contacts->upsertForUser($user->user_id, $data['contact']);
            }

            if (!empty($data['password'])) {
                $this->credentials->rotatePassword($user->user_id, Hash::make($data['password']));
            }

            foreach ($data['addresses'] ?? [] as $addr) {
                $this->addresses->create(array_merge($addr, ['user_id' => $user->user_id]));
            }

            return $this->users->findWithProfile($user->user_id);
        });
    }

    public function updateProfile(string $id, array $data): User
    {
        return DB::transaction(function () use ($id, $data) {
            $fields = array_filter([
                'full_name' => $data['full_name'] ?? null,
                'username' => $data['username'] ?? null,
                'date_of_birth' => $data['date_of_birth'] ?? null,
                'is_active' => $data['is_active'] ?? null,
            ], fn($v) => $v !== null);

            if ($fields) $this->users->update($id, $fields);

            if (!empty($data['contact'])) {
                $this->contacts->upsertForUser($id, $data['contact']);
            }

            if (!empty($data['password'])) {
                $this->credentials->rotatePassword($id, Hash::make($data['password']));
            }

            return $this->users->findWithProfile($id);
        });
    }

    public function deactivateProfile(string $id): bool
    {
        $this->sessions->revokeAllForUser($id);
        return $this->users->deactivate($id);
    }

    public function deleteProfile(string $id): bool { return $this->users->delete($id); }

    public function addAddress(string $userId, array $data): UserAddress {
        return $this->addresses->create(array_merge($data, ['user_id' => $userId]));
    }

    public function setPrimaryAddress(string $userId, string $addressId): bool {
        return $this->addresses->setPrimary($userId, $addressId);
    }

    public function recordLocation(string $userId, array $data): UserLocation {
        return $this->locations->create(array_merge($data, ['user_id' => $userId]));
    }

    public function latestLocation(string $userId): ?UserLocation {
        return $this->locations->latestForUser($userId);
    }

    public function activeSessions(string $userId): Collection {
        return $this->sessions->activeForUser($userId);
    }

    public function revokeSession(string $sessionId): bool {
        return $this->sessions->revoke($sessionId);
    }
}
```

## 5. Service provider

```php
// app/Providers/RepositoryServiceProvider.php
<?php
namespace App\Providers;

use App\Repositories\Contracts\UserAddressRepositoryInterface;
use App\Repositories\Contracts\UserContactRepositoryInterface;
use App\Repositories\Contracts\UserCredentialRepositoryInterface;
use App\Repositories\Contracts\UserLocationRepositoryInterface;
use App\Repositories\Contracts\UserRepositoryInterface;
use App\Repositories\Contracts\UserSessionRepositoryInterface;
use App\Repositories\Eloquent\UserAddressRepository;
use App\Repositories\Eloquent\UserContactRepository;
use App\Repositories\Eloquent\UserCredentialRepository;
use App\Repositories\Eloquent\UserLocationRepository;
use App\Repositories\Eloquent\UserRepository;
use App\Repositories\Eloquent\UserSessionRepository;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    private array $repositoryBindings = [
        UserRepositoryInterface::class => UserRepository::class,
        UserContactRepositoryInterface::class => UserContactRepository::class,
        UserCredentialRepositoryInterface::class => UserCredentialRepository::class,
        UserAddressRepositoryInterface::class => UserAddressRepository::class,
        UserSessionRepositoryInterface::class => UserSessionRepository::class,
        UserLocationRepositoryInterface::class => UserLocationRepository::class,
    ];

    public function register(): void
    {
        foreach ($this->repositoryBindings as $interface => $implementation) {
            $this->app->bind($interface, $implementation);
        }
    }
}
```

Register in `bootstrap/providers.php`: add `App\Providers\RepositoryServiceProvider::class`.

## 6. HTTP layer

### Form requests — `app/Http/Requests/Profile/`

```php
// StoreProfileRequest.php
<?php
namespace App\Http\Requests\Profile;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreProfileRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'full_name' => ['required', 'string', 'max:120'],
            'username' => ['required', 'string', 'max:60', 'unique:users,username'],
            'date_of_birth' => ['nullable', 'date', 'before:today'],
            'is_active' => ['nullable', 'boolean'],
            'password' => ['nullable', 'string', 'min:8'],
            'contact' => ['nullable', 'array'],
            'contact.email' => ['nullable', 'email:rfc', 'unique:user_contacts,email'],
            'contact.phone_number' => ['nullable', 'string', 'max:20'],
            'addresses' => ['nullable', 'array'],
            'addresses.*.address_type' => ['nullable', Rule::in(['home','billing','shipping'])],
            'addresses.*.street_line_1' => ['nullable', 'string', 'max:200'],
            'addresses.*.street_line_2' => ['nullable', 'string', 'max:200'],
            'addresses.*.city' => ['nullable', 'string', 'max:100'],
            'addresses.*.state_province' => ['nullable', 'string', 'max:100'],
            'addresses.*.postal_code' => ['nullable', 'string', 'max:20'],
            'addresses.*.country_code' => ['nullable', 'string', 'size:2'],
            'addresses.*.is_primary' => ['nullable', 'boolean'],
        ];
    }
}
```

```php
// UpdateProfileRequest.php
<?php
namespace App\Http\Requests\Profile;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateProfileRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'full_name' => ['sometimes', 'string', 'max:120'],
            'username' => ['sometimes', 'string', 'max:60', Rule::unique('users', 'username')->ignore($this->route('user'), 'user_id')],
            'date_of_birth' => ['sometimes', 'nullable', 'date', 'before:today'],
            'is_active' => ['sometimes', 'boolean'],
            'password' => ['sometimes', 'nullable', 'string', 'min:8'],
            'contact' => ['sometimes', 'array'],
            'contact.email' => ['sometimes', 'nullable', 'email:rfc', Rule::unique('user_contacts', 'email')->ignore($this->route('user'), 'user_id')],
            'contact.phone_number' => ['sometimes', 'nullable', 'string', 'max:20'],
        ];
    }
}
```

`StoreAddressRequest`: address fields with `Rule::in(['home','billing','shipping'])`. All fields nullable. `authorize()` returns true.

`StoreLocationRequest`: lat `between:-90,90`; lon `between:-180,180`; `accuracy_meters` integer min 0; recorded_at date; source `Rule::in(['gps','ip','manual'])`. `authorize()` returns true.

### Resources — `app/Http/Resources/Profile/`

`ContactResource`, `AddressResource`, `SessionResource`, `LocationResource` map their public columns (all columns including PK and user_id).

`ProfileResource` exposes user fields plus:
- `'contact' => new ContactResource($this->whenLoaded('contact'))`
- `'addresses' => AddressResource::collection($this->whenLoaded('addresses'))`
- `'sessions' => SessionResource::collection($this->whenLoaded('sessions'))`
- `'locations' => LocationResource::collection($this->whenLoaded('locations'))`
- Cast `'date_of_birth' => $this->date_of_birth?->toDateString()`

### Controller — `app/Http/Controllers/Api/ProfileController.php`

```php
<?php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\Profile\StoreAddressRequest;
use App\Http\Requests\Profile\StoreLocationRequest;
use App\Http\Requests\Profile\StoreProfileRequest;
use App\Http\Requests\Profile\UpdateProfileRequest;
use App\Http\Resources\Profile\AddressResource;
use App\Http\Resources\Profile\LocationResource;
use App\Http\Resources\Profile\ProfileResource;
use App\Http\Resources\Profile\SessionResource;
use App\Services\ProfileService;
use Illuminate\Http\JsonResponse;

class ProfileController extends Controller
{
    public function __construct(private readonly ProfileService $service) {}

    public function index(): JsonResponse
    {
        return response()->json(ProfileResource::collection($this->service->listProfiles()));
    }

    public function show(string $user): JsonResponse
    {
        $profile = $this->service->getProfile($user);
        if (!$profile) return response()->json(['message' => 'Not found'], 404);
        return response()->json(new ProfileResource($profile));
    }

    public function store(StoreProfileRequest $request): JsonResponse
    {
        $profile = $this->service->registerProfile($request->validated());
        return response()->json(new ProfileResource($profile), 201);
    }

    public function update(UpdateProfileRequest $request, string $user): JsonResponse
    {
        $profile = $this->service->updateProfile($user, $request->validated());
        return response()->json(new ProfileResource($profile));
    }

    public function destroy(string $user): JsonResponse
    {
        return response()->json(['deleted' => $this->service->deleteProfile($user)]);
    }

    public function deactivate(string $user): JsonResponse
    {
        return response()->json(['deactivated' => $this->service->deactivateProfile($user)]);
    }

    public function addAddress(StoreAddressRequest $request, string $user): JsonResponse
    {
        $address = $this->service->addAddress($user, $request->validated());
        return response()->json(new AddressResource($address), 201);
    }

    public function setPrimaryAddress(string $user, string $address): JsonResponse
    {
        return response()->json(['primary' => $this->service->setPrimaryAddress($user, $address)]);
    }

    public function recordLocation(StoreLocationRequest $request, string $user): JsonResponse
    {
        $location = $this->service->recordLocation($user, $request->validated());
        return response()->json(new LocationResource($location), 201);
    }

    public function latestLocation(string $user): JsonResponse
    {
        $location = $this->service->latestLocation($user);
        return response()->json(['data' => $location ? new LocationResource($location) : null]);
    }

    public function activeSessions(string $user): JsonResponse
    {
        return response()->json(SessionResource::collection($this->service->activeSessions($user)));
    }

    public function revokeSession(string $user, string $session): JsonResponse
    {
        return response()->json(['revoked' => $this->service->revokeSession($session)]);
    }
}
```

### Routes — `routes/api.php` (run `php artisan install:api` first)

```php
<?php
use App\Http\Controllers\Api\ProfileController;
use Illuminate\Support\Facades\Route;

Route::prefix('profiles')->group(function () {
    Route::get('/',          [ProfileController::class, 'index']);
    Route::post('/',         [ProfileController::class, 'store']);
    Route::get('/{user}',    [ProfileController::class, 'show']);
    Route::patch('/{user}',  [ProfileController::class, 'update']);
    Route::delete('/{user}', [ProfileController::class, 'destroy']);
    Route::post('/{user}/deactivate', [ProfileController::class, 'deactivate']);
    Route::post('/{user}/addresses',                  [ProfileController::class, 'addAddress']);
    Route::post('/{user}/addresses/{address}/primary',[ProfileController::class, 'setPrimaryAddress']);
    Route::get('/{user}/locations/latest', [ProfileController::class, 'latestLocation']);
    Route::post('/{user}/locations',       [ProfileController::class, 'recordLocation']);
    Route::get('/{user}/sessions',                  [ProfileController::class, 'activeSessions']);
    Route::delete('/{user}/sessions/{session}',     [ProfileController::class, 'revokeSession']);
});
```

## 7. Verification

1. `php artisan migrate` — all 9 migrations should run (cache, jobs, 6 profile tables, personal_access_tokens from `install:api`).
2. `php artisan route:list --path=profiles` — should show 12 routes.
3. Boot server (`php artisan serve`) and hit:

```
curl -X POST http://127.0.0.1:8000/api/profiles \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"full_name":"Demo User","username":"demo_user1","date_of_birth":"1995-04-01","contact":{"email":"demo_user1@example.com","phone_number":"+66812345678"},"password":"supersecret","addresses":[{"address_type":"home","street_line_1":"123 Main","city":"Bangkok","country_code":"TH","is_primary":true}]}'
```

Expect 201 with nested contact + addresses, UUID `user_id`. Then `GET /api/profiles/{user_id}` should return same payload with empty `sessions`/`locations` arrays.

## Constraints

- No extra packages (no spatie, no laravel-data, etc.).
- No factories/seeders unless asked.
- No auth middleware on routes (service is the focus, not auth).
- Don't write multi-paragraph docblocks. One-line comments only when truly needed.
- Don't add `email_verified_at` / Laravel-default user fields — we follow the schema strictly.
- Use `$this->model->newQuery()` inside repositories rather than holding stateful query builders.
- All write paths that touch multiple tables wrap in `DB::transaction`.

When done, run the smoke test in section 7 and report the JSON response.
