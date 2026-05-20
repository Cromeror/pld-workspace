Flujo: Inicio de Sesión y Recuperación de Contraseña

## Resumen

Flujo de autenticación para usuarios registrados. El usuario ingresa su identificador (email o nickName) y contraseña, acepta el aviso de privacidad en el paso 1 y el sistema valida. Si la cuenta tiene múltiples workspaces (Notaría / Inmobiliaria), se muestra una pantalla de selección antes de emitir el JWT definitivo. Incluye subflujo de recuperación de contraseña. Una vez emitido el JWT, el control pasa al flujo de [post-login](post-login.md).

## Actores

- **WORKSPACE_ADMIN** — ingresa credenciales y selecciona workspace cuando tiene más de uno activo.
- **AUXILIARY** — ingresa credenciales y selecciona workspace cuando pertenece a más de uno.
- **SUPERADMIN** — ingresa credenciales y accede directamente al sistema sin selección de workspace (no pertenece a ninguno).
- **Sistema** — valida credenciales, verifica estado de cuenta, evalúa workspaces disponibles y emite JWT.

## Precondiciones

- El usuario debe estar registrado y tener una cuenta activa en el sistema.
- Para el subflujo de recuperación: el correo electrónico o nickName debe estar registrado.

## Pasos

<!-- jarvis:diagram src=inicio-sesion.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Inicio de Sesión
direction: LR
nodes[29]{id,label,shape}:
  ingresa-correo-electronico-o-nickname,Ingresa correo electrónico o nickName,data
  ingresa-la-contrasena,Ingresa la contraseña,data
  los-datos-son-correctos?,¿Los datos son correctos?,decision
  volver-a-ingresar-los-datos-verifica-tus-datos,Volver a ingresar los datos 'Verifica tus datos',process
  tiene-multiples-workspaces?,¿Tiene múltiples workspaces?,decision
  selecciona-el-workspace-notaria-inmobiliaria,Selecciona el workspace Notaría / Inmobiliaria,data
  emite-jwt-con-rol-seleccionado,Emite JWT con rol seleccionado,data
  ingresa-al-sistema,Ingresa al sistema,process
  fin,Fin,terminator
  recuperar-contrasena,Recuperar contraseña,process
  ingresa-el-correo-electronico-nickname-para-recuperar-contrasena,Ingresa el correo electrónico/nickName para recuperar contraseña,data
  el-correo-esta-registrado?,¿El correo está registrado?,decision
  mensaje-generico-de-revisar-tu-correo,mensaje generico de revisar tu correo,process
  el-correo-es-correcto-y-esta-registrado,El correo es correcto y está registrado,process
  envio-de-enlace-de-recuperacion-al-correo-electronico,Envío de enlace de recuperación al correo electrónico,process
  abrir-enlace-para-cambiar-contrasena,Abrir enlace para cambiar contraseña,process
  ingresa-la-nueva-contrasena,Ingresa la nueva contraseña,data
  confirma-la-nueva-contrasena,Confirma la nueva contraseña,data
  los-datos-coinciden?,¿Los datos coinciden?,decision
  volver-a-ingresar-los-datos-verifica-tus-datos-2,Volver a ingresar los datos 'Verifica tus datos',process
  datos-almacenados,Datos Almacenados,data
  inicio-de-sesion,Inicio de sesion,offpage
  inicio-de-sesion-2,Inicio de sesion,offpage
  inicio,inicio,terminator
  en-inicio-de-sesion-selecciona-check-aviso-de-privacidad,en inicio de sesion selecciona (check) aviso de privacidad,data
  la-cuenta-esta-activa?,¿La cuenta esta activa?,decision
  el-sistema-selecciona-por-defecto-el-unico-workspace-disponible,El sistema selecciona por defecto el único workspace disponible,process
  es-superadmin?,¿Es superadmin?,decision
  enlace-valido,Enlace valido,decision
