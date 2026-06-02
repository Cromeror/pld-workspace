# Design: separar-domain-infra-auth-users

## Contexto y objetivo

Reorganizar `@pld-api/domain-auth-users` para separar **dominio puro** de **infraestructura** dentro del mismo paquete, creando una subcapa interna `infrastructure/`. La separación es estructural: el barrel `index.ts` sigue re-exportando exactamente los mismos símbolos públicos, por lo que **no hay breaking changes** para los consumidores en `apps/auth-users/src/`.

Decisiones de diseño ya fijadas (no se re-discuten aquí):

- **No** se crea un tipo `User` puro separado de `UserEntity`. En un monolito el mapeo doble no aporta valor.
- `UserEntity` se mueve a `infrastructure/entities/`.
- `JtiStore` (interfaz) permanece en dominio; `InMemoryJtiStore` se mueve a infraestructura.
- `index.ts` re-exporta los mismos símbolos — sin breaking changes.
- `listWorkspaceAdminsPaginated` vive en un `AdminAdapter` dentro de la app, no en `UsersPort`.
- Refactor incremental: cada paso compila.

Hecho relevante verificado en el código actual: **todos** los consumidores de la app importan desde el barrel raíz `@pld-api/domain-auth-users`, nunca por rutas relativas internas del paquete. El `package.json` apunta `main`/`types` a `./src/index.ts`. Esto significa que mientras el barrel mantenga los re-exports, mover archivos internamente es transparente.

---

## 1. Estructura objetivo del paquete

```
packages/domain-auth-users/src/
├── index.ts                          ← barrel público (re-exporta dominio + infra)
│
├── ports/                            ← DOMINIO — interfaces puras
│   ├── auth.port.ts                  ← dominio puro
│   └── users.port.ts                 ← dominio puro
│
├── crypto/                           ← DOMINIO — reglas puras (Node crypto, sin ORM/JWT)
│   ├── password.ts                   ← dominio puro
│   └── password-policy.ts            ← dominio puro
│
├── jwt/                              ← DOMINIO — tipos e interfaces puras
│   ├── types.ts                      ← dominio puro (JwtPayload, ActivityType, etc.)
│   └── jti-store.ts                  ← dominio puro (SOLO la interfaz JtiStore + token JTI_STORE)
│
└── infrastructure/                   ← INFRAESTRUCTURA — depende de typeorm / jsonwebtoken
    ├── factory.ts                    ← infra (requiere DataSource; ensambla adapters)
    ├── entities/
    │   └── user.entity.ts            ← infra (decoradores TypeORM)
    ├── adapters/
    │   ├── auth.adapter.ts           ← infra (jsonwebtoken + DataSource.query)
    │   └── users.adapter.ts          ← infra (TypeORM Repository)
    └── jwt/
        ├── sign.ts                   ← infra (wrapper de jsonwebtoken)
        └── jti-store.impl.ts         ← infra (InMemoryJtiStore: setInterval, estado mutable)
```

### Notas sobre el reparto de capas

| Archivo | Capa | Por qué |
|---|---|---|
| `ports/auth.port.ts` | Dominio | Solo interfaces y DTOs; sin imports de runtime de infra |
| `ports/users.port.ts` | Dominio | Idem. Tipa `UserEntity` con `import type` (sin acoplamiento de runtime) |
| `crypto/password.ts` | Dominio | Usa `node:crypto`/scrypt — política del lenguaje, no infra externa |
| `crypto/password-policy.ts` | Dominio | Reglas de validación puras |
| `jwt/types.ts` | Dominio | Tipos del payload, sin runtime |
| `jwt/jti-store.ts` | Dominio | Queda **solo** la `interface JtiStore` + el token `JTI_STORE` |
| `infrastructure/factory.ts` | Infra | Recibe `DataSource`, instancia adapters concretos |
| `infrastructure/entities/user.entity.ts` | Infra | Decoradores `@Entity`, `@Column` (TypeORM) |
| `infrastructure/adapters/users.adapter.ts` | Infra | `Repository<UserEntity>`, queries |
| `infrastructure/adapters/auth.adapter.ts` | Infra | `jsonwebtoken`, `DataSource.query`, JTI |
| `infrastructure/jwt/sign.ts` | Infra | `import * as jwt from 'jsonwebtoken'` |
| `infrastructure/jwt/jti-store.impl.ts` | Infra | `InMemoryJtiStore` (estado, `setInterval`) |

