# Spec: wizard-stepper

## Requirement: Paso 4 visible solo para Persona Moral

El wizard MUST omitir el paso "Responsable del cumplimiento de la Ley" cuando `profileType === INDIVIDUAL`.
El stepper visual MUST reflejar esta omisión mostrando exactamente 5 pasos para PF y 6 para PM.

#### Scenario: Wizard PF muestra 5 pasos

- GIVEN un administrador selecciona Notario (siempre PF) o Inmobiliaria Persona física
- WHEN navega por el wizard
- THEN el stepper muestra exactamente 5 pasos: Sujeto obligado, Datos de identificación, Datos de Contacto, Actividad vulnerable, Revisión y validación
- AND el paso "Responsable del cumplimiento de la Ley" NO aparece en el stepper

#### Scenario: Wizard PM muestra 6 pasos

- GIVEN un administrador selecciona Inmobiliaria Persona moral
- WHEN navega por el wizard
- THEN el stepper muestra 6 pasos incluyendo "Responsable del cumplimiento de la Ley" entre Actividad vulnerable y Revisión y validación
