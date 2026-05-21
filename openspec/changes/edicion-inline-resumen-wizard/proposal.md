# Proposal: edicion-inline-resumen-wizard

## Problema

## Contexto

El wizard de registro de sujeto obligado tiene un paso final de 'Revisión y validación' (ReviewStep) que muestra un resumen de todos los datos ingresados. Cada sección tiene un ícono de lápiz (editar) que actualmente navega al step correspondiente, sacando al usuario del resumen. Para la actividad secundaria, abrir el modal AddVulnerableActivityModal completo.

El comportamiento deseado es diferente: al hacer clic en el ícono de editar de cualquier sección del resumen (principal o secundaria), en lugar de navegar al step o abrir el modal, se debe reemplazar inline esa sección con el formulario de edición correspondiente, directamente dentro de la vista de Revisión y validación. El usuario edita ahí mismo y confirma, volviendo a ver el resumen actualizado sin haber salido del paso 5.

## Alcance

### Actividad principal
Todas las secciones del resumen principal tienen botón de editar:
- Sujeto obligado (tipo de sujeto + tipo de persona)
- Datos personales / identificación (PF o PM)
- Datos de contacto
- Actividad vulnerable
- Responsable del cumplimiento (solo PM)

Al hacer clic en cualquiera de esos lápices, esa sección se reemplaza inline por el formulario de edición. El resto del resumen permanece visible. Al guardar/confirmar la edición, vuelve a mostrarse el resumen actualizado.

### Actividad secundaria
La actividad secundaria actualmente se agrega vía el modal AddVulnerableActivityModal. Una vez registrada, aparece en el resumen como SecondActivityBlock con sus propias secciones. El mismo comportamiento inline aplica para las secciones de la actividad secundaria: editar inline sin abrir el modal.

## Restricciones
- Solo frontend (React/TypeScript). Sin cambios en BE ni DB.
- No rediseñar los formularios existentes — reutilizar los step-components ya existentes (PhysicalIdentificationStep, ContactStep, VulnerableActivityStep, etc.) como formularios de edición inline.
- La persistencia al guardar la edición debe llamar al mismo servicio que ya usa cada step (registrationService.saveIdentification, saveContact, etc.).
- El flujo 'Confirmar' final del wizard solo se puede activar cuando no hay ninguna sección en modo edición.
- Mantener compatibilidad con el wizard modal de actividad secundaria (AddVulnerableActivityModal) que ya existe para agregar la actividad secundaria por primera vez.

## Estado

pm_tech_review
