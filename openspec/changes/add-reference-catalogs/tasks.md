# Tasks: Add reference catalogs

## Phase 0 — Preflight

- [ ] 0.1 Confirmar dataset INEGI o similar para los 32 estados y ~2475 municipios. Descargar archivo CSV/JSON oficial.
- [ ] 0.2 Convertir dataset a formato TS via script (puede ser manual con Excel→JSON→TS o automatizado con `node`).
- [ ] 0.3 Validar que los `code` son únicos por nivel.

## Phase 1 — Backend (pld-api)

### 1.1 Vulnerable activities

- [ ] 1.1.1 Editar `pld-api/apps/auth-users/src/catalogs/catalogs.data.ts`. Agregar `CATALOGS_VULNERABLE_ACTIVITIES` con las 10 entries definidas en `design.md` §D3.
- [ ] 1.1.2 Editar `pld-api/apps/auth-users/src/catalogs/catalogs.controller.ts`. Agregar handler `vulnerableActivities()` con `@Get('vulnerable-activities')` y `@Header('Cache-Control', 'public, max-age=86400, immutable')`.
- [ ] 1.1.3 Test: `vulnerableActivities()` devuelve array de 10. Drift detection: `expect(CATALOGS_VULNERABLE_ACTIVITIES.length).toBe(Object.keys(ActividadVulnerable).length)`.

### 1.2 Countries

- [ ] 1.2.1 Agregar `COUNTRIES` a `catalogs.data.ts` con seed `[{ code: 'MX', name: 'México' }]`.
- [ ] 1.2.2 Agregar handler `countries()` con `@Get('countries')` y cache header.
- [ ] 1.2.3 Test del handler.

### 1.3 Administrative structures

- [ ] 1.3.1 Definir tipo `AdministrativeStructure` en `catalogs.data.ts`.
- [ ] 1.3.2 Agregar constante `ADMINISTRATIVE_STRUCTURES: Record<string, AdministrativeStructure>` con clave `MX` y los 3 niveles.
- [ ] 1.3.3 Agregar handler `administrativeStructure(@Param('countryCode'))` con `@Get('administrative-divisions/:countryCode/structure')` y cache header.
- [ ] 1.3.4 Si `countryCode` no existe, lanzar `NotFoundException`.
- [ ] 1.3.5 Test: structure para MX devuelve 3 levels; structure para `XX` devuelve 404.

### 1.4 Administrative entries

- [ ] 1.4.1 Definir tipo `AdministrativeDivisionEntry` en `catalogs.data.ts`.
- [ ] 1.4.2 Agregar constante `ADMINISTRATIVE_ENTRIES` con MX nivel 1 (32 estados) + nivel 2 (~2475 municipios).
- [ ] 1.4.3 Agregar handler `administrativeEntries(@Param('countryCode'), @Query('level'), @Query('parentCode'))` con `@Get('administrative-divisions/:countryCode/entries')` y cache header.
- [ ] 1.4.4 Validaciones:
  - Si `level` no es número → 400.
  - Si `level > 1 && !parentCode` → 400.
  - Si `countryCode` no existe → 404.
- [ ] 1.4.5 Filtrar `ADMINISTRATIVE_ENTRIES` por `countryCode + level + (parentCode || null para level=1)`.
- [ ] 1.4.6 Test: nivel 1 devuelve 32 con `parentCode===null`; nivel 2 con parentCode válido devuelve N>0; level=2 sin parentCode devuelve 400; countryCode `XX` devuelve 404.

### 1.5 Validación local BE

- [ ] 1.5.1 Levantar BE: `/stack-up --no-web`.
- [ ] 1.5.2 `curl http://localhost:9001/pld-api/auth-users/catalogs/vulnerable-activities` devuelve 10 entries.
- [ ] 1.5.3 `curl http://localhost:9001/pld-api/auth-users/catalogs/countries` devuelve [{MX,México}].
- [ ] 1.5.4 `curl http://localhost:9001/pld-api/auth-users/catalogs/administrative-divisions/MX/structure` devuelve 3 levels.
- [ ] 1.5.5 `curl http://localhost:9001/pld-api/auth-users/catalogs/administrative-divisions/MX/entries?level=1` devuelve 32 estados.
- [ ] 1.5.6 `curl ".../administrative-divisions/MX/entries?level=2&parentCode=AGU"` devuelve municipios de Aguascalientes.
- [ ] 1.5.7 `curl -I` cualquier endpoint muestra `Cache-Control: public, max-age=86400, immutable`.
- [ ] 1.5.8 `cd pld-api && pnpm nx run auth-users:test` pasa.

## Phase 2 — Frontend (pld-web)

### 2.1 Tipos compartidos

- [ ] 2.1.1 Crear `pld-web/src/types/CatalogEntry.ts`:
  ```ts
  export type CatalogEntry = { key: string; label: string };
  ```
- [ ] 2.1.2 Crear `pld-web/src/types/Country.ts`:
  ```ts
  export type Country = { code: string; name: string };
  ```
- [ ] 2.1.3 Crear `pld-web/src/types/AdministrativeDivision.ts` con `AdministrativeLevel`, `AdministrativeStructure`, `AdministrativeDivisionEntry`.

### 2.2 Servicio

- [ ] 2.2.1 Editar `pld-web/src/services/catalogsService.ts`. Agregar:
  ```ts
  const getVulnerableActivities = async (): Promise<CatalogEntry[]> => { ... };
  const getCountries = async (): Promise<Country[]> => { ... };
  const getAdministrativeStructure = async (countryCode: string): Promise<AdministrativeStructure> => { ... };
  const getAdministrativeEntries = async (
    countryCode: string, level: number, parentCode?: string,
  ): Promise<AdministrativeDivisionEntry[]> => { ... };
  ```
- [ ] 2.2.2 Sumar al export `catalogsService`.

### 2.3 Queries

- [ ] 2.3.1 Editar `pld-web/src/queries/catalogsQueries.ts`. Agregar:
  - `useGetVulnerableActivities` con `staleTime: 24h`.
  - `useGetCountries` con `staleTime: 24h`.
  - `useGetAdministrativeStructure(countryCode?)` con `enabled: !!countryCode`, `staleTime: 24h`.
  - `useGetAdministrativeEntries(countryCode?, level?, parentCode?)` con `enabled: !!countryCode && !!level && (level === 1 || !!parentCode)`, `staleTime: 24h`.

### 2.4 Validación FE

- [ ] 2.4.1 `cd pld-web && yarn tsc -b --noEmit` pasa sin errores.
- [ ] 2.4.2 Smoke: importar `useGetCountries` en una página de prueba, verificar que el request se hace y la respuesta se cachea.

## Phase 3 — Specs delta

- [ ] 3.1 Aplicar el delta de [specs.md](specs.md) a `openspec/specs/catalogs/spec.md` durante `/sdd-archive`.

## Phase 4 — Commits

- [ ] 4.1 Commit BE en `pld-api`: `feat(catalogs): add vulnerable-activities, countries, administrative-divisions endpoints`.
- [ ] 4.2 Commit FE en `pld-web`: `feat(catalogs): consume reference catalogs endpoints`.
- [ ] 4.3 (Opcional) `/sdd-archive add-reference-catalogs`.
