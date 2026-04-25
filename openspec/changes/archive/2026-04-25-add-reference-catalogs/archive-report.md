# Archive Report: add-reference-catalogs

**Closed**: 2026-04-25
**Status**: IMPLEMENTED & SMOKE-TESTED
**Sub-repos**: pld-api + pld-web (cross-repo)

## Summary

Catálogos de referencia bajo `/catalogs/*` para consumo del wizard de registro de sujeto obligado. Patrón consistente con `GET /catalogs/beneficiario`: datos estáticos en TS, cache HTTP 24h, sin auth. Arquitectura preparada para multi-país desde el inicio; seed v1 carga solo México.

## Endpoints delivered

- `GET /catalogs/vulnerable-activities` — 10 actividades del Art. 17 LFPIORPI con labels en español.
- `GET /catalogs/countries` — 1 país en seed v1 (`MX`).
- `GET /catalogs/administrative-divisions/:countryCode/structure` — estructura jerárquica del país (404 si countryCode desconocido).
- `GET /catalogs/administrative-divisions/:countryCode/entries?level=N&parentCode=X` — entries filtrados en cascada (400 si falta parentCode con level>1, 404 si countryCode desconocido).

Todos con header `Cache-Control: public, max-age=86400, immutable`. Sin `JwtAuthGuard`.

## BE files (new/modified)

### Modified
- `apps/auth-users/src/catalogs/catalogs.data.ts` — agregadas interfaces (`Country`, `AdministrativeLevel`, `AdministrativeStructure`, `AdministrativeDivisionEntry`) y constantes (`CATALOGS_VULNERABLE_ACTIVITIES`, `COUNTRIES`, `ADMINISTRATIVE_STRUCTURES`, `ADMINISTRATIVE_ENTRIES`).
- `apps/auth-users/src/catalogs/catalogs.controller.ts` — 4 nuevos handlers.
- `apps/auth-users/src/catalogs/catalogs.controller.spec.ts` — 18 tests adicionales (drift detection del enum, count, shape, filtros, referential integrity de municipios).

### New
- `apps/auth-users/src/catalogs/data/mx-states.ts` — 32 entidades federativas con códigos INEGI 2-dígitos.
- `apps/auth-users/src/catalogs/data/mx-municipalities.ts` — 2478 municipios con códigos 5-dígitos (estado 2 + municipio 3) y `parentCode` referenciando estados. Dataset cargado desde `cisnerosnow/json-estados-municipios-mexico`.

## FE files (new/modified)

### New
- `src/types/CatalogEntry.ts`
- `src/types/Country.ts`
- `src/types/AdministrativeDivision.ts` — `AdministrativeLevel`, `AdministrativeStructure`, `AdministrativeDivisionEntry`.

### Modified
- `src/services/catalogsService.ts` — 4 nuevas funciones (`getVulnerableActivities`, `getCountries`, `getAdministrativeStructure`, `getAdministrativeEntries`).
- `src/queries/catalogsQueries.ts` — 4 nuevos hooks con `staleTime: 24h` y `enabled` condicional.

## Tests

`pnpm nx run auth-users:test`: 24/24 tests passing del módulo catalogs (3 originales + 21 nuevos).

Drift detection enforced: `expect(CATALOGS_VULNERABLE_ACTIVITIES.length).toBe(Object.keys(ActividadVulnerable).length)`.

Referential integrity enforced: cada municipio MX tiene `parentCode` apuntando a un estado existente; codes únicos por nivel.

## Decisiones aplicadas

- **D1** (proposal): patrón `beneficiario` reusado.
- **D2** (proposal): keys de vulnerable-activities idénticas al enum BE.
- **D4** (proposal): modelo multi-país desde el inicio. Endpoint con `:countryCode` en URL — agregar Colombia, USA, etc. es solo cargar datos.
- **D5** (proposal): seed v1 solo MX, con 32 estados + 2478 municipios. Nivel 3 (colonias) NO cargado — el FE tiene fallback a InputText libre.
- **R3** (refinements): hooks viven en `catalogsQueries.ts` existente, no en carpeta nueva. `staleTime: 24h` alineado con `Cache-Control` del BE.

## Pendientes documentados

- **Revisión legal de labels**: las 10 etiquetas de `vulnerable-activities` son aproximaciones razonables al lenguaje del Art. 17 LFPIORPI. Pendiente revisión por abogado/PO antes de prod.
- **Open question**: ¿el catálogo cambia según `userRole`? Por ahora catálogo único. Si se diferencia, agregar `?userRole=` query param en futuro change.
- **SEPOMEX para colonias**: out of scope v1; backlog UX para autocompletar nivel 3.

## Verificación post-deploy

`curl` smoke tests confirman:
- 200 con shape correcto en los 4 endpoints
- 32 estados MX, 16 alcaldías CDMX (parentCode='09'), 10 vulnerable-activities
- Cache-Control header presente
- 404 con countryCode desconocido
- 400 con `level>1 && !parentCode`

## Specs delta

Aplicar `specs.md` a `openspec/specs/catalogs/spec.md`:

- Nuevo Requirement: Endpoint de actividades vulnerables (4 scenarios).
- Nuevo Requirement: Endpoint de países soportados (2 scenarios).
- Nuevo Requirement: Endpoint de estructura administrativa por país (2 scenarios).
- Nuevo Requirement: Endpoint de entradas administrativas filtradas (4 scenarios).
- Nuevo Requirement: Endpoints públicos (1 scenario).

## Commits

- `pld-api` (develop): `8fed315 feat(catalogs): add vulnerable-activities, countries, administrative-divisions endpoints`
- `pld-web` (main): `6c09d07 feat(catalogs): consume reference catalogs endpoints`
