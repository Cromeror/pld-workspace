# Step PM — Responsable del cumplimiento (solo Persona Moral)

⚠ **Pendiente confirmar si este step existe como tal en el diseño**.

El BE tiene endpoint `POST /admin/registration/:id/compliance-responsible` que solo aplica para PM. Los flowcharts del SDD lo describen como step propio. Pero el stepper visible (5 pasos para ambos PF y PM) sugiere que la UI lo integra en otro lugar.

Posibles ubicaciones según el diseño:
1. **Step propio invisible en stepper** — aparece solo cuando `profileType=PM` pero no suma al contador de pasos.
2. **Sub-sección del Step 2 (Identificación PM)** — el form de PM incluye al responsable como expandible.
3. **Sub-sección del Step 5 (Revisión PM)** — el responsable se completa en revisión.

## Endpoint si es step propio

`POST /admin/registration/:id/compliance-responsible`
Body: `{ firstName, paternalSurname, maternalSurname, birthDate?, rfc, curp, nationalityCountry?, designationDate? }`

## Screenshots a documentar

Si el diseño confirma que es step propio:
- `empty.png`, `with-data-valid.png`, errores, loading, etc.

Si NO existe como step y se integra en otro:
- Dejar este README como nota para el equipo y poner los screenshots correspondientes en `step-2-identification/persona-moral/` o `step-5-review/persona-moral/`.
