# Verify Report: add-current-user-endpoint

**Change**: add-current-user-endpoint
**Date**: 2026-04-25
**Verified by**: sdd-verify (sonnet-4-6)

---

## Completeness

| Metric | Value |
|--------|-------|
| Tasks total | 21 |
| Tasks complete (✅) | 18 |
| Tasks incomplete | 3 |

**Incomplete tasks:**
- `1.3` — Execute migration locally; smoke `POST /auth/login` → HTTP 201. [MANUAL]
- `3.4` — Smoke: `GET /auth/me` with valid Bearer → 200 + DTO. Without token → 401. [MANUAL]
- `8.5` — Smoke manual: login flows, page refresh, logout cache check. [MANUAL]

> Tasks 8.1 and 8.2 (`nx run auth-users:build`, `nx run cross:build`) are marked `[ ]` in tasks.md but were covered via `tsc --noEmit` checks (see Build section). Root-owned `dist/` prevents full Nx build; flagged as WARNING.

---

## Build & Type Check Execution

**BE — `pld-api/packages/domain-auth-users` (tsc --noEmit)**:
```
yarn tsc --noEmit -p packages/domain-auth-users/tsconfig.json
Done in 1.28s.
EXIT_CODE: 0
```
✅ Passed

**BE — `pld-api/apps/auth-users` (tsc --noEmit)**:
```
yarn tsc --noEmit -p apps/auth-users/tsconfig.app.json
Done in 2.04s.
EXIT_CODE: 0
```
✅ Passed

**FE — `pld-web` (tsc -b --noEmit)**:
```
yarn tsc -b --noEmit
Done in 3.00s.
EXIT_CODE: 0
```
✅ Passed

**FE — `pld-web` (yarn build)**:
```
✓ 371 modules transformed.
✓ built in 2.83s.
EXIT_CODE: 0
```
✅ Passed

**Tests**: No new unit tests were written for `GET /auth/me`, `getCurrentUser`, `touchLastLogin`, or `mustChangePassword`. Existing tests (`catalogs.controller.spec.ts`, `registration.service.spec.ts`) do not cover the new code paths. All spec scenarios are UNTESTED at the automated level. This is a significant gap — see Issues section.

**Coverage**: Not configured.

---

## Quick-checks (grep / ls)

| Check | Result |
|-------|--------|
| `grep -rn "jwt-decode\|jwtDecode" pld-web/src` | **0 results** ✅ |
| `grep -n "jwt-decode" pld-web/package.json` | **0 results** ✅ |
| `ls pld-web/src/hooks/useCurrentUserRole.ts` | **File not found** ✅ |
| `useLogout` adopted everywhere (outside LoginForm) | Only defined in `useLogout.ts` itself; no consumer components found (see INV-17 note) |

---

## Spec Compliance Matrix

> Tests column is `(none)` for all scenarios — no automated tests were written for this change. All auto-checkable invariants are verified statically only.

### Auth spec (auth/spec.md)

| INV | Description | Test | Static | Result |
|-----|-------------|------|--------|--------|
| INV-1 | Valid JWT → 200 with public DTO, Cache-Control header, no internal fields | none | Code present (controller + DTO mapping) | ❌ UNTESTED (static: code correct) |
| INV-2 | No Authorization header → 401 | none | `@UseGuards(JwtAuthGuard)` in controller | ❌ UNTESTED (static: correct) |
| INV-3 | Invalid JWT signature → 401 | none | JwtAuthGuard handles signature | ❌ UNTESTED (static: correct) |
| INV-4 | Expired JWT → 401 | none | JwtAuthGuard handles exp | ❌ UNTESTED (static: correct) |
| INV-5 | Soft-deleted user → 401 | none | TypeORM `@DeleteDateColumn` auto-excludes deleted rows from `findOne` (no `withDeleted` flag used); controller throws `UnauthorizedException` if `getCurrentUser` returns null | ❌ UNTESTED (static: correct) |
| INV-6 | Inactive user (activo=false) → 401 | none | Controller: `if (!currentUser \|\| !currentUser.activo) throw new UnauthorizedException()` | ❌ UNTESTED (static: correct) |
| INV-7 | lastLoginAt updated on each successful login | none | `AuthAdapter.login()` calls `usersPort.touchLastLogin(user.id)` before `issueToken`; wrapped in try/catch so failure doesn't block login | ❌ UNTESTED (static: correct) |

