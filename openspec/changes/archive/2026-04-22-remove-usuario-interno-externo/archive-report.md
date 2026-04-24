# Archive Report: remove-usuario-interno-externo

**Closed**: 2026-04-22
**Status**: IMPLEMENTED & SMOKE-TESTED

## Summary

Removed the orphan enum values `USUARIO_INTERNO` and `USUARIO_EXTERNO` from `UserRole`. Audit confirmed zero production code paths branching on them. Only 3 test users existed in dev DB (emails with timestamp suffixes, created during earlier exploration) — deleted outright (Option B) rather than migrated to `AUXILIAR`.

## Implemented files

### Modified
- `packages/shared-types/src/enums.ts` — removed the two orphan values from `UserRole`.
- `apps/auth-users/src/users/dto/create-user.dto.ts` — Swagger `example` updated from `USUARIO_INTERNO` to `AUXILIAR`.
- `query.sql` — removed the two values from the `users.role` MySQL native ENUM.

### New
- `packages/persistence/migrations/20260422000000-remove-usuario-interno-externo.sql` — `ALTER TABLE users MODIFY COLUMN role ENUM(...)` with 4 values.
- `packages/persistence/migrations/20260422000000-remove-usuario-interno-externo.down.sql` — reverse migration adding the 2 values back.

## Deviation from plan

**Phase 2 (Deprecation) skipped**. Rationale: audit found only 3 dev-DB test users with `USUARIO_INTERNO`, no production references in code. The soft-deprecation step (adding `@deprecated` JSDoc, updating docs) exists to protect ongoing integrations during a deprecation window. With no such integrations, going straight to Phase 4 (removal) was safe.

**Phase 3 (Data Migration) resolved via Option B (delete)** instead of the default Option A (migrate to `AUXILIAR`). The 3 users had timestamp-generated emails (`x1776796641@t.com`, `rfc1776797049@t.com`, `i1776796641@t.com`), created 2026-04-21 during test flows — no business value in preserving them.

**Critical caveat for production**: when this change runs in a real environment, re-run the audit (Phase 1.1). If production has real users with these roles, **Option A (migrate to AUXILIAR)** is still the recommended default — do not blindly replicate the Option B used in dev.

## Verification results (2026-04-22)

| Test | Result |
|---|---|
| `tsc --noEmit` after enum change | ✅ clean |
| `POST /auth/login admin@pld.com/Admin123!` | ✅ HTTP 201 + JWT |
| `POST /system-users/interno-externo-auxiliar role: "USUARIO_INTERNO"` | ✅ HTTP 400 with validator message listing only 4 valid roles |
| `POST /system-users/interno-externo-auxiliar role: "USUARIO_EXTERNO"` | ✅ HTTP 400 (same) |
| `POST /system-users/interno-externo-auxiliar role: "AUXILIAR"` | ✅ HTTP 201 (regression OK) |
| `SELECT COUNT(*) FROM users WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO')` after migration | ✅ 0 |
| MySQL ENUM reflects only 4 values in `DESCRIBE users` | ✅ |

## Follow-ups flagged

- **Task 5.5 (optional)**: rename `CreateUserInternoExternoDto` → `CreateAuxiliarUserDto`. The class name no longer reflects its domain meaning. Trivial rename, not blocking — file a separate issue.
- **Decision 3 follow-up**: evaluate migrating `users.role` from native MySQL ENUM to `varchar` with app-layer validation, matching the convention used by the new `registration` tables. The current mix (ENUM for `users.role`, `varchar` for new tables) is inconsistent.
- The `rename-to-english` change will further consolidate naming — `UserRole` values (`NOTARIO`, `INMOBILIARIA`) will be revisited there.
