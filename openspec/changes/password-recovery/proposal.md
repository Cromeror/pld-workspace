# Proposal: Password recovery end-to-end

**Status**: draft
**Created**: 2026-05-02
**Sub-repos**: pld-api + pld-web

## Intent

Implementar el flujo completo de recuperación de contraseña por enlace mágico ("¿Olvidaste tu contraseña?") tal como aparece en el subflujo de [docs/FLUJO_LOGIN.md](../../../docs/FLUJO_LOGIN.md). Hoy un usuario que pierde su password queda bloqueado y depende de que el SUPERADMIN lo regenere manualmente desde la base. Esta historia habilita el camino self-service: el usuario solicita el enlace por email, abre el link recibido, ingresa una nueva contraseña y vuelve a `/login`.

Cubre **frontend** (4 vistas en `pld-web` + cableado del link en login + límite client-side de reintentos), **backend** (3 endpoints REST públicos en `pld-api/apps/auth-users` + tabla nueva + columna nueva en `users` + JWT invalidation por `password_changed_at` + throttler + cron de cleanup), y un **refactor preparatorio** para extraer `CredentialsMailService` antes de tocar el módulo nuevo. Consume el package puro `@pld-api/mail` ya implementado y verificado e2e — no extrae nada de mail aquí.

Refinement cerrado en Jarvis (thread `ffcf3339-7f80-4889-b60f-c221f3c30f1f`, iter 4 finalizada).

## Scope

### In Scope

#### pld-api (auth-users + persistence + domain-auth-users)

- **Refactor preparatorio (commit aparte)**: extraer `CredentialsMailService` en `apps/auth-users/src/mail/` consumiendo `@pld-api/mail`. Migrar `AuxiliariesService` y `RegistrationService` a inyectarlo en lugar de usar el `MailGatewayClient` directo. Eliminar los `dispatchCredentialsEmail` privados duplicados.
- **Módulo nuevo `apps/auth-users/src/password-recovery/`** siguiendo el patrón controller → service → adapter de `auxiliaries/` y `admin/registration/`:
  - `password-recovery.controller.ts` — 3 endpoints públicos (sin JWT), con decorators de throttler.
  - `password-recovery.service.ts` — request / verify / confirm + cron diario de cleanup.
  - `password-recovery.adapter.ts` — queries TypeORM aisladas.
  - `password-recovery.module.ts` — registra entity, importa `MailModule` y `UsersModule`.
  - DTOs Zod 3.25 (`request-recovery.dto.ts`, `confirm-recovery.dto.ts`, `verify-recovery-query.dto.ts`).
  - `PasswordRecoveryMailService` en `apps/auth-users/src/mail/` que envuelve `renderPasswordRecovery` + `MailGatewayClient`.
- **3 endpoints REST públicos**:
  - `POST /password-recovery/request { email }` → 202 universal (no revela si el email existe). Throttler 5/15min IP + 3/h email. Inserta token, dispatch fire-and-forget post-commit.
  - `GET /password-recovery/verify?token=…` → 200 `{ valid: true }` o 410 `{ valid: false, reason: 'expired'|'used'|'unknown' }`. Throttler 20/1min IP. Read-only, sin transacción.
  - `POST /password-recovery/confirm { token, newPassword }` → 204. Throttler 5/1min IP. Transacción única con `pessimistic_write` en token; actualiza password, marca token usado, invalida tokens previos del mismo user, setea `users.password_changed_at`.
