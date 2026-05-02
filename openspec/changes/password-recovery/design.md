# Design: Password recovery end-to-end

**Status**: draft
**Created**: 2026-05-02
**Sub-repos**: pld-api + pld-web
**Refines**: [proposal.md](./proposal.md)
**Refinement source**: Jarvis thread `ffcf3339-7f80-4889-b60f-c221f3c30f1f` iter 4 (final)
**Flow reference**: [docs/FLUJO_LOGIN.md](../../../docs/FLUJO_LOGIN.md)
**UI mockups**: [docs/designs/password-recovery/](../../../docs/designs/password-recovery/)

## 1. Architecture decisions

### 1.1 Backend module layout (`pld-api`)

Feature module dedicado bajo `apps/auth-users/src/password-recovery/`, siguiendo el patrón **controller -> service -> adapter** ya consolidado en `auxiliaries/` y `admin/registration/`. La capa adapter aísla todas las queries TypeORM; el service no conoce el repositorio directamente.

```
pld-api/apps/auth-users/src/password-recovery/
├── password-recovery.module.ts          # Imports MailModule, UsersModule, TypeOrmModule.forFeature([PasswordRecoveryToken])
├── password-recovery.controller.ts      # 3 endpoints públicos + decorators @Throttle
├── password-recovery.service.ts         # request / verify / confirm + cron cleanup
├── password-recovery.adapter.ts         # queries TypeORM aisladas (insert, find by token, lock-and-update, cleanup)
├── entities/
│   └── password-recovery-token.entity.ts
└── dto/
    ├── request-recovery.dto.ts          # Zod 3.25 schema + DTO
    ├── verify-recovery-query.dto.ts
    └── confirm-recovery.dto.ts
```

**Decisión**: la entity vive bajo `apps/auth-users/src/password-recovery/entities/` (no en `packages/persistence/`) porque es exclusiva del módulo y nunca se consume desde otra app. Las migraciones sí van en `packages/persistence/migrations/` por convención del repo (TypeORM CLI single-source).

### 1.2 Mail layer (refactor preparatorio + service nuevo)

El package puro `@pld-api/mail` ya está mergeado y verificado e2e (incluye `MailGatewayClient`, `renderPasswordRecovery`, `validateMailConfig` con Zod 3.25). **Esta historia solo lo consume.** No se extrae nada del package.

Dos services intermedios viven en `apps/auth-users/src/mail/`, ambos `@Injectable()` con la misma forma:

| Service | Origen | Consumers |
|---|---|---|
| `CredentialsMailService` (RC-11b — refactor preparatorio) | Extracción de los `dispatchCredentialsEmail` privados duplicados en `AuxiliariesService` y `RegistrationService` | `AuxiliariesService`, `RegistrationService` |
| `PasswordRecoveryMailService` (RC-11) | Nuevo | `PasswordRecoveryService` |

Ambos consumen `MailGatewayClient` + `ConfigService`, encapsulan render de la plantilla y construcción de URLs absolutas a partir de `FRONTEND_LOGIN_URL_BASE`. `mail.module.ts` los provee y exporta.

**Orden ejecutable**: el refactor de `CredentialsMailService` va en commit aparte ANTES del módulo nuevo. No cambia comportamiento observable, deja la base limpia y permite hacer rollback granular.

### 1.3 Frontend module layout (`pld-web`)

```
pld-web/src/features/password-recovery/
├── routes.tsx                                # 4 rutas públicas
├── components/
│   ├── PasswordRecoveryLayout.tsx            # frame compartido (logo, copy, footer)
│   └── PasswordPolicyHints.tsx               # mismo shape que en login (mirror BE)
├── pages/
│   ├── RequestEmailPage.tsx                  # step-1
│   ├── EmailSentPage.tsx                     # step-2 (universal — RC-24)
│   ├── NewPasswordPage.tsx                   # step-3 (verify on mount + form)
│   └── ResultPage.tsx                        # éxito + token expirado/usado
├── schemas/
│   ├── request-email.schema.ts               # Zod 4
│   ├── new-password.schema.ts                # Zod 4 — espeja password-policy.ts
│   └── verify-token.schema.ts
├── hooks/
│   ├── useRequestRecovery.ts                 # React Query mutation
│   ├── useVerifyToken.ts                     # React Query suspense query
│   └── useConfirmRecovery.ts                 # React Query mutation
└── state/
    └── retryCounter.ts                       # contador in-component para RC-23
```

Servicio HTTP en `pld-web/src/services/password-recovery.ts`. Tipos en `pld-web/src/types/password-recovery.ts`. Interceptor 401 con código `password-changed` se modifica en `pld-web/src/services/api/interceptors.ts` (o el archivo que corresponda al cliente axios actual).

