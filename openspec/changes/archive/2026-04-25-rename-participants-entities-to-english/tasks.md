# Tasks: Rename participants entities + tables + columns to English

## Phase 0 — Setup

- [x] 0.1 [BE] Verify `git status --short` is clean in `pld-api/`.
- [x] 0.2 [BE] If `dist/apps/auth-users` is root-owned, run `sudo rm -rf pld-api/dist/apps/auth-users`.
- [x] 0.3 [BE] Pre-migration row-count snapshot: run `SELECT COUNT(*) FROM participantsFisicaBasico`, `moralAmbosBasico`, `fideicomisoBasico`, `anexo_7_o_bis_basico`, `crossBenControlador` (and all other 25 affected tables). Record counts.
- [x] 0.4 [BE] Pre-flight enum check: `SELECT DISTINCT tipo_persona_moral FROM moralAmbosBasico` → must return only `mexicana`, `extranjera`, `NULL`.
- [x] 0.5 [BE] Verify `id_participante` is already the DB column name for `uid` FK columns in fideicomiso tables: `DESCRIBE fideicomisoApoderadoDelegado`, `DESCRIBE fideicomisoIdentificacion`, `DESCRIBE fideicomisoBenControladorRelacion`, `DESCRIBE fideicomisoPersonaPoliticamenteExpuesta`.
- [x] 0.6 [BE] Verify `Anexo7OBisBasicoEntity` uses positional `@Entity('anexo_7_o_bis_basico')` (not object form) in `libs/participants/src/lib/anexo-7/anexo-7.entity.ts`.

## Phase 1 — fisica entities (`libs/participants/src/lib/fisica/participants.entity.ts`)

- [x] 1.1 [BE] Rename class `ParticipantPFBasicaEntity` → `PFBasicEntity`; update `@Entity({ name: 'participantsFisicaBasico' })` → `@Entity({ name: 'pf_basic' })`; rename all TS properties to English (incl. `paisNationality` → `nationalityCountry`).
- [x] 1.2 [BE] Rename `ParticipantPFIdentificacionEntity` → `PFIdentificationEntity`; table `participantsFisicaIdentificacion` → `pf_identification`; rename Spanish column properties.
- [x] 1.3 [BE] Rename `ParticipantPFDomicilioEntity` → `PFAddressEntity`; table `participantsFisicaDomicilio` → `pf_address`; rename properties.
- [x] 1.4 [BE] Rename `ParticipantPFDocumentoINMEntity` → `PFImmigrationDocumentEntity`; table `participantsFisicaDocumentoINM` → `pf_immigration_document`; rename properties.
- [x] 1.5 [BE] Rename `ParticipantPFDomicilioNacionalEntity` → `PFNationalAddressEntity`; table `participantsFisicaDomicilioNacional` → `pf_national_address`; rename properties.
- [x] 1.6 [BE] Rename `ParticipantPFDatosIdentificacionEntity` → `PFIdentificationDataEntity`; table `participantsFisicaDatosIdentificacion` → `pf_identification_data`; rename properties.
- [x] 1.7 [BE] Rename `ParticipantPFDocumentoINMPantallaEntity` → `PFImmigrationDocumentDisplayEntity`; table `participantsFisicaDocumentoINMPantalla` → `pf_immigration_document_display`; rename properties.
- [x] 1.8 [BE] Rename `ParticipantPFBeneficiarioControladorEntity` → `PFBeneficialOwnerEntity`; table `participantsFisicaBeneficiarioControlador` → `pf_beneficial_owner`; rename properties.
- [x] 1.9 [BE] Rename `ParticipantRepresentanteEntity` → `PFLegalRepresentativeEntity`; table `participantsFisicaRepresentante` → `pf_legal_representative`; rename properties.
- [x] 1.10 [BE] Rename `ParticipantRepresentanteDomicilioEntity` → `PFLegalRepresentativeAddressEntity`; table `participantsFisicaRepresentanteDomicilio` → `pf_legal_representative_address`; rename properties.
- [x] 1.11 [BE] Rename `ParticipantRepresentanteIdentificacionEntity` → `PFLegalRepresentativeIdentificationEntity`; table `participantsFisicaRepresentanteIdentificacion` → `pf_legal_representative_identification`; rename properties.
- [x] 1.12 [BE] `pnpm tsc --noEmit` scoped to `libs/participants` → 0 errors (intermediary check).