### Caso límite: `ports/users.port.ts` referencia `UserEntity`

`users.port.ts` (dominio) hace `import type { UserEntity } from '../entities/user.entity'` y usa `UserEntity` en firmas (`getUserById(): Promise<UserEntity | null>`, `Partial<UserEntity>`, `Omit<UserEntity,'passwordHash'>`). Esto es una dependencia dominio→infra que, en estricta hexagonal, sería una violación.

**Decisión coherente con "no crear tipo `User` puro"**: se acepta esta referencia **solo a nivel de tipo** (`import type`), porque:

- Es borrada en compilación: `import type` no genera ningún `require`/`import` de runtime, así que el dominio NO arrastra el grafo de TypeORM en ejecución.
- Evitar el mapeo doble fue una decisión explícita del tech-review.

El import pasa a apuntar a la nueva ruta: `import type { UserEntity } from '../infrastructure/entities/user.entity'`. Se documenta como excepción consciente en la tabla de invariantes (sección 3). Lo mismo aplica a `auth.adapter.ts` que ya es infra (sin problema).

---

## 2. Pasos de migración (secuencia incremental)

Cada paso deja el paquete y la app compilando (`tsc --noEmit`) y, en el paso final, arrancando. Los pasos están ordenados para que ningún commit intermedio quede roto.

### Paso 1 — Separar `jti-store.ts` (interfaz vs implementación)

1. Crear `infrastructure/jwt/jti-store.impl.ts` con la clase `InMemoryJtiStore` (mueve el bloque de `setInterval` + `consume`).
2. En `jwt/jti-store.ts` dejar **solo**:
   - `export interface JtiStore { consume(jti: string, expiresAt: number): boolean }`
   - `export const JTI_STORE = 'JTI_STORE'`
   - (mover el comentario `// TODO: reemplazar por adapter Redis...` al `.impl.ts`)
3. `jti-store.impl.ts` importa la interfaz: `import type { JtiStore } from '../../jwt/jti-store'`.
4. `index.ts` (provisional, se consolida en Paso 4): re-exportar `InMemoryJtiStore` desde el nuevo path.

Resultado: dominio JTI puro; implementación in-memory en infra. Compila.

### Paso 2 — Mover archivos de infraestructura a `infrastructure/`

Mover (sin tocar contenido todavía, salvo imports):

- `entities/user.entity.ts`      → `infrastructure/entities/user.entity.ts`
- `adapters/auth.adapter.ts`     → `infrastructure/adapters/auth.adapter.ts`
- `adapters/users.adapter.ts`    → `infrastructure/adapters/users.adapter.ts`
- `factory.ts`                   → `infrastructure/factory.ts`
- `jwt/sign.ts`                  → `infrastructure/jwt/sign.ts`

Usar `git mv` para preservar historial. Borrar las carpetas vacías `entities/` y `adapters/` resultantes.

### Paso 3 — Actualizar imports internos dentro del paquete

Reajustar rutas relativas tras el movimiento. Cambios concretos:

**`infrastructure/factory.ts`** (ahora un nivel más profundo):
- `'./ports/auth.port'` → `'../ports/auth.port'`
- `'./ports/users.port'` → `'../ports/users.port'`
- `'./adapters/users.adapter'` → `'./adapters/users.adapter'` (sigue relativo dentro de infra)
- `'./adapters/auth.adapter'` → `'./adapters/auth.adapter'`
- `'./jwt/jti-store'` (type `JtiStore`) → `'../jwt/jti-store'`

