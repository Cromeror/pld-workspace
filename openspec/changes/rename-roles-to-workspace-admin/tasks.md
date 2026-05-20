# Tasks: rename-roles-to-workspace-admin

**Change**: rename-roles-to-workspace-admin
**Generated**: 2026-05-19
**Updated**: 2026-05-19
**Deps**: spec.md + design.md

---

## BE (pld-api) — Commit 1

### Migración SQL

- [ ] Crear archivo UP `packages/persistence/migrations/20260519000000-rename-roles-to-workspace-admin.sql` con los 5 pasos: DELETE usuarios no-SUPERADMIN, expand ENUM, UPDATE backfill, restrict ENUM, RENAME COLUMN `user_role` → `activity_type` (pld-api/packages/persistence/migrations/)
- [ ] Crear archivo DOWN `packages/persistence/migrations/20260519000000-rename-roles-to-workspace-admin.down.sql` con rollback estructural: RENAME back + expand ENUM + restrict ENUM legacy (sin UPDATE de datos — no hay rollback lossy porque no hay datos que preservar) (pld-api/packages/persistence/migrations/)
- [ ] Aplicar migración UP contra mysql dev local y verificar que `users.role` no contiene `'NOTARY'`/`'REAL_ESTATE'`, que `registration.activity_type` existe con los valores previos intactos, y que no quedan usuarios no-SUPERADMIN

### Enum compartido

- [ ] Reemplazar `NOTARY = 'NOTARY'` y `REAL_ESTATE = 'REAL_ESTATE'` por `WORKSPACE_ADMIN = 'WORKSPACE_ADMIN'` en `UserRole` (pld-api/packages/shared-types/src/enums.ts:1-6)

### Tipos JWT y ActivityType

- [ ] Renombrar `ObligatedProfile` → `ActivityType` (`'NOTARY' | 'REAL_ESTATE'`) en `types.ts`; actualizar type literal `UserRole` de `'NOTARY' | 'REAL_ESTATE' | 'SUPERADMIN'` a `'WORKSPACE_ADMIN' | 'SUPERADMIN' | 'AUXILIARY'`; agregar interfaces `JwtWorkspace { id: string; activityType: ActivityType }` y `JwtPayload { sub, email, role, workspace?: JwtWorkspace }` (pld-api/packages/domain-auth-users/src/jwt/types.ts)

### Entity TypeORM

- [ ] Cambiar `@Column({ name: 'user_role' }) userRole!: UserRole` a `@Column({ name: 'activity_type' }) activityType!: ActivityType`; importar `ActivityType` desde `@pld-api/domain-auth-users`; eliminar import de `UserRole` si ya no se usa (pld-api/apps/auth-users/src/admin/registration/entities/registration.entity.ts:24-26)

### Adapter y Service del wizard de registro

- [ ] Renombrar parámetro `userRole: UserRole` → `activityType: ActivityType` y asignación `userRole: input.userRole` → `activityType: input.activityType` en `createRegistration`; importar `ActivityType`; eliminar import de `UserRole` si ya no se usa (pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts:103-124)
- [ ] En `registration.service.ts` línea 110: `dto.userRole === 'NOTARY'` → `dto.activityType === 'NOTARY'` (pld-api/apps/auth-users/src/admin/registration/registration.service.ts:110)
- [ ] En `registration.service.ts` línea 132: `userRole: dto.userRole` → `activityType: dto.activityType` (pld-api/apps/auth-users/src/admin/registration/registration.service.ts:132)
- [ ] En `registration.service.ts` línea 494 (finalize → createUser): `role: registration.userRole` → `role: UserRole.WORKSPACE_ADMIN` (pld-api/apps/auth-users/src/admin/registration/registration.service.ts:494)
- [ ] En `registration.service.ts` líneas 594-598 (linkSecondary): `primary.userRole` y `secondary.userRole` → `.activityType` (pld-api/apps/auth-users/src/admin/registration/registration.service.ts:594-598)

### DTO de creación de registro

