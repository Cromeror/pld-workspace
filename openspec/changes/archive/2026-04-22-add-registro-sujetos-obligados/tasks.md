# Tasks: add-registro-sujetos-obligados

Hierarchical checklist. Each task sized to fit a single session (~30 min to 2 h).
Patrón: `controller → adapter → service` + `TypeOrmModule.forFeature` registered per module.
Reference docs: [REGISTRO_SCHEMA.md](../../../temp-docs/REGISTRO_SCHEMA.md) (schema), [FLUJO_REGISTRO_SUPERADMIN.md](../../../temp-docs/FLUJO_REGISTRO_SUPERADMIN.md) (flow), `design.md` (decisions), `specs.md` (contract).

---

## Phase 1 — Infrastructure

### 1.1 Guards and decorators (reusable)

- [x] 1.1.1 Create `apps/auth-users/src/shared/auth/roles.decorator.ts` — export `ROLES_KEY` constant (string) and `@Roles(...roles: UserRole[])` using `SetMetadata(ROLES_KEY, roles)` from `@nestjs/common`. Export `UserRole` from `@pld-api/shared-types` for convenience (no re-export).
- [x] 1.1.2 Create `apps/auth-users/src/shared/auth/roles.guard.ts` — `@Injectable` class implementing `CanActivate`. Inject `Reflector`. Read `ROLES_KEY` via `reflector.getAllAndOverride(ROLES_KEY, [context.getHandler(), context.getClass()])`. If metadata is missing, `return true` (passthrough — the `JwtAuthGuard` still protects). Compare against `request.user.role`. Mismatch → `throw new ForbiddenException('Insufficient role')`.

### 1.2 Extend `users` table

- [x] 1.2.1 Edit `packages/domain-auth-users/src/entities/user.entity.ts` — add TypeORM `@Column({ name: 'profile_type', type: 'varchar', length: 30, nullable: true })` and `@Column({ name: 'profile_id', type: 'char', length: 36, nullable: true })`. Type these as `TipoPersonaParticipante | null` and `string | null`. Keep the class free of NestJS imports (Decision 8 applies in reverse — this package is pure).
- [x] 1.2.2 Verify `apps/auth-users` still compiles after the entity change (`pnpm nx run auth-users:build`).

### 1.3 Entities (TypeORM, app-layer)

Create all seven under `apps/auth-users/src/admin/registration/entities/`:

- [x] 1.3.1 `reporting-entity-address.entity.ts` — `@Entity({ name: 'reporting_entity_address' })`. Columns: `id` (uuid PK), `street`, `exterior_number`, `interior_number` (nullable), `neighborhood`, `municipality`, `city`, `state`, `postal_code`, `country`, `road_type` (nullable), `locality` (nullable), `created_at`, `updated_at`.
- [x] 1.3.2 `compliance-responsible.entity.ts` — `@Entity({ name: 'compliance_responsible' })`. Columns per schema (`first_name`, `paternal_surname`, `maternal_surname` required; `rfc`, `curp` required; `birth_date`, `nationality_country`, `designation_date` nullable).
- [x] 1.3.3 `physical-person-profile.entity.ts` — `@Entity({ name: 'physical_person_profile' })`. Required: `first_name`, `paternal_surname`, `maternal_surname`, `birth_date`, `rfc`, `curp`. Optional: `nationality_country`, `birth_country`. FK `reporting_entity_address_id` nullable.
- [x] 1.3.4 `moral-person-profile.entity.ts` — `@Entity({ name: 'moral_person_profile' })`. Required: `corporate_name`, `rfc`. Optional: `incorporation_date`, `nationality_country`. FKs: `reporting_entity_address_id` nullable, `compliance_responsible_id` **nullable in DB, enforced at /finalize** (Decision 9 — document in JSDoc comment on the column).
- [x] 1.3.5 `vulnerable-activity.entity.ts` — `@Entity({ name: 'vulnerable_activity' })`. Columns: `activity` (varchar, validated against `ActividadVulnerable` enum), `start_date` (nullable), `reporting_entity_address_id` NOT NULL (FK), `activity_performed_at_address` text NOT NULL.
- [x] 1.3.6 `contact.entity.ts` — `@Entity({ name: 'contact' })`. Columns: `id`, `registration_id` FK (no TypeORM relation — plain column), `country_code` (nullable), `phone` (nullable), `email` NOT NULL, `cellphone` NOT NULL.
- [x] 1.3.7 `registration.entity.ts` — `@Entity({ name: 'registration' })`. Columns per schema + denormalized `rfc` (Decision 7) + all the enum string fields (`profile_type`, `user_role`, `current_step`, `status`). FKs: `physical_profile_id`, `moral_profile_id`, `vulnerable_activity_id`, `user_id`, `started_by_user_id`. Timestamps: `expires_at`, `created_at`, `updated_at`, `deleted_at` (nullable).

