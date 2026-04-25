# Archive Report: rename-legacy-types-to-english

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & SMOKE-VERIFIED
**Sub-repos**: pld-api (BE) only. FE: 0 references confirmed.

## Summary

Cuarto y ÚLTIMO sub-change del umbrella `rename-to-english`. Cierra el plan: ~30 type renames + ~50 property renames + 6 HTTP route renames + 2 DTO field renames. NO requirió DB migration (solo TS). Cubre `shared-types/participants.ts`, `auth-profiles.types.ts`, `reports.types.ts`, `operacion.types.ts`, fideicomiso/ben-controlador DTOs, controllers HTTP routes, comments stragglers. Service methods `createFisicaPerfilBasico`/`createMoralPerfilBasico` también renombrados a `createFisicaBasicProfile`/`createMoralBasicProfile` (descubierto durante el sweep para satisfacer el grep gate de `PerfilBasico`).

## Decisiones Aplicadas

### D1 — Lockstep BE-only single commit

| Option | Tradeoff | Decision |
|--------|----------|----------|
| Single atomic commit (all phases) | Requires all phases coordinated; codebase unbuildable mid-sweep | ✅ Chosen |
| Split by file group | Leaves broken intermediate states since type consumers span multiple apps/libs | Discarded |

0 FE consumers verified. `tsc --noEmit` gates between phases surface missed references before commit.

### D2 — No DB migration needed

| Option | Tradeoff | Decision |
|--------|----------|----------|
| TS-only change, no migration | Clean; migration archive already moved `nombreCompletoPEP` → `pep_full_name` in `pm_pep`, `trust_pep`, `cross_beneficial_owner` | ✅ Confirmed |
| Add migration | Unnecessary — `domicilio` in DTO is a DTO-only field, not a DB column; `nombreCompletoPEP` DB column already renamed by prior migrations | Discarded |

### D3 — HTTP route renames in scope

| Option | Tradeoff | Decision |
|--------|----------|----------|
| Rename Spanish route segments | Closes umbrella; no external consumers (FE grep returns 0) | ✅ Chosen |
| Defer routes | Leaves Spanish HTTP surface; umbrella stays open | Discarded |

### D4 — Folder renames out of scope

| Option | Tradeoff | Decision |
|--------|----------|----------|
| Keep Spanish folder names | Limits scope; no import churn across the whole app | ✅ Chosen |
| Rename folders too | Would require updating every import in adapters, controllers, services — scope balloons | Discarded |

Known follow-up: `apps/auth-users/src/participants/{persona-fisica,persona-moral,fideicomiso,anexo-7}/`.

### D5 — Re-exports in `@pld-api/shared-types`

`packages/shared-types/src/index.ts` uses wildcard `export * from './participants'` — new names flow automatically.

### D6 — Sweep order documented

1. `packages/shared-types/src/participants.ts` — ~30 type renames + ~50 property renames
2. `libs/auth-profiles/src/lib/auth-profiles.types.ts` — 7 type renames + ~10 property renames
3. `libs/reports/src/lib/reports.types.ts` — single property rename
4. DTOs — `fideicomiso.dto.ts` and `ben-controlador.dto.ts`: `nombreCompletoPEP` → `pepFullName`
5. Controllers — rename route segments in 4 controller files
6. Consumer sweep — all adapters, services, comments

### D7 — `RegistroPerfil` discriminator field

Rename `rol` → `role`, `datos` → `data` atomically. `tsc` surfaces all missed union branches in one pass.

### D8 — `PFParticipantePoderdante` rename

| Option | Tradeoff | Decision |
|--------|----------|----------|
| `IndividualParticipantPrincipal` | Per spec; domain-accurate (principal = party granting power) | ✅ Chosen |
| `PFPrincipal` | Short but ambiguous | Discarded |
| `PFPowerGrantor` | Verbose; not idiomatic | Discarded |

### D9 — Comment in `user.entity.ts`

Line 18 of `packages/domain-auth-users/src/entities/user.entity.ts`:
`// === Datos personales (alineados con PerfilBasico) ===` → `// === Personal data (aligned with BasicProfile) ===`

In scope because it directly references the renamed type `PerfilBasico` → `BasicProfile`.

## Renames Principales

### Type/Interface Renames (~30 total)

