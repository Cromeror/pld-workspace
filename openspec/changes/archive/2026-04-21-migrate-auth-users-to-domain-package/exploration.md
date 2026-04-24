# Exploration: migrate-auth-users-to-domain-package

## 1. Estado actual — código a migrar

### `apps/auth-users/src/auth/`
| Archivo | Qué hace | NestJS? |
|---|---|---|
| `auth.controller.ts` | Controller HTTP: `POST auth/login` | ACOPLADO |
| `auth.adapter.ts` | Lógica de login: busca usuario, verifica password, firma JWT | ACOPLADO (`@Injectable`, `UnauthorizedException`) |
| `dto/login.dto.ts` | DTO con validación (email + password) via class-validator + Swagger | ACOPLADO (Swagger) |
| `dto/create-auth.dto.ts` | Stub vacío — sin uso real | — |
| `dto/update-auth.dto.ts` | Stub vacío | — |
| `entities/auth.entity.ts` | Stub vacío (`export class Auth {}`) — sin TypeORM | — |

### `apps/auth-users/src/users/`
| Archivo | Qué hace | NestJS? |
|---|---|---|
| `users.controller.ts` | Controller HTTP: 2 endpoints de creación de usuarios, guardado con `JwtAuthGuard` | ACOPLADO |
| `users.adapter.ts` | Lógica de negocio: valida unicidad email/phone, genera password, hashea, crea usuario | ACOPLADO (`@Injectable`, excepciones NestJS) |
| `dto/create-user.dto.ts` | DTOs para notario/inmobiliario e interno/externo con class-validator + Swagger | ACOPLADO (Swagger, `@nestjs/mapped-types`) |
| `dto/update-user.dto.ts` | PartialType wrappers de los DTOs de creación | ACOPLADO |
| `entities/user.entity.ts` | Stub vacío (`export class User {}`) — sin TypeORM | — |

### `libs/users/`
| Archivo | Qué hace | NestJS? |
|---|---|---|
| `src/lib/users.entity.ts` | `@Entity('users')` con TypeORM: id UUID, nombre, email, passwordHash, role, activo, soft-delete | ACOPLADO (TypeORM decoradores) |
| `src/lib/users.service.ts` | CRUD: createUser, getUserByEmail, getUserByPhone, getUserById, updateUser, deleteUser (soft), listUsers | ACOPLADO (`@Injectable`, `@InjectRepository`) |
| `src/lib/users.module.ts` | `UsersModule` NestJS que registra `UserEntity` con TypeOrmModule | ACOPLADO |
| `src/lib/users.types.ts` | Interfaces/tipos puros: `PerfilBasico`, `PFParticipante`, `PMParticipante`, etc. | **PURO** |
| `src/index.ts` | Re-exports de todo lo anterior | — |

### `libs/jwt/`
| Archivo | Qué hace | NestJS? |
|---|---|---|
| `src/lib/jwt.service.ts` | Wrapper de `@nestjs/jwt` — sign/verify async | ACOPLADO |
| `src/lib/jwt.strategy.ts` | Passport JWT strategy — extrae token del header `x-jwt` o Bearer | ACOPLADO (Passport + NestJS) |
| `src/lib/jwt.module.ts` | `CustomJwtModule`: registra JwtModule, PassportModule, JwtService, JwtStrategy | ACOPLADO |
| `src/guards/jwt-auth.guard.ts` | `JwtAuthGuard extends AuthGuard('jwt')` | ACOPLADO |
| `src/decorators/current-user.decorator.ts` | `@CurrentUser()` param decorator | ACOPLADO |

### `libs/core/src/services/password.function.ts`
- **PURO** — solo depende de Node.js `crypto` nativo (sin bcrypt, sin NestJS).
- Exporta: `generateSecurePassword(length)`, `hashPassword(password)`, `verifyPassword(stored, candidate)`.
- Algoritmo: scryptSync (salt:derived hex, timing-safe equal).

---

## 2. Entidad TypeORM