- [ ] Renombrar campo `userRole!: UserRole` → `activityType!: ActivityType`; cambiar `@IsIn([UserRole.NOTARY, UserRole.REAL_ESTATE])` → `@IsIn(['NOTARY', 'REAL_ESTATE'])`; actualizar `@ApiProperty`; importar `ActivityType`; eliminar import de `UserRole` si ya no se usa (pld-api/apps/auth-users/src/admin/registration/dto/create-registration.dto.ts:17-25)

### Guards y controladores de auxiliares

- [ ] Cambiar `@Roles(UserRole.NOTARY, UserRole.REAL_ESTATE)` → `@Roles(UserRole.WORKSPACE_ADMIN)` y actualizar comentarios OpenAPI en las líneas 35-47 (pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.controller.ts:28)
- [ ] Cambiar `ALLOWED_PARENT_ROLES` de `[UserRole.NOTARY, UserRole.REAL_ESTATE]` → `[UserRole.WORKSPACE_ADMIN]`; actualizar JSDoc y mensaje de `ForbiddenException` (pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts:17-20)

### DTO de creación de usuario

- [ ] Actualizar el example Swagger de `UserRole.NOTARY` → `UserRole.WORKSPACE_ADMIN`; `libs/catalogs` re-exporta desde `shared-types` — sin cambios estructurales en el paquete (pld-api/apps/auth-users/src/users/dto/create-user.dto.ts:30)

### DTOs select-profile y switch-workspace

- [ ] Renombrar campo `profile` → `activityType` en `select-profile.dto.ts`; cambiar tipo a `ActivityType` (importar desde `@pld-api/domain-auth-users`); actualizar `@IsIn`, `@ApiProperty` y cualquier handler que consuma este campo (pld-api/apps/auth-users/src/auth/dto/select-profile.dto.ts)
- [ ] Renombrar campo `profile` → `activityType` en `switch-workspace.dto.ts`; mismo patrón que `select-profile.dto.ts` (pld-api/apps/auth-users/src/auth/dto/switch-workspace.dto.ts)

### JWT signing y strategy

- [ ] Actualizar `auth.adapter.ts` para firmar el JWT con el nuevo shape `{ sub, email, role, workspace: { id, activityType } }` cuando el usuario es `WORKSPACE_ADMIN`; SUPERADMIN sigue sin `workspace`; eliminar el campo plano `workspaceId` del payload (pld-api/apps/auth-users/src/auth/auth.adapter.ts)
- [ ] Actualizar `jwt.strategy.ts` para extraer `workspace.id` y `workspace.activityType` del payload validado; eliminar cualquier lectura del campo plano `workspaceId` (pld-api/apps/auth-users/src/auth/jwt.strategy.ts)

### GET /auth/me

- [ ] Actualizar el handler y/o adapter de `GET /auth/me` para que la respuesta incluya `workspace: { id, activityType }` cuando el usuario tiene `role = WORKSPACE_ADMIN`; SUPERADMIN no lleva `workspace` (buscar en `pld-api/apps/auth-users/src/auth/`)

### Spec de tests

- [ ] Cambiar fixture `userRole: UserRole.NOTARY` → `activityType: 'NOTARY'`; ajustar imports para eliminar `UserRole` si ya no se usa (pld-api/apps/auth-users/src/admin/registration/registration.service.spec.ts:24)

### Build y tests BE

- [ ] Ejecutar `nx run auth-users:build` y confirmar cero errores de compilación
- [ ] Ejecutar `nx run-many --target=test --all` y confirmar que todos los tests pasan

---

## FE (pld-web) — Commit 2

### Tipos y enums

- [ ] Eliminar `NOTARY` y `REAL_ESTATE` del objeto `UserRole`; agregar `WORKSPACE_ADMIN: "WORKSPACE_ADMIN"` (pld-web/src/types/UserRole.ts:3-8)
- [ ] Renombrar `ObligatedProfile` → `ActivityType` (`"NOTARY" | "REAL_ESTATE"`); actualizar `JwtPayload` para incluir `workspace?: JwtWorkspace` (con `id: string` y `activityType: ActivityType`) y eliminar `workspaceId` plano; actualizar todas las referencias a `ObligatedProfile` en el mismo archivo (pld-web/src/types/auth.ts)
- [ ] Cambiar los 4 campos `userRole` → `activityType` con tipo `ActivityType` en `CreateRegistrationDto`, `RegistrationDetail`, `RegistrationState` y `SecondActivityState`; actualizar import de `ObligatedProfile` → `ActivityType` (pld-web/src/types/registration.ts:30,36,122,152,163)

