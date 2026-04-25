# Design: Rename legacy types + HTTP routes to English

TS-only atomic rename closing the `rename-to-english` umbrella. No DB migrations. Single pld-api commit covers ~30 type renames, ~50 property renames, 6 HTTP route segment renames, and 2 DTO field renames. Consumer sweep driven by `tsc --noEmit` after each phase.

## Decisions

### D1 — Lockstep single commit

| Option | Tradeoff | Decision |
|--------|----------|----------|
| Single atomic commit (all phases) | Requires all phases coordinated; codebase unbuildable mid-sweep | ✅ Chosen |
| Split by file group | Leaves broken intermediate states since type consumers span multiple apps/libs | Discarded |

0 FE consumers verified. `tsc --noEmit` gates between phases surface missed references before commit.

### D2 — No DB migration needed

| Option | Tradeoff | Decision |
|--------|----------|----------|
| TS-only change, no migration | Clean; migration archive already moved `nombreCompletoPEP` → `pep_full_name` in `pm_pep`, `trust_pep`, `cross_beneficial_owner` | ✅ Confirmed |
| Add migration | Unnecessary — `domicilio` in DTO is a DTO-only field, not a DB column; `nombreCompletoPEP` DB column already renamed by prior migrations (verified in `20260425030000` and `20260425030001`) | Discarded |

Pre-flight: `grep 'nombreCompletoPEP\|domicilio' pld-api/packages/persistence/migrations/*.sql` confirms DB column is already `pep_full_name`; DTO field is independent.

### D3 — HTTP route renames in scope

| Option | Tradeoff | Decision |
|--------|----------|----------|
| Rename Spanish route segments | Closes umbrella; no external consumers (FE grep returns 0) | ✅ Chosen |
| Defer routes | Leaves Spanish HTTP surface; umbrella stays open | Discarded |

Outer path segments `persona-fisica`, `persona-moral`, `anexo-7` remain Spanish (folder names, D4). Only the sub-route segments change.

### D4 — Folder renames out of scope

| Option | Tradeoff | Decision |
|--------|----------|----------|
| Keep Spanish folder names | Limits scope; no import churn across the whole app | ✅ Chosen |
| Rename folders too | Would require updating every import in adapters, controllers, services — scope balloons beyond this change | Discarded |

Known follow-up: `apps/auth-users/src/participants/{persona-fisica,persona-moral,fideicomiso,anexo-7}/`.

### D5 — Re-exports in `@pld-api/shared-types`

`packages/shared-types/src/index.ts` uses wildcard `export * from './participants'` — no per-name re-exports to update. The new names flow automatically. However, any consumer importing a specific old name by string (not possible in TS module syntax) is caught by tsc.

### D6 — Sweep order (within single commit)

1. `packages/shared-types/src/participants.ts` — ~30 type renames + ~50 property renames (largest file, foundation for all consumers).
2. `packages/shared-types/src/index.ts` — verify wildcard re-export covers new names (no edit needed unless explicit re-exports exist; confirmed wildcard only).
3. `libs/auth-profiles/src/lib/auth-profiles.types.ts` — 7 type renames + ~10 property renames; update import of `PerfilBasico`, `PFParticipante`, `PMParticipante` from `@pld-api/shared-types`.
4. `libs/auth-profiles/src/index.ts` — wildcard re-export; no edit needed.
5. `libs/reports/src/lib/reports.types.ts` — single property rename `actividadVulnerable` → `vulnerableActivity`.
6. DTOs — `fideicomiso.dto.ts` and `ben-controlador.dto.ts`: `nombreCompletoPEP` → `pepFullName`.
7. Controllers — rename route segments in 4 controller files.
8. Consumer sweep — `libs/participants/src/lib/moral/participants.entity.ts` imports `PMBeneficiarioControlTipoControl` from `@pld-api/shared-types`; update to new name. Sweep all adapters, services, and the comment in `packages/domain-auth-users/src/entities/user.entity.ts`.
9. `tsc --noEmit` + `nx run auth-users:build` + `nx run cross:build` + exhaustive grep.

### D7 — `RegistroPerfil` discriminator field

| Option | Tradeoff | Decision |
|--------|----------|----------|
| Rename `rol` → `role`, `datos` → `data` atomically | tsc surfaces all missed union branches in one pass | ✅ Chosen |
| Keep discriminator as-is | Leaves Spanish identifier in union type; umbrella stays open | Discarded |

`RegistroPerfil` union is defined in `auth-profiles.types.ts`. Confirmed no external consumers outside `pld-api/libs/auth-profiles` (grep found zero imports of `@pld-api/auth-profiles` from apps; `AuthProfilesService` body is empty).

### D8 — `PFParticipantePoderdante` rename

| Option | Tradeoff | Decision |
|--------|----------|----------|
| `PFPrincipal` | Short, domain-accurate (principal = party granting power) | ✅ Chosen |
| `PFPowerGrantor` | Verbose; not idiomatic | Discarded |
| `PFPowerOfAttorneyGrantor` | Overly long | Discarded |

