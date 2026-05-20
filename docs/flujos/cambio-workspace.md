Flujo: Cambio de workspace (switch en sesión)

## Resumen

Acción del usuario `WORKSPACE_ADMIN` ya autenticado que cambia su workspace activo (NOTARY ↔ REAL_ESTATE) desde el menú de usuario en la barra superior, sin re-loguearse. Aplica solo cuando la cuenta tiene más de un workspace `COMPLETED`. El sistema re-emite el JWT con el `activityType` y `workspaceId` alternativos, limpia el cache local y redirige al post-login redirect del rol. Diseños en [disenos/cambio-workspace/](disenos/cambio-workspace/).

## Actores

- **WORKSPACE_ADMIN** — único rol habilitado para hacer switch. Abre el menú, selecciona "Cambiar a Notario" o "Cambiar a Inmobiliaria".
- **Sistema** — valida que el usuario tenga una registration `COMPLETED` para el `activityType` alternativo y re-emite el JWT.

## Precondiciones

- El usuario completó el flujo de [inicio de sesión](inicio-sesion.md) y se encuentra dentro de la shell autenticada del flujo de [post-login](post-login.md).
- El JWT vigente tiene `role === WORKSPACE_ADMIN` y `workspace.activityType` (NOTARY o REAL_ESTATE).
- El usuario tiene al menos una segunda registration `COMPLETED` para el `activityType` alternativo. (Esta precondición **no se verifica en el FE antes de mostrar la opción** — ver Inconsistencias.)

## Pasos

<!-- jarvis:diagram src=cambio-workspace.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Cambio de workspace
direction: LR
nodes[10]{id,label,shape}:
  inicio,inicio,terminator
  usuario-abre-menu-de-iniciales,Usuario abre menú de iniciales,process
  click-en-cambiar-a-notario-inmobiliaria,Click en: Cambiar a Notario / Inmobiliaria,process
  post-auth-switch-workspace-activitytype-alternativo,POST /auth/switch-workspace { activityType alternativo },data
  existe-registration-completed-para-activitytype-alternativo?,¿Existe registration COMPLETED para activityType alternativo?,decision
  re-emite-jwt-con-nuevo-workspaceid-y-activitytype,Re-emite JWT con nuevo workspaceId y activityType,data
  guarda-token-nuevo-y-limpia-cache-de-queries,Guarda token nuevo y limpia cache de queries,process
  redirige-al-post-login-redirect-del-rol,Redirige al post-login redirect del rol,process
  responde-403-perfil-no-disponible,Responde 403 Perfil no disponible,process
  fin,fin,terminator
edges[10]{from,to,label}:
  inicio "inicio",usuario-abre-menu-de-iniciales "Usuario abre menú de iniciales",
  usuario-abre-menu-de-iniciales "Usuario abre menú de iniciales",click-en-cambiar-a-notario-inmobiliaria "Click en: Cambiar a Notario / Inmobiliaria",
  click-en-cambiar-a-notario-inmobiliaria "Click en: Cambiar a Notario / Inmobiliaria",post-auth-switch-workspace-activitytype-alternativo "POST /auth/switch-workspace { activityType alternativo }",
  post-auth-switch-workspace-activitytype-alternativo "POST /auth/switch-workspace { activityType alternativo }",existe-registration-completed-para-activitytype-alternativo? "¿Existe registration COMPLETED para activityType alternativo?",
  existe-registration-completed-para-activitytype-alternativo? "¿Existe registration COMPLETED para activityType alternativo?",re-emite-jwt-con-nuevo-workspaceid-y-activitytype "Re-emite JWT con nuevo workspaceId y activityType",si
  existe-registration-completed-para-activitytype-alternativo? "¿Existe registration COMPLETED para activityType alternativo?",responde-403-perfil-no-disponible "Responde 403 Perfil no disponible",no
  re-emite-jwt-con-nuevo-workspaceid-y-activitytype "Re-emite JWT con nuevo workspaceId y activityType",guarda-token-nuevo-y-limpia-cache-de-queries "Guarda token nuevo y limpia cache de queries",
  guarda-token-nuevo-y-limpia-cache-de-queries "Guarda token nuevo y limpia cache de queries",redirige-al-post-login-redirect-del-rol "Redirige al post-login redirect del rol",
  redirige-al-post-login-redirect-del-rol "Redirige al post-login redirect del rol",fin "fin",
  responde-403-perfil-no-disponible "Responde 403 Perfil no disponible",fin "fin",
