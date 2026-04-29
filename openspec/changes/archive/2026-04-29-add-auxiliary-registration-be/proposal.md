# Proposal: Add auxiliary registration endpoint (BE)

## Intent

Habilitar a notarías e inmobiliarias (`role IN (NOTARY, REAL_ESTATE)`) registrar usuarios `AUXILIARY` asociados a su cuenta. Hoy el rol `AUXILIARY` existe en el enum pero no hay flujo de alta. Bloquea la UI ya diseñada (capturas en [docs/designs/auxiliary-registration/](../../../docs/designs/auxiliary-registration/)).

Sub-repo afectado: **pld-api** únicamente.

## Scope

### In Scope
- `POST /registration/auxiliaries` — JWT, role guard `NOTARY`/`REAL_ESTATE`, DTO único, transacción única.
- Migración: tabla nueva `auxiliary_profile (id, parent_user_id, rfc, address_id, timestamps)` + index por `parent_user_id`.
- Extender enum `ProfileType` con `AUXILIARY` en `pld-api/packages/shared-types/src/enums.ts`.
- Reemplazar type `AuxiliaryRegistration` ([pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts:17](../../../pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts#L17)) con la forma completa del DTO.
- Reuso de `users` (datos personales + credenciales), `reporting_entity_address` (domicilio).
- Errores: 400/401/403/409.

### Out of Scope
- Envío de email con la password (sin servicio de mail — feature posterior, ya marcada en el diagrama BE).
- Listado/edición/borrado de auxiliares.
- Endpoint para el tipo `Clientes` (placeholder UI únicamente).
- Persistencia parcial / wizard BE (no aplica — el front maneja los 3 pasos en cliente).

## Approach

Replicar el patrón `controller → adapter → service` ya usado en `pld-api/apps/auth-users/src/admin/registration/`. Una sola request lleva todo el payload; el service ejecuta una transacción TypeORM con 3 inserts (`reporting_entity_address` → `auxiliary_profile` → `users`) más generación+hash de password con bcrypt. Validaciones (RFC, email, longitudes) reusan los DTOs existentes del módulo registration. Response devuelve `{ user, temporaryPassword }` — la password se expone una sola vez en cleartext (mismo criterio que `POST /:id/finalize` de PF/PM).

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-api/apps/auth-users/src/registration/auxiliaries/**` | New | Controller, service, adapter, DTO, módulo Nest. |
| `pld-api/packages/persistence/migrations/` | New | Migration `create-auxiliary-profile` + down. |
| `pld-api/packages/persistence/src/entities/auxiliary-profile.entity.ts` | New | TypeORM entity. |
| `pld-api/packages/shared-types/src/enums.ts` | Modified | Agregar `ProfileType.AUXILIARY`. |
| `pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts` | Modified | Reemplazar shape de `AuxiliaryRegistration`. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Romper login existente | Low | Smoke `POST /auth/login` post-merge (regla `verify`). |
| Drift con UI (campos faltantes) | Med | DTO espejado contra capturas en `docs/designs/auxiliary-registration/`; mismo change FE referencia mismo contrato. |
| `parent_user.role` inválido en runtime | Low | Validar en service antes del INSERT; 403 si no es NOTARY/REAL_ESTATE. |
| Password en logs | Med | No loguear DTO ni response; bcrypt antes de cualquier INSERT. |

## Rollback Plan

1. Revertir el merge del PR.
2. Ejecutar la migration `down` para drop de `auxiliary_profile`.
3. Eliminar `ProfileType.AUXILIARY` del enum (sin datos persistidos antes del rollback, no hay registros que limpiar).
4. Validar smoke de login y `POST /admin/registration` del flujo PF/PM.

## Dependencies

- Ninguna externa. Depende de `users` y `reporting_entity_address` ya existentes (migration `20260421000000-create-registration-schema.sql`).

## Success Criteria

- [ ] `POST /registration/auxiliaries` con JWT NOTARY/REAL_ESTATE + DTO válido → 201 con `{ user, temporaryPassword }`.
- [ ] JWT con role distinto → 403; sin JWT → 401; DTO inválido → 400; email duplicado → 409.
- [ ] Auxiliar creado tiene `users.role = 'AUXILIARY'`, `profile_type = 'AUXILIARY'`, `profile_id` apuntando a `auxiliary_profile.id`, `must_change_password = true`.
- [ ] `POST /auth/login` del auxiliar recién creado con la password temporal devuelve 201 + JWT.
- [ ] Smoke del flujo PF/PM (`POST /admin/registration`) sigue funcionando.
- [ ] `nx run cross:build` y `nx run auth-users:build` verdes.
