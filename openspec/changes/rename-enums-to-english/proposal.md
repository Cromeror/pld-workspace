# Proposal: Rename shared enums to English (names + values)

## Intent

Align FE+BE on English enum identifiers and string values so the API↔Web contract speaks one language. Today `pld-api/packages/shared-types/src/enums.ts` mixes Spanish names (`TipoPersonaParticipante`, `ActividadVulnerable`) and Spanish values (`'NOTARIO'`, `'PERSONA_FISICA'`) that travel in JWT claims, request/response bodies, and DB ENUM columns; `pld-web` mirrors them by hand. Renaming closes Phase 1 of the archived `rename-to-english` umbrella in a single atomic, lockstep change across both sub-repos. Display strings in the FE stay in Spanish — only code identifiers move.

## Scope

### In Scope
- BE: rename the 8 enums in `packages/shared-types/src/enums.ts` (names + values per locked decisions): `UserRole` (values only), `TipoPersonaParticipante`→`ProfileType`, `ActividadVulnerable`→`VulnerableActivity`, `TipoMoneda`→`CurrencyType`, `FormaPago`→`PaymentMethod`, `TipoReporte`→`ReportType`, `ModalidadAtencion`→`AttendanceMode`, `Nacionalidad`→`Nationality`.
- BE: cascade renames in all consumers (`apps/auth-users`, `libs/auth-profiles`, `libs/operacion`, `libs/reports`, plus exhaustive grep sweep) including DTO Swagger examples, `IsIn` validators, string-literal unions, and tests.
- DB: 3-step migration on `users.role` ENUM (expand → backfill `UPDATE` → restrict) plus equivalent migration for any registration columns storing Spanish values (`profile_type`, `vulnerable_activity`); down migration reverses each step.
- FE: rename mirror types (`src/types/UserRole.ts`, `src/types/registration.ts`, `src/types/CurrentUser.ts` literal union, `src/types/PersonType.ts`) and cascade through wizard (`reporting-entity/schemas.ts`, `ReportingEntityTypeStep`, `ReviewStep`, `ReportingEntityRegistrationPage.tsx`, `postLoginRedirect.ts`).
- Single atomic commit per sub-repo; coordinated merge.

### Out of Scope
- Renaming `UserEntity` properties (`nombre`, `apellidoPaterno`, …) — `rename-users-table-to-english`.
- Renaming participant entity classes/tables in `libs/participants` — `rename-participants-entities-to-english`.
- Renaming legacy types (`RegistroPerfil`, `PFParticipante`) — `rename-legacy-types-to-english`.
- Local `TipoPersonaMoral` enum inside `participants.entity.ts` (deferred with participants).
- FE display strings (JSX labels, placeholders, toasts, zod messages) — stay Spanish.

## Approach

Lockstep, no dual-support window. Four phases executed in order in a single change, one commit per sub-repo:

1. **BE source rename + call-site cascade** — edit `enums.ts`, then `tsc --noEmit` finds every consumer; fix until clean. Update `IsIn` validators and Swagger examples to new strings.
2. **DB migration up/down** — wrap `users.role` (and registration tables' VARCHAR/ENUM columns) in: `ALTER … MODIFY … ENUM(old_values + new_values)` → `UPDATE … SET col = 'NEW' WHERE col = 'OLD'` (verify row counts) → `ALTER … MODIFY … ENUM(new_values)`. Down reverses.
3. **FE mirror types + cascade** — rename mirror files/types, fix wizard literals (`"NOTARIO"` → `"NOTARY"`, etc.), let `tsc -b` flag misses, run `yarn build`.
4. **Build/smoke gate** — `nx run auth-users:build` + `nx run cross:build` + FE `yarn build` all 0 errors; smoke `POST /auth/login` returns 201 with JWT carrying `role: "NOTARY"|"REAL_ESTATE"|"AUXILIARY"|"SUPERADMIN"`.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-api/packages/shared-types/src/enums.ts` | Modified | Source of truth: 7 enums renamed, values updated. |
| `pld-api/apps/auth-users/src/admin/registration/registration.service.ts` | Modified | Replace `'NOTARIO'` literal at L108 + spec fixture. |
| `pld-api/apps/auth-users/src/admin/registration/dto/create-registration.dto.ts` | Modified | `IsIn` + Swagger examples L19,20,22. |
| `pld-api/apps/auth-users/src/users/dto/create-user.dto.ts` | Modified | Swagger examples L30,72. |
| `pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts` | Modified | Literal union L22-23. |
| `pld-api/libs/operacion/src/lib/operacion.types.ts` | Modified | `ActividadVulnerable`/`TipoMoneda`/`FormaPago` consumers. |
| `pld-api/libs/reports/src/lib/reports.types.ts` | Modified | `TipoReporte`/`TipoPersonaParticipante`/`ActividadVulnerable` consumers. |
| `pld-api/packages/persistence/migrations/` | New | Up + down migration for ENUM columns. |
| `pld-web/src/types/UserRole.ts` | Modified | Mirror values. |
| `pld-web/src/types/registration.ts` | Modified | Rename to `ProfileType`. |
| `pld-web/src/types/CurrentUser.ts` | Modified | Profile-type literal union L12. |
| `pld-web/src/types/PersonType.ts` | Modified | English variants (`INDIVIDUAL_MEXICAN`, …). |
| `pld-web/src/components/organisms/reporting-entity/schemas.ts` | Modified | Zod enums L9,14-15. |
| `pld-web/src/components/organisms/reporting-entity/ReportingEntityTypeStep/index.tsx` | Modified | Literal references. |
| `pld-web/src/components/organisms/reporting-entity/ReviewStep/index.tsx` | Modified | L58,61 literal compares. |
| `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx` | Modified | L30,125,137,169,264,359-364 literals. |
| `pld-web/src/config/postLoginRedirect.ts` | Modified | `UserRole.SUPERADMIN` reference. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Enum sweep miss → silent runtime mismatch (string compared to renamed value) | High | `tsc --noEmit` on both repos + grep for old strings (`NOTARIO`, `PERSONA_FISICA`, etc.) returns 0 outside migrations; smoke `POST /auth/login` after each phase. |
| DB migration partial failure leaves rows in old values | Med | Wrap up-migration in transaction; assert row counts before and after `UPDATE`; abort if mismatch. |
| JWT issued before migration consumed after (or vice versa) during deploy | Med | Deploy BE first (it accepts both old+new during expand step); run migration; deploy FE last. Document deploy order in tasks. |
| FE display strings accidentally renamed | Low | Reviewer checks JSX/zod-message diffs; only `.ts`/`.tsx` identifiers and zod `.enum([…])` arrays change values. |
| Other consumers found after the audit | Med | Sub-agent runs exhaustive `grep -r` for every old name + value across `apps`, `libs`, `packages`, `src`. |

## Rollback Plan

1. `git revert` the FE rename commit in `pld-web` → mirrors back to Spanish; FE again sends old values.
2. `git revert` the BE rename commit in `pld-api` → enums back to Spanish.
3. Run the down migration: restrict ENUM to new+old → `UPDATE col = 'OLD' WHERE col = 'NEW'` → restrict ENUM to old values only.
4. Verify smoke: `POST /auth/login` returns 201 with old role values (`NOTARIO`, etc.).

## Dependencies

- TypeORM migration runner already wired in `pld-api/packages/persistence`.
- No external service depends on the enum string values (verified: only FE consumes the JWT `role` claim and request/response bodies).

## Success Criteria

- [ ] `tsc --noEmit` returns 0 errors in `pld-api` and `pld-web`.
- [ ] `nx run auth-users:build` and `nx run cross:build` exit 0.
- [ ] `yarn build` (FE: `tsc -b && vite build`) exits 0.
- [ ] `grep -r "'NOTARIO'\|'INMOBILIARIA'\|'AUXILIAR'\|'PERSONA_FISICA'\|'PERSONA_MORAL'\|'FIDEICOMISO'\|TipoPersonaParticipante\|ActividadVulnerable\|FormaPago\|TipoReporte\|ModalidadAtencion\|Nacionalidad\|TipoMoneda" pld-api/{apps,libs,packages}/*/src pld-web/src` returns 0 matches (excluding `migrations/`).
- [ ] DB ENUM values for `users.role` are exactly `('SUPERADMIN','NOTARY','REAL_ESTATE','AUXILIARY')`; no row with old values remains.
- [ ] Smoke: `POST /auth/login` for a notary user returns HTTP 201 with JWT whose `role` claim is `"NOTARY"`.
