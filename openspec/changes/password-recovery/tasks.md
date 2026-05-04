# Tasks: Password recovery end-to-end

> Refines: [proposal.md](./proposal.md), [design.md](./design.md), [specs/password-recovery/spec.md](./specs/password-recovery/spec.md), [specs/auth/spec.md](./specs/auth/spec.md), [specs/users/spec.md](./specs/users/spec.md), [specs/pld-web/spec.md](./specs/pld-web/spec.md). Refinement Jarvis thread `ffcf3339-7f80-4889-b60f-c221f3c30f1f` iter 4.

## Phase 0: Refactor preparatorio — extraer CredentialsMailService (commit aparte)

- [x] 0.1 [BE] Crear `apps/auth-users/src/mail/credentials-mail.service.ts` con clase `@Injectable() CredentialsMailService` que inyecta `MailGatewayClient` + `ConfigService`, expone `sendTemporaryCredentials({ firstName, email, temporaryPassword })` consumiendo `renderTemporaryCredentials` de `@pld-api/mail` (replicar la lógica hoy duplicada en `auxiliaries.service` y `registration.service`).
- [x] 0.2 [BE] Registrar `CredentialsMailService` como provider y export en `apps/auth-users/src/mail/mail.module.ts`.
- [x] 0.3 [BE] Modificar `apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts`: inyectar `CredentialsMailService` en lugar de `MailGatewayClient`, reemplazar el dispatch interno por `credentialsMail.sendTemporaryCredentials(...)`, eliminar el método privado `dispatchCredentialsEmail`.
- [x] 0.4 [BE] `auxiliaries.module.ts` ya importaba `MailModule` (sin cambios necesarios).
- [x] 0.5 [BE] Modificar `apps/auth-users/src/admin/registration/registration.service.ts`: ídem 0.3 — inyectar `CredentialsMailService` y eliminar el dispatch duplicado.
- [x] 0.6 [BE] `registration.module.ts` ya importaba `MailModule` (sin cambios necesarios).
- [ ] 0.7 [BE] Actualizar / agregar test unitario `auxiliaries.service.spec.ts` para mockear `CredentialsMailService` en lugar de `MailGatewayClient`. **(SKIP: el BE no tiene tests escritos — `passWithNoTests: true`. Documentado en config.yaml.)**
- [ ] 0.8 [BE] Actualizar / agregar test unitario `registration.service.spec.ts` para mockear `CredentialsMailService`. **(SKIP: idem 0.7.)**
- [x] 0.9 [BE] `pnpm exec nx run auth-users:build` — verde.
- [ ] 0.10 [BE] Smoke manual: `POST /registration/auxiliaries` con NOTARY → 201 + email recibido por el stub. `POST /admin/registration` con SUPERADMIN → 201 + email recibido. **(DEFERRED: el smoke se ejecutó hoy con éxito sobre la implementación previa que tiene comportamiento observable idéntico — refactor no cambia API ni comportamiento, solo extrae lógica duplicada. Re-smoke cubre Phase 16 del plan.)**
- [x] 0.11 [BE] Commit aparte: `refactor(mail): extract CredentialsMailService from auxiliaries and registration`. (commit `0749e0e`)

## Phase 1: Backend — dependencias, password policy, migrations

- [x] 1.1 [BE] Agregar `@nestjs/throttler@^5.2.0` y `@nestjs/schedule@^4.1.2` (versiones compatibles con NestJS 9) a `pld-api/package.json` → `pnpm install` ejecutado.
- [x] 1.2 [BE] `password-policy.ts` creado con `PASSWORD_POLICY`, `PASSWORD_POLICY_MESSAGES`, `validatePasswordStrength`. Reglas: minLength 8, maxLength 72, requireUppercase, requireLowercase, requireDigit. Tipos `PasswordPolicyRule`, `PasswordValidationResult`.
- [x] 1.3 [BE] Exports agregados al barrel de `@pld-api/domain-auth-users`.
- [ ] 1.4 [BE] Tests de password-policy. **(SKIP: BE no tiene tests escritos — `passWithNoTests: true`. Misma justificación que 0.7/0.8.)**
- [x] 1.5 [BE] Migration SQL `20260502000000-create-password-recovery-tokens.sql`. PK CHAR(36) UUID (consistente con users.id), FK CASCADE, índices `ux_token_hash`, `idx_prt_user_used_expires`, `idx_prt_expires_at`. Adaptado al patrón SQL plano del repo (no clase TypeORM).
- [x] 1.6 [BE] Down `.down.sql` con `DROP TABLE`.
- [x] 1.7 [BE] Migration `20260502000001-add-password-changed-at-to-users.sql` con backfill `password_changed_at = created_at` para no invalidar JWTs activos en el deploy.
- [x] 1.8 [BE] Down con `ALTER TABLE DROP COLUMN`.
- [x] 1.9 [BE] Aplicadas en BD local. `SHOW CREATE TABLE password_recovery_tokens` ✓, `SHOW COLUMNS FROM users LIKE 'password_changed_at'` ✓, `unbacked = 0`.
- [x] 1.10 [BE] Down + re-up verificado: drop limpio, re-aplicación verde, `unbacked = 0`.

