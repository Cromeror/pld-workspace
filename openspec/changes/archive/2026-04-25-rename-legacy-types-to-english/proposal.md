# Proposal: Rename legacy types + HTTP routes to English

## Intent

Phase 4 (final) of the `rename-to-english` umbrella. After enums, users table, and participants entities/columns were Anglicized in the prior three sub-changes, the only Spanish identifiers remaining in pld-api are: (a) legacy type/interface definitions in `packages/shared-types/src/participants.ts` (~250 lines, ~30 types, ~50 properties), (b) auth-profiles types in `libs/auth-profiles/src/lib/auth-profiles.types.ts`, (c) one stray prop in `libs/reports`, (d) two stray DTO fields (`nombreCompletoPEP`, `domicilio`), and (e) HTTP route segments in participants controllers (e.g. `@Post('domicilio')`). FE has 0 consumers of these types or routes (verified). Closing this sub-change closes the umbrella.

## Scope

### In Scope
- ~30 type/interface renames in `pld-api/packages/shared-types/src/participants.ts` (`PerfilBasico`, `ConNombre`, `ConNacimiento`, `ConFiscales`, `ConCorreo`, `ConDomicilio`, `ConContacto`, `ConIdentificacion`, `ConPEP`, `ConPEPObligatorio`, `ConBC`, `PFParticipante*`, `PMParticipante*`).
- ~50 property renames inside those types (`nombre`, `apellidoPaterno`, `apellidoMaterno`, `fechaNacimiento`, `paisNacimiento`, `lugarNacimiento`, `correo`, `ocupacionProfesionGiro`, `domicilio`, `contacto`, `identificacion`, `beneficiarioControlador`, `idParticipante`, `idRepresentante`, `nombreDocumento`, `numero`, `autoridadEmisora`, `fechaVencimiento`, `nombreCompleto`, `actividadVulnerable`, `domicilioActividad`, `fechaInicial`).
- Auth-profiles types renamed (`PerfilAcciones` → `ProfileActions`, `PerfilActualizacion` → `ProfileUpdate`, `RegistroSuperadmin` → `SuperadminRegistration`, `RegistroAuxiliar` → `AuxiliaryRegistration`, `RegistroNotarioInmobiliariaPF` → `NotaryRealEstatePFRegistration`, `RegistroNotarioInmobiliariaPM` → `NotaryRealEstatePMRegistration`, `RegistroPerfil` → `ProfileRegistration`) plus their props (`puedeEditar`, `puedeEliminar`, `puedeActivarDesactivar`, `nombreCompleto`, `domicilio`, `persona`, `personaMoral`, `actividadVulnerable`, `rol`, `datos`).
- Reports type prop `actividadVulnerable` → `vulnerableActivity` in `libs/reports/src/lib/reports.types.ts`.
- DTO field renames: `nombreCompletoPEP` → `pepFullName` in `apps/auth-users/.../fideicomiso/dto/fideicomiso.dto.ts:145` and `apps/cross/.../beneficiario-controlador/dto/ben-controlador.dto.ts:189`.
- HTTP route segment renames in `apps/auth-users/src/participants/**/*.controller.ts`: `domicilio` → `address`, `domicilio-nacional` → `national-address`, `domicilio-territorial-nacional` → `national-territorial-address`, `representante-domicilio` → `representative-address` (full sweep).
- `@pld-api/shared-types` re-exports updated to new names.
- All consumer call sites (controllers, adapters, services, DTOs) updated to new identifiers.
- Single atomic commit on pld-api.

### Out of Scope
- Folder renames under `apps/auth-users/src/participants/{persona-fisica,persona-moral,fideicomiso,anexo-7}/` — folder names remain Spanish (documented as known follow-up).
- Swagger `description:` Spanish display copy.
- Inline comments in Spanish describing domain terms.
- Migration files preserving Spanish history (already archived).
- pld-web — confirmed 0 consumers of these types or routes.
- Backwards-compat type aliases.
- DB migrations — these are TS-only renames; verify `domicilio`/`nombreCompletoPEP` are not DB columns (expected to be DTO-only after participants rename).

## Approach

Lockstep, BE-only, single PR, single atomic commit on pld-api. Six phases with `tsc --noEmit` gate after each.

1. **Phase A — shared-types core**: rename ~30 types and ~50 props in `packages/shared-types/src/participants.ts`. Update `index.ts` re-exports.
2. **Phase B — auth-profiles types**: rename types and props in `libs/auth-profiles/src/lib/auth-profiles.types.ts`. The `RegistroPerfil` discriminated union: rename discriminator field `rol` → `role` (values already English from sub-change 1) and `datos` → `data`.
3. **Phase C — reports type**: single prop rename in `libs/reports/src/lib/reports.types.ts`.
4. **Phase D — DTO stray fields**: rename `nombreCompletoPEP` and `domicilio` in the two identified DTOs; pre-flight verify these are NOT DB columns post-participants-rename.
5. **Phase E — HTTP route segments**: sweep `apps/auth-users/src/participants/**/*.controller.ts` and rename Spanish route segments. Pre-flight verify FE has 0 consumers (`grep` in pld-web for the four old route segments).
6. **Phase F — consumer sweep**: update all callers (controllers, adapters, services, DTOs, mappers) to new identifiers. `tsc --noEmit` clean.