**`infrastructure/adapters/users.adapter.ts`**:
- `'../entities/user.entity'` → `'../entities/user.entity'` (sigue válido dentro de infra)
- `'../crypto/password'` → `'../../crypto/password'`
- `'../ports/users.port'` → `'../../ports/users.port'`

**`infrastructure/adapters/auth.adapter.ts`**:
- `'../crypto/password'` → `'../../crypto/password'`
- `'../jwt/sign'` → `'../jwt/sign'` (ahora ambos en infra)
- `'../ports/auth.port'`, `'../ports/users.port'` → `'../../ports/...'`
- `'../jwt/types'` → `'../../jwt/types'`
- `'../jwt/jti-store'` (type) → `'../../jwt/jti-store'`
- `'../entities/user.entity'` → `'../entities/user.entity'`

**`infrastructure/jwt/sign.ts`**:
- `'./types'` → `'../../jwt/types'`

**`infrastructure/jwt/jti-store.impl.ts`**:
- `import type { JtiStore }` → `'../../jwt/jti-store'`

**`ports/users.port.ts`** (dominio, referencia de tipo a entity):
- `'../entities/user.entity'` → `'../infrastructure/entities/user.entity'` (mantener `import type`)

Verificar con `tsc --noEmit` dentro del paquete antes de continuar.

### Paso 4 — Consolidar `index.ts` (re-exports desde paths nuevos)

Reescribir el barrel para que los símbolos públicos salgan de las rutas nuevas, manteniendo **exactamente** los mismos nombres exportados:

```ts
// Factory (infra)
export { createAuthUsersDomain } from './infrastructure/factory';
export type { AuthUsersDomainConfig } from './infrastructure/factory';

// Ports (dominio)
export type { AuthPort, LoginInput, LoginOutput, AvailableWorkspace } from './ports/auth.port';
export type { UsersPort, CreateUserInput, CreateUserOutput, CurrentUserDto, MarkPasswordChangedInput } from './ports/users.port';

// Entity (infra)
export { UserEntity } from './infrastructure/entities/user.entity';

// JWT sign (infra)
export { signToken, verifyToken } from './infrastructure/jwt/sign';
export type { SignOptions } from './infrastructure/jwt/sign';

// JWT types (dominio)
export type { JwtPayload, JwtWorkspace, TempTokenPayload, UserRole, ActivityType, LoginResult } from './jwt/types';

// JTI: interfaz + token (dominio) | implementación (infra)
export type { JtiStore } from './jwt/jti-store';
export { JTI_STORE } from './jwt/jti-store';
export { InMemoryJtiStore } from './infrastructure/jwt/jti-store.impl';

// Crypto (dominio)
export { hashPassword, verifyPassword, generateSecurePassword } from './crypto/password';
export {
  PASSWORD_POLICY,
  PASSWORD_POLICY_MESSAGES,
  validatePasswordStrength,
} from './crypto/password-policy';
export type { PasswordPolicyRule, PasswordValidationResult } from './crypto/password-policy';
```

El conjunto de símbolos exportados es idéntico al actual (`index.ts` líneas 1–28). Diff = solo rutas de origen.

### Paso 5 — Verificar imports en `apps/auth-users/src/`

**Verificación, no cambio.** Auditoría del código actual confirma que TODOS los consumidores importan desde el barrel raíz `@pld-api/domain-auth-users` (`UserEntity`, `createAuthUsersDomain`, `InMemoryJtiStore`, `JTI_STORE`, `hashPassword`, `generateSecurePassword`, `ActivityType`, `JtiStore`, `UsersPort`, `AuthPort`, `TempTokenPayload`, etc.). No hay ningún import por ruta relativa interna del paquete ni por `domain-auth-users/src/...`.

Acción: ejecutar `grep -rn "domain-auth-users/src\|packages/domain-auth-users" apps/` y confirmar 0 resultados. Si aparecen (no esperado), reapuntarlos al barrel. Si el grep está limpio, este paso es un no-op confirmado y no se toca la app.

### Paso 6 — Verificación final

