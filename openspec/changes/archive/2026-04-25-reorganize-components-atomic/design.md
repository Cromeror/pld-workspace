# Design: Reorganize `pld-web` components under strict Atomic Design

Pure structural change — no API, no behavior, no sequence diagrams. Cada decisión referencia su ID (D1..D12) acordado con el user.

## Technical Approach

Mechanical reorganization of `pld-web/src/components/` into the strict Atomic Design taxonomy defined by the spec. Three phases, in this order: (a) move 52 components with `git mv`, (b) rewrite all `@/components/...` imports, (c) delete emptied legacy folders. Verification via `yarn tsc -b --noEmit` then `yarn build`. The `@/` alias from `vite.config.ts` and `tsconfig.app.json` already covers every new path — no tooling change.

## Architecture Decisions

### D1 — Eliminate `ui/` folder

**Choice**: promote every primitive in `components/ui/` to `components/atoms/`. Remove `ui/` entirely.
**Alternatives**: keep `ui/` as the atomic level name.
**Rationale**: the project already adopted `atoms/` (Atomic Design canonical name); keeping `ui/` would split primitives across two coexisting names and contradict INV-1.

### D2 — Subfolder by feature inside `organisms/` and `molecules/`

**Choice**: organisms always nested as `organisms/<feature>/<Name>/`. Molecules nested as `molecules/<feature>/<Name>/` only when feature-specific; generic molecules stay flat at `molecules/<Name>/`.
**Alternatives**: flat `organisms/` with all 30+ components as siblings.
**Rationale**: 30+ flat siblings would be unbrowsable. Feature subfolder gives a stable mental model: read level → read feature → read component.

### D3 — Atoms remain flat

**Choice**: no feature subfolders inside `atoms/`. Every primitive at `atoms/<Name>/index.tsx`.
**Alternatives**: nest by category (`atoms/inputs/`, `atoms/feedback/`, ...).
**Rationale**: atoms are domain-agnostic primitives. Categorical subfolders invite endless bikeshedding ("is `Calendar` input or feedback?") and contradict INV-2.

### D4 — `git mv` for renames

**Choice**: every move uses `git mv` so blame and history follow.
**Alternatives**: `cp` + `rm`.
**Rationale**: history loss would block `git log --follow` and `git blame` post-rename, making future investigation expensive.

### D5 — Mechanical import rewrite

**Choice**: scripted rewrite via `find ... -exec sed -i ...` over `pld-web/src/` for each `@/components/<old-path>` → `@/components/<new-path>` pair. Pairs are explicit and ordered (longer prefixes first, so `external-users/PersonTypeAccordion` is rewritten before bare `external-users/`).
**Alternatives**: hand-edit each import.
**Rationale**: 28 imports across 6 files plus 3 cross-imports inside `external-users/` — manual editing risks typos and missed sites.

### D6 — Normalize bare `.tsx` to `<Name>/index.tsx`

**Choice**: `ValidationItem.tsx`, `ValidationInfoItem.tsx`, and `BeneficiaryController/BeneficiaryOption.tsx` become `<Name>/index.tsx`.
**Alternatives**: keep them as bare files in their new homes.
**Rationale**: every other component already uses the folder pattern (INV-10). Mixing both forms breaks the convention promise.

### D7 — Wizard placement under `organisms/wizard/`

**Choice**: the generic `Wizard` organism moves from `organisms/Wizard/` to `organisms/wizard/Wizard/`.
**Alternatives**: leave `Wizard` flat under `organisms/` as the only exception.
**Rationale**: INV-4 forbids flat organisms. Treating `wizard` as a feature-style subfolder keeps the rule absolute (no exceptions). Cost: one extra folder level. Benefit: one rule, no exceptions to remember.

### D8 — Icons untouched

**Choice**: `components/icons/` stays as-is alongside `atoms/`, `molecules/`, `organisms/`.
**Alternatives**: nest under `atoms/icons/`.
**Rationale**: `icons/` is the idiomatic React convention for SVG component collections. Folding it under `atoms/` adds noise (`atoms/icons/Check/index.tsx`) without benefit. INV-1 explicitly allows `icons/` as a fourth top-level child.

### D9 — Empty old folders removed

**Choice**: after moves complete, delete `ui/`, `auth/`, `create-users/`, `external-users/`, `reporting-entity/` entirely.
**Alternatives**: leave them empty as "transition aid".
**Rationale**: empty folders are noise and break INV-7.

### D10 — Single PR/commit

**Choice**: one atomic commit for the whole reorganization.
**Alternatives**: split per feature folder (`auth` first, then `external-users`, etc.).
**Rationale**: split commits leave intermediate states where some imports still reference old paths and TypeScript breaks. Atomic commit keeps every commit green.

