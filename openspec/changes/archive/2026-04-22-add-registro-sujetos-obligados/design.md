# Design: Registration of reporting entities

## Context

Backend design for the flow in [temp-docs/FLUJO_REGISTRO_SUPERADMIN.md](../../../temp-docs/FLUJO_REGISTRO_SUPERADMIN.md) using the schema fixed in [temp-docs/REGISTRO_SCHEMA.md](../../../temp-docs/REGISTRO_SCHEMA.md). The artefact exists to lock architectural decisions before `tasks.md` is written — `specs.md` is the contract, this doc is the why.

The module lives at **app layer**: `apps/auth-users/src/admin/registration/`. The `admin.module.ts` is already an empty shell. No `packages/domain-registration` is created (see Decision 8).

---

## Decision 1 — Completeness model

**Problem**: track at which step a draft stopped and whether it is finalized. No existing entity captures this.

**Alternatives considered**:

- **A. Dedicated `registration` table** holding `status`, `current_step`, FK to the profile row, FK to the nullable `user_id`.
- **B. Columns on `UserEntity`** (`registrationStep`, `registrationCompleted`).
- **C. Boolean flag per step on each participant entity** (`ParticipantPFBasicaEntity.completed`, etc.).
- **D. Polymorphic `registration_progress` table** over any registration type.

**Chosen: A**.

**Why**:
- The `user` does not exist until the draft finalizes — B and D are structurally impossible without changing the finalize semantics.
- A lets us express "the draft" as a first-class aggregate separate from the profile, which is the entity that survives the draft.
- A makes the listing query trivial: one SELECT over `registration` joined with its nested profile.
- C distributes state — detecting "current step" requires N SELECTs.

**Columns**: see `registration` table in `temp-docs/REGISTRO_SCHEMA.md` §ER diagram. Highlights:
- `status` and `current_step` are `varchar(30)` with app-layer enum validation (matches the existing convention — `users.role` is also `varchar`, not native MySQL ENUM).
- `deleted_at` is reserved for the future `DELETE` endpoint but remains `NULL` in this change.
- `user_id` is nullable until `POST /finalize` flips `status = COMPLETED`.

---

## Decision 2 — When is the `user` created

**Problem**: The legacy `POST /system-users/notario-inmobiliario` creates a `user` immediately. In the new wizard, should the draft start from an existing `user` (legacy flow) or create it lazily?

**Alternatives**:

- **A. Legacy** — SUPERADMIN creates the `user` first, then `POST /admin/registration` consumes that `userId`.
- **B. Lazy** — `POST /admin/registration` just starts the draft. `user` is created inside `POST /finalize` transaction.

**Chosen: B**.

**Why**:
- **Abandonment**: with A, an abandoned draft leaves a half-configured user row sitting in the system with an active password. B keeps `users` clean of drafts that never complete.
- **Email source**: the email captured in the wizard is the user's login — it is collected in `POST /contact`, not at step 0. With A, we would have to pre-ask it or patch the user's email at finalize (both ugly).
- **Atomicity**: the `users` row is created inside the same transaction that flips `registration.status = COMPLETED`. If the transaction fails, neither the user nor the status change apply — the draft stays `IN_PROGRESS` and can be retried.
- **Temp password single-exposure**: the cleartext is returned exactly once, inside the `finalize` response. With A, we would need a separate "reveal password" mechanism or force the SUPERADMIN to remember it across sessions.

**Consequence**: `user.role` is copied from `registration.user_role`, and `user.profile_type + user.profile_id` are filled atomically.

---

## Decision 3 — User ↔ Profile relation

**Problem**: how does a `user` point at its reporting-entity profile (`physical_person_profile` or `moral_person_profile`)?

**Alternatives**:

- **A. `profile_type` + `profile_id` on `users`**, resolved in app code by dispatch on `profile_type`.
- **B. Two nullable FKs** (`physical_profile_id`, `moral_profile_id`) on `users`.
- **C. Shared PK** — `profile.id = user.id`.
- **D. FK from profile to user**.

**Chosen: A**.

