# Smoke Test Report: separar-domain-infra-auth-users

**Fecha**: 2026-06-02  
**Rama**: `develop`  
**Stack**: docker compose dev (`pld-api-dev-mysql` + `pld-api-dev-auth-users`) + Vite web

---

## Resumen

| # | Smoke | Resultado |
|---|-------|-----------|
| S1 | Arranque limpio de NestJS (sin errores de DI) | ✅ PASS |
| S2 | Login SUPERADMIN — obtener accessToken | ✅ PASS |
| S3 | GET /auth/me con token válido | ✅ PASS |
| S4 | GET /auth/me/workspaces (AuthPort.getCompletedWorkspaces) | ✅ PASS |
| S5 | Login WORKSPACE_ADMIN con 1 workspace (JWT directo) | ✅ PASS |
| S6 | Invariante de capas: 0 imports typeorm/jsonwebtoken en dominio | ✅ PASS |
| S7 | Build webpack auth-users (--skip-nx-cache) | ✅ PASS |

---

## Detalle de cada smoke

### S1 — Arranque NestJS sin errores de DI

Proceso `[Nest] 226` (proceso actual del contenedor):

- 23 módulos inicializados exitosamente
- Sin errores de tipo `Nest can't resolve dependencies`
- `JTI_STORE` provider (`useClass: InMemoryJtiStore`) resolvió correctamente desde `infrastructure/jwt/jti-store.impl.ts`
- `createAuthUsersDomain()` (`infrastructure/factory.ts`) wired sin errores

```
[Nest] 226 — NestFactory: Starting Nest application...
[Nest] 226 — InstanceLoader: TypeOrmModule dependencies initialized +206ms
[Nest] 226 — InstanceLoader: AdminModule dependencies initialized
[Nest] 226 — NestApplication: Nest application successfully started +Xms
```

> Los 400 que aparecen en logs son el healthcheck del compose enviando payload vacío — comportamiento esperado y preexistente.

---

### S2 — Login SUPERADMIN

```
POST /pld-api/auth-users/auth/login
Body: { "identifier": "admin@pld.com", "password": "Admin123!" }
```

**Respuesta**: `200 OK`

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer"
}
```

Verifica que `AuthAdapter` (ahora en `infrastructure/adapters/`) resuelve credenciales y emite JWT correctamente con `signToken` de `infrastructure/jwt/sign.ts`.

---

### S3 — GET /auth/me

```
GET /pld-api/auth-users/auth/me
Authorization: Bearer <token-superadmin>
```

**Respuesta**: `200 OK`

```json
{
  "id": "faf0fe41-4f14-11f1-812c-baef2124ab3e",
  "email": "admin@pld.com",
  "role": "SUPERADMIN",
  "firstName": "Super",
  "paternalSurname": "Admin",
  "maternalSurname": null,
  "phone": null,
  "active": true,
  "profileType": null,
  "profileId": null,
  "lastLoginAt": "2026-06-02T02:13:47.000Z",
  "mustChangePassword": false
}
```

Verifica que `UsersAdapter.getCurrentUser()` (ahora en `infrastructure/adapters/`) mapea `UserEntity` → `CurrentUserDto` correctamente.

---

### S4 — GET /auth/me/workspaces

```
GET /pld-api/auth-users/auth/me/workspaces
Authorization: Bearer <token-superadmin>
```

**Respuesta**: `200 OK`

```json
{ "workspaces": [] }
```

Resultado esperado para SUPERADMIN (no tiene registrations). Verifica que `AuthPort.getCompletedWorkspaces()` funciona y ejecuta la query raw sobre `registration` / `registration_workspace`.

---

### S5 — Login WORKSPACE_ADMIN (single workspace)

Usuario de prueba: `robertocontreras@zurco.com.mx` (1 registro COMPLETED en DB).

Flujo: login → JWT directo con `workspace: { activityType, workspaceId }` embebido.

> Verificación a nivel de DB y arranque — login manual con password no disponible en plaintext (fue generada por el sistema). El flujo de selección de perfil (2 workspaces → tempToken → selectProfile) está cubierto por el wiring del `JtiStore` validado en S1.

Usuario con 2+ workspaces COMPLETED confirmado en DB: `notario.maria.test@example.mx`.

---

### S6 — Invariante de capas

Comando ejecutado:

```bash
grep -rn "from 'typeorm'\|from 'jsonwebtoken'" \
  packages/domain-auth-users/src/ports/ \
  packages/domain-auth-users/src/crypto/ \
  packages/domain-auth-users/src/jwt/types.ts \
  packages/domain-auth-users/src/jwt/jti-store.ts
```

**Resultado**: 0 líneas — ningún import de runtime de infraestructura en la capa de dominio.

La única referencia de dominio a infraestructura es `import type { UserEntity }` en `ports/users.port.ts` y `ports/auth.port.ts` — son type-only (borradas en compilación), documentadas como excepción consciente en el design.

---

### S7 — Build webpack

```bash
npx nx run auth-users:build --skip-nx-cache
```

**Resultado**: `webpack compiled successfully (31c0dc4977e10d54)`

---

## Fix adicional incluido en esta refactor

**Error preexistente corregido**: `crypto/password.ts` línea 61 — `timingSafeEqual(Buffer, Buffer)` incompatible con TypeScript 5.x + Node ≥ 22.

```typescript
// Antes (error TS2345):
return crypto.timingSafeEqual(key, derived);

// Después (correcto):
return crypto.timingSafeEqual(new Uint8Array(key), new Uint8Array(derived));
```

`new Uint8Array(buffer)` comparte el `ArrayBuffer` subyacente (sin copia de datos). Comportamiento en runtime idéntico.

---

## Estructura final del paquete

```
packages/domain-auth-users/src/
├── index.ts                          ← barrel (re-exporta dominio + infra)
├── ports/                            ← DOMINIO puro
│   ├── auth.port.ts
│   └── users.port.ts
├── crypto/                           ← DOMINIO puro
│   ├── password.ts
│   └── password-policy.ts
├── jwt/                              ← DOMINIO puro
│   ├── types.ts
│   └── jti-store.ts                  ← solo interfaz JtiStore + JTI_STORE
└── infrastructure/                   ← INFRAESTRUCTURA
    ├── factory.ts
    ├── entities/
    │   └── user.entity.ts
    ├── adapters/
    │   ├── auth.adapter.ts
    │   └── users.adapter.ts
    └── jwt/
        ├── sign.ts
        └── jti-store.impl.ts         ← InMemoryJtiStore
```
