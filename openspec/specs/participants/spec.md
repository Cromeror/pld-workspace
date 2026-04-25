# Participants Specification

## Purpose

Governs entity classes, database schema, enum values, module registration, and code cleanliness for the `libs/participants` domain in `pld-api`. All identifiers (class names, TS properties, DB table names, column names, enum values) MUST be English after this change.

---

## Requirements

### Requirement: PF entity class names (fisica)

`pld-api/libs/participants/src/lib/fisica/participants.entity.ts` MUST export exactly the following 11 entity classes. MUST NOT export the old Spanish-named classes.

| New (English) | Old (Spanish) |
|---|---|
| `PFBasicEntity` | `ParticipantPFBasico` |
| `PFIdentificationEntity` | `ParticipantPFIdentificacion` |
| `PFAddressEntity` | `ParticipantPFDomicilio` |
| `PFImmigrationDocumentEntity` | `ParticipantPFDocumentoINM` |
| `PFNationalAddressEntity` | `ParticipantPFDomicilioNacional` |
| `PFIdentificationDataEntity` | `ParticipantPFDatosIdentificacion` |
| `PFImmigrationDocumentDisplayEntity` | `ParticipantPFDocumentoINMPantalla` |
| `PFBeneficialOwnerEntity` | `ParticipantPFBeneficiarioControlador` |
| `PFLegalRepresentativeEntity` | `ParticipantPFRepresentanteLegal` |
| `PFRepresentativeAddressEntity` | `ParticipantPFDomicilioRepresentante` |
| `PFRepresentativeIdentificationEntity` | `ParticipantPFIdentificacionRepresentante` |

#### INV-1: PF entity classes export English names

- GIVEN `fisica/participants.entity.ts` has been updated
- WHEN `tsc --noEmit` runs on `libs/participants`
- THEN all 11 English class names SHALL be importable
- AND the old Spanish class names SHALL NOT resolve (compile error if referenced)

---

### Requirement: PM entity class names (moral)

`pld-api/libs/participants/src/lib/moral/participants.entity.ts` MUST export exactly the following 8 entity classes. MUST NOT export old Spanish names.

| New (English) | Old (Spanish) |
|---|---|
| `PMBasicEntity` | `ParticipantePMBasico` |
| `PMAddressEntity` | `ParticipantePMDomicilio` |
| `PMLegalRepresentativeEntity` | `ParticipantePMRepresentanteLegal` |
| `PMRepresentativeIdentificationEntity` | `ParticipantePMIdentificacionRepresentante` |
| `PMPEPEntity` | `PersonaPoliticamenteExpuestaEntity` |
| `PMImmigrationDocumentEntity` | `ParticipantePMDocumentoINM` |
| `PMNationalAddressEntity` | `ParticipantePMDomicilioNacional` |
| `PMBeneficialOwnerEntity` | `ParticipantePMBeneficiarioControlador` |

#### INV-2: PM entity classes export English names

- GIVEN `moral/participants.entity.ts` has been updated
- WHEN `tsc --noEmit` runs on `libs/participants`
- THEN all 8 English class names SHALL be importable
- AND old Spanish class names SHALL NOT resolve

---

### Requirement: Trust entity class names (fideicomiso)

`pld-api/libs/participants/src/lib/fideicomiso/participants.entity.ts` MUST export exactly the following 5 entity classes.

| New (English) | Old (Spanish) |
|---|---|
| `TrustBasicEntity` | `FideicomisoBasico` |
| `TrustAttorneyEntity` | `ApoderadoDelegadoEntity` |
| `TrustIdentificationEntity` | `IdentificacionEntity` |
| `TrustBeneficialOwnerEntity` | `FideicomisobeneficiarioControlador` |
| `TrustPEPEntity` | `FideicomisoPersonaPoliticamenteExpuesta` |

#### INV-3: Trust entity classes export English names

- GIVEN `fideicomiso/participants.entity.ts` has been updated
- WHEN `tsc --noEmit` runs on `libs/participants`
- THEN all 5 English class names SHALL be importable
- AND old Spanish class names SHALL NOT resolve

