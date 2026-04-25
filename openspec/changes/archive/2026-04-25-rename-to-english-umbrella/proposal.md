# Proposal: rename-to-english

## Status
draft

## Why

The `pld-api` monorepo was built with Spanish identifiers throughout — table names, column names, enum values, entity class names, TypeScript types, and file names. This creates several compounding problems:

1. **Inconsistency with new code**: the `add-registro-sujetos-obligados` change already uses English exclusively (`physical_person_profile`, `reporting_entity_address`, `compliance_responsible`, etc.). Having two conventions in the same repo raises the maintenance burden on every future change.
2. **Onboarding friction**: engineers not fluent in Spanish cannot read entity schemas, SQL queries, or error messages without translation.
3. **Tool and AI tooling noise**: linters, AI code assistants, and search tools optimise for English identifiers. Mixed-language code produces false positives and unreliable suggestions.
4. **API contract clarity**: enum values that travel in JSON (`role: "NOTARIO"`, `tipoPersona: "PERSONA_FISICA"`) are part of the public contract with the front end and any future third-party integration. English values are more self-explanatory and align with REST conventions.

## What Changes (grouped by phase)

### Phase 1 — Enums (packages/shared-types)

#### Enum name renames
| Old name | New name |
|---|---|
| `TipoPersonaParticipante` | `ProfileType` |
| `ActividadVulnerable` | `VulnerableActivity` |
| `TipoMoneda` | `CurrencyType` |
| `FormaPago` | `PaymentMethod` |
| `TipoReporte` | `ReportType` |
| `ModalidadAtencion` | `AttendanceMode` |
| `Nacionalidad` | `Nationality` |
| `TipoPersonaMoral` (local to moral entity) | `LegalEntityType` |

#### Enum value renames (UserRole — name stays, values change)
| Old value | New value |
|---|---|
| `NOTARIO` | `NOTARY` |
| `INMOBILIARIA` | `REAL_ESTATE` |
| `AUXILIAR` | `AUXILIARY` |
| `USUARIO_INTERNO` | `INTERNAL_USER` |
| `USUARIO_EXTERNO` | `EXTERNAL_USER` |
| `SUPERADMIN` | `SUPERADMIN` (no change) |

#### Enum value renames (ProfileType, formerly TipoPersonaParticipante)
| Old value | New value |
|---|---|
| `PERSONA_FISICA` | `INDIVIDUAL` |
| `PERSONA_MORAL` | `LEGAL_ENTITY` |
| `FIDEICOMISO` | `TRUST` |

#### Enum value renames (VulnerableActivity, formerly ActividadVulnerable)
| Old value | New value |
|---|---|
| `TRANSMISION_DERECHOS_REALES_INMUEBLES` | `REAL_ESTATE_RIGHTS_TRANSFER` |
| `PODERES_IRREVOCABLES` | `IRREVOCABLE_POWERS` |
| `CONSTITUCION_SOCIEDADES` | `COMPANY_INCORPORATION` |
| `FUSION` | `MERGER` |
| `ESCISION` | `SPIN_OFF` |
| `AUMENTO_CAPITAL` | `CAPITAL_INCREASE` |
| `DISMINUCION_CAPITAL` | `CAPITAL_DECREASE` |
| `TRANSMISION_ACCIONES_PARTES` | `SHARES_TRANSFER` |
| `FIDEICOMISOS` | `TRUSTS` |
| `MUTUOS_PRESTAMOS_CREDITOS` | `LOANS_AND_CREDITS` |

#### Enum value renames (Nationality, PaymentMethod, CurrencyType, AttendanceMode)
| Old value | New value |
|---|---|
| `MEXICANA` | `MEXICAN` |
| `EXTRANJERA` | `FOREIGN` |
| `EFECTIVO` | `CASH` |
| `TRANSFERENCIA` | `WIRE_TRANSFER` |
| `CHEQUE` | `CHECK` |
| `TARJETA` | `CARD` |
| `OTRO` | `OTHER` |
| `POR_CUENTA` | `OWN_ACCOUNT` |
| `CON_REPRESENTANTE` | `WITH_REPRESENTATIVE` |
| `POR_TIPO_CLIENTE` | `BY_CLIENT_TYPE` |
| `AVISOS_POR_PERIODO` | `NOTICES_BY_PERIOD` |
| `OPERACIONES_POR_ACTO` | `OPERATIONS_BY_ACT` |
| `POR_USO_EFECTIVO` | `BY_CASH_USE` |
| `mexicana` (TipoPersonaMoral) | `MEXICAN` |
| `extranjera` (TipoPersonaMoral) | `FOREIGN` |

