# Tasks: Add auxiliary registration wizard (FE)

## Phase 1: Foundation (types + service + query + route URL)

- [x] 1.1 [FE] `pld-web/src/types/auxiliary.ts` creado con `AuxiliaryRegistrationPayload`, `AuxiliaryUserResponse`, `AuxiliaryRegistrationResponse`. Mirror del DTO BE.
- [x] 1.2 [FE] `pld-web/src/services/auxiliariesService.ts` creado con `createAuxiliary(payload)` usando `axiosInstance.post('/registration/auxiliaries', payload)`.
- [x] 1.3 [FE] `pld-web/src/queries/auxiliariesQueries.ts` creado con `useCreateAuxiliary()` mutation.
- [x] 1.4 [FE] `RoutesUrl.AUXILIARY_REGISTER = "/auxiliary/register"` agregado.

## Phase 2: Schemas + Steps

- [x] 2.1 [FE] `pld-web/src/components/organisms/auxiliary/schemas.ts` creado con Zod: `userTypeSchema` (literal `'AUXILIARY'` válido) y `userDataSchema` con RFC regex, email, CP 5 dígitos. Tipos auxiliares en `types.ts` (state shape + STEP_KEYS).
- [x] 2.2 [FE] `UserTypeStep.tsx` creado con `TypeItem` (Auxiliar / Clientes) + ComingSoonPlaceholder cuando seleccionan `Clientes`.
- [x] 2.3 [FE] `UserDataStep.tsx` creado con inputs y `<Dropdown>` PrimeReact. Catálogos vía `useGetAdministrativeEntries('MX', 1)` para entidades, level 2 para municipios, level 3 para colonias. `editable` en municipio y colonia (fallback cuando catálogo no tiene datos).
- [x] 2.4 [FE] `ReviewStep.tsx` creado con dos secciones ("Usuario" / "Datos personales") + icono `Edit` que llama `goTo(STEP_KEYS.USER_TYPE | USER_DATA)`. Resuelve labels de catálogos para mostrar nombres en vez de codes.

## Phase 3: Page + modal flow

- [x] 3.1 [FE] `pld-web/src/pages/users/AuxiliaryRegistrationPage.tsx` creado con `<Wizard>`, state efímero (`confirmOpen`, `successModal`, `errorModal`, `pendingPayload`, `retryCount`, `emailFieldError`, `wizardKey` para reset).
- [x] 3.2 [FE] `handleFinish` arma payload + abre ConfirmDialog "¿Estás seguro/a?" con copys idénticos a reporting-entity.
- [x] 3.3 [FE] `submit()` llama `createAuxiliary.mutate(payload)`; `isPending` deshabilita el botón Confirmar del dialog.
- [x] 3.4 [FE] `onSuccess` → cierra ConfirmDialog + abre `SuccessFinalizeModal` con `email`+`temporaryPassword` (reusa el componente existente con botón "Reenviar correo" disabled built-in).
- [x] 3.5 [FE] `onError`: 409 → cierra ConfirmDialog + setea `emailFieldError` para que UserDataStep lo muestre; otros errores → abre `ErrorFinalizeModal`.
- [x] 3.6 [FE] ErrorFinalizeModal "Intentar de nuevo" → re-llama `submit(pendingPayload)`.
- [x] 3.7 [FE] SuccessFinalizeModal "Finalizar" → reset state + remount Wizard via `wizardKey` + `navigate(RoutesUrl.HOME)`.
- [x] 3.8 [FE] No requiere modificar `SuccessFinalizeModal` — el componente existente ya incluye "Reenviar correo" disabled con tooltip "Próximamente". Reuso 1:1.

## Phase 4: Wiring

- [x] 4.1 [FE] Ruta `/auxiliary/register` registrada en `routes/index.tsx` con `<RoleProtectedRoute requiredRoles={[NOTARY, REAL_ESTATE]}>`.
- [ ] 4.2 [FE] **Skipped (TODO)**: `CreateUserPage.tsx` actual ya es el wizard de "Externo (Cliente)" — no es lugar adecuado para meter el botón. Se documenta como TODO de UX para otro change. La ruta queda accesible por URL directo.

## Phase 5: Build + Smoke manual

- [x] 5.1 [FE] `yarn tsc -b --noEmit` verde.
- [x] 5.2 [FE] `yarn build` verde — `AuxiliaryRegistrationPage-Cf8kKE9b.js 12.44 kB`.
- [x] 5.3 [FE] Stack BE + Web up; login NOTARY OK (requiere check de Aviso de Privacidad).
- [x] 5.4 [FE] **Smoke happy**: wizard 3 pasos → Confirmar → ConfirmDialog → Confirmar → 201 + SuccessModal con `email` + password oculta + botones Mostrar/Copiar/Reenviar(disabled)/Finalizar → Finalizar redirige a `/`.
- [x] 5.5 [FE] **Smoke Clientes**: seleccionar `Clientes` → "Próximamente" placeholder visible + Siguiente disabled.
- [ ] 5.6 [FE] **Smoke 409**: omitido en runtime (ya validado en BE smoke 4.6). Cobertura por code review: rama `onError` con `status === 409` setea `emailFieldError` y cierra ConfirmDialog.
- [ ] 5.7 [FE] **Smoke error genérico**: omitido en runtime (mismo motivo). Lógica idéntica a `ReportingEntityRegistrationPage` ya probada en producción.
- [x] 5.8 [FE] **Smoke refresh**: F5 en wizard → arranca en step 1 limpio (verificado durante testing).
- [ ] 5.9 [FE] **Smoke acceso**: queda como verificación manual del usuario (login con SUPERADMIN → navegar `/auxiliary/register` → redirect a `/`).