**`UserEntity`** — `libs/users/src/lib/users.entity.ts` — tabla `users`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | UUID PK (generated) | |
| `nombre` | varchar(120) | |
| `apellidoPaterno` | varchar(120), nullable | |
| `apellidoMaterno` | varchar(120), nullable | |
| `email` | varchar(160) | unique index `ux_users_email` |
| `telefono` | varchar(30), nullable | |
| `passwordHash` | varchar(255) | |
| `lastLoginAt` | timestamp, nullable | |
| `activo` | boolean, default true | |
| `role` | varchar → enum `UserRole` | de `@pld-api/catalogs` |
| `createdAt` / `updatedAt` | timestamps auto | |
| `deletedAt` | timestamp nullable | soft-delete |

Sin relaciones TypeORM con otras entidades. No hay join/FK declarado.

---

## 3. Endpoints HTTP vigentes

Prefix app: `/pld-api/auth-users` (confirmado por convención; controller prefix en código: `auth` y `system-users`).

| Método | Path completo | Guard | Request DTO | Response shape |
|---|---|---|---|---|
| POST | `/pld-api/auth-users/auth/login` | ninguno | `LoginDto` {email, password} | `{ token: string }` |
| POST | `/pld-api/auth-users/system-users/notario-inmobiliario` | `JwtAuthGuard` | `CreateUserNotarioInmobiliarioDto` | `{ id, nombre, email, role, ..., password: string }` |
| POST | `/pld-api/auth-users/system-users/interno-externo-auxiliar` | `JwtAuthGuard` | `CreateUserInternoExternoDto` | `{ id, nombre, email, role, ..., password: string }` |

---

## 4. Dependencias entrantes (importadores de `libs/users` y `libs/jwt`)

**`@pld-api/users`** importado en:
- `apps/auth-users/src/auth/auth.adapter.ts`
- `apps/auth-users/src/users/users.adapter.ts`
- `libs/auth-profiles/src/lib/auth-profiles.types.ts` (solo tipos, no runtime)

**`@pld-api/jwt`** importado en:
- `apps/auth-users/src/users/users.controller.ts`
- `apps/auth-users/src/auth-users.module.ts`
- `apps/auth-users/src/participants/persona-fisica/participants.controller.ts`
- `apps/auth-users/src/participants/anexo-7/anexo-7.controller.ts`

**Nota crítica**: `libs/jwt` es consumido por controllers de participantes — no solo por auth/users. Desacoplar `JwtAuthGuard` afectaría esos controllers también.

---

## 5. Clasificación PURO / ACOPLADO

| Pieza | Clasificación | Razón |
|---|---|---|
| `password.function.ts` | **PURO** | Solo `node:crypto` |
| `users.types.ts` (PerfilBasico, etc.) | **PURO** | Interfaces TypeScript puras |
| `auth.entity.ts` (stub vacío) | — | Sin código real |
| `users.entity.ts` (UserEntity) | ACOPLADO | Decoradores TypeORM (no NestJS pero requiere reflexión) |
| `users.service.ts` | ACOPLADO | `@Injectable`, `@InjectRepository`, NestJS DI |
| `users.module.ts` | ACOPLADO | NestJS Module + TypeOrmModule |
| `auth.adapter.ts` | ACOPLADO | `@Injectable`, excepciones NestJS |
| `users.adapter.ts` | ACOPLADO | `@Injectable`, excepciones NestJS |
| `jwt.service.ts` | ACOPLADO | `@Injectable`, envuelve `@nestjs/jwt` |
| `jwt.strategy.ts` | ACOPLADO | Passport + NestJS |
| `jwt.module.ts` | ACOPLADO | NestJS Module |
| `JwtAuthGuard` | ACOPLADO | `AuthGuard` de Passport/NestJS |
| `@CurrentUser` decorator | ACOPLADO | `ExecutionContext` NestJS |
| `LoginDto` / `CreateUser*Dto` | ACOPLADO | Swagger decorators (`@ApiProperty`) |

