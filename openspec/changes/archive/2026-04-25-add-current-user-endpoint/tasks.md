# Tasks: Add GET /auth/me endpoint and adopt it in the web

## Phase 1 — BE: Migration + Entity [BE]

- [x] 1.1 Crear migración TypeORM `{ts}-add-must-change-password.ts` en `pld-api/packages/persistence/migrations/`. Estrategia: ADD col `DEFAULT false`, UPDATE backfill, ALTER DEFAULT a `true`. Down: `DROP COLUMN must_change_password`.
- [x] 1.2 Añadir campo `@Column({ name: 'must_change_password', type: 'boolean', default: true }) mustChangePassword: boolean` en `pld-api/packages/domain-auth-users/src/entities/user.entity.ts`.
- [ ] 1.3 Ejecutar migración localmente; verificar `must_change_password` presente en `users`; smoke `POST /auth/login` → HTTP 201. [MANUAL — user must run migration via stack-up]

## Phase 2 — BE: UsersPort + Adapter [BE]

- [x] 2.1 Añadir `getUserById(id: string): Promise<CurrentUserDto | null>` y `touchLastLogin(id: string): Promise<void>` a `pld-api/packages/domain-auth-users/src/ports/users.port.ts`.
- [x] 2.2 Implementar `getUserById` en `pld-api/packages/domain-auth-users/src/adapters/users.adapter.ts` usando la función pura `toCurrentUserDto(user: UserEntity)`. Excluir `passwordHash`, `createdAt`, `updatedAt`, `deletedAt`.
- [x] 2.3 Implementar `touchLastLogin` en el mismo adapter: `UPDATE users SET last_login_at = NOW() WHERE id = :id`.
- [x] 2.4 En `UsersAdapter.createUser()` setear `mustChangePassword = true` explícitamente en el insert de la entity.

## Phase 3 — BE: AuthAdapter + AuthController [BE]

- [x] 3.1 En `pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts`, dentro de `login()`, llamar `usersPort.touchLastLogin(userId)` después de `verifyCredentials()` y antes de `issueToken()`. Envolver en try/catch, loguear error, no fallar login.
- [x] 3.2 Añadir handler `@Get('me') @UseGuards(JwtAuthGuard)` en `pld-api/apps/auth-users/src/auth/auth.controller.ts`. Usa `@CurrentUser()`, delega a `usersPort.getUserById(id)`. Si null o `activo === false` → `throw new UnauthorizedException()`.
- [x] 3.3 Agregar `@Header('Cache-Control', 'private, no-store')` al handler de 3.2.
- [ ] 3.4 Smoke: `GET /auth/me` con Bearer válido → 200 + DTO sin campos internos. Sin token → 401. [MANUAL — requires running stack]
- [x] 3.5 Builds: `nx run auth-users:build` y `nx run cross:build` → 0 errores.

## Phase 4 — Web: Tipos + Servicio + Hook [Web]

- [x] 4.1 Crear `pld-web/src/types/CurrentUser.ts` con `CurrentUser` type que coincida con `CurrentUserDto` del BE. Reutilizar `UserRole` enum existente.
- [x] 4.2 Crear `pld-web/src/services/userService.ts` con `getCurrentUser(): Promise<CurrentUser>` → `axios.get('/auth/me').then(r => r.data)`.
- [x] 4.3 Crear `pld-web/src/queries/userQueries.ts` con `useCurrentUser()`: `queryKey: ['currentUser']`, `queryFn: userService.getCurrentUser`, `enabled: useGlobalStore(s => s.isAuthenticated)`, `staleTime: Infinity`, `gcTime: Infinity`.

## Phase 5 — Web: Logout + Interceptor [Web]

- [x] 5.1 Crear `pld-web/src/lib/forceLogout.ts` con `setForceLogout(fn)` + `triggerForceLogout()`. El helper recibe `queryClient` en boot via setter para evitar import circular.
- [x] 5.2 Crear `pld-web/src/hooks/useLogout.ts` que combina `globalStore.logout()` + `queryClient.removeQueries({ queryKey: ['currentUser'] })`.
- [x] 5.3 Actualizar `pld-web/src/config/axios.ts`: interceptor 401 llama `forceLogout` (limpia store + React Query cache) en lugar de solo limpiar localStorage.
- [x] 5.4 Actualizar `pld-web/src/store/globalStore.ts`: eliminar campos de decoded JWT claims; dejar solo `isAuthenticated`, `token`, `login(token)`, `logout()`.

## Phase 6 — Web: Adopción en rutas + LoginForm [Web]

- [x] 6.1 Actualizar `pld-web/src/routes/RoleProtectedRoute.tsx`: usar `useCurrentUser()`. Mientras `isLoading` → `<div>Cargando...</div>`. Si error o rol no en `requiredRoles` → `<Navigate to="/" />`.
- [x] 6.2 Actualizar `pld-web/src/components/auth/LoginForm/index.tsx` con el flow D13: login → `globalStore.login(token)` → `queryClient.fetchQuery(['currentUser'])` → `navigate(getPostLoginRedirect(user.role))`. Si `fetchQuery` lanza: toast de error + `globalStore.logout()` + reset botón.
- [x] 6.3 Reemplazar todas las llamadas directas a `store.logout()` desde componentes por el hook `useLogout()`.

## Phase 7 — Web: Cleanup [Web]

- [x] 7.1 Eliminar `pld-web/src/hooks/useCurrentUserRole.ts`.
- [x] 7.2 Correr `yarn remove jwt-decode` en `pld-web/` y confirmar 0 imports: `grep -rn "jwt-decode\|jwtDecode" pld-web/src` → sin resultados.

## Phase 8 — Verificación final [Both]

- [ ] 8.1 `cd pld-api && nx run auth-users:build` → 0 errores.
- [ ] 8.2 `cd pld-api && nx run cross:build` → 0 errores.
- [x] 8.3 `cd pld-web && yarn tsc -b --noEmit` → 0 errores.
- [x] 8.4 `cd pld-web && yarn build` → 0 errores.
- [ ] 8.5 Smoke manual: login SUPERADMIN → redirige a wizard. Login NOTARIO → redirige a home. Refrescar página con token → autenticado, rol correcto. Logout → cache limpio (DevTools Network no muestra `GET /auth/me` tras logout).
