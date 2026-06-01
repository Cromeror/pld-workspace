## Verification Report: superadmin-users-table

_Generado: 2026-06-01_

### Completeness

| Métrica | Valor |
|---------|-------|
| Tasks totales | 36 |
| Tasks completas | 33 |
| Tasks incompletas | 3 |

Incompletas (todas en Phase 9 — Smoke manual):
- [ ] 9.1 [Cross] Stack BE + Web up. Login como admin@pld.com → navegar `/admin/users` → tabla visible.
- [ ] 9.2 [Cross] Verificar que la tabla solo muestra usuarios con tipo WORKSPACE_ADMIN (no AUXILIARY, no SUPERADMIN).
- [ ] 9.3 [Web] `yarn build` en `pld-web` verde.

---

### Build & Tests

**BE Build**: ✅ `NX Successfully ran target build for project auth-users`
**FE Type Check**: ⚠️ Errores presentes (ver detalle abajo)
**BE Tests**: ❌ 1 suite failed — 0 tests ran (error de compilación TS en fixture)

---

### Correctness Matrix

| Componente | Estado | Notas |
|-----------|--------|-------|
| BE: `UsersPort.listWorkspaceAdminsPaginated` con sort+search | ❌ | Firma solo tiene `(page, limit)` — sin `sortBy`, `sortOrder`, `search` |
| BE: `UsersAdapter.listWorkspaceAdminsPaginated` con ORDER BY dinámico y LIKE | ❌ | Implementado sin sort/search — solo `ORDER BY created_at DESC` fijo |
| BE: `ListUsersQueryDto` con sortBy/sortOrder/search | ❌ | Solo tiene `page` y `limit` — faltan los 3 campos de Phase 8 |
| BE: `ListUsersResponseDto` con UserSummaryDto | ✅ | Estructura completa: id, username, role, createdAt, contact, profile, responsible, activities |
| BE: `AdminUsersService.listWorkspaceAdmins` con sort+search params | ❌ | Solo acepta `(page, limit)` — no propaga sort/search |
| BE: `AdminUsersController` — guards SUPERADMIN + handler @Get() | ✅ | Guards correctos, handler llama al service |
| BE: `AdminUsersModule` — wiring completo | ❌ CRÍTICO | El módulo está vacío (`@Module({})`) — no tiene `providers`, `controllers`, ni `imports`. La app buildea pero el endpoint no funciona en runtime. |
| BE: `AdminModule` importa `AdminUsersModule` | ✅ | Correctamente importado en `admin.module.ts` |
| FE: `src/types/admin.ts` — UserSummary + ListUsersResponse | ✅ | Tipos completos incluyendo contact/profile/responsible/activities |
| FE: `adminUsersService.listUsers` con sortBy/sortOrder/search | ✅ | Acepta los 5 params, los pasa en query params |
| FE: `useListUsers` con sortBy/sortOrder/search | ✅ | queryKey incluye los 3 params, los propaga al service |
| FE: `SuperAdminUsersPage` — sorting state + debounce search | ✅ | Estado sortBy/sortOrder, debounce 400ms, reset page=1 al cambiar |
| FE: `PaginatedTable` — `header: string \| ReactNode` | ✅ | Tipo cambiado correctamente |
| FE: `SortHeader` con ícono SVG asc/desc (color #535862) | ✅ | SVG correcto, toggle asc/desc implementado |
| FE: Input búsqueda con lupa izquierda, debounce 400ms | ✅ | Implementado con setTimeout/clearTimeout |
| FE: Ruta `/admin/users` protegida SUPERADMIN | ✅ | `RoleProtectedRoute requiredRoles={[UserRole.SUPERADMIN]}` |
| FE: e2e `superadmin-users-table.spec.ts` | ✅ | Cubre: tabla visible, filas, headers, paginador, sort click |
| FE: e2e `admin-users-role-guard.spec.ts` | ✅ | Verifica que WORKSPACE_ADMIN no accede a `/admin/users` |
| BE: e2e spec `admin-users.spec.ts` | ⚠️ | Archivo existe y tiene cobertura amplia, pero **no compila** — `contactCountryCode` faltante en fixtures de test |

---

### Issues Found

**CRITICAL** (bloquea archive):

1. **`AdminUsersModule` vacío** (`apps/auth-users/src/admin/users/admin-users.module.ts`). El módulo tiene `@Module({})` sin `controllers: [AdminUsersController]`, sin `providers: [AdminUsersService]`, sin `imports: [DomainAuthUsersModule]`. El build pasa porque Webpack no valida el wiring en tiempo de compilación, pero en runtime el endpoint GET /admin/users devuelve 404. Debe corregirse antes de archive.

2. **`UsersPort.listWorkspaceAdminsPaginated` sin sort/search** — La firma en `packages/domain-auth-users/src/ports/users.port.ts` (línea 117) solo declara `(page: number, limit: number)`. Las tareas 8.1–8.4 marcan como completadas la extensión del port, el adapter, el DTO y el controller, pero ninguno de estos tiene los parámetros. El endpoint no soporta sort ni search en runtime.

3. **`ListUsersQueryDto` sin sortBy/sortOrder/search** — `apps/auth-users/src/admin/users/dto/list-users-query.dto.ts` solo tiene `page` y `limit`. Tarea 8.2 marcada `[x]` incorrectamente.

4. **BE tests fallando** — `admin-users.spec.ts` no compila porque `WorkspaceAdminRow` ahora exige `contactCountryCode` (campo agregado al tipo) pero las fixtures del test no lo incluyen. Error: `Property 'contactCountryCode' is missing`. Ningún test se ejecutó (0 tests, 1 suite failed).

**WARNING** (debería corregirse):

5. **FE TypeScript errors** — `yarn tsc -b --noEmit` retorna 2 con errores en archivos fuera del alcance de este change (`AddVulnerableActivityModal`, `ReviewStep`, `ReportingEntityRegistrationPage`, `AuxiliaryRegistrationPage`). El único error en archivos del change es `TablePagination.tsx(50,63): TS6133: 'totalRecords' is declared but its value is never read` — variable recibida en props pero no usada en el cuerpo de la función. Tarea 8.10 marcada `[x]` incorrectamente.

6. **Cambios sin commitear en pld-web** — Git status muestra `MM src/pages/admin/SuperAdminUsersPage.tsx`, `MM src/queries/adminUsersQueries.ts`, `MM src/services/adminUsersService.ts`, `M src/types/admin.ts`, ` M src/components/molecules/PaginatedTable/index.tsx`, y archivos e2e sin stage. No es bloqueante pero los cambios deben commitearse antes del archive.

7. **`UsersAdapter` sin search/sort** — La implementación en `packages/domain-auth-users/src/adapters/users.adapter.ts` usa `ORDER BY u.created_at DESC` fijo y no tiene cláusula LIKE. Tarea 8.3 marcada `[x]` incorrectamente.

**SUGGESTION**:

8. La columna RFC en la propuesta original usa `profileType` como fuente de datos — en la implementación actual el RFC se obtiene de `row.profile.rfc` (que ya viene de `physical_person_profile.rfc` o `moral_person_profile.rfc` según el adapter). El campo `curp` viene de `row.responsible.curp` que en PM es el CURP del `compliance_responsible`. Esto es correcto según el diseño del adapter, pero conviene documentarlo.

---

### Verdict

**FAIL**

El change tiene 4 issues críticos: el módulo NestJS está vacío (endpoint 404 en runtime), los parámetros sort/search de Phase 8 no están implementados en BE (port, adapter, DTO, service marcados incorrectamente como `[x]`), y los tests BE no compilan. La funcionalidad FE de sort/search está completa pero no tiene contraparte BE funcionando.