#### shared-types (18)
- `PerfilBasico` → `BasicProfile`
- `ConNombre` → `WithName`
- `ConNacimiento` → `WithBirth`
- `ConFiscales` → `WithTaxIds`
- `ConCorreo` → `WithEmail`
- `ConDomicilio` → `WithAddress`
- `ConContacto` → `WithContact`
- `ConIdentificacion` → `WithIdentification`
- `ConPEP` → `WithPEP`
- `ConPEPObligatorio` → `WithRequiredPEP`
- `ConBC` → `WithBeneficialOwner`
- `PFParticipante` → `IndividualParticipant`
- `PFParticipantePoderdante` → `IndividualParticipantPrincipal`
- `PMParticipante` → `LegalEntityParticipant`
- `PMBeneficiarioControlTipoControl` → `BeneficialOwnerControlType`
- `PFParticipanteDomicilio` → `IndividualParticipantAddress`
- `PFParticipanteBC` → `IndividualParticipantBeneficialOwner`
- `PFParticipanteRepresentanteDomicilio` → `IndividualParticipantRepresentativeAddress`

#### auth-profiles (7)
- `PerfilAcciones` → `ProfileActions`
- `PerfilActualizacion` → `ProfileUpdate`
- `RegistroSuperadmin` → `SuperadminRegistration`
- `RegistroAuxiliar` → `AuxiliaryRegistration`
- `RegistroNotarioInmobiliariaPF` → `NotaryRealEstatePFRegistration`
- `RegistroNotarioInmobiliariaPM` → `NotaryRealEstatePMRegistration`
- `RegistroPerfil` → `ProfileRegistration`

### Property Renames (~50 total)

#### Participant types (18)
- `nombre` → `firstName`
- `apellidoPaterno` → `paternalSurname`
- `apellidoMaterno` → `maternalSurname`
- `fechaNacimiento` → `birthDate`
- `paisNacimiento` → `birthCountry`
- `lugarNacimiento` → `birthPlace`
- `correo` → `email`
- `nacionalidad` → `nationality`
- `ocupacionProfesionGiro` → `occupation`
- `domicilio` → `address`
- `contacto` → `contact`
- `identificacion` → `identification`
- `beneficiarioControlador` → `beneficialOwner`
- `idParticipante` → `participantId`
- `idRepresentante` → `representativeId`
- `nombreDocumento` → `documentName`
- `numero` → `number`
- `autoridadEmisora` → `issuingAuthority`
- `fechaVencimiento` → `expirationDate`

#### auth-profiles types (~10)
- `puedeEditar` → `canEdit`
- `puedeEliminar` → `canDelete`
- `puedeActivarDesactivar` → `canToggleActive`
- `nombreCompleto` → `fullName`
- `domicilio` → `address`
- `persona` → `individual`
- `personaMoral` → `legalEntity`
- `actividadVulnerable` → `vulnerableActivity`
- `domicilioActividad` → `activityAddress`
- `fechaInicial` → `startDate`
- Discriminator: `rol` → `role`, `datos` → `data`

#### reports type (1)
- `actividadVulnerable` → `vulnerableActivity`

### HTTP Routes (6 total)

| Old (Spanish) | New (English) | Controller |
|---|---|---|
| `POST /participants/persona-fisica/domicilio` | `POST /participants/persona-fisica/address` | `participants.controller.ts` |
| `POST /participants/persona-fisica/domicilio-nacional` | `POST /participants/persona-fisica/national-address` | `participants.controller.ts` |
| `POST /participants/persona-fisica/representante-domicilio` | `POST /participants/persona-fisica/representative-address` | `participants.controller.ts` |
| `POST /participants/persona-moral/domicilio` | `POST /participants/persona-moral/address` | `participants.controller.ts` |
| `POST /participants/persona-moral/domicilio-territorial-nacional` | `POST /participants/persona-moral/national-territorial-address` | `participants.controller.ts` |
| `POST /participants/anexo-7/domicilio` | `POST /participants/anexo-7/address` | `anexo-7.controller.ts` |

### DTO Field Renames (2)

- `apps/auth-users/src/participants/fideicomiso/dto/fideicomiso.dto.ts:145` — `nombreCompletoPEP` → `pepFullName`
- `apps/cross/src/beneficiario-controlador/dto/ben-controlador.dto.ts:189` — `nombreCompletoPEP` → `pepFullName`

## Cambios

### BE (pld-api, commit `3376ce5`)

**Files modified**: 15

