# Auth Specification

## Purpose

Define el contrato de autenticación: el endpoint HTTP de login, el puerto `AuthPort`, y los contratos de firma/verificación JWT puros. Preserva el comportamiento observable de `POST auth/login` bit-a-bit.

## Requirements

### Requirement: Login con credenciales válidas

El sistema MUST responder HTTP 201 con `{ token: string }` cuando se reciben credenciales correctas.

#### Scenario: Login exitoso

- GIVEN un usuario activo existe con email `user@example.com` y password correcto
- WHEN un cliente envía `POST /pld-api/auth-users/auth/login` con `{ email, password }`
- THEN el servidor responde HTTP 201
- AND el body es `{ token: string }` donde `token` es un JWT firmado válido
- AND el token contiene el claim `userId` igual al `id` del usuario

### Requirement: Login con credenciales inválidas

El sistema MUST responder HTTP 401 con el shape de error uniforme definido en la spec `http-error-formatting` cuando las credenciales son incorrectas o el usuario no existe.

#### Scenario: Password incorrecto

- GIVEN un usuario existe con email `user@example.com`
- WHEN un cliente envía `POST /pld-api/auth-users/auth/login` con password erróneo
- THEN el servidor responde HTTP 401
- AND el body tiene `statusCode=401`, `message=["Credenciales inválidas"]`, `timestamp`, `path`, `method=POST`, `errorDetails.message=["Credenciales inválidas"]`

#### Scenario: Email no registrado

- GIVEN ningún usuario existe con el email enviado
- WHEN un cliente envía `POST /pld-api/auth-users/auth/login`
- THEN el servidor responde HTTP 401 con el mismo shape de error

### Requirement: Puerto AuthPort

El paquete `domain-auth-users` MUST exponer una interfaz `AuthPort` con al menos los métodos `verifyCredentials` e `issueToken`. La implementación concreta NO MUST ser expuesta al consumidor.

#### Scenario: verifyCredentials retorna usuario o null

- GIVEN una instancia de `AuthPort`
- WHEN se llama `verifyCredentials({ email, password })` con credenciales válidas
- THEN retorna el objeto del usuario sin exponer el `passwordHash`
- WHEN se llama con credenciales inválidas
- THEN retorna `null`

#### Scenario: issueToken produce JWT con userId

- GIVEN una instancia de `AuthPort` y un userId válido
- WHEN se llama `issueToken({ userId })`
- THEN retorna un JWT firmado que incluye el claim `userId`

### Requirement: JWT puro — sign y verify

La lógica de firma y verificación JWT MUST residir en el paquete `domain-auth-users` sin dependencia de `@nestjs/jwt`. El módulo MUST NOT importar ningún símbolo de `@nestjs/*`.

#### Scenario: Token expirado rechazado con 401

- GIVEN un cliente presenta un token con `exp` en el pasado
- WHEN el gateway valida el token vía `JwtAuthGuard` local
- THEN el servidor responde HTTP 401

#### Scenario: Token con firma inválida rechazado con 401

- GIVEN un cliente presenta un token con firma manipulada
- WHEN el gateway valida el token
- THEN el servidor responde HTTP 401

#### Scenario: Token válido permite acceso

- GIVEN un cliente presenta un token activo y correctamente firmado
- WHEN accede a un endpoint protegido con `JwtAuthGuard`
- THEN el servidor responde con el recurso solicitado (no 401)

### Requirement: JwtAuthGuard local en `auth-users`

`JwtAuthGuard` y `JwtStrategy` MUST existir como copias locales en `apps/auth-users/src/shared/auth/`. La app SHOULD NOT importar estos símbolos desde `@pld-api/jwt`.

> Nota 2026-05-02: el monorepo tenía una segunda copia en `apps/cross/src/shared/auth/`. La app `cross` fue eliminada y la convención original ("copia por app") aplica hoy a una sola app.

#### Scenario: auth-users usa guard local

- GIVEN el archivo `apps/auth-users/src/shared/auth/jwt-auth.guard.ts` existe
- WHEN se inspecciona el módulo principal de auth-users
- THEN los controllers protegidos referencian el guard desde la ruta local
- AND no existe import de `JwtAuthGuard` desde `@pld-api/jwt`

### Requirement: GET /auth/me — identidad del usuario autenticado

