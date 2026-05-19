---
type: agent-reference
date: 2026-05-19
status: aplicado
affects: pld-api/apps/auth-users/src/auth/, pld-api/packages/domain-auth-users/src/jwt/, docs/flujos/
---

# ADR-002 — Separación de UserRole y ActivityType: unificación en WORKSPACE_ADMIN

**Contexto**: el sistema tenía dos valores de `UserRole` para sujetos obligados: `NOTARY` y `REAL_ESTATE`. Estos valores cumplían dos funciones a la vez: identificaban el rol de seguridad del usuario (permisos de acceso) y el tipo de actividad vulnerable que ejerce (dominio de negocio). Esa ambigüedad generaba confusión en documentación, guards y queries — p. ej. `role IN (NOTARY, REAL_ESTATE)` aparecía en guards de autorización mezclado con lógica de negocio.

**Decisión**: unificar `NOTARY` y `REAL_ESTATE` en un único valor de rol de seguridad `WORKSPACE_ADMIN`. El tipo de actividad vulnerable se expresa a través de `ActivityType` (`workspace.activityType`), que sigue usando los valores `NOTARY` y `REAL_ESTATE` como identificadores de actividad.

La separación queda así:

| Concepto | Dónde vive | Valores |
|---|---|---|
| Rol de seguridad (`UserRole`) | `users.role`, JWT `role` | `WORKSPACE_ADMIN`, `SUPERADMIN`, `AUXILIARY` |
| Tipo de actividad (`ActivityType`) | `registration_workspace.activityType` | `NOTARY`, `REAL_ESTATE` |

**Consecuencias**:

- Los guards de autorización usan `role = WORKSPACE_ADMIN` en lugar de `role IN (NOTARY, REAL_ESTATE)`.
- El JWT incluye `role: WORKSPACE_ADMIN` para todos los sujetos obligados. El `workspaceId` permite resolver el `activityType` cuando se necesita lógica diferenciada por tipo de actividad.
- `sourceUserRole` en el flujo de actividad secundaria se renombra a `sourceActivityType` — lo que se propaga es el tipo de actividad del registro principal, no un rol.
- La documentación de flujos y test-users usa el rol unificado en la columna "Actores / Rol", y aclara el `activityType` como atributo separado.
- Los valores `NOTARY` y `REAL_ESTATE` NO desaparecen del sistema — siguen siendo válidos como `activityType` de workspace y como identificadores en `availableProfiles` durante la selección de perfil.
