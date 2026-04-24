# Proposal: Registration of reporting entities (SUPERADMIN wizard)

> Backend implementation of the SUPERADMIN-driven registration flow described in [temp-docs/FLUJO_REGISTRO_SUPERADMIN.md](../../../temp-docs/FLUJO_REGISTRO_SUPERADMIN.md) with the schema fixed in [temp-docs/REGISTRO_SCHEMA.md](../../../temp-docs/REGISTRO_SCHEMA.md).

## Why

A `SUPERADMIN` must be able to onboard `Notarías` and `Inmobiliarias` as reporting entities (sujetos obligados) through a stepped wizard — `identification → contact → vulnerable activity → (compliance responsible, PM only) → finalize`. Every step has its own "Guardar información" checkpoint and it MUST be possible to resume a partially-completed registration after hours or days.

Today there is no backend for this:
- `libs/participants/*` persists isolated tables (in Spanish) but there is no aggregate tying the wizard steps together.
- There is no tracking of `current_step` / `status`, no per-draft TTL, no RFC-based deduplication.
- There is no `RolesGuard` — anyone with a valid JWT reaches every endpoint today.
- The platform user (`users` row) tied to the reporting entity does not exist until the registration completes, so creating the user up-front would leak half-configured accounts.

This change creates the `admin/registration/*` module with a new, normalized, English-named schema that covers the full flow and sets the precedent for subsequent refactors of `libs/participants/*`.

## What Changes

### New tables (all in English, all UUID PKs)

- **`registration`** — draft of the wizard. Columns: `id`, `profile_type` (`PERSONA_FISICA | PERSONA_MORAL`), `user_role` (`NOTARIO | INMOBILIARIA`), `current_step` (`IDENTIFICATION | CONTACT | VULNERABLE_ACTIVITY | COMPLIANCE_RESPONSIBLE | COMPLETED`), `status` (`IN_PROGRESS | COMPLETED | CANCELLED`), `physical_profile_id`, `moral_profile_id`, `vulnerable_activity_id`, `user_id` (null until COMPLETED), `started_by_user_id` (SUPERADMIN who kicked off the draft), `expires_at`, `created_at`, `updated_at`, `deleted_at`.
- **`physical_person_profile`** — persistent profile for a PF reporting entity. Required: `first_name`, `paternal_surname`, `maternal_surname`, `birth_date`, `rfc`, `curp`. Optional: `nationality_country`, `birth_country`. FK nullable to `reporting_entity_address`.
- **`moral_person_profile`** — persistent profile for a PM reporting entity. Required: `corporate_name`, `rfc`, `compliance_responsible_id` (NOT NULL). Optional: `incorporation_date`, `nationality_country`. FK nullable to `reporting_entity_address`.
- **`contact`** — N per registration (`registration_id` FK). Columns: `country_code`, `phone`, `email`, `cellphone`.
- **`reporting_entity_address`** — reusable address record used by PF profile, PM profile and `vulnerable_activity`. Columns: `street`, `exterior_number`, `interior_number?`, `neighborhood`, `municipality`, `city`, `state`, `postal_code`, `country`, `road_type?`, `locality?`.
- **`compliance_responsible`** — PM-only natural person designated as compliance officer. Required: `first_name`, `paternal_surname`, `maternal_surname`, `rfc`, `curp`. Optional: `birth_date`, `nationality_country`, `designation_date`.
- **`vulnerable_activity`** — declared vulnerable activity tied to the registration. Columns: `activity` (enum `ActividadVulnerable`), `start_date`, `reporting_entity_address_id` (NOT NULL — activity address), `activity_performed_at_address` (text).

### Modified table

- **`users`** — add `profile_type` (nullable enum `PERSONA_FISICA | PERSONA_MORAL`) and `profile_id` (nullable UUID). Invariant: both NULL or both set. A `user` has 0 or 1 profile.

### New endpoints (all under `/admin/registration`, guarded by `JwtAuthGuard + RolesGuard + @Roles(UserRole.SUPERADMIN)`)

