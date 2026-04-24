# Design: Migrar `auth-users` a `packages/domain-auth-users`

## Technical Approach

Crear `packages/domain-auth-users` puro (sin `@nestjs/*`). Expone una factory `createAuthUsersDomain({ ds })` que retorna `{ authPort, usersPort }`. La entidad `UserEntity` y el adapter TypeORM viven dentro del paquete; el DataSource se inyecta desde la app (patrón `@pld-api/persistence`). JWT puro (sign/verify) vive en el paquete; `JwtAuthGuard` + `JwtStrategy` se duplican simétricamente en `apps/auth-users` y `apps/cross`. `libs/jwt` y `libs/users` quedan como bridges re-exportando del paquete; eliminación física en Fase 5.

## Architecture Decisions

| Decisión | Elegida | Alternativa descartada | Rationale |
|---|---|---|---|
| Granularidad del paquete | Un paquete `domain-auth-users` | Dos paquetes (`domain-users` + `domain-auth`) | Cohesión natural: login consulta usuarios; crear usuario hashea password. Un paquete reduce overhead de configs sin perder modularidad (submódulos internos). Partir es trivial en Fase 5 si se requiere. |
| Factory | `createAuthUsersDomain({ ds: DataSource })` → `{ authPort, usersPort }` | Clases instanciadas con `new` / DI container propio | Firma consistente con `createDataSource` de `@pld-api/persistence`. Función pura, fácil de mockear, sin estado global. Los apps Nest la llaman una vez y registran los ports como `useValue` en un Provider. |
| Puertos separados vs unificado | Dos ports: `AuthPort` + `UsersPort` | `AuthUsersPort` único con todos los métodos | Un endpoint `/auth/login` consume solo `AuthPort.login`; los endpoints `/system-users/*` consumen solo `UsersPort`. Separar permite que cada controller reciba el contrato mínimo que necesita y facilita futuras particiones. |
| `UserEntity` | Copia literal `libs/users/src/lib/users.entity.ts` → `packages/domain-auth-users/src/entities/user.entity.ts` | Reescribirla o partirla | La entidad ya es TypeORM puro (sin NestJS). Mover sin tocar columnas ni tabla `users` evita riesgo de migración de datos y preserva idempotencia. |
| JWT — qué se mueve | `signToken`, `verifyToken`, `JwtPayload` → paquete (usan `jsonwebtoken` puro) | Mover también `JwtAuthGuard` + `JwtStrategy` al paquete | Guards y strategies dependen de `@nestjs/passport` y `@nestjs/common` → no pueden vivir en un paquete puro. Se duplican en cada app (mismo patrón que `HttpErrorInterceptor` en Fase 2). |
| Crypto | `hashPassword` + `verifyPassword` se mueven literal (scryptSync). `generateSecurePassword` se reescribe con `crypto.randomBytes` | Mantener `Math.random()` actual | Riesgo 1 = B del proposal: debilidad criptográfica documentada. `crypto.randomBytes` corrige sin cambiar firma pública. bcrypt NO se introduce — se mantiene `node:crypto` nativo (ya funcional, sin dep externa). |
| `libs/jwt` bridge | `libs/jwt/src/index.ts` re-exporta `signToken`/`verifyToken`/`JwtPayload` desde `@pld-api/domain-auth-users`; `JwtAuthGuard` y `JwtStrategy` quedan como stubs que lanzan `throw new Error('moved to apps/*/src/shared/auth — import locally')` | Eliminar `libs/jwt` ahora | Controllers de participantes (`persona-fisica`, `anexo-7`) aún importan `@pld-api/jwt`. Bridge evita romperlos en esta fase; Fase 5 elimina el directorio tras migrar esos imports. |
| `libs/users` bridge | `libs/users/src/index.ts` re-exporta únicamente los tipos puros de `users.types.ts` (`PerfilBasico`, `PFParticipante`, `PMParticipante`, etc.). `UserEntity`, `UsersService`, `UsersModule` se vacían | Mover también `users.types.ts` al paquete | Esos tipos pertenecen conceptualmente a participantes, no a auth-users (consumidos por `libs/auth-profiles`). Moverlos ahora violaría el alcance; se resuelven en fase `domain-participants`. |

## Data Flow

```
HTTP Client                apps/auth-users (NestJS)                 packages/domain-auth-users (puro)         TypeORM / MySQL
     │                             │                                            │                                   │
     │ POST /auth/login            │                                            │                                   │
     ├────────────────────────────►│ AuthController.login(dto)                  │                                   │
     │                             │   ↓                                        │                                   │
     │                             │ authPort.login({email, password}) ────────►│ authAdapter.login()               │
     │                             │                                            │   ↓                               │
     │                             │                                            │ usersRepo.findOne({email}) ──────►│ SELECT * FROM users
     │                             │                                            │   ◄───────────────────────────────┤
     │                             │                                            │   ↓ verifyPassword()              │
     │                             │                                            │   ↓ signToken(payload)            │
     │                             │                                  { token } │                                   │
     │ 201 { token }               │  ◄─────────────────────────────────────────│                                   │
     ◄────────────────────────────┤                                             │                                   │
```