### D11 — Apply order

**Choice**: (a) move all files; (b) rewrite all imports; (c) delete empty folders; (d) `yarn tsc -b --noEmit` + `yarn build`.
**Alternatives**: incremental per-folder migration.
**Rationale**: same reason as D10 — only the final state is buildable, no intermediate state is.

### D12 — Path alias `@/`

**Choice**: every rewritten import uses `@/components/<level>/<feature?>/<Name>` via the existing `@/` alias.
**Alternatives**: relative imports.
**Rationale**: alias is already configured (`vite.config.ts`, `tsconfig.app.json`). Relative imports across 5+ folder levels (`../../../../components/...`) are unreadable.

## Reorganization Map

### → `atoms/` (flat, 9 entries)

| Current Path | New Path |
|---|---|
| `components/atoms/Dialog/index.tsx` | `components/atoms/Dialog/index.tsx` |
| `components/ui/Button/index.tsx` | `components/atoms/Button/index.tsx` |
| `components/ui/InputText/index.tsx` | `components/atoms/InputText/index.tsx` |
| `components/ui/Checkbox/index.tsx` | `components/atoms/Checkbox/index.tsx` |
| `components/ui/Dropdown/index.tsx` | `components/atoms/Dropdown/index.tsx` |
| `components/ui/RadioButton/index.tsx` | `components/atoms/RadioButton/index.tsx` |
| `components/ui/Calendar/index.tsx` | `components/atoms/Calendar/index.tsx` |
| `components/external-users/TypeItem/index.tsx` | `components/atoms/TypeItem/index.tsx` |
| `components/create-users/ValidationInfoItem.tsx` | `components/atoms/ValidationInfoItem/index.tsx` |

### → `molecules/` (3 generic + 3 feature-scoped)

| Current Path | New Path |
|---|---|
| `components/molecules/Stepper/index.tsx` | `components/molecules/Stepper/index.tsx` |
| `components/molecules/ConfirmDialog/index.tsx` | `components/molecules/ConfirmDialog/index.tsx` |
| `components/molecules/AdministrativeDivisionsField/index.tsx` | `components/molecules/AdministrativeDivisionsField/index.tsx` |
| `components/external-users/PersonTypeAccordion/index.tsx` | `components/molecules/external-users/PersonTypeAccordion/index.tsx` |
| `components/external-users/BeneficiaryController/BeneficiaryOption.tsx` | `components/molecules/external-users/BeneficiaryOption/index.tsx` |
| `components/create-users/ValidationItem.tsx` | `components/molecules/create-users/ValidationItem/index.tsx` |

### → `organisms/auth/` (3)

| Current Path | New Path |
|---|---|
| `components/auth/LoginForm/index.tsx` | `components/organisms/auth/LoginForm/index.tsx` |
| `components/auth/ForgotPasswordForm/index.tsx` | `components/organisms/auth/ForgotPasswordForm/index.tsx` |
| `components/auth/RecoverPasswordForm/index.tsx` | `components/organisms/auth/RecoverPasswordForm/index.tsx` |

### → `organisms/wizard/` (1)

| Current Path | New Path |
|---|---|
| `components/organisms/Wizard/index.tsx` | `components/organisms/wizard/Wizard/index.tsx` |

### → `organisms/external-users/` (10)

| Current Path | New Path |
|---|---|
| `components/external-users/IdentityData/index.tsx` | `components/organisms/external-users/IdentityData/index.tsx` |
| `components/external-users/IdentificationData/index.tsx` | `components/organisms/external-users/IdentificationData/index.tsx` |
| `components/external-users/Modality/index.tsx` | `components/organisms/external-users/Modality/index.tsx` |
| `components/external-users/PersonType/index.tsx` | `components/organisms/external-users/PersonType/index.tsx` |
| `components/external-users/PoliticallyExposedPerson/index.tsx` | `components/organisms/external-users/PoliticallyExposedPerson/index.tsx` |
| `components/external-users/PrivateAddress/index.tsx` | `components/organisms/external-users/PrivateAddress/index.tsx` |
| `components/external-users/TypeUser/index.tsx` | `components/organisms/external-users/TypeUser/index.tsx` |
| `components/external-users/ReviewAndValidation/index.tsx` | `components/organisms/external-users/ReviewAndValidation/index.tsx` |
| `components/external-users/BeneficiaryController/index.tsx` | `components/organisms/external-users/BeneficiaryController/index.tsx` |
| `components/external-users/BeneficiaryControllerData/index.tsx` | `components/organisms/external-users/BeneficiaryControllerData/index.tsx` |
| `components/external-users/BeneficiaryControllerData/CreateBeneficiaryController.tsx` | `components/organisms/external-users/BeneficiaryControllerData/CreateBeneficiaryController.tsx` |

