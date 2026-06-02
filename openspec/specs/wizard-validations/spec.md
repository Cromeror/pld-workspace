# Spec: wizard-validations

## Requirement: Campos requeridos Paso 1 — Identificación PF

El sistema MUST exigir únicamente `nombre`, `primerApellido` y `segundoApellido` como requeridos.
Los campos `rfc`, `curp` y `fechaNacimiento` MUST ser opcionales; si se envían, MUST cumplir su formato respectivo (RFC 13 chars, CURP 18 chars, fecha YYYY-MM-DD).

#### Scenario: Registro exitoso sin RFC, CURP ni fecha de nacimiento

- GIVEN un administrador inicia wizard con tipo Notario, Persona física
- WHEN completa solo `nombre`, `primerApellido`, `segundoApellido` en Paso 1
- THEN el wizard avanza al Paso 2 sin errores de validación
- AND el BE persiste el registro con `rfc = NULL`, `curp = NULL`, `birthDate = NULL`

#### Scenario: RFC inválido cuando se proporciona

- GIVEN el campo RFC está visible y no es requerido
- WHEN el usuario ingresa un RFC con formato incorrecto
- THEN el FE muestra error de validación antes de permitir avanzar

---

## Requirement: Nuevo campo `lugarNacimiento` (PF, opcional)

El sistema MUST aceptar `lugarNacimiento` como campo opcional en Paso 1 para Persona física.
Si se proporciona, MUST persistirse en `physical_person_profile.birth_place`.

#### Scenario: Registro con `lugarNacimiento`

- GIVEN el administrador llena el campo "Lugar de nacimiento"
- WHEN confirma el wizard
- THEN el BE persiste `birth_place` con el valor ingresado

#### Scenario: Registro sin `lugarNacimiento`

- GIVEN el administrador deja vacío el campo "Lugar de nacimiento"
- WHEN confirma el wizard
- THEN el BE persiste `birth_place = NULL` sin error

---

## Requirement: Campos requeridos Paso 2 — Contacto

El sistema MUST exigir `correoElectronico` y `celular` (10 dígitos). Los campos `claveLada` y `numeroTelefono` MUST ser opcionales.

#### Scenario: Avance con solo correo y celular

- GIVEN el formulario de contacto está visible
- WHEN el usuario ingresa correo válido y celular de 10 dígitos, sin teléfono ni lada
- THEN el wizard avanza sin errores

#### Scenario: Celular con formato incorrecto

- GIVEN el campo celular está visible
- WHEN el usuario ingresa menos de 10 dígitos
- THEN el FE muestra error "El celular debe tener exactamente 10 dígitos"

---

## Requirement: Campos requeridos Paso 3 — Actividad vulnerable

El sistema MUST exigir únicamente la selección de `actividadVulnerable`. El domicilio completo y `fechaInicial` MUST ser opcionales.

#### Scenario: Avance sin domicilio

- GIVEN el formulario de actividad vulnerable está visible
- WHEN el usuario selecciona una actividad y no llena ningún campo de domicilio
- THEN el wizard avanza al siguiente paso sin errores
- AND el BE persiste la actividad con `reportingEntityAddressId = NULL`

#### Scenario: Domicilio parcial no bloquea el avance

- GIVEN el usuario no ha llenado campos de domicilio
- WHEN hace clic en Siguiente
- THEN el FE no muestra errores de campos de domicilio requeridos

---

## Requirement: `rfc` opcional en creación de draft

El sistema MUST aceptar la creación del draft de registro sin `rfc` para Persona física.

#### Scenario: Draft creado sin RFC

- GIVEN el wizard avanza desde Paso 1 sin RFC
- WHEN el FE llama `POST /admin/registration`
- THEN el BE crea el draft con `rfc = NULL` y retorna `201` con el `id`
