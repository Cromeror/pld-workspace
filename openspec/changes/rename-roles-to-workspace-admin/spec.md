# Spec: Rename roles NOTARY/REAL_ESTATE to WORKSPACE_ADMIN

**Change**: rename-roles-to-workspace-admin
**Status**: done
**Created**: 2026-05-19

---

## Requisitos funcionales

### RF-01 — Enum `UserRole` unificado
El enum `UserRole` en `packages/shared-types/src/enums.ts` contiene exactamente los valores `SUPERADMIN`, `WORKSPACE_ADMIN`, `AUXILIARY`. Los valores `NOTARY` y `REAL_ESTATE` no existen en el enum.

### RF-02 — Columna `users.role` migrada
Todos los registros de la tabla `users` que tenían `role IN ('NOTARY', 'REAL_ESTATE')` pasan a `role = 'WORKSPACE_ADMIN'`. La columna `ENUM` de MySQL refleja el nuevo set de valores.

### RF-03 — Columna `registration.activity_type` (renombrada)
La columna antes llamada `registration.user_role` se renombra a `registration.activity_type`. Los valores almacenados (`'NOTARY'`, `'REAL_ESTATE'`) no cambian.

### RF-04 — DTO `POST /admin/registration` actualizado
El campo del payload se renombra de `userRole` a `activityType`. El decorador `@IsIn` acepta los valores `'NOTARY'` y `'REAL_ESTATE'` (tipo de actividad vulnerable, sin cambio de valores).

### RF-05 — Guard `POST /registration/auxiliaries` actualizado
El decorador `@Roles(...)` en `AuxiliariesController` lista únicamente `UserRole.WORKSPACE_ADMIN`. No hace referencia a `NOTARY` ni `REAL_ESTATE`.

### RF-06 — Tipo FE sincronizado
`pld-web/src/types/UserRole.ts` contiene exactamente las claves `SUPERADMIN`, `WORKSPACE_ADMIN`, `AUXILIARY`. Las claves `NOTARY` y `REAL_ESTATE` no existen.

### RF-07 — Redirect post-login por rol actualizado
`pld-web/src/config/postLoginRedirect.ts` mapea `UserRole.WORKSPACE_ADMIN` → `RoutesUrl.REGISTER`. Las entradas previas de `NOTARY` y `REAL_ESTATE` se eliminan.

### RF-08 — Documentación limpia
Ningún archivo `.md` del workspace menciona `NOTARY` o `REAL_ESTATE` en contexto de rol de usuario (solo en contexto de tipo de actividad de negocio / `activityType`).

---

## Requisitos no funcionales

### RNF-01 — Sin downtime durante la migración
La migración SQL se ejecuta como una sola transacción. El `UPDATE` precede al `ALTER TABLE` para evitar violaciones de constraint durante la transición.

### RNF-02 — Rollback posible
La migración TypeORM implementa `up()` y `down()`. El `down()` revierte el `UPDATE`, el `ALTER ENUM` y el `RENAME COLUMN`. Limitación documentada: el `down()` no recupera la distinción `NOTARY` vs `REAL_ESTATE` en `users.role` (ambos colapsan a `WORKSPACE_ADMIN`); requiere snapshot previo para rollback exacto.

### RNF-03 — Sin lógica de negocio modificada
El refactor cambia únicamente identificadores (enum values, nombres de campo, nombres de columna). Ningún flujo de registro, login ni creación de auxiliares cambia su comportamiento.

### RNF-04 — Compilación sin errores
`nx run auth-users:build` y `yarn build` (tsc + vite en pld-web) pasan en verde tras el cambio. El compilador TS actúa como red de seguridad: actualizar el enum antes de compilar garantiza que cualquier referencia huérfana a `UserRole.NOTARY` / `UserRole.REAL_ESTATE` sea detectada estáticamente.

