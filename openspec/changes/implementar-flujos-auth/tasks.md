# Tasks: Implementar flujos de autenticación end-to-end

> Refines: [proposal.md](./proposal.md), [design.md](./design.md), [specs/auth/spec.md](./specs/auth/spec.md), [specs/pld-web/spec.md](./specs/pld-web/spec.md).
>
> Marcas: [BE] = pld-api, [Web] = pld-web, [Cross] = ambos sub-repos.
> Sin tests automatizados — verificación manual al final de cada fase.

## Phase 1: Infrastructure — BE expone `GET /auth/me/workspaces`

- [x] 1.1 [BE] En `pld-api/packages/domain-auth-users/src/ports/auth.port.ts` agregar la firma `getCompletedWorkspaces(userId: string): Promise<Array<{ activityType: ActivityType; workspaceId: string }>>` al `AuthPort` interface.
- [x] 1.2 [BE] En `pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts` implementar `getCompletedWorkspaces(userId)` que delega a la query existente `queryCompletedRegistrations(userId)` (hoy privada) — convertir a método público o exponer un wrapper público con el mapping al shape `{ activityType, workspaceId }`. NO modificar el SQL.
- [x] 1.3 [BE] Verificar barrel de `@pld-api/domain-auth-users`: `ActivityType` y el tipo del response ya deberían estar exportados. Si no, agregarlos.
- [x] 1.4 [BE] En `pld-api/apps/auth-users/src/auth/auth.controller.ts` agregar el handler `@Get('me/workspaces') @UseGuards(JwtAuthGuard) @Header('Cache-Control', 'private, no-store')`. Extrae `userId` de `req.user.id`, llama `authPort.getCompletedWorkspaces(userId)`, retorna `{ workspaces }`.
- [x] 1.5 [BE] `pnpm exec nx run auth-users:build` debe pasar.

## Phase 2: Implementation — pld-web — rename del layout (preparatorio)

- [x] 2.1 [Web] `git mv "pld-web/src/layouts/ AuthenticatedLayout.tsx" pld-web/src/layouts/AuthenticatedLayout.tsx` (con `git mv` para preservar history). _Nota: archivo era untracked, se usó `mv`._
- [x] 2.2 [Web] En `pld-web/src/routes/index.tsx` actualizar el import: `import AuthenticatedLayout from "@/layouts/AuthenticatedLayout"` (sin espacio).
- [x] 2.3 [Web] `grep -rn "layouts/ AuthenticatedLayout" pld-web/src` debe retornar cero matches. Repetir con variantes (` AuthenticatedLayout`, `\" AuthenticatedLayout\"`).
- [x] 2.4 [Web] `yarn tsc -b --noEmit` en `pld-web` debe pasar.

## Phase 3: Implementation — pld-web — service + hook `useAvailableWorkspaces`

- [x] 3.1 [Web] En `pld-web/src/types/auth.ts` agregar `AvailableWorkspace` y `AvailableWorkspacesResponse` (mirror del response BE — ver design.md "Contratos API ↔ Web").
- [x] 3.2 [Web] En `pld-web/src/services/userService.ts` (o crear handler en `authService.ts` si encaja mejor — decisión documentada en design.md "Open Questions") agregar `getAvailableWorkspaces(): Promise<AvailableWorkspacesResponse>` que llama `GET /auth/me/workspaces` vía `axiosInstance`. _Decisión: vive en userService._
- [x] 3.3 [Web] En `pld-web/src/queries/authQueries.ts` (o `userQueries.ts`) agregar `useAvailableWorkspaces`. La query debe:
  - `queryKey: ['availableWorkspaces']`
  - `queryFn: getAvailableWorkspaces`
  - `enabled: authService.isAuthenticated()` (no llamar si no hay token)
  - Retornar tipos correctos.
