# Skill Registry — pld workspace

Registro resuelto en SDD init. El orquestador inyecta las compact rules que correspondan en los prompts de sub-agentes.

Convención de paths: TODOS los paths en este documento son absolutos desde el root del workspace (`/home/cristobal/work/pld/`). Ejemplo: `pld-api/libs/catalogs/src/lib/catalogs.types.ts`, no `libs/catalogs/...`.

---

## User Skills (trigger table)

Skills a nivel usuario (`~/.claude/skills/`). Filtradas para excluir `sdd-*`, `_shared` y `skill-registry`.

| Skill | Trigger context | Path |
|-------|-----------------|------|
| branch-pr | Crear PR / preparar branch para review. Valida issue-first enforcement. | ~/.claude/skills/branch-pr/SKILL.md |
| issue-creation | Crear issue de GitHub (bug o feature). Valida templates y labels. | ~/.claude/skills/issue-creation/SKILL.md |
| judgment-day | Review adversarial con dos jueces paralelos. Trigger: "judgment day", "doble review", "juzgar". | ~/.claude/skills/judgment-day/SKILL.md |
| humanizer | Eliminar signos de texto AI-generado. Trigger: editar prompts, docs públicos, READMEs. | ~/.claude/skills/humanizer/SKILL.md |
| skill-creator | Crear nuevo skill siguiendo el spec. Trigger: "crear skill", "add skill". | ~/.claude/skills/skill-creator/SKILL.md |

## Project Skills

Skills a nivel proyecto (`pld/.claude/skills/`):

| Skill | Trigger context | Path |
|-------|-----------------|------|
| stack-up | Levantar el stack dev (BE docker + Web Vite). Flags: `--no-be`, `--no-web`, `--rebuild`. | pld/.claude/skills/stack-up/SKILL.md |
| stack-down | Bajar el stack dev. Flags: `--no-be`, `--no-web`, `--volumes` (borra mysql data). | pld/.claude/skills/stack-down/SKILL.md |
| stack-status | Reportar estado: `docker compose ps` + PID y health HTTP de Web. Flags: `--be`, `--web`. | pld/.claude/skills/stack-status/SKILL.md |
| stack-logs | Logs de BE (docker) y Web (tail del log Vite). Flags: `-f`, `--tail N`, `--be [svc]`, `--web`. | pld/.claude/skills/stack-logs/SKILL.md |

## Project Conventions

- **Workspace CLAUDE.md**: [pld/CLAUDE.md](../CLAUDE.md) — Jarvis `project_id: pld`, sin integraciones activas.
- **Arquitectura dev local**: [pld/docs/architecture.md](../docs/architecture.md) — topología workspace, flujos web↔API, utilidades locales.
- **Docs de diseño del BE** (movidos desde `pld-api/temp-docs/`): [pld/docs/](../docs/) — PLAN_REORGANIZACION_MONOREPO, DIAGNOSTICO_SERVICIO_USUARIOS, FLUJO_LOGIN, FLUJO_REGISTRO_SUPERADMIN, REGISTRO_SCHEMA, CHECKLIST_DUDAS_PENDIENTES.
- **Contexto de migración BE** (archivos de trabajo vivo, BE-only): [pld/.claude/CHECKLIST_MIGRACION.md](../.claude/CHECKLIST_MIGRACION.md) y [pld/.claude/DECISIONES_MIGRACION.md](../.claude/DECISIONES_MIGRACION.md).
- **SDD**: `pld/openspec/` es el único openspec del workspace. Los sub-repos no tienen el suyo propio.
- **Regla anulada**: la antigua regla de `pld-api` ("docs van a `temp-docs/`, no `docs/`") YA NO APLICA. Todo lo de `temp-docs` se movió a `pld/docs/` del workspace.

---

## Compact Rules

