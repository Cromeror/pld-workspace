# Tasks: Consolidar `catalogs` en `auth-users` con Cache HTTP

## Phase 1: Interceptor local en ambos gateways

- [x] 1.1 Copiar `libs/core/src/interceptors/http.error.interceptor.ts` a `apps/auth-users/src/shared/http-error.interceptor.ts` (contenido idéntico)
- [x] 1.2 Copiar el mismo archivo a `apps/cross/src/shared/http-error.interceptor.ts`
- [x] 1.3 En `apps/auth-users/src/auth-users.module.ts`, cambiar `import { HttpErrorInterceptor } from '@pld-api/core'` por import relativo a `./shared/http-error.interceptor`
- [x] 1.4 En `apps/auth-users/src/main.ts`, mismo cambio de import
- [x] 1.5 En `apps/cross/src/app.module.ts`, cambiar el import a `./shared/http-error.interceptor`
- [x] 1.6 En `apps/cross/src/main.ts`, mismo cambio de import
- [x] 1.7 Reiniciar contenedor y verificar login HTTP 201 + shape de error 401 mantiene los 6 campos esperados

## Phase 2: Módulo catalogs dentro de auth-users

- [x] 2.1 Crear `apps/auth-users/src/catalogs/catalogs.data.ts` con `CatalogEntry` interface + `CATALOGS_BENEFICIARIO` (copiar labels literalmente desde `apps/catalogs/src/beneficiario/beneficiario.adapter.ts`)
- [x] 2.2 Crear `apps/auth-users/src/catalogs/catalogs.controller.ts` con `@Controller('catalogs')`, método `beneficiario()` con `@Get('beneficiario')` y `@Header('Cache-Control', 'public, max-age=86400, immutable')`
- [x] 2.3 Crear `apps/auth-users/src/catalogs/catalogs.module.ts` registrando el controller
- [x] 2.4 En `apps/auth-users/src/auth-users.module.ts`, agregar `CatalogsModule` a `imports`

## Phase 3: Test unitario

- [x] 3.1 Crear `apps/auth-users/src/catalogs/catalogs.controller.spec.ts` con 2 tests: (a) `CATALOGS_BENEFICIARIO` tiene 3 entradas con keys `beneficiario-1/2/3`; (b) `controller.beneficiario()` retorna el mismo array
- [x] 3.2 Ejecutar `docker run --rm -v $(pwd):/app -w /app node:20-alpine npx nx run auth-users:test --skip-nx-cache` y verificar exit code 0 + tests corriendo (no `passWithNoTests`)

## Phase 4: Verificación pre-eliminación

- [x] 4.1 Reiniciar `auth-users` y con curl validar `GET http://localhost:9001/pld-api/auth-users/catalogs/beneficiario` responde 200
- [x] 4.2 Con `curl -i` confirmar header `Cache-Control: public, max-age=86400, immutable` en la respuesta
- [x] 4.3 Confirmar body idéntico al viejo endpoint (3 objetos, keys, labels)
- [x] 4.4 Validar `POST /auth/login` sigue 201 con JWT (regresión)
- [x] 4.5 Ejecutar `nx run cross:build --skip-nx-cache` y verificar exit 0

## Phase 5: Eliminación de la app vieja

- [x] 5.1 Borrar directorio `apps/catalogs/` completo (`rm -rf`)
- [x] 5.2 En `docker-compose.yml`, remover el servicio `catalogs` y labels de Traefik asociadas
- [x] 5.3 En `docker-compose.dev.yml`, remover el bloque comentado de `catalogs`
- [x] 5.4 En `package.json`, remover script `catalogs:debug` y quitar `&& npx nx run catalogs:build ...` del script `build`
- [x] 5.5 Revisar `tsconfig.base.json` y `nx.json` por referencias residuales a la app `catalogs` (no al lib homónimo). Si existen, removerlas. (nx.json limpio; tsconfig solo tiene la lib puente, se mantiene)
- [x] 5.6 Ejecutar `pnpm install` para regenerar el lockfile si es necesario

## Phase 6: Verificación final

- [x] 6.1 `docker compose -f docker-compose.dev.yml down` + `up --build` y validar que levantan solo `pld-api-dev-mysql` + `pld-api-dev-auth-users`
- [x] 6.2 Login HTTP 201 + catalogs HTTP 200 con header correcto
- [x] 6.3 `nx run auth-users:build --skip-nx-cache` → exit 0
- [x] 6.4 `nx run cross:build --skip-nx-cache` → exit 0
- [x] 6.5 Actualizar checklist interno (`.claude/CHECKLIST_MIGRACION.md`) marcando Fase 2 como completada