### Users spec (users/spec.md)

| INV | Description | Test | Static | Result |
|-----|-------------|------|--------|--------|
| INV-8 | New users have mustChangePassword=true | none | `UsersAdapter.createUser()` sets `mustChangePassword: true` explicitly (line 63) | ❌ UNTESTED (static: correct) |
| INV-9 | Pre-existing rows after migration have mustChangePassword=false | none | Migration: `ADD COLUMN DEFAULT false`, then `UPDATE SET false`, then `ALTER DEFAULT true` — correct 3-step strategy | ❌ UNTESTED (MANUAL — requires running migration) |
| INV-10 | getUserById returns DTO without internal fields | none | **DEVIATED**: `getUserById` still returns `UserEntity` (full entity). Implementation added `getCurrentUser(id)` as a separate port method that returns `CurrentUserDto`. The endpoint correctly calls `getCurrentUser`, not `getUserById`. Spec requirement for `getUserById` signature is unmet, but endpoint behavior is correct. | ⚠️ WARNING — see Issues |

### FE spec (pld-web/spec.md)

| INV | Description | Test | Static | Result |
|-----|-------------|------|--------|--------|
| INV-11 | Boot with token — single fetch, then cached | none | `staleTime: Infinity`, `gcTime: Infinity`, `enabled: isAuthenticated` in `useCurrentUser` | ❌ UNTESTED (static: correct) |
| INV-12 | Boot without token — query disabled | none | `enabled: isAuthenticated` (false when no token) | ❌ UNTESTED (static: correct) |
| INV-13 | Successful login waits /auth/me before navigating | none | `LoginForm`: `loginStore(token)` → `queryClient.fetchQuery(['currentUser'])` → `navigate(getPostLoginRedirect(user.role))` (sequential await) | ❌ UNTESTED (static: correct) |
| INV-14 | Error in /auth/me post-login does not navigate | none | `LoginForm` catch block: `setServerError(...)`, `logoutStore()`, no navigate call. Button re-enables because `isPending` (from `useLoginMutation`) goes false when the outer `try` block finishes | ⚠️ WARNING — see Issues (button state caveat) |
| INV-15 | RoleProtectedRoute shows loading while isLoading | none | `if (isLoading) return <div>Cargando...</div>` | ❌ UNTESTED (static: correct) |
| INV-16 | RoleProtectedRoute redirects to / if role not in requiredRoles | none | `if (isError \|\| !user \|\| !requiredRoles.includes(user.role)) return <Navigate to={RoutesUrl.HOME} />` | ❌ UNTESTED (static: correct) |
| INV-17 | Logout clears currentUser cache | none | `useLogout` hook: `logoutStore()` + `queryClient.removeQueries({ queryKey: ['currentUser'] })`. However, `useLogout` is only defined — no component was found importing it (task 6.3). `forceLogout.ts` correctly calls `removeQueries`. | ⚠️ WARNING — see Issues |
| INV-18 | jwt-decode removed from project | none | `grep` returns 0 results in `src/` and `package.json` | ✅ PASS |
| INV-19 | useCurrentUserRole deleted | none | `ls` returns "No such file" | ✅ PASS |
| INV-20 | Clean build | — | `tsc -b --noEmit` EXIT 0; `yarn build` EXIT 0 | ✅ PASS |

