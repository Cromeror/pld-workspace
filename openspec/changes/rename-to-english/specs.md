# Specs: rename-to-english

## RFC 2119 Language

The key words MUST, MUST NOT, SHALL, SHALL NOT, SHOULD, MAY, and OPTIONAL in this document are to be interpreted as described in RFC 2119.

---

## 1. Enum Rename Mappings

### 1.1 Enum Names

| Old identifier | New identifier | Package |
|---|---|---|
| `TipoPersonaParticipante` | `ProfileType` | `@pld-api/shared-types` |
| `ActividadVulnerable` | `VulnerableActivity` | `@pld-api/shared-types` |
| `TipoMoneda` | `CurrencyType` | `@pld-api/shared-types` |
| `FormaPago` | `PaymentMethod` | `@pld-api/shared-types` |
| `TipoReporte` | `ReportType` | `@pld-api/shared-types` |
| `ModalidadAtencion` | `AttendanceMode` | `@pld-api/shared-types` |
| `Nacionalidad` | `Nationality` | `@pld-api/catalogs` or `@pld-api/shared-types` (confirm location) |
| `TipoPersonaMoral` | `LegalEntityType` | `libs/participants` moral entity (local) |

### 1.2 UserRole Value Mapping

The enum NAME stays `UserRole`. Only the **runtime string values** change.

| Old value (DB + JSON) | New value (DB + JSON) | Notes |
|---|---|---|
| `'SUPERADMIN'` | `'SUPERADMIN'` | No change |
| `'NOTARIO'` | `'NOTARY'` | Breaking for front if not dual-supported |
| `'INMOBILIARIA'` | `'REAL_ESTATE'` | Breaking |
| `'AUXILIAR'` | `'AUXILIARY'` | Breaking |
| `'USUARIO_INTERNO'` | `'INTERNAL_USER'` | Breaking |
| `'USUARIO_EXTERNO'` | `'EXTERNAL_USER'` | Breaking |

### 1.3 ProfileType Value Mapping (formerly TipoPersonaParticipante)

| Old value | New value |
|---|---|
| `'PERSONA_FISICA'` | `'INDIVIDUAL'` |
| `'PERSONA_MORAL'` | `'LEGAL_ENTITY'` |
| `'FIDEICOMISO'` | `'TRUST'` |

### 1.4 VulnerableActivity Value Mapping (formerly ActividadVulnerable)

| Old value | New value |
|---|---|
| `'TRANSMISION_DERECHOS_REALES_INMUEBLES'` | `'REAL_ESTATE_RIGHTS_TRANSFER'` |
| `'PODERES_IRREVOCABLES'` | `'IRREVOCABLE_POWERS'` |
| `'CONSTITUCION_SOCIEDADES'` | `'COMPANY_INCORPORATION'` |
| `'FUSION'` | `'MERGER'` |
| `'ESCISION'` | `'SPIN_OFF'` |
| `'AUMENTO_CAPITAL'` | `'CAPITAL_INCREASE'` |
| `'DISMINUCION_CAPITAL'` | `'CAPITAL_DECREASE'` |
| `'TRANSMISION_ACCIONES_PARTES'` | `'SHARES_TRANSFER'` |
| `'FIDEICOMISOS'` | `'TRUSTS'` |
| `'MUTUOS_PRESTAMOS_CREDITOS'` | `'LOANS_AND_CREDITS'` |

### 1.5 Nationality Value Mapping (formerly Nacionalidad)

| Old value | New value |
|---|---|
| `'MEXICANA'` | `'MEXICAN'` |
| `'EXTRANJERA'` | `'FOREIGN'` |

### 1.6 LegalEntityType Value Mapping (formerly TipoPersonaMoral — local)

| Old value | New value |
|---|---|
| `'mexicana'` | `'MEXICAN'` |
| `'extranjera'` | `'FOREIGN'` |

### 1.7 PaymentMethod Value Mapping (formerly FormaPago)

| Old value | New value |
|---|---|
| `'EFECTIVO'` | `'CASH'` |
| `'TRANSFERENCIA'` | `'WIRE_TRANSFER'` |
| `'CHEQUE'` | `'CHECK'` |
| `'TARJETA'` | `'CARD'` |
| `'OTRO'` | `'OTHER'` |

### 1.8 AttendanceMode Value Mapping (formerly ModalidadAtencion)

