# Design: rename-to-english

## Overview

This document captures the architectural decisions for migrating all Spanish identifiers in `pld-api` to English. It covers five key decisions that shape implementation risk, coordination requirements, and rollback complexity.

---

## Decision 1 — Big Bang vs Dual Support

**Chosen: B — Dual Support (recommended)**

### Options considered

| Option | Description | Risk |
|---|---|---|
| A — Big Bang | Deploy backend + front simultaneously, one coordinated release | High: any front delay leaves users broken |
| B — Dual Support | Backend accepts old + new values for a window; front migrates independently | Low: independent deployability |

### Rationale for B

The enum values affected (`NOTARIO`, `PERSONA_FISICA`, `ACTIVIDAD_VULNERABLE`, etc.) are embedded in:
- JWT tokens issued and cached in browsers.
- Front-end form submissions and API request bodies.
- Any third-party integration consuming the API.

A Big Bang requires perfectly coordinated deployment of at least two separate repositories (pld-api and pld-web) with zero time gap. In practice, deployments are never perfectly atomic. A window of even a few minutes with old front + new back would break all authenticated users.

Dual Support allows:
1. Backend deployed first with both old and new values accepted (input).
2. Responses immediately return new English values (output-only enforcement).
3. Front migrates at its own pace within the deprecation window.
4. Cleanup PR removes the compat layer once front confirms full migration.

### Deprecation window

4 weeks from Phase 1 deployment. The signal to remove the compat layer is confirmation from the front-end team that `pld-web` no longer sends legacy enum values in any environment (staging + production).

### Compat layer location

The compat layer lives in the **adapter** layer (`auth.adapter.ts`, `users.adapter.ts`, and any adapter that processes enum fields from HTTP requests). It MUST NOT reach the domain/entity layer. Implementation pattern:

```typescript
// In adapter — input normalization
function normalizeRole(raw: string): UserRole {
  const legacyMap: Record<string, UserRole> = {
    'NOTARIO': UserRole.NOTARY,
    'INMOBILIARIA': UserRole.REAL_ESTATE,
    'AUXILIAR': UserRole.AUXILIARY,
    'USUARIO_INTERNO': UserRole.INTERNAL_USER,
    'USUARIO_EXTERNO': UserRole.EXTERNAL_USER,
  };
  const normalized = legacyMap[raw] ?? (raw as UserRole);
  if (legacyMap[raw]) {
    logger.warn(`[DEPRECATED] Received legacy role value "${raw}". Use "${normalized}" instead.`);
  }
  return normalized;
}
```

---

## Decision 2 — One Change vs Multiple Sub-Changes

**Chosen: Divide into 3 sequential sub-changes under this umbrella**

### Threshold for splitting

The project has approximately 80+ individual renames across 8 phases, spanning 5+ libs/packages, 20+ entity files, and multiple migration files. This clearly exceeds a manageable single PR.

### Proposed sub-changes

| Sub-change | Scope | Blocking dependency |
|---|---|---|
| `rename-enums-to-english` | Phase 1 only — all enum names and values | None (runs first) |
| `rename-participants-entities-to-english` | Phases 2–5 — all `libs/participants` entities and tables | `rename-enums-to-english` complete |
| `rename-users-and-types-to-english` | Phases 6–8 — users table, auth-profiles types, TS interfaces | `rename-enums-to-english` complete (can run parallel with sub-change 2) |

### Benefits

- Each sub-change is independently reviewable (< 400 lines of diff each).
- If `rename-participants-entities-to-english` is blocked (e.g., DB downtime required), `rename-users-and-types-to-english` can proceed independently.
- Conflicts with `add-registro-sujetos-obligados` are isolated to the sub-change that touches overlapping files.

This `rename-to-english` change functions as the umbrella proposal and is archived after all three sub-changes are complete.

---

## Decision 3 — Migration SQL Strategy

**Chosen: (1) table renames → (2) data UPDATE → (3) column renames**

### Rationale

1. **Table renames first**: `RENAME TABLE` is atomic and does not lock for long. TypeORM can immediately point to the new name. Old queries that hardcode the old table name will fail fast and visibly.

2. **Data UPDATE second**: enum value data must be migrated while the column still has its old name (to avoid referencing a renamed column in the WHERE clause). Example:
   ```sql
   UPDATE users SET role = 'NOTARY' WHERE role = 'NOTARIO';
   UPDATE users SET role = 'REAL_ESTATE' WHERE role = 'INMOBILIARIA';
   ```
   This runs BEFORE the column is renamed from `nombre` to `first_name`, so the UPDATE itself is not affected.

3. **Column renames last**: `ALTER TABLE ... CHANGE COLUMN` is not fully atomic in MySQL for large tables; it may cause brief locking. Doing it after the data migration ensures no stale data remains in old format.

### Per-phase migration files

Each sub-change generates its own migration file(s):
- `Migration_YYYYMMDD_EnumDataUpdate` — data migrations for enum values in `users`, `pf_basic`, `pm_basic`, etc.
- `Migration_YYYYMMDD_PFTableRenames` — table renames for persona física.
- `Migration_YYYYMMDD_PFColumnRenames` — column renames for persona física.
- (etc. for each entity group)

