# Step 1 — Sujeto obligado

Decisión inicial: tipo de sujeto obligado (Notario/Inmobiliaria) × tipo de persona (PF/PM).

## Endpoint que dispara este step (en `onNext`)

`POST /admin/registration/`
Body: `{ profileType: 'PERSONA_FISICA'|'PERSONA_MORAL', userRole: 'NOTARIO'|'INMOBILIARIA', rfc: string }`

⚠ Pendiente confirmar: ¿el RFC se ingresa en este step o en el step 2? Los screenshots iniciales no muestran input de RFC en step 1, pero el BE lo requiere en `POST /`.

## Screenshots a documentar

- `accordion-collapsed.png` — ambos accordions (Notario, Inmobiliaria) colapsados, sin selección, botón Siguiente disabled.
- `accordion-expanded-notario.png` — Notario expandido mostrando opciones PF/PM.
- `accordion-expanded-inmobiliaria.png` — Inmobiliaria expandida mostrando opciones PF/PM.
- `notario-pf-selected.png` — Notario + PF seleccionado, Siguiente habilitado.
- `notario-pm-selected.png` — Notario + PM seleccionado.
- `inmobiliaria-pf-selected.png` — Inmobiliaria + PF.
- `inmobiliaria-pm-selected.png` — Inmobiliaria + PM.
- `with-rfc-input.png` (si aplica) — variante con input de RFC. Si el RFC se pide acá, captura cómo se integra al accordion.
- `error-409-duplicate-rfc.png` — UI mostrando "Ya existe un registro en progreso con este RFC".
- `modal-resume-existing.png` — modal preguntando "¿Continuar registro existente o empezar uno nuevo?".