### Routing y guards

- [ ] Reemplazar las dos entradas `NOTARY` y `REAL_ESTATE` por una sola entrada `WORKSPACE_ADMIN: RoutesUrl.REGISTER` en `POST_LOGIN_REDIRECT_BY_ROLE` (pld-web/src/config/postLoginRedirect.ts:11-17)
- [ ] Cambiar las dos ocurrencias de `<RoleProtectedRoute requiredRoles={[UserRole.NOTARY, UserRole.REAL_ESTATE]}>` → `<RoleProtectedRoute requiredRoles={[UserRole.WORKSPACE_ADMIN]}>` (pld-web/src/routes/index.tsx:69,79)

### AuthenticatedLayout

- [ ] Actualizar `AuthenticatedLayout.tsx` para leer `workspace.id` en lugar de `workspaceId` plano del token y `workspace.activityType` en lugar de `availableProfiles`; ajustar toda la lógica que dependía del campo plano (pld-web/src/components/layouts/AuthenticatedLayout.tsx)

### Consumidores del campo en componentes y páginas

- [ ] Renombrar las 14 ocurrencias de `userRole` → `activityType` y `UserRole.NOTARY`/`UserRole.REAL_ESTATE` → `'NOTARY'`/`'REAL_ESTATE'`; renombrar prop `sourceUserRole` → `sourceActivityType`; cambiar la pre-selección de actividad para leer `registration.activityType` en lugar del rol del token (pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx:38,59,93-94,104,107,115,149,242,244,250,258,394-396,450)
- [ ] Renombrar las 7 ocurrencias de `userRole` → `activityType` y prop `sourceUserRole` → `sourceActivityType`; cambiar la pre-selección para leer `registration.activityType` en lugar del rol del token (pld-web/src/components/organisms/reporting-entity/AddVulnerableActivityModal/index.tsx:110,147,175,285,315,322)
- [ ] Renombrar `userRole` → `activityType`; cambiar comparaciones `UserRole.NOTARY` → `'NOTARY'` y `UserRole.REAL_ESTATE` → `'REAL_ESTATE'` (pld-web/src/components/organisms/reporting-entity/ReviewStep/index.tsx:135-143,221-228)
- [ ] Renombrar prop `userRole` → `activityType` en la API del componente (pld-web/src/components/organisms/reporting-entity/ReportingEntityTypeStep/index.tsx:9,57,74,81)
- [ ] Cambiar `userRole: z.enum([UserRole.NOTARY, UserRole.REAL_ESTATE])` → `activityType: z.enum(['NOTARY', 'REAL_ESTATE'])` (pld-web/src/components/organisms/reporting-entity/schemas.ts:9)

### Hook de GET /auth/me

- [ ] Actualizar el tipo de respuesta del hook que consume `GET /auth/me` para usar `JwtWorkspace` en lugar del campo plano `workspaceId`; asegurar que los consumidores de ese hook lean `workspace.id` y `workspace.activityType` (buscar en pld-web/src/queries/ o pld-web/src/api/)

### Cliente HTTP (verificación)

- [ ] Ejecutar `grep -rn "userRole" pld-web/src/queries pld-web/src/api 2>/dev/null` y renombrar cualquier hit a `activityType` en los payloads de `POST /admin/registration` (pld-web/src/queries/ y pld-web/src/api/)

### Build FE

- [ ] Ejecutar `tsc --noEmit` en `pld-web` y confirmar cero errores
- [ ] Ejecutar `yarn build` en `pld-web` y confirmar salida limpia

---

## Docs

