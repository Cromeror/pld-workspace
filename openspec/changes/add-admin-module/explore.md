# Exploration: Add Admin Module to auth-users

**Change:** `add-admin-module`  
**Date:** 2026-04-21  
**Status:** Complete

---

## Context

The user wants to add an `admin` module to the `auth-users` NestJS application, exposing functionality under the `/admin/*` path prefix. This exploration maps the existing structure and conventions of the service, identifies how roles/permissions currently work, and proposes scoping and approach options.

---

## Current auth-users Structure

### Application root

`apps/auth-users/src/auth-users.module.ts`

The root `AppModule` wires everything together. Notable facts:
- **No sub-modules** for most features: controllers and adapters are registered directly in `AppModule` (only `CatalogsModule` and the external `ParticipantsModule` lib are separate NestJS modules).
- Domain logic is exposed through two port tokens injected via factory: `AUTH_PORT` (`AuthPort`) and `USERS_PORT` (`UsersPort`), both produced by `createAuthUsersDomain()` from `packages/domain-auth-users`.
- JWT is configured inline with `JwtModule.registerAsync` + `PassportModule`.
- `HttpErrorInterceptor` is a global interceptor registered as a provider.

### Folder tree

```
apps/auth-users/src/
├── auth/
│   ├── auth.controller.ts          — POST /auth/login (no guard, public)
│   └── dto/login.dto.ts
├── catalogs/
│   ├── catalogs.module.ts          — Standalone NestJS module (only example)
│   ├── catalogs.controller.ts      — GET /catalogs/beneficiario (no guard, public, @ApiTags)
│   └── catalogs.data.ts
├── configs/
│   └── common.configs.ts           — port=9001, prefixUrl='/pld-api/auth-users'
├── participants/
│   ├── persona-fisica/
│   │   ├── participants.controller.ts    — POST /participants-fisica/* (@UseGuards(JwtAuthGuard))
│   │   ├── participants.adapter.ts       — @Injectable(), wraps ParticipantsFisicaService
│   │   └── dto/create/persona-fisica.dto.ts
│   ├── persona-moral/
│   │   ├── participants.controller.ts    — POST /participants-moral/* (@UseGuards(JwtAuthGuard))
│   │   └── participant.adapter.ts
│   ├── fideicomiso/
│   │   ├── fideicomiso.controller.ts     — POST /fideicomiso/* (@UseGuards(JwtAuthGuard))
│   │   └── fideicomiso.adapter.ts
│   └── anexo-7/
│       ├── anexo-7.controller.ts         — POST /anexo-7/* (@UseGuards(JwtAuthGuard))
│       └── anexo-7.adapter.ts
├── shared/
│   ├── auth/
│   │   ├── jwt-auth.guard.ts       — extends AuthGuard('jwt'), pure pass-through
│   │   ├── jwt.strategy.ts         — validates JWT, injects {id, email, role} into request.user
│   │   └── current-user.decorator.ts — @CurrentUser() param decorator (defined, not yet widely used)
│   └── http-error.interceptor.ts   — maps domain errors → uniform HTTP response
└── users/
    ├── users.controller.ts         — POST /system-users/* (@UseGuards(JwtAuthGuard))
    └── dto/create-user.dto.ts
```

### Naming conventions

| Concept | Convention |
|---------|-----------|
| Controller file | `<feature>.controller.ts` |
| Adapter file | `<feature>.adapter.ts` |
| DTO file | `<feature>.dto.ts` or `create-<feature>.dto.ts` / `update-<feature>.dto.ts` |
| Route prefix | kebab-case (`participants-fisica`, `sistema-users`, `anexo-7`) |
| Guard application | `@UseGuards(JwtAuthGuard)` at class level |
| Swagger tags | `@ApiTags('catalogs')` — only used in catalogs; inconsistently applied |
| `@ApiOperation` | Used in `anexo-7` controller for each method; not universal |

### Controller → Adapter → Service pattern (concrete examples)

**Example 1 — persona-fisica (adapter wraps lib service):**
- `participants.controller.ts` → injects `ParticipantsAdapter` (adapter)
- `participants.adapter.ts` → `@Injectable()`, injects `ParticipantsFisicaService` from `@pld-api/participants` lib
- `ParticipantsFisicaService` (in `libs/participants`) contains the actual business logic

**Example 2 — users (adapter = port, no extra adapter class):**
- `users.controller.ts` → injects `USERS_PORT` token directly
- `USERS_PORT` resolves to `UsersAdapter` from `packages/domain-auth-users` (the domain adapter implements the port interface)
- No intermediate adapter class in the app layer — the port is the adapter here

**Example 3 — auth (direct port injection):**
- `auth.controller.ts` → injects `AUTH_PORT` token
- `AUTH_PORT` resolves to `AuthAdapter` from `packages/domain-auth-users`
- Calls `authPort.login()`, which calls `usersPort.getUserByEmail()` + `verifyPassword()` + `signToken()`

