# Delta for Auth

## ADDED Requirements

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

## MODIFIED Requirements

### Requirement: Login actualiza lastLoginAt antes de emitir el token

(Previously: `POST /auth/login` solo emitía el JWT sin actualizar `lastLoginAt`.)

El sistema MUST actualizar `users.last_login_at = NOW()` después de verificar las credenciales y ANTES de firmar el JWT. La operación MUST completarse dentro de la misma unidad de trabajo que la autenticación. El contrato HTTP de `POST /auth/login` (HTTP 201 + `{ token }`) MUST NOT cambiar.

#### INV-7: lastLoginAt se actualiza en cada login exitoso

**Given** un usuario activo con credenciales correctas,
**When** el cliente envía `POST /auth/login`,
**Then** el servidor SHALL responder HTTP 201 con `{ token: string }`,
**AND** el campo `last_login_at` del usuario en BD SHALL reflejar el timestamp del login recién efectuado.
