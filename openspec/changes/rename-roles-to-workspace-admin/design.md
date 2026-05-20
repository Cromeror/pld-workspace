# Technical Design — rename-roles-to-workspace-admin

**Status**: updated
**Created**: 2026-05-19
**Sub-repos**: pld-api + pld-web + docs

## 0. Resumen ejecutivo

Refactor de identificadores en dos ejes ortogonales:

1. **Rol de seguridad** (`UserRole`): se colapsan `NOTARY` y `REAL_ESTATE` en un único valor `WORKSPACE_ADMIN`. Afecta enum compartido, ENUM SQL `users.role`, guards (`@Roles(...)`), routing FE.
2. **Tipo de actividad del negocio** (campo `registration.user_role` → `registration.activity_type`): se renombra la columna y el campo TS (`userRole` → `activityType`). Los **valores de string `'NOTARY'` / `'REAL_ESTATE'` que viven en esa columna NO cambian** — siguen siendo identificadores de actividad vulnerable (Notario vs Inmobiliaria), no roles.

Tipo TS para esos valores: se reutiliza `ActivityType` (`'NOTARY' | 'REAL_ESTATE'`) renombrando el alias `ActivityType` → `ActivityType` en `pld-api/packages/domain-auth-users/src/jwt/types.ts` y `pld-web/src/types/auth.ts`. Donde el código FE actualmente tipa `userRole: UserRole`, pasa a `activityType: ActivityType`.

## 1. Cambios BE (pld-api)

### 1.1 Enum compartido

**`pld-api/packages/shared-types/src/enums.ts`** (líneas 1–6)

Antes:
```ts
export enum UserRole {
  SUPERADMIN = 'SUPERADMIN',
  NOTARY = 'NOTARY',
  REAL_ESTATE = 'REAL_ESTATE',
  AUXILIARY = 'AUXILIARY',
}
```

Después:
```ts
export enum UserRole {
  SUPERADMIN = 'SUPERADMIN',
  WORKSPACE_ADMIN = 'WORKSPACE_ADMIN',
  AUXILIARY = 'AUXILIARY',
}
```

### 1.2 Tipos JWT (domain-auth-users)

**`pld-api/packages/domain-auth-users/src/jwt/types.ts`**

Se renombra `ActivityType` → `ActivityType` y se actualiza el shape del payload JWT.

Antes:
```ts
export type UserRole = 'NOTARY' | 'REAL_ESTATE' | 'SUPERADMIN';
export type ActivityType = 'NOTARY' | 'REAL_ESTATE';
```

Después:
```ts
export type UserRole = 'WORKSPACE_ADMIN' | 'SUPERADMIN' | 'AUXILIARY';
export type ActivityType = 'NOTARY' | 'REAL_ESTATE';

export interface JwtWorkspace {
  id: string;
  activityType: ActivityType;
}

export interface JwtPayload {
  sub: string;
  email: string;
  role: UserRole;
  workspace?: JwtWorkspace; // presente solo cuando role === 'WORKSPACE_ADMIN'
}
```

`ActivityType` reemplaza a `ActivityType` en todos los archivos BE y FE. Actualizar todas las referencias en ambos repos.

Nota: agregamos `AUXILIARY` al tipo literal `UserRole` para alinear con el enum compartido (omisión previa; trivial y consistente — incluir).

#### JWT shape definitivo

El JWT emitido por `POST /auth/select-profile` (y en flujo de workspace único) lleva:

```json
{ "sub": "...", "email": "...", "role": "WORKSPACE_ADMIN", "workspace": { "id": "uuid", "activityType": "NOTARY" } }
```

- `workspaceId` plano **se elimina** del JWT — queda encapsulado en `workspace.id`.
- SUPERADMIN **no lleva** `workspace` (sin cambio).
- Archivos a actualizar por este shape:
  - `pld-api/packages/domain-auth-users/src/jwt/types.ts` (este archivo — new interfaces)
  - `pld-api/apps/auth-users/src/auth/auth.adapter.ts` — donde se firma el JWT
  - `pld-api/apps/auth-users/src/auth/jwt.strategy.ts` — donde se valida y extrae el payload
  - `pld-web/src/types/auth.ts` — JwtPayload shape en FE
  - `pld-web/src/components/layouts/AuthenticatedLayout.tsx` — usa `availableProfiles` y `workspaceId` del token

### 1.3 DTOs con `@IsIn([...])` / `@IsEnum([...])`

