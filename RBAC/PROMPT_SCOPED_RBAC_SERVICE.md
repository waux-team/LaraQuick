# Prompt — Add Scoped RBAC (Organisations + Teams) to Profile Service

Copy everything below the `---` line into a Claude Code / LLM session inside the Laravel 13 project that already has the Profile service. This prompt produces a fully scoped Role & Permission system. Do not modify any existing migration, model, repository, service, or route unless explicitly stated.

---

You are working in a Laravel 13 project (PHP 8.3+, PostgreSQL 15+) that already has a Profile service. You are adding a **fully scoped RBAC system** supporting two scope levels: Organisation and Team (org → team hierarchy). Do not touch any existing file unless explicitly instructed.

Assume DB is configured. `SESSION_DRIVER=file`.

## Design decisions baked into this prompt

- Permissions structured as `resource:action` (e.g. `profiles:delete`)
- Users can hold **multiple roles simultaneously across different scopes** (e.g. admin in Org A, viewer in Team X)
- Role nesting: roles inherit parent role permissions via `role_inheritance`
- Permission resolution = **union of all scopes** (most permissive wins)
- Org-level role **cascades** to all teams within that org during resolution
- `super_admin` role bypasses **all** scope checks globally
- Explicit `org_members` and `team_members` tables (separate from role assignment)
- Full `permission_audit_log` capturing `scope_type` and `scope_id` per event
- `expires_at` on both `user_roles` and `user_permissions`

## Style rules (identical to Profile service)

- Single-line compact style for short methods: `public function foo(): Bar { return $this->bar; }`
- Properties on one line: `protected $fillable = ['id','col1','col2'];`
- `$casts` on one line: `protected $casts = ['flag' => 'boolean'];`
- No space after `!` in conditions
- Controller property name: `$service`
- All controller methods return `JsonResponse`; wrap every return in `response()->json()`
- 404 pattern: `if (!$record) return response()->json(['message' => 'Not found'], 404);`
- Service input param: `$data`
- `use Illuminate\Support\Collection;`
- Concrete repository constructors: `public function __construct(ModelClass $model) { parent::__construct($model); }`
- `$this->model->newQuery()` inside every repository method
- All multi-table writes in `DB::transaction`

---

## 1. Database migrations (11 new files — do not touch existing ones)

### `organisations`

| col | type | notes |
|---|---|---|
| org_id | UUID | PK |
| name | string(120) | |
| slug | string(80) | unique |
| description | text | nullable |
| is_active | boolean | default true |
| timestamps | | |

Index `slug`.

### `teams`

| col | type | notes |
|---|---|---|
| team_id | UUID | PK |
| org_id | UUID | FK→organisations.org_id ON DELETE CASCADE |
| name | string(120) | |
| slug | string(80) | |
| description | text | nullable |
| is_active | boolean | default true |
| timestamps | | |

Unique index on `(org_id, slug)`. Index `org_id`.

### `org_members`

| col | type | notes |
|---|---|---|
| org_member_id | UUID | PK |
| org_id | UUID | FK→organisations.org_id ON DELETE CASCADE |
| user_id | UUID | FK→users.user_id ON DELETE CASCADE |
| invited_by | UUID | nullable, FK→users.user_id (no cascade) |
| joined_at | timestamp | nullable |
| timestamps | | |

Unique index on `(org_id, user_id)`. Index `user_id`.

### `team_members`

| col | type | notes |
|---|---|---|
| team_member_id | UUID | PK |
| team_id | UUID | FK→teams.team_id ON DELETE CASCADE |
| user_id | UUID | FK→users.user_id ON DELETE CASCADE |
| invited_by | UUID | nullable, FK→users.user_id (no cascade) |
| joined_at | timestamp | nullable |
| timestamps | | |

Unique index on `(team_id, user_id)`. Index `user_id`.

### `roles`

| col | type | notes |
|---|---|---|
| role_id | UUID | PK |
| name | string(80) | unique |
| display_name | string(120) | |
| description | text | nullable |
| is_system | boolean | default false |
| scope_hint | enum('global','org','team') | nullable — advisory only, not enforced |
| timestamps | | |

### `role_inheritance`

| col | type | notes |
|---|---|---|
| parent_role_id | UUID | FK→roles.role_id ON DELETE CASCADE |
| child_role_id | UUID | FK→roles.role_id ON DELETE CASCADE |
| PK | | composite (parent_role_id, child_role_id) |

No timestamps. Index both FK columns individually.

### `permissions`

| col | type | notes |
|---|---|---|
| permission_id | UUID | PK |
| resource | string(80) | |
| action | string(80) | |
| name | string(160) | unique — always `resource:action` |
| display_name | string(200) | nullable |
| description | text | nullable |
| timestamps | | |

Unique index on `(resource, action)`.

### `role_permissions`

| col | type | notes |
|---|---|---|
| role_id | UUID | FK→roles.role_id ON DELETE CASCADE |
| permission_id | UUID | FK→permissions.permission_id ON DELETE CASCADE |
| PK | | composite (role_id, permission_id) |

No timestamps.

### `user_roles`

| col | type | notes |
|---|---|---|
| user_role_id | UUID | PK |
| user_id | UUID | FK→users.user_id ON DELETE CASCADE |
| role_id | UUID | FK→roles.role_id ON DELETE CASCADE |
| scope_type | enum('global','org','team') | default 'global' |
| scope_id | UUID | nullable — org_id or team_id depending on scope_type |
| assigned_by | UUID | nullable, FK→users.user_id (no cascade) |
| assigned_at | timestamp | default now() |
| expires_at | timestamp | nullable |
| timestamps | | |

Unique index on `(user_id, role_id, scope_type, scope_id)`. Index `(user_id, scope_type, scope_id)`.

### `user_permissions`

| col | type | notes |
|---|---|---|
| user_permission_id | UUID | PK |
| user_id | UUID | FK→users.user_id ON DELETE CASCADE |
| permission_id | UUID | FK→permissions.permission_id ON DELETE CASCADE |
| scope_type | enum('global','org','team') | default 'global' |
| scope_id | UUID | nullable |
| granted_by | UUID | nullable, FK→users.user_id (no cascade) |
| granted_at | timestamp | default now() |
| expires_at | timestamp | nullable |
| timestamps | | |

Unique index on `(user_id, permission_id, scope_type, scope_id)`. Index `(user_id, scope_type, scope_id)`.

### `permission_audit_log`

| col | type | notes |
|---|---|---|
| audit_id | UUID | PK |
| actor_id | UUID | nullable |
| target_user_id | UUID | nullable |
| action | enum('role_assigned','role_revoked','permission_granted','permission_revoked','role_created','role_deleted','permission_created','permission_deleted','member_added','member_removed') | |
| subject_type | string(50) | |
| subject_id | UUID | nullable |
| subject_name | string(200) | |
| scope_type | string(20) | nullable |
| scope_id | UUID | nullable |
| metadata | json | nullable |
| created_at | timestamp | only — no updated_at |

Index `(target_user_id)`, `(actor_id)`, `(created_at)`, `(scope_type, scope_id)`.

---

## 2. Models — `app/Models/`

```php
// Organisation.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Organisation extends Model
{
    use HasUuids;
    protected $table = 'organisations';
    protected $primaryKey = 'org_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['org_id']; }
    protected $fillable = ['org_id','name','slug','description','is_active'];
    protected $casts = ['is_active' => 'boolean'];

    public function teams(): HasMany { return $this->hasMany(Team::class, 'org_id', 'org_id'); }
    public function members(): HasMany { return $this->hasMany(OrgMember::class, 'org_id', 'org_id'); }
}
```

```php
// Team.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Team extends Model
{
    use HasUuids;
    protected $table = 'teams';
    protected $primaryKey = 'team_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['team_id']; }
    protected $fillable = ['team_id','org_id','name','slug','description','is_active'];
    protected $casts = ['is_active' => 'boolean'];

    public function organisation(): BelongsTo { return $this->belongsTo(Organisation::class, 'org_id', 'org_id'); }
    public function members(): HasMany { return $this->hasMany(TeamMember::class, 'team_id', 'team_id'); }
}
```