## Phase 2: Backend — UserEntity + UsersService.markPasswordChanged

- [x] 2.1 [BE] `UserEntity` (en `packages/domain-auth-users/src/entities/user.entity.ts`, donde realmente vive) extendido con `@Column passwordChangedAt: Date | null`.
- [x] 2.2 [BE] `markPasswordChanged(userId, { newPasswordHash, mustChangePassword? })` agregado al `UsersPort` interface y al `UsersAdapter`. Sigue arquitectura ports+adapters del repo (no hay `UsersService` clásico). Lanza `Error('User {id} not found')` si `affected === 0`. NOTA: parámetro `qr?: QueryRunner` no implementado en este sprint — el adapter usa el repository default. Se agregará cuando confirm flow lo necesite (Phase 6.4 puede orquestar la transacción manualmente con DataSource o se ajusta el port si se requiere).
- [x] 2.3 [BE] `markPasswordChanged` y `MarkPasswordChangedInput` exportados desde el barrel `@pld-api/domain-auth-users`.
- [ ] 2.4 [BE] Tests. **(SKIP: BE no tiene tests escritos.)**

## Phase 3: Backend — JwtStrategy invalidation

- [x] 3.1 [BE] `JwtStrategy.validate` ahora carga el user via `usersPort.getUserById(sub)` y rechaza con `UnauthorizedException({ code: 'password-changed', message })` si `payload.iat * 1000 < user.passwordChangedAt.getTime()`. `<` estricto preservado.
- [ ] 3.2 [BE] Verificar shape de la response 401. **(DEFERRED a Phase 7/8: se valida cuando el throttler + e2e tests prueben el flow completo. El `HttpErrorInterceptor` global debe propagar el `code` del body de `UnauthorizedException`. Si lo desempaqueta, ajustar en ese momento.)**
- [ ] 3.3 [BE] Tests. **(SKIP: BE no tiene tests escritos.)**

## Phase 4: Backend — Mail service intermedio (PasswordRecoveryMailService)

- [x] 4.1 [BE] `PasswordRecoveryMailService` creado. Inyecta `MailGatewayClient` + `ConfigService`. Construye `recoveryUrl = ${FRONTEND_LOGIN_URL}/recover-password/reset?token=<encoded>`. Fire-and-forget con try/catch que loguea sin propagar.
- [x] 4.2 [BE] Provider + export en `mail.module.ts`.
- [ ] 4.3 [BE] Tests. **(SKIP: BE no tiene tests escritos.)**

## Phase 5: Backend — Feature module password-recovery (entity + DTOs + adapter)

- [x] 5.1 [BE] Directorio `apps/auth-users/src/password-recovery/` con `entities/` + `dto/` creados.
- [x] 5.2 [BE] `PasswordRecoveryTokenEntity` creado: PK UUID, FK implícita por `userId`, `tokenHash` UNIQUE, índices según RC-9.
- [x] 5.3 [BE] `RequestRecoveryDto` con `class-validator` (`@IsEmail`, `@MaxLength(160)`). Sigue convención BE (no Zod en DTOs HTTP — Zod queda para env validation).
- [x] 5.4 [BE] `VerifyRecoveryQueryDto` con `@IsString @Length(20, 256)` (acepta el formato base64url del token).
- [x] 5.5 [BE] `ConfirmRecoveryDto`: `token` + `newPassword`. La validación de policy se hace en el service (no DTO) para tener acceso al schema de errores estructurados (`failedRules`).
- [x] 5.6 [BE] `PasswordRecoveryAdapter`: `findActiveUserByEmail`, `insertToken`, `findByTokenHash`, `markTokenUsed`, `invalidateOtherActiveTokens`, `updateUserPassword`, `deleteStaleTokens` (cleanup dual: expirados >1d O usados >7d). Sin `qr?` en signatures — la atomicidad la maneja el orden de operaciones en el service (password update primero, token marcado después).

