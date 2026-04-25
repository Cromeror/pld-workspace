# Proposal: Rename users table + UserEntity + system-user HTTP contracts to English

## Intent

Close Phase 2 of the `rename-to-english` umbrella by aligning the `users` table, `UserEntity`, the `CurrentUserDto` (`GET /auth/me`) shape, and the `POST /system-users/*` HTTP contracts on English identifiers. Today the entity carries Spanish columns (`nombre`, `apellido_paterno`, `apellido_materno`, `telefono`, `activo`) and the `system-users` controller exposes Spanish field names and Spanish kebab-case URL paths (`/notario-inmobiliario`, `/interno-externo-auxiliar`). FE consumer sweep returned 0 hits for these endpoints and 0 readers of the Spanish DTO props, so the contract change is safe. Display strings (Swagger `description:` examples, JSX labels) stay Spanish — only code identifiers, DB columns, and URL paths move. Single lockstep change across both sub-repos.

## Scope

### In Scope
- BE entity + persistence: rename 5 columns + 5 TS properties on `UserEntity` (`nombre|apellidoPaterno|apellidoMaterno|telefono|activo` → `firstName|paternalSurname|maternalSurname|phone|active`).
- BE port: rename props on `CreateUserInput` and `CurrentUserDto` in `users.port.ts`; cascade through `users.adapter.ts` (`toCurrentUserDto` mapper + `createUser` insert).
- BE HTTP DTOs: rename props in both DTO classes inside `apps/auth-users/src/users/dto/create-user.dto.ts` (`CreateUserNotarioInmobiliarioDto`, `CreateUserInternoExternoDto`) — Swagger decorators + class-validator stay; `description:` Spanish copy preserved.
- BE controller: update `apps/auth-users/src/users/users.controller.ts` mappings (8 references) and rename URL paths `/notario-inmobiliario` → `/notary-real-estate`, `/interno-externo-auxiliar` → `/internal-external-auxiliary`.
- BE registration: align `registration.adapter.ts:404-419` createUser call and rename locals in `registration.service.ts:441-443` (`userNombre|...` → `userFirstName|...`).
- DB: TypeORM migration renaming 5 columns on `users`; down migration reverses.
- FE: update `pld-web/src/types/CurrentUser.ts` mirror (5 props). 0 component consumers — verified.
- Single atomic commit per sub-repo.

### Out of Scope
- Renaming participant entities/tables in `libs/participants` — `rename-participants-entities-to-english`.
- Renaming legacy types (`RegistroPerfil`, `PFParticipante`) — `rename-legacy-types-to-english`.
- Display strings: Swagger `description:` Spanish examples, JSX labels, toasts, zod messages — stay Spanish.
- JWT claims (unaffected — only `sub`, `email`, `role`).
- Backwards-compat alias for old URL paths or DTO field names.

## Approach

Lockstep, no dual-support window. Four phases inside a single change, one commit per sub-repo:

1. **BE entity + port + adapter rename** — edit `UserEntity` columns/props, `CreateUserInput`/`CurrentUserDto` props, `toCurrentUserDto` mapper keys+values, `createUser` insert keys. `tsc --noEmit` flags every miss.
2. **BE HTTP layer** — rename DTO props in `create-user.dto.ts`, controller mappings + URL paths in `users.controller.ts`, registration adapter call + service locals.
3. **DB migration up/down** — single TypeORM migration with 5 `RENAME COLUMN` statements on `users` (and reverse in down).
4. **FE type alignment** — rename 5 props in `CurrentUser.ts`. `tsc -b --noEmit` + `yarn build` confirm 0 consumers broke.