---

## 6. Crypto — detalle

**Archivo**: `libs/core/src/services/password.function.ts`  
**Dependencia**: `node:crypto` (built-in, sin paquetes externos — no bcrypt)  
**Funciones**:
- `generateSecurePassword(length: number): string` — genera con `Math.random()` (nota: no usa `crypto.randomBytes` para selección, solo para el seed implícito — riesgo menor de predictibilidad)
- `hashPassword(password: string): string` — scryptSync 64-byte, formato `salt:derived`
- `verifyPassword(stored: string, candidate: string): boolean` — timing-safe via `crypto.timingSafeEqual`

Exportado por `libs/core/src/index.ts` como parte de `@pld-api/core`.

---

## 7. Riesgos conocidos

1. **`generateSecurePassword` usa `Math.random()`**, no `crypto.randomBytes` para la selección de caracteres — débil criptográficamente. Debería corregirse al migrar.
2. **`libs/jwt` es consumido por controllers de participantes** — moverlo o eliminarlo rompe esos controllers. Se debe mantener `JwtAuthGuard` accesible o re-exportar desde la app.
3. **Sin tests** — `passWithNoTests: true` oculta ausencia total. La migración no tiene red de seguridad.
4. **`users.types.ts` es enorme** (PerfilBasico + todos los tipos PF/PM/Fideicomiso/BC/PEP) — no todo es de "usuarios", parte pertenece a participantes. Riesgo de migrar demasiado o muy poco.
5. **`libs/auth-profiles`** tiene tipos que importan `@pld-api/users` — si se mueve `users.types.ts` a `domain-auth-users`, `auth-profiles` necesita actualizar su import.
6. **Primer paquete con acceso real a BD** — `packages/persistence` ya tiene `createDataSource` helper, pero no hay ejemplo previo de `domain-*` con TypeORM directo. Patrón a establecer con cuidado.

---

## 8. Comparación de enfoques

### Enfoque A: Un solo paquete `domain-auth-users`
Pros: un solo lugar para el contrato, factory única `createAuthUsersDomain({ ds })`, menor overhead de tsconfig/package.json. Cohesión natural (login siempre necesita acceso a usuarios).  
Cons: mezcla responsabilidades de autenticación y gestión de usuarios en un paquete; si en el futuro se separan apps, hay que volver a partir.  
Esfuerzo: **Medio**

### Enfoque B: Dos paquetes `domain-users` + `domain-auth`
Pros: separación de responsabilidades, `domain-auth` depende de `domain-users` de forma explícita, más alineado con DDD.  
Cons: overhead de dos paquetes con sus configs; `domain-auth` necesita `domain-users` como dependencia — circular si no se modela bien; complejidad mayor para el primer paquete con BD.  
Esfuerzo: **Alto**

### Enfoque C: Solo lógica pura (crypto + tipos), repositorios en la app
Pros: mínimo riesgo, sin tocar TypeORM en packages, los adapters existentes casi no cambian.  
Cons: no cumple la meta del change (extraer lógica de negocio real a un paquete sin NestJS); la lógica de creación de usuario sigue acoplada a NestJS; deja la deuda intacta.  
Esfuerzo: **Bajo**

---

## 9. Recomendación

**Enfoque A** — un solo paquete `packages/domain-auth-users`. El dominio de autenticación y usuarios está fuertemente acoplado (login requiere buscar usuario; crear usuario requiere hashear password). Separarlos ahora agrega complejidad sin beneficio inmediato. El paquete puede internamente tener submódulos `users/` y `auth/` con interfaces separadas, dejando abierta la opción de partir en Fase 5 si se necesita. Usar `createDataSource` de `@pld-api/persistence` como patrón ya disponible. Corregir `generateSecurePassword` para usar `crypto.randomBytes` al migrar.

---

## Ready for Proposal

Yes. Alcance claro, entidades y endpoints identificados, riesgos documentados. El orquestador puede proceder con `sdd-propose`.