### 1.4 Internal enums

- [x] 1.4.1 Create `apps/auth-users/src/admin/registration/registration.enums.ts` — export `RegistrationStatus` (`IN_PROGRESS`, `COMPLETED`, `CANCELLED`) and `RegistrationStep` (`IDENTIFICATION`, `CONTACT`, `VULNERABLE_ACTIVITY`, `COMPLIANCE_RESPONSIBLE`, `COMPLETED`) and a `STEP_ORDER: RegistrationStep[]` for state-machine math. **Do not** promote these to `@pld-api/shared-types` — they are internal to this module.
- [x] 1.4.2 Reuse `UserRole`, `TipoPersonaParticipante`, `ActividadVulnerable` from `@pld-api/shared-types`. Do NOT create duplicates.

### 1.5 SQL migration

- [x] 1.5.1 Create `20260421000000-create-registration-schema.sql` (location: follow the existing migration convention in the repo — verify under `packages/persistence` or `apps/auth-users`; if none exists, drop at repo root `migrations/` and document the path in the PR). Contents per `design.md` §Migration plan. Order: `reporting_entity_address`, `compliance_responsible`, `physical_person_profile`, `moral_person_profile`, `vulnerable_activity`, `registration`, `contact` (contact FK deferred after registration), then `ALTER TABLE users`.
- [x] 1.5.2 Add indexes: `idx_registration_rfc_status` on `(rfc, status)`, `idx_registration_expires_at` on `(expires_at)`, `idx_registration_user_id` on `(user_id)`, `idx_contact_registration_id` on `(registration_id)`.
- [x] 1.5.3 Write the reverse migration file (`20260421000000-create-registration-schema.down.sql` or inline `-- DOWN` block per project convention). Order: drop `contact`, `registration`, `vulnerable_activity`, `moral_person_profile`, `physical_person_profile`, `compliance_responsible`, `reporting_entity_address`; then `ALTER TABLE users DROP COLUMN profile_id, DROP COLUMN profile_type`.
- [x] 1.5.4 Run the migration in dev (`docker-compose.dev.yml` MySQL at `:13306`) and verify with `SHOW TABLES` that all 7 new tables exist and `DESCRIBE users` shows the 2 new columns.

### 1.6 Module wiring

- [x] 1.6.1 Create `apps/auth-users/src/admin/registration/registration.module.ts` — `@Module({ imports: [TypeOrmModule.forFeature([RegistrationEntity, PhysicalPersonProfileEntity, MoralPersonProfileEntity, ContactEntity, ReportingEntityAddressEntity, ComplianceResponsibleEntity, VulnerableActivityEntity, UserEntity])], controllers: [RegistrationController], providers: [RegistrationService, RegistrationAdapter, RolesGuard] })`.
- [x] 1.6.2 Edit `apps/auth-users/src/admin/admin.module.ts` — `imports: [RegistrationModule]`, `exports: [RegistrationModule]` (if other admin sub-modules will consume it in the future; otherwise just imports).
- [x] 1.6.3 Edit `apps/auth-users/src/auth-users.module.ts` — add `AdminModule` to the `imports` array.

