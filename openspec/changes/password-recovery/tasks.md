# Tasks: Password recovery end-to-end

> Refines: [proposal.md](./proposal.md), [design.md](./design.md), [specs/password-recovery/spec.md](./specs/password-recovery/spec.md), [specs/auth/spec.md](./specs/auth/spec.md), [specs/users/spec.md](./specs/users/spec.md), [specs/pld-web/spec.md](./specs/pld-web/spec.md). Refinement Jarvis thread `ffcf3339-7f80-4889-b60f-c221f3c30f1f` iter 4.

## Phase 0: Refactor preparatorio — extraer CredentialsMailService (commit aparte)

- [ ] 0.1 [BE] Crear `apps/auth-users/src/mail/credentials-mail.service.ts` con clase `@Injectable() CredentialsMailService` que inyecta `MailGatewayClient` + `ConfigService`, expone `sendCredentialsEmail({ to, firstName, temporaryPassword, loginUrl })` consumiendo `renderCredentialsEmail` de `@pld-api/mail` (replicar la lógica hoy duplicada en `auxiliaries.service` y `registration.service`).
- [ ] 0.2 [BE] Registrar `CredentialsMailService` como provider y export en `apps/auth-users/src/mail/mail.module.ts`.
- [ ] 0.3 [BE] Modificar `apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts`: inyectar `CredentialsMailService` en lugar de `MailGatewayClient`, reemplazar el dispatch interno por `credentialsMailService.sendCredentialsEmail(...)`, eliminar el método privado `dispatchCredentialsEmail`.
- [ ] 0.4 [BE] Modificar `apps/auth-users/src/registration/auxiliaries/auxiliaries.module.ts`: importar `MailModule` (si no estaba ya) para exponer `CredentialsMailService`; remover provider local de `MailGatewayClient` si quedó huérfano.
- [ ] 0.5 [BE] Modificar `apps/auth-users/src/admin/registration/registration.service.ts`: ídem 0.3 — inyectar `CredentialsMailService` y eliminar el dispatch duplicado.
- [ ] 0.6 [BE] Modificar `apps/auth-users/src/admin/registration/registration.module.ts` (o el `*.module.ts` que provee `RegistrationService`): importar `MailModule` para resolver `CredentialsMailService`.
- [ ] 0.7 [BE] Actualizar / agregar test unitario `auxiliaries.service.spec.ts` para mockear `CredentialsMailService` en lugar de `MailGatewayClient`.
- [ ] 0.8 [BE] Actualizar / agregar test unitario `registration.service.spec.ts` para mockear `CredentialsMailService`.
- [ ] 0.9 [BE] `pnpm exec nx run auth-users:build` — verde.
- [ ] 0.10 [BE] Smoke manual: `POST /registration/auxiliaries` con NOTARY → 201 + email recibido por el stub. `POST /admin/registration` con SUPERADMIN → 201 + email recibido. Comportamiento observable idéntico al baseline.
- [ ] 0.11 [BE] Commit aparte: `refactor(mail): extract CredentialsMailService from auxiliaries and registration`.

## Phase 1: Backend — dependencias, password policy, migrations

