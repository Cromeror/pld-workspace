Flujo: Inicio de Sesión y Recuperación de Contraseña

## Resumen

Flujo de autenticación para usuarios registrados. El usuario ingresa sus credenciales, acepta el aviso de privacidad y el sistema valida. Si la cuenta tiene múltiples perfiles (Notaría / Inmobiliaria), se muestra una pantalla de selección antes de emitir el JWT. Incluye subflujo de recuperación de contraseña.

## Actores

- **Usuario** (NOTARY / REAL_ESTATE) — ingresa credenciales y selecciona perfil cuando aplica.
- **Superadmin** (SUPERADMIN) — ingresa credenciales y accede directamente al sistema sin selección de perfil.
- **Sistema** — valida credenciales, verifica estado de cuenta, evalúa perfiles disponibles y emite JWT.

## Precondiciones

- El usuario debe estar registrado y tener una cuenta activa en el sistema.
- Para el subflujo de recuperación: el correo electrónico debe estar registrado.

## Pasos

<!-- jarvis:diagram src=flujo.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Inicio de Sesión
direction: LR
nodes[28]{id,label,shape}:
  ingresa-correo-electronico-o-nickname,Ingresa correo electrónico o nickName,data
  ingresa-la-contrasena,Ingresa la contraseña,data
  los-datos-son-correctos?,¿Los datos son correctos?,decision
  volver-a-ingresar-los-datos-verifica-tus-datos,Volver a ingresar los datos 'Verifica tus datos',process
  tiene-multiples-espacios-de-trabajo?,¿Tiene múltiples espacios de trabajo?,decision
  selecciona-el-espacio-de-trabajo-notaria-inmobiliaria,Selecciona el espacio de trabajo Notaría / Inmobiliaria,data
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
  selecciona-check-aviso-de-privacidad,selecciona(check) aviso de privacidad,data
  el-sistema-verifica-la-cuenta-esta-inactiva?,El sistema verifica: ¿La cuenta esta inactiva?,decision
  el-sistema-selecciona-por-defecto-el-unico-espacio-de-trabajo-disponible,El sistema selecciona por defecto el unico espacio de trabajo disponible,process
  es-superadmin?,¿Es superadmin?,decision
