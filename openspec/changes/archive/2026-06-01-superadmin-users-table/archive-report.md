## Archive Report: superadmin-users-table

**Archived**: 2026-06-01
**Archive path**: openspec/changes/archive/2026-06-01-superadmin-users-table/

### Summary

Tabla de gestión de usuarios WORKSPACE_ADMIN para el SUPERADMIN.

Implementado:
- BE: GET /admin/users paginado, filtrado por WORKSPACE_ADMIN, protegido con JWT SUPERADMIN
- BE: sort server-side por fullName/curp/rfc/phone/email/activityType con ORDER BY dinámico
- BE: búsqueda server-side con LIKE sobre nombre, email, RFC, CURP
- FE: SuperAdminUsersPage con PaginatedTable, sort headers clickeables, input búsqueda con debounce 400ms
- FE: ruta /admin/users protegida con RoleProtectedRoute SUPERADMIN
- E2E: specs Playwright para tabla y role guard

### Specs Synced
No había delta specs/ — change sin specs formales.

### Archive Contents
- proposal.md ✅
- tasks.md ✅ (33/36 completas — 3 de smoke manual pendientes, no bloqueantes)
- tech-review.md ✅
- data.yaml ✅
- verify-report.md ✅
- iterations/ ✅

### Issues Corregidos Antes del Archive
1. AdminUsersModule estaba vacío — corregido con controllers/providers
2. Phase 8 BE (sort/search) no implementada — implementada en port/adapter/DTO/service/controller
3. Tests BE no compilaban — fixture con contactCountryCode faltante corregido
4. FE: prop totalRecords → total en PaginatedTable/TablePagination

### SDD Cycle Complete
Change planificado, implementado, verificado y archivado.