**Compliance summary**: 2/20 invariants PASS (automated); 15 UNTESTED (no tests written); 3 WARNING/DEVIATION. No CRITICAL (all automatable code paths are structurally correct).

---

## Correctness (Static — Structural Evidence)

| Requirement | Status | File:Line Evidence |
|-------------|--------|--------------------|
| `GET /auth/me` handler exists with `@Get('me')` | ✅ Implemented | `auth.controller.ts:28-37` |
| `@UseGuards(JwtAuthGuard)` on the handler | ✅ Implemented | `auth.controller.ts:29` |
| `@Header('Cache-Control', 'private, no-store')` | ✅ Implemented | `auth.controller.ts:30` |
| Returns public DTO (no passwordHash/dates) | ✅ Implemented | `users.adapter.ts:13-28` (`toCurrentUserDto`) |
| `null` or `activo=false` → `UnauthorizedException` | ✅ Implemented | `auth.controller.ts:33-35` |
| `touchLastLogin` called before `issueToken` in login | ✅ Implemented | `auth.adapter.ts:22-28` |
| `mustChangePassword` column in `UserEntity` | ✅ Implemented | `user.entity.ts:55-56` |
| Migration: ADD DEFAULT false → backfill → ALTER DEFAULT true | ✅ Implemented | `20260425000000-add-must-change-password.sql` |
| Migration down: DROP COLUMN | ✅ Implemented | `.down.sql` |
| `createUser` sets `mustChangePassword: true` explicitly | ✅ Implemented | `users.adapter.ts:63` |
| `CurrentUserDto` interface defined in port | ✅ Implemented | `users.port.ts:17-31` |
| `getCurrentUser(id)` in port and adapter | ✅ Implemented | `users.port.ts:40`, `users.adapter.ts:83-87` |
| `touchLastLogin(id)` in port and adapter | ✅ Implemented | `users.port.ts:42`, `users.adapter.ts:89-91` |
| `CurrentUser` FE type matches BE DTO shape | ✅ Implemented | `CurrentUser.ts:1-16` |
| `userService.getCurrentUser` → `GET /auth/me` | ✅ Implemented | `userService.ts:4-5` |
| `useCurrentUser` query with correct config | ✅ Implemented | `userQueries.ts:7-17` |
| `forceLogout` cleans localStorage + store + RQ cache | ✅ Implemented | `forceLogout.ts:6-13` |
| 401 interceptor calls `forceLogout` | ✅ Implemented | `axios.ts:31-34` |
| `globalStore` has no decoded JWT claims | ✅ Implemented | `globalStore.ts` — only `isAuthenticated`, `isAuthReady`, `token` (via localStorage) |
| `useCurrentUser` in `RoleProtectedRoute` | ✅ Implemented | `RoleProtectedRoute.tsx:15` |
| LoginForm flow: login → fetchQuery → navigate | ✅ Implemented | `LoginForm/index.tsx:66-79` |
| jwt-decode removed | ✅ Implemented | grep: 0 results |
| `useCurrentUserRole.ts` deleted | ✅ Implemented | file not found |

---

## Coherence (Design)

