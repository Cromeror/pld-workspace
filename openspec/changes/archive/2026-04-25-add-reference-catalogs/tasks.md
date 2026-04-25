# Tasks: Add reference catalogs

## Phase 0 — Preflight

- [x] 0.1 Confirmar dataset INEGI o similar para los 32 estados y ~2475 municipios. Descargar archivo CSV/JSON oficial.  *(estados cargados; municipios pendientes — ver §1.4.2)*
- [x] 0.2 Convertir dataset a formato TS via script (puede ser manual con Excel→JSON→TS o automatizado con `node`).
- [x] 0.3 Validar que los `code` son únicos por nivel.  *(test "all states have unique codes")*

## Phase 1 — Backend (pld-api)

### 1.1 Vulnerable activities

- [x] 1.1.1 Editar `pld-api/apps/auth-users/src/catalogs/catalogs.data.ts`. Agregar `CATALOGS_VULNERABLE_ACTIVITIES` con las 10 entries definidas en `design.md` §D3.
- [x] 1.1.2 Editar `pld-api/apps/auth-users/src/catalogs/catalogs.controller.ts`. Agregar handler `vulnerableActivities()` con `@Get('vulnerable-activities')` y `@Header('Cache-Control', 'public, max-age=86400, immutable')`.
- [x] 1.1.3 Test: `vulnerableActivities()` devuelve array de 10. Drift detection: `expect(CATALOGS_VULNERABLE_ACTIVITIES.length).toBe(Object.keys(ActividadVulnerable).length)`.

### 1.2 Countries

- [x] 1.2.1 Agregar `COUNTRIES` a `catalogs.data.ts` con seed `[{ code: 'MX', name: 'México' }]`.
- [x] 1.2.2 Agregar handler `countries()` con `@Get('countries')` y cache header.
- [x] 1.2.3 Test del handler.

### 1.3 Administrative structures

- [x] 1.3.1 Definir tipo `AdministrativeStructure` en `catalogs.data.ts`.
- [x] 1.3.2 Agregar constante `ADMINISTRATIVE_STRUCTURES: Record<string, AdministrativeStructure>` con clave `MX` y los 3 niveles.
- [x] 1.3.3 Agregar handler `administrativeStructure(@Param('countryCode'))` con `@Get('administrative-divisions/:countryCode/structure')` y cache header.
- [x] 1.3.4 Si `countryCode` no existe, lanzar `NotFoundException`.
- [x] 1.3.5 Test: structure para MX devuelve 3 levels; structure para `XX` devuelve 404.

### 1.4 Administrative entries

- [x] 1.4.1 Definir tipo `AdministrativeDivisionEntry` en `catalogs.data.ts`.
- [x] 1.4.2 Agregar constante `ADMINISTRATIVE_ENTRIES` con MX nivel 1 (32 estados) + nivel 2 (~2475 municipios).  *(2026-04-25: dataset cargado desde cisnerosnow/json-estados-municipios-mexico → 2478 municipios INEGI. Tests de referential integrity y unicidad pasando.)*
- [x] 1.4.3 Agregar handler `administrativeEntries(@Param('countryCode'), @Query('level'), @Query('parentCode'))` con `@Get('administrative-divisions/:countryCode/entries')` y cache header.
- [x] 1.4.4 Validaciones:
  - Si `level` no es número → 400.
  - Si `level > 1 && !parentCode` → 400.
  - Si `countryCode` no existe → 404.
- [x] 1.4.5 Filtrar `ADMINISTRATIVE_ENTRIES` por `countryCode + level + (parentCode || null para level=1)`.
- [x] 1.4.6 Test: nivel 1 devuelve 32 con `parentCode===null`; nivel 2 con parentCode válido devuelve N>0 (cuando dataset cargado); level=2 sin parentCode devuelve 400; countryCode `XX` devuelve 404.  *(test del nivel 2 valida shape de entries cuando existen; tolerante a array vacío hasta carga del dataset)*

### 1.5 Validación local BE