---

## Phase 2 — Implementation

### 2.1 DTOs

Under `apps/auth-users/src/admin/registration/dto/`:

- [x] 2.1.1 `create-registration.dto.ts` — `CreateRegistrationDto`: `profileType` (`@IsEnum(TipoPersonaParticipante)`, restrict to PF/PM — reject `FIDEICOMISO` via custom validator or `@IsIn`), `userRole` (`@IsIn([UserRole.NOTARIO, UserRole.INMOBILIARIA])`), `rfc` (`@IsString @Length(12,13) @Matches(regexRFC)`).
- [x] 2.1.2 `list-registrations-query.dto.ts` — `ListRegistrationsQueryDto`: `status?: RegistrationStatus` (`@IsOptional @IsEnum`), `includeExpired?: boolean` (`@IsOptional @IsBoolean` + `@Transform(({value}) => value === 'true')`).
- [x] 2.1.3 `physical-identification.dto.ts` — `PhysicalIdentificationDto` with all PF required fields per specs §4 (first/paternal/maternal names, birthDate, rfc, curp) + optional nationalityCountry, birthCountry.
- [x] 2.1.4 `moral-identification.dto.ts` — `MoralIdentificationDto` with `corporateName`, `rfc` required; `incorporationDate`, `nationalityCountry` optional.
- [x] 2.1.5 `contact.dto.ts` — `ContactDto`: `countryCode?`, `phone?`, `email` (`@IsEmail`), `cellphone` required.
- [x] 2.1.6 `vulnerable-activity.dto.ts` — `VulnerableActivityDto` with `activity` (`@IsEnum(ActividadVulnerable)`), `startDate?`, nested `address: AddressDto` (`@ValidateNested() @Type(() => AddressDto)`), `activityPerformedAtAddress` required. Define `AddressDto` inline or as a sibling class with the 11 fields.
- [x] 2.1.7 `compliance-responsible.dto.ts` — `ComplianceResponsibleDto` with `firstName`, `paternalSurname`, `maternalSurname`, `rfc`, `curp` required; `birthDate?`, `nationalityCountry?`, `designationDate?`.
- [x] 2.1.8 `finalize.dto.ts` — `FinalizeDto` — empty class or with reserved `notifyByEmail?: boolean` (not used in this change, documented as reserved).

### 2.2 Service — state machine, dedupe, TTL, finalize

File: `apps/auth-users/src/admin/registration/registration.service.ts` — `@Injectable`, inject all 8 repositories + `ConfigService` (or read `process.env.REGISTRATION_TTL_DAYS`).