edges[33]{from,to,label}:
  ingresa-correo-electronico-o-nickname "Ingresa correo electrónico o nickName",ingresa-la-contrasena "Ingresa la contraseña",
  ingresa-la-contrasena "Ingresa la contraseña",selecciona-check-aviso-de-privacidad "selecciona(check) aviso de privacidad",
  los-datos-son-correctos? "¿Los datos son correctos?",volver-a-ingresar-los-datos-verifica-tus-datos "Volver a ingresar los datos 'Verifica tus datos'",no
  volver-a-ingresar-los-datos-verifica-tus-datos "Volver a ingresar los datos 'Verifica tus datos'",inicio-de-sesion "Inicio de sesion",
  selecciona-el-espacio-de-trabajo-notaria-inmobiliaria "Selecciona el espacio de trabajo Notaría / Inmobiliaria",emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",
  recuperar-contrasena "Recuperar contraseña",ingresa-el-correo-electronico-nickname-para-recuperar-contrasena "Ingresa el correo electrónico/nickName para recuperar contraseña",
  ingresa-la-nueva-contrasena "Ingresa la nueva contraseña",confirma-la-nueva-contrasena "Confirma la nueva contraseña",
  confirma-la-nueva-contrasena "Confirma la nueva contraseña",los-datos-coinciden? "¿Los datos coinciden?",
  los-datos-coinciden? "¿Los datos coinciden?",volver-a-ingresar-los-datos-verifica-tus-datos-2 "Volver a ingresar los datos 'Verifica tus datos'",no
  los-datos-coinciden? "¿Los datos coinciden?",datos-almacenados "Datos Almacenados",si
  volver-a-ingresar-los-datos-verifica-tus-datos-2 "Volver a ingresar los datos 'Verifica tus datos'",ingresa-la-nueva-contrasena "Ingresa la nueva contraseña",
  datos-almacenados "Datos Almacenados",inicio-de-sesion-2 "Inicio de sesion",Redirecciona
  tiene-multiples-espacios-de-trabajo? "¿Tiene múltiples espacios de trabajo?",el-sistema-selecciona-por-defecto-el-unico-espacio-de-trabajo-disponible "El sistema selecciona por defecto el unico espacio de trabajo disponible",No
  tiene-multiples-espacios-de-trabajo? "¿Tiene múltiples espacios de trabajo?",selecciona-el-espacio-de-trabajo-notaria-inmobiliaria "Selecciona el espacio de trabajo Notaría / Inmobiliaria",Sí
  emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",ingresa-al-sistema "Ingresa al sistema",
  ingresa-al-sistema "Ingresa al sistema",fin "Fin",
  ingresa-el-correo-electronico-nickname-para-recuperar-contrasena "Ingresa el correo electrónico/nickName para recuperar contraseña",el-correo-esta-registrado? "¿El correo está registrado?",
  el-correo-esta-registrado? "¿El correo está registrado?",mensaje-generico-de-revisar-tu-correo "mensaje generico de revisar tu correo",No
  el-correo-esta-registrado? "¿El correo está registrado?",el-correo-es-correcto-y-esta-registrado "El correo es correcto y está registrado",Sí
  el-correo-es-correcto-y-esta-registrado "El correo es correcto y está registrado",envio-de-enlace-de-recuperacion-al-correo-electronico "Envío de enlace de recuperación al correo electrónico",
  envio-de-enlace-de-recuperacion-al-correo-electronico "Envío de enlace de recuperación al correo electrónico",abrir-enlace-para-cambiar-contrasena "Abrir enlace para cambiar contraseña",
  abrir-enlace-para-cambiar-contrasena "Abrir enlace para cambiar contraseña",ingresa-la-nueva-contrasena "Ingresa la nueva contraseña",
  inicio-de-sesion "Inicio de sesion",ingresa-correo-electronico-o-nickname "Ingresa correo electrónico o nickName",
  inicio-de-sesion "Inicio de sesion",recuperar-contrasena "Recuperar contraseña",
  inicio "inicio",inicio-de-sesion "Inicio de sesion",
  selecciona-check-aviso-de-privacidad "selecciona(check) aviso de privacidad",los-datos-son-correctos? "¿Los datos son correctos?",
  el-sistema-verifica-la-cuenta-esta-inactiva? "El sistema verifica: ¿La cuenta esta inactiva?",volver-a-ingresar-los-datos-verifica-tus-datos "Volver a ingresar los datos 'Verifica tus datos'",si
  el-sistema-verifica-la-cuenta-esta-inactiva? "El sistema verifica: ¿La cuenta esta inactiva?",es-superadmin? "¿Es superadmin?",no
  los-datos-son-correctos? "¿Los datos son correctos?",el-sistema-verifica-la-cuenta-esta-inactiva? "El sistema verifica: ¿La cuenta esta inactiva?",si
  mensaje-generico-de-revisar-tu-correo "mensaje generico de revisar tu correo",inicio-de-sesion "Inicio de sesion",
  el-sistema-selecciona-por-defecto-el-unico-espacio-de-trabajo-disponible "El sistema selecciona por defecto el unico espacio de trabajo disponible",emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",
  es-superadmin? "¿Es superadmin?",tiene-multiples-espacios-de-trabajo? "¿Tiene múltiples espacios de trabajo?",no
  es-superadmin? "¿Es superadmin?",emite-jwt-con-rol-seleccionado "Emite JWT con rol seleccionado",si
