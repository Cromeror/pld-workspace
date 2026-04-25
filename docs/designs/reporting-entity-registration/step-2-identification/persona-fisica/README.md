# Step 2 (PF) — Datos de identificación

Form para Persona Física. Campos del DTO `PhysicalIdentificationDto` del BE.

## Endpoint en `onNext`

`POST /admin/registration/:id/identification`
Body: `{ firstName, paternalSurname, maternalSurname, birthDate (YYYY-MM-DD), rfc (13 chars), curp (18 chars), nationalityCountry?, birthCountry? }`

## Screenshots a documentar

- `empty.png` — al entrar al step, todos los campos vacíos. (✅ ya enviado en chat)
- `with-data-valid.png` — todos los campos llenos con datos válidos, Siguiente habilitado.
- `error-rfc-invalid.png` — RFC con formato inválido, error inline.
- `error-curp-invalid.png` — CURP con formato inválido.
- `error-required-fields.png` — usuario intenta avanzar sin completar.
- `loading.png` — POST en progreso.
- `error-server.png` — error 500 o validación BE.