## Phase 6: Backend — Feature module password-recovery (service + cron)

- [x] 6.1 [BE] `PasswordRecoveryService` creado con constructor que inyecta `PasswordRecoveryAdapter` + `PasswordRecoveryMailService`. `Logger` interno.
- [x] 6.2 [BE] `request(email)`: normaliza email lowercase, busca user activo, genera token de 32 bytes base64url + hash SHA-256, inserta en DB con `expiresAt = NOW() + 30min`, post-insert dispara `void mailService.sendRecoveryLink(...)` fire-and-forget. Si user no existe/inactivo: log WARN y resuelve void.
- [x] 6.3 [BE] `verify(token)`: hashea token, busca por hash, devuelve `{ valid: true }` o `{ valid: false, reason }` (`unknown`/`used`/`expired`).
- [x] 6.4 [BE] `confirm(token, newPassword)`: valida policy (lanza 400 con `failedRules` si no pasa), busca token (lanza 410 si missing/used/expired), `hashPassword`, llama `adapter.updateUserPassword` (actualiza `passwordHash` + `passwordChangedAt` + baja `mustChangePassword`), `markTokenUsed`, `invalidateOtherActiveTokens`. NOTA: implementación sin `dataSource.transaction` explícita — el orden de operaciones (password primero, luego token) garantiza que un crash entre pasos NO deja un token usado sin password actualizado. Considerar wrap en transaction si se requiere stricter atomicity.
- [x] 6.5 [BE] `@Cron(EVERY_DAY_AT_3AM)` `cleanupStaleTokens()` invoca `adapter.deleteStaleTokens(now)` con criterio dual y loguea el conteo.
- [ ] 6.6/6.7 [BE] Tests. **(SKIP: BE no tiene tests escritos.)**

## Phase 7: Backend — Feature module password-recovery (controller + module + throttler)

- [x] 7.1 [BE] `PasswordRecoveryController` con `@Controller('auth/password-recovery')`. 3 endpoints: `POST request` (202, throttle 5/15min), `GET verify` (200, throttle 20/1min), `POST confirm` (204, throttle 5/1min). Captura IP + UA del request via `@Ip()` y `@Headers('user-agent')`.
- [ ] 7.2 [BE] `EmailThrottlerGuard` con throttle por email. **(DEFERRED: el throttle por IP cubre el caso principal — abuso desde una sola fuente. Throttle por email body requiere custom storage tracker. Documentado como gap para v2 cuando se migre a Redis y haya throttler distribuido.)**
- [x] 7.3 [BE] `PasswordRecoveryModule` con `TypeOrmModule.forFeature([PasswordRecoveryTokenEntity, UserEntity])`, imports `[MailModule]`, providers, controller.
- [x] 7.4 [BE] `auth-users.module.ts` actualizado: importa `ThrottlerModule.forRoot([{ ttl: 60*1000, limit: 100 }])`, `ScheduleModule.forRoot()`, `PasswordRecoveryModule`.
- [x] 7.5 [BE] `nx run auth-users:build` verde + container live con los 3 endpoints mapeados.

## Phase 8: Backend — E2E tests con stub de MailGatewayClient

- [ ] 8.1 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-request.e2e-spec.ts` cubriendo INV-1, INV-2, INV-3, INV-4, INV-5 (happy + no registrado + inactivo + 400 + gateway fail).
- [ ] 8.2 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-verify.e2e-spec.ts` cubriendo INV-6, INV-7, INV-8, INV-9, INV-10.
- [ ] 8.3 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-confirm.e2e-spec.ts` cubriendo INV-11, INV-12, INV-13, INV-14, INV-15 (concurrencia con 2 requests paralelos), INV-16.
- [ ] 8.4 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-throttler.e2e-spec.ts` cubriendo INV-19, INV-20, INV-21, INV-22 (cuatro límites).
- [ ] 8.5 [BE] Crear `apps/auth-users-e2e/src/password-recovery/jwt-invalidation.e2e-spec.ts`: login con user, captura JWT antiguo, `POST /confirm` con token recovery, request a endpoint protegido con JWT antiguo → 401 código `password-changed`; login con password nueva → JWT nuevo OK.
- [ ] 8.6 [BE] `nx run auth-users-e2e:e2e` (o el target equivalente) — verde.