- [ ] 1.1 [BE] Agregar `@nestjs/throttler` y `@nestjs/schedule` a `pld-api/package.json` (workspace root) → `pnpm install`. Verificar que `pnpm-lock.yaml` se actualizó.
- [ ] 1.2 [BE] Crear `pld-api/packages/domain-auth-users/src/crypto/password-policy.ts` con constantes `PASSWORD_POLICY` (minLength 8, maxLength 72, requireUppercase, requireLowercase, requireDigit), `PASSWORD_POLICY_MESSAGES` (en español) y la función `validatePasswordStrength(plain: string): { ok: boolean; failedRules: string[] }`.
- [ ] 1.3 [BE] Exportar `validatePasswordStrength`, `PASSWORD_POLICY` y `PASSWORD_POLICY_MESSAGES` desde el barrel `pld-api/packages/domain-auth-users/src/index.ts` (o el index público equivalente).
- [ ] 1.4 [BE] Crear `pld-api/packages/domain-auth-users/src/crypto/password-policy.spec.ts` con tests para INV-27, INV-28, INV-29, INV-30 (password OK, sin mayúscula, <8 chars, >72 chars).
- [ ] 1.5 [BE] Crear migration `pld-api/packages/persistence/migrations/<ts>-create-password-recovery-tokens.ts` con `up()`: `CREATE TABLE password_recovery_tokens (id BIGINT PK auto-increment, user_id BIGINT NOT NULL FK users(id) ON DELETE CASCADE, token CHAR(36) NOT NULL UNIQUE, expires_at DATETIME NOT NULL, used_at DATETIME NULL, created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP)` + `INDEX ix_user_id (user_id)` + `INDEX ix_expires_at (expires_at)`.
- [ ] 1.6 [BE] Implementar `down()` de la migration anterior con `DROP TABLE password_recovery_tokens`.
- [ ] 1.7 [BE] Crear migration separada `pld-api/packages/persistence/migrations/<ts>-add-password-changed-at-to-users.ts` con `up()`: `ALTER TABLE users ADD COLUMN password_changed_at DATETIME NULL AFTER password_hash` + backfill `UPDATE users SET password_changed_at = created_at WHERE password_changed_at IS NULL` (todo en la misma transacción de la migration).
- [ ] 1.8 [BE] Implementar `down()` con `ALTER TABLE users DROP COLUMN password_changed_at`.
- [ ] 1.9 [BE] Aplicar `up` de ambas migrations en BD local. Verificar via `SHOW CREATE TABLE password_recovery_tokens` y `SHOW COLUMNS FROM users LIKE 'password_changed_at'`. Verificar backfill: `SELECT COUNT(*) FROM users WHERE password_changed_at IS NULL` → 0.
- [ ] 1.10 [BE] Aplicar `down` en BD local; verificar drop de tabla y columna; volver a aplicar `up` para dejar BD lista.

## Phase 2: Backend — UserEntity + UsersService.markPasswordChanged

- [ ] 2.1 [BE] Modificar `apps/auth-users/src/users/entities/user.entity.ts` (o ruta equivalente del `UserEntity` consumido por TypeORM): agregar `@Column({ name: 'password_changed_at', type: 'datetime', nullable: true }) passwordChangedAt: Date | null`.
- [ ] 2.2 [BE] Agregar método `markPasswordChanged(userId: string, options: { newPasswordHash: string; mustChangePassword?: boolean }, qr?: QueryRunner): Promise<void>` en `apps/auth-users/src/users/users.service.ts` (o donde viva `UsersService`). El método actualiza `password_hash`, `password_changed_at = NOW()` y opcionalmente `must_change_password`. Usa `qr.manager` si recibe `QueryRunner`, si no, repository default. Lanza si el user no existe.
- [ ] 2.3 [BE] Exportar `markPasswordChanged` (asegurar que `UsersModule` exporte `UsersService`; ya debería estar pero verificar).
- [ ] 2.4 [BE] Crear/extender `apps/auth-users/src/users/users.service.spec.ts` con tests para INV-4, INV-5, INV-6, INV-7 (tres campos atómicamente, respeta QueryRunner externo, sin opt no toca flag, error si user no existe).

## Phase 3: Backend — JwtStrategy invalidation

- [ ] 3.1 [BE] Modificar `apps/auth-users/src/shared/auth/jwt.strategy.ts`: en `validate(payload)`, tras cargar el user, comparar `payload.iat * 1000 < user.passwordChangedAt?.getTime()` y, si se cumple, lanzar `UnauthorizedException({ code: 'password-changed', message: 'Tu contraseña fue cambiada. Iniciá sesión de nuevo.' })`. Mantener `<` estricto para INV-3.
<!-- 3.2 eliminado 2026-05-02 — apps/cross fue removida del monorepo. -->
- [x] 3.2 ~~Replicar en apps/cross~~ — N/A: app eliminada.
- [ ] 3.3 [BE] Verificar que el shape de la response 401 lo emite el `HttpErrorInterceptor` global con el código accesible (campo `code` o `errorDetails.code`). Si el interceptor desempaqueta `UnauthorizedException` y pierde el código, ajustar el filter o pasar el código como `metadata` siguiendo el patrón ya usado por otros errores tipados del repo.
- [ ] 3.4 [BE] Crear/extender `apps/auth-users/src/shared/auth/jwt.strategy.spec.ts` con tests para INV-1, INV-2, INV-3, INV-4, INV-6 (iat < pwdChangedAt → 401 password-changed; iat >= → pasa; `<` estricto en igualdad; NULL nunca dispara; firma inválida no usa el código).
<!-- 3.5 eliminado 2026-05-02 — apps/cross fue removida del monorepo. INV-5 deja de aplicar. -->
- [x] 3.5 ~~Crear/extender `apps/cross/.../jwt.strategy.spec.ts`~~ — N/A: app eliminada.

