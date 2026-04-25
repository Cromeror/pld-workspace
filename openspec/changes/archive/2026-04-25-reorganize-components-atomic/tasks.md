# Tasks: Reorganize `pld-web` components under strict Atomic Design

## Phase 1 — Preflight [Web]

- [x] 1.1 Verificar que `git status` en `pld-web/` está limpio (sin cambios no commiteados).
- [x] 1.2 Ejecutar `find pld-web/src/components -maxdepth 2 -type d` y confirmar que existen: `atoms/`, `molecules/`, `organisms/`, `auth/`, `create-users/`, `external-users/`, `reporting-entity/`, `ui/`, `icons/`.

## Phase 2 — Atoms: mover desde `ui/` y `external-users/` [Web]

- [x] 2.1 `git mv pld-web/src/components/ui/Button pld-web/src/components/atoms/Button`
- [x] 2.2 `git mv pld-web/src/components/ui/InputText pld-web/src/components/atoms/InputText`
- [x] 2.3 `git mv pld-web/src/components/ui/Checkbox pld-web/src/components/atoms/Checkbox`
- [x] 2.4 `git mv pld-web/src/components/ui/Dropdown pld-web/src/components/atoms/Dropdown`
- [x] 2.5 `git mv pld-web/src/components/ui/RadioButton pld-web/src/components/atoms/RadioButton`
- [x] 2.6 `git mv pld-web/src/components/ui/Calendar pld-web/src/components/atoms/Calendar`
- [x] 2.7 `git mv pld-web/src/components/external-users/TypeItem pld-web/src/components/atoms/TypeItem`
- [x] 2.8 Normalizar bare file: `mkdir -p pld-web/src/components/atoms/ValidationInfoItem && git mv pld-web/src/components/create-users/ValidationInfoItem.tsx pld-web/src/components/atoms/ValidationInfoItem/index.tsx`

## Phase 3 — Molecules: mover desde `create-users/` y `external-users/` [Web]

- [x] 3.1 Normalizar bare file: `mkdir -p pld-web/src/components/molecules/create-users/ValidationItem && git mv pld-web/src/components/create-users/ValidationItem.tsx pld-web/src/components/molecules/create-users/ValidationItem/index.tsx`
- [x] 3.2 `mkdir -p pld-web/src/components/molecules/external-users && git mv pld-web/src/components/external-users/PersonTypeAccordion pld-web/src/components/molecules/external-users/PersonTypeAccordion`
- [x] 3.3 Extraer BeneficiaryOption: `mkdir -p pld-web/src/components/molecules/external-users/BeneficiaryOption && git mv pld-web/src/components/external-users/BeneficiaryController/BeneficiaryOption.tsx pld-web/src/components/molecules/external-users/BeneficiaryOption/index.tsx`

## Phase 4 — Organisms auth [Web]

- [x] 4.1 `mkdir -p pld-web/src/components/organisms/auth && git mv pld-web/src/components/auth/LoginForm pld-web/src/components/organisms/auth/LoginForm`
- [x] 4.2 `git mv pld-web/src/components/auth/ForgotPasswordForm pld-web/src/components/organisms/auth/ForgotPasswordForm`
- [x] 4.3 `git mv pld-web/src/components/auth/RecoverPasswordForm pld-web/src/components/organisms/auth/RecoverPasswordForm`

## Phase 5 — Organisms wizard [Web]

- [x] 5.1 `mkdir -p pld-web/src/components/organisms/wizard && git mv pld-web/src/components/organisms/Wizard pld-web/src/components/organisms/wizard/Wizard`

## Phase 6 — Organisms external-users (10 componentes) [Web]