#### `pld-api/apps/auth-users/src/admin/registration/dto/create-registration.dto.ts`

El campo `userRole` representa **tipo de actividad**, no rol. Se renombra a `activityType` y se tipa con `ActivityType`.

Antes (líneas 17–25):
```ts
@ApiProperty({
    description: 'User role for the new account. Only NOTARY and REAL_ESTATE are allowed.',
    enum: [UserRole.NOTARY, UserRole.REAL_ESTATE],
    example: UserRole.NOTARY,
})
@IsIn([UserRole.NOTARY, UserRole.REAL_ESTATE], {
    message: 'userRole must be NOTARY or REAL_ESTATE',
})
userRole!: UserRole;
```

Después:
```ts
import type { ActivityType } from '@pld-api/domain-auth-users';
// ...
@ApiProperty({
    description: 'Vulnerable activity type of the reporting entity.',
    enum: ['NOTARY', 'REAL_ESTATE'],
    example: 'NOTARY',
})
@IsIn(['NOTARY', 'REAL_ESTATE'], {
    message: 'activityType must be NOTARY or REAL_ESTATE',
})
activityType!: ActivityType;
```

Eliminar el import de `UserRole` si ya no se usa (queda solo `ProfileType`).

#### `pld-api/apps/auth-users/src/auth/dto/select-profile.dto.ts` y `switch-workspace.dto.ts`

Renombrar el campo `profile` → `activityType` en ambos DTOs. El tipo pasa de comentario textual a `ActivityType` explícito.

Antes (ambos DTOs):
```ts
profile: string; // 'NOTARY' | 'REAL_ESTATE'
```

Después:
```ts
import type { ActivityType } from '@pld-api/domain-auth-users';
// ...
activityType: ActivityType;
```

Actualizar `@IsIn`, `@ApiProperty` y cualquier referencia al campo `profile` en los handlers que consuman estos DTOs.

#### `pld-api/apps/auth-users/src/users/dto/create-user.dto.ts`

Este DTO importa `UserRole` desde el legacy `@pld-api/catalogs` (línea 3). `libs/catalogs` **re-exporta desde `shared-types`** — no requiere cambios estructurales propios. Solo actualizar el ejemplo Swagger:

- `example: UserRole.NOTARY` (línea 30) → `example: UserRole.WORKSPACE_ADMIN`.

### 1.4 Guards / `@Roles(...)`

#### `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.controller.ts` (línea 28)

Antes: `@Roles(UserRole.NOTARY, UserRole.REAL_ESTATE)`
Después: `@Roles(UserRole.WORKSPACE_ADMIN)`

Comentarios de OpenAPI en líneas 35–47 (`NOTARY/REAL_ESTATE`) → `WORKSPACE_ADMIN`.

#### `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts` (líneas 17–20)

Antes:
```ts
const ALLOWED_PARENT_ROLES: ReadonlyArray<UserRole> = [
    UserRole.NOTARY,
    UserRole.REAL_ESTATE,
];
```

Después:
```ts
const ALLOWED_PARENT_ROLES: ReadonlyArray<UserRole> = [UserRole.WORKSPACE_ADMIN];
```

Actualizar comentario JSDoc (líneas 23–30) y mensaje de `ForbiddenException` (línea 47) — sustituir "NOTARY/REAL_ESTATE" por "WORKSPACE_ADMIN".

### 1.5 Entity TypeORM: rename de columna

#### `pld-api/apps/auth-users/src/admin/registration/entities/registration.entity.ts` (líneas 24–26)

Antes:
```ts
/** Validated in app layer against UserRole — stored as varchar(30) */
@Column({ name: 'user_role', type: 'varchar', length: 30 })
userRole!: UserRole;
```

Después:
```ts
import type { ActivityType } from '@pld-api/domain-auth-users';
// ...
/** Type of vulnerable activity — 'NOTARY' or 'REAL_ESTATE'. */
@Column({ name: 'activity_type', type: 'varchar', length: 30 })
activityType!: ActivityType;
```

### 1.6 Adapter y service del wizard de registro

#### `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts`

- Líneas 103–124 (`createRegistration`): cambiar parámetro `userRole: UserRole` → `activityType: ActivityType`, y `userRole: input.userRole` → `activityType: input.activityType`.
- Importar `ActivityType` desde `@pld-api/domain-auth-users`; eliminar `UserRole` si ya no se usa en otras funciones del adapter (probablemente sí — `createUser` sigue recibiendo `role: UserRole`).

