# Archive Report: rename-enums-to-english

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & SMOKE-VERIFIED
**Sub-repos**: pld-api (BE) + pld-web (FE) + DB migration

## Summary

Primer sub-change del split de `rename-to-english` umbrella. Renombró 8 enums en `shared-types` (nombres + valores) eliminando todos los identificadores en español del contrato API↔Web. Migration de MySQL aplicada en 3 pasos (expand ENUM → backfill data → restrict ENUM) sobre `users.role` + UPDATEs defensivas en `registration.user_role` y `registration.profile_type` (VARCHAR). Strings visibles al usuario (labels JSX, placeholders, mensajes de error de zod, copy de modales) NO se tocaron — quedan en español por decisión del producto.

## Decisiones aplicadas

- **D1 — Lockstep**: rename atómico cross-repo en una sola PR por sub-repo, sin ventana de dual-support. Workspace coordina ambos sub-repos.
- **D2 — Migración 3-pasos**: ENUM expand (acepta old + new) → UPDATE backfill → ENUM restrict (solo new). Down simétrica.
- **D3 — Deploy order**: BE primero (con migration aplicada), FE después.
- **D4 — Registration tables**: VARCHAR(30), no ENUM — basta UPDATE idempotente.
- **D5 — `TipoPersonaMoral` local enum**: deferido al sub-change de participants.
- **D6 — `PersonType` FE incluido**: rename para consistencia con `ProfileType` semantically.
- **D7 — DTO + Swagger**: validators y examples actualizados.
- **D8 — Test fixtures**: `registration.service.spec.ts` actualizado.
- **D9 — JWT cutover**: tokens emitidos pre-migration con role en español → 401 (axios interceptor → forceLogout).
- **D10 — Sweep order**: BE source → BE call-sites → DB → FE source → FE call-sites → builds.

## Renames aplicados

### UserRole (nombre intacto, valores cambian)
- `NOTARIO` → `NOTARY`
- `INMOBILIARIA` → `REAL_ESTATE`
- `AUXILIAR` → `AUXILIARY`
- `SUPERADMIN` → unchanged

### Enums renombrados (nombre + valores)
- `TipoPersonaParticipante` → `ProfileType` (`PERSONA_FISICA`/`PERSONA_MORAL`/`FIDEICOMISO` → `INDIVIDUAL`/`LEGAL_ENTITY`/`TRUST`)
- `ActividadVulnerable` → `VulnerableActivity` (10 valores; nota: `FIDEICOMISOS` → `TRUST_AGREEMENTS` para distinguir del `ProfileType.TRUST`)
- `TipoMoneda` → `CurrencyType` (valores `MXN`/`USD` intactos)
- `FormaPago` → `PaymentMethod`
- `TipoReporte` → `ReportType`
- `ModalidadAtencion` → `AttendanceMode`
- `Nacionalidad` → `Nationality`

### FE-only
- `PersonTypeEnum` valores en español → inglés (`FIDEICOMISO_ANEXO8` → `TRUST_ANNEX_8`, etc.)

## Cambios

### BE (pld-api, commit `6b481ea`)
- `packages/shared-types/src/enums.ts` — fuente de los renames.
- `packages/shared-types/src/participants.ts` — referencias `TipoPersonaParticipante` → `ProfileType`.
- `packages/domain-auth-users/src/{entities/user.entity.ts,ports/users.port.ts}` — `profileType` tipado como `ProfileType`.
- `packages/persistence/migrations/20260425010000-rename-enums-to-english.{sql,down.sql}` — nuevas migraciones.
- `libs/{catalogs,auth-profiles,operacion,reports,participants}/` — imports actualizados.
- `apps/auth-users/src/admin/registration/` — DTOs, service, adapter, spec actualizados (8 archivos).
- `apps/auth-users/src/users/dto/create-user.dto.ts` — Swagger examples.
- `apps/auth-users/src/catalogs/{catalogs.data.ts,catalogs.controller.spec.ts}` — keys del catálogo VulnerableActivity.
- `apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts` — Nationality.
- 23 archivos modificados, 2 creados.

