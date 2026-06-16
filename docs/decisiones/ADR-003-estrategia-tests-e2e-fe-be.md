---
type: agent-reference
date: 2026-06-15
status: aplicado
affects: pld-web/e2e/tests/, pld-api/apps/auth-users/src/**/*.e2e.spec.ts, docs/flujos/
---

# ADR-003 — Estrategia de tests e2e con trazabilidad FE ↔ BE

**Contexto**: las pruebas del proyecto parten de lo **funcional**. Cada flujo de
usuario se especifica como un bloque **Gherkin** en el header de su spec e2e del
FE (`pld-web/e2e/tests/.../*.spec.ts`). Ese Gherkin es la **fuente de verdad**
del comportamiento esperado y se usa para TDD.

El FE prueba la experiencia (wizard, validaciones de UI, modales). Pero hay
**casos que el FE no contempla** — atomicidad de transacciones, integridad
referencial, validación de payload a nivel API, estados de error del servidor —
que solo se pueden asegurar en el BE. Si esos casos fallan, el FE los descubre
tarde (en su propio e2e o en producción).

**Decisión**: el BE tiene sus propios tests **e2e** (`*.e2e.spec.ts`) que
**derivan del mismo Gherkin del FE** y cubren la **porción que le corresponde al
BE**, manteniendo **trazabilidad explícita** hacia los scenarios del FE.

## Objetivo

Que cuando el FE corra sus e2e, encuentre **menos errores de origen BE**, porque
el BE ya validó por endpoint la parte de cada flujo que es su responsabilidad.

## Flujo de trabajo (cada vez que escribimos un e2e de BE)

1. **Ir al Gherkin del FE** del flujo correspondiente (en `pld-web/e2e/tests/`).
2. **Clasificar cada scenario** por responsabilidad:
   - **FE-only** (UI): estado de botones, placeholders, dropdowns read-only,
     stepper, modales, reset del wizard, contador de reintentos. → NO van al BE.
   - **BE** (verificable por endpoint): persistencia paso a paso, validación de
     payload (400), reglas de negocio (409/410/403), atomicidad del finalize,
     efecto final en BD. → SÍ van al e2e del BE.
3. **Implementar en el BE** los scenarios BE, y además los **casos especiales que
   el FE no contempla** (rollback, FKs, dedupe, concurrencia).
4. **Anotar la trazabilidad**: cada test del BE referencia el/los scenario(s) del
   Gherkin que cubre (ver convención abajo).

## Convención de trazabilidad

En el e2e del BE, cada `describe`/`it` debe poder rastrearse al Gherkin:

- El `describe` por flujo usa el **mismo nombre de flujo** que el spec FE
  (`Notario PF`, `Inmobiliaria PF`, `Inmobiliaria PM`).
- Cada `it` referencia el scenario del Gherkin que materializa, citando su texto.
  Ejemplo:

  ```ts
  // Gherkin (notario-pf.spec.ts) → "Finalizar exitoso muestra credenciales"
  //   + porción BE: persiste usuario WORKSPACE_ADMIN y marca COMPLETED.
  it('happy path → draft COMPLETED + usuario WORKSPACE_ADMIN', async () => { ... });
  ```

- Los casos **BE-only** (sin scenario FE equivalente) se marcan como tales:

  ```ts
  // BE-only (el FE no lo contempla): atomicidad — finalize incompleto no deja basura.
  it('finalize SIN compliance → 409 y NO crea usuario ni completa el draft', ...);
  ```

## Convención de nombres de archivos de test (BE)

| Sufijo | Qué prueba | Toca DB real | Entra por HTTP |
|---|---|---|---|
| `*.spec.ts` | Unitario (mockea repos/deps) | No | No |
| `*.workspace.spec.ts` (y similares de integración) | Integración con BD directa (repos TypeORM / SQL) | Sí | No |
| `*.e2e.spec.ts` | E2E real: app Nest levantada, requests HTTP, verificación en BD | Sí | Sí |

> El sufijo debe usar punto (`.e2e.spec.ts`), no guion (`.e2e-spec.ts`), para que
> el `testMatch` por defecto de Jest (`**/?(*.)+(spec|test).[jt]s`) lo recoja.

## Dominios separados (no mezclar en un mismo spec)

Hay dos dominios de "registration" distintos; cada uno con su e2e:

| Dominio | Módulo | Ruta | Operador | Finalize |
|---|---|---|---|---|
| **Sujeto obligado** | `admin/registration/` | `/admin/registration` | SUPERADMIN | crea usuario `WORKSPACE_ADMIN` |
| **Cliente externo** | `registration/external-clients/` | `/registration/external-clients` | WORKSPACE_ADMIN | registra un cliente (no crea usuario) |

## Qué le toca al BE vs al FE (guía rápida)

| Tipo de scenario | Responsable |
|---|---|
| Qué muestra/oculta la UI, estado de botones, placeholders | FE |
| Stepper (5 vs 6 pasos), navegación, modales, reset | FE |
| Campo read-only / dropdown disabled | FE |
| Persistencia paso a paso (cada POST guarda) | BE |
| Validación de payload por endpoint (400 ante body inválido) | BE |
| Reglas de negocio (Notario solo PF, dedupe RFC, draft expirado) | BE |
| Atomicidad del finalize (rollback, no dejar basura) | BE |
| Efecto final en BD (status, FKs, usuario creado) | BE |
| Auth/roles (401/403) | BE (y FE en su capa de guard de ruta) |

## Estado de referencia (sujeto obligado, 2026-06-15)

- E2E BE: `pld-api/.../registration.e2e.spec.ts` — happy path de los 3 flujos
  (Notario PF, Inmobiliaria PF, Inmobiliaria PM con compliance) verificando BD,
  casos de fallo/atomicidad, **validación de payload por endpoint** (identificación,
  contacto, compliance, domicilio → 400) y **contacto múltiple** (N POSTs → N filas).
  Deriva de los Gherkins en
  `pld-web/e2e/tests/superadmin/{notario-pf,inmobiliaria-pf,inmobiliaria-pm}.spec.ts`.

### Decisión de negocio registrada (2026-06-15) — domicilio de actividad vulnerable

`street` (calle) y `exteriorNumber` (número exterior) estaban **opcionales en el
FE y obligatorios en el BE** (desalineación). Decisión: **son OBLIGATORIOS** (un
domicilio de actividad vulnerable incompleto no sirve para PLD). Acción tomada:
el FE se alineó al BE (`addressSchema` ahora los marca requeridos), el Gherkin de
los 3 flujos sumó esos campos al `Scenario Outline` de domicilio, y el e2e del BE
los cubre. Lección: cuando FE y BE difieren en si un campo es requerido, se
resuelve por decisión de negocio y se alinean ambos + el Gherkin.

### Pendientes documentados (it.todo en el spec)

Reglas de negocio del BE aún sin cubrir, listadas como `it.todo`: RFC duplicado
→ 409, draft expirado → 410, rol no-SUPERADMIN → 403, segundo POST identification
→ 200 (overwrite). No son de campos requeridos (eso quedó cerrado) sino de
estados/flujo. El "Paso 1 no persiste en BE" del Gherkin NO es testeable en BE:
es una selección de UI sin persistencia (no hay endpoint), queda como FE-only.

## Consecuencias

- Antes de escribir un e2e de BE, SIEMPRE se revisa el Gherkin del FE del flujo.
- Los nombres de flujo y la cita de scenarios mantienen la trazabilidad FE → BE.
- El FE encuentra menos errores de BE en sus e2e porque el BE ya cubrió su parte.