```php
// OrgMember.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class OrgMember extends Model
{
    use HasUuids;
    protected $table = 'org_members';
    protected $primaryKey = 'org_member_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['org_member_id']; }
    protected $fillable = ['org_member_id','org_id','user_id','invited_by','joined_at'];
    protected $casts = ['joined_at' => 'datetime'];

    public function organisation(): BelongsTo { return $this->belongsTo(Organisation::class, 'org_id', 'org_id'); }
    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// TeamMember.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class TeamMember extends Model
{
    use HasUuids;
    protected $table = 'team_members';
    protected $primaryKey = 'team_member_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['team_member_id']; }
    protected $fillable = ['team_member_id','team_id','user_id','invited_by','joined_at'];
    protected $casts = ['joined_at' => 'datetime'];

    public function team(): BelongsTo { return $this->belongsTo(Team::class, 'team_id', 'team_id'); }
    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// Role.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Role extends Model
{
    use HasUuids;
    protected $table = 'roles';
    protected $primaryKey = 'role_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['role_id']; }
    protected $fillable = ['role_id','name','display_name','description','is_system','scope_hint'];
    protected $casts = ['is_system' => 'boolean'];

    public function permissions(): BelongsToMany { return $this->belongsToMany(Permission::class, 'role_permissions', 'role_id', 'permission_id'); }
    public function parents(): BelongsToMany { return $this->belongsToMany(Role::class, 'role_inheritance', 'child_role_id', 'parent_role_id'); }
    public function children(): BelongsToMany { return $this->belongsToMany(Role::class, 'role_inheritance', 'parent_role_id', 'child_role_id'); }
}
```

```php
// Permission.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Permission extends Model
{
    use HasUuids;
    protected $table = 'permissions';
    protected $primaryKey = 'permission_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['permission_id']; }
    protected $fillable = ['permission_id','resource','action','name','display_name','description'];

    public function roles(): BelongsToMany { return $this->belongsToMany(Role::class, 'role_permissions', 'permission_id', 'role_id'); }
}
```

```php
// UserRole.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UserRole extends Model
{
    use HasUuids;
    protected $table = 'user_roles';
    protected $primaryKey = 'user_role_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['user_role_id']; }
    protected $fillable = ['user_role_id','user_id','role_id','scope_type','scope_id','assigned_by','assigned_at','expires_at'];
    protected $casts = ['assigned_at' => 'datetime', 'expires_at' => 'datetime'];

    public function role(): BelongsTo { return $this->belongsTo(Role::class, 'role_id', 'role_id'); }
    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// UserPermission.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UserPermission extends Model
{
    use HasUuids;
    protected $table = 'user_permissions';
    protected $primaryKey = 'user_permission_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public function uniqueIds(): array { return ['user_permission_id']; }
    protected $fillable = ['user_permission_id','user_id','permission_id','scope_type','scope_id','granted_by','granted_at','expires_at'];
    protected $casts = ['granted_at' => 'datetime', 'expires_at' => 'datetime'];

    public function permission(): BelongsTo { return $this->belongsTo(Permission::class, 'permission_id', 'permission_id'); }
    public function user(): BelongsTo { return $this->belongsTo(User::class, 'user_id', 'user_id'); }
}
```

```php
// PermissionAuditLog.php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class PermissionAuditLog extends Model
{
    use HasUuids;
    protected $table = 'permission_audit_log';
    protected $primaryKey = 'audit_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public $timestamps = false;
    public function uniqueIds(): array { return ['audit_id']; }
    protected $fillable = ['audit_id','actor_id','target_user_id','action','subject_type','subject_id','subject_name','scope_type','scope_id','metadata','created_at'];
    protected $casts = ['metadata' => 'array', 'created_at' => 'datetime'];
}
```

**Append to existing `app/Models/User.php`** (add after the `locations()` relation — change nothing else):

```php
public function orgMemberships(): HasMany { return $this->hasMany(OrgMember::class, 'user_id', 'user_id'); }
public function teamMemberships(): HasMany { return $this->hasMany(TeamMember::class, 'user_id', 'user_id'); }
public function userRoles(): HasMany { return $this->hasMany(UserRole::class, 'user_id', 'user_id'); }
public function userPermissions(): HasMany { return $this->hasMany(UserPermission::class, 'user_id', 'user_id'); }
```

Add to `User.php` use block: `use Illuminate\Database\Eloquent\Relations\HasMany;`

---

## 3. Repositories — interfaces in `app/Repositories/Contracts/`

```php
// OrganisationRepositoryInterface.php
<?php
namespace App\Repositories\Contracts;
interface OrganisationRepositoryInterface extends BaseRepositoryInterface
{
    public function findBySlug(string $slug);
    public function findWithTeams(string $id);
    public function getTeamIds(string $orgId): array;
}
```

```php
// TeamRepositoryInterface.php
<?php
namespace App\Repositories\Contracts;
interface TeamRepositoryInterface extends BaseRepositoryInterface
{
    public function findByOrgAndSlug(string $orgId, string $slug);
    public function listByOrg(string $orgId);
    public function getOrgId(string $teamId): ?string;
}
```

```php
// RoleRepositoryInterface.php
<?php
namespace App\Repositories\Contracts;
interface RoleRepositoryInterface extends BaseRepositoryInterface
{
    public function findByName(string $name);
    public function findWithPermissions(string $id);
    public function allWithPermissions();
    public function syncPermissions(string $roleId, array $permissionIds): void;
    public function addParent(string $childRoleId, string $parentRoleId): void;
    public function removeParent(string $childRoleId, string $parentRoleId): void;
    public function getParentIds(string $roleId): array;
}
```

```php
// PermissionRepositoryInterface.php
<?php
namespace App\Repositories\Contracts;
interface PermissionRepositoryInterface extends BaseRepositoryInterface
{
    public function findByName(string $name);
    public function findByResourceAction(string $resource, string $action);
    public function firstOrCreate(string $resource, string $action, array $extra = []);
}
```

```php
// UserRoleRepositoryInterface.php
<?php
namespace App\Repositories\Contracts;
use Illuminate\Support\Collection;
interface UserRoleRepositoryInterface
{
    // Returns UserRole models (with role.permissions eager-loaded) filtered by scope
    public function activeForUser(string $userId, ?string $scopeType = null, ?string $scopeId = null): Collection;
    public function assign(string $userId, string $roleId, string $scopeType, ?string $scopeId, ?string $assignedBy, ?\DateTimeInterface $expiresAt): void;
    public function revoke(string $userId, string $roleId, string $scopeType, ?string $scopeId): bool;
    public function directPermissionsForUser(string $userId, ?string $scopeType = null, ?string $scopeId = null): Collection;
    public function grantPermission(string $userId, string $permissionId, string $scopeType, ?string $scopeId, ?string $grantedBy, ?\DateTimeInterface $expiresAt): void;
    public function revokePermission(string $userId, string $permissionId, string $scopeType, ?string $scopeId): bool;
    public function hasSuperAdmin(string $userId): bool;
}
```

```php
// MemberRepositoryInterface.php
<?php
namespace App\Repositories\Contracts;
interface MemberRepositoryInterface
{
    public function addOrgMember(string $orgId, string $userId, ?string $invitedBy = null): void;
    public function removeOrgMember(string $orgId, string $userId): bool;
    public function addTeamMember(string $teamId, string $userId, ?string $invitedBy = null): void;
    public function removeTeamMember(string $teamId, string $userId): bool;
    public function isOrgMember(string $orgId, string $userId): bool;
    public function isTeamMember(string $teamId, string $userId): bool;
    public function orgMemberCount(string $orgId): int;
    public function teamMemberCount(string $teamId): int;
}
```

---

## 4. Repositories — Eloquent in `app/Repositories/Eloquent/`

```php
// OrganisationRepository.php
<?php
namespace App\Repositories\Eloquent;
use App\Models\Organisation;
use App\Repositories\Contracts\OrganisationRepositoryInterface;

class OrganisationRepository extends BaseRepository implements OrganisationRepositoryInterface
{
    public function __construct(Organisation $model) { parent::__construct($model); }

    public function findBySlug(string $slug) {
        return $this->model->newQuery()->where('slug', $slug)->first();
    }

    public function findWithTeams(string $id) {
        return $this->model->newQuery()->with('teams')->find($id);
    }

    public function getTeamIds(string $orgId): array {
        return $this->model->newQuery()
            ->with('teams:team_id,org_id')
            ->find($orgId)?->teams->pluck('team_id')->toArray() ?? [];
    }
}
```

```php
// TeamRepository.php
<?php
namespace App\Repositories\Eloquent;
use App\Models\Team;
use App\Repositories\Contracts\TeamRepositoryInterface;

class TeamRepository extends BaseRepository implements TeamRepositoryInterface
{
    public function __construct(Team $model) { parent::__construct($model); }

    public function findByOrgAndSlug(string $orgId, string $slug) {
        return $this->model->newQuery()->where('org_id', $orgId)->where('slug', $slug)->first();
    }

    public function listByOrg(string $orgId) {
        return $this->model->newQuery()->where('org_id', $orgId)->get();
    }

    public function getOrgId(string $teamId): ?string {
        return $this->model->newQuery()->where('team_id', $teamId)->value('org_id');
    }
}
```

