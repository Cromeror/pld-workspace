# Catálogos — arquitectura

## Resumen

El módulo de catálogos (`apps/auth-users/src/catalogs/`) expone catálogos fijos del dominio: países, actividades vulnerables y supuestos de beneficiario controlador. Son listas estáticas servidas con `Cache-Control` de 24h.

La resolución de **códigos postales / división administrativa** ya NO vive acá: se delegó al servicio externo NOT47, consumido directamente por el frontend. Ver [ADR-004](docs/decisiones/ADR-004-division-administrativa-servicio-externo.md).

---

## Archivos clave

| Archivo | Rol |
|---------|-----|
| `apps/auth-users/src/catalogs/catalogs.controller.ts` | Endpoints HTTP de catálogos fijos |
| `apps/auth-users/src/catalogs/catalogs.data.ts` | Constantes de catálogo y tipos |
| `apps/auth-users/src/catalogs/data/countries.ts` | Catálogo de países (ISO 3166-1) |

---

## Endpoints

| Endpoint | Devuelve |
|----------|----------|
| `GET /catalogs/countries` | Países con `postalCodeRegex` para MX |
| `GET /catalogs/vulnerable-activities` | Actividades vulnerables (fe pública, inmuebles) |
| `GET /catalogs/beneficiario` | Supuestos de beneficiario controlador |

Todos con `Cache-Control: public, max-age=86400, immutable`.

---

## Código postal (servicio externo)

El frontend resuelve el CP contra NOT47 vía `pld-web/src/services/postalCodeService.ts`:

```
GET {VITE_POSTAL_API_URL}/catalogs/postal-codes/{cp}
→ { success, data: { postal_code, state, municipality, neighborhoods: [{label, value}] } }
```

Devuelve entidad federativa, municipio y la lista de colonias por nombre. La localidad no la provee el servicio (se captura como campo libre en el FE). No hay dataset local ni fallback: si el servicio no responde, los campos de domicilio quedan editables manualmente.