- [x] 2.2.1 Private helper `assertActive(registration)` — throws `GoneException` (HTTP 410) if `expires_at < NOW()`, `ConflictException` if `status === COMPLETED`, `NotFoundException` if `status === CANCELLED`.
- [x] 2.2.2 Private helper `assertStepOrder(registration, expectedPrior: RegistrationStep)` — uses `STEP_ORDER` indices; throws `ConflictException` with `"Missing prior step: <name>"` if `STEP_ORDER.indexOf(current_step) < STEP_ORDER.indexOf(expectedPrior)`.
- [x] 2.2.3 Private helper `advanceStep(registration, newStep)` — updates only if `STEP_ORDER.indexOf(newStep) > STEP_ORDER.indexOf(current_step)`.
- [x] 2.2.4 `startDraft(dto, startedByUserId)` — opens TypeORM transaction, runs `SELECT id FROM registration WHERE rfc = ? AND status = 'IN_PROGRESS' FOR UPDATE`. If hit → throw `ConflictException({ registrationId: existing.id })`. Else INSERT with `expires_at = NOW() + REGISTRATION_TTL_DAYS`, `current_step = IDENTIFICATION`, `status = IN_PROGRESS`. Validate `profileType !== FIDEICOMISO` and `userRole === NOTARIO → profileType === PERSONA_FISICA` before the SELECT.
- [x] 2.2.5 `list(query)` — builds `WHERE` dynamically: `expires_at > NOW()` unless `includeExpired=true`; `status = ?` if provided; always `status != CANCELLED` unless `includeExpired` implies otherwise (keep CANCELLED hidden always). Order `updated_at DESC`.
- [x] 2.2.6 `getDetail(id)` — hydrates the aggregate: `registration` + matching profile (PF or PM) + contacts[] + vulnerable_activity (with its address) + compliance_responsible (PM) + addresses. Returns `null` if not found → controller maps to 404. Returns `expired` marker or throws directly for 410.
- [x] 2.2.7 `saveIdentification(id, dto)` — `assertActive`, assert DTO shape matches `profile_type` (dispatch on incoming DTO's class or a discriminator the controller passes). INSERT/UPDATE profile. Validate `dto.rfc === registration.rfc`. UPDATE `registration.physical_profile_id` or `moral_profile_id`. Don't advance `current_step` beyond `IDENTIFICATION` (it starts there).
- [x] 2.2.8 `appendContact(id, dto)` — `assertActive`, `assertStepOrder(registration, IDENTIFICATION)`. INSERT `contact`. `advanceStep(registration, CONTACT)` only if `current_step === IDENTIFICATION`.
- [x] 2.2.9 `saveVulnerableActivity(id, dto)` — `assertActive`, `assertStepOrder(CONTACT)`. Assert `contacts.length >= 1`. UPSERT `reporting_entity_address` + `vulnerable_activity`. UPDATE `registration.vulnerable_activity_id`. `advanceStep(VULNERABLE_ACTIVITY)`.
- [x] 2.2.10 `saveComplianceResponsible(id, dto)` — `assertActive`, `assertStepOrder(VULNERABLE_ACTIVITY)`. Assert `profile_type === PERSONA_MORAL` (else 409). UPSERT `compliance_responsible`. UPDATE `moral_person_profile.compliance_responsible_id`. `advanceStep(COMPLIANCE_RESPONSIBLE)`.
- [x] 2.2.11 `finalize(id)` — transaction: `assertActive`. Assert required prior step (`VULNERABLE_ACTIVITY` for PF, `COMPLIANCE_RESPONSIBLE` for PM). Assert `contacts.length >= 1`. Generate random 12-char password (`crypto.randomBytes(9).toString('base64')`). Hash with bcrypt (reuse existing helper in `packages/domain-auth-users`; if none exists for hashing, call `bcrypt.hash(pwd, 10)` directly). INSERT `users` with `email = contacts[0].email`, `role = registration.user_role`, `profile_type = registration.profile_type`, `profile_id = registration.physical_profile_id ?? registration.moral_profile_id`. Catch email-already-exists as `ConflictException("Email already registered")`. UPDATE `registration.user_id`, `status = COMPLETED`, `current_step = COMPLETED`. Return `{ registrationId, userId, email, tempPassword, status, currentStep, message }`.

### 2.3 Adapter

- [x] 2.3.1 Create `apps/auth-users/src/admin/registration/registration.adapter.ts` — `@Injectable`. Inject `RegistrationService`. Methods 1:1 with service but shape response DTOs (plain objects with camelCase keys). Matches the pattern from `apps/auth-users/src/participants/persona-fisica/participants.adapter.ts`.

### 2.4 Controller

File: `apps/auth-users/src/admin/registration/registration.controller.ts`.

- [x] 2.4.1 Class-level: `@Controller('admin/registration')`, `@UseGuards(JwtAuthGuard, RolesGuard)`, `@Roles(UserRole.SUPERADMIN)`, `@ApiTags('admin-registration')`, `@ApiBearerAuth()`.
- [x] 2.4.2 `POST /` → `startDraft(@Body() dto, @CurrentUser() user)`. Response `201`.
- [x] 2.4.3 `GET /` → `list(@Query() query: ListRegistrationsQueryDto)`. Response `200`.
- [x] 2.4.4 `GET /:id` → `getDetail(@Param('id', ParseUUIDPipe) id)`. Response `200` or `404`/`410`.
- [x] 2.4.5 `POST /:id/identification` → controller inspects `registration.profile_type` and validates the corresponding DTO (use `class-transformer.plainToInstance` + `validate` manually, since Nest body-pipe picks one class per route). Alternative: expose two sibling routes `/:id/identification/pf` and `/.../pm` but specs fix a single route → dispatch in-controller.
- [x] 2.4.6 `POST /:id/contact` → `appendContact`. Response `201`.
- [x] 2.4.7 `POST /:id/vulnerable-activity` → `saveVulnerableActivity`. Response `201` or `200` on overwrite (`@HttpCode` via conditional or keep constant `201` and document in Swagger `@ApiResponse`).
- [x] 2.4.8 `POST /:id/compliance-responsible` → `saveComplianceResponsible`. Response `201`.
- [x] 2.4.9 `POST /:id/finalize` → `finalize`. Response `201` (new resource `users` created). Add `@ApiOperation({ summary: "Finalizes draft; returns tempPassword in cleartext ONCE" })` and a prominent JSDoc warning that the response MUST NOT be logged at body level.

### 2.5 Swagger smoke

- [x] 2.5.1 Run `pnpm nx run auth-users:serve`, open `/api` (Swagger UI), confirm all 8 endpoints appear under the `admin-registration` tag with the right DTOs and response shapes.

---

## Phase 3 — Testing

### 3.1 Login bloqueante

- [x] 3.1.1 `POST /auth/login` with `admin@pld.com` / `Admin123!` → **HTTP 201** + body `{ token }`. MUST pass before any downstream test.

### 3.2 Happy path PF (Notaría) — curl script

- [x] 3.2.1 `POST /admin/registration` `{ profileType: "PERSONA_FISICA", userRole: "NOTARIO", rfc: "XAXX010101000" }` → `201` + draft id.
- [x] 3.2.2 `POST /admin/registration/:id/identification` with PF payload → `201`, `currentStep = IDENTIFICATION`.
- [x] 3.2.3 `POST /admin/registration/:id/contact` → `201`, `currentStep = CONTACT`, `totalContacts = 1`.
- [x] 3.2.4 `POST /admin/registration/:id/vulnerable-activity` → `201`, `currentStep = VULNERABLE_ACTIVITY`.
- [x] 3.2.5 `POST /admin/registration/:id/finalize` → `201`, `status = COMPLETED`, body contains `tempPassword` cleartext + `userId`.
- [x] 3.2.6 `SELECT * FROM users WHERE id = :userId` → row has `role = NOTARIO`, `profile_type = PERSONA_FISICA`, `profile_id = registration.physical_profile_id`.

### 3.3 Happy path PM (Inmobiliaria)

- [x] 3.3.1 `POST /admin/registration` `{ profileType: "PERSONA_MORAL", userRole: "INMOBILIARIA", rfc: "ABC900101X01" }` → `201`.
- [x] 3.3.2 `POST /:id/identification` with PM payload (`corporateName`) → `201`.
- [x] 3.3.3 `POST /:id/contact` → `201`.
- [x] 3.3.4 `POST /:id/vulnerable-activity` → `201`.
- [x] 3.3.5 `POST /:id/compliance-responsible` → `201`, `currentStep = COMPLIANCE_RESPONSIBLE`, `moral_person_profile.compliance_responsible_id` set.
- [x] 3.3.6 `POST /:id/finalize` → `201`, `user.role = INMOBILIARIA`, `user.profile_type = PERSONA_MORAL`.

### 3.4 Casos adversos

- [x] 3.4.1 `POST /admin/registration` without JWT → `401`.
- [x] 3.4.2 Valid JWT with `role = AUXILIAR` → `403`.
- [x] 3.4.3 `POST /admin/registration` with `{ userRole: "NOTARIO", profileType: "PERSONA_MORAL" }` → `400`.
- [x] 3.4.4 Start a draft with RFC `X`, then call again with the same RFC while first is `IN_PROGRESS` → `409` with `{ registrationId }`.
- [x] 3.4.5 Skip `CONTACT`: on fresh draft, `POST /:id/vulnerable-activity` → `409` (`"Missing prior step: CONTACT"`).
- [x] 3.4.6 `POST /:id/compliance-responsible` on a PF draft → `409`.
- [x] 3.4.7 `POST /:id/finalize` on a PM draft still at `VULNERABLE_ACTIVITY` → `409`.
- [x] 3.4.8 `POST /:id/contact` on a `COMPLETED` draft → `409` (`"Registration already finalized"`).
- [x] 3.4.9 Manually `UPDATE registration SET expires_at = NOW() - INTERVAL 1 DAY WHERE id=:id`. Then `GET /:id` → `410 Gone`. `POST /:id/contact` → `410`.
- [x] 3.4.10 `GET /admin/registration` without filter → expired draft is absent. `GET /admin/registration?includeExpired=true` → present.
- [x] 3.4.11 Finalize with email that already exists in `users` → `409`, draft remains `IN_PROGRESS`.
- [x] 3.4.12 `GET /admin/registration/<random-uuid>` → `404`.

### 3.5 Resume flow

- [x] 3.5.1 Start draft + identification + 1 contact → walk away.
- [x] 3.5.2 `GET /admin/registration?status=IN_PROGRESS` → draft listed with `currentStep = CONTACT`.
- [x] 3.5.3 `GET /admin/registration/:id` → body has the saved identification + `contacts.length === 1` + `vulnerableActivity = null`.
- [x] 3.5.4 Continue with `POST /:id/vulnerable-activity` and `finalize`. Draft completes successfully.

### 3.6 Regresión

- [x] 3.6.1 `POST /auth/login admin@pld.com/Admin123!` → `201` (bloqueante, re-ejecutar al final).
- [x] 3.6.2 `POST /system-users/notario-inmobiliario` (existing endpoint) → `201` (not broken by AdminModule import).
- [x] 3.6.3 `POST /participants-fisica/basica` → `201` (legacy controller unchanged).
- [x] 3.6.4 `GET /catalogs/beneficiario` → `200`.
- [x] 3.6.5 `nx run auth-users:build`, `nx run catalogs:build`, `nx run cross:build` all succeed.

---

## Phase 4 — Cleanup

- [x] 4.1.1 `grep -r '@Roles' apps/auth-users/` — exports clean, 1 definition, ≥ 1 usage (`RegistrationController`).
- [x] 4.1.2 `grep -r 'RolesGuard' apps/auth-users/` — 1 definition + registration in `RegistrationModule`.
- [x] 4.1.3 Verify `apps/auth-users/src/admin/admin.module.ts` imports `RegistrationModule` and nothing else.
- [x] 4.1.4 Verify `auth-users.module.ts` imports `AdminModule` and compilation passes (`nx run auth-users:build`).
- [x] 4.1.5 Document `REGISTRATION_TTL_DAYS` in the app's env template (`.env.example` if present) or in `docker-compose.dev.yml` service env.
- [x] 4.1.6 Run the archive report via `/sdd-archive` — it captures the final state and closes the change.

---

## Dependency order

- Phase 1.1, 1.2, 1.3, 1.4 can run in parallel. 1.5 (migration) depends on 1.3 (entities — to validate column names match).
- Phase 1.6 (module wiring) depends on 1.1 + 1.3 + 1.4.
- Phase 2.1 (DTOs) can start in parallel with Phase 1.
- Phase 2.2 (service) depends on 1.3 + 1.4 + 2.1.
- Phase 2.3 depends on 2.2.
- Phase 2.4 depends on 2.1 + 2.3 + 1.1.
- Phase 3 requires all of Phase 1 + 2 and a running dev env with the migration applied.
- Phase 4 requires Phase 3.

**Estimated effort**: 12–16 h of apply + 3–4 h of manual verification. ~3 apply sessions.
