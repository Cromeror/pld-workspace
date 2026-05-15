# Catálogo de países — arquitectura y extensibilidad

## Resumen

El módulo de catálogos expone datos geográficos (divisiones administrativas y códigos postales) de forma agnóstica al país. Agregar un nuevo país no requiere tocar el controlador — solo registrar datos en `catalogs.data.ts`.

---

## Archivos clave

| Archivo | Rol |
|---------|-----|
| `apps/auth-users/src/catalogs/catalogs.controller.ts` | Endpoints HTTP — sin lógica de país |
| `apps/auth-users/src/catalogs/catalogs.data.ts` | Registries y tipos — punto de extensión |
| `apps/auth-users/src/catalogs/data/mx-states.ts` | Entidades federativas MX |
| `apps/auth-users/src/catalogs/data/mx-municipalities.ts` | Municipios MX |
| `apps/auth-users/src/catalogs/data/mx-postal-codes.ts` | Loader de CPs MX con caché |
| `apps/auth-users/src/catalogs/data/mx-postal-codes.json` | Dataset MX — 32k CPs agrupados por CP |

---

## Endpoints

### `GET /catalogs/administrative-divisions/:countryCode/structure`
Devuelve la definición de niveles del país.

```json
{
  "levels": [
    { "level": 1, "fieldName": "state", "label": "Entidad federativa" },
    { "level": 2, "fieldName": "municipality", "label": "Delegación o municipio" },
    { "level": 3, "fieldName": "locality", "label": "Localidad" },
    { "level": 4, "fieldName": "neighborhood", "label": "Colonia" }
  ]
}
```

### `GET /catalogs/administrative-divisions/:countryCode/entries?level=N&parentCode=X`
Devuelve entries de un nivel, filtradas por padre. `parentCode` requerido para level > 1. Level 3 (localidad) devuelve `[]` — el frontend cae a input libre.

### `GET /catalogs/postal-codes/:countryCode/:cp`
Resuelve un CP a sus divisiones administrativas con nombres.

```json
{
  "countryCode": "MX",
  "cp": "62000",
  "divisions": [
    { "level": 1, "fieldName": "state", "code": "17", "name": "Morelos" },
    { "level": 2, "fieldName": "municipality", "code": "17050", "name": "Cuernavaca" },
    { "level": 4, "fieldName": "neighborhood", "code": "Centro", "name": "Centro" }
  ]
}
```

---

## Registries (punto de extensión)

### `ADMINISTRATIVE_STRUCTURES`
Define los niveles de cada país. La clave es el `countryCode`.

```ts
export const ADMINISTRATIVE_STRUCTURES = {
  MX: {
    levels: [
      { level: 1, fieldName: 'state', label: 'Entidad federativa' },
      { level: 2, fieldName: 'municipality', label: 'Delegación o municipio' },
      { level: 3, fieldName: 'locality', label: 'Localidad' },
      { level: 4, fieldName: 'neighborhood', label: 'Colonia' },
    ],
  },
};
```

### `ADMINISTRATIVE_DIVISION_DATASETS`
Arrays de entries por país. `ADMINISTRATIVE_ENTRIES` se construye aplanando todos los valores — el controlador lo usa como fuente única.

```ts
export const ADMINISTRATIVE_DIVISION_DATASETS = {
  MX: [...ADMINISTRATIVE_ENTRIES_MX_STATES, ...ADMINISTRATIVE_ENTRIES_MX_MUNICIPALITIES],
};
```

### `POSTAL_CODE_DATASETS`
Funciones de lookup por país. Cada función recibe un CP y devuelve la entry o `undefined`. El caché se construye lazy al primer uso.

```ts
export const POSTAL_CODE_DATASETS = {
  MX: getMxPostalCodeEntries,
};
```

---

## Formato del JSON de códigos postales

Un registro por CP, colonias como divisiones de level 4:

```json
{
  "countryCode": "MX",
  "cp": "62000",
  "divisions": [
    { "level": 1, "code": "17" },
    { "level": 2, "code": "17050" },
    { "level": 4, "code": "Centro" }
  ]
}
```

El `countryCode` en cada entry es metadata de integridad — el dataset se identifica por su clave en el registry.

---

## Agregar un nuevo país (ej. US)

1. Crear `data/us-states.ts` y `data/us-counties.ts` con arrays de `AdministrativeDivisionEntry`.
2. Crear `data/us-postal-codes.json` con el mismo formato y `data/us-postal-codes.ts` con su loader.
3. En `catalogs.data.ts` registrar en los tres registries:

```ts
ADMINISTRATIVE_STRUCTURES:        { MX: ..., US: { levels: [...] } }
ADMINISTRATIVE_DIVISION_DATASETS: { MX: ..., US: [...usStates, ...usCounties] }
POSTAL_CODE_DATASETS:             { MX: ..., US: getUsPostalCodeEntries }
```

El controlador y el frontend no requieren cambios.
