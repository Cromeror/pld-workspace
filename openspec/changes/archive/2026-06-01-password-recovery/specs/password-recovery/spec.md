# Delta for Password Recovery

> Capability nueva. Define los 3 endpoints REST públicos de recuperación de contraseña por enlace mágico, la tabla `password_recovery_tokens`, el throttler in-memory, el cron de cleanup y la policy compartida de contraseña. Flujo BE de referencia: [docs/FLUJO_LOGIN.md](../../../../docs/FLUJO_LOGIN.md). Refinement Jarvis thread `ffcf3339-7f80-4889-b60f-c221f3c30f1f` iter 4.

## ADDED Requirements

### Requirement: Endpoint POST /password-recovery/request

El sistema MUST exponer `POST /pld-api/auth-users/password-recovery/request` como endpoint público (sin `JwtAuthGuard`). El endpoint MUST aceptar `{ email: string }` validado vía Zod (`IsEmail`). El servidor MUST responder HTTP 202 con body vacío para CUALQUIER outcome observable: email registrado, email no registrado, usuario inactivo, usuario soft-deleted, fallo del gateway de mail. El response MUST NOT diferenciar el outcome para impedir enumeración (RC-6).

Cuando el email corresponde a un usuario activo no eliminado, el sistema MUST insertar una row en `password_recovery_tokens` con `token = UUID v4`, `user_id`, `expires_at = NOW() + 30 minutes`, `used_at = NULL`, `created_at = NOW()`. El dispatch del email MUST ejecutarse fire-and-forget POST-COMMIT (RC-21). Si el dispatch falla, el sistema MUST loggear el error sin afectar el response.

#### INV-1: Email registrado retorna 202 y dispara correo

- GIVEN un usuario activo no eliminado con email `user@example.com`
- WHEN un cliente envía `POST /pld-api/auth-users/password-recovery/request` con `{ email: "user@example.com" }`
- THEN el servidor SHALL responder HTTP 202 sin body
- AND SHALL existir una nueva row en `password_recovery_tokens` con `user_id` correspondiente, `expires_at` ≈ NOW() + 30min y `used_at = NULL`
- AND el sistema SHALL invocar el `MailGatewayClient` con asunto y cuerpo HTML producidos por `renderPasswordRecovery`
- AND el cuerpo SHALL incluir un `recoveryUrl` con el formato `${FRONTEND_LOGIN_URL_BASE}/recover-password/reset?token=<uuid>`

#### INV-2: Email no registrado retorna 202 sin side-effects

- GIVEN ningún usuario existe con email `nope@example.com`
- WHEN un cliente envía `POST /password-recovery/request` con ese email
- THEN el servidor SHALL responder HTTP 202 sin body
- AND NO SHALL insertar rows en `password_recovery_tokens`
- AND NO SHALL invocar el `MailGatewayClient`

#### INV-3: Usuario inactivo o soft-deleted retorna 202 sin side-effects

- GIVEN un usuario con `active = false` o `deletedAt` no nulo, con email `inactive@example.com`
- WHEN un cliente envía `POST /password-recovery/request` con ese email
- THEN el servidor SHALL responder HTTP 202 sin body
- AND NO SHALL insertar token ni invocar `MailGatewayClient`

#### INV-4: Email mal formado retorna 400

- GIVEN body con `{ email: "no-es-email" }`
- WHEN un cliente envía `POST /password-recovery/request`
- THEN el servidor SHALL responder HTTP 400 con el shape de error uniforme

#### INV-5: Fallo del gateway no afecta el response

- GIVEN un usuario activo y un `MailGatewayClient` que lanza error en `send`
- WHEN un cliente envía `POST /password-recovery/request` con su email
- THEN el servidor SHALL responder HTTP 202 sin body
- AND el token SHALL haber sido persistido (commit de la transacción ocurrió antes del dispatch)
- AND el error del gateway SHALL aparecer únicamente en logs