Build/smoke gate after all phases: `nx run auth-users:build`, `nx run cross:build`, FE `yarn build`, `POST /auth/login` 201 + JWT, `GET /auth/me` returns DTO with English props, `POST /system-users/notary-real-estate` 201.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-api/packages/domain-auth-users/src/entities/user.entity.ts` | Modified | 5 `@Column` names + TS property names. |
| `pld-api/packages/domain-auth-users/src/ports/users.port.ts` | Modified | `CreateUserInput` + `CurrentUserDto` prop renames. |
| `pld-api/packages/domain-auth-users/src/adapters/users.adapter.ts` | Modified | `toCurrentUserDto` keys + values; `createUser` insert keys. |
| `pld-api/apps/auth-users/src/users/users.controller.ts` | Modified | 8 mapping references + 2 URL path renames. |
| `pld-api/apps/auth-users/src/users/dto/create-user.dto.ts` | Modified | Both DTO classes' props (Swagger + class-validator). Spanish `description:` preserved. |
| `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts` | Modified | L404-419 createUser call keys. |
| `pld-api/apps/auth-users/src/admin/registration/registration.service.ts` | Modified | L441-443 local var renames. |
| `pld-api/packages/persistence/migrations/` | New | Up + down migration: 5 `RENAME COLUMN` on `users`. |
| `pld-web/src/types/CurrentUser.ts` | Modified | 5 prop renames. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| BE call-site sweep miss → silent runtime mismatch | Med | `tsc --noEmit` on `pld-api`; grep for old TS identifiers (`nombre|apellidoPaterno|apellidoMaterno|telefono|activo`) returns 0 outside migration files and Swagger `description:` strings. |
| HTTP contract breaking change for external callers (Postman, third-party docs) | Low | 0 FE consumers verified for the 2 renamed endpoints; document the breaking change in tasks; bump endpoint docs. |
| DB migration partial failure mid-rename | Low | 5 `RENAME COLUMN` statements wrapped in transaction; pre-flight `DESCRIBE users` snapshot; rollback via down migration. |
| FE type miss → runtime undefined when reading current user | Low | `yarn build` (`tsc -b`) returns 0; FE consumer grep for the 5 old props returns 0 hits. |
| Test fixture drift (`*.spec.ts` referencing old props) | Med | Grep `*.spec.ts` for old identifiers; update fixtures in same commit. |

## Rollback Plan

1. `git revert` the FE rename commit in `pld-web` → `CurrentUser.ts` mirror back to Spanish.
2. `git revert` the BE rename commit in `pld-api` → entity, ports, adapters, controller, DTOs, URL paths, registration locals back to Spanish.
3. Run the down migration → 5 `RENAME COLUMN` statements reverse on `users`.
4. Verify smoke: `POST /auth/login` returns 201 + JWT; `GET /auth/me` returns DTO with `nombre|apellidoPaterno|...`; `POST /system-users/notario-inmobiliario` returns 201.

## Dependencies

- TypeORM migration runner already wired in `pld-api/packages/persistence` (used by Phase-1 enum migration).
- No external service depends on these column names or HTTP field names (verified: 0 FE consumers, 0 third-party integrations identified).
- `CreateUserOutput` extends `Omit<UserEntity, 'passwordHash'>` — auto-picks renamed entity fields.

## Success Criteria

- [ ] `tsc --noEmit` returns 0 errors in `pld-api` and `pld-web`.
- [ ] `nx run auth-users:build` and `nx run cross:build` exit 0.
- [ ] FE `yarn tsc -b --noEmit` and `yarn build` exit 0.
- [ ] `DESCRIBE users` shows columns: `first_name`, `paternal_surname`, `maternal_surname`, `phone`, `active` (no `nombre|apellido_paterno|apellido_materno|telefono|activo`).
- [ ] Smoke: `POST /auth/login` returns HTTP 201 + JWT.
- [ ] Smoke: `GET /auth/me` returns 200 with body containing `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active`.
- [ ] Smoke: `POST /system-users/notary-real-estate` returns 201; `POST /system-users/internal-external-auxiliary` returns 201.
- [ ] Grep: `nombre|apellidoPaterno|apellidoMaterno|telefono|activo` as TS identifiers in `pld-api/{apps,libs,packages}/*/src` and `pld-web/src` returns 0 matches (excluding migration files and Swagger `description:` strings).
- [ ] Grep: `notario-inmobiliario|interno-externo-auxiliar` returns 0 matches in BE source (excluding migration history).
- [ ] Display copy in Swagger `description:` examples and JSX remains Spanish.