## Phase 4: Backend — Mail service intermedio (PasswordRecoveryMailService)

- [ ] 4.1 [BE] Crear `apps/auth-users/src/mail/password-recovery-mail.service.ts` con clase `@Injectable() PasswordRecoveryMailService` que inyecta `MailGatewayClient` + `MailConfigService`. Método `sendRecoveryLink({ to, firstName, token })` que compone `recoveryUrl = ${FRONTEND_LOGIN_URL_BASE}/recover-password/reset?token=<uuid>`, llama `renderPasswordRecovery({ firstName, recoveryUrl })` y dispatcha vía `MailGatewayClient.send`.
- [ ] 4.2 [BE] Registrar `PasswordRecoveryMailService` como provider y export en `apps/auth-users/src/mail/mail.module.ts`.
- [ ] 4.3 [BE] Crear `apps/auth-users/src/mail/password-recovery-mail.service.spec.ts` con tests para INV-31 (recoveryUrl en body HTML, subject de renderPasswordRecovery) e INV-32 (mensaje "30 minutos" en body — verificable contra `renderPasswordRecovery` real con stubbed config).

## Phase 5: Backend — Feature module password-recovery (entity + DTOs + adapter)

- [ ] 5.1 [BE] Crear directorio `apps/auth-users/src/password-recovery/` con subdirectorios `entities/` y `dto/`.
- [ ] 5.2 [BE] Crear `apps/auth-users/src/password-recovery/entities/password-recovery-token.entity.ts` con `@Entity('password_recovery_tokens') PasswordRecoveryToken { id, userId, token (CHAR 36 unique), expiresAt, usedAt nullable, createdAt @CreateDateColumn, user @ManyToOne(User) onDelete CASCADE }`.
- [ ] 5.3 [BE] Crear `apps/auth-users/src/password-recovery/dto/request-recovery.dto.ts` con Zod 3.25 schema `requestRecoverySchema = z.object({ email: z.string().email() })` + tipo derivado `RequestRecoveryDto`.
- [ ] 5.4 [BE] Crear `apps/auth-users/src/password-recovery/dto/verify-recovery-query.dto.ts` con Zod schema validando `token: z.string().uuid()` + tipo `VerifyRecoveryQueryDto`.
- [ ] 5.5 [BE] Crear `apps/auth-users/src/password-recovery/dto/confirm-recovery.dto.ts` con Zod schema: `token: z.string().uuid()`, `newPassword: z.string().min(PASSWORD_POLICY.minLength).max(PASSWORD_POLICY.maxLength).refine(v => validatePasswordStrength(v).ok, { message: 'No cumple la policy' })` (importa de `@pld-api/domain-auth-users`).
- [ ] 5.6 [BE] Crear `apps/auth-users/src/password-recovery/password-recovery.adapter.ts` con métodos: `findActiveUserByEmail(email): Promise<UserEntity | null>` (filtra `active = true`, `deleted_at IS NULL`); `insertToken(qr, userId, token, expiresAt)`; `findTokenByToken(token, { lock?: 'pessimistic_write', qr? })`; `markTokenUsed(qr, tokenId, usedAt)`; `invalidateOtherActiveTokens(qr, userId, exceptId, usedAt)`; `deleteExpiredOrUsedBefore(now)` (cron query con criterio dual de la spec INV-23/24/25/26).

## Phase 6: Backend — Feature module password-recovery (service + cron)