---

### Requirement: Annex-7 entity class names

`pld-api/libs/participants/src/lib/anexo-7/anexo-7.entity.ts` MUST export exactly the following 5 entity classes.

| New (English) | Old (Spanish) |
|---|---|
| `Annex7BasicEntity` | `Anexo7OBis` |
| `Annex7AddressEntity` | `Anexo7Domicilio` |
| `Annex7LegalRepresentativeEntity` | `Anexo7Representante` |
| `Annex7IdentificationEntity` | `Anexo7Identificacion` |
| `Annex7OfficerEntity` | `Anexo7Funcionario` |

#### INV-4: Annex-7 entity classes export English names

- GIVEN `anexo-7/anexo-7.entity.ts` has been updated
- WHEN `tsc --noEmit` runs on `libs/participants`
- THEN all 5 English class names SHALL be importable
- AND old Spanish class names SHALL NOT resolve

---

### Requirement: Cross beneficial owner entity

The beneficial-owner entity in `pld-api/libs/participants/src/lib/beneficiario-controlador/` MUST export `CrossBeneficialOwnerEntity`. MUST NOT export `CrossBenControladorEntity`.

#### INV-5: CrossBeneficialOwnerEntity replaces CrossBenControladorEntity

- GIVEN `beneficiario-controlador/*.entity.ts` has been updated
- WHEN `tsc --noEmit` runs on `libs/participants` and `apps/cross`
- THEN `CrossBeneficialOwnerEntity` SHALL be importable
- AND `CrossBenControladorEntity` SHALL NOT resolve

---

### Requirement: English DB table names after migration

The MySQL database MUST contain the new English table names after the up migration runs. Old Spanish table names MUST NOT exist.

| Domain | English table | Replaces Spanish table |
|---|---|---|
| fisica | `pf_basic` | `participantsFisicaBasico` |
| fisica | `pf_identification` | `participantsFisicaIdentificacion` |
| fisica | `pf_address` | `participantsFisicaDomicilio` |
| fisica | `pf_immigration_document` | `participantsFisicaDocumentoINM` |
| fisica | `pf_national_address` | `participantsFisicaDomicilioNacional` |
| fisica | `pf_identification_data` | `participantsFisicaDatosIdentificacion` |
| fisica | `pf_immigration_document_display` | `participantsFisicaDocumentoINMPantalla` |
| fisica | `pf_beneficial_owner` | `participantsFisicaBeneficiarioControlador` |
| fisica | `pf_legal_representative` | `participantsFisicaRepresentanteLegal` |
| fisica | `pf_representative_address` | `participantsFisicaDomicilioRepresentante` |
| fisica | `pf_representative_identification` | `participantsFisicaIdentificacionRepresentante` |
| moral | `pm_basic` | `participantsMoralBasico` |
| moral | `pm_address` | `participantsMoralDomicilio` |
| moral | `pm_legal_representative` | `participantsMoralRepresentanteLegal` |
| moral | `pm_representative_identification` | `participantsMoralIdentificacionRepresentante` |
| moral | `pm_pep` | `participantsMoralPEP` |
| moral | `pm_immigration_document` | `participantsMoralDocumentoINM` |
| moral | `pm_national_address` | `participantsMoralDomicilioNacional` |
| moral | `pm_beneficial_owner` | `participantsMoralBeneficiarioControlador` |
| fideicomiso | `trust_basic` | `fideicomisoBasico` |
| fideicomiso | `trust_attorney` | `fideicomisoApoderado` |
| fideicomiso | `trust_identification` | `fideicomisoIdentificacion` |
| fideicomiso | `trust_beneficial_owner` | `fideicomisobeneficiarioControlador` |
| fideicomiso | `trust_pep` | `fideicomisoPersonaPoliticamenteExpuesta` |
| anexo-7 | `annex7_basic` | `anexo7OBis` |
| anexo-7 | `annex7_address` | `anexo7Domicilio` |
| anexo-7 | `annex7_legal_representative` | `anexo7Representante` |
| anexo-7 | `annex7_identification` | `anexo7Identificacion` |
| anexo-7 | `annex7_officer` | `anexo7Funcionario` |
| cross | `cross_beneficial_owner` | `crossBenControlador` |