```

## Casos alternos

- **Cuenta inactiva**: credenciales correctas pero cuenta desactivada → el sistema muestra "Verifica tus datos" y regresa a Inicio de Sesión (mismo mensaje que credenciales incorrectas).
- **Recuperación de contraseña**: accesible desde la pantalla de Inicio de Sesión; el subflujo completo (nodos, edges, pasos) está documentado en este mismo archivo.
- **Rol único**: si el usuario solo tiene un perfil registrado, el sistema lo selecciona por defecto sin mostrar pantalla de selección.

## Reglas de negocio

- El aviso de privacidad debe aceptarse (check) antes de que el sistema procese las credenciales. El check se resetea en cada intento fallido — el usuario debe volver a marcarlo para reintentar.
- El JWT se emite **solo después** de determinar el rol — ya sea por selección del usuario o por defecto.
- Un usuario puede tener máximo dos perfiles: `NOTARY` y `REAL_ESTATE`. El `SUPERADMIN` no tiene perfiles de workspace — accede directamente al sistema.
- La validación de credenciales incorrectas y cuenta inactiva devuelve el mismo mensaje al usuario ("Verifica tus datos") para no revelar el estado de la cuenta.

## Notas

- El diagrama BE (toon) es la fuente de verdad; el `.drawio` es una sombra generada por Jarvis.
- Los diseños UI de la pantalla de selección de perfil están pendientes — es una pantalla nueva sin diseño aprobado.

### Notas técnicas

- **JWT payload**: incluye `{ sub, email, role, workspaceId? }` — `role` refleja el perfil seleccionado (`NOTARY` o `REAL_ESTATE`); `workspaceId` es el `registration_workspace.id` del workspace activo. SUPERADMIN no lleva `workspaceId`.
- **Selección de perfil**: `POST /auth/login` detecta cuántos registros `COMPLETED` tiene el usuario. Si tiene dos (uno NOTARY, uno REAL_ESTATE), emite un temp token (`exp: 2min`, `tempSession: true`) y responde `{ tempToken, requiresProfileSelection: true, profiles: [...] }`. El FE guarda el `tempToken` en `sessionStorage` y redirige a `/auth/select-profile`. `POST /auth/select-profile` recibe `{ tempToken, profile }`, valida, resuelve el `workspaceId` correspondiente y emite el JWT definitivo.
- **Temp token**: expira en 2 minutos. Si el usuario no completa la selección en ese tiempo, el BE devuelve 401 y el FE redirige a login con mensaje de sesión expirada. Se almacena en `sessionStorage` (se limpia al cerrar la pestaña).
- **`passwordChangedAt`**: `JwtStrategy` invalida tokens emitidos antes del último cambio de contraseña (`iat * 1000 < passwordChangedAt`).

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
| Diseños UI | [disenos/](disenos/) |

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon que agrupa nodos del diagrama BE por pantalla UI. nodos_diagrama lista todos los nodos que esa pantalla implementa. disenos lista screenshots disponibles. nota registra inconsistencias o vacíos pendientes." -->

```toon
steps[4]{step_ui,label_ui,variante,nodos_diagrama,disenos,nota}:
  1,Inicio de sesión,-,"inicio+inicio-de-sesion+ingresa-correo-electronico-o-nickname+ingresa-la-contrasena+selecciona-check-aviso-de-privacidad+los-datos-son-correctos?+el-sistema-verifica-la-cuenta-esta-inactiva?+volver-a-ingresar-los-datos-verifica-tus-datos",,VACÍO: sin diseño UI
  2,Selección de perfil,-,"es-superadmin?+tiene-multiples-espacios-de-trabajo?+selecciona-el-espacio-de-trabajo-notaria-inmobiliaria+el-sistema-selecciona-por-defecto-el-unico-espacio-de-trabajo-disponible+emite-jwt-con-rol-seleccionado+ingresa-al-sistema+fin",,VACÍO: pantalla nueva — aplica solo cuando el usuario tiene NOTARY + REAL_ESTATE registrados; SUPERADMIN omite esta pantalla
  3,Recuperar contraseña — ingresar correo,-,"recuperar-contrasena+ingresa-el-correo-electronico-nickname-para-recuperar-contrasena+el-correo-esta-registrado?+mensaje-generico-de-revisar-tu-correo+el-correo-es-correcto-y-esta-registrado","disenos/paso-3-solicitar-correo/1.png+disenos/paso-3-correo-enviado/image.png",
  4,Recuperar contraseña — nueva contraseña,-,"envio-de-enlace-de-recuperacion-al-correo-electronico+abrir-enlace-para-cambiar-contrasena+ingresa-la-nueva-contrasena+confirma-la-nueva-contrasena+los-datos-coinciden?+volver-a-ingresar-los-datos-verifica-tus-datos-2+datos-almacenados+inicio-de-sesion-2","disenos/paso-4-nueva-contrasena/image.png+disenos/pagina-resultado/image.png",
```
