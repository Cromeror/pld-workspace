# Archive Report: add-registro-sujetos-obligados

**Closed**: 2026-04-22
**Status**: IMPLEMENTED & SMOKE-TESTED

## Summary

SUPERADMIN-driven wizard to onboard reporting entities (Notarías, Inmobiliarias) as Persona Física or Persona Moral. Multi-step registration with resume support, RFC dedupe, TTL-based expiration, and atomic user creation on finalize.

## Endpoints delivered (all under `/admin/registration/*`)

- `POST /` — start draft with RFC dedupe (409 returns `registrationId` of existing draft).
- `GET /` — list, filterable by `?status=` and `?includeExpired=`.
- `GET /:id` — detail; `410 Gone` if expired.
- `POST /:id/identification` — PF or PM payload per `profileType`.
- `POST /:id/contact` — append-only, N per registration.
- `POST /:id/vulnerable-activity` — activity + address (single).
- `POST /:id/compliance-responsible` — PM only; 409 on PF.
- `POST /:id/finalize` — creates `users` row + returns `tempPassword` once in cleartext.

## Implemented files

### New
- `apps/auth-users/src/shared/auth/roles.decorator.ts`
- `apps/auth-users/src/shared/auth/roles.guard.ts`
- `apps/auth-users/src/admin/registration/registration.module.ts`
- `apps/auth-users/src/admin/registration/registration.controller.ts`
- `apps/auth-users/src/admin/registration/registration.service.ts`
- `apps/auth-users/src/admin/registration/registration.adapter.ts`
- `apps/auth-users/src/admin/registration/registration.enums.ts`
- `apps/auth-users/src/admin/registration/entities/*.entity.ts` (7 entities)
- `apps/auth-users/src/admin/registration/dto/*.dto.ts` (9 DTOs)
- `packages/persistence/migrations/20260421000000-create-registration-schema.sql` (+ `.down.sql`)

### Modified
- `packages/domain-auth-users/src/entities/user.entity.ts` — added `profileType`, `profileId`.
- `apps/auth-users/src/admin/admin.module.ts` — wired `RegistrationModule`.
- `apps/auth-users/src/shared/http-error.interceptor.ts` — propagates structured HttpException fields (e.g. `registrationId`).
- `query.sql` — appended new tables DDL for fresh container setups.
- `docker-compose.dev.yml` — `REGISTRATION_TTL_DAYS` env var.

## Adjustments during implementation (deltas from plan)

1. **State machine guards reinforced**: the `assertStepOrder` helper allowed same-step transitions incorrectly. Replaced with explicit FK presence checks in `appendContact` (requires profile FK) and `saveComplianceResponsible` (requires `vulnerableActivityId`).
2. **`vulnerable_activity.activity` column widened**: VARCHAR(30) → VARCHAR(80). The longest `ActividadVulnerable` value (`TRANSMISION_DERECHOS_REALES_INMUEBLES`) is 37 chars.
3. **`users.nombre` during finalize**: initially populated with email prefix placeholder. Corrected to read from the real profile (`firstName` for PF, `corporateName` for PM).
4. **409 dedupe response**: service already built `{ message, registrationId }`, but the global `HttpErrorInterceptor` stripped extra fields. Interceptor updated to propagate structured properties from HttpException payloads.
5. **Admin user seed reset**: dev DB had `admin@pld.com` with unknown password hash. Reset locally to `Admin123!` to satisfy the login invariant. Not a production change.

## Deferred (future changes)

- `DELETE /admin/registration/:id` cancellation (scaffolding present: `status = CANCELLED`, `deleted_at` column; no endpoint).
- Post-completion edits (`PATCH` on completed drafts).
- Detailed per-step audit (who touched what).
- `?rfc=` filter on list endpoint.
- Automated tests (only manual smoke so far — Jest scaffolding exists with `passWithNoTests: true`).

## Smoke test results (2026-04-22)

| Test | Result |
|---|---|
| `POST /auth/login admin@pld.com/Admin123!` | ✅ HTTP 201 + JWT |
| PF happy path (5 endpoints) | ✅ all 201 |
| PM happy path (6 endpoints + finalize) | ✅ all 201 |
| Login new user with returned `tempPassword` | ✅ 201 |
| Dedupe by RFC | ✅ 409 + `registrationId` |
| Skip step (contact before identification) | ✅ 409 |
| Compliance-responsible on PF | ✅ 409 |
| Finalize PM without compliance | ✅ 409 |
| Write-once post-COMPLETED | ✅ 409 |
| AUXILIAR role | ✅ 403 |
| NOTARIO + PERSONA_MORAL | ✅ 400 |
| Unauthenticated | ✅ 401 |
| Nonexistent UUID | ✅ 404 |
| Invalid RFC | ✅ 400 |
| Expired GET / POST | ✅ 410 Gone |
| List filter (default hides expired, `?includeExpired=true` shows) | ✅ |
| Finalize with duplicate email | ✅ 409, draft stays `IN_PROGRESS` |
| Resume flow (list shows `IN_PROGRESS` with correct `currentStep`) | ✅ |
| Regression: `GET /catalogs/beneficiario` | ✅ 200 |
| Regression: `nx run cross:build` | ✅ |
| Regression: legacy `/system-users/*` + `/participants-*` routes present in Swagger | ✅ |
| Final login invariant re-check | ✅ 201 |
| `tsc --noEmit` | ✅ clean |

## Open questions closed during implementation

- **Leyendas "FE PÚBLICA" / "TRANSMISIÓN DE BIENES INMUEBLES"** → frontend copy, not domain values.
- **Post-cierre edits** → explicitly deferred.
- **RFC dedupe key** → captured in `POST /` body, denormalized on `registration.rfc`.
- **Compliance Responsible location** → `moral_person_profile.compliance_responsible_id` (not on registration).

## Risks noted for future changes

- `users.role` remains MySQL native `ENUM` (legacy). The new `profile_type` column uses `varchar(30)` per the design. A future `rename-to-english` change is expected to normalize.
- `UserEntity.nombre` was retrofitted from the profile at finalize — when a future change splits participants schema or renames columns, revisit to ensure consistency.
- Password cleartext in `POST /finalize` 201 response body is at risk of leaking via access logs. Recommend audit before production deploy.