Build/smoke gate at the end: `nx run auth-users:build`, `nx run cross:build`, `tsc --noEmit` on `packages/shared-types`, `libs/auth-profiles`, `libs/reports`. Smoke: `POST /auth/login` returns 201 + JWT. Final exhaustive grep of old identifiers returns 0 in `pld-api/{apps,libs,packages}/*/src`.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-api/packages/shared-types/src/participants.ts` | Modified | ~30 type renames + ~50 property renames. |
| `pld-api/packages/shared-types/src/index.ts` | Modified | Re-exports updated to new names. |
| `pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts` | Modified | 7 type renames + 10 property renames; `RegistroPerfil` discriminator field `rol`→`role`, `datos`→`data`. |
| `pld-api/libs/reports/src/lib/reports.types.ts` | Modified | `actividadVulnerable` → `vulnerableActivity`. |
| `pld-api/apps/auth-users/src/participants/fideicomiso/dto/fideicomiso.dto.ts` | Modified | `nombreCompletoPEP` → `pepFullName`. |
| `pld-api/apps/cross/src/beneficiario-controlador/dto/ben-controlador.dto.ts` | Modified | `nombreCompletoPEP` → `pepFullName`. |
| `pld-api/apps/auth-users/src/participants/**/*.controller.ts` | Modified | Spanish route segments renamed (`domicilio`, `domicilio-nacional`, `domicilio-territorial-nacional`, `representante-domicilio`). |
| `pld-api/apps/auth-users/src/participants/**/*.adapter.ts` and `*.service.ts` | Modified | Consumers of renamed shared-types and auth-profiles types. |
| `pld-api/apps/auth-users/src/auth-profiles/**/*` | Modified | Consumers of `RegistroPerfil`, `PerfilAcciones`, `Registro*` variants. |
| `pld-api/libs/reports/src/lib/reports.service.ts` and consumers | Modified | Updated property accessors. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Sweep miss across ~30 types + ~50 props + many consumers | High | `tsc --noEmit` after each phase; final exhaustive grep returns 0 hits for the locked identifier list (see Success Criteria). |
| HTTP route consumers outside the repo (Postman collections, external clients) | Med | Workspace-coordinated; no external consumers identified. Document in archive notes. |
| FE consumes the renamed routes silently | Low | Pre-flight `grep` in pld-web for `domicilio`, `domicilio-nacional`, `domicilio-territorial-nacional`, `representante-domicilio` route patterns before applying Phase E. Default true (no consumers expected). |
| `RegistroPerfil` discriminated union breaks if `rol` field is renamed inconsistently | Med | Rename discriminator field `rol`→`role` atomically; tsc surfaces missed branches across `auth-profiles` adapter/service consumers. |
| Stray DTO field corresponds to a real DB column (domicilio) | Low | Pre-flight `DESCRIBE` on relevant tables; participants rename already moved DB columns to English (`address`). Expect zero hits. |
| `dist/apps/auth-users` permission errors blocking rebuild (recurrent in prior phases) | Med | Cleanup `dist/` before `nx run` if needed during apply. |

## Rollback Plan

1. `git revert <commit>` on pld-api → all type, property, DTO, and HTTP route changes back to Spanish.
2. No DB rollback needed — change has 0 schema deltas.
3. Verify smoke: `POST /auth/login` returns 201 + JWT; `GET /participants/.../address` returns 404 (because route reverted to `/domicilio`); old type names compile in shared-types.

## Dependencies

- Sub-changes 1–3 of the umbrella (`rename-enums-to-english`, `rename-users-table-to-english`, `rename-participants-entities-to-english`) already archived. This change assumes their renames are landed (entity classes, table names, column names, enums all English).
- No external service depends on these types or routes (FE has 0 references — verified).

## Success Criteria

- [ ] `tsc --noEmit` returns 0 errors in `pld-api/packages/shared-types`, `pld-api/libs/auth-profiles`, `pld-api/libs/reports`, `pld-api/apps/auth-users`, `pld-api/apps/cross`.
- [ ] `nx run auth-users:build` and `nx run cross:build` exit 0.
- [ ] Smoke: `POST /auth/login` returns HTTP 201 + JWT.
- [ ] Grep in `pld-api/{apps,libs,packages}/*/src` returns 0 matches (excluding migration files and Swagger `description:` strings) for: `RegistroPerfil`, `RegistroSuperadmin`, `RegistroAuxiliar`, `RegistroNotario`, `PerfilAcciones`, `PerfilActualizacion`, `PerfilBasico`, `PFParticipante`, `PMParticipante`, `ConNombre`, `ConNacimiento`, `ConCorreo`, `ConDomicilio`, `ConContacto`, `ConIdentificacion`, `ConPEP`, `ConBC`, `nombreCompleto\b`, `actividadVulnerable\b`, `domicilioActividad`, `fechaInicial\b`, `nombreCompletoPEP`, `puedeEditar`, `puedeEliminar`, `puedeActivarDesactivar`.
- [ ] Grep returns 0 matches for the renamed HTTP route segments in controller decorators: `@Post\('domicilio'\)`, `@Post\('domicilio-nacional'\)`, `@Post\('domicilio-territorial-nacional'\)`, `@Post\('representante-domicilio'\)`.
- [ ] Display copy in Swagger `description:` examples and Spanish folder names under `apps/auth-users/src/participants/` remain unchanged (out-of-scope items preserved).
- [ ] `rename-to-english` umbrella is fully closed; any remaining Spanish identifiers in pld-api come from intentional decisions (folders, comments, display copy).