- [x] 6.1 `mkdir -p pld-web/src/components/organisms/external-users && git mv pld-web/src/components/external-users/PersonType pld-web/src/components/organisms/external-users/PersonType`
- [x] 6.2 `git mv pld-web/src/components/external-users/TypeUser pld-web/src/components/organisms/external-users/TypeUser`
- [x] 6.3 `git mv pld-web/src/components/external-users/Modality pld-web/src/components/organisms/external-users/Modality`
- [x] 6.4 `git mv pld-web/src/components/external-users/IdentityData pld-web/src/components/organisms/external-users/IdentityData`
- [x] 6.5 `git mv pld-web/src/components/external-users/IdentificationData pld-web/src/components/organisms/external-users/IdentificationData`
- [x] 6.6 `git mv pld-web/src/components/external-users/PrivateAddress pld-web/src/components/organisms/external-users/PrivateAddress`
- [x] 6.7 `git mv pld-web/src/components/external-users/PoliticallyExposedPerson pld-web/src/components/organisms/external-users/PoliticallyExposedPerson`
- [x] 6.8 `git mv pld-web/src/components/external-users/BeneficiaryController pld-web/src/components/organisms/external-users/BeneficiaryController`
- [x] 6.9 `git mv pld-web/src/components/external-users/BeneficiaryControllerData pld-web/src/components/organisms/external-users/BeneficiaryControllerData`
- [x] 6.10 `git mv pld-web/src/components/external-users/ReviewAndValidation pld-web/src/components/organisms/external-users/ReviewAndValidation`

## Phase 7 — Organisms reporting-entity (9 componentes) [Web]

- [x] 7.1 `mkdir -p pld-web/src/components/organisms/reporting-entity && git mv pld-web/src/components/reporting-entity/ReportingEntityTypeStep pld-web/src/components/organisms/reporting-entity/ReportingEntityTypeStep`
- [x] 7.2 `git mv pld-web/src/components/reporting-entity/ContactStep pld-web/src/components/organisms/reporting-entity/ContactStep`
- [x] 7.3 `git mv pld-web/src/components/reporting-entity/ComplianceResponsibleStep pld-web/src/components/organisms/reporting-entity/ComplianceResponsibleStep`
- [x] 7.4 `git mv pld-web/src/components/reporting-entity/PhysicalIdentificationStep pld-web/src/components/organisms/reporting-entity/PhysicalIdentificationStep`
- [x] 7.5 `git mv pld-web/src/components/reporting-entity/MoralIdentificationStep pld-web/src/components/organisms/reporting-entity/MoralIdentificationStep`
- [x] 7.6 `git mv pld-web/src/components/reporting-entity/VulnerableActivityStep pld-web/src/components/organisms/reporting-entity/VulnerableActivityStep`
- [x] 7.7 `git mv pld-web/src/components/reporting-entity/ReviewStep pld-web/src/components/organisms/reporting-entity/ReviewStep`
- [x] 7.8 `git mv pld-web/src/components/reporting-entity/SuccessFinalizeModal pld-web/src/components/organisms/reporting-entity/SuccessFinalizeModal`
- [x] 7.9 `git mv pld-web/src/components/reporting-entity/ErrorFinalizeModal pld-web/src/components/organisms/reporting-entity/ErrorFinalizeModal`

## Phase 8 — Reescritura de imports (sed pass) [Web]

Ejecutar en orden estricto (prefijos más largos primero) sobre `pld-web/src/**/*.{ts,tsx}`:

