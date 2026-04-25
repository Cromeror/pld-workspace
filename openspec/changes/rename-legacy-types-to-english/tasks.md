# Tasks: Rename legacy types + HTTP routes to English

## Phase 0 — Pre-flight Setup

- [ ] 0.1 [BE] Run `git status` in `pld-api/`; confirm clean working tree before starting.
- [ ] 0.2 [BE] Run `sudo rm -rf pld-api/dist/apps/auth-users pld-api/dist/apps/cross` to clear any root-owned dist artifacts.
- [ ] 0.3 [BE] Run `grep -rn "nombreCompletoPEP\|domicilio" pld-api/packages/persistence/migrations/*.sql`; confirm 0 matches (DB column already `pep_full_name`; DTO field is HTTP-only).
- [ ] 0.4 [BE] Run `grep -rn "domicilio\|domicilio-nacional\|domicilio-territorial-nacional\|representante-domicilio" pld-web/src`; confirm 0 matches (no FE consumers of renamed routes).

## Phase 1 — shared-types/participants.ts (Foundation)

- [ ] 1.1 [BE] In `pld-api/packages/shared-types/src/participants.ts`, rename all ~30 types: `PerfilBasico`→`BasicProfile`, all `Con*`→`With*` mixins (`ConNombre`→`WithName`, `ConNacimiento`→`WithBirth`, `ConFiscales`→`WithTaxIds`, `ConCorreo`→`WithEmail`, `ConDomicilio`→`WithAddress`, `ConContacto`→`WithContact`, `ConIdentificacion`→`WithIdentification`, `ConPEP`→`WithPEP`, `ConPEPObligatorio`→`WithRequiredPEP`, `ConBC`→`WithBeneficialOwner`), all `PF*`→`Individual*` variants (`PFParticipante`→`IndividualParticipant`, `PFParticipanteDomicilio`→`IndividualParticipantAddress`, `PFParticipantePrincipal`→`IndividualParticipantPrincipal`, `PFParticipanteBC`→`IndividualParticipantBeneficialOwner`, `PFParticipantePEP`→`IndividualParticipantPEP`, `PFParticipanteRepresentante`→`IndividualParticipantRepresentative`, `PFParticipanteRepresentanteDomicilio`→`IndividualParticipantRepresentativeAddress`), `PMParticipante`→`LegalEntityParticipant`, `PFParticipantePoderdante`→`IndividualParticipantPrincipal` (use proposal name; design D8 says `PFPrincipal` but spec table says `IndividualParticipantPrincipal`), `PMBeneficiarioControlTipoControl`→`BeneficialOwnerControlType`, and any remaining `PM*` type variants.
- [ ] 1.2 [BE] In the same file, rename all ~50 properties: `nombre`→`firstName`, `apellidoPaterno`→`paternalSurname`, `apellidoMaterno`→`maternalSurname`, `fechaNacimiento`→`birthDate`, `paisNacimiento`→`birthCountry`, `lugarNacimiento`→`birthPlace`, `correo`→`email`, `nacionalidad`→`nationality`, `ocupacionProfesionGiro`→`occupation`, `domicilio`→`address`, `contacto`→`contact`, `identificacion`→`identification`, `beneficiarioControlador`→`beneficialOwner`, `idParticipante`→`participantId`, `idRepresentante`→`representativeId`, `nombreDocumento`→`documentName`, `numero`→`number`, `autoridadEmisora`→`issuingAuthority`, `fechaVencimiento`→`expirationDate`.
- [ ] 1.3 [BE] Run `pnpm --filter @pld-api/shared-types tsc --noEmit` from `pld-api/`; document expected errors from downstream consumers (they haven't been updated yet) — confirm errors are only "cannot find name" for old identifiers, not internal errors.

## Phase 2 — shared-types/index.ts (Verify)

- [ ] 2.1 [BE] Open `pld-api/packages/shared-types/src/index.ts`; confirm it uses wildcard `export * from './participants'` — no per-name re-exports to update. If any explicit named re-exports for old names exist, update them to new names.

## Phase 3 — auth-profiles types

- [ ] 3.1 [BE] In `pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts`, update the import line: `PerfilBasico`→`BasicProfile`, `PFParticipante`→`IndividualParticipant`, `PMParticipante`→`LegalEntityParticipant` from `@pld-api/shared-types`.
- [ ] 3.2 [BE] In the same file, rename all 7 types: `PerfilAcciones`→`ProfileActions`, `PerfilActualizacion`→`ProfileUpdate`, `RegistroSuperadmin`→`SuperadminRegistration`, `RegistroAuxiliar`→`AuxiliaryRegistration`, `RegistroNotarioInmobiliariaPF`→`NotaryRealEstatePFRegistration`, `RegistroNotarioInmobiliariaPM`→`NotaryRealEstatePMRegistration`, `RegistroPerfil`→`ProfileRegistration`.
- [ ] 3.3 [BE] In the same file, rename all ~10 properties: `puedeEditar`→`canEdit`, `puedeEliminar`→`canDelete`, `puedeActivarDesactivar`→`canToggleActive`, `nombreCompleto`→`fullName`, `domicilio`→`address`, `persona`→`individual`, `personaMoral`→`legalEntity`, `actividadVulnerable`→`vulnerableActivity`, `domicilioActividad`→`activityAddress`, `fechaInicial`→`startDate`; rename discriminator fields `rol`→`role` and `datos`→`data` in the `ProfileRegistration` union.
- [ ] 3.4 [BE] Verify `pld-api/libs/auth-profiles/src/index.ts` uses wildcard re-export — no edit needed if so.

## Phase 4 — reports.types.ts

- [ ] 4.1 [BE] In `pld-api/libs/reports/src/lib/reports.types.ts`, rename property `actividadVulnerable`→`vulnerableActivity` in the `FiltroReporte` interface (single property change).

## Phase 5 — DTOs

- [ ] 5.1 [BE] In `pld-api/apps/auth-users/src/participants/fideicomiso/dto/fideicomiso.dto.ts` line ~145, rename `nombreCompletoPEP`→`pepFullName`. Update any `@ApiProperty()` `name:` key if it mirrors the TS field name; Swagger `description:` Spanish display copy may stay.
- [ ] 5.2 [BE] In `pld-api/apps/cross/src/beneficiario-controlador/dto/ben-controlador.dto.ts` line ~189, rename `nombreCompletoPEP`→`pepFullName`. Same Swagger rule applies.

## Phase 6 — HTTP Route Segments (Controllers)

- [ ] 6.1 [BE] In `pld-api/apps/auth-users/src/participants/persona-fisica/participants.controller.ts`: change `@Post('domicilio')`→`@Post('address')`, `@Post('domicilio-nacional')`→`@Post('national-address')`, `@Post('representante-domicilio')`→`@Post('representative-address')`.
- [ ] 6.2 [BE] In `pld-api/apps/auth-users/src/participants/persona-moral/participants.controller.ts`: change `@Post('domicilio')`→`@Post('address')`, `@Post('domicilio-territorial-nacional')`→`@Post('national-territorial-address')`.
- [ ] 6.3 [BE] In `pld-api/apps/auth-users/src/participants/anexo-7/anexo-7.controller.ts`: change `@Post('domicilio')`→`@Post('address')`.

## Phase 7 — Consumer Sweep

- [ ] 7.1 [BE] In `pld-api/libs/participants/src/lib/moral/participants.entity.ts`, update import of `PMBeneficiarioControlTipoControl`→`BeneficialOwnerControlType` from `@pld-api/shared-types`; update all usages of that enum within the file.
- [ ] 7.2 [BE] In `pld-api/packages/domain-auth-users/src/entities/user.entity.ts` line ~18, update comment: `// === Datos personales (alineados con PerfilBasico) ===`→`// === Personal data (aligned with BasicProfile) ===`.
- [ ] 7.3 [BE] Sweep all adapter files under `pld-api/apps/auth-users/src/participants/**/` — update any property accesses or type annotations referencing old Spanish property names (`nombre`, `apellidoPaterno`, `domicilio`, etc.) to their English equivalents. Run `pnpm tsc --noEmit` to surface all remaining mismatches.
- [ ] 7.4 [BE] Sweep all adapter/service files under `pld-api/apps/auth-users/src/auth-profiles/**/` — update references to renamed `RegistroPerfil`→`ProfileRegistration`, `PerfilAcciones`→`ProfileActions`, `Registro*` variants, and their discriminator fields (`rol`→`role`, `datos`→`data`).
- [ ] 7.5 [BE] Sweep `pld-api/libs/reports/src/lib/reports.service.ts` and any consumer — update property access `.actividadVulnerable`→`.vulnerableActivity`.

## Phase 8 — Verification Gates

- [ ] 8.1 [BE] Run `pnpm tsc --noEmit` from `pld-api/`; MUST return 0 errors.
- [ ] 8.2 [BE] Run `pnpm nx run auth-users:build` from `pld-api/`; MUST exit 0. If it fails with permission errors on `dist/`, re-run `sudo rm -rf pld-api/dist/apps/auth-users` then retry.
- [ ] 8.3 [BE] Run `pnpm nx run cross:build` from `pld-api/`; MUST exit 0.
- [ ] 8.4 [BE] Run exhaustive grep for old type names — `grep -rn "RegistroPerfil\|RegistroSuperadmin\|RegistroAuxiliar\|RegistroNotario\|PerfilAcciones\|PerfilActualizacion\|PerfilBasico\|PFParticipante\|PMParticipante\|ConNombre\|ConNacimiento\|ConCorreo\|ConDomicilio\|ConContacto\|ConIdentificacion\|ConPEP\|ConBC" pld-api/apps pld-api/libs pld-api/packages`; MUST return 0 matches.
- [ ] 8.5 [BE] Run exhaustive grep for old property names — `grep -rn "nombreCompleto\b\|actividadVulnerable\b\|domicilioActividad\|fechaInicial\b\|nombreCompletoPEP\|puedeEditar\|puedeEliminar\|puedeActivarDesactivar" pld-api/apps pld-api/libs pld-api/packages`; MUST return 0 matches.
- [ ] 8.6 [BE] Run grep for old HTTP route decorators — `grep -rn "@Post\('domicilio'\)\|@Post\('domicilio-nacional'\)\|@Post\('domicilio-territorial-nacional'\)\|@Post\('representante-domicilio'\)" pld-api/apps`; MUST return 0 matches.

## Phase 9 — Smoke Test

- [ ] 9.1 [BE] Restart `auth-users` service (`docker compose restart auth-users` or equivalent in `pld-api/`).
- [ ] 9.2 [BE] Send `POST /auth/login` with valid credentials; MUST receive HTTP 201 with a JWT token in the response body.
- [ ] 9.3 [BE] Create single atomic commit on `pld-api`: `refactor(shared-types): rename legacy Spanish types, props, DTOs, and HTTP routes to English`.
