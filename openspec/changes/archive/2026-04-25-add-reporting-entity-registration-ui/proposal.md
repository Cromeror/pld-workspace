# Proposal: Reporting Entity Registration UI (PF + PM wizard)

**Status**: draft
**Created**: 2026-04-25
**Sub-repo**: pld-web (FE-only)

## Intent

Implementar la página `/admin/reporting-entity/register` que permite a un SUPERADMIN registrar sujetos obligados (Notarios o Inmobiliarias) tanto Persona Física (PF) como Persona Moral (PM) mediante un wizard multi-step. El flujo persiste paso-a-paso en BE, soporta reanudar drafts incompletos vía localStorage, y termina con un modal que muestra una vez la `tempPassword` del usuario creado.

## Scope

**In** (FE — `pld-web`):

### Página y ruta
- Crear `src/pages/admin/ReportingEntityRegistrationPage.tsx`.
- Ruta `/admin/reporting-entity/register` protegida por `RoleProtectedRoute` (solo SUPERADMIN).
- Constante `RoutesUrl.REPORTING_ENTITY_REGISTER` agregada a `routes-urls.ts`.

### Routing y autorización
- Crear `src/hooks/useCurrentUserRole.ts` que decodifica el JWT (sin verificar firma — el BE valida) y devuelve `role: UserRole | null`. Usar `jwt-decode` (lib pequeña, ~2KB).
- Crear `src/routes/RoleProtectedRoute.tsx`: extiende `ProtectedRoute` con prop `requiredRoles: UserRole[]`. Redirect a `RoutesUrl.HOME` si el rol no aplica.
- Crear `src/config/postLoginRedirect.ts` con mapping `Record<UserRole, string>`. Default fallback a `RoutesUrl.HOME`.
- Modificar `LoginForm`: tras login exitoso, leer rol del JWT y redirect según mapping (en lugar de `RoutesUrl.CREATE_USER` hardcoded).

### Wizard del registro
- Crear `src/components/reporting-entity/`:
  - `ReportingEntityTypeStep/` — step 1 (selector Notario/Inmobiliaria + PF/PM).
  - `PhysicalIdentificationStep/` — step 2 PF.
  - `MoralIdentificationStep/` — step 2 PM.
  - `ContactStep/` — step 3 (lista dinámica de contactos, mín. 1).
  - `VulnerableActivityStep/` — step 4 (dropdown actividad + dropdowns geográficos en cascada).
  - `ComplianceResponsibleStep/` — step PM-5 (solo PM, `shouldSkip` para PF).
  - `ReviewStep/` — step final (resumen + botón Confirmar).
  - `schemas.ts` — schemas zod de los steps.
- Reusar el organism `<Wizard />` ya existente (`src/components/organisms/Wizard/`).

### Extensión del Wizard organism
- Agregar a `WizardStep<S>` los hooks `onNext` y `onPrevious` (retornan `Promise<boolean | void>`).
- En el motor del Wizard:
  - `onNext` async: dispara antes de avanzar; si lanza o retorna `false`, no avanza.
  - `onPrevious` async: dispara antes de retroceder; mismo comportamiento.
  - Estado `isTransitioning` y `transitionError` expuestos vía ctx.
  - Loading en botón Siguiente y disabled en Regresar durante transición.
  - Banner de error inline en card cuando `transitionError !== null`.

### Modales
- Migrar `src/components/ui/Dialog/` → `src/components/atoms/Dialog/` (Q13). Actualizar el único consumer en `external-users/`.
- Crear `src/components/molecules/ConfirmDialog/` con API:
  ```ts
  { open, title, description, cancelLabel, confirmLabel, onCancel, onConfirm, isConfirming }
  ```
- Crear `src/components/reporting-entity/SuccessFinalizeModal/` con tempPassword + email + 3 botones: Copiar, Reenviar (maquetado sin función), Finalizar.
- Crear `src/components/reporting-entity/ErrorFinalizeModal/` con contador de reintentos (máx 3).

### Componente multi-país de divisiones administrativas
- Crear `src/components/molecules/AdministrativeDivisionsField/` que recibe `countryCode + value + onChange` y renderiza N dropdowns en cascada según la `structure` del catálogo. Internamente consume `useGetAdministrativeStructure` y `useGetAdministrativeEntries`.

### Service y queries del registro
- Crear `src/services/registrationService.ts` con funciones:
  - `startDraft({ profileType, userRole, rfc })` → POST `/`.
  - `getRegistration(id)` → GET `/:id` (resume).
  - `discardRegistration(id)` → DELETE `/:id` (Q21).
  - `saveIdentification(id, dto)` → POST `/:id/identification`.
  - `appendContact(id, dto)` → POST `/:id/contact`.
  - `saveVulnerableActivity(id, dto)` → POST `/:id/vulnerable-activity`.
  - `saveComplianceResponsible(id, dto)` → POST `/:id/compliance-responsible`.
  - `finalize(id)` → POST `/:id/finalize`.
