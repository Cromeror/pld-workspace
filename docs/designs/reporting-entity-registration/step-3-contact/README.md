# Step 3 — Datos de contacto

Form dinámico con N filas de contacto. Mínimo 1 contacto requerido.

## Endpoint en `onNext`

`POST /admin/registration/:id/contact` (una llamada por contacto nuevo).
Body: `{ countryCode?, phone?, email, cellphone }`

⚠ Hack v1 (Q3 cerrada): la UI tiene 3 inputs (`Clave lada`, `Numero de teléfono`, `Correo electrónico`). El BE requiere `cellphone` que no aparece. Mientras no se actualice el diseño, el FE manda el mismo valor de `phone` también como `cellphone`.

## Screenshots a documentar

- `empty.png` — sin contactos, solo botón "+ Agregar contacto" visible. (✅ ya enviado)
- `one-row-empty.png` — primera fila agregada, inputs vacíos. (✅ ya enviado)
- `one-row-filled.png` — primera fila llena, Siguiente habilitado. (✅ ya enviado)
- `multiple-rows-filled.png` — 2+ contactos llenos.
- `error-email-invalid.png` — error inline en correo electrónico.
- `error-required.png` — fila incompleta.
- `delete-button-hover.png` — hover/click sobre el botón de basurero.
- `loading.png` — durante los N POSTs.
- `partial-save-error.png` — escenario donde algunos contactos se guardaron y otros fallaron.
