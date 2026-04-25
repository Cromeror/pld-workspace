# Archive Report: rename-users-table-to-english

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & SMOKE-VERIFIED
**Sub-repos**: pld-api (BE) + pld-web (FE) + DB migration

## Summary

Segundo sub-change del split de `rename-to-english` umbrella. Renombró las 5 propiedades en español de `UserEntity` (`nombre`, `apellidoPaterno`, `apellidoMaterno`, `telefono`, `activo`) a sus equivalentes en inglés (`firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active`), tanto en TS como en columnas MySQL. Renombró también las URLs y body fields de los endpoints HTTP `POST /system-users/notario-inmobiliario` y `POST /system-users/interno-externo-auxiliar` a `notary-real-estate` e `internal-external-auxiliary`. Lockstep cross-repo, single commit por sub-repo. Display strings (Swagger `description:`, `example:`, JSX labels) intactos en español.

## Decisiones aplicadas

- **D1 — Lockstep**: rename atómico cross-repo en una sola PR por sub-repo.
- **D2 — Migración 1-paso por columna**: `ALTER TABLE users CHANGE COLUMN ...` por columna (5 ALTERs). No hace falta expand+backfill+restrict porque los datos son strings opacos.
- **D3 — Rutas HTTP renombradas**: cero consumers FE de los endpoints viejos (verificado), no se mantiene alias.
- **D4 — Swagger display preservado**: `description:` y `example:` quedan en español. Solo TS prop names cambian.
- **D5 — `@Column(name:)`**: actualizado en todos los decoradores.
- **D6 — `getUserByPhone(telefono)`**: parámetro y `where` clause renombrados.
- **D7 — Test fixture sweep**: ejecutado, 0 fixtures con identificadores en español residuales.
- **D8 — `CreateUserOutput` polimorfismo**: extiende `Omit<UserEntity, 'passwordHash'>` → recoge automáticamente las props nuevas. Sin cambio explícito.

## Renames aplicados

### TS (entity, port, adapter, controller, DTOs, registration)
- `nombre` → `firstName`
- `apellidoPaterno` → `paternalSurname`
- `apellidoMaterno` → `maternalSurname`
- `telefono` → `phone`
- `activo` → `active`

### MySQL columns (`users`)
- `nombre` → `first_name` VARCHAR(120) NOT NULL
- `apellido_paterno` → `paternal_surname` VARCHAR(120) NULL
- `apellido_materno` → `maternal_surname` VARCHAR(120) NULL
- `telefono` → `phone` VARCHAR(30) NULL
- `activo` → `active` TINYINT(1) NOT NULL DEFAULT 1

### URL paths
- `POST /system-users/notario-inmobiliario` → `POST /system-users/notary-real-estate`
- `POST /system-users/interno-externo-auxiliar` → `POST /system-users/internal-external-auxiliary`

### Local vars (registration.service.ts:441-443)
- `userNombre`, `userApellidoPaterno`, `userApellidoMaterno` → `userFirstName`, `userPaternalSurname`, `userMaternalSurname`

## Cambios

### BE (pld-api, commit `695a2a1`)
- `packages/domain-auth-users/src/entities/user.entity.ts` — 5 `@Column(name:)` + 5 props.
- `packages/domain-auth-users/src/ports/users.port.ts` — `CreateUserInput`, `CurrentUserDto`, `getUserByPhone`.
- `packages/domain-auth-users/src/adapters/users.adapter.ts` — `toCurrentUserDto`, `createUser`, `getUserByPhone`.
- `apps/auth-users/src/users/dto/create-user.dto.ts` — ambos DTOs (TS props; description/example en español).
- `apps/auth-users/src/users/users.controller.ts` — 2 `@Post()` paths + handlers.
- `apps/auth-users/src/auth/auth.controller.ts` — `currentUser.activo` → `.active` (encontrado en sweep).
- `apps/auth-users/src/admin/registration/registration.adapter.ts` — internal createUser.
- `apps/auth-users/src/admin/registration/registration.service.ts` — local vars + call-sites.
- `packages/persistence/migrations/20260425020000-rename-users-columns-to-english.{sql,down.sql}` — nuevas migraciones.
- 8 archivos modificados, 2 creados.

### FE (pld-web, commit `7879794`)
- `src/types/CurrentUser.ts` — 5 type properties.
- 0 consumers en código (verificado con grep).
- 1 archivo modificado.

## Validación

```
BE:
  nx run auth-users:build → SUCCESS
  nx run cross:build → SUCCESS
  tsc --noEmit (domain-auth-users, auth-users, shared-types) → 0 errores
  grep "nombre:|apellidoPaterno:|apellidoMaterno:|telefono:|.activo\b" pld-api/{apps,libs,packages}/*/src → 0 matches en código TS (excluido description: y .sql)

FE:
  yarn tsc -b --noEmit → 0 errores
  yarn build → SUCCESS
  grep ".nombre|.apellidoPaterno|.apellidoMaterno|.telefono|.activo" pld-web/src --include='*.ts' --include='*.tsx' → 0 matches en código

DB (post-migration):
  DESCRIBE users → first_name, paternal_surname, maternal_surname, phone, active confirmados
  11 filas preservadas

Smoke live:
  POST /auth/login (smoke-renames@pld.local) → HTTP 201 + JWT
  GET /auth/me con Bearer → HTTP 200 + DTO con keys en inglés (firstName/paternalSurname/maternalSurname/phone/active), 0 keys en español
  POST /system-users/notary-real-estate → HTTP 201
  POST /system-users/internal-external-auxiliary → HTTP 201
  POST /system-users/notario-inmobiliario (path viejo) → HTTP 404 ✅

Display strings:
  Swagger description: en español intactos ✅
  example: 'Carlos'/'Ramírez'/'Lucía' intactos ✅
```

## Commits

- pld-api: `695a2a1 refactor(users): rename UserEntity props + DB columns + HTTP DTOs to english`
- pld-web: `7879794 refactor(types): rename CurrentUser props to english to match BE`

## Specs Synced

| Domain | Action | Details |
|--------|--------|---------|
| auth | Updated | Added 3 requirements for UserEntity props, CreateUserInput, CurrentUserDto |
| users | Updated | Added 2 requirements for UserEntity column renames and HTTP endpoint paths |
| pld-web | Updated | Added 1 requirement for CurrentUser type mirror |

## Archive Contents

- proposal.md ✅
- design.md ✅
- tasks.md ✅
- specs/auth/spec.md ✅
- specs/users/spec.md ✅
- specs/pld-web/spec.md ✅

## Source of Truth Updated

The following specs now reflect the new behavior:
- `openspec/specs/auth/spec.md` — Updated with 3 new requirements for UserEntity props and DTOs
- `openspec/specs/users/spec.md` — Updated with 2 new requirements for DB columns and HTTP paths
- `openspec/specs/pld-web/spec.md` — Updated with 1 new requirement for CurrentUser type

## SDD Cycle Complete

The change has been fully planned, implemented, verified, and archived. Ready for the next change in the umbrella (`rename-participants-entities-to-english` or `rename-legacy-types-to-english`).
