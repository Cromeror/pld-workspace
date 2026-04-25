# Tasks: Discard registration endpoint

## Phase 1 — Adapter

- [x] 1.1 Editar `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts`. Agregar método `softDeleteRegistration(id)`.
- [x] 1.2 Verificar que `RegistrationStatus.CANCELLED` exista en el enum.  *(ya existía en `registration.enums.ts:4`)*
- [x] 1.3 Verificar que la columna `deleted_at` esté en `RegistrationEntity`.  *(ya existía en `entities/registration.entity.ts:69-70`, prevista por el comentario "reserved for future DELETE endpoint")*

## Phase 2 — Service

- [x] 2.1 Editar `pld-api/apps/auth-users/src/admin/registration/registration.service.ts`. Agregar método `discard(id, currentUser)`.
- [x] 2.2 Patrón sigue al del resto del service (params: `id, _currentUser`; return: `Promise<void>`).

## Phase 3 — Controller

- [x] 3.1 Editar `pld-api/apps/auth-users/src/admin/registration/registration.controller.ts`. Agregar handler `@Delete(':id')` con `@HttpCode(204)` y ApiResponse decorators.
- [x] 3.2 El controller ya tiene `@UseGuards(JwtAuthGuard, RolesGuard) @Roles(UserRole.SUPERADMIN)` a nivel de clase. El nuevo handler hereda esos guards automáticamente.

## Phase 4 — Tests

- [x] 4.1 Agregados tests al nuevo `registration.service.spec.ts` (preferí spec del service donde vive la lógica, no del controller que solo delega). Cubre 5 casos: not-found 404, idempotente CANCELLED, conflict COMPLETED, expired 410, soft-delete IN_PROGRESS exitoso.
- [ ] 4.2 (Opcional) Test del adapter validando el UPDATE.  *(skipped — el call a `softDeleteRegistration` ya está cubierto en el spec del service via mock; un test integration del adapter requiere setup TypeORM real, fuera de scope)*

## Phase 5 — Validación local

- [x] 5.1 `cd pld-api && pnpm nx run auth-users:test` pasa.  *(29/29 tests passing — 24 catalogs + 5 nuevos service)*
- [ ] 5.2 Levantar BE: `/stack-up --no-web`.  *(pendiente: smoke manual con stack levantado)*
- [ ] 5.3 Hacer login como SUPERADMIN y obtener JWT.  *(pendiente smoke)*
- [ ] 5.4 Crear draft: `POST /admin/registration { profileType, userRole, rfc }` → guardar id.  *(pendiente smoke)*
- [ ] 5.5 Descartar: `curl -X DELETE -H "Authorization: Bearer <jwt>" .../admin/registration/<id>` → 204.  *(pendiente smoke)*
- [ ] 5.6 Verificar BD: `SELECT status, deleted_at FROM registration WHERE id=<id>` → `CANCELLED, <timestamp>`.  *(pendiente smoke)*
- [ ] 5.7 Reintentar `POST /admin/registration` con el mismo RFC → 201 (no 409, RFC liberado).  *(pendiente smoke)*
- [ ] 5.8 DELETE el mismo draft otra vez → 204 idempotente.  *(pendiente smoke)*

## Phase 6 — Specs delta

- [ ] 6.1 Aplicar el delta de [specs.md](specs.md) a `openspec/specs/...` durante `/sdd-archive`.

## Phase 7 — Commit

- [ ] 7.1 Commit BE: `feat(admin-registration): add DELETE endpoint for draft discard`.
- [ ] 7.2 (Opcional) `/sdd-archive add-discard-registration-endpoint`.