#### INV-6: SHOW TABLES lists English names; Spanish names absent

- GIVEN the up migration has executed successfully
- WHEN `SHOW TABLES` is run against the database
- THEN all 30 English table names SHALL appear
- AND all 30 old Spanish table names SHALL NOT appear

#### INV-6b: Row counts preserved after table renames

- GIVEN each participant table contains N rows before migration
- WHEN the up migration runs
- THEN each renamed table SHALL contain the same N rows
- AND no row data SHALL be altered by the rename operation alone

---

### Requirement: English column names on pf_basic

`pf_basic` MUST have columns with English names. The `paisNationality` bug MUST be fixed: column renamed to `nationality_country`. MUST NOT retain any Spanish column names.

Required columns: `id`, `first_name`, `paternal_surname`, `maternal_surname`, `birth_date`, `birth_country`, `nationality_country`, `birth_place`, `occupation`, `email`, `passport_number`, `passport_issuing_country`, `residence_country`, `nationality`, `curp`, `rfc`, `created_at`, `updated_at`.

#### INV-7: pf_basic has English column names

- GIVEN the up migration has executed
- WHEN `DESCRIBE pf_basic` is run
- THEN `first_name`, `paternal_surname`, `maternal_surname`, `nationality_country` SHALL exist
- AND no column named `pais_nationality` or using a Spanish identifier SHALL exist

---

### Requirement: Zero Spanish column names on all renamed tables

All 30 renamed tables MUST have zero Spanish column names after migration. The principle applies uniformly to `pm_basic`, `trust_basic`, `annex7_basic`, and all other renamed tables. Concrete column lists are in the design artifact.

#### INV-8: No Spanish column names on any renamed table

- GIVEN the up migration has executed on all 30 tables
- WHEN any renamed table is inspected with `DESCRIBE`
- THEN zero columns with Spanish-derived names SHALL exist on that table

---

### Requirement: LegalEntityType enum replaces TipoPersonaMoral

`moral/participants.entity.ts` MUST export enum `LegalEntityType` with values `MEXICAN` and `FOREIGN` (uppercase). MUST NOT export `TipoPersonaMoral`.

#### INV-9: LegalEntityType enum exported with uppercase values

- GIVEN `moral/participants.entity.ts` has been updated
- WHEN the module is compiled
- THEN `LegalEntityType.MEXICAN` and `LegalEntityType.FOREIGN` SHALL be accessible
- AND `TipoPersonaMoral` SHALL NOT be exported from that file

---

### Requirement: Enum data migration on pm_basic

Existing rows in `pm_basic` (after rename from `participantsMoralBasico`) MUST have column `legal_entity_type` contain only `'MEXICAN'`, `'FOREIGN'`, or `NULL`. No row SHALL contain lowercase `'mexicana'` or `'extranjera'` after migration.

#### INV-10: pm_basic.legal_entity_type values migrated to uppercase

- GIVEN `participantsMoralBasico` rows contain `tipo_persona_moral` values `'mexicana'` or `'extranjera'`
- WHEN the up migration runs
- THEN `SELECT DISTINCT legal_entity_type FROM pm_basic` SHALL return only `MEXICAN`, `FOREIGN`, and/or `NULL`
- AND no row SHALL contain the lowercase Spanish values

#### INV-10b: Down migration restores lowercase Spanish enum values

- GIVEN the up migration has executed and `pm_basic` has uppercase enum values
- WHEN the down migration runs
- THEN `SELECT DISTINCT tipo_persona_moral FROM participantsMoralBasico` SHALL return only `'mexicana'`, `'extranjera'`, and/or `NULL`

---

### Requirement: Module registration includes all 30 entities