#### `pld-api/apps/auth-users/src/admin/registration/registration.service.ts`

- Línea 110: `dto.userRole === 'NOTARY'` → `dto.activityType === 'NOTARY'`.
- Línea 132: `userRole: dto.userRole` → `activityType: dto.activityType`.
- Línea 494 (finalize → createUser): `role: registration.userRole` → `role: UserRole.WORKSPACE_ADMIN`. (Antes el role del user se derivaba del tipo de actividad; ahora el rol siempre es `WORKSPACE_ADMIN` para sujetos obligados.)
- Líneas 594–598 (linkSecondary): `primary.userRole` y `secondary.userRole` → `.activityType`. La regla "roles opuestos" sigue válida (un workspace registra NOTARY y otro REAL_ESTATE como actividades).

#### `pld-api/apps/auth-users/src/admin/registration/registration.service.spec.ts`

- Línea 24: `userRole: UserRole.NOTARY` → `activityType: 'NOTARY'` (cambiar tipado del fixture; importar `ActivityType` si hace falta o usar literal).

### 1.7 Migración SQL

**Nombre**: `20260519000000-rename-roles-to-workspace-admin.sql`
**Down**: `20260519000000-rename-roles-to-workspace-admin.down.sql`

Última migración previa: `20260514000000-auxiliary-profile-workspace.sql`. El timestamp 2026-05-19 respeta la secuencia.

#### UP

```sql
-- ================================================================
-- Migration: rename-roles-to-workspace-admin (UP)
-- Created: 2026-05-19
-- Description:
--   0. Delete non-SUPERADMIN users (no data to preserve in dev/staging)
--   1. Collapse NOTARY/REAL_ESTATE into WORKSPACE_ADMIN in users.role
--   2. Rename registration.user_role → registration.activity_type
-- ================================================================

USE `pld_api_bd`;

-- ----------------------------------------------------------------
-- 0. Purge non-SUPERADMIN users — no rollback needed (no data to preserve).
--    Must run BEFORE the ENUM ALTER to avoid constraint violations.
-- ----------------------------------------------------------------
DELETE FROM `users` WHERE `role` != 'SUPERADMIN';

-- ----------------------------------------------------------------
-- 1. Expand users.role ENUM to accept WORKSPACE_ADMIN alongside old values
-- ----------------------------------------------------------------
ALTER TABLE `users` MODIFY COLUMN `role` ENUM(
    'SUPERADMIN',
    'NOTARY',
    'REAL_ESTATE',
    'AUXILIARY',
    'WORKSPACE_ADMIN'
) NOT NULL;

-- ----------------------------------------------------------------
-- 2. Backfill: collapse NOTARY/REAL_ESTATE into WORKSPACE_ADMIN
--    (rows may still exist if DELETE above was skipped — safe no-op if empty)
-- ----------------------------------------------------------------
UPDATE `users`
SET `role` = 'WORKSPACE_ADMIN'
WHERE `role` IN ('NOTARY', 'REAL_ESTATE');

-- ----------------------------------------------------------------
-- 3. Restrict users.role ENUM to the new set
-- ----------------------------------------------------------------
ALTER TABLE `users` MODIFY COLUMN `role` ENUM(
    'SUPERADMIN',
    'WORKSPACE_ADMIN',
    'AUXILIARY'
) NOT NULL;

-- ----------------------------------------------------------------
-- 4. Rename registration.user_role → registration.activity_type
--    (values stay 'NOTARY' / 'REAL_ESTATE' — they identify the
--     vulnerable activity, not the security role).
-- ----------------------------------------------------------------
ALTER TABLE `registration`
    CHANGE COLUMN `user_role` `activity_type` VARCHAR(30) NOT NULL;
```

#### DOWN

```sql
-- ================================================================
-- Migration: rename-roles-to-workspace-admin (DOWN)
-- ================================================================
-- NOTE: The DELETE in step 0 of the UP migration is intentionally
-- non-reversible — there were no non-SUPERADMIN users to preserve.
-- This rollback only undoes the structural changes.
-- ================================================================

USE `pld_api_bd`;

-- 1. Rename activity_type back to user_role
ALTER TABLE `registration`
    CHANGE COLUMN `activity_type` `user_role` VARCHAR(30) NOT NULL;

-- 2. Expand users.role ENUM to accept both old + new
ALTER TABLE `users` MODIFY COLUMN `role` ENUM(
    'SUPERADMIN',
    'NOTARY',
    'REAL_ESTATE',
    'AUXILIARY',
    'WORKSPACE_ADMIN'
) NOT NULL;

-- 3. Restrict back to the legacy set
ALTER TABLE `users` MODIFY COLUMN `role` ENUM(
    'SUPERADMIN',
    'NOTARY',
    'REAL_ESTATE',
    'AUXILIARY'
) NOT NULL;
```