Down migrations reverse in opposite order: (1) column renames back, (2) data UPDATE back, (3) table renames back.

---

## Decision 4 — Renamed Files (git history strategy)

**Chosen: `git mv` for file renames where the file name changes meaningfully; otherwise rename in-place**

### Files to rename

Most entity files are named `participants.entity.ts` — this name does not need to change (it is already generic). The class names inside change but the file names can stay.

Files that SHOULD be renamed:
- `libs/auth-profiles/src/lib/auth-profiles.types.ts` → keep name, only rename identifiers inside.
- No file renames are strictly required for the entity files since they already use generic names.

### git mv policy

- Use `git mv` only when the file name itself contains a Spanish word (none identified in the current audit).
- Prefer renaming identifiers inside files to avoid confusing git history in cases where the file name is already acceptable.
- If a file is renamed, the PR description MUST note it explicitly so reviewers can find it with `git log --follow`.

### Folder structure

No folder renames are proposed. The current folder naming (`fisica/`, `moral/`, `fideicomiso/`) uses Spanish but is a structural concern better addressed in a separate `reorganize-structure` change.

---

## Decision 5 — Deprecation Window Management

**Chosen: 4-week window with explicit sign-off from front-end team**

### Timeline

| Week | Action |
|---|---|
| 0 | `rename-enums-to-english` deployed to production. Dual support active. |
| 1–3 | `pld-web` migrates to new English enum values in staging, then production. |
| 4 | Front-end team confirms 100% migration in all environments. |
| 4+ | Cleanup PR: remove `normalizeRole` and equivalent compat functions from adapters. |

### Signal to remove dual support

1. Front-end team opens a GitHub issue or comments in the cleanup ticket confirming migration complete.
2. Log monitoring shows zero `[DEPRECATED]` warn logs in production for ≥ 48 hours.
3. Both conditions must be met before cleanup PR merges.

---

## Impact on Front End (JSON Contract Changes)

### Fields that change in HTTP responses

| Endpoint | Field | Before | After |
|---|---|---|---|
| `POST /auth/login` | `roles[]` | `["NOTARIO"]` | `["NOTARY"]` |
| `GET /users/:id` | `role` | `"AUXILIAR"` | `"AUXILIARY"` |
| `GET /participants/fisica/:id` | _(any profile type field)_ | `"PERSONA_FISICA"` | `"INDIVIDUAL"` |
| `GET /participants/moral/:id` | _(any profile type field)_ | `"PERSONA_MORAL"` | `"LEGAL_ENTITY"` |
| `GET /operaciones/:id` | `actividadVulnerable` | `"TRANSMISION_DERECHOS_REALES_INMUEBLES"` | `"REAL_ESTATE_RIGHTS_TRANSFER"` |
| Any endpoint returning `tipoPersona` | `tipoPersona` | `"FIDEICOMISO"` | `"TRUST"` |

During the dual support window, **responses return new values immediately** — only requests are normalized.

---

## Impact on Swagger / OpenAPI

The OpenAPI schema generated by NestJS Swagger will update automatically once the enum values change in TypeScript. Specifically:

- `@ApiProperty({ enum: UserRole })` decorators will reflect new values.
- Any `@ApiProperty({ example: 'NOTARIO' })` must be updated manually to `'NOTARY'`.
- The Swagger UI will show new enum values, which may confuse front-end developers if not communicated.

**Action required**: update all `@ApiProperty` example values in controllers/DTOs as part of Phase 1.

---

## Sequence Diagram — Dual Support Flow (Phase 1)

```
Front (old values)       Adapter (compat)         Domain/DB (new values)
       |                       |                         |
       | POST /auth/login      |                         |
       | role: "NOTARIO"  ---> |                         |
       |                       | normalizeRole()         |
       |                       | NOTARIO -> NOTARY       |
       |                       | log WARN                |
       |                       | ----------------------> |
       |                       |                 UserRole.NOTARY
       |                       | <---------------------- |
       |                       | { role: "NOTARY" }      |
       | <--------------------- |                         |
       | role: "NOTARY"        |                         |
       |                       |                         |
```

Front receives `"NOTARY"` in the response even while sending `"NOTARIO"`. This is the expected behavior during the deprecation window.

---

## Risk Summary

| Risk | Mitigation |
|---|---|
| Front breaks because it receives unexpected `"NOTARY"` values | Coordinate with front team BEFORE Phase 1 deployment; they must handle new values gracefully. |
| MySQL column rename locks large tables | Schedule during maintenance window; use `pt-online-schema-change` if table is > 1M rows. |
| TypeORM sync mismatch after migration | Disable `synchronize: true` in production (already the case — confirm). Run `nx run auth-users:migration:run` explicitly. |
| Enum data migration misses rows | Verify with `SELECT COUNT(*) FROM users WHERE role NOT IN ('SUPERADMIN','NOTARY','REAL_ESTATE','AUXILIARY','INTERNAL_USER','EXTERNAL_USER')` post-migration. |
| Conflict with `add-registro-sujetos-obligados` branch | The new change creates new tables only; no overlap with renamed legacy tables. Merge `add-registro-sujetos-obligados` first. |