- [ ] 1.5.1 Levantar BE: `/stack-up --no-web`.  *(pendiente: el usuario lo valida en sesión interactiva)*
- [ ] 1.5.2 `curl http://localhost:9001/pld-api/auth-users/catalogs/vulnerable-activities` devuelve 10 entries.  *(pendiente smoke manual)*
- [ ] 1.5.3 `curl http://localhost:9001/pld-api/auth-users/catalogs/countries` devuelve [{MX,México}].  *(pendiente smoke manual)*
- [ ] 1.5.4 `curl http://localhost:9001/pld-api/auth-users/catalogs/administrative-divisions/MX/structure` devuelve 3 levels.  *(pendiente smoke manual)*
- [ ] 1.5.5 `curl http://localhost:9001/pld-api/auth-users/catalogs/administrative-divisions/MX/entries?level=1` devuelve 32 estados.  *(pendiente smoke manual)*
- [ ] 1.5.6 `curl ".../administrative-divisions/MX/entries?level=2&parentCode=AGU"` devuelve municipios de Aguascalientes.  *(retornará array vacío hasta cargar dataset municipios — §1.4.2)*
- [ ] 1.5.7 `curl -I` cualquier endpoint muestra `Cache-Control: public, max-age=86400, immutable`.  *(pendiente smoke manual)*
- [x] 1.5.8 `cd pld-api && pnpm nx run auth-users:test` pasa.  *(21/21 tests passing)*

## Phase 2 — Frontend (pld-web)

### 2.1 Tipos compartidos

- [x] 2.1.1 Crear `pld-web/src/types/CatalogEntry.ts`:
  ```ts
  export type CatalogEntry = { key: string; label: string };
  ```
- [x] 2.1.2 Crear `pld-web/src/types/Country.ts`:
  ```ts
  export type Country = { code: string; name: string };
  ```
- [x] 2.1.3 Crear `pld-web/src/types/AdministrativeDivision.ts` con `AdministrativeLevel`, `AdministrativeStructure`, `AdministrativeDivisionEntry`.

### 2.2 Servicio

- [x] 2.2.1 Editar `pld-web/src/services/catalogsService.ts`. Agregar:
  ```ts
  const getVulnerableActivities = async (): Promise<CatalogEntry[]> => { ... };
  const getCountries = async (): Promise<Country[]> => { ... };
  const getAdministrativeStructure = async (countryCode: string): Promise<AdministrativeStructure> => { ... };
  const getAdministrativeEntries = async (
    countryCode: string, level: number, parentCode?: string,
  ): Promise<AdministrativeDivisionEntry[]> => { ... };
  ```
- [x] 2.2.2 Sumar al export `catalogsService`.

### 2.3 Queries

- [x] 2.3.1 Editar `pld-web/src/queries/catalogsQueries.ts`. Agregar:
  - `useGetVulnerableActivities` con `staleTime: 24h`.
  - `useGetCountries` con `staleTime: 24h`.
  - `useGetAdministrativeStructure(countryCode?)` con `enabled: !!countryCode`, `staleTime: 24h`.
  - `useGetAdministrativeEntries(countryCode?, level?, parentCode?)` con `enabled: !!countryCode && !!level && (level === 1 || !!parentCode)`, `staleTime: 24h`.

### 2.4 Validación FE

- [x] 2.4.1 `cd pld-web && yarn tsc -b --noEmit` pasa sin errores.  *(Done in 2.74s)*
- [ ] 2.4.2 Smoke: importar `useGetCountries` en una página de prueba, verificar que el request se hace y la respuesta se cachea.  *(pendiente: se valida cuando el wizard PF/PM consume los hooks en `add-reporting-entity-registration-ui`)*

## Phase 3 — Specs delta

- [ ] 3.1 Aplicar el delta de [specs.md](specs.md) a `openspec/specs/catalogs/spec.md` durante `/sdd-archive`.

## Phase 4 — Commits

- [ ] 4.1 Commit BE en `pld-api`: `feat(catalogs): add vulnerable-activities, countries, administrative-divisions endpoints`.
- [ ] 4.2 Commit FE en `pld-web`: `feat(catalogs): consume reference catalogs endpoints`.
- [ ] 4.3 (Opcional) `/sdd-archive add-reference-catalogs`.