El sistema MUST exponer `GET /pld-api/auth-users/auth/me` protegido con `JwtAuthGuard`. El endpoint MUST retornar los datos públicos del usuario identificado por el claim `sub` del JWT. El body MUST NOT incluir `passwordHash`, `createdAt`, `updatedAt` ni `deletedAt`.

El response MUST incluir el header `Cache-Control: private, no-store`.

#### INV-1: Token válido retorna 200 con DTO público

**Given** un cliente presenta un JWT válido, activo y con `sub` que referencia un usuario existente, activo y no eliminado,
**When** envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 200 con body `{ id, email, role, nombre, apellidoPaterno, apellidoMaterno, telefono, activo, profileType, profileId, lastLoginAt, mustChangePassword }`,
**AND** el header de respuesta SHALL incluir `Cache-Control: private, no-store`,
**AND** el body SHALL NOT contener `passwordHash`, `createdAt`, `updatedAt` ni `deletedAt`.

#### INV-2: Token ausente retorna 401

**Given** un cliente no envía header `Authorization`,
**When** envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 401.

#### INV-3: Token con firma inválida retorna 401

**Given** un cliente presenta un JWT con firma manipulada,
**When** envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 401.

#### INV-4: Token expirado retorna 401

**Given** un cliente presenta un JWT con `exp` en el pasado,
**When** envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 401.

#### INV-5: Token referencia usuario con soft-delete retorna 401

**Given** un JWT válido cuyo `sub` corresponde a un usuario con `deletedAt` no nulo,
**When** el cliente envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 401.

#### INV-6: Token referencia usuario inactivo retorna 401

**Given** un JWT válido cuyo `sub` corresponde a un usuario con `activo = false`,
**When** el cliente envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 401.

### Requirement: Login actualiza lastLoginAt antes de emitir el token

(Previously: `POST /auth/login` solo emitía el JWT sin actualizar `lastLoginAt`.)

El sistema MUST actualizar `users.last_login_at = NOW()` después de verificar las credenciales y ANTES de firmar el JWT. La operación MUST completarse dentro de la misma unidad de trabajo que la autenticación. El contrato HTTP de `POST /auth/login` (HTTP 201 + `{ token }`) MUST NOT cambiar.

#### INV-7: lastLoginAt se actualiza en cada login exitoso

**Given** un usuario activo con credenciales correctas,
**When** el cliente envía `POST /auth/login`,
**Then** el servidor SHALL responder HTTP 201 con `{ token: string }`,
**AND** el campo `last_login_at` del usuario en BD SHALL reflejar el timestamp del login recién efectuado.

### Requirement: UserRole ENUM — columna users.role

La columna `users.role` en MySQL MUST aceptar exactamente los valores `'SUPERADMIN'`, `'NOTARY'`, `'REAL_ESTATE'`, `'AUXILIARY'` al finalizar la migración. Los valores previos en español MUST NOT ser aceptados por el ENUM después de la restricción final.

#### INV-1: Columna role acepta solo valores English post-migración

**Given** la migración up fue ejecutada a completitud,
**When** se inspecciona la definición del ENUM en `INFORMATION_SCHEMA`,
**Then** los valores permitidos SHALL ser exactamente `('SUPERADMIN','NOTARY','REAL_ESTATE','AUXILIARY')`,
**AND** los valores `'NOTARIO'`, `'INMOBILIARIA'`, `'AUXILIAR'` SHALL NOT existir en la definición.

#### INV-2: Filas existentes migradas a valores English

**Given** la migración up fue ejecutada,
**When** se consulta `SELECT role FROM users`,
**Then** ninguna fila SHALL contener `'NOTARIO'`, `'INMOBILIARIA'` ni `'AUXILIAR'`,
**AND** todas las filas con `'NOTARIO'` previo SHALL tener `role = 'NOTARY'`,
**AND** todas las filas con `'INMOBILIARIA'` previo SHALL tener `role = 'REAL_ESTATE'`,
**AND** todas las filas con `'AUXILIAR'` previo SHALL tener `role = 'AUXILIARY'`,
**AND** las filas con `'SUPERADMIN'` previo SHALL permanecer sin cambios.

#### INV-5: Down migration restaura valores Spanish

**Given** la migración up fue ejecutada,
**When** se ejecuta la migración down,
**Then** la definición del ENUM SHALL ser restaurada a los valores Spanish originales,
**AND** todas las filas SHALL haber sido revertidas a sus valores Spanish correspondientes.

