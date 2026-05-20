# Proposal: Rename roles NOTARY/REAL_ESTATE to WORKSPACE_ADMIN

**Status**: draft
**Created**: 2026-05-19
**Sub-repos**: pld-api + pld-web + docs

## Intent

El enum `UserRole` tiene dos valores que representan el mismo nivel de acceso: `NOTARY` (notario) y `REAL_ESTATE` (inmobiliaria). Ambos administran su workspace y pueden crear auxiliares — son funcionalmente idénticos desde el punto de vista de permisos. Tener nombres distintos genera confusión en guards, DTOs y código de routing, y fuerza repetición (`@Roles(UserRole.NOTARY, UserRole.REAL_ESTATE)`) en cada punto de control.

El cambio unifica ambos en un único rol semántico: `WORKSPACE_ADMIN`. El tipo de actividad del negocio (`NOTARY` / `REAL_ESTATE`) queda almacenado en la columna `registration.activity_type` — que se renombra desde `user_role` para reflejar que su propósito es distinto al rol de seguridad. Los valores de string en esa columna (`'NOTARY'`, `'REAL_ESTATE'`) no cambian.

Hacer el cambio ahora, mientras el sistema está en desarrollo activo y antes de que haya datos de producción significativos, minimiza el costo de la migración SQL y reduce el riesgo de divergencia entre BE, FE y documentación.

## Scope

### In Scope

#### pld-api

- **`packages/shared-types/src/enums.ts`**: reemplazar `NOTARY` y `REAL_ESTATE` por `WORKSPACE_ADMIN` en el enum `UserRole`.
- **DTOs con `@IsIn([...])`** que listan roles explícitos: actualizar para incluir `WORKSPACE_ADMIN` y eliminar `NOTARY` / `REAL_ESTATE`.
- **Guards y decorators `@Roles(...)`**: sustituir toda ocurrencia de `UserRole.NOTARY` y `UserRole.REAL_ESTATE` por `UserRole.WORKSPACE_ADMIN`.
- **Migración SQL** (TypeORM, `packages/persistence/migrations/`):
  - `UPDATE users SET role='WORKSPACE_ADMIN' WHERE role IN ('NOTARY','REAL_ESTATE')`.
  - `ALTER TABLE users MODIFY COLUMN role ENUM(...)` para reflejar el nuevo set de valores.
  - Renombrar columna `registration.user_role` → `registration.activity_type` (los valores de string almacenados no cambian).
- **Cualquier referencia** en `apps/auth-users/` a `UserRole.NOTARY` o `UserRole.REAL_ESTATE` (seeds, factories de test, comentarios con impacto).

#### pld-web

- **`src/types/UserRole.ts`**: reemplazar `NOTARY` y `REAL_ESTATE` por `WORKSPACE_ADMIN`.
- **`src/config/postLoginRedirect.ts`** (o equivalente): casos `NOTARY` / `REAL_ESTATE` → `WORKSPACE_ADMIN`.
- **`src/routes/RoleProtectedRoute.tsx`** y cualquier array `requiredRoles` que liste `UserRole.NOTARY` o `UserRole.REAL_ESTATE`.
- **Payload `POST /admin/registration`**: renombrar el campo `userRole` → `activityType` en el cliente HTTP y en los tipos que lo consumen.
- **Cualquier componente o hook** que compare directamente con los valores de string `'NOTARY'` o `'REAL_ESTATE'` para lógica de rol.

#### docs

- **`docs/flujos/**/*.md`**: ocurrencias de `NOTARY` / `REAL_ESTATE` en contexto de rol de usuario → `WORKSPACE_ADMIN`.
- **`docs/arquitecturas/**/*.md`**: ídem.
- **`test-users.md`** (si existe): actualizar rol en la tabla de usuarios de prueba.
- **`REGISTRO_SCHEMA.md`** (si existe): actualizar la sección que describe el campo de rol.
- Cualquier otro `.md` del workspace que mencione `NOTARY` o `REAL_ESTATE` como rol (no como tipo de actividad de negocio).

### Out of Scope

- Cambiar los valores `'NOTARY'` / `'REAL_ESTATE'` almacenados en `registration.activity_type`. Esos strings representan el tipo de actividad vulnerable del negocio y no son roles de seguridad — permanecen intactos.
- Rediseñar el modelo de permisos (granularidad fina, scopes, ABAC).
- Añadir nuevos roles o eliminar `SUPERADMIN` / `AUXILIARY`.
- Modificar la lógica de negocio de ningún flujo (registro, login, creación de auxiliares) — solo se cambian los identificadores.
- Migraciones de datos en entornos de producción gestionados externamente (el script SQL se entrega, la ejecución la coordina DevOps).

## Approach

El cambio es principalmente un **refactor de identificadores** sin lógica nueva. Se ejecuta en dos commits separados para facilitar la revisión y el rollback independiente:

1. **BE — migración + código** (commit 1):
   - Crear la migración TypeORM que actualiza la tabla `users` y renombra la columna en `registration`.
   - Actualizar `packages/shared-types/src/enums.ts`.
   - Buscar con `grep -r "UserRole.NOTARY\|UserRole.REAL_ESTATE"` en `pld-api/` y reemplazar en guards, DTOs, services y tests.
   - Verificar que `nx run auth-users:build` pase en verde.

