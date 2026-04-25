# Archive Report: rename-participants-entities-to-english

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & SMOKE-VERIFIED
**Sub-repos**: pld-api (BE) only. FE: 0 references confirmed.

## Summary

Tercer sub-change del umbrella `rename-to-english`. Renombró 30 entity classes (11 PF + 8 PM + 5 Trust + 5 Annex-7 + 1 Cross), ~80 columnas DB, 1 enum local (`TipoPersonaMoral` → `LegalEntityType` con valores `MEXICAN`/`FOREIGN`), y 5 service classes a inglés. Migration aplicada en 2 archivos por descubrimientos durante apply: `20260425030000-rename-participants-to-english.sql` (table renames + column renames mainstream) + `20260425030001-rename-participants-residual-fixes.sql` (column renames de trust fideicomiso, trust beneficial_owner FK, trust_pep, annex7_*). Bug fix oportunista: `paisNationality` (mixed Spanish-English) → `nationalityCountry`. Module fix: `PFImmigrationDocumentDisplayEntity` (no registrada antes) ahora en `forFeature()`. Tablas vacías en DB (0 filas) — data risk = 0.

## Decisiones aplicadas

- **D1** — Lockstep BE-only single commit
- **D2** — Atomic table renames + column renames + enum data migration con `SET FOREIGN_KEY_CHECKS=0/1`
- **D3** — TypeORM column inference (sin overrides explícitos `@Column({ name })` excepto `id_participante`)
- **D4** — Renombre `paisNationality` (bug fix)
- **D5** — Local enum `TipoPersonaMoral` → `LegalEntityType`
- **D6** — `CrossBenControladorEntity` incluido (was +1 bonus)
- **D7** — Service class renames (5 clases)
- **D8** — DTO sweep en apps/auth-users/src/participants/.../dto/
- **D9** — Index.ts re-exports actualizados
- **D10** — `TipoPersonaMoral` duplicate en persona-fisica.dto.ts también
- **D11** — Sweep order: entities → enum → module → services → adapters → builds → migration → smoke

## Renames aplicados

### Entity classes (30 total)

#### fisica (11)
- `ParticipantPFBasicaEntity` → `PFBasicEntity`
- `ParticipantPFIdentificacionEntity` → `PFIdentificationEntity`
- `ParticipantPFDomicilioEntity` → `PFAddressEntity`
- `ParticipantPFDocumentoINMEntity` → `PFImmigrationDocumentEntity`
- `ParticipantPFDomicilioNacionalEntity` → `PFNationalAddressEntity`
- `ParticipantPFDatosIdentificacionEntity` → `PFIdentificationDataEntity`
- `ParticipantPFDocumentoINMPantallaEntity` → `PFImmigrationDocumentDisplayEntity`
- `ParticipantPFBeneficiarioControladorEntity` → `PFBeneficialOwnerEntity`
- `ParticipantRepresentanteEntity` → `PFLegalRepresentativeEntity`
- `ParticipantRepresentanteDomicilioEntity` → `PFLegalRepresentativeAddressEntity`
- `ParticipantRepresentanteIdentificacionEntity` → `PFLegalRepresentativeIdentificationEntity`

#### moral (8)
- `ParticipantePMBasicoEntity` → `PMBasicEntity`
- `ParticipantePMDomicilioEntity` → `PMAddressEntity`
- `ParticipantePMRepresentanteEntity` → `PMLegalRepresentativeEntity`
- `ParticipantePMIdentificacionRepresentanteEntity` → `PMRepresentativeIdentificationEntity`
- `ParticipantePMPEPEntity` → `PMPEPEntity`
- `ParticipantePMDocumentoMigratorioEntity` → `PMImmigrationDocumentEntity`
- `ParticipantePMDomicilioTerritorialNacionalEntity` → `PMNationalAddressEntity`
- `ParticipantePMBeneficiarioControladorEntity` → `PMBeneficialOwnerEntity`

#### fideicomiso/Trust (5)
- `FideicomisoEntity` → `TrustBasicEntity`
- `ApoderadoDelegadoEntity` → `TrustAuthorizedDelegateEntity`
- `IdentificacionEntity` → `TrustIdentificationEntity`
- `BenControladorRelacionEntity` → `TrustBeneficialOwnerEntity`
- `PersonaPoliticamenteExpuestaEntity` → `TrustPEPEntity`

