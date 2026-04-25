# Tasks: Reporting Entity Registration UI

## Phase 0 — Verificaciones (V1, V2, V3)

- [ ] 0.1 Verificar shape exacto de `GET /admin/registration/:id` leyendo el adapter del BE: `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts` método `findRegistrationById`. Documentar campos hidratados (relaciones anidadas vs IDs).
- [ ] 0.2 Verificar valores exactos del enum `RegistrationStep` (o equivalente) que el BE devuelve en `current_step`. Actualizar `BE_STEP_TO_FE_KEY` con los valores reales.
- [ ] 0.3 Confirmar que `add-reference-catalogs` está mergeado y los endpoints accesibles vía `curl`.
- [ ] 0.4 Confirmar que `add-discard-registration-endpoint` está mergeado y `DELETE /admin/registration/:id` accesible.
- [ ] 0.5 Instalar `jwt-decode` en pld-web: `yarn add jwt-decode`.
- [ ] 0.6 Instalar `zod` y `@hookform/resolvers` (zod) si no están: `yarn add zod` (resolvers ya está).

## Phase 1 — Wizard organism extendido

- [ ] 1.1 Editar `pld-web/src/components/organisms/Wizard/index.tsx`. Agregar a `WizardStep<S>`:
  ```ts
  onNext?: (ctx: WizardContext<S>) => Promise<boolean | void>;
  onPrevious?: (ctx: WizardContext<S>) => void | Promise<void>;
  ```
- [ ] 1.2 Extender `WizardContext<S>` con `isTransitioning: boolean` y `transitionError: Error | null`.
- [ ] 1.3 Modificar `goNext` y `goBack`:
  - Si el step actual tiene `onNext`/`onPrevious`, ejecutarlo antes.
  - Setear `isTransitioning=true` durante.
  - Si retorna `false` o lanza, `setTransitionError(err)` y NO avanzar.
  - Si OK, limpiar transitionError y avanzar.
- [ ] 1.4 Modificar render: agregar banner inline rojo cuando `transitionError !== null`.
- [ ] 1.5 Modificar render: botón Siguiente con `loading={isTransitioning}`, botón Regresar con `disabled={isFirst || isTransitioning}`.
- [ ] 1.6 Verificar que `CreateUserPage.tsx` (legacy) sigue funcionando — `onNext`/`onPrevious` son opcionales.

## Phase 2 — Atomic Design housekeeping

- [ ] 2.1 `git mv pld-web/src/components/ui/Dialog pld-web/src/components/atoms/Dialog`.
- [ ] 2.2 Actualizar el único consumer: `pld-web/src/components/external-users/BeneficiaryControllerData/CreateBeneficiaryController.tsx` (`@/components/atoms/Dialog`).
- [ ] 2.3 `tsc -b --noEmit` pasa.

## Phase 3 — Atomic Design: ConfirmDialog molecule

- [ ] 3.1 Crear `pld-web/src/components/molecules/ConfirmDialog/index.tsx` que componga `atoms/Dialog` + `ui/Button`. Props: `{ open, title, description, cancelLabel?, confirmLabel?, onCancel, onConfirm, isConfirming? }`.
- [ ] 3.2 Defaults: `cancelLabel = "Cancelar"`, `confirmLabel = "Confirmar"`.
- [ ] 3.3 Loading state: si `isConfirming`, botón Confirm muestra spinner y Cancel está disabled.

## Phase 4 — Auth y rutas

