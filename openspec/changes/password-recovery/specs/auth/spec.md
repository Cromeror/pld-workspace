# Delta for Auth

> Esta delta agrega un nuevo criterio de invalidación de JWT a la `JwtStrategy`: tokens emitidos antes de `users.password_changed_at` MUST ser rechazados con HTTP 401 + código `password-changed`. No modifica el contrato HTTP de `POST /auth/login`. Refinement Jarvis thread `ffcf3339-7f80-4889-b60f-c221f3c30f1f` iter 4.

## ADDED Requirements

### Requirement: JwtStrategy rechaza tokens emitidos antes de password_changed_at

`JwtStrategy.validate()` (única copia local en `apps/auth-users/src/shared/auth/jwt.strategy.ts` — la app `apps/cross` fue eliminada el 2026-05-02 y el INV-5 original deja de aplicar) MUST comparar el claim `iat` del JWT contra `users.password_changed_at` del usuario referenciado por `sub`. Si `iat * 1000 < user.password_changed_at`, MUST rechazar con HTTP 401 (RC-10).

La response 401 MUST seguir el shape de error uniforme definido en la spec `http-error-formatting` y MUST incluir un código de error identificable como `password-changed` para que el frontend pueda diferenciar este caso de un 401 genérico (token inválido / expirado / firma manipulada).

La verificación MUST ejecutarse en cada request a endpoint protegido — no se mantiene denylist en memoria; el chequeo se basa exclusivamente en la columna `password_changed_at`.

#### INV-1: JWT con iat anterior a password_changed_at retorna 401 con código password-changed

- GIVEN un usuario activo con `users.password_changed_at = T`
- AND un JWT válido emitido en `T - 1 minute` (claim `iat` correspondiente)
- WHEN un cliente envía un request a un endpoint protegido con `JwtAuthGuard` usando ese token
- THEN el servidor SHALL responder HTTP 401
- AND el body SHALL seguir el shape de error uniforme
- AND el body SHALL contener un código identificable como `password-changed` (campo `errorDetails.code`, `code` o equivalente — la implementación define el campo concreto pero el valor SHALL ser `password-changed`)

#### INV-2: JWT con iat posterior a password_changed_at permite acceso

- GIVEN un usuario activo con `users.password_changed_at = T`
- AND un JWT válido emitido en `T + 1 minute`
- WHEN un cliente envía un request a un endpoint protegido con ese token
- THEN el servidor SHALL responder con el recurso solicitado (no 401 por el motivo password-changed)

#### INV-3: JWT con iat exactamente igual a password_changed_at permite acceso

- GIVEN un usuario activo con `users.password_changed_at = T` (en milisegundos)
- AND un JWT con `iat * 1000 = T`
- WHEN un cliente envía un request a un endpoint protegido
- THEN el servidor SHALL responder con el recurso solicitado (la comparación SHALL ser estrictamente `iat * 1000 < T`, no `<=`)

#### INV-4: Usuario con password_changed_at NULL nunca dispara el rechazo

- GIVEN un usuario con `users.password_changed_at = NULL` (caso teórico, post-backfill no debería ocurrir)
- AND un JWT válido con cualquier `iat`
- WHEN un cliente envía un request a un endpoint protegido
- THEN el servidor SHALL NOT rechazar por motivo password-changed
- AND la decisión 401/200 SHALL depender únicamente de los demás chequeos (firma, expiración, soft-delete, activo)

<!-- INV-5 eliminado 2026-05-02 — apps/cross fue removida del monorepo. La única copia de JwtStrategy queda en apps/auth-users (cubierta por INV-1..INV-4). -->
#### INV-5: ~~La verificación aplica en ambas apps~~ — N/A: app `cross` eliminada.

#### INV-6: 401 por firma inválida o token expirado no usa código password-changed

- GIVEN un JWT con firma manipulada o `exp` en el pasado
- WHEN un cliente envía un request a un endpoint protegido
- THEN el servidor SHALL responder HTTP 401
- AND el body NO SHALL contener el código `password-changed` (otros 401 mantienen su código actual / sin código)