Factory se instancia en `AuthUsersModule`:
```
createAuthUsersDomain({ ds }) → { authPort, usersPort }
→ registrados como providers { provide: 'AUTH_PORT', useValue: authPort }, idem usersPort.
```

## File Changes

| File (absoluto) | Action | Description |
|---|---|---|
| `/home/cristobal/work/pld-api/packages/domain-auth-users/package.json` | Create | `name: @pld-api/domain-auth-users`, deps: `typeorm`, `jsonwebtoken` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/index.ts` | Create | Re-exporta factory, ports, tipos, `UserEntity`, `signToken`/`verifyToken` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/factory.ts` | Create | `createAuthUsersDomain({ ds })` → `{ authPort, usersPort }` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/ports/auth.port.ts` | Create | `interface AuthPort { login(input): Promise<{ token }> }` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/ports/users.port.ts` | Create | `interface UsersPort { createUser, getUserByEmail, getUserByPhone, getUserById, updateUser, deleteUser, listUsers }` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/entities/user.entity.ts` | Create | Copia literal de `libs/users/src/lib/users.entity.ts` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/adapters/users.adapter.ts` | Create | Implementa `UsersPort` con `ds.getRepository(UserEntity)` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts` | Create | Implementa `AuthPort`: busca user, verifica password, firma token |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/crypto/password.ts` | Create | `hashPassword`, `verifyPassword` (scrypt). `generateSecurePassword` con `crypto.randomBytes` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/jwt/sign.ts` | Create | `signToken(payload, secret, opts)` y `verifyToken(token, secret)` puros sobre `jsonwebtoken` |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/jwt/types.ts` | Create | `JwtPayload` shape |
| `/home/cristobal/work/pld-api/packages/domain-auth-users/src/__tests__/factory.smoke.spec.ts` | Create | Smoke test del factory (mock DataSource) |
| `/home/cristobal/work/pld-api/tsconfig.base.json` | Modify | Agrega `"@pld-api/domain-auth-users": ["packages/domain-auth-users/src/index.ts"]` |
| `/home/cristobal/work/pld-api/apps/auth-users/src/auth/auth.controller.ts` | Modify | Inyecta `AuthPort` vía `@Inject('AUTH_PORT')`; elimina dependencia de `auth.adapter.ts` |
| `/home/cristobal/work/pld-api/apps/auth-users/src/users/users.controller.ts` | Modify | Inyecta `UsersPort`; import de `JwtAuthGuard` → ruta local `../shared/auth/jwt-auth.guard` |
| `/home/cristobal/work/pld-api/apps/auth-users/src/auth-users.module.ts` | Modify | Llama `createAuthUsersDomain({ ds })`, registra providers `AUTH_PORT` + `USERS_PORT`, importa `JwtStrategy` local |
| `/home/cristobal/work/pld-api/apps/auth-users/src/shared/auth/jwt-auth.guard.ts` | Create | Copia literal de `libs/jwt/src/guards/jwt-auth.guard.ts` |
| `/home/cristobal/work/pld-api/apps/auth-users/src/shared/auth/jwt.strategy.ts` | Create | Copia literal; usa `verifyToken` del paquete |
| `/home/cristobal/work/pld-api/apps/auth-users/src/shared/auth/current-user.decorator.ts` | Create | Copia del decorator de `libs/jwt` |
| `/home/cristobal/work/pld-api/apps/cross/src/shared/auth/jwt-auth.guard.ts` | Create | Copia simétrica |
| `/home/cristobal/work/pld-api/apps/cross/src/shared/auth/jwt.strategy.ts` | Create | Copia simétrica |
| `/home/cristobal/work/pld-api/apps/cross/src/shared/auth/current-user.decorator.ts` | Create | Copia simétrica |
| `/home/cristobal/work/pld-api/apps/auth-users/src/auth/auth.adapter.ts` | Delete | Lógica migrada al paquete |
| `/home/cristobal/work/pld-api/apps/auth-users/src/users/users.adapter.ts` | Delete | Idem |
| `/home/cristobal/work/pld-api/libs/users/src/lib/users.service.ts` | Empty | Archivo vaciado (export `{}`); físico en Fase 5 |
| `/home/cristobal/work/pld-api/libs/users/src/lib/users.entity.ts` | Empty | Vaciado; referencias redirigen al paquete |
| `/home/cristobal/work/pld-api/libs/users/src/lib/users.module.ts` | Empty | Vaciado |
| `/home/cristobal/work/pld-api/libs/users/src/index.ts` | Modify | Solo re-exporta `users.types.ts` |
| `/home/cristobal/work/pld-api/libs/core/src/services/password.function.ts` | Empty | Vaciado; re-export desde paquete para compatibilidad transitoria |
| `/home/cristobal/work/pld-api/libs/jwt/src/lib/jwt.service.ts` | Empty | Vaciado |
| `/home/cristobal/work/pld-api/libs/jwt/src/lib/jwt.strategy.ts` | Empty | Vaciado |
| `/home/cristobal/work/pld-api/libs/jwt/src/lib/jwt.module.ts` | Empty | Vaciado |
| `/home/cristobal/work/pld-api/libs/jwt/src/guards/jwt-auth.guard.ts` | Modify | Lanza `throw new Error('import from apps/<gateway>/src/shared/auth')` |
| `/home/cristobal/work/pld-api/libs/jwt/src/index.ts` | Modify | Re-exporta solo `signToken`/`verifyToken`/`JwtPayload` del paquete |