2. **FE + docs** (commit 2):
   - Actualizar `src/types/UserRole.ts` en `pld-web`.
   - Reemplazar referencias en routing, guards y payload del cliente HTTP.
   - Actualizar archivos `.md` relevantes en `docs/`.
   - Verificar que `yarn build` (tsc + vite) pase en verde.

El orden BE → FE permite que el BE esté alineado antes de actualizar el cliente. Si hay un entorno de staging compartido, la ventana entre ambos commits debe ser mínima.

## Affected Areas

### pld-api

| Área | Impacto | Detalle |
|------|---------|---------|
| `packages/shared-types/src/enums.ts` | Modified | Reemplazar `NOTARY`, `REAL_ESTATE` → `WORKSPACE_ADMIN` en `UserRole` |
| `apps/auth-users/src/**` (guards, DTOs, services) | Modified | Toda referencia a `UserRole.NOTARY` / `UserRole.REAL_ESTATE` |
| `packages/persistence/migrations/<ts>-rename-roles.ts` | New | Migration SQL: UPDATE users + ALTER ENUM + RENAME COLUMN |
| Tests / seeds / factories | Modified | Sustituir valores de rol en fixtures |

### pld-web

| Área | Impacto | Detalle |
|------|---------|---------|
| `src/types/UserRole.ts` | Modified | Reemplazar `NOTARY`, `REAL_ESTATE` → `WORKSPACE_ADMIN` |
| `src/config/postLoginRedirect.ts` | Modified | Casos de routing por rol |
| `src/routes/RoleProtectedRoute.tsx` | Modified | Arrays `requiredRoles` |
| Cliente HTTP (payload `POST /admin/registration`) | Modified | Campo `userRole` → `activityType` |
| Componentes que comparan rol por string | Modified | Cualquier `=== 'NOTARY'` o `=== 'REAL_ESTATE'` para lógica de rol |

### docs

| Área | Impacto | Detalle |
|------|---------|---------|
| `docs/flujos/**/*.md` | Modified | `NOTARY` / `REAL_ESTATE` como rol → `WORKSPACE_ADMIN` |
| `docs/arquitecturas/**/*.md` | Modified | Ídem |
| `test-users.md`, `REGISTRO_SCHEMA.md` | Modified | Tablas de usuarios y esquemas |

## Risks

| Riesgo | Probabilidad | Mitigación |
|--------|-------------|------------|
| Dejar una referencia huérfana a `UserRole.NOTARY` / `REAL_ESTATE` en código | Media | Búsqueda exhaustiva con `grep -r` antes de commit; el compilador TS la detecta si el enum se actualiza primero |
| Confundir `registration.activity_type` (valores `NOTARY`/`REAL_ESTATE` que NO cambian) con el enum de rol | Alta | El límite se documenta explícitamente en specs y se verifica con un test de integración que inserta con `activityType: 'NOTARY'` y comprueba que el campo llega intacto |
| Ventana de inconsistencia BE/FE en staging entre los dos commits | Baja | Mantener la ventana mínima; desplegar ambos commits en el mismo pipeline o coordinar el merge |
| Rollback de la migración SQL destruye datos de usuarios ya creados con `WORKSPACE_ADMIN` | Baja | Solo aplica si hay datos en staging post-migración; el down de la migration usa UPDATE inverso y ALTER ENUM — documentar en el rollback plan |
| Documentación actualizada de forma incompleta (algún `.md` olvidado) | Media | `grep -r "UserRole.*NOTARY\|UserRole.*REAL_ESTATE" docs/` como checklist de cierre en sdd-verify |

## Rollback Plan

1. Revertir el commit FE (primero el FE, luego el BE si es necesario).
2. Ejecutar el `down()` de la migration TypeORM:
   - `UPDATE users SET role='NOTARY' WHERE role='WORKSPACE_ADMIN' AND <criterio>` — **nota**: el criterio de cuál era `NOTARY` vs `REAL_ESTATE` se pierde; ambos colapsarían a un valor. Si se requiere rollback exacto, usar un respaldo previo.
   - `ALTER TABLE users MODIFY COLUMN role ENUM(...)` al set anterior.
   - `RENAME COLUMN registration.activity_type` → `registration.user_role`.
3. Reversar el enum en `packages/shared-types/src/enums.ts` y rebuildar.
4. **Limitación**: el rollback de datos en `users.role` no recupera la distinción `NOTARY` vs `REAL_ESTATE` una vez unificados. Si esto es crítico, tomar snapshot de la tabla antes de la migración.

## Dependencies

- No hay dependencias de otros cambios SDD activos.
- La migración TypeORM requiere que `packages/persistence` esté configurado y las migraciones previas aplicadas.
- El campo `registration.activity_type` asume que el código que escribe en esa tabla sigue usando los valores `'NOTARY'` / `'REAL_ESTATE'` — no se toca esa lógica de escritura.
