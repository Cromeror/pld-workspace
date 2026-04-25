# Archive Report: migrate-yup-to-zod

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & BUILD-VERIFIED
**Sub-repo**: pld-web (FE-only)

## Summary

Migración completa de yup → zod en pld-web. 7 archivos con schemas migrados, 4 schemas exportados para consumo del Wizard, eliminación de los flags `*Validated` redundantes, y `yup` removido del repo.

## Changes

### Auth forms (3)
- `LoginForm` — `acceptedTerms` con `.refine(v === true)`.
- `ForgotPasswordForm` — schema simple email.
- `RecoverPasswordForm` — cross-field `confirmPassword === password` con `.refine` sobre el objeto + `path: ['confirmPassword']`.

### Wizard content-components (4)
Cada uno exporta su schema como named export para consumo via `safeParse`:
- `IdentityData` → `identitySchema`.
- `PrivateAddress` → `privateAddressSchema`.
- `IdentificationData` → `identificationSchema`.
- `PoliticallyExposedPerson` → `pepSchema` con `superRefine` para validación condicional (subscriberPosition / relativePosition / relativePEPFullName según las booleans).

Todos eliminan la prop `onValidated` y los useEffect/useRef asociados.

### Consumer cleanup
- `CreateUserPage.tsx`:
  - Importa los 4 schemas.
  - 4 `isValid:` cambiados a `(s) => xxxSchema.safeParse(s.xxxData).success`.
  - Eliminadas las 4 funciones `is*Complete`.
  - State sin los 4 flags `*Validated`.
  - initialState sin los flags.
  - Renders sin `onValidated={...}`.
- `CreateBeneficiaryController.tsx`:
  - Eliminados 4 `onValidated={...}` props en los renders de IdentityData/PrivateAddress/IdentificationData/PoliticallyExposedPerson.
  - Eliminados 4 handlers `handle*Validated` huérfanos (satisfacer `noUnusedLocals`).

### Cleanup
- `yarn remove yup`.
- 0 imports de yup en `src/`.

## Validación

- `yarn tsc -b --noEmit` → Done in 2.97s, 0 errores.
- `yarn build` → built in 2.82s, 0 errores.
- Bundle de `ReportingEntityRegistrationPage` redujo: 106 KB → 47 KB (yup-only-imports removidos del grafo).

## Decisiones aplicadas

- **D1**: schemas co-locados con cada componente (no `src/validation/`).
- **D2**: change atómico (no dividido en sub-changes).
- **D3**: error messages con `{ message: '...' }`. Para required usé `.min(1, 'msg')` (porque `required_error` solo fire en `undefined`, no en `""`).
- **D5**: PEP usa `superRefine` con `ctx.addIssue({ path: [...], message: ... })`.
- **D7**: `acceptedTerms` con `.refine(v => v === true, { message: ... })`.
- **D8 ajustada**: el design proponía v3, pero la versión instalada es zod **4.3.6** (sesión previa). Verificado que `@hookform/resolvers/zod` la soporta. Build limpio.
- **D9**: sin spec nueva — change frontend-internal, no observable contract shift.

## Desviaciones

1. **`identitySchema` FormFields type**: usé `z.input` en lugar de `z.infer` porque `birthDate` con `.refine(v !== null)` produce input `Date | null` y output `Date`. RHF necesita el input.

2. **CreateBeneficiaryController state**: el componente legacy aún tiene los flags `*Validated` en su state interno. Solo limpié los handlers huérfanos. Re-diseñar el componente legacy estaba fuera de scope.

## Commits

- `pld-web` (main): `d08403c refactor(web): migrate yup schemas to zod, remove redundant *Validated flags`

## Pendientes (no bloqueantes)

- Validación manual en browser (Phase 5 de tasks.md): los 9 sub-tests de UX que requieren clicks reales para verificar mensajes de error idénticos al pre-migración.