Nota: `packages/persistence` no usa `project.json` (usa solo `package.json` con `main`/`types` apuntando a `src/index.ts`). `domain-auth-users` sigue el mismo patrón — **no se crea `project.json`**.

## Interfaces / Contracts

```typescript
// packages/domain-auth-users/src/ports/auth.port.ts
export interface LoginInput { email: string; password: string; }
export interface LoginOutput { token: string; }
export interface AuthPort {
  login(input: LoginInput): Promise<LoginOutput>;
}

// packages/domain-auth-users/src/ports/users.port.ts
import type { UserEntity } from '../entities/user.entity';

export interface CreateUserInput {
  nombre: string;
  apellidoPaterno?: string;
  apellidoMaterno?: string;
  email: string;
  telefono?: string;
  role: string;
}
export interface CreateUserOutput extends Omit<UserEntity, 'passwordHash'> {
  password: string; // plaintext devuelto una sola vez
}

export interface UsersPort {
  createUser(input: CreateUserInput): Promise<CreateUserOutput>;
  getUserByEmail(email: string): Promise<UserEntity | null>;
  getUserByPhone(telefono: string): Promise<UserEntity | null>;
  getUserById(id: string): Promise<UserEntity | null>;
  updateUser(id: string, patch: Partial<UserEntity>): Promise<UserEntity>;
  deleteUser(id: string): Promise<void>; // soft
  listUsers(): Promise<UserEntity[]>;
}

// packages/domain-auth-users/src/factory.ts
import type { DataSource } from 'typeorm';
import type { AuthPort } from './ports/auth.port';
import type { UsersPort } from './ports/users.port';

export interface AuthUsersDomainConfig {
  ds: DataSource;
  jwtSecret: string;
  jwtExpiresIn?: string; // default '1d'
}

export function createAuthUsersDomain(
  cfg: AuthUsersDomainConfig
): { authPort: AuthPort; usersPort: UsersPort };
```

## Testing Strategy

| Layer | Qué se prueba | Cómo |
|---|---|---|
| Unit (smoke) | `createAuthUsersDomain({ ds })` retorna `{ authPort, usersPort }` con métodos esperados | `packages/domain-auth-users/src/__tests__/factory.smoke.spec.ts` con `DataSource` mockeado (objeto `{ getRepository: jest.fn() }`) |
| Smoke (manual) | `POST /pld-api/auth-users/auth/login` → 201 + `{ token }` | `curl -i -X POST ... -d '{"email":"admin@...","password":"..."}'` |
| Regresión | `POST /system-users/*` responde 401 sin token y 201 con `JwtAuthGuard` válido | `curl` con/sin header `x-jwt` |
| Compilación | `apps/auth-users`, `apps/cross` y paquete compilan | `nx run auth-users:build`, `nx run cross:build`, `tsc --noEmit` en el paquete |
| Crypto | `generateSecurePassword` usa `crypto.randomBytes` | Lectura directa del código migrado (verificación en `sdd-verify`) |

Alineado con el proposal: no se construye battery completa. Un solo smoke test establece el patrón factory+adapter para futuros paquetes `domain-*`.

## Migration / Rollout

1. Rama dedicada `feat/domain-auth-users`.
2. Crear paquete completo + bridges + duplicar guards. Mantener `libs/*` vaciados (no borrados).
3. Verificación manual con `curl` de los 3 endpoints (login + 2 system-users) + smoke test.
4. Merge → redeploy normal. Sin feature flags (el cambio es invisible para el cliente: endpoints 1:1).

**Revert**: `git revert` del commit restaura `libs/users`, `libs/jwt`, `libs/core/services/password.function.ts` y los adapters. El paquete `packages/domain-auth-users` desaparece del árbol; alias en `tsconfig.base.json` también retrocede. Sin datos persistidos afectados (misma tabla `users`, mismo schema).

## Open Questions

Ninguna. Todas las decisiones del riesgo (JWT = A, Crypto = B) y de granularidad (paquete único = A) están cerradas en el proposal.