### → `organisms/reporting-entity/` (9)

| Current Path | New Path |
|---|---|
| `components/reporting-entity/ComplianceResponsibleStep/index.tsx` | `components/organisms/reporting-entity/ComplianceResponsibleStep/index.tsx` |
| `components/reporting-entity/ContactStep/index.tsx` | `components/organisms/reporting-entity/ContactStep/index.tsx` |
| `components/reporting-entity/ErrorFinalizeModal/index.tsx` | `components/organisms/reporting-entity/ErrorFinalizeModal/index.tsx` |
| `components/reporting-entity/MoralIdentificationStep/index.tsx` | `components/organisms/reporting-entity/MoralIdentificationStep/index.tsx` |
| `components/reporting-entity/PhysicalIdentificationStep/index.tsx` | `components/organisms/reporting-entity/PhysicalIdentificationStep/index.tsx` |
| `components/reporting-entity/ReportingEntityTypeStep/index.tsx` | `components/organisms/reporting-entity/ReportingEntityTypeStep/index.tsx` |
| `components/reporting-entity/ReviewStep/index.tsx` | `components/organisms/reporting-entity/ReviewStep/index.tsx` |
| `components/reporting-entity/SuccessFinalizeModal/index.tsx` | `components/organisms/reporting-entity/SuccessFinalizeModal/index.tsx` |
| `components/reporting-entity/VulnerableActivityStep/index.tsx` | `components/organisms/reporting-entity/VulnerableActivityStep/index.tsx` |

### → `icons/` (12, untouched per D8)

`Key`, `Login`, `Fingerprint`, `Check`, `CheckCircle`, `ArrowRight`, `Plus`, `Edit`, `Info`, `ChevronDown`, `NewRecord`, `WhatsApp` — no change.

## Import Rewrite Plan

Apply the following replacements in this exact order over `pld-web/src/**/*.{ts,tsx}` via `find ... -exec sed -i 's|<old>|<new>|g' {} +`. **Order matters**: longer/more specific prefixes MUST run before shorter ones to avoid partial overwrites (e.g. `external-users/TypeItem` before `external-users/`).

| # | Old import | New import |
|---|---|---|
| 1 | `@/components/ui/Button` | `@/components/atoms/Button` |
| 2 | `@/components/ui/InputText` | `@/components/atoms/InputText` |
| 3 | `@/components/ui/Checkbox` | `@/components/atoms/Checkbox` |
| 4 | `@/components/ui/Dropdown` | `@/components/atoms/Dropdown` |
| 5 | `@/components/ui/RadioButton` | `@/components/atoms/RadioButton` |
| 6 | `@/components/ui/Calendar` | `@/components/atoms/Calendar` |
| 7 | `@/components/external-users/TypeItem` | `@/components/atoms/TypeItem` |
| 8 | `@/components/create-users/ValidationInfoItem` | `@/components/atoms/ValidationInfoItem` |
| 9 | `@/components/create-users/ValidationItem` | `@/components/molecules/create-users/ValidationItem` |
| 10 | `@/components/external-users/PersonTypeAccordion` | `@/components/molecules/external-users/PersonTypeAccordion` |
| 11 | `@/components/external-users/BeneficiaryController/BeneficiaryOption` | `@/components/molecules/external-users/BeneficiaryOption` |
| 12 | `@/components/auth/LoginForm` | `@/components/organisms/auth/LoginForm` |
| 13 | `@/components/auth/ForgotPasswordForm` | `@/components/organisms/auth/ForgotPasswordForm` |
| 14 | `@/components/auth/RecoverPasswordForm` | `@/components/organisms/auth/RecoverPasswordForm` |
| 15 | `@/components/organisms/Wizard` | `@/components/organisms/wizard/Wizard` |
| 16 | `@/components/external-users/IdentityData` | `@/components/organisms/external-users/IdentityData` |
| 17 | `@/components/external-users/IdentificationData` | `@/components/organisms/external-users/IdentificationData` |
| 18 | `@/components/external-users/Modality` | `@/components/organisms/external-users/Modality` |
| 19 | `@/components/external-users/PersonType` | `@/components/organisms/external-users/PersonType` |
| 20 | `@/components/external-users/PoliticallyExposedPerson` | `@/components/organisms/external-users/PoliticallyExposedPerson` |
| 21 | `@/components/external-users/PrivateAddress` | `@/components/organisms/external-users/PrivateAddress` |
| 22 | `@/components/external-users/TypeUser` | `@/components/organisms/external-users/TypeUser` |
| 23 | `@/components/external-users/ReviewAndValidation` | `@/components/organisms/external-users/ReviewAndValidation` |
| 24 | `@/components/external-users/BeneficiaryController` | `@/components/organisms/external-users/BeneficiaryController` |
| 25 | `@/components/external-users/BeneficiaryControllerData` | `@/components/organisms/external-users/BeneficiaryControllerData` |
| 26 | `@/components/reporting-entity/` | `@/components/organisms/reporting-entity/` |

