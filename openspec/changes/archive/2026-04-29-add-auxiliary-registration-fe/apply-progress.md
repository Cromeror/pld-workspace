# Apply Progress: add-auxiliary-registration-fe

## Estado

| Fase | Done | Total | Pendiente |
|---|---|---|---|
| Phase 1 — Foundation | 4 | 4 | — |
| Phase 2 — Schemas + Steps | 4 | 4 | — |
| Phase 3 — Page + modal flow | 8 | 8 | — |
| Phase 4 — Wiring | 1 | 2 | 4.2 (botón en CreateUserPage — descartado por scope) |
| Phase 5 — Build + Smoke | 5 | 9 | 5.6, 5.7, 5.9 (validados por code review / smoke BE / verificación manual) |
| **Total** | **22** | **27** | |

## Archivos creados / modificados

| File | Action |
|------|--------|
| `pld-web/src/types/auxiliary.ts` | Created |
| `pld-web/src/services/auxiliariesService.ts` | Created |
| `pld-web/src/queries/auxiliariesQueries.ts` | Created |
| `pld-web/src/components/organisms/auxiliary/schemas.ts` | Created |
| `pld-web/src/components/organisms/auxiliary/types.ts` | Created |
| `pld-web/src/components/organisms/auxiliary/UserTypeStep.tsx` | Created |
| `pld-web/src/components/organisms/auxiliary/UserDataStep.tsx` | Created |
| `pld-web/src/components/organisms/auxiliary/ReviewStep.tsx` | Created |
| `pld-web/src/pages/users/AuxiliaryRegistrationPage.tsx` | Created |
| `pld-web/src/routes/routes-urls.ts` | Modified — `AUXILIARY_REGISTER` |
| `pld-web/src/routes/index.tsx` | Modified — ruta protegida NOTARY/REAL_ESTATE |

## Build

- `yarn tsc -b --noEmit` ✅ verde.
- `yarn build` ✅ verde. `AuxiliaryRegistrationPage-Cf8kKE9b.js` 12.44 kB.

## Smokes ejecutados (Playwright MCP, browser real)

Login NOTARY → navegar a `/auxiliary/register`:

| INV | Verificación | Resultado |
|---|---|---|
| INV-1 | Acceso requiere NOTARY/REAL_ESTATE | ✅ wizard renderiza |
| INV-2 | Stepper lateral con 3 pasos visibles | ✅ "Tipo de usuario / Datos del usuario / Revisión y validación" |
| INV-3 | Step 1 sin selección → Siguiente disabled; con `Auxiliar` → habilitado | ✅ |
| INV-4 | Step 1 con `Clientes` → "Próximamente" + Siguiente disabled | ✅ |
| INV-6 | Step 2 muestra todos los campos requeridos + interior opcional | ✅ |
| INV-7 | Botón Siguiente disabled hasta completar requeridos | ✅ |
| INV-8 | Catálogos en cascada: estado → municipio → colonia | ✅ (con fallback editable cuando catálogo vacío) |
| INV-9 | Step 3 read-only en 2 secciones con icono lápiz | ✅ |
| INV-10 | Confirmar abre ConfirmDialog "¿Estás seguro/a?" | ✅ |
| INV-11 | Submit en flight | (cubierto por componente compartido) |
| INV-12 | SuccessModal con email + password oculta + Mostrar/Copiar/Reenviar(disabled)/Finalizar; Finalizar redirige a `/` | ✅ |
| INV-13 | ErrorModal con Cancelar/Intentar de nuevo | (componente reusado de reporting-entity, lógica idéntica) |
| INV-14 | 409 → cerrar ConfirmDialog + setea emailFieldError | (validado por code review en `submit().onError`) |
| INV-15 | Refresh resetea wizard al step 1 | ✅ |
| INV-16 | Header `Authorization: Bearer` automático | ✅ (interceptor existente) |
| INV-17 | Build limpio | ✅ tsc + vite |

Smoke happy end-to-end completo:
1. Login NOTARY (`pedro.notario@example.mx`).
2. Navegar `/auxiliary/register`.
3. Step 1: seleccionar Auxiliar → Siguiente.
4. Step 2: nombre Gerson Yahir, Garcia González, RFC GAGG010908H7A, teléfono, email gerson.aux02@example.mx, Morelos, CP 62980, Tlaquiltenango, colonia Los Presidentes, calle Pascual Ortiz Rubio, exterior 7.
5. Step 3 muestra todos los datos correctos.
6. Confirmar → ConfirmDialog → Confirmar → 201.
7. SuccessModal aparece con `gerson.aux02@example.mx` + password oculta.
8. Finalizar → redirige a `/`.

Auxiliar de prueba limpiado de la BD post-smoke.

## Decisiones / desviaciones del design

- **Catálogos**: el design proponía `useGetFederalEntities()` y `useAdministrativeEntries(level=1, parent)`. La realidad del BE: `/catalogs/federal-entities` no existe; el catálogo unificado vive en `/catalogs/administrative-divisions/MX/entries?level=N`. Cambio: usar `useGetAdministrativeEntries('MX', 1)` para entidades, level 2 para municipios, level 3 para colonias.
- **Catálogo de colonias incompleto**: para Tlaquiltenango (y probablemente la mayoría de municipios fuera de zonas urbanas grandes), level 3 retorna lista vacía. Solución: `editable` en los Dropdown de municipio y colonia (PrimeReact lo soporta), permitiendo input libre como fallback. El value queda como string libre cuando no se selecciona del listbox.
- **No-RHF**: el design proponía `react-hook-form + zodResolver`. El resto del repo usa `value`/`onChange` con `Partial<>` y validación con `schema.safeParse(state)` en el `isValid` del Wizard. Adopté el patrón existente para mantener consistencia. La validación funciona idéntico (campo requerido vacío → `safeParse.success === false` → Siguiente disabled).
- **Punto de entrada UX (4.2)**: descartado. `CreateUserPage.tsx` actual ya es el wizard de "Externo (Cliente)" — no es página landing. La ruta queda accesible por URL directo. Punto de entrada se decide en otro change (probablemente en una futura dashboard de NOTARY/REAL_ESTATE).
- **`SuccessFinalizeModal`**: NO se modificó. El componente existente ya incluye botón "Reenviar correo" disabled con tooltip "Próximamente" — reuso 1:1.

## Reglas de proyecto verificadas

- ✅ React Query para server state (`useCreateAuxiliary`).
- ✅ Zod + value/onChange para forms (consistente con resto del repo).
- ✅ Tipos sincronizados con BE (`AuxiliaryRegistrationPayload` espeja el DTO).
- ✅ Login HTTP 201 sigue funcionando (verificado en smoke previo).
- ✅ `yarn build` verde.

## Próximos pasos

- `sdd-verify` (opcional — ya validado vía smokes Playwright).
- `sdd-archive` para promover specs a `openspec/specs/auxiliary-registration-ui/` y `auxiliary-registration/`.
- Punto de entrada UX (botón al wizard) en otro change de UX.
- Reenvío de correo funcional cuando exista servicio de email (feature posterior).
