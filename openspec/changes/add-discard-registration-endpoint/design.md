# Design: Discard registration endpoint

## Decisions

### D1 — Soft-delete sobre hard-delete

El schema ([REGISTRO_SCHEMA.md](../../../docs/REGISTRO_SCHEMA.md)) tiene previstas:
- `status` enum con valor `CANCELLED`.
- `deleted_at` columna nullable timestamp.

El adapter actualiza ambos campos en lugar de hacer DELETE FROM. Razones:
- **Auditoría**: queda registro de qué drafts se cancelaron y cuándo.
- **Trazabilidad**: si en el futuro hay sospecha de comportamiento anómalo (cancelaciones masivas, abuso), la información está.
- **Coherente con el diseño previsto**: el schema lo anticipó; honramos esa decisión.

### D2 — HTTP DELETE idiomático

El verbo HTTP correcto para "descartar/borrar" un recurso es `DELETE`, aunque internamente sea soft-delete. Coherente con el resto de la API.

### D3 — Status code: 204 No Content

Sin body en respuesta exitosa. Patrón estándar de REST para DELETE soft.

### D4 — Validaciones de estado

| Condición | Status | Razón |
|---|---|---|
| Registration no existe | 404 | Idempotente: borrar algo que no existe no debe crear ruido |
| `status = COMPLETED` | 409 | No se puede descartar un draft ya finalizado (rompe integridad: el `user` ya fue creado) |
| `status = CANCELLED` (ya fue descartado antes) | 204 | Idempotente: descartar un draft ya descartado es no-op exitoso |
| `expires_at < NOW` | 410 Gone | Coherente con el comportamiento del GET; el draft ya no está accesible |
| `status = IN_PROGRESS`, no expirado | 204 | Caso normal: marca CANCELLED + deleted_at = NOW |

### D5 — Auth: SUPERADMIN

Mismo patrón que el resto del módulo `admin/registration`. El `RolesGuard` ya está cableado:

```ts
@Delete(':id')
@HttpCode(HttpStatus.NO_CONTENT)
@Roles(UserRole.SUPERADMIN)
@ApiOperation({ summary: 'Discard registration draft (soft-delete)' })
@ApiResponse({ status: 204, description: 'Draft discarded' })
@ApiResponse({ status: 401, description: 'Unauthorized' })
@ApiResponse({ status: 403, description: 'Forbidden — requires SUPERADMIN' })
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

### D6 — `RegistrationService.discard`

```ts
async discard(id: string, _currentUser: { id: string }): Promise<void> {
  const registration = await this.adapter.findRegistrationById(id);

  // Si no existe → 404
  if (!registration) {
    throw new NotFoundException('Registration not found');
  }

  // Si ya está cancelado → idempotente, retornar 204
  if (registration.status === RegistrationStatus.CANCELLED) {
    return;
  }

  // Si está completed → 409
  if (registration.status === RegistrationStatus.COMPLETED) {
    throw new ConflictException('Cannot discard a finalized registration');
  }

  // Si expiró → 410
  if (registration.expiresAt < new Date()) {
    throw new HttpException(
      { message: 'Registration draft has expired', statusCode: HttpStatus.GONE },
      HttpStatus.GONE,
    );
  }

  // Caso normal: soft-delete
  await this.adapter.softDeleteRegistration(id);
}
```

### D7 — `RegistrationAdapter.softDeleteRegistration`

```ts
async softDeleteRegistration(id: string): Promise<void> {
  await this.registrationRepo.update(id, {
    status: RegistrationStatus.CANCELLED,
    deletedAt: new Date(),
  });
}
```

Sin transacción adicional — el UPDATE es atómico. El adapter ya tiene patrones similares para otros updates de `registration`.

### D8 — Sin auditoría adicional en v1

Por ahora no se registra `cancelled_by_user_id` ni `cancellation_reason`. El draft ya tiene `started_by_user_id` que indica quién lo creó. Si se necesita rastrear quién canceló, agregar columna en futuro change sin romper compat.

### D9 — Sin endpoint de "restaurar"

Una vez cancelado, el draft no se puede restaurar vía API. Si se necesita revivir un draft, queda como tarea de admin DB (UPDATE manual). v1 no expone esto para evitar abuso.

### D10 — Listado: comportamiento existente

El listado (`GET /admin/registration`) ya filtra `status != CANCELLED` por default. No se altera. Si se quiere ver cancelados en el futuro, agregar query param `?status=CANCELLED` (ya soportado por el current `ListRegistrationsQueryDto`).

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| RFC bloqueado por draft cancelado | Bajo | `POST /admin/registration` ya filtra IN_PROGRESS para dedupe; CANCELLED no bloquea |
| Borrado accidental por SUPERADMIN | Medio (pérdida de progreso del usuario) | Soft-delete preserva la data; UI confirma con `ConfirmDialog` antes de llamar |
| Race con finalize concurrente | Bajo (mismo registration) | El service valida estado antes de actualizar; si finalize gana, discard devuelve 409 |
| Performance: UPDATE indexado | Trivial | id es PK |

## Open questions

- ¿Soft-delete se cascadea a `contact`, `vulnerable_activity`, `physical_person_profile`, etc.? Por ahora NO — esos rows quedan huérfanos referenciados por un registration cancelled. No molesta porque el listado y getDetail filtran CANCELLED.
- ¿El TTL aplica a CANCELLED? El `expires_at` se mantiene; un cron de limpieza eventual podría borrar definitivo los CANCELLED expirados. Out of scope v1.
