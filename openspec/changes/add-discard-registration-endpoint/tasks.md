# Tasks: Discard registration endpoint

## Phase 1 — Adapter

- [ ] 1.1 Editar `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts`. Agregar método:
  ```ts
  async softDeleteRegistration(id: string): Promise<void> {
    await this.registrationRepo.update(id, {
      status: RegistrationStatus.CANCELLED,
      deletedAt: new Date(),
    });
  }
  ```
- [ ] 1.2 Verificar que `RegistrationStatus.CANCELLED` exista en el enum del adapter o entity. Si no, importarlo.
- [ ] 1.3 Verificar que la columna `deleted_at` esté en `RegistrationEntity`. Si no, agregarla con `@Column({ name: 'deleted_at', type: 'timestamp', nullable: true })`. Verificar también que el schema BD ya la tenga (debe estar — está en `REGISTRO_SCHEMA.md` y en la migración existente).

## Phase 2 — Service

- [ ] 2.1 Editar `pld-api/apps/auth-users/src/admin/registration/registration.service.ts`. Agregar método `discard(id, currentUser)`:
  - findRegistrationById.
  - Si null → NotFoundException.
  - Si status === CANCELLED → return (idempotente).
  - Si status === COMPLETED → ConflictException.
  - Si expires_at < new Date() → HttpException 410.
  - Sino → adapter.softDeleteRegistration(id).
- [ ] 2.2 Asegurarse que el patrón sigue al del resto del service (params: `id, currentUser`; return: `Promise<void>`).

## Phase 3 — Controller

- [ ] 3.1 Editar `pld-api/apps/auth-users/src/admin/registration/registration.controller.ts`. Agregar handler:
  ```ts
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Discard registration draft (soft-delete)' })
  @ApiParam({ name: 'id', description: 'Registration UUID' })
  @ApiResponse({ status: 204, description: 'Draft discarded' })
  @ApiResponse({ status: 404, description: 'Not found' })
  @ApiResponse({ status: 409, description: 'Cannot discard a finalized draft' })
  @ApiResponse({ status: 410, description: 'Draft expired' })
  discard(
    @Param('id', ParseUUIDPipe) id: string,
    @CurrentUser() user: { id: string },
  ): Promise<void> {
    return this.service.discard(id, user);
  }
  ```
- [ ] 3.2 El controller ya tiene `@UseGuards(JwtAuthGuard, RolesGuard) @Roles(UserRole.SUPERADMIN)` a nivel de clase. El nuevo handler hereda esos guards automáticamente.

## Phase 4 — Tests

- [ ] 4.1 Agregar tests al `registration.controller.spec.ts` (o crearlo si no existe). Cubrir 5 casos:
  - Descarte exitoso devuelve 204.
  - Idempotente con CANCELLED.
  - 409 con COMPLETED.
  - 410 con expirado.
  - 404 con id inexistente.
- [ ] 4.2 (Opcional) Test del adapter validando el UPDATE.

## Phase 5 — Validación local

- [ ] 5.1 `cd pld-api && pnpm nx run auth-users:test` pasa.
- [ ] 5.2 Levantar BE: `/stack-up --no-web`.
- [ ] 5.3 Hacer login como SUPERADMIN y obtener JWT.
- [ ] 5.4 Crear draft: `POST /admin/registration { profileType, userRole, rfc }` → guardar id.
- [ ] 5.5 Descartar: `curl -X DELETE -H "Authorization: Bearer <jwt>" .../admin/registration/<id>` → 204.
- [ ] 5.6 Verificar BD: `SELECT status, deleted_at FROM registration WHERE id=<id>` → `CANCELLED, <timestamp>`.
- [ ] 5.7 Reintentar `POST /admin/registration` con el mismo RFC → 201 (no 409, RFC liberado).
- [ ] 5.8 DELETE el mismo draft otra vez → 204 idempotente.

## Phase 6 — Specs delta

- [ ] 6.1 Aplicar el delta de [specs.md](specs.md) a `openspec/specs/...` durante `/sdd-archive`.

## Phase 7 — Commit

- [ ] 7.1 Commit BE: `feat(admin-registration): add DELETE endpoint for draft discard`.
- [ ] 7.2 (Opcional) `/sdd-archive add-discard-registration-endpoint`.