## 2. Cambios FE (pld-web)

### 2.1 Tipo `UserRole`

#### `pld-web/src/types/UserRole.ts` (líneas 3–8)

Antes:
```ts
export const UserRole = {
  SUPERADMIN: "SUPERADMIN",
  NOTARY: "NOTARY",
  REAL_ESTATE: "REAL_ESTATE",
  AUXILIARY: "AUXILIARY",
} as const;
```

Después:
```ts
export const UserRole = {
  SUPERADMIN: "SUPERADMIN",
  WORKSPACE_ADMIN: "WORKSPACE_ADMIN",
  AUXILIARY: "AUXILIARY",
} as const;
```

### 2.2 Routing y guards

#### `pld-web/src/config/postLoginRedirect.ts` (líneas 11–17)

Antes:
```ts
export const POST_LOGIN_REDIRECT_BY_ROLE: Partial<Record<UserRole, string>> = {
  [UserRole.SUPERADMIN]: RoutesUrl.REPORTING_ENTITY_REGISTER,
  [UserRole.NOTARY]: RoutesUrl.REGISTER,
  [UserRole.REAL_ESTATE]: RoutesUrl.REGISTER,
};
```

Después:
```ts
export const POST_LOGIN_REDIRECT_BY_ROLE: Partial<Record<UserRole, string>> = {
  [UserRole.SUPERADMIN]: RoutesUrl.REPORTING_ENTITY_REGISTER,
  [UserRole.WORKSPACE_ADMIN]: RoutesUrl.REGISTER,
};
```

#### `pld-web/src/routes/index.tsx` (líneas 69, 79)

Antes (en ambas):
```tsx
<RoleProtectedRoute requiredRoles={[UserRole.NOTARY, UserRole.REAL_ESTATE]}>
```

Después:
```tsx
<RoleProtectedRoute requiredRoles={[UserRole.WORKSPACE_ADMIN]}>
```

### 2.3 Tipos del wizard de registro (FE)

El campo `userRole` del wizard significa **tipo de actividad**, no rol. Se renombra a `activityType` y se tipa con `ActivityType`.

#### `pld-web/src/types/auth.ts`

Renombrar el alias existente:

Antes:
```ts
export type ObligatedProfile = "NOTARY" | "REAL_ESTATE";
```

Después:
```ts
export type ActivityType = "NOTARY" | "REAL_ESTATE";
```

Actualizar todas las referencias a `ObligatedProfile` en el mismo archivo (JwtPayload, etc.) y en los consumidores FE.

#### `pld-web/src/types/registration.ts`

- Línea 30: cambiar `import type { UserRole } from "./UserRole";` por `import type { ActivityType } from "./auth";`.
- Línea 36 (en `CreateRegistrationDto`): `userRole: UserRole;` → `activityType: ActivityType;`.
- Línea 122 (en `RegistrationDetail`): `userRole: UserRole;` → `activityType: ActivityType;`.
- Línea 152 (en `RegistrationState`): `userRole?: UserRole;` → `activityType?: ActivityType;`.
- Línea 163 (en `SecondActivityState`): `userRole: UserRole;` → `activityType: ActivityType;`.

### 2.4 Consumidores del campo

Búsqueda exhaustiva: `grep -rn "userRole" pld-web/src/` (ya hecho — ver lista en sección 4). Cada `state.userRole` / `dto.userRole` / `second.userRole` se renombra a `.activityType` y cada comparación contra `UserRole.NOTARY` / `UserRole.REAL_ESTATE` pasa a comparar contra los literales `'NOTARY'` / `'REAL_ESTATE'`.

Archivos:

- `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx` — 14 ocurrencias (líneas 38, 59, 93–94, 104, 107, 115, 149, 242, 244, 250, 258, 394–396, 450). Reemplazos:
  - `userRole` (campo) → `activityType` en todos los objetos y props.
  - `=== UserRole.NOTARY` → `=== 'NOTARY'`; `=== UserRole.REAL_ESTATE` → `=== 'REAL_ESTATE'`.
  - Inicialización `userRole: UserRole.NOTARY` (línea 94) → `activityType: 'NOTARY'`.
  - Prop `sourceUserRole` del modal (línea 450) → mantener nombre `sourceUserRole` o renombrar a `sourceActivityType` por coherencia (preferido: renombrar).