### 1.4 Por qué módulo dedicado y no extender `auth/` existente

`apps/auth-users/src/auth/` agrupa login, refresh, logout y la `JwtStrategy`. Meter recovery ahí mezcla dominios: recovery tiene su propia entity, su propia tabla, su propio cron, y throttling agresivo distinto al resto. Mantenerlo aislado preserva el `auth/` actual estable y permite testear el feature en aislamiento (RC-20).

## 2. Data model

### 2.1 Tabla `password_recovery_tokens` (RC-9)

Migration TypeORM `<ts>-create-password-recovery-tokens.ts` en `packages/persistence/migrations/`.

```sql
CREATE TABLE password_recovery_tokens (
  id           BIGINT       NOT NULL AUTO_INCREMENT,
  user_id      BIGINT       NOT NULL,
  token_hash   CHAR(64)     NOT NULL,           -- SHA-256 hex del token bruto
  expires_at   DATETIME(3)  NOT NULL,
  used_at      DATETIME(3)  NULL,
  created_at   DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  CONSTRAINT pk_password_recovery_tokens PRIMARY KEY (id),
  CONSTRAINT uq_password_recovery_tokens_token_hash UNIQUE (token_hash),
  CONSTRAINT fk_password_recovery_tokens_user_id
    FOREIGN KEY (user_id) REFERENCES users (id)
    ON DELETE CASCADE,
  INDEX ix_password_recovery_tokens_user_used_expires (user_id, used_at, expires_at),
  INDEX ix_password_recovery_tokens_expires_at (expires_at)
) ENGINE=InnoDB CHARSET=utf8mb4;
```

**Decisiones clave**:

- **`token_hash` solo (sin guardar el token bruto)**: mismo patrón que session tokens / API keys. Si la BD se filtra, el atacante no puede usar los tokens. El servicio recibe el token bruto del usuario, lo hashea y compara contra `token_hash`. SHA-256 hex (64 chars) es suficiente — no necesitamos bcrypt aquí porque el token tiene 256 bits de entropía y solo vive 30 minutos.
- **UNIQUE en `token_hash`**: defensa en profundidad contra colisiones (probabilidad astronómica con 256 bits) y permite lookup O(1).
- **INDEX compuesto `(user_id, used_at, expires_at)`**: cubre las 2 queries calientes:
  1. Confirm: invalidar todos los tokens previos del user → `WHERE user_id = ? AND used_at IS NULL`.
  2. Throttling por user (futuro): contar requests recientes por usuario.
- **INDEX en `expires_at`**: cron de cleanup hace `WHERE expires_at < ?`.
- **`ON DELETE CASCADE`**: si se borra el user, sus tokens se limpian automáticamente. Coherente con el resto del schema.
- **`DATETIME(3)`**: precisión milisegundo, alineado con `iat` del JWT (segundos en el JWT, ms en la DB → ver §4.3).

### 2.2 Columna nueva `users.password_changed_at` (RC-10)

Migration separada `<ts>-add-password-changed-at-to-users.ts`.

```sql
ALTER TABLE users
  ADD COLUMN password_changed_at DATETIME(3) NULL AFTER password_hash;

-- Backfill crítico para no invalidar sesiones activas en producción.
UPDATE users
   SET password_changed_at = created_at
 WHERE password_changed_at IS NULL;
```

**Decisión clave — backfill = `created_at`**: Cualquier JWT activo tiene `iat >= user.created_at` por construcción (fue emitido después de que el user se creó). Al backfillear `password_changed_at = created_at`, todos los JWTs activos pasan la comparación `iat * 1000 >= password_changed_at` y siguen siendo válidos. Si dejáramos `NULL` y la lógica fuera "rechazar si NULL", invalidaríamos todas las sesiones al desplegar. Si dejáramos `NULL` y la lógica fuera "tratar NULL como `0`", igual sirve, pero el backfill explícito es más auditable y deja la columna NOT NULL-friendly para una constraint futura.

**`down()`** de la migration solo `DROP COLUMN`; el backfill se pierde sin consecuencia (la columna deja de existir).

### 2.3 Entity TypeORM

```ts
@Entity('password_recovery_tokens')
export class PasswordRecoveryToken {
  @PrimaryGeneratedColumn({ type: 'bigint' })
  id: string;

  @Column({ name: 'user_id', type: 'bigint' })
  userId: string;

  @Column({ name: 'token_hash', type: 'char', length: 64, unique: true })
  tokenHash: string;

  @Column({ name: 'expires_at', type: 'datetime', precision: 3 })
  expiresAt: Date;

  @Column({ name: 'used_at', type: 'datetime', precision: 3, nullable: true })
  usedAt: Date | null;

  @CreateDateColumn({ name: 'created_at', type: 'datetime', precision: 3 })
  createdAt: Date;

  @ManyToOne(() => User, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user?: User;
}
```

