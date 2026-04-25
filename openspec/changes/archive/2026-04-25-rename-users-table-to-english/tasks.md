# Tasks: Rename users table + UserEntity + system-user HTTP contracts to English

> **Timestamp locked**: use `20260425020000` (resolved; previous migration was `20260425010000`).
> **getUserByPhone**: adapter parameter renames `telefono` → `phone`; `where: { telefono }` → `where: { phone }`.

## Phase 0 — Setup [Both]

- [x] 0.1 Verify `git status --short` clean in workspace root, `pld-api/`, and `pld-web/`. [Both]
- [x] 0.2 Confirm stack is up: `docker compose -f pld-api/docker-compose.dev.yml ps` shows `mysql` and `auth-users` healthy; verify `20260425020000` migration filename does not collide. [BE]
- [x] 0.3 Snapshot `docker exec -i pld-api-dev-mysql mysql -uroot -psecret pld_api_bd -e "DESCRIBE users;"` — confirm exact current column types before writing migration. [BE]

## Phase 1 — BE: Entity + Port [BE]

- [x] 1.1 Edit `pld-api/packages/domain-auth-users/src/entities/user.entity.ts`: rename 5 TS properties (`nombre`→`firstName`, `apellidoPaterno`→`paternalSurname`, `apellidoMaterno`→`maternalSurname`, `telefono`→`phone`, `activo`→`active`) AND update each `@Column({ name: '...' })` to the new English DB column names (`first_name`, `paternal_surname`, `maternal_surname`, `phone`, `active`). [BE]
- [x] 1.2 Edit `pld-api/packages/domain-auth-users/src/ports/users.port.ts` — `CreateUserInput`: rename `nombre`→`firstName`, `apellidoPaterno`→`paternalSurname`, `apellidoMaterno`→`maternalSurname`, `telefono`→`phone` (no `activo` in input). [BE]
- [x] 1.3 Edit `pld-api/packages/domain-auth-users/src/ports/users.port.ts` — `CurrentUserDto`: rename all 5 properties to English. [BE]
- [x] 1.4 Edit `pld-api/packages/domain-auth-users/src/ports/users.port.ts` — `UsersPort.getUserByPhone(telefono: string)` → `getUserByPhone(phone: string)`. [BE]

## Phase 2 — BE: Adapter [BE]

- [x] 2.1 Edit `pld-api/packages/domain-auth-users/src/adapters/users.adapter.ts` — `toCurrentUserDto` mapper: rename output keys and source references (`user.nombre`→`user.firstName`, etc.) to match new entity props. [BE]
- [x] 2.2 Edit `pld-api/packages/domain-auth-users/src/adapters/users.adapter.ts` — `createUser`: rename input destructure (`{ nombre, apellidoPaterno, ... }` → `{ firstName, paternalSurname, ... }`) AND `repo.create({...})` keys to new English names. [BE]
- [x] 2.3 Edit `pld-api/packages/domain-auth-users/src/adapters/users.adapter.ts` — `getUserByPhone(telefono)` → `getUserByPhone(phone)` AND `where: { telefono }` → `where: { phone }`. [BE]
- [x] 2.4 Check remaining `repo.findOne({ where: { ... } })` calls for any residual Spanish property names; rename if found. [BE]

## Phase 3 — BE: HTTP layer [BE]

- [x] 3.1 Edit `pld-api/apps/auth-users/src/users/dto/create-user.dto.ts` — `CreateUserNotarioInmobiliarioDto`: rename 5 TS properties to English (`nombre`→`firstName`, etc.); preserve Swagger `description:` Spanish copy and realistic Spanish example values. [BE]
- [x] 3.2 Edit `pld-api/apps/auth-users/src/users/dto/create-user.dto.ts` — `CreateUserInternoExternoDto`: same 5 prop renames; same preservation of `description:` and `example:`. [BE]
- [x] 3.3 Edit `pld-api/apps/auth-users/src/users/users.controller.ts`: change `@Post('notario-inmobiliario')` → `@Post('notary-real-estate')` and `@Post('interno-externo-auxiliar')` → `@Post('internal-external-auxiliary')`. [BE]
- [x] 3.4 Edit `pld-api/apps/auth-users/src/users/users.controller.ts`: update all handler body references (`dto.nombre`→`dto.firstName`, etc.) in both handlers; drop redundant `as string` casts if types now align. [BE]

## Phase 4 — BE: Registration internals [BE]

