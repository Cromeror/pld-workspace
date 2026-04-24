# Specs: Registration of reporting entities (`/admin/registration/*`)

> RFC 2119 — **MUST / SHALL**, **SHOULD**, **MAY**.
>
> Cross-cutting rules that apply to every endpoint below:
>
> - Every endpoint MUST require `JwtAuthGuard` + `RolesGuard` + `@Roles(UserRole.SUPERADMIN)`. Missing JWT → `401`. Valid JWT with another role → `403`.
> - Every POST body MUST be validated by `class-validator` inside its DTO. Missing/invalid field → `400` with `{ statusCode, message: string[], error: 'Bad Request' }`.
> - `:id` MUST be a UUID v4. Invalid format → `400`.
> - `profile_type` ∈ `{ PERSONA_FISICA, PERSONA_MORAL }` (from `TipoPersonaParticipante`).
> - `user_role` ∈ `{ NOTARIO, INMOBILIARIA }` (subset of `UserRole`).
> - `current_step` ∈ `{ IDENTIFICATION, CONTACT, VULNERABLE_ACTIVITY, COMPLIANCE_RESPONSIBLE, COMPLETED }`.
> - `status` ∈ `{ IN_PROGRESS, COMPLETED, CANCELLED }`.
> - For `status = COMPLETED`, any further `POST` on a step endpoint MUST return `409` (`"Registration already finalized"`).
> - For `expires_at < NOW()`, `GET /admin/registration/:id` MUST return `410 Gone`. Step `POST`s on expired drafts MUST return `410 Gone` as well.
> - The state machine ordering is strict: `IDENTIFICATION → CONTACT → VULNERABLE_ACTIVITY → [COMPLIANCE_RESPONSIBLE (PM only)] → COMPLETED`. Skipping a step MUST return `409` (`"Missing prior step: <stepName>"`).
> - On an `IN_PROGRESS` draft, re-POSTing a step that was already saved MUST overwrite it (idempotent with overwrite). The step is rewritten, `current_step` MUST NOT regress. Response is `200 OK` on overwrite, `201 Created` on first write.
> - Error body is uniform: `{ statusCode, message, error, timestamp, path }` (already enforced by the shared `HttpErrorInterceptor`).

---

## 1. `POST /admin/registration` — Start a draft

### Description
Creates an `IN_PROGRESS` registration draft. The `user` row is NOT created here.

### Preconditions
- Caller MUST be authenticated as `SUPERADMIN`.

### Request body (`CreateRegistrationDto`)
```jsonc
{
  "profileType": "PERSONA_FISICA" | "PERSONA_MORAL",   // MUST @IsEnum(TipoPersonaParticipante)
  "userRole":    "NOTARIO" | "INMOBILIARIA",           // MUST @IsIn([NOTARIO, INMOBILIARIA])
  "rfc":         "string"                              // MUST, regex (13 chars PF / 12 chars PM)
}
```