- [ ] 6.1 [BE] Crear `apps/auth-users/src/password-recovery/password-recovery.service.ts` con constructor que inyecta `PasswordRecoveryAdapter`, `UsersService`, `PasswordRecoveryMailService`, `DataSource` (TypeORM) y `Logger`.
- [ ] 6.2 [BE] Implementar `request(email: string): Promise<void>` — buscar user activo; si match, abrir transacción para insertar token (UUID v4 generado vía `crypto.randomUUID()`), `expiresAt = NOW() + 30min`; tras commit, hacer `setImmediate(() => mailService.sendRecoveryLink(...))` con catch que loguea sin propagar. Sin lock externo. Sin bifurcación de timing observable más allá de lo necesario.
- [ ] 6.3 [BE] Implementar `verify(token: string): Promise<{ valid: true } | { valid: false; reason: 'expired'|'used'|'unknown' }>` — read-only, sin transacción. Lookup por token; aplicar reglas de INV-6/7/8/9.
- [ ] 6.4 [BE] Implementar `confirm(token: string, newPassword: string): Promise<void>` — `dataSource.transaction(async qr => { ... })` con: SELECT FOR UPDATE del token (`pessimistic_write`); validar reglas de INV-12/13 (lanza con shape 410); hashear newPassword vía `hashPassword` de `@pld-api/domain-auth-users`; llamar `usersService.markPasswordChanged(userId, { newPasswordHash, mustChangePassword: false }, qr)`; `markTokenUsed(qr, token.id, now)`; `invalidateOtherActiveTokens(qr, userId, token.id, now)`. Commit. Si cualquier paso lanza, rollback.
- [ ] 6.5 [BE] Implementar método `cleanupExpiredTokens()` decorado con `@Cron(CronExpression.EVERY_DAY_AT_3AM, { name: 'password-recovery-cleanup' })` que llama al adapter con criterio dual: `expires_at < NOW() - 1 day` OR (`used_at IS NOT NULL AND used_at < NOW() - 7 days`). Loguea conteo afectado.
- [ ] 6.6 [BE] Crear `apps/auth-users/src/password-recovery/password-recovery.service.spec.ts` con tests unitarios para: request match (inserta + dispatcha), request no-match (no inserta + no dispatcha), request user inactivo (no inserta), request gateway failure (response no se afecta, error logueado), verify happy/expired/used/unknown, confirm happy (todos los side-effects), confirm expired/used → 410 sin side-effects, confirm crash entre steps deja todo intacto (INV-16), cleanup borra solo tokens fuera de ventana (INV-23/24/25/26).
- [ ] 6.7 [BE] Crear `apps/auth-users/src/password-recovery/password-recovery.adapter.spec.ts` con tests de queries básicas contra test DB (insert, lookup, lock, mark used, invalidate others, cleanup query).

## Phase 7: Backend — Feature module password-recovery (controller + module + throttler)

- [ ] 7.1 [BE] Crear `apps/auth-users/src/password-recovery/password-recovery.controller.ts` con `@Controller('password-recovery')` (prefijo global `pld-api/auth-users` ya configurado en `main.ts`). 3 handlers públicos:
  - `@Post('request') @HttpCode(202) @Throttle({ default: { limit: 5, ttl: 15*60*1000 } }) request(@Body) → service.request`. Sin `@UseGuards`.
  - `@Get('verify') @Throttle({ default: { limit: 20, ttl: 60*1000 } }) verify(@Query('token'))` → 200 o 410 con shape `{ valid, reason? }`.
  - `@Post('confirm') @HttpCode(204) @Throttle({ default: { limit: 5, ttl: 60*1000 } }) confirm(@Body)` → service.confirm; éxito devuelve void.
- [ ] 7.2 [BE] Implementar (o reusar) `EmailThrottlerGuard` que aplica límite por email del body en `request`: 3/hora con clave `email:<lowercased>`. Si throttle hits, igual responder 202 (universal) pero NO procesar (corto-circuito antes de adapter). Decorator custom o guard adicional sobre el endpoint.
- [ ] 7.3 [BE] Crear `apps/auth-users/src/password-recovery/password-recovery.module.ts` con `TypeOrmModule.forFeature([PasswordRecoveryToken])`, imports `[MailModule, UsersModule]`, providers `[PasswordRecoveryService, PasswordRecoveryAdapter]`, controllers `[PasswordRecoveryController]`. Exporta nada (consumo interno).
- [ ] 7.4 [BE] Modificar `apps/auth-users/src/app/app.module.ts` (o `auth-users.module.ts`) para importar `ThrottlerModule.forRoot([{ ttl: 60*1000, limit: 100 }])` (config base global), `ScheduleModule.forRoot()` y `PasswordRecoveryModule`.
- [ ] 7.5 [BE] Verificar via `nx run auth-users:lint` y `nx run auth-users:build` — verde.

## Phase 8: Backend — E2E tests con stub de MailGatewayClient

