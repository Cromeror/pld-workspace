---
type: agent-reference
---

# Diseños — Registro de Sujeto Obligado

Screenshots y assets de UI/UX para `/admin/reporting-entity/register`.

Flujo BE: `docs/flujos/registro-sujeto-obligado/flujo.md`
Componentes FE: `pld-web/src/components/organisms/reporting-entity/`

When user provides an image path, read it directly — do not scan folders.

## Mapa de steps

| Step | Variante | Carpeta | Componente FE |
|---|---|---|---|
| 1 — Sujeto obligado | PF+PM | `paso-1-sujeto-obligado/` | `ReportingEntityTypeStep/` |
| 2 — Identificación | PF | `paso-2-identificacion/persona-fisica/` | `PhysicalIdentificationStep/` |
| 2 — Identificación | PM | `paso-2-identificacion/persona-moral/` | `MoralIdentificationStep/` |
| 3 — Contacto | PF+PM | `paso-3-contacto/` | `ContactStep/` |
| 4 — Actividad vulnerable | PF+PM | `paso-4-actividad-vulnerable/` | `VulnerableActivityStep/` |
| 5 — Responsable cumplimiento | PM | `paso-5-responsable-cumplimiento/` | `ComplianceResponsibleStep/` |
| 5/6 — Revisión | PF | `paso-5-revision-persona-fisica/` | `ReviewStep/` |
| 6 — Revisión | PM | _(sin diseño aún)_ | `ReviewStep/` |
| Result modal | PF+PM | `pagina-resultado/` | `SuccessFinalizeModal/` |

## Convenciones de naming

Cada screenshot usa el patrón `<estado>--<descripcion-corta>.jpg`. Ejemplos:
- `empty.jpg` — estado inicial sin datos
- `with-data-filled.jpg` — formulario lleno con datos válidos
- `error-validation.jpg` — campos inválidos con errores inline
- `loading.jpg` — durante POST async

Cuando un step tiene variantes PF/PM, van en sub-carpetas.

## Estados típicos por step

1. **Inicial / vacío** — al entrar sin datos
2. **Completo** — todos los campos válidos, botón Siguiente habilitado
3. **Validación fallida** — errores inline visibles
4. **Loading** — botón en estado loading durante POST
5. **Error de servidor** — banner 4xx/5xx

## Pendientes

- [x] Step 1 — Notario/Inmobiliaria + PF/PM
- [x] Step 2 PF — Datos de identificación
- [x] Step 2 PM — Datos de identificación
- [x] Step 3 — Contacto
- [x] Step 4 — Actividad vulnerable + domicilio
- [x] Step 5 PM — Responsable del cumplimiento
- [x] Step 5 PF — Revisión
- [ ] Step 6 PM — Revisión
- [ ] Result page — tempPassword modal
