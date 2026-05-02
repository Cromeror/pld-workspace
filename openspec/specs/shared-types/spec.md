# Specification: shared-types (pld-api)

## Requirements

### Requirement: shared-types exports only English type names

`pld-api/packages/shared-types/src/participants.ts` MUST export exactly the English-named types. MUST NOT export any Spanish-named types.

| New (English) | Old (Spanish) |
|---|---|
| `BasicProfile` | `PerfilBasico` |
| `WithName` | `ConNombre` |
| `WithBirth` | `ConNacimiento` |
| `WithTaxIds` | `ConFiscales` |
| `WithEmail` | `ConCorreo` |
| `WithAddress` | `ConDomicilio` |
| `WithContact` | `ConContacto` |
| `WithIdentification` | `ConIdentificacion` |
| `WithPEP` | `ConPEP` |
| `WithRequiredPEP` | `ConPEPObligatorio` |
| `WithBeneficialOwner` | `ConBC` |
| `IndividualParticipant` | `PFParticipante` |
| `IndividualParticipantAddress` | `PFParticipanteDomicilio` |
| `IndividualParticipantPrincipal` | `PFParticipantePrincipal` |
| `IndividualParticipantBeneficialOwner` | `PFParticipanteBC` |
| `IndividualParticipantPEP` | `PFParticipantePEP` |
| `IndividualParticipantRepresentative` | `PFParticipanteRepresentante` |
| `IndividualParticipantRepresentativeAddress` | `PFParticipanteRepresentanteDomicilio` |
| `LegalEntityParticipant` | `PMParticipante` |

#### INV-1: shared-types participants.ts exports English type names only

- GIVEN `packages/shared-types/src/participants.ts` has been updated
- WHEN `tsc --noEmit` runs on `packages/shared-types`
- THEN all English type names SHALL be importable
- AND Spanish type names (`PerfilBasico`, `PFParticipante`, `PMParticipante`, `ConNombre`, etc.) SHALL NOT resolve

---

### Requirement: shared-types property names are English

All properties within the types exported from `packages/shared-types/src/participants.ts` MUST use English names.

| English property | Replaces Spanish |
|---|---|
| `firstName` | `nombre` |
| `paternalSurname` | `apellidoPaterno` |
| `maternalSurname` | `apellidoMaterno` |
| `birthDate` | `fechaNacimiento` |
| `birthCountry` | `paisNacimiento` |
| `birthPlace` | `lugarNacimiento` |
| `email` | `correo` |
| `nationality` | `nacionalidad` |
| `occupation` | `ocupacionProfesionGiro` |
| `address` | `domicilio` |
| `contact` | `contacto` |
| `identification` | `identificacion` |
| `beneficialOwner` | `beneficiarioControlador` |
| `participantId` | `idParticipante` |
| `representativeId` | `idRepresentante` |
| `documentName` | `nombreDocumento` |
| `number` | `numero` |
| `issuingAuthority` | `autoridadEmisora` |
| `expirationDate` | `fechaVencimiento` |

#### INV-2: English property names compile; Spanish names absent

- GIVEN all property renames are applied in `participants.ts`
- WHEN any consumer accesses `.nombre` or `.apellidoPaterno` on a shared-types instance
- THEN TypeScript SHALL report a compile error (property does not exist)
- AND `.firstName` and `.paternalSurname` SHALL resolve without error

---

### Requirement: auth-profiles types use English names

`pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts` MUST export the English-named types. MUST NOT export Spanish variants.

| New (English) | Old (Spanish) |
|---|---|
| `ProfileActions` | `PerfilAcciones` |
| `ProfileUpdate` | `PerfilActualizacion` |
| `SuperadminRegistration` | `RegistroSuperadmin` |
| `AuxiliaryRegistration` | `RegistroAuxiliar` |
| `NotaryRealEstatePFRegistration` | `RegistroNotarioInmobiliariaPF` |
| `NotaryRealEstatePMRegistration` | `RegistroNotarioInmobiliariaPM` |
| `ProfileRegistration` | `RegistroPerfil` |

