# Archive Report: add-reporting-entity-registration-ui

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & SMOKE-TESTED
**Sub-repo**: pld-web (FE-only)

## Summary

Wizard PF/PM completo en `/admin/reporting-entity/register` para que un SUPERADMIN registre sujetos obligados (Notarías o Inmobiliarias) en flujo único de 5 (PF) o 6 (PM) pasos. Persistencia paso-a-paso al BE, resume vía localStorage, modales de confirmación + éxito + error con reintentos.

## Decisiones aplicadas (Q1-Q21 + R1-R4)

Consolidación de 21 decisiones del thread jarvis `5a231957-c89e-46ef-9122-82f482247a67` y 4 refinamientos posteriores. Resumen:

- **Q1**: eventos del Wizard `onNext` / `onPrevious` / `onFinish`.
- **Q2**: persistencia paso-a-paso (POST por step).
- **Q3**: múltiples contactos en step 3 (mín 1, hack `cellphone = phone` en v1).
- **Q4**: resume vía `localStorage['pld:activeRegistrationId']`.
- **Q5**: `RoleProtectedRoute` + JWT decode con `jwt-decode` (sin `GET /me` que no existe).
- **Q6**: sin botón Cancelar, solo Regresar/Siguiente.
- **Q7**: `/admin/reporting-entity/register` + mapping `postLoginRedirect.ts` por rol.
- **Q9**: PF + PM en flujo único, `shouldSkip` para compliance step en PF.
- **Q11**: RFC en step 2 + 2 POSTs encadenados (`POST /` + `POST /:id/identification`).
- **Q12**: botón "Reenviar correo" maquetado, sin función (deuda BE).
- **Q13**: `ui/Dialog` movido a `atoms/Dialog`, nuevo `molecules/ConfirmDialog`.
- **Q14**: cerrar modal éxito → reset wizard a step 1 en misma URL.
- **Q15**: `country` y `city` agregados al form (BE los requiere).
- **Q16**: `<AdministrativeDivisionsField>` consume catálogos multi-país (cascada).
- **Q17**: stepper 5/6 vía `shouldSkip` (automático).
- **Q18**: `finalButtonLabel="Confirmar"` + modal cancela cierra, modal confirma dispara POST.
- **Q19**: spinner botón + Regresar disabled + banner error inline.
- **Q20**: `BE_STEP_TO_FE_KEY` mapping.
- **R1**: 3 reintentos máx en error modal + race condition COMPLETED → silent clean.
- **R4**: dropdown país visualmente activo pero no interactivo.

## FE files (new)

```
src/
├── components/
│   ├── atoms/Dialog/                   (movido desde ui/Dialog)
│   ├── molecules/
│   │   ├── ConfirmDialog/index.tsx
│   │   └── AdministrativeDivisionsField/index.tsx
│   ├── organisms/Wizard/index.tsx       (extendido con onNext/onPrevious/isTransitioning/transitionError)
│   └── reporting-entity/
│       ├── ReportingEntityTypeStep/     (step 1)
│       ├── PhysicalIdentificationStep/  (step 2 PF)
│       ├── MoralIdentificationStep/     (step 2 PM)
│       ├── ContactStep/                 (step 3 — array dinámico, persistedAt tracking)
│       ├── VulnerableActivityStep/      (step 4 — actividad + AdministrativeDivisionsField + country locked)
│       ├── ComplianceResponsibleStep/   (step PM-5)
│       ├── ReviewStep/                  (último — render condicional PF/PM)
│       ├── SuccessFinalizeModal/        (tempPassword + 3 botones)
│       ├── ErrorFinalizeModal/          (contador máx 3 reintentos)
│       └── schemas.ts                   (6 schemas zod)
├── pages/admin/ReportingEntityRegistrationPage.tsx
├── types/{UserRole, registration}.ts
├── hooks/useCurrentUserRole.ts
├── routes/RoleProtectedRoute.tsx
├── config/postLoginRedirect.ts
├── services/registrationService.ts
└── queries/registrationQueries.ts
```

## FE files (modified)

