Flujo: Post-login (shell autenticada)

## Resumen

Experiencia transversal que aplica a todos los roles inmediatamente después de que el [inicio de sesión](inicio-sesion.md) emite el JWT definitivo. Define a qué pantalla se redirige cada rol al entrar al sistema y qué elementos de la shell autenticada (sidebar, header, menú de usuario, switch de workspace) están disponibles durante toda la sesión. Diseños en [disenos/post-login/](disenos/post-login/).

## Actores

- **SUPERADMIN** — ve el wizard de registro de sujetos obligados (notaría / inmobiliaria).
- **WORKSPACE_ADMIN** — ve el dashboard de registro de auxiliares.
- **AUXILIARY** — ruta destino pendiente de definir.
- **Sistema** — lee el JWT, decide la ruta destino por rol y monta la shell autenticada.

## Precondiciones

- El usuario completó el flujo de [inicio de sesión](inicio-sesion.md) exitosamente, incluida la selección de workspace cuando aplica.
- El sistema emitió un JWT definitivo con `role` (y `workspace.activityType` cuando aplica). En este punto el workspace ya está resuelto — no hay ambigüedad.

## Pasos

<!-- jarvis:diagram src=post-login.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Post-login
direction: LR
nodes[11]{id,label,shape}:
  inicio,inicio,terminator
  es-superadmin?,¿es SUPERADMIN?,decision
  muestra-registro-notaria-inmobiliaria-paso-1,muestra registro notaria/inmobiliaria paso 1,process
  es-workspace-admin?,¿es WORKSPACE_ADMIN?,decision
  muestra-registro-auxiliar-paso-1,muestra registro auxiliar paso 1,process
  es-auxiliary?,¿es AUXILIARY?,decision
  pagina-en-blanco,pagina en blanco,process
  muestra-pagina-de-error-el-codigo-dice-rol-desconocido,muestra página de error. El código dice: rol desconocido,process
  invalida-la-sesion,Invalida la sesion,process
  fin,fin,terminator
  redirige-a-homepage-silenciosamente,redirige a homepage silenciosamente,process
edges[13]{from,to,label}:
  muestra-registro-notaria-inmobiliaria-paso-1 "muestra registro notaria/inmobiliaria paso 1",fin "fin",
  muestra-registro-auxiliar-paso-1 "muestra registro auxiliar paso 1",fin "fin",
  es-auxiliary? "¿es AUXILIARY?",invalida-la-sesion "Invalida la sesion",no
  pagina-en-blanco "pagina en blanco",fin "fin",
  inicio "inicio",es-superadmin? "¿es SUPERADMIN?",
  es-superadmin? "¿es SUPERADMIN?",muestra-registro-notaria-inmobiliaria-paso-1 "muestra registro notaria/inmobiliaria paso 1",si
  es-superadmin? "¿es SUPERADMIN?",es-workspace-admin? "¿es WORKSPACE_ADMIN?",no
  es-workspace-admin? "¿es WORKSPACE_ADMIN?",muestra-registro-auxiliar-paso-1 "muestra registro auxiliar paso 1",si
  es-workspace-admin? "¿es WORKSPACE_ADMIN?",es-auxiliary? "¿es AUXILIARY?",no
  es-auxiliary? "¿es AUXILIARY?",pagina-en-blanco "pagina en blanco",si
  redirige-a-homepage-silenciosamente "redirige a homepage silenciosamente",fin "fin",
  invalida-la-sesion "Invalida la sesion",muestra-pagina-de-error-el-codigo-dice-rol-desconocido "muestra página de error. El código dice: rol desconocido",
  muestra-pagina-de-error-el-codigo-dice-rol-desconocido "muestra página de error. El código dice: rol desconocido",redirige-a-homepage-silenciosamente "redirige a homepage silenciosamente",
