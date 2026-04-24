# Tasks: remove-usuario-interno-externo

## Status
TODO

Legend: `[ ]` not started · `[x]` done · `[~]` in progress · `[!]` blocked

---

## Phase 1 — Audit (confirm blast radius before touching code)

- [x] **1.1** Run audit query against the live database and record results.
  ```sql
  SELECT role, COUNT(*) AS total
  FROM users
  WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO')
  GROUP BY role;
  ```
  - If result = 0 rows: proceed to Phase 2 — Deprecation immediately.
  - If result > 0 rows: open Design Decision 2 discussion with the team before proceeding. Phase 2 is BLOCKED until migration strategy is approved.
  - Record result in a comment on the PR or in the team's Slack channel for traceability.
  - Est: 15 min

- [x] **1.2** Confirm no TypeScript code (guards, services, interceptors) branches on `USUARIO_INTERNO` or `USUARIO_EXTERNO` using a codebase search.
  ```
  Search pattern: USUARIO_INTERNO|USUARIO_EXTERNO
  Scope: apps/, libs/, packages/ (exclude openspec/, docs/, query.sql)
  ```
  - Expected: only hits in `enums.ts` and `create-user.dto.ts`.
  - If unexpected hits found: escalate before proceeding — this change's scope may need to expand.
  - Est: 15 min

---

## Phase 2 — Deprecation (soft removal — can merge to develop immediately)

- [x] **2.1** Add `/** @deprecated Use SUPERADMIN, NOTARIO, INMOBILIARIA, or AUXILIAR instead. Will be removed in Phase 2. */` JSDoc above `USUARIO_INTERNO` and `USUARIO_EXTERNO` in `packages/shared-types/src/enums.ts`.
  - File: `packages/shared-types/src/enums.ts`
  - Est: 15 min

- [x] **2.2** Update Swagger example in `CreateUserInternoExternoDto` — change `example: UserRole.USUARIO_INTERNO` to `example: UserRole.AUXILIAR`.
  - File: `apps/auth-users/src/users/dto/create-user.dto.ts`, line 72
  - Est: 10 min

- [x] **2.3** Update `temp-docs/FLUJO_REGISTRO_SUPERADMIN.md`:
  - Remove `USUARIO_INTERNO`, `USUARIO_EXTERNO` from the "Tipos de usuario del sistema" bullet (line 7).
  - Update the enum table row for `UserRole` (line 24) to list only the 4 active values.
  - Est: 20 min

- [x] **2.4** Verify TypeScript compiles with no errors after 2.1–2.3.
  ```
  nx run auth-users:build
  nx run catalogs:build
  nx run cross:build
  ```
  - Est: 10 min

- [x] **2.5** Manual smoke test: `POST /auth/login` with a valid `NOTARIO` credential → expect HTTP 201 + JWT.
  - Est: 10 min

---

## Phase 3 — Data Migration (only if audit in 1.1 found rows; skip otherwise)

- [x] **3.1** Take a full backup of the `users` table before migration.
  ```sql
  CREATE TABLE users_backup_pre_role_migration AS SELECT * FROM users;
  ```
  - Est: 10 min (+ backup time depending on table size)

- [x] **3.2** Run migration with team and DBA approval:
  ```sql
  UPDATE users
  SET role = 'AUXILIAR', updated_at = NOW()
  WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO');
  ```
  - Est: 10 min

- [x] **3.3** Verify migration: confirm 0 rows remain with orphan roles.
  ```sql
  SELECT COUNT(*) FROM users WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO');
  -- Expected: 0
  ```
  - Est: 5 min

- [x] **3.4** Notify affected users (if any) that their role was changed to `AUXILIAR`, per team's communication policy.
  - Est: 30 min (depends on team process)

---

## Phase 4 — Elimination (hard removal — requires Phase 1 + Phase 3 complete)

- [x] **4.1** Remove `USUARIO_INTERNO` and `USUARIO_EXTERNO` lines from `UserRole` enum.
  - File: `packages/shared-types/src/enums.ts`
  - Est: 10 min

- [x] **4.2** Update `query.sql` DDL reference: remove `'USUARIO_INTERNO'` and `'USUARIO_EXTERNO'` from the ENUM value list in the `users` table definition (lines 32–33).
  - File: `query.sql`
  - Note: this is a documentation update. No migration runner executes this file.
  - Est: 10 min

- [x] **4.3** Verify TypeScript compiles with no errors after 4.1–4.2.
  ```
  nx run auth-users:build
  nx run catalogs:build
  nx run cross:build
  ```
  - Est: 10 min

- [x] **4.4** Verify Swagger: start `auth-users` in dev mode and confirm `role` enum in `POST /system-users/*` shows only 4 active values. `USUARIO_INTERNO` and `USUARIO_EXTERNO` must not appear.
  - Est: 10 min

---

## Phase 5 — Verification (post-deploy checks)

- [x] **5.1** Smoke test: `POST /auth/login` with credentials for each active role (`SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR`) → expect HTTP 201 + JWT for each.
  - Est: 15 min

- [x] **5.2** Negative test: `POST /system-users/interno-externo` (or equivalent endpoint) with `role: "USUARIO_INTERNO"` → expect HTTP 400 with validation error.
  - Est: 10 min

- [x] **5.3** Negative test: same as 5.2 but with `role: "USUARIO_EXTERNO"` → expect HTTP 400.
  - Est: 10 min

- [x] **5.4** Database verification:
  ```sql
  SELECT COUNT(*) FROM users WHERE role IN ('USUARIO_INTERNO', 'USUARIO_EXTERNO');
  -- Expected: 0
  ```
  - Est: 5 min

- [x] **5.5** Optional follow-up: open a separate issue / SDD change to rename `CreateUserInternoExternoDto` to a name that reflects the current domain (e.g. `CreateAuxiliarUserDto`), and to evaluate migrating `users.role` from `varchar` to a MySQL ENUM type.
  - Est: 10 min (just filing the issue)

---

## Summary

| Phase | Tasks | Est. Total | Blocking Condition |
|---|---|---|---|
| 1 — Audit | 1.1, 1.2 | ~30 min | None |
| 2 — Deprecation | 2.1–2.5 | ~65 min | Phase 1 complete |
| 3 — Data Migration | 3.1–3.4 | ~55 min (+ DBA time) | Phase 1 found rows |
| 4 — Elimination | 4.1–4.4 | ~40 min | Phase 1 + Phase 3 complete |
| 5 — Verification | 5.1–5.5 | ~50 min | Phase 4 deployed |
| **Total** | | **~4h** | |
