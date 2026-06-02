# Tasks: separar-domain-infra-auth-users

## Paso 1: Separar jti-store (interfaz vs implementación)

- [x] 1.1 Crear el directorio `packages/domain-auth-users/src/infrastructure/jwt/` (mkdir -p)
- [x] 1.2 Crear `packages/domain-auth-users/src/infrastructure/jwt/jti-store.impl.ts` con la clase `InMemoryJtiStore` extraída de `jwt/jti-store.ts`: incluye el bloque `setInterval`, el método `consume`, y el comentario `// TODO: reemplazar por adapter Redis...`; importa la interfaz con `import type { JtiStore } from '../../jwt/jti-store'`
- [x] 1.3 Editar `packages/domain-auth-users/src/jwt/jti-store.ts` para dejar únicamente: `export interface JtiStore { consume(jti: string, expiresAt: number): boolean }` y `export const JTI_STORE = 'JTI_STORE'`; eliminar la clase `InMemoryJtiStore` y todos sus imports
- [x] 1.4 Editar `packages/domain-auth-users/src/index.ts` para re-exportar `InMemoryJtiStore` desde `'./infrastructure/jwt/jti-store.impl'` (ajuste provisional que se consolida en Paso 4)
- [x] 1.5 [verificación] Ejecutar `pnpm --filter @pld-api/domain-auth-users exec tsc --noEmit` y confirmar 0 errores — solo error preexistente en crypto/password.ts (Buffer/ArrayBufferView, existía antes de esta refactor)

## Paso 2: Mover archivos de infraestructura a `infrastructure/`

- [x] 2.1 Crear los directorios faltantes: `packages/domain-auth-users/src/infrastructure/entities/` y `packages/domain-auth-users/src/infrastructure/adapters/` (el directorio `infrastructure/jwt/` ya existe del Paso 1)
- [x] 2.2 `git mv packages/domain-auth-users/src/entities/user.entity.ts packages/domain-auth-users/src/infrastructure/entities/user.entity.ts`
- [x] 2.3 `git mv packages/domain-auth-users/src/adapters/auth.adapter.ts packages/domain-auth-users/src/infrastructure/adapters/auth.adapter.ts`
- [x] 2.4 `git mv packages/domain-auth-users/src/adapters/users.adapter.ts packages/domain-auth-users/src/infrastructure/adapters/users.adapter.ts`
- [x] 2.5 `git mv packages/domain-auth-users/src/factory.ts packages/domain-auth-users/src/infrastructure/factory.ts`
- [x] 2.6 `git mv packages/domain-auth-users/src/jwt/sign.ts packages/domain-auth-users/src/infrastructure/jwt/sign.ts`
- [x] 2.7 Eliminar los directorios que quedaron vacíos: `packages/domain-auth-users/src/entities/` y `packages/domain-auth-users/src/adapters/`

## Paso 3: Actualizar imports internos dentro del paquete

- [x] 3.1 Editar `packages/domain-auth-users/src/infrastructure/factory.ts`: actualizar todos los imports relativos — `'./ports/auth.port'` → `'../ports/auth.port'`, `'./ports/users.port'` → `'../ports/users.port'`, `'./jwt/jti-store'` → `'../jwt/jti-store'`; los imports de adapters (`'./adapters/users.adapter'`, `'./adapters/auth.adapter'`) siguen válidos dentro de infra
- [x] 3.2 Editar `packages/domain-auth-users/src/infrastructure/adapters/users.adapter.ts`: actualizar `'../crypto/password'` → `'../../crypto/password'` y `'../ports/users.port'` → `'../../ports/users.port'`; el import de `'../entities/user.entity'` sigue válido dentro de infra
- [x] 3.3 Editar `packages/domain-auth-users/src/infrastructure/adapters/auth.adapter.ts`: actualizar `'../crypto/password'` → `'../../crypto/password'`, `'../ports/auth.port'` → `'../../ports/auth.port'`, `'../ports/users.port'` → `'../../ports/users.port'`, `'../jwt/types'` → `'../../jwt/types'`, `'../jwt/jti-store'` → `'../../jwt/jti-store'`; el import de `'../jwt/sign'` sigue válido (ambos en infra) y el de `'../entities/user.entity'` también
- [x] 3.4 Editar `packages/domain-auth-users/src/infrastructure/jwt/sign.ts`: actualizar `'./types'` → `'../../jwt/types'`
- [x] 3.5 Editar `packages/domain-auth-users/src/infrastructure/jwt/jti-store.impl.ts`: verificar que el import type apunta a `'../../jwt/jti-store'` (correcto desde el Paso 1.2)
- [x] 3.6 Editar `packages/domain-auth-users/src/ports/users.port.ts`: actualizar `import type { UserEntity } from '../entities/user.entity'` → `import type { UserEntity } from '../infrastructure/entities/user.entity'` (mantener `import type`, no convertir a import de runtime). También corregido `ports/auth.port.ts` que tenía import inline a `../entities/user.entity`.
- [x] 3.7 [verificación] Ejecutar `pnpm --filter @pld-api/domain-auth-users exec tsc --noEmit` — solo error preexistente en crypto/password.ts, 0 errores nuevos

## Paso 4: Consolidar index.ts (re-exports desde paths nuevos)

- [x] 4.1 Reescribir `packages/domain-auth-users/src/index.ts` con exactamente los re-exports actualizados a paths nuevos
- [x] 4.2 Comparar símbolo por símbolo — mismos exports que el barrel anterior, solo cambian los paths de origen
- [x] 4.3 [verificación] Ejecutar `pnpm --filter @pld-api/domain-auth-users exec tsc --noEmit` — solo error preexistente, 0 errores nuevos

## Paso 5: Verificar consumidores en apps/auth-users

- [x] 5.1 Ejecutar `grep -rn "domain-auth-users/src\|packages/domain-auth-users" apps/` — 0 resultados (todos usan barrel raíz)
- [x] 5.2 No fue necesario corregir ningún import (no-op confirmado)
- [x] 5.3 [verificación] `tsc -p apps/auth-users/tsconfig.app.json --noEmit` — 0 errores

## Paso 6: Verificación final

- [x] 6.1 [verificación] `nx reset` ejecutado exitosamente
- [x] 6.2 [verificación] `nx run auth-users:build` — webpack compiled successfully. `domain-auth-users` no tiene target build en Nx (es librería sin config nx target — normal)
- [x] 6.3 [verificación] Levantar el stack dev y confirmar arranque limpio de `auth-users` — 23 módulos inicializados, sin errores de DI, JTI_STORE + createAuthUsersDomain resueltos
- [x] 6.4 [verificación] Smoke test del flujo de login — S1-S7 todos PASS. Ver smoke-test-report.md
- [x] 6.5 [verificación] Invariante de capas — 0 imports de typeorm/jsonwebtoken en ports/, crypto/, jwt/types.ts, jwt/jti-store.ts