## 3. Endpoint flow + transactions (RC-21)

### 3.1 `POST /auth/password-recovery/request`

**Body**: `{ email: string }`. Throttler: 5/15min IP + 3/h email. Response: 200 universal vacío.

**Flujo**:

1. Validar DTO (Zod). Mal-formado → 400.
2. Buscar user por email (lowercased + trimmed). **No bifurcar la respuesta** — el flujo continúa "as if" haya match siempre, con timing-equalization (§4.4).
3. Si match y user activo:
   - **Transacción** (1 sola, sin lock externo):
     ```sql
     INSERT INTO password_recovery_tokens (user_id, token_hash, expires_at) VALUES (?, ?, NOW() + INTERVAL 30 MINUTE);
     ```
   - Capturar el token bruto en una closure de scope local.
4. **POST-COMMIT** (fuera de la transacción): `setImmediate(() => mailService.sendRecoveryLink(...))` — fire-and-forget. Errores se loguean, no propagan.
5. **Siempre** retornar 200 vacío al cliente, en tiempo equivalente al caso "match" (§4.4).

**Justificación de la transacción separada**: dispatch del mail dentro de la transacción bloquearía el row lock 1-2s mientras el gateway responde. Si el gateway tiene latencia, eso degrada throughput general de la BD. Trade-off aceptado: si el server cae entre commit y `setImmediate`, el token queda huérfano hasta el cron — el usuario reintenta (RC-21).

**Sin lock**: insert simple, no race significativa. Si dos requests del mismo user llegan en paralelo, se generan 2 tokens, ambos válidos hasta que uno se use o el cron los borre. El segundo `confirm` invalidará al primero (§3.3).

### 3.2 `GET /auth/password-recovery/verify?token=...`

Throttler: 20/1min IP. Response: 200 `{ valid: true }` o 410 `{ valid: false, reason: 'expired' | 'used' | 'unknown' }`.

**Flujo**:

1. Validar query (Zod): token presente, formato base64url 43 chars (32 bytes).
2. Calcular `tokenHash = sha256(token)`.
3. **Read-only, sin transacción**:
   ```sql
   SELECT used_at, expires_at FROM password_recovery_tokens WHERE token_hash = ? LIMIT 1;
   ```
4. Decidir:
   - No row → 410 `{ valid: false, reason: 'unknown' }`.
   - `used_at IS NOT NULL` → 410 `{ valid: false, reason: 'used' }`.
   - `expires_at < NOW()` → 410 `{ valid: false, reason: 'expired' }`.
   - Else → 200 `{ valid: true }`.

**Sin transacción** (RC-21): solo lectura, idempotente, no muta estado. El throttler 20/1min IP mitiga enumeración masiva (RC-22) — con tokens de 256 bits la enumeración es matemáticamente imposible, pero el throttler también protege contra DoS.

**No exponemos `userId` ni nada del user** en la respuesta. La vista FE solo necesita saber "render form" vs "redirect a /expired".

### 3.3 `POST /auth/password-recovery/confirm`

**Body**: `{ token: string, newPassword: string }`. Throttler: 5/1min IP. Response: 200 vacío en éxito, 400 si policy fail, 410 si token inválido.

**Flujo (1 transacción atómica con `pessimistic_write`)**:

1. Validar DTO (Zod): token formato + `newPassword` cumple `validatePasswordStrength` (§4.5).
2. `tokenHash = sha256(token)`.
3. **`dataSource.transaction(async (qr) => { ... })`**:
   - **Lock**:
     ```sql
     SELECT id, user_id, used_at, expires_at
       FROM password_recovery_tokens
      WHERE token_hash = ?
      FOR UPDATE;
     ```
     (TypeORM: `qr.manager.findOne(PasswordRecoveryToken, { where: { tokenHash }, lock: { mode: 'pessimistic_write' } })`).
   - Validar fuera-de-rango → 410 con reason. La transacción aborta sin escritura.
   - `UsersService.markPasswordChanged(userId, qr)`:
     ```sql
     UPDATE users
        SET password_hash = ?,
            password_changed_at = ?,
            must_change_password = FALSE
      WHERE id = ?;
     ```
     Mismo timestamp `now` se usa abajo.
   - Marcar token usado:
     ```sql
     UPDATE password_recovery_tokens SET used_at = ? WHERE id = ?;
     ```
   - Invalidar tokens previos del mismo user:
     ```sql
     UPDATE password_recovery_tokens
        SET used_at = ?
      WHERE user_id = ? AND used_at IS NULL AND id <> ?;
     ```
4. Commit. Response 200.

**Justificación de la transacción + lock (RC-21)**:

