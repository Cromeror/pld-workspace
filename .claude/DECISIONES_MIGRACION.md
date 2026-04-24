# Decisiones de Migración — Persistentes

> **Ámbito**: BE-only. Contenido específico del sub-repo `pld-api/` (migración de `libs/` acoplados a NestJS → `packages/` puros). Vivía originalmente en `pld-api/.claude/DECISIONES_MIGRACION.md` y fue movido al workspace el 2026-04-24 como parte de la consolidación de contexto Claude.
>
> **Paths**: todas las referencias a archivos del BE se expresan desde el root del workspace como `pld-api/...`.
>
> **Nota original**: archivo interno ignorado por git. Registra reglas de comportamiento que Claude debe seguir durante la migración a packages, sin tener que preguntarlas en cada sesión.

---

## Regla 1 — Preferencia `packages/` sobre `libs/` en código migrado

**Regla**: Cuando se toque código (nuevo o existente que se esté migrando), los imports de enums, types, constantes, y utilidades deben usar **`@pld-api/<package>`** (de `packages/`), NO `@pld-api/<lib>` (de `libs/`), si existe el equivalente migrado.

**Equivalencias actuales**:

| En vez de (libs) | Usar (packages) |
|---|---|
| `@pld-api/core` (scalars UUID, Email, etc.) | `@pld-api/shared-types` |
| `@pld-api/catalogs` (enums UserRole, Nacionalidad, etc.) | `@pld-api/shared-types` |
| `@pld-api/address-contact-id` (Domicilio, Contacto, TipoIdentificacion) | `@pld-api/shared-types` |
| `@pld-api/persistencia-sql` (eliminada) | `@pld-api/persistence` |

**Aplica a**: imports nuevos, refactors, dominios migrados a `packages/domain-*`, código que el usuario pida mover.

**No aplica a**: código legacy que no se está tocando en la tarea actual (patrón "edición cero").

---

## Regla 2 — Preservar características 1:1 al migrar endpoints

**Regla**: Cuando se migra un endpoint de una app a otra (ej. consolidación), debe mantener **exactamente**:

- Método HTTP y ruta relativa.
- Shape de request (DTO, validaciones, decoradores).
- Shape de response (campos, orden, tipos).
- Códigos HTTP de éxito y error.
- Headers relevantes.
- Comportamiento de auth (público vs protegido con JWT).

**Acción**: antes de consolidar/migrar un endpoint, verificar estas 6 características en el origen y reflejarlas fielmente en el destino.

**No aplica a**: la URL base/prefix del servidor, que sí puede cambiar si se consolidan servicios (ej. `9002/pld-api/catalogs/...` → `9001/pld-api/auth-users/catalogs/...`). Ese cambio se comunica como parte del change.

---

## Regla 3 — Detección de imports `libs/` en código tocado → preguntar

**Regla**: Si durante una migración se encuentran imports que siguen apuntando a `libs/` (en vez del equivalente de `packages/`), **NO actuar silenciosamente**. Preguntar al usuario:

- ¿Se migra ese import también (alcance se expande)?
- ¿Se deja como está (riesgo: el código queda a medio camino)?
- ¿Se escala el change a incluir la migración?

**Razón**: evitar cambios ocultos que el usuario no aprobó y mantener cambios pequeños y revisables.

---

## Regla 4 — Checklist por tarea

Antes de escribir/modificar código en una tarea de migración, Claude debe:

1. Listar imports `@pld-api/*` del origen.
2. Marcar cuáles tienen equivalente en `packages/` (Regla 1).
3. Avisar al usuario si encuentra casos no obvios antes de proceder.
4. Aplicar Regla 2 (preservar características).
5. Validar post-cambio: el endpoint responde igual + login sigue OK (regla global del proyecto).

---

## Histórico de decisiones previas

- **2026-04-20** — Descartado `packages/shared-crypto`. `hashPassword`, `verifyPassword`, `generateSecurePassword` se migrarán a `packages/domain-auth-users` en Fase 4.
- **2026-04-20** — Descartada Fase 1 (unificar apps en `apps/api`). Las 3 apps coexisten.
- **2026-04-20** — 16 archivos scaffold vacíos de Nx restaurados. Se mantienen como placeholders futuros.
- **2026-04-20** — Backend SDD: `openspec` (archivos en `openspec/`).
- **2026-04-20** — `libs/catalogs` y `libs/address-contact-id` quedan como libs puente (re-export) mientras no se migren sus consumidores.
- **2026-04-20** — `HttpErrorInterceptor`: mover a `apps/<gateway>/src/shared/` en cada app que lo use (auth-users, cross) y dejar de importarlo desde `@pld-api/core`. Es código NestJS complementario a `@pld-api/shared-errors` (no lo reemplaza). Se aplica como parte de la consolidación de `catalogs` (Fase 2) para auth-users; cross se ajusta en paralelo.
