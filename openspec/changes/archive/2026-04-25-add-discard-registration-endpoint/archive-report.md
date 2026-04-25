# Archive Report: add-discard-registration-endpoint

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & SMOKE-TESTED
**Sub-repo**: pld-api (BE-only)

## Summary

Endpoint `DELETE /admin/registration/:id` para descartar borradores (soft-delete). Marca `status = CANCELLED` y `deleted_at = NOW()`, liberando el RFC para reuso. Bloqueante para Q21 del wizard PF/PM (cambio de profileType con draft activo).

## Endpoint delivered

```
DELETE /pld-api/auth-users/admin/registration/:id
```

| Status | Caso |
|---|---|
| 204 No Content | Soft-delete exitoso |
| 204 No Content | Idempotente (ya CANCELLED) |
| 404 Not Found | Registration no existe |
| 409 Conflict | Status COMPLETED (no se puede descartar finalizado) |
| 410 Gone | Expirado por TTL |
| 401 Unauthorized | Sin JWT |
| 403 Forbidden | Rol distinto a SUPERADMIN |

Soft-delete preserva auditoría: el row permanece en BD con `status = CANCELLED` y `deleted_at`, accesible por consulta directa.

## BE files (modified)

- `apps/auth-users/src/admin/registration/registration.adapter.ts` — agregado método `softDeleteRegistration(id)`.
- `apps/auth-users/src/admin/registration/registration.service.ts` — agregado método `discard(id, currentUser)` con validaciones 404/409/410 + idempotencia.
- `apps/auth-users/src/admin/registration/registration.controller.ts` — agregado handler `@Delete(':id') @HttpCode(NO_CONTENT)` con 7 ApiResponse decorators. Hereda `JwtAuthGuard + RolesGuard + @Roles(SUPERADMIN)` a nivel de clase.

### New
- `apps/auth-users/src/admin/registration/registration.service.spec.ts` — 5 tests del método `discard` con mock del adapter (404, idempotente CANCELLED, 409 COMPLETED, 410 expired, 204 soft-delete exitoso).

## Tests

`pnpm nx run auth-users:test`: 29/29 tests passing (24 catalogs + 5 nuevos service).

## Decisiones aplicadas

- **D1**: soft-delete sobre hard-delete. El schema ya tenía previstas las columnas `status = CANCELLED` y `deleted_at` (REGISTRO_SCHEMA.md).
- **D3**: 204 No Content sin body — patrón estándar REST para DELETE.
- **D4**: matriz de validación de estado (404/409/410/idempotencia 204).
- **D5**: auth `@Roles(SUPERADMIN)` heredado del controller.
- **D8**: sin auditoría adicional `cancelled_by_user_id` en v1. El draft ya tiene `started_by_user_id`.
- **D9**: sin endpoint de "restaurar" — out of scope v1.

## Verificación post-deploy

`curl` smoke tests:
- POST /admin/registration → 201 + draft id
- DELETE /:id → 204
- DELETE /:id otra vez → 204 idempotente (no crea ruido)
- POST /admin/registration con mismo RFC → 201 (RFC liberado, dedupe filtra solo IN_PROGRESS)
- DELETE /:id con UUID inexistente → 404
- DELETE /:id sin JWT → 401

## Specs delta

Nuevo archivo `openspec/specs/admin-registration/spec.md` con:
- Requirement: Endpoint DELETE para descartar drafts (8 scenarios).
- Requirement: Soft-delete preserva auditoría (1 scenario).

## Desviación menor

El design proponía "test del controller". Implementación real: tests del **service** (donde vive la lógica). Razón: el controller solo delega (`return this.service.discard(id, user)`); testear el service con mock del adapter cubre los 5 escenarios HTTP sin levantar el módulo NestJS completo. Práctica habitual en NestJS cuando el controller es thin wrapper.

## Commits

- `pld-api` (develop): `259de7d feat(admin-registration): add DELETE endpoint for draft discard`