- `pld-web/src/components/organisms/reporting-entity/AddVulnerableActivityModal/index.tsx` — 7 ocurrencias (110, 147, 175, 285, 315, 322 y prop `sourceUserRole`). Aplicar mismo patrón.
- `pld-web/src/components/organisms/reporting-entity/ReviewStep/index.tsx` — líneas 135–143 y 221–228. Sustituir `userRole` → `activityType`; comparaciones contra `UserRole.NOTARY` → `'NOTARY'`.
- `pld-web/src/components/organisms/reporting-entity/ReportingEntityTypeStep/index.tsx` — líneas 9, 57, 74, 81. Prop `userRole` → `activityType` en la API del componente.
- `pld-web/src/components/organisms/reporting-entity/schemas.ts` — línea 9: `userRole: z.enum([UserRole.NOTARY, UserRole.REAL_ESTATE])` → `activityType: z.enum(['NOTARY', 'REAL_ESTATE'])`.

### 2.5 Cliente HTTP

Cualquier hook/query que arme el payload de `POST /admin/registration` debe pasar `activityType` en vez de `userRole`. Buscar en `pld-web/src/queries/` y `pld-web/src/api/` con `grep -rn "userRole" pld-web/src/queries pld-web/src/api 2>/dev/null` y aplicar el rename. (No hubo hits en el grep general anterior, pero verificar al implementar.)

### 2.6 JwtPayload en FE

`pld-web/src/types/auth.ts` debe actualizarse para reflejar el nuevo shape del JWT:

```ts
export type ActivityType = "NOTARY" | "REAL_ESTATE";

export interface JwtWorkspace {
  id: string;
  activityType: ActivityType;
}

export interface JwtPayload {
  sub: string;
  email: string;
  role: string;
  workspace?: JwtWorkspace; // presente solo cuando role === 'WORKSPACE_ADMIN'
}
```

- Eliminar el campo plano `workspaceId` del `JwtPayload` FE.
- Actualizar `AuthenticatedLayout.tsx` que actualmente lee `workspaceId` y `availableProfiles` del token — pasar a leer `workspace.id` y `workspace.activityType`.

### 2.7 GET /auth/me — nuevo shape de respuesta

`GET /auth/me` debe reflejar el nuevo shape cuando el usuario es `WORKSPACE_ADMIN`:

```json
{
  "sub": "...",
  "email": "...",
  "role": "WORKSPACE_ADMIN",
  "workspace": { "id": "uuid", "activityType": "NOTARY" }
}
```

- BE: actualizar el handler/adapter de `GET /auth/me` en `pld-api` para incluir `workspace: { id, activityType }` cuando el usuario tiene `role = WORKSPACE_ADMIN`.
- FE: actualizar el tipo de respuesta del hook que consume `GET /auth/me` para usar `JwtWorkspace` en lugar de `workspaceId` plano.

### 2.8 Wizard de registro — leer `activityType` del registration

Los componentes `ReportingEntityRegistrationPage` y `AddVulnerableActivityModal` actualmente leen el rol del usuario logueado para pre-seleccionar la actividad vulnerable. Deben cambiar a leer `registration.activityType` (que viene del BE como parte del objeto `registration`).

- `ReportingEntityRegistrationPage.tsx`: remover la lectura del rol del JWT para pre-selección; usar `registration.activityType` del query de registro activo.
- `AddVulnerableActivityModal/index.tsx`: misma corrección — prop `sourceActivityType` debe venir del objeto `registration`, no del token.

## 3. Cambios docs

Reemplazos donde `NOTARY` / `REAL_ESTATE` aparecen **como rol de usuario** → `WORKSPACE_ADMIN`. Donde aparecen **como tipo de actividad** (smoke tests, RFC fixtures, descripciones de "tipo de sujeto obligado") se mantienen.

Archivos identificados:

- `docs/flujos/README.md` (líneas 15, 18, 19): tabla de flujos — columna "Rol".
  - Reemplazar `NOTARY, REAL_ESTATE` por `WORKSPACE_ADMIN` (mantener `SUPERADMIN` aparte donde corresponde).