```php
// RoleRepository.php
<?php
namespace App\Repositories\Eloquent;
use App\Models\Role;
use App\Repositories\Contracts\RoleRepositoryInterface;

class RoleRepository extends BaseRepository implements RoleRepositoryInterface
{
    public function __construct(Role $model) { parent::__construct($model); }

    public function findByName(string $name) {
        return $this->model->newQuery()->where('name', $name)->first();
    }

    public function findWithPermissions(string $id) {
        return $this->model->newQuery()->with(['permissions','parents','children'])->find($id);
    }

    public function allWithPermissions() {
        return $this->model->newQuery()->with(['permissions','parents'])->get();
    }

    public function syncPermissions(string $roleId, array $permissionIds): void {
        $this->findOrFail($roleId)->permissions()->sync($permissionIds);
    }

    public function addParent(string $childRoleId, string $parentRoleId): void {
        $this->findOrFail($childRoleId)->parents()->syncWithoutDetaching([$parentRoleId]);
    }

    public function removeParent(string $childRoleId, string $parentRoleId): void {
        $this->findOrFail($childRoleId)->parents()->detach($parentRoleId);
    }

    public function getParentIds(string $roleId): array {
        $role = $this->model->newQuery()->with('parents:role_id')->find($roleId);
        return $role ? $role->parents->pluck('role_id')->toArray() : [];
    }
}
```

```php
// PermissionRepository.php
<?php
namespace App\Repositories\Eloquent;
use App\Models\Permission;
use App\Repositories\Contracts\PermissionRepositoryInterface;

class PermissionRepository extends BaseRepository implements PermissionRepositoryInterface
{
    public function __construct(Permission $model) { parent::__construct($model); }

    public function findByName(string $name) {
        return $this->model->newQuery()->where('name', $name)->first();
    }

    public function findByResourceAction(string $resource, string $action) {
        return $this->model->newQuery()->where('resource', $resource)->where('action', $action)->first();
    }

    public function firstOrCreate(string $resource, string $action, array $extra = []) {
        return $this->model->newQuery()->firstOrCreate(
            ['resource' => $resource, 'action' => $action],
            array_merge(['name' => "{$resource}:{$action}"], $extra)
        );
    }
}
```

```php
// UserRoleRepository.php
<?php
namespace App\Repositories\Eloquent;
use App\Models\UserPermission;
use App\Models\UserRole;
use App\Repositories\Contracts\UserRoleRepositoryInterface;
use Carbon\Carbon;
use Illuminate\Support\Collection;

class UserRoleRepository implements UserRoleRepositoryInterface
{
    public function __construct(
        private readonly UserRole $userRole,
        private readonly UserPermission $userPermission,
    ) {}

    public function activeForUser(string $userId, ?string $scopeType = null, ?string $scopeId = null): Collection {
        $q = $this->userRole->newQuery()
            ->with('role.permissions')
            ->where('user_id', $userId)
            ->where(fn($q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', Carbon::now()));

        if ($scopeType !== null) {
            $q->where('scope_type', $scopeType);
            $scopeId === null ? $q->whereNull('scope_id') : $q->where('scope_id', $scopeId);
        }

        return $q->get()->pluck('role')->filter()->values();
    }

    public function assign(string $userId, string $roleId, string $scopeType, ?string $scopeId, ?string $assignedBy, ?\DateTimeInterface $expiresAt): void {
        $this->userRole->newQuery()->updateOrCreate(
            ['user_id' => $userId, 'role_id' => $roleId, 'scope_type' => $scopeType, 'scope_id' => $scopeId],
            ['assigned_by' => $assignedBy, 'assigned_at' => Carbon::now(), 'expires_at' => $expiresAt]
        );
    }

    public function revoke(string $userId, string $roleId, string $scopeType, ?string $scopeId): bool {
        $q = $this->userRole->newQuery()
            ->where('user_id', $userId)->where('role_id', $roleId)->where('scope_type', $scopeType);
        $scopeId === null ? $q->whereNull('scope_id') : $q->where('scope_id', $scopeId);
        return (bool) $q->delete();
    }

    public function directPermissionsForUser(string $userId, ?string $scopeType = null, ?string $scopeId = null): Collection {
        $q = $this->userPermission->newQuery()
            ->with('permission')
            ->where('user_id', $userId)
            ->where(fn($q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', Carbon::now()));

        if ($scopeType !== null) {
            $q->where('scope_type', $scopeType);
            $scopeId === null ? $q->whereNull('scope_id') : $q->where('scope_id', $scopeId);
        }

        return $q->get()->pluck('permission')->filter()->values();
    }

    public function grantPermission(string $userId, string $permissionId, string $scopeType, ?string $scopeId, ?string $grantedBy, ?\DateTimeInterface $expiresAt): void {
        $this->userPermission->newQuery()->updateOrCreate(
            ['user_id' => $userId, 'permission_id' => $permissionId, 'scope_type' => $scopeType, 'scope_id' => $scopeId],
            ['granted_by' => $grantedBy, 'granted_at' => Carbon::now(), 'expires_at' => $expiresAt]
        );
    }

    public function revokePermission(string $userId, string $permissionId, string $scopeType, ?string $scopeId): bool {
        $q = $this->userPermission->newQuery()
            ->where('user_id', $userId)->where('permission_id', $permissionId)->where('scope_type', $scopeType);
        $scopeId === null ? $q->whereNull('scope_id') : $q->where('scope_id', $scopeId);
        return (bool) $q->delete();
    }

    public function hasSuperAdmin(string $userId): bool {
        return $this->userRole->newQuery()
            ->whereHas('role', fn($q) => $q->where('name', 'super_admin'))
            ->where('user_id', $userId)
            ->where(fn($q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', Carbon::now()))
            ->exists();
    }
}
```

```php
// MemberRepository.php
<?php
namespace App\Repositories\Eloquent;
use App\Models\OrgMember;
use App\Models\TeamMember;
use App\Repositories\Contracts\MemberRepositoryInterface;
use Carbon\Carbon;

class MemberRepository implements MemberRepositoryInterface
{
    public function __construct(
        private readonly OrgMember $orgMember,
        private readonly TeamMember $teamMember,
    ) {}

    public function addOrgMember(string $orgId, string $userId, ?string $invitedBy = null): void {
        $this->orgMember->newQuery()->firstOrCreate(
            ['org_id' => $orgId, 'user_id' => $userId],
            ['invited_by' => $invitedBy, 'joined_at' => Carbon::now()]
        );
    }

    public function removeOrgMember(string $orgId, string $userId): bool {
        return (bool) $this->orgMember->newQuery()
            ->where('org_id', $orgId)->where('user_id', $userId)->delete();
    }

    public function addTeamMember(string $teamId, string $userId, ?string $invitedBy = null): void {
        $this->teamMember->newQuery()->firstOrCreate(
            ['team_id' => $teamId, 'user_id' => $userId],
            ['invited_by' => $invitedBy, 'joined_at' => Carbon::now()]
        );
    }

    public function removeTeamMember(string $teamId, string $userId): bool {
        return (bool) $this->teamMember->newQuery()
            ->where('team_id', $teamId)->where('user_id', $userId)->delete();
    }

    public function isOrgMember(string $orgId, string $userId): bool {
        return $this->orgMember->newQuery()->where('org_id', $orgId)->where('user_id', $userId)->exists();
    }

    public function isTeamMember(string $teamId, string $userId): bool {
        return $this->teamMember->newQuery()->where('team_id', $teamId)->where('user_id', $userId)->exists();
    }

    public function orgMemberCount(string $orgId): int {
        return $this->orgMember->newQuery()->where('org_id', $orgId)->count();
    }

    public function teamMemberCount(string $teamId): int {
        return $this->teamMember->newQuery()->where('team_id', $teamId)->count();
    }
}
```

---

## 5. Services

### `app/Services/ScopedPermissionService.php`

This is the core service. The `userCan()` method implements the full resolution chain.

