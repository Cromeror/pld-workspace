# Flujo: Actividad Secundaria (segunda actividad vulnerable)

## Resumen

Flujo iniciado desde el wizard de registro de sujeto obligado cuando el usuario presiona "+ Agregar actividad vulnerable" en el paso 4. Crea un registro independiente en el BE y lo enlaza al registro principal al finalizar.

## Actores

- **SUPERADMIN** — inicia el registro principal y decide agregar una segunda actividad vulnerable.
- **Sistema** — crea un draft independiente para la actividad secundaria y lo enlaza al registro principal al confirmar.

## Precondiciones

- El flujo de [registro de sujeto obligado](registro-sujeto-obligado.md) debe estar activo con un `registrationId` en estado `draft`.
- El usuario presionó "+ Agregar actividad vulnerable" en el paso 4 del wizard principal.
- Punto de entrada en el diagrama padre: nodos `agregar-segunda-actividad-vulnerable-pf` / `agregar-segunda-actividad-vulnerable-inmobiliaria-pm` (página "Agregar segunda actividad vulnerable").
- El modal recibe el `sourceActivityType` del registro principal (`NOTARY` o `REAL_ESTATE`) para derivar el tipo de actividad opuesto. **No recibe ni reutiliza el** `registrationId` **principal** — crea su propio draft al iniciar el paso de identificación.

## Pasos

<!-- jarvis:diagram src=flujo.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Actividad Secundaria
direction: LR
nodes[6]{id,label,shape}:
  inicio,inicio,terminator
  sourceactivitytype-notary,sourceActivityType = NOTARY,decision
  registra-persona-fisica?,¿registra persona fisica?,decision
  muestra-formulario-pm,muestra formulario PM,process
  muestra-formulario-pf,muestra formulario PF,process
  regresa-al-flujo-principal-de-registro-con-los-datos-registrado-e-indetificacion-del-workspace-recibido-del-be,Regresa al flujo principal de registro con los datos registrado e indetificacion del workspace recibido del BE,process
edges[7]{from,to,label}:
  inicio "inicio",sourceactivitytype-notary "sourceActivityType = NOTARY",
  sourceactivitytype-notary "sourceActivityType = NOTARY",muestra-formulario-pf "muestra formulario PF",
  sourceactivitytype-notary "sourceActivityType = NOTARY",registra-persona-fisica? "¿registra persona fisica?",
  registra-persona-fisica? "¿registra persona fisica?",muestra-formulario-pm "muestra formulario PM",
  registra-persona-fisica? "¿registra persona fisica?",muestra-formulario-pf "muestra formulario PF",
  muestra-formulario-pm "muestra formulario PM",regresa-al-flujo-principal-de-registro-con-los-datos-registrado-e-indetificacion-del-workspace-recibido-del-be "Regresa al flujo principal de registro con los datos registrado e indetificacion del workspace recibido del BE",
  muestra-formulario-pf "muestra formulario PF",regresa-al-flujo-principal-de-registro-con-los-datos-registrado-e-indetificacion-del-workspace-recibido-del-be "Regresa al flujo principal de registro con los datos registrado e indetificacion del workspace recibido del BE",
```

## Casos alternos

- **Inmobiliaria como fuente**: el rol opuesto es Notario — solo PF (forzado, no hay selección de tipo de perfil).
- **Notario como fuente**: el rol opuesto es Inmobiliaria — el usuario puede elegir PF o PM.

## Reglas de negocio

- El rol de la actividad secundaria es siempre el **opuesto** al registro principal.
- Si la fuente es **Notario**: el opuesto es Inmobiliaria — puede ser PF o PM (usuario elige).
- Si la fuente es **Inmobiliaria**: el opuesto es Notario — solo PF (forzado).
- Solo se permite **una** actividad secundaria por registro principal.
- El paso **Datos de identificación** de la actividad secundaria se **prellena por defecto** con los datos de identificación del registro principal (es la misma persona registrando una segunda actividad — solicitud del Product Owner). El operador puede editarlos antes de avanzar.
  - Si la secundaria es del **mismo tipo de persona** que el principal (PF→PF o PM→PM): se copian **todos** los campos de identificación, incluidos RFC y CURP.
  - Si el tipo **difiere** (PF principal → PM secundaria, o viceversa): solo se copia la **nacionalidad**. El RFC no se copia (el formato PF de 13 caracteres difiere del PM de 12) y no hay mapeo entre nombre/apellidos y razón social.
- El paso **Datos de contacto** de la actividad secundaria también se **prellena por defecto** con los contactos del registro principal (mismo sujeto obligado). Los contactos se **clonan como nuevos** (sin `persistedAt`/`contactId`) para persistirse en el draft secundario propio. El operador puede editarlos o eliminarlos.

## Notas

- El wizard UI del modal vive en [disenos/](disenos/).
- El diagrama BE (toon) es la fuente de verdad; el `.drawio` es una sombra generada por Jarvis.

### Notas técnicas

- **Crear draft (lazy)**: el `POST /registration/start-draft` se llama al avanzar del paso 01 (tipo persona) al paso 02 (identificación) — en `persistStep` del paso `REPORTING_ENTITY` o al entrar a `IDENTIFICATION`. Cancelar antes no deja rastro en BE.
- **Link al principal**: `linkSecondaryActivity(primaryRegistrationId, secondaryRegistrationId)` se llama en `handleConfirm`. El modal debe recibir `primaryRegistrationId` como prop (renombrado desde `registrationId`) — solo se usa en este punto final.
- **Cancelar post-draft**: `onClose` sin cleanup — el draft secundario queda incompleto en BE pero nunca se finaliza ni enlaza, por lo que no afecta el registro principal.
- **Prellenado de identificación**: la página pasa al modal `sourceIdentification` (la identificación del principal) y `sourceProfileType`. El modal deriva el `initialState.identification` con `prefillIdentification()` y lo recomputa si el operador cambia el tipo de persona en el Paso 01. El `key` del modal incluye `profileType` para forzar remount cuando cambia el tipo del principal.
- **Prellenado de contactos**: la página pasa `sourceContacts` (los contactos del principal). El modal deriva `initialState.contacts` con `prefillContacts()`, que clona cada contacto descartando `persistedAt`/`contactId` para que `persistStep` los envíe (`appendContact`) al draft secundario. No depende del tipo de persona.

## Referencias

| Componente | Archivo |
|---|---|
| Modal FE | `pld-web/src/components/organisms/reporting-entity/AddVulnerableActivityModal/index.tsx` |
| Página principal | `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx` |
| Servicio FE | `pld-web/src/services/registrationService.ts` |
| Servicio BE | `pld-api/apps/auth-users/src/admin/registration/registration.service.ts` |
