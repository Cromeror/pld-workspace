# Specs delta: catalogs

Extiende [openspec/specs/catalogs/spec.md](../../specs/catalogs/spec.md) con nuevos Requirements para los 3 catálogos del change.

## Requirements added

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

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/MX/entries?level=2&parentCode=AGU`
- WHEN el servidor procesa
- THEN responde HTTP 200
- AND el body es un array de entradas con `level=2` y `parentCode='AGU'`
- AND el array NO está vacío (Aguascalientes tiene municipios)

#### Scenario: Nivel > 1 sin parentCode

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/MX/entries?level=2`
- WHEN no se proporciona parentCode
- THEN responde HTTP 400 con mensaje "parentCode required for level > 1"

#### Scenario: País no soportado

- GIVEN un cliente envía `GET /catalogs/administrative-divisions/XX/entries?level=1`
- WHEN XX no existe
- THEN responde HTTP 404

### Requirement: Endpoints públicos

Los endpoints `vulnerable-activities`, `countries`, y `administrative-divisions/*` NO DEBEN requerir token JWT.

#### Scenario: Acceso anónimo

- GIVEN cualquiera de los 4 endpoints sin header `Authorization`
- THEN responde HTTP 200
