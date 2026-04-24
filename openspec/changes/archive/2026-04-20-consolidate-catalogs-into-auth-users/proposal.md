# Proposal: Consolidar `catalogs` en `auth-users` con Cache HTTP

## Intent

`apps/catalogs` es un microservicio de 35 líneas que sirve 1 endpoint con 3 strings hardcodeados. Mantenerlo como proceso independiente implica overhead operativo (container, Dockerfile, build target) sin beneficio: no tiene BD, no escala diferente, no falla de forma aislada útil.

Consolidarlo en `auth-users` elimina un proceso, reduce el footprint de infra y libera cache HTTP del lado del cliente para que el frontend lea sub-milisegundo. Alineado con §9.Fase 2 del plan vigente.

## Scope

### In Scope
- Mover lógica de `apps/catalogs/src/beneficiario/` a `apps/auth-users/src/catalogs/`.
- Exponer `GET /catalogs/beneficiario` dentro del gateway actual.
- Agregar header `Cache-Control: public, max-age=86400, immutable` a los endpoints de catálogos.
- Eliminar `apps/catalogs/` (directorio, entrada en `nx.json`, scripts de `package.json`, servicios en `docker-compose.yml` y `docker-compose.dev.yml`).
- Mover `HttpErrorInterceptor` a `apps/auth-users/src/shared/` y duplicarlo en `apps/cross/src/shared/`. Dejar de importarlo desde `@pld-api/core` en ambas apps.

### Out of Scope
- Unificar los 3 vocabularios de "tipos de control de beneficiario" (bloqueo §11.1, Fase 6).
- Exponer enums de `@pld-api/shared-types` como endpoints HTTP (se evalúa después).
- Migrar `catalogs` a `packages/domain-catalogs` (descartado — no justifica paquete puro).

## Approach

1. Crear módulo NestJS `CatalogsModule` dentro de `auth-users`.
2. Datos estáticos en `catalogs.data.ts` (constante exportada `as const`).
3. Controller con decorador `@Header('Cache-Control', ...)` en cada `@Get`.
4. Route prefix del controller: `catalogs` → endpoints quedan en `/pld-api/auth-users/catalogs/*`.
5. Validar respuesta idéntica (shape, contenido) con `curl`.
6. Eliminar app vieja tras validación.

## Affected Areas

| Area | Impact | Description |
|---|---|---|
| `apps/auth-users/src/catalogs/` | New | Módulo NestJS con controller + data |
| `apps/auth-users/src/shared/http-error.interceptor.ts` | New | Copia local del interceptor (ya no importa de `@pld-api/core`) |
| `apps/auth-users/src/auth-users.module.ts` | Modified | Importa `CatalogsModule` + usa el interceptor local |
| `apps/auth-users/src/main.ts` | Modified | Registra el interceptor desde la ruta local |
| `apps/cross/src/shared/http-error.interceptor.ts` | New | Copia local (simétrico con auth-users) |
| `apps/cross/src/app.module.ts` | Modified | Usa el interceptor local |
| `apps/cross/src/main.ts` | Modified | Registra el interceptor desde la ruta local |
| `apps/catalogs/` | Removed | Directorio completo |
| `docker-compose.yml` | Modified | Remover servicio `catalogs` |
| `docker-compose.dev.yml` | Modified | Remover bloque comentado |
| `nx.json` / `tsconfig.base.json` | Modified | Remover `catalogs` project |
| `package.json` | Modified | Remover script `catalogs:debug`, ajustar `build` |

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Frontend rompe por cambio de base URL | Med | Documentar nueva URL; mantener ruta relativa `catalogs/beneficiario` |
| Header `Cache-Control` cachea datos obsoletos | Low | TTL 24 h es aceptable para labels estáticos; redeploy invalida |
| Ruta actual `localhost:9002/pld-api/catalogs/beneficiario` queda sin servir | Med | Comunicar cambio: nueva URL es `localhost:9001/pld-api/auth-users/catalogs/beneficiario` |

## Rollback Plan

1. `git revert` del commit de consolidación.
2. Restaurar `apps/catalogs/` desde el revert.
3. Reconstruir contenedor de `catalogs` (puerto 9002).
4. Validar `GET http://localhost:9002/pld-api/catalogs/beneficiario` responde 200.

Si el rollback ocurre después de desplegar: comunicar al frontend que vuelva a la URL anterior.

## Dependencies

- `apps/auth-users` operativo y con login funcionando (verificado, HTTP 201).
- `docker-compose.dev.yml` con servicio `auth-users` activo.
- Coordinación con frontend para comunicar el cambio de URL pública.

## Success Criteria

- [ ] `GET http://localhost:9001/pld-api/auth-users/catalogs/beneficiario` responde 200 con las 3 labels originales.
- [ ] Response incluye header `Cache-Control: public, max-age=86400, immutable`.
- [ ] `apps/catalogs/` eliminada completamente (filesystem, Nx, scripts, compose).
- [ ] `curl POST /auth/login` sigue respondiendo 201 con JWT (regresión cero).
- [ ] `docker compose -f docker-compose.dev.yml up --build` levanta solo MySQL + auth-users.
- [ ] Tiempo de respuesta del endpoint cacheado: <5 ms server-side en segunda llamada.
- [ ] `HttpErrorInterceptor` reside en `apps/auth-users/src/shared/` y `apps/cross/src/shared/`; ningún import activo desde `@pld-api/core` de ese símbolo.
- [ ] `apps/cross` compila y arranca sin errores (verificado con `nx run cross:build`).
