# Apply Progress: add-auxiliary-registration-be

## Estado final

**26/26 tasks completas.** Endpoint `POST /registration/auxiliaries` implementado, lintea limpio, builds verdes, y los 8 smokes pasaron + regresión PF/PM y login.

## Archivos creados / modificados

| File | Action |
|------|--------|
| `pld-api/packages/persistence/migrations/20260428000000-create-auxiliary-profile.sql` | Created |
| `pld-api/packages/persistence/migrations/20260428000000-create-auxiliary-profile.down.sql` | Created |
| `pld-api/packages/shared-types/src/enums.ts` | Modified — `ProfileType.AUXILIARY = 'AUXILIARY'` |
| `pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts` | Modified — shape de `AuxiliaryRegistration` |
| `pld-api/apps/auth-users/src/registration/auxiliaries/entities/auxiliary-profile.entity.ts` | Created |
| `pld-api/apps/auth-users/src/registration/auxiliaries/dto/create-auxiliary.dto.ts` | Created |
| `pld-api/apps/auth-users/src/registration/auxiliaries/dto/auxiliary-response.dto.ts` | Created |
| `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.adapter.ts` | Created |
| `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts` | Created |
| `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.controller.ts` | Created |
| `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.module.ts` | Created |
| `pld-api/apps/auth-users/src/auth-users.module.ts` | Modified — registra `AuxiliariesModule` |

## Lint

`pnpm exec nx run auth-users:lint` — **igual al baseline**: 3 errores y 24 warnings, todos en archivos pre-existentes (`http-error.interceptor.ts`, `jwt.strategy.ts`, `persona-moral.dto.ts`, `_currentUser` en `registration.service.ts`). El código nuevo no agrega ningún warning ni error.

## Build

- `pnpm exec nx run auth-users:build` ✅ — `webpack compiled successfully`.
- `pnpm exec nx run cross:build` ✅ — `webpack compiled successfully` (regresión).

> Tras destrabar permisos: `sudo rm -rf /home/cristobal/work/pld/pld-api/dist/apps/auth-users` (residuo de docker build previo del 27/abr 16:08).

## Migration

- Up aplicada vía `docker compose exec -T mysql mysql … < migration.sql`.
- Schema verificado con `SHOW CREATE TABLE auxiliary_profile`: PK, FKs (`fk_auxiliary_profile_parent_user`, `fk_auxiliary_profile_address`) ON DELETE RESTRICT, index `idx_auxiliary_profile_parent`.
- Down probado: `DROP TABLE` limpio + re-up restaura schema sin errores.

## Smokes manuales

Stack levantado con `docker compose -f pld-api/docker-compose.dev.yml up -d`. Auth `auth-users` reiniciado tras la migration para cargar la nueva entity.

| # | Test | Resultado |
|---|---|---|
| 4.5 | `POST /registration/auxiliaries` con JWT NOTARY válido | ✅ HTTP 201 con `{ user: {role:"AUXILIARY", profileType:"AUXILIARY", profileId, mustChangePassword:true}, temporaryPassword }` |
| 4.6 | Mismo email duplicado | ✅ HTTP 409 `Email already registered` |
| 4.7 | JWT SUPERADMIN | ✅ HTTP 403 `Insufficient role` |
| 4.8 | Sin `Authorization` | ✅ HTTP 401 `Unauthorized` |
| 4.9 | DTO sin `rfc` | ✅ HTTP 400 con array de mensajes del validator |
| 4.10 | Login del auxiliar con la `temporaryPassword` | ✅ HTTP 201 + JWT (`role:AUXILIARY`); `GET /auth/me` confirma `mustChangePassword:true` |
| 4.11 | Regresión `POST /admin/registration` con SUPERADMIN | ✅ HTTP 201 con draft IN_PROGRESS |
| 4.12 | Regresión login NOTARY existente | ✅ HTTP 201 con token |

Auxiliar de prueba (`gerson.aux01@example.mx`) limpiado de la BD tras smokes.

## Decisiones / desviaciones del design

- **`reporting_entity_address.city`** es NOT NULL en BD pero el form de auxiliary no captura `city`. Decisión: al insertar la dirección, **`city = municipality`** como fallback. Documentado como comentario en `auxiliaries.adapter.ts`. Sin impacto en el módulo `admin/registration/`.
- **Pre-check de email** fail-fast antes de los inserts, además del catch de `ER_DUP_ENTRY` (defensa autoritativa por race). UK `ux_users_email` cubre la atomicidad real.
- **Reuso de `ReportingEntityAddressEntity`** del módulo `admin/registration/` (no se duplica). El nuevo `AuxiliariesModule` la registra en su `TypeOrmModule.forFeature`.
- **`AuxiliariesModule` en `auth-users.module.ts`**: el módulo raíz se llama así (no `app.module.ts`). Registrado al lado de `AdminModule`.
- **No-null assertion** en `manager.queryRunner!` reemplazado por guard explícito (lint clean).
- **Field name del JWT en login**: `token` (no `accessToken`). Lo usé tal cual sin cambiar el contrato existente.

## Reglas de proyecto verificadas

- ✅ Patrón controller → adapter → service.
- ✅ Sin imports de `@nestjs/*` ni `TypeOrmModule` en `packages/domain-*`.
- ✅ N/A web (este change es BE only).
- ✅ Login HTTP 201 sigue funcionando (smokes 4.10 + 4.12).

## Próximos pasos

- `sdd-verify` para validar contra spec (opcional — ya validado vía smokes).
- `sdd-archive` para promover el spec a `openspec/specs/auxiliary-registration/`.
- Continuar con `add-auxiliary-registration-fe`.
