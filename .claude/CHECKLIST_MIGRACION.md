# Checklist Interno — Migración a Packages

> **Ámbito**: BE-only. Contenido específico del sub-repo `pld-api/` (migración de `libs/` acoplados a NestJS → `packages/` puros). Vivía originalmente en `pld-api/.claude/CHECKLIST_MIGRACION.md` y fue movido al workspace el 2026-04-24 como parte de la consolidación de contexto Claude.
>
> **Paths**: todas las referencias a archivos del BE se expresan desde el root del workspace como `pld-api/...` (ej. `pld-api/libs/users/src/lib/users.entity.ts`).
>
> **Nota original**: este archivo es de trabajo interno entre el usuario y Claude. No se publica como documentación del proyecto. Gitignoreado en el workspace.

---

## Fase 0 — Fundaciones (✅ COMPLETA)

- [x] Migrar de npm a pnpm workspaces
- [x] Crear `packages/shared-types` con scalars (UUID, Email, RFC, etc.)
- [x] Crear `packages/shared-errors` con DomainError + subclases
- [x] Crear `packages/persistence` con `createDataSource` puro
- [x] Crear `packages/contracts` con EventPublisher
- [x] Registrar aliases en `tsconfig.base.json`
- [x] README.md por paquete
- [x] Validación: login sigue funcionando (HTTP 201)

---

## Pendientes migrar desde `libs/catalogs` → `packages/shared-types`

Enums ya movidos a `packages/shared-types/src/enums.ts` ✅
`libs/catalogs/src/lib/catalogs.types.ts` reemplazado por re-export desde `@pld-api/shared-types` ✅

- [x] `UserRole`
- [x] `TipoPersonaParticipante`
- [x] `Nacionalidad`
- [x] `ModalidadAtencion`
- [x] `ActividadVulnerable`
- [x] `TipoMoneda`
- [x] `FormaPago`
- [x] `TipoReporte`

Archivos que aún importan desde `@pld-api/catalogs` (funcionan vía re-export, migrar cuando se toque cada dominio):

- [ ] [pld-api/libs/users/src/lib/users.entity.ts](../pld-api/libs/users/src/lib/users.entity.ts)
- [ ] [pld-api/libs/participants/src/lib/moral/participants.entity.ts](../pld-api/libs/participants/src/lib/moral/participants.entity.ts)
- [ ] [pld-api/libs/participants/src/lib/fisica/participants.entity.ts](../pld-api/libs/participants/src/lib/fisica/participants.entity.ts)
- [ ] [pld-api/apps/auth-users/src/users/users.adapter.ts](../pld-api/apps/auth-users/src/users/users.adapter.ts)
- [ ] [pld-api/apps/auth-users/src/users/dto/create-user.dto.ts](../pld-api/apps/auth-users/src/users/dto/create-user.dto.ts)
- [ ] [pld-api/apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts](../pld-api/apps/auth-users/src/participants/persona-fisica/dto/create/persona-fisica.dto.ts)

Al final de todas las migraciones: borrar `libs/catalogs` completo.

---

## Pendientes migrar desde `libs/core`

Contenido de [pld-api/libs/core/src/index.ts](../pld-api/libs/core/src/index.ts):

### A `packages/shared-types`
- [x] `lib/core.types.ts` — eliminado, re-export desde `@pld-api/shared-types` en `pld-api/libs/core/src/index.ts`

### A `packages/domain-auth-users` (Fase 4, NO ahora)
- [ ] `services/password.function.ts` — `hashPassword`, `verifyPassword`, `generateSecurePassword`

### A `apps/api/src/shared/` (Fase 1 si se hace, o queda en gateway)
- [ ] `interceptors/http.error.interceptor.ts` — es NestJS puro, no va a packages

### Pendientes para evaluar más adelante

**Archivos scaffold vacíos de Nx** (mantener como placeholders para funcionalidad futura).
Evaluar cuando se migre cada dominio en Fase 4+: si se usan → desarrollar; si siguen vacíos → eliminar.