#### INV-3: auth-profiles.types.ts exports English type names only

- GIVEN `auth-profiles.types.ts` has been updated
- WHEN `tsc --noEmit` runs on `libs/auth-profiles`
- THEN all 7 English type names SHALL be importable
- AND all 7 Spanish type names SHALL NOT resolve

---

### Requirement: ProfileActions properties are English

`ProfileActions` MUST have properties `canEdit`, `canDelete`, `canToggleActive`. MUST NOT have `puedeEditar`, `puedeEliminar`, `puedeActivarDesactivar`.

#### INV-4: ProfileActions has English property names

- GIVEN `ProfileActions` type has been updated
- WHEN a consumer accesses `.puedeEditar` on a `ProfileActions` instance
- THEN TypeScript SHALL report a compile error
- AND `.canEdit`, `.canDelete`, `.canToggleActive` SHALL resolve without error

---

### Requirement: SuperadminRegistration properties are English

`SuperadminRegistration` MUST have `fullName` and `address`. MUST NOT have `nombreCompleto` or `domicilio`.

#### INV-5: SuperadminRegistration uses English property names

- GIVEN `SuperadminRegistration` has been updated
- WHEN `tsc --noEmit` runs
- THEN `.fullName` and `.address` SHALL exist on `SuperadminRegistration`
- AND `.nombreCompleto` and `.domicilio` SHALL NOT exist

---

### Requirement: NotaryRealEstatePFRegistration properties are English

`NotaryRealEstatePFRegistration` MUST have `individual`, `vulnerableActivity`, `activityAddress`, and `startDate`. MUST NOT have `persona`, `actividadVulnerable`, `domicilioActividad`, or `fechaInicial`.

#### INV-6: NotaryRealEstatePFRegistration uses English property names

- GIVEN `NotaryRealEstatePFRegistration` has been updated
- WHEN `tsc --noEmit` runs
- THEN `.individual`, `.vulnerableActivity`, `.activityAddress`, `.startDate` SHALL exist
- AND `.persona`, `.actividadVulnerable`, `.domicilioActividad`, `.fechaInicial` SHALL NOT exist

---

### Requirement: NotaryRealEstatePMRegistration uses legalEntity

`NotaryRealEstatePMRegistration` MUST have `legalEntity`. MUST NOT have `personaMoral`.

#### INV-7: NotaryRealEstatePMRegistration.legalEntity replaces personaMoral

- GIVEN `NotaryRealEstatePMRegistration` has been updated
- WHEN a consumer accesses `.personaMoral` on that type
- THEN TypeScript SHALL report a compile error
- AND `.legalEntity` SHALL resolve without error

---

### Requirement: ProfileRegistration discriminator uses English keys

`ProfileRegistration` discriminated union MUST use discriminator fields `role` and `data`. MUST NOT use `rol` or `datos`.

#### INV-8: ProfileRegistration discriminator fields are English

- GIVEN `ProfileRegistration` discriminated union has been updated
- WHEN `tsc --noEmit` runs on `libs/auth-profiles` and all consumers
- THEN branching on `.role` SHALL type-narrow correctly to each registration variant
- AND references to `.rol` or `.datos` SHALL produce compile errors

---

### Requirement: reports.types.ts uses vulnerableActivity

`pld-api/libs/reports/src/lib/reports.types.ts` MUST use property name `vulnerableActivity`. MUST NOT use `actividadVulnerable`.

#### INV-9: reports.types.ts vulnerableActivity replaces actividadVulnerable

- GIVEN `reports.types.ts` has been updated
- WHEN `tsc --noEmit` runs on `libs/reports`
- THEN `.vulnerableActivity` SHALL exist on the affected type
- AND `.actividadVulnerable` SHALL NOT resolve

---

### Requirement: DTO fields use English names

`apps/auth-users/.../fideicomiso/dto/fideicomiso.dto.ts` MUST use `pepFullName`. MUST NOT use `nombreCompletoPEP`.