### Rules
- Endpoint MUST reject `userRole = NOTARIO` with `profileType = PERSONA_MORAL` → `400` (`"NOTARIO role only supports PERSONA_FISICA"`).
- Endpoint MUST open a DB transaction and, inside it, `SELECT ... FOR UPDATE` any `registration` with `status = 'IN_PROGRESS'` AND same `rfc` (on the nested profile's rfc column — captured as denormalized column on `registration` for dedupe, or materialized in `POST /identification`; see design §Decision 7). Dedupe logic MUST live in the service and MUST be race-safe.
- If a live duplicate exists → `409` with `{ message: "A draft with this RFC is already in progress", registrationId: "<uuid>" }`.
- Endpoint MUST set `expires_at = NOW() + REGISTRATION_TTL_DAYS` (env, default `30`).
- Endpoint MUST set `started_by_user_id = request.user.id`.

### Response shapes / status codes
- `201 Created` →
  ```jsonc
  {
    "id": "uuid",
    "profileType": "...",
    "userRole":    "...",
    "currentStep": "IDENTIFICATION",
    "status":      "IN_PROGRESS",
    "expiresAt":   "ISO-8601",
    "createdAt":   "ISO-8601"
  }
  ```
- `400` — validation or incoherent combination.
- `401` — no JWT.
- `403` — wrong role.
- `409` — active duplicate (RFC).

### Scenarios

**SHALL create an IN_PROGRESS PF draft**
- **Given** a valid SUPERADMIN JWT and no other draft with RFC `AAAA800101ZZZ` in progress
- **When** the client `POST /admin/registration` with `{ profileType: "PERSONA_FISICA", userRole: "NOTARIO", rfc: "AAAA800101ZZZ" }`
- **Then** the response MUST be `201` with `currentStep = "IDENTIFICATION"` and `status = "IN_PROGRESS"`
- **And** a row MUST exist in `registration` with `expires_at ≈ NOW() + 30d`, `started_by_user_id = caller.id`, `user_id = NULL`.

**SHALL reject NOTARIO + PERSONA_MORAL**
- **Given** a valid SUPERADMIN JWT
- **When** `POST /admin/registration` with `{ profileType: "PERSONA_MORAL", userRole: "NOTARIO", rfc: "..." }`
- **Then** response MUST be `400` with message mentioning the incoherent combination.

**SHALL reject duplicate RFC while another is IN_PROGRESS**
- **Given** an existing `registration` with `rfc = "X"` and `status = "IN_PROGRESS"`
- **When** `POST /admin/registration` is called again with the same `rfc`
- **Then** response MUST be `409` with body `{ message, registrationId }` where `registrationId` is the existing draft id.

**SHALL reject when caller is not SUPERADMIN**
- **Given** a valid JWT with `role = AUXILIAR`
- **When** `POST /admin/registration`
- **Then** response MUST be `403`.

---

## 2. `GET /admin/registration` — List drafts

### Description
Lists registration drafts. Hides expired and cancelled rows by default.

### Query params
- `status` (optional) — `IN_PROGRESS | COMPLETED`.
- `includeExpired` (optional, boolean) — defaults `false`.

### Rules
- Default filter MUST be `expires_at > NOW() AND status != 'CANCELLED'`.
- `?includeExpired=true` MUST drop the `expires_at` filter.
- Response MUST be ordered by `updated_at DESC`.

### Response
- `200 OK` →
  ```jsonc
  {
    "items": [
      { "id", "profileType", "userRole", "currentStep", "status",
        "expiresAt", "createdAt", "updatedAt" }
    ],
    "total": N
  }
  ```

### Scenarios

**SHALL hide expired drafts by default**
- **Given** 3 drafts: 2 `IN_PROGRESS` with `expires_at > NOW()`, 1 with `expires_at < NOW()`
- **When** `GET /admin/registration`
- **Then** response MUST be `200` with `items.length === 2`.

**SHALL return expired drafts when includeExpired=true**
- **Given** the same fixture
- **When** `GET /admin/registration?includeExpired=true`
- **Then** `items.length === 3`.

---

## 3. `GET /admin/registration/:id` — Draft detail

### Description
Returns the hydrated draft (registration + nested profile + contacts + vulnerable activity + compliance responsible). Used by the front to resume an `IN_PROGRESS` draft.

### Rules
- If draft not found → `404`.
- If `expires_at < NOW()` → `410 Gone` with `{ message: "Registration draft has expired" }`.
- If `status = CANCELLED` → `404` (behaves as not found for clients).

### Response (`200 OK`)
```jsonc
{
  "id", "profileType", "userRole", "currentStep", "status",
  "expiresAt", "createdAt", "updatedAt",
  "physicalProfile": null | { /* PhysicalPersonProfileEntity + address */ },
  "moralProfile":    null | { /* MoralPersonProfileEntity + address + complianceResponsible */ },
  "contacts":        [ { /* ContactEntity */ } ],
  "vulnerableActivity": null | { /* VulnerableActivityEntity + address */ },
  "userId":          null | "uuid"
}
```

### Scenarios

**SHALL return a PF draft stopped at CONTACT**
- **Given** a PF draft with identification saved and 1 contact saved, no vulnerable activity yet
- **When** `GET /admin/registration/<id>`
- **Then** `200` with `currentStep = "CONTACT"`, `physicalProfile != null`, `contacts.length === 1`, `vulnerableActivity = null`, `moralProfile = null`, `userId = null`.

**SHALL return 410 on expired draft**
- **Given** a draft with `expires_at = NOW() - 1 hour`
- **When** `GET /admin/registration/<id>`
- **Then** response MUST be `410` with `{ message: "Registration draft has expired" }`.

**SHALL return 404 on unknown id**
- **When** `GET /admin/registration/<random-uuid>`
- **Then** `404`.

---

## 4. `POST /admin/registration/:id/identification` — Step 1

### Description
Persists the reporting entity's identification data. DTO shape depends on `registration.profile_type`.

### Preconditions
- Draft MUST exist, not expired, `status = IN_PROGRESS`.
- If `current_step` is beyond `IDENTIFICATION`, this is an overwrite — allowed while `status = IN_PROGRESS`. `current_step` MUST NOT regress.

### Request body
- When `profileType = PERSONA_FISICA` → `PhysicalIdentificationDto`:
  ```jsonc
  {
    "firstName":        "string",  // MUST
    "paternalSurname":  "string",  // MUST
    "maternalSurname":  "string",  // MUST
    "birthDate":        "YYYY-MM-DD",  // MUST
    "rfc":              "string",  // MUST, MUST equal registration.rfc captured in §1
    "curp":             "string",  // MUST
    "nationalityCountry": "string?",
    "birthCountry":       "string?"
  }
  ```
- When `profileType = PERSONA_MORAL` → `MoralIdentificationDto`:
  ```jsonc
  {
    "corporateName":      "string",  // MUST
    "rfc":                "string",  // MUST, MUST equal registration.rfc
    "incorporationDate":  "YYYY-MM-DD?",
    "nationalityCountry": "string?"
  }
  ```

### Rules
- MUST reject if the DTO shape does not match `registration.profile_type` → `409` (`"Draft is <PF|PM>; use the corresponding payload"`).
- MUST validate `dto.rfc === registration.rfc` captured in §1. Mismatch → `400` (`"rfc does not match draft"`).
- MUST INSERT or UPDATE the profile row (idempotent with overwrite keyed by `registration.physical_profile_id` or `moral_profile_id`).
- MUST update `registration` setting `physical_profile_id | moral_profile_id` and (if `current_step` is before `IDENTIFICATION`) advance `current_step = IDENTIFICATION`.

### Response
- `201` on first write; `200` on overwrite. Body: `{ registrationId, currentStep, profileId }`.
- `400 | 401 | 403 | 404 | 409 | 410` per cross-cutting rules.

### Scenarios

**SHALL save PF identification and advance current_step**
- **Given** a PF draft in `IDENTIFICATION` (just created)
- **When** `POST /admin/registration/<id>/identification` with valid PF payload
- **Then** `201` with `currentStep = "IDENTIFICATION"` and `profileId` returned
- **And** `physical_person_profile` has a new row
- **And** `registration.physical_profile_id` points to it.

**SHALL reject PF payload on a PM draft**
- **Given** a PM draft
- **When** `POST /.../identification` with a PF payload (has `firstName`, no `corporateName`)
- **Then** `409` (`"Draft is PM; use the corresponding payload"`).

**SHALL overwrite on re-POST while IN_PROGRESS**
- **Given** a PF draft with identification already saved
- **When** `POST /.../identification` again with a different `firstName`
- **Then** `200` with the updated data
- **And** `physical_person_profile` still has a single row for this draft (UPDATE, not INSERT).

**SHALL reject overwrite after COMPLETED**
- **Given** a draft with `status = COMPLETED`
- **When** `POST /.../identification`
- **Then** `409` (`"Registration already finalized"`).

---

## 5. `POST /admin/registration/:id/contact` — Step 2 (N allowed)

### Description
Appends one contact to the draft. Can be called N times.

### Preconditions
- Draft in `IN_PROGRESS`, not expired.
- `current_step` MUST be `IDENTIFICATION` or later. Otherwise `409`.

### Request body (`ContactDto`)
```jsonc
{
  "countryCode": "string?",
  "phone":       "string?",   // SHOULD be validated by regex at the front; back only rejects empty strings
  "email":       "string",    // MUST @IsEmail
  "cellphone":   "string"     // MUST
}
```

### Rules
- MUST INSERT a new row into `contact` with `registration_id = :id`.
- When `current_step = IDENTIFICATION`, MUST advance `current_step = CONTACT` on the first contact inserted. Subsequent calls do NOT regress or advance.
- MUST NOT overwrite existing contacts — contacts are append-only in this change. (Deletion/edit is deferred.)

### Response
- `201 Created` with `{ contactId, currentStep, totalContacts }`.
- `400 | 401 | 403 | 404 | 409 | 410` per cross-cutting rules.

### Scenarios

**SHALL save first contact and advance**
- **Given** draft in `IDENTIFICATION`
- **When** `POST /.../contact` with valid body
- **Then** `201` with `currentStep = "CONTACT"`, `totalContacts = 1`.

**SHALL allow multiple contacts**
- **Given** a draft with 1 contact
- **When** another valid `POST /.../contact`
- **Then** `201` with `totalContacts = 2`; `currentStep` remains `CONTACT`.

**SHALL reject if identification missing**
- **Given** draft in `IDENTIFICATION` state but with no `physical_profile_id` nor `moral_profile_id` (theoretical — should not happen via API)
- **When** `POST /.../contact`
- **Then** `409` (`"Missing prior step: IDENTIFICATION"`).

**SHALL reject invalid email**
- **When** body has `email: "not-an-email"`
- **Then** `400`.

---

## 6. `POST /admin/registration/:id/vulnerable-activity` — Step 3

### Description
Persists the declared vulnerable activity and its address.

### Preconditions
- Draft `IN_PROGRESS`, not expired.
- `current_step` MUST be `CONTACT` or later, AND at least 1 `contact` row MUST exist.

### Request body (`VulnerableActivityDto`)
```jsonc
{
  "activity":    "TRANSMISION_DERECHOS_REALES_INMUEBLES" | ...,  // MUST @IsEnum(ActividadVulnerable)
  "startDate":   "YYYY-MM-DD?",
  "address": {
    "street":         "string",   // MUST
    "exteriorNumber": "string",   // MUST
    "interiorNumber": "string?",
    "neighborhood":   "string",   // MUST
    "municipality":   "string",   // MUST
    "city":           "string",   // MUST
    "state":          "string",   // MUST
    "postalCode":     "string",   // MUST
    "country":        "string",   // MUST
    "roadType":       "string?",
    "locality":       "string?"
  },
  "activityPerformedAtAddress": "string"   // MUST — free-text description
}
```

### Rules
- MUST INSERT (or UPDATE, on overwrite) `reporting_entity_address` and `vulnerable_activity`.
- MUST link `registration.vulnerable_activity_id` to the row.
- MUST advance `current_step` to `VULNERABLE_ACTIVITY`.
- On overwrite, UPDATE the existing `vulnerable_activity` and its `reporting_entity_address` — never leave orphaned rows.

### Response
- `201` on first write; `200` on overwrite.
- `{ registrationId, currentStep, vulnerableActivityId, addressId }`.

### Scenarios

**SHALL save activity and advance**
- **Given** draft in `CONTACT` with ≥1 contact
- **When** valid `POST /.../vulnerable-activity`
- **Then** `201` with `currentStep = "VULNERABLE_ACTIVITY"`.

**SHALL reject invalid enum**
- **When** body has `activity = "INVENTED"`
- **Then** `400`.

**SHALL reject if no contact saved**
- **Given** draft in `IDENTIFICATION` (no contacts yet)
- **When** `POST /.../vulnerable-activity`
- **Then** `409` (`"Missing prior step: CONTACT"`).

---

## 7. `POST /admin/registration/:id/compliance-responsible` — Step 4 (PM only)

### Description
Persists the compliance officer for a PM reporting entity. This step does NOT exist in the PF flow.

### Preconditions
- Draft MUST be PM (`profile_type = PERSONA_MORAL`).
- `current_step` MUST be `VULNERABLE_ACTIVITY`.
- `registration.moral_profile_id` MUST be set.

### Request body (`ComplianceResponsibleDto`)
```jsonc
{
  "firstName":         "string",  // MUST
  "paternalSurname":   "string",  // MUST
  "maternalSurname":   "string",  // MUST
  "rfc":               "string",  // MUST
  "curp":              "string",  // MUST
  "birthDate":         "YYYY-MM-DD?",
  "nationalityCountry":"string?",
  "designationDate":   "YYYY-MM-DD?"
}
```

### Rules
- MUST reject with `409` if `profile_type = PERSONA_FISICA` (`"This step only applies to PERSONA_MORAL drafts"`).
- MUST INSERT or UPDATE `compliance_responsible`.
- MUST UPDATE `moral_person_profile.compliance_responsible_id` (column is `NOT NULL` — this is where it gets populated).
- MUST advance `current_step = COMPLIANCE_RESPONSIBLE`.

### Response
- `201` / `200` (overwrite) with `{ registrationId, currentStep, complianceResponsibleId }`.

### Scenarios

**SHALL save compliance responsible and advance**
- **Given** PM draft in `VULNERABLE_ACTIVITY`
- **When** valid `POST /.../compliance-responsible`
- **Then** `201` with `currentStep = "COMPLIANCE_RESPONSIBLE"`
- **And** `moral_person_profile.compliance_responsible_id` MUST be set.

**SHALL reject on PF draft**
- **Given** PF draft
- **When** `POST /.../compliance-responsible`
- **Then** `409` (`"This step only applies to PERSONA_MORAL drafts"`).

**SHALL reject if vulnerable-activity not saved**
- **Given** PM draft in `CONTACT`
- **When** `POST /.../compliance-responsible`
- **Then** `409` (`"Missing prior step: VULNERABLE_ACTIVITY"`).

---

## 8. `POST /admin/registration/:id/finalize` — Close draft, create user

### Description
Atomic closure of the draft. Creates the `user`, hashes a generated password, wires the `user` to its profile, flips `status = COMPLETED`, and returns the password in cleartext **once**.

### Preconditions
- Draft `IN_PROGRESS`, not expired.
- For PF: `current_step = VULNERABLE_ACTIVITY`.
- For PM: `current_step = COMPLIANCE_RESPONSIBLE`.
- `contact.length ≥ 1`. The primary contact (first inserted) MUST have a valid email.

### Request body
Empty or optional `FinalizeDto` (reserved for future metadata like notifyByEmail). Implementations MAY accept `{}`.

### Rules
- MUST run the whole body in a single transaction.
- MUST generate a 12+ char random password, hash it with `bcrypt` (cost factor consistent with existing users — look up `packages/domain-auth-users` helper).
- MUST INSERT a new `users` row: `email = contacts[0].email`, `password_hash`, `role = registration.user_role`, `profile_type = registration.profile_type`, `profile_id = registration.physical_profile_id | moral_profile_id`, `active = true`.
- MUST fail with `409` (`"Email already registered"`) if `contacts[0].email` collides with an existing user.
- MUST UPDATE `registration`: `user_id = users.id`, `status = COMPLETED`, `current_step = COMPLETED`.
- MUST return the cleartext password exactly ONCE in the response. It MUST NOT be persisted or exposed by any other endpoint.

### Response
- `201 Created` →
  ```jsonc
  {
    "registrationId": "uuid",
    "userId":         "uuid",
    "email":          "string",
    "tempPassword":   "string",
    "status":         "COMPLETED",
    "currentStep":    "COMPLETED",
    "message":        "Información de perfil actualizada"
  }
  ```
- `409` on missing prior step / email already taken / already finalized.
- `410` on expired draft.

### Scenarios

**SHALL finalize PF draft**
- **Given** PF draft in `VULNERABLE_ACTIVITY` with ≥1 contact
- **When** `POST /.../finalize`
- **Then** `201` with `tempPassword` (cleartext, 12+ chars), `userId`, `email = contacts[0].email`
- **And** `users` has a new row with `role = NOTARIO`, `profile_type = PERSONA_FISICA`, `profile_id = registration.physical_profile_id`
- **And** `registration.status = COMPLETED` and `registration.user_id = users.id`.

**SHALL finalize PM draft**
- **Given** PM draft in `COMPLIANCE_RESPONSIBLE` with ≥1 contact
- **When** `POST /.../finalize`
- **Then** `201` with the same shape, `role = INMOBILIARIA`, `profile_type = PERSONA_MORAL`, `profile_id = registration.moral_profile_id`.

**SHALL reject PM draft stuck at VULNERABLE_ACTIVITY**
- **Given** PM draft in `VULNERABLE_ACTIVITY` (no compliance responsible yet)
- **When** `POST /.../finalize`
- **Then** `409` (`"Missing prior step: COMPLIANCE_RESPONSIBLE"`).

**SHALL reject if no contact saved**
- **Given** draft where first contact was skipped (impossible via state machine, but defensive)
- **When** `POST /.../finalize`
- **Then** `409` (`"At least one contact is required"`).

**SHALL reject when email already exists**
- **Given** draft with `contacts[0].email = "duplicate@x.com"`, and a `users` row with that email already
- **When** `POST /.../finalize`
- **Then** `409` (`"Email already registered"`)
- **And** the draft remains `IN_PROGRESS` (transaction rolled back).

**SHALL reject second finalize call**
- **Given** a draft already `COMPLETED`
- **When** `POST /.../finalize` again
- **Then** `409` (`"Registration already finalized"`). The password is NOT re-emitted — it is gone.

---

## 9. Cross-cutting error matrix

| Case | HTTP |
|---|---|
| Missing/invalid JWT | `401` |
| Valid JWT, role != SUPERADMIN | `403` |
| Draft `:id` not found | `404` |
| DTO validation fails (missing field, wrong type, bad regex, bad email, bad enum) | `400` |
| Incoherent combination `userRole + profileType` (NOTARIO + PM) | `400` |
| RFC of draft does not match DTO rfc on identification | `400` |
| Duplicate RFC in `IN_PROGRESS` on `POST /admin/registration` | `409` |
| Skipping a state-machine step | `409` |
| PM-only endpoint called on PF draft (or vice versa) | `409` |
| `POST` on `status = COMPLETED` draft | `409` |
| `finalize` with email already existing in `users` | `409` |
| Any `GET /admin/registration/:id` or step `POST` on an expired draft | `410` |

Error body: `{ statusCode, message, error, timestamp, path }`.
