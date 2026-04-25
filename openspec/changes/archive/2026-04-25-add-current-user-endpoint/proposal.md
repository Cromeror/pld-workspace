# Proposal: Add `GET /auth/me` endpoint and adopt it in the web

## Intent

The BE must own user identity. During the wizard work, `pld-web` was wired to decode the JWT client-side via `jwt-decode` to read role/profile — explicitly against the user's intent. We need a backend-owned `GET /auth/me` and the FE must consume it instead of decoding tokens locally.

## Scope

### In Scope
- BE: `GET /auth/me` under existing `AuthController` returning the agreed shape (decision D-B).
- BE: new `users.must_change_password` column (migration: add `BOOLEAN NOT NULL DEFAULT true`, backfill existing rows to `false`; new users via `POST /system-users/*` default to `true`).
- BE: bump `users.last_login_at = NOW()` inside `AuthAdapter.login()` after credential check, before token issuance.
- Web: `useCurrentUser` React Query hook (`userService` + `userQueries`) and adoption in `RoleProtectedRoute` and `LoginForm` (login waits for `/auth/me` before navigating).
- Web: remove `jwt-decode` dependency and delete `useCurrentUserRole`.

### Out of Scope
- Endpoint to actually change the password (follow-up).
- Refresh-token / session rotation changes.
- Adding `userType` field — `role` already encodes it.

## Approach

BE: extend `UsersPort` with a `getById(userId)` returning the public DTO; `AuthController.me()` reads `req.user.sub` from `JwtAuthGuard` and delegates to the adapter. New TypeORM column on `User` entity + SQL migration. `AuthAdapter.login` updates `lastLoginAt` before signing the JWT.

Web: `userService.getCurrentUser()` calls the new endpoint; `useCurrentUser` caches it under a stable React Query key and is used by `RoleProtectedRoute` and `LoginForm`. `globalStore` stops holding decoded JWT claims; `jwt-decode` is removed.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-api/apps/auth-users/src/auth/auth.controller.ts` | Modified | Add `GET /auth/me` (JWT guarded). |
| `pld-api/packages/domain-auth-users/src/entities/user.entity.ts` | Modified | Add `mustChangePassword` column. |
| `pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts` | Modified | Bump `lastLoginAt`; expose `getCurrentUser` mapping. |
| `pld-api/packages/domain-auth-users/src/ports/users.port.ts` (+ adapter) | Modified | Add `getById` returning public DTO. |
| `pld-api/packages/persistence/migrations/` | New | Migration adding `must_change_password` + backfill. |
| `pld-web/src/services/userService.ts` | New | HTTP call to `GET /auth/me`. |
| `pld-web/src/queries/userQueries.ts` | New | `useCurrentUser` React Query hook. |
| `pld-web/src/routes/RoleProtectedRoute.tsx` | Modified | Read role from `useCurrentUser`. |
| `pld-web/src/components/auth/LoginForm/index.tsx` | Modified | Await `/auth/me` before navigate; loading state. |
| `pld-web/src/store/globalStore.ts` | Modified | Drop JWT-decoded claims. |
| `pld-web/src/hooks/useCurrentUserRole.ts` | Removed | Replaced by `useCurrentUser`. |
| `pld-web/package.json` | Modified | Remove `jwt-decode` dep. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Migration default conflicts with existing rows | Med | Two-step: add with default `true`, then `UPDATE users SET must_change_password = false` for pre-existing rows in same migration. |
| `LoginForm` blocks indefinitely if `/auth/me` errors | Med | Surface error toast and reset button state; do not navigate on failure. |
| Stale React Query cache after logout | Low | Invalidate `currentUser` key on logout. |
| Login HTTP 201 contract regression from `lastLoginAt` write | Low | Update happens after credential verification; wrap in same transaction; smoke `POST /auth/login` after each phase. |

## Rollback Plan

1. Revert FE commits → `LoginForm` and `RoleProtectedRoute` go back to JWT decoding (re-add `jwt-decode`).
2. Revert BE controller commit → `GET /auth/me` removed; FE gracefully handles 404 by falling back to logout.
3. Roll back migration with the down script (`DROP COLUMN must_change_password`).
4. `lastLoginAt` write is non-destructive — no rollback needed beyond reverting the adapter change.

## Dependencies

- Existing `JwtAuthGuard` and `JwtStrategy` in `apps/auth-users`.
- TypeORM migration runner already wired in `pld-api/packages/persistence`.

## Success Criteria

- [ ] `GET /auth/me` returns 200 with the agreed shape for a valid JWT, 401 otherwise.
- [ ] `must_change_password` column exists; existing users = `false`, new users created via `POST /system-users/*` = `true`.
- [ ] `lastLoginAt` is updated on every successful `POST /auth/login` (still HTTP 201 + JWT).
- [ ] `pld-web` no longer depends on `jwt-decode`; `useCurrentUserRole` deleted.
- [ ] `LoginForm` only navigates after `/auth/me` resolves; button shows loading until role is known.
- [ ] `nx run cross:build` and `yarn build` (web) succeed.