> Nota 2026-05-02: el DTO equivalente en `apps/cross/.../beneficiario-controlador/dto/ben-controlador.dto.ts` también debía usar `pepFullName`. La app `cross` fue eliminada; el DTO equivalente sigue vivo en el lib `libs/participants/src/lib/beneficiario-controlador/` (ya en inglés tras el rename original).

#### INV-10: pepFullName replaces nombreCompletoPEP in DTOs

- GIVEN the DTO has been updated
- WHEN `tsc --noEmit` runs on `apps/auth-users`
- THEN `pepFullName` SHALL exist as a decorated DTO property
- AND `nombreCompletoPEP` SHALL NOT exist in source

---

### Requirement: HTTP route segments are English in participants controllers

All controllers under `apps/auth-users/src/participants/**/*.controller.ts` MUST use English path segments. Spanish path segments MUST NOT appear in controller decorators.

| New (English) | Old (Spanish) |
|---|---|
| `address` | `domicilio` |
| `national-address` | `domicilio-nacional` |
| `national-territorial-address` | `domicilio-territorial-nacional` |
| `representative-address` | `representante-domicilio` |

#### INV-11: Controllers use English route segments

- GIVEN all participant controllers have been updated
- WHEN a client sends `POST .../address`
- THEN the controller SHALL handle the request and return the expected response
- AND `POST .../domicilio` SHALL return 404

#### Scenario: Old Spanish route is rejected after rename

- GIVEN the participants controller route is renamed to `address`
- WHEN `GET .../domicilio` is requested
- THEN the server SHALL respond HTTP 404 (route not found)

---

### Requirement: No Spanish identifiers remain in source (cleanliness gate)

After all phases, the following patterns MUST return zero matches in `pld-api/{apps,libs,packages}/*/src` (excluding migration files and Swagger `description:` strings).

Patterns: `RegistroPerfil`, `RegistroSuperadmin`, `RegistroAuxiliar`, `RegistroNotario`, `PerfilAcciones`, `PerfilActualizacion`, `PerfilBasico`, `PFParticipante`, `PMParticipante`, `ConNombre`, `ConNacimiento`, `ConCorreo`, `ConDomicilio`, `ConContacto`, `ConIdentificacion`, `ConPEP`, `ConBC`, `nombreCompleto\b`, `actividadVulnerable\b`, `domicilioActividad`, `fechaInicial\b`, `nombreCompletoPEP`, `puedeEditar`, `puedeEliminar`, `puedeActivarDesactivar`.

#### INV-12: Exhaustive grep of old identifiers returns zero matches

- GIVEN all six phases have been applied
- WHEN the exhaustive grep runs against `pld-api/{apps,libs,packages}/*/src`
- THEN the command SHALL return zero matches

#### Scenario: Spanish controller decorator segments absent

- GIVEN all participant controllers have been updated
- WHEN `grep -rn "@Post\('domicilio'\)\|@Post\('domicilio-nacional'\)\|@Post\('domicilio-territorial-nacional'\)\|@Post\('representante-domicilio'\)" pld-api/apps` is run
- THEN the command SHALL return zero matches

---

### Requirement: Builds succeed after all renames

`nx run auth-users:build` MUST exit 0 after the change is applied. (Histórico: también `nx run cross:build`; la app `cross` fue eliminada el 2026-05-02.)

#### INV-13: Build commands exit 0

- GIVEN all type renames, property renames, DTO field renames, route renames, and consumer updates are applied
- WHEN `nx run auth-users:build` is executed
- THEN it SHALL exit 0

---

### Requirement: Login smoke test passes after change

`POST /auth/login` with valid credentials MUST return HTTP 201 with a JWT token after the change is applied.

#### INV-14: POST /auth/login returns 201 + JWT

- GIVEN all renames are applied and `auth-users` service is running
- WHEN a valid login request is sent to `POST /auth/login`
- THEN the server SHALL respond HTTP 201
- AND the response body SHALL contain a JWT token

---

## MODIFIED Requirements (from prior phases)

### Requirement: Exports de enums en packages/shared-types