- [x] 3.4 [Web] Verificar que `queryClient.clear()` (ya invocado por `useSwitchWorkspaceMutation.onSuccess`) limpia también esta query. No requiere código extra. _Confirmado: `clear()` borra todas las queries._
- [x] 3.5 [Web] Verificar que `useLogout` (en `pld-web/src/hooks/useLogout`) también invalida o resetea el cache de React Query. _Ajuste: agregada `removeQueries({ queryKey: ['availableWorkspaces'] })`._

## Phase 4: Implementation — pld-web — `AuthenticatedLayout` consume el hook

- [x] 4.1 [Web] En `pld-web/src/layouts/AuthenticatedLayout.tsx` reemplazar el cálculo actual de `alternativeActivityType` (toggle hardcodeado) por la derivación desde `useAvailableWorkspaces`:
  - `const { data, isLoading: isLoadingWorkspaces } = useAvailableWorkspaces();`
  - `const alternativeWorkspace = data?.workspaces.find((w) => w.activityType !== currentActivityType);`
  - `const alternativeActivityType = alternativeWorkspace?.activityType;`
- [x] 4.2 [Web] Recalcular `canSwitch` como `role === UserRole.WORKSPACE_ADMIN && !!currentActivityType && !!alternativeActivityType && !isLoadingWorkspaces`.
- [x] 4.3 [Web] Mantener la tabla `ACTIVITY_LABELS` como source del display string en español. El label sigue siendo `Cambiar a ${ACTIVITY_LABELS[alternativeActivityType]}`.
- [x] 4.4 [Web] `yarn tsc -b --noEmit` debe pasar.

## Phase 5: Implementation — pld-web — toast onError en switch

- [x] 5.1 [Web] En `pld-web/src/queries/authQueries.ts`, dentro de `useSwitchWorkspaceMutation`, agregar `onError` que:
  - Extrae el `message` del error (Axios → `error.response?.data?.message` o equivalente; fallback al `error.message`).
  - Llama `showToast({ id: "switch-workspace-error", severity: "error", summary: "Error al cambiar de workspace", detail: <message> | "No se pudo cambiar de workspace" })`.
- [x] 5.2 [Web] Verificar que en `AuthenticatedLayout` el `onError` local (que solo hace `setIsSwitching(false)`) sigue funcionando — no remover; los dos handlers compondrán naturalmente.
- [x] 5.3 [Web] Importar `showToast` en `authQueries.ts` desde `@/lib/toast`.

## Phase 6: Implementation — pld-web — manejo de rol desconocido

- [x] 6.1 [Web] En `pld-web/src/config/postLoginRedirect.ts` agregar un helper:
  - `export const isKnownRole = (role: UserRole | string | null | undefined): role is UserRole => role === UserRole.SUPERADMIN || role === UserRole.WORKSPACE_ADMIN || role === UserRole.AUXILIARY;`
- [x] 6.2 [Web] Crear `pld-web/src/pages/error/GenericErrorPage.tsx` — componente mínimo que renderiza:
  - Logo + título "Ocurrió un error" + subtítulo "Por favor, intenta más tarde" + botón "Volver al inicio de sesión" que navega a `/login`.
  - NO mencionar "rol desconocido" ni términos técnicos.
- [x] 6.3 [Web] En `pld-web/src/routes/routes-urls.ts` agregar `ERROR: "/error"`.
- [x] 6.4 [Web] En `pld-web/src/routes/index.tsx` agregar la ruta `/error` (fuera de `AuthenticatedLayout`, sin guards).
- [x] 6.5 [Web] En `pld-web/src/components/organisms/auth/LoginForm/index.tsx`, tras el `await queryClient.fetchQuery(...)` que carga el `currentUser`, agregar:
  - `if (!isKnownRole(user.role)) { authService.logout(); queryClient.clear(); await navigate(RoutesUrl.ERROR); return; }`
  - Mantener el flujo actual (`navigate(getPostLoginRedirect(user.role))`) cuando el rol sí es conocido.