After step 26 a single `grep -r "@/components/\(ui\|auth\|create-users\|external-users\|reporting-entity\)" pld-web/src` MUST return zero matches (per INV-5).

### Files touched by import rewrite

External consumers (6 files):
- `pld-web/src/pages/auth/LoginPage.tsx`
- `pld-web/src/pages/auth/ForgotPasswordPage.tsx`
- `pld-web/src/pages/auth/RecoverPasswordPage.tsx`
- `pld-web/src/pages/users/CreateUserPage.tsx`
- `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx`
- `pld-web/src/components/reporting-entity/ReportingEntityTypeStep/index.tsx` (cross-reference to `external-users/PersonTypeAccordion` and `external-users/TypeItem`)

Internal cross-imports inside `external-users/` and `create-users/` (3 files, will become `organisms/external-users/...` after move):
- `external-users/BeneficiaryControllerData/index.tsx` → imports `ValidationItem`
- `external-users/BeneficiaryControllerData/CreateBeneficiaryController.tsx` → imports `ValidationItem`
- `external-users/ReviewAndValidation/index.tsx` → imports `ValidationItem`

Total: 9 files modified, 28 import statements rewritten.

## File Ops Table

Abbreviated form below. Full source/destination table is in **Reorganization Map** above.

| Operation | Source | Destination |
|---|---|---|
| `git mv` (×6) | `components/ui/{Button,InputText,Checkbox,Dropdown,RadioButton,Calendar}/` | `components/atoms/<Name>/` |
| `git mv` (×1) | `components/external-users/TypeItem/` | `components/atoms/TypeItem/` |
| `git mv` + rename | `components/create-users/ValidationInfoItem.tsx` | `components/atoms/ValidationInfoItem/index.tsx` |
| `git mv` + rename | `components/create-users/ValidationItem.tsx` | `components/molecules/create-users/ValidationItem/index.tsx` |
| `git mv` (×1) | `components/external-users/PersonTypeAccordion/` | `components/molecules/external-users/PersonTypeAccordion/` |
| `git mv` + rename | `components/external-users/BeneficiaryController/BeneficiaryOption.tsx` | `components/molecules/external-users/BeneficiaryOption/index.tsx` |
| `git mv` (×3) | `components/auth/{LoginForm,ForgotPasswordForm,RecoverPasswordForm}/` | `components/organisms/auth/<Name>/` |
| `git mv` (×1) | `components/organisms/Wizard/` | `components/organisms/wizard/Wizard/` |
| `git mv` (×10) | `components/external-users/{IdentityData,IdentificationData,Modality,PersonType,PoliticallyExposedPerson,PrivateAddress,TypeUser,ReviewAndValidation,BeneficiaryController,BeneficiaryControllerData}/` | `components/organisms/external-users/<Name>/` |
| `git mv` (×9) | `components/reporting-entity/<Name>/` | `components/organisms/reporting-entity/<Name>/` |
| Modify | 9 files (6 consumers + 3 internal cross-imports) | Import paths rewritten |
| Delete | `components/ui/` | empty after moves |
| Delete | `components/auth/` | empty after moves |
| Delete | `components/create-users/` | empty after moves |
| Delete | `components/external-users/` | empty after moves |
| Delete | `components/reporting-entity/` | empty after moves |

## Tradeoffs

| Tradeoff | Choice | Rationale |
|---|---|---|
| Atomic Design strict vs feature-first | **Strict** — atomic level first, feature subfolder second (organisms always, molecules only when feature-specific) | Single rule, scales with the codebase, matches the existing `atoms/`/`molecules/`/`organisms/` skeleton already in place. |
| Allow flat organisms as exception (e.g. `Wizard`) vs require feature subfolder always | **Always require feature subfolder** (D7) | One rule, no exceptions to remember. The cost is an extra folder level for `Wizard`; the benefit is INV-4 stays absolute. |
| Branch from this point on naming | Any future component MUST obey: atoms flat; molecules flat if generic, subfoldered if feature-specific; organisms always under feature subfolder; icons stay in `icons/` | Makes the convention enforceable in code review and documents the rule for future contributors. |

## Open Questions

- None. Every move target, import rewrite pair, and apply-order step is determined.
