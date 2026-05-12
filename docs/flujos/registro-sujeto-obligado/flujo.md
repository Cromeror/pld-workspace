# Flujo de registro de sujetos obligados (SUPERADMIN)

> Diagrama de referencia para el registro de sujetos obligados por parte de un `SUPERADMIN`. Dos tipos de sujeto obligado (Notarías, Inmobiliarias) y dos tipos de perfil (Persona Física, Persona Moral).

## Referencias de código

### Enums canónicos (`packages/shared-types/src/enums.ts`)

| Concepto | Enum / tipo | Archivo | Valores |
|---|---|---|---|
| Tipo de usuario | `UserRole` | [pld-api/packages/shared-types/src/enums.ts:1](../pld-api/packages/shared-types/src/enums.ts#L1) | `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR`, `USUARIO_INTERNO`, `USUARIO_EXTERNO` |
| Tipo de perfil | `TipoPersonaParticipante` | [pld-api/packages/shared-types/src/enums.ts:10](../pld-api/packages/shared-types/src/enums.ts#L10) | `PERSONA_FISICA`, `PERSONA_MORAL`, `FIDEICOMISO` |
| Actividad vulnerable | `ActividadVulnerable` | [pld-api/packages/shared-types/src/enums.ts:26](../pld-api/packages/shared-types/src/enums.ts#L26) | `TRANSMISION_DERECHOS_REALES_INMUEBLES`, `PODERES_IRREVOCABLES`, `CONSTITUCION_SOCIEDADES`, `FUSION`, `ESCISION`, `AUMENTO_CAPITAL`, `DISMINUCION_CAPITAL`, `TRANSMISION_ACCIONES_PARTES`, `FIDEICOMISOS`, `MUTUOS_PRESTAMOS_CREDITOS` |

> Los mismos enums están re-exportados desde [pld-api/libs/catalogs/src/lib/catalogs.types.ts](../pld-api/libs/catalogs/src/lib/catalogs.types.ts) con nota de deprecation — preferir `@pld-api/shared-types`.

## Change SDD asociado

[openspec/changes/add-registro-sujetos-obligados/](../../openspec/changes/add-registro-sujetos-obligados/)

## Schema de base de datos

[REGISTRO_SCHEMA.md](../../arquitecturas/REGISTRO_SCHEMA.md)

## Diagrama

Diagrama solo del lado **BE** — el wizard de UI (capturas, copys, validación visual) vive en [disenos/](disenos/). Cada caja morada representa un endpoint del módulo `/admin/registration/*` y persiste un sub-recurso del registro.

> ⚠️ Dentro del flujo PM, un nodo "Persona Física" refiere al **responsable del cumplimiento de la Ley**, no al tipo de perfil.
> 📝 Las leyendas de "Actividad vulnerable" (FE PÚBLICA / TRANSMISIÓN DE BIENES INMUEBLES) son copy del front según el tipo de sujeto obligado — no son valores a persistir.

<!-- jarvis:diagram src=FLUJO_REGISTRO_REPORTING_ENTITIES.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Inicio
direction: LR
nodes[8]{id,label,shape}:
  crear-sujeto-obligado,Crear sujeto obligado,process
  tipo-de-sujeto-obligado-a-registrar,Tipo de sujeto obligado a registrar,decision
  inicio,inicio,process
  tipo-de-perfil,TIPO DE PERFIL,decision
  persona-fisica,persona fisica,offpage
  persona-moral,persona moral,offpage
  notarias,Notarias,offpage
  inmobiliarias,Inmobiliarias,offpage
edges[8]{from,to,label}:
  tipo-de-sujeto-obligado-a-registrar "Tipo de sujeto obligado a registrar",notarias "Notarias",
  crear-sujeto-obligado "Crear sujeto obligado",tipo-de-sujeto-obligado-a-registrar "Tipo de sujeto obligado a registrar",
  tipo-de-sujeto-obligado-a-registrar "Tipo de sujeto obligado a registrar",inmobiliarias "Inmobiliarias",
  inicio "inicio",crear-sujeto-obligado "Crear sujeto obligado",
  tipo-de-perfil "TIPO DE PERFIL",persona-fisica "persona fisica",elige
  tipo-de-perfil "TIPO DE PERFIL",persona-moral "persona moral",elige
  inmobiliarias "Inmobiliarias",tipo-de-perfil "TIPO DE PERFIL",
  notarias "Notarias",persona-fisica "persona fisica",unica opcion disponible
---
page: Registro persona fisica
direction: LR
nodes[49]{id,label,shape}:
  persona-fisica,persona fisica,offpage
  step-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable,step 1: Datos de identificación de quien realiza la Actividad Vulnerable,process
  nombre,Nombre*,data
  apellido-paterno,Apellido paterno*,data
  apellido-materno,Apellido materno*,data
  fecha-de-nacimiento,Fecha de nacimiento*,data
  rfc,RFC*,data
  curp,CURP*,data
  pais-de-nacionalidad,País de nacionalidad,data
  pais-de-nacimiento,País de nacimiento,data
  step-2-datos-de-contacto,step 2: Datos de Contacto,process
  clave-lada,Clave lada,data
  numero-de-telefono,Número de teléfono*,data
  correo-electronico,Correo electrónico*,data
  celular,Celular*,data
  step-3-actividad-vulnerable,step 3: Actividad Vulnerable,process
  actividad-vulnerable,Actividad vulnerable*,data
  fecha-inicial,Fecha inicial,data
  seccion-domicilio-principal-de-la-actividad-vulnerable,seccion: Domicilio principal de la actividad vulnerable,process
  codigo-postal,Código postal:,data
  entidad-federativa,Entidad federativa:,data
  delegacion-o-municipio,Delegación o municipio,data
  localidad,Localidad,data
  colonia,Colonia,data
  tipo-de-vialidad,Tipo de vialidad,data
  nombre-de-la-calle-o-v,Nombre de la calle o v...,data
  numero-exterior,Número exterior,data
  numero-interior,Número interior,data
  codigo-postal-2,Código postal,data
  guarda-la-informacion,Guarda la información,process
  los-campos-estan-completos?,¿Los campos estan completos?,decision
  ingresa-el-dato-faltante,Ingresa el dato faltante,process
  guarda-la-informacion-2,Guarda la información,process
  los-campos-estan-completos?-2,¿Los campos estan completos?,decision
  los-campos-estan-completos?-3,¿Los campos estan completos?,decision
  ingresa-el-dato-faltante-2,Ingresa el dato faltante,process
  actividad-vulnerable-realizada-en-el-domicilio,Actividad vulnerable realizada en el domicilio:,data
  ingresa-el-dato-faltante-3,Ingresa el dato faltante,process
  guarda-la-informacion-3,Guarda la información,process
  mensaje-informacion-de-perfil-actualizada,Mensaje: Información de perfil Actualizada,process
  fin,fin,process
  desea-agregar-otra-actividad-vulnerable?,¿Desea agregar otra actividad vulnerable?,decision
  es-notario?,¿Es notario?,decision
  notarias,Notarias,offpage
  inmobiliarias,Inmobiliarias,offpage
  actividad-vulnerable-2,Actividad vulnerable,offpage
  agrega-otro-contacto?,¿agrega otro contacto?,decision
  agregar-persona-vulerable,agregar persona vulerable,offpage
  retorno-actividad-vulnerable,retorno actividad vulnerable,offpage
edges[52]{from,to,label}:
  persona-fisica "persona fisica",step-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable "step 1: Datos de identificación de quien realiza la Actividad Vulnerable",
  step-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable "step 1: Datos de identificación de quien realiza la Actividad Vulnerable",nombre "Nombre*",
  nombre "Nombre*",apellido-paterno "Apellido paterno*",
  apellido-paterno "Apellido paterno*",apellido-materno "Apellido materno*",
  apellido-materno "Apellido materno*",fecha-de-nacimiento "Fecha de nacimiento*",
  fecha-de-nacimiento "Fecha de nacimiento*",rfc "RFC*",
  rfc "RFC*",curp "CURP*",
  curp "CURP*",pais-de-nacionalidad "País de nacionalidad",
  pais-de-nacionalidad "País de nacionalidad",pais-de-nacimiento "País de nacimiento",
  step-2-datos-de-contacto "step 2: Datos de Contacto",clave-lada "Clave lada",
  clave-lada "Clave lada",numero-de-telefono "Número de teléfono*",
  numero-de-telefono "Número de teléfono*",correo-electronico "Correo electrónico*",
  correo-electronico "Correo electrónico*",celular "Celular*",
  agrega-otro-contacto? "¿agrega otro contacto?",step-2-datos-de-contacto "step 2: Datos de Contacto",si  entonces se agrega otro formulario de contacto
  celular "Celular*",agrega-otro-contacto? "¿agrega otro contacto?",
  step-3-actividad-vulnerable "step 3: Actividad Vulnerable",actividad-vulnerable-2 "Actividad vulnerable",
  actividad-vulnerable "Actividad vulnerable*",fecha-inicial "Fecha inicial",
  fecha-inicial "Fecha inicial",seccion-domicilio-principal-de-la-actividad-vulnerable "seccion: Domicilio principal de la actividad vulnerable",
  seccion-domicilio-principal-de-la-actividad-vulnerable "seccion: Domicilio principal de la actividad vulnerable",codigo-postal "Código postal:",
  codigo-postal "Código postal:",entidad-federativa "Entidad federativa:",
  entidad-federativa "Entidad federativa:",delegacion-o-municipio "Delegación o municipio",
  delegacion-o-municipio "Delegación o municipio",localidad "Localidad",
  localidad "Localidad",colonia "Colonia",
  colonia "Colonia",tipo-de-vialidad "Tipo de vialidad",
  tipo-de-vialidad "Tipo de vialidad",nombre-de-la-calle-o-v "Nombre de la calle o v...",
  nombre-de-la-calle-o-v "Nombre de la calle o v...",numero-exterior "Número exterior",
  numero-exterior "Número exterior",numero-interior "Número interior",
  codigo-postal-2 "Código postal",actividad-vulnerable-realizada-en-el-domicilio "Actividad vulnerable realizada en el domicilio:",
  numero-interior "Número interior",codigo-postal-2 "Código postal",
  guarda-la-informacion "Guarda la información",step-2-datos-de-contacto "step 2: Datos de Contacto",siguiente step
  los-campos-estan-completos? "¿Los campos estan completos?",ingresa-el-dato-faltante "Ingresa el dato faltante",no estan completos los datos
  pais-de-nacimiento "País de nacimiento",los-campos-estan-completos? "¿Los campos estan completos?",continuar
  ingresa-el-dato-faltante "Ingresa el dato faltante",step-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable "step 1: Datos de identificación de quien realiza la Actividad Vulnerable",
  guarda-la-informacion-2 "Guarda la información",step-3-actividad-vulnerable "step 3: Actividad Vulnerable",siguiente paso
  los-campos-estan-completos?-2 "¿Los campos estan completos?",ingresa-el-dato-faltante-2 "Ingresa el dato faltante",no
  los-campos-estan-completos?-3 "¿Los campos estan completos?",ingresa-el-dato-faltante-3 "Ingresa el dato faltante",no
  agrega-otro-contacto? "¿agrega otro contacto?",los-campos-estan-completos?-2 "¿Los campos estan completos?",no  entonces continua
  los-campos-estan-completos?-2 "¿Los campos estan completos?",guarda-la-informacion-2 "Guarda la información",si estan completos los datos paso 2
  ingresa-el-dato-faltante-2 "Ingresa el dato faltante",step-2-datos-de-contacto "step 2: Datos de Contacto",
  desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",los-campos-estan-completos?-3 "¿Los campos estan completos?",no deseo agregar otra
  actividad-vulnerable-realizada-en-el-domicilio "Actividad vulnerable realizada en el domicilio:",desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",
  ingresa-el-dato-faltante-3 "Ingresa el dato faltante",step-3-actividad-vulnerable "step 3: Actividad Vulnerable",
  guarda-la-informacion-3 "Guarda la información",mensaje-informacion-de-perfil-actualizada "Mensaje: Información de perfil Actualizada",
  los-campos-estan-completos?-3 "¿Los campos estan completos?",guarda-la-informacion-3 "Guarda la información",si estan completos datos paso 3
  mensaje-informacion-de-perfil-actualizada "Mensaje: Información de perfil Actualizada",fin "fin",
  es-notario? "¿Es notario?",notarias "Notarias",si
  es-notario? "¿Es notario?",inmobiliarias "Inmobiliarias",no
  desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",agregar-persona-vulerable "agregar persona vulerable",si deseo agregar otra
  retorno-actividad-vulnerable "retorno actividad vulnerable",actividad-vulnerable "Actividad vulnerable*",
  retorno-actividad-vulnerable "retorno actividad vulnerable",actividad-vulnerable-realizada-en-el-domicilio "Actividad vulnerable realizada en el domicilio:",
  agregar-persona-vulerable "agregar persona vulerable",es-notario? "¿Es notario?",
  los-campos-estan-completos? "¿Los campos estan completos?",guarda-la-informacion "Guarda la información",
---
page: Registro personal moral
direction: LR
nodes[53]{id,label,shape}:
  persona-moral,persona moral,offpage
  paso-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable,paso 1: Datos de identificación de quien realiza la Actividad Vulnerable,process
  denominacion-o-razon-social,Denominación o razón social*:,data
  fecha-de-constitucion,Fecha de constitución:,data
  rfc,RFC*,data
  pais-de-nacionalidad,País de nacionalidad:,data
  los-campos-estan-completos?,¿Los campos están completos?,decision
  ingresa-el-dato-faltante,Ingresa el dato faltante,process
  guarda-la-informacion,Guarda la información,process
  paso-2-datos-de-contacto,paso 2: Datos de Contacto,process
  clave-lada,Clave lada,data
  numero-de-telefono,Número de teléfono:,data
  correo-electronico,Correo electrónico*,data
  celular,Celular*,data
  los-campos-estan-completos?-2,¿Los campos están completos?,decision
  ingresa-el-dato-faltante-2,Ingresa el dato faltante,process
  guarda-la-informacion-2,Guarda la información,process
  paso-3-actividad-vulnerable,paso 3: Actividad Vulnerable,process
  actividad-vulnerable,Actividad vulnerable*,data
  fecha-inicial,Fecha Inicial,data
  seccion-domicilio-principal-de-la-actividad-vulnerable,seccion: Domicilio principal de la actividad vulnerable,process
  codigo-postal,Código postal,data
  entidad-federativa,Entidad federativa,data
  delegacion-o-municipio,Delegación o municipio,data
  localidad,Localidad,data
  colonia,Colonia,data
  tipo-de-vialidad,Tipo de vialidad,data
  nombre-de-la-calle-o-v,Nombre de la calle o v...,data
  numero-exterior,Número exterior,data
  codigo-postal-2,Código postal:,data
  los-campos-estan-completos?-3,¿Los campos están completos?,decision
  ingresa-el-dato-faltante-3,Ingresa el dato faltante,process
  guarda-la-informacion-3,Guarda la información,process
  paso-4-responsable-del-cumplimiento-de-la-ley,paso 4: Responsable del Cumplimiento de la Ley,process
  nombre,Nombre*,data
  apellido-paterno,Apellido paterno*,data
  apellido-materno,Apellido materno*,data
  fecha-de-nacimiento,Fecha de nacimiento,data
  rfc-2,RFC*,data
  curp,CURP*,data
  pais-de-nacionalidad-2,País de nacionalidad,data
  fecha-de-designacion,Fecha de designación,data
  los-campos-estan-completos?-4,¿Los campos están completos?,decision
  ingresa-el-dato-faltante-4,Ingresa el dato faltante,process
  agrega-otro-contacto?,¿agrega otro contacto?,decision
  actividad-vulnerable-desde-persona-moral,Actividad vulnerable desde persona moral,offpage
  actividad-vulnerable-realizada-en-el-domicilio,Actividad vulnerable realizada en el domicilio:,data
  guarda-la-informacion-4,Guarda la información,process
  mensaje-informacion-de-perfil-actualizada,Mensaje: Información de perfil Actualizada,process
  fin,fin,process
  desea-agregar-otra-actividad-vulnerable?,¿Desea agregar otra actividad vulnerable?,decision
  agregar-persona-vulerable,agregar persona vulerable,offpage
  retorno-actividad-vulnerable,retorno actividad vulnerable,offpage
edges[57]{from,to,label}:
  persona-moral "persona moral",paso-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable "paso 1: Datos de identificación de quien realiza la Actividad Vulnerable",
  paso-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable "paso 1: Datos de identificación de quien realiza la Actividad Vulnerable",denominacion-o-razon-social "Denominación o razón social*:",
  denominacion-o-razon-social "Denominación o razón social*:",fecha-de-constitucion "Fecha de constitución:",
  fecha-de-constitucion "Fecha de constitución:",rfc "RFC*",
  rfc "RFC*",pais-de-nacionalidad "País de nacionalidad:",
  pais-de-nacionalidad "País de nacionalidad:",los-campos-estan-completos? "¿Los campos están completos?",continuar
  ingresa-el-dato-faltante "Ingresa el dato faltante",paso-1-datos-de-identificacion-de-quien-realiza-la-actividad-vulnerable "paso 1: Datos de identificación de quien realiza la Actividad Vulnerable",
  los-campos-estan-completos? "¿Los campos están completos?",ingresa-el-dato-faltante "Ingresa el dato faltante",No
  los-campos-estan-completos? "¿Los campos están completos?",guarda-la-informacion "Guarda la información",Sí
  guarda-la-informacion "Guarda la información",paso-2-datos-de-contacto "paso 2: Datos de Contacto",siguiente paso
  paso-2-datos-de-contacto "paso 2: Datos de Contacto",clave-lada "Clave lada",
  clave-lada "Clave lada",numero-de-telefono "Número de teléfono:",
  numero-de-telefono "Número de teléfono:",correo-electronico "Correo electrónico*",
  correo-electronico "Correo electrónico*",celular "Celular*",
  celular "Celular*",agrega-otro-contacto? "¿agrega otro contacto?",
  ingresa-el-dato-faltante-2 "Ingresa el dato faltante",paso-2-datos-de-contacto "paso 2: Datos de Contacto",
  los-campos-estan-completos?-2 "¿Los campos están completos?",ingresa-el-dato-faltante-2 "Ingresa el dato faltante",No
  los-campos-estan-completos?-2 "¿Los campos están completos?",guarda-la-informacion-2 "Guarda la información",Sí
  guarda-la-informacion-2 "Guarda la información",paso-3-actividad-vulnerable "paso 3: Actividad Vulnerable",
  paso-3-actividad-vulnerable "paso 3: Actividad Vulnerable",actividad-vulnerable-desde-persona-moral "Actividad vulnerable desde persona moral",
  actividad-vulnerable "Actividad vulnerable*",fecha-inicial "Fecha Inicial",
  fecha-inicial "Fecha Inicial",seccion-domicilio-principal-de-la-actividad-vulnerable "seccion: Domicilio principal de la actividad vulnerable",
  seccion-domicilio-principal-de-la-actividad-vulnerable "seccion: Domicilio principal de la actividad vulnerable",codigo-postal "Código postal",
  codigo-postal "Código postal",entidad-federativa "Entidad federativa",
  entidad-federativa "Entidad federativa",delegacion-o-municipio "Delegación o municipio",
  delegacion-o-municipio "Delegación o municipio",localidad "Localidad",
  localidad "Localidad",colonia "Colonia",
  colonia "Colonia",tipo-de-vialidad "Tipo de vialidad",
  tipo-de-vialidad "Tipo de vialidad",nombre-de-la-calle-o-v "Nombre de la calle o v...",
  nombre-de-la-calle-o-v "Nombre de la calle o v...",numero-exterior "Número exterior",
  numero-exterior "Número exterior",codigo-postal-2 "Código postal:",
  codigo-postal-2 "Código postal:",actividad-vulnerable-realizada-en-el-domicilio "Actividad vulnerable realizada en el domicilio:",
  desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",los-campos-estan-completos?-3 "¿Los campos están completos?",No deseo agregar otra
  ingresa-el-dato-faltante-3 "Ingresa el dato faltante",paso-3-actividad-vulnerable "paso 3: Actividad Vulnerable",
  los-campos-estan-completos?-3 "¿Los campos están completos?",ingresa-el-dato-faltante-3 "Ingresa el dato faltante",No estan completos
  los-campos-estan-completos?-3 "¿Los campos están completos?",guarda-la-informacion-3 "Guarda la información",Sí
  guarda-la-informacion-3 "Guarda la información",paso-4-responsable-del-cumplimiento-de-la-ley "paso 4: Responsable del Cumplimiento de la Ley",
  paso-4-responsable-del-cumplimiento-de-la-ley "paso 4: Responsable del Cumplimiento de la Ley",nombre "Nombre*",
  nombre "Nombre*",apellido-paterno "Apellido paterno*",
  apellido-paterno "Apellido paterno*",apellido-materno "Apellido materno*",
  apellido-materno "Apellido materno*",fecha-de-nacimiento "Fecha de nacimiento",
  fecha-de-nacimiento "Fecha de nacimiento",rfc-2 "RFC*",
  rfc-2 "RFC*",curp "CURP*",
  curp "CURP*",pais-de-nacionalidad-2 "País de nacionalidad",
  pais-de-nacionalidad-2 "País de nacionalidad",fecha-de-designacion "Fecha de designación",
  fecha-de-designacion "Fecha de designación",los-campos-estan-completos?-4 "¿Los campos están completos?",
  ingresa-el-dato-faltante-4 "Ingresa el dato faltante",paso-4-responsable-del-cumplimiento-de-la-ley "paso 4: Responsable del Cumplimiento de la Ley",
  los-campos-estan-completos?-4 "¿Los campos están completos?",ingresa-el-dato-faltante-4 "Ingresa el dato faltante",No
  los-campos-estan-completos?-4 "¿Los campos están completos?",guarda-la-informacion-4 "Guarda la información",Sí
  agrega-otro-contacto? "¿agrega otro contacto?",paso-2-datos-de-contacto "paso 2: Datos de Contacto",si  entonces se agrega otro formulario de contacto
  agrega-otro-contacto? "¿agrega otro contacto?",los-campos-estan-completos?-2 "¿Los campos están completos?",no  entonces continua
  retorno-actividad-vulnerable "retorno actividad vulnerable",actividad-vulnerable "Actividad vulnerable*",
  retorno-actividad-vulnerable "retorno actividad vulnerable",actividad-vulnerable-realizada-en-el-domicilio "Actividad vulnerable realizada en el domicilio:",
  actividad-vulnerable-realizada-en-el-domicilio "Actividad vulnerable realizada en el domicilio:",desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",
  guarda-la-informacion-4 "Guarda la información",mensaje-informacion-de-perfil-actualizada "Mensaje: Información de perfil Actualizada",
  mensaje-informacion-de-perfil-actualizada "Mensaje: Información de perfil Actualizada",fin "fin",
  desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",agregar-persona-vulerable "agregar persona vulerable",si
---
page: Actividad vulnerable
direction: LR
nodes[6]{id,label,shape}:
  actividad-vulnerable,Actividad vulnerable,offpage
  es-inmobiliaria?,¿Es inmobiliaria?,decision
  selecciona-por-defecto-fe-publica-notarios-y-corredores,selecciona por defecto: FE PÚBLICA (NOTARIOS Y CORREDORES),process
  selecciona-por-defecto-transmision-de-bienes-inmuebles,selecciona por defecto: TRANSMISION DE BIENES INMUEBLES,process
  actividad-vulnerable-desde-persona-moral,Actividad vulnerable desde persona moral,offpage
  retorno-actividad-vulnerable,retorno actividad vulnerable,offpage
edges[6]{from,to,label}:
  actividad-vulnerable "Actividad vulnerable",es-inmobiliaria? "¿Es inmobiliaria?",
  es-inmobiliaria? "¿Es inmobiliaria?",selecciona-por-defecto-fe-publica-notarios-y-corredores "selecciona por defecto: FE PÚBLICA (NOTARIOS Y CORREDORES)",no
  es-inmobiliaria? "¿Es inmobiliaria?",selecciona-por-defecto-transmision-de-bienes-inmuebles "selecciona por defecto: TRANSMISION DE BIENES INMUEBLES",si
  selecciona-por-defecto-fe-publica-notarios-y-corredores "selecciona por defecto: FE PÚBLICA (NOTARIOS Y CORREDORES)",retorno-actividad-vulnerable "retorno actividad vulnerable",
  selecciona-por-defecto-transmision-de-bienes-inmuebles "selecciona por defecto: TRANSMISION DE BIENES INMUEBLES",retorno-actividad-vulnerable "retorno actividad vulnerable",
  actividad-vulnerable-desde-persona-moral "Actividad vulnerable desde persona moral",es-inmobiliaria? "¿Es inmobiliaria?",
```

> ⚠️ Envío de correo con la password queda **fuera de scope** en este flujo: no hay servicio de email implementado. La password temporal se devuelve en el response del `POST /:id/finalize` y la UI la muestra en modal copiable. Cuando exista servicio de email, se sumará en una feature posterior.

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon que agrupa nodos del diagrama BE por step UI. Cada fila = una pantalla del wizard. nodos_diagrama lista todos los nodos BE que esa pantalla implementa (decisiones internas incluidas). disenos lista los screenshots disponibles. nota registra inconsistencias o vacíos pendientes." -->

```toon
steps[8]{step_ui,label_ui,variante,paginas_diagrama,nodos_diagrama,disenos,nota}:
  1,Sujeto obligado,PF+PM,Inicio,"inicio+crear-sujeto-obligado+tipo-de-sujeto-obligado-a-registrar+notarias+inmobiliarias+tipo-de-perfil+persona-fisica+persona-moral","paso-1-sujeto-obligado/common.jpg|persona-fisica.jpg|persona-moral.jpg","Notario→solo PF (única opción); Inmobiliaria→PF o PM. La decisión tipo-de-perfil ocurre en esta misma pantalla antes de avanzar"
  2,Datos de identificación,PF,"Registro persona fisica","step-1-datos-de-identificacion+nombre+apellido-paterno+apellido-materno+fecha-de-nacimiento+rfc+curp+pais-de-nacionalidad+pais-de-nacimiento+los-campos-estan-completos?+ingresa-el-dato-faltante+guarda-la-informacion",paso-2-identificacion/persona-fisica.jpg,Validación inline — los nodos decision/ingresa-dato-faltante son comportamiento FE no pantalla separada
  2,Datos de identificación,PM,"Registro personal moral","paso-1-datos-de-identificacion+denominacion-o-razon-social+fecha-de-constitucion+rfc+pais-de-nacionalidad+los-campos-estan-completos?+ingresa-el-dato-faltante+guarda-la-informacion",paso-2-identificacion/persona-moral.jpg,
  3,Datos de Contacto,PF+PM,"Registro persona fisica|Registro personal moral","step-2-datos-de-contacto|paso-2-datos-de-contacto+clave-lada+numero-de-telefono+correo-electronico+celular+agrega-otro-contacto?+los-campos-estan-completos?-2+ingresa-el-dato-faltante-2+guarda-la-informacion-2","paso-3-contacto/1.jpg|paso-3-contacto/2.jpg","1.jpg=vacío; 2.jpg=con contacto. agrega-otro-contacto? es loop dentro de la misma pantalla"
  4,Actividad vulnerable,PF+PM,"Registro persona fisica|Registro personal moral|Actividad vulnerable","step-3-actividad-vulnerable|paso-3-actividad-vulnerable+actividad-vulnerable-2|actividad-vulnerable-desde-persona-moral+es-inmobiliaria?+selecciona-por-defecto-fe-publica|selecciona-por-defecto-transmision+retorno-actividad-vulnerable+actividad-vulnerable+fecha-inicial+seccion-domicilio+codigo-postal+entidad-federativa+delegacion-o-municipio+localidad+colonia+tipo-de-vialidad+nombre-de-la-calle+numero-exterior+numero-interior+actividad-vulnerable-realizada-en-el-domicilio+desea-agregar-otra-actividad-vulnerable?+agregar-persona-vulnerable+los-campos-estan-completos?-3+ingresa-el-dato-faltante-3+guarda-la-informacion-3","paso-4-actividad-vulnerable/persona-fisica.jpg|persona-fisica-actividad-vulnerable.jpg|persona-moral.jpg|persona-moral-con-actividad-vulnerable.jpg","La página Actividad vulnerable del diagrama es lógica interna de esta pantalla — pre-selecciona el valor según es-inmobiliaria? y retorna. Campo READ-ONLY no editable por usuario. Loop agregar actividad ocurre dentro de la misma pantalla"
  5,Responsable del cumplimiento,PM,"Registro personal moral","paso-4-responsable-del-cumplimiento-de-la-ley+nombre+apellido-paterno+apellido-materno+fecha-de-nacimiento+rfc-2+curp+pais-de-nacionalidad-2+fecha-de-designacion+los-campos-estan-completos?-4+ingresa-el-dato-faltante-4+guarda-la-informacion-4",paso-5-responsable-cumplimiento/1.jpg,VACÍO: step existe en diseño y componente FE pero NO conectado al wizard PM
  6,Revisión y validación,PF,"Registro persona fisica","los-campos-estan-completos?-3→[UI: pantalla resumen]→[UI: modal confirmación]→guarda-la-informacion-3+mensaje-informacion-de-perfil-actualizada+fin","paso-5-revision-persona-fisica/notario.jpg|notario-con-actividad-vulnerable.jpg|inmobiliaria.jpg|inmobiliaria-con-actividad-vulnerable.jpg|confirmacion.jpg","La revisión y el modal de confirmación son estado del front — no tienen nodo BE propio. guarda-la-informacion-3 es el POST que se dispara al confirmar"
  6,Revisión y validación,PM,"Registro personal moral","los-campos-estan-completos?-4→[UI: pantalla resumen]→[UI: modal confirmación]→guarda-la-informacion-4+mensaje-informacion-de-perfil-actualizada+fin",,Sin diseño UI — implementar con resumen genérico similar a revisión PF hasta que el cliente provea mockup
  -,Modal resultado,PF+PM,-,fin,"pagina-resultado/result.jpg|result-with-error.jpg","INCONSISTENCIA: copy del modal promete envío de correo — el servicio de email no existe. Ver nota ⚠️ al pie del diagrama"
```