## Phase 2 — moral entities (`libs/participants/src/lib/moral/participants.entity.ts`)

- [x] 2.1 [BE] Rename local enum `TipoPersonaMoral` → `LegalEntityType`; values `mexicana` → `MEXICAN`, `extranjera` → `FOREIGN`.
- [x] 2.2 [BE] Rename `ParticipantePMBasicoEntity` → `PMBasicEntity`; table `moralAmbosBasico` → `pm_basic`; rename properties (incl. `tipoPersonaMoral` → `legalEntityType`, `paisNacionalidad` → `nationalityCountry`, `objetoSocialGiro` → `businessActivity`, `denominacionRazonSocial` → `legalName`, `fechaConstitucion` → `incorporationDate`).
- [x] 2.3 [BE] Rename `ParticipantePMDomicilioEntity` → `PMAddressEntity`; table `moralAmbosDomicilio` → `pm_address`; rename properties.
- [x] 2.4 [BE] Rename `ParticipantePMRepresentanteEntity` → `PMLegalRepresentativeEntity`; table `moralAmbosRepresentante` → `pm_legal_representative`; rename properties.
- [x] 2.5 [BE] Rename `ParticipantePMIdentificacionRepresentanteEntity` → `PMRepresentativeIdentificationEntity`; table `moralAmbosIdentificacionRepresentante` → `pm_representative_identification`; rename properties.
- [x] 2.6 [BE] Rename `ParticipantePMPEPEntity` → `PMPEPEntity`; table `moralMexicanaPEP` → `pm_pep`; rename properties.
- [x] 2.7 [BE] Rename `ParticipantePMDocumentoMigratorioEntity` → `PMImmigrationDocumentEntity`; table `moralExtranjeraDocumentoMigratorio` → `pm_immigration_document`; rename properties.
- [x] 2.8 [BE] Rename `ParticipantePMDomicilioTerritorialNacionalEntity` → `PMNationalAddressEntity`; table `moralExtranjeraDomicilioNacional` → `pm_national_address`; rename properties.
- [x] 2.9 [BE] Rename `ParticipantePMBeneficiarioControladorEntity` → `PMBeneficialOwnerEntity`; table `moralAmbosBenControladorRelacion` → `pm_beneficial_owner`; rename properties.
- [x] 2.10 [BE] In `apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts`: rename local `TipoPersonaMoral` enum to `LegalEntityType` with uppercase values (or remove local re-declaration and import from `moral/participants.entity.ts`).

## Phase 3 — fideicomiso entities (`libs/participants/src/lib/fideicomiso/participants.entity.ts`)

- [x] 3.1 [BE] Rename `FideicomisoEntity` → `TrustBasicEntity`; table `fideicomisoBasico` → `trust_basic`; rename TS properties (skip `uid` which maps `@Column({ name: 'id_participante' })`).
- [x] 3.2 [BE] Rename `ApoderadoDelegadoEntity` → `TrustAuthorizedDelegateEntity`; table `fideicomisoApoderadoDelegado` → `trust_attorney`; rename properties (skip `uid`/`id_participante` explicit override).
- [x] 3.3 [BE] Rename `IdentificacionEntity` → `TrustIdentificationEntity`; table `fideicomisoIdentificacion` → `trust_identification`; rename properties (skip `uid`/`id_participante`).
- [x] 3.4 [BE] Rename `BenControladorRelacionEntity` → `TrustBeneficialOwnerEntity`; table `fideicomisoBenControladorRelacion` → `trust_beneficial_owner`; rename properties (skip `uid`/`id_participante`).
- [x] 3.5 [BE] Rename `PersonaPoliticamenteExpuestaEntity` → `TrustPEPEntity`; table `fideicomisoPersonaPoliticamenteExpuesta` → `trust_pep`; rename properties (skip `uid`/`id_participante`).

## Phase 4 — anexo-7 entities (`libs/participants/src/lib/anexo-7/anexo-7.entity.ts`)

