# Spec: separar-domain-infra-auth-users

## 1. Estructura objetivo del paquete

Después de la refactor, `packages/domain-auth-users/src/` tiene la siguiente estructura:

```
packages/domain-auth-users/src/
├── index.ts                            ← sin cambios en símbolos exportados
├── ports/
│   ├── auth.port.ts                    ← dominio puro (sin cambios)
│   └── users.port.ts                   ← dominio puro (sin cambios)
├── crypto/
│   ├── password.ts                     ← dominio puro (sin cambios)
│   └── password-policy.ts              ← dominio puro (sin cambios)
├── jwt/
│   ├── types.ts                        ← dominio puro (sin cambios)
│   └── jti-store.ts                    ← solo la interfaz JtiStore + constante JTI_STORE
└── infrastructure/
    ├── factory.ts                      ← movido desde src/factory.ts
    ├── entities/
    │   └── user.entity.ts              ← movido desde src/entities/user.entity.ts
    ├── adapters/
    │   ├── auth.adapter.ts             ← movido desde src/adapters/auth.adapter.ts
    │   └── users.adapter.ts            ← movido desde src/adapters/users.adapter.ts
    └── jwt/
        ├── sign.ts                     ← movido desde src/jwt/sign.ts
        └── jti-store.impl.ts           ← InMemoryJtiStore extraído de src/jwt/jti-store.ts
```

## 2. Reglas de capas

### 2.1 Capa de dominio

Archivos que pertenecen al dominio:

| Archivo | Importaciones permitidas |
|---|---|
| `ports/auth.port.ts` | Tipos locales del paquete, tipos de TS estándar |
| `ports/users.port.ts` | Tipos locales del paquete, tipos de TS estándar |
| `crypto/password.ts` | Módulos built-in de Node.js (`crypto`) |
| `crypto/password-policy.ts` | Tipos locales del paquete, tipos de TS estándar |
| `jwt/types.ts` | Tipos de TS estándar |
| `jwt/jti-store.ts` | Tipos de TS estándar |

**Prohibido en la capa de dominio**: ningún archivo listado arriba puede importar `typeorm`, `jsonwebtoken`, ni ninguna otra dependencia de infraestructura. Esta invariante es verificable estáticamente.

### 2.2 Capa de infraestructura

Archivos que pertenecen a `infrastructure/`:

| Archivo | Dependencias externas permitidas |
|---|---|
| `infrastructure/entities/user.entity.ts` | `typeorm` |
| `infrastructure/adapters/auth.adapter.ts` | `typeorm`, `jsonwebtoken`, tipos del dominio |
| `infrastructure/adapters/users.adapter.ts` | `typeorm`, tipos del dominio |
| `infrastructure/jwt/sign.ts` | `jsonwebtoken`, tipos del dominio |
| `infrastructure/jwt/jti-store.impl.ts` | Tipos del dominio (`JtiStore`) |
| `infrastructure/factory.ts` | `typeorm` (solo `DataSource`), tipos e implementaciones de `infrastructure/` |

La capa de infraestructura **puede** importar desde la capa de dominio del mismo paquete. La capa de dominio **nunca** importa desde `infrastructure/`.

### 2.3 Invariante principal

> Ningún archivo bajo `ports/`, `crypto/`, o `jwt/` (excepto `infrastructure/jwt/`) contiene una importación de `typeorm` o `jsonwebtoken`, ni directa ni transitiva.

## 3. Separación de jti-store

### 3.1 Estado actual

`src/jwt/jti-store.ts` contiene dos cosas distintas:
- La interfaz `JtiStore` — contrato de dominio puro
- La clase `InMemoryJtiStore` — implementación concreta (infraestructura)
- La constante `JTI_STORE` — token de inyección de NestJS

### 3.2 Estado objetivo

**`src/jwt/jti-store.ts`** (dominio — sin cambio de ruta):
- Exporta únicamente la interfaz `JtiStore`
- Exporta la constante `JTI_STORE` (token de inyección; es un string sin dependencias)
- No contiene ninguna clase de implementación

**`src/infrastructure/jwt/jti-store.impl.ts`** (infraestructura — nuevo archivo):
- Contiene la clase `InMemoryJtiStore`
- Implementa la interfaz `JtiStore` importada desde `../../jwt/jti-store`
- No agrega dependencias externas nuevas

