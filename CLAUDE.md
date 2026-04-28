# pld workspace

Coordinador del proyecto PLD (Prevención de Lavado de Dinero, México). El workspace root orquesta dos sub-repos independientes:

- [pld-api/](pld-api/) — Monorepo Nx 14 + pnpm, NestJS 9, TypeORM + MySQL 8, JWT. 3 apps (`auth-users`, `cross`, antes había `catalogs` consolidada en `auth-users`). En migración de `libs/` acoplados a NestJS → `packages/` puros.
- [pld-web/](pld-web/) — Vite 7 + React 19 + TS 5.9 + Tailwind 4 + PrimeReact 10 + React Query + Zustand. Consume la API de `pld-api`.

Todo el contexto Claude (skill-registry, openspec, docs, settings) vive en este root. Los sub-repos NO tienen capa Claude propia — abrir Claude siempre desde `/home/cristobal/work/pld/`.

Arquitectura de dev local y contratos API→Web: ver [docs/architecture.md](docs/architecture.md).

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

## Documentación de flujos de registro (`docs/FLUJO_*.md` + `docs/designs/<flujo>/`)

Aplica al crear o actualizar archivos `docs/FLUJO_REGISTRO_*.md` y carpetas relacionadas en `docs/designs/`.

- **Separar BE y UI**:
  - El `.md` del flujo contiene **solo el diagrama BE** (mermaid) — endpoints, tablas, transacciones, validación autoritativa. Mismo estilo que [FLUJO_REGISTRO_REPORTING_ENTITIES.md](docs/FLUJO_REGISTRO_REPORTING_ENTITIES.md).
  - El flujo UI vive en `docs/designs/<nombre-flujo>/` con una subcarpeta por paso (`step-1-*`, `step-2-*`, …) y carpetas `shared/`, `result-page/` cuando aplique. Las capturas y notas de UI van ahí.
- **No mezclar planos UI y BE en el mismo diagrama**. Si un nodo es estado de cliente (validación visual, "ingresa el dato faltante"), va en las capturas UI, no en el mermaid del `.md`.
- **Linkear ambos sentidos**: el `.md` referencia `docs/designs/<flujo>/` para las vistas; el `README.md` o `index.md` de la carpeta de designs referencia el `.md` del flujo.
- Cuando el usuario provee una imagen del flujo BE (cajas/decisiones), traducirla a mermaid en el `.md`. Cuando provee mockups UI, archivarlos en `docs/designs/<flujo>/step-N-*/`.

<!-- JARVIS:BEGIN hash=ws-pld-root -->
## Jarvis MCP (project)

**project_id**: `pld` (PLD workspace)

Pasá este `project_id` al llamar tools de Jarvis que lo acepten.

### Active integrations
_(ninguna configurada)_
<!-- JARVIS:END -->
