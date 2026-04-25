# Design: Migrate yup → zod

## Decisions

### D1 — Schema location: co-located with component (option A from refine)

Each content-component continues to own its schema. The schema is declared at the top of `index.tsx` and exported as a named const (`identitySchema`, `privateAddressSchema`, `identificationSchema`, `pepSchema`). Auth-form schemas (login, forgot-password, recover-password) stay internal — they're not consumed by the Wizard.

**Why**: schemas are 1:1 with their component. Moving them to `src/validation/` would split related code and complicate the import graph without reuse benefit. If a schema gets reused across features later, promote it then.

### D2 — Single atomic change (option from refine A2)

One SDD change covers all 7 migrations + the wizard simplification + the yup uninstall. Rationale: the uninstall requires zero remaining yup imports; splitting into sub-changes would leave the repo in a mixed state where `yup` is still in `package.json` but partially unused.

### D3 — Error message pattern

Preserve exact Spanish error text. Use zod's `{ message: '...' }` parameter on constraint methods:

```ts
// Before (yup)
email: yup.string().email("Correo inválido").required("Email requerido")

// After (zod)
email: z.string({ required_error: "Email requerido" }).email("Correo inválido").min(1, "Email requerido")
```

For required fields, use `.min(1, 'msg')` on strings (zod's `required_error` only fires on `undefined`, not empty string — RHF typically initializes fields to `""`).

### D4 — Remove `*Validated` flags and `onValidated` callbacks

The flags exist in `State` only as a signal for step-validators in `CreateUserPage.tsx`. Once validators become `schema.safeParse(state.field).success`, the flags are dead code. Content-components drop their `onValidated` prop.

Impact: content-components have one less prop. Their internal yup/zod schema + `zodResolver` still drive the form's field-level error display via react-hook-form — nothing visual changes.

### D5 — PEP conditional validation (L2 from refine)

The yup schema in `PoliticallyExposedPerson` uses field-level `.when()` conditions. Zod equivalent: `.superRefine` or `.refine` on the object. Use `.superRefine` when multiple conditional paths exist so errors can be attached to the right field via `ctx.addIssue({ path: [...], message: '...' })`.

```ts
const pepSchema = z.object({
  isSubscriberPEP: z.boolean(),
  subscriberPosition: z.string().optional(),
  hasRelativePEP: z.boolean(),
  relativePosition: z.string().optional(),
  relativePEPFullName: z.string().optional(),
}).superRefine((data, ctx) => {
  if (data.isSubscriberPEP && !data.subscriberPosition) {
    ctx.addIssue({ code: 'custom', path: ['subscriberPosition'], message: 'Cargo requerido' });
  }
  if (data.hasRelativePEP && (!data.relativePosition || !data.relativePEPFullName)) {
    if (!data.relativePosition) ctx.addIssue({ code: 'custom', path: ['relativePosition'], message: 'Cargo requerido' });
    if (!data.relativePEPFullName) ctx.addIssue({ code: 'custom', path: ['relativePEPFullName'], message: 'Nombre requerido' });
  }
});
```

### D6 — `birthDate` as Date object

The current yup schema is `yup.date().nullable().required(...)`. The `Calendar` component from PrimeReact (via RHF `Controller`) emits `Date` objects. Keep `z.date({ required_error: '...' }).nullable().refine(v => v !== null, 'msg')` — no coercion needed.

### D7 — `acceptedTerms` literal-true (L3 from refine)

```ts
// yup
acceptedTerms: yup.boolean().oneOf([true], 'msg').required('msg')

// zod
acceptedTerms: z.boolean().refine(v => v === true, { message: 'msg' })
```

### D8 — zod version

Use latest zod v3 (3.23+) for maximum compatibility with `@hookform/resolvers@5`. Zod v4 is newer but resolvers matrix is still catching up. If v4 works end-to-end during apply, switch; otherwise stick to v3.

### D9 — No new spec file

`pld/openspec/specs/` has BE-only specs. This change is frontend-internal refactor without an observable contract shift. Document the decision here in design.md and reference it from `archive-report.md`; do not create a `frontend-validation` spec.

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Zod message API differs → subtle text changes | Low (visual regression in error messages) | Take screenshots of each form with invalid state before and after. Compare. |
| `zodResolver` incompatible with `@hookform/resolvers@5.2.2` | Medium (blocks migration) | First task: create a throwaway branch, import `zodResolver`, confirm it types. Bump resolver if needed. |
| Forgotten yup import somewhere | Low (build fails) | `grep -r "yup" src/` before uninstall. Gate the uninstall behind passing `tsc`. |
| PEP schema misconfigured → wrong error path | Medium (user sees errors on wrong field) | Manually test all 4 PEP paths (both booleans true/false combinations). |
| `safeParse(undefined)` behavior on initial render | Low (button disabled as expected) | Verified L6 in refine: `safeParse(undefined)` fails → button stays disabled. That's the desired initial UX. |

## Open questions

None at design time. Any surprise during apply will surface as a task addition.
