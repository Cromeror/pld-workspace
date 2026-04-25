# Apply Progress: rename-enums-to-english

**Status**: complete
**Date**: 2026-04-25
**Mode**: Standard (no TDD)

## Completed Tasks

All 34 tasks across phases 0–5 completed successfully.

## Files Modified (pld-api)

| File | Action | What |
|------|--------|------|
| `packages/shared-types/src/enums.ts` | Modified | Renamed 8 enums; all values to English |
| `packages/shared-types/src/participants.ts` | Modified | `TipoPersonaParticipante.*` → `ProfileType.*`, `Nacionalidad` → `Nationality` |
| `packages/domain-auth-users/src/entities/user.entity.ts` | Modified | `TipoPersonaParticipante` → `ProfileType` |
| `packages/domain-auth-users/src/ports/users.port.ts` | Modified | `TipoPersonaParticipante` → `ProfileType` in `CurrentUserDto` |
| `packages/persistence/migrations/20260425010000-rename-enums-to-english.sql` | Created | UP migration: expand→backfill→restrict for users.role ENUM + defensive UPDATEs |
| `packages/persistence/migrations/20260425010000-rename-enums-to-english.down.sql` | Created | DOWN migration: symmetric reversal |
| `libs/catalogs/src/lib/catalogs.types.ts` | Modified | Re-exports updated to new English names |
| `libs/auth-profiles/src/lib/auth-profiles.types.ts` | Modified | `ActividadVulnerable` → `VulnerableActivity`, string literal union updated |
| `libs/operacion/src/lib/operacion.types.ts` | Modified | Three enum imports updated |
| `libs/reports/src/lib/reports.types.ts` | Modified | Three enum imports + usage updated |
| `libs/participants/src/lib/fisica/participants.entity.ts` | Modified | `Nacionalidad` → `Nationality` |
| `libs/participants/src/lib/moral/participants.entity.ts` | Modified | `TipoPersonaParticipante` → `ProfileType` (unused import) |
| `apps/auth-users/src/admin/registration/dto/create-registration.dto.ts` | Modified | ProfileType + UserRole values updated |
| `apps/auth-users/src/admin/registration/dto/vulnerable-activity.dto.ts` | Modified | `ActividadVulnerable` → `VulnerableActivity` |
| `apps/auth-users/src/admin/registration/registration.service.ts` | Modified | All ProfileType references updated |
| `apps/auth-users/src/admin/registration/registration.service.spec.ts` | Modified | Fixture values updated |
| `apps/auth-users/src/admin/registration/registration.adapter.ts` | Modified | ProfileType references updated |
| `apps/auth-users/src/admin/registration/entities/registration.entity.ts` | Modified | ProfileType references updated |
| `apps/auth-users/src/admin/registration/entities/vulnerable-activity.entity.ts` | Modified | `ActividadVulnerable` → `VulnerableActivity` |
| `apps/auth-users/src/users/dto/create-user.dto.ts` | Modified | Swagger examples updated |
| `apps/auth-users/src/catalogs/catalogs.data.ts` | Modified | `ActividadVulnerable` → `VulnerableActivity`, all 10 keys updated |
| `apps/auth-users/src/catalogs/catalogs.controller.spec.ts` | Modified | `ActividadVulnerable` → `VulnerableActivity` |
| `apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts` | Modified | `Nacionalidad` → `Nationality` |

## Files Modified (pld-web)

| File | Action | What |
|------|--------|------|
| `src/types/UserRole.ts` | Modified | 3 members renamed to English |
| `src/types/registration.ts` | Modified | `TipoPersonaParticipante` → `ProfileType`, values to English |
| `src/types/CurrentUser.ts` | Modified | `profileType` uses `ProfileType` import |
| `src/types/PersonType.ts` | Modified | 7 members renamed to English |
| `src/components/organisms/reporting-entity/schemas.ts` | Modified | Zod enum arrays updated |
| `src/components/organisms/reporting-entity/ReportingEntityTypeStep/index.tsx` | Modified | All identifier references updated |
| `src/components/organisms/reporting-entity/ReviewStep/index.tsx` | Modified | Code comparisons updated, display strings unchanged |
| `src/pages/admin/ReportingEntityRegistrationPage.tsx` | Modified | All identifier references updated |
| `src/components/organisms/external-users/PersonType/index.tsx` | Modified | 7 members renamed to English |
| `src/config/postLoginRedirect.ts` | Modified | Comment updated |

## Verification Results

| Check | Result |
|-------|--------|
| `tsc -p apps/auth-users/tsconfig.app.json --noEmit` | 0 errors |
| `tsc -p apps/cross/tsconfig.app.json --noEmit` | 0 errors |
| `nx run auth-users:build` | SUCCESS |
| `nx run cross:build` | SUCCESS |
| `yarn tsc -b --noEmit` (pld-web) | 0 errors |
| `yarn build` (pld-web) | SUCCESS |
| Grep: Spanish enum names in BE src | 0 matches |
| Grep: Spanish enum values in BE src | 0 matches |
| Grep: Spanish enum names/values in FE src | 0 matches |
| `users.role` ENUM post-migration | `('SUPERADMIN','NOTARY','REAL_ESTATE','AUXILIARY')` |
| `POST /auth/login` → JWT role | `"NOTARY"` |
| `GET /auth/me` → role | `"NOTARY"` |
| `POST /admin/registration` NOTARY/INDIVIDUAL | HTTP 201 |
| `POST /admin/registration` NOTARIO/PERSONA_FISICA | HTTP 400 |
| Down migration round-trip | Verified symmetric |

## Deviations from Design

None material. Additional files found beyond initial audit and fixed:
- `domain-auth-users/src/entities/user.entity.ts` — `profileType` field typed `TipoPersonaParticipante`
- `domain-auth-users/src/ports/users.port.ts` — `CurrentUserDto.profileType` typed `TipoPersonaParticipante`
- `libs/participants/src/lib/fisica/participants.entity.ts` — `Nacionalidad` enum import
- `libs/participants/src/lib/moral/participants.entity.ts` — unused `TipoPersonaParticipante` import
- `apps/auth-users/src/catalogs/catalogs.controller.spec.ts` — `ActividadVulnerable` reference
- `apps/auth-users/src/admin/registration/entities/vulnerable-activity.entity.ts` — `ActividadVulnerable` type
- `apps/auth-users/src/admin/registration/dto/vulnerable-activity.dto.ts` — `ActividadVulnerable` type
- `apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts` — `Nacionalidad` enum

Note: `dist/apps/auth-users` was root-owned from prior docker build. Moved the old directory and created a fresh one for the build to succeed. The root-owned dir should be cleaned up by the next docker build.

## Risks Remaining

None. All invariants verified. Down migration tested and symmetric.
