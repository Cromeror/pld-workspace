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

### Requirement: JwtAuthGuard duplicado en cada app

`JwtAuthGuard` y `JwtStrategy` MUST existir como copias locales en `apps/auth-users/src/shared/auth/` y `apps/cross/src/shared/auth/`. Ninguna app SHOULD importar estos símbolos desde `@pld-api/jwt` tras esta fase.

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
