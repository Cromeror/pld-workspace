# Delta for Users

> Esta delta agrega la columna `users.password_changed_at` (con backfill desde `created_at`) y un método reusable `UsersService.markPasswordChanged` que centraliza la actualización transaccional de hash + timestamp + flag de must_change. Refinement Jarvis thread `ffcf3339-7f80-4889-b60f-c221f3c30f1f` iter 4.

## ADDED Requirements

### Requirement: Columna users.password_changed_at

La entidad `UserEntity` MUST incluir la columna `password_changed_at DATETIME NULL` (RC-10). La migración TypeORM MUST agregar la columna y MUST backfillear todas las filas pre-existentes con `password_changed_at = users.created_at` en la misma transacción para evitar invalidar sesiones activas tras el deploy.

El down de la migración MUST hacer drop de la columna.

#### INV-1: Columna creada con nullable y default sin valor

- GIVEN la migración `add-password-changed-at-to-users` fue ejecutada
- WHEN se inspecciona `INFORMATION_SCHEMA.COLUMNS` para `users.password_changed_at`
- THEN SHALL existir la columna con tipo `DATETIME`
- AND `IS_NULLABLE` SHALL ser `YES`
- AND `COLUMN_DEFAULT` SHALL ser `NULL` (sin auto-update on row change)

#### INV-2: Backfill copia created_at a password_changed_at en la misma transacción

- GIVEN la migración up se ejecuta sobre una BD con N rows pre-existentes en `users`
- WHEN la migración completa sin errores
- THEN todas las rows pre-existentes SHALL tener `password_changed_at = created_at`
- AND ninguna row SHALL tener `password_changed_at = NULL` post-migración

#### INV-3: Down de la migración elimina la columna

- GIVEN la migración up fue ejecutada
- WHEN se ejecuta `down`
- THEN la columna `password_changed_at` NO SHALL existir en `users`

### Requirement: UsersService.markPasswordChanged reusable

`UsersService` MUST exponer el método `markPasswordChanged(userId: string, options: { newPasswordHash: string; mustChangePassword?: boolean }, qr?: QueryRunner): Promise<void>` (RC-16). El método MUST actualizar `users.password_hash`, `users.password_changed_at = NOW()` y, si `options.mustChangePassword === false`, también `users.must_change_password = false`. El método MUST aceptar un `QueryRunner` opcional para operar dentro de una transacción externa (consumido por el flujo de password-recovery confirm).

El método MUST ser usado por `POST /password-recovery/confirm` y MUST estar disponible para futuras rutas de cambio de password autenticado.

#### INV-4: markPasswordChanged actualiza los tres campos atómicamente

- GIVEN un usuario existente con `password_hash = "old"`, `password_changed_at = T0`, `must_change_password = true`
- WHEN se invoca `usersService.markPasswordChanged(userId, { newPasswordHash: "new", mustChangePassword: false })`
- THEN `users.password_hash` SHALL ser `"new"`
- AND `users.password_changed_at` SHALL ser ≈ NOW() (estrictamente posterior a T0)
- AND `users.must_change_password` SHALL ser `false`

#### INV-5: markPasswordChanged respeta QueryRunner externo

- GIVEN un `QueryRunner` con transacción abierta
- WHEN se invoca `markPasswordChanged(userId, opts, qr)` desde el caller
- AND el caller hace rollback antes del commit
- THEN `users.password_hash`, `users.password_changed_at` y `users.must_change_password` SHALL permanecer en sus valores previos al request

#### INV-6: markPasswordChanged sin mustChangePassword no toca el flag

- GIVEN un usuario con `must_change_password = true`
- WHEN se invoca `markPasswordChanged(userId, { newPasswordHash: "new" })` (sin la opción `mustChangePassword`)
- THEN `users.must_change_password` SHALL permanecer `true`
- AND `users.password_hash` y `users.password_changed_at` SHALL haber sido actualizados

#### INV-7: markPasswordChanged sobre user inexistente lanza error

- GIVEN un `userId` que no existe o está soft-deleted
- WHEN se invoca `markPasswordChanged(userId, opts)`
- THEN el método SHALL lanzar un error
- AND ninguna fila de `users` SHALL haber sido modificada