**Key observation:** There are two adapter patterns in use:
1. App-layer `@Injectable()` adapter wrapping a lib service (participants pattern)
2. Direct port injection from the domain package (users/auth pattern — newer, preferred after migration)

### JWT guard and @CurrentUser

- `JwtAuthGuard` (`shared/auth/jwt-auth.guard.ts`): simple `AuthGuard('jwt')` extension; applies to any controller via `@UseGuards(JwtAuthGuard)`.
- `JwtStrategy` (`shared/auth/jwt.strategy.ts`): reads JWT from `X-JWT` header (primary) or `Authorization: Bearer` header (fallback). Validates against `JWT_SECRET`. Injects `{ id, email, role }` as `request.user`.
- `@CurrentUser()` decorator (`shared/auth/current-user.decorator.ts`): defined and ready, but **not yet used in any controller** — controllers don't read the current user from the token yet.

---

## Domain Package: `packages/domain-auth-users`

Structure:
```
packages/domain-auth-users/src/
├── adapters/
│   ├── auth.adapter.ts    — AuthPort impl: login, verifyCredentials, issueToken
│   └── users.adapter.ts   — UsersPort impl: full CRUD + list already implemented
├── crypto/password.ts
├── entities/user.entity.ts
├── factory.ts             — createAuthUsersDomain() → { authPort, usersPort }
├── jwt/
│   ├── sign.ts
│   └── types.ts           — JwtPayload { sub, email, role }
└── ports/
    ├── auth.port.ts
    └── users.port.ts      — listUsers(), updateUser(), deleteUser() already defined
```

**Critical finding:** `UsersPort` already defines:
- `createUser(input)` — ✅ used
- `getUserByEmail(email)` — ✅ used internally
- `getUserById(id)` — ✅ defined, not exposed via HTTP
- `updateUser(id, patch)` — ✅ defined, **not exposed via HTTP yet**
- `deleteUser(id)` — ✅ defined (soft-delete), **not exposed via HTTP yet**
- `listUsers()` — ✅ defined, **not exposed via HTTP yet**

The domain package is pure (no `@nestjs/*` imports). Admin logic that is user-management-focused can **reuse the existing `UsersPort`** without any domain changes.

---

## Existing Admin/Role Concept

### UserRole enum (`packages/shared-types/src/enums.ts`)

```ts
export enum UserRole {
  SUPERADMIN = 'SUPERADMIN',
  NOTARIO     = 'NOTARIO',
  INMOBILIARIA = 'INMOBILIARIA',
  AUXILIAR    = 'AUXILIAR',
  USUARIO_INTERNO = 'USUARIO_INTERNO',
  USUARIO_EXTERNO = 'USUARIO_EXTERNO',
}
```

Re-exported via `libs/catalogs/src/lib/catalogs.types.ts` for backward compatibility.

**`SUPERADMIN` is the designated admin role.** The role is embedded in the JWT payload (`JwtPayload.role`), meaning after login the token already carries the role.

### Current admin enforcement: NONE

- **No `RolesGuard`** exists in the codebase.
- **No `@Roles()` decorator** exists.
- `JwtStrategy.validate()` puts `role` into `request.user`, but no guard reads it.
- `UsersController` (POST `/system-users/*`) is protected by JWT but **accessible to any authenticated role** — no role check.
- `@CurrentUser()` is defined but never called — no controller currently reads the authenticated user's role.

### `auth-profiles.types.ts` (legacy lib)

`libs/auth-profiles/src/lib/auth-profiles.types.ts` contains richer typing: `RegistroSuperadmin`, `PerfilActualizacion`, `LoginResult` with `roles: UserRole[]`. This lib is from the pre-migration era and is **not currently wired into any controller or service** in auth-users. It serves as a reference for domain vocabulary.

---

## Proposed Scope for First Slice

Based on existing domain capabilities (ports already implemented), PLD/AML domain context, and what the `UsersPort` already supports, the first slice should expose the following endpoints under `/admin`:

### `/admin/users` — User Management

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/users` | List all users |
| GET | `/admin/users/:id` | Get user by ID |
| POST | `/admin/users` | Create user (any role) |
| PATCH | `/admin/users/:id` | Update user data or role |
| DELETE | `/admin/users/:id` | Soft-delete user |
| PATCH | `/admin/users/:id/activate` | Enable/disable user (`activo` flag) |
| PATCH | `/admin/users/:id/password-reset` | Force password reset (generate new temp password) |

All backed by existing `UsersPort` methods. Only `activate` and `password-reset` need thin wrappers; all others are direct `usersPort.*` calls.

### Out of scope for first slice (future)
- `/admin/catalogs` CRUD — catalogs are static data files now; no persistence layer
- `/admin/participants` management — relies on `@pld-api/participants` lib which has its own complexity
- `/admin/audit-log` — no audit infrastructure exists yet
- Role assignment via separate endpoint — can be handled via PATCH update for now

---

## Approach Options

### Option A — Dedicated `admin/` folder in auth-users with an `AdminModule` (recommended)

**Structure:**
```
apps/auth-users/src/admin/
├── admin.module.ts
├── users/
│   ├── admin-users.controller.ts   — @Controller('admin/users')
│   ├── admin-users.adapter.ts      — thin adapter, calls UsersPort
│   └── dto/
│       ├── update-admin-user.dto.ts
│       └── activate-user.dto.ts
└── guards/
    └── roles.guard.ts              — @Injectable(), checks request.user.role === 'SUPERADMIN'