### 3.3 Re-exportación

`index.ts` re-exporta `InMemoryJtiStore` desde su nueva ubicación en `infrastructure/jwt/jti-store.impl.ts`. El símbolo exportado al exterior del paquete no cambia.

## 4. Contrato de index.ts

Los símbolos públicos exportados por `index.ts` no cambian. Solo cambian los paths de origen internos en las sentencias `export ... from '...'`.

Símbolos que deben seguir exportándose con el mismo nombre y forma:

| Símbolo | Tipo |
|---|---|
| `createAuthUsersDomain` | función |
| `AuthUsersDomainConfig` | tipo |
| `AuthPort`, `LoginInput`, `LoginOutput`, `AvailableWorkspace` | tipos |
| `UsersPort`, `CreateUserInput`, `CreateUserOutput`, `CurrentUserDto`, `MarkPasswordChangedInput` | tipos |
| `UserEntity` | clase |
| `signToken`, `verifyToken` | funciones |
| `JwtPayload`, `JwtWorkspace`, `TempTokenPayload`, `UserRole`, `ActivityType`, `LoginResult` | tipos |
| `SignOptions` | tipo |
| `InMemoryJtiStore`, `JTI_STORE` | clase y constante |
| `JtiStore` | tipo |
| `hashPassword`, `verifyPassword`, `generateSecurePassword` | funciones |
| `PASSWORD_POLICY`, `PASSWORD_POLICY_MESSAGES`, `validatePasswordStrength` | constantes/función |
| `PasswordPolicyRule`, `PasswordValidationResult` | tipos |

**No se eliminan ni renombran símbolos**. No se agregan símbolos nuevos como parte de este change.

## 5. Consumidores en apps/auth-users

Los siguientes módulos de `apps/auth-users/src/` importan `UserEntity` u otros símbolos del paquete:

| Archivo | Símbolos importados |
|---|---|
| `auth-users.module.ts` | `UserEntity` |
| `password-recovery/password-recovery.module.ts` | `UserEntity` |
| `password-recovery/password-recovery.adapter.ts` | `UserEntity` |
| `admin/registration/registration.module.ts` | `UserEntity` |
| `admin/registration/registration.adapter.ts` | `UserEntity`, `ActivityType` |
| `admin/registration/registration.service.ts` | `UserEntity`, `generateSecurePassword`, `hashPassword` |
| `registration/auxiliaries/auxiliaries.adapter.ts` | `UserEntity` |
| `registration/auxiliaries/auxiliaries.module.ts` | `UserEntity` |
| `registration/auxiliaries/auxiliaries.service.ts` | `UserEntity`, `generateSecurePassword`, `hashPassword` |

**Requisito**: todos estos archivos continúan importando desde `@pld-api/domain-auth-users` sin cambios en sus propias sentencias `import`. El re-enrutamiento interno del paquete es transparente para ellos.

Si algún archivo en `apps/auth-users/src/` importa directamente desde un subpath interno del paquete (p. ej. `@pld-api/domain-auth-users/src/entities/user.entity`), ese import debe corregirse para apuntar al barrel `@pld-api/domain-auth-users`.

## 6. AdminAdapter y queries de backoffice

### 6.1 Fuera de UsersPort

El método `listWorkspaceAdminsPaginated` — y cualquier query de backoffice con lógica de paginación, filtros administrativos o joins multi-tabla — **no pertenece a `UsersPort`**.

`UsersPort` es un contrato de dominio que define operaciones sobre usuarios individuales o colecciones pequeñas. Las queries de backoffice tienen requisitos distintos (paginación, joins, proyecciones específicas) que no son parte del modelo de dominio.

### 6.2 Lugar correcto

Estas queries viven en `apps/auth-users/src/admin/users/admin-users.adapter.ts` (archivo a crear si no existe):

- Inyecta `DataSource` directamente via NestJS DI
- Construye queries con QueryBuilder o SQL crudo según necesidad
- No implementa ninguna interfaz de `UsersPort`
- Es consumido por el servicio o controlador de admin correspondiente

### 6.3 Invariante

> `UsersPort` en `packages/domain-auth-users/src/ports/users.port.ts` no declara métodos de listado paginado con filtros de backoffice. Si un método requiere `DataSource` para funcionar, no pertenece a un port de dominio.