**Why**:
- A encodes the invariant "a user has 0 or 1 profile of exactly one type" in 2 columns instead of 2 nullable FKs that must be mutually exclusive.
- B leaks the type discriminator into the schema (two FK columns that must be mutually exclusive — constraint enforced by triggers or app logic anyway).
- C couples two aggregates whose lifetimes differ (profile can exist before the user, e.g. during the draft).
- D inverts the ownership: a "user" is the public-facing identity, so "user has profile" is the natural direction.

**Integrity**: app code MUST enforce "both null or both set" on `users.profile_type` + `users.profile_id`. A DB CHECK constraint is out of scope (MySQL 8 supports CHECK but our migration format is plain SQL — adding CHECKs across all tables is a separate hygiene change).

---

## Decision 4 — DTO strategy

**Problem**: one DTO per step or one master DTO with class-validator groups?

**Chosen: one DTO per step**, 1:1 with the endpoint. Same approach as the legacy `participants-*` controllers.

**Why**:
- NestJS 9 + class-validator 0.14 groups are brittle; `@ApiBody` Swagger generation gets confused.
- One DTO per step is readable (≤ 40 LOC each), Swagger groups them clearly, and each endpoint has a single source of truth.
- The PF vs PM split for identification becomes two separate DTOs (`PhysicalIdentificationDto`, `MoralIdentificationDto`). The controller dispatches on `registration.profile_type` and validates the matching DTO.

**DTO list** (see §File structure below): `CreateRegistrationDto`, `PhysicalIdentificationDto`, `MoralIdentificationDto`, `ContactDto`, `VulnerableActivityDto` (with nested `AddressDto`), `ComplianceResponsibleDto`, `FinalizeDto` (optional body).

---

## Decision 5 — Strict state machine vs flexible

**Problem**: can a step be posted out of order?

**Chosen: strict ordering**. `IDENTIFICATION → CONTACT → VULNERABLE_ACTIVITY → [COMPLIANCE_RESPONSIBLE] → COMPLETED`.

**Why**:
- The diagram is linear; the UX builds on in-order advancement.
- Strict ordering makes `current_step` trustworthy — if it says `CONTACT`, identification is guaranteed to exist.
- Out-of-order inserts create hard-to-detect orphaned states (a `vulnerable_activity` without a profile).

**Rule**: re-POSTing a step that was already saved **is allowed** while `status = IN_PROGRESS`. The row is UPDATEd (idempotent with overwrite); `current_step` never regresses. This supports the front's "go back and correct" flow without polluting the DB.

**Implementation**: the service computes `pasoAnteriorEsperado` as `STATE_ORDER[STATE_ORDER.indexOf(currentStep) - 1]` and rejects with `409` when the client tries to jump forward.

---

## Decision 6 — Post-completion edits

**Problem**: once `status = COMPLETED`, can steps be edited?

**Chosen: write-once after COMPLETED**. Any `POST` on a step endpoint for a `COMPLETED` draft returns `409` (`"Registration already finalized"`).

**Why**:
- A `COMPLETED` draft is a "signed" snapshot used to issue the user. Editing it post-hoc would require cascading changes to the `users` row.
- The user explicitly confirmed post-completion edits are a separate concern and will be modeled in a future change.

**Reserved for a future change** (outside this scope):
- `PATCH /admin/registration/:id/...` endpoints on `COMPLETED` drafts.
- `DELETE /admin/registration/:id` cancellation (hence `status = CANCELLED` + `deleted_at` stay in the schema now, unused).

---

## Decision 7 — RFC-based dedupe with FOR UPDATE

**Problem**: two SUPERADMINs simultaneously calling `POST /admin/registration` with the same `rfc` could race-insert two `IN_PROGRESS` drafts.

**Alternatives**:

- **A. Unique DB index on `(rfc, status)` where `status = IN_PROGRESS`.** Partial indexes not supported in MySQL 8 → impossible natively.
- **B. Unique index on `(rfc, status)` unconditionally.** Would require `status` to be `varchar` and accept the false uniqueness on `COMPLETED` as well — blocks legitimate re-registrations.
- **C. Application-layer SELECT ... FOR UPDATE inside a transaction**, before the INSERT.
- **D. Optimistic insert with a dedicated `unique(rfc, 'IN_PROGRESS_SENTINEL')` generated column.** Cute but relies on MySQL 8 generated columns + extra complexity for future readers.

