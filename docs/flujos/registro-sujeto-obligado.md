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

Diagrama solo del lado **BE** — el wizard de UI (capturas, copys, validación visual) vive en [disenos/registro-sujeto-obligado/](disenos/registro-sujeto-obligado/). Cada caja morada representa un endpoint del módulo `/admin/registration/*` y persiste un sub-recurso del registro.

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
  inicio,inicio,terminator
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
nodes[47]{id,label,shape}:
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
  fin,fin,terminator
  desea-agregar-otra-actividad-vulnerable?,¿Desea agregar otra actividad vulnerable?,decision
  actividad-vulnerable-2,Actividad vulnerable,offpage
  agrega-otro-contacto?,¿agrega otro contacto?,decision
  agregar-segunda-actividad-vulnerable-pf,agregar segunda actividad vulnerable pf,offpage
  retorno-actividad-vulnerable,retorno actividad vulnerable,offpage
  continuar-proceso-de-registro-principal,Continuar proceso de registro principal,offpage
edges[50]{from,to,label}:
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
  desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",agregar-segunda-actividad-vulnerable-pf "agregar segunda actividad vulnerable pf",si deseo agregar otra
  retorno-actividad-vulnerable "retorno actividad vulnerable",actividad-vulnerable "Actividad vulnerable*",
  retorno-actividad-vulnerable "retorno actividad vulnerable",actividad-vulnerable-realizada-en-el-domicilio "Actividad vulnerable realizada en el domicilio:",
  continuar-proceso-de-registro-principal "Continuar proceso de registro principal",los-campos-estan-completos?-3 "¿Los campos estan completos?",
  los-campos-estan-completos? "¿Los campos estan completos?",guarda-la-informacion "Guarda la información",
---
page: Registro personal moral
direction: LR
nodes[54]{id,label,shape}:
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
  fin,fin,terminator
  desea-agregar-otra-actividad-vulnerable?,¿Desea agregar otra actividad vulnerable?,decision
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,agregar segunda actividad vulnerable inmobiliaria pm,offpage
  retorno-actividad-vulnerable,retorno actividad vulnerable,offpage
  continuar-proceso-de-registro-principal,Continuar proceso de registro principal,offpage
edges[58]{from,to,label}:
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
  desea-agregar-otra-actividad-vulnerable? "¿Desea agregar otra actividad vulnerable?",agregar-segunda-actividad-vulnerable-inmobiliaria-pm "agregar segunda actividad vulnerable inmobiliaria pm",si
  continuar-proceso-de-registro-principal "Continuar proceso de registro principal",los-campos-estan-completos?-3 "¿Los campos están completos?",
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
---
page: Agregar segunda actividad vulnerable
direction: LR
nodes[6]{id,label,shape}:
  es-notario?,¿Es notario?,decision
  notarias,Notarias,offpage
  inmobiliarias,Inmobiliarias,offpage
  agregar-segunda-actividad-vulnerable-pf,agregar segunda actividad vulnerable pf,offpage
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,agregar segunda actividad vulnerable inmobiliaria pm,offpage
  continuar-proceso-de-registro-principal,Continuar proceso de registro principal,offpage
