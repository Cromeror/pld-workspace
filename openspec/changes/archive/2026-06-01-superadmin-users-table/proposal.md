# Proposal: superadmin-users-table

## Problema

Queremos implementar una pantalla de gestión de usuarios para el SUPERADMIN que muestre una tabla con los usuarios WORKSPACE_ADMIN registrados en el sistema.

## Contexto del diseño (imagen compartida)
La tabla tiene las siguientes columnas:
- NOMBRE COMPLETO (con avatar de iniciales)
- RFC
- NO. CELULAR
- CORREO
- TIPO DE SUJETO (badge/chip, ej: 'Auxiliar')
- ACCIONES (iconos ojo 👁️ y basura 🗑️)

Footer con paginación: 'Mostrar 10 registros', botones Anterior / páginas numeradas / Siguiente, y 'Página 1 de 10'.

## Requerimientos

### Backend (pld-api)
- Endpoint `GET /admin/users` protegido con JWT + rol SUPERADMIN.
- Devuelve lista paginada de usuarios con rol WORKSPACE_ADMIN.
- Response shape: `{ data: User[], total: number, page: number, pageSize: number }`.
- El SUPERADMIN NO puede ver ni crear Auxiliares — el endpoint solo devuelve WORKSPACE_ADMIN, nunca AUXILIARY.
- El endpoint de registro de auxiliares `POST /registration/auxiliaries` debe rechazar JWTs de SUPERADMIN con 403 (ya lo hace por RolesGuard, confirmar).
- Agregar pruebas e2e para los endpoints nuevos.

### Frontend (pld-web)
- Nueva página accesible solo para SUPERADMIN.
- Implementar con AG Grid (ya instalado o instalarlo) o componente tabla reutilizable si ya existe.
- Columnas según el diseño: avatar de iniciales, nombre completo, RFC, celular, correo, tipo de sujeto (badge), acciones (ver, eliminar).
- Paginación server-side: 'Mostrar N registros' + navegación numérica.
- Pruebas e2e (Playwright, ya configurado en pld-web/e2e/).

## Restricciones
- El SUPERADMIN NO puede ver ni crear auxiliares — tanto en BE (endpoint filtrado) como en FE (sin ruta ni botón de 'Registrar auxiliar').
- Usar AG Grid para la tabla si no hay un componente tabla suficientemente capaz en el sistema de diseño actual.
- No afectar flujos existentes de registro de sujetos obligados ni de auxiliares.

## Archivos relevantes conocidos
- `pld-api/apps/auth-users/src/admin/registration/` — módulo admin existente con patrón a seguir
- `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx` — única página admin actual
- `pld-web/e2e/` — configuración Playwright existente
- `pld-web/src/routes/index.tsx` — router con RoleProtectedRoute
- `pld-api/apps/auth-users/src/registration/auxiliaries/auxiliaries.controller.ts` — endpoint auxiliares (verificar guard)

Quiero agregar dos features a la tabla de usuarios del SuperAdmin:

## 1. Ordenamiento por columnas
El usuario debe poder hacer click en el header de cualquier columna (NOMBRE COMPLETO/RAZÓN SOCIAL, CURP/RFC, NO. TELÉFONO, CORREO, TIPO DE SUJETO) para ordenar ascendente/descendente. La columna ACCIONES no tiene ordenamiento. El ícono es una flecha SVG personalizada (flecha vertical con punta abajo, color #535862) que se muestra junto al label de cada columna ordenable. El ordenamiento debe ser server-side (pasar sortBy y sortOrder al endpoint GET /admin/users).

## 2. Búsqueda por texto
Agregar un input de búsqueda (placeholder 'Buscar') con ícono de lupa a la izquierda, alineado a la derecha junto al botón 'Columnas' existente. La búsqueda filtra por nombre, email, RFC o CURP. Debe ser server-side (pasar parámetro search al endpoint). Debounce de ~400ms antes de disparar la query.

## Contexto actual
Ya existe SuperAdminUsersPage.tsx con PaginatedTable (PrimeReact DataTable + paginación custom TablePagination). El endpoint GET /admin/users ya existe con page y limit. Hay que extenderlo con los parámetros sortBy, sortOrder y search.

## Estado

pm_tech_review