### Requirement: Endpoint GET /password-recovery/verify

El sistema MUST exponer `GET /pld-api/auth-users/password-recovery/verify?token=<uuid>` como endpoint público (sin `JwtAuthGuard`). El endpoint MUST validar el query param `token` con shape UUID v4. El handler MUST ser read-only — sin transacción ni locks (RC-21).

El servidor MUST responder HTTP 200 con `{ valid: true }` cuando el token existe, no fue usado y `expires_at > NOW()`. En cualquier otro caso (token desconocido, expirado o ya usado) el servidor MUST responder HTTP 410 con `{ valid: false, reason: 'expired' | 'used' | 'unknown' }` (RC-7).

#### INV-6: Token válido retorna 200

- GIVEN un token persistido con `used_at = NULL` y `expires_at > NOW()`
- WHEN un cliente envía `GET /password-recovery/verify?token=<uuid>`
- THEN el servidor SHALL responder HTTP 200 con body `{ valid: true }`

#### INV-7: Token expirado retorna 410 con reason expired

- GIVEN un token persistido con `used_at = NULL` y `expires_at < NOW()`
- WHEN un cliente envía `GET /password-recovery/verify?token=<uuid>`
- THEN el servidor SHALL responder HTTP 410 con body `{ valid: false, reason: "expired" }`

#### INV-8: Token usado retorna 410 con reason used

- GIVEN un token persistido con `used_at` no nulo
- WHEN un cliente envía `GET /password-recovery/verify?token=<uuid>`
- THEN el servidor SHALL responder HTTP 410 con body `{ valid: false, reason: "used" }`

#### INV-9: Token desconocido retorna 410 con reason unknown

- GIVEN un UUID v4 que no existe en `password_recovery_tokens`
- WHEN un cliente envía `GET /password-recovery/verify?token=<uuid>`
- THEN el servidor SHALL responder HTTP 410 con body `{ valid: false, reason: "unknown" }`

#### INV-10: Token con shape inválido retorna 400

- GIVEN query string con `token=not-a-uuid`
- WHEN un cliente envía `GET /password-recovery/verify`
- THEN el servidor SHALL responder HTTP 400

### Requirement: Endpoint POST /password-recovery/confirm

El sistema MUST exponer `POST /pld-api/auth-users/password-recovery/confirm` como endpoint público (sin `JwtAuthGuard`). El endpoint MUST aceptar `{ token: string, newPassword: string }` validado vía Zod. El handler MUST ejecutar una transacción atómica única con `pessimistic_write` lock sobre la row del token (RC-21). El servidor MUST responder HTTP 204 sin body cuando la operación tiene éxito (RC-8).

Dentro de la transacción el sistema MUST:

1. Hacer SELECT FOR UPDATE de la row del token.
2. Validar que `used_at IS NULL` y `expires_at > NOW()`. Si no cumple, abortar con HTTP 410 (`{ reason: 'expired' | 'used' | 'unknown' }`).
3. Validar `newPassword` contra la password policy compartida (mínimo 8, máximo 72, requiere mayúscula + minúscula + número). Si falla, abortar con HTTP 400.
4. Actualizar `users.password_hash` con el nuevo hash, `users.password_changed_at = NOW()` y opcionalmente flipear `users.must_change_password = false` vía `UsersService.markPasswordChanged`.
5. Marcar el token usado con `used_at = NOW()`.
6. Invalidar todos los tokens previos del mismo `user_id` (que aún no estén usados ni expirados) marcándolos `used_at = NOW()`.
7. Hacer commit.

Si CUALQUIER paso falla, la transacción MUST hacer rollback completo.

#### INV-11: Confirm exitoso retorna 204 y aplica cambios

