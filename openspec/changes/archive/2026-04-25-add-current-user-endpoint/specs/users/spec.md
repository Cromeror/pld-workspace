# Delta for Users

## ADDED Requirements

### Requirement: Columna must_change_password en UserEntity

La entidad `UserEntity` MUST incluir la columna `must_change_password BOOLEAN NOT NULL DEFAULT true`. La migración MUST agregar la columna y MUST hacer backfill de todas las filas pre-existentes a `false` en la misma transacción.

#### INV-8: Nuevos usuarios vía POST /system-users/* tienen must_change_password = true

**Given** el endpoint `POST /pld-api/auth-users/system-users/notario-inmobiliario` o `POST /pld-api/auth-users/system-users/interno-externo-auxiliar`,
**When** se crea un nuevo usuario,
**Then** el campo `must_change_password` SHALL ser `true` sin necesidad de enviarlo en el payload.

#### INV-9: Filas pre-existentes tras migración tienen must_change_password = false

**Given** la migración se ejecuta sobre una BD con registros existentes en la tabla `users`,
**When** la migración completa sin errores,
**Then** todos los registros que existían antes de la migración SHALL tener `must_change_password = false`,
**AND** los registros nuevos creados después de la migración SHALL tener `must_change_password = true` por default.

## ADDED Requirements (cont.)

### Requirement: Puerto UsersPort — método getCurrentUser con DTO público

`UsersPort` MUST exponer un método `getCurrentUser(id: string): Promise<CurrentUserDto | null>` que retorna el DTO público `{ id, email, role, nombre, apellidoPaterno, apellidoMaterno, telefono, activo, profileType, profileId, lastLoginAt, mustChangePassword }`. MUST NOT exponer `passwordHash`, `createdAt`, `updatedAt` ni `deletedAt`. El método existente `getUserById(id)` que retorna `UserEntity` se MANTIENE intacto para uso interno (otros consumers ya dependen de él).

#### INV-10: getCurrentUser retorna DTO sin campos internos

**Given** un userId válido de usuario existente, activo y no eliminado,
**When** se llama `getCurrentUser(id)`,
**Then** el resultado SHALL contener `{ id, email, role, nombre, apellidoPaterno, apellidoMaterno, telefono, activo, profileType, profileId, lastLoginAt, mustChangePassword }`,
**AND** SHALL NOT contener `passwordHash`, `createdAt`, `updatedAt` ni `deletedAt`.

**Given** un userId que no existe o está soft-deleted,
**When** se llama `getCurrentUser(id)`,
**Then** SHALL retornar `null`.