```php
<?php
namespace App\Services;

use App\Models\PermissionAuditLog;
use App\Repositories\Contracts\OrganisationRepositoryInterface;
use App\Repositories\Contracts\PermissionRepositoryInterface;
use App\Repositories\Contracts\RoleRepositoryInterface;
use App\Repositories\Contracts\UserRoleRepositoryInterface;
use Carbon\Carbon;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;

class ScopedPermissionService
{
    private const SUPER_ADMIN = 'super_admin';

    public function __construct(
        private readonly RoleRepositoryInterface $roles,
        private readonly PermissionRepositoryInterface $permissions,
        private readonly UserRoleRepositoryInterface $userRoles,
        private readonly OrganisationRepositoryInterface $orgs,
    ) {}

    // ── Core check ────────────────────────────────────────────────────────

    /**
     * Check if a user has a permission, optionally within a specific scope.
     *
     * Resolution order (union — most permissive wins):
     *   1. Global super_admin → always true
     *   2. Collect roles from: global scope + requested scope + org cascade (if team scope, also pull org roles)
     *   3. Direct user_permissions matching global or the specific scope
     *   4. Walk role_inheritance up each role's parent chain
     *
     * @param string      $userId
     * @param string      $permission  e.g. 'profiles:delete'
     * @param string|null $scopeType   'org' | 'team' | null (global)
     * @param string|null $scopeId     UUID of the org or team
     */
    public function userCan(string $userId, string $permission, ?string $scopeType = null, ?string $scopeId = null): bool
    {
        // 1. Global super_admin bypass
        if ($this->userRoles->hasSuperAdmin($userId)) return true;

        // 2. Collect all relevant roles (union of global + scoped + org cascade)
        $roles = $this->collectRoles($userId, $scopeType, $scopeId);

        // 3. Check direct permissions (global + scoped)
        $directGlobal = $this->userRoles->directPermissionsForUser($userId, 'global', null);
        if ($directGlobal->contains('name', $permission)) return true;

        if ($scopeType && $scopeId) {
            $directScoped = $this->userRoles->directPermissionsForUser($userId, $scopeType, $scopeId);
            if ($directScoped->contains('name', $permission)) return true;
        }

        // 4. Check role permissions + inherited
        $visited = [];
        foreach ($roles as $role) {
            if ($role->permissions->contains('name', $permission)) return true;
            if ($this->inheritedCan($role->role_id, $permission, $visited)) return true;
        }

        return false;
    }

    /**
     * Collect all roles applicable to this user in this scope context:
     *  - Always includes global-scope roles
     *  - If scopeType = 'team': also loads org-scope roles for the team's parent org (cascade)
     *  - If scopeType = 'org' or 'team': also loads the exact scope roles
     */
    private function collectRoles(string $userId, ?string $scopeType, ?string $scopeId): Collection
    {
        // Global roles always apply
        $roles = $this->userRoles->activeForUser($userId, 'global', null);

        if (!$scopeType || !$scopeId) return $roles;

        // Exact scope roles
        $roles = $roles->merge($this->userRoles->activeForUser($userId, $scopeType, $scopeId));

        // If team scope: also pull org-level roles for the parent org (cascade down)
        if ($scopeType === 'team') {
            $orgId = $this->orgs->getTeamIds($scopeId) ? null : null; // resolve orgId from team
            // Use TeamRepositoryInterface via the org repo shortcut isn't available here;
            // instead query the team's org_id from the UserRole's scope chain:
            $orgId = DB::table('teams')->where('team_id', $scopeId)->value('org_id');
            if ($orgId) {
                $roles = $roles->merge($this->userRoles->activeForUser($userId, 'org', $orgId));
            }
        }

        return $roles->unique('role_id')->values();
    }

    private function inheritedCan(string $roleId, string $permission, array &$visited): bool
    {
        if (in_array($roleId, $visited, true)) return false;
        $visited[] = $roleId;
        foreach ($this->roles->getParentIds($roleId) as $parentId) {
            $parent = $this->roles->findWithPermissions($parentId);
            if (!$parent) continue;
            if ($parent->permissions->contains('name', $permission)) return true;
            if ($this->inheritedCan($parentId, $permission, $visited)) return true;
        }
        return false;
    }

    public function resolveAllPermissions(string $userId, ?string $scopeType = null, ?string $scopeId = null): Collection
    {
        $roles = $this->collectRoles($userId, $scopeType, $scopeId);
        $fromRoles = $roles->flatMap(fn($r) => $r->permissions);

        $direct = $this->userRoles->directPermissionsForUser($userId, 'global', null);
        if ($scopeType && $scopeId) {
            $direct = $direct->merge($this->userRoles->directPermissionsForUser($userId, $scopeType, $scopeId));
        }

        $visited = [];
        $inherited = collect();
        foreach ($roles as $role) {
            $this->collectInherited($role->role_id, $visited, $inherited);
        }

        return $direct->merge($fromRoles)->merge($inherited)->unique('permission_id')->values();
    }

    private function collectInherited(string $roleId, array &$visited, Collection &$acc): void
    {
        if (in_array($roleId, $visited, true)) return;
        $visited[] = $roleId;
        foreach ($this->roles->getParentIds($roleId) as $parentId) {
            $parent = $this->roles->findWithPermissions($parentId);
            if (!$parent) continue;
            $acc = $acc->merge($parent->permissions);
            $this->collectInherited($parentId, $visited, $acc);
        }
    }

    // ── Role CRUD ─────────────────────────────────────────────────────────

    public function listRoles(): Collection { return $this->roles->allWithPermissions(); }
    public function getRole(string $id) { return $this->roles->findWithPermissions($id); }

    public function createRole(array $data)
    {
        return DB::transaction(function () use ($data) {
            $role = $this->roles->create(array_filter([
                'name' => $data['name'],
                'display_name' => $data['display_name'],
                'description' => $data['description'] ?? null,
                'is_system' => $data['is_system'] ?? false,
                'scope_hint' => $data['scope_hint'] ?? null,
            ], fn($v) => $v !== null));
            if (!empty($data['permission_ids'])) {
                $this->roles->syncPermissions($role->role_id, $data['permission_ids']);
            }
            $this->audit(null, null, 'role_created', 'role', $role->role_id, $role->name);
            return $this->roles->findWithPermissions($role->role_id);
        });
    }

    public function updateRole(string $id, array $data)
    {
        return DB::transaction(function () use ($id, $data) {
            $fields = array_filter([
                'display_name' => $data['display_name'] ?? null,
                'description' => $data['description'] ?? null,
                'scope_hint' => $data['scope_hint'] ?? null,
            ], fn($v) => $v !== null);
            if ($fields) $this->roles->update($id, $fields);
            if (isset($data['permission_ids'])) {
                $this->roles->syncPermissions($id, $data['permission_ids']);
            }
            return $this->roles->findWithPermissions($id);
        });
    }

    public function deleteRole(string $id): bool
    {
        $role = $this->roles->find($id);
        if (!$role || $role->is_system) return false;
        $deleted = $this->roles->delete($id);
        if ($deleted) $this->audit(null, null, 'role_deleted', 'role', $id, $role->name);
        return $deleted;
    }

    public function addRoleParent(string $childId, string $parentId): void { $this->roles->addParent($childId, $parentId); }
    public function removeRoleParent(string $childId, string $parentId): void { $this->roles->removeParent($childId, $parentId); }
    public function syncRolePermissions(string $roleId, array $permissionIds): void { $this->roles->syncPermissions($roleId, $permissionIds); }

    // ── Permission CRUD ───────────────────────────────────────────────────

    public function listPermissions(): Collection { return $this->permissions->all(); }
    public function getPermission(string $id) { return $this->permissions->find($id); }

    public function createPermission(array $data)
    {
        $data['name'] = "{$data['resource']}:{$data['action']}";
        $perm = $this->permissions->create($data);
        $this->audit(null, null, 'permission_created', 'permission', $perm->permission_id, $perm->name);
        return $perm;
    }

    public function deletePermission(string $id): bool
    {
        $perm = $this->permissions->find($id);
        if (!$perm) return false;
        $deleted = $this->permissions->delete($id);
        if ($deleted) $this->audit(null, null, 'permission_deleted', 'permission', $id, $perm->name);
        return $deleted;
    }

    // ── Scoped role assignment ────────────────────────────────────────────

    public function assignRole(string $userId, string $roleId, string $scopeType = 'global', ?string $scopeId = null, ?string $actorId = null, ?string $expiresAt = null): void
    {
        $expires = $expiresAt ? Carbon::parse($expiresAt) : null;
        DB::transaction(function () use ($userId, $roleId, $scopeType, $scopeId, $actorId, $expires) {
            $this->userRoles->assign($userId, $roleId, $scopeType, $scopeId, $actorId, $expires);
            $role = $this->roles->find($roleId);
            $this->audit($actorId, $userId, 'role_assigned', 'role', $roleId, $role?->name ?? $roleId, $scopeType, $scopeId);
        });
    }

    public function revokeRole(string $userId, string $roleId, string $scopeType = 'global', ?string $scopeId = null, ?string $actorId = null): bool
    {
        return DB::transaction(function () use ($userId, $roleId, $scopeType, $scopeId, $actorId) {
            $revoked = $this->userRoles->revoke($userId, $roleId, $scopeType, $scopeId);
            if ($revoked) {
                $role = $this->roles->find($roleId);
                $this->audit($actorId, $userId, 'role_revoked', 'role', $roleId, $role?->name ?? $roleId, $scopeType, $scopeId);
            }
            return $revoked;
        });
    }

    public function getUserRoles(string $userId, ?string $scopeType = null, ?string $scopeId = null): Collection {
        return $this->userRoles->activeForUser($userId, $scopeType, $scopeId);
    }

    // ── Scoped direct permission grant ────────────────────────────────────

    public function grantPermission(string $userId, string $permissionId, string $scopeType = 'global', ?string $scopeId = null, ?string $actorId = null, ?string $expiresAt = null): void
    {
        $expires = $expiresAt ? Carbon::parse($expiresAt) : null;
        DB::transaction(function () use ($userId, $permissionId, $scopeType, $scopeId, $actorId, $expires) {
            $this->userRoles->grantPermission($userId, $permissionId, $scopeType, $scopeId, $actorId, $expires);
            $perm = $this->permissions->find($permissionId);
            $this->audit($actorId, $userId, 'permission_granted', 'permission', $permissionId, $perm?->name ?? $permissionId, $scopeType, $scopeId);
        });
    }

    public function revokePermission(string $userId, string $permissionId, string $scopeType = 'global', ?string $scopeId = null, ?string $actorId = null): bool
    {
        return DB::transaction(function () use ($userId, $permissionId, $scopeType, $scopeId, $actorId) {
            $revoked = $this->userRoles->revokePermission($userId, $permissionId, $scopeType, $scopeId);
            if ($revoked) {
                $perm = $this->permissions->find($permissionId);
                $this->audit($actorId, $userId, 'permission_revoked', 'permission', $permissionId, $perm?->name ?? $permissionId, $scopeType, $scopeId);
            }
            return $revoked;
        });
    }

    public function getUserPermissions(string $userId, ?string $scopeType = null, ?string $scopeId = null): Collection {
        return $this->resolveAllPermissions($userId, $scopeType, $scopeId);
    }

    // ── Audit ─────────────────────────────────────────────────────────────

    public function getAuditLog(string $userId): Collection {
        return PermissionAuditLog::query()->where('target_user_id', $userId)->orderBy('created_at','desc')->get();
    }

    private function audit(?string $actorId, ?string $targetUserId, string $action, string $subjectType, ?string $subjectId, string $subjectName, ?string $scopeType = null, ?string $scopeId = null, array $metadata = []): void
    {
        PermissionAuditLog::create([
            'actor_id' => $actorId,
            'target_user_id' => $targetUserId,
            'action' => $action,
            'subject_type' => $subjectType,
            'subject_id' => $subjectId,
            'subject_name' => $subjectName,
            'scope_type' => $scopeType,
            'scope_id' => $scopeId,
            'metadata' => $metadata ?: null,
            'created_at' => Carbon::now(),
        ]);
    }
}
```