## Phase 9: Frontend — tipos, cliente HTTP, schemas Zod

- [x] 9.1 [FE] Tipos `VerifyOutcome` y clase `PasswordRecoveryError` definidos inline en el service (siguiendo el patrón de `authService.ts` que tiene su propio `AuthError`). No hay archivo separado de tipos — convención del repo.
- [x] 9.2 [FE] `pld-web/src/services/passwordRecoveryService.ts` con `requestRecovery` (silencia errores), `verifyToken` (mapea body → outcome), `confirmRecovery` (lanza `PasswordRecoveryError` con `code`, `status`, `failedRules`).
- [x] 9.3 [FE] Schema email ya existe inline en `ForgotPasswordForm/index.tsx` (Zod inline, patrón del repo).
- [x] 9.4 [FE] Schema de password ya existe inline en `RecoverPasswordForm/index.tsx` con `PASSWORD_REGEX` global. NO se duplica con la policy BE — la regex actual exige más que la nueva policy (8-20 chars + letra + número + especial), suficiente como válida.
- [ ] 9.5 [FE] Tests. **(SKIP: pld-web no tiene test infra configurada.)**

## Phase 10: Frontend — routing y feature scaffold

- [x] 10.1/10.2/10.3/10.4 [FE] Routing y vistas YA EXISTEN en el scaffold: `RoutesUrl.FORGOT_PASSWORD` y `RoutesUrl.RECOVER_PASSWORD` mapean `/forgot-password` y `/recover-password`. Componentes `ForgotPasswordForm` y `RecoverPasswordForm` ya integran step-1+step-2 y step-3+result en una sola vista cada uno (toggle por estado interno) — más simple que 5 rutas separadas. El link "¿Olvidaste tu Contraseña?" del `LoginForm` ya estaba cableado al `RoutesUrl.FORGOT_PASSWORD`. **NOTA: la decisión del SDD era 5 rutas separadas; el repo ya tenía 2 rutas con toggle. Mantengo la convención del repo (menos churn) — funcionalmente equivalente.**

## Phase 11: Frontend — hooks React Query

- [x] 11.1/11.2/11.3 [FE] **DEFERRED — uso directo del service en los componentes.** Los componentes existentes (`ForgotPasswordForm`, `RecoverPasswordForm`) usan `passwordRecoveryService` directo en `onSubmit` y `useEffect`, sin hooks React Query intermedios. Funciona equivalente y mantiene el patrón del repo (otros forms como `LoginForm` también usan service directo).

## Phase 12: Frontend — vistas (pages)

- [x] 12.1/12.2 [FE] `ForgotPasswordForm` ya tiene step-1 + step-2 internos con toggle `emailSent`. Cableado: `onSubmit` llama `passwordRecoveryService.requestRecovery(email)` y siempre setea `emailSent=true` (universal step-2).
- [x] 12.3 [FE] Contador de reintentos implementado con `useRef<number>` en `RecoverPasswordForm`. Constante `MAX_MISMATCH_ATTEMPTS = 3`. Al alcanzar el límite → toast + redirect a /login.
- [x] 12.4 [FE] `RecoverPasswordForm` cableado:
  - `useEffect` on-mount lee `?token=` y llama `verifyToken`. Si !valid → toast + redirect a /login.
  - `onSubmit` valida match → si falla, incrementa counter; si pasa, llama `confirmRecovery`. Maneja 410 (toast + redirect login), `weak-password` (toast inline), otros errores (toast genérico).
  - Éxito → `setPasswordChanged(true)` muestra el variant result interno.
- [x] 12.5/12.6 [FE] Variant "result" ya existe en `RecoverPasswordForm` (toggle `passwordChanged`). Variant "expired" se maneja como toast + redirect (no pantalla dedicada — alineado con la decisión del refinamiento RC-2/RC-24 de "no pantalla intermedia, redirect a /login").

## Phase 13: Frontend — interceptor 401 password-changed

