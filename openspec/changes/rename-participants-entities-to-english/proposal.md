# Proposal: Rename participants entities + tables + columns to English

## Intent

Phase 3 of the `rename-to-english` umbrella. Audit confirmed 29 participant entities (11 fisica + 8 moral + 5 fideicomiso + 5 anexo-7) plus 1 bonus `CrossBenControladorEntity`, all carrying Spanish class names, table names, and column names. Sub-change is BE-only — FE consumer sweep returned 0 hits for any participant entity class or table name. Also fixes the `paisNationality` typo (mixed Spanish-English) and a missed entity registration (`ParticipantPFDocumentoINMPantallaEntity` is defined but absent from `participants.module.ts` `forFeature()`). Local enum `TipoPersonaMoral` (`mexicana|extranjera` lowercase) becomes `LegalEntityType` (`MEXICAN|FOREIGN` uppercase) requiring a one-line data migration.

## Scope

### In Scope
- 30 entity classes renamed (29 participants + 1 cross beneficial owner) across `pld-api/libs/participants/src/lib/{fisica,moral,fideicomiso,anexo-7,beneficiario-controlador}`.
- 30 DB tables renamed via `RENAME TABLE`.
- ~80 TS property renames; matching `ALTER TABLE … CHANGE COLUMN` per renamed property.
- Bug fix: `paisNationality` → `nationalityCountry` (col `nationality_country`).
- Local enum `TipoPersonaMoral` → `LegalEntityType` with uppercase values; `UPDATE pm_basic SET legal_entity_type=...` data migration.
- Module registration fix: add `PFImmigrationDocumentDisplayEntity` (was unregistered) to `participants.module.ts` `forFeature()`.
- Service file class renames (`BenControladorService` → `CrossBeneficialOwnerService`, etc.) and repository injection updates.
- App-side adapter consumers in `apps/auth-users/src/participants/{persona-fisica,persona-moral,fideicomiso,anexo-7}/...adapter.ts` and `apps/cross/src/beneficiario-controlador/ben-controlador.adapter.ts`.
- DTO files in `apps/auth-users/src/participants/**/dto/*` ONLY when they reference renamed entity property names directly.
- Single TypeORM migration `20260425030000-rename-participants-to-english.{up,down}.sql` covering all 30 table renames, ~80 column renames, and the enum data migration, wrapped between `SET FOREIGN_KEY_CHECKS=0/1`.
- Single atomic commit on pld-api.

### Out of Scope
- Legacy types in `shared-types/participants.ts` (`PFParticipante`, `PMParticipante`, `RegistroPerfil`, etc.) — deferred to `rename-legacy-types-to-english`.
- Display strings (Swagger `description:` Spanish copy, error messages, validation labels) — stay Spanish.
- pld-web — confirmed 0 references.
- Backwards-compat aliases for any old class/table/column names.

## Approach

Lockstep, no dual-support window. Seven phases inside a single change, single commit on pld-api:

1. **Phase A — fisica** (11 entities): rename classes + props (incl. `paisNationality` bug). `tsc --noEmit` after.
2. **Phase B — moral** (8 entities + local enum): rename classes + props; rewrite `TipoPersonaMoral` enum to `LegalEntityType`.
3. **Phase C — fideicomiso** (5 entities): rename classes + props.
4. **Phase D — anexo-7** (5 entities): rename classes + props.
5. **Phase E — beneficiario-controlador** (1 entity): rename `CrossBenControlador*` → `CrossBeneficialOwner*`.
6. **Phase F — module + service consumers**: update `participants.module.ts` (incl. missed registration), each subfolder's `participants.service.ts`, and all app adapters.
7. **Phase G — migration up/down**: one atomic SQL migration. `SET FOREIGN_KEY_CHECKS=0`; 30 `RENAME TABLE`; ~80 `ALTER TABLE … CHANGE COLUMN`; `UPDATE pm_basic SET legal_entity_type='MEXICAN' WHERE legal_entity_type='mexicana'` (and `FOREIGN`); `SET FOREIGN_KEY_CHECKS=1`. Down mirrors in reverse.

TypeORM strategy: most `@Column` decorators omit `name:`, so renaming the TS property changes the inferred snake_case column name. Migration renames the DB column to match the new property name in the same atomic step — no transition window.