- Crear `src/queries/registrationQueries.ts` con mutations correspondientes (`useMutation`).
- Constantes en `src/config/constants.ts`: `LOCAL_STORAGE_KEYS.ACTIVE_REGISTRATION_ID`.

### Tipos
- Crear `src/types/registration.ts` con:
  - Enums locales `UserRole`, `TipoPersonaParticipante` (replicas de los del BE).
  - DTOs: `CreateRegistrationDto`, `PhysicalIdentificationDto`, `MoralIdentificationDto`, `ContactDto`, `VulnerableActivityDto`, `AddressDto`, `ComplianceResponsibleDto`.
  - `RegistrationState` (state del Wizard).
  - Mapping `BE_STEP_TO_FE_KEY`.

**Out**:
- Catálogos BE: ya creados en `add-reference-catalogs` (dependencia bloqueante).
- Endpoint DELETE BE: ya creado en `add-discard-registration-endpoint` (dependencia bloqueante).
- Endpoint `/me` o cualquier cambio BE: NO se requiere (rol leído del JWT).
- Migración yup → zod: trackeada en `migrate-yup-to-zod` (independiente). El wizard nace con zod directamente.
- Listado de sujetos obligados (`/admin/reporting-entity` listado): out of scope v1.
- Edición de sujetos creados: out of scope v1.
- Múltiples idiomas: solo español en v1.
- Migración del wizard external-users (CreateUserPage legacy): no se toca, queda como feature legacy aparte.

## Motivation

1. Es la feature core del proyecto: registrar sujetos obligados es la razón de ser del módulo `admin/registration` del BE.
2. El BE ya está listo (verificado en archive-report del change `add-registro-sujetos-obligados`).
3. Los catálogos y el endpoint DELETE son dependencias resueltas.
4. El diseño Figma está completo (screenshots droppeados en `docs/designs/reporting-entity-registration/`).

## Approach

Se sigue el flujo paso-a-paso definido en las 21 decisiones (Q1-Q21) + 4 refinamientos (R1-R4) registradas en el thread de jarvis `5a231957-c89e-46ef-9122-82f482247a67`.

Resumen del flujo:

1. SUPERADMIN se loguea → redirect a `/admin/reporting-entity/register` (vía mapping configurable post-login).
2. Step 1: selecciona userRole + profileType localmente (sin POST).
3. Step 2 (`onNext`):
   - Si no hay `registrationId`: dispara `POST /` (crea draft) → setState({ registrationId }).
   - Dispara `POST /:id/identification` con el body PF o PM.
   - Si 409 al crear (RFC duplicado): muestra modal de resume con id existente.
4. Step 3 (`onNext`): dispara N `POST /:id/contact` por cada contacto agregado.
5. Step 4 (`onNext`): dispara `POST /:id/vulnerable-activity` con address + activity.
6. Step PM-5 (solo PM, `shouldSkip` para PF, `onNext`): dispara `POST /:id/compliance-responsible`.
7. Step Review: muestra resumen, botón "Confirmar".
8. Click Confirmar → abre `<ConfirmDialog>` "¿Estás seguro/a?".
9. Click Confirmar del modal: dispara `POST /:id/finalize`. Si éxito → modal de éxito con tempPassword. Si error → modal de error con contador de reintentos (máx 3).
10. Click Finalizar del modal de éxito: limpia localStorage, resetea wizard a step 1.

## Rollback plan

Revertir todos los commits del change. La página y los componentes desaparecen. La ruta queda 404. El BE sigue funcionando independiente.

## Affected surfaces

- FE únicamente.
- Carpeta nueva `src/components/reporting-entity/` (~10 componentes).
- Carpeta nueva `src/pages/admin/`.
- Move: `ui/Dialog` → `atoms/Dialog`.
- Carpeta nueva `molecules/ConfirmDialog/`.
- Carpeta nueva `molecules/AdministrativeDivisionsField/`.
- Modificación del organism `Wizard/` para agregar `onNext`/`onPrevious` async.
- Modificación de `LoginForm` para usar mapping post-login.
- Nuevo: `RoleProtectedRoute`, `useCurrentUserRole`, `postLoginRedirect`, `registrationService`, `registrationQueries`, `types/registration`.
- Modificación menor: `routes/index.tsx` (registro de la ruta), `routes-urls.ts` (constante).

## Dependencies

Bloqueada por:
1. **`add-reference-catalogs`** — necesita los endpoints `/catalogs/vulnerable-activities`, `/catalogs/countries`, `/catalogs/administrative-divisions/*`.
2. **`add-discard-registration-endpoint`** — necesita el endpoint DELETE para Q21.

NO bloqueada por:
- `migrate-yup-to-zod` (independiente; el wizard nace con zod directamente, instala `zod` localmente si es necesario).
- `rename-to-english` (NO se aplica antes; el wizard usa enums actuales en español).

## Open questions

- ¿Existe una manera de mostrar un toast de éxito tras cerrar el modal de éxito al volver al step 1? Para evitar perder feedback. Decisión durante implementación.
- ¿El listado posterior de sujetos obligados merece otro change? Sí, `add-reporting-entity-list` cuando llegue.