- [ ] 8.1 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-request.e2e-spec.ts` cubriendo INV-1, INV-2, INV-3, INV-4, INV-5 (happy + no registrado + inactivo + 400 + gateway fail).
- [ ] 8.2 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-verify.e2e-spec.ts` cubriendo INV-6, INV-7, INV-8, INV-9, INV-10.
- [ ] 8.3 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-confirm.e2e-spec.ts` cubriendo INV-11, INV-12, INV-13, INV-14, INV-15 (concurrencia con 2 requests paralelos), INV-16.
- [ ] 8.4 [BE] Crear `apps/auth-users-e2e/src/password-recovery/password-recovery-throttler.e2e-spec.ts` cubriendo INV-19, INV-20, INV-21, INV-22 (cuatro límites).
- [ ] 8.5 [BE] Crear `apps/auth-users-e2e/src/password-recovery/jwt-invalidation.e2e-spec.ts`: login con user, captura JWT antiguo, `POST /confirm` con token recovery, request a endpoint protegido con JWT antiguo → 401 código `password-changed`; login con password nueva → JWT nuevo OK.
- [ ] 8.6 [BE] `nx run auth-users-e2e:e2e` (o el target equivalente) — verde.

## Phase 9: Frontend — tipos, cliente HTTP, schemas Zod

- [ ] 9.1 [FE] Crear `pld-web/src/types/password-recovery.ts` con tipos: `RequestRecoveryRequest`, `VerifyRecoveryResponse = { valid: true } | { valid: false; reason: 'expired'|'used'|'unknown' }`, `ConfirmRecoveryRequest`, `PasswordChangedErrorCode = 'password-changed'`.
- [ ] 9.2 [FE] Crear `pld-web/src/services/password-recovery.ts` con: `requestRecovery(email): Promise<void>` (no rechaza ante 4xx/5xx — captura y resuelve void), `verifyToken(token): Promise<VerifyRecoveryResponse>` (mapea 200 → valid:true, 410 → valid:false con reason del body), `confirmRecovery(token, newPassword): Promise<void>` (lanza errores tipados ante 410/400/429).
- [ ] 9.3 [FE] Crear `pld-web/src/features/password-recovery/schemas/request-email.schema.ts` con Zod 4 `z.object({ email: z.email('Email inválido') })`.
- [ ] 9.4 [FE] Crear `pld-web/src/features/password-recovery/schemas/new-password.schema.ts` con Zod 4 espejando `validatePasswordStrength` BE: minLength 8, maxLength 72, requireUppercase, requireLowercase, requireDigit. Mensajes en español alineados con `PASSWORD_POLICY_MESSAGES`. Schema de form completo con `confirmPassword` y `.refine` de match.
- [ ] 9.5 [FE] Crear `pld-web/src/features/password-recovery/schemas/__tests__/new-password.schema.test.ts` con tests para INV-23, INV-24, INV-25 (sin mayúscula falla, <8 falla, OK pasa).

## Phase 10: Frontend — routing y feature scaffold

- [ ] 10.1 [FE] Crear directorio `pld-web/src/features/password-recovery/` con subdirectorios `pages/`, `components/`, `schemas/`, `hooks/`, `state/`.
- [ ] 10.2 [FE] Modificar `pld-web/src/router/` (o el archivo del routing principal) para registrar las 5 rutas públicas: `/recover-password` (step-1), `/recover-password/sent` (step-2), `/recover-password/reset` (step-3, lee `?token=`), `/recover-password/done` (result success), `/recover-password/expired` (result error). Sin auth guard.
- [ ] 10.3 [FE] Crear `pld-web/src/features/password-recovery/components/PasswordRecoveryLayout.tsx` con frame compartido (logo + copy + footer) reutilizable por las 4 vistas.
- [ ] 10.4 [FE] Modificar `pld-web/src/features/login/components/LoginForm.tsx`: cablear el link "¿Olvidaste tu contraseña?" a `<Link to="/recover-password">`.

## Phase 11: Frontend — hooks React Query

- [ ] 11.1 [FE] Crear `pld-web/src/features/password-recovery/hooks/useRequestRecovery.ts` con `useMutation` que llama `requestRecovery(email)`. `onSettled` (no `onSuccess`) navega a `/recover-password/sent` SIEMPRE (RC-24).
- [ ] 11.2 [FE] Crear `pld-web/src/features/password-recovery/hooks/useVerifyToken.ts` con `useQuery` (suspense o staleTime 0) que llama `verifyToken(token)` al montar. Disabled si no hay token.
- [ ] 11.3 [FE] Crear `pld-web/src/features/password-recovery/hooks/useConfirmRecovery.ts` con `useMutation` que llama `confirmRecovery(token, newPassword)`. Maneja 204 → navigate `/done`, 410 → navigate `/expired`, 400/429/5xx → setError inline.

## Phase 12: Frontend — vistas (pages)

- [ ] 12.1 [FE] Crear `pld-web/src/features/password-recovery/pages/RequestEmailPage.tsx`: form RHF + `request-email.schema`, campo email + botón "Recuperar". Submit usa `useRequestRecovery`. Captura en `docs/designs/password-recovery/step-1-*` como referencia visual.
- [ ] 12.2 [FE] Crear `pld-web/src/features/password-recovery/pages/EmailSentPage.tsx`: copy "Revisa tu Bandeja de Entrada", botón "Volver a Iniciar Sesión" → `/login`. Sin reintento, sin estado, sin params (INV-7/8/9).
- [ ] 12.3 [FE] Crear `pld-web/src/features/password-recovery/state/retryCounter.ts` exportando `useRetryCounter()` que mantiene contador in-component (useRef o useState) con métodos `increment`, `reset`, `value`. Sin localStorage.
- [ ] 12.4 [FE] Crear `pld-web/src/features/password-recovery/pages/NewPasswordPage.tsx`:
  - Lee `token` de query. Si falta → redirige a `/recover-password/expired` (INV-29).
  - `useVerifyToken({ token })` on-mount. Si 410 → redirige a `/recover-password/expired` (INV-10). Si 200 → renderiza form (INV-11).
  - Form RHF + `new-password.schema`: campos `newPassword` + `confirmPassword`. Validación de match + policy.
  - Submit Zod-fail por mismatch → `retryCounter.increment()`. Si counter >= 3, `navigate('/login', { state: { toast: 'Demasiados intentos…' } })` (INV-16).
  - Submit Zod-OK → `useConfirmRecovery.mutate`. 204 → `/recover-password/done` (INV-14). 410 → `/recover-password/expired` (INV-15). 400/429/5xx → error inline (INV-13 ya cubierto por validación previa).
- [ ] 12.5 [FE] Crear `pld-web/src/features/password-recovery/pages/ResultPage.tsx` con prop `variant: 'success' | 'expired'`. Variant success: copy "Contraseña actualizada", botón "Iniciar sesión" → `/login`. Variant expired: copy "Tu enlace expiró o ya fue usado", botón "Solicitar nuevo enlace" → `/recover-password`.
- [ ] 12.6 [FE] Wirear `ResultPage` en el router con dos rutas (`/recover-password/done` con `variant="success"` y `/recover-password/expired` con `variant="expired"`) en `routes.tsx` del feature.

## Phase 13: Frontend — interceptor 401 password-changed

- [ ] 13.1 [FE] Modificar `pld-web/src/services/api/interceptors.ts` (o el archivo del cliente axios — confirmar al implementar): en el response interceptor, detectar `err.response?.status === 401` y `err.response?.data?.code === 'password-changed'` (o `errorDetails.code` según shape uniforme). Si match: `authStore.logout()` + `navigate('/login', { state: { toast: { kind: 'info', message: 'Tu contraseña fue cambiada. Volvé a iniciar sesión.' } } })`. Sin toast genérico.
- [ ] 13.2 [FE] Edge case: si la URL actual está bajo `/recover-password/*`, NO disparar el redirect (evita loop si el confirm dispara este interceptor por re-fetch).
- [ ] 13.3 [FE] 401 sin código `password-changed` mantiene comportamiento previo (INV-27).

## Phase 14: Frontend — tests

- [ ] 14.1 [FE] Crear `pld-web/src/features/password-recovery/pages/__tests__/RequestEmailPage.test.tsx`: submit con éxito 202 → navega a /sent; submit con 429 → navega a /sent; submit con 5xx → navega a /sent; submit con email mal formado → bloquea + no llama backend (INV-5/6).
- [ ] 14.2 [FE] Crear `pld-web/src/features/password-recovery/pages/__tests__/EmailSentPage.test.tsx`: botón "Volver a Iniciar Sesión" navega a /login (INV-7); no hay botón de reenviar (INV-9).
- [ ] 14.3 [FE] Crear `pld-web/src/features/password-recovery/pages/__tests__/NewPasswordPage.test.tsx`: verify 410 → redirige a /expired; verify 200 → renderiza form; mismatch passwords incrementa counter; 3 mismatches → navigate /login + toast; submit OK 204 → navigate /done; confirm 410 → navigate /expired; reload resetea counter (INV-10/11/12/14/15/16/17/18); submit sin token en URL → navigate /expired (INV-29).
- [ ] 14.4 [FE] Crear `pld-web/src/services/__tests__/password-recovery.test.ts`: `requestRecovery` no rechaza ante 429/5xx (INV-21); `verifyToken` mapea 410 a `{ valid: false, reason }` (INV-22).
- [ ] 14.5 [FE] Crear `pld-web/src/services/api/__tests__/interceptors.test.ts`: 401 con código `password-changed` → logout + navigate + sin toast genérico (INV-26); 401 sin código → comportamiento previo (INV-27).

## Phase 15: Frontend — build verification

- [ ] 15.1 [FE] `cd pld-web && yarn tsc -b --noEmit` — verde, sin errores nuevos (INV-30).
- [ ] 15.2 [FE] `cd pld-web && yarn build` — verde, sin warnings nuevos.
- [ ] 15.3 [FE] `cd pld-web && yarn lint` — verde respecto al baseline.

## Phase 16: Smoke manual con Playwright (1500px viewport)

- [ ] 16.1 [SMOKE] Levantar stack completo via `/stack-up` (BE + Web).
- [ ] 16.2 [SMOKE] Resize Playwright a 1500px de ancho como primera acción.
- [ ] 16.3 [SMOKE] Login con user existente; capturar JWT en el storage; logout.
- [ ] 16.4 [SMOKE] Navegar a `/login`; click en "¿Olvidaste tu contraseña?" → verifica navegación a `/recover-password`. Captura en `docs/designs/password-recovery/evidence/01-login-link.png`.
- [ ] 16.5 [SMOKE] Submit email → verifica navegación a `/recover-password/sent`. Captura en `evidence/02-email-sent.png`.
- [ ] 16.6 [SMOKE] Recuperar el token bruto del email stub o de logs/DB (`SELECT token FROM password_recovery_tokens WHERE user_id = ? ORDER BY created_at DESC LIMIT 1`). Construir URL `/recover-password/reset?token=<uuid>`.
- [ ] 16.7 [SMOKE] Abrir URL con token; verificar que renderiza form (verify 200). Captura `evidence/03-new-password.png`.
- [ ] 16.8 [SMOKE] Probar 3 mismatches consecutivos → verificar redirect a `/login` con toast (RC-23). Captura `evidence/04-retry-limit.png`. Volver a step-1.
- [ ] 16.9 [SMOKE] Repetir flujo: submit email, recuperar token nuevo, abrir reset URL.
- [ ] 16.10 [SMOKE] Submit con passwords coincidentes y válidas → verifica redirect a `/recover-password/done`. Captura `evidence/05-done.png`.
- [ ] 16.11 [SMOKE] Login con la password nueva → 201. Captura `evidence/06-login-new-password.png`.
- [ ] 16.12 [SMOKE] Hacer un request a endpoint protegido con el JWT viejo (capturado en 16.3) → 401 con código `password-changed`. Captura del network tab `evidence/07-jwt-invalidation.png`.
- [ ] 16.13 [SMOKE] Probar token expirado: forzar `expires_at` en BD a NOW() - 1 hour, abrir URL → verifica redirect a `/recover-password/expired`. Captura `evidence/08-expired.png`.
- [ ] 16.14 [SMOKE] Probar throttle request: 6 submits rápidos del mismo email → verifica que la 6° aún navega a /sent (universal) pero que el log BE muestra solo 5 inserts. Captura `evidence/09-throttle.png`.

## Phase 17: Commits y archive

- [ ] 17.1 Commit 1 (Phase 0): `refactor(mail): extract CredentialsMailService from auxiliaries and registration`.
- [ ] 17.2 Commit 2 (Phases 1-8): `feat(password-recovery): add backend module with throttler, cron and JWT invalidation`.
- [ ] 17.3 Commit 3 (Phases 9-15): `feat(password-recovery): add frontend recovery flow with 401 password-changed interceptor`.
- [ ] 17.4 Commit 4 (Phase 16 + evidence): `docs(password-recovery): add Playwright smoke evidence at 1500px`.
- [ ] 17.5 Tras `sdd-verify` verde, ejecutar `sdd-archive password-recovery` para sincronizar specs y mover el change a `openspec/changes/archive/<ts>-password-recovery/`.