- `src/components/organisms/Wizard/index.tsx` — API extendida.
- `src/components/auth/LoginForm/index.tsx` — usa `getPostLoginRedirect`.
- `src/components/external-users/BeneficiaryControllerData/CreateBeneficiaryController.tsx` — import path actualizado tras rename Dialog.
- `src/routes/index.tsx` — registro de la ruta con `RoleProtectedRoute`.
- `src/routes/routes-urls.ts` — agregada `REPORTING_ENTITY_REGISTER`.
- `src/config/constants.ts` — agregada `LOCAL_STORAGE_KEYS.ACTIVE_REGISTRATION_ID`.
- `package.json` — agregadas `zod`, `jwt-decode`.

## Validación

- `yarn tsc -b --noEmit` → 0 errores.
- `yarn build` → 0 errores. Bundle de la página: 106 KB / 28 KB gzipped.
- `yarn eslint` en archivos nuevos → 0 errores.

### Smoke tests E2E (BE)

✅ Flujo PF completo: Login → POST / → POST identification → POST contact → POST vulnerable-activity → POST finalize → user creado con tempPassword → GET /:id status COMPLETED.

✅ Flujo PM completo: idem + POST compliance-responsible.

✅ Edge cases:
- 409 RFC duplicado IN_PROGRESS (devuelve `registrationId` del existente).
- DELETE libera RFC para reintentar POST con mismo RFC.
- 404 GET/DELETE con id inexistente.
- 401 sin JWT en endpoints admin.
- 200 catalogs públicos sin JWT.
- 409 finalize con draft sin steps completos.

## Desviaciones documentadas

1. **Resume v1 parcial**: `findRegistrationById` del BE no carga relations. v1 detecta el draft activo pero el usuario re-tipea desde step 2. Documentado como TODO en código. No bloquea el flujo principal.

2. **Q21 (cambio profileType con DELETE) no implementado**: el flujo "cambiar tipo en step 1 con draft activo → modal → DELETE → reset" requería complejidad adicional en `ReportingEntityTypeStep`. La página tiene `wizardKey` para reset clean post-finalize, suficiente para el flujo principal. Edge case documentado como TODO.

3. **RHF no se usa**: los step components manejan state directo con `value`+`onChange`. Validación via `safeParse` en `isValid`. Schemas zod siguen disponibles si en iteración futura se quiere migrar a RHF.

4. **Node 21 + Vite 7**: incompatibilidad oficial (Vite pide Node 20.19+ o 22.12+). Se instaló con `--ignore-engines`. tsc + build + dev funcionan correctamente. Documentado como deuda operacional.

## Specs delta

Crear `openspec/specs/reporting-entity-registration-ui/spec.md` con 15 invariantes Given/When/Then (INV-1 a INV-15) cubriendo:
- Acceso restringido SUPERADMIN
- Redirect post-login configurable
- Step 1 sin persistencia BE
- 2 POSTs encadenados en step 2
- 409 RFC duplicado abre modal de resume
- Múltiples contactos
- Step PM compliance con shouldSkip
- Confirmar abre modal antes de finalizar
- Modal de éxito muestra password una sola vez
- Reintentos máx 3
- Resume IN_PROGRESS / COMPLETED race condition
- Cambio profileType destruye draft (parcialmente implementado)
- Country dropdown visualmente activo no interactivo
- Cellphone hack v1

## Commits

- `pld-web` (main): `c24fa2c feat(reporting-entity): add registration wizard with PF/PM flow`
- `pld-web` (main): `d29c317 feat(wizard): add onNext/onPrevious async hooks, move Dialog to atoms, add ConfirmDialog`

## Pendientes (no bloqueantes)

- Validación visual en browser (tests 11.1-11.10 del tasks.md original).
- Migración yup → zod de los forms legacy (`migrate-yup-to-zod` change separado, en backlog).
- Q21 completo: cambio profileType en step 1 con draft activo dispara DELETE.
- Resume completo v2: pre-llenar state desde GET /:id con fetches adicionales para profile/contact/vulnerable-activity.
- Bugs heredados del BE (lastLoginAt, regex RFC, ValidationPipe).