**Chosen: C**.

**Why**:
- C works in any MySQL version and any storage engine.
- MySQL 8 InnoDB `SELECT ... FOR UPDATE` takes a gap lock on the matching rows, preventing race between two concurrent transactions.
- The operation is rare (SUPERADMIN starts a draft manually), so the tiny lock contention cost is acceptable.

**Implementation**: `RegistrationService.startDraft(dto)` opens a transaction, runs `SELECT id FROM registration WHERE rfc = ? AND status = 'IN_PROGRESS' FOR UPDATE`, and either returns `409 { registrationId: existing.id }` or INSERTs and commits.

**RFC denormalization**: because `rfc` lives on `physical_person_profile` / `moral_person_profile`, and the profile row is created in step 1 (`POST /identification`), we denormalize the `rfc` onto the `registration` row at `POST /admin/registration` time — it is part of the step-0 DTO. This is the only denormalized field; it is frozen at step 0 and the step-1 DTO MUST match it (specs §4). If the SUPERADMIN made a typo they cancel the draft and start a new one (cancellation is deferred, so for now they live with it until the TTL expires).

**Consequence for schema**: the `registration` table gets an implicit `rfc` column not shown in the REGISTRO_SCHEMA.md high-level diagram but required for dedupe. This is documented as part of this decision and materialized in the migration.

---

## Decision 8 — Module placement (app vs package)

**Problem**: should the registration module live in `packages/domain-*` (pure) or `apps/auth-users/src/admin/` (app)?

**Chosen: app layer** (`apps/auth-users/src/admin/registration/`).

**Why**:
- The module is inherently NestJS-flavoured: `@UseGuards`, `@Roles`, `@ApiBearerAuth`, `Reflector`, `TypeOrmModule.forFeature`. Pulling it into a pure package means re-wrapping every decorator with a port/adapter pair — indirection without reuse (no other app consumes this).
- No cross-app reuse. Only `auth-users` serves these routes.
- The pattern in the project is: core business model → package; Nest-bound plumbing → app. This module is plumbing.

**Consequence**: when/if another app needs the module (unlikely), extraction is a mechanical move (controllers & DTOs → package, keep Nest imports). Not worth doing prospectively.

---

## Decision 9 — `moral_person_profile.compliance_responsible_id` NOT NULL

**Problem**: during the PM flow, the `moral_person_profile` row is inserted at step 1, but the compliance officer is only captured at step 4. If the FK is `NOT NULL`, step 1 cannot insert the profile until step 4 exists — which is backwards.

**Alternatives**:

- **A. Keep `NOT NULL`, defer profile insert to step 4.** Steps 1–3 store their data elsewhere (on `registration`) and the `moral_person_profile` INSERT happens inside `POST /compliance-responsible` alongside the `compliance_responsible` INSERT.
- **B. Nullable + enforce at `finalize`.** `moral_person_profile` inserted at step 1; `compliance_responsible_id` filled at step 4. `POST /finalize` checks the column is set.
- **C. Break into two rows** — separate the "ever-resident" data of the PM profile from the "requires compliance" link.

**Chosen: B**.

**Why**:
- A requires `registration` to carry the denormalized PM identification fields (corporate name, incorporation date, ...) from step 1 to step 4 just to delay the INSERT. Ugly and forces a migration round-trip when the user corrects an earlier step.
- B keeps the profile row shaped normally at the cost of letting `compliance_responsible_id` be temporarily null during `IN_PROGRESS`. The `POST /finalize` validation enforces the invariant before the draft becomes `COMPLETED`, and `user.profile_id` is only set at finalize — so no "live" user ever points at a profile with a null compliance officer.
- C over-engineers the schema for a wizard step.

**Consequence**: `moral_person_profile.compliance_responsible_id` is `NULL`-able in the DDL in this change. The REGISTRO_SCHEMA.md §ER diagram shows it as "required" meaning "required for a finalized PM profile", which is enforced in app code at `POST /finalize`.

> **Note**: this deviates slightly from REGISTRO_SCHEMA.md wording, which implies DB-level `NOT NULL`. The deviation is intentional and documented here — the invariant is preserved at the finalize boundary. A future hygiene change could add a DB trigger or CHECK to enforce it at rest, but this change prefers the app-layer enforcement to keep the migration simple.

