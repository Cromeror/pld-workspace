---
type: agent-reference
---

# Diseños — Registro de Sujeto Obligado

Screenshots y assets de UI/UX para `/admin/reporting-entity/register`.

Flujo BE: `docs/flujos/registro-sujeto-obligado/flujo.md`
Componentes FE: `pld-web/src/components/organisms/reporting-entity/`

When user provides an image path, read it directly — do not scan folders.

## Mapa de steps — Flujo principal

| Step | Variante | Archivo | Componente FE |
|---|---|---|---|
| 1 — Sujeto obligado | común | `paso-1-sujeto-obligado/common.jpg` | `ReportingEntityTypeStep/` |
| 1 — Sujeto obligado | PF Notario | `paso-1-sujeto-obligado/persona-fisica.jpg` | `ReportingEntityTypeStep/` |
| 1 — Sujeto obligado | PM Inmobiliaria | `paso-1-sujeto-obligado/persona-moral.jpg` | `ReportingEntityTypeStep/` |
| 2 — Identificación | PF | `paso-2-identificacion/persona-fisica.jpg` | `PhysicalIdentificationStep/` |
| 2 — Identificación | PM | `paso-2-identificacion/persona-moral.jpg` | `MoralIdentificationStep/` |
| 3 — Contacto | vacío | `paso-3-contacto/1.jpg` | `ContactStep/` |
| 3 — Contacto | con contacto | `paso-3-contacto/2.jpg` | `ContactStep/` |
| 4 — Actividad vulnerable | PF | `paso-4-actividad-vulnerable/persona-fisica.jpg` | `VulnerableActivityStep/` |
| 4 — Actividad vulnerable | PF+AV extra | `paso-4-actividad-vulnerable/persona-fisica-actividad-vulnerable.jpg` | `VulnerableActivityStep/` |
| 4 — Actividad vulnerable | PM | `paso-4-actividad-vulnerable/persona-moral.jpg` | `VulnerableActivityStep/` |
| 4 — Actividad vulnerable | PM+AV extra | `paso-4-actividad-vulnerable/persona-moral-con-actividad-vulnerable.jpg` | `VulnerableActivityStep/` |
| 5 — Responsable cumplimiento | PM | `paso-5-responsable-cumplimiento/1.jpg` | `ComplianceResponsibleStep/` |
| 5/6 — Revisión | PF Notario | `paso-5-revision-persona-fisica/notario.jpg` | `ReviewStep/` |
| 5/6 — Revisión | PF Notario+AV | `paso-5-revision-persona-fisica/notario-con-actividad-vulnerable.jpg` | `ReviewStep/` |
| 5/6 — Revisión | PF Inmobiliaria | `paso-5-revision-persona-fisica/inmobiliaria.jpg` | `ReviewStep/` |
| 5/6 — Revisión | PF Inmobiliaria+AV | `paso-5-revision-persona-fisica/inmobiliaria-con-actividad-vulnerable.jpg` | `ReviewStep/` |
| 5/6 — Confirmación | PF+PM | `paso-5-revision-persona-fisica/confirmacion.jpg` | `ConfirmDialog` |
| 6 — Revisión | PM | _(pendiente)_ | `ReviewStep/` |
| Result | éxito | `pagina-resultado/result.jpg` | `SuccessFinalizeModal/` |
| Result | error | `pagina-resultado/result-with-error.jpg` | `SuccessFinalizeModal/` |

## Mapa de steps — Modal "Agregar actividad vulnerable"

### Notario → Inmobiliaria PM (6 pasos)

| Paso | Estado | Archivo |
|---|---|---|
| 01 — Tipo de persona | vacío | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-01-tipo-persona.jpg` |
| 01 — Tipo de persona | PM seleccionado | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-01-tipo-persona-seleccionado.jpg` |
| 02 — Identificación | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-02-identificacion.jpg` |
| 03 — Contacto | vacío | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-vacio.jpg` |
| 03 — Contacto | lleno | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-lleno.jpg` |
| 04 — Actividad vulnerable | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-04-actividad-vulnerable.jpg` |
| 05 — Responsable cumplimiento | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-05-responsable-cumplimiento.jpg` |
| 06 — Revisión | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-06-revision.jpg` |

### Notario → Inmobiliaria PF (5 pasos)

Pasos 02 y 03 comparten diseño con la variante PM (ver carpeta `modal-agregar-actividad-vulnerable/`).

| Paso | Estado | Archivo |
|---|---|---|
| 01 — Tipo de persona | PF seleccionado | `modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-01-tipo-persona.jpg` |
| 02 — Identificación | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-02-identificacion.jpg` _(compartido)_ |
| 03 — Contacto vacío | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-vacio.jpg` _(compartido)_ |
| 03 — Contacto lleno | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-lleno.jpg` _(compartido)_ |
| 04 — Actividad vulnerable | vacío | `modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-04-actividad-vulnerable-vacio.jpg` |
| 04 — Actividad vulnerable | lleno | `modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-04-actividad-vulnerable-lleno.jpg` |
| 05 — Revisión | — | `modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-05-revision.jpg` |

### Inmobiliaria → Notaria PF (5 pasos)

Pasos 02 y 03 comparten diseño con la variante PM (ver carpeta `modal-agregar-actividad-vulnerable/`).

| Paso | Estado | Archivo |
|---|---|---|
| 01 — Tipo de persona | — | `modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-01-tipo-persona.jpg` |
| 02 — Identificación | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-02-identificacion.jpg` _(compartido)_ |
| 03 — Contacto vacío | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-vacio.jpg` _(compartido)_ |
| 03 — Contacto lleno | — | `modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-lleno.jpg` _(compartido)_ |
| 04 — Actividad vulnerable | vacío | `modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-04-actividad-vulnerable-vacio.jpg` |
| 04 — Actividad vulnerable | lleno | `modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-04-actividad-vulnerable-lleno.jpg` |
| 05 — Revisión | — | `modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-05-revision.jpg` |

## Pendientes de diseño

- [x] Step 1 — Notario/Inmobiliaria + PF/PM
- [x] Step 2 PF — Datos de identificación
- [x] Step 2 PM — Datos de identificación
- [x] Step 3 — Contacto
- [x] Step 4 — Actividad vulnerable + domicilio
- [x] Step 5 PM — Responsable del cumplimiento
- [x] Step 5/6 PF — Revisión (Notario, Inmobiliaria, con/sin AV, confirmación)
- [ ] Step 6 PM — Revisión Persona Moral
- [x] Modal AV — Notario→Inmobiliaria PM (6 pasos)
- [x] Modal AV — Notario→Inmobiliaria PF (5 pasos)
- [x] Modal AV — Inmobiliaria→Notaria PF (5 pasos)
- [x] Result page — éxito y error