1. `pnpm tsc --noEmit` (o `nx run domain-auth-users:build` + `nx run auth-users:build`) en el monorepo — debe pasar sin errores.
2. Levantar la app `auth-users` (stack dev) y confirmar arranque limpio: el `JTI_STORE` provider (`useClass: InMemoryJtiStore`) y `createAuthUsersDomain` deben resolverse en el módulo Nest.
3. Smoke test del flujo de login con selección de perfil (ejercita `signToken`, `JtiStore.consume`, `AuthAdapter`) para validar que el wiring de infra sigue intacto.
4. Correr la suite de specs del paquete y de la app si existe.

---

## 3. Invariantes de capa

Reglas de importación dentro del paquete. "Runtime" = `import` normal; "type-only" = `import type` (borrado en compilación).

| Subcarpeta | Puede importar (runtime) | Puede importar (type-only) | Prohibido |
|---|---|---|---|
| `ports/` | nada externo de runtime | tipos de dominio; `UserEntity` (excepción documentada, type-only) | `typeorm`, `jsonwebtoken`, cualquier valor de `infrastructure/` |
| `crypto/` | `node:crypto` | tipos propios | `typeorm`, `jsonwebtoken`, `infrastructure/`, `ports/` |
| `jwt/types.ts` | nada | nada | `typeorm`, `jsonwebtoken`, `infrastructure/` |
| `jwt/jti-store.ts` | nada (solo interfaz + const string) | nada | cualquier import de infra; `InMemoryJtiStore` |
| `infrastructure/**` | `typeorm`, `jsonwebtoken`, `node:crypto`, `@pld-api/shared-*`, dominio (`ports/`, `crypto/`, `jwt/`) | lo que necesite | — (la infra es el borde, puede depender hacia adentro) |

Regla direccional: **infra → dominio permitido; dominio → infra prohibido en runtime**. La única dependencia dominio→infra tolerada es `ports/users.port.ts` tipando con `UserEntity` mediante `import type`, justificada por la decisión de no duplicar el modelo en un monolito. No se permite ampliar esta excepción a imports de runtime ni a otros archivos de dominio.

Chequeo automatizable (opcional, recomendado como guardrail futuro): un test/lint que falle si `ports/`, `crypto/` o `jwt/types.ts`|`jwt/jti-store.ts` contienen `from 'typeorm'` o `from 'jsonwebtoken'` en imports de runtime.

---

## 4. AdminAdapter — patrón de referencia para queries de backoffice

Las queries de backoffice (ej. `listWorkspaceAdminsPaginated`, que está en stash y se reimplementa **después** de esta refactor) **no** van en `UsersPort` ni en `UsersAdapter` del paquete de dominio. Su lugar es un adapter propio de la app:

```
apps/auth-users/src/admin/users/admin-users.adapter.ts
```

### Por qué no en `UsersPort`

- `UsersPort` modela el contrato de dominio de **usuarios** (crear, buscar por email/id, cambiar password). Es estable y consumido por auth/registration.
- Una query paginada de backoffice que **compone múltiples tablas** (p. ej. `user` + `registration` + `registration_workspace` para listar admins de workspaces con su estado) es una preocupación de lectura/reporte específica de la app de administración, no del dominio de autenticación.
- Meterla en `UsersPort` infla la interfaz de dominio con concerns de presentación/reporte y acopla el paquete a tablas que no le pertenecen (`registration*`).

### Patrón a seguir

`AdminUsersAdapter` es un provider Nest dentro de la app que:

1. **Inyecta `DataSource` directamente** (no `UsersPort`, no `Repository<UserEntity>` salvo que necesite una entidad concreta para una operación trivial).
2. **No implementa una port de dominio.** Es un read-model/adapter de aplicación; opcionalmente puede implementar una interfaz local definida en la propia app (`admin/users/admin-users.port.ts`) si el controller quiere depender de una abstracción, pero esa interfaz vive en la app, no en el paquete.
3. **Compone datos de múltiples tablas** vía `DataSource.query(...)` o un QueryBuilder, devolviendo DTOs de lectura propios de la vista de backoffice (paginación, filtros, joins).
4. Reside bajo `apps/auth-users/src/admin/users/` junto a su módulo (`admin-users.module.ts`) y controller.

