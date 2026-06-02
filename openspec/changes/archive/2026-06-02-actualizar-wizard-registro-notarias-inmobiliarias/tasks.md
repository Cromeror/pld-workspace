# Tasks: actualizar-wizard-registro-notarias-inmobiliarias

## BE

- [x] T-BE-01: Actualizar `physical-identification.dto.ts` — `nombre`, `primerApellido`, `segundoApellido` requeridos; `rfc`, `curp`, `fechaNacimiento` opcionales; agregar campo `lugarNacimiento` opcional
- [x] T-BE-02: Actualizar `physical-person-profile.entity.ts` — agregar columna `lugarNacimiento` nullable
- [x] T-BE-03: Actualizar `contact.dto.ts` — `correoElectronico` y `celular` requeridos; `numeroTelefono` opcional
- [x] T-BE-04: Actualizar `vulnerable-activity.dto.ts` — `actividadVulnerable` requerido; todos los campos de domicilio opcionales
- [x] T-BE-05: Actualizar `registration.adapter.ts` — mapear `lugarNacimiento` del DTO a la entidad; ajustar lógica de campos opcionales

## FE

- [x] T-FE-01: Actualizar `schemas.ts` — schema Paso 1: `nombre`, `primerApellido`, `segundoApellido` requeridos; `rfc`, `curp`, `fechaNacimiento` opcionales; agregar `lugarNacimiento` opcional
- [x] T-FE-02: Actualizar `schemas.ts` — schema Paso 2: `correoElectronico` y `celular` requeridos; `numeroTelefono` opcional
- [x] T-FE-03: Actualizar `schemas.ts` — schema Paso 3: `actividadVulnerable` requerido; campos de domicilio opcionales
- [x] T-FE-04: Actualizar `PhysicalIdentificationStep/index.tsx` — quitar `required` de `rfc`, `curp`, `fechaNacimiento`; agregar campo `lugarNacimiento` opcional
- [x] T-FE-05: Actualizar `ContactStep/index.tsx` — marcar `correoElectronico` y `celular` como requeridos; quitar `required` de `numeroTelefono`
- [x] T-FE-06: Actualizar `VulnerableActivityStep/index.tsx` — marcar `actividadVulnerable` como requerido; quitar `required` de campos de domicilio
- [x] T-FE-07: Actualizar `ReportingEntityRegistrationPage.tsx` — ocultar Paso 4 cuando `tipoPersona === 'PERSONA_FISICA'`

## Testing

- [x] T-TEST-01: Verificar en FE que el stepper omite Paso 4 para PERSONA_FISICA y lo muestra para PERSONA_MORAL
- [x] T-TEST-02: Verificar que el wizard completa el registro exitosamente con solo los campos requeridos (nombre, apellidos, correo, celular, actividadVulnerable)
- [x] T-TEST-03: Verificar que el BE acepta el payload con `lugarNacimiento` y lo persiste, y también acepta el payload sin ese campo
