# Step 2 (PM) — Datos de identificación

Form para Persona Moral. Campos del DTO `MoralIdentificationDto` del BE.

## Endpoint en `onNext`

`POST /admin/registration/:id/identification`
Body: `{ corporateName, incorporationDate? (YYYY-MM-DD), rfc (12 chars), nationalityCountry? }`

⚠ Diferencias clave con PF:
- Sin `firstName`, `paternalSurname`, `maternalSurname`.
- RFC de **12** caracteres, no 13.
- Sin `curp`.
- Sin `birthCountry` (no aplica para personas morales).

## Screenshots a documentar

- `empty.png` — al entrar al step.
- `with-data-valid.png` — datos válidos.
- `error-rfc-invalid.png` — RFC PM con formato inválido (12 chars).
- `loading.png`, `error-server.png` — análogos a PF.

## ⚠ Pendiente clave

¿El form PM incluye al "Responsable del cumplimiento" como sub-sección de este step, o es un step aparte (step PM compliance)? El stepper visible muestra solo 5 pasos para PM también — esto sugiere que el responsable se integra en algún step existente. Confirmar con el diseño.