### Phase 2 — Entities and tables (libs/participants — persona física)

#### Table renames
| Old table | New table | Entity class |
|---|---|---|
| `participantsFisicaBasico` | `pf_basic` | `ParticipantPFBasicaEntity` → `PFBasicEntity` |
| `participantsFisicaIdentificacion` | `pf_identification` | `ParticipantPFIdentificacionEntity` → `PFIdentificationEntity` |
| `participantsFisicaDomicilio` | `pf_address` | `ParticipantPFDomicilioEntity` → `PFAddressEntity` |
| `participantsFisicaDocumentoINM` | `pf_immigration_document` | `ParticipantPFDocumentoINMEntity` → `PFImmigrationDocumentEntity` |
| `participantsFisicaDomicilioNacional` | `pf_national_address` | `ParticipantPFDomicilioNacionalEntity` → `PFNationalAddressEntity` |
| `participantsFisicaDatosIdentificacion` | `pf_identification_data` | `ParticipantPFDatosIdentificacionEntity` → `PFIdentificationDataEntity` |
| `participantsFisicaDocumentoINMPantalla` | `pf_immigration_document_display` | `ParticipantPFDocumentoINMPantallaEntity` → `PFImmigrationDocumentDisplayEntity` |
| `participantsFisicaBeneficiarioControlador` | `pf_beneficial_owner` | `ParticipantPFBeneficiarioControladorEntity` → `PFBeneficialOwnerEntity` |
| `participantsFisicaRepresentante` | `pf_legal_representative` | `ParticipantRepresentanteEntity` → `PFLegalRepresentativeEntity` |
| `participantsFisicaRepresentanteDomicilio` | `pf_representative_address` | `ParticipantRepresentanteDomicilioEntity` → `PFRepresentativeAddressEntity` |
| `participantsFisicaRepresentanteIdentificacion` | `pf_representative_identification` | `ParticipantRepresentanteIdentificacionEntity` → `PFRepresentativeIdentificationEntity` |

#### Column renames (pf_basic)
| Old column | New column |
|---|---|
| `nombre` | `first_name` |
| `apellidoPaterno` → DB col (inferred) | `paternal_surname` |
| `apellidoMaterno` → DB col (inferred) | `maternal_surname` |
| `fechaNacimiento` → DB col (inferred) | `birth_date` |
| `paisNacimiento` → DB col (inferred) | `birth_country` |
| `paisNacionalidad` → DB col (inferred) | `nationality_country` |
| `lugarNacimiento` → DB col (inferred) | `birth_place` |
| `ocupacionProfesionGiro` → DB col (inferred) | `occupation` |
| `correo` → DB col (inferred) | `email` |
| `numeroPasaporte` → DB col (inferred) | `passport_number` |
| `paisEmisorPasaporte` → DB col (inferred) | `passport_issuing_country` |
| `paisResidencia` → DB col (inferred) | `residence_country` |

> Note: columns without explicit `name:` in the entity use TypeORM's default camelCase→snake_case conversion. Audit each entity to confirm actual DB column names before writing migrations.

### Phase 3 — Entities and tables (libs/participants — persona moral)

| Old table | New table | Entity class |
|---|---|---|
| `moralAmbosBasico` | `pm_basic` | `ParticipantePMBasicoEntity` → `PMBasicEntity` |
| `moralAmbosDomicilio` | `pm_address` | `ParticipantePMDomicilioEntity` → `PMAddressEntity` |
| `moralAmbosRepresentante` | `pm_legal_representative` | `ParticipantePMRepresentanteEntity` → `PMLegalRepresentativeEntity` |
| `moralAmbosIdentificacionRepresentante` | `pm_representative_identification` | `ParticipantePMIdentificacionRepresentanteEntity` → `PMRepresentativeIdentificationEntity` |
| `moralMexicanaPEP` | `pm_pep` | `ParticipantePMPEPEntity` → `PMPEPEntity` |
| `moralExtranjeraDocumentoMigratorio` | `pm_immigration_document` | `ParticipantePMDocumentoMigratorioEntity` → `PMImmigrationDocumentEntity` |
| `moralExtranjeraDomicilioNacional` | `pm_national_address` | `ParticipantePMDomicilioTerritorialNacionalEntity` → `PMNationalAddressEntity` |
| `moralAmbosBenControladorRelacion` | `pm_beneficial_owner` | `ParticipantePMBeneficiarioControladorEntity` → `PMBeneficialOwnerEntity` |