`pld-api/libs/participants/src/lib/participants.module.ts` MUST register all 30 renamed entity classes via `TypeOrmModule.forFeature([...])`. This includes `PFImmigrationDocumentDisplayEntity`, which was previously unregistered.

#### INV-11: All 30 entities registered in participants.module.ts

- GIVEN `participants.module.ts` has been updated
- WHEN the NestJS module bootstraps
- THEN all 30 entity repositories SHALL be injectable without "EntityMetadataNotFoundError"
- AND `PFImmigrationDocumentDisplayEntity` SHALL be among the registered entities

---

### Requirement: nationalityCountry property replaces paisNationality

`PFBasicEntity` MUST declare property `nationalityCountry` mapped to DB column `nationality_country`. Property `paisNationality` MUST NOT exist anywhere in source.

#### INV-12: paisNationality is fully removed

- GIVEN all renames are applied
- WHEN `grep -rn "paisNationality" pld-api/{apps,libs,packages}/*/src` is run
- THEN the command SHALL return zero matches

---

### Requirement: No Spanish TypeScript identifiers in source (cleanliness gate)

After all phases complete, the following grep patterns MUST return zero matches in `pld-api/{apps,libs,packages}/*/src`.

| Pattern | Meaning |
|---|---|
| `ParticipantPF\|ParticipantePM` | Old fisica/moral class prefixes |
| `FideicomisoEntity\|ApoderadoDelegadoEntity` | Old fideicomiso classes |
| `IdentificacionEntity\b` | Old identification class |
| `BenControladorRelacionEntity` | Old cross entity variant |
| `PersonaPoliticamenteExpuestaEntity` | Old PEP entity |
| `Anexo7OBis\|Anexo7Domicilio\|Anexo7Representante\|Anexo7Identificacion\|Anexo7Funcionario` | Old annex-7 classes |
| `CrossBenControlador` | Old cross beneficial owner prefix |

#### INV-13: Grep for old Spanish class names returns zero matches

- GIVEN all renames are applied
- WHEN the grep command above is run
- THEN it SHALL return zero matches

#### INV-14: TipoPersonaMoral and paisNationality absent from source

- GIVEN all renames are applied
- WHEN `grep -rn "TipoPersonaMoral\|paisNationality" pld-api/{apps,libs,packages}/*/src` is run
- THEN the command SHALL return zero matches

#### INV-15: Lowercase enum literals absent from source

- GIVEN all renames are applied
- WHEN `grep -rn "'mexicana'\|'extranjera'" pld-api/{apps,libs,packages}/*/src` is run
- THEN the command SHALL return zero matches (migration `.sql` history files excluded)

---

### Requirement: Build succeeds after all renames

`nx run auth-users:build`, `nx run cross:build`, and `tsc --noEmit` on `libs/participants` MUST all exit 0 after the change is applied.

#### INV-16: Builds exit 0

- GIVEN all entity renames, module updates, service updates, and migration are applied
- WHEN `nx run auth-users:build` and `nx run cross:build` are executed
- THEN both SHALL exit 0
- AND `tsc --noEmit` on `libs/participants` SHALL exit 0

---

### Requirement: Down migration reverses all changes

A `.down.sql` migration MUST exist that fully reverses the up migration: table renames, column renames, enum data migration, and FK check toggling, applied in reverse order.

#### INV-17: Down migration restores pre-change DB state

- GIVEN the up migration has been applied
- WHEN the down migration runs
- THEN all 30 English table names SHALL be renamed back to Spanish
- AND all renamed columns SHALL be restored to their original Spanish names
- AND `SET FOREIGN_KEY_CHECKS` SHALL bracket the operation to avoid FK violations

---

### Requirement: Login smoke test unaffected

After the migration is applied and `auth-users` restarts, the login endpoint MUST respond normally. Participant tables are not in the login path.

#### INV-18: POST /auth/login returns 201 + JWT after migration

- GIVEN the up migration has executed and `auth-users` service has restarted
- WHEN a valid credentials request is sent to `POST /auth/login`
- THEN the server SHALL respond HTTP 201 with a JWT token
