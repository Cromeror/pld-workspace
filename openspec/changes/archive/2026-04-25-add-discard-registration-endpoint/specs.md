# Specs: Discard registration endpoint

Nuevo Requirement para el módulo `admin/registration`. Si existe un spec del módulo, se extiende; si no, este change crea uno bajo `openspec/specs/admin-registration/spec.md` durante el archive.

## Requirements added

### Requirement: Endpoint DELETE para descartar drafts

El sistema DEBE exponer `DELETE /admin/registration/:id` que marca el draft como `CANCELLED` (soft-delete).

#### Scenario: Descarte exitoso de draft IN_PROGRESS

- GIVEN un draft con `status = IN_PROGRESS` y no expirado
- AND un cliente autenticado con rol `SUPERADMIN`
- WHEN envía `DELETE /pld-api/auth-users/admin/registration/{id}`
- THEN responde HTTP 204 No Content sin body
- AND la entity en BD tiene `status = CANCELLED` y `deleted_at` con timestamp actual

#### Scenario: Idempotencia con draft ya cancelado

- GIVEN un draft con `status = CANCELLED`
- WHEN un SUPERADMIN envía DELETE al mismo id
- THEN responde HTTP 204 No Content
- AND no se hacen escrituras adicionales en BD

#### Scenario: Conflicto con draft completado

- GIVEN un draft con `status = COMPLETED` (ya finalizado, user creado)
- WHEN un SUPERADMIN envía DELETE
- THEN responde HTTP 409 Conflict
- AND el body incluye mensaje "Cannot discard a finalized registration"

#### Scenario: Draft expirado

- GIVEN un draft con `expires_at < NOW`
- WHEN un SUPERADMIN envía DELETE
- THEN responde HTTP 410 Gone

#### Scenario: Draft no encontrado

- GIVEN un id que no corresponde a ningún registration
- WHEN un SUPERADMIN envía DELETE
- THEN responde HTTP 404 Not Found

#### Scenario: Sin autenticación

- GIVEN cliente sin header Authorization válido
- WHEN envía DELETE
- THEN responde HTTP 401 Unauthorized

#### Scenario: Rol insuficiente

- GIVEN cliente autenticado con rol `NOTARIO` (o cualquiera distinto de SUPERADMIN)
- WHEN envía DELETE a un draft existente
- THEN responde HTTP 403 Forbidden

#### Scenario: Liberación de RFC

- GIVEN un draft con `rfc = X` y `status = IN_PROGRESS`
- WHEN se descarta vía DELETE (status pasa a CANCELLED)
- AND posteriormente un SUPERADMIN intenta `POST /admin/registration` con el mismo `rfc = X`
- THEN responde HTTP 201 (no 409), porque el dedupe filtra solo IN_PROGRESS

### Requirement: Soft-delete preserva auditoría

El registro en BD NO se elimina físicamente al descartar. Debe permanecer accesible mediante consulta directa a BD para fines de auditoría.

#### Scenario: Row persiste tras DELETE

- GIVEN un draft descartado vía endpoint
- WHEN se ejecuta `SELECT * FROM registration WHERE id = ?`
- THEN se devuelve el row con `status = CANCELLED` y `deleted_at IS NOT NULL`