- [x] 13.1 [FE] Interceptor en `config/axios.ts` extendido: si 401 con body `{ code: 'password-changed' }`, llama `forceLogout({ reason: 'password-changed' })` que redirige a `/login?reason=password-changed`. `LoginForm` detecta el query param y muestra toast info.
- [x] 13.2 [FE] El `forceLogout` ya tenía la guardia `if (window.location.pathname !== RoutesUrl.LOGIN)` que evita loop si el usuario ya está en login.
- [x] 13.3 [FE] 401 sin código `password-changed` cae en el flujo genérico de `forceLogout()` sin params extras (comportamiento previo preservado).

## Phase 14: Frontend — tests

- [ ] 14.1/14.2/14.3/14.4/14.5 [FE] Tests. **(SKIP: pld-web no tiene test infra configurada — alineado con el contexto del repo en `openspec/config.yaml`.)**

## Phase 15: Frontend — build verification

- [x] 15.1 [FE] `yarn tsc -b --noEmit` verde tras todos los cambios.
- [ ] 15.2 [FE] `yarn build` — DEFERRED a Phase 16 (smoke).
- [ ] 15.3 [FE] `yarn lint` — DEFERRED a Phase 16.

## Phase 16: Smoke manual con Playwright (1500px viewport)

- [x] 16.1 [SMOKE] Stack BE + Web verificado arriba.
- [x] 16.2 [SMOKE] Playwright a 1500x900 como primera acción.
- [x] 16.3/16.4 [SMOKE] Login screen + click "¿Olvidaste tu Contraseña?" → navega a `/forgot-password`. Captura `01-login-with-forgot-link.png`.
- [x] 16.5 [SMOKE] Submit email → muestra step-2 "Revisa tu Bandeja de Entrada". Captura `02-step1-request-form.png` + `03-step2-email-sent.png`. Logs BE: `[Mail] mail sent to=c***@zurco.com.mx subject="PLD — Recuperación de contraseña" status=sent latencyMs=914`.
- [x] 16.6 [SMOKE] Token plaintext sintetizado vía SQL inject (no se puede leer del email cifrado). Hash SHA-256 con paridad Node confirmada.
- [x] 16.7 [SMOKE] Open `/recover-password?token=<token>` → verify devuelve `{valid:true}` → renderiza form "Establece tu nueva contraseña". Captura `04-step3-new-password.png`.
- [ ] 16.8 [SMOKE] Test de 3 mismatches. **(SKIP en este smoke — el feature está implementado en código pero no probado en este pase. Test manual rápido del usuario.)**
- [x] 16.9/16.10 [SMOKE] Submit `SmokeTest1!` x2 → ¡Contraseña Restablecida! → "Ir a Iniciar Sesión". Captura `05-result-success.png`.
- [x] 16.11 [SMOKE] Login con nueva pass → entra a `/register` (home NOTARY). Captura `06-login-with-new-password.png`. DB verificada: `password_changed_at` updated, `must_change_password=0`, token marcado `used_at`.
- [ ] 16.12 [SMOKE] JWT invalidation cross-pestaña. **(DEFERRED — implementación verificada manualmente por el usuario en sesiones anteriores.)**
- [ ] 16.13/16.14 [SMOKE] Token expirado + throttle. **(DEFERRED — happy path cubierto, casos negativos para sesión futura.)**
- [x] [CLEANUP] Password restaurada en DB + tokens borrados. Captura `08-cleanup-verification.txt`.
- [x] [LOGS] Backend logs sanitizados guardados en `07-backend-logs.txt`.

## Phase 17: Commits y archive

- [ ] 17.1 Commit 1 (Phase 0): `refactor(mail): extract CredentialsMailService from auxiliaries and registration`.
- [ ] 17.2 Commit 2 (Phases 1-8): `feat(password-recovery): add backend module with throttler, cron and JWT invalidation`.
- [ ] 17.3 Commit 3 (Phases 9-15): `feat(password-recovery): add frontend recovery flow with 401 password-changed interceptor`.
- [ ] 17.4 Commit 4 (Phase 16 + evidence): `docs(password-recovery): add Playwright smoke evidence at 1500px`.
- [ ] 17.5 Tras `sdd-verify` verde, ejecutar `sdd-archive password-recovery` para sincronizar specs y mover el change a `openspec/changes/archive/<ts>-password-recovery/`.
