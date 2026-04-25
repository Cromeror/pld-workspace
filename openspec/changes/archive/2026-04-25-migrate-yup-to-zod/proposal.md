# Proposal: Migrate yup → zod in pld-web

**Status**: draft
**Created**: 2026-04-24
**Author**: workspace pld
**Sub-repo**: pld-web

## Intent

Replace the `yup` validation library with `zod` across all form schemas in `pld-web`, and collapse the Wizard's duplicated per-step validators into pure `schema.safeParse(state).success` calls.

## Scope

**In**:
- 7 files currently using yup:
  - `pld-web/src/components/auth/LoginForm/index.tsx`
  - `pld-web/src/components/auth/ForgotPasswordForm/index.tsx`
  - `pld-web/src/components/auth/RecoverPasswordForm/index.tsx`
  - `pld-web/src/components/external-users/IdentityData/index.tsx`
  - `pld-web/src/components/external-users/PrivateAddress/index.tsx`
  - `pld-web/src/components/external-users/IdentificationData/index.tsx`
  - `pld-web/src/components/external-users/PoliticallyExposedPerson/index.tsx`
- `pld-web/src/pages/users/CreateUserPage.tsx` — remove the 4 duplicated step-validators (`isIdentityComplete`, `isPrivateAddressComplete`, `isIdentificationComplete`, `isPepComplete`) and the 4 `*Validated` flags in `State`.
- `pld-web/package.json` — add `zod`, remove `yup`.

**Out**:
- No BE changes.
- No changes to the Wizard organism itself (its API already supports `isValid: (state) => boolean`).
- No changes to content-components' visual output or UX.
- No additions of testing infrastructure (pld-web has none yet).

## Motivation

1. **User preference**: the user prefers zod for schema validation.
2. **Eliminate duplication**: today each content-component has a yup schema AND `CreateUserPage.tsx` re-implements the same validation as 4 imperative functions. With zod schemas exported and consumed via `safeParse`, the second copy disappears.
3. **Remove redundant state**: the `*Validated` flags exist only because content-components push `onValidated(boolean)` upward. With schema-driven validation, the wizard reads directly from state — the flags are no longer needed.

## Approach

1. Install `zod`, verify `@hookform/resolvers@5.2.2` exposes `zodResolver`.
2. Migrate auth forms (3) — internal schemas, no export needed.
3. Migrate wizard content-components (4) — each exports its schema as named export.
4. Rewire `CreateUserPage.tsx` to import those schemas and use `safeParse(...).success` in each `step.isValid`.
5. Remove `*Validated` flags from `State` and `onValidated` from content-component props.
6. Uninstall `yup`. Verify `tsc -b` and `yarn build` pass.
7. Manual smoke test in browser for each form.

## Rollback plan

Revert the commit. `yup` remains in `yarn.lock` history; `yarn install` restores it. Content-components and CreateUserPage.tsx go back to yup schemas + flags-based validation.

## Affected surfaces

- Frontend only. No BE touched.
- No migration of persistent data.
- No change to HTTP contracts.