- **Sin atomicidad**: si el server cae entre `markPasswordChanged` e `marcar usado`, el token sigue válido aunque la password ya cambió → reuso del token = recambiar password sin autorización.
- **Sin `pessimistic_write`**: dos requests concurrentes con el mismo token leen `used_at IS NULL`, ambos hashean la nueva password y ambos llaman `markPasswordChanged`. La password termina con el último valor escrito (no determinístico) y ambos tokens marcan usado a la vez. Lock previene esto serializando.
- **Invalidación de tokens previos en el mismo commit**: si fuera fuera de la transacción y caemos en el medio, queda un token activo con la password vieja todavía cambiable.

`UsersService.markPasswordChanged(userId, qr?)` (RC-16) acepta opcionalmente el QueryRunner para participar en una transacción externa. Reusable por futuras rutas de cambio de password autenticado.

## 4. Security

### 4.1 Token shape

- **Generación**: `crypto.randomBytes(32).toString('base64url')` → 43 caracteres URL-safe, 256 bits de entropía.
- **Storage**: solo el `sha256(token)` en hex (64 chars). El bruto no toca disco.
- **Vida útil**: 30 minutos (configurable en service constant `RECOVERY_TOKEN_TTL_MINUTES = 30`).
- **Transporte**: query string (`?token=...`) en el link del email. Aceptable porque (a) HTTPS cifra la URL en transit, (b) los logs de FE/BE no deben loguear query params en endpoints de auth, (c) referer no aplica porque la URL es nuestra propia app.

**Por qué no UUID v4** (cambio respecto al proposal): UUID v4 son 122 bits de entropía y un schema "GUESS-able" (formato conocido). 256 bits + base64url cuesta lo mismo de generar y eleva el bar. La proposal mencionaba `CHAR(36) UK` para UUID v4 — el design lo refina a `CHAR(64)` para el hash. Si el equipo prefiere UUID v4, se puede revertir sin tocar nada del flujo (es solo el shape del bruto y el ancho de la columna).

### 4.2 Throttler in-memory (RC-22)

`@nestjs/throttler` con storage default (in-memory, por proceso). Configuración global vía `ThrottlerModule.forRoot()` con un único TTL/limit base, y override por endpoint vía `@Throttle({ default: { limit: N, ttl: T } })`.

| Endpoint | IP-based | Email-based / extra |
|---|---|---|
| `POST /request` | 5 / 15 min | 3 / hora por email (clave: `email:<lowercased>`) |
| `GET /verify` | 20 / 1 min | — |
| `POST /confirm` | 5 / 1 min | — |

**Email-based throttle en request**: implementado via `ThrottlerGuard` custom que combina IP + email del body. Mitiga el caso "atacante rota IPs (proxy pool) pero ataca al mismo email para flood de inbox". Si el throttle por email se hits, igual respondemos 200 (universal — §4.4) pero NO insertamos token ni mandamos mail.

**Trade-off documentado (RC-17)**: storage in-memory NO escala horizontalmente. Cada réplica cuenta separado. Aceptado para v1 (1 réplica). Migración a Redis fuera de scope; queda agendado en risks de la proposal.

### 4.3 JWT invalidation

`JwtStrategy.validate(payload)` en `apps/auth-users/src/shared/auth/jwt.strategy.ts` (única copia tras la eliminación de `apps/cross` el 2026-05-02):

```ts
async validate(payload: JwtPayload) {
  const user = await this.usersService.findById(payload.sub);
  if (!user) throw new UnauthorizedException();

  // payload.iat es Unix seconds; password_changed_at es ms-precision Date.
  const iatMs = payload.iat * 1000;
  if (user.passwordChangedAt && iatMs < user.passwordChangedAt.getTime()) {
    throw new UnauthorizedException({ code: 'password-changed' });
  }
  // ... resto de la validación existente
}
```

**Shape del error** alineado con el formato uniforme del repo:
```json
{ "statusCode": 401, "code": "password-changed", "message": "..." }
```

**Por qué timestamp y no denylist**: denylist de JTIs requiere un store (Redis/DB), gestión de TTL, y polling en cada request. La columna `password_changed_at` es O(1) en el SELECT del user que ya hacemos en `validate()`, no requiere infra extra, y es semánticamente clara (auditable: "cuándo cambió esta password").

**Coordinación con `iat`**: `iat` viene en segundos (estándar JWT). `password_changed_at` se guarda con precisión ms. Multiplicamos `iat * 1000` para comparar. **Caso borde**: si un user cambia su password y obtiene un nuevo JWT en el mismo segundo, `iat * 1000 == password_changed_at` (ambos truncan al segundo en el JWT). Por eso usamos `<` estricto, no `<=` — el JWT del propio confirm pasa.

