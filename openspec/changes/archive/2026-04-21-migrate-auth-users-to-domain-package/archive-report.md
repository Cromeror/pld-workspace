# Archive Report

**Change**: `migrate-auth-users-to-domain-package`
**Fecha archivo**: 2026-04-21
**Backend**: openspec

---

## Specs Synced

| Dominio | Acción | Detalle |
|---|---|---|
| `auth` | Created | Nuevo spec (4 requirements, 10 scenarios) — contrato de autenticación, login HTTP 201, JWT puro |
| `users` | Created | Nuevo spec (5 requirements, 14 scenarios) — puerto UsersPort, entidad UserEntity, password hash con scrypt + timingSafeEqual |
| `domain-packages` | Created | Nuevo spec (5 requirements, 11 scenarios) — patrón arquitectónico canónico para paquetes sin `@nestjs/*` |

**Nota**: eran full specs (no deltas), ya que `openspec/specs/` no tenía estos dominios. Se copiaron directamente a la fuente de verdad.

## Archive Contents

- ✅ `proposal.md` — ~430 palabras, 8 decisiones + scope claro (in/out)
- ✅ `specs/auth/spec.md` — 4 requirements
- ✅ `specs/users/spec.md` — 5 requirements
- ✅ `specs/domain-packages/spec.md` — 5 requirements
- ✅ `design.md` — arquitectura factory pattern, decisiones de TypeORM
- ✅ `tasks.md` — 47 tasks totales (43 done + 2 phases skipped + 2 fases diferidas)
- ✅ `exploration.md` — análisis de código existente pre-cambio
- ✅ `verify-report.md` — PASS WITH WARNINGS (43/47 tasks done, 2 fases skipped conscientemente)

## Source of Truth Updated

Los siguientes specs ahora reflejan el comportamiento vigente del sistema:
- `openspec/specs/auth/spec.md`
- `openspec/specs/users/spec.md`
- `openspec/specs/domain-packages/spec.md`

## Resultado Observable — antes/después

| Aspecto | Antes | Después |
|---|---|---|
| Lógica auth dispersa en | `libs/users`, `libs/jwt`, `libs/core/services/password.function.ts`, `apps/auth-users/src/{auth,users}/` | Centralizada en `packages/domain-auth-users/` (puro TS, sin `@nestjs/*`) |
| Generación de password | `Math.random()` en `password.function.ts` | `crypto.randomBytes()` + `scrypt` en `packages/domain-auth-users/src/crypto/password.ts` |
| `JwtAuthGuard` + `JwtStrategy` | Importados de `@pld-api/jwt` en 6+ archivos | Copias locales en `apps/auth-users/src/shared/auth/` y `apps/cross/src/shared/auth/` |
| Contrato de negocio | Adapters privados en `libs/` | Puertos públicos (`AuthPort`, `UsersPort`) en paquete domain |
| Compilación del paquete | N/A (no existía) | Aislada, sin NestJS runtime (tsc --noEmit -p packages/domain-auth-users/tsconfig.json → 0) |
| Factory | N/A | `createAuthUsersDomain({ ds, jwtSecret, jwtExpiresIn })` retorna `{ authPort, usersPort }` |
| HTTP POST /auth/login | Retorna `{ token }` (HTTP 201) | Mismo shape, HTTP 201, token válido (verificado manual) |
| tsconfig.base.json | 0 entries de domain | 1 entry: `@pld-api/domain-auth-users` → `packages/domain-auth-users/src/index.ts` |

## Warnings No Resueltos (aceptados)

1. **Phase 7 — Smoke test diferido**: Propuesta de test (`typeof x === 'function'`) rechazada durante apply por tautológica (validaba shape, no comportamiento real). Diferido a Fase 5 para estrategia multi-paquete coherente.

2. **Phase 4 — Cross gateway JWT wiring**: `apps/cross` no tiene endpoints protegidos hoy. Endpoint `POST /crear-beneficiario` es open. Work deferred a Fase 5 cuando `domain-participants` esté migrado.

3. **Cobertura de tests**: Tests de comportamiento real de auth/users diferidos a Fase 5 (setup coherente de tests + benchmarks).

Todos están fuera del alcance inicial del change y son atacados en el plan mayor.

## Trazabilidad

Archivo completo preservado en:
`openspec/changes/archive/2026-04-21-migrate-auth-users-to-domain-package/`

Estructura:
```
archive/2026-04-21-migrate-auth-users-to-domain-package/
├── proposal.md
├── exploration.md
├── specs/
│   ├── auth/spec.md
│   ├── users/spec.md
│   └── domain-packages/spec.md
├── design.md
├── tasks.md
├── verify-report.md
└── archive-report.md
```

## SDD Cycle Complete

Proposal → Specs → Design → Tasks → Apply → Verify → Archive ✅

Ready for next change: **`migrate-participants-to-domain-package`** (Fase 4.2 del plan mayor).
