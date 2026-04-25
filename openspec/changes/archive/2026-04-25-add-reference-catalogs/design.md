# Design: Reference catalogs (vulnerable activities + multi-country admin divisions)

## Decisions

### D1 — Reusar el patrón `beneficiario`

El módulo `apps/auth-users/src/catalogs/` ya tiene el patrón establecido. Mantenerlo:

```ts
export interface CatalogEntry { key: string; label: string; }
export const CATALOGS_X: readonly CatalogEntry[] = [...] as const;

@Get('x')
@Header('Cache-Control', 'public, max-age=86400, immutable')
x(): readonly CatalogEntry[] { return CATALOGS_X; }
```

Para los catálogos que no encajan exactamente en `CatalogEntry`, definir tipos específicos (`Country`, `AdministrativeLevel`, etc.) pero seguir la misma forma estructural.

### D2 — Vulnerable activities: keys idénticas al enum

Las `key` de `CATALOGS_VULNERABLE_ACTIVITIES` coinciden letra por letra con los valores del enum `ActividadVulnerable` de `@pld-api/shared-types`. Esto es crítico: el FE manda al endpoint `POST /admin/registration/:id/vulnerable-activity` el campo `activity` con uno de esos valores; el BE valida con `@IsEnum(ActividadVulnerable)`.

### D3 — Labels propuestos (revisión legal pendiente)

| key | label propuesto |
|---|---|
| TRANSMISION_DERECHOS_REALES_INMUEBLES | Transmisión de derechos reales sobre inmuebles |
| PODERES_IRREVOCABLES | Otorgamiento de poderes irrevocables |
| CONSTITUCION_SOCIEDADES | Constitución de personas morales |
| FUSION | Fusión de personas morales |
| ESCISION | Escisión de personas morales |
| AUMENTO_CAPITAL | Aumento de capital social |
| DISMINUCION_CAPITAL | Disminución de capital social |
| TRANSMISION_ACCIONES_PARTES | Transmisión de acciones o partes sociales |
| FIDEICOMISOS | Constitución o modificación de fideicomisos |
| MUTUOS_PRESTAMOS_CREDITOS | Otorgamiento de mutuos, préstamos o créditos |

Aproximaciones razonables al lenguaje del Art. 17 LFPIORPI. Marcado como **TODO de revisión legal/PO** antes de prod en `archive-report.md`.

### D4 — Modelo multi-país de divisiones administrativas

#### Tipos

```ts
type Country = {
  code: string;     // ISO 3166-1 alpha-2: 'MX', 'US', 'CO'
  name: string;     // 'México', 'Estados Unidos', 'Colombia'
};

type AdministrativeLevel = {
  level: number;        // 1, 2, 3
  fieldName: string;    // 'state', 'municipality', 'neighborhood'
  label: string;        // 'Entidad federativa', 'Delegación o municipio', 'Colonia'
};

type AdministrativeStructure = {
  countryCode: string;
  levels: AdministrativeLevel[];   // ordenados de menor a mayor especificidad
};

type AdministrativeDivisionEntry = {
  countryCode: string;
  level: number;
  code: string;                     // identificador único en su nivel y país
  name: string;                     // nombre legible
  parentCode: string | null;        // null si nivel 1; sino code del padre
};
```

#### Endpoints

```
GET /catalogs/countries
  → Country[]

GET /catalogs/administrative-divisions/:countryCode/structure
  → AdministrativeStructure
  404 si countryCode desconocido

GET /catalogs/administrative-divisions/:countryCode/entries?level=N&parentCode=X
  → AdministrativeDivisionEntry[]
  - level requerido (1-N)
  - parentCode requerido si level > 1
  - 400 si faltan params requeridos
  - 404 si countryCode desconocido
```

Ejemplo Mexico:
```
GET /catalogs/countries
  → [{ code: 'MX', name: 'México' }]

GET /catalogs/administrative-divisions/MX/structure
  → { countryCode: 'MX',
      levels: [
        { level: 1, fieldName: 'state',        label: 'Entidad federativa' },
        { level: 2, fieldName: 'municipality', label: 'Delegación o municipio' },
        { level: 3, fieldName: 'neighborhood', label: 'Colonia' }
      ]
    }

GET /catalogs/administrative-divisions/MX/entries?level=1
  → [{ countryCode: 'MX', level: 1, code: 'AGU', name: 'Aguascalientes', parentCode: null }, ... 32]

GET /catalogs/administrative-divisions/MX/entries?level=2&parentCode=AGU
  → [{ countryCode: 'MX', level: 2, code: '01001', name: 'Aguascalientes', parentCode: 'AGU' }, ...]
```

### D5 — Datos seed v1: solo México

| Recurso | Cantidad | Tamaño aprox |
|---|---|---|
| Countries | 1 (MX) | <1 KB |
| Structure MX | 1 estructura, 3 levels | <1 KB |
| MX nivel 1 | 32 estados | ~2 KB |
| MX nivel 2 | ~2475 municipios | ~140 KB |
| MX nivel 3 | NO cargado | — |