| Decision | Followed? | Notes |
|----------|-----------|-------|
| D1 — Route `/auth/me` in AuthController | ✅ Yes | `@Get('me')` in `auth.controller.ts` |
| D2 — Reuse `JwtAuthGuard` + `@CurrentUser()` decorator | ✅ Yes | `auth.controller.ts:29,31` |
| D3 — DB lookup + null/activo=false → 401 | ⚠️ Deviated | Design says "handler calls `usersPort.getUserById`" but implementation uses `usersPort.getCurrentUser`. Behavior is identical; method name differs from design text. |
| D4 — Pure `toCurrentUserDto` in adapter | ✅ Yes | `users.adapter.ts:13-28` |
| D5 — Always go to DB | ✅ Yes | `getCurrentUser` always queries `repo.findOne` |
| D6 — `Cache-Control: private, no-store` | ✅ Yes | `auth.controller.ts:30` |
| D7 — Migration: ADD DEFAULT false → backfill → ALTER DEFAULT true | ✅ Yes | Migration file matches strategy exactly |
| D8 — `createUser` sets `mustChangePassword = true` | ✅ Yes | `users.adapter.ts:63` |
| D9/D10 — `touchLastLogin` fire-and-forget with try/catch | ✅ Yes | `auth.adapter.ts:22-28` (comment cites D9/D10) |
| D16 — `forceLogout.ts` separate module (no circular imports) | ⚠️ Deviated | Design specified setter injection (`setForceLogout(fn)`) to avoid circular deps. Implementation imports `queryClient` directly from `queryClient.ts` — no circular dep in practice because `queryClient.ts` has no upstream imports. Simpler and functionally equivalent. |
| FE: `useCurrentUser` adopted in `RoleProtectedRoute` | ✅ Yes | `RoleProtectedRoute.tsx:15` |
| FE: LoginForm D13 flow | ✅ Yes | `LoginForm/index.tsx:66-79` |

---

## Issues Found

### CRITICAL
None.

---

### WARNING

**W1 — INV-10: `getUserById` port signature still returns `UserEntity`**
- Spec (users/spec.md): "El método `getUserById(id)` de `UsersPort` MUST retornar el DTO público `CurrentUserDto`."
- Implementation: `getUserById` still returns `Promise<UserEntity | null>` (unchanged). A new method `getCurrentUser` was added to return `CurrentUserDto`.
- Impact: The spec requirement for `getUserById` is unmet at the port interface level. However, the endpoint behavior is correct because `auth.controller.ts` calls `getCurrentUser`, not `getUserById`. The port comment on line 37 calls out the backward-compat reason.
- Fix options: (a) change `getUserById` return type to `CurrentUserDto` and update all callers, or (b) update the spec to reflect the two-method design.
- Files: `pld-api/packages/domain-auth-users/src/ports/users.port.ts:38`

**W2 — INV-14: Button re-enable state relies on mutation's `isPending`, not fetchQuery's loading**
- After `mutateAsync` resolves (HTTP 201), `isPending` from `useLoginMutation` goes to `false`. If `queryClient.fetchQuery` then throws (INV-14 scenario), the button is already re-enabled at that point — which is correct behavior.
- However: there is a race window between `loginStore(token)` and the catch block where `isPending` is `false` but `fetchQuery` is still in-flight. During that window the button could be clicked again. Minor UX concern, not a hard spec violation.
- No code change required unless explicit debounce/guard is desired.

**W3 — INV-17: `useLogout` hook is defined but no component appears to import it (task 6.3 incomplete)**
- Task 6.3: "Reemplazar todas las llamadas directas a `store.logout()` desde componentes por el hook `useLogout()`."
- `grep` found no component files importing `useLogout` (only its own file). The `forceLogout.ts` module handles 401 correctly. `LoginForm` calls `logoutStore()` directly from `useGlobalStore` in the fetchQuery error path.
- Impact: if there are logout buttons elsewhere (not found in the scan), they may bypass the React Query cache invalidation. At the current codebase scope, no such buttons were found — so functionally there is no cache leak. Still, task 6.3 is marked complete but appears partially done.
- Files: `pld-web/src/hooks/useLogout.ts` (defined but not imported anywhere outside its own file)

**W4 — No unit tests written for any new code path**
- 15 of 18 automatable invariants are UNTESTED (no test files added for `auth.controller GET /me`, `UsersAdapter.getCurrentUser`, `UsersAdapter.touchLastLogin`, `forceLogout`, `useCurrentUser`, `RoleProtectedRoute`, `LoginForm` new flow).
- The SKILL.md contract requires: "A spec scenario is only COMPLIANT when a test that covers it has PASSED." By this strict definition, the 15 UNTESTED scenarios are non-compliant.
- This does not block shipping the feature, but should be addressed before the feature reaches production.