#### Column renames (pm_basic)
| Old property / inferred column | New column |
|---|---|
| `tipoPersonaMoral` | `legal_entity_type` |
| `fechaConstitucion` | `incorporation_date` |
| `paisNacionalidad` | `nationality_country` |
| `objetoSocialGiro` | `business_purpose` |
| `denominacionRazonSocial` | `corporate_name` |

### Phase 4 — Entities and tables (libs/participants — fideicomiso)

| Old table | New table | Entity class |
|---|---|---|
| `fideicomisoBasico` | `trust_basic` | `FideicomisoEntity` → `TrustBasicEntity` |
| `fideicomisoApoderadoDelegado` | `trust_attorney` | `ApoderadoDelegadoEntity` → `TrustAttorneyEntity` |
| `fideicomisoIdentificacion` | `trust_identification` | `IdentificacionEntity` → `TrustIdentificationEntity` |
| `fideicomisoBenControladorRelacion` | `trust_beneficial_owner` | `BenControladorRelacionEntity` → `TrustBeneficialOwnerEntity` |
| `fideicomisoPersonaPoliticamenteExpuesta` | `trust_pep` | `PersonaPoliticamenteExpuestaEntity` → `TrustPEPEntity` |

#### Column renames (trust_basic)
| Old property / inferred column | New column |
|---|---|
| `denominacionRazonSocial` | `corporate_name` |
| `numeroReferencia` | `reference_number` |

### Phase 5 — Entities and tables (libs/participants — anexo-7)

| Old table | New table | Entity class |
|---|---|---|
| `anexo_7_o_bis_basico` | `annex_7_basic` | `Anexo7OBisBasicoEntity` → `Annex7BasicEntity` |
| `anexo_7_domicilio` | `annex_7_address` | `Anexo7DomicilioEntity` → `Annex7AddressEntity` |
| `anexo_7_representante_legal` | `annex_7_legal_representative` | `Anexo7RepresentanteLegalEntity` → `Annex7LegalRepresentativeEntity` |
| `anexo_7_identificacion` | `annex_7_identification` | `Anexo7IdentificacionEntity` → `Annex7IdentificationEntity` |
| `anexo_7_funcionario` | `annex_7_officer` | `Anexo7FuncionarioEntity` → `Annex7OfficerEntity` |

#### Column renames (annex_7_basic — Spanish property names only)
| Old property / inferred column | New column |
|---|---|
| `tipoAnexo` | `annex_type` |
| `nombreDenominacion` | `corporate_name` |
| `fechaConstitucion` | `incorporation_date` |
| `actividadObjetoSocial` | `business_purpose` |
| `nombrePersonaMoralDerechoPublico` | `public_law_entity_name` |
| `fechaCreacionRFC` | `rfc_creation_date` |
| `numeroExterior` | `exterior_number` |
| `numeroInterior` | `interior_number` |
| `codigoPostal` | `postal_code` |
| `pais` | `country` |

### Phase 6 — Entities and tables (packages/domain-auth-users)

#### Column renames (users table)
| Old property / DB column | New DB column |
|---|---|
| `nombre` / `nombre` | `first_name` |
| `apellidoPaterno` / `apellido_paterno` | `paternal_surname` |
| `apellidoMaterno` / `apellido_materno` | `maternal_surname` |
| `telefono` / `telefono` | `phone` |
| `activo` / `activo` | `active` |

### Phase 7 — Types, DTOs, and interfaces (libs/auth-profiles and shared)