- `docs/flujos/registro-auxiliar/flujo.md`:
  - Línea 3: "ejecutado por un sujeto obligado (`NOTARY` o `REAL_ESTATE`)" → "ejecutado por un sujeto obligado (`WORKSPACE_ADMIN`)".
  - Línea 9: "Solo `NOTARY` o `REAL_ESTATE` pueden registrar auxiliares" → "Solo `WORKSPACE_ADMIN` puede registrar auxiliares".
  - Línea 41: "`role IN (NOTARY, REAL_ESTATE)`" → "`role = WORKSPACE_ADMIN`".
  - Línea 65: "`parent.role IN (NOTARY, REAL_ESTATE)`" → "`parent.role = WORKSPACE_ADMIN`".
- `docs/flujos/inicio-sesion/flujo.md`:
  - Línea 9: "Usuario (NOTARY / REAL_ESTATE)" → "Usuario (`WORKSPACE_ADMIN`)".
  - Línea 106: "máximo dos perfiles: `NOTARY` y `REAL_ESTATE`" — **se mantiene** (describe el `availableProfiles` que es `ActivityType`, no rol).
  - Línea 113: "`role` refleja el perfil seleccionado (`NOTARY` o `REAL_ESTATE`)" → "`role` es `WORKSPACE_ADMIN`; `activeProfile` indica el `ActivityType` seleccionado (`NOTARY` o `REAL_ESTATE`)" (revisar el contrato real del JWT — si solo lleva `role` + `workspaceId`, ajustar el wording).
  - Línea 114: detalles del select-profile flow — mantener referencias a `NOTARY` / `REAL_ESTATE` como `ActivityType`.
  - Línea 140 (tabla): mantener (es `availableProfiles`).
- `docs/flujos/registro-cliente/flujo.md` (línea 9): "Operador (NOTARY / REAL_ESTATE)" → "Operador (`WORKSPACE_ADMIN`)".
- `docs/flujos/registro-auxiliar/smoke.md`:
  - Líneas 24–25, 32, 81, 102, 104, 176: estos describen usuarios de prueba y tipo de actividad. Mantener nombres `pedro.notario` y `contacto@inmobiliaria-test.mx`. Los headers "Login NOTARY" pueden quedar como referencia al **tipo de actividad del usuario** (no a su rol). Documentar al inicio del smoke que `role = WORKSPACE_ADMIN` para ambos, y `activityType` es lo que los distingue.
- `docs/flujos/actividad-secundaria/flujo.md` (líneas 17, 30, 36–38): describe `sourceUserRole` como `NOTARY` / `REAL_ESTATE`. **Renombrar el concepto** a `sourceActivityType` para coherencia con BE/FE. Actualizar bloques `toon` también.
- `docs/test-users.md` (líneas 21, 31, 33, 47, 55, 58): tabla de usuarios. La columna "Tipo" puede separarse en "Rol" (`WORKSPACE_ADMIN`) y "Actividad" (`NOTARY` / `REAL_ESTATE`). Headers de sección como "## NOTARY — Persona física" → "## Notario — Persona física (`activityType=NOTARY`)" o similar.
- Otros `.md` no listados que aparezcan en `grep -rn "NOTARY\|REAL_ESTATE" docs/ | grep -v drawio`: revisar caso a caso. Heurística: si la frase habla de "rol", "permiso", "guard", "@Roles" → reemplazar por `WORKSPACE_ADMIN`. Si habla de "tipo de sujeto obligado", "actividad", "perfil" → mantener.

## 4. Orden de aplicación (secuencia segura)

Dos commits, el primero exclusivamente BE y migración para evitar inconsistencias en runtime:

### Commit 1 — BE + migración (en este orden estricto)