### 4.4 Anti-enumeration (request endpoint)

**Response universal**: 200 vacío para todos los outcomes (email registrado, no registrado, inactivo, throttled, error de gateway). El cliente no puede distinguir.

**Timing-equalization**: el path "no match" debe consumir ~el mismo tiempo que el path "match" para no filtrar info por side-channel. Implementación:

```ts
async request(email: string) {
  const t0 = Date.now();
  try {
    const user = await this.adapter.findActiveUserByEmail(email);
    if (user) {
      const token = this.generateToken();
      await this.adapter.insertToken(user.id, sha256(token), this.expiresAt());
      this.dispatchMailPostCommit(user, token);
    }
  } catch (err) {
    this.logger.error(err);
  } finally {
    await this.padToBaseline(t0, RECOVERY_REQUEST_BASELINE_MS); // ~150ms
  }
}
```

`padToBaseline` espera hasta cumplir el baseline si el request tardó menos. El baseline se calibra observando p95 del path con match en dev (~120ms espera). No es defensa perfecta (atacante con muchas muestras puede inferir la varianza), pero eleva el costo significativamente.

**No exponer `userId` en NINGUNA respuesta**. Ni en `verify`, ni en `confirm`, ni en `request`. El FE no necesita el userId — el token amarra al user en el BE.

### 4.5 Password policy compartida

Archivo único de verdad: `pld-api/packages/domain-auth-users/src/crypto/password-policy.ts`.

```ts
export const PASSWORD_POLICY = {
  minLength: 12,
  maxLength: 72, // bcrypt hard limit
  requireLetter: true,
  requireDigit: true,
  requireSpecial: true,
  specialChars: '.@$!%*?&',
} as const;

export const PASSWORD_POLICY_MESSAGES = {
  tooShort: `La contraseña debe tener al menos ${PASSWORD_POLICY.minLength} caracteres.`,
  tooLong: `La contraseña no puede superar los ${PASSWORD_POLICY.maxLength} caracteres.`,
  missingLetter: 'La contraseña debe incluir al menos una letra.',
  missingDigit: 'La contraseña debe incluir al menos un número.',
  missingSpecial: `La contraseña debe incluir al menos un carácter especial (${PASSWORD_POLICY.specialChars}).`,
} as const;

export function validatePasswordStrength(plain: string): { valid: true } | { valid: false; reason: keyof typeof PASSWORD_POLICY_MESSAGES } {
  // ... check minLength, maxLength, regexes
}
```

**Backend (Zod 3.25)** — `confirm-recovery.dto.ts`:
```ts
import { PASSWORD_POLICY, PASSWORD_POLICY_MESSAGES, validatePasswordStrength } from '@pld-api/domain-auth-users';
import { z } from 'zod';

export const confirmRecoverySchema = z.object({
  token: z.string().min(43).max(43),
  newPassword: z.string()
    .min(PASSWORD_POLICY.minLength, PASSWORD_POLICY_MESSAGES.tooShort)
    .max(PASSWORD_POLICY.maxLength, PASSWORD_POLICY_MESSAGES.tooLong)
    .refine(v => validatePasswordStrength(v).valid === true, { message: 'La contraseña no cumple la política.' }),
});
```

**Frontend (Zod 4)** — `pld-web/src/features/password-recovery/schemas/new-password.schema.ts` reescribe la misma policy. Las constantes se duplican (no se importan del package — el FE no consume `pld-api/packages/`). Riesgo de drift mitigado por sdd-verify (compara los valores de las constantes en ambos lados).

**Por qué min 12 y no 8** (refinamiento del proposal): la proposal menciona "min 8". El design eleva a 12 por buenas prácticas modernas (NIST SP 800-63B sugiere 8 para regulado, pero PLD/AML maneja datos sensibles y 12 con required-classes es estándar bancario). El refinamiento iter 4 confirmó "12+ chars, 1 letra, 1 número, 1 especial de `.@$!%*?&`".

**Sin historial de passwords** (RC-15 iter 3 confirmado iter 4): no se compara contra passwords anteriores. Trade-off aceptado: simplicidad > defensa contra reuso. Si el equipo lo pide después, va en historia separada.

## 5. Frontend

### 5.1 Routing

Rutas públicas (sin auth guard) en `pld-web/src/router/`:

| Path | Componente | Propósito |
|---|---|---|
| `/recuperar-contrasena/solicitar` | `RequestEmailPage` | Step-1: form de email |
| `/recuperar-contrasena/enviado` | `EmailSentPage` | Step-2 universal (RC-24) |
| `/recuperar-contrasena?token=...` | `NewPasswordPage` | Step-3: verify + form |
| `/recuperar-contrasena/exito` | `ResultPage` (success variant) | Confirmación final |