- GIVEN un token válido y una password que cumple la policy
- WHEN un cliente envía `POST /password-recovery/confirm` con ese token y `newPassword`
- THEN el servidor SHALL responder HTTP 204 sin body
- AND `users.password_hash` SHALL coincidir con `verifyPassword(stored, newPassword) === true`
- AND `users.password_changed_at` SHALL reflejar el timestamp del confirm
- AND la row del token SHALL tener `used_at` no nulo
- AND cualquier otro token activo del mismo `user_id` SHALL tener `used_at` no nulo tras el commit

#### INV-12: Confirm con token expirado retorna 410

- GIVEN un token con `expires_at < NOW()`
- WHEN un cliente envía `POST /password-recovery/confirm` con ese token y password válida
- THEN el servidor SHALL responder HTTP 410 con body `{ valid: false, reason: "expired" }`
- AND `users.password_hash` SHALL permanecer sin cambios
- AND `users.password_changed_at` SHALL permanecer sin cambios

#### INV-13: Confirm con token usado retorna 410

- GIVEN un token con `used_at` no nulo
- WHEN un cliente envía `POST /password-recovery/confirm`
- THEN el servidor SHALL responder HTTP 410 con body `{ valid: false, reason: "used" }`
- AND no SHALL haber side-effects sobre `users`

#### INV-14: Confirm con password que viola la policy retorna 400

- GIVEN un token válido
- WHEN un cliente envía `POST /password-recovery/confirm` con `newPassword = "abc"` (sin mayúscula, sin número, <8 chars)
- THEN el servidor SHALL responder HTTP 400
- AND el body SHALL listar los criterios incumplidos
- AND la row del token NO SHALL haberse marcado como usada
- AND `users.password_hash` NO SHALL haberse actualizado

#### INV-15: Concurrencia con mismo token aplica solo una vez

- GIVEN un token válido y dos requests concurrentes a `POST /password-recovery/confirm` con el mismo token
- WHEN ambos requests llegan al servidor en paralelo
- THEN exactamente UNO SHALL responder HTTP 204
- AND el otro SHALL responder HTTP 410 con `reason: "used"`
- AND `users.password_hash` SHALL haber cambiado exactamente una vez

#### INV-16: Crash entre actualizar password y marcar usado deja todo intacto

- GIVEN un token válido y un fallo simulado entre el UPDATE de `users` y el UPDATE de `password_recovery_tokens.used_at`
- WHEN la transacción aborta
- THEN `users.password_hash` SHALL permanecer en el valor previo al request
- AND la row del token SHALL conservar `used_at = NULL`
- AND `users.password_changed_at` SHALL permanecer en el valor previo

### Requirement: Tabla password_recovery_tokens

La migración TypeORM MUST crear una tabla nueva `password_recovery_tokens` con la siguiente estructura (RC-9):

| Columna | Tipo | Restricción |
|---|---|---|
| `id` | BIGINT | PK auto-increment |
| `token` | CHAR(36) | UNIQUE, NOT NULL, UUID v4 |
| `user_id` | BIGINT | FK `users(id)`, NOT NULL |
| `expires_at` | DATETIME | NOT NULL |
| `used_at` | DATETIME | NULL |
| `created_at` | DATETIME | NOT NULL, default CURRENT_TIMESTAMP |

La migración MUST crear índices sobre `user_id` y `expires_at`. El down de la migración MUST hacer drop de la tabla.

#### INV-17: Tabla creada con shape correcto

- GIVEN la migración `create-password-recovery-tokens` fue ejecutada
- WHEN se inspecciona `INFORMATION_SCHEMA.COLUMNS` para `password_recovery_tokens`
- THEN SHALL existir la tabla con las columnas listadas y tipos correctos
- AND `token` SHALL tener constraint UNIQUE
- AND `user_id` SHALL tener FK a `users(id)`
- AND SHALL existir índice sobre `user_id` y sobre `expires_at`

#### INV-18: Down de la migración borra la tabla

- GIVEN la migración up fue ejecutada
- WHEN se ejecuta `down`
- THEN la tabla `password_recovery_tokens` NO SHALL existir
- AND no SHALL quedar índices residuales asociados

