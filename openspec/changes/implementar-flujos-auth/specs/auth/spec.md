# Delta for Auth

> Esta delta agrega un endpoint nuevo `GET /auth/me/workspaces` que devuelve las registrations `COMPLETED` del usuario autenticado. Habilita al FE a decidir si mostrar el menú-item de switch sin tener que llamar al endpoint de switch y manejar 403.

## ADDED Requirements

### Requirement: GET /auth/me/workspaces — workspaces COMPLETED del usuario autenticado

El sistema MUST exponer `GET /pld-api/auth-users/auth/me/workspaces` protegido con `JwtAuthGuard`. El endpoint MUST retornar la lista de workspaces (registrations en estado `COMPLETED`) a los que el usuario identificado por el claim `sub` puede acceder, expresados como pares `{ activityType, workspaceId }`.

El response MUST incluir el header `Cache-Control: private, no-store`.

Para `SUPERADMIN` el endpoint MUST retornar lista vacía (`workspaces: []`), porque `SUPERADMIN` no pertenece a workspaces.

El endpoint MUST NOT requerir parámetros de query ni body. La identidad se resuelve exclusivamente desde el JWT.

#### INV-1: WORKSPACE_ADMIN con dos registrations COMPLETED retorna ambas

- GIVEN un usuario `WORKSPACE_ADMIN` con dos registrations COMPLETED, una `NOTARY` y otra `REAL_ESTATE`
- WHEN un cliente envía `GET /auth/me/workspaces` con un JWT válido de ese usuario
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL ser `{ workspaces: [{ activityType: 'NOTARY', workspaceId: '<uuid>' }, { activityType: 'REAL_ESTATE', workspaceId: '<uuid>' }] }`
- AND el header `Cache-Control: private, no-store` SHALL estar presente

#### INV-2: WORKSPACE_ADMIN con una sola registration COMPLETED retorna solo esa

- GIVEN un usuario `WORKSPACE_ADMIN` con una registration COMPLETED `NOTARY` y ninguna otra
- WHEN un cliente envía `GET /auth/me/workspaces`
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL ser `{ workspaces: [{ activityType: 'NOTARY', workspaceId: '<uuid>' }] }`

#### INV-3: SUPERADMIN retorna lista vacía

- GIVEN un usuario `SUPERADMIN`
- WHEN un cliente envía `GET /auth/me/workspaces` con un JWT válido de ese usuario
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL ser `{ workspaces: [] }`

#### INV-4: AUXILIARY retorna sus registrations COMPLETED si aplica

- GIVEN un usuario `AUXILIARY` con al menos una registration COMPLETED accesible
- WHEN un cliente envía `GET /auth/me/workspaces`
- THEN el servidor SHALL responder HTTP 200
- AND el body SHALL contener las registrations COMPLETED accesibles para ese AUXILIARY

#### INV-5: Token ausente o inválido retorna 401

- GIVEN un cliente no envía header `Authorization` o envía un JWT inválido / expirado / con firma manipulada
- WHEN envía `GET /auth/me/workspaces`
- THEN el servidor SHALL responder HTTP 401
- AND el body SHALL seguir el shape de error uniforme

#### INV-6: Registrations no COMPLETED no aparecen en el listado

- GIVEN un usuario con dos registrations: una `COMPLETED` (NOTARY) y otra `IN_PROGRESS` (REAL_ESTATE)
- WHEN un cliente envía `GET /auth/me/workspaces`
- THEN el body SHALL contener únicamente la registration COMPLETED (NOTARY)
- AND la registration `IN_PROGRESS` SHALL NOT aparecer