### `app/Services/OrganisationService.php`

```php
<?php
namespace App\Services;

use App\Repositories\Contracts\MemberRepositoryInterface;
use App\Repositories\Contracts\OrganisationRepositoryInterface;
use App\Repositories\Contracts\TeamRepositoryInterface;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;

class OrganisationService
{
    public function __construct(
        private readonly OrganisationRepositoryInterface $orgs,
        private readonly TeamRepositoryInterface $teams,
        private readonly MemberRepositoryInterface $members,
    ) {}

    public function listOrgs(): Collection { return $this->orgs->all(); }
    public function getOrg(string $id) { return $this->orgs->findWithTeams($id); }

    public function createOrg(array $data) {
        return $this->orgs->create($data);
    }

    public function updateOrg(string $id, array $data) {
        $fields = array_filter([
            'name' => $data['name'] ?? null,
            'description' => $data['description'] ?? null,
            'is_active' => $data['is_active'] ?? null,
        ], fn($v) => $v !== null);
        return $this->orgs->update($id, $fields);
    }

    public function deleteOrg(string $id): bool { return $this->orgs->delete($id); }

    // Teams
    public function listTeams(string $orgId): Collection { return $this->teams->listByOrg($orgId); }

    public function createTeam(string $orgId, array $data) {
        return $this->teams->create(array_merge($data, ['org_id' => $orgId]));
    }

    public function updateTeam(string $id, array $data) {
        $fields = array_filter([
            'name' => $data['name'] ?? null,
            'description' => $data['description'] ?? null,
            'is_active' => $data['is_active'] ?? null,
        ], fn($v) => $v !== null);
        return $this->teams->update($id, $fields);
    }

    public function deleteTeam(string $id): bool { return $this->teams->delete($id); }

    // Membership
    public function addOrgMember(string $orgId, string $userId, ?string $invitedBy = null): void {
        $this->members->addOrgMember($orgId, $userId, $invitedBy);
    }

    public function removeOrgMember(string $orgId, string $userId): bool {
        return $this->members->removeOrgMember($orgId, $userId);
    }

    public function addTeamMember(string $teamId, string $userId, ?string $invitedBy = null): void {
        $this->members->addTeamMember($teamId, $userId, $invitedBy);
    }

    public function removeTeamMember(string $teamId, string $userId): bool {
        return $this->members->removeTeamMember($teamId, $userId);
    }
}
```

---

## 6. Service provider

Append to `$repositoryBindings` in `app/Providers/RepositoryServiceProvider.php`:

```php
\App\Repositories\Contracts\OrganisationRepositoryInterface::class => \App\Repositories\Eloquent\OrganisationRepository::class,
\App\Repositories\Contracts\TeamRepositoryInterface::class => \App\Repositories\Eloquent\TeamRepository::class,
\App\Repositories\Contracts\RoleRepositoryInterface::class => \App\Repositories\Eloquent\RoleRepository::class,
\App\Repositories\Contracts\PermissionRepositoryInterface::class => \App\Repositories\Eloquent\PermissionRepository::class,
```

Add manual bindings for the two non-base-repository classes inside `register()`, after the loop:

```php
$this->app->bind(
    \App\Repositories\Contracts\UserRoleRepositoryInterface::class,
    \App\Repositories\Eloquent\UserRoleRepository::class
);
$this->app->bind(
    \App\Repositories\Contracts\MemberRepositoryInterface::class,
    \App\Repositories\Eloquent\MemberRepository::class
);
```

---

## 7. Middleware — `app/Http/Middleware/RequiresScopedPermission.php`

```php
<?php
namespace App\Http\Middleware;

use App\Services\ScopedPermissionService;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class RequiresScopedPermission
{
    public function __construct(private readonly ScopedPermissionService $service) {}

    /**
     * Usage examples:
     *   ->middleware('spermission:profiles:delete')            — global check
     *   ->middleware('spermission:profiles:delete,org')        — org-scoped (reads {org} route param)
     *   ->middleware('spermission:profiles:delete,team')       — team-scoped (reads {team} route param)
     *
     * Route parameters must be named {org} and {team} respectively.
     * All listed permissions use AND logic.
     */
    public function handle(Request $request, Closure $next, string ...$args): Response
    {
        $permissions = array_filter($args, fn($a) => str_contains($a, ':'));
        $scopeHint   = collect($args)->first(fn($a) => in_array($a, ['org','team']));

        $scopeType = null;
        $scopeId   = null;

        if ($scopeHint === 'org' && $request->route('org')) {
            $scopeType = 'org';
            $scopeId   = $request->route('org');
        } elseif ($scopeHint === 'team' && $request->route('team')) {
            $scopeType = 'team';
            $scopeId   = $request->route('team');
        }

        $userId = $request->route('user') ?? $request->input('user_id');
        if (!$userId) return response()->json(['message' => 'Unauthenticated'], 401);

        foreach ($permissions as $permission) {
            if (!$this->service->userCan($userId, trim($permission), $scopeType, $scopeId)) {
                return response()->json(['message' => 'Forbidden', 'required' => $permission, 'scope' => $scopeType], 403);
            }
        }

        return $next($request);
    }
}
```

Register in `bootstrap/app.php`:

```php
$middleware->alias(['spermission' => \App\Http\Middleware\RequiresScopedPermission::class]);
```

---

## 8. Form requests — `app/Http/Requests/Rbac/`

```php
// StoreOrganisationRequest.php
<?php
namespace App\Http\Requests\Rbac;
use Illuminate\Foundation\Http\FormRequest;
class StoreOrganisationRequest extends FormRequest
{
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'name' => ['required','string','max:120'],
            'slug' => ['required','string','max:80','unique:organisations,slug','regex:/^[a-z0-9\-]+$/'],
            'description' => ['nullable','string'],
            'is_active' => ['nullable','boolean'],
        ];
    }
}
```

```php
// StoreTeamRequest.php
<?php
namespace App\Http\Requests\Rbac;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
class StoreTeamRequest extends FormRequest
{
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'name' => ['required','string','max:120'],
            'slug' => ['required','string','max:80','regex:/^[a-z0-9\-]+$/',
                Rule::unique('teams')->where('org_id', $this->route('org'))],
            'description' => ['nullable','string'],
            'is_active' => ['nullable','boolean'],
        ];
    }
}
```

```php
// StoreRoleRequest.php
<?php
namespace App\Http\Requests\Rbac;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
class StoreRoleRequest extends FormRequest
{
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'name' => ['required','string','max:80','unique:roles,name','regex:/^[a-z0-9_]+$/'],
            'display_name' => ['required','string','max:120'],
            'description' => ['nullable','string'],
            'is_system' => ['nullable','boolean'],
            'scope_hint' => ['nullable', Rule::in(['global','org','team'])],
            'permission_ids' => ['nullable','array'],
            'permission_ids.*' => ['uuid','exists:permissions,permission_id'],
        ];
    }
}
```

```php
// StorePermissionRequest.php
<?php
namespace App\Http\Requests\Rbac;
use Illuminate\Foundation\Http\FormRequest;
class StorePermissionRequest extends FormRequest
{
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'resource' => ['required','string','max:80','regex:/^[a-z0-9_]+$/'],
            'action' => ['required','string','max:80','regex:/^[a-z0-9_]+$/'],
            'display_name' => ['nullable','string','max:200'],
            'description' => ['nullable','string'],
        ];
    }
    public function withValidator($validator): void {
        $validator->after(function ($v) {
            if (\App\Models\Permission::query()->where('resource', $this->resource)->where('action', $this->action)->exists()) {
                $v->errors()->add('name', "Permission {$this->resource}:{$this->action} already exists.");
            }
        });
    }
}
```

```php
// AssignScopedRoleRequest.php
<?php
namespace App\Http\Requests\Rbac;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
class AssignScopedRoleRequest extends FormRequest
{
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'role_id' => ['required','uuid','exists:roles,role_id'],
            'scope_type' => ['required', Rule::in(['global','org','team'])],
            'scope_id' => ['nullable','uuid'],
            'actor_id' => ['nullable','uuid','exists:users,user_id'],
            'expires_at' => ['nullable','date','after:now'],
        ];
    }
}
```

```php
// GrantScopedPermissionRequest.php
<?php
namespace App\Http\Requests\Rbac;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
class GrantScopedPermissionRequest extends FormRequest
{
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'permission_id' => ['required','uuid','exists:permissions,permission_id'],
            'scope_type' => ['required', Rule::in(['global','org','team'])],
            'scope_id' => ['nullable','uuid'],
            'actor_id' => ['nullable','uuid','exists:users,user_id'],
            'expires_at' => ['nullable','date','after:now'],
        ];
    }
}
```

```php
// MemberRequest.php
<?php
namespace App\Http\Requests\Rbac;
use Illuminate\Foundation\Http\FormRequest;
class MemberRequest extends FormRequest
{
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'user_id' => ['required','uuid','exists:users,user_id'],
            'invited_by' => ['nullable','uuid','exists:users,user_id'],
        ];
    }
}
```

---

## 9. Resources — `app/Http/Resources/Rbac/`

```php
// PermissionResource.php
<?php
namespace App\Http\Resources\Rbac;
use Illuminate\Http\Resources\Json\JsonResource;
class PermissionResource extends JsonResource
{
    public function toArray($request): array {
        return ['permission_id'=>$this->permission_id,'resource'=>$this->resource,'action'=>$this->action,'name'=>$this->name,'display_name'=>$this->display_name,'description'=>$this->description];
    }
}
```

```php
// RoleResource.php
<?php
namespace App\Http\Resources\Rbac;
use Illuminate\Http\Resources\Json\JsonResource;
class RoleResource extends JsonResource
{
    public function toArray($request): array {
        return [
            'role_id'=>$this->role_id,'name'=>$this->name,'display_name'=>$this->display_name,
            'description'=>$this->description,'is_system'=>$this->is_system,'scope_hint'=>$this->scope_hint,
            'permissions'=>PermissionResource::collection($this->whenLoaded('permissions')),
            'parents'=>RoleResource::collection($this->whenLoaded('parents')),
        ];
    }
}
```

```php
// OrganisationResource.php
<?php
namespace App\Http\Resources\Rbac;
use Illuminate\Http\Resources\Json\JsonResource;
class OrganisationResource extends JsonResource
{
    public function toArray($request): array {
        return [
            'org_id'=>$this->org_id,'name'=>$this->name,'slug'=>$this->slug,
            'description'=>$this->description,'is_active'=>$this->is_active,
            'teams'=>TeamResource::collection($this->whenLoaded('teams')),
            'created_at'=>$this->created_at,
        ];
    }
}
```

```php
// TeamResource.php
<?php
namespace App\Http\Resources\Rbac;
use Illuminate\Http\Resources\Json\JsonResource;
class TeamResource extends JsonResource
{
    public function toArray($request): array {
        return [
            'team_id'=>$this->team_id,'org_id'=>$this->org_id,'name'=>$this->name,
            'slug'=>$this->slug,'description'=>$this->description,'is_active'=>$this->is_active,
            'created_at'=>$this->created_at,
        ];
    }
}
```

```php
// AuditLogResource.php
<?php
namespace App\Http\Resources\Rbac;
use Illuminate\Http\Resources\Json\JsonResource;
class AuditLogResource extends JsonResource
{
    public function toArray($request): array {
        return [
            'audit_id'=>$this->audit_id,'actor_id'=>$this->actor_id,'target_user_id'=>$this->target_user_id,
            'action'=>$this->action,'subject_type'=>$this->subject_type,'subject_id'=>$this->subject_id,
            'subject_name'=>$this->subject_name,'scope_type'=>$this->scope_type,'scope_id'=>$this->scope_id,
            'metadata'=>$this->metadata,'created_at'=>$this->created_at,
        ];
    }
}
```

---

## 10. Controllers — `app/Http/Controllers/Api/`

```php
// OrganisationController.php
<?php
namespace App\Http\Controllers\Api;
use App\Http\Controllers\Controller;
use App\Http\Requests\Rbac\MemberRequest;
use App\Http\Requests\Rbac\StoreOrganisationRequest;
use App\Http\Requests\Rbac\StoreTeamRequest;
use App\Http\Resources\Rbac\OrganisationResource;
use App\Http\Resources\Rbac\TeamResource;
use App\Services\OrganisationService;
use Illuminate\Http\JsonResponse;

class OrganisationController extends Controller
{
    public function __construct(private readonly OrganisationService $service) {}

    public function index(): JsonResponse { return response()->json(OrganisationResource::collection($this->service->listOrgs())); }
    public function show(string $org): JsonResponse {
        $o = $this->service->getOrg($org);
        if (!$o) return response()->json(['message' => 'Not found'], 404);
        return response()->json(new OrganisationResource($o));
    }
    public function store(StoreOrganisationRequest $request): JsonResponse {
        return response()->json(new OrganisationResource($this->service->createOrg($request->validated())), 201);
    }
    public function update(StoreOrganisationRequest $request, string $org): JsonResponse {
        return response()->json(new OrganisationResource($this->service->updateOrg($org, $request->validated())));
    }
    public function destroy(string $org): JsonResponse {
        return response()->json(['deleted' => $this->service->deleteOrg($org)]);
    }

    // Teams
    public function teams(string $org): JsonResponse { return response()->json(TeamResource::collection($this->service->listTeams($org))); }
    public function storeTeam(StoreTeamRequest $request, string $org): JsonResponse {
        return response()->json(new TeamResource($this->service->createTeam($org, $request->validated())), 201);
    }
    public function updateTeam(StoreTeamRequest $request, string $org, string $team): JsonResponse {
        return response()->json(new TeamResource($this->service->updateTeam($team, $request->validated())));
    }
    public function destroyTeam(string $org, string $team): JsonResponse {
        return response()->json(['deleted' => $this->service->deleteTeam($team)]);
    }

    // Members
    public function addOrgMember(MemberRequest $request, string $org): JsonResponse {
        $v = $request->validated();
        $this->service->addOrgMember($org, $v['user_id'], $v['invited_by'] ?? null);
        return response()->json(['message' => 'Member added'], 201);
    }
    public function removeOrgMember(string $org, string $user): JsonResponse {
        return response()->json(['removed' => $this->service->removeOrgMember($org, $user)]);
    }
    public function addTeamMember(MemberRequest $request, string $org, string $team): JsonResponse {
        $v = $request->validated();
        $this->service->addTeamMember($team, $v['user_id'], $v['invited_by'] ?? null);
        return response()->json(['message' => 'Member added'], 201);
    }
    public function removeTeamMember(string $org, string $team, string $user): JsonResponse {
        return response()->json(['removed' => $this->service->removeTeamMember($team, $user)]);
    }
}
```

