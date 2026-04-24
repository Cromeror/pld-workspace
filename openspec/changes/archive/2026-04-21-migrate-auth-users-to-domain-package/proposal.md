# Proposal: Migrar `auth-users` a `packages/domain-auth-users`

## Intent

Extraer la lógica de negocio de autenticación y gestión de usuarios — hoy dispersa en `apps/auth-users/src/auth`, `apps/auth-users/src/users`, `libs/users`, `libs/jwt` y `libs/core/services/password.function.ts` — a un paquete puro bajo `packages/`, sin `@nestjs/*` ni decoradores. Alineado con §4.Fase 4.1 del plan de reorganización del monorepo. Es el primer paquete `domain-*` con acceso real a BD y establece el patrón para las siguientes subfases (participants, etc.).

## Scope

### In Scope
- Crear `packages/domain-auth-users` con: contratos (interfaces), `UserEntity` TypeORM, adapter TypeORM, factory `createAuthUsersDomain({ ds })`, crypto (`hashPassword`, `verifyPassword` basados en `scrypt` + `timingSafeEqual`; `generateSecurePassword`), lógica pura JWT (sign/verify).
- Reemplazar controllers de `apps/auth-users/src/{auth,users}` para consumir la interfaz expuesta por el paquete en lugar de los `*.adapter.ts` actuales.
- Duplicar `JwtAuthGuard` + `JwtStrategy` en `apps/auth-users/src/shared/auth/` y `apps/cross/src/shared/auth/` (mismo patrón simétrico que `HttpErrorInterceptor` en Fase 2).
- Rewire de los controllers de participantes (`persona-fisica`, `anexo-7`) que hoy importan `@pld-api/jwt` para que consuman el guard local del gateway.
- Registrar alias `@pld-api/domain-auth-users` en `tsconfig.base.json`.
- Vaciar código migrado en `libs/users`, `libs/core/services/password.function.ts`, `libs/jwt` — eliminación física es Fase 5.

### Out of Scope
- Renombrar `auth-users` → `gateway` (Fase 5.2).
- Simplificar prefix `/pld-api/auth-users` → `/pld-api` (Fase 5.1).
- Eliminar físicamente `libs/` (Fase 5).
- Migrar `domain-participants` (siguiente subfase).
- Tests unitarios exhaustivos — solo 1 smoke test del patrón factory+adapter.

## Approach

**Decisión A** — un solo paquete `packages/domain-auth-users` con submódulos internos `users/`, `auth/`, `crypto/`, `jwt/`. Cohesión natural (login requiere buscar usuario). El paquete expone interfaces (contrato) y una implementación TypeORM; los controllers consumen la interfaz vía factory `createAuthUsersDomain({ ds })`.

**Decisión riesgo libs/jwt — A**: mover lógica pura (firma/verify de JWT) al paquete. `JwtAuthGuard` + `JwtStrategy` se duplican en cada app que los consume (mismo patrón que `HttpErrorInterceptor`). `libs/jwt` queda como puente vacío; su eliminación es Fase 5.

## Affected Areas

| Area | Impact | Description |
|---|---|---|
| `packages/domain-auth-users/` | New | Paquete puro: contratos, entity, adapter, factory, crypto, JWT puro |
| `apps/auth-users/src/auth/auth.controller.ts` | Modified | Consume interfaz del paquete via factory |
| `apps/auth-users/src/users/users.controller.ts` | Modified | Idem |
| `apps/auth-users/src/auth/auth.adapter.ts` | Removed | Lógica migrada al paquete |
| `apps/auth-users/src/users/users.adapter.ts` | Removed | Idem |
| `apps/auth-users/src/shared/auth/{jwt-auth.guard,jwt.strategy}.ts` | New | Copia local |
| `apps/cross/src/shared/auth/{jwt-auth.guard,jwt.strategy}.ts` | New | Copia local simétrica |
| `apps/auth-users/src/auth-users.module.ts` | Modified | Registra factory + guard/strategy locales |
| `libs/users/`, `libs/jwt/`, `libs/core/services/password.function.ts` | Emptied | Se vacían; eliminación física en Fase 5 |
| `tsconfig.base.json` | Modified | Añade alias `@pld-api/domain-auth-users` |

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Login cambia status o shape | Low | Contrato de login preservado 1:1; validar con `curl` que sigue en HTTP 201 y `{ token }` |
| `generateSecurePassword` con `Math.random()` deja debilidad criptográfica al migrar | Med | **Decisión B**: reemplazar por `crypto.randomBytes` al mover. Firma preservada, cambio aislado |
| Controllers de participantes importan `@pld-api/jwt` y se rompen | High | Rewire a guard local del gateway (`apps/auth-users/src/shared/auth/`) dentro de este change; `libs/jwt` puede vaciarse sin bridge |
| Primer paquete con TypeORM real — patrón no establecido | Med | Usar `createDataSource` de `@pld-api/persistence` como fundamento; documentar factory en el paquete |

## Rollback Plan

1. `git revert` del commit de migración.
2. Restaurar adapters y `libs/` desde el revert.
3. Validar `POST /auth/login` → 201 con JWT.
4. Remover alias `@pld-api/domain-auth-users` de `tsconfig.base.json` si quedó colgado.

## Dependencies

- `@pld-api/persistence` con `createDataSource` operativo.
- `@pld-api/shared-types`, `@pld-api/shared-errors`, `@pld-api/contracts` disponibles.
- Fase 2 (`consolidate-catalogs-into-auth-users`) archivada — patrón de duplicación simétrica establecido.

## Success Criteria

- [ ] `packages/domain-auth-users` existe, compila y no importa `@nestjs/*` ni usa decoradores NestJS.
- [ ] `POST /pld-api/auth-users/auth/login` responde HTTP 201 con `{ token }` — regresión cero.
- [ ] Los 2 endpoints `POST /system-users/*` responden con el mismo shape y siguen exigiendo `JwtAuthGuard`.
- [ ] `generateSecurePassword` usa `crypto.randomBytes` (verificado por lectura del código migrado).
- [ ] `hashPassword`/`verifyPassword` usan `scrypt` + `timingSafeEqual` (preserva comportamiento actual).
- [ ] `JwtAuthGuard` + `JwtStrategy` residen en `apps/auth-users/src/shared/auth/` y `apps/cross/src/shared/auth/`.
- [ ] Controllers de participantes (`persona-fisica`, `anexo-7`) consumen el guard local, no `@pld-api/jwt`.
- [ ] `libs/users`, `libs/jwt`, `libs/core/services/password.function.ts` vaciados (no eliminados).
- [ ] Alias `@pld-api/domain-auth-users` registrado en `tsconfig.base.json`.
- [ ] 1 smoke test mínimo valida el patrón factory → adapter → login.