edges[35]{from,to,label}:
  ingresa-correo-electronico-o-nickname "Ingresa correo electrónico o nickName",ingresa-la-contrasena "Ingresa la contraseña",
  ingresa-la-contrasena "Ingresa la contraseña",en-inicio-de-sesion-selecciona-check-aviso-de-privacidad "en inicio de sesion selecciona (check) aviso de privacidad",
  los-datos-son-correctos? "¿Los datos son correctos?",volver-a-ingresar-los-datos-verifica-tus-datos "Volver a ingresar los datos 'Verifica tus datos'",no
  volver-a-ingresar-los-datos-verifica-tus-datos "Volver a ingresar los datos 'Verifica tus datos'",inicio-de-sesion "Inicio de sesion",
  selecciona-el-workspace-notaria-inmobiliaria "Selecciona el workspace Notaría / Inmobiliaria",emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",
  recuperar-contrasena "Recuperar contraseña",ingresa-el-correo-electronico-nickname-para-recuperar-contrasena "Ingresa el correo electrónico/nickName para recuperar contraseña",
  ingresa-la-nueva-contrasena "Ingresa la nueva contraseña",confirma-la-nueva-contrasena "Confirma la nueva contraseña",
  confirma-la-nueva-contrasena "Confirma la nueva contraseña",los-datos-coinciden? "¿Los datos coinciden?",
  los-datos-coinciden? "¿Los datos coinciden?",volver-a-ingresar-los-datos-verifica-tus-datos-2 "Volver a ingresar los datos 'Verifica tus datos'",no
  los-datos-coinciden? "¿Los datos coinciden?",datos-almacenados "Datos Almacenados",si
  volver-a-ingresar-los-datos-verifica-tus-datos-2 "Volver a ingresar los datos 'Verifica tus datos'",ingresa-la-nueva-contrasena "Ingresa la nueva contraseña",
  datos-almacenados "Datos Almacenados",inicio-de-sesion-2 "Inicio de sesion",Redirecciona
  tiene-multiples-workspaces? "¿Tiene múltiples workspaces?",el-sistema-selecciona-por-defecto-el-unico-workspace-disponible "El sistema selecciona por defecto el único workspace disponible",No
  tiene-multiples-workspaces? "¿Tiene múltiples workspaces?",selecciona-el-workspace-notaria-inmobiliaria "Selecciona el workspace Notaría / Inmobiliaria",Sí
  emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",ingresa-al-sistema "Ingresa al sistema",
  ingresa-al-sistema "Ingresa al sistema",fin "Fin",
  ingresa-el-correo-electronico-nickname-para-recuperar-contrasena "Ingresa el correo electrónico/nickName para recuperar contraseña",el-correo-esta-registrado? "¿El correo está registrado?",
  el-correo-esta-registrado? "¿El correo está registrado?",mensaje-generico-de-revisar-tu-correo "mensaje generico de revisar tu correo",No
  el-correo-esta-registrado? "¿El correo está registrado?",el-correo-es-correcto-y-esta-registrado "El correo es correcto y está registrado",Sí
  el-correo-es-correcto-y-esta-registrado "El correo es correcto y está registrado",envio-de-enlace-de-recuperacion-al-correo-electronico "Envío de enlace de recuperación al correo electrónico",
  envio-de-enlace-de-recuperacion-al-correo-electronico "Envío de enlace de recuperación al correo electrónico",abrir-enlace-para-cambiar-contrasena "Abrir enlace para cambiar contraseña",
  abrir-enlace-para-cambiar-contrasena "Abrir enlace para cambiar contraseña",enlace-valido "Enlace valido",
  inicio-de-sesion "Inicio de sesion",ingresa-correo-electronico-o-nickname "Ingresa correo electrónico o nickName",
  inicio-de-sesion "Inicio de sesion",recuperar-contrasena "Recuperar contraseña",
  inicio "inicio",inicio-de-sesion "Inicio de sesion",
  en-inicio-de-sesion-selecciona-check-aviso-de-privacidad "en inicio de sesion selecciona (check) aviso de privacidad",los-datos-son-correctos? "¿Los datos son correctos?",
  la-cuenta-esta-activa? "¿La cuenta esta activa?",volver-a-ingresar-los-datos-verifica-tus-datos "Volver a ingresar los datos 'Verifica tus datos'",no
  la-cuenta-esta-activa? "¿La cuenta esta activa?",es-superadmin? "¿Es superadmin?",si
  los-datos-son-correctos? "¿Los datos son correctos?",la-cuenta-esta-activa? "¿La cuenta esta activa?",si
  mensaje-generico-de-revisar-tu-correo "mensaje generico de revisar tu correo",inicio-de-sesion "Inicio de sesion",
  el-sistema-selecciona-por-defecto-el-unico-workspace-disponible "El sistema selecciona por defecto el único workspace disponible",emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",
  es-superadmin? "¿Es superadmin?",tiene-multiples-workspaces? "¿Tiene múltiples workspaces?",no
  es-superadmin? "¿Es superadmin?",emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",si
  enlace-valido "Enlace valido",ingresa-la-nueva-contrasena "Ingresa la nueva contraseña",si
  enlace-valido "Enlace valido",inicio-de-sesion "Inicio de sesion",no
