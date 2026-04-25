# Reporting Entity Registration UI Specification

## Purpose

Wizard frontend (`/admin/reporting-entity/register`) que permite al SUPERADMIN registrar sujetos obligados (Notarías o Inmobiliarias) en perfil Persona Física o Persona Moral, con persistencia paso a paso al BE, resume vía localStorage y modales de confirmación + éxito + error con reintentos.

Implementación inicial en change archivado [2026-04-25-add-reporting-entity-registration-ui](../changes/archive/2026-04-25-add-reporting-entity-registration-ui/).

## Behavioral Invariants

### INV-1: Acceso restringido a SUPERADMIN

**Given** un usuario autenticado con rol distinto de SUPERADMIN,
**When** intenta navegar a `/admin/reporting-entity/register`,
**Then** el sistema SHALL redirigir a `/` sin renderizar el wizard.

### INV-2: Redirect post-login configurable por rol

**Given** un usuario que se loguea exitosamente con rol SUPERADMIN,
**When** el login completa,
**Then** el sistema SHALL navegar a `/admin/reporting-entity/register`.

**Given** un usuario que se loguea con rol NOTARIO, INMOBILIARIA o AUXILIAR,
**When** el login completa,
**Then** el sistema SHALL navegar a `/` (fallback DEFAULT_REDIRECT).

### INV-3: Step 1 sin persistencia BE

**Given** el usuario está en step 1,
**When** selecciona Notario+PF y clickea Siguiente,
**Then** el wizard SHALL avanzar a step 2 sin hacer ningún request al BE.

### INV-4: Step 2 dispara dos POSTs encadenados

**Given** step 2 con datos válidos y `registrationId === undefined`,
**When** el usuario clickea Siguiente,
**Then** el sistema SHALL ejecutar `POST /admin/registration/` primero,
**AND** SHALL guardar `registrationId` retornado en state y localStorage,
**AND** SHALL ejecutar `POST /:id/identification` después,
**AND** SHALL avanzar a step 3 si ambos retornan 2xx.

**Given** step 2 con datos válidos y `registrationId` ya seteado (reanudando),
**When** el usuario clickea Siguiente,
**Then** el sistema SHALL ejecutar solo `POST /:id/identification`.

### INV-5: RFC duplicado abre modal de resume

**Given** step 2 con un RFC ya en uso por otro draft IN_PROGRESS,
**When** el usuario clickea Siguiente y `POST /` retorna 409,
**Then** el sistema SHALL mostrar mensaje de error indicando que ya existe un registro
**AND** SHALL no avanzar al step 3.

### INV-6: Step 3 múltiples contactos

**Given** step 3 sin contactos,
**When** el usuario revisa el form,
**Then** el botón Siguiente SHALL estar disabled.

**Given** step 3 con al menos 1 contacto válido,
**When** el usuario clickea Siguiente,
**Then** el sistema SHALL ejecutar N `POST /:id/contact` (uno por contacto no persistido)
**AND** SHALL avanzar a step 4 si todos retornan 2xx.

### INV-7: Step PM compliance solo en flujo PM

**Given** flujo PF (`profileType = PERSONA_FISICA`),
**When** se renderiza el stepper,
**Then** el stepper SHALL mostrar 5 pasos
**AND** el step "Responsable del cumplimiento" NO SHALL estar visible.

**Given** flujo PM (`profileType = PERSONA_MORAL`),
**When** se renderiza el stepper,
**Then** el stepper SHALL mostrar 6 pasos incluyendo "Responsable del cumplimiento".

### INV-8: Confirmar abre modal antes de finalizar

**Given** step de Revisión con datos válidos,
**When** el usuario clickea "Confirmar",
**Then** el sistema SHALL abrir el ConfirmDialog "¿Estás seguro/a?"
**AND** SHALL no ejecutar `POST /finalize` todavía.

**Given** ConfirmDialog abierto,
**When** el usuario clickea "Cancelar",
**Then** el sistema SHALL solo cerrar el modal
**AND** el wizard SHALL permanecer en el step de Revisión.

**Given** ConfirmDialog abierto,
**When** el usuario clickea "Confirmar",
**Then** el sistema SHALL ejecutar `POST /:id/finalize`
**AND** mostrar loading durante la operación
**AND** abrir SuccessFinalizeModal si retorna 2xx
**AND** abrir ErrorFinalizeModal si retorna error.

### INV-9: Modal de éxito muestra password una sola vez

**Given** finalize exitoso,
**When** se abre el SuccessFinalizeModal,
**Then** el modal SHALL mostrar el `tempPassword` y el `email` del user creado.

**Given** SuccessFinalizeModal abierto,
**When** el usuario clickea "Finalizar",
**Then** el sistema SHALL limpiar localStorage,
**AND** SHALL resetear el wizard al step 1 con state vacío,
**AND** SHALL cerrar el modal,
**AND** la URL SHALL permanecer en `/admin/reporting-entity/register`.

### INV-10: Reintentos del modal de error

**Given** finalize falla,
**When** se abre el ErrorFinalizeModal por primera vez,
**Then** el botón "Intentar de nuevo" SHALL estar habilitado.

**Given** el usuario ha hecho 3 reintentos fallidos,
**When** se abre el ErrorFinalizeModal por cuarta vez,
**Then** el botón "Intentar de nuevo" SHALL estar disabled
**AND** el modal SHALL mostrar mensaje "Se alcanzó el número máximo de reintentos".

**Given** el usuario refresca la página tras 3 reintentos fallidos,
**When** la página re-carga,
**Then** el contador de reintentos SHALL resetearse a 0
**AND** el `registrationId` permanece en localStorage.

### INV-11: Resume con draft IN_PROGRESS

**Given** localStorage contiene `pld:activeRegistrationId` válido,
**When** el usuario carga la página `/admin/reporting-entity/register`,
**Then** el sistema SHALL ejecutar `GET /:id` para validar el draft.

### INV-12: Resume con draft COMPLETED (race condition)

**Given** localStorage contiene `pld:activeRegistrationId`,
**When** `GET /:id` retorna draft con `status = COMPLETED`,
**Then** el sistema SHALL limpiar localStorage silenciosamente
**AND** SHALL no mostrar banner de error,
**AND** el wizard SHALL arrancar en step 1 limpio.

### INV-13: Country dropdown visualmente activo no interactivo

**Given** step 4 (actividad vulnerable),
**When** el dropdown de país se renderiza,
**Then** SHALL mostrar "México" seleccionado con apariencia visual normal,
**AND** click en el dropdown NO SHALL abrir el overlay,
**AND** la selección NO SHALL cambiar.

### INV-14: Cellphone hack v1

**Given** step 3 con contactos,
**When** se ejecuta `POST /:id/contact` por cada contacto,
**Then** el body SHALL incluir `cellphone` con el mismo valor que `phone` del contacto correspondiente.

(Documentado como deuda técnica hasta que el diseño Figma agregue el input `cellphone` o el BE flexibilice la validación.)

### INV-15: Build limpio

**Given** la implementación es completa,
**When** se ejecuta `yarn tsc -b --noEmit`,
**Then** SHALL pasar con cero errores.

**When** se ejecuta `yarn build`,
**Then** SHALL producir un bundle de producción con cero errores.
