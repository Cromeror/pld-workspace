# Tasks: Migrate yup → zod

## Phase 0 — Preflight

- [x] 0.1 `cd pld-web && yarn add zod` (install latest stable v3; evaluate v4 later).
- [x] 0.2 Verify `@hookform/resolvers` exposes `zodResolver`: `node -e "console.log(Object.keys(require('@hookform/resolvers/zod')))"` should list `zodResolver`.
- [x] 0.3 If `zodResolver` is missing, bump `@hookform/resolvers` to the version that includes it and re-run 0.2.
- [x] 0.4 Baseline: `yarn tsc -b --noEmit` passes. Take screenshot of each form in invalid state (login, forgot, recover, identity, private-address, identification, PEP) for visual diff after.

## Phase 1 — Auth forms

- [x] 1.1 Migrate `pld-web/src/components/auth/LoginForm/index.tsx` — swap yup → zod. Preserve error messages. Handle `acceptedTerms` literal-true via `.refine`.
- [x] 1.2 Migrate `pld-web/src/components/auth/ForgotPasswordForm/index.tsx`.
- [x] 1.3 Migrate `pld-web/src/components/auth/RecoverPasswordForm/index.tsx`.
- [x] 1.4 `yarn tsc -b --noEmit` passes. Manual smoke test of the 3 auth forms in browser: valid submit, invalid states, error text unchanged.

## Phase 2 — Wizard content-components

- [x] 2.1 Migrate `pld-web/src/components/external-users/IdentityData/index.tsx`. Export `export const identitySchema = z.object({...})`. Drop `onValidated` prop. Drop `useEffect` that calls `onValidatedRef.current(isValid)`.
- [x] 2.2 Migrate `pld-web/src/components/external-users/PrivateAddress/index.tsx`. Export `privateAddressSchema`. Drop `onValidated`.
- [x] 2.3 Migrate `pld-web/src/components/external-users/IdentificationData/index.tsx`. Export `identificationSchema`. Drop `onValidated`.
- [x] 2.4 Migrate `pld-web/src/components/external-users/PoliticallyExposedPerson/index.tsx`. Export `pepSchema` with `superRefine` for conditional fields. Drop `onValidated`.
- [x] 2.5 `yarn tsc -b --noEmit` passes (CreateUserPage.tsx will break temporarily at this point — acceptable intermediate state).

## Phase 3 — Wizard consumer cleanup

- [x] 3.1 In `pld-web/src/pages/users/CreateUserPage.tsx`:
  - Remove `isIdentityComplete`, `isPrivateAddressComplete`, `isIdentificationComplete`, `isPepComplete` functions.
  - Import `identitySchema`, `privateAddressSchema`, `identificationSchema`, `pepSchema` from the content-components.
  - Replace `isValid:` in the 4 corresponding steps with `(s) => xxxSchema.safeParse(s.xxxData).success`.
- [x] 3.2 Remove from `State` type: `identityDataValidated`, `privateAddressDataValidated`, `identificationDataValidated`, `politicallyExposedPersonDataValidated`.
- [x] 3.3 Remove from `initialState` object: those 4 flags.
- [x] 3.4 Remove from the `render` callbacks the `onValidated={...}` props (they no longer exist in content-components).
- [x] 3.5 `yarn tsc -b --noEmit` passes.

## Phase 4 — Cleanup

- [x] 4.1 `grep -r "yup\|yupResolver" pld-web/src/` must return 0 matches.
- [x] 4.2 `cd pld-web && yarn remove yup`.
- [x] 4.3 `grep -E '"yup"' pld-web/package.json pld-web/yarn.lock` must return 0 matches.
- [x] 4.4 `yarn tsc -b --noEmit` passes.
- [x] 4.5 `yarn build` passes.

## Phase 5 — Manual verification

- [ ] 5.1 `/stack-up` (if not running). Open http://localhost:4200.
- [ ] 5.2 **Login form**: empty submit shows all errors in Spanish. Invalid email shows "Correo electrónico inválido". Short password shows "La contraseña debe tener al menos 8 caracteres". Unchecked terms shows terms error. Valid submit navigates to `/create-user`.
- [ ] 5.3 **Forgot password form**: empty email error, valid email proceeds.
- [ ] 5.4 **Recover password form**: validation parity with yup version.
- [ ] 5.5 **Create user wizard — step Identity**: Next button disabled until all required fields filled with valid values. CURP/RFC/email errors display. Button enables when schema is satisfied.
- [ ] 5.6 **Create user wizard — step Private address**: Next disabled until all required fields. Button enables on complete.
- [ ] 5.7 **Create user wizard — step Identification**: same.
- [ ] 5.8 **Create user wizard — step PEP**: if `isSubscriberPEP=true`, Next disabled until `subscriberPosition` filled. If `hasRelativePEP=true`, Next disabled until both `relativePosition` and `relativePEPFullName` filled. Both false → Next enabled immediately.
- [ ] 5.9 **Create user wizard — full flow**: complete all steps, hit "Validar" on Review. Verify no console errors.

## Phase 6 — Commit

- [ ] 6.1 `git add` only pld-web files. Commit with conventional message: `refactor(web): migrate yup schemas to zod, remove redundant *Validated flags`.
- [ ] 6.2 Optional: archive this change with `/sdd-archive migrate-yup-to-zod`.