### branch-pr
**Aplica cuando**: se crea un PR en `pld-api/` o `pld-web/`.
- El PR body declara qué sub-repo se toca. Cambios cross-repo requieren links PR coordinados.
- Nunca `git push --force` a `main` / `master`.
- Rama principal de `pld-api`: `main`. Rama activa de trabajo: `develop`.
- Antes de abrir PR en `pld-api`: remover `accessToken` de Nx Cloud en `pld-api/nx.json` si aún está commiteado (secreto).

### issue-creation
**Aplica cuando**: se crea un bug/feature issue.
- Indicar sub-repo afectado (`pld-api` vs `pld-web`) en el título o labels.
- Incluir pasos de reproducción y ambiente (dev local con `docker-compose.dev.yml`, etc.).

### judgment-day
**Aplica cuando**: el usuario pide review adversarial dual ("judgment day", "juzgar").
- Lanzar dos sub-agentes juez independientes en paralelo; sintetizar; iterar hasta 2 rondas; escalar si sigue fallando.

### humanizer
**Aplica cuando**: se edita prosa de usuario (docs, PR descriptions, issue bodies).
- Remover em-dash abusivos, rule-of-three, atribuciones vagas, simbolismos inflados.

### stack-up / stack-down / stack-status / stack-logs
**Aplica cuando**: se arranca/detiene/inspecciona el stack dev local del workspace (BE docker + Web Vite).
- SIEMPRE ejecutar desde el root del workspace (`/home/cristobal/work/pld/`). Las skills rechazan si se corren desde otro CWD.
- BE = `docker compose -f pld-api/docker-compose.dev.yml ...`. Servicios: `mysql` (13306), `auth-users` (9001). `cross` queda comentado en el compose (si se necesita, editar el YAML).
- Web = Vite (`yarn dev` en `pld-web/`) como proceso background. PID en `pld/.stack/web.pid`, log en `pld/.stack/web.log`. Directorio `.stack/` gitignoreado.
- Puerto de Vite: 4200 (fijo por `strictPort: true` en `pld-web/vite.config.ts`). Si se cambia, actualizar `stack-status` y `stack-logs` y esta regla.
- Validación post `stack-up`: `curl http://localhost:9001/pld-api/auth-users/docs` debe devolver 200 (Swagger del BE). `curl http://localhost:4200` debe devolver 200 (Vite).
- `/stack-down --volumes` BORRA la DB local — usar solo cuando se quiere reset total.

---

### BE-only rules — pld-api (Nx 14 / NestJS 9 / TypeORM / MySQL)

> **Ámbito**: las reglas de esta sección aplican SOLO al trabajar dentro de `pld-api/`. Contenido originalmente vivido en `pld-api/.atl/skill-registry.md`. Los archivos [pld/.claude/CHECKLIST_MIGRACION.md](../.claude/CHECKLIST_MIGRACION.md) y [pld/.claude/DECISIONES_MIGRACION.md](../.claude/DECISIONES_MIGRACION.md) también son BE-only.

**Aplica cuando**: se toca código bajo `pld-api/apps/`, `pld-api/libs/`, `pld-api/packages/`, `pld-api/tools/`, `pld-api/nx.json`, `pld-api/tsconfig.base.json`, `pld-api/package.json`, `pld-api/pnpm-workspace.yaml`.