- **Tabla nueva `password_recovery_tokens`** (migration TypeORM): `id BIGINT PK`, `token CHAR(36) UK` (UUID v4), `user_id BIGINT FK users(id)`, `expires_at DATETIME`, `used_at DATETIME NULL`, `created_at DATETIME`. Indexes por `user_id` y `expires_at`.
- **Columna nueva `users.password_changed_at DATETIME NULL`** (migration TypeORM separada). Backfill = `users.created_at` para evitar invalidar sesiones existentes en producción.
- **JWT invalidation**: `JwtStrategy` rechaza con 401 si `iat * 1000 < user.password_changed_at`. Response usa el shape de error uniforme con código `password-changed`. El frontend lo intercepta y desloguea sin toast genérico.
- **`UsersService.markPasswordChanged(userId, qr)`** — método reusable (DRY) que actualiza `password_hash` + `password_changed_at` + opcionalmente flips `must_change_password = false`. Lo consume confirm + cualquier futura ruta de cambio de password.
- **Password policy compartida** en `pld-api/packages/domain-auth-users/src/crypto/password-policy.ts`: `validatePasswordStrength(plain)` + constants (min 8, max 72, requiere mayúscula + minúscula + número). Misma policy se mirroreará en pld-web vía Zod 4.
- **Throttler**: instalar `@nestjs/throttler`, configurar storage in-memory (no Redis por ahora — RC-17), aplicar a los 3 endpoints. Response 429 genérica.
- **Schedule**: instalar `@nestjs/schedule`, registrar cron diario en el service que borra tokens con `expires_at < NOW() - 1 day` o `used_at IS NOT NULL AND used_at < NOW() - 7 days`.
- **Plantilla de mail**: consume `renderPasswordRecovery` de `@pld-api/mail` (subject + bodyHtml). NO se modifica el package. Construye `recoveryUrl` como `${FRONTEND_LOGIN_URL_BASE}/recover-password?token=…`.

#### pld-web

- **4 vistas nuevas** bajo `pld-web/src/features/password-recovery/` (capturas en [docs/designs/password-recovery/](../../../docs/designs/password-recovery/)):
  1. `step-1-request-email/` — formulario con campo email + botón "Recuperar".
  2. `step-2-email-sent/` — confirmación "Revisa tu Bandeja de Entrada" con botón "Volver a Iniciar Sesión". **Universal**: se renderiza para CUALQUIER outcome del request (registrado/no registrado/inactivo/429/error de red) — RC-24.
  3. `step-3-new-password/` — formulario con `newPassword` + `confirmPassword`. Valida que coincidan + password policy mirror. Límite client-side: tras 3 intentos fallidos consecutivos redirige a `/login` con toast "Demasiados intentos, vuelve a solicitar el enlace" (RC-23). Antes de mostrar el form, ejecuta `GET /verify` con el token de la query; si responde 410, redirige a la vista de error.
  4. `result-page/` — éxito ("Contraseña actualizada, vuelve a iniciar sesión") y error de token expirado/usado.
- **Cableado del link "¿Olvidaste tu contraseña?"** en `LoginForm` apuntando a `/recover-password`.
- **Routing nuevo** en React Router: `/recover-password` (step-1), `/recover-password/sent` (step-2), `/recover-password/reset?token=…` (step-3), `/recover-password/done` (result success), `/recover-password/expired` (result error). Rutas públicas sin guard.
- **Cliente HTTP** `pld-web/src/services/password-recovery.ts` con `requestRecovery`, `verifyToken`, `confirmRecovery`. Mirror del DTO BE en `pld-web/src/types/password-recovery.ts`.
- **Schemas Zod 4** en `pld-web/src/features/password-recovery/schemas/` espejando `validatePasswordStrength` del backend.
- **Estado del wizard**: nada persistido entre sesiones; cada vista lee/escribe sus propios campos vía React Hook Form. El token vive en la URL.
- **Interceptor 401 con código `password-changed`**: forzar logout + redirect a `/login` sin toast genérico.

### Out of Scope