```php
// RoleController.php
<?php
namespace App\Http\Controllers\Api;
use App\Http\Controllers\Controller;
use App\Http\Requests\Rbac\StoreRoleRequest;
use App\Http\Resources\Rbac\RoleResource;
use App\Services\ScopedPermissionService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class RoleController extends Controller
{
    public function __construct(private readonly ScopedPermissionService $service) {}

    public function index(): JsonResponse { return response()->json(RoleResource::collection($this->service->listRoles())); }
    public function show(string $role): JsonResponse {
        $r = $this->service->getRole($role);
        if (!$r) return response()->json(['message' => 'Not found'], 404);
        return response()->json(new RoleResource($r));
    }
    public function store(StoreRoleRequest $request): JsonResponse {
        return response()->json(new RoleResource($this->service->createRole($request->validated())), 201);
    }
    public function update(StoreRoleRequest $request, string $role): JsonResponse {
        return response()->json(new RoleResource($this->service->updateRole($role, $request->validated())));
    }
    public function destroy(string $role): JsonResponse {
        return response()->json(['deleted' => $this->service->deleteRole($role)]);
    }
    public function addParent(Request $request, string $role): JsonResponse {
        $request->validate(['parent_role_id' => ['required','uuid','exists:roles,role_id']]);
        $this->service->addRoleParent($role, $request->input('parent_role_id'));
        return response()->json(['message' => 'Parent added']);
    }
    public function removeParent(string $role, string $parent): JsonResponse {
        $this->service->removeRoleParent($role, $parent);
        return response()->json(['message' => 'Parent removed']);
    }
}
```

```php
// PermissionController.php
<?php
namespace App\Http\Controllers\Api;
use App\Http\Controllers\Controller;
use App\Http\Requests\Rbac\StorePermissionRequest;
use App\Http\Resources\Rbac\PermissionResource;
use App\Services\ScopedPermissionService;
use Illuminate\Http\JsonResponse;

class PermissionController extends Controller
{
    public function __construct(private readonly ScopedPermissionService $service) {}

    public function index(): JsonResponse { return response()->json(PermissionResource::collection($this->service->listPermissions())); }
    public function show(string $permission): JsonResponse {
        $p = $this->service->getPermission($permission);
        if (!$p) return response()->json(['message' => 'Not found'], 404);
        return response()->json(new PermissionResource($p));
    }
    public function store(StorePermissionRequest $request): JsonResponse {
        return response()->json(new PermissionResource($this->service->createPermission($request->validated())), 201);
    }
    public function destroy(string $permission): JsonResponse {
        return response()->json(['deleted' => $this->service->deletePermission($permission)]);
    }
}
```

```php
// UserRbacController.php
<?php
namespace App\Http\Controllers\Api;
use App\Http\Controllers\Controller;
use App\Http\Requests\Rbac\AssignScopedRoleRequest;
use App\Http\Requests\Rbac\GrantScopedPermissionRequest;
use App\Http\Resources\Rbac\AuditLogResource;
use App\Http\Resources\Rbac\PermissionResource;
use App\Http\Resources\Rbac\RoleResource;
use App\Services\ScopedPermissionService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class UserRbacController extends Controller
{
    public function __construct(private readonly ScopedPermissionService $service) {}

    public function roles(Request $request, string $user): JsonResponse {
        $roles = $this->service->getUserRoles($user, $request->query('scope_type'), $request->query('scope_id'));
        return response()->json(RoleResource::collection($roles));
    }

    public function assignRole(AssignScopedRoleRequest $request, string $user): JsonResponse {
        $v = $request->validated();
        $this->service->assignRole($user, $v['role_id'], $v['scope_type'], $v['scope_id'] ?? null, $v['actor_id'] ?? null, $v['expires_at'] ?? null);
        return response()->json(['message' => 'Role assigned'], 201);
    }

    public function revokeRole(Request $request, string $user, string $role): JsonResponse {
        $request->validate(['scope_type' => ['required','in:global,org,team'], 'scope_id' => ['nullable','uuid']]);
        $revoked = $this->service->revokeRole($user, $role, $request->input('scope_type'), $request->input('scope_id'));
        return response()->json(['revoked' => $revoked]);
    }

    public function permissions(Request $request, string $user): JsonResponse {
        $perms = $this->service->getUserPermissions($user, $request->query('scope_type'), $request->query('scope_id'));
        return response()->json(PermissionResource::collection($perms));
    }

    public function grantPermission(GrantScopedPermissionRequest $request, string $user): JsonResponse {
        $v = $request->validated();
        $this->service->grantPermission($user, $v['permission_id'], $v['scope_type'], $v['scope_id'] ?? null, $v['actor_id'] ?? null, $v['expires_at'] ?? null);
        return response()->json(['message' => 'Permission granted'], 201);
    }

    public function revokePermission(Request $request, string $user, string $permission): JsonResponse {
        $request->validate(['scope_type' => ['required','in:global,org,team'], 'scope_id' => ['nullable','uuid']]);
        $revoked = $this->service->revokePermission($user, $permission, $request->input('scope_type'), $request->input('scope_id'));
        return response()->json(['revoked' => $revoked]);
    }

    public function canDo(Request $request, string $user, string $permission): JsonResponse {
        $can = $this->service->userCan($user, $permission, $request->query('scope_type'), $request->query('scope_id'));
        return response()->json(['can' => $can, 'scope_type' => $request->query('scope_type'), 'scope_id' => $request->query('scope_id')]);
    }

    public function audit(string $user): JsonResponse {
        return response()->json(AuditLogResource::collection($this->service->getAuditLog($user)));
    }
}
```

---

## 11. Routes — append to `routes/api.php`

```php
use App\Http\Controllers\Api\OrganisationController;
use App\Http\Controllers\Api\PermissionController;
use App\Http\Controllers\Api\RoleController;
use App\Http\Controllers\Api\UserRbacController;

// Organisations + nested teams + members
Route::prefix('orgs')->group(function () {
    Route::get('/',          [OrganisationController::class, 'index']);
    Route::post('/',         [OrganisationController::class, 'store']);
    Route::get('/{org}',     [OrganisationController::class, 'show']);
    Route::patch('/{org}',   [OrganisationController::class, 'update']);
    Route::delete('/{org}',  [OrganisationController::class, 'destroy']);
    // Org members
    Route::post('/{org}/members',          [OrganisationController::class, 'addOrgMember']);
    Route::delete('/{org}/members/{user}', [OrganisationController::class, 'removeOrgMember']);
    // Teams
    Route::get('/{org}/teams',             [OrganisationController::class, 'teams']);
    Route::post('/{org}/teams',            [OrganisationController::class, 'storeTeam']);
    Route::patch('/{org}/teams/{team}',    [OrganisationController::class, 'updateTeam']);
    Route::delete('/{org}/teams/{team}',   [OrganisationController::class, 'destroyTeam']);
    // Team members
    Route::post('/{org}/teams/{team}/members',          [OrganisationController::class, 'addTeamMember']);
    Route::delete('/{org}/teams/{team}/members/{user}', [OrganisationController::class, 'removeTeamMember']);
});

// Roles
Route::prefix('roles')->group(function () {
    Route::get('/',          [RoleController::class, 'index']);
    Route::post('/',         [RoleController::class, 'store']);
    Route::get('/{role}',    [RoleController::class, 'show']);
    Route::patch('/{role}',  [RoleController::class, 'update']);
    Route::delete('/{role}', [RoleController::class, 'destroy']);
    Route::post('/{role}/parents',              [RoleController::class, 'addParent']);
    Route::delete('/{role}/parents/{parent}',   [RoleController::class, 'removeParent']);
});

// Permissions
Route::prefix('permissions')->group(function () {
    Route::get('/',               [PermissionController::class, 'index']);
    Route::post('/',              [PermissionController::class, 'store']);
    Route::get('/{permission}',   [PermissionController::class, 'show']);
    Route::delete('/{permission}',[PermissionController::class, 'destroy']);
});

// User RBAC (scoped roles, permissions, can-check, audit)
Route::prefix('profiles/{user}/rbac')->group(function () {
    Route::get('/roles',                       [UserRbacController::class, 'roles']);
    Route::post('/roles',                      [UserRbacController::class, 'assignRole']);
    Route::delete('/roles/{role}',             [UserRbacController::class, 'revokeRole']);
    Route::get('/permissions',                 [UserRbacController::class, 'permissions']);
    Route::post('/permissions',                [UserRbacController::class, 'grantPermission']);
    Route::delete('/permissions/{permission}', [UserRbacController::class, 'revokePermission']);
    Route::get('/can/{permission}',            [UserRbacController::class, 'canDo'])->where('permission','.*');
    Route::get('/audit',                       [UserRbacController::class, 'audit']);
});
```

