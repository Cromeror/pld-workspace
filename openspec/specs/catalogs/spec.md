# Catalogs Specification

## Purpose

Entregar datos estáticos de referencia (labels de formularios, enums) para alimentar controles del frontend (selects, radios, checkboxes). El servicio optimiza latencia vía cache HTTP del lado del cliente. No expone BD ni requiere autenticación.

## Requirements

### Requirement: Endpoint de labels de beneficiario controlador

El sistema DEBE exponer `GET /catalogs/beneficiario` dentro del gateway `auth-users`, retornando el array fijo de 3 labels que describen criterios legales de beneficiario controlador PLD.

#### Scenario: Respuesta exitosa con las 3 labels

- GIVEN el gateway auth-users está corriendo
- WHEN un cliente envía `GET /pld-api/auth-users/catalogs/beneficiario`
- THEN el servidor responde HTTP 200
- AND el body es un array con exactamente 3 objetos `{ key: string, label: string }`
- AND las keys son `beneficiario-1`, `beneficiario-2`, `beneficiario-3` en ese orden

#### Scenario: Contenido de labels conservado exactamente del servicio anterior

- GIVEN la lista actual de labels en `apps/catalogs/src/beneficiario/beneficiario.adapter.ts`
- WHEN se compara con la respuesta del nuevo endpoint
- THEN los valores de `label` son idénticos carácter por carácter

### Requirement: Header de cache HTTP en endpoints de catálogos

Cada endpoint bajo `/catalogs/*` DEBE incluir en la respuesta el header `Cache-Control: public, max-age=86400, immutable` para permitir cache agresivo del lado del cliente.

#### Scenario: Header presente en respuesta exitosa

- GIVEN un cliente llama `GET /pld-api/auth-users/catalogs/beneficiario`
- WHEN el servidor responde 200
- THEN el header `Cache-Control` está presente
- AND su valor es exactamente `public, max-age=86400, immutable`

### Requirement: Endpoint público sin autenticación

El endpoint `GET /catalogs/beneficiario` NO DEBE requerir token JWT ni ningún otro credencial.

#### Scenario: Cliente anónimo accede exitosamente

- GIVEN un cliente sin header `Authorization`
- WHEN envía `GET /pld-api/auth-users/catalogs/beneficiario`
- THEN el servidor responde HTTP 200

### Requirement: Latencia sub-5ms en segunda llamada

El endpoint DEBE responder en menos de 5 ms de tiempo server-side en la segunda llamada consecutiva (descartado JIT warmup inicial).

#### Scenario: Medición consecutiva

- GIVEN el servidor recibió al menos 1 llamada previa al endpoint
- WHEN un cliente envía una nueva llamada `GET /pld-api/auth-users/catalogs/beneficiario`
- THEN el tiempo entre recepción de request y envío de response es menor a 5 ms

### Requirement: Test unitario del controller

El módulo DEBE incluir un test unitario Jest que valide el contenido retornado por `CatalogsController.beneficiario()`.

#### Scenario: Test pasa contra los 3 elementos esperados

- GIVEN el archivo `apps/auth-users/src/catalogs/catalogs.controller.spec.ts` existe
- WHEN se ejecuta `nx run auth-users:test --skip-nx-cache`
- THEN el test corre (no es saltado por `passWithNoTests`)
- AND termina con exit code 0
- AND verifica que el resultado es un array de exactamente 3 elementos con las keys `beneficiario-1`, `beneficiario-2`, `beneficiario-3`

### Requirement: Endpoint de actividades vulnerables

El sistema DEBE exponer `GET /catalogs/vulnerable-activities` retornando el array fijo de las 10 actividades del Art. 17 LFPIORPI.

#### Scenario: Respuesta exitosa con las 10 actividades

