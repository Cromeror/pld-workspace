# Step 5 (PF) — Revisión y validación

Resumen de los datos cargados con opción de editar cada bloque + botón "Validar" que dispara `POST /finalize`.

## Endpoint en `onFinish`

`POST /admin/registration/:id/finalize`
Devuelve: `{ user, tempPassword: string (cleartext, una sola vez) }`

Tras éxito: `navigate('/admin/reporting-entity/register/result', { state: { tempPassword, userEmail } })`.

## Screenshots a documentar

- `summary-pf.png` — resumen completo de todos los datos PF (identificación, contactos, actividad vulnerable, domicilio).
- `edit-buttons.png` — botones de "Editar" por bloque que llevan al step correspondiente vía `goTo(key)`.
- `validate-button.png` — botón "Validar" (label final del Wizard).
- `loading.png` — durante el `POST /finalize`.
- `error-server.png` — error 4xx/5xx.