**Decisión sobre el path en español**: el repo `pld-web` usa rutas en español (`/iniciar-sesion`, `/registro`). Mantenemos la convención. La proposal mencionaba `/recover-password/*` (inglés) — se ajusta al lenguaje del producto.

**Token en query string del path raíz** (`/recuperar-contrasena?token=...`) en lugar de subpath (`/recuperar-contrasena/reset?token=...`) para que el link del email sea más corto y no exponga el subpath "reset". Step-3 detecta `token` en la query y monta el flujo de verify+form. Sin `token` en la URL, redirige a step-1.

**Sin ruta separada para "expirado"**: el step-3 maneja inline el caso de token inválido — muestra un error amigable con CTA "Solicitar un nuevo enlace" que lleva a step-1. Una ruta dedicada `/recuperar-contrasena/expirado` es UX innecesaria (estado deep-link que el usuario nunca debería bookmarkear).

### 5.2 Step-1: Request email

`RequestEmailPage`:

1. Form RHF + Zod 4: campo email (required, formato).
2. Submit → `useRequestRecovery.mutateAsync({ email })`.
3. **`onSettled` (RC-24)**: navega a `/recuperar-contrasena/enviado` SIEMPRE — éxito, error 4xx, error 5xx, network error. Sin importar la respuesta.
4. Si el throttler responde 429, igual navegamos a step-2 (universal).

**Por qué `onSettled` y no `onSuccess`**: `onSuccess` solo se dispara con 2xx. `onSettled` con cualquier outcome. RC-24 explícito: el step-2 es universal para TODOS los outcomes.

### 5.3 Step-2: Email sent (universal)

`EmailSentPage`:

- Copy "Revisa tu Bandeja de Entrada" + ilustración.
- Botón único "Volver a Iniciar Sesión" → `/iniciar-sesion`.
- **Sin reintento**, sin "no llegó el correo, reenviar". Si el usuario quiere reintentar, vuelve manualmente a step-1.
- Sin estado, sin params: es una página puramente presentacional. Si el usuario llega aquí por deep-link directo, igual funciona — es un dead-end visual.

### 5.4 Step-3: New password

`NewPasswordPage`:

1. **Verify on mount**: lee `token` de la query. Si falta → `<Navigate to="/recuperar-contrasena/solicitar" replace />`.
2. `useVerifyToken({ token })` (React Query suspense):
   - 200 `{ valid: true }` → renderiza form.
   - 410 → renderiza inline error "Tu enlace expiró o ya fue usado" + CTA "Solicitar un nuevo enlace" que navega a step-1.
   - Error de red → renderiza el mismo error inline (no diferenciamos para no revelar nada).
3. Form RHF + Zod 4 (`new-password.schema.ts`):
   - `newPassword`: aplica `validatePasswordStrength` mirror.
   - `confirmPassword`: `.refine(v => v === values.newPassword)`.
4. Submit:
   - Si Zod falla en "no coinciden" → incrementar `retryCounter`. Si `retryCounter >= 3`, redirigir a `/iniciar-sesion` con toast "Demasiados intentos, vuelve a solicitar el enlace" (RC-23).
   - Si Zod pasa → `useConfirmRecovery.mutateAsync({ token, newPassword })`.
   - Si BE responde 200 → navegar a `/recuperar-contrasena/exito`.
   - Si BE responde 410 → render error inline (token expiró durante el form fill) + CTA "Solicitar nuevo enlace".
   - Si BE responde 400 (policy) → mostrar errores de campo desde `error.response.data.errors`.

**Contador in-component**: vive en un `useRef` o `useState` local. Se resetea al desmontar (cambio de página o reload). NO persiste en localStorage — el límite es por sesión visual, no por usuario.

**Por qué 3** (RC-23 iter 4 — pendiente IF de validar): equilibrio entre fricción (1 sería frustrante con typos) y defensa (10 sería trivial brute-force visual). Configurable como constante `RETRY_LIMIT = 3`.

### 5.5 Step-4: Result page (éxito)

`ResultPage` con variante `success`:

- Copy "Contraseña actualizada" + ilustración.
- CTA "Iniciar sesión" → `/iniciar-sesion`.
- Sin estado.

### 5.6 Cableado en login

`pld-web/src/features/login/components/LoginForm.tsx`: el link "¿Olvidaste tu contraseña?" pasa de placeholder/`#` a `<Link to="/recuperar-contrasena/solicitar">`. Sin cambio en el resto del form.

### 5.7 Interceptor 401 con código `password-changed` (RC-10)

Modifica `pld-web/src/services/api/interceptors.ts` (o el archivo del cliente axios — confirmar al implementar):