- Recuperación por SMS o por número de teléfono (el form de login acepta teléfono pero la recuperación es solo por email).
- Endpoint de "reenviar correo" (en step-2 no hay reintento — el botón lleva a `/login`).
- Cambio de password autenticado desde el perfil del usuario (`/account/change-password`).
- Migrar el throttler a storage Redis (queda in-memory hasta que haya múltiples instancias).
- Rate limiting global del resto de los endpoints de la API (solo los 3 nuevos).
- Bloqueo automático de cuenta tras N intentos de login fallidos.
- Auditoría / log persistente de intentos de recuperación más allá del cron de cleanup.
- Notificación por email "tu contraseña fue cambiada" tras confirm exitoso (queda como mejora futura).

## Approach

Replicar el patrón **controller → service → adapter** ya usado en `apps/auth-users/src/registration/auxiliaries/` y `apps/auth-users/src/admin/registration/`. El módulo `password-recovery/` queda autocontenido: entity propia, DTOs propios, migraciones bajo `packages/persistence/migrations/`. La capa de mail usa el package puro `@pld-api/mail` a través de un service intermedio de dominio (`PasswordRecoveryMailService`) que centraliza la construcción del `recoveryUrl` y el render de la plantilla.

Las **transacciones por endpoint** siguen el plan auditado en RC-21:

| Endpoint | Transacciones | Lock | Justificación |
|---|---|---|---|
| `POST /request` | 1 (insert token) + dispatch fire-and-forget POST-COMMIT | sin lock | Dispatch dentro de la transacción bloquearía 1-2s mientras manda el correo. Si cae el server entre commit y send, el token queda huérfano hasta el cron — usuario reintenta. |
| `GET /verify` | 0 (read-only) | sin lock | Solo lectura, idempotente. |
| `POST /confirm` | 1 atómica | `pessimistic_write` en token | Sin lock, dos requests concurrentes con el mismo token podrían cambiar la password dos veces. Sin atomicidad, si cae el server entre actualizar password y marcar usado, el token queda válido aunque la password ya cambió. La invalidación de otros tokens del user TIENE que ir en el mismo commit. |

El **JWT invalidation** se implementa comparando `iat * 1000 < user.password_changed_at` en `JwtStrategy.validate()`. No se mantiene un denylist de tokens — basta con la columna timestamp. El backfill de `password_changed_at = users.created_at` evita romper sesiones activas en producción al desplegar la migración.

El **throttler** se aplica vía decorators Nest sobre los 3 endpoints. Storage en memoria del proceso (suficiente para 1 réplica). Migrar a Redis cuando haya escalado horizontal — fuera de scope.

El **cron de cleanup** vive como método del `PasswordRecoveryService` decorado con `@Cron(CronExpression.EVERY_DAY_AT_3AM)`. Borra tokens expirados hace +1 día y tokens usados hace +7 días. La ventana de 7 días para `used_at` permite auditar reintentos cercanos.

El **frontend** sigue el patrón de las features existentes en `pld-web/src/features/` (auxiliary-registration, reporting-entity-registration). Cada vista es una página independiente con su propio form + schema Zod. El estado no se comparte entre pasos (el token vive en la URL, no en Zustand). Las transiciones se hacen vía `useNavigate()` directamente.

## Affected Areas

### pld-api