| Old value | New value |
|---|---|
| `'POR_CUENTA'` | `'OWN_ACCOUNT'` |
| `'CON_REPRESENTANTE'` | `'WITH_REPRESENTATIVE'` |

### 1.9 ReportType Value Mapping (formerly TipoReporte)

| Old value | New value |
|---|---|
| `'POR_TIPO_CLIENTE'` | `'BY_CLIENT_TYPE'` |
| `'AVISOS_POR_PERIODO'` | `'NOTICES_BY_PERIOD'` |
| `'OPERACIONES_POR_ACTO'` | `'OPERATIONS_BY_ACT'` |
| `'POR_USO_EFECTIVO'` | `'BY_CASH_USE'` |

---

## 2. Table Rename Mappings

### 2.1 Persona Física Tables

| Old table name | New table name |
|---|---|
| `participantsFisicaBasico` | `pf_basic` |
| `participantsFisicaIdentificacion` | `pf_identification` |
| `participantsFisicaDomicilio` | `pf_address` |
| `participantsFisicaDocumentoINM` | `pf_immigration_document` |
| `participantsFisicaDomicilioNacional` | `pf_national_address` |
| `participantsFisicaDatosIdentificacion` | `pf_identification_data` |
| `participantsFisicaDocumentoINMPantalla` | `pf_immigration_document_display` |
| `participantsFisicaBeneficiarioControlador` | `pf_beneficial_owner` |
| `participantsFisicaRepresentante` | `pf_legal_representative` |
| `participantsFisicaRepresentanteDomicilio` | `pf_representative_address` |
| `participantsFisicaRepresentanteIdentificacion` | `pf_representative_identification` |

### 2.2 Persona Moral Tables

| Old table name | New table name |
|---|---|
| `moralAmbosBasico` | `pm_basic` |
| `moralAmbosDomicilio` | `pm_address` |
| `moralAmbosRepresentante` | `pm_legal_representative` |
| `moralAmbosIdentificacionRepresentante` | `pm_representative_identification` |
| `moralMexicanaPEP` | `pm_pep` |
| `moralExtranjeraDocumentoMigratorio` | `pm_immigration_document` |
| `moralExtranjeraDomicilioNacional` | `pm_national_address` |
| `moralAmbosBenControladorRelacion` | `pm_beneficial_owner` |

### 2.3 Fideicomiso Tables

| Old table name | New table name |
|---|---|
| `fideicomisoBasico` | `trust_basic` |
| `fideicomisoApoderadoDelegado` | `trust_attorney` |
| `fideicomisoIdentificacion` | `trust_identification` |
| `fideicomisoBenControladorRelacion` | `trust_beneficial_owner` |
| `fideicomisoPersonaPoliticamenteExpuesta` | `trust_pep` |

### 2.4 Anexo-7 Tables

| Old table name | New table name |
|---|---|
| `anexo_7_o_bis_basico` | `annex_7_basic` |
| `anexo_7_domicilio` | `annex_7_address` |
| `anexo_7_representante_legal` | `annex_7_legal_representative` |
| `anexo_7_identificacion` | `annex_7_identification` |
| `anexo_7_funcionario` | `annex_7_officer` |

---

## 3. Column Rename Mappings (per entity)

### 3.1 `users` table (domain-auth-users)

| Old DB column | New DB column | TypeScript property |
|---|---|---|
| `nombre` | `first_name` | `nombre` → `firstName` |
| `apellido_paterno` | `paternal_surname` | `apellidoPaterno` → `paternalSurname` |
| `apellido_materno` | `maternal_surname` | `apellidoMaterno` → `maternalSurname` |
| `telefono` | `phone` | `telefono` → `phone` |
| `activo` | `active` | `activo` → `active` |

### 3.2 `pf_basic` table (formerly participantsFisicaBasico)

| Old DB column (inferred) | New DB column | TypeScript property |
|---|---|---|
| `nombre` | `first_name` | `nombre` → `firstName` |
| `apellido_paterno` | `paternal_surname` | `apellidoPaterno` → `paternalSurname` |
| `apellido_materno` | `maternal_surname` | `apellidoMaterno` → `maternalSurname` |
| `fecha_nacimiento` | `birth_date` | `fechaNacimiento` → `birthDate` |
| `pais_nacimiento` | `birth_country` | `paisNacimiento` → `birthCountry` |
| `pais_nacionalidad` | `nationality_country` | `paisNacionalidad` → `nationalityCountry` |
| `lugar_nacimiento` | `birth_place` | `lugarNacimiento` → `birthPlace` |
| `ocupacion_profesion_giro` | `occupation` | `ocupacionProfesionGiro` → `occupation` |
| `correo` | `email` | `correo` → `email` |
| `numero_pasaporte` | `passport_number` | `numeroPasaporte` → `passportNumber` |
| `pais_emisor_pasaporte` | `passport_issuing_country` | `paisEmisorPasaporte` → `passportIssuingCountry` |
| `pais_residencia` | `residence_country` | `paisResidencia` → `residenceCountry` |

