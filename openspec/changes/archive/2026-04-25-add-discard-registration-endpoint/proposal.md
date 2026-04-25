# Proposal: Add DELETE endpoint to discard registration drafts

**Status**: draft
**Created**: 2026-04-25
**Sub-repo**: pld-api (BE-only)

## Intent

Exponer `DELETE /admin/registration/:id` para descartar (soft-delete) un borrador de registro de sujeto obligado. El endpoint marca el draft como `status = CANCELLED` y setea `deleted_at`, liberando el RFC para que pueda reusarse en un nuevo draft.

## Scope

**In** (BE — `pld-api`):
- Agregar handler `discard()` al `registration.controller.ts` con `@Delete(':id')`.
- Agregar método `discard()` al `RegistrationService`.
- Agregar método `softDeleteRegistration()` al `RegistrationAdapter` que actualiza la entity:
  - `status = CANCELLED`.
  - `deleted_at = NOW()`.
- Validaciones HTTP:
  - 204 No Content si soft-delete exitoso.
  - 404 si el registration no existe.
  - 409 si el registration ya está en estado `COMPLETED` (no se puede descartar uno ya finalizado).
  - 410 Gone si el registration expiró por TTL.
  - 401 si no hay JWT.
  - 403 si el JWT no tiene rol `SUPERADMIN`.
- Test del controller (4 casos: 204, 404, 409, 410).
- Extender `openspec/specs/...` (nuevo spec o anexo a uno existente del módulo `admin/registration`).

**Out**:
- Hard-delete (la columna `deleted_at` ya está prevista en el schema; usamos soft-delete que respeta la auditoría).
- Restauración de drafts cancelados.
- Endpoint de listado que incluya `CANCELLED` por default (los listados existentes ya tienen `?includeExpired` y filtran `CANCELLED`; no se altera).
- Cualquier cambio en el FE — el FE consume este endpoint en el change `add-reporting-entity-registration-ui`.

## Motivation

1. **Bloqueante para Q21 del wizard PF/PM**: cuando el usuario reanuda un draft y cambia el `profileType` o `userRole` en step 1, el wizard debe descartar el draft actual y crear uno nuevo. Sin DELETE, el RFC queda bloqueado por 30 días (TTL).
2. **Esquema BD ya preparado**: `REGISTRO_SCHEMA.md` previó las columnas `status = CANCELLED` y `deleted_at` desde el diseño inicial. Solo falta exponer el endpoint.
3. **Liberar RFC**: la deduplicación en `POST /admin/registration` filtra por `status = IN_PROGRESS`. Marcar como `CANCELLED` libera el RFC inmediatamente para reintentos.
4. **Auditoría preservada**: el soft-delete mantiene el row en BD para análisis posterior si fuera requerido.

## Approach

1. Agregar `softDeleteRegistration` al `RegistrationAdapter` con UPDATE que cambia `status` y `deleted_at`.
2. Agregar método `discard` al `RegistrationService` con validaciones de estado (encontrado, no completado, no expirado).
3. Registrar handler en el controller con guards JWT + Roles.
4. Tests unitarios cubriendo los 4 escenarios HTTP.
5. Smoke test manual con `curl`.

## Rollback plan

Revertir el commit. El método del adapter, service y handler dejan de existir; los registros con `status = CANCELLED` permanecen en BD pero no se pueden crear nuevos vía API.

## Affected surfaces

- BE: `apps/auth-users/src/admin/registration/` (controller, service, adapter — 3 archivos modificados).
- BE: tests del controller (1 archivo modificado).
- HTTP: nuevo endpoint `DELETE /pld-api/auth-users/admin/registration/:id`.
- BD: ningún cambio de schema (las columnas ya existen).

## Dependencies

Es **dependencia bloqueante** del change [add-reporting-entity-registration-ui](../add-reporting-entity-registration-ui/proposal.md). Implementar después de `add-reference-catalogs` y antes de `add-reporting-entity-registration-ui`.
