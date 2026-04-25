# Users Specification

## Purpose

Define el contrato de gestión de usuarios del sistema: el puerto `UsersPort`, la entidad `UserEntity`, las reglas de negocio de creación y los endpoints HTTP preservados 1:1.

## Requirements

### Requirement: Puerto UsersPort

El paquete `domain-auth-users` MUST exponer una interfaz `UsersPort` con métodos para crear, buscar y gestionar usuarios. La implementación TypeORM subyacente NO MUST ser expuesta al consumidor.

| Método | Descripción |
|---|---|
| `createUser(data)` | Crea usuario, hashea password, valida unicidad |
| `getUserByEmail(email)` | Busca por email; retorna `null` si no existe |
| `getUserById(id)` | Busca por UUID; retorna `null` si no existe |
| `getUserByPhone(phone)` | Busca por teléfono; retorna `null` si no existe |

#### Scenario: createUser retorna usuario con password en texto claro (solo en creación)

- GIVEN una instancia de `UsersPort` y datos válidos de nuevo usuario
- WHEN se llama `createUser(data)`
- THEN el usuario es persistido con `passwordHash` almacenado (no el password en claro)
- AND el método retorna el objeto de usuario incluyendo el `password` en texto claro (una sola vez, para entregarlo al creador)

### Requirement: Unicidad de email

El sistema MUST rechazar la creación de un usuario cuando el email ya está registrado.

#### Scenario: Email duplicado genera error de conflicto

- GIVEN un usuario con email `existing@example.com` existe en la base de datos
- WHEN se llama `createUser` con el mismo email
- THEN el método lanza un error de conflicto (sin crear el registro)
- AND el error es propagado por el controller como HTTP 409 o equivalente de negocio

### Requirement: Hash de password con algoritmo seguro

El sistema MUST hashear los passwords usando `scrypt` con sal aleatoria y comparación timing-safe. MUST NOT almacenar passwords en texto claro.

#### Scenario: Password almacenado como hash

- GIVEN se crea un usuario con password `"Abc123!"`
- WHEN se consulta el registro persistido
- THEN el campo `passwordHash` no contiene `"Abc123!"`
- AND la función `verifyPassword(stored, "Abc123!")` retorna `true`

#### Scenario: Comparación timing-safe previene timing attacks

- GIVEN dos passwords con el mismo prefijo pero diferente sufijo
- WHEN se comparan contra el mismo hash almacenado
- THEN ambas verificaciones toman un tiempo estadísticamente similar

### Requirement: Generación de password seguro

El sistema MUST generar passwords temporales usando una fuente criptográficamente fuerte. MUST NOT usar `Math.random()` para la selección de caracteres.

#### Scenario: generateSecurePassword usa crypto.randomBytes

- GIVEN se solicita un password temporal de longitud N
- WHEN se llama `generateSecurePassword(N)`
- THEN la implementación usa `crypto.randomBytes` (o equivalente CSPRNG) para la selección de caracteres
- AND el resultado tiene exactamente N caracteres
- AND el resultado contiene caracteres del alfabeto configurado (letras + números + símbolos)

### Requirement: Endpoint crear usuario notario/inmobiliario

El sistema MUST exponer `POST /pld-api/auth-users/system-users/notario-inmobiliario` protegido con `JwtAuthGuard`, aceptando los campos del DTO correspondiente y retornando el usuario creado con el password en texto claro.

#### Scenario: Creación exitosa de usuario notario

- GIVEN un cliente autenticado (JWT válido) envía `POST /pld-api/auth-users/system-users/notario-inmobiliario` con datos válidos
- WHEN el servidor procesa la solicitud
- THEN responde HTTP 201
- AND el body incluye `{ id, nombre, email, role, ..., password: string }` donde `password` es el generado

#### Scenario: Acceso sin JWT rechazado

- GIVEN un cliente sin token envía `POST /pld-api/auth-users/system-users/notario-inmobiliario`
- WHEN el servidor procesa la solicitud
- THEN responde HTTP 401

### Requirement: Endpoint crear usuario interno/externo/auxiliar

El sistema MUST exponer `POST /pld-api/auth-users/system-users/interno-externo-auxiliar` protegido con `JwtAuthGuard`, con comportamiento simétrico al endpoint anterior.

#### Scenario: Creación exitosa de usuario interno

- GIVEN un cliente autenticado envía `POST /pld-api/auth-users/system-users/interno-externo-auxiliar` con datos válidos
- WHEN el servidor procesa la solicitud
- THEN responde HTTP 201
- AND el body incluye `{ id, nombre, email, role, ..., password: string }`

### Requirement: Entidad UserEntity — campos obligatorios

La entidad `UserEntity` MUST persitir al menos los campos de la tabla `users` tal como existen hoy.

| Campo | Tipo | Restricción |
|---|---|---|
| `id` | UUID | PK, generado |
| `nombre` | varchar(120) | NOT NULL |
| `email` | varchar(160) | UNIQUE, NOT NULL |
| `passwordHash` | varchar(255) | NOT NULL |
| `role` | enum `UserRole` | NOT NULL |
| `activo` | boolean | default true |
| `deletedAt` | timestamp | nullable, soft-delete |

#### Scenario: Registro persiste con campos completos

- GIVEN se invoca `createUser` con todos los campos requeridos
- WHEN se consulta el registro por id
- THEN todos los campos están presentes y correctos

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