- [ ] 4.1 Crear `pld-web/src/types/UserRole.ts` con replica del enum BE: `SUPERADMIN | NOTARIO | INMOBILIARIA | AUXILIAR`.
- [ ] 4.2 Crear `pld-web/src/hooks/useCurrentUserRole.ts` que decodifica JWT con `jwt-decode`.
- [ ] 4.3 Crear `pld-web/src/routes/RoleProtectedRoute.tsx` que extiende `ProtectedRoute` con prop `requiredRoles: UserRole[]`.
- [ ] 4.4 Crear `pld-web/src/config/postLoginRedirect.ts` con `POST_LOGIN_REDIRECT_BY_ROLE`, `DEFAULT_REDIRECT`, función `getPostLoginRedirect(role)`.
- [ ] 4.5 Modificar `pld-web/src/components/auth/LoginForm/index.tsx`: tras login exitoso, `useCurrentUserRole` + `getPostLoginRedirect` para determinar destino.
- [ ] 4.6 Agregar `RoutesUrl.REPORTING_ENTITY_REGISTER = "/admin/reporting-entity/register"` a `routes-urls.ts`.
- [ ] 4.7 Registrar la nueva ruta en `routes/index.tsx` con `<RoleProtectedRoute requiredRoles={['SUPERADMIN']}>`.

## Phase 5 — Tipos y servicio

- [ ] 5.1 Crear `pld-web/src/types/registration.ts` con:
  - `TipoPersonaParticipante` enum local.
  - DTOs: `CreateRegistrationDto`, `PhysicalIdentificationDto`, `MoralIdentificationDto`, `ContactDto`, `VulnerableActivityDto`, `AddressDto`, `ComplianceResponsibleDto`, `RegistrationDetail`.
  - `RegistrationState` (state del Wizard).
  - `BE_STEP_TO_FE_KEY` mapping (verificar valores reales del BE en Phase 0).
  - Constante `LOCAL_STORAGE_KEYS.ACTIVE_REGISTRATION_ID`.
- [ ] 5.2 Crear `pld-web/src/services/registrationService.ts` con 8 funciones (startDraft, getRegistration, discardRegistration, saveIdentification, appendContact, saveVulnerableActivity, saveComplianceResponsible, finalize).
- [ ] 5.3 Crear `pld-web/src/queries/registrationQueries.ts` con `useStartDraft`, `useDiscardRegistration`, `useSaveIdentification`, etc. (todas mutations) + `useGetRegistration` (query).

## Phase 6 — Schemas zod

- [ ] 6.1 Crear `pld-web/src/components/reporting-entity/schemas.ts` con todos los schemas zod:
  - `physicalIdentificationSchema`
  - `moralIdentificationSchema`
  - `contactSchema`
  - `vulnerableActivitySchema` (incluye `addressSchema` anidado)
  - `complianceResponsibleSchema`
  - `reportingEntityTypeSchema` (step 1)

## Phase 7 — Componente AdministrativeDivisionsField

- [ ] 7.1 Crear `pld-web/src/components/molecules/AdministrativeDivisionsField/index.tsx`.
- [ ] 7.2 Props: `{ countryCode, value: Record<string, string>, onChange, errors? }`.
- [ ] 7.3 Lógica:
  - `useGetAdministrativeStructure(countryCode)` → obtener levels.
  - Por cada level, renderizar `<Dropdown>` con `useGetAdministrativeEntries(countryCode, level, parentCode)`.
  - Cuando cambia un nivel, resetear los siguientes en el state padre.
  - Si `useGetAdministrativeEntries` devuelve array vacío → fallback a `<InputText>` libre.

## Phase 8 — Step components (reporting-entity)

- [ ] 8.1 `ReportingEntityTypeStep/` — accordion Notario/Inmobiliaria con sub-selector PF/PM. Recibe prop `hasActiveDraft` para detectar cambios y disparar confirm dialog (Q21).
- [ ] 8.2 `PhysicalIdentificationStep/` — form PF (firstName, paternalSurname, maternalSurname, birthDate, rfc, curp, nationalityCountry?, birthCountry?). Schema: `physicalIdentificationSchema`.
- [ ] 8.3 `MoralIdentificationStep/` — form PM (corporateName, incorporationDate?, rfc, nationalityCountry?). Schema: `moralIdentificationSchema`.
- [ ] 8.4 `ContactStep/` — array dinámico de contactos con add/remove. Mín 1. Botón "Agregar contacto". Hack v1: cellphone = phone al persistir.
- [ ] 8.5 `VulnerableActivityStep/` — dropdown actividad (consume `useGetVulnerableActivities`) + datos del domicilio (`<AdministrativeDivisionsField>` + inputs libres). Country como dropdown sin click (R4).
- [ ] 8.6 `ComplianceResponsibleStep/` — solo PM. Form con responsable (firstName, paternalSurname, maternalSurname, birthDate?, rfc, curp, nationalityCountry?, designationDate?). `shouldSkip: s => s.profileType !== 'PERSONA_MORAL'`.
- [ ] 8.7 `ReviewStep/` — render condicional según profileType. Resumen de todos los datos. Botones Editar por bloque que llaman `goTo(key)`.