### Requirement: JWT role claim — valores English

`POST /auth/login` MUST emitir el JWT con el claim `role` usando los nuevos valores English. No MUST NOT existir lógica de compatibilidad hacia atrás que decodifique valores Spanish del claim `role`.

#### INV-3: Login exitoso emite JWT con role en English

**Given** un usuario con `role = 'NOTARY'` en BD (post-migración),
**When** el cliente envía `POST /auth/login` con credenciales válidas,
**Then** el servidor SHALL responder HTTP 201 con `{ token: string }`,
**AND** el claim `role` del JWT SHALL ser `"NOTARY"` (o `"REAL_ESTATE"` / `"AUXILIARY"` / `"SUPERADMIN"` según corresponda),
**AND** el claim `role` SHALL NOT contener valores Spanish.

#### INV-4: GET /auth/me retorna role en English

**Given** un JWT válido cuyo `sub` referencia un usuario activo post-migración,
**When** el cliente envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 200,
**AND** el campo `role` en el body SHALL ser uno de `"SUPERADMIN"`, `"NOTARY"`, `"REAL_ESTATE"`, `"AUXILIARY"`.

### Requirement: GET /auth/me/workspaces — workspaces COMPLETED del usuario autenticado

El sistema MUST exponer `GET /pld-api/auth-users/auth/me/workspaces` protegido con `JwtAuthGuard`. El endpoint MUST retornar la lista de workspaces (registrations en estado `COMPLETED`) a los que el usuario identificado por el claim `sub` puede acceder, expresados como pares `{ activityType, workspaceId }`.

El response MUST incluir el header `Cache-Control: private, no-store`.

Para `SUPERADMIN` el endpoint MUST retornar lista vacía (`workspaces: []`), porque `SUPERADMIN` no pertenece a workspaces.

El endpoint MUST NOT requerir parámetros de query ni body. La identidad se resuelve exclusivamente desde el JWT.

#### INV-1: WORKSPACE_ADMIN con dos registrations COMPLETED retorna ambas

- GIVEN un usuario `WORKSPACE_ADMIN` con dos registrations COMPLETED, una `NOTARY` y otra `REAL_ESTATE`
- WHEN un cliente envía `GET /auth/me/workspaces` con un JWT válido de ese usuario
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL ser `{ workspaces: [{ activityType: 'NOTARY', workspaceId: '<uuid>' }, { activityType: 'REAL_ESTATE', workspaceId: '<uuid>' }] }`
- AND el header `Cache-Control: private, no-store` SHALL estar presente

#### INV-2: WORKSPACE_ADMIN con una sola registration COMPLETED retorna solo esa

- GIVEN un usuario `WORKSPACE_ADMIN` con una registration COMPLETED `NOTARY` y ninguna otra
- WHEN un cliente envía `GET /auth/me/workspaces`
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL ser `{ workspaces: [{ activityType: 'NOTARY', workspaceId: '<uuid>' }] }`

#### INV-3: SUPERADMIN retorna lista vacía

- GIVEN un usuario `SUPERADMIN`
- WHEN un cliente envía `GET /auth/me/workspaces` con un JWT válido de ese usuario
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL ser `{ workspaces: [] }`

#### INV-4: AUXILIARY retorna sus registrations COMPLETED si aplica

- GIVEN un usuario `AUXILIARY` con al menos una registration COMPLETED accesible
- WHEN un cliente envía `GET /auth/me/workspaces`
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL contener las registrations COMPLETED accesibles para ese AUXILIARY

#### INV-5: Token ausente o inválido retorna 401

- GIVEN un cliente no envía header `Authorization` o envía un JWT inválido / expirado / con firma manipulada
- WHEN envía `GET /auth/me/workspaces`
- THEN el servidor SHALL responder HTTP 401
- AND el body SHALL seguir el shape de error uniforme

#### INV-6: Registrations no COMPLETED no aparecen en el listado

- GIVEN un usuario con dos registrations: una `COMPLETED` (NOTARY) y otra `IN_PROGRESS` (REAL_ESTATE)
- WHEN un cliente envía `GET /auth/me/workspaces`
- THEN el body SHALL contener únicamente la registration COMPLETED (NOTARY)
- AND la registration `IN_PROGRESS` SHALL NOT aparecer
