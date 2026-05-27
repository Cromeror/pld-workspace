# Tasks: superadmin-users-table

> Refines: [proposal.md](./proposal.md), [tech-review.md](./tech-review.md)
>
> Marcas: [BE] = pld-api, [Web] = pld-web
> Sin tests unitarios BE — verificación e2e + smoke manual.

## Phase 1: BE — Port + Adapter

- [x] 1.1 [BE] En `pld-api/packages/domain-auth-users/src/ports/users.port.ts` agregar firma `listWorkspaceAdminsPaginated(page: number, limit: number): Promise<{ data: UserEntity[]; total: number }>`.
- [x] 1.2 [BE] En `pld-api/packages/domain-auth-users/src/adapters/users.adapter.ts` implementar `listWorkspaceAdminsPaginated(page, limit)` usando TypeORM `findAndCount` con `where: { role: UserRole.WORKSPACE_ADMIN }`, `skip: (page - 1) * limit`, `take: limit`, `order: { createdAt: 'DESC' }`.
- [x] 1.3 [BE] Exportar el nuevo método desde el barrel de `@pld-api/domain-auth-users` si aplica.

## Phase 2: BE — DTOs

- [x] 2.1 [BE] Crear `pld-api/apps/auth-users/src/admin/users/dto/list-users-query.dto.ts` con `ListUsersQueryDto`: `page?: number` (default 1, min 1) y `limit?: number` (default 10, min 1, max 100), usando `class-validator` + `@Type(() => Number)` de `class-transformer`.
- [x] 2.2 [BE] Crear `pld-api/apps/auth-users/src/admin/users/dto/list-users-response.dto.ts` con `UserSummaryDto` (id, firstName, paternalSurname, maternalSurname, email, phone, username, profileType, role, createdAt — sin passwordHash ni passwordChangedAt) y `ListUsersResponseDto` (`{ data: UserSummaryDto[]; total: number; page: number; pageSize: number }`).

## Phase 3: BE — Service + Controller + Module

- [x] 3.1 [BE] Crear `pld-api/apps/auth-users/src/admin/users/admin-users.service.ts` con `AdminUsersService` que inyecta `UsersPort` (token `USERS_PORT`) y expone `listWorkspaceAdmins(page, limit)` que llama al adapter y mapea a `ListUsersResponseDto`.
- [x] 3.2 [BE] Crear `pld-api/apps/auth-users/src/admin/users/admin-users.controller.ts` con `@Controller('admin/users')`, `@UseGuards(JwtAuthGuard, RolesGuard)`, `@Roles(UserRole.SUPERADMIN)` a nivel clase. Handler `@Get()` con `@Query() query: ListUsersQueryDto`, llama `adminUsersService.listWorkspaceAdmins(query.page, query.limit)`.
- [x] 3.3 [BE] Crear `pld-api/apps/auth-users/src/admin/users/admin-users.module.ts` con `AdminUsersModule` que importa `DomainAuthUsersModule` (para `UsersPort`), providers `[AdminUsersService]`, controllers `[AdminUsersController]`.
- [x] 3.4 [BE] Registrar `AdminUsersModule` en `pld-api/apps/auth-users/src/admin/admin.module.ts`.
- [x] 3.5 [BE] `pnpm exec nx run auth-users:build` debe pasar verde.

## Phase 4: BE — E2E tests

- [x] 4.1 [BE] Crear `pld-api/apps/auth-users/src/admin/users/admin-users.spec.ts` (o en el directorio e2e del proyecto si existe). Casos:
  - `GET /pld-api/auth-users/admin/users` sin JWT → 401.
  - `GET /pld-api/auth-users/admin/users` con JWT WORKSPACE_ADMIN → 403.
  - `GET /pld-api/auth-users/admin/users` con JWT SUPERADMIN → 200 con `{ data: [...], total, page, pageSize }`, todos los items tienen `role: 'WORKSPACE_ADMIN'`.
  - `GET /pld-api/auth-users/admin/users?page=1&limit=5` → `pageSize: 5`, `data.length <= 5`.
- [x] 4.2 [BE] Confirmar que `POST /pld-api/auth-users/registration/auxiliaries` con JWT SUPERADMIN → 403 (guard existente). Agregar caso al e2e si no existe ya.

## Phase 5: Web — Service + Query

