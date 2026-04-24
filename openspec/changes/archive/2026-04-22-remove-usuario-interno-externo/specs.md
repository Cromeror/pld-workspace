# Specs: remove-usuario-interno-externo

## Status
DRAFT

## Scope

This spec covers the behavioral contract changes introduced by removing `USUARIO_INTERNO` and `USUARIO_EXTERNO` from the `UserRole` enum in two phases. It is a delta spec — it does not re-describe the full user creation flow, only the changes from it.

## Definitions

- **Phase 1**: Deprecation period. Values remain in the enum but are marked `@deprecated`. Existing behavior is preserved. Swagger example is updated.
- **Phase 2**: Hard removal. Values are no longer part of the enum. Any request carrying them MUST be rejected.
- **Orphan roles**: `USUARIO_INTERNO`, `USUARIO_EXTERNO` — values with no associated business logic, no guards, and no queries.
- **Active roles**: `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR`.

---

## Requirements

### R-01 — Phase 1: Swagger example MUST NOT use deprecated values

After Phase 1 is merged, the Swagger `example` for the `role` field in `CreateUserInternoExternoDto` MUST be an active role (e.g. `AUXILIAR`). The `enum` dropdown in Swagger MAY still show `USUARIO_INTERNO` and `USUARIO_EXTERNO` during Phase 1 (they remain in the TypeScript enum).

### R-02 — Phase 2: Enum contract change

After Phase 2 is merged, the `UserRole` enum SHALL contain exactly four values: `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR`. The values `USUARIO_INTERNO` and `USUARIO_EXTERNO` SHALL NOT appear.

### R-03 — Phase 2: Rejection of deprecated values at API boundary

After Phase 2 is merged, any request to a user-creation endpoint (`POST /system-users/*`) that carries `role: "USUARIO_INTERNO"` or `role: "USUARIO_EXTERNO"` MUST be rejected with HTTP 400. This is enforced automatically by `@IsEnum(UserRole)` once the values are removed.

### R-04 — Phase 2: No orphan rows at rest

Before Phase 2 is deployed, it MUST be confirmed (via audit query) that no `users` row carries `role = 'USUARIO_INTERNO'` or `role = 'USUARIO_EXTERNO'`. If orphan rows are found, they MUST be migrated or removed according to Design Decision 2 before Phase 2 ships.

### R-05 — Login continuity

Active users with roles `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, or `AUXILIAR` MUST be able to authenticate successfully (HTTP 201 + JWT) after both phases.

### R-06 — Docs updated

`temp-docs/FLUJO_REGISTRO_SUPERADMIN.md` MUST be updated to remove all references to `USUARIO_INTERNO` and `USUARIO_EXTERNO` in Phase 1.

---

## Scenarios

### Scenario 1 — Phase 1: Request with deprecated role still accepted

```
Given Phase 1 has been deployed (values are @deprecated but still in the enum)
When a client sends POST /system-users/interno-externo with role: "USUARIO_INTERNO"
Then the server returns HTTP 201 (class-validator still accepts the value)
And the Swagger UI example for the role field shows "AUXILIAR" (not "USUARIO_INTERNO")
```

### Scenario 2 — Phase 2: Request with removed role is rejected

```
Given Phase 2 has been deployed (values removed from UserRole enum)
When a client sends POST /system-users/interno-externo with role: "USUARIO_INTERNO"
Then the server returns HTTP 400
And the response body contains a validation error indicating role is invalid
```

### Scenario 3 — Phase 2: Request with removed role is rejected (USUARIO_EXTERNO)

```
Given Phase 2 has been deployed (values removed from UserRole enum)
When a client sends POST /system-users/interno-externo with role: "USUARIO_EXTERNO"
Then the server returns HTTP 400
And the response body contains a validation error indicating role is invalid
```

### Scenario 4 — Login continuity for active roles

```
Given Phase 2 has been deployed
When a user with role NOTARIO sends POST /auth/login with valid credentials
Then the server returns HTTP 201
And the response includes a valid JWT
```

### Scenario 5 — Swagger enum updated after Phase 2

```
Given Phase 2 has been deployed
When a developer opens the Swagger UI for POST /system-users/*
Then the role field enum list shows only: SUPERADMIN, NOTARIO, INMOBILIARIA, AUXILIAR
And USUARIO_INTERNO and USUARIO_EXTERNO do not appear
```

### Scenario 6 — Audit gate before Phase 2

```
Given the audit query is run: SELECT COUNT(*) FROM users WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO')
When the result is greater than 0
Then Phase 2 MUST NOT be deployed until a data migration plan is approved and executed
```

---

## Out of Scope

- Adding new roles to `UserRole` — separate change.
- Changing `users.role` from `varchar` to a MySQL ENUM type — separate change (noted as a gap in `FLUJO_REGISTRO_SUPERADMIN.md`).
- Changes to `CreateUserInternoExternoDto` class name or endpoint — separate change.