| Old identifier | New identifier | File |
|---|---|---|
| `RegistroPerfil` | `ProfileRegistration` | `auth-profiles.types.ts` |
| `RegistroSuperadmin` | `SuperadminRegistration` | same |
| `RegistroAuxiliar` | `AuxiliaryRegistration` | same |
| `RegistroNotarioInmobiliariaPF` | `NotaryRealEstatePFRegistration` | same |
| `RegistroNotarioInmobiliariaPM` | `NotaryRealEstatePMRegistration` | same |
| `PerfilAcciones` | `ProfileActions` | same |
| `PerfilActualizacion` | `ProfileUpdate` | same |
| `PerfilBasico` | `BasicProfile` | `shared-types` |
| `PFParticipante` | `IndividualParticipant` | `shared-types` |
| `PMParticipante` | `LegalEntityParticipant` | `shared-types` |
| `RegistroSuperadmin.nombreCompleto` | `fullName` | same |
| `RegistroSuperadmin.domicilio` | `address` | same |
| `rol` (literal string field) | `role` | `RegistroPerfil` union |
| Hard-coded `'NOTARIO' \| 'INMOBILIARIA'` string literals | `UserRole.NOTARY \| UserRole.REAL_ESTATE` | same |

### Phase 8 — Migrations SQL

- `RENAME TABLE` for all table renames (atomic in MySQL).
- `ALTER TABLE ... CHANGE COLUMN` for column renames.
- `UPDATE ... SET col = 'NEW_VALUE' WHERE col = 'OLD_VALUE'` for enum data migration.
- Down migrations for every up migration (reversible).

## Apps / Libs / Packages Affected

| Artifact | Reason |
|---|---|
| `packages/shared-types` | Enum definitions and type renames |
| `libs/participants` | Entity class names, table names, column names |
| `packages/domain-auth-users` | `UserEntity` column renames, `UserRole` consumers |
| `libs/auth-profiles` | Type and interface renames |
| `apps/auth-users` | Controllers, adapters, services that reference Spanish identifiers |
| `apps/cross` | If it consumes any of the above types/entities |
| `apps/catalogs` | If it re-exports catalog enums |
| All migration files | SQL renames |

## Alternatives Discarded

1. **Leave as-is** — rejected. The inconsistency will worsen as new English-only code grows alongside old Spanish code.
2. **Rename code identifiers only, leave DB columns/tables in Spanish** — rejected. Inconsistency between code and DB is more confusing than either pure convention.
3. **Rename only new touchpoints going forward** — rejected. Existing code paths will never be cleaned up in practice; the rot compounds.
4. **Single giant PR** — see "Postura sobre el tamaño" below.

## Rollback Plan

Each phase has a down migration:
- Table renames: `RENAME TABLE new_name TO old_name`.
- Column renames: `ALTER TABLE ... CHANGE COLUMN new_col old_col <original_type>`.
- Data updates: `UPDATE ... SET col = 'OLD_VALUE' WHERE col = 'NEW_VALUE'`.
- Enum dual support (Phase 1): revert the compat layer removal by re-adding accepted old values.
- TypeScript renames: `git revert <phase commit hash>` — the change is purely additive-then-swap if dual support is used.

**Critical**: migrations must be committed in the correct order (up = rename forward, down = rename back). Each phase's migration file is independent and reversible without affecting other phases.

## Validation Post-Change (per phase)

After each phase:
1. `POST /auth/login` with valid credentials returns HTTP 201 + JWT.
2. Endpoints that exercise the renamed entities return expected data (no 500 errors).
3. `nx run auth-users:build` and `nx run cross:build` succeed.
4. Affected unit/integration tests pass.

## Postura sobre el Tamaño: ¿Un Change o Varios?

**Recomendación: dividir en 3 sub-changes secuenciales.**

Rationale: with ~80+ renames across 8 phases, a single PR would be:
- Impossible to review meaningfully.
- High risk of merge conflicts with parallel work (particularly `add-registro-sujetos-obligados`).
- Hard to roll back selectively.

Proposed split:
1. **`rename-enums-to-english`** — Phase 1 only. Highest impact on front-end contracts; needs its own deprecation window.
2. **`rename-participants-entities-to-english`** — Phases 2–5 (libs/participants). Pure DB + entity rename, no contract change for external API consumers.
3. **`rename-users-and-types-to-english`** — Phases 6–8 (domain-auth-users, auth-profiles, remaining types). Completes the migration.

Each sub-change follows the same SDD lifecycle. This `rename-to-english` change acts as the **umbrella proposal** that documents intent, conventions, and rollback strategy for all three.
