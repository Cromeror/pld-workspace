# Design: remove-usuario-interno-externo

## Status
DRAFT

---

## Decision 1 — Single change vs two separate changes

**Decision: single change, two sequential phases within the same PR series.**

### Rationale
The total scope is small: 5 files touched, no new modules, no new endpoints. Splitting into two changes would add ceremony (two sets of artifacts, two PRs, two deploys) without meaningful risk reduction. The deprecation phase (Phase 1) and the elimination phase (Phase 2) are better tracked as tasks within one change, with Phase 2 gated by the audit result.

### Alternative discarded
Two separate SDD changes (`remove-usuario-interno-externo-deprecate` + `remove-usuario-interno-externo-delete`). Discarded because overhead is not justified at this scope and would complicate the dependency between the data audit and code removal.

---

## Decision 2 — What to do with existing rows carrying orphan roles

**Decision: migrate to `AUXILIAR` if rows exist; block Phase 2 until migration is confirmed.**

### Rationale
`USUARIO_INTERNO` and `USUARIO_EXTERNO` were conceived as internal/external operator roles, which maps most closely to `AUXILIAR` in the active role set. Deleting those users would be destructive and may violate audit trail requirements in a PLD (anti-money laundering) system. Migrating to `AUXILIAR` preserves the account, allows the user to continue operating, and avoids data loss.

### Migration SQL (to run manually, confirmed by DBA, before Phase 2 deploy)
```sql
-- Run audit first
SELECT role, COUNT(*) AS total
FROM users
WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO')
GROUP BY role;

-- If rows found, migrate after team approval
UPDATE users
SET role = 'AUXILIAR', updated_at = NOW()
WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO');
```

### Alternatives discarded
- **Delete the users**: Destructive, may erase audit-relevant accounts in a compliance system.
- **Block login (set activo = false)**: Punishes users for an internal enum cleanup. Not justified.
- **Do nothing / carry orphan rows**: Rows with a role value not in the enum cause silent inconsistency. After Phase 2, any code path that maps `role → UserRole` (e.g., JWT payload deserialization) could fail unpredictably.

---

## Decision 3 — Column type: varchar vs MySQL ENUM

**Decision: no schema migration required. The column is `varchar`.**

### Evidence
`packages/domain-auth-users/src/entities/user.entity.ts:46`:
```typescript
@Column({ name: 'role', type: 'varchar' })
role!: UserRole;
```

The DDL in `query.sql:27` does declare the column as `ENUM(...)`, but this file is a reference/initialization script, not a TypeORM migration runner. The actual entity definition uses `varchar`. Validation is enforced in application code by `@IsEnum(UserRole)` in the DTO layer, not by the database column type.

### Implications
- Phase 2 requires no `ALTER TABLE`. Only an `UPDATE` for data migration (if orphan rows exist) and an edit to `query.sql` to keep it consistent as documentation.
- The fact that `users.role` is `varchar` means the DB has no constraint preventing invalid strings. This is a pre-existing gap noted in `temp-docs/FLUJO_REGISTRO_SUPERADMIN.md` and is out of scope for this change.

### Alternative discarded
- **Migrate column to MySQL ENUM type**: Valid long-term improvement, but adds a destructive DDL migration (`ALTER TABLE ... MODIFY COLUMN`). Out of scope; tracked as a separate gap.

---

## Decision 4 — Deprecation timeline

**Decision: ship Phase 1 and Phase 2 in the same sprint, with Phase 2 gated only by the data audit.**

### Rationale
The deprecation period exists to catch any consumer that might be sending `USUARIO_INTERNO`/`USUARIO_EXTERNO`. Since the audit confirmed no business logic uses these values, and the front-end has no UI surface for them (the Swagger example is the only API hint), a long deprecation window adds no safety. The only gate before Phase 2 is the database audit. If the audit returns 0 rows, Phase 2 can follow immediately. If rows exist, Phase 2 waits for the data migration to complete — estimated 1 day if escalated to DBA.

### Alternatives discarded
- **2-week deprecation window**: Unnecessary given confirmed no consumers.
- **1-sprint gap**: Same reasoning. No external clients depend on these values.

---

## Impact on Swagger

After Phase 2, the `@ApiProperty({ enum: UserRole })` decorators in all user DTOs that use `UserRole` will automatically reflect the reduced enum. Swagger UI will show only `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR` in the `role` dropdown. No additional decorator change is required beyond updating the `example` value in Phase 1.

Affected DTO: `CreateUserNotarioInmobiliarioDto` (line 30) uses `example: UserRole.NOTARIO` — already safe. `CreateUserInternoExternoDto` (line 72) uses `example: UserRole.USUARIO_INTERNO` — must be updated in Phase 1.

---

## Impact on Front-End

No front-end impact expected. The values `USUARIO_INTERNO` and `USUARIO_EXTERNO` were never presented in any UI flow (confirmed: no guards, no route conditions, no Swagger-generated client usage). The Swagger example update prevents future front-end developers from accidentally using the deprecated values.

---

## File Change Map

```
packages/shared-types/src/enums.ts
  Phase 1: add /** @deprecated */ above USUARIO_INTERNO and USUARIO_EXTERNO
  Phase 2: remove both lines

apps/auth-users/src/users/dto/create-user.dto.ts
  Phase 1: line 72 — change example: UserRole.USUARIO_INTERNO → UserRole.AUXILIAR

temp-docs/FLUJO_REGISTRO_SUPERADMIN.md
  Phase 1: update table row "Tipo de usuario" to remove USUARIO_INTERNO, USUARIO_EXTERNO
           update line 7 narrative reference

query.sql
  Phase 2: remove 'USUARIO_INTERNO' and 'USUARIO_EXTERNO' from ENUM(...) list (lines 32-33)

libs/catalogs/src/lib/catalogs.types.ts
  No change — re-exports UserRole by reference; picks up enum change automatically.
```

---

## Risks

- **Data migration risk (medium)**: If orphan rows exist in production, migrating them to `AUXILIAR` changes user permissions. Requires DBA sign-off and a backup before running.
- **query.sql drift (low)**: If `query.sql` is used to initialize a new environment before Phase 2 is merged, the DB would still accept the old values (because the column is `varchar`, not ENUM). This is a documentation inconsistency, not a runtime failure.
- **DTO class name confusion (low)**: `CreateUserInternoExternoDto` will retain its name after Phase 2. The class name references a concept that no longer exists in the domain. Renaming it is recommended in a follow-up change.