Build/smoke gate after all phases: `nx run auth-users:build`, `nx run cross:build`, `tsc --noEmit` on `libs/participants`, `POST /auth/login` returns 201 + JWT, exhaustive grep for old identifiers returns 0 hits in `pld-api/{apps,libs,packages}/*/src`.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-api/libs/participants/src/lib/fisica/participants.entity.ts` | Modified | 11 classes + props renamed; `paisNationality` typo fixed. |
| `pld-api/libs/participants/src/lib/moral/participants.entity.ts` | Modified | 8 classes + props renamed; `TipoPersonaMoral` → `LegalEntityType`. |
| `pld-api/libs/participants/src/lib/fideicomiso/participants.entity.ts` | Modified | 5 classes + props renamed. |
| `pld-api/libs/participants/src/lib/anexo-7/participants.entity.ts` | Modified | 5 classes + props renamed. |
| `pld-api/libs/participants/src/lib/beneficiario-controlador/*.entity.ts` | Modified | `CrossBenControladorEntity` → `CrossBeneficialOwnerEntity`. |
| `pld-api/libs/participants/src/lib/participants.module.ts` | Modified | All renamed classes; add missing `PFImmigrationDocumentDisplayEntity`. |
| `pld-api/libs/participants/src/lib/{fisica,moral,fideicomiso,anexo-7}/participants.service.ts` | Modified | Repository injections by new entity class. |
| `pld-api/libs/participants/src/lib/beneficiario-controlador/ben-controlador.service.ts` | Modified | Class rename to `CrossBeneficialOwnerService`. |
| `pld-api/apps/auth-users/src/participants/{persona-fisica,persona-moral,fideicomiso,anexo-7}/...adapter.ts` | Modified | Service usage + property accesses. |
| `pld-api/apps/auth-users/src/participants/**/dto/*` | Modified (conditional) | Only files that reference entity property names directly. |
| `pld-api/apps/cross/src/beneficiario-controlador/ben-controlador.adapter.ts` | Modified | New service class name. |
| `pld-api/packages/persistence/migrations/20260425030000-rename-participants-to-english.{up,down}.sql` | New | 30 `RENAME TABLE`, ~80 `CHANGE COLUMN`, enum data UPDATE, FK checks toggle. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Sweep miss across 30 classes / ~80 columns / many consumers | High | `tsc --noEmit` after every phase; final exhaustive grep for `ParticipantPF|ParticipantPM|Fideicomiso|Anexo7|ApoderadoDelegado|BenControladorRelacion|PersonaPoliticamenteExpuesta|IdentificacionEntity|CrossBenControlador|paisNationality` returns 0 in src trees. |
| TypeORM inferred column name divergence if migration not applied | High | Migration is single atomic file; apply in full before BE restart; success criteria includes `DESCRIBE` checks. |
| FK constraint violation during 30-table rename | Med | `SET FOREIGN_KEY_CHECKS=0` at migration start, `=1` at end; mirror in down. |
| Enum data migration fails on rows with unexpected values | Low | Pre-flight `SELECT DISTINCT tipo_persona_moral FROM participantsMoralBasico` at apply time; only `mexicana|extranjera|NULL` expected per audit. |
| `dist/apps/auth-users` permission errors blocking rebuild (recurrent) | Med | Cleanup `dist/` before `nx run` if needed (apply phase). |
| HTTP contract change to participants endpoints if DTO props leak entity names | Low | Audit DTO files in apply; only rename DTO props that mirror entity props. FE has 0 readers either way. |
| Data preservation in non-empty dev DB | Low | Migration uses `RENAME TABLE`/`CHANGE COLUMN` (data-preserving); sub-agent verifies row counts before/after. |

## Rollback Plan

1. `git revert <commit>` on pld-api → entities, services, module, adapters, DTOs back to Spanish.
2. Run `20260425030000-rename-participants-to-english.down.sql` → `SET FOREIGN_KEY_CHECKS=0`; reverse enum UPDATE (`'MEXICAN'`→`'mexicana'`, `'FOREIGN'`→`'extranjera'`); reverse all `CHANGE COLUMN`; reverse all `RENAME TABLE`; `SET FOREIGN_KEY_CHECKS=1`.
3. Verify smoke: `POST /auth/login` returns 201 + JWT; `SHOW TABLES LIKE 'participants%'` shows old Spanish names; `SELECT DISTINCT tipo_persona_moral FROM participantsMoralBasico` returns `mexicana|extranjera|NULL`.

## Dependencies

- TypeORM migration runner already wired in `pld-api/packages/persistence` (used by previous phases of the umbrella).
- No external service depends on these table or column names (FE has 0 references — verified).
- Phases 1 and 2 of the umbrella (`rename-enums-to-english`, `rename-users-table-to-english`) already archived; this change is independent of them.

## Success Criteria

- [ ] `tsc --noEmit` returns 0 errors in `pld-api/libs/participants` and on the apps that consume participant services.
- [ ] `nx run auth-users:build` and `nx run cross:build` exit 0.
- [ ] `SHOW TABLES;` lists new English participant table names (`pf_basic`, `pm_basic`, `trust_basic`, `annex7_basic`, `cross_beneficial_owner`, etc.); old Spanish names absent.
- [ ] `DESCRIBE pm_basic` shows column `legal_entity_type` (not `tipo_persona_moral`); `SELECT DISTINCT legal_entity_type FROM pm_basic` returns only `MEXICAN|FOREIGN|NULL`.
- [ ] `DESCRIBE pf_basic` shows column `nationality_country` (not `pais_nationality` and not `paisNationality`).
- [ ] Smoke: `POST /auth/login` returns HTTP 201 + JWT.
- [ ] Grep in `pld-api/{apps,libs,packages}/*/src` returns 0 matches for: `ParticipantPF|ParticipantePM|Fideicomiso(?!Controller)|Anexo7|ApoderadoDelegado|BenControladorRelacion|PersonaPoliticamenteExpuesta|IdentificacionEntity|CrossBenControlador|paisNationality|'mexicana'|'extranjera'` (excluding migration files and Swagger `description:` strings).
- [ ] `participants.module.ts` `forFeature()` includes `PFImmigrationDocumentDisplayEntity`.
- [ ] Display copy in Swagger `description:` examples remains Spanish.