---

## 12. Seeder — `database/seeders/RbacSeeder.php`

```php
<?php
namespace Database\Seeders;

use App\Models\Permission;
use App\Models\Role;
use Illuminate\Database\Seeder;

class RbacSeeder extends Seeder
{
    private array $matrix = [
        'profiles'      => ['read','create','update','delete','deactivate'],
        'addresses'     => ['read','create','update','delete'],
        'locations'     => ['read','create'],
        'sessions'      => ['read','revoke'],
        'roles'         => ['read','create','update','delete'],
        'permissions'   => ['read','create','delete'],
        'orgs'          => ['read','create','update','delete'],
        'teams'         => ['read','create','update','delete'],
        'members'       => ['add','remove'],
        'rbac'          => ['assign_role','revoke_role','grant_permission','revoke_permission','view_audit'],
    ];

    private array $roleDefs = [
        'viewer' => [
            'display_name' => 'Viewer', 'description' => 'Read-only access.', 'is_system' => true, 'scope_hint' => null,
            'perms' => ['profiles:read','addresses:read','locations:read','sessions:read','orgs:read','teams:read'],
            'inherits' => [],
        ],
        'member' => [
            'display_name' => 'Member', 'description' => 'Standard org/team member.', 'is_system' => true, 'scope_hint' => 'team',
            'perms' => ['profiles:update','addresses:create','locations:create'],
            'inherits' => ['viewer'],
        ],
        'moderator' => [
            'display_name' => 'Moderator', 'description' => 'Can manage profiles and addresses.', 'is_system' => true, 'scope_hint' => 'org',
            'perms' => ['profiles:deactivate','addresses:update','addresses:delete','sessions:revoke'],
            'inherits' => ['member'],
        ],
        'admin' => [
            'display_name' => 'Administrator', 'description' => 'Full scope management.', 'is_system' => true, 'scope_hint' => 'org',
            'perms' => ['profiles:create','profiles:delete','roles:read','permissions:read','orgs:update','teams:create','teams:update','teams:delete','members:add','members:remove','rbac:assign_role','rbac:revoke_role','rbac:grant_permission','rbac:revoke_permission','rbac:view_audit'],
            'inherits' => ['moderator'],
        ],
        'super_admin' => [
            'display_name' => 'Super Administrator', 'description' => 'Bypasses all checks globally.', 'is_system' => true, 'scope_hint' => 'global',
            'perms' => [], // receives ALL permissions below
            'inherits' => ['admin'],
        ],
    ];

    public function run(): void
    {
        $permMap = [];
        foreach ($this->matrix as $resource => $actions) {
            foreach ($actions as $action) {
                $perm = Permission::firstOrCreate(
                    ['resource' => $resource, 'action' => $action],
                    ['name' => "{$resource}:{$action}", 'display_name' => ucfirst($resource).' '.ucfirst(str_replace('_',' ',$action))]
                );
                $permMap["{$resource}:{$action}"] = $perm->permission_id;
            }
        }

        $roleMap = [];
        foreach ($this->roleDefs as $name => $def) {
            $role = Role::firstOrCreate(
                ['name' => $name],
                ['display_name' => $def['display_name'], 'description' => $def['description'], 'is_system' => $def['is_system'], 'scope_hint' => $def['scope_hint']]
            );
            $roleMap[$name] = $role;
        }

        foreach ($this->roleDefs as $name => $def) {
            $ids = $name === 'super_admin'
                ? array_values($permMap)
                : array_values(array_filter(array_map(fn($p) => $permMap[$p] ?? null, $def['perms'])));
            $roleMap[$name]->permissions()->sync($ids);
        }

        foreach ($this->roleDefs as $name => $def) {
            foreach ($def['inherits'] as $parentName) {
                $roleMap[$name]->parents()->syncWithoutDetaching([$roleMap[$parentName]->role_id]);
            }
        }
    }
}
```

Register in `DatabaseSeeder.php`: `$this->call(RbacSeeder::class);`

---

## 13. Verification

```bash
php artisan migrate
php artisan db:seed --class=RbacSeeder
php artisan route:list --path=orgs
php artisan route:list --path=roles
php artisan route:list --path=permissions
php artisan route:list --path=rbac
```

Expected: 14 org routes, 7 role routes, 4 permission routes, 8 user-rbac routes = **33 total new routes**.

### Smoke test sequence

```bash
BASE=http://127.0.0.1:8000/api

# Create user
USER=$(curl -s -X POST $BASE/profiles -H 'Content-Type: application/json' \
  -d '{"full_name":"Scoped User","username":"scoped_user_1"}' | jq -r '.user_id')

# Create org
ORG=$(curl -s -X POST $BASE/orgs -H 'Content-Type: application/json' \
  -d '{"name":"Acme Corp","slug":"acme-corp"}' | jq -r '.org_id')

# Create team inside org
TEAM=$(curl -s -X POST $BASE/orgs/$ORG/teams -H 'Content-Type: application/json' \
  -d '{"name":"Engineering","slug":"engineering"}' | jq -r '.team_id')

# Add user as org member
curl -s -X POST $BASE/orgs/$ORG/members -H 'Content-Type: application/json' \
  -d "{\"user_id\":\"$USER\"}" | jq

# Get admin role id
ADMIN_ID=$(curl -s $BASE/roles | jq -r '.[] | select(.name=="admin") | .role_id')

# Assign admin role at org scope
curl -s -X POST $BASE/profiles/$USER/rbac/roles -H 'Content-Type: application/json' \
  -d "{\"role_id\":\"$ADMIN_ID\",\"scope_type\":\"org\",\"scope_id\":\"$ORG\"}" | jq

# Check permission globally — should be false (no global role)
curl -s "$BASE/profiles/$USER/rbac/can/profiles:delete" | jq

# Check permission with org scope — should be true (admin in org = can delete)
curl -s "$BASE/profiles/$USER/rbac/can/profiles:delete?scope_type=org&scope_id=$ORG" | jq

# Check permission with team scope — should also be true (org cascades to team)
curl -s "$BASE/profiles/$USER/rbac/can/profiles:delete?scope_type=team&scope_id=$TEAM" | jq

# View scoped permissions
curl -s "$BASE/profiles/$USER/rbac/permissions?scope_type=org&scope_id=$ORG" | jq '[.[] | .name]'

# Audit log
curl -s "$BASE/profiles/$USER/rbac/audit" | jq '[.[] | {action, subject_name, scope_type, scope_id}]'
```

Expected: global check = `false`, org-scoped check = `true`, team-scoped check = `true` (via org cascade).

---

## Constraints

- No Spatie or other RBAC packages.
- Do not modify any existing migration, model, or repository except the two `HasMany` appends to `User.php`.
- `is_system` roles return `false` from `deleteRole()` silently — never throw.
- `userCan()` never throws — always returns `bool`.
- `permission_audit_log` is append-only: no `updated_at`, no delete endpoint.
- `scope_hint` on roles is advisory only — it is NOT enforced as a DB constraint.
- Middleware `spermission` reads `{org}` and `{team}` route params by name — routes must use those exact parameter names for scoped gates to work.
- `collectRoles()` uses a raw `DB::table('teams')` lookup to find `org_id` from a team — this avoids injecting `TeamRepository` into `ScopedPermissionService` and keeps the service dependency list lean.