1. Escribir la migración SQL (`.sql` + `.down.sql`) sin aplicarla aún.
2. Actualizar `packages/shared-types/src/enums.ts` (UserRole enum).
3. Actualizar `packages/domain-auth-users/src/jwt/types.ts`: renombrar `ObligatedProfile` → `ActivityType`; actualizar `UserRole` literal; agregar interfaces `JwtWorkspace` y `JwtPayload`.
4. Actualizar `entities/registration.entity.ts` (column rename + tipo `ActivityType`).
5. Actualizar `registration.adapter.ts` (firma + asignación con `ActivityType`).
6. Actualizar `registration.service.ts` (4 ocurrencias: validación NOTARY-only-INDIVIDUAL, asignación al adapter, finalize createUser con `WORKSPACE_ADMIN`, linkSecondary).
7. Actualizar `registration.service.spec.ts` (fixture con `activityType`).
8. Actualizar `create-registration.dto.ts` (`userRole` → `activityType: ActivityType`).
9. Actualizar `auxiliaries.controller.ts` (`@Roles`) y `auxiliaries.service.ts` (`ALLOWED_PARENT_ROLES`).
10. Actualizar `users/dto/create-user.dto.ts` (example Swagger `NOTARY` → `WORKSPACE_ADMIN`).
11. `libs/catalogs`: re-exporta desde `shared-types` — sin cambios estructurales (solo verificar compile).
12. Actualizar `select-profile.dto.ts` y `switch-workspace.dto.ts`: renombrar campo `profile` → `activityType: ActivityType`.
13. Actualizar `auth.adapter.ts`: firmar JWT con shape `{ sub, email, role, workspace: { id, activityType } }` (eliminar `workspaceId` plano).
14. Actualizar `jwt.strategy.ts`: extraer `workspace.id` y `workspace.activityType` del payload; eliminar lectura de `workspaceId` plano.
15. Actualizar handler/adapter de `GET /auth/me`: responder con `workspace: { id, activityType }`.
16. Correr `nx run auth-users:build` y `nx run-many --target=test --all`. Si el ambiente local lo permite, aplicar la migración contra mysql dev.

### Commit 2 — FE + docs

1. Actualizar `src/types/UserRole.ts` (enum FE).
2. Actualizar `src/types/auth.ts`: renombrar `ObligatedProfile` → `ActivityType`; actualizar `JwtPayload` con `workspace?: JwtWorkspace` (eliminar `workspaceId` plano).
3. Actualizar `src/types/registration.ts` (4 campos `userRole` → `activityType: ActivityType`).
4. Actualizar `src/config/postLoginRedirect.ts` y `src/routes/index.tsx`.
5. Actualizar `AuthenticatedLayout.tsx`: leer `workspace.id` y `workspace.activityType` en lugar de `workspaceId` plano y `availableProfiles`.
6. Actualizar todos los consumidores listados en §2.4 (page, modal, ReviewStep, TypeStep, schemas) y §2.8 (leer `registration.activityType` para pre-selección).
7. Verificar `pld-web/src/queries/` y `pld-web/src/api/` por payloads con `userRole`.
8. Actualizar tipo de respuesta del hook de `GET /auth/me` en FE.
9. Actualizar docs según §3 (flujos, smoke, test-users).
10. Verificar con `tsc --noEmit` y `yarn build`. Smoke test manual en `/register` y `/register/auxiliary` post-login.

## 5. ADR (decisión arquitectónica)

### ADR: el "rol" y el "tipo de actividad" son dimensiones ortogonales

**Decisión**: separar el enum `UserRole` (rol de seguridad — qué puede hacer el usuario) del tipo de actividad vulnerable (qué tipo de negocio reporta — `NOTARY` / `REAL_ESTATE`). El primero vive en `UserRole` y en `users.role` (ENUM SQL). El segundo vive en `ActivityType` (type alias TS) y en `registration.activity_type` (VARCHAR).

**Por qué**:
- Permite agregar nuevos tipos de sujeto obligado (ej. casinos, joyerías) sin tocar el modelo de permisos.
- Permite que un mismo usuario tenga **dos workspaces de tipos distintos** (NOTARY + REAL_ESTATE) sin que su `role` cambie — solo el `workspaceId` activo y el `activeProfile` cambian.
- Evita la duplicación de `@Roles(UserRole.NOTARY, UserRole.REAL_ESTATE)` en cada controlador.
- Hace explícito en el esquema SQL que esos strings son "qué tipo de actividad reporto", no "qué permiso tengo".

**Implicaciones**:
- En el JWT, `role` es `WORKSPACE_ADMIN` para sujetos obligados. La distinción NOTARY/REAL_ESTATE se transporta en `workspace.activityType` del JWT.
- El campo plano `workspaceId` desaparece del JWT — queda en `workspace.id`.
- El select-profile flow sigue eligiendo entre `ActivityType`s, pero el DTO cambia `profile` → `activityType`.
- `GET /auth/me` responde con `workspace: { id, activityType }` para usuarios `WORKSPACE_ADMIN`.
- ADR file: archivar este bloque en `docs/decisiones/ADR-XXXX-roles-vs-activity-type.md` (next ADR number — verificar al implementar).

## 6. Riesgos remanentes

