# Archive Report

**Change**: `consolidate-catalogs-into-auth-users`
**Fecha archivo**: 2026-04-20
**Backend**: openspec

---

## Specs Synced

| Dominio | Acción | Detalle |
|---|---|---|
| `catalogs` | Created | Nuevo spec (6 requirements, 6 scenarios) — endpoint `GET /catalogs/beneficiario` con cache HTTP |
| `http-error-formatting` | Created | Nuevo spec (3 requirements, 4 scenarios) — interceptor local en cada gateway |
| `deployment` | Created | Nuevo spec (5 requirements, 7 scenarios) — eliminación de `apps/catalogs` |

**Nota**: eran full specs (no deltas), ya que `openspec/specs/` estaba vacío al inicio del change. Se copiaron directamente.

## Archive Contents

- ✅ `proposal.md` — 380 palabras, 4 deliverables in scope
- ✅ `specs/catalogs/spec.md` — 6 requirements
- ✅ `specs/http-error-formatting/spec.md` — 3 requirements
- ✅ `specs/deployment/spec.md` — 5 requirements
- ✅ `design.md` — 6 decisiones arquitectónicas documentadas
- ✅ `tasks.md` — 29/29 tasks completadas
- ✅ `verify-report.md` — PASS WITH WARNINGS

## Source of Truth Updated

Los siguientes specs ahora reflejan el comportamiento vigente del sistema:
- `openspec/specs/catalogs/spec.md`
- `openspec/specs/http-error-formatting/spec.md`
- `openspec/specs/deployment/spec.md`

## Resultado Observable

| Antes | Después |
|---|---|
| 3 apps NestJS (`auth-users`, `catalogs`, `cross`) | 2 apps (`auth-users`, `cross`) |
| 3 containers en dev (9001, 9002, 9003) | 2 containers (9001, 9003) |
| 0 tests en todo el monorepo | 5 tests pasando en auth-users |
| `HttpErrorInterceptor` importado de `@pld-api/core` | Copia local en cada gateway |
| `GET /pld-api/catalogs/beneficiario` (puerto 9002) | `GET /pld-api/auth-users/catalogs/beneficiario` (puerto 9001) |
| Sin cache HTTP | `Cache-Control: public, max-age=86400, immutable` |

## Warnings No Resueltos (aceptados)

1. Latencia <5ms asumida por memoria, no medida con benchmark formal.
2. Shape de error de `ValidationPipe` no probado runtime en este change.

Ambos están fuera del alcance inicial y se atacan en Fase 5 del plan mayor (setup de tests + benchmarks).

## Trazabilidad

Archivo completo preservado en:
`openspec/changes/archive/2026-04-20-consolidate-catalogs-into-auth-users/`

## SDD Cycle Complete

Proposal → Specs → Design → Tasks → Apply → Verify → Archive ✅

Ready for next change.