## Phase 9 — Modales finales

- [ ] 9.1 `SuccessFinalizeModal/` — muestra tempPassword + email + 3 botones (Copiar contraseña, Reenviar correo maquetado, Finalizar). Click Finalizar limpia localStorage + reset wizard.
- [ ] 9.2 `ErrorFinalizeModal/` — botones Cancelar / Intentar de nuevo. Contador interno de reintentos máx 3.

## Phase 10 — Página y orquestación

- [ ] 10.1 Crear `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx` que:
  - Maneja state via `<Wizard initialState={...}>`.
  - useEffect inicial: lee localStorage, dispara `getRegistration` si hay activeId, abre modal de resume si IN_PROGRESS, silent clean si COMPLETED (R1).
  - Define el array `steps[]` con los 7 steps y sus `onNext`, `isValid`, `shouldSkip`.
  - Renderiza `<Wizard>` + breadcrumb dinámico (`<Breadcrumb>` o inline) + los modales (resume, confirm, success, error).
- [ ] 10.2 El `onFinish` del Wizard solo abre el ConfirmDialog. La operación real `POST /finalize` la dispara `onConfirm` del modal.

## Phase 11 — Tests manuales

- [ ] 11.1 Login como SUPERADMIN → debe redirigir a `/admin/reporting-entity/register`.
- [ ] 11.2 Crear sujeto obligado completo PF: Notario+PF, llenar 5 steps, Confirmar, ver tempPassword, Finalizar, volver al step 1.
- [ ] 11.3 Crear sujeto PM: Inmobiliaria+PM, llenar 6 steps incluyendo Responsable, finalizar.
- [ ] 11.4 Test resume: empezar registro, refrescar, modal pregunta, "Continuar", llegar al step donde quedó.
- [ ] 11.5 Test resume: empezar registro, refrescar, "Empezar nuevo" → DELETE BE + reset.
- [ ] 11.6 Test cambio de profileType (Q21): empezar PM, llegar a step 3, Regresar a step 1, cambiar a PF, modal pregunta, confirm → DELETE BE + reset.
- [ ] 11.7 Test reintentos: simular error en finalize (apagar BE momentáneamente), verificar contador hasta 3.
- [ ] 11.8 Test 409 RFC duplicado: crear PF con RFC X, antes de finalizar abrir nueva pestaña, intentar crear otro con mismo RFC → modal de resume.
- [ ] 11.9 Test rol no SUPERADMIN: login con admin@pld.com (que es SUPERADMIN — necesitamos otro user para test). Si no hay, verificar lógica decodificando un JWT manual.
- [ ] 11.10 Verificar visual: comparar con screenshots en `docs/designs/reporting-entity-registration/`.

## Phase 12 — Validación final

- [ ] 12.1 `cd pld-web && yarn tsc -b --noEmit` pasa.
- [ ] 12.2 `yarn build` pasa.
- [ ] 12.3 Smoke browser completo end-to-end.

## Phase 13 — Specs delta

- [ ] 13.1 Aplicar [specs.md](specs.md) a `openspec/specs/reporting-entity-registration-ui/spec.md` durante `/sdd-archive`.

## Phase 14 — Commits

- [ ] 14.1 Recomendado: dividir en 2-3 commits granulares en `pld-web`:
  - `feat(wizard): add onNext/onPrevious async hooks + isTransitioning state`
  - `refactor(components): move ui/Dialog to atoms, add ConfirmDialog molecule`
  - `feat(reporting-entity): add registration wizard with PF/PM flow`
- [ ] 14.2 (Opcional) `/sdd-archive add-reporting-entity-registration-ui`.