---

## Decision 10 — TTL configuration

**Chosen: env var `REGISTRATION_TTL_DAYS`, default `30`**. The service reads it once at start and computes `expires_at = NOW() + REGISTRATION_TTL_DAYS` when inserting the draft.

**Why env var**: the value differs per environment (dev: short TTLs to test expiry; prod: months). Hard-coding would require a redeploy to change. Keeping it in app config (not DB) avoids a config table for a single scalar.

**No cron purge**: expired drafts are NOT deleted. The listing filter hides them. Detail endpoint returns `410 Gone`. Cleanup is an operational task for later (a future job could `DELETE FROM registration WHERE expires_at < NOW() - INTERVAL 6 MONTH`).

---

## Sequence diagram — Happy path PF (Notaría)

```
SUPERADMIN          Controller          Service              DB
     │                   │                  │                  │
     │ POST /admin/registration {PF, NOTARIO, rfc=X}           │
     ├──────────────────►│                  │                  │
     │                   │ startDraft(dto)  │                  │
     │                   ├─────────────────►│ BEGIN            │
     │                   │                  │ SELECT ... FOR UPDATE WHERE rfc=X AND status=IN_PROGRESS
     │                   │                  ├─────────────────►│
     │                   │                  │ (no match)       │
     │                   │                  │ INSERT registration (current_step=IDENTIFICATION, status=IN_PROGRESS, expires_at=NOW()+30d)
     │                   │                  │ COMMIT           │
     │                   │◄─────────────────┤                  │
     │ ◄─── 201 {id, currentStep: IDENTIFICATION}               │
     │                                                          │
     │ POST /admin/registration/:id/identification {firstName, rfc=X, ...}
     ├──────────────────►│ saveIdentification                   │
     │                   ├─────────────────►│                  │
     │                   │                  │ INSERT physical_person_profile
     │                   │                  │ UPDATE registration SET physical_profile_id=..., current_step stays IDENTIFICATION
     │                   │                  ├─────────────────►│
     │ ◄─── 201 {currentStep: IDENTIFICATION, profileId}        │
     │                                                          │
     │ POST /admin/registration/:id/contact {email, cellphone, ...}
     ├──────────────────►│ appendContact    │                  │
     │                   ├─────────────────►│ INSERT contact   │
     │                   │                  │ UPDATE registration SET current_step=CONTACT (first time only)
     │                   │                  ├─────────────────►│
     │ ◄─── 201 {currentStep: CONTACT, totalContacts: 1}        │
     │                                                          │
     │ POST /admin/registration/:id/vulnerable-activity { activity, address, ... }
     ├──────────────────►│ saveVulnerableActivity              │
     │                   ├─────────────────►│ INSERT reporting_entity_address
     │                   │                  │ INSERT vulnerable_activity
     │                   │                  │ UPDATE registration SET vulnerable_activity_id=..., current_step=VULNERABLE_ACTIVITY
     │                   │                  ├─────────────────►│
     │ ◄─── 201 {currentStep: VULNERABLE_ACTIVITY}              │
     │                                                          │
     │ POST /admin/registration/:id/finalize                    │
     ├──────────────────►│ finalize         │                  │
     │                   ├─────────────────►│ BEGIN            │
     │                   │                  │ SELECT first contact (email)
     │                   │                  │ SELECT users WHERE email=... (expect empty)
     │                   │                  │ INSERT users (role=NOTARIO, profile_type=PF, profile_id=physical_profile_id, password_hash=bcrypt(random))
     │                   │                  │ UPDATE registration SET user_id=..., status=COMPLETED, current_step=COMPLETED
     │                   │                  │ COMMIT           │
     │                   │◄─────────────────┤                  │
     │ ◄─── 201 {userId, email, tempPassword, status: COMPLETED}│
```

---

## Sequence diagram — Happy path PM (Inmobiliaria)