| Area | Impact | Description |
|------|--------|-------------|
| `pld-api/apps/auth-users/src/password-recovery/**` | New | Módulo completo: controller, service, adapter, DTOs, entity, módulo Nest. |
| `pld-api/apps/auth-users/src/mail/password-recovery-mail.service.ts` | New | Wrapper de `renderPasswordRecovery` + `MailGatewayClient`. |
| `pld-api/apps/auth-users/src/mail/credentials-mail.service.ts` | New (refactor) | Extraído de `dispatchCredentialsEmail` duplicado en auxiliaries + registration. |
| `pld-api/apps/auth-users/src/mail/mail.module.ts` | Modified | Provee `CredentialsMailService` y `PasswordRecoveryMailService`. |
| `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts` | Modified (refactor) | Inyecta `CredentialsMailService`; elimina `dispatchCredentialsEmail` privado. |
| `pld-api/apps/auth-users/src/admin/registration/registration.service.ts` | Modified (refactor) | Idem. |
| `pld-api/apps/auth-users/src/shared/auth/jwt.strategy.ts` | Modified | Comparar `iat` con `user.password_changed_at` y rechazar con código `password-changed`. (Única copia: `apps/cross` fue eliminada el 2026-05-02). |
| `pld-api/apps/auth-users/src/users/users.service.ts` | Modified | Agregar `markPasswordChanged(userId, qr?)`. |
| `pld-api/packages/persistence/migrations/<ts>-create-password-recovery-tokens.ts` | New | Tabla nueva + indexes. |
| `pld-api/packages/persistence/migrations/<ts>-add-password-changed-at-to-users.ts` | New | Columna nueva + backfill `= created_at`. |
| `pld-api/packages/domain-auth-users/src/crypto/password-policy.ts` | New | `validatePasswordStrength` + constants. |
| `pld-api/apps/auth-users/src/app/app.module.ts` | Modified | Importar `ThrottlerModule.forRoot(...)`, `ScheduleModule.forRoot()`, `PasswordRecoveryModule`. |
| `pld-api/apps/auth-users/package.json` (vía workspace root) | Modified | Agregar `@nestjs/throttler` + `@nestjs/schedule`. |

### pld-web

| Area | Impact | Description |
|------|--------|-------------|
| `pld-web/src/features/password-recovery/**` | New | 4 vistas + schemas + hooks. |
| `pld-web/src/services/password-recovery.ts` | New | Cliente HTTP. |
| `pld-web/src/types/password-recovery.ts` | New | Mirror DTOs BE. |
| `pld-web/src/router/**` | Modified | 5 rutas públicas nuevas (`/recover-password/*`). |
| `pld-web/src/features/login/components/LoginForm.tsx` | Modified | Cablear el link "¿Olvidaste tu contraseña?" a `/recover-password`. |
| `pld-web/src/services/api/interceptors.ts` (o equivalente) | Modified | Manejar 401 con código `password-changed` → forzar logout sin toast. |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Race en `confirm` cambia password dos veces | Med | Transacción única + `pessimistic_write` en token (RC-21). Tests concurrentes en sdd-verify. |
| Server crash entre commit y dispatch del email | Low | Token queda huérfano hasta cron de cleanup; usuario reintenta. Aceptado conscientemente. |
| Enumeration de emails vía `/request` | Med | Response 202 universal (no diferencia registrado vs no registrado) + throttler 5/15min IP + 3/h email. |
| Enumeration de tokens vía `/verify` | Med | Throttler 20/1min IP (RC-22). Tokens son UUID v4 (122 bits de entropía). |
| Romper sesiones activas al desplegar `password_changed_at` | High | Backfill `= users.created_at` en la misma migration. Smoke de login post-deploy. |
| JWT antiguo todavía válido tras password change | High | `JwtStrategy` compara `iat * 1000 < user.password_changed_at` en cada request. Frontend desloguea con código `password-changed`. |
| Drift de password policy BE vs FE | Med | Constantes compartidas vía `@pld-api/domain-auth-users` mirroreadas en Zod 4 del FE. sdd-verify chequea match. |
| Throttler in-memory falla con múltiples instancias | Low | Documentado como limitación. Migrar a Redis cuando haya escalado — fuera de scope. |
| Cron borra tokens en uso | Low | Ventana conservadora: solo expirados +1 día y usados +7 días. |
| 4 vistas FE drift respecto a las capturas | Med | Capturas en `docs/designs/password-recovery/` son fuente de verdad; smoke Playwright a 1500px. |
| Email no llega (gateway down) | Med | Dispatch fire-and-forget con log de error; usuario reintenta. Step-2 universal evita revelar el problema. |

## Rollback Plan