- [x] 8.1 Aplicar las 26 sustituciones del Import Rewrite Plan (diseño §Import Rewrite Plan) en orden mediante `find pld-web/src -type f \( -name '*.ts' -o -name '*.tsx' \) -exec sed -i 's|...|...|g' {} +`. Orden obligatorio:
  1. `@/components/external-users/TypeItem` → `@/components/atoms/TypeItem`
  2. `@/components/create-users/ValidationInfoItem` → `@/components/atoms/ValidationInfoItem`
  3. `@/components/create-users/ValidationItem` → `@/components/molecules/create-users/ValidationItem`
  4. `@/components/external-users/PersonTypeAccordion` → `@/components/molecules/external-users/PersonTypeAccordion`
  5. `@/components/external-users/BeneficiaryController/BeneficiaryOption` → `@/components/molecules/external-users/BeneficiaryOption`
  6. `@/components/external-users/BeneficiaryControllerData` → `@/components/organisms/external-users/BeneficiaryControllerData`
  7. `@/components/external-users/BeneficiaryController` → `@/components/organisms/external-users/BeneficiaryController`
  8. `@/components/external-users/ReviewAndValidation` → `@/components/organisms/external-users/ReviewAndValidation`
  9. `@/components/external-users/IdentityData` → `@/components/organisms/external-users/IdentityData`
  10. `@/components/external-users/IdentificationData` → `@/components/organisms/external-users/IdentificationData`
  11. `@/components/external-users/Modality` → `@/components/organisms/external-users/Modality`
  12. `@/components/external-users/PersonType` → `@/components/organisms/external-users/PersonType`
  13. `@/components/external-users/PoliticallyExposedPerson` → `@/components/organisms/external-users/PoliticallyExposedPerson`
  14. `@/components/external-users/PrivateAddress` → `@/components/organisms/external-users/PrivateAddress`
  15. `@/components/external-users/TypeUser` → `@/components/organisms/external-users/TypeUser`
  16. `@/components/ui/Button` → `@/components/atoms/Button`
  17. `@/components/ui/InputText` → `@/components/atoms/InputText`
  18. `@/components/ui/Checkbox` → `@/components/atoms/Checkbox`
  19. `@/components/ui/Dropdown` → `@/components/atoms/Dropdown`
  20. `@/components/ui/RadioButton` → `@/components/atoms/RadioButton`
  21. `@/components/ui/Calendar` → `@/components/atoms/Calendar`
  22. `@/components/auth/LoginForm` → `@/components/organisms/auth/LoginForm`
  23. `@/components/auth/ForgotPasswordForm` → `@/components/organisms/auth/ForgotPasswordForm`
  24. `@/components/auth/RecoverPasswordForm` → `@/components/organisms/auth/RecoverPasswordForm`
  25. `@/components/organisms/Wizard` → `@/components/organisms/wizard/Wizard`
  26. `@/components/reporting-entity/` → `@/components/organisms/reporting-entity/`
- [x] 8.2 Verificar cero residuos: `grep -rn "@/components/ui\|@/components/auth/\|@/components/create-users\|@/components/external-users\|@/components/reporting-entity" pld-web/src` DEBE retornar 0 matches (INV-5).
- [x] 8.3 Verificar que los imports relativos internos (`./ `) de componentes ya movidos siguen resolviendo (no se modificaron rutas relativas — revisión visual de `organisms/external-users/BeneficiaryControllerData/index.tsx` y `CreateBeneficiaryController.tsx`).

## Phase 9 — Cleanup [Web]

- [x] 9.1 Confirmar que `pld-web/src/components/ui/`, `auth/`, `create-users/`, `external-users/` y `reporting-entity/` están vacíos (solo pueden quedar archivos ocultos como `.DS_Store`).
- [x] 9.2 Eliminar carpetas vacías: `rm -rf pld-web/src/components/ui pld-web/src/components/auth pld-web/src/components/create-users pld-web/src/components/external-users pld-web/src/components/reporting-entity`
- [x] 9.3 Verificar estructura final: `find pld-web/src/components -maxdepth 1 -type d` debe retornar SOLO `atoms`, `molecules`, `organisms`, `icons` (más el root) (INV-1, INV-7).

## Phase 10 — Verificación y commit [Web]

- [x] 10.1 `cd pld-web && yarn tsc -b --noEmit` → exit 0, 0 diagnósticos (INV-8).
- [x] 10.2 `cd pld-web && yarn build` → exit 0 (INV-8).
- [x] 10.3 `grep -rn "@/components/ui\|@/components/auth/\|@/components/create-users\|@/components/external-users\|@/components/reporting-entity" pld-web/src` → 0 matches (INV-5).
- [ ] 10.4 Smoke manual: levantar stack, navegar login → credenciales válidas → wizard renderiza correctamente (INV-9).
- [ ] 10.5 Commit atómico en `pld-web/`: `refactor(components): enforce strict Atomic Design taxonomy`.