```ts
client.interceptors.response.use(
  resp => resp,
  err => {
    if (err.response?.status === 401 && err.response?.data?.code === 'password-changed') {
      authStore.getState().logout(); // limpia tokens locales
      navigate('/iniciar-sesion', {
        state: { toast: { kind: 'info', message: 'Tu contraseña fue cambiada. Volvé a iniciar sesión.' } },
      });
      return Promise.reject(err);
    }
    return Promise.reject(err);
  },
);
```

**Sin toast genérico**: el toast es específico ("contraseña cambiada"), distinto al "sesión expirada" del 401 normal.

**Trade-off**: si el flujo de confirm en la misma tab dispara este interceptor por algún re-fetch durante el form, lo manejamos comprobando que la URL actual NO sea `/recuperar-contrasena/*` antes de redirigir. Edge case raro pero documentado.

## 6. Cron cleanup (RC-15)

Implementado como método del `PasswordRecoveryService` decorado con `@Cron(CronExpression.EVERY_DAY_AT_3AM)` de `@nestjs/schedule`:

```ts
@Cron(CronExpression.EVERY_DAY_AT_3AM, { name: 'password-recovery-cleanup' })
async cleanupExpiredTokens() {
  const cutoff = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000); // -7 días
  const result = await this.adapter.deleteExpiredBefore(cutoff);
  this.logger.log(`Cleaned up ${result.affected ?? 0} expired recovery tokens`);
}
```

**Query** (en el adapter):
```sql
DELETE FROM password_recovery_tokens
 WHERE expires_at < ?;
```

**Decisión: criterio único de borrado por `expires_at`** (refinamiento del proposal): el proposal mencionaba dos criterios (`expires_at < NOW()-1d` y `used_at IS NOT NULL AND used_at < NOW()-7d`). El design lo simplifica: un token usado expira a los pocos minutos de ser usado de hecho (`expires_at` original sigue corriendo), así que después de 7 días `expires_at < NOW() - 7d` cubre AMBOS casos: tokens vencidos sin usar (más viejos que 7 días) y tokens usados (también más viejos que 7 días desde su `expires_at` original).

**Ventana conservadora**: 7 días permite auditar reintentos cercanos sin acumular indefinidamente.

**`ScheduleModule.forRoot()`** se importa una vez en `app.module.ts` (raíz de auth-users). El cron solo corre en la app `auth-users`, NO en `cross` (cross no carga este módulo).

## 7. Trade-offs explícitos

| Decisión | Trade-off | Aceptado porque |
|---|---|---|
| Throttler in-memory (`@nestjs/throttler` default storage) | NO escala horizontal — cada réplica cuenta separado, atacante con N réplicas tiene N veces el límite | v1 corre 1 réplica. Migración a Redis es trivial cuando escalemos (sumar `@nestjs/throttler-storage-redis`). Documentado en risks. |
| Sin queue / Redis para mail dispatch | Si el gateway está down al momento del fire-and-forget, el mail se pierde silenciosamente. El token queda huérfano hasta el cron de cleanup | Frecuencia esperada baja (gateway tiene su propia retry policy interna). Usuario reintenta con un nuevo request (penalty: tener que volver a step-1, aceptable). Step-2 universal evita revelar el problema. |
| Sin historial de passwords | Usuario puede reusar la password anterior | RC-15 iter 3+4 confirmado: scope reducido. Si auditoría lo pide después, va en historia separada con su propia tabla `password_history`. |
| Token bruto en query string (no en path o body POST) | Aparece en logs de proxy/CDN si esos no están bien configurados | HTTPS cifra la URL en transit. Logs de FE/BE en endpoints de auth NO loguean query params (revisar config en deploy). Trade-off vs UX (link clickable en email). |
| `password_changed_at` solo en user row, sin denylist de JTIs | No podemos invalidar un JWT específico sin invalidar todos los JWTs anteriores del user | Para recovery es exactamente lo que queremos: cambiar password = invalidar todo lo previo. Para casos futuros (logout específico de un device) habrá que sumar denylist. Documentado. |
| Min length 12 (vs 8 mencionado en proposal) | Más fricción para usuarios | PLD/AML estándar bancario. Refinamiento iter 4 confirmado. |
| Schemas Zod duplicados BE (Zod 3) y FE (Zod 4) | Posible drift al cambiar policy | sdd-verify chequea match BE↔FE de constantes. Si en el futuro Zod 4 está estable en BE también, se puede unificar via package `@pld-api/domain-auth-users` consumido por FE (por ahora FE no consume de pld-api). |

## 8. Testing strategy

### 8.1 Unit tests (backend)

**Package mail** (`@pld-api/mail`): ya cubierto. Esta historia no agrega tests del package.

**Feature backend** (`apps/auth-users/src/password-recovery/__tests__/`):

