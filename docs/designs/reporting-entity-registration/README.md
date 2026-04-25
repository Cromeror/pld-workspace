# Diseños — Registro de Sujeto Obligado

Screenshots y assets de UI/UX para la feature de registro de sujeto obligado (`/admin/reporting-entity/register`).

## Convenciones de naming

Cada screenshot dentro de su carpeta usa el patrón:

```
<estado>--<descripcion-corta>.png
```

Ejemplos:
- `empty.png` — estado inicial sin datos.
- `accordion-collapsed.png` — accordions cerrados.
- `accordion-expanded-notario.png` — Notario expandido.
- `pf-selected.png` — opción Persona física seleccionada.
- `with-data-filled.png` — formulario lleno con datos válidos.
- `error-validation.png` — mostrando errores de validación.
- `error-server-409.png` — mostrando error de servidor.
- `loading.png` — durante request.

Cuando un step tiene variantes por `profileType` (PF vs PM), van en sub-carpetas.

## Estructura

```
reporting-entity-registration/
├── README.md                              ← este archivo
├── shared/                                ← componentes transversales del wizard
│   └── (breadcrumb, stepper, layout general)
├── step-1-reporting-entity/               ← Step 1: Sujeto obligado (Notario/Inmobiliaria + PF/PM)
├── step-2-identification/
│   ├── persona-fisica/                    ← form PF (firstName, lastName, RFC 13, CURP, etc.)
│   └── persona-moral/                     ← form PM (corporateName, RFC 12, sin CURP, etc.)
├── step-3-contact/                        ← Datos de contacto (1+ filas dinámicas)
├── step-4-vulnerable-activity/            ← Actividad vulnerable + domicilio
├── step-pm-compliance-responsible/        ← (Solo PM) Responsable del cumplimiento. Si NO existe en el diseño, dejar la carpeta vacía con un .keep
├── step-5-review/
│   ├── persona-fisica/                    ← Resumen + Validar (PF)
│   └── persona-moral/                     ← Resumen + Validar (PM)
└── result-page/                           ← Pantalla post-finalize (tempPassword)
```

## Estados típicos a documentar por step

Para cada step, droppear screenshots de los siguientes estados (los que existan en el diseño):

1. **Inicial / vacío** — al entrar al step sin datos.
2. **En proceso** — usuario interactuando, datos parciales.
3. **Completo** — todos los campos válidos, botón Siguiente habilitado.
4. **Validación fallida** — campos inválidos con errores inline.
5. **Loading** — durante POST async (botón en estado loading).
6. **Error de servidor** — banner de error de 4xx/5xx.
7. **Variantes por estado del Wizard** — si el step se ve distinto cuando se reanuda un draft, etc.

## Cómo trabajar con esta carpeta

1. El usuario droppea los screenshots respetando el naming.
2. Claude (en futuras sesiones) lee esta carpeta como fuente de verdad de UI.
3. Las decisiones de implementación citan paths de aquí (`docs/designs/reporting-entity-registration/step-3-contact/empty.png`) en el design.md del SDD change.
4. Si hay diseño Figma compartido, agregar URL en el README de cada step.

## Pendientes conocidos (al 2026-04-24)

- [x] Step 1 — Notario expandido con PF/PM seleccionable.
- [x] Step 2 PF — Datos de identificación con campos vacíos.
- [x] Step 3 — Vacío y con form de contacto agregado.
- [ ] Step 2 PM — pendiente.
- [ ] Step 4 — pendiente.
- [ ] Step 5 PF — pendiente.
- [ ] Step 5 PM — pendiente.
- [ ] Step PM Compliance Responsible — pendiente (puede no existir en el diseño y quedar integrado en otro step).
- [ ] Result page (tempPassword) — pendiente.
