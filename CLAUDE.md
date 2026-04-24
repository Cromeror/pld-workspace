# pld workspace

Coordinador del proyecto PLD (Prevención de Lavado de Dinero, México). El workspace root orquesta dos sub-repos independientes:

- [pld-api/](pld-api/) — Monorepo Nx 14 + pnpm, NestJS 9, TypeORM + MySQL 8, JWT. 3 apps (`auth-users`, `cross`, antes había `catalogs` consolidada en `auth-users`). En migración de `libs/` acoplados a NestJS → `packages/` puros.
- [pld-web/](pld-web/) — Vite 7 + React 19 + TS 5.9 + Tailwind 4 + PrimeReact 10 + React Query + Zustand. Consume la API de `pld-api`.

Todo el contexto Claude (skill-registry, openspec, docs, settings) vive en este root. Los sub-repos NO tienen capa Claude propia — abrir Claude siempre desde `/home/cristobal/work/pld/`.

Arquitectura de dev local y contratos API→Web: ver [docs/architecture.md](docs/architecture.md).

<!-- JARVIS:BEGIN hash=ws-pld-root -->
## Jarvis MCP (project)

**project_id**: `pld` (PLD workspace)

Pasá este `project_id` al llamar tools de Jarvis que lo acepten.

### Active integrations
_(ninguna configurada)_
<!-- JARVIS:END -->