### FE (pld-web, commit `824ac22`)
- `src/types/{UserRole,registration,CurrentUser,PersonType}.ts` — types actualizados.
- `src/config/postLoginRedirect.ts` — referencias UserRole.
- `src/components/organisms/reporting-entity/{schemas,ReportingEntityTypeStep,ReviewStep}/...` — wizard zod + comparaciones.
- `src/pages/admin/ReportingEntityRegistrationPage.tsx` — múltiples uses.
- `src/components/organisms/external-users/PersonType/index.tsx` — `FIDEICOMISO_ANEXO8` → `TRUST_ANNEX_8`.
- 10 archivos modificados.

## Validación

```
BE:
  nx run shared-types:build → 0 errores
  nx run auth-users:build → 0 errores
  nx run cross:build → 0 errores
  grep "TipoPersonaParticipante|ActividadVulnerable|TipoMoneda|FormaPago|TipoReporte|ModalidadAtencion|Nacionalidad" pld-api/{apps,libs,packages}/*/src → 0 matches
  grep "'NOTARIO'|'INMOBILIARIA'|'AUXILIAR'|'PERSONA_FISICA'|'PERSONA_MORAL'|'FIDEICOMISO'|'EFECTIVO'|'TRANSFERENCIA'|'CHEQUE'|'TARJETA'|'MEXICANA'|'EXTRANJERA'" pld-api/{apps,libs,packages}/*/src → 0 matches

FE:
  yarn tsc -b --noEmit → 0 errores
  yarn build → 0 errores
  grep ídem → 0 matches en pld-web/src

DB (post-migration):
  DESCRIBE users → role ENUM('SUPERADMIN','NOTARY','REAL_ESTATE','AUXILIARY')
  SELECT DISTINCT role FROM users → solo valores nuevos

Smoke live:
  POST /auth/login (smoke-test@pld.local) → HTTP 201 + JWT con role:"NOTARY"
  GET /auth/me con Bearer → HTTP 200 + role:"NOTARY"
  POST /admin/registration (NOTARY/INDIVIDUAL) → HTTP 201
  POST /admin/registration (NOTARIO/PERSONA_FISICA) → HTTP 400 ✅ (rechaza valores viejos)

Display strings JSX:
  "Notario", "Inmobiliaria", "Persona Física", "Persona Moral" intactos en JSX text content ✅
```

## Commits

- pld-api: `6b481ea refactor(shared-types): rename enums to english (UserRole values, ProfileType, VulnerableActivity, etc)`
- pld-web: `824ac22 refactor(types): rename enums to english to match BE contract`

## Specs Synced

| Domain | Action | Details |
|--------|--------|---------|
| shared-types | Created | 11 new requirements for enum exports and consumers |
| auth | Updated | Added 2 requirements for UserRole ENUM and JWT role claim in English |
| pld-web | Updated | Added 7 requirements for mirror types and call-site updates |

## Archive Contents

- proposal.md ✅
- design.md ✅
- tasks.md ✅
- specs/auth/spec.md ✅
- specs/shared-types/spec.md ✅
- specs/pld-web/spec.md ✅
- apply-progress.md ✅

## Source of Truth Updated

The following specs now reflect the new behavior:
- `openspec/specs/shared-types/spec.md` — Created with 11 requirements
- `openspec/specs/auth/spec.md` — Updated with 2 new requirements (UserRole ENUM + JWT role claim)
- `openspec/specs/pld-web/spec.md` — Updated with 7 new requirements (mirror types + call-sites)

## SDD Cycle Complete

The change has been fully planned, implemented, verified, and archived. Ready for the next change in the umbrella (`rename-users-table-to-english`, `rename-participants-entities-to-english`, or `rename-legacy-types-to-english`).