Esqueleto de referencia:

```ts
// apps/auth-users/src/admin/users/admin-users.adapter.ts
import { Injectable } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import type { DataSource } from 'typeorm';

@Injectable()
export class AdminUsersAdapter {
  constructor(@InjectDataSource() private readonly ds: DataSource) {}

  async listWorkspaceAdminsPaginated(params: {
    page: number;
    pageSize: number;
    /* filtros... */
  }): Promise<{ items: WorkspaceAdminRow[]; total: number }> {
    // JOIN user + registration + registration_workspace
    // SELECT paginado + COUNT, devuelve DTO de backoffice
  }
}
```

Esto es consistente con `AuthAdapter`, que ya hace `DataSource.query(...)` directo contra `registration`/`registration_workspace` para resolver workspaces COMPLETED — el mismo principio aplicado en la capa de la app: **las queries que cruzan agregados de varios módulos usan `DataSource` directo, no la port de dominio de un solo agregado**.

Nota: la creación efectiva de `admin/users/` y la reimplementación de `listWorkspaceAdminsPaginated` quedan **fuera del alcance** de este change (segunda fase). Este documento solo fija el patrón objetivo para que el stash se reaplique en el lugar correcto.

---

## 5. Riesgos y mitigaciones

| # | Riesgo | Impacto | Mitigación |
|---|---|---|---|
| R1 | Romper el contrato público del barrel (símbolo renombrado/omitido al reescribir `index.ts`) | Alto — rompe toda la app | Comparar la lista de `export`s del `index.ts` nuevo contra el actual símbolo por símbolo (sección 4 del design = misma lista, distinto path). `tsc` de la app lo detecta de inmediato. |
| R2 | Rutas relativas mal ajustadas tras `git mv` (Paso 3) | Medio — el paquete no compila | Hacer Paso 3 como commit aislado y correr `tsc --noEmit` del paquete antes de tocar `index.ts`. La tabla de remapeo de imports (Paso 3) es la checklist. |
| R3 | Dependencia dominio→infra accidental de runtime (alguien convierte un `import type` en `import` normal en `ports/`) | Medio — reintroduce el acoplamiento que esta refactor elimina | Mantener `import type` explícito en `ports/users.port.ts`; documentar la excepción (sección 3); recomendar guardrail de lint (grep de `from 'typeorm'`/`'jsonwebtoken'` en dominio). |
| R4 | `InMemoryJtiStore` deja de exportarse o el provider Nest `useClass: InMemoryJtiStore` no resuelve | Alto — login con multi-perfil falla en runtime | Verificar export en barrel (Paso 4) y arranque de `auth-users.module.ts` (Paso 6.2); smoke test del flujo de selección de perfil (Paso 6.3). |
| R5 | `git mv` no preserva historial o deja carpetas vacías huérfanas | Bajo — ruido en el repo | Usar `git mv`; borrar explícitamente `entities/` y `adapters/` vacíos; revisar `git status` antes de commit. |
| R6 | Imports relativos internos en la app (no esperados) que apunten a `domain-auth-users/src/...` | Bajo — improbable según auditoría | Paso 5: `grep` confirmatorio; si aparece algo, reapuntar al barrel raíz. |
| R7 | Caché de build de Nx/TS sirviendo paths viejos tras el movimiento | Bajo — falsos negativos/positivos en compilación | Limpiar caché (`nx reset` / borrar `dist`/`tsbuildinfo`) antes de la verificación final del Paso 6. |
| R8 | Pasos no atómicos: un commit intermedio queda sin compilar | Medio — viola la restricción de negocio | Cada paso (1→6) es un commit con `tsc --noEmit` verde antes de avanzar; el orden está diseñado para que el barrel apunte a paths válidos en todo momento. |
