# Tech Review: edicion-inline-resumen-wizard-v2

## Resumen del Propose

**Problema:**

## Contexto

El wizard de registro de sujeto obligado tiene un paso final de 'Revisión y validación' (ReviewStep) que muestra un resumen de todos los datos ingresados. Cada sección tiene un ícono de lápiz (editar). El comportamiento actual:
- En el wizard principal: navega al step correspondiente, sacando al usuario del resumen
- En el modal de actividad secundaria (AddVulnerableActivityModal): también navega al step dentro del modal

## Comportamiento deseado

Al hacer clic en el ícono de editar de cualquier sección del resumen, esa sección se reemplaza inline por el formulario de edición correspondiente, directamente dentro de la vista de Revisión y validación. El usuario edita ahí mismo, confirma, y vuelve a ver el resumen actualizado sin salir del paso de revisión.

## Alcance

### Frontend
- El componente ReviewStep es reutilizable — ya se usa tanto en el wizard principal como en el AddVulnerableActivityModal. La implementación inline aplica automáticamente a ambos contextos.
- Secciones con edición inline: Datos personales/identificación (PF y PM), Datos de contacto, Actividad vulnerable, Responsable del cumplimiento (solo PM)
- La sección 'Sujeto obligado' (tipo de sujeto + tipo de persona) queda read-only — cambiarla implicaría resetear toda la estructura del registro
- SecondActivityBlock (actividad secundaria ya registrada): mismas secciones editables inline
- Dentro del AddVulnerableActivityModal, en su paso de revisión, también aplica edición inline
- Reutilizar los step-components existentes (PhysicalIdentificationStep, ContactStep, VulnerableActivityStep, etc.) como formularios de edición inline
- Deshabilitar botón Confirmar del wizard mientras haya una sección en modo edición

### Backend
- Actualmente POST /:id/vulnerable-activity siempre crea una nueva actividad vulnerable (no hace upsert). Si el usuario edita desde el resumen, generaría un registro duplicado.
- Se necesita implementar PUT /:id/vulnerable-activity en el BE (NestJS) que actualice la actividad vulnerable existente del registration, incluyendo su dirección y divisiones.
- El resto de endpoints ya hacen upsert correctamente: saveIdentification, saveComplianceResponsible. Contacto ya tiene PUT /:id/contact/:contactId.

## Restricciones
- Reutilizar step-components existentes, no rediseñar formularios
- No romper el flujo actual de agregar actividad secundaria por primera vez
- No cambiar el flujo de navegación entre steps del wizard (pasos 1-4)
- No cambiar esquema DB — solo lógica de actualización en adapter

## Límites de la Solución (PM)

### Dentro del alcance

- FE: edición inline de secciones en ReviewStep — Identificación (PF/PM), Contacto, Actividad vulnerable, Responsable del cumplimiento
- FE: edición inline aplica automáticamente en wizard principal y AddVulnerableActivityModal (mismo componente ReviewStep)
- FE: SecondActivityBlock con edición inline de sus secciones
- FE: botón Confirmar deshabilitado mientras haya sección en modo edición
- FE: sección Sujeto obligado queda read-only
- BE: implementar PUT /:id/vulnerable-activity para actualizar actividad existente (dirección + divisiones)
- FE: agregar updateVulnerableActivity en registrationService apuntando al nuevo PUT

### Fuera del alcance

- Cambios en esquema DB
- Rediseño de formularios existentes
- Flujo agregar actividad secundaria por primera vez (sigue con modal)
- Navegación entre steps 1-4 del wizard

### Restricciones de negocio

- Reutilizar step-components existentes como formularios inline
- No romper flujo actual de AddVulnerableActivityModal
- Solo actualizar lógica en adapter BE, sin migraciones DB

## Revisión Técnica (DEV)

### Validación de límites

- undefined `undefined`

### Límites técnicos adicionales

- BE: nuevo endpoint PUT /admin/registration/:id/vulnerable-activity en registration.controller.ts + registration.service.ts + registration.adapter.ts
- FE ReviewStep: agregar estado local editingSection: string | null. Al hacer clic en lápiz, en lugar de llamar onEdit(stepKey), se setea editingSection=stepKey
- FE ReviewStep: cuando editingSection !== null, la sección correspondiente renderiza el step-component en lugar del bloque de resumen
- FE ReviewStep: el step-component inline recibe el valor actual del state y un onSave(updatedValue) que persiste y limpia editingSection
- FE ReviewStep: necesita recibir registrationId y profileType como props adicionales para poder llamar a registrationService
- FE ReviewStep: botón Confirmar del wizard padre se deshabilita cuando editingSection !== null — requiere exponer editing state hacia arriba vía callback onEditingChange o prop
- FE registrationService: agregar método updateVulnerableActivity(id, dto) → PUT /:id/vulnerable-activity
- SecondActivityBlock también necesita editingSection propio ya que tiene secciones independientes de la actividad principal

### Approach propuesto

Agregar estado editingSection a ReviewStep. Cada sección con lápiz, en lugar de llamar onEdit(stepKey) del padre, setea editingSection localmente. La sección en edición renderiza el step-component correspondiente reutilizando el componente existente. Al guardar, llama al servicio y vuelve a modo resumen. El botón Confirmar del wizard se deshabilita via prop/callback mientras editingSection !== null. BE: nuevo PUT vulnerable-activity en NestJS que busca la actividad existente por workspace y la actualiza junto con su dirección y divisiones.

### Estimación


## Estado

finalized