**Para sdd-apply / sdd-verify / judgment-day**:
- **Validación crítica**: tras cualquier cambio en el BE, `POST http://localhost:9001/pld-api/auth-users/auth/login` con `admin@pld.com` / `Admin123!` debe devolver HTTP 201 + JWT. Script de validación vive en `pld/.claude/CHECKLIST_MIGRACION.md`.
- **Separación libs/ vs packages/**: `pld-api/libs/` usa decoradores NestJS (`@Module`, `@Injectable`). `pld-api/packages/` es código puro — NO debe importar `@nestjs/*`.
- **Patrón estructural**: controller → adapter → service. Respetar al migrar.
- **Entities TypeORM**: permanecen en sus libs/packages de dominio. La config de conexión vive en `pld-api/packages/persistence`.
- **Tests**: Jest configurado pero sin tests escritos. `passWithNoTests: true` oculta la ausencia. Al migrar, empezar a agregar tests.
- **Preferencia `packages/` sobre `libs/`** en código migrado (ver Regla 1 de DECISIONES_MIGRACION.md):
  - En vez de `@pld-api/core` → usar `@pld-api/shared-types`.
  - En vez de `@pld-api/catalogs` → usar `@pld-api/shared-types`.
  - En vez de `@pld-api/address-contact-id` → usar `@pld-api/shared-types`.
  - `@pld-api/persistencia-sql` ya fue eliminado → usar `@pld-api/persistence`.

**Para branch-pr / issue-creation (BE-only)**:
- `pld-api/docker-compose.yml` (prod) y `pld-api/docker-compose.dev.yml` (dev aislado, MySQL en puerto host `13306`).
- Antes de PR: remover `accessToken` de Nx Cloud en `pld-api/nx.json:17` (secreto commiteado).

---

### Web-only rules — pld-web (Vite 7 / React 19 / TS 5.9 / Tailwind 4 / PrimeReact / React Query)

**Aplica cuando**: se toca código bajo `pld-web/src/`, `pld-web/public/`, `pld-web/package.json`, `pld-web/vite.config.ts`, `pld-web/eslint.config.js`, `pld-web/tsconfig*.json`.

- **Stack fijo**: React 19, TS 5.9, Vite 7, Tailwind 4, PrimeReact 10, React Query 5, Zustand 5, React Hook Form 7 + Yup, react-router 7.
- **HTTP client**: axios. Ver [pld-web/src/services/](../../pld-web/src/services/) para la base config.
- **State**: React Query para server state, Zustand para client state global.
- **Validación de formularios**: React Hook Form + @hookform/resolvers + yup. No mezclar con zod.
- **Comunicación con BE**: la web consume `http://localhost:9001/pld-api/auth-users/*` en dev. Ver [pld/docs/architecture.md](../docs/architecture.md) § Contratos API→Web para endpoints y shapes.
- **Package manager**: yarn (yarn.lock presente). No mezclar con npm/pnpm.
- **Lint/format**: eslint 9 + typescript-eslint 8. Reglas propias de React (eslint-plugin-react-hooks, react-refresh).

---

### Cross-repo rules (API ↔ Web)

**Aplica cuando**: un change toca AMBOS sub-repos, o cuando se agrega/modifica un endpoint en `pld-api` que debe ser consumido desde `pld-web`.

- **Contrato cambia en BE → web debe actualizarse en el mismo change SDD**. No mergear el BE sin actualizar la web si hay breaking change.
- **Shapes de respuesta** (DTOs de request/response del BE) deben quedar reflejados en los types de `pld-web/src/types/` o `pld-web/src/services/`.
- **Base URL** en dev: `http://localhost:9001/pld-api/auth-users` (ver [pld/docs/architecture.md](../docs/architecture.md)).
- **Swagger del BE**: `http://localhost:9001/pld-api/auth-users/docs` — fuente de verdad de los contratos.
- **Auth**: JWT por header. La web guarda el token en... (definir al implementar; típicamente Zustand + localStorage).

---

### Workspace (coordinador) — always applies

- Claude siempre se abre desde `/home/cristobal/work/pld/`. Los sub-repos NO tienen capa Claude propia.
- `pld-api/` y `pld-web/` son repos git independientes con sus propios remotes. El root del workspace es un repo git SEPARADO (master, sin remote) que trackea SOLO la capa de coordinación.
- Los sub-repos están gitignoreados a nivel workspace. Nunca `git add pld-api/` ni `git add pld-web/` desde el root.
- Nunca commitear secretos. `.env*` gitignoreado en root y en sub-repos.
- SDD (`openspec/`) es singular: vive en el root del workspace.
- `docs/architecture.md` describe la arquitectura de **dev local** (topología, flujos, utilidades de debug locales). NO es la arquitectura productiva del proyecto — la real está documentada en otros lados (Confluence, wiki interna) y ESTE doc la respeta, solo agrega utilidades locales.
