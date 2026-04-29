# Auxiliary Registration Specification

## Purpose

Endpoint único `POST /registration/auxiliaries` para que sujetos obligados (`NOTARY`, `REAL_ESTATE`) registren usuarios con role `AUXILIARY` asociados a su cuenta. Una sola request, una sola transacción, sin wizard backend. Flujo y diagrama en [docs/FLUJO_REGISTRO_AUXILIARY.md](../../../../docs/FLUJO_REGISTRO_AUXILIARY.md).

## Requirements

### Requirement: Autenticación y autorización

El endpoint `POST /registration/auxiliaries` MUST requerir JWT válido y MUST permitir solo roles `NOTARY` o `REAL_ESTATE`.

#### Scenario: Sin JWT

- GIVEN cliente sin header `Authorization` válido
- WHEN envía `POST /pld-api/auth-users/registration/auxiliaries`
- THEN responde HTTP 401 Unauthorized

#### Scenario: JWT con role no permitido

- GIVEN JWT con role `SUPERADMIN`, `AUXILIARY` o cualquier otro distinto de NOTARY/REAL_ESTATE
- WHEN envía POST con DTO válido
- THEN responde HTTP 403 Forbidden

#### Scenario: Defensa en profundidad sobre parent_user

- GIVEN JWT con role `NOTARY` pero el row `users` correspondiente al `sub` del JWT tiene `role` distinto (estado inconsistente)
- WHEN se procesa la request
- THEN el service MUST re-validar `users.role` y responder HTTP 403 si no es NOTARY/REAL_ESTATE

### Requirement: Validación del DTO

El DTO MUST validar requeridos y formatos vía `class-validator`. Cuerpo inválido → HTTP 400.

| Campo | Tipo | Requerido | Validación |
|---|---|---|---|
| `firstName` | string | ✅ | `IsString`, `Length(1,120)` |
| `paternalSurname` | string | ✅ | `IsString`, `Length(1,120)` |
| `maternalSurname` | string | ✅ | `IsString`, `Length(1,120)` |
| `rfc` | string | ✅ | regex RFC del módulo registration |
| `phone` | string | ✅ | `IsString`, `Length(7,30)` |
| `email` | string | ✅ | `IsEmail` |
| `address.state` | string | ✅ | `IsString`, no vacío |
| `address.postalCode` | string | ✅ | `IsString`, regex CP MX (5 dígitos) |
| `address.municipality` | string | ✅ | `IsString`, no vacío |
| `address.neighborhood` | string | ✅ | `IsString`, no vacío |
| `address.street` | string | ✅ | `IsString`, no vacío |
| `address.exteriorNumber` | string | ✅ | `IsString`, no vacío |
| `address.interiorNumber` | string | ❌ | `IsOptional`, `IsString` |

El DTO MUST NOT aceptar `cellphone`, `country`, `road_type` ni `locality` — campos rechazados por `forbidNonWhitelisted: true`.

#### Scenario: DTO completo válido

- GIVEN JWT NOTARY/REAL_ESTATE válido
- AND DTO con todos los requeridos correctos
- WHEN envía POST
- THEN responde HTTP 201

#### Scenario: Falta un campo requerido

- GIVEN DTO sin `rfc` (o cualquier requerido)
- WHEN envía POST
- THEN responde HTTP 400 con mensaje del validator

#### Scenario: Email mal formado

- GIVEN `email = "no-es-email"`
- WHEN envía POST
- THEN responde HTTP 400

#### Scenario: Campo no whitelisted

- GIVEN body que incluye `cellphone: "5551234567"`
- WHEN envía POST
- THEN responde HTTP 400 (whitelist forbids non-allowed)

### Requirement: Email único en users

El sistema MUST rechazar registros si el `email` ya existe en `users` (cualquier role).

#### Scenario: Email duplicado

- GIVEN existe `users.email = "x@y.com"` (cualquier role, incluyendo soft-deleted opcionalmente)
- WHEN se envía POST con el mismo email
- THEN responde HTTP 409 Conflict
- AND el body incluye mensaje indicando email duplicado

### Requirement: Persistencia transaccional

El service MUST ejecutar una transacción TypeORM única que inserta tres rows: `reporting_entity_address`, `auxiliary_profile`, `users`. Si cualquiera falla, MUST hacer rollback completo.

#### Scenario: Inserts exitosos

- GIVEN DTO válido y email no duplicado
- WHEN el service ejecuta la transacción
- THEN inserta una row en `reporting_entity_address` con `country = 'México'` por default
- AND inserta una row en `auxiliary_profile` con `parent_user_id = JWT.sub`, `rfc`, `address_id`
- AND inserta una row en `users` con `role='AUXILIARY'`, `profile_type='AUXILIARY'`, `profile_id=auxiliary_profile.id`, `must_change_password=true`, `active=true`, `password_hash` bcrypt
- AND las tres rows comparten la misma transacción

#### Scenario: Rollback ante fallo en INSERT users

- GIVEN INSERT en `users` falla (ej. constraint UK email duplicado por race condition)
- WHEN la transacción aborta
- THEN no quedan rows en `reporting_entity_address` ni `auxiliary_profile` correspondientes
- AND responde HTTP 409 (o 500 si fue un fallo distinto)

### Requirement: Generación y exposición de password temporal

El service MUST generar una password aleatoria, persistir solo el hash bcrypt en `users.password_hash`, y devolverla en cleartext UNA SOLA VEZ en el response del 201.

#### Scenario: Password en response

- GIVEN registro exitoso
- WHEN responde HTTP 201
- THEN el body contiene `{ user, temporaryPassword }`
- AND `user` NO incluye `passwordHash`
- AND `temporaryPassword` está en cleartext

#### Scenario: Password no se loguea

- GIVEN registro exitoso
- WHEN se inspeccionan logs de aplicación
- THEN ni el DTO ni el `temporaryPassword` aparecen en logs

### Requirement: Login post-registro

El auxiliar recién creado MUST poder autenticarse con la password temporal vía `POST /auth/login` y obtener un JWT con `must_change_password: true`.

#### Scenario: Login con password temporal

- GIVEN un auxiliar recién registrado y su `temporaryPassword`
- WHEN hace `POST /pld-api/auth-users/auth/login` con email + temporaryPassword
- THEN responde HTTP 201 con JWT
- AND el payload del JWT (o el response) indica `must_change_password = true`

### Requirement: No side-effects fuera de scope

El endpoint MUST NOT enviar emails, MUST NOT crear rows en `registration`, `contact`, `vulnerable_activity`, `compliance_responsible`, `physical_person_profile` ni `moral_person_profile`.

#### Scenario: Tablas no afectadas

- GIVEN registro exitoso de un auxiliar
- WHEN se inspecciona la BD
- THEN no se crean rows en `registration` ni en ninguna tabla del flujo PF/PM
- AND no se invoca ningún servicio de email

### Requirement: Smoke regression del flujo PF/PM

La introducción del nuevo endpoint MUST NOT romper `POST /admin/registration` (flujo SUPERADMIN para PF/PM).

#### Scenario: Flujo PF/PM sigue verde

- GIVEN un SUPERADMIN
- WHEN ejecuta `POST /admin/registration` con `{ profileType: "INDIVIDUAL", userRole: "NOTARY", rfc: "..." }`
- THEN responde HTTP 201 con id del draft (igual que antes del cambio)
