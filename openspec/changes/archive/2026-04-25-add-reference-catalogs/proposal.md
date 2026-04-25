# Proposal: Add reference catalogs (vulnerable activities + multi-country administrative divisions)

**Status**: draft
**Created**: 2026-04-25
**Sub-repos**: pld-api + pld-web (cross-repo)

## Intent

Exponer un conjunto de catálogos de referencia bajo `/catalogs/*` necesarios para el wizard de registro de sujeto obligado. Los catálogos siguen el patrón ya establecido por `GET /catalogs/beneficiario` (datos estáticos en TS, cache HTTP 24h, sin auth).

Los catálogos a crear son:
- `vulnerable-activities` — 10 actividades del Art. 17 LFPIORPI con labels en español.
- `countries` — lista de países soportados.
- `administrative-divisions/:countryCode/structure` — estructura jerárquica de divisiones administrativas para un país.
- `administrative-divisions/:countryCode/entries` — entries por nivel y `parentCode`, en cascada.

**Arquitectura preparada para multi-país desde el inicio**, aunque la seed v1 contiene solo México.

## Scope

**In** (BE — `pld-api`):
- Extender `apps/auth-users/src/catalogs/catalogs.data.ts` con nuevas constantes:
  - `CATALOGS_VULNERABLE_ACTIVITIES` — 10 entries.
  - `COUNTRIES` — seed con `[{ code: 'MX', name: 'México' }]`.
  - `ADMINISTRATIVE_STRUCTURES` — registro `{ MX: { levels: [...] } }` con 3 niveles (state, municipality, neighborhood).
  - `ADMINISTRATIVE_ENTRIES` — array con MX nivel 1 (32 estados) + MX nivel 2 (~2475 municipios). Nivel 3 NO cargado (texto libre en FE).
- Agregar handlers al `catalogs.controller.ts`:
  - `GET /catalogs/vulnerable-activities`
  - `GET /catalogs/countries`
  - `GET /catalogs/administrative-divisions/:countryCode/structure`
  - `GET /catalogs/administrative-divisions/:countryCode/entries?level=N&parentCode=X`
- Tests del controller que validen count, sincronización con enum (vulnerable-activities), filtrado por level/parentCode (administrative-divisions).
- Extender `openspec/specs/catalogs/spec.md` con nuevos Requirements.

**In** (FE — `pld-web`):
- Tipo `CatalogEntry` en `src/types/CatalogEntry.ts` (idéntico al BE).
- Tipos: `Country`, `AdministrativeLevel`, `AdministrativeStructure`, `AdministrativeDivisionEntry`.
- Funciones en `src/services/catalogsService.ts`: `getVulnerableActivities`, `getCountries`, `getAdministrativeStructure(countryCode)`, `getAdministrativeEntries(countryCode, level, parentCode?)`.
- Hooks en `src/queries/catalogsQueries.ts`: `useGetVulnerableActivities`, `useGetCountries`, `useGetAdministrativeStructure`, `useGetAdministrativeEntries`. Todos con `staleTime: 24h`.

**Out**:
- Componente UI `<AdministrativeDivisionsField />` que consume estos hooks: vive en el change `add-reporting-entity-registration-ui`.
- Catálogos no requeridos por el wizard PF/PM v1 (taxRegimes, genders, etc. que están en `catalogsService` apuntando a endpoints inexistentes — quedan como deuda separada).
- Carga de localidades/colonias (nivel 3 de MX): texto libre en v1.
- Otros países más allá de MX: estructura preparada, datos no cargados.
- No se mueve el enum `ActividadVulnerable`. Queda en `pld-api/packages/shared-types/src/enums.ts` como verdad canónica.

## Motivation

1. **Bloqueante para el wizard PF/PM**: el step 4 "Actividad vulnerable" necesita las 10 opciones. El step 4 también necesita dropdowns geográficos en cascada para state/municipality.
2. **Single source of truth multi-país**: la estructura administrativa (estados → municipios) vive en BE, accesible desde cualquier feature futura.
3. **Type safety**: el FE recibe tipos consistentes con el BE.
4. **Cache eficiente**: doble nivel (`Cache-Control` HTTP + `staleTime` React Query). Catálogos no consumen ancho de banda repetidamente.
5. **Escalable**: agregar Colombia, USA, España es solo cargar datos sin cambiar API.

## Approach

1. BE: agregar las constantes a `catalogs.data.ts` con los datos seed.
2. BE: registrar handlers en `catalogs.controller.ts` con cache header `public, max-age=86400, immutable`.
3. BE: tests del controller (count, drift detection enum, filtros).
4. BE: extender spec catalogs.
5. FE: agregar tipos compartidos.
6. FE: agregar funciones al service y hooks al queries.
7. Manual smoke test: `curl` los 4 endpoints, validar shape y headers de cache.

## Rollback plan

Revertir commits BE y FE. Los endpoints dejan de existir; los hooks ya no se importan. El módulo `catalogs/` vuelve a tener solo `beneficiario`.

## Affected surfaces

- BE: `apps/auth-users/src/catalogs/` (3 archivos modificados, 0 nuevos).
- FE: `src/types/`, `src/services/catalogsService.ts`, `src/queries/catalogsQueries.ts` (1 archivo nuevo, 2 modificados).
- Spec OpenSpec: `openspec/specs/catalogs/spec.md` (extensión).
- HTTP: 4 endpoints nuevos.

## Dependencies

Este change es **dependencia bloqueante** del change [add-reporting-entity-registration-ui](../add-reporting-entity-registration-ui/proposal.md). Implementar primero éste.

Construido **contra los enums actuales en español** (`ActividadVulnerable.TRANSMISION_DERECHOS_REALES_INMUEBLES`, etc.). El change `rename-to-english` queda en backlog y no se aplica antes.