- [ ] `pld-api/libs/core/src/lib/core.service.ts` + `core.module.ts`
- [ ] `pld-api/libs/anexos/src/lib/anexos.service.ts` + `anexos.module.ts`
- [ ] `pld-api/libs/auth-profiles/src/lib/auth-profiles.service.ts` + `auth-profiles.module.ts`
- [ ] `pld-api/libs/operacion/src/lib/operacion.service.ts` + `operacion.module.ts`
- [ ] `pld-api/libs/risk/src/lib/risk.service.ts` + `risk.module.ts`
- [ ] `pld-api/libs/reports/src/lib/reports.service.ts` + `reports.module.ts`
- [ ] `pld-api/libs/catalogs/src/lib/catalogs.service.ts` + `catalogs.module.ts`
- [ ] `pld-api/libs/address-contact-id/src/lib/address.service.ts` + `address.module.ts`

**Libs puente (solo re-exportan desde `packages/shared-types`)**.
Evaluar cuando se migre el consumidor: si ya no se importan → eliminar la lib y el alias en `tsconfig.base.json`.

- [ ] `pld-api/libs/catalogs` — re-exporta enums. Consumidores: `pld-api/libs/users`, `pld-api/libs/participants` (2), `pld-api/apps/auth-users` (3).
- [ ] `pld-api/libs/address-contact-id` — re-exporta types de persona. Consumidores: `pld-api/apps/auth-users/users/users.adapter.ts` (1 import de `Contacto`).

Archivos que importan `@pld-api/core` y hay que actualizar:

- [ ] [pld-api/apps/auth-users/src/users/users.adapter.ts](../pld-api/apps/auth-users/src/users/users.adapter.ts)
- [ ] [pld-api/apps/auth-users/src/auth/auth.adapter.ts](../pld-api/apps/auth-users/src/auth/auth.adapter.ts)
- [ ] [pld-api/apps/auth-users/src/auth-users.module.ts](../pld-api/apps/auth-users/src/auth-users.module.ts)
- [ ] [pld-api/apps/catalogs/src/beneficiario.module.ts](../pld-api/apps/catalogs/src/beneficiario.module.ts)
- [ ] [pld-api/apps/cross/src/app.module.ts](../pld-api/apps/cross/src/app.module.ts)

---

## Migración `libs/persistencia-sql` → `packages/persistence` ✅ COMPLETADA

- [x] Imports migrados en `pld-api/apps/auth-users/src/auth-users.module.ts`
- [x] Imports migrados en `pld-api/apps/cross/src/app.module.ts`
- [x] `pld-api/libs/persistencia-sql/` eliminado
- [x] Alias `@pld-api/persistencia-sql` removido de `pld-api/tsconfig.base.json`
- [x] `autoLoadEntities: true` añadido a `mysqlConfigFromEnv` (requerido por NestJS)
- [x] Login validado HTTP 201

---

## Otros libs a evaluar (contenido mínimo, solo types)

Evaluados todos. Resultado:

- [ ] [pld-api/libs/anexos](../pld-api/libs/anexos) — solo types, **código muerto** (0 consumidores en apps). Se quedará hasta Fase 4 (`domain-operacion`).
- [ ] [pld-api/libs/auth-profiles](../pld-api/libs/auth-profiles) — solo types, **código muerto** (0 consumidores en apps). Se queda hasta Fase 4 (`domain-auth-users`).
- [ ] [pld-api/libs/operacion](../pld-api/libs/operacion) — solo types, **código muerto** (0 consumidores en apps). Se queda hasta Fase 4 (`domain-operacion`).
- [ ] [pld-api/libs/risk](../pld-api/libs/risk) — solo types, **código muerto** (0 consumidores en apps). Se queda hasta Fase 4 (`domain-risk`).
- [ ] [pld-api/libs/reports](../pld-api/libs/reports) — solo types, **código muerto** (0 consumidores en apps). Se queda hasta Fase 4 (`domain-reports`).
- [x] [pld-api/libs/address-contact-id](../pld-api/libs/address-contact-id) — migrado a `packages/shared-types/src/person.ts` (Domicilio, Contacto, TipoIdentificacion, Identificacion). Re-export mantenido en lib vieja.

