# Delta for Auth

## MODIFIED Requirements

### Requirement: CurrentUserDto — English property names

`GET /auth/me` MUST return a `CurrentUserDto` with English property names. MUST NOT return Spanish-named equivalents.

| Property | Type | Notes |
|---|---|---|
| `id` | `string` | Unchanged |
| `email` | `string` | Unchanged |
| `role` | `UserRole` | English enum, unchanged from prior change |
| `profileType` | `ProfileType \| null` | Unchanged |
| `profileId` | `string \| null` | Unchanged |
| `lastLoginAt` | `string \| null` | Unchanged |
| `mustChangePassword` | `boolean` | Unchanged |
| `firstName` | `string` | **Renamed** from `nombre` |
| `paternalSurname` | `string \| null` | **Renamed** from `apellidoPaterno` |
| `maternalSurname` | `string \| null` | **Renamed** from `apellidoMaterno` |
| `phone` | `string \| null` | **Renamed** from `telefono` |
| `active` | `boolean` | **Renamed** from `activo` |

#### INV-1: GET /auth/me returns English DTO properties

- GIVEN a valid JWT for an active user (post-migration)
- WHEN the client sends `GET /auth/me`
- THEN the server SHALL respond HTTP 200
- AND the response body SHALL contain `firstName`, `paternalSurname`, `maternalSurname`, `phone`, `active`
- AND the response body SHALL NOT contain `nombre`, `apellidoPaterno`, `apellidoMaterno`, `telefono`, `activo`

#### INV-2: All other DTO properties unchanged

- GIVEN a valid JWT for an active user
- WHEN the client sends `GET /auth/me`
- THEN `id`, `email`, `role`, `profileType`, `profileId`, `lastLoginAt`, `mustChangePassword` SHALL be present and unmodified in shape

#### INV-3: POST /auth/login contract unchanged

- GIVEN valid credentials
- WHEN the client sends `POST /auth/login`
- THEN the server SHALL respond HTTP 201 with `{ token: string }`
- AND the JWT payload SHALL NOT contain `firstName`, `paternalSurname`, `maternalSurname`, `phone`, or `active`

#### INV-4: Cache-Control header preserved

- GIVEN a valid JWT
- WHEN the client sends `GET /auth/me`
- THEN the response SHALL include `Cache-Control: private, no-store`