```

## Casos alternos

- **Cuenta inactiva**: credenciales correctas pero cuenta desactivada → el sistema muestra "Verifica tus datos" y regresa a Inicio de Sesión (mismo mensaje que credenciales incorrectas).
- **Recuperación de contraseña**: accesible desde la pantalla de Inicio de Sesión; el subflujo completo (nodos, edges, pasos) está documentado en este mismo archivo.
- **Workspace único**: si el usuario solo tiene un workspace registrado, el sistema lo selecciona por defecto sin mostrar pantalla de selección.
- **Enlace de recuperación inválido**: si el usuario abre el enlace y el token no es válido (expirado o ya consumido), el sistema redirige a Inicio de Sesión sin mostrar el formulario de nueva contraseña.

## Reglas de negocio

- El aviso de privacidad debe aceptarse (check) en el paso 1 de inicio de sesión antes de que el sistema procese las credenciales. El check se resetea en cada intento fallido — el usuario debe volver a marcarlo para reintentar. No aplica al subflujo de recuperación de contraseña.
- El JWT definitivo se emite **solo después** de determinar el workspace activo — ya sea por selección del usuario o por defecto cuando es único.
- Un usuario `WORKSPACE_ADMIN` o `AUXILIARY` puede pertenecer a máximo dos workspaces, uno por cada `activityType`: `NOTARY` y `REAL_ESTATE`. `SUPERADMIN` no pertenece a workspaces — accede directamente al sistema.
- La pantalla de selección de workspace aplica por igual a `WORKSPACE_ADMIN` y a `AUXILIARY` cuando tienen más de un workspace activo. Solo `SUPERADMIN` la omite siempre.
- La validación de credenciales incorrectas y cuenta inactiva devuelve el mismo mensaje al usuario ("Verifica tus datos") para no revelar el estado de la cuenta.

## Notas

### Notas técnicas

- **JWT payload**: incluye `{ sub, email, role, workspaceId? }` — `role` puede ser `WORKSPACE_ADMIN`, `AUXILIARY` o `SUPERADMIN`; `workspace.activityType` indica el tipo de actividad del workspace activo (`NOTARY` o `REAL_ESTATE`); `workspaceId` es el `registration_workspace.id` del workspace activo. SUPERADMIN no lleva `workspaceId` ni `workspace`.
- **Selección de workspace**: `POST /auth/login` detecta cuántos workspaces `COMPLETED` tiene el usuario. Si tiene dos (uno NOTARY, uno REAL_ESTATE), emite un temp token (`exp: 2min`, `tempSession: true`) y responde `{ tempToken, requiresProfileSelection: true, profiles: [...] }`. El FE guarda el `tempToken` en `sessionStorage` y redirige a `/auth/select-profile`. `POST /auth/select-profile` recibe `{ tempToken, profile }` en el body (el guard lo extrae del body, no del header), valida, resuelve el `workspaceId` correspondiente y emite el JWT definitivo. (Nota: las rutas y el endpoint conservan el nombre `select-profile` por compatibilidad; conceptualmente se selecciona un workspace.)
- **Temp token**: expira en 2 minutos. Si el usuario no completa la selección en ese tiempo, el BE devuelve 401 y el FE redirige a login con mensaje de sesión expirada. Se almacena en `sessionStorage` (se limpia al cerrar la pestaña).
- **`passwordChangedAt`**: `JwtStrategy` invalida tokens emitidos antes del último cambio de contraseña (`iat * 1000 < passwordChangedAt`).
- **Campo de login**: los únicos identificadores válidos son **email** y **nickName**. El teléfono no es un campo de login. El diseño UI actual muestra "Correo Electrónico o Teléfono" — es una inconsistencia; debe corregirse a "Correo Electrónico o Nickname". Aplica igual al subflujo de recuperación de contraseña.
- **Aviso de privacidad**: el check de aviso de privacidad aplica únicamente en el paso 1 (inicio de sesión). El subflujo de recuperación de contraseña no lo requiere.
- **Post-login**: una vez emitido el JWT definitivo, el control pasa al flujo de [post-login](post-login.md), que decide la ruta destino según `role` y monta la shell autenticada. El FE llama `GET /auth/me` tras guardar el token para obtener los datos del usuario; si falla, muestra error y hace logout.
- **Post-cambio de contraseña**: al llegar a login con `?reason=password-changed` (redireccionado desde el subflujo de recuperación), el FE muestra un toast informativo: "Tu contraseña fue actualizada. Inicia sesión con la nueva contraseña."

## Referencias

| Componente | Archivo |
|---|---|
| Página FE | `pld-web/src/pages/auth/LoginPage.tsx` |
| Formulario FE | `pld-web/src/components/organisms/auth/LoginForm/` |
| Servicio FE | `pld-web/src/services/authService.ts` |
| Queries FE | `pld-web/src/queries/authQueries.ts` |
| Controller BE | `pld-api/apps/auth-users/src/auth/auth.controller.ts` |
| Adapter BE | `pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts` |
| JWT types | `pld-api/packages/domain-auth-users/src/jwt/types.ts` |
| Diseños UI | [disenos/inicio-sesion/](disenos/inicio-sesion/) |

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon paso a paso del flujo. refs usa prefijos BE:/FE:/UI:/FLUJO: para apuntar a donde se resuelve cada paso en el sistema." -->

```toon
steps[4]{step_ui,label_ui,variante,nodos_diagrama,disenos,nota,refs}:
  1,Inicio de sesión,-,"inicio+inicio-de-sesion+ingresa-correo-electronico-o-nickname+ingresa-la-contrasena+en-inicio-de-sesion-selecciona-check-aviso-de-privacidad+los-datos-son-correctos?+la-cuenta-esta-activa?+volver-a-ingresar-los-datos-verifica-tus-datos",disenos/inicio-sesion/paso-1-login/image.jpg,diseño UI dice 'Correo Electrónico o Teléfono' — identificador válido es solo email o nickName (no teléfono); pendiente corregir en el diseño,"BE:pld-api/apps/auth-users/src/auth/auth.controller.ts+FE:pld-web/src/components/organisms/auth/LoginForm/+FE:pld-web/src/services/authService.ts"
  2,Selección de workspace,-,"es-superadmin?+tiene-multiples-workspaces?+selecciona-el-workspace-notaria-inmobiliaria+el-sistema-selecciona-por-defecto-el-unico-workspace-disponible+emite-jwt-con-rol-seleccionado+ingresa-al-sistema+fin","disenos/inicio-sesion/paso-2-seleccion-perfil/seleccionado.jpg+disenos/inicio-sesion/paso-2-seleccion-perfil/sin-seleccion.jpg",Aplica a WORKSPACE_ADMIN y AUXILIARY cuando tienen NOTARY + REAL_ESTATE registrados; SUPERADMIN siempre omite esta pantalla. La ruta del endpoint conserva el nombre 'select-profile' por compatibilidad.,"BE:pld-api/apps/auth-users/src/auth/auth.controller.ts(POST /auth/select-profile)+BE:pld-api/packages/domain-auth-users/src/adapters/auth.adapter.ts+BE:pld-api/packages/domain-auth-users/src/jwt/types.ts+FE:pld-web/src/queries/authQueries.ts+FLUJO:post-login.md"
  3,Recuperar contraseña — ingresar correo,-,"recuperar-contrasena+ingresa-el-correo-electronico-nickname-para-recuperar-contrasena+el-correo-esta-registrado?+mensaje-generico-de-revisar-tu-correo+el-correo-es-correcto-y-esta-registrado","disenos/inicio-sesion/paso-3-solicitar-correo/1.png+disenos/inicio-sesion/paso-3-correo-enviado/image.png",,"BE:pld-api/apps/auth-users/src/auth/auth.controller.ts+FE:pld-web/src/services/authService.ts"
  4,Recuperar contraseña — nueva contraseña,-,"envio-de-enlace-de-recuperacion-al-correo-electronico+abrir-enlace-para-cambiar-contrasena+enlace-valido+ingresa-la-nueva-contrasena+confirma-la-nueva-contrasena+los-datos-coinciden?+volver-a-ingresar-los-datos-verifica-tus-datos-2+datos-almacenados+inicio-de-sesion-2","disenos/inicio-sesion/paso-4-nueva-contrasena/image.png+disenos/inicio-sesion/pagina-resultado/image.png",enlace inválido o expirado redirige a inicio-de-sesion (rama no de enlace-valido),"BE:pld-api/apps/auth-users/src/auth/auth.controller.ts+FE:pld-web/src/services/authService.ts"
```