Criterio aplicado: si hay consumidor real (address-contact-id) → migrar ya. Si está muerto → esperar a que su dominio se migre.

---

## Fases siguientes (alto nivel)

### Fase 2 — Consolidar `catalogs` en `auth-users` ✅ COMPLETADA (2026-04-20)
- [x] Eliminada app `pld-api/apps/catalogs`
- [x] Endpoint `GET /catalogs/beneficiario` ahora vive dentro de `auth-users` con `Cache-Control: public, max-age=86400, immutable`
- [x] `HttpErrorInterceptor` movido desde `pld-api/libs/core` a copia local en `pld-api/apps/auth-users/src/shared/` y `pld-api/apps/cross/src/shared/`
- [x] Primer test unitario del proyecto (`catalogs.controller.spec.ts`, 5 tests pasando)
- [x] Login HTTP 201 preservado, ambas apps compilan
- [ ] Pendiente futuro: unificar vocabularios de "tipos de control" (ver §11.1 del plan)
- [ ] Pendiente futuro: eliminar `pld-api/libs/catalogs` (lib puente mientras no se migren los 6 consumidores)

**Bloqueado — dudas del usuario sobre `pld-api/apps/catalogs`**:
- Qué hace exactamente el frontend con los `beneficiario-1/2/3` que devuelve el endpoint
- Por qué coexisten 3 vocabularios distintos para "tipos de control":
  - `beneficiario-1/2/3` en `pld-api/apps/catalogs` (eliminada; el endpoint vive en auth-users)
  - `TipoControlEjercido` (impone/ejerce/dirige) en DTOs de participantes
  - `PMBeneficiarioControlTipoControl` (IMPONE_DECISIONES/DIRIGE_ADMINISTRACION/VOTA_25_PORCIENTO) en `pld-api/libs/users/users.types.ts`
- Si los datos persistidos en BD usan uno de esos vocabularios o strings libres
- Posible unificación en un solo enum en `shared-types` — NO aplicar hasta resolver dudas

### Fase 3 — `domain-audit` + EventPublisher
- [ ] Decidir qué auditar (A/B/C — **pendiente respuesta del usuario**)
- [ ] Crear `packages/domain-audit`
- [ ] Implementación local de `EventPublisher`
- [ ] Wire-up en gateway

### Fase 4 — Migrar dominios complejos
- [ ] `packages/domain-users`
- [ ] `packages/domain-auth-users` (aquí va el crypto pospuesto)
- [ ] `packages/domain-participants` (fisica, moral, fideicomiso, anexo-7, beneficiario)

### Fase 5 — Cleanup
- [ ] Eliminar `pld-api/libs/` completo
- [ ] Actualizar `pld-api/tsconfig.base.json`
- [ ] Evaluar upgrade de Nx 14 → 22 (warning de peer deps)
- [ ] Remover `accessToken` de `pld-api/nx.json` (secreto commiteado)
- [ ] **Simplificar `prefixUrl`** en `pld-api/apps/auth-users/src/configs/common.configs.ts` → de `/pld-api/auth-users` a `/pld-api`. URLs públicas pasan de `/pld-api/auth-users/auth/login` a `/pld-api/auth/login`. Reconfirmar consumo real del frontend antes de ejecutar.

---

## Decisiones abiertas

- [ ] **Auditoría**: ¿A/B/C? (define Fase 3)
  - A: solo cambios de estado
  - B: A + acciones de auth (login OK/KO, logout)
  - C: B + lecturas sensibles

- [ ] **Fase 1 (unificar apps en `apps/api`)**: saltada de momento. Revisar si se retoma cuando estén más dominios migrados.

- [ ] **Bugs del diagnóstico de users pendientes**: ver [pld/docs/DIAGNOSTICO_SERVICIO_USUARIOS.md](../docs/DIAGNOSTICO_SERVICIO_USUARIOS.md) — no se han corregido aún.