- [x] 4.1 Edit `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts` lines 404-419: rename `createUser` input type props (`nombre`/`apellidoPaterno`/`apellidoMaterno` → `firstName`/`paternalSurname`/`maternalSurname`) AND the `repo.create({...})` insert keys. [BE]
- [x] 4.2 Edit `pld-api/apps/auth-users/src/admin/registration/registration.service.ts` lines 441-443: rename local variables `userNombre`→`userFirstName`, `userApellidoPaterno`→`userPaternalSurname`, `userApellidoMaterno`→`userMaternalSurname` AND the call-site keys passed to the adapter. [BE]
- [x] 4.3 Grep sweep — `grep -rn "nombre:\|apellidoPaterno:\|apellidoMaterno:\|telefono:\|\.activo\b" pld-api/{apps,libs,packages}/*/src` (excluding `.sql` and Swagger `description:` strings) → must return 0 matches. Update any remaining hits in same step. [BE]
- [x] 4.4 Grep sweep — `grep -rn "\.spec\.ts" old props: `grep -rn "nombre:\|apellidoPaterno:\|apellidoMaterno:\|telefono:\|activo:" pld-api/{apps,libs,packages}/*/src/**/*.spec.ts` → 0 matches; update test fixtures if found (INV-12, D7). [BE]

## Phase 5 — BE: DB migration [BE]

- [x] 5.1 Create `pld-api/packages/persistence/migrations/20260425020000-rename-users-columns-to-english.sql` with 5 `ALTER TABLE users CHANGE COLUMN` statements using exact types from the 0.3 snapshot (`first_name VARCHAR(120) NOT NULL`, `paternal_surname VARCHAR(120) NULL`, `maternal_surname VARCHAR(120) NULL`, `phone VARCHAR(30) NULL`, `active TINYINT(1) NOT NULL DEFAULT 1`). [BE]
- [x] 5.2 Create `pld-api/packages/persistence/migrations/20260425020000-rename-users-columns-to-english.down.sql` with 5 symmetric `CHANGE COLUMN` reversals (English → Spanish, original types preserved). [BE]
- [x] 5.3 Apply up migration: `docker exec -i pld-api-dev-mysql mysql -uroot -psecret pld_api_bd < pld-api/packages/persistence/migrations/20260425020000-rename-users-columns-to-english.sql`. [BE]
- [x] 5.4 Verify: `docker exec -i pld-api-dev-mysql mysql -uroot -psecret pld_api_bd -e "DESCRIBE users; SELECT COUNT(*) FROM users;"` — shows English columns; row count matches pre-migration snapshot (INV-5, INV-6). [BE]
- [x] 5.5 Restart auth-users: `docker compose -f pld-api/docker-compose.dev.yml restart auth-users`. [BE]

## Phase 6 — BE: Builds + grep verification [BE]

- [x] 6.1 `cd pld-api && nx run auth-users:build` → 0 errors (INV-13). [BE]
- [x] 6.2 `cd pld-api && nx run cross:build` → 0 errors (INV-13). [BE]
- [x] 6.3 `cd pld-api && tsc --noEmit` for `domain-auth-users` and `shared-types` → 0 errors (INV-7). [BE]
- [x] 6.4 Final grep — `grep -rn "nombre:\|apellidoPaterno:\|apellidoMaterno:\|telefono:\|\.activo\b\|notario-inmobiliario\|interno-externo-auxiliar" pld-api/{apps,libs,packages}/*/src` → 0 matches in TS source (INV-12). [BE]

## Phase 7 — FE [Web]

- [x] 7.1 Edit `pld-web/src/types/CurrentUser.ts`: rename 5 properties to English (`nombre`→`firstName`, `apellidoPaterno`→`paternalSurname`, `apellidoMaterno`→`maternalSurname`, `telefono`→`phone`, `activo`→`active`) (INV-14). [Web]
- [x] 7.2 Grep sweep — `grep -rn "\.nombre\b\|\.apellidoPaterno\|\.apellidoMaterno\|\.telefono\b\|\.activo\b" pld-web/src --include='*.ts' --include='*.tsx'` → 0 identifier matches (INV-16). [Web]
- [x] 7.3 Visual check: confirm JSX text-content strings (`"Nombre"`, `"Apellido"`, `"Teléfono"`) remain unchanged as display labels — do NOT translate (INV-17). [Web]
- [x] 7.4 `cd pld-web && yarn tsc -b --noEmit` → 0 errors (INV-15). [Web]
- [x] 7.5 `cd pld-web && yarn build` → 0 errors (INV-15). [Web]

## Phase 8 — Smoke [Both]

- [x] 8.1 Smoke `POST /auth/login` with an existing user → HTTP 201 + JWT (INV-3). [Both]
- [x] 8.2 Smoke `GET /auth/me` with that JWT → HTTP 200; body CONTAINS `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active`; body DOES NOT contain `nombre`, `apellidoPaterno`, `apellidoMaterno`, `telefono`, `activo` (INV-1, INV-2). [Both]
- [x] 8.3 Smoke `POST /system-users/notary-real-estate` with English-named body → HTTP 201 (INV-10a). [Both]
- [x] 8.4 Smoke `POST /system-users/internal-external-auxiliary` with English-named body → HTTP 201 (INV-10b). [Both]
- [x] 8.5 Smoke `POST /system-users/notario-inmobiliario` (old path) → HTTP 404 (INV-10c). [Both]
- [x] 8.6 Down migration round-trip dry-verify: confirm `.down.sql` reverses each column exactly (symmetric check; do NOT apply unless rollback needed) (INV-11). [Both]
