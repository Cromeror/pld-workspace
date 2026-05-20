# Proposal: Implementar flujos de autenticación end-to-end

**Status**: draft
**Created**: 2026-05-20
**Sub-repos**: pld-api + pld-web

## Why

Los tres flujos de autenticación del producto — [inicio de sesión](../../../docs/flujos/inicio-sesion.md), [post-login](../../../docs/flujos/post-login.md) y [cambio de workspace](../../../docs/flujos/cambio-workspace.md) — ya están documentados como fuente de verdad del sistema. La implementación actual cubre la mayor parte del camino feliz (login, selección de workspace, recuperación de contraseña, toast post-cambio, redirect post-login para `SUPERADMIN` y `WORKSPACE_ADMIN`), pero arrastra siete inconsistencias documentadas que producen comportamiento visible defectuoso:

- El menú de switch ofrece una opción de workspace que el usuario no tiene (BE responde 403, sin feedback al usuario).
- El path real del layout autenticado tiene un espacio inicial (`pld-web/src/layouts/ AuthenticatedLayout.tsx`) — bug menor pero contagia imports.
- Los roles `AUXILIARY` y "desconocido" caen al `DEFAULT_REDIRECT = HOME` en `postLoginRedirect.ts` cuando deberían tener un destino propio (placeholder) o invalidar la sesión y redirigir a una pantalla genérica de error.
- La UI dice "Correo Electrónico o Teléfono" cuando el BE solo acepta email o `nickName`.
- La label del switch hardcodea el toggle `NOTARY ↔ REAL_ESTATE` en lugar de derivarla de las registrations COMPLETED reales del usuario.

Este change cierra el gap entre los flujos documentados y el código, resolviendo las inconsistencias y dejando un baseline de verificación manual antes de avanzar a tests automatizados.

## What Changes

### BE (pld-api)

- **Endpoint nuevo** `GET /auth/me/workspaces` (o equivalente — ver design) que devuelve las registrations `COMPLETED` del usuario autenticado, para que el FE pueda decidir si mostrar el item de switch en el menú sin tener que llamar primero al switch y manejar el 403.

### Web (pld-web)

- **Inconsistencia 1** — Hook nuevo `useAvailableWorkspaces` que consume el endpoint nuevo del BE. `AuthenticatedLayout` deriva `canSwitch` de la lista real y solo muestra el item cuando existe la registration alternativa COMPLETED.
- **Inconsistencia 2** — `useSwitchWorkspaceMutation` gana un handler `onError` que llama `showToast({ severity: "error", ... })` con el `message` del BE; fallback "No se pudo cambiar de workspace".
- **Inconsistencia 3** — Label del switch derivado de la registration alternativa real (no del toggle hardcodeado). Si en el futuro hay un tercer `activityType`, la UI sigue funcionando sin tocar lógica.
- **Inconsistencia 4** — Renombrar `pld-web/src/layouts/ AuthenticatedLayout.tsx` (con espacio inicial) a `AuthenticatedLayout.tsx` y corregir el import en `pld-web/src/routes/index.tsx`.
- **Inconsistencia 5** — `getPostLoginRedirect` (o un guard equivalente) detecta rol no reconocido, invalida la sesión local (`authService.logout()` + `queryClient.clear()`) y navega a `/error` (ruta y página nuevas). La pantalla genérica NO menciona el motivo concreto — solo "Ocurrió un error".
- **Inconsistencia 6** — Pantalla `AuxiliaryPlaceholderPage` nueva en `pld-web/src/pages/auxiliary/`, ruta `/auxiliary` agregada a `RoutesUrl` y al router, mapping `[UserRole.AUXILIARY]: RoutesUrl.AUXILIARY` en `POST_LOGIN_REDIRECT_BY_ROLE`.
- **Inconsistencia 7** — Cambiar el texto "Correo Electrónico o Teléfono" → "Correo Electrónico o Nickname" en `LoginForm/index.tsx` (label del campo email) y en `ForgotPasswordForm/index.tsx` ("Ingresa tu correo o teléfono para continuar" → "Ingresa tu correo electrónico o nickName para continuar").