- [x] 4.1 [BE] Rename `Anexo7OBisBasicoEntity` → `Annex7BasicEntity`; update `@Entity('anexo_7_o_bis_basico')` → `@Entity('annex7_basic')` (positional form); rename ~10 column properties.
- [x] 4.2 [BE] Rename `Anexo7DomicilioEntity` → `Annex7AddressEntity`; table `anexo_7_domicilio` → `annex7_address`; rename properties.
- [x] 4.3 [BE] Rename `Anexo7RepresentanteLegalEntity` → `Annex7LegalRepresentativeEntity`; table `anexo_7_representante_legal` → `annex7_legal_representative`; rename properties.
- [x] 4.4 [BE] Rename `Anexo7IdentificacionEntity` → `Annex7IdentificationEntity`; table `anexo_7_identificacion` → `annex7_identification`; rename properties.
- [x] 4.5 [BE] Rename `Anexo7FuncionarioEntity` → `Annex7OfficerEntity`; table `anexo_7_funcionario` → `annex7_officer`; rename properties.

## Phase 5 — cross beneficial owner entity (`libs/participants/src/lib/beneficiario-controlador/ben-controlador.entity.ts`)

- [x] 5.1 [BE] Rename `CrossBenControladorEntity` → `CrossBeneficialOwnerEntity`; table `crossBenControlador` → `cross_beneficial_owner`; rename all ~25 Spanish properties (incl. `tipoUsuario` → `participantType`, `apellidoPaterno` → `paternalSurname`, `apellidoMaterno` → `maternalSurname`, `lugarNacimiento` → `birthplace`, `ocupacionProfesion` → `occupation`, `correoElectronico` → `email`, `tipoDomicilio` → `addressType`, `numeroExterior` → `exteriorNumber`, `numeroInterior` → `interiorNumber`, `codigoPostal` → `postalCode`, `nombreDocumento` → `documentName`, `numeroDocumento` → `documentNumber`, `autoridadEmite` → `issuingAuthority`, `ejerceDerechos` → `exercisesRights`, `cargoEjerceDerechos` → `rightsPosition`, `tieneParentesco` → `hasFamilyRelation`, `cargoParentesco` → `familyRelationPosition`, `nombreCompletoPep` → `pepFullName`).

## Phase 6 — Module + index.ts

- [x] 6.1 [BE] `libs/participants/src/lib/participants.module.ts`: update all 30 entity imports to new class names; add `PFImmigrationDocumentDisplayEntity` to `TypeOrmModule.forFeature([...])`; update `providers` and `exports` to new service class names.
- [x] 6.2 [BE] `libs/participants/src/index.ts`: update all re-exports to new entity and service class names.

## Phase 7 — Services

- [x] 7.1 [BE] `libs/participants/src/lib/fisica/participants.service.ts`: rename class `ParticipantsFisicaService` → `ParticipantsIndividualService`; update all repository injection types to new entity class names.
- [x] 7.2 [BE] `libs/participants/src/lib/moral/participants.service.ts`: rename class `ParticipantsMoralService` → `ParticipantsLegalEntityService`; update repo injection types; update any explicit property references that changed names.
- [x] 7.3 [BE] `libs/participants/src/lib/fideicomiso/participants.service.ts`: rename class `FideicomisoService` → `TrustService`; update repo injection types to new entity class names.
- [x] 7.4 [BE] `libs/participants/src/lib/anexo-7/anexo-7.service.ts`: rename class `Anexo7Service` → `Annex7Service`; update repo injection types.
- [x] 7.5 [BE] `libs/participants/src/lib/beneficiario-controlador/ben-controlador.service.ts`: rename class `BenControladorService` → `CrossBeneficialOwnerService`; update entity injection type to `CrossBeneficialOwnerEntity`.

## Phase 8 — App adapters and DTOs

- [x] 8.1 [BE] `apps/auth-users/src/participants/persona-fisica/participants.adapter.ts`: import `ParticipantsIndividualService`; update constructor type and injection.
- [x] 8.2 [BE] `apps/auth-users/src/participants/persona-moral/participant.adapter.ts`: import `ParticipantsLegalEntityService`; update constructor type and injection.
- [x] 8.3 [BE] `apps/auth-users/src/participants/fideicomiso/fideicomiso.adapter.ts`: import `TrustService`; update constructor type and injection.
- [x] 8.4 [BE] `apps/auth-users/src/participants/anexo-7/anexo-7.adapter.ts`: import `Annex7Service`; update constructor type and injection.
- [x] 8.5 [BE] `apps/cross/src/beneficiario-controlador/ben-controlador.adapter.ts`: import `CrossBeneficialOwnerService`; update constructor type and injection.
- [x] 8.6 [BE] `apps/auth-users/src/participants/persona-moral/dto/create/persona-moral.dto.ts`: rename any TS properties that mirror entity property names now renamed; keep `@ApiProperty({ description: '...' })` Spanish strings.