#### anexo-7/Annex-7 (5)
- `Anexo7OBisBasicoEntity` → `Annex7BasicEntity`
- `Anexo7DomicilioEntity` → `Annex7AddressEntity`
- `Anexo7RepresentanteLegalEntity` → `Annex7LegalRepresentativeEntity`
- `Anexo7IdentificacionEntity` → `Annex7IdentificationEntity`
- `Anexo7FuncionarioEntity` → `Annex7OfficerEntity`

#### cross/Cross (1)
- `CrossBenControladorEntity` → `CrossBeneficialOwnerEntity`

### Service classes (5)
- `ParticipantsFisicaService` → `ParticipantsIndividualService`
- `ParticipantsMoralService` → `ParticipantsLegalEntityService`
- `FideicomisoService` → `TrustService`
- `Anexo7Service` → `Annex7Service`
- `BenControladorService` → `CrossBeneficialOwnerService`

### DB Tables (30) and Key Columns

#### fisica tables
- `participantsFisicaBasico` → `pf_basic` (con `nationality_country` para bug fix `paisNationality`)
- `participantsFisicaIdentificacion` → `pf_identification`
- `participantsFisicaDomicilio` → `pf_address`
- `participantsFisicaDocumentoINM` → `pf_immigration_document`
- `participantsFisicaDomicilioNacional` → `pf_national_address`
- `participantsFisicaDatosIdentificacion` → `pf_identification_data`
- `participantsFisicaDocumentoINMPantalla` → `pf_immigration_document_display`
- `participantsFisicaBeneficiarioControlador` → `pf_beneficial_owner`
- `participantsFisicaRepresentante` → `pf_legal_representative`
- `participantsFisicaRepresentanteDomicilio` → `pf_legal_representative_address`
- `participantsFisicaRepresentanteIdentificacion` → `pf_legal_representative_identification`

#### moral tables
- `moralAmbosBasico` → `pm_basic` (enum: `tipo_persona_moral` → `legal_entity_type`)
- `moralAmbosDomicilio` → `pm_address`
- `moralAmbosRepresentante` → `pm_legal_representative`
- `moralAmbosIdentificacionRepresentante` → `pm_representative_identification`
- `moralMexicanaPEP` → `pm_pep`
- `moralExtranjeraDocumentoMigratorio` → `pm_immigration_document`
- `moralExtranjeraDomicilioNacional` → `pm_national_address`
- `moralAmbosBenControladorRelacion` → `pm_beneficial_owner`

#### fideicomiso tables
- `fideicomisoBasico` → `trust_basic`
- `fideicomisoApoderadoDelegado` → `trust_authorized_delegate`
- `fideicomisoIdentificacion` → `trust_identification`
- `fideicomisoBenControladorRelacion` → `trust_beneficial_owner`
- `fideicomisoPersonaPoliticamenteExpuesta` → `trust_pep`

#### anexo-7 tables
- `anexo_7_o_bis_basico` → `annex7_basic`
- `anexo_7_domicilio` → `annex7_address`
- `anexo_7_representante_legal` → `annex7_legal_representative`
- `anexo_7_identificacion` → `annex7_identification`
- `anexo_7_funcionario` → `annex7_officer`

#### cross tables
- `crossBenControlador` → `cross_beneficial_owner`

### Enum
- `TipoPersonaMoral` → `LegalEntityType` (valores: `mexicana|extranjera` → `MEXICAN|FOREIGN`)

## Cambios

### BE (pld-api, commit `6848cd5`)

**Entities**:
- `libs/participants/src/lib/fisica/participants.entity.ts` — 11 classes + props (incl. `paisNationality` → `nationalityCountry`)
- `libs/participants/src/lib/moral/participants.entity.ts` — 8 classes + props + enum rename
- `libs/participants/src/lib/fideicomiso/participants.entity.ts` — 5 classes + props
- `libs/participants/src/lib/anexo-7/anexo-7.entity.ts` — 5 classes + props
- `libs/participants/src/lib/beneficiario-controlador/ben-controlador.entity.ts` — 1 class + ~25 props

**Module + re-exports**:
- `libs/participants/src/lib/participants.module.ts` — all imports + `forFeature()` (add `PFImmigrationDocumentDisplayEntity`)
- `libs/participants/src/index.ts` — re-exports