```
SUPERADMIN          Controller          Service              DB
     │                   │                  │                  │
     │ POST /admin/registration {PM, INMOBILIARIA, rfc=Y}       │
     ├──────────────────►│ startDraft       │                  │
     │                   │◄─── 201 {id, IDENTIFICATION}         │
     │                                                          │
     │ POST /admin/registration/:id/identification {corporateName, rfc=Y, ...}
     ├──────────────────►│                  │                  │
     │                   │                  │ INSERT moral_person_profile (compliance_responsible_id = NULL)
     │                   │                  │ UPDATE registration SET moral_profile_id=...
     │ ◄─── 201 {IDENTIFICATION, profileId}                     │
     │                                                          │
     │ POST /admin/registration/:id/contact                     │
     ├──────────────────►│                  │                  │
     │ ◄─── 201 {CONTACT}                                       │
     │                                                          │
     │ POST /admin/registration/:id/vulnerable-activity         │
     ├──────────────────►│                  │                  │
     │ ◄─── 201 {VULNERABLE_ACTIVITY}                           │
     │                                                          │
     │ POST /admin/registration/:id/compliance-responsible      │
     ├──────────────────►│                  │                  │
     │                   │                  │ INSERT compliance_responsible
     │                   │                  │ UPDATE moral_person_profile SET compliance_responsible_id=...
     │                   │                  │ UPDATE registration SET current_step=COMPLIANCE_RESPONSIBLE
     │ ◄─── 201 {COMPLIANCE_RESPONSIBLE}                        │
     │                                                          │
     │ POST /admin/registration/:id/finalize                    │
     ├──────────────────►│                  │                  │
     │                   │                  │ BEGIN            │
     │                   │                  │ INSERT users (role=INMOBILIARIA, profile_type=PM, profile_id=moral_profile_id)
     │                   │                  │ UPDATE registration SET user_id=..., status=COMPLETED
     │                   │                  │ COMMIT           │
     │ ◄─── 201 {userId, email, tempPassword, COMPLETED}        │
```

---

## Endpoint → table write mapping

| Endpoint | Tables written | Fields on `registration` updated |
|---|---|---|
| `POST /admin/registration` | `registration` (INSERT) | `id`, `profile_type`, `user_role`, `rfc`, `current_step=IDENTIFICATION`, `status=IN_PROGRESS`, `started_by_user_id`, `expires_at` |
| `POST /:id/identification` (PF) | `physical_person_profile` (INSERT/UPDATE) | `physical_profile_id` |
| `POST /:id/identification` (PM) | `moral_person_profile` (INSERT/UPDATE) | `moral_profile_id` |
| `POST /:id/contact` | `contact` (INSERT) | `current_step=CONTACT` (first time only) |
| `POST /:id/vulnerable-activity` | `reporting_entity_address` (INSERT/UPDATE), `vulnerable_activity` (INSERT/UPDATE) | `vulnerable_activity_id`, `current_step=VULNERABLE_ACTIVITY` |
| `POST /:id/compliance-responsible` | `compliance_responsible` (INSERT/UPDATE), `moral_person_profile` (UPDATE `compliance_responsible_id`) | `current_step=COMPLIANCE_RESPONSIBLE` |
| `POST /:id/finalize` | `users` (INSERT) | `user_id`, `status=COMPLETED`, `current_step=COMPLETED` |
| `GET /admin/registration` / `GET /:id` | (read only) | — |

---

## File structure

```
apps/auth-users/src/admin/
├── admin.module.ts                       # Existing, becomes: imports: [RegistrationModule]
└── registration/
    ├── registration.module.ts            # NestJS @Module — TypeOrmModule.forFeature + providers + controller
    ├── registration.controller.ts        # @Controller('admin/registration')  — 8 endpoints, class-level @UseGuards + @Roles(SUPERADMIN)
    ├── registration.service.ts           # State machine, RFC dedupe (SELECT FOR UPDATE), TTL, bcrypt user creation
    ├── registration.adapter.ts           # @Injectable — TypeORM writes, thin wrapper over repositories
    ├── registration.enums.ts             # RegistrationStatus, RegistrationStep (internal; NOT exported from shared-types)
    ├── dto/
    │   ├── create-registration.dto.ts
    │   ├── physical-identification.dto.ts
    │   ├── moral-identification.dto.ts
    │   ├── contact.dto.ts
    │   ├── vulnerable-activity.dto.ts    # includes nested AddressDto
    │   ├── compliance-responsible.dto.ts
    │   ├── finalize.dto.ts
    │   └── list-registrations-query.dto.ts
    └── entities/
        ├── registration.entity.ts
        ├── physical-person-profile.entity.ts
        ├── moral-person-profile.entity.ts
        ├── contact.entity.ts
        ├── reporting-entity-address.entity.ts
        ├── compliance-responsible.entity.ts
        └── vulnerable-activity.entity.ts

apps/auth-users/src/shared/auth/
├── jwt-auth.guard.ts                     # Existing
├── jwt.strategy.ts                       # Existing
├── current-user.decorator.ts             # Existing
├── roles.guard.ts                        # NEW — @Injectable, Reflector, checks request.user.role against metadata
└── roles.decorator.ts                    # NEW — SetMetadata(ROLES_KEY, roles)

packages/domain-auth-users/src/entities/user.entity.ts
                                          # MODIFIED — add profile_type + profile_id columns
```