### Requirement: Throttler aplicado a los 3 endpoints

El sistema MUST aplicar `@nestjs/throttler` con storage in-memory a los 3 endpoints públicos del módulo (RC-17, RC-22). La response al exceder el límite MUST ser HTTP 429 con shape genérico, sin revelar el límite exacto.

| Endpoint | Límite |
|---|---|
| `POST /password-recovery/request` | 5 requests / 15min por IP **y** 3 requests / hora por email |
| `GET /password-recovery/verify` | 20 requests / 1min por IP |
| `POST /password-recovery/confirm` | 5 requests / 1min por IP |

#### INV-19: Throttler en request por IP

- GIVEN una IP que ya envió 5 requests exitosos a `POST /password-recovery/request` en los últimos 15 minutos
- WHEN envía un 6° request desde la misma IP
- THEN el servidor SHALL responder HTTP 429
- AND no SHALL insertar token ni invocar gateway de mail

#### INV-20: Throttler en request por email

- GIVEN un email que ya disparó 3 requests a `POST /password-recovery/request` en la última hora (desde IPs diferentes)
- WHEN se envía un 4° request con el mismo email
- THEN el servidor SHALL responder HTTP 429

#### INV-21: Throttler en verify

- GIVEN una IP que envió 20 requests a `GET /password-recovery/verify` en los últimos 60 segundos
- WHEN envía un 21° request desde la misma IP
- THEN el servidor SHALL responder HTTP 429

#### INV-22: Throttler en confirm

- GIVEN una IP que envió 5 requests a `POST /password-recovery/confirm` en los últimos 60 segundos
- WHEN envía un 6° request desde la misma IP
- THEN el servidor SHALL responder HTTP 429

### Requirement: Cron diario de cleanup

El módulo `password-recovery` MUST registrar un job programado con `@nestjs/schedule` (`@Cron(CronExpression.EVERY_DAY_AT_3AM)`) que borra rows de `password_recovery_tokens` (RC-15). El cron MUST eliminar rows que cumplan al menos UNA de:

- `expires_at < NOW() - INTERVAL 1 DAY` (tokens expirados hace más de 1 día).
- `used_at IS NOT NULL AND used_at < NOW() - INTERVAL 7 DAY` (tokens usados hace más de 7 días).

El cron MUST NOT borrar tokens activos (`used_at IS NULL AND expires_at > NOW()`) ni tokens recién expirados/usados dentro de la ventana.

#### INV-23: Cron borra tokens expirados +1 día

- GIVEN un token con `expires_at = NOW() - 2 days` y `used_at = NULL`
- WHEN el cron diario ejecuta
- THEN la row SHALL ser eliminada de `password_recovery_tokens`

#### INV-24: Cron borra tokens usados +7 días

- GIVEN un token con `used_at = NOW() - 8 days`
- WHEN el cron diario ejecuta
- THEN la row SHALL ser eliminada

#### INV-25: Cron preserva tokens activos

- GIVEN un token con `expires_at = NOW() + 10 minutes` y `used_at = NULL`
- WHEN el cron diario ejecuta
- THEN la row SHALL permanecer en la tabla sin cambios

#### INV-26: Cron preserva tokens dentro de la ventana de gracia

- GIVEN un token con `used_at = NOW() - 3 days`
- WHEN el cron diario ejecuta
- THEN la row SHALL permanecer en la tabla (ventana de auditoría de 7 días)

### Requirement: Password policy compartida

El paquete `@pld-api/domain-auth-users` MUST exportar una función `validatePasswordStrength(plain: string): { ok: boolean; failedRules: string[] }` y constantes asociadas (mínimo 8, máximo 72, requiere mayúscula + minúscula + número) (RC-14). Esta policy MUST ser la única fuente de verdad para validación de complejidad de password en el backend. El frontend MUST espejar las mismas reglas vía Zod 4.