- `password-recovery.service.spec.ts`:
  - `request`: match → inserta token + dispatcha mail; no match → no inserta + dispatcha; usuario inactivo → no inserta + no dispatcha; throttle email → no inserta. Ambos paths consumen ~el baseline (timing-equalization).
  - `verify`: válido → 200; expirado / usado / unknown → 410 con reason correspondiente.
  - `confirm`: happy path → password update + token usado + tokens previos invalidados, todo en 1 commit. Lock contention simulado con 2 transacciones concurrentes (jest fake transactions o real test DB) → segunda falla con 410.
  - Cron `cleanupExpiredTokens`: borra solo tokens > 7 días.
- `password-recovery.adapter.spec.ts`: queries básicas con test DB.
- `users.service.spec.ts`: nuevo método `markPasswordChanged` con y sin `qr` externo.
- `jwt.strategy.spec.ts` (auth-users + cross): JWT con `iat * 1000 < password_changed_at` → 401 código `password-changed`; `iat * 1000 >= password_changed_at` → pass.

### 8.2 E2E backend

`apps/auth-users-e2e/src/password-recovery.e2e-spec.ts` con stub de `MailGatewayClient`:

- Happy path completo: `request` → captura token del stub → `verify` → `confirm` → login con nueva password OK.
- `request` con email no registrado: 200 idéntico al happy path; stub no recibe call.
- Throttle: 6° request en <15min desde misma IP → 429.
- Throttle email: 4° request en <1h al mismo email → 200 universal pero stub recibe solo 3 calls.
- `verify` con token tampered: 410 unknown.
- `confirm` con password débil: 400 con error de policy.
- `confirm` con token usado: 410 used.
- Race en `confirm`: 2 requests paralelos con mismo token → 1 OK, 1 410.
- JWT post-confirm: token viejo → 401 código `password-changed`; token nuevo (emitido por el login post-confirm) → 200.

### 8.3 Unit tests (frontend)

`pld-web/src/features/password-recovery/__tests__/`:

- `RequestEmailPage.test.tsx`: submit con éxito, error 5xx, throttle 429 — los 3 navegan a `/recuperar-contrasena/enviado`.
- `NewPasswordPage.test.tsx`: verify on mount válido renderiza form; inválido renderiza error inline; 3 fallos consecutivos de "no coinciden" → navega a `/iniciar-sesion` con toast.
- `interceptors.test.ts`: 401 código `password-changed` → logout + navigate + toast; 401 sin código → comportamiento previo.

### 8.4 Smoke manual con Playwright (RC-4 iter 4 confirmado)

Al final de `sdd-apply`, ejecutar smoke a 1500px de viewport (preferencia global):

1. Login con user existente.
2. Logout, click en "¿Olvidaste tu contraseña?".
3. Submit email → step-2.
4. Recuperar el token del email stub (en dev, dump del último `mail.send` call al log) o de la DB directamente (`SELECT token_hash, ...`).
5. Construir URL con token y abrirla en Playwright.
6. Form de nueva password → submit.
7. Verificar redirect a `/recuperar-contrasena/exito`.
8. Login con la password nueva → OK.
9. Verificar que el JWT viejo (capturado en paso 1) ya no funciona (request a endpoint protegido → 401 código `password-changed`).
10. Capturas evidence en `docs/designs/password-recovery/evidence/` con timestamps.

## 9. Open questions / pendientes (no bloquean implementación)

- **IF-1**: `FRONTEND_LOGIN_URL` por ambiente. Hoy `.env.example` lo provee — confirmar que dev/staging/prod tienen valores correctos antes de deploy.
- **IF-2**: credenciales del gateway de mail por ambiente. Idem.
- **RC-23 número de reintentos**: propuesto 3, validar con producto antes del smoke.
- **Política de logging**: confirmar que el infrastructure-level (proxy, CDN) NO loguea query strings en endpoints de auth. Revisar al deploy.

## 10. Implementation order (referencia para sdd-tasks)

1. Refactor preparatorio: extraer `CredentialsMailService` (commit aparte, no rompe nada).
2. Migrations: tabla `password_recovery_tokens` + columna `users.password_changed_at` con backfill.
3. Password policy compartida en `packages/domain-auth-users`.
4. `UsersService.markPasswordChanged`.
5. Backend feature module (entity + DTOs + adapter + service + controller + module).
6. `PasswordRecoveryMailService` en `apps/auth-users/src/mail/`.
7. Throttler + Schedule modules en `app.module.ts`.
8. JWT strategy update (auth-users + cross).
9. Tests backend (unit + e2e con stub).
10. Frontend: routes, pages, hooks, schemas, interceptor.
11. Tests frontend.
12. Smoke Playwright a 1500px + evidence.