1. Revertir el merge del PR (FE + BE en commits separados — BE primero, FE después).
2. Ejecutar `down` de la migration `add-password-changed-at-to-users` (drop column).
3. Ejecutar `down` de la migration `create-password-recovery-tokens` (drop table).
4. Desinstalar `@nestjs/throttler` y `@nestjs/schedule` (revert del workspace root).
5. Smoke de login + del flujo de creación de auxiliares (que el refactor de `CredentialsMailService` no rompió nada).
6. Si solo el FE necesita rollback: revertir el PR FE deja el BE inerte (3 endpoints públicos sin consumidor — bajo riesgo).

## Dependencies

- **`@pld-api/mail`** ya implementado y mergeado (incluye `MailGatewayClient`, `renderPasswordRecovery`, `validateMailConfig` con Zod 3.25). Esta historia solo lo consume.
- **`FRONTEND_LOGIN_URL`** ya está en `.env.example` y la lee `MailConfigService`. Esta historia agrega construcción de `recoveryUrl` arriba de esa base.
- **`@nestjs/throttler`** y **`@nestjs/schedule`** se instalan en esta historia (no estaban antes).
- Sin dependencias de UX externas — capturas en `docs/designs/password-recovery/` son fuente de verdad.

## Success Criteria

### Backend

- [ ] `POST /password-recovery/request` con email registrado → 202 + email enviado con `recoveryUrl` válido.
- [ ] `POST /password-recovery/request` con email no registrado / inactivo → 202 (idéntico al caso anterior, sin diferencia observable).
- [ ] Throttler `request`: 6° request desde misma IP en <15min → 429.
- [ ] `GET /password-recovery/verify?token=…` con token válido → 200 `{ valid: true }`.
- [ ] `GET /verify` con token expirado / usado / inexistente → 410 `{ valid: false, reason }`.
- [ ] Throttler `verify`: 21° request desde misma IP en <1min → 429.
- [ ] `POST /password-recovery/confirm` con token + password válida → 204; password cambiada en BD; token marcado `used_at`; otros tokens del user invalidados; `users.password_changed_at = NOW()`.
- [ ] `POST /confirm` con password que viola la policy → 400.
- [ ] `POST /confirm` con token expirado / usado → 410.
- [ ] Tras confirm, JWTs emitidos antes de `password_changed_at` retornan 401 con código `password-changed`.
- [ ] Cron diario corre y borra tokens viejos sin afectar tokens activos.
- [ ] Tests e2e con stub de `MailGatewayClient` cubren los 3 endpoints + JWT invalidation.
- [ ] `nx run auth-users:build` verde.
- [ ] Refactor de `CredentialsMailService` no cambia el comportamiento observable de auxiliaries + registration (smoke).

### Frontend

- [ ] Click en "¿Olvidaste tu contraseña?" desde `/login` lleva a `/recover-password`.
- [ ] Submit del email lleva SIEMPRE a `/recover-password/sent` (RC-24), independientemente del outcome BE.
- [ ] Botón "Volver a Iniciar Sesión" en step-2 lleva a `/login`.
- [ ] Abrir `/recover-password/reset?token=<válido>` muestra el form de nueva password; con token inválido redirige a `/recover-password/expired`.
- [ ] Submit con passwords que coinciden + cumple policy → llama `POST /confirm` → redirige a `/recover-password/done`.
- [ ] 3 intentos consecutivos de passwords que no coinciden → redirect a `/login` con toast "Demasiados intentos…" (RC-23).
- [ ] Schemas Zod 4 espejan la password policy del BE (mismas reglas y mensajes).
- [ ] Tras confirm exitoso, si el usuario tenía un JWT antiguo, el siguiente request retorna 401 con código `password-changed` y el FE desloguea sin toast genérico.
- [ ] Smoke Playwright a 1500px del flujo completo (request → email stub → verify → confirm → login con nueva password) verde.
- [ ] `yarn build` (tsc -b && vite build) verde, sin warnings nuevos.