### 3.3 `pm_basic` table (formerly moralAmbosBasico)

| Old DB column (inferred) | New DB column | TypeScript property |
|---|---|---|
| `tipo_persona_moral` | `legal_entity_type` | `tipoPersonaMoral` → `legalEntityType` |
| `fecha_constitucion` | `incorporation_date` | `fechaConstitucion` → `incorporationDate` |
| `pais_nacionalidad` | `nationality_country` | `paisNacionalidad` → `nationalityCountry` |
| `objeto_social_giro` | `business_purpose` | `objetoSocialGiro` → `businessPurpose` |
| `denominacion_razon_social` | `corporate_name` | `denominacionRazonSocial` → `corporateName` |

### 3.4 `trust_basic` table (formerly fideicomisoBasico)

| Old DB column (inferred) | New DB column | TypeScript property |
|---|---|---|
| `denominacion_razon_social` | `corporate_name` | `denominacionRazonSocial` → `corporateName` |
| `numero_referencia` | `reference_number` | `numeroReferencia` → `referenceNumber` |

### 3.5 Address columns (all address tables — pf_address, pm_address, etc.)

Pattern applied consistently wherever the following columns appear:

| Old column | New column |
|---|---|
| `calle` | `street` |
| `numero_exterior` | `exterior_number` (already English-named) |
| `numero_interior` | `interior_number` |
| `colonia` | `neighborhood` |
| `municipio` | `municipality` |
| `ciudad` | `city` |
| `estado` | `state` |
| `codigo_postal` | `postal_code` |
| `pais` | `country` |
| `telefono` | `phone` |
| `correo` | `email` |
| `extension` | `extension` (no change) |

### 3.6 Representative / attorney columns (pf_legal_representative, pm_legal_representative, trust_attorney)

| Old column | New column |
|---|---|
| `nombre` | `first_name` |
| `primer_apellido` | `paternal_surname` |
| `segundo_apellido` | `maternal_surname` |
| `fecha_nacimiento` | `birth_date` |
| `pais_nacimiento` | `birth_country` |
| `lugar_nacimiento` | `birth_place` |
| `correo_electronico` | `email` |

### 3.7 PEP columns (pm_pep, trust_pep)

| Old column | New column |
|---|---|
| `es_suscribe_pep` / `ejerce_derechos` | `is_pep_signatory` / `exercises_rights` |
| `cargo_suscribe` / `cargo_ejerce_derechos` | `pep_signatory_position` |
| `tiene_parentesco_pep` / `tiene_parentesco` | `has_pep_relative` |
| `cargo_parentesco` | `relative_position` |
| `nombre_completo_pep` | `pep_full_name` |

### 3.8 Annex-7 columns (annex_7_basic)

| Old column (inferred) | New column |
|---|---|
| `tipo_anexo` | `annex_type` |
| `nombre_denominacion` | `corporate_name` |
| `fecha_constitucion` | `incorporation_date` |
| `actividad_objeto_social` | `business_purpose` |
| `nombre_persona_moral_derecho_publico` | `public_law_entity_name` |
| `fecha_creacion_rfc` | `rfc_creation_date` |

---

## 4. TypeScript Type / Interface Renames

### 4.1 libs/auth-profiles

| Old name | New name |
|---|---|
| `RegistroPerfil` | `ProfileRegistration` |
| `RegistroSuperadmin` | `SuperadminRegistration` |
| `RegistroAuxiliar` | `AuxiliaryRegistration` |
| `RegistroNotarioInmobiliariaPF` | `NotaryRealEstatePFRegistration` |
| `RegistroNotarioInmobiliariaPM` | `NotaryRealEstatePMRegistration` |
| `PerfilAcciones` | `ProfileActions` |
| `PerfilActualizacion` | `ProfileUpdate` |