- [x] 5.1 [Web] En `pld-web/src/types/admin.ts` (crear si no existe) agregar `UserSummary` y `ListUsersResponse` (mirror del response BE).
- [x] 5.2 [Web] Crear `pld-web/src/services/adminUsersService.ts` con `listUsers(page: number, limit: number): Promise<ListUsersResponse>` usando `axiosInstance.get('/admin/users', { params: { page, limit } })`.
- [x] 5.3 [Web] Crear `pld-web/src/queries/adminUsersQueries.ts` con `useListUsers(page: number, limit: number)` — `queryKey: ['admin', 'users', page, limit]`, `queryFn: () => listUsers(page, limit)`, `enabled: true`.

## Phase 6: Web — Página + Ruta

- [x] 6.1 [Web] En `pld-web/src/routes/routes-urls.ts` agregar `ADMIN_USERS: '/admin/users'`.
- [x] 6.2 [Web] Crear `pld-web/src/pages/admin/SuperAdminUsersPage.tsx` con `<DataTable>` de PrimeReact. Columnas según diseño: avatar de iniciales (template custom), nombre completo (firstName + paternalSurname + maternalSurname), RFC (desde `profileType`? — o campo pendiente), celular (phone), correo (email), tipo de sujeto (badge chip con `profileType`), acciones (íconos `Eye` + `Trash2` de lucide-react, disabled/sin handler en este change). Paginación lazy server-side con `paginator`, `rows`, `totalRecords`, `onPage`. Selector "Mostrar N registros" con `rowsPerPageOptions={[5, 10, 20, 50]}`.
- [x] 6.3 [Web] En `pld-web/src/routes/index.tsx` agregar ruta lazy para `SuperAdminUsersPage` en `/admin/users`, dentro del bloque `RoleProtectedRoute requiredRoles={[UserRole.SUPERADMIN]}`.
- [x] 6.4 [Web] `yarn tsc -b --noEmit` en `pld-web` debe pasar.

## Phase 7: Web — E2E Playwright

- [x] 7.1 [Web] Crear `pld-web/e2e/tests/authenticated/superadmin-users-table.spec.ts` reutilizando el storageState existente de SUPERADMIN (`e2e/fixtures/.auth.json` o equivalente). Casos:
  - Navegar a `/admin/users` → tabla visible (`[data-testid="users-table"]` o selector de DataTable).
  - Al menos 1 fila visible con columnas: nombre, correo, tipo de sujeto, acciones.
  - Selector "Mostrar N registros" cambia el número de filas mostradas.
  - Un usuario WORKSPACE_ADMIN que navega a `/admin/users` → redirige (RoleProtectedRoute).

## Phase 8: Server-side sorting

- [x] 8.1 [BE] Extender `WorkspaceAdminPort.listWorkspaceAdminsPaginated` con `sortBy?: string` y `sortOrder?: 'asc' | 'desc'`.
- [x] 8.2 [BE] Extender `ListUsersQueryDto` con `sortBy` (enum: fullName|curp|rfc|phone|email|activityType) y `sortOrder` (asc|desc).
- [x] 8.3 [BE] Actualizar `UsersAdapter` con ORDER BY dinámico: GROUP BY subquery para custom sort, subquery simple para default createdAt.
- [x] 8.4 [BE] Pasar sort params desde Controller → Service → Port.
- [x] 8.5 [BE] Agregar tests e2e: verifica que sortBy/sortOrder se reenvían al port.
- [x] 8.6 [FE] Cambiar `ColumnDef.header` de `string` a `string | ReactNode` en `PaginatedTable`.
- [x] 8.7 [FE] Extender `adminUsersService.listUsers` y `useListUsers` con sortBy/sortOrder.
- [x] 8.8 [FE] Agregar estado `sortBy`/`sortOrder`, componente `SortHeader` con SVG toggle asc/desc, reset page=1 al cambiar sort.
- [x] 8.9 [BE] `pnpm jest` pasa verde.
- [x] 8.10 [Web] `tsc --noEmit` pasa verde.

## Phase 9: Smoke manual

- [ ] 9.1 [Cross] Stack BE + Web up. Login como admin@pld.com → navegar `/admin/users` → tabla visible.
- [ ] 9.2 [Cross] Verificar que la tabla solo muestra usuarios con tipo WORKSPACE_ADMIN (no AUXILIARY, no SUPERADMIN).
- [ ] 9.3 [Web] `yarn build` en `pld-web` verde.
