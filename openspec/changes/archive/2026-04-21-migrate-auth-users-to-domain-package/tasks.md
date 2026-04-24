# Tasks: migrate-auth-users-to-domain-package

## Phase 1 — Foundation

- [x] 1.1 Crear `packages/domain-auth-users/package.json` con `name: @pld-api/domain-auth-users`, `main`/`types` apuntando a `src/index.ts`, deps: `typeorm`, `jsonwebtoken`; siguiendo patrón de `packages/persistence/package.json`
- [x] 1.2 Crear `packages/domain-auth-users/tsconfig.json` extendiendo `tsconfig.base.json`, con `experimentalDecorators: true` y `emitDecoratorMetadata: true`
- [x] 1.3 Modificar `tsconfig.base.json`: agregar `"@pld-api/domain-auth-users": ["packages/domain-auth-users/src/index.ts"]` en `paths`
- [x] 1.4 Verificar: `tsc --noEmit` en la raíz pasa sin errores de alias

## Phase 2 — Package: lógica pura

- [x] 2.1 Crear `packages/domain-auth-users/src/entities/user.entity.ts` — copia literal de `libs/users/src/lib/users.entity.ts` (sin tocar columnas ni tabla `users`)
- [x] 2.2 Crear `packages/domain-auth-users/src/ports/auth.port.ts` — interfaces `LoginInput`, `LoginOutput`, `AuthPort` con métodos `login`, `verifyCredentials`, `issueToken`
- [x] 2.3 Crear `packages/domain-auth-users/src/ports/users.port.ts` — interfaces `CreateUserInput`, `CreateUserOutput`, `UsersPort` con 7 métodos según design
- [x] 2.4 Crear `packages/domain-auth-users/src/crypto/password.ts` — `hashPassword`/`verifyPassword` (scrypt + timingSafeEqual) y `generateSecurePassword` con `crypto.randomBytes` (reemplaza `Math.random()`)
- [x] 2.5 Crear `packages/domain-auth-users/src/jwt/types.ts` — `JwtPayload` shape
- [x] 2.6 Crear `packages/domain-auth-users/src/jwt/sign.ts` — `signToken(payload, secret, opts)` y `verifyToken(token, secret)` sobre `jsonwebtoken` puro (sin `@nestjs/jwt`)
- [x] 2.7 Crear `packages/domain-auth-users/src/adapters/users.adapter.ts` — implementa `UsersPort` usando `ds.getRepository(UserEntity)`; llama `hashPassword`/`generateSecurePassword` en `createUser`
- [x] 2.8 Crear `packages/domain-auth-users/src/adapters/auth.adapter.ts` — implementa `AuthPort`: busca usuario, llama `verifyPassword`, llama `signToken`
- [x] 2.9 Crear `packages/domain-auth-users/src/factory.ts` — `createAuthUsersDomain(cfg: { ds, jwtSecret, jwtExpiresIn? })` → `{ authPort, usersPort }`
- [x] 2.10 Crear `packages/domain-auth-users/src/index.ts` — re-exporta factory, ports (interfaces), `UserEntity`, `signToken`/`verifyToken`, `JwtPayload`, funciones crypto
- [x] 2.11 Verificar: ningún archivo bajo `packages/domain-auth-users/src/` contiene `import ... from '@nestjs/...'`

## Phase 3 — Gateway wiring: auth-users

- [x] 3.1 Crear `apps/auth-users/src/shared/auth/jwt-auth.guard.ts` — copia de `libs/jwt/src/guards/jwt-auth.guard.ts`
- [x] 3.2 Crear `apps/auth-users/src/shared/auth/jwt.strategy.ts` — copia local; usa `verifyToken` de `@pld-api/domain-auth-users`
- [x] 3.3 Crear `apps/auth-users/src/shared/auth/current-user.decorator.ts` — copia del decorator de `libs/jwt`
- [x] 3.4 Modificar `apps/auth-users/src/auth-users.module.ts` — llamar `createAuthUsersDomain({ ds, jwtSecret, jwtExpiresIn })`, registrar `AUTH_PORT`/`USERS_PORT` como `useValue`, importar `JwtStrategy` local, remover imports de `libs/jwt` y `libs/users`
- [x] 3.5 Modificar `apps/auth-users/src/auth/auth.controller.ts` — inyectar `AuthPort` via `@Inject('AUTH_PORT')`; eliminar dependencia de `auth.adapter.ts`
- [x] 3.6 Modificar `apps/auth-users/src/users/users.controller.ts` — inyectar `UsersPort` via `@Inject('USERS_PORT')`; cambiar import de `JwtAuthGuard` a `../shared/auth/jwt-auth.guard`; eliminar dependencia de `users.adapter.ts`
- [x] 3.7 Eliminar `apps/auth-users/src/auth/auth.adapter.ts` y `apps/auth-users/src/users/users.adapter.ts`
- [x] 3.8 Verificar: `pnpm tsc --noEmit -p apps/auth-users/tsconfig.app.json` pasa sin errores; grep checks limpios