Field renames within `RegistroPerfil` (→ `ProfileRegistration`):
- `rol` → `role` (the discriminant field)
- Literal string `'NOTARIO' | 'INMOBILIARIA'` → `UserRole.NOTARY | UserRole.REAL_ESTATE`

Field renames within `RegistroSuperadmin` (→ `SuperadminRegistration`):
- `nombreCompleto` → `fullName`
- `domicilio` → `address`

### 4.2 packages/shared-types (if applicable)

| Old name | New name |
|---|---|
| `PerfilBasico` | `BasicProfile` |
| `PFParticipante` | `IndividualParticipant` |
| `PMParticipante` | `LegalEntityParticipant` |

---

## 5. Dual Support Compatibility Contract

Given that the front end currently sends and receives enum values in Spanish (e.g., `role: "NOTARIO"`), the backend MUST support both old and new values for a **deprecation window of 4 weeks** after Phase 1 deployment.

### 5.1 Endpoints accepting old enum values during deprecation window

| Endpoint | Field | Accepted old values | Deadline |
|---|---|---|---|
| `POST /auth/login` | `role` (in response) | `NOTARIO`, `INMOBILIARIA`, `AUXILIAR`, `USUARIO_INTERNO`, `USUARIO_EXTERNO` | Deprecation window end |
| `GET /users` | `role` (in response) | same | same |
| `POST /admin/registration` | `user_role` (request + response) | `NOTARIO`, `INMOBILIARIA` | same |
| Any endpoint returning `profileType` | old values | `PERSONA_FISICA`, `PERSONA_MORAL`, `FIDEICOMISO` | same |
| Any endpoint returning `actividadVulnerable` / `activity` | old values | all 10 Spanish values | same |

### 5.2 Compat layer requirements

- The adapter layer MUST transform incoming old values to new enum members before processing.
- The adapter layer SHOULD log a `WARN` when an old value is received: `[DEPRECATED] Received legacy enum value "<old>" for field "<field>". Use "<new>" instead.`
- Responses SHALL return only the new English values after Phase 1 deployment — the compat layer is input-only.
- The compat layer MUST be removed at the end of the deprecation window. The `sdd-cleanup` sub-change handles this.

### 5.3 Validation scenarios

**Scenario: Login with NOTARIO role (during deprecation window)**
```
Given the user has role NOTARY in the DB
When POST /auth/login is called with valid credentials
Then the response SHALL include role: "NOTARY"
And HTTP 201 is returned
```

**Scenario: Login after cleanup (post deprecation window)**
```
Given the user has role NOTARY in the DB
And the compat layer has been removed
When POST /auth/login is called with valid credentials
Then the response SHALL include role: "NOTARY"
And any client sending role: "NOTARIO" SHALL receive HTTP 400
```

**Scenario: Table rename does not break existing queries**
```
Given a RENAME TABLE migration has been applied for participantsFisicaBasico -> pf_basic
When any endpoint that SELECTs from participantsFisicaBasico is called
Then the TypeORM entity MUST reference the new table name
And the response data is identical to pre-migration
```

---

## 6. Migration SQL Requirements

- Each up migration MUST have a corresponding down migration.
- Table renames MUST use `RENAME TABLE old TO new` (atomic in MySQL 8).
- Column renames MUST use `ALTER TABLE t CHANGE COLUMN old new <full_column_definition>`.
- Data migrations for enum values MUST use `UPDATE t SET col = 'NEW' WHERE col = 'OLD'`.
- Data migrations MUST run BEFORE the column is renamed (to avoid referencing a non-existent column name in WHERE clauses).
- Migrations MUST be idempotent where possible (check before rename using `INFORMATION_SCHEMA`).
- The migration execution order SHALL be: (1) table renames, (2) data updates, (3) column renames.

---

## 7. Requirements Summary

- R1: All enum names and values defined in `packages/shared-types` MUST be in English after this change.
- R2: All table names in `libs/participants` MUST follow snake_case English naming after Phase 2–5 migrations.
- R3: All column names in entity files MUST have an explicit `name:` property set to the English snake_case equivalent.
- R4: The `users` table columns MUST be renamed as specified in §3.1.
- R5: Dual support MUST be active for a minimum of 4 weeks before cleanup.
- R6: Every SQL migration MUST be reversible via a down migration.
- R7: `POST /auth/login` MUST return HTTP 201 after every phase migration.
- R8: TypeScript type and interface renames MUST be backward-compatible within the same package via re-exports for one release cycle (or handled atomically if the package is internal-only).