- **Ventana FE/BE entre commits**: si el FE aún manda `userRole` y el BE ya espera `activityType`, el wizard rompe. Mitigación: desplegar ambos en el mismo pipeline o coordinar el merge.
- **`libs/catalogs` legacy**: si define su propio `UserRole`, queda como referencia muerta a `NOTARY`/`REAL_ESTATE`. Tratamiento: actualizar en commit 1 o documentar como deuda técnica si la migración a `packages/` aún lo deprecará.
- **JWT en sesiones vivas**: usuarios con token vigente que tenga `role: 'NOTARY'` siguen pasando guards mientras el ENUM SQL acepte ambos valores (durante el deploy). Mitigación: forzar logout post-migración o revisar el `JwtStrategy.validate()` para mapear `'NOTARY'`/`'REAL_ESTATE'` → `'WORKSPACE_ADMIN'` durante una ventana de gracia (opcional, simple).
- **Down-migration**: no hay rollback lossy porque el DELETE previo elimina usuarios no-SUPERADMIN (no hay datos que preservar). El rollback DOWN solo deshace los cambios estructurales (ENUM + RENAME COLUMN).

## 7. Archivos tocados (checklist final)

### BE (pld-api)

- [ ] `packages/shared-types/src/enums.ts`
- [ ] `packages/domain-auth-users/src/jwt/types.ts` (rename `ObligatedProfile` → `ActivityType`; new `JwtWorkspace`, `JwtPayload`)
- [ ] `packages/persistence/migrations/20260519000000-rename-roles-to-workspace-admin.sql` (new)
- [ ] `packages/persistence/migrations/20260519000000-rename-roles-to-workspace-admin.down.sql` (new)
- [ ] `apps/auth-users/src/admin/registration/entities/registration.entity.ts`
- [ ] `apps/auth-users/src/admin/registration/registration.adapter.ts`
- [ ] `apps/auth-users/src/admin/registration/registration.service.ts`
- [ ] `apps/auth-users/src/admin/registration/registration.service.spec.ts`
- [ ] `apps/auth-users/src/admin/registration/dto/create-registration.dto.ts`
- [ ] `apps/auth-users/src/registration/auxiliaries/auxiliaries.controller.ts`
- [ ] `apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts`
- [ ] `apps/auth-users/src/users/dto/create-user.dto.ts` (example Swagger)
- [ ] `apps/auth-users/src/auth/dto/select-profile.dto.ts` (rename `profile` → `activityType: ActivityType`)
- [ ] `apps/auth-users/src/auth/dto/switch-workspace.dto.ts` (rename `profile` → `activityType: ActivityType`)
- [ ] `apps/auth-users/src/auth/auth.adapter.ts` (JWT shape: `workspace: { id, activityType }`)
- [ ] `apps/auth-users/src/auth/jwt.strategy.ts` (extraer `workspace.id` y `workspace.activityType`)
- [ ] `apps/auth-users/src/auth/[me handler]` (`GET /auth/me` → responder con `workspace: { id, activityType }`)

### FE (pld-web)

- [ ] `src/types/UserRole.ts`
- [ ] `src/types/auth.ts` (rename `ObligatedProfile` → `ActivityType`; actualizar `JwtPayload` con `workspace?: JwtWorkspace`)
- [ ] `src/types/registration.ts`
- [ ] `src/config/postLoginRedirect.ts`
- [ ] `src/routes/index.tsx`
- [ ] `src/components/layouts/AuthenticatedLayout.tsx` (leer `workspace.id` y `workspace.activityType`)
- [ ] `src/pages/admin/ReportingEntityRegistrationPage.tsx` (leer `registration.activityType` para pre-selección)
- [ ] `src/components/organisms/reporting-entity/ReportingEntityTypeStep/index.tsx`
- [ ] `src/components/organisms/reporting-entity/AddVulnerableActivityModal/index.tsx` (leer `registration.activityType`)
- [ ] `src/components/organisms/reporting-entity/ReviewStep/index.tsx`
- [ ] `src/components/organisms/reporting-entity/schemas.ts`
- [ ] `src/queries/**` y `src/api/**` (verificar payloads `userRole` y hook de `GET /auth/me`)

### docs

- [ ] `docs/flujos/README.md`
- [ ] `docs/flujos/registro-auxiliar/flujo.md`
- [ ] `docs/flujos/registro-auxiliar/smoke.md`
- [ ] `docs/flujos/inicio-sesion/flujo.md`
- [ ] `docs/flujos/registro-cliente/flujo.md`
- [ ] `docs/flujos/actividad-secundaria/flujo.md`
- [ ] `docs/test-users.md`
- [ ] `docs/decisiones/ADR-XXXX-roles-vs-activity-type.md` (new)