#### INV-27: Password con todos los criterios pasa

- GIVEN una password `"Abcd1234"`
- WHEN se invoca `validatePasswordStrength`
- THEN SHALL retornar `{ ok: true, failedRules: [] }`

#### INV-28: Password sin mayúscula falla

- GIVEN una password `"abcd1234"`
- WHEN se invoca `validatePasswordStrength`
- THEN SHALL retornar `{ ok: false, failedRules: [..., 'uppercase', ...] }`

#### INV-29: Password con menos de 8 caracteres falla

- GIVEN una password `"Abc12"`
- WHEN se invoca `validatePasswordStrength`
- THEN SHALL retornar `{ ok: false, failedRules: [..., 'minLength', ...] }`

#### INV-30: Password con más de 72 caracteres falla

- GIVEN una password de 73 caracteres
- WHEN se invoca `validatePasswordStrength`
- THEN SHALL retornar `{ ok: false, failedRules: [..., 'maxLength', ...] }`

### Requirement: Plantilla del correo de recuperación

El módulo MUST consumir `renderPasswordRecovery` de `@pld-api/mail` sin modificar el package (RC-12'). El service intermedio `PasswordRecoveryMailService` (en `apps/auth-users/src/mail/`) MUST construir el `recoveryUrl` como `${FRONTEND_LOGIN_URL_BASE}/recover-password/reset?token=<uuid>` usando `MailConfigService` para obtener `FRONTEND_LOGIN_URL_BASE`.

#### INV-31: Service intermedio existe y compone recoveryUrl

- GIVEN una invocación a `PasswordRecoveryMailService.sendRecoveryLink({ to, firstName, recoveryUrl })`
- WHEN se inspecciona el llamado al `MailGatewayClient.send`
- THEN el body HTML SHALL contener un `<a href>` apuntando al `recoveryUrl` recibido
- AND el subject SHALL ser el producido por `renderPasswordRecovery`

#### INV-32: Token expira en 30 minutos en el correo

- GIVEN una invocación de `renderPasswordRecovery` desde el service
- WHEN se construye el body HTML
- THEN el contenido SHALL indicar al usuario que el enlace expira en 30 minutos

### Requirement: Tokens UUID v4 con entropía suficiente

El sistema MUST generar tokens usando UUID v4 (122 bits de entropía) producidos por una fuente CSPRNG (RC-9). El sistema MUST NOT generar tokens con `Math.random()` ni con secuencias derivadas de `user_id`.

#### INV-33: Token persistido tiene shape UUID v4

- GIVEN un POST `/password-recovery/request` exitoso
- WHEN se inspecciona la row creada
- THEN `token` SHALL coincidir con la regex `^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`

### Requirement: Módulo password-recovery autocontenido

El feature module `apps/auth-users/src/password-recovery/` MUST seguir el patrón controller → service → adapter ya usado en `auxiliaries/` y `admin/registration/` (RC-20). El módulo MUST registrar la entity vía `TypeOrmModule.forFeature` y MUST importar `MailModule` (que provee `PasswordRecoveryMailService`) y `UsersModule` (que provee `UsersService.markPasswordChanged`).

El módulo MUST NOT exponer endpoints que requieran JWT — los 3 endpoints son públicos.

#### INV-34: Endpoints son públicos

- GIVEN el controller `PasswordRecoveryController`
- WHEN se inspeccionan los handlers
- THEN ninguno SHALL tener `@UseGuards(JwtAuthGuard)`
- AND ninguno SHALL referenciar `@CurrentUser()` ni equivalentes

#### INV-35: Adapter aísla queries TypeORM

- GIVEN el archivo `password-recovery.adapter.ts`
- WHEN se inspecciona el contenido
- THEN SHALL contener todas las queries hacia `password_recovery_tokens`
- AND el `password-recovery.service.ts` NO SHALL contener `Repository<...>` directos sobre la entity (delega al adapter)