- GIVEN el gateway auth-users está corriendo
- WHEN un cliente envía `GET /pld-api/auth-users/catalogs/vulnerable-activities`
- THEN el servidor responde HTTP 200
- AND el body es un array con exactamente 10 objetos `{ key: string, label: string }`
- AND las `key` coinciden letra por letra con los valores del enum `ActividadVulnerable`

#### Scenario: Cache header presente

- GIVEN un cliente llama al endpoint
- WHEN el servidor responde 200
- THEN incluye header `Cache-Control: public, max-age=86400, immutable`

#### Scenario: Endpoint público sin autenticación

- GIVEN un cliente sin header `Authorization`
- WHEN envía `GET /pld-api/auth-users/catalogs/vulnerable-activities`
- THEN responde HTTP 200

#### Scenario: Drift detection — test enforcement

- GIVEN se agrega un valor nuevo al enum `ActividadVulnerable` sin actualizar el catálogo
- WHEN se ejecuta `nx run auth-users:test`
- THEN el test falla con mensaje claro indicando el desync

### Requirement: Endpoint de países soportados

El sistema DEBE exponer `GET /catalogs/countries` retornando los países donde el sistema PLD soporta sujetos obligados.

#### Scenario: Respuesta exitosa con seed inicial

- GIVEN el gateway auth-users está corriendo
- WHEN un cliente envía `GET /pld-api/auth-users/catalogs/countries`
- THEN el servidor responde HTTP 200
- AND el body es un array con al menos 1 objeto
- AND cada objeto tiene shape `{ code: string, name: string }`
- AND la entrada con `code='MX'` está presente con `name='México'`

#### Scenario: Cache header

- GIVEN llamada al endpoint
- THEN response incluye `Cache-Control: public, max-age=86400, immutable`

### Requirement: Endpoint de estructura administrativa por país

El sistema DEBE exponer `GET /catalogs/administrative-divisions/:countryCode/structure` retornando los niveles administrativos del país.

#### Scenario: Estructura para México

- GIVEN un cliente envía `GET /pld-api/auth-users/catalogs/administrative-divisions/MX/structure`
- WHEN el servidor procesa la petición
- THEN responde HTTP 200
- AND el body tiene shape `{ countryCode: 'MX', levels: AdministrativeLevel[] }`
- AND levels contiene exactamente 3 entradas con fieldName `state`, `municipality`, `neighborhood` en ese orden
- AND cada nivel incluye `{ level, fieldName, label }` con label en español

#### Scenario: País no soportado

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/XX/structure`
- WHEN XX no está en la lista de países soportados
- THEN responde HTTP 404

### Requirement: Endpoint de entradas administrativas filtradas

El sistema DEBE exponer `GET /catalogs/administrative-divisions/:countryCode/entries?level=N&parentCode=X` retornando entradas filtradas en cascada.

#### Scenario: Nivel 1 sin parentCode

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/MX/entries?level=1`
- WHEN el servidor procesa
- THEN responde HTTP 200
- AND el body es un array de exactamente 32 entradas (estados de MX)
- AND cada entrada tiene `parentCode === null`

#### Scenario: Nivel 2 con parentCode

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/MX/entries?level=2&parentCode=09`
- WHEN el servidor procesa
- THEN responde HTTP 200
- AND el body es un array de entradas con `level=2` y `parentCode='09'`
- AND el array NO está vacío (CDMX tiene 16 alcaldías)

#### Scenario: Nivel > 1 sin parentCode

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/MX/entries?level=2`
- WHEN no se proporciona parentCode
- THEN responde HTTP 400 con mensaje "parentCode required for level > 1"

#### Scenario: País no soportado

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/XX/entries?level=1`
- WHEN XX no existe
- THEN responde HTTP 404

### Requirement: Endpoints públicos del módulo catalogs

Los endpoints `vulnerable-activities`, `countries`, y `administrative-divisions/*` NO DEBEN requerir token JWT.

#### Scenario: Acceso anónimo

- GIVEN cualquiera de los 4 endpoints sin header `Authorization`
- THEN responde HTTP 200