- [ ] En la tabla de flujos, columna "Rol", reemplazar `NOTARY, REAL_ESTATE` por `WORKSPACE_ADMIN` donde aplica (docs/flujos/README.md:15,18,19)
- [ ] Actualizar líneas 3, 9, 41, 65 donde se menciona `NOTARY` / `REAL_ESTATE` como rol de usuario → `WORKSPACE_ADMIN` (docs/flujos/registro-auxiliar/flujo.md)
- [ ] Actualizar cabeceras de sección del smoke — agregar nota al inicio indicando `role = WORKSPACE_ADMIN` para ambos usuarios; mantener nombres `pedro.notario` y referencias a `activityType` (docs/flujos/registro-auxiliar/smoke.md:24-25,32,81,102,104,176)
- [ ] Línea 9: "Usuario (NOTARY / REAL_ESTATE)" → "Usuario (`WORKSPACE_ADMIN`)"; línea 113: actualizar wording del claim JWT (`role = WORKSPACE_ADMIN`; `workspace.activityType` = `ActivityType`) (docs/flujos/inicio-sesion/flujo.md:9,113)
- [ ] Línea 9: "Operador (NOTARY / REAL_ESTATE)" → "Operador (`WORKSPACE_ADMIN`)" (docs/flujos/registro-cliente/flujo.md:9)
- [ ] Renombrar concepto `sourceUserRole` → `sourceActivityType` en el texto y bloques toon; actualizar líneas 17, 30, 36-38 (docs/flujos/actividad-secundaria/flujo.md)
- [ ] Separar columna "Tipo" en "Rol" (`WORKSPACE_ADMIN`) y "Actividad" (`NOTARY`/`REAL_ESTATE`); actualizar headers de sección (docs/test-users.md:21,31,33,47,55,58)
- [ ] Ejecutar `grep -rn "NOTARY\|REAL_ESTATE" docs/ | grep -v drawio` y revisar hits residuales — reemplazar los que estén en contexto de rol; dejar los que estén en contexto de tipo de actividad
- [ ] Crear ADR `ADR-XXXX-roles-vs-activity-type.md` (verificar siguiente número en `docs/decisiones/`) con el contenido de la sección 5 del design (docs/decisiones/)

---

## Verificación

- [ ] Smoke test ESC-01: `POST /auth/login` con usuario cuyo `role` era `'NOTARY'` → confirmar JWT retorna `role: "WORKSPACE_ADMIN"` y `workspace: { id, activityType: "NOTARY" }` (sin `'NOTARY'` en claims de rol)
- [ ] Smoke test ESC-02: repetir con usuario ex-`'REAL_ESTATE'` → confirmar `role: "WORKSPACE_ADMIN"` y `workspace.activityType: "REAL_ESTATE"`
- [ ] Smoke test ESC-03: con token `WORKSPACE_ADMIN`, `POST /registration/auxiliaries` → HTTP 201 (guard acepta)
- [ ] Smoke test ESC-04: SUPERADMIN, `POST /admin/registration` con `{ activityType: "NOTARY" }` → HTTP 201; con `{ activityType: "WORKSPACE_ADMIN" }` → HTTP 400
- [ ] Smoke test ESC-05: consultar un registro en BD con `activity_type` — confirmar que el valor es `'NOTARY'` o `'REAL_ESTATE'` sin alteración
- [ ] Smoke test ESC-06: login con usuario `WORKSPACE_ADMIN` en pld-web → confirmar redirección a `/register`
- [ ] Smoke test ESC-07: `POST /auth/select-profile` con `{ activityType: "NOTARY" }` → JWT devuelto contiene `workspace: { id: "...", activityType: "NOTARY" }`; con payload legacy `{ profile: "NOTARY" }` → HTTP 400
- [ ] Smoke test ESC-08: `GET /auth/me` con token `WORKSPACE_ADMIN` → respuesta incluye `workspace: { id, activityType }` sin `workspaceId` plano
- [ ] Verificar ESC-09: `grep -r "NOTARY\|REAL_ESTATE" docs/` filtrando contexto de rol → cero ocurrencias en contexto de rol