edges[6]{from,to,label}:
  es-notario? "¿Es notario?",notarias "Notarias",no es notario debe registrar actividad vulnerable de notarias
  es-notario? "¿Es notario?",inmobiliarias "Inmobiliarias",si es notario debe registrar actividad vulnerable inmobiliarias
  notarias "Notarias",continuar-proceso-de-registro-principal "Continuar proceso de registro principal",
  inmobiliarias "Inmobiliarias",continuar-proceso-de-registro-principal "Continuar proceso de registro principal",
  agregar-segunda-actividad-vulnerable-pf "agregar segunda actividad vulnerable pf",es-notario? "¿Es notario?",
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm "agregar segunda actividad vulnerable inmobiliaria pm",es-notario? "¿Es notario?",
```

> ⚠️ Envío de correo con la password queda **fuera de scope** en este flujo: no hay servicio de email implementado. La password temporal se devuelve en el response del `POST /:id/finalize` y la UI la muestra en modal copiable. Cuando exista servicio de email, se sumará en una feature posterior.

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon que relaciona nodos del diagrama BE con sus screenshots de diseño UI. Úsalo para navegar directo al archivo correcto sin leer el flujo completo." -->

```toon
mapping[42]{nodo,pagina_flujo,variante,diseno,nota}:
  crear-sujeto-obligado,Inicio,PF+PM,disenos/registro-sujeto-obligado/paso-1-sujeto-obligado/common.jpg,
  tipo-de-sujeto-obligado+tipo-de-perfil,Inicio,PF Notario,disenos/registro-sujeto-obligado/paso-1-sujeto-obligado/persona-fisica.jpg,Notario solo admite PF — única opción disponible
  tipo-de-sujeto-obligado+tipo-de-perfil,Inicio,PM Inmobiliaria,disenos/registro-sujeto-obligado/paso-1-sujeto-obligado/persona-moral.jpg,Inmobiliaria admite PF y PM
  step-1-datos-identificacion,Registro persona fisica,PF,disenos/registro-sujeto-obligado/paso-2-identificacion/persona-fisica.jpg,
  paso-1-datos-identificacion,Registro personal moral,PM,disenos/registro-sujeto-obligado/paso-2-identificacion/persona-moral.jpg,
  step-2-datos-contacto,Registro persona fisica,PF,disenos/registro-sujeto-obligado/paso-3-contacto/1.jpg,estado vacío — sin contactos
  step-2-datos-contacto,Registro persona fisica,PF,disenos/registro-sujeto-obligado/paso-3-contacto/2.jpg,estado con contacto agregado
  step-3-actividad-vulnerable+domicilio,Registro persona fisica,PF Notario,disenos/registro-sujeto-obligado/paso-4-actividad-vulnerable/persona-fisica.jpg,actividad pre-seleccionada READ-ONLY — no editable por usuario
  step-3-actividad-vulnerable+domicilio,Registro persona fisica,PF Notario múltiple,disenos/registro-sujeto-obligado/paso-4-actividad-vulnerable/persona-fisica-actividad-vulnerable.jpg,múltiples actividades agregadas
  paso-3-actividad-vulnerable+domicilio,Registro personal moral,PM Inmobiliaria,disenos/registro-sujeto-obligado/paso-4-actividad-vulnerable/persona-moral.jpg,actividad pre-seleccionada READ-ONLY
  paso-3-actividad-vulnerable+domicilio,Registro personal moral,PM Inmobiliaria múltiple,disenos/registro-sujeto-obligado/paso-4-actividad-vulnerable/persona-moral-con-actividad-vulnerable.jpg,múltiples actividades agregadas
  paso-4-responsable-cumplimiento,Registro personal moral,PM,disenos/registro-sujeto-obligado/paso-5-responsable-cumplimiento/1.jpg,step existe en diseño y componente FE pero NO está conectado al wizard
  revision-final,Registro persona fisica,PF Notario,disenos/registro-sujeto-obligado/paso-5-revision-persona-fisica/notario.jpg,
  revision-final,Registro persona fisica,PF Notario+AV,disenos/registro-sujeto-obligado/paso-5-revision-persona-fisica/notario-con-actividad-vulnerable.jpg,
  revision-final,Registro persona fisica,PF Inmobiliaria,disenos/registro-sujeto-obligado/paso-5-revision-persona-fisica/inmobiliaria.jpg,
  revision-final,Registro persona fisica,PF Inmobiliaria+AV,disenos/registro-sujeto-obligado/paso-5-revision-persona-fisica/inmobiliaria-con-actividad-vulnerable.jpg,
  confirmacion-modal,Revision,PF+PM,disenos/registro-sujeto-obligado/paso-5-revision-persona-fisica/confirmacion.jpg,dialog ¿Estás seguro/a? antes de finalizar
  resultado-exito,Result,PF+PM,disenos/registro-sujeto-obligado/pagina-resultado/result.jpg,modal muestra tempPassword — copy incorrecto promete envío de correo que NO existe
  resultado-error,Result,PF+PM,disenos/registro-sujeto-obligado/pagina-resultado/result-with-error.jpg,
  agregar-segunda-actividad-vulnerable-pf+es-notario?,Agregar segunda actividad vulnerable,Notario PF paso-01 vacío,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-01-tipo-persona.jpg,tipo persona READ-ONLY — Inmobiliaria seleccionada por sistema (cruce notario→inmobiliaria)
  agregar-segunda-actividad-vulnerable-pf+es-notario?,Agregar segunda actividad vulnerable,Notario PF paso-01 seleccionado,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-01-tipo-persona-seleccionado.jpg,
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario PF paso-02,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-02-identificacion.jpg,
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario PF paso-03 vacío,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-vacio.jpg,
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario PF paso-03 lleno,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-lleno.jpg,
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario PF paso-04,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-04-actividad-vulnerable.jpg,pre-selecciona TRANSMISION DE BIENES INMUEBLES — flujo inmobiliaria
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario PF paso-05,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-05-responsable-cumplimiento.jpg,solo aplica cuando tipo persona es PM
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario PF paso-06,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-06-revision.jpg,
  resultado-exito-modal,Agregar segunda actividad vulnerable,PF+PM,disenos/registro-sujeto-obligado/pagina-resultado/result.jpg,modal éxito tras guardar segunda actividad vulnerable
  agregar-segunda-actividad-vulnerable-pf+es-notario?,Agregar segunda actividad vulnerable,Notario→Inmobiliaria PF paso-01,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-01-tipo-persona.jpg,Inmobiliaria+PF pre-seleccionados READ-ONLY
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario→Inmobiliaria PF paso-02,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-02-identificacion.jpg,mismo diseño que PM — paso compartido
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario→Inmobiliaria PF paso-03 vacío,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-vacio.jpg,mismo diseño que PM — paso compartido
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario→Inmobiliaria PF paso-03 lleno,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-lleno.jpg,mismo diseño que PM — paso compartido
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario→Inmobiliaria PF paso-04 vacío,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-04-actividad-vulnerable-vacio.jpg,
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario→Inmobiliaria PF paso-04 lleno,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-04-actividad-vulnerable-lleno.jpg,
  agregar-segunda-actividad-vulnerable-pf,Agregar segunda actividad vulnerable,Notario→Inmobiliaria PF paso-05,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-pf/notario-agrega-inmobiliaria-pf-paso-05-revision.jpg,sin paso responsable — es PF
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm+es-notario?,Agregar segunda actividad vulnerable,Inmobiliaria→Notaria PF paso-01,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-01-tipo-persona.jpg,Notario+PF pre-seleccionados READ-ONLY — notaria solo admite PF
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,Agregar segunda actividad vulnerable,Inmobiliaria→Notaria PF paso-02,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-02-identificacion.jpg,mismo diseño — paso compartido
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,Agregar segunda actividad vulnerable,Inmobiliaria→Notaria PF paso-03 vacío,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-vacio.jpg,mismo diseño — paso compartido
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,Agregar segunda actividad vulnerable,Inmobiliaria→Notaria PF paso-03 lleno,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable/notario-agrega-inmobiliaria-pm-paso-03-contacto-lleno.jpg,mismo diseño — paso compartido
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,Agregar segunda actividad vulnerable,Inmobiliaria→Notaria PF paso-04 vacío,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-04-actividad-vulnerable-vacio.jpg,pre-selecciona FE PÚBLICA
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,Agregar segunda actividad vulnerable,Inmobiliaria→Notaria PF paso-04 lleno,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-04-actividad-vulnerable-lleno.jpg,
  agregar-segunda-actividad-vulnerable-inmobiliaria-pm,Agregar segunda actividad vulnerable,Inmobiliaria→Notaria PF paso-05,disenos/registro-sujeto-obligado/modal-agregar-actividad-vulnerable-notaria-pf/inmobiliaria-agrega-notaria-pf-paso-05-revision.jpg,sin responsable de cumplimiento — es PF
```