## Phase 7: Implementation — pld-web — placeholder AUXILIARY

- [x] 7.1 [Web] Crear `pld-web/src/pages/auxiliary/AuxiliaryPlaceholderPage.tsx` — componente mínimo. Texto: "Bienvenido al panel de auxiliar" o equivalente. Sin lógica de negocio.
- [x] 7.2 [Web] En `pld-web/src/routes/routes-urls.ts` agregar `AUXILIARY: "/auxiliary"`.
- [x] 7.3 [Web] En `pld-web/src/routes/index.tsx` agregar la ruta `/auxiliary` como hijo de `AuthenticatedLayout`, protegida por `RoleProtectedRoute` con `requiredRoles: [UserRole.AUXILIARY]`, con `withSuspense(AuxiliaryPlaceholderPage)`.
- [x] 7.4 [Web] En `pld-web/src/config/postLoginRedirect.ts` agregar `[UserRole.AUXILIARY]: RoutesUrl.AUXILIARY` al objeto `POST_LOGIN_REDIRECT_BY_ROLE`.

## Phase 8: Implementation — pld-web — labels Nickname

- [x] 8.1 [Web] En `pld-web/src/components/organisms/auth/LoginForm/index.tsx` modificar el label del campo email:
  - De: `"Correo electrónico"` → A: `"Correo Electrónico o Nickname"` (texto exacto a decidir, debe coincidir con el flujo y casing del diseño).
- [x] 8.2 [Web] En `pld-web/src/components/organisms/auth/ForgotPasswordForm/index.tsx` modificar el subtítulo:
  - De: `"Ingresa tu correo o teléfono para continuar"` → A: `"Ingresa tu correo electrónico o nickName para continuar"`.
- [x] 8.3 [Web] `grep -rn "Teléfono\|teléfono\|Telefono\|telefono" pld-web/src/components/organisms/auth/ pld-web/src/pages/auth/` no debe retornar matches relacionados al campo de identificador (cualquier match restante debe ser placeholder de un campo de teléfono real, no del identificador de login). _Verificado: cero matches._

## Phase 9: Verification — manual smoke + build

- [x] 9.1 [Cross] Levantar stack (`docker-compose` BE + `yarn dev` Web) y aplicar migraciones si aplica. _BE up (mysql healthy, auth-users running 9001), Web up Vite 4200._
- [x] 9.2 [BE] `curl -X GET http://localhost:9001/pld-api/auth-users/auth/me/workspaces -H "Authorization: Bearer <jwt-workspace-admin>"` → 200 con `{ workspaces: [...] }`. Repetir con SUPERADMIN → `{ workspaces: [] }`. Repetir sin token → 401.
  - Sin token → **401** ✅
  - SUPERADMIN → **200 `{"workspaces":[]}`** con `Cache-Control: private, no-store` ✅
  - WORKSPACE_ADMIN (notario.test@example.mx tras select-profile NOTARY) → **200** con `{workspaces:[{NOTARY, ...}, {REAL_ESTATE, ...}]}` ✅
- [ ] 9.3 [Web] Smoke login feliz `WORKSPACE_ADMIN` con dos workspaces COMPLETED:
  - Llega a select-profile, elige uno, queda en `/register`.
  - Abre menú → ve "Cambiar a <alternativo>".
  - Click → switch exitoso → vuelve a `/register` con cache limpia.
  - _PENDIENTE — smoke browser manual. Datos: notario.test@example.mx / Notario123!_
- [ ] 9.4 [Web] Smoke switch con error: forzar 403 (manualmente borrando la registration alternativa en BD o mockeando el response) → toast `severity=error` visible con el detail del BE. _PENDIENTE — smoke browser manual._
- [ ] 9.5 [Web] Smoke login `WORKSPACE_ADMIN` con UNA sola registration COMPLETED:
  - Login directo (sin select-profile).
  - Abre menú → NO debe aparecer el item de switch.
  - _PENDIENTE — smoke browser manual (requiere usuario con 1 sola registration)._
