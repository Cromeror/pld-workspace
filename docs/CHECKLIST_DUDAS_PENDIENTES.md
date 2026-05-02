# Checklist de dudas y decisiones pendientes

**Última actualización**: 2026-04-21 (tras cierre Fase 4.1)
**Propósito**: consolidar todas las decisiones abiertas que surgieron durante las conversaciones y el plan de reorganización. Se resuelven una a una.

---

## A) Homogenización de tipos en `shared-types` (surgió hoy)

- [ ] **A.1** Mover enums `BCRespuesta` y `PMBeneficiarioControlTipoControl` de `participants.ts` → `enums.ts` (están mal ubicados — son enums dentro de un archivo de tipos).
- [ ] **A.2** Reemplazar el `Domicilio` inline dentro de `PFParticipanteRepresentanteDomicilio` por el `Domicilio` canónico de `person.ts` (duplicado estructural).
- [ ] **A.3** Reemplazar el `Contacto` inline dentro del mismo tipo por el `Contacto` canónico de `person.ts`.
- [ ] **A.4** Decidir estilo de composición: mixins `Con*` (ConNombre, ConNacimiento, ConDomicilio...) vs interfaces planas. Hoy coexisten ambos. ¿Cuál dejamos como canónico?
- [ ] **A.5** Separar `PerfilBasico` (hoy mezcla identidad + participante): ¿vivir en `domain-auth-users` el aspecto identidad (passwordHash, role, activo) y en `shared-types` solo el aspecto participante?
- [ ] **A.6** `LoginRequest`, `LoginResult`, `PasswordRecoveryRequest` viven en `libs/auth-profiles` — son DTOs de auth. ¿Mover a `domain-auth-users` como parte de su contrato?

---

## B) Libs puente y código muerto

- [ ] **B.1** `libs/core` — hoy reexporta desde `@pld-api/shared-types` para compatibilidad. 6 libs aún importan de `@pld-api/core` (anexos, risk, operacion, auth-profiles, reports, users ya eliminado). ¿Migrar los 6 a `@pld-api/shared-types` y eliminar `libs/core`?
- [ ] **B.2** `libs/core/src/interceptors/http.error.interceptor.ts` — sigue vivo, ¿alguien aún lo consume o ya todos usan la copia local? Verificar.
- [ ] **B.3** `libs/core/src/services/mail.service.ts` — ¿se usa? ¿destino final?
- [ ] **B.4** `libs/core/src/services/token.service.ts` — ¿se usa? ¿se consolida con `domain-auth-users/jwt`?
- [ ] **B.5** `libs/core/src/lib/core.service.ts` y `core.module.ts` — ¿tienen lógica o son scaffolding vacío?
- [ ] **B.6** `libs/address-contact-id` — solo re-exporta `Domicilio`, `Contacto`, `Identificacion` desde `shared-types`. Consumidores: 1 archivo. ¿Migrar ese consumidor y eliminar la lib?
- [ ] **B.7** `libs/catalogs` — solo re-exporta enums desde `shared-types`. Consumidores: 6 archivos. ¿Migrar y eliminar?
- [ ] **B.8** `libs/anexos`, `libs/auth-profiles`, `libs/operacion`, `libs/risk`, `libs/reports` — solo tienen types; 0 consumidores en apps. ¿Mantener hasta que se cree el `domain-*` correspondiente, o mover los types ya a `shared-types/` ahora?

---

## C) Fase 4.2 — `domain-participants`

- [ ] **C.1** Scope: ¿un solo paquete o dividir en sub-paquetes (`domain-pf`, `domain-pm`, `domain-fideicomiso`, `domain-anexo-7`)?
- [ ] **C.2** `libs/participants/` tiene 28+ entities TypeORM. ¿Migración literal o refactor de entities durante la migración?
- [ ] **C.3** FKs cruzadas entre entities (PF → BC, PM → PEP, etc.) — ¿se preservan literales o se rediseñan?
- [x] **C.4** Absorción de `apps/cross`: **resuelto 2026-05-02** — la app `cross` fue eliminada (no había consumidores). El lib `@pld-api/participants/beneficiario-controlador` se conserva por si el endpoint `POST /crear-beneficiario` se reactiva desde `auth-users`.
- [ ] **C.5** Los 4 controllers actuales de participantes (fisica, moral, fideicomiso, anexo-7) en `apps/auth-users/` — ¿refactor a ports + factory igual que `domain-auth-users`?

