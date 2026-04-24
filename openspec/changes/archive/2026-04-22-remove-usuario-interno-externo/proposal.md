# Proposal: remove-usuario-interno-externo

## Status
PROPOSED

## Why

`UserRole.USUARIO_INTERNO` and `UserRole.USUARIO_EXTERNO` are orphan values in the enum. No guard, role-check, business logic, or endpoint conditions distinguish them from the four active roles (`SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR`). They exist in the enum definition, the Swagger example of `CreateUserInternoExternoDto`, and the DDL reference in `query.sql`, but they are never enforced or queried.

Keeping them increases the public API surface unnecessarily, misleads front-end developers reading Swagger, and creates confusion in a compliance-sensitive domain (PLD).

## What Changes

### Phase 1 — Deprecation (soft removal)

1. Mark `USUARIO_INTERNO` and `USUARIO_EXTERNO` with a `/** @deprecated */` JSDoc comment in `packages/shared-types/src/enums.ts`.
2. Update the Swagger `example` in `CreateUserInternoExternoDto` (`apps/auth-users/src/users/dto/create-user.dto.ts`) to use `UserRole.AUXILIAR` instead.
3. Run audit query against the live database:
   ```sql
   SELECT role, COUNT(*) AS total
   FROM users
   WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO')
   GROUP BY role;
   ```
   If rows exist, the team must decide the migration strategy (see Design Decision 2) before Phase 2 can proceed.
4. Update `temp-docs/FLUJO_REGISTRO_SUPERADMIN.md` to remove references to the deprecated values.

### Phase 2 — Elimination (hard removal)

1. Remove `USUARIO_INTERNO` and `USUARIO_EXTERNO` from the `UserRole` enum in `packages/shared-types/src/enums.ts`.
2. Migrate any existing `users` rows with those roles (if the audit found any) via a targeted `UPDATE` statement.
3. `query.sql` is a DDL reference file, not a migration runner — update it to reflect the new valid values.
4. No `ALTER TABLE` migration is required because `users.role` is `varchar`, not a MySQL ENUM type (confirmed in `packages/domain-auth-users/src/entities/user.entity.ts:46`).

## Apps / Libs / Packages Affected

| Artifact | Change |
|---|---|
| `packages/shared-types/src/enums.ts` | Phase 1: add `@deprecated`. Phase 2: remove values. |
| `libs/catalogs/src/lib/catalogs.types.ts` | Re-exports `UserRole` — no code change needed; picks up enum change automatically. |
| `apps/auth-users/src/users/dto/create-user.dto.ts` | Phase 1: update Swagger `example` in `CreateUserInternoExternoDto`. |
| `query.sql` | Phase 2: remove `'USUARIO_INTERNO'`, `'USUARIO_EXTERNO'` from ENUM list comment. |
| `temp-docs/FLUJO_REGISTRO_SUPERADMIN.md` | Phase 1: update narrative references. |

## Alternatives Discarded

- **Keep them "just in case"**: No justification exists. No existing user, guard, or query uses them. Keeping them adds confusion and false optionality in a regulated domain.
- **Eliminate directly without deprecation (single phase)**: Riskier if any production records carry those roles. A soft deprecation + data audit before hard removal is safer and adds zero significant time given how small the change is.

## Rollback Plan

- **Phase 1 rollback**: `git revert` the commit. No data migration involved.
- **Phase 2 rollback** (if no data migration was run): `git revert` the commit. Because `role` is `varchar`, TypeORM will accept the old values again as soon as the enum is restored.
- **Phase 2 rollback** (if data migration ran): `git revert` the commit AND manually restore the migrated rows to their original role value using a backup taken before migration. This must be documented in the deploy runbook.

## Post-Change Validation

1. `POST /auth/login` with a valid `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, or `AUXILIAR` credential returns HTTP 201 + JWT.
2. `POST /system-users/*` with `role: "USUARIO_INTERNO"` or `role: "USUARIO_EXTERNO"` returns HTTP 400 (after Phase 2).
3. `SELECT COUNT(*) FROM users WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO')` returns 0.
4. Swagger UI on `POST /system-users/*` does not list `USUARIO_INTERNO` or `USUARIO_EXTERNO` in the `role` enum dropdown.