- [ ] 9.6 [Web] Smoke login `SUPERADMIN`:
  - Login directo, queda en `/admin/reporting-entity/register`.
  - Abre menú → NO ve item de switch (regla independiente del hook, pero verificar).
  - _PENDIENTE — smoke browser manual. Datos: admin@pld.com / Admin123!_
- [ ] 9.7 [Web] Smoke login `AUXILIARY`:
  - Login directo → llega a `/auxiliary` y ve `AuxiliaryPlaceholderPage`.
  - Otro rol intenta navegar manualmente a `/auxiliary` → redirige a `/`.
  - _PENDIENTE — smoke browser manual (requiere usuario AUXILIARY)._
- [ ] 9.8 [Web] Smoke rol desconocido: manipular el response del BE (proxy/mock) para que `GET /auth/me` retorne `role: "FOO"`:
  - Login termina invalidando el token (verificar localStorage vacío) y aterriza en `/error` con `GenericErrorPage`.
  - El texto visible NO menciona "rol desconocido".
  - _PENDIENTE — smoke browser manual (requiere mock/proxy)._
- [ ] 9.9 [Web] Smoke recuperación de contraseña (existing happy path):
  - Click "¿Olvidaste tu Contraseña?" → `/forgot-password`. Verificar que el subtítulo dice "correo electrónico o nickName" (no "teléfono").
  - Submit con email → email-sent universal.
  - Abrir link del mail (stub) → recover-password con token → submit nueva password → success → redirige a `/login?reason=password-changed` → toast info visible.
  - _PENDIENTE — smoke browser manual._
- [ ] 9.10 [Web] Smoke labels nickName:
  - Página de login muestra "Correo Electrónico o Nickname" en lugar de "Teléfono".
  - _PENDIENTE — smoke browser manual (visible en http://localhost:4200/login)._
- [x] 9.11 [Cross] Confirmar rename del layout:
  - `ls pld-web/src/layouts/` no muestra el archivo con espacio inicial.
  - El import en `routes/index.tsx` está sin espacio.
  - _Verificado: solo existe `AuthenticatedLayout.tsx` sin espacio. Import correcto. Cero referencias residuales con espacio._
- [x] 9.12 [Web] `yarn tsc -b --noEmit && yarn build` en `pld-web` debe terminar con código 0.
  - _tsc ✅. vite build ✅. Chunks nuevos `AuxiliaryPlaceholderPage-*.js` y `GenericErrorPage-*.js` presentes en dist/._
- [x] 9.13 [BE] `pnpm exec nx run auth-users:build` debe terminar verde.
  - _Build verde, webpack compiled successfully._

## Phase 10: Cleanup

- [x] 10.1 [Web] Verificar que no hay imports residuales del layout con espacio inicial (re-grep). _Cero matches con `'" AuthenticatedLayout"'`, `'/ AuthenticatedLayout'`, `'layouts/ AuthenticatedLayout'`._
- [x] 10.2 [Web] Verificar que `DEFAULT_REDIRECT = HOME` sigue siendo el fallback técnico en `postLoginRedirect.ts`, pero ya NO se debería alcanzar para roles conocidos (todos tienen mapping explícito). _Confirmado: los 3 roles tienen mapping (`SUPERADMIN` → REPORTING_ENTITY_REGISTER, `WORKSPACE_ADMIN` → REGISTER, `AUXILIARY` → AUXILIARY). `DEFAULT_REDIRECT` solo se usa si `role` es null/undefined (caso filtrado por `isKnownRole` en LoginForm)._
- [x] 10.3 [Cross] Confirmar que ningún test automatizado fue introducido en este change (regla del proposal: solo verificación manual). _Cero archivos `.test.*` ni `.spec.*` nuevos en `git status` de pld-api ni pld-web._