### Flujos felices

- **Ya implementados — solo verificación manual**: login email/password + check aviso privacidad, selección de workspace con tempToken, recuperación de contraseña (request → enlace → nueva contraseña), toast `?reason=password-changed`, redirect post-login para `SUPERADMIN` y `WORKSPACE_ADMIN`, shell autenticada (sidebar, header, menú con email + logout), switch de workspace (`POST /auth/switch-workspace` + `queryClient.clear()` + redirect).
- **Nuevos**: redirect post-login para `AUXILIARY` (a placeholder) y para rol desconocido (a `/error`).

## Impact

### Capabilities openspec afectadas

| Capability | Delta | Sub-repo |
|---|---|---|
| `auth` | ADDED Requirement: `GET /auth/me/workspaces` | pld-api |
| `pld-web` | ADDED Requirements: hook `useAvailableWorkspaces`, switch deriva label de registrations reales, toast `onError` en switch, rename de layout, redirect rol desconocido invalida sesión, redirect `AUXILIARY` → placeholder, label `nickName` en login y recuperación | pld-web |

### Archivos nuevos vs modificados

Detalle completo en [design.md](./design.md), sección "Archivos afectados".

## Rollback Plan

1. **Web**: revertir el PR FE deja al BE inerte (el endpoint nuevo queda sin consumidor — bajo riesgo).
2. **BE**: revertir el endpoint `GET /auth/me/workspaces` no requiere migration ni cambio de schema; solo borrar el handler del controller, adapter y módulo.
3. **Rename del layout**: si el rename rompe imports en algún punto no detectado, restaurar el archivo con espacio inicial y revertir el import en `routes/index.tsx`.
4. **`/error` y `/auxiliary`**: páginas estáticas — su remoción no requiere acciones de datos. Solo borrar rutas y mapping en `postLoginRedirect.ts`.
5. **Sesiones activas**: ningún cambio invalida JWTs ya emitidos. Un `AUXILIARY` logueado durante el deploy sigue funcionando: el siguiente render del layout llama al nuevo redirect y lo lleva al placeholder.

## Dependencies

- `pld-web` depende del endpoint nuevo del BE para resolver la Inconsistencia 1. El BE puede mergearse primero (no rompe la web actual — el endpoint queda sin consumidor) y luego el FE.
- Sin dependencias externas de UX nuevas — los diseños existentes en `docs/flujos/disenos/` cubren los pasos felices. Las pantallas nuevas (`AuxiliaryPlaceholderPage`, `GenericErrorPage`) son placeholders mínimos sin diseño pendiente.

## Success Criteria

- [ ] `GET /auth/me/workspaces` con JWT válido de `WORKSPACE_ADMIN` retorna lista de registrations COMPLETED del usuario.
- [ ] `GET /auth/me/workspaces` con JWT de `SUPERADMIN` retorna lista vacía o equivalente documentado.
- [ ] El item "Cambiar a Notario / Inmobiliaria" en el menú aparece solo cuando existe la registration alternativa COMPLETED.
- [ ] Al fallar el switch (403 del BE), se muestra toast `severity=error` con el `message` del BE.
- [ ] La label del switch refleja el `activityType` alternativo real de la registration disponible, no un toggle hardcodeado.
- [ ] El archivo `pld-web/src/layouts/AuthenticatedLayout.tsx` existe sin espacio inicial y el import en `routes/index.tsx` apunta al path corregido.
- [ ] Login con rol desconocido invalida la sesión local y redirige a `/error`.
- [ ] Login con `role === AUXILIARY` redirige a `/auxiliary` y muestra la pantalla placeholder.
- [ ] El label del campo email en `LoginForm` dice "Correo Electrónico o Nickname".
- [ ] El subtítulo de `ForgotPasswordForm` dice "Ingresa tu correo electrónico o nickName para continuar".
- [ ] `yarn build` en `pld-web` verde (tsc -b && vite build).
- [ ] Verificación manual de los 3 flujos felices (login + selección workspace + switch + recuperación) sin regresiones.