Para el nivel 3 (`neighborhood`), el FE renderiza un input de texto libre. La cascada se interrumpe: cuando el usuario selecciona municipality, el campo neighborhood ya no es dropdown.

Si en el futuro se cargan colonias (vía SEPOMEX o INEGI), se llenan las entries de nivel 3 sin cambiar API.

### D6 — Cache: 24h immutable

Todos los endpoints incluyen header `public, max-age=86400, immutable`. Ambos niveles (HTTP + React Query) caching con misma vida.

### D7 — Endpoints públicos sin autenticación

Idéntico a `beneficiario`. Catálogos son datos de referencia, no info sensible. Permite renderizar selects sin esperar al login.

### D8 — Almacenamiento BE: constantes TS

V1: las 32 entidades + 2475 municipios se hardcodean como constantes TS en `catalogs.data.ts`.

```ts
export const ADMINISTRATIVE_ENTRIES: readonly AdministrativeDivisionEntry[] = [
  // MX nivel 1 (32 estados)
  { countryCode: 'MX', level: 1, code: 'AGU', name: 'Aguascalientes', parentCode: null },
  // ... 31 más
  // MX nivel 2 (2475 municipios)
  { countryCode: 'MX', level: 2, code: '01001', name: 'Aguascalientes', parentCode: 'AGU' },
  // ... 2474 más
] as const;
```

Tamaño total ~150 KB de TS bundleados al BE. Aceptable para v1.

Si crece a múltiples países o a nivel 3, migrar a tablas MySQL. Out of scope v1.

### D9 — Filtrado en el handler

```ts
@Get(':countryCode/entries')
@Header('Cache-Control', 'public, max-age=86400, immutable')
entries(
  @Param('countryCode') countryCode: string,
  @Query('level') level: string,
  @Query('parentCode') parentCode?: string,
): readonly AdministrativeDivisionEntry[] {
  const levelNum = parseInt(level, 10);
  if (isNaN(levelNum)) throw new BadRequestException('level must be a number');
  if (levelNum > 1 && !parentCode) {
    throw new BadRequestException('parentCode required for level > 1');
  }
  return ADMINISTRATIVE_ENTRIES.filter(e =>
    e.countryCode === countryCode &&
    e.level === levelNum &&
    (levelNum === 1 || e.parentCode === parentCode),
  );
}
```

### D10 — FE: hooks con `staleTime` 24h

```ts
export const useGetCountries = () => useQuery({
  queryKey: ['countries'],
  queryFn: catalogsService.getCountries,
  staleTime: 24 * 60 * 60 * 1000,
});

export const useGetAdministrativeStructure = (countryCode?: string) => useQuery({
  queryKey: ['adminStructure', countryCode],
  queryFn: () => catalogsService.getAdministrativeStructure(countryCode!),
  enabled: !!countryCode,
  staleTime: 24 * 60 * 60 * 1000,
});

export const useGetAdministrativeEntries = (
  countryCode?: string, level?: number, parentCode?: string,
) => useQuery({
  queryKey: ['adminEntries', countryCode, level, parentCode],
  queryFn: () => catalogsService.getAdministrativeEntries(countryCode!, level!, parentCode),
  enabled: !!countryCode && !!level && (level === 1 || !!parentCode),
  staleTime: 24 * 60 * 60 * 1000,
});
```

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Labels mal redactados legalmente | Bajo (texto visible al usuario) | TODO en archive-report para revisión por abogado/PO |
| BE agrega valor al enum y olvida actualizar el catálogo | Medio | Test que valida `CATALOGS_VULNERABLE_ACTIVITIES.length === Object.keys(ActividadVulnerable).length` |
| Dataset de municipios MX desactualizado o incorrecto | Bajo | Seed inicial desde INEGI (referencia oficial); actualizable en futuro change |
| FE consume antes de deploy BE | Alto (404 al cargar wizard) | Este change debe mergear y deployar antes del wizard UI |
| Bundle BE crece ~150 KB con municipios | Bajo (BE no se sirve al cliente) | Aceptable; futura migración a tablas si crece |

## Open questions

- ¿El catálogo cambia según `userRole` (NOTARIO vs INMOBILIARIA)? El flowchart sugiere distintos títulos de sección según rol, pero el catálogo de valores parece ser el mismo. Para v1: catálogo único. Si se confirma diferenciación, agregar query param `?userRole=` en futuro change.
- ¿Cuándo se carga el dataset SEPOMEX para autocompletar colonias? Out of scope v1; queda como mejora UX en backlog.

## Notas

- Las **keys** del catálogo de actividades vulnerables están en español (consistente con el enum BE actual). El change `rename-to-english` cambiaría esto en el futuro pero **no se aplica antes** del wizard.
- La arquitectura multi-país está pensada para que cuando se agregue Colombia/USA/etc., **solo sea cargar datos** sin tocar API ni componentes FE.
