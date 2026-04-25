# Tasks: Rename shared enums to English (names + values)

> **Decision locked**: `VulnerableActivity.FIDEICOMISOS` → `TRUST_AGREEMENTS` (confirmed by user). Use `TRUST_AGREEMENTS` everywhere — NOT `TRUSTS`, NOT `TRUST`.

## Phase 0 — Setup [Both]

- [x] 0.1 Verify `git status --short` clean in workspace root, `pld-api/`, and `pld-web/`. [Both]
- [x] 0.2 Confirm stack is up: `docker compose -f pld-api/docker-compose.dev.yml ps` shows `mysql` and `auth-users` healthy. [BE]

## Phase 1 — BE: shared-types enum rename [BE]

- [x] 1.1 Edit `pld-api/packages/shared-types/src/enums.ts`: rename all 8 enums + values per the full mapping below. `UserRole` keeps its name; all 7 others are renamed. Values: `NOTARIO`→`NOTARY`, `INMOBILIARIA`→`REAL_ESTATE`, `AUXILIAR`→`AUXILIARY`; `TipoPersonaParticipante`→`ProfileType` with `INDIVIDUAL`/`LEGAL_ENTITY`/`TRUST`; `ActividadVulnerable`→`VulnerableActivity` with 10 English values (including `FIDEICOMISOS`→`TRUST_AGREEMENTS`); `TipoMoneda`→`CurrencyType` (`MXN`/`USD` unchanged); `FormaPago`→`PaymentMethod` (`CASH`/`WIRE_TRANSFER`/`CHECK`/`CARD`/`OTHER`); `TipoReporte`→`ReportType` (4 values); `ModalidadAtencion`→`AttendanceMode` (`OWN_ACCOUNT`/`WITH_REPRESENTATIVE`); `Nacionalidad`→`Nationality` (`MEXICAN`/`FOREIGN`). [BE]
- [x] 1.2 Edit `pld-api/packages/shared-types/src/participants.ts`: update lines that reference `TipoPersonaParticipante.*` → `ProfileType.*` (L40, L103, L178). Property names (`denominacionFiduciario`, `identificadorFideicomiso`, `rfcFiduciario`) are out of scope — do NOT rename them. [BE]
- [x] 1.3 Edit `pld-api/packages/shared-types/src/index.ts`: verify re-exports include all new enum names; remove any re-export of old Spanish names. [BE]
- [x] 1.4 Verify shared-types compiles: `cd pld-api && yarn nx run shared-types:build` → 0 errors. [BE]

## Phase 2 — BE: call-site cascade [BE]

- [x] 2.1 Edit `pld-api/apps/auth-users/src/admin/registration/dto/create-registration.dto.ts` (L3, L7-9, L11-13, L18-23): update import to `ProfileType`; `IsIn(['NOTARY','REAL_ESTATE'])` for `userRole`; `IsIn(['INDIVIDUAL','LEGAL_ENTITY'])` for `profileType`; Swagger `@ApiProperty({ enum: ... })` and example values → English. [BE]
- [x] 2.2 Edit `pld-api/apps/auth-users/src/admin/registration/registration.service.ts` (L106-111, L180, L198, L209, L226, L329-330, L386, L400, L407): update imports and comparisons from old Spanish literals/names to new English equivalents. [BE]
- [x] 2.3 Edit `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts` (L358): `TipoPersonaParticipante.PERSONA_FISICA` → `ProfileType.INDIVIDUAL`. [BE]
- [x] 2.4 Edit `pld-api/apps/auth-users/src/admin/registration/registration.service.spec.ts` (L23-24): update fixture literals to new English enum values. [BE]
- [x] 2.5 Edit `pld-api/apps/auth-users/src/users/dto/create-user.dto.ts` (L30, L72): update Swagger examples to `UserRole.NOTARY`/`UserRole.AUXILIARY`. [BE]
- [x] 2.6 Edit `pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts` (L22-23): change string literal union `'AUXILIAR'|'NOTARIO'|'INMOBILIARIA'` → `'AUXILIARY'|'NOTARY'|'REAL_ESTATE'`. [BE]
- [x] 2.7 Edit `pld-api/libs/operacion/src/lib/operacion.types.ts`: update imports from `ActividadVulnerable`/`TipoMoneda`/`FormaPago` → `VulnerableActivity`/`CurrencyType`/`PaymentMethod`; update any usage sites. [BE]
- [x] 2.8 Edit `pld-api/libs/reports/src/lib/reports.types.ts`: update imports from `TipoReporte`/`TipoPersonaParticipante`/`ActividadVulnerable` → `ReportType`/`ProfileType`/`VulnerableActivity`; update usage sites. [BE]
- [x] 2.9 Edit `pld-api/apps/auth-users/src/catalogs/catalogs.data.ts` (L84): `ActividadVulnerable.FIDEICOMISOS` → `VulnerableActivity.TRUST_AGREEMENTS`. [BE]
- [x] 2.10 Check `pld-api/apps/auth-users/src/admin/registration/registration.controller.ts` (L144): update ApiOperation summary if it contains Spanish enum names. [BE]
- [x] 2.11 Grep sweep — names: `grep -rn "TipoPersonaParticipante\|ActividadVulnerable\|TipoMoneda\|FormaPago\|TipoReporte\|ModalidadAtencion\|Nacionalidad" pld-api/{apps,libs,packages}/*/src` → 0 matches (INV-9). [BE]
- [x] 2.12 Grep sweep — values: `grep -rn "'NOTARIO'\|'INMOBILIARIA'\|'AUXILIAR'\|'PERSONA_FISICA'\|'PERSONA_MORAL'\|'FIDEICOMISO'\|'EFECTIVO'\|'TRANSFERENCIA'\|'CHEQUE'\|'TARJETA'\|'MEXICANA'\|'EXTRANJERA'" pld-api/{apps,libs,packages}/*/src` → 0 matches (INV-10). [BE]
- [x] 2.13 Build: `cd pld-api && nx run auth-users:build` → 0 errors (INV-8, INV-11). [BE]
- [x] 2.14 Build: `cd pld-api && nx run cross:build` → 0 errors (INV-8, INV-11). [BE]

