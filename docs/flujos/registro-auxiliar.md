# Flujo: registro de auxiliares

Registro de usuarios `AUXILIARY` ejecutado por un sujeto obligado (`WORKSPACE_ADMIN`). El auxiliar queda asociado al `registration_workspace` primario del sujeto obligado que lo registra.

**UI vs BE**: wizard de 3 pasos en cliente (Tipo de usuario → Datos del usuario → Revisión). BE tiene un único endpoint — recibe todo el payload junto, persiste en transacción única, sin borrador ni pasos intermedios. Capturas en [disenos/registro-auxiliar/](disenos/registro-auxiliar/).

## Restricciones

- Solo `WORKSPACE_ADMIN` puede registrar auxiliares. Nunca `SUPERADMIN` ni `AUXILIARY`.
- `workspaceId` se infiere del JWT — el front no lo manda. El service resuelve: `users.id → registration.user_id (COMPLETED) → registration_workspace (is_primary = true)`.
- El auxiliar tiene su propia tabla de perfil (`auxiliary_profile`). `users` guarda credenciales, role y FK al perfil vía `profile_type + profile_id`.
- Password generada al guardar, devuelta en el `201` una sola vez. `must_change_password = true`. Envío por correo fuera de scope.

## Asociación al workspace

`auxiliary_profile.workspace_id` (uuid, NOT NULL, FK → `registration_workspace.id`, `ON DELETE RESTRICT`).

- Se infiere del JWT del registrador — el front **no** lo manda.
- El service valida que exista una `registration` con `status = COMPLETED` para el usuario autenticado, y obtiene su workspace primario (`is_primary = true`).
- Si no hay registration completada o no hay workspace primario → `403`.

> **Decisión anterior revertida**: el diseño original usaba `parent_user_id → users.id`. Se migró a `workspace_id` para alinear la relación con la unidad organizacional del dominio.

## Schema

```
auxiliary_profile
  id                CHAR(36) PK
  workspace_id      CHAR(36) NOT NULL  FK → registration_workspace.id  ON DELETE RESTRICT
  rfc               VARCHAR(13) NOT NULL
  address_id        CHAR(36) NOT NULL  FK → reporting_entity_address.id  ON DELETE RESTRICT
  created_at, updated_at

INDEX idx_auxiliary_profile_workspace (workspace_id)
```

Piezas reutilizadas: `users`, `reporting_entity_address` (domicilio del auxiliar). No se usa `contact`, `registration` ni `vulnerable_activity`.

## Endpoint

`POST /registration/auxiliaries` — JWT requerido, `role = WORKSPACE_ADMIN`.

| Campo | Tipo | Req | Destino |
|---|---|---|---|
| `firstName` | string | ✅ | `users.first_name` |
| `paternalSurname` | string | ✅ | `users.paternal_surname` |
| `maternalSurname` | string | ✅ | `users.maternal_surname` |
| `rfc` | string | ✅ | `auxiliary_profile.rfc` |
| `phone` | string | ✅ | `users.phone` |
| `email` | string | ✅ | `users.email` |
| `address.state` | string | ✅ | `address_division` level 1 |
| `address.postalCode` | string | ✅ | `reporting_entity_address.postal_code` |
| `address.municipality` | string | ✅ | `address_division` level 2 |
| `address.neighborhood` | string | ✅ | `address_division` level 4 |
| `address.street` | string | ✅ | `reporting_entity_address.street` |
| `address.exteriorNumber` | string | ✅ | `reporting_entity_address.exterior_number` |
| `address.interiorNumber` | string | ❌ | `reporting_entity_address.interior_number` |

Errores: `400` DTO inválido · `401` sin JWT · `403` rol no permitido o sin registration completada · `409` email duplicado.

Response `201`: `{ user: <UserDTO>, temporaryPassword: "<plain>" }`.

## Lógica BE (transacción única)

1. Validar `parent.role = WORKSPACE_ADMIN` — `403` si no.
2. Resolver workspace: `findCompletedRegistrationByUserId` → `findPrimaryWorkspaceByRegistrationId` — `403` si alguno falla.
3. Pre-check email único → `409` si existe.
4. `INSERT reporting_entity_address`
5. `INSERT auxiliary_profile` (`workspace_id`, `rfc`, `address_id`)
6. Generar + hashear password.
7. `INSERT users` (`role=AUXILIARY`, `profile_type=AUXILIARY`, `profile_id=auxiliary_profile.id`, `must_change_password=true`)
8. Response `201` con `temporaryPassword` en claro.

## Diagrama

<!-- jarvis:diagram src=flujo.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Registro Auxiliar
direction: LR
nodes[8]{id,label,shape}:
  formulario-de-registro,Formulario de Registro,process
  nombre-s-apellido-paterno-apellido-materno-telefono-correo-electronico-rfc-direccion,Nombre(s)  Apellido Paterno  Apellido Materno  Teléfono  Correo electrónico  RFC  Dirección,data
  formulario-completo?,¿Formulario completo?,decision
  revisar-el-dato-faltante,Revisar el dato faltante,process
  modal-con-contrasena-creada,Modal con contraseña creada,process
  datos-almacenados,Datos Almacenados,data
  fin,fin,terminator
  inicio,Inicio,terminator
edges[8]{from,to,label}:
  formulario-de-registro "Formulario de Registro",nombre-s-apellido-paterno-apellido-materno-telefono-correo-electronico-rfc-direccion "Nombre(s)  Apellido Paterno  Apellido Materno  Teléfono  Correo electrónico  RFC  Dirección",
  nombre-s-apellido-paterno-apellido-materno-telefono-correo-electronico-rfc-direccion "Nombre(s)  Apellido Paterno  Apellido Materno  Teléfono  Correo electrónico  RFC  Dirección",formulario-completo? "¿Formulario completo?",
  formulario-completo? "¿Formulario completo?",revisar-el-dato-faltante "Revisar el dato faltante",No
  revisar-el-dato-faltante "Revisar el dato faltante",formulario-de-registro "Formulario de Registro",
  formulario-completo? "¿Formulario completo?",modal-con-contrasena-creada "Modal con contraseña creada",Sí
  modal-con-contrasena-creada "Modal con contraseña creada",datos-almacenados "Datos Almacenados",
  datos-almacenados "Datos Almacenados",fin "fin",
  inicio "Inicio",formulario-de-registro "Formulario de Registro",
```

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon que agrupa nodos del diagrama BE por pantalla UI." -->

```toon
steps[5]{step_ui,label_ui,variante,nodos_diagrama,disenos,nota}:
  1,Tipo de usuario,-,,,"disenos/registro-auxiliar/paso-1-tipo-usuario/1.png|2.png","Selección Auxiliar/Clientes es estado del front. Clientes=placeholder Próximamente, sin flujo."
  2,Datos del usuario,-,,"Req+Auth+Workspace+Dedupe","disenos/registro-auxiliar/paso-2-datos-usuario/1.png|2.png","Todo el DTO se captura aquí. Front bloquea Siguiente hasta completar."
  3,Revisión y validación,-,,"TX+InsAddr+InsProf+Pwd+InsUser+R201","disenos/registro-auxiliar/paso-3-revision/1.png|2.png","El POST ocurre al confirmar en este step."
  -,Modal resultado éxito,-,,R201,disenos/registro-auxiliar/pagina-resultado/success.png,Muestra temporaryPassword copiable.
  -,Modal resultado error,-,,"E403|E409",disenos/registro-auxiliar/pagina-resultado/error.png,
```