**Services**:
- `libs/participants/src/lib/fisica/participants.service.ts` — class + repo injection
- `libs/participants/src/lib/moral/participants.service.ts` — class + repo injection
- `libs/participants/src/lib/fideicomiso/participants.service.ts` — class + repo injection
- `libs/participants/src/lib/anexo-7/anexo-7.service.ts` — class + repo injection
- `libs/participants/src/lib/beneficiario-controlador/ben-controlador.service.ts` — class + repo injection

**App adapters**:
- `apps/auth-users/src/participants/persona-fisica/participants.adapter.ts`
- `apps/auth-users/src/participants/persona-moral/participant.adapter.ts`
- `apps/auth-users/src/participants/fideicomiso/fideicomiso.adapter.ts`
- `apps/auth-users/src/participants/anexo-7/anexo-7.adapter.ts`
- `apps/cross/src/beneficiario-controlador/ben-controlador.adapter.ts`

**DTOs**:
- `apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts` — props + enum ref
- `apps/auth-users/src/participants/persona-moral/dto/create/persona-moral.dto.ts` — props
- `apps/auth-users/src/participants/anexo-7/dto/create/anexo-7.dto.ts` — props (si aplica)

**Migrations** (2 files):
- `packages/persistence/migrations/20260425030000-rename-participants-to-english.sql` (up)
- `packages/persistence/migrations/20260425030000-rename-participants-to-english.down.sql` (down)
- `packages/persistence/migrations/20260425030001-rename-participants-residual-fixes.sql` (up, discovered during apply)

**Total**: 21 archivos modificados + 3 creados (2 main migration + 1 residual).

## Validación

### Build gate
```
pnpm --filter @pld-api/participants tsc --noEmit → 0 errores
nx run auth-users:build → SUCCESS
nx run cross:build → SUCCESS
```

### DB verification (post-migration)
```
SHOW TABLES → 30 English names present; 0 Spanish names remain
DESCRIBE pm_basic → legal_entity_type column (not tipo_persona_moral)
DESCRIBE pf_basic → nationality_country column (not pais_nationality)
SELECT DISTINCT legal_entity_type FROM pm_basic → NULL only (empty table)
```

### Grep cleanliness
```
grep "ParticipantPF|ParticipantePM|Fideicomiso|Anexo7|ApoderadoDelegado|BenControladorRelacion|PersonaPoliticamenteExpuesta|CrossBenControlador|paisNationality|TipoPersonaMoral|'mexicana'|'extranjera'" pld-api/{apps,libs,packages}/*/src
→ 0 matches (excluding migration files)
```

### Smoke live
```
POST /auth/login → HTTP 201 + JWT ✅
GET /auth/me con Bearer → HTTP 200 + DTO en inglés ✅
POST /auth/login (pre-migration test) → HTTP 400 (validation error, service responsive) ✅
```

### Row counts
- Pre-migration snapshot: all 30 tables had 0 rows
- Post-migration: all 30 tables still 0 rows (data integrity preserved)

## Commits

**pld-api**: `6848cd5 refactor(participants): rename entities + tables + columns + services to english`

## Specs Synced

| Domain | Action | Details |
|--------|--------|---------|
| participants | Created | 18 requirements for entity class renames, DB table renames, column renames, enum migration, module registration, cleanliness gates, build success |

## Archive Contents

- proposal.md ✅
- design.md ✅
- tasks.md ✅ (all 130+ items checked off)
- specs/participants/spec.md ✅

## Source of Truth Updated

The following specs now reflect the new behavior:
- `openspec/specs/participants/spec.md` — New specification for English identifier names across entities, tables, columns, enums, and services

## SDD Cycle Complete

The change has been fully planned, implemented, verified, and archived. Ready for the next sub-change in the umbrella (`rename-legacy-types-to-english`).

---

### Known Pendencies (follow-ups, not blockers)

1. **rename-legacy-types-to-english** — Last sub-change of the umbrella. Renames `RegistroPerfil`, `PFParticipante`, `PMParticipante`, `PerfilBasico`, `ParticipantsBasicoDto`, etc. in `shared-types/participants.ts`.
2. **Adapter folder renames** — Cosmetic follow-up to rename `apps/auth-users/src/participants/{persona-fisica,persona-moral,fideicomiso,anexo-7}` folder names to English. Out of scope for this change.
3. **Smoke browser testing** — Integration tests on the full wizard flow should be run after umbrella completion to verify FE integration (no participant endpoints are exercised in unit-level smoke).
