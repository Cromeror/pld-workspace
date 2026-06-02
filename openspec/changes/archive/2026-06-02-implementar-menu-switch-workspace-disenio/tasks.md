# Tasks — implementar-menu-switch-workspace-disenio

## Phase 1: Shared config

- [x] 1.1 Crear `src/config/activityLabels.ts` con `ACTIVITY_LABELS: Record<ActivityType, string>` exportado (`NOTARY → "Notario"`, `REAL_ESTATE → "Inmobiliaria"`)
- [x] 1.2 Actualizar `src/routes/routes-urls.ts` — agregar `PROFILE: "/profile"`
- [x] 1.3 Crear `src/pages/ProfilePage.tsx` — página placeholder vacía (patrón de otras páginas placeholder del proyecto)
- [x] 1.4 Actualizar `src/routes/index.tsx` — agregar ruta `/profile` como hija de `AuthenticatedLayout`, lazy-loaded, sin RoleProtectedRoute

## Phase 2: WorkspaceSubmenu molecule

- [x] 2.1 Crear `src/components/molecules/WorkspaceSubmenu/index.tsx` con:
  - Props: `workspaces: AvailableWorkspace[]`, `currentActivityType: ActivityType`, `defaultOpen?: boolean`, `onSwitch: (activityType: ActivityType) => void`, `isSwitching?: boolean`
  - Estado local `expanded` iniciando en `defaultOpen ?? true`
  - Header "Seleccionar tipo de perfil" con `ChevronDown` (rotate-180 cuando expandido)
  - Lista de filas: una por workspace, con `RadioButton` (checked si activo, disabled si activo o isSwitching), fondo oscuro si activo, click dispara `onSwitch(w.activityType)` si inactivo
  - Labels via `ACTIVITY_LABELS` importado de `src/config/activityLabels.ts`

## Phase 3: AuthenticatedLayout refactor

- [x] 3.1 Refactorizar `handleSwitchWorkspace` para recibir `activityType: ActivityType` como argumento (eliminar captura de `alternativeActivityType`)
- [x] 3.2 Eliminar `alternativeWorkspace`, `alternativeActivityType` y `canSwitch` (reemplazados por lógica inline en menuItems)
- [x] 3.3 Reemplazar `menuItems` por la nueva estructura:
  - "Ver perfil" (command: navigate a PROFILE)
  - Item con `template: () => <WorkspaceSubmenu .../>` y `style: { padding: 0 }` (solo si WORKSPACE_ADMIN + workspaces disponibles)
  - separator
  - "Cerrar sesión"
- [x] 3.4 Eliminar import de `ACTIVITY_LABELS` inline y el `switchItem` legacy
- [x] 3.5 Importar `WorkspaceSubmenu` y `ACTIVITY_LABELS` desde sus nuevos módulos
