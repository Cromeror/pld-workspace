# Flujo post-login (shell autenticada)

> Documenta qué ve cada rol inmediatamente después de autenticarse y qué elementos de la shell están disponibles durante toda la sesión. Aplica a todos los roles — es transversal a los flujos de registro, auxiliares y clientes.

## Referencias de código

| Concepto | Archivo |
|---|---|
| Layout autenticado | [pld-web/src/layouts/AuthenticatedLayout.tsx](../../pld-web/src/layouts/AuthenticatedLayout.tsx) |
| Redirect post-login por rol | [pld-web/src/config/postLoginRedirect.ts](../../pld-web/src/config/postLoginRedirect.ts) |
| Guards de ruta | [pld-web/src/routes/index.tsx](../../pld-web/src/routes/index.tsx) |
| Hook usuario actual | [pld-web/src/queries/userQueries.ts](../../pld-web/src/queries/userQueries.ts) |
| Switch workspace | [pld-web/src/queries/authQueries.ts](../../pld-web/src/queries/authQueries.ts) |

---

## Redirect post-login por rol

| Rol | Ruta destino | Notas |
|---|---|---|
| `SUPERADMIN` | `/admin/reporting-entity/register` | Wizard de registro de sujetos obligados |
| `WORKSPACE_ADMIN` | `/register` | Dashboard / inicio del sujeto obligado |
| `AUXILIARY` | _(pendiente definir)_ | |

---

## Shell autenticada

La shell es el layout que envuelve todas las páginas post-login. Consiste en:

- **Sidebar izquierdo** (solo desktop): logo + ícono de home
- **Header**: logo (solo mobile) + botón de menú de usuario (iniciales del usuario)

### Menú de usuario

El botón circular con las iniciales del usuario abre un dropdown con:

| Ítem | Condición | Acción |
|---|---|---|
| Email del usuario | Siempre | Disabled (solo informativo) |
| `Cambiar a Notario` / `Cambiar a Inmobiliaria` | Solo `WORKSPACE_ADMIN` con dos registrations `COMPLETED` | Llama `POST /auth/switch-workspace` con el `activityType` alternativo |
| `Cerrar sesión` | Siempre | Invalida token y redirige al login |

> Las iniciales se construyen con la primera letra de `firstName` + primera letra de `paternalSurname` del usuario autenticado (`GET /auth/me`).

### Switch de workspace

- Solo visible para `WORKSPACE_ADMIN`
- La condición actual (`canSwitch`) verifica que el JWT tenga `workspace.activityType` — **pendiente**: debe verificar además que exista una segunda registration `COMPLETED` antes de mostrarse (para evitar el 403 que ocurre cuando el usuario solo tiene una actividad registrada)
- Al hacer switch exitoso, redirige al post-login redirect del rol

---

## Diseños UI

_(Pendiente — agregar capturas del menú de usuario en `disenos/` una vez aprobado el diseño)_

---

## Inconsistencias conocidas

- `canSwitch` en `AuthenticatedLayout.tsx` no verifica si el usuario tiene una segunda registration `COMPLETED` — puede mostrar la opción de switch y recibir 403 si solo tiene una actividad registrada.