### D9 — Comment in `user.entity.ts`

Line 18 of `packages/domain-auth-users/src/entities/user.entity.ts`:
`// === Datos personales (alineados con PerfilBasico) ===` → `// === Personal data (aligned with BasicProfile) ===`

In scope because it directly references the renamed type `PerfilBasico` → `BasicProfile`.

## API Contract Changes

### HTTP routes

| Before | After |
|--------|-------|
| `POST /participants/persona-fisica/domicilio` | `POST /participants/persona-fisica/address` |
| `POST /participants/persona-fisica/domicilio-nacional` | `POST /participants/persona-fisica/national-address` |
| `POST /participants/persona-fisica/representante-domicilio` | `POST /participants/persona-fisica/representative-address` |
| `POST /participants/persona-moral/domicilio` | `POST /participants/persona-moral/address` |
| `POST /participants/persona-moral/domicilio-territorial-nacional` | `POST /participants/persona-moral/national-territorial-address` |
| `POST /participants/anexo-7/domicilio` | `POST /participants/anexo-7/address` |

Outer segments (`persona-fisica`, `persona-moral`, `anexo-7`) unchanged — folder names, D4.

### Request body fields (DTOs)

| File | Before | After |
|------|--------|-------|
| `apps/auth-users/src/participants/fideicomiso/dto/fideicomiso.dto.ts` | `nombreCompletoPEP` | `pepFullName` |
| `apps/cross/src/beneficiario-controlador/dto/ben-controlador.dto.ts` | `nombreCompletoPEP` | `pepFullName` |

## File Changes

| File | Action | Notes |
|------|--------|-------|
| `packages/shared-types/src/participants.ts` | Modify | ~30 type renames + ~50 property renames; `PFParticipantePoderdante` → `PFPrincipal`, `PMBeneficiarioControlTipoControl` → `BeneficialOwnerControlType`, all `Con*` mixins → `With*` pattern |
| `packages/shared-types/src/index.ts` | Verify | Wildcard `export * from './participants'` — no edit needed; new names flow automatically |
| `libs/auth-profiles/src/lib/auth-profiles.types.ts` | Modify | 7 type renames + ~10 prop renames; update import of `PerfilBasico`→`BasicProfile`, `PFParticipante`→`PFParticipant`, `PMParticipante`→`PMParticipant`; discriminator `rol`→`role`, `datos`→`data` |
| `libs/auth-profiles/src/index.ts` | Verify | Wildcard re-export — no edit needed |
| `libs/reports/src/lib/reports.types.ts` | Modify | `actividadVulnerable` → `vulnerableActivity` in `FiltroReporte` interface |
| `apps/auth-users/src/participants/fideicomiso/dto/fideicomiso.dto.ts` | Modify | `nombreCompletoPEP` → `pepFullName` (line 145) |
| `apps/cross/src/beneficiario-controlador/dto/ben-controlador.dto.ts` | Modify | `nombreCompletoPEP` → `pepFullName` (line 189) |
| `apps/auth-users/src/participants/persona-fisica/participants.controller.ts` | Modify | `@Post('domicilio')` → `address`, `@Post('domicilio-nacional')` → `national-address`, `@Post('representante-domicilio')` → `representative-address` |
| `apps/auth-users/src/participants/persona-moral/participants.controller.ts` | Modify | `@Post('domicilio')` → `address`, `@Post('domicilio-territorial-nacional')` → `national-territorial-address` |
| `apps/auth-users/src/participants/anexo-7/anexo-7.controller.ts` | Modify | `@Post('domicilio')` → `address` |
| `libs/participants/src/lib/moral/participants.entity.ts` | Modify | Update import of `PMBeneficiarioControlTipoControl` → `BeneficialOwnerControlType` from `@pld-api/shared-types` |
| `packages/domain-auth-users/src/entities/user.entity.ts` | Modify | Line 18 comment: `PerfilBasico` → `BasicProfile` |

## Testing Strategy

| Layer | What | Approach |
|-------|------|----------|
| Type-check (per phase) | All TS renames consistent | `pnpm tsc --noEmit` after each of the 8 sweep steps — 0 errors before proceeding |
| Build | App targets compile | `nx run auth-users:build`, `nx run cross:build` → exit 0 |
| Grep | 0 old identifiers remain | Exhaustive grep in `pld-api/{apps,libs,packages}/*/src` excluding migration files and Swagger `description:` strings |
| Smoke | API functional | `POST /auth/login` → 201 + JWT |

## Migration / Rollout

No migration required. DB columns were renamed in prior sub-changes (`20260425030000`, `20260425030001`). DTO field renames are HTTP API surface only.

Rollback: `git revert <commit>` on pld-api — reverts all type, property, DTO, and HTTP route changes. No DB rollback needed.

Known prerequisite: `dist/apps/auth-users` may be root-owned. Run `sudo rm -rf pld-api/dist/apps/auth-users` before `nx run auth-users:build` if build fails with permission errors (recurrent issue from prior phases).

## Open Questions

- None — design is fully specified. All ambiguities resolved via codebase inspection.
