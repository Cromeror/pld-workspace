# HTTP Error Formatting Specification

## Purpose

Estandarizar el shape de las respuestas de error HTTP de los gateways NestJS. El interceptor vive localmente en cada app (no importado desde `libs/core`).

## Requirements

### Requirement: Interceptor local en cada gateway

Cada gateway NestJS (`auth-users`, `cross`) DEBE contener su propia copia de `HttpErrorInterceptor` en `apps/<gateway>/src/shared/http-error.interceptor.ts`. Ningún gateway DEBE importar ese símbolo desde `@pld-api/core`.

#### Scenario: auth-users usa la copia local

- GIVEN el archivo `apps/auth-users/src/shared/http-error.interceptor.ts` existe
- WHEN se inspecciona `apps/auth-users/src/auth-users.module.ts` y `main.ts`
- THEN la referencia a `HttpErrorInterceptor` apunta a la ruta local
- AND no existe import `from '@pld-api/core'` para ese símbolo

#### Scenario: cross usa la copia local

- GIVEN el archivo `apps/cross/src/shared/http-error.interceptor.ts` existe
- WHEN se inspecciona `apps/cross/src/app.module.ts` y `main.ts`
- THEN la referencia a `HttpErrorInterceptor` apunta a la ruta local
- AND no existe import `from '@pld-api/core'` para ese símbolo

### Requirement: Shape uniforme de error

Todas las respuestas de error de los gateways DEBEN tener el shape `{ statusCode, message, timestamp, path, method, errorDetails }`, donde `message` y `errorDetails.message` son arrays de strings.

#### Scenario: Error 401 en login inválido mantiene shape

- GIVEN un cliente envía `POST /pld-api/auth-users/auth/login` con password incorrecto
- WHEN el servidor responde
- THEN el body incluye campos `statusCode=401`, `message=["Credenciales inválidas"]`, `timestamp` (ISO 8601), `path`, `method=POST`, `errorDetails.message=["Credenciales inválidas"]`

#### Scenario: Error de validación mantiene shape

- GIVEN un cliente envía body inválido a un endpoint con ValidationPipe activo
- WHEN el servidor responde
- THEN el body incluye `statusCode=400` y los mensajes de validación como array en `message` y `errorDetails.message`

### Requirement: Comportamiento idéntico al interceptor previo

El comportamiento observable (códigos HTTP, shape, logging) DEBE ser bit-a-bit idéntico al `HttpErrorInterceptor` que vive actualmente en `libs/core`.

#### Scenario: Response antes/después son equivalentes

- GIVEN el mismo error es lanzado antes y después del cambio
- WHEN se comparan las respuestas HTTP
- THEN los campos del body son idénticos excepto `timestamp` (siempre distinto)