## Phase 4 — Gateway wiring: cross — SKIPPED

`apps/cross` no importa `@pld-api/jwt` hoy (endpoint `POST /crear-beneficiario` abierto, sin guard). Duplicar guard/strategy ahí sería trabajo especulativo. La app será absorbida en Fase 5 del plan mayor cuando `domain-participants` esté migrado. Phases 4.1–4.5 omitidas conscientemente.

## Phase 5 — Rewire participantes

- [x] 5.1 Modificar `apps/auth-users/src/participants/persona-fisica/participants.controller.ts` — cambiar import desde `@pld-api/jwt` a `../../shared/auth/jwt-auth.guard` + `../../shared/auth/current-user.decorator`
- [x] 5.2 Modificar `apps/auth-users/src/participants/persona-moral/participants.controller.ts` — idem
- [x] 5.3 Modificar `apps/auth-users/src/participants/fideicomiso/fideicomiso.controller.ts` — idem
- [x] 5.4 Modificar `apps/auth-users/src/participants/anexo-7/anexo-7.controller.ts` — idem
- [x] 5.5 Verificar en `apps/cross/src/`: si hay importers de `@pld-api/jwt` repetir el rewire apuntando a `apps/cross/src/shared/auth/`
- [x] 5.6 Verificar: `grep -r "@pld-api/jwt" apps/` NO devuelve resultados

## Phase 6 — Cleanup de libs

- [x] 6.1 Vaciar `libs/users/src/lib/users.entity.ts` — contenido `export {};`
- [x] 6.2 Vaciar `libs/users/src/lib/users.service.ts` — contenido `export {};`
- [x] 6.3 Vaciar `libs/users/src/lib/users.module.ts` — contenido `export {};`
- [x] 6.4 Modificar `libs/users/src/index.ts` — solo re-exporta `users.types.ts` (tipos de participantes)
- [x] 6.5 Modificar `libs/jwt/src/guards/jwt-auth.guard.ts` — lanza `throw new Error('import from apps/<gateway>/src/shared/auth')`
- [x] 6.6 Modificar `libs/jwt/src/index.ts` — solo re-exporta `signToken`/`verifyToken`/`JwtPayload` desde `@pld-api/domain-auth-users`
- [x] 6.7 Vaciar `libs/jwt/src/lib/jwt.service.ts`, `jwt.strategy.ts`, `jwt.module.ts` — contenido `export {};`
- [x] 6.8 Vaciar `libs/core/src/services/password.function.ts` — contenido `export {};`
- [x] 6.9 Verificar: `tsc --noEmit` y `nx run-many --target=build` pasan sin errores

## Phase 7 — Testing — SKIPPED

El smoke test de solo-shape (`typeof x === 'function'`) resultó burdo — no prueba comportamiento real (hash/verify round-trip, JWT sign/verify, login con credenciales inválidas). Se descartó la implementación y los artefactos (`src/__tests__/`, `jest.config.ts`, `tsconfig.spec.json`) se eliminaron. Se difiere a Fase 5 del plan mayor (`Setup de tests mínimo`) para diseñar una estrategia de tests coherente para todos los paquetes en lugar de tests tautológicos aislados.

## Phase 8 — Verificación manual

- [x] 8.1 Login válido → HTTP 201 + `{ token }` ✅
- [x] 8.2 system-users sin token → HTTP 401 ✅
- [x] 8.3 system-users con Bearer → HTTP 201 + user + password generado ✅
- [x] 8.4 `generateSecurePassword` usa `crypto.randomBytes` (verificado en `packages/domain-auth-users/src/crypto/password.ts`) ✅

**Bug encontrado y fixeado durante verificación**: `TypeOrmModule.forFeature([UserEntity])` faltaba en `auth-users.module.ts`. Sin él, el DataSource no conocía la entity del paquete (`autoLoadEntities: true` solo descubre las registradas vía `forFeature`). Fix: agregar import de `UserEntity` y el `forFeature([UserEntity])` al arreglo de imports. Una línea, confirmado operativo.