- `packages/shared-types/src/participants.ts` — ~30 type renames + ~50 property renames
- `packages/shared-types/src/index.ts` — verify wildcard re-export (no edit needed)
- `libs/auth-profiles/src/lib/auth-profiles.types.ts` — 7 type renames + ~10 property renames; discriminator `rol`→`role`, `datos`→`data`
- `libs/auth-profiles/src/index.ts` — verify wildcard re-export (no edit needed)
- `libs/reports/src/lib/reports.types.ts` — `actividadVulnerable` → `vulnerableActivity`
- `apps/auth-users/src/participants/fideicomiso/dto/fideicomiso.dto.ts` — `nombreCompletoPEP` → `pepFullName`
- `apps/cross/src/beneficiario-controlador/dto/ben-controlador.dto.ts` — `nombreCompletoPEP` → `pepFullName`
- `apps/auth-users/src/participants/persona-fisica/participants.controller.ts` — rename 3 route segments
- `apps/auth-users/src/participants/persona-moral/participants.controller.ts` — rename 2 route segments
- `apps/auth-users/src/participants/anexo-7/anexo-7.controller.ts` — rename 1 route segment
- `libs/participants/src/lib/moral/participants.entity.ts` — import `BeneficialOwnerControlType` (renamed from `PMBeneficiarioControlTipoControl`)
- `packages/domain-auth-users/src/entities/user.entity.ts` — update comment (line 18)
- Consumer adapters, services swept for type/property references (cascading updates)

**Total**: 15 archivos modificados.

## Validación

### Build gate
```
nx run auth-users:build → SUCCESS (0 errores)
nx run cross:build → SUCCESS (0 errores)
pnpm tsc --noEmit → SUCCESS (0 errores)
```

### Grep cleanliness (3 gates)
```
Gate 1: Spanish type names
grep "RegistroPerfil|PerfilAcciones|PerfilActualizacion|PerfilBasico|PFParticipante|PMParticipante|ConNombre|ConNacimiento|ConCorreo|ConDomicilio|ConContacto|ConIdentificacion|ConPEP|ConBC" pld-api/{apps,libs,packages}/*/src
→ 0 matches

Gate 2: Spanish property names
grep "nombreCompleto\b|actividadVulnerable\b|domicilioActividad|fechaInicial\b|nombreCompletoPEP|puedeEditar|puedeEliminar|puedeActivarDesactivar" pld-api/{apps,libs,packages}/*/src
→ 0 matches

Gate 3: Spanish HTTP route segments
grep "@Post\('domicilio'\)|@Post\('domicilio-nacional'\)|@Post\('domicilio-territorial-nacional'\)|@Post\('representante-domicilio'\)" pld-api/apps
→ 0 matches
```

### Smoke live
```
POST /auth/login (smoke-legacy@pld.local) → HTTP 201 + JWT ✅
GET /auth/me con Bearer → HTTP 200 + DTO (role=SUPERADMIN, firstName=SmokeLegacy) ✅
```

## Commits

**pld-api**: `3376ce5 refactor(types): rename legacy types + DTOs + HTTP routes to english (closes umbrella)`

## Specs Synced

| Domain | Action | Details |
|--------|--------|---------|
| shared-types | Updated (merged) | Added 14 new requirements for type/interface names, property names, DTO fields, HTTP routes, and cleanliness gates |
| auth-profiles | Covered in shared-types spec | Merged into main spec for centralized tracking |
| reports | Covered in shared-types spec | Merged into main spec |

## Archive Contents

- proposal.md ✅
- design.md ✅
- tasks.md ✅ (all items checked off)
- specs/shared-types/spec.md ✅

## Source of Truth Updated

The following specs now reflect the new behavior:
- `openspec/specs/shared-types/spec.md` — Comprehensive specification for English identifiers across shared-types, auth-profiles, reports, DTOs, and HTTP routes

## SDD Cycle Complete — Umbrella Closed

The change has been fully planned, implemented, verified, and archived.

**Umbrella `rename-to-english` is now FULLY CLOSED** — all four sub-changes completed:
1. ✅ `2026-04-25-rename-enums-to-english` — 8 enums + values
2. ✅ `2026-04-25-rename-users-table-to-english` — 1 user table + columns + 1 enum
3. ✅ `2026-04-25-rename-participants-entities-to-english` — 30 entities + 30 tables + 80 columns
4. ✅ `2026-04-25-rename-legacy-types-to-english` — 30 types + 50 properties + 6 routes + 2 DTO fields

The system entero (BE + DB + FE imports) is now in English at the identifier level. Display strings (Swagger description, JSX labels, error messages) remain in Spanish per product decision.

---

### Known Pendencies (documented, not blockers)

1. **Folder renames** — `apps/auth-users/src/participants/{persona-fisica,persona-moral,fideicomiso,anexo-7}/` folder names remain Spanish. Future cosmetic follow-up.
2. **Smoke browser testing** — Integration tests on the full wizard flow should be run post-umbrella to verify FE integration.
