# pld workspace

Coordinador del proyecto PLD (Prevención de Lavado de Dinero, México). El workspace root orquesta dos sub-repos independientes:

- [pld-api/](pld-api/) — Monorepo Nx 14 + pnpm, NestJS 9, TypeORM + MySQL 8, JWT. App única `auth-users` (las antiguas `catalogs` y `cross` ya fueron consolidadas / eliminadas). En migración de `libs/` acoplados a NestJS → `packages/` puros.
- [pld-web/](pld-web/) — Vite 7 + React 19 + TS 5.9 + Tailwind 4 + PrimeReact 10 + React Query + Zustand. Consume la API de `pld-api`.

Todo el contexto Claude (skill-registry, openspec, docs, settings) vive en este root. Los sub-repos NO tienen capa Claude propia — abrir Claude siempre desde `/home/cristobal/work/pld/`.

Arquitectura de dev local y contratos API→Web: ver [docs/arquitecturas/architecture.md](docs/arquitecturas/architecture.md).

## Interacción con el usuario

Cuando haya 2–6 alternativas para elegir, usar `/interactive-options` para presentarlas. No usar para preguntas binarias sí/no — esas se responden inline.


## Contexto de sesión

**Leer [`docs/CONTEXT.md`](docs/CONTEXT.md) al inicio de cualquier sesión** que involucre UI, flujos de registro o documentación. Es el hub central que apunta a todas las fuentes de verdad del proyecto — componentes, flujos BE, diseños UI, usuarios de prueba. No escanear carpetas: navegar desde ahí.

## Commit policy (workspace + sub-repos)

Aplica a commits en este repo y en los sub-repos (`pld-api/`, `pld-web/`). Sin excepción, salvo que el usuario pida explícitamente lo contrario en el mismo turno.

- **Una sola línea**. Subject únicamente, sin body, sin footer.
- **Conventional Commits**: `tipo(scope): descripción`.
  - Tipos permitidos: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `style`, `perf`, `build`, `ci`, `revert`.
  - `scope` opcional pero preferido cuando el cambio es local a un módulo (ej: `auth`, `components`, `stack`).
- **Subject ≤72 caracteres**, imperativo, minúscula inicial, sin punto final.
- **Sin `Co-Authored-By`** salvo pedido explícito.
- **Sin heredoc multi-línea**: usar `git commit -m "tipo(scope): descripción"` directo.
- Si el cambio no cabe en una línea, preferir partirlo en varios commits antes que agregar body.

## Documentación de flujos (`docs/flujos/<nombre-flujo>/`)

Aplica al crear o actualizar flujos en `docs/flujos/`.

- `flujo.md` es la **fuente de verdad del sistema completo** — describe el flujo end-to-end incluyendo pasos de usuario, validaciones, lógica BE y referencias a diseños UI. No es exclusivo del BE.
- El toon en `flujo.md` representa el flujo del sistema; puede referenciar tanto decisiones de BE (endpoints, tablas) como de UI (pantallas, estados visuales). Las notas e inconsistencias del llm-index son la guía para alinear implementación y diseño.
- Los diseños UI viven en `disenos/` dentro del mismo folder, con subcarpetas por paso (`paso-N-*`) y `pagina-resultado/` cuando aplique.
- **Linkear ambos sentidos**: `flujo.md` referencia `disenos/`; el `README.md` de `disenos/` referencia `flujo.md`.
- Cuando el usuario provee mockups UI, archivar en `disenos/paso-N-*/` y actualizar el llm-index en `flujo.md`.
- Decisiones técnicas relevantes al flujo → crear ADR en `docs/decisiones/`.

<!-- JARVIS:BEGIN hash=ws-pld-root -->
## Jarvis MCP (project)

**project_id**: `pld` (PLD workspace)

Pasá este `project_id` al llamar tools de Jarvis que lo acepten.

### Active integrations
_(ninguna configurada)_
<!-- JARVIS:END -->
