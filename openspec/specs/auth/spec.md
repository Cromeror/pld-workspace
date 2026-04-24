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
