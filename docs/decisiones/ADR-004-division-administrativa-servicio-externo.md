---
type: agent-reference
date: 2026-06-30
status: aplicado
affects: pld-api/apps/auth-users/src/catalogs/, pld-api/apps/auth-users/src/*/registration*, pld-web/src/services/, pld-web/src/components/organisms/*/AddressStep|UserDataStep|VulnerableActivityStep
---

# ADR-004 — División administrativa vía servicio externo NOT47; baja del catálogo geográfico local

**Contexto**: el proyecto era dueño de un catálogo geográfico local de México (~160k líneas: `mx-neighborhoods.ts`, `mx-postal-codes.json` de 9.1 MB, `mx-states/municipalities/localities`), modelado como 4 niveles jerárquicos (`state`/`municipality`/`locality`/`neighborhood`) con `code`/`parentCode`. El endpoint `GET /catalogs/postal-codes/:cc/:cp` devolvía una respuesta plana con **una sola división por nivel**, por lo que un CP con múltiples colonias (p. ej. 52760, con 9) colapsaba a una y, cuando el código de colonia no resolvía, se mostraba el id en vez del nombre. La dirección se persistía en una tabla separada `address_division` (una fila por nivel).

La empresa ofrece un servicio externo (NOT47) que resuelve el CP correctamente:

```
GET {VITE_POSTAL_API_URL}/catalogs/postal-codes/{cp}
→ { success, data: { postal_code, state, municipality, neighborhoods: [{label, value}, ...] } }
```

Verificado contra 10 CP de distintos estados: devuelve **siempre** `state`, `municipality` y `neighborhoods` (niveles 1, 2 y 4) por nombre, y **nunca** `locality` (nivel 3). El dataset local viejo tampoco traía localidad en el 63% de los CP.

**Decisión**:

1. **El FE consume NOT47 directo** (nuevo `pld-web/src/services/postalCodeService.ts`, axios propio sin el interceptor de auth), no vía proxy en pld-api. La URL base viene de `VITE_POSTAL_API_URL` por ambiente.
2. **Se elimina el catálogo geográfico local del BE**: los 3 endpoints `administrative-divisions/structure`, `administrative-divisions/entries` y `postal-codes/:cc/:cp`; los datasets `mx-*`; y los registries/tipos `ADMINISTRATIVE_*`, `POSTAL_CODE_DATASETS`. Se conservan `countries`, `beneficiario`, `vulnerable-activities`.
3. **Se elimina la cascada del FE**: `AdministrativeDivisionsField`, `useDivisionLevel`, tipos `AdministrativeDivision`.
4. **La colonia** se elige de un dropdown editable con las colonias del servicio, permitiendo texto libre.
5. **La localidad** se mantiene en UI y modelo (el diseño la pide y es correcto) como **campo libre editable** — el servicio no la provee, así que se captura a mano. Nunca se autocompleta: comportamiento esperado, no bug.
6. **El shape de `divisions` se aplana**: la dirección deja de guardar un array jerárquico y pasa a columnas planas de texto `state`/`municipality`/`locality`/`neighborhood` en la tabla `address`. Se elimina la tabla `address_division` y la entidad `AddressDivisionEntity`. Migración `20260630000000-address-flatten-divisions.sql`: ALTER address + backfill desde address_division por field_name + DROP address_division.

**Consecuencias**:

- El payload de registro (cliente, auxiliar, sujeto obligado, external-clients) envía `state/municipality/locality/neighborhood` como strings, no `divisions[]`.
- El DTO compartido `AddressDivisionDto` se renombró a `AddressDivisionFieldsDto` (campos planos); `AddressDto` y `AuxiliaryAddressDto` lo extienden.
- Sin fallback local: si NOT47 no responde, el autocompletado no ocurre y los campos quedan editables manualmente — **nunca se bloquea el registro**.
- CORS: NOT47 responde el preflight con `access-control-allow-methods: GET` y `allow-headers: *`; el navegador acepta el origen del pld-web.
- `VITE_POSTAL_API_URL` usa la misma URL (`https://not47-api-qa.zurco.com.mx/not47-api`) en QA y PROD — confirmado con el equipo.
- La migración es destructiva (DROP address_division): correr backfill y respaldar antes en QA/PROD.

---

## Actualización 2026-07-10 — el punto 1 (FE directo) queda SUPERSEDED por proxy en pld-api

**Motivo**: la afirmación de CORS de §35 resultó falsa en el navegador. El preflight OPTIONS de NOT47 devuelve `access-control-allow-methods` pero **nunca `Access-Control-Allow-Origin`** (y además manda `access-control-allow-credentials: true`, combinación inválida). Verificado con Playwright: el GET del FE a NOT47 se bloquea con `net::ERR_FAILED` + `blocked by CORS policy: No 'Access-Control-Allow-Origin' header`. `curl` no lo detectaba porque no aplica política CORS. Consecuencia: `#state`/`#municipality` nunca autocompletaban en QA y fallaban 16 e2e de `registro-auxiliar`.

**Cambio**: el FE ya **no** pega directo a NOT47. pld-api actúa de **proxy server-side** (sin navegador ⇒ sin CORS):

- BE: `GET /pld-api/auth-users/catalogs/postal-codes/:cp` (público) en `apps/auth-users/src/catalogs/` → `PostalCodesService` llama a NOT47 con axios y **reenvía el body tal cual**. URL de NOT47 en env `NOT47_API_URL`.
- FE: `postalCodeService.ts` usa el axios normal (`@/config/axios`, base `VITE_API_URL`) y el path del proxy. Mismo shape `PostalCodeLookup` ⇒ los 3 consumidores (auxiliar, cliente, sujeto obligado), el hook y los tipos **no cambian**.
- Se elimina `VITE_POSTAL_API_URL` de los `.env` de pld-web (queda huérfana).

Verificado: proxy responde 200 (62010→Morelos/Cuernavaca) y reenvía 404 (62749); e2e local `registro-auxiliar` 21/21 verde sin tocar el spec.