## Phase 3 — BE: DB migration [BE]

- [x] 3.1 Read `pld-api/packages/persistence/migrations/20260421000000-create-registration-schema.sql` to confirm exact column names (`profile_type`, `user_role`) and whether `vulnerable_activity` table + `activity` column exist. [BE]
- [x] 3.2 Create `pld-api/packages/persistence/migrations/{TS}-rename-enums-to-english.sql` with: (a) 3-step ENUM migration for `users.role` (expand → backfill → restrict per D2); (b) defensive `UPDATE` for `registration.profile_type` (PERSONA_FISICA→INDIVIDUAL, PERSONA_MORAL→LEGAL_ENTITY, FIDEICOMISO→TRUST); (c) defensive `UPDATE` for `registration.user_role` (NOTARIO→NOTARY, INMOBILIARIA→REAL_ESTATE, AUXILIAR→AUXILIARY); (d) if `vulnerable_activity.activity` column exists, defensive `UPDATE` covering all 10 values including `FIDEICOMISOS`→`TRUST_AGREEMENTS`. [BE]
- [x] 3.3 Create `pld-api/packages/persistence/migrations/{TS}-rename-enums-to-english.down.sql` with symmetric reversal: restrict → backfill → expand for `users.role`; reverse UPDATEs for `registration` and `vulnerable_activity` tables. [BE]
- [x] 3.4 Apply up migration: `docker exec -i pld-api-dev-mysql mysql -uroot -psecret pld_api_bd < pld-api/packages/persistence/migrations/{TS}-rename-enums-to-english.sql`. [BE]
- [x] 3.5 Verify migration: `docker exec -i pld-api-dev-mysql mysql -uroot -psecret pld_api_bd -e "DESCRIBE users; SELECT DISTINCT role FROM users;"` — ENUM definition shows only English values; data rows contain no Spanish values (INV-1, INV-2). [BE]

## Phase 4 — FE: types + cascade [Web]