```

## Casos alternos

- **Usuario sin segunda registration COMPLETED**: el BE responde 403 con "Perfil no disponible". El FE no muestra feedback al usuario — el menú-item simplemente vuelve a estar habilitado sin explicación (ver Inconsistencias).
- **SUPERADMIN intenta llamar el endpoint**: el BE responde 403 "Operación no permitida para este rol". Escenario teórico — el menú nunca muestra la opción de switch para SUPERADMIN, así que solo aplica si alguien llama el endpoint directo.
- **Token expirado durante el switch**: el `JwtAuthGuard` rechaza con 401; el FE redirige al login estándar.

## Reglas de negocio

- **Solo `WORKSPACE_ADMIN` puede hacer switch.** El menú no muestra la opción para `SUPERADMIN` ni `AUXILIARY`, y el BE rechaza con 403 si llega la llamada.
- **Activity types soportados**: únicamente `NOTARY` y `REAL_ESTATE`. El switch es siempre un toggle entre ambos.
- **Persistencia de sesión**: el switch reemplaza el JWT vigente; no requiere re-login.
- **Cache local**: al cambiar de workspace, se invalida todo el cache de React Query para evitar mostrar datos del workspace anterior.
- **Destino post-switch**: el usuario va al mismo post-login redirect del rol (`/register` para `WORKSPACE_ADMIN`), no a una pantalla intermedia de confirmación.

## Notas

### Notas técnicas

- **Endpoint**: `POST /auth/switch-workspace`, protegido por `JwtAuthGuard`. Recibe `SwitchWorkspaceDto { activityType: 'NOTARY' | 'REAL_ESTATE' }`. Retorna `{ accessToken, tokenType: 'Bearer' }`.
- **Validaciones BE** (`auth.adapter.ts:146-176`):
  - Si `role === SUPERADMIN` → `ForbiddenError("Operación no permitida para este rol")` (403).
  - Busca `registration.status = COMPLETED AND activity_type = <alternativo>`. Si no existe → `ForbiddenError("Perfil no disponible")` (403).
  - Si pasa, re-emite el JWT con `workspace.id` y `workspace.activityType` del workspace alternativo. `role` se conserva.
- **FE — cálculo de `canSwitch`** (`AuthenticatedLayout.tsx:33`): solo verifica `role === WORKSPACE_ADMIN && !!currentActivityType`. **No consulta al BE** si existe la segunda registration, por lo que puede ofrecer la opción y recibir 403.
- **FE — toggle de `activityType` alternativo** (`AuthenticatedLayout.tsx:34-38`): hardcodeado, si el actual es `NOTARY` el alternativo es `REAL_ESTATE`, y viceversa.
- **FE — onSuccess de la mutation** (`authQueries.ts:46-50`): `authService.saveToken(data.accessToken)` + `queryClient.clear()` (limpia **todo** el cache, no solo queries workspace-scoped).
- **FE — onError**: no hay manejador explícito; solo se setea `isSwitching = false` para volver a habilitar el menú-item. **No se muestra mensaje al usuario.**
- **Estado loading**: mientras la mutation está en flight, el menú-item se deshabilita (`disabled: isSwitching`). No hay spinner adicional.
- **Post-switch en FE**: `navigate(getPostLoginRedirect(UserRole.WORKSPACE_ADMIN))` → `/register`. El menú se cierra automáticamente al re-renderizar.

## Referencias

| Componente | Archivo |
|---|---|
| Trigger UI (menú) | `pld-web/src/layouts/AuthenticatedLayout.tsx` |
| Mutation FE | `pld-web/src/queries/authQueries.ts` (`useSwitchWorkspaceMutation`) |
| Service FE | `pld-web/src/services/authService.ts` (`switchWorkspace`) |
| Controller BE | `pld-api/apps/auth-users/src/auth/auth.controller.ts` (`POST /auth/switch-workspace`) |
| Adapter BE | `pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts` (`switchWorkspace`) |
| JWT types | `pld-api/packages/domain-auth-users/src/jwt/types.ts` |
| Diseños UI | [disenos/cambio-workspace/](disenos/cambio-workspace/) |
| Flujo previo | [post-login.md](post-login.md) |

## Inconsistencias

- **`canSwitch` no verifica segunda registration `COMPLETED`**: el menú puede ofrecer el switch a un workspace que el usuario no tiene activo. El BE responde 403, pero el usuario no recibe feedback visual del error.
- **Sin manejo visual de error en FE**: si el `POST /auth/switch-workspace` falla (403 u otro), el menú-item vuelve a habilitarse sin mostrar mensaje. El usuario queda sin contexto de por qué no pasó nada.
- **Label hardcodeado NOTARY ↔ REAL_ESTATE**: el FE calcula la alternativa con un toggle local. Si en el futuro se agregan más `activityType`, hay que cambiar la lógica.
- **Path del archivo de layout con espacio inicial**: el archivo real es `pld-web/src/layouts/ AuthenticatedLayout.tsx` (con espacio al inicio del nombre). Bug menor, conviene renombrar.

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon paso a paso del flujo. refs usa prefijos BE:/FE:/UI:/FLUJO: para apuntar a donde se resuelve cada paso en el sistema." -->

```toon
steps[3]{step_ui,label_ui,variante,nodos_diagrama,disenos,nota,refs}:
  1,Menú de usuario — switch disponible,WORKSPACE_ADMIN expandido+colapsado,"usuario-abre-menu-de-iniciales+click-en-cambiar-a-notario-inmobiliaria","disenos/cambio-workspace/menu-usuario-workspace-admin-expandido.jpg+disenos/cambio-workspace/menu-usuario-workspace-admin-colapsado.jpg",El menú-item se deshabilita durante el switch (isSwitching). Variantes SUPERADMIN y AUXILIARY no aplican.,"FE:pld-web/src/layouts/AuthenticatedLayout.tsx"
  2,Ejecución del switch,-,"post-auth-switch-workspace-activitytype-alternativo+existe-registration-completed-para-activitytype-alternativo?+re-emite-jwt-con-nuevo-workspaceid-y-activitytype+guarda-token-nuevo-y-limpia-cache-de-queries+redirige-al-post-login-redirect-del-rol",,Re-emite JWT y limpia cache de React Query. El destino es el post-login redirect del rol.,"FE:pld-web/src/queries/authQueries.ts+FE:pld-web/src/services/authService.ts+BE:pld-api/apps/auth-users/src/auth/auth.controller.ts+BE:pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts+FLUJO:post-login.md"
  3,Error 403 sin feedback,-,"responde-403-perfil-no-disponible",,El usuario no recibe mensaje visual; el menú-item solo vuelve a habilitarse.,"BE:pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts+FE:pld-web/src/layouts/AuthenticatedLayout.tsx"
```
