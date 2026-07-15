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

## Documentación de flujos (`docs/flujos/`)

Aplica al crear o actualizar flujos en `docs/flujos/`. Estructura plana — sin subcarpetas por flujo.

- Cada flujo es un archivo `<nombre-flujo>.md` en la raíz de `docs/flujos/` (ej. `inicio-sesion.md`, `registro-auxiliar.md`).
- El diagrama drawio acompaña al md con el mismo nombre: `<nombre-flujo>.drawio`.
- `<nombre-flujo>.md` es la **fuente de verdad del sistema completo** — describe el flujo end-to-end incluyendo pasos de usuario, validaciones, lógica BE y referencias a diseños UI.
- El toon en el `.md` representa el flujo del sistema; puede referenciar tanto decisiones de BE (endpoints, tablas) como de UI (pantallas, estados visuales). Las notas e inconsistencias del llm-index son la guía para alinear implementación y diseño.
- Decisiones técnicas relevantes al flujo → crear ADR en `docs/decisiones/`.

### Mockups de diseño — recepción y versionado (tool `pld_design_receiver`)

Los diseños UI (mockups, screenshots de Figma, etc.) YA NO se archivan a mano en
`docs/flujos/disenos/`. Se reciben y versionan con la tool de Jarvis
`pld_design_receiver`, que las guarda en `docs/design/<slug>/vN/`.

**Cuándo usarla**: cuando el usuario adjunta una o más imágenes en el chat y el mensaje
indica que es un diseño/mockup de un flujo (ej. "te paso el diseño de X", "nueva versión
de Y", "acá están las pantallas de Z"). Disparadores típicos: "diseño", "mockup",
"pantallas", "flujo de <nombre>", "nueva versión de <nombre>", o simplemente imágenes
adjuntas sin más contexto en una conversación sobre un flujo de negocio.

**Cómo invocarla**:
1. Normalizar lo que el usuario nombró a un slug kebab-case, ej. "flujo cliente externo
   fiduciaria" → `flujo-cliente-externo-fiduciaria`. Si el usuario ya usó ese mismo nombre
   antes, reusar el MISMO slug (no crear uno nuevo aunque la redacción varíe un poco) para
   que las imágenes se acumulen como versiones del mismo diseño.
2. Si el payload (imágenes en base64) es grande, escribir un archivo temporal con el JSON
   de input y usar `--input-file`; si es chico, `--input` inline sirve.
3. Ejecutar:
   ```
   jarvis tool run pld_design_receiver --input-file '<path-al-json>'
   ```
   con:
   ```json
   {
     "design_slug": "<slug>",
     "images": [{ "filename": "...", "content_base64": "..." }],
     "project_id": "pld"
   }
   ```
4. **Cada llamada crea una versión nueva** (`v1`, `v2`, ...) con TODAS las imágenes de esa
   entrega — nunca mezcla ni pisa una versión anterior. No hay forma de "agregar a v1"
   después de creada; una entrega complementaria siempre es una versión nueva que debe
   incluir todo lo que se quiere que tenga esa versión (no solo el delta).
5. Confirmarle al usuario qué versión se creó y con cuántas imágenes (la tool ya devuelve
   ese resumen en texto — reportalo tal cual).

Esta carpeta (`docs/design/`) es el insumo para `pld_design_converter` (fase 2, en
desarrollo), que leerá una versión y armará el Gherkin correspondiente para
`pld-web/e2e/tests/`.

<!-- JARVIS:BEGIN hash=ws-pld-root -->
## Jarvis MCP (project)

**project_id**: `pld` (PLD workspace)

Pasá este `project_id` al llamar tools de Jarvis que lo acepten.

### Active integrations
_(ninguna configurada)_
<!-- JARVIS:END -->

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