| Method | Route | Purpose |
|---|---|---|
| `POST` | `/admin/registration` | Start a draft — body `{ profileType, userRole, rfc }`. Enforces RFC dedupe (409 if another `IN_PROGRESS` draft has the same RFC). |
| `GET` | `/admin/registration` | List. Query: `?status=`, `?includeExpired=true`. Hides `expires_at < NOW()` by default. |
| `GET` | `/admin/registration/:id` | Detail. Returns `410 Gone` if the draft has expired. |
| `POST` | `/admin/registration/:id/identification` | Step 1 — body DTO picks PF vs PM by `profileType` of the draft. |
| `POST` | `/admin/registration/:id/contact` | Step 2 — can be called N times. |
| `POST` | `/admin/registration/:id/vulnerable-activity` | Step 3 — activity + address. |
| `POST` | `/admin/registration/:id/compliance-responsible` | PM only — 409 on PF drafts. |
| `POST` | `/admin/registration/:id/finalize` | Creates the `user`, hashes a generated password, returns it once in cleartext. |

### New cross-cutting primitives

- `RolesGuard` — `apps/auth-users/src/shared/auth/roles.guard.ts`. First implementation; implements `CanActivate`, reads `ROLES_KEY` via `Reflector`, matches against `request.user.role`.
- `@Roles(...UserRole[])` — `apps/auth-users/src/shared/auth/roles.decorator.ts`. `SetMetadata(ROLES_KEY, roles)`.

### Env var

- `REGISTRATION_TTL_DAYS` (default `30`) — controls `expires_at = NOW() + REGISTRATION_TTL_DAYS`.

## Apps / Libs / Packages affected

| Area | Impact | Description |
|---|---|---|
| `apps/auth-users/src/admin/admin.module.ts` | Modified | Register `RegistrationModule`. Currently empty. |
| `apps/auth-users/src/admin/registration/**` | New | Controller, service (state machine + dedupe + TTL), adapter, DTOs, entities (7 tables). |
| `apps/auth-users/src/shared/auth/roles.guard.ts` | New | Reusable `RolesGuard`. |
| `apps/auth-users/src/shared/auth/roles.decorator.ts` | New | Reusable `@Roles(...)`. |
| `apps/auth-users/src/auth-users.module.ts` | Modified | Import `AdminModule`. |
| `packages/domain-auth-users/src/entities/user.entity.ts` | Modified | Add columns `profile_type`, `profile_id`. |
| `packages/shared-types/src/enums.ts` | Sin cambios | Reuse `UserRole`, `TipoPersonaParticipante`, `ActividadVulnerable`. |
| `libs/participants/**` | Sin cambios | Legacy entities untouched. |
| `packages/persistence` | Sin cambios | Reuses `mysqlConfigFromEnv`. No new datasource. |
| Migrations SQL | New | One consolidated migration (`20260421000000-create-registration-schema.sql`) creating 7 tables + 2 columns in `users`. |

> **No `packages/domain-registration`.** The flow is app-layer: the state machine is heavily coupled to NestJS decorators, TypeORM repositories and the `JwtAuthGuard` of `apps/auth-users`. Pushing it to a pure package without first migrating `participants` adds indirection with no gain.

## Alternativas descartadas

1. **Monolithic `POST /admin/registration` accepting the full payload.** Rejected — the flow is explicitly resumable across sessions and the UX expects per-step feedback.
2. **Create the `user` up-front (step 0).** Rejected — leaves half-configured users in the system if the SUPERADMIN abandons the flow. Instead, create the `user` atomically in `POST /finalize` using the primary `contact.email`.
3. **Dedupe drafts by `user_id`.** Rejected — the `user` does not exist until finalize. Dedupe must live on a field known at step 0; chose `rfc` (captured at step 1, but the `POST /admin/registration` body accepts it to enable dedupe before any profile row is written).
4. **Enum-typed columns at the DB level.** Rejected for portability and to keep the enum evolution cheap. Columns are `varchar(30)` with enum validation at the DTO/entity layer, matching the existing convention (`UserEntity.role` is `varchar`).
5. **Nullable `moral_person_profile.compliance_responsible_id`.** Rejected — a PM reporting entity without a compliance officer is not a valid final state. Keeping it NOT NULL forces the flow to insert the compliance row before the PM profile row (state machine enforces this).
6. **Soft-delete + cancellation endpoint in this change.** Rejected for scope — the `deleted_at` column and `CANCELLED` status stay in the schema so nothing breaks later, but the `DELETE /admin/registration/:id` endpoint is deferred to a future change.
7. **Per-step audit table.** Rejected for scope — only `started_by_user_id` is tracked. A dedicated audit layer lands in a future change.

## Plan de rollback

