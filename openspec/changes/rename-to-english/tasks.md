# Tasks: rename-to-english

> This is the umbrella task list for the full migration. Each phase maps to a sub-change. Phases 2–8 are broken into sub-changes as described in `design.md`.
>
> Sub-change mapping:
> - Phase 1 → `rename-enums-to-english`
> - Phases 2–5 → `rename-participants-entities-to-english`
> - Phases 6–8 → `rename-users-and-types-to-english`
> - Phase 9 → cleanup (runs after deprecation window ends)

---

## Phase 1 — Audit

**Goal**: produce a definitive inventory of every Spanish identifier before touching code.

- [ ] 1.1 Audit `packages/shared-types/src/enums.ts` — list every enum name and every value that needs renaming. Confirm there are no additional enum files in other packages. (~30 min)
- [ ] 1.2 Audit `libs/participants/src/lib/fisica/participants.entity.ts` — list every TypeORM `@Column({ name: ... })` that is missing an explicit `name:` (those rely on TypeORM's camelCase→snake_case conversion). Document the actual DB column names per column. (~45 min)
- [ ] 1.3 Audit `libs/participants/src/lib/moral/participants.entity.ts` — same as 1.2. (~30 min)
- [ ] 1.4 Audit `libs/participants/src/lib/fideicomiso/participants.entity.ts` — same as 1.2. (~30 min)
- [ ] 1.5 Audit `libs/participants/src/lib/anexo-7/anexo-7.entity.ts` — same as 1.2. (~30 min)
- [ ] 1.6 Audit `libs/participants/src/lib/beneficiario-controlador/ben-controlador.entity.ts` — list entity class names, table names, column names. (~20 min)
- [ ] 1.7 Audit `packages/domain-auth-users/src/entities/user.entity.ts` — confirm actual DB column names for `nombre`, `apellido_paterno`, `apellido_materno`, `telefono`, `activo`. (~20 min)
- [ ] 1.8 Audit `libs/auth-profiles/src/lib/auth-profiles.types.ts` and all types in `packages/shared-types` — list all Spanish TypeScript type/interface/field names. (~30 min)
- [ ] 1.9 Grep for raw SQL strings in `apps/` and `libs/` that reference any Spanish table or column name directly (not via TypeORM). Document any found. (~45 min)
- [ ] 1.10 Verify that `synchronize: false` is set in TypeORM config for all environments (preventing accidental auto-migration). (~15 min)

---

## Phase 2 — Enums (`rename-enums-to-english` sub-change)

**Goal**: rename all enum names and values in `packages/shared-types`, add dual support compat layer, update all consumers.

- [ ] 2.1 Rename enum names in `packages/shared-types/src/enums.ts`:
  - `TipoPersonaParticipante` → `ProfileType`
  - `ActividadVulnerable` → `VulnerableActivity`
  - `TipoMoneda` → `CurrencyType`
  - `FormaPago` → `PaymentMethod`
  - `TipoReporte` → `ReportType`
  - `ModalidadAtencion` → `AttendanceMode`
  - `Nacionalidad` → `Nationality`
  Add re-exports with old names for one release cycle: `export { ProfileType as TipoPersonaParticipante }` etc. (~45 min)
- [ ] 2.2 Rename enum values for `UserRole` (keep enum name):
  `NOTARIO` → `NOTARY`, `INMOBILIARIA` → `REAL_ESTATE`, `AUXILIAR` → `AUXILIARY`, `USUARIO_INTERNO` → `INTERNAL_USER`, `USUARIO_EXTERNO` → `EXTERNAL_USER`. (~30 min)
- [ ] 2.3 Rename enum values for `ProfileType` (formerly `TipoPersonaParticipante`):
  `PERSONA_FISICA` → `INDIVIDUAL`, `PERSONA_MORAL` → `LEGAL_ENTITY`, `FIDEICOMISO` → `TRUST`. (~20 min)
- [ ] 2.4 Rename enum values for `VulnerableActivity` (formerly `ActividadVulnerable`) — all 10 values per specs.md §1.4. (~30 min)
- [ ] 2.5 Rename enum values for `Nationality`, `PaymentMethod`, `AttendanceMode`, `ReportType` per specs.md §1.5–1.9. (~30 min)
- [ ] 2.6 Update all imports of old enum names across `libs/`, `packages/`, `apps/` (global search and replace). Verify TypeScript compilation after. (~60 min)
- [ ] 2.7 Add `normalizeRole()` compat function in `apps/auth-users/src/shared/auth/` (or a shared utils file) per design.md Decision 1. Add equivalent functions for `ProfileType` and `VulnerableActivity`. (~45 min)
- [ ] 2.8 Wire compat functions in all adapter files that receive enum values from HTTP requests (`auth.adapter.ts`, `users.adapter.ts`, participant adapters if any). (~45 min)
- [ ] 2.9 Update all `@ApiProperty({ example: 'NOTARIO' })` and similar Swagger annotations to use new values. (~30 min)
- [ ] 2.10 Write SQL data migration for `UserRole` values in `users` table:
  ```sql
  UPDATE users SET role = 'NOTARY' WHERE role = 'NOTARIO';
  UPDATE users SET role = 'REAL_ESTATE' WHERE role = 'INMOBILIARIA';
  -- etc.
  ```
  With down migration reversing each UPDATE. (~45 min)
- [ ] 2.11 Write SQL data migration for `ProfileType` values in any column that stores `PERSONA_FISICA`, `PERSONA_MORAL`, `FIDEICOMISO`. Audit which tables store these values first (from Phase 1 audit). (~45 min)
- [ ] 2.12 Run data migrations in dev environment. Verify with SELECT COUNT sanity checks. (~30 min)
- [ ] 2.13 Smoke test: `POST /auth/login` returns HTTP 201 + JWT with `role: "NOTARY"`. (~20 min)
- [ ] 2.14 Smoke test: front-end sends `role: "NOTARIO"` → compat layer normalizes → response returns `role: "NOTARY"`. Verify `[DEPRECATED]` warn appears in logs. (~20 min)

---

## Phase 3 — Persona Física Entities (`rename-participants-entities-to-english` sub-change, part A)

**Goal**: rename all PF tables and columns in `libs/participants/src/lib/fisica/`.

- [ ] 3.1 Update `ParticipantPFBasicaEntity` → `PFBasicEntity`: rename class, set `@Entity({ name: 'pf_basic' })`, add explicit `name:` to every `@Column` using new English snake_case names per specs.md §3.2. (~60 min)
- [ ] 3.2 Update `ParticipantPFIdentificacionEntity` → `PFIdentificationEntity`: rename class, set `@Entity({ name: 'pf_identification' })`, update columns. (~30 min)
- [ ] 3.3 Update `ParticipantPFDomicilioEntity` → `PFAddressEntity`: rename class, set `@Entity({ name: 'pf_address' })`, update address columns per specs.md §3.5. (~30 min)
- [ ] 3.4 Update remaining PF entities (DocINM, DomicilioNacional, DatosIdentificacion, DocINMPantalla, BeneficiarioControlador, Representante, RepresentanteDomicilio, RepresentanteIdentificacion) — one sub-task per entity, same pattern. (~60 min)
- [ ] 3.5 Update `libs/participants/src/lib/fisica/participants.service.ts` to reference new entity class names and any renamed TypeScript properties. (~45 min)
- [ ] 3.6 Update all consumers of old PF entity class names in `apps/auth-users` controllers/adapters. (~30 min)
- [ ] 3.7 Write SQL migration `RENAME TABLE participantsFisicaBasico TO pf_basic` (and all other PF table renames). With down migration. (~45 min)
- [ ] 3.8 Write SQL migration for PF column renames (`ALTER TABLE pf_basic CHANGE COLUMN nombre first_name VARCHAR(120) NOT NULL` etc.) for all PF tables. With down migration. (~60 min)
- [ ] 3.9 Run migrations in dev. Smoke test: `GET /participants/fisica/:id` returns 200 with data. (~20 min)

---

## Phase 4 — Persona Moral Entities (`rename-participants-entities-to-english` sub-change, part B)

**Goal**: rename all PM tables and columns in `libs/participants/src/lib/moral/`.

- [ ] 4.1 Update `ParticipantePMBasicoEntity` → `PMBasicEntity`: rename class, set `@Entity({ name: 'pm_basic' })`, add `@Column({ name: '...' })` for all columns per specs.md §3.3. Rename `TipoPersonaMoral` → `LegalEntityType` within this file. (~60 min)
- [ ] 4.2 Update `ParticipantePMDomicilioEntity` → `PMAddressEntity`, `ParticipantePMRepresentanteEntity` → `PMLegalRepresentativeEntity`, and remaining PM entities. (~45 min)
- [ ] 4.3 Update `ParticipantePMPEPEntity` → `PMPEPEntity`: rename class, update PEP column names per specs.md §3.7. (~30 min)
- [ ] 4.4 Update `libs/participants/src/lib/moral/participants.service.ts` and consumers. (~30 min)
- [ ] 4.5 Write SQL migration: `RENAME TABLE moralAmbosBasico TO pm_basic` (and all other PM table renames). With down migration. (~30 min)
- [ ] 4.6 Write SQL migration: PM column renames for `pm_basic` and all other PM tables. With down migration. (~60 min)
- [ ] 4.7 Run migrations in dev. Smoke test: `GET /participants/moral/:id` returns 200 with data. (~20 min)

---

## Phase 5 — Fideicomiso Entities (`rename-participants-entities-to-english` sub-change, part C)

**Goal**: rename all fideicomiso tables and columns in `libs/participants/src/lib/fideicomiso/`.

- [ ] 5.1 Update `FideicomisoEntity` → `TrustBasicEntity`: `@Entity({ name: 'trust_basic' })`, column renames per specs.md §3.4. (~30 min)
- [ ] 5.2 Update `ApoderadoDelegadoEntity` → `TrustAttorneyEntity`, `IdentificacionEntity` → `TrustIdentificationEntity`, `BenControladorRelacionEntity` → `TrustBeneficialOwnerEntity`, `PersonaPoliticamenteExpuestaEntity` → `TrustPEPEntity`. (~45 min)
- [ ] 5.3 Update `libs/participants/src/lib/fideicomiso/participants.service.ts` and consumers. (~20 min)
- [ ] 5.4 Write SQL migration: fideicomiso table renames. With down. (~20 min)
- [ ] 5.5 Write SQL migration: fideicomiso column renames. With down. (~30 min)
- [ ] 5.6 Run migrations in dev. Smoke test: fideicomiso endpoints return 200. (~15 min)

---

## Phase 6 — Anexo-7 Entities (`rename-participants-entities-to-english` sub-change, part D)

**Goal**: rename all anexo-7 tables and columns in `libs/participants/src/lib/anexo-7/`.

- [ ] 6.1 Update `Anexo7OBisBasicoEntity` → `Annex7BasicEntity`: `@Entity('annex_7_basic')`, column renames per specs.md §3.8. Note: `tipoAnexo` currently uses a MySQL ENUM type (`['anexo-7', 'anexo-7-bis']`) — migrate to `varchar` or change values to English equivalents (`'annex-7'`, `'annex-7-bis'`). Document decision before applying. (~60 min)
- [ ] 6.2 Update `Anexo7DomicilioEntity` → `Annex7AddressEntity`, `Anexo7RepresentanteLegalEntity` → `Annex7LegalRepresentativeEntity`, `Anexo7IdentificacionEntity` → `Annex7IdentificationEntity`, `Anexo7FuncionarioEntity` → `Annex7OfficerEntity`. (~45 min)
- [ ] 6.3 Update service and consumers. (~20 min)
- [ ] 6.4 Write SQL migration: anexo-7 table renames. With down. (~20 min)
- [ ] 6.5 Write SQL migration: anexo-7 column renames. With down. (~30 min)
- [ ] 6.6 Run migrations in dev. Smoke test: anexo-7 endpoints return 200. (~15 min)

---

## Phase 7 — Users Table and Auth Types (`rename-users-and-types-to-english` sub-change)

**Goal**: rename columns in the `users` table and rename TypeScript types in `libs/auth-profiles` and `packages/shared-types`.

- [ ] 7.1 Update `UserEntity` in `packages/domain-auth-users/src/entities/user.entity.ts`:
  - `nombre` → `firstName`, `@Column({ name: 'first_name' })`
  - `apellidoPaterno` → `paternalSurname`, `@Column({ name: 'paternal_surname' })`
  - `apellidoMaterno` → `maternalSurname`, `@Column({ name: 'maternal_surname' })`
  - `telefono` → `phone`, `@Column({ name: 'phone' })`
  - `activo` → `active`, `@Column({ name: 'active' })`
  (~45 min)
- [ ] 7.2 Update all consumers of `UserEntity` properties in `apps/auth-users` (adapters, services, DTOs). (~60 min)
- [ ] 7.3 Rename TypeScript interfaces in `libs/auth-profiles/src/lib/auth-profiles.types.ts` per specs.md §4.1. Add re-exports with old names for the same release cycle. (~45 min)
- [ ] 7.4 Rename `PerfilBasico`, `PFParticipante`, `PMParticipante` in `packages/shared-types` per specs.md §4.2. Add re-exports. (~30 min)
- [ ] 7.5 Update field names inside renamed interfaces (`nombreCompleto` → `fullName`, `domicilio` → `address`, `rol` → `role` in `ProfileRegistration`). (~30 min)
- [ ] 7.6 Update all consumers of renamed interfaces in `apps/auth-users`. (~45 min)
- [ ] 7.7 Write SQL migration: `users` column renames. With down migration. (~30 min)
- [ ] 7.8 Run migrations in dev. Smoke test: `POST /auth/login` returns 201 + JWT, `GET /users/:id` returns correct user data. (~20 min)

---

## Phase 8 — Beneficiario Controlador Entities

- [ ] 8.1 Audit `libs/participants/src/lib/beneficiario-controlador/ben-controlador.entity.ts` — list entity class names, table names, column names (from Phase 1 audit task 1.6). (~20 min)
- [ ] 8.2 Update entity class names, `@Entity` name, and `@Column` names to English equivalents. (~30 min)
- [ ] 8.3 Update service and consumers. (~20 min)
- [ ] 8.4 Write SQL migration: beneficiario controlador table and column renames. With down. (~30 min)
- [ ] 8.5 Run migrations in dev. Smoke test. (~15 min)

---

## Phase 9 — Cleanup (post deprecation window)

**Prerequisite**: front-end team has confirmed full migration and zero `[DEPRECATED]` log entries in production for ≥ 48 hours.

- [ ] 9.1 Remove `normalizeRole()` and all compat functions from adapter files. (~30 min)
- [ ] 9.2 Remove re-export aliases for old enum names (`TipoPersonaParticipante`, etc.) from `packages/shared-types`. (~20 min)
- [ ] 9.3 Remove re-export aliases for old interface names from `libs/auth-profiles`. (~20 min)
- [ ] 9.4 Run full build: `nx run-many --target=build --all`. Fix any TS errors from removed re-exports. (~30 min)
- [ ] 9.5 Update Swagger examples that still reference old values (if any remain). (~20 min)
- [ ] 9.6 Archive this umbrella change via `sdd-archive rename-to-english`. (~15 min)

---

## Phase 10 — Testing and Validation (per sub-change)

Run after each sub-change deployment to staging/production:

- [ ] 10.1 `POST /auth/login` with valid `NOTARY` role user → HTTP 201 + JWT. (~10 min)
- [ ] 10.2 `GET /users` → all users returned with new English role values. (~10 min)
- [ ] 10.3 `GET /participants/fisica/:id` → HTTP 200, data correct. (~10 min)
- [ ] 10.4 `GET /participants/moral/:id` → HTTP 200, data correct. (~10 min)
- [ ] 10.5 `nx run auth-users:build`, `nx run cross:build`, `nx run catalogs:build` all succeed. (~15 min)
- [ ] 10.6 (During deprecation window) Send old enum value `NOTARIO` to any endpoint → response uses `NOTARY`, `[DEPRECATED]` warn in logs. (~10 min)
- [ ] 10.7 (Post cleanup) Send old enum value `NOTARIO` → HTTP 400. (~10 min)
- [ ] 10.8 Run `SELECT COUNT(*) FROM users WHERE role NOT IN ('SUPERADMIN','NOTARY','REAL_ESTATE','AUXILIARY','INTERNAL_USER','EXTERNAL_USER')` → 0 rows. (~5 min)
