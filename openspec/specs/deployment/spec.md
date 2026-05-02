# Deployment Specification

## Purpose

Eliminar el proceso, container, configuración y scripts de `apps/catalogs` tras la consolidación. Reducir el footprint operativo.

## Requirements

### Requirement: Eliminación de la app `catalogs`

El directorio `apps/catalogs/` DEBE ser eliminado del repositorio.

#### Scenario: No existe el directorio

- GIVEN el repositorio en estado post-cambio
- WHEN se ejecuta `ls apps/`
- THEN `catalogs` no aparece en el listado

### Requirement: Limpieza de scripts y targets

El proyecto NO DEBE contener referencias a `catalogs` como target de Nx, script npm o alias de tsconfig.

#### Scenario: package.json sin script `catalogs:debug`

- GIVEN `package.json` post-cambio
- WHEN se inspecciona la sección `scripts`
- THEN no existe clave `catalogs:debug`
- AND el script `build` no invoca `nx run catalogs:build`

#### Scenario: tsconfig.base.json sin alias catalogs-app

- GIVEN `tsconfig.base.json` post-cambio
- WHEN se inspeccionan las `paths`
- THEN no existe ninguna ruta que apunte a `apps/catalogs/`

> Nota: el alias `@pld-api/catalogs` (libs) se mantiene porque es la lib puente de enums, no la app.

### Requirement: Limpieza de docker-compose

Los archivos `docker-compose.yml` y `docker-compose.dev.yml` NO DEBEN contener el servicio `catalogs` ni referencias a sus imágenes, puertos, labels o volúmenes.

#### Scenario: docker-compose.yml sin servicio catalogs

- GIVEN `docker-compose.yml` post-cambio
- WHEN se parsea el archivo
- THEN la clave `services.catalogs` no existe
- AND no hay labels de Traefik apuntando a `pld-catalogs`

#### Scenario: docker-compose.dev.yml sin bloque catalogs

- GIVEN `docker-compose.dev.yml` post-cambio
- WHEN se parsea
- THEN no aparecen bloques (activos o comentados) para `catalogs`

### Requirement: Cero regresiones en login

El endpoint `POST /auth/login` DEBE seguir respondiendo HTTP 201 con JWT válido para credenciales correctas.

#### Scenario: Login con credenciales válidas funciona

- GIVEN MySQL corriendo con usuario `admin@pld.com` (password `Admin123!`, role `SUPERADMIN`)
- AND `docker compose -f docker-compose.dev.yml up -d` ejecutado
- WHEN se envía `POST /pld-api/auth-users/auth/login` con credenciales válidas
- THEN el servidor responde HTTP 201
- AND el body contiene campo `token` con un JWT parseable

### Requirement: Compilación de apps restantes

Los proyectos `auth-users` y `cross` DEBEN compilar sin errores tras el cambio.

#### Scenario: nx build en auth-users

- GIVEN el repositorio post-cambio
- WHEN se ejecuta `nx run auth-users:build --skip-nx-cache`
- THEN el comando termina con exit code 0
- AND genera `dist/apps/auth-users/main.js`

<!-- Scenario "nx build en cross" eliminado 2026-05-02 — apps/cross fue removida del monorepo. -->

