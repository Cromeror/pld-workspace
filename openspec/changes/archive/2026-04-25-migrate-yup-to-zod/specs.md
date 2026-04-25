# Specs: Migrate yup → zod

No spec delta. This change is a frontend-internal library swap with no observable contract change at the workspace spec level.

See [design.md §D9](design.md) for rationale.

## Behavioral invariants preserved

The following observable behaviors MUST remain unchanged after the change is applied:

### INV-1 — Error messages (Given/When/Then)

**Given** any form in pld-web that previously displayed a yup error message,
**When** the same invalid input is entered,
**Then** the exact Spanish text SHALL be displayed in the same visual location.

### INV-2 — Field-level validation triggers

**Given** a form with fields wired to react-hook-form,
**When** a field changes value,
**Then** validation SHALL run and errors SHALL update with the same cadence as before (onChange mode).

### INV-3 — Wizard step-gating

**Given** a wizard step with a schema,
**When** the corresponding slice of state does not satisfy the schema,
**Then** the "Siguiente" button SHALL be disabled.

**When** the slice satisfies the schema,
**Then** the button SHALL be enabled.

### INV-4 — PEP conditional fields

**Given** the PEP step is active,
**When** `isSubscriberPEP=true` and `subscriberPosition` is empty,
**Then** the step SHALL be considered invalid.

**When** `hasRelativePEP=true` and either `relativePosition` or `relativePEPFullName` is empty,
**Then** the step SHALL be considered invalid.

**When** both booleans are `false`,
**Then** the step SHALL be considered valid regardless of the optional fields.

### INV-5 — Button labels

**Given** the wizard is on the last step,
**When** the button is rendered,
**Then** it SHALL display `"Validar"`.

**Given** the wizard is not on the last step,
**When** the button is rendered,
**Then** it SHALL display `"Siguiente"`.

### INV-6 — No visible regression on build

**Given** the migration is complete,
**When** `yarn tsc -b --noEmit` is run,
**Then** it SHALL pass with zero errors.

**When** `yarn build` is run,
**Then** it SHALL produce a production bundle with zero errors.