---

## D) Fase 3 — `domain-audit` (BLOQUEADA aún)

- [ ] **D.1** Alcance de auditoría — decisión pendiente desde antes: A (solo cambios de estado) / B (A + auth actions) / C (B + sensitive reads). Sigue sin resolverse.
- [ ] **D.2** Destino del log: tabla TypeORM / archivo / evento async al broker. ¿Cuál?
- [ ] **D.3** `EventPublisher` interface ya existe en `@pld-api/contracts` (NoopEventPublisher). ¿Seguir con ese patrón o otra cosa?

---

## E) Fase 5 — Cleanup y hardening

- [ ] **E.1** Upgrade Nx 14 → 22 (warning de peer deps desde Fase 0). ¿Hacerlo ya o al final?
- [ ] **E.2** Remover `accessToken` de Nx Cloud commiteado en [pld-api/nx.json:17](../pld-api/nx.json#L17) (secreto expuesto). **Urgente, no depende del plan**.
- [ ] **E.3** Simplificar prefix global: `/pld-api/auth-users` → `/pld-api`. Requiere coordinación con frontend.
- [ ] **E.4** Renombrar `apps/auth-users` → `apps/gateway` (opcional). Afecta docker, scripts, no rutas HTTP.
- [ ] **E.5** Setup de tests real: remover `passWithNoTests: true`, diseñar estrategia de tests de comportamiento (no tautológicos) para crypto + JWT + adapters de `domain-auth-users`.
- [ ] **E.6** Actualizar README con estructura real del monorepo post-reorganización.
- [x] **E.7** ~~Apagar definitivamente `apps/cross` si se absorbe~~ — **eliminado 2026-05-02** (no había consumidores; lib `@pld-api/participants/beneficiario-controlador` conservado).
- [ ] **E.8** Eliminar `apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts` del `include` explícito en `tsconfig.app.json` si ya no es necesario (podría ser otra deuda npm-era).

---

## F) Bugs conocidos pendientes de `apps/auth-users`

Documentados en [DIAGNOSTICO_SERVICIO_USUARIOS.md](DIAGNOSTICO_SERVICIO_USUARIOS.md):

- [x] **F.1** DTOs cruzados en `users.adapter.ts` → RESUELTO por la migración a `domain-auth-users` (2026-04-21). Ambos endpoints reciben el DTO correcto. Verificado con curl.
- [ ] **F.1-bis** Los DTOs `CreateUserNotarioInmobiliarioDto` y `CreateUserInternoExternoDto` aceptan los 6 roles con `@IsEnum(UserRole)`. Un `POST /system-users/notario-inmobiliario` con `role: USUARIO_INTERNO` devuelve 201. Fix sugerido: sustituir `@IsEnum(UserRole)` por `@IsIn([UserRole.NOTARIO, UserRole.INMOBILIARIA])` en el primer DTO y `@IsIn([USUARIO_INTERNO, USUARIO_EXTERNO, AUXILIAR])` en el segundo.
- [ ] **F.2** Regex de RFC con `&amp;` HTML-escapado en [pld-api/apps/auth-users/src/users/dto/create-user.dto.ts:62](../pld-api/apps/auth-users/src/users/dto/create-user.dto.ts#L62). Confirmado 2026-04-21: la clase `[A-ZÑ&amp;]` acepta literalmente `a`/`m`/`p`/`;` como válidos. RFCs obviamente inválidos (`XXX`) se rechazan; RFCs con minúsculas (`amp010101000`) pasan indebidamente. Fix: reemplazar `&amp;` por `&` en la regex. Cambio de 1 carácter, aislado.
- [ ] **F.3** `ValidationPipe` con `{ whitelist: false, forbidNonWhitelisted: true }` en [pld-api/apps/auth-users/src/main.ts:42](../pld-api/apps/auth-users/src/main.ts#L42). Confirmado 2026-04-21: `forbidNonWhitelisted` es inefectivo sin `whitelist: true`. Request con campos extra (`hackField`, `injected`) devuelve 201 en lugar de 400. Falso sentido de seguridad. Fix: cambiar `whitelist: false` por `whitelist: true`. Riesgo: si algún cliente envía campos adicionales hoy ignorados, empezará a recibir 400.
- [ ] **F.4** `lastLoginAt` nunca se actualiza en el flujo de login. Confirmado 2026-04-21: bug heredado literal en [pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts:16-23](../pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts#L16-L23). Tras múltiples logins exitosos, `SELECT last_login_at FROM users WHERE email='admin@pld.com'` sigue devolviendo NULL. Fix sugerido (opción A): agregar `await this.usersPort.updateUser(user.id, { lastLoginAt: new Date() })` antes de emitir el token. 1 línea. Evaluar si debe ir en try/catch silencioso para que un fallo del UPDATE no bloquee el login.

---

## G) Decisiones de arquitectura abiertas

- [ ] **G.1** `UserEntity` hoy vive en `packages/domain-auth-users`. ¿Las entidades de otros dominios también irán a su paquete (`domain-participants`), o se consolidan en un solo lugar?
- [ ] **G.2** Factory pattern de `domain-auth-users` → ¿se replica literal para `domain-participants`? Implica `createParticipantsDomain({ ds }): { pfPort, pmPort, fideicomisoPort, anexo7Port, bcPort }`.
- [ ] **G.3** `TypeOrmModule.forFeature([UserEntity])` tuvo que agregarse en el gateway para que el DataSource reconociera la entity. ¿Cómo se hará para `domain-participants` con 28 entities? ¿Exportar un array `ALL_ENTITIES` desde el paquete?
- [ ] **G.4** JWT — `JwtAuthGuard` y `JwtStrategy` se duplicaron en `apps/auth-users/src/shared/auth/`. Si aparece `apps/gateway` o `apps/worker`, ¿también se duplica o se extrae a un paquete NestJS-aware `packages/nest-auth-adapters`?

---

## H) Infraestructura y operaciones

- [ ] **H.1** Docker: `docker-compose.yml` de prod tiene todo comentado. ¿Completar o eliminar el archivo?
- [ ] **H.2** Permisos `dist/` y `node_modules/.cache/nx/` siguen quedando como root tras `docker compose`. ¿Solución definitiva? (user en el Dockerfile.dev).
- [ ] **H.3** MySQL → ¿migraciones formales con TypeORM migrations, o seguimos con `query.sql` al volumen?
- [ ] **H.4** ¿Plan de entorno de staging antes de deploy a prod?

---

## I) Documentación

- [ ] **I.1** Cada paquete en `packages/` necesita README propio que explique: propósito, interfaces públicas, cómo consumirlo, ejemplo mínimo.
- [x] **I.2** ~~El plan de reorganización actual vive en `temp-docs/`. ¿Se mueve a `docs/` una vez terminado, o se reemplaza por ADRs?~~ Resuelto 2026-04-24: al consolidar el workspace, `pld-api/temp-docs/` se movió a `pld/docs/`. La regla "docs van a temp-docs" del sub-repo BE quedó anulada. ADRs queda para decisión futura.
- [ ] **I.3** `openspec/specs/` es la fuente de verdad de specs. ¿Se publica algo user-facing desde ahí (ej. OpenAPI generado, docs de contrato)?

---

## Método de resolución

Resolver en orden sugerido: **E.2 (seguridad)** → **A.1-A.3 (quick wins homogenización)** → **F (bugs)** → **B (libs puente)** → **C (Fase 4.2)** → **D (Fase 3 audit)** → **E (resto Fase 5)** → **G-I**.

Cada ítem se marca `[x]` cuando se cierra y se agrega una línea: `→ resolución: <decisión y fecha>`.