### Requirement: Exports de enums en packages/shared-types

`pld-api/packages/shared-types/src/enums.ts` MUST exportar exactamente 8 enums bajo sus nombres English. Los nombres Spanish previos MUST NOT existir como exports del paquete.

| Nombre anterior | Nombre nuevo | Valores nuevos (English) |
|----------------|-------------|--------------------------|
| `UserRole` (valores Spanish) | `UserRole` | `SUPERADMIN`, `NOTARY`, `REAL_ESTATE`, `AUXILIARY` |
| `TipoPersonaParticipante` | `ProfileType` | `INDIVIDUAL`, `LEGAL_ENTITY`, `TRUST` |
| `ActividadVulnerable` | `VulnerableActivity` | valores English correspondientes |
| `TipoMoneda` | `CurrencyType` | valores English correspondientes |
| `FormaPago` | `PaymentMethod` | valores English correspondientes |
| `TipoReporte` | `ReportType` | valores English correspondientes |
| `ModalidadAtencion` | `AttendanceMode` | valores English correspondientes |
| `Nacionalidad` | `Nationality` | `MEXICAN`, `FOREIGN` |

#### INV-6: Exports con nombres English solamente

**Given** el archivo `packages/shared-types/src/enums.ts` fue modificado,
**When** se compila el paquete y se inspeccionan sus exports,
**Then** SHALL exportar `UserRole`, `ProfileType`, `VulnerableActivity`, `CurrencyType`, `PaymentMethod`, `ReportType`, `AttendanceMode`, `Nationality`,
**AND** SHALL NOT exportar `TipoPersonaParticipante`, `ActividadVulnerable`, `TipoMoneda`, `FormaPago`, `TipoReporte`, `ModalidadAtencion` ni `Nacionalidad`.

#### INV-7: Valores de enum en English

**Given** los enums están exportados con nombres English,
**When** se consultan sus valores en runtime,
**Then** cada enum SHALL contener únicamente los valores English especificados en la tabla de la propuesta,
**AND** ningún valor Spanish (`'NOTARIO'`, `'PERSONA_FISICA'`, `'EFECTIVO'`, etc.) SHALL existir en ningún enum.

### Requirement: Consumers internos usan solo nombres nuevos

Todos los consumers BE (`apps/auth-users`, `libs/auth-profiles`, `libs/operacion`, `libs/reports`) MUST importar únicamente los nombres English del paquete `shared-types`. (Histórico: la app `apps/cross` también era consumer; fue eliminada el 2026-05-02.)

#### INV-8: Consumers actualizados — imports con nombres English

**Given** la refactorización está completa,
**When** se compila `nx run auth-users:build`,
**Then** el comando SHALL completar con código de salida 0,
**AND** ningún consumer SHALL referenciar los nombres Spanish anteriores.

#### INV-9: Grep de nombres Spanish retorna cero matches

**Given** la refactorización está completa,
**When** se ejecuta `grep -rn "TipoPersonaParticipante|ActividadVulnerable|TipoMoneda|FormaPago|TipoReporte|ModalidadAtencion|Nacionalidad" pld-api/{apps,libs,packages}/*/src`,
**Then** el comando SHALL retornar cero matches.

#### INV-10: Grep de valores Spanish retorna cero matches (excl. migrations)

**Given** la refactorización está completa,
**When** se ejecuta `grep -rn "'NOTARIO'|'INMOBILIARIA'|'AUXILIAR'|'PERSONA_FISICA'|'PERSONA_MORAL'|'FIDEICOMISO'|'EFECTIVO'|'TRANSFERENCIA'|'CHEQUE'|'TARJETA'|'MEXICANA'|'EXTRANJERA'" pld-api/{apps,libs,packages}/*/src` excluyendo archivos de migraciones SQL,
**Then** el comando SHALL retornar cero matches.

#### INV-11: Build limpio post-rename

**Given** todos los renames y cascadas están aplicados,
**When** se ejecuta `nx run auth-users:build`,
**Then** SHALL completar con código de salida 0 y cero errores de compilación TypeScript.