```

`AdminModule` is imported into `AppModule`. It imports `RolesGuard` and applies `@UseGuards(JwtAuthGuard, RolesGuard)` at controller level. A `@Roles('SUPERADMIN')` decorator (using `SetMetadata`) enforces access.

**Pros:**
- Clean, self-contained — mirrors the `CatalogsModule` pattern (the only self-contained module today)
- No pollution of existing modules
- `RolesGuard` + `@Roles()` decorator can be reused by future admin sub-modules
- Aligns with libs/ → packages/ migration: admin logic stays in app layer, domain stays pure
- Easiest to expand: add `admin/catalogs/`, `admin/participants/` subfolders later

**Cons:**
- Introduces a new module pattern (most controllers are in the flat AppModule, not sub-modules) — small inconsistency to document
- Requires creating a `RolesGuard` and `@Roles()` decorator (small effort, ~30 lines total)

**Effort:** ~S (2-3 hours including guard + decorator + controller + adapter + DTOs for users)

**Migration impact:** Minimal. Admin stays in app layer. The domain package (`domain-auth-users`) does not change — `UsersPort` already supports all needed operations.

---

### Option B — Shared admin guard + existing modules expose `/admin/*` prefixed routes

**Concept:** Add `RolesGuard` globally or per-controller, then add `@Roles('SUPERADMIN')` + a parallel `@Controller('admin/...')` route set inside each existing controller or as separate controller classes.

**Pros:**
- No new module; everything stays flat in AppModule
- Reuses existing adapter instances

**Cons:**
- Existing controllers grow with admin routes or require duplication (two controllers for the same feature)
- No clean separation — admin-only routes mixed with regular routes in the same file
- Harder to audit security surface: which routes are admin-only is implicit
- Harder to add cross-cutting admin concerns (response shaping, pagination)

**Effort:** Similar to A, but worse long-term maintainability

**Migration impact:** Same as A

---

### Option C — Global prefix via a dedicated `AdminApp` or API gateway layer

**Concept:** Spin up a separate NestJS app (`apps/admin`) that proxies or duplicates the admin endpoints, using the same `domain-auth-users` package.

**Pros:**
- Total isolation; different port/process
- Can have different rate limiting, auth config, etc.

**Cons:**
- Heavy overhead for what is essentially a sub-feature of auth-users
- Doubles the deployment surface
- Overkill for current stage of the project
- Conflicts with the request to "follow the structure of auth-users service"

**Effort:** L (4-6 hours just for scaffolding, plus ongoing duplication)

---

## Recommended Approach

**Option A — `admin/` subfolder with `AdminModule`**

Rationale:
1. **Minimal change, maximum clarity**: a dedicated folder makes the admin surface immediately visible and auditable.
2. **Reuses existing domain**: `UsersPort` already implements every needed operation — no domain changes required.
3. **Establishes the right primitives**: `RolesGuard` + `@Roles()` are small, reusable, and needed regardless of approach.
4. **Consistent with the existing `CatalogsModule` pattern**: the project already has one self-contained module — `AdminModule` follows the same shape.
5. **Migration-safe**: admin code stays in the app layer; `packages/domain-auth-users` remains pure TypeScript.
6. **First slice is small**: 1 controller, 1 adapter, 1 guard, 1 decorator, ~3 DTOs. Can be done in a single PR.

---

## Open Questions for the User

1. **Who creates SUPERADMIN users?** The current `POST /system-users/*` allows any JWT holder to create users of any role. Should admin user creation (SUPERADMIN role assignment) be restricted to the `/admin` module only, or remain as-is?

2. **Password reset behavior:** Should `PATCH /admin/users/:id/password-reset` generate a new temporary password (returned in the response, same pattern as `createUser`) or trigger an email? There is a `mail.service.ts` in `libs/core` — is it wired up?

3. **Activate/deactivate:** `UserEntity` has an `activo: boolean` column. Should activate/deactivate be a dedicated endpoint (`PATCH /admin/users/:id/activate`) or just part of the general `PATCH /admin/users/:id` update?

4. **Pagination:** `UsersPort.listUsers()` returns all users with no pagination. Is that acceptable for the first slice, or should the admin endpoint accept `?page=&limit=` parameters?

5. **`@ApiTags` and Swagger conventions:** Currently only `catalogs` has `@ApiTags`. Should admin endpoints have `@ApiTags('admin')` globally, or per-resource (e.g., `@ApiTags('admin/users')`)? Should Swagger `@ApiBearerAuth` be applied consistently?

6. **Role guard scope:** Should `RolesGuard` live in `shared/auth/` (alongside `JwtAuthGuard`) or in `admin/guards/`? The former makes it available to any module; the latter keeps admin concerns isolated.
