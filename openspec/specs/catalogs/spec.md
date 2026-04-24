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