### RNF-05 — Ventana de inconsistencia mínima
Los dos commits (BE y FE+docs) se despliegan en el mismo pipeline o en ventana controlada. No se deja el BE actualizado con el FE apuntando a valores obsoletos por más de un ciclo de despliegue.

---

## Escenarios

### ESC-01 — Login con usuario ex-NOTARY emite `WORKSPACE_ADMIN` en JWT

**Given** un usuario cuyo `role` en `users` era `'NOTARY'` antes de la migración y ahora es `'WORKSPACE_ADMIN'`
**When** hace `POST /auth/login` con credenciales válidas
**Then** el JWT retornado incluye `role: "WORKSPACE_ADMIN"` y el `workspaceId` correspondiente a su registro; el valor `'NOTARY'` no aparece en ningún claim del token.

### ESC-02 — Login con usuario ex-REAL_ESTATE emite `WORKSPACE_ADMIN` en JWT

**Given** un usuario cuyo `role` en `users` era `'REAL_ESTATE'` antes de la migración y ahora es `'WORKSPACE_ADMIN'`
**When** hace `POST /auth/login` con credenciales válidas
**Then** el JWT retornado incluye `role: "WORKSPACE_ADMIN"` y el `workspaceId` correspondiente a su registro; el valor `'REAL_ESTATE'` no aparece en ningún claim del token.

### ESC-03 — WORKSPACE_ADMIN puede crear auxiliar

**Given** un usuario autenticado con `role: WORKSPACE_ADMIN` en su JWT
**When** hace `POST /registration/auxiliaries` con un body válido (`CreateAuxiliaryDto`)
**Then** el `RolesGuard` acepta la petición (HTTP 201) porque `@Roles(UserRole.WORKSPACE_ADMIN)` está configurado en `AuxiliariesController`.

### ESC-04 — `POST /admin/registration` acepta `activityType: NOTARY` o `activityType: REAL_ESTATE`

**Given** un SUPERADMIN autenticado
**When** hace `POST /admin/registration` con body `{ activityType: "NOTARY", profileType: "INDIVIDUAL", rfc: "..." }` o `{ activityType: "REAL_ESTATE", ... }`
**Then** la validación del DTO pasa (HTTP 201); el campo `activityType` persiste correctamente en `registration.activity_type`.

**And** si envía `{ activityType: "WORKSPACE_ADMIN", ... }`
**Then** la validación falla (HTTP 400) porque `WORKSPACE_ADMIN` no es un valor de actividad válido.

### ESC-05 — Consulta de `registration.activity_type` devuelve valores sin cambio

**Given** un registro existente en la tabla `registration` con `activity_type = 'NOTARY'` (columna renombrada)
**When** se consulta ese registro vía la API o directamente en la BD
**Then** el valor retornado es `'NOTARY'`; la migración no alteró los valores almacenados, solo el nombre de la columna.

**And** lo mismo aplica para registros con `activity_type = 'REAL_ESTATE'`.

### ESC-06 — FE redirige `WORKSPACE_ADMIN` a `/register` post-login

**Given** que `postLoginRedirect.ts` mapea `UserRole.WORKSPACE_ADMIN` → `RoutesUrl.REGISTER`
**When** el usuario con rol `WORKSPACE_ADMIN` completa el login exitosamente
**Then** la aplicación navega a la ruta `/register`; no existe ningún caso `NOTARY` ni `REAL_ESTATE` en el mapa de redirect.

### ESC-07 — Documentación no menciona `NOTARY`/`REAL_ESTATE` como rol de usuario

**Given** el repositorio tras aplicar el cambio
**When** se ejecuta `grep -r "NOTARY\|REAL_ESTATE" docs/` filtrando por contexto de rol de usuario
**Then** no se encuentran ocurrencias en contexto de rol (e.g., "rol NOTARY", "UserRole.NOTARY"); las únicas ocurrencias permitidas son en contexto de tipo de actividad (e.g., "`activityType: NOTARY`", "actividad vulnerable de notaría").
