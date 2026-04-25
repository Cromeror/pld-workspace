# Delta for Users

## MODIFIED Requirements

### Requirement: users MySQL table — English column names

The `users` table MUST have these columns post-migration. MUST NOT retain Spanish column names.

| Column | Replaces |
|---|---|
| `first_name` | `nombre` |
| `paternal_surname` | `apellido_paterno` |
| `maternal_surname` | `apellido_materno` |
| `phone` | `telefono` |
| `active` | `activo` |

Unchanged columns: `id`, `email`, `password_hash`, `last_login_at`, `role`, `profile_type`, `profile_id`, `must_change_password`, `created_at`, `updated_at`, `deleted_at`.

#### INV-5: Schema has English column names after up migration

- GIVEN the up migration has executed successfully
- WHEN `DESCRIBE users` is run
- THEN columns `first_name`, `paternal_surname`, `maternal_surname`, `phone`, `active` SHALL exist
- AND columns `nombre`, `apellido_paterno`, `apellido_materno`, `telefono`, `activo` SHALL NOT exist

#### INV-6: All rows preserved after migration

- GIVEN the users table contains N rows before migration
- WHEN the up migration runs
- THEN the table SHALL still contain N rows
- AND all field values SHALL be identical (column rename only, no data transform)

#### INV-11: Down migration restores Spanish column names exactly

- GIVEN the up migration has executed
- WHEN the down migration runs
- THEN columns `nombre`, `apellido_paterno`, `apellido_materno`, `telefono`, `activo` SHALL be restored
- AND columns `first_name`, `paternal_surname`, `maternal_surname`, `phone`, `active` SHALL NOT exist

---

### Requirement: UserEntity — English TypeScript properties

`UserEntity` MUST declare TypeScript properties `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active` with matching `@Column` names. MUST NOT declare Spanish-named properties.

#### INV-7: UserEntity properties are English

- GIVEN `user.entity.ts` has been updated
- WHEN `tsc --noEmit` is run on `domain-auth-users`
- THEN it SHALL exit 0
- AND the entity SHALL expose `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active` as mapped columns
- AND SHALL NOT expose `nombre`, `apellidoPaterno`, `apellidoMaterno`, `telefono`, `activo`

---

### Requirement: UsersPort interfaces — English property names

`UsersPort.CreateUserInput` MUST require `firstName` and accept optional `paternalSurname`, `maternalSurname`, `phone`. `UsersPort.CurrentUserDto` MUST include `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active`. Both interfaces MUST NOT reference Spanish names.

#### INV-8: CreateUserInput uses English props

- GIVEN `users.port.ts` has been updated
- WHEN a caller builds a `CreateUserInput` object
- THEN `firstName` SHALL be required
- AND `paternalSurname`, `maternalSurname`, `phone` SHALL be optional
- AND Spanish property names SHALL NOT be present in the interface

#### INV-9: UsersPort.CurrentUserDto uses English props

- GIVEN `users.port.ts` has been updated
- WHEN a caller reads a `CurrentUserDto` from the port
- THEN `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active` SHALL be accessible
- AND Spanish property names SHALL NOT compile

---

### Requirement: POST /system-users/* — English URL paths and request bodies

`POST /system-users/notary-real-estate` and `POST /system-users/internal-external-auxiliary` MUST accept request bodies with English property names. Old kebab-case paths MUST return 404.

| Endpoint | DTO | Required English fields |
|---|---|---|
| `POST /system-users/notary-real-estate` | `CreateUserNotarioInmobiliarioDto` | `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `email`, `role` |
| `POST /system-users/internal-external-auxiliary` | `CreateUserInternoExternoDto` | `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `email`, `role` |

Swagger `description:` copy MAY remain Spanish.

#### INV-10a: notary-real-estate endpoint accepts English body and returns 201

- GIVEN a valid authenticated request with `firstName`, `paternalSurname`, `maternalSurname`, `phone` in the body
- WHEN the client sends `POST /system-users/notary-real-estate`
- THEN the server SHALL respond HTTP 201

#### INV-10b: internal-external-auxiliary endpoint accepts English body and returns 201

- GIVEN a valid authenticated request with English-named properties in the body
- WHEN the client sends `POST /system-users/internal-external-auxiliary`
- THEN the server SHALL respond HTTP 201

#### INV-10c: Old Spanish URL paths return 404

- GIVEN the migration is complete
- WHEN the client sends `POST /system-users/notario-inmobiliario` or `POST /system-users/interno-externo-auxiliar`
- THEN the server SHALL respond HTTP 404

---

### Requirement: No Spanish TypeScript identifiers in source

After change, grep for Spanish TS identifiers in `pld-api/{apps,libs,packages}/*/src` MUST return zero matches (excluding migration files and Swagger `description:` strings).

#### INV-12: Grep returns zero Spanish TS identifier matches

- GIVEN all renames are applied
- WHEN `grep -rn "nombre:\|apellidoPaterno:\|apellidoMaterno:\|telefono:\|\.activo\b" pld-api/{apps,libs,packages}/*/src` is run
- THEN the command SHALL return zero matches

#### INV-13: Build succeeds after rename

- GIVEN all renames and migration are applied
- WHEN `nx run auth-users:build` and `nx run cross:build` are run
- THEN both SHALL exit 0
- AND `tsc --noEmit` for `shared-types` and `domain-auth-users` SHALL exit 0
