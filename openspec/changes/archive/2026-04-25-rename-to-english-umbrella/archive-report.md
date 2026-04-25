# Archive Report: rename-to-english (umbrella)

**Closed**: 2026-04-25
**Status**: ARCHIVED AS UMBRELLA — split en sub-changes
**Sub-repos**: pld-api + pld-web

## Por qué se archiva sin implementar

La propia proposal recomendaba (líneas 254-268) dividir en 3 sub-changes secuenciales por tamaño y riesgo. Audit del repo confirmó que aplicar las 79 tareas de un saque era inviable: tocaría BE + FE + DB + contratos públicos en una sola PR imposible de revisar y con merge conflicts garantizados.

Además, desde que se redactó la proposal aparecieron consumers nuevos (`add-current-user-endpoint` con `CurrentUserDto`, wizard con literales `UserRole.NOTARIO`, etc.) que el plan original no contemplaba.

## Plan de ejecución revisado (post-audit 2026-04-25)

Este umbrella se reemplaza por 4 sub-changes en orden de urgencia:

1. **`rename-enums-to-english`** — enum names + values (`NOTARIO`→`NOTARY`, `PERSONA_FISICA`→`INDIVIDUAL`, etc.). Fixes contrato API↔Web. **Highest priority.**
2. **`rename-users-table-to-english`** — `UserEntity` props (`nombre`→`firstName`, etc.) + DB columns. Fixes `CurrentUserDto` shape.
3. **`rename-participants-entities-to-english`** — 29 entity classes + 28 tables en `libs/participants`. BE-only, alto volumen.
4. **`rename-legacy-types-to-english`** — `RegistroPerfil`, `PFParticipante`, etc. Cleanup BE.

## Artifacts preservados

- `proposal.md` — sigue siendo referencia de scope total y rationale.
- `specs.md`, `design.md`, `tasks.md` — quedan como background; cada sub-change escribe los suyos.

## Audit findings (2026-04-25)

- Phase 1-7 del plan original siguen 100% aplicables (nada se renombró todavía).
- `USUARIO_INTERNO`/`USUARIO_EXTERNO` ya removidos por change `2026-04-22-remove-usuario-interno-externo`.
- Migración `20260421000000-create-registration-schema.sql` ya usa nombres en inglés en su esquema (consistente con el destino).
- Nuevo scope detectado (no en proposal original): `pld-web/src/types/CurrentUser.ts`, `PersonType.ts`, wizard `schemas.ts`, `ReportingEntityRegistrationPage.tsx` literales, `ReviewStep`, `ReportingEntityTypeStep`.