The schema is additive. Nothing previous is dropped or renamed.

1. `git revert` of the feature commits.
2. Run the reverse migration: `DROP TABLE contact, vulnerable_activity, compliance_responsible, physical_person_profile, moral_person_profile, reporting_entity_address, registration; ALTER TABLE users DROP COLUMN profile_id, DROP COLUMN profile_type;` (order matters — FKs cascade up).
3. `AdminModule` back to `@Module({})`.
4. Smoke: `POST /auth/login` with `admin@pld.com` / `Admin123!` → **HTTP 201** + JWT.
5. Smoke: legacy endpoints (`POST /participants-fisica/*`, `POST /system-users/*`, catalog GETs) still return their prior statuses.

Because the deploy is app-layer only and adds new routes, **no backend downtime is required**. Sequence for prod: (1) run migration, (2) deploy new image, (3) verify smokes. Rollback is deploy-previous-image + reverse migration.

Impacto de rollback: **bajo**. New tables have no foreign keys from legacy tables. The two new columns on `users` are both nullable — existing rows remain valid.

## Validación post-cambio

1. **Login bloqueante** — `POST /auth/login` with `admin@pld.com` / `Admin123!` → **HTTP 201** with `{ token }`. Must pass before any other check.
2. **Guard** — `POST /admin/registration` with no JWT → `401`; with `USUARIO_INTERNO` JWT → `403`; with `SUPERADMIN` JWT → `201`.
3. **Happy path PF (Notaría)** — Start draft, send `identification + contact + vulnerable-activity + finalize`. Response of finalize contains a generated `tempPassword` + the new `userId`, and the `user` has `profile_type=PERSONA_FISICA` and `profile_id` pointing at `physical_person_profile`.
4. **Happy path PM (Inmobiliaria)** — Start draft, send `identification + contact + vulnerable-activity + compliance-responsible + finalize`. Same verification as PF, plus `moral_person_profile.compliance_responsible_id` is set.
5. **Dedupe** — Start a draft with RFC `X`, start another with same `X` → `409` with the existing `registration.id`.
6. **TTL** — Insert a draft with `expires_at = NOW() - 1 day` via SQL, then `GET /admin/registration` → it is NOT listed. `GET /admin/registration?includeExpired=true` lists it. `GET /admin/registration/:id` on that row → `410 Gone`.
7. **State machine** — On an `IN_PROGRESS` PF draft with `current_step = IDENTIFICATION`, call `POST /:id/vulnerable-activity` (skipping CONTACT) → `409`. Calling `/compliance-responsible` on a PF draft → `409`.
8. **Write-once after COMPLETED** — `POST /:id/contact` on a `COMPLETED` draft → `409`.
9. **Regresión** — `POST /participants-fisica/basica`, `POST /system-users/notario-inmobiliario`, `GET /catalogs/*` all return their pre-change statuses.

## Dependencies

- `JwtAuthGuard` at `apps/auth-users/src/shared/auth/jwt-auth.guard.ts` (exists).
- `UserEntity` at `packages/domain-auth-users/src/entities/user.entity.ts` (exists — extended here).
- `UserRole`, `TipoPersonaParticipante`, `ActividadVulnerable` enums in `@pld-api/shared-types` (exist).
- TypeORM 0.3 + MySQL 8 via `packages/persistence` `mysqlConfigFromEnv` (exists).
- bcrypt already in `packages/domain-auth-users` for password hashing (exists — reused for the generated temp password in `finalize`).

## Success Criteria

- [ ] 8 new endpoints return the status codes and shapes defined in `specs.md`.
- [ ] `RolesGuard` + `@Roles()` decorator are reusable from any controller in `auth-users`.
- [ ] Migration SQL creates 7 new tables + 2 new `users` columns idempotently (`IF NOT EXISTS`).
- [ ] Happy paths PF (Notaría) and PM (Inmobiliaria + compliance responsible) verified via curl/Postman.
- [ ] Resume an `IN_PROGRESS` draft via `GET /admin/registration/:id` succeeds.
- [ ] RFC dedupe, TTL hiding, `410 Gone` on expired detail, and state-machine 409s verified.
- [ ] `POST /auth/login` returns `HTTP 201` after the change.
- [ ] `libs/participants/*` entities are unchanged.
- [ ] Swagger (`/api`) groups the new endpoints under tag `admin-registration`.