**W5 — BE `nx run auth-users:build` and `nx run cross:build` not verified (root-owned dist/)**
- Tasks 8.1 and 8.2 are marked `[ ]` in tasks.md. tsc --noEmit passes (0 errors), but the full Nx build cannot run because `dist/apps/auth-users/` is root-owned (Docker leftover). Static type check gives high confidence, but bundle/emit was not verified.
- Mitigation: run `sudo chown -R $USER dist/` in `pld-api/` then `nx run auth-users:build` to fully clear this.

---

### SUGGESTION

**S1 — `authService.logout()` in `authService.ts` only removes the token from localStorage without touching React Query cache. It is exported but no call site was found — consider removing it to avoid confusion with `useLogout`.**

**S2 — `forceLogout.ts` redirects via `window.location.href` (hard reload). Consider using React Router's `navigate` for SPA-native redirect, though the hard reload is safe and clears React state as a side effect.**

**S3 — `touchLastLogin` uses TypeORM `repo.update(id, { lastLoginAt: new Date() })` which issues a JavaScript `new Date()` rather than a DB-side `NOW()`. The spec says "last_login_at = NOW()". For sub-second precision and timezone correctness prefer a raw query: `repo.query('UPDATE users SET last_login_at = NOW() WHERE id = ?', [id])`. Low priority since the difference is milliseconds.**

---

## Open Items (Manual Smoke Required)

| Item | INV | Condition |
|------|-----|-----------|
| Run migration; verify `must_change_password` column exists | INV-9 | Migration not yet applied to local DB |
| `GET /auth/me` with valid Bearer → HTTP 200 + correct DTO fields | INV-1 | Requires stack running + migration applied |
| `GET /auth/me` without token → HTTP 401 | INV-2 | Requires stack running |
| `POST /auth/login` → DB `last_login_at` updated | INV-7 | Requires stack running |
| Create user via POST endpoint → `must_change_password = true` in DB | INV-8 | Requires stack running + migration applied |
| Login SUPERADMIN → redirect to wizard. Login NOTARIO → redirect to home | INV-13/16 | Browser smoke |
| Page refresh with valid token → stays authenticated | INV-11 | Browser smoke |
| Logout → Network tab shows no `GET /auth/me` after logout | INV-17 | Browser smoke |

---

## Build Evidence Summary

| Command | Exit Code | Status |
|---------|-----------|--------|
| `grep jwt-decode pld-web/src/` | 1 (no matches) | ✅ |
| `ls pld-web/src/hooks/useCurrentUserRole.ts` | 2 (not found) | ✅ |
| `tsc --noEmit -p domain-auth-users/tsconfig.json` | 0 | ✅ |
| `tsc --noEmit -p apps/auth-users/tsconfig.app.json` | 0 | ✅ |
| `pld-web: yarn tsc -b --noEmit` | 0 | ✅ |
| `pld-web: yarn build` | 0 (371 modules) | ✅ |
| `nx run auth-users:build` | NOT RUN (root-owned dist/) | ⚠️ |
| `nx run cross:build` | NOT RUN (root-owned dist/) | ⚠️ |

---

## Verdict

**PASS WITH WARNINGS** (status: `ready-with-manual`)

All automatable static checks pass. Build and type checks are clean across BE and FE. No CRITICAL issues. Five WARNINGs: the most significant are the missing unit tests (W4) and the `getUserById` spec deviation (W1). The three remaining incomplete tasks are all MANUAL smoke tests requiring the stack to be running with the migration applied.

The change is structurally sound and ready for manual smoke testing. Recommended before archive:
1. Address W1 (align spec or port signature).
2. Address W3 (confirm `useLogout` is wired to any actual logout UI buttons, or acknowledge they don't exist yet).
3. Run manual smoke items above.
4. Consider adding at least one integration/unit test for `GET /auth/me` to close the UNTESTED gap.