---

## Env vars (new)

| Name | Default | Purpose |
|---|---|---|
| `REGISTRATION_TTL_DAYS` | `30` | Days added to `NOW()` when computing `registration.expires_at`. Read once at module load via `ConfigService` or `process.env`. |

No other env vars are added. `TYPEORM_*` and JWT secrets already exist.

---

## Migration plan

One consolidated SQL migration. Name follows the `YYYYMMDDHHMMSS-<description>.sql` convention expected by the deployment tooling:

```
20260421000000-create-registration-schema.sql
```

The file creates, in order:
1. `reporting_entity_address` (no FKs — referenced by others).
2. `compliance_responsible` (no FKs — referenced by `moral_person_profile`).
3. `physical_person_profile` (FK to `reporting_entity_address`).
4. `moral_person_profile` (FK to `reporting_entity_address`, FK to `compliance_responsible`, nullable per Decision 9).
5. `vulnerable_activity` (FK to `reporting_entity_address`).
6. `contact` (FK to `registration` created next, so the contact FK is deferred to an `ALTER TABLE` at the bottom OR `contact` is created *after* `registration`).
7. `registration` (FKs to profiles, activity, `user_id` nullable, `started_by_user_id` NOT NULL, denormalized `rfc` column per Decision 7).
8. `ALTER TABLE users ADD COLUMN profile_type VARCHAR(30) NULL, ADD COLUMN profile_id CHAR(36) NULL`.
9. Indexes: `idx_registration_rfc_status`, `idx_registration_expires_at`, `idx_contact_registration_id`.

Migration MUST use `IF NOT EXISTS` where supported to keep re-runs idempotent in dev.

Reverse migration drops in reverse order (tables first, then columns on `users`).

---

## Open risks / trade-offs

- **Denormalized `rfc` on `registration`**: the true source of truth lives on the profile row. If someone UPDATEs the profile's `rfc`, the denormalized one on `registration` goes stale. Mitigation: `POST /identification` validates `dto.rfc === registration.rfc` and rejects mismatches; no endpoint allows changing RFC post step 0.
- **`moral_person_profile.compliance_responsible_id` nullable in DB, enforced in app**: a buggy future caller could insert a PM profile and skip the compliance step. Mitigation: the `/finalize` guard + the fact that the state machine prevents skipping.
- **No DB CHECK on `users.(profile_type, profile_id)` coherence**: app code is the only guardrail. Mitigation: the only writer of those columns is `/finalize`, and it sets both atomically.
- **Temp password exposure**: if the `201` response is logged upstream (access logs, APM), the cleartext password leaks. Mitigation: document in the controller that this endpoint's response MUST NOT be logged at body level; verify `HttpErrorInterceptor` / logging config does not capture success bodies.

---

## Nota sobre leyendas del diagrama

Las leyendas *"Para el caso de Notarías deberá decir: FE PÚBLICA (SERVIDORES PÚBLICOS...)"* y *"En caso de inmobiliarias deberá decir: TRANSMISIÓN DE BIENES INMUEBLES"* son **copy del front** — etiquetas de sección mostradas al SUPERADMIN según el tipo de sujeto obligado. No son valores de dominio. El backend recibe y valida solamente el enum `ActividadVulnerable` en el DTO. Esta nota queda registrada para evitar confusión en revisiones posteriores.