- [x] 4.1 Edit `pld-web/src/types/UserRole.ts`: rename const-object members `NOTARIO`→`NOTARY`, `INMOBILIARIA`→`REAL_ESTATE`, `AUXILIAR`→`AUXILIARY` (INV-12). [Web]
- [x] 4.2 Edit `pld-web/src/types/registration.ts`: rename `TipoPersonaParticipante` const+type to `ProfileType`; rename values to `INDIVIDUAL`/`LEGAL_ENTITY`/`TRUST`; update all internal references (INV-13). [Web]
- [x] 4.3 Edit `pld-web/src/types/CurrentUser.ts` (L12): update `profileType` field from `"PERSONA_FISICA" | "PERSONA_MORAL" | null` to `"INDIVIDUAL" | "LEGAL_ENTITY" | null` (or import `ProfileType` from registration) (INV-14). [Web]
- [x] 4.4 Edit `pld-web/src/types/PersonType.ts`: rename const-object members to English per D6 mapping: `FISICA_MEXICANA`→`INDIVIDUAL_MEXICAN`, `FISICA_EXTRANJERA`→`INDIVIDUAL_FOREIGN`, `MORAL_MEXICANA`→`LEGAL_ENTITY_MEXICAN`, `MORAL_EXTRANJERA`→`LEGAL_ENTITY_FOREIGN`, `PUBLICA_ANEXO7`→`PUBLIC_ANNEX_7`, `PUBLICA_ANEXO7BIS`→`PUBLIC_ANNEX_7_BIS`, `FIDEICOMISO_ANEXO8`→`TRUST_ANNEX_8`. [Web]
- [x] 4.5 Edit `pld-web/src/components/organisms/reporting-entity/schemas.ts` (L9, L14-15): update `z.enum([...])` arrays to new English values; update imports. [Web]
- [x] 4.6 Edit `pld-web/src/components/organisms/reporting-entity/ReportingEntityTypeStep/index.tsx` (L30, L36, L71, L74, L81, L84): update all `UserRole.NOTARIO`/`INMOBILIARIA` → new names; update all `TipoPersonaParticipante.PERSONA_FISICA`/`PERSONA_MORAL` → `ProfileType.INDIVIDUAL`/`LEGAL_ENTITY`. Do NOT change display strings like `"Notario"` or `"Persona Física"` (INV-19). [Web]
- [x] 4.7 Edit `pld-web/src/components/organisms/reporting-entity/ReviewStep/index.tsx` (L52, L58, L61): update comparisons from `"NOTARIO"`/`"REAL_ESTATE"` and `TipoPersonaParticipante.PERSONA_MORAL` → `ProfileType.LEGAL_ENTITY`; verify display labels remain in Spanish (INV-16, INV-19). [Web]
- [x] 4.8 Edit `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx` (L30, L125, L137, L169, L264, L359-364): update imports + all literal references to `UserRole.NOTARIO`/`INMOBILIARIA` and `TipoPersonaParticipante.*` → new English equivalents (INV-16). [Web]
- [x] 4.9 Edit `pld-web/src/components/organisms/external-users/PersonType/index.tsx` (L80-81): `PersonTypeEnum.FIDEICOMISO_ANEXO8` → `PersonTypeEnum.TRUST_ANNEX_8`. [Web]
- [x] 4.10 Check `pld-web/src/config/postLoginRedirect.ts` (L13): update any literal references to Spanish enum values or comments using old names. [Web]
- [x] 4.11 Grep sweep: `grep -rn "'NOTARIO'\|'INMOBILIARIA'\|'AUXILIAR'\|'PERSONA_FISICA'\|'PERSONA_MORAL'\|'FIDEICOMISO'\|TipoPersonaParticipante" pld-web/src` → 0 matches (INV-18). [Web]
- [x] 4.12 VISUAL check: confirm JSX text content `"Notario"`, `"Inmobiliaria"`, `"Persona Física"`, `"Persona Moral"` remain unchanged as display labels — do NOT translate these to English (INV-19). [Web]
- [x] 4.13 Type-check: `cd pld-web && yarn tsc -b --noEmit` → 0 errors (INV-17). [Web]
- [x] 4.14 Build: `cd pld-web && yarn build` → 0 errors (INV-17). [Web]

## Phase 5 — Smoke + Verify [Both]

- [x] 5.1 Restart auth-users container: `docker compose -f pld-api/docker-compose.dev.yml restart auth-users`. [BE]
- [x] 5.2 Smoke `POST /auth/login` with a notary/real_estate/auxiliary test user → HTTP 201; JWT `role` claim = `"NOTARY"` / `"REAL_ESTATE"` / `"AUXILIARY"` (INV-3). [BE]
- [x] 5.3 Smoke `GET /auth/me` with the JWT from 5.2 → HTTP 200; `role` = English value; `profileType` = `null` / `"INDIVIDUAL"` / `"LEGAL_ENTITY"` (INV-4). [BE]
- [x] 5.4 Smoke `POST /admin/registration` with `userRole: "NOTARY"`, `profileType: "INDIVIDUAL"` → HTTP 201. Then repeat with `userRole: "NOTARIO"` → HTTP 400 (D7). [BE]
- [x] 5.5 Down-migration round-trip: apply `.down.sql`; verify `DESCRIBE users` shows Spanish ENUM values (INV-5); re-apply `.sql`; verify English ENUM restored. [BE]
- [x] 5.6 Final cross-repo grep: `grep -rn "'NOTARIO'\|'PERSONA_FISICA'\|TipoPersonaParticipante\|ActividadVulnerable\|FormaPago\|TipoReporte\|ModalidadAtencion\|Nacionalidad\|TipoMoneda" pld-api/{apps,libs,packages}/*/src pld-web/src` → 0 matches (excluding migration `.sql` files). [Both]