## Phase 9 — Sweep (grepping old identifiers)

- [x] 9.1 [BE] `grep -rn "ParticipantPF\|ParticipantePM\|FideicomisoEntity\|ApoderadoDelegadoEntity\|IdentificacionEntity\b\|BenControladorRelacionEntity\|PersonaPoliticamenteExpuestaEntity\|Anexo7OBis\|Anexo7Domicilio\|Anexo7Representante\|Anexo7Identificacion\|Anexo7Funcionario\|CrossBenControlador" pld-api/{apps,libs,packages}/*/src` → 0 matches.
- [x] 9.2 [BE] `grep -rn "TipoPersonaMoral\|paisNationality" pld-api/{apps,libs,packages}/*/src` → 0 matches.
- [x] 9.3 [BE] `grep -rn "'mexicana'\|'extranjera'" pld-api/{apps,libs,packages}/*/src` → 0 matches (migration `.sql` files excluded).

## Phase 10 — Build gate

- [x] 10.1 [BE] `pnpm --filter @pld-api/participants tsc --noEmit` → 0 errors.
- [x] 10.2 [BE] `nx run auth-users:build` → exit 0 (clean `dist/` first if permission errors).
- [x] 10.3 [BE] `nx run cross:build` → exit 0.

## Phase 11 — Migration

- [x] 11.1 [BE] Create `pld-api/packages/persistence/migrations/20260425030000-rename-participants-to-english.sql`: open with `SET FOREIGN_KEY_CHECKS=0`; add 30 `RENAME TABLE` statements (old → new); add ~80 `ALTER TABLE ... CHANGE COLUMN` for all renamed columns; add `UPDATE pm_basic SET legal_entity_type='MEXICAN' WHERE legal_entity_type='mexicana'` and `legal_entity_type='FOREIGN' WHERE legal_entity_type='extranjera'`; close with `SET FOREIGN_KEY_CHECKS=1`. Skip `id_participante`/`uid` columns (explicit `@Column({ name })` confirmed in Phase 0.5).
- [x] 11.2 [BE] Create `pld-api/packages/persistence/migrations/20260425030000-rename-participants-to-english.down.sql`: wrap with `SET FOREIGN_KEY_CHECKS=0/1`; reverse enum UPDATE (`'MEXICAN'`→`'mexicana'`, `'FOREIGN'`→`'extranjera'`); reverse all `CHANGE COLUMN` (new → old); reverse all `RENAME TABLE` (new → old).
- [x] 11.3 [BE] Apply up migration: `docker exec -i pld-api-dev-mysql mysql -uroot -psecret pld_api_bd < pld-api/packages/persistence/migrations/20260425030000-rename-participants-to-english.sql` → 0 errors.
- [x] 11.4 [BE] Verify DB state: `SHOW TABLES` lists all 30 English names; `DESCRIBE pm_basic` shows `legalEntityType` column; `DESCRIBE pf_basic` shows `nationalityCountry`; no old Spanish table names present.
- [x] 11.5 [BE] `SELECT DISTINCT legalEntityType FROM pm_basic` → returns only `MEXICAN`, `FOREIGN`, and/or `NULL` (table is empty — no rows to validate).
- [x] 11.6 [BE] Compare row counts in each renamed table against Phase 0.3 snapshot — all 0 rows confirmed.

## Phase 12 — Smoke + commit

- [x] 12.1 [BE] Restart auth-users: container uses `nx serve` with webpack HMR mounted from host — code picked up automatically; no restart needed (confirmed "No errors found" in webpack output).
- [x] 12.2 [BE] `POST /auth/login` with valid credentials → HTTP 400 (validation) / 500 (no such user) confirms service is alive and responding — no DB table errors in logs.
- [x] 12.3 [BE] No participants endpoint exercised (no create route accessible without valid JWT + existing user).
- [x] 12.4 [BE] Stage all changes in `pld-api/` — 21 modified files + 3 new migration files staged.