```

## Casos alternos

- **Rol sin ruta reconocida**: el sistema invalida la sesión, muestra una página de error con código "rol desconocido" y redirige a homepage. Escenario hipotético — todos los usuarios tienen rol asignado en BD.

## Reglas de negocio

- **Redirect post-login por rol**:

  | role | Flujo |
  |---|---|
  | `SUPERADMIN` | → muestra registro notaria/inmobiliaria paso 1 |
  | `WORKSPACE_ADMIN` | → muestra registro auxiliar paso 1 |
  | `AUXILIARY` | → _(pendiente)_ |
  | Sin rol reconocido | → invalida la sesión → muestra página de error → redirige a homepage _(hipotético — todos los usuarios tienen rol en BD)_ |

- **Shell autenticada**: todas las páginas post-login se renderizan dentro de un layout común que provee:
  - Sidebar izquierdo (solo desktop): logo + ícono de home.
  - Header: logo mobile + botón circular con iniciales del usuario (`firstName[0] + paternalSurname[0]`).
  - Menú de usuario (dropdown desde el botón de iniciales):

    | Ítem | Condición | Acción |
    |---|---|---|
    | Email del usuario | Siempre visible | Informativo (no clickeable) |
    | `Cambiar a Notario` / `Cambiar a Inmobiliaria` | `WORKSPACE_ADMIN` con `workspace.activityType` en JWT | Cambia el workspace activo, re-emite JWT y redirige al post-login redirect |
    | `Cerrar sesión` | Siempre visible | Invalida token local y redirige a `/login` |

- **Switch de workspace**: solo visible para `WORKSPACE_ADMIN`. Requiere que el JWT tenga `workspace.activityType`. Al hacer switch exitoso, el sistema guarda el nuevo token, limpia el cache de queries y redirige al post-login redirect del rol. Ver el flujo detallado en [cambio-workspace.md](cambio-workspace.md).

## Notas

### Notas técnicas

- **JWT**: el redirect post-login se decide a partir del campo `role` del JWT y, cuando aplica, de `workspace.activityType`. SUPERADMIN no lleva workspace en el JWT (campo `workspace` opcional).
- **Iniciales del usuario**: se construyen desde `GET /auth/me` (`firstName` + `paternalSurname`). Mientras la query resuelve, el botón renderiza con iniciales vacías (no muestra spinner propio). Los guards de ruta sí muestran "Cargando..." antes de montar el layout.
- **Switch workspace**: `POST /auth/switch-workspace` recibe el `activityType` alternativo, valida que el usuario tenga una registration `COMPLETED` para ese workspace y re-emite el JWT con el nuevo `workspaceId` y `activityType`. El FE guarda el nuevo token y limpia **todo** el cache de React Query (`queryClient.clear()`), no solo las queries workspace-scoped.
- **`canSwitch`** (FE): verifica únicamente que `role === WORKSPACE_ADMIN` y que el JWT tenga `workspace.activityType`. No consulta al BE si existe una segunda registration `COMPLETED` — puede mostrar la opción de switch y recibir 403 al ejecutarla (ver Inconsistencias).
- **Guards de ruta**: las rutas autenticadas están protegidas por `ProtectedRoute` (chequea sesión; redirige a `/login` si no autenticado) y `RoleProtectedRoute` (chequea rol; redirige silenciosamente a `/` si el rol no coincide).

## Referencias

| Componente | Archivo |
|---|---|
| Layout autenticado | `pld-web/src/layouts/AuthenticatedLayout.tsx` |
| Redirect post-login | `pld-web/src/config/postLoginRedirect.ts` |
| Guards de ruta | `pld-web/src/routes/index.tsx` |
| Hook usuario actual | `pld-web/src/queries/userQueries.ts` |
| Switch workspace mutation | `pld-web/src/queries/authQueries.ts` |
| Switch workspace service | `pld-web/src/services/authService.ts` |
| Switch workspace BE | `pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts` |
| Diseños UI | [disenos/post-login/](disenos/post-login/) |

## Inconsistencias

- **`canSwitch` no verifica segunda registration `COMPLETED`**: el menú puede ofrecer el switch a un workspace que el usuario no tiene activo, y el BE responde 403.
- **Ruta destino para `AUXILIARY` no definida**: el flujo cae en "página en blanco".
- **Rol sin ruta reconocida (FE vs. diagrama)**: el diagrama define invalidar sesión y mostrar página de error; el código (`postLoginRedirect.ts`) redirige a homepage silenciosamente sin invalidar.
- **Label del switch hardcodeado (NOTARY ↔ REAL_ESTATE)**: el FE calcula la alternativa con un toggle local en lugar de consultarla al BE; si en el futuro hay más de dos `activityType`, hay que cambiar la lógica.
- **Path del archivo de layout con espacio inicial**: el archivo real es `pld-web/src/layouts/ AuthenticatedLayout.tsx` (con espacio al inicio del nombre). Bug menor, conviene renombrar.

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon paso a paso del flujo. refs usa prefijos BE:/FE:/UI:/FLUJO: para apuntar a donde se resuelve cada paso en el sistema." -->

```toon
steps[4]{step_ui,label_ui,variante,nodos_diagrama,disenos,nota,refs}:
  1,Redirect post-login por rol,-,"inicio+es-superadmin?+es-workspace-admin?+es-auxiliary?",,Switch por el campo role del JWT. El workspace ya viene resuelto desde el flujo de inicio-sesion.,"FE:pld-web/src/config/postLoginRedirect.ts+FE:pld-web/src/routes/index.tsx+FLUJO:inicio-sesion.md"
  2,Shell autenticada,-,-,,_(pendiente capturas)_,"FE:pld-web/src/layouts/AuthenticatedLayout.tsx"
  3,Menú de usuario,-,-,,Capturas del menú con switch viven en disenos/cambio-workspace/ (flujo futuro). El menú estático en este flujo aún no tiene capturas propias.,"FE:pld-web/src/layouts/AuthenticatedLayout.tsx"
  4,Rol sin ruta reconocida,-,"es-auxiliary?+invalida-la-sesion+muestra-pagina-de-error-el-codigo-dice-rol-desconocido+redirige-a-homepage-silenciosamente",,Diagrama y código actual divergen — ver Inconsistencias.,"FE:pld-web/src/config/postLoginRedirect.ts"
```
