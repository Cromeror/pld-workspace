# Design — implementar-menu-switch-workspace-disenio

## Decisiones

### D1 — Renderizado del submenú con MenuItem.template
**Elegido**: Mantener `<Menu popup model={menuItems}>` y usar `template: () => <WorkspaceSubmenu .../>` con `style: { padding: 0 }` en el MenuItem del submenú.  
**Rechazado**: dropdown custom (reimplementar posicionamiento/click-outside/accesibilidad), submenú nativo PrimeReact (no soporta RadioButtons interactivos).  
**Razón**: `MenuItem.template` permite React nodes arbitrarios sin perder la infraestructura del popup.

### D2 — WorkspaceSubmenu como molecule nuevo
**Elegido**: Crear `src/components/molecules/WorkspaceSubmenu/index.tsx`.  
**Rechazado**: Reusar `PersonTypeAccordion` (props/layout incompatibles); extraer Accordion genérico (sobre-ingeniería para 2 casos).  
**Razón**: Replicar el patrón chevron (≈10 líneas) es más limpio que acoplar dos componentes con responsabilidades distintas. Se reutiliza el icono `ChevronDown` y la clase `rotate-180`.

### D3 — Soporte N workspaces
**Elegido**: `WorkspaceSubmenu` recibe `workspaces: AvailableWorkspace[]` completo e itera.  
**Rechazado**: Mantener `find` de un solo alternativo.  
**Razón**: REQ-03 exige N workspaces. El workspace activo se deriva por `w.activityType === currentActivityType`.

### D4 — Estados visuales por fila
**Elegido**: Fila activa = `RadioButton checked disabled` + fondo oscuro; fila inactiva = `RadioButton` vacío + hover + click dispara switch.  
**Rechazado**: RadioButton group formal (complica disabled selectivo).  
**Razón**: Reproduce el comportamiento "no puedo cambiar al que ya estoy".

### D5 — Modelo de menuItems por rol
**Elegido**:
```
[Ver perfil]                    ← siempre (command: navigate /profile)
[WorkspaceSubmenu template]     ← solo WORKSPACE_ADMIN con workspaces
{separator}
[Cerrar sesión]
```
Email eliminado. Gating: `role === WORKSPACE_ADMIN && !isLoadingWorkspaces && workspaces.length > 0`.

### D6 — Ruta /profile placeholder
**Elegido**: `PROFILE: "/profile"` en `RoutesUrl`, hija de `AuthenticatedLayout` sin `RoleProtectedRoute`, `ProfilePage` lazy-loaded.  
**Razón**: Consistencia con el router existente y `withSuspense`.

### D7 — ACTIVITY_LABELS en módulo compartido
**Elegido**: Mover a `src/config/activityLabels.ts`, importar desde el submenú.  
**Rechazado**: Duplicar en submenú; pasar por props (constante estática).  
**Razón**: Single source of truth.

## Archivos afectados

| Archivo | Acción | Detalle |
|---|---|---|
| `src/components/molecules/WorkspaceSubmenu/index.tsx` | Crear | Acordeón + filas con RadioButton |
| `src/config/activityLabels.ts` | Crear | `ACTIVITY_LABELS` exportado |
| `src/layouts/AuthenticatedLayout.tsx` | Modificar | Nuevos menuItems, WorkspaceSubmenu template |
| `src/routes/routes-urls.ts` | Modificar | Agregar `PROFILE` |
| `src/routes/index.tsx` | Modificar | Ruta /profile lazy |
| `src/pages/ProfilePage.tsx` | Crear | Placeholder vacío |

## Notas de implementación

1. `template` item: `{ template: () => <WorkspaceSubmenu .../>, style: { padding: 0 } }`
2. `handleSwitchWorkspace` refactor: pasa a recibir `activityType: ActivityType` como argumento
3. Workspace activo: `w.activityType === currentActivityType` (del JWT, no workspaceId)
4. `WorkspaceSubmenu` arranca con `expanded = true` (REQ-02)
5. Filas inactivas: `<button type="button">`; fila activa: `<button disabled>`
6. Atom `RadioButton` se usa tal cual (checked, disabled)
