Flujo: Registro de Cliente

## Resumen

Flujo para registrar un cliente asociado a un sujeto obligado. El operador ingresa los datos del cliente según el tipo de persona (Física, Moral, Anexo 6-Bis, Anexo 7 o Fideicomiso). Cada tipo tiene su propio subflujo con apartados de identidad, domicilio, identificación oficial, beneficiario controlador y PEP. Los subflujos de Persona Física están organizados por nacionalidad (Mexicana / Extranjera) y modalidad (acude por su cuenta / acude con representante legal).

## Actores

- **Operador** (NOTARY / REAL_ESTATE) — ingresa y envía los datos del cliente.
- **Sistema** — valida los datos por apartado, persiste la información y enlaza al beneficiario controlador cuando aplica.

## Precondiciones

- El operador debe estar autenticado con un JWT válido.
- Debe existir al menos un workspace activo asociado al operador.

## Pasos

<!-- jarvis:diagram src=flujo.drawio notation=ansi-iso-5807 -->

```toon
diagram: flow
notation: ansi-iso-5807
page: Registro de Cliente
direction: TD
nodes[9]{id,label,shape}:
  inicio,inicio,terminator
  cliente-usuario,Cliente / Usuario,process
  con-beneficiario-controlador,Con beneficiario controlador se muestra la lista con los nombres agregados,process
  completar-anexo-bc,Completar anexo de beneficiario controlador,process
  tipo-persona?,Tipo de PERSONA,decision
  pf,PERSONA FÍSICA,offpage
  pm,PERSONA MORAL,offpage
  a6bis,ANEXO 6-Bis,offpage
  a7,ANEXO 7,offpage
  fideicomiso,FIDEICOMISO,offpage
edges[9]{from,to,label}:
  inicio "inicio",cliente-usuario "Cliente / Usuario",
  cliente-usuario "Cliente / Usuario",con-beneficiario-controlador "Con beneficiario controlador",
  con-beneficiario-controlador "Con beneficiario controlador",completar-anexo-bc "Completar anexo de beneficiario controlador",
  completar-anexo-bc "Completar anexo de beneficiario controlador",tipo-persona? "Tipo de PERSONA",
  cliente-usuario "Cliente / Usuario",tipo-persona? "Tipo de PERSONA",
  tipo-persona? "Tipo de PERSONA",pf "PERSONA FÍSICA",Persona Física
  tipo-persona? "Tipo de PERSONA",pm "PERSONA MORAL",Persona Moral
  tipo-persona? "Tipo de PERSONA",a6bis "ANEXO 6-Bis",Anexo 6-Bis
  tipo-persona? "Tipo de PERSONA",a7 "ANEXO 7",Anexo 7
  tipo-persona? "Tipo de PERSONA",fideicomiso "FIDEICOMISO",Fideicomiso
```

```toon
diagram: flow
notation: ansi-iso-5807
page: Persona Física
direction: TD
nodes[65]{id,label,shape}:
  pf-inicio,Persona Física,offpage
  nacionalidad?,NACIONALIDAD,decision
  mexicana,MEXICANA,decision
  extranjera,EXTRANJERA,decision
  mod-mex?,MODALIDAD,decision
  mod-ext?,MODALIDAD EXTRANJERA,decision
  mex-cuenta,Acude por su cuenta,offpage
  mex-rep,Acude con su representante legal,offpage
  ext-cuenta,Acude por su cuenta,offpage
  ext-rep,Acude con representante,offpage
  anexo-mex-cuenta,Anexo 3 PF Mexicana — acude por su cuenta,process
  anexo-mex-rep,Anexo 3 PF Mexicana — acude con representante,process
  anexo-ext-cuenta,Anexo 3 PF Extranjera — acude por su cuenta,process
  anexo-ext-rep,Anexo 3 PF Extranjera — acude con representante,process
  ap1-mc-nombre,Nombre Completo / Apellidos,data
  ap1-mc-fnac,Fecha de Nacimiento,data
  ap1-mc-pnac,País de Nacimiento,data
  ap1-mc-pnac2,País de Nacionalidad,data
  ap1-mc-lnac,Lugar de Nacimiento,data
  ap1-mc-ocup,Ocupación Profesión actividad o giro,data
  ap1-mc-email,Correo Electrónico,data
  ap1-mc-curp,CURP,data
  ap1-mc-rfc,RFC,data
  ap1-mc-val?,¿Los campos están completos?,decision
  ap1-mc-falta,Ingresa el dato faltante,process
  ap1-mc-guarda,Guarda la información,process
  dom-mc-tipo?,Domicilio Particular,decision
  dom-mc-nacional,Nacional,data
  dom-mc-extranjero,Extranjero,data
  dom-mc-calle,Calle avenida o vía,data
  dom-mc-next,Número exterior,data
  dom-mc-nint,Número interior,data
  dom-mc-col,Colonia o Urbanización,data
  dom-mc-mun,Municipio demarcación territorial o política,data
  dom-mc-ciudad,Ciudad o población,data
  dom-mc-ef,Entidad Federativa Estado provincia dpto,data
  dom-mc-cp,Código Postal,data
  dom-mc-pais,País,data
  dom-mc-tel,Teléfono en que se puede localizar con la extensión,data
  dom-mc-ext,Extensión,data
  dom-mc-val?,¿Los campos están completos?,decision
  dom-mc-falta,Ingresa el dato faltante,process
  dom-mc-guarda,Guarda la información,process
  id-mc-header,DATOS DE IDENTIFICACIÓN IFE INE pasaporte etc,process
  id-mc-nombre,Nombre del documento,data
  id-mc-num,Número que lo identifique,data
  id-mc-aut,Autoridad que lo emite,data
  id-mc-val?,¿Los campos están completos?,decision
  id-mc-falta,Ingresa el dato faltante,process
  id-mc-guarda,Guarda la información,process
  bc-header,BENEFICIARIO CONTROLADOR DUEÑO BENEFICIARIO,process
  bc-sel?,Selección,decision
  bc-agregar,Agregar Beneficiario controlador,process
  bc-nombre,Nombre(s) Primer Apellido y Segundo Apellido,data
  bc-guarda,Guarda la información,process
  pep-header,PERSONA POLÍTICAMENTE EXPUESTA cliente o poderdante,process
  pep-?,¿Usted desempeña o ha desempeñado funciones públicas destacadas PEP?,decision
  pep-cargo,Qué cargo que ocupa,data
  pep-familiar?,¿Es usted cónyuge o tiene parentesco hasta el segundo grado con PEP?,decision
  pep-fcargo,Qué cargo que ocupa,data
  pep-fnombre,Nombre completo del PEP,data
  pep-val?,¿Los campos están completos?,decision
  pep-falta,Ingresa el dato faltante,process
  pep-guarda,Guarda la información,process
  fin-pf,Fin,terminator
edges[60]{from,to,label}:
  pf-inicio "Persona Física",nacionalidad? "NACIONALIDAD",
  nacionalidad? "NACIONALIDAD",mexicana "MEXICANA",Mexicana
  nacionalidad? "NACIONALIDAD",extranjera "EXTRANJERA",Extranjera
  mexicana "MEXICANA",mod-mex? "MODALIDAD",
  extranjera "EXTRANJERA",mod-ext? "MODALIDAD EXTRANJERA",
  mod-mex? "MODALIDAD",mex-cuenta "Acude por su cuenta",Acude por su cuenta
  mod-mex? "MODALIDAD",mex-rep "Acude con su representante legal",Acude con representante
  mod-ext? "MODALIDAD EXTRANJERA",ext-cuenta "Acude por su cuenta",Acude por su cuenta
  mod-ext? "MODALIDAD EXTRANJERA",ext-rep "Acude con representante",Acude con representante
  mex-cuenta "Acude por su cuenta",anexo-mex-cuenta "Anexo 3 PF Mexicana — acude por su cuenta",
  mex-rep "Acude con su representante legal",anexo-mex-rep "Anexo 3 PF Mexicana — acude con representante",
  ext-cuenta "Acude por su cuenta",anexo-ext-cuenta "Anexo 3 PF Extranjera — acude por su cuenta",
  ext-rep "Acude con representante",anexo-ext-rep "Anexo 3 PF Extranjera — acude con representante",
  anexo-mex-cuenta "Anexo 3 PF Mexicana — acude por su cuenta",ap1-mc-nombre "Nombre Completo / Apellidos",
  ap1-mc-nombre "Nombre Completo / Apellidos",ap1-mc-fnac "Fecha de Nacimiento",
  ap1-mc-fnac "Fecha de Nacimiento",ap1-mc-pnac "País de Nacimiento",
  ap1-mc-pnac "País de Nacimiento",ap1-mc-pnac2 "País de Nacionalidad",
  ap1-mc-pnac2 "País de Nacionalidad",ap1-mc-lnac "Lugar de Nacimiento",
  ap1-mc-lnac "Lugar de Nacimiento",ap1-mc-ocup "Ocupación Profesión actividad o giro",
  ap1-mc-ocup "Ocupación Profesión actividad o giro",ap1-mc-email "Correo Electrónico",
  ap1-mc-email "Correo Electrónico",ap1-mc-curp "CURP",
  ap1-mc-curp "CURP",ap1-mc-rfc "RFC",
  ap1-mc-rfc "RFC",ap1-mc-val? "¿Los campos están completos?",
  ap1-mc-val? "¿Los campos están completos?",ap1-mc-falta "Ingresa el dato faltante",no
  ap1-mc-falta "Ingresa el dato faltante",ap1-mc-nombre "Nombre Completo / Apellidos",
  ap1-mc-val? "¿Los campos están completos?",ap1-mc-guarda "Guarda la información",si
  ap1-mc-guarda "Guarda la información",dom-mc-tipo? "Domicilio Particular",
  dom-mc-tipo? "Domicilio Particular",dom-mc-nacional "Nacional",Nacional
  dom-mc-tipo? "Domicilio Particular",dom-mc-extranjero "Extranjero",Extranjero
  dom-mc-nacional "Nacional",dom-mc-calle "Calle avenida o vía",
  dom-mc-extranjero "Extranjero",dom-mc-calle "Calle avenida o vía",
  dom-mc-calle "Calle avenida o vía",dom-mc-next "Número exterior",
  dom-mc-next "Número exterior",dom-mc-nint "Número interior",
  dom-mc-nint "Número interior",dom-mc-col "Colonia o Urbanización",
  dom-mc-col "Colonia o Urbanización",dom-mc-mun "Municipio demarcación territorial o política",
  dom-mc-mun "Municipio demarcación territorial o política",dom-mc-ciudad "Ciudad o población",
  dom-mc-ciudad "Ciudad o población",dom-mc-ef "Entidad Federativa Estado provincia dpto",
  dom-mc-ef "Entidad Federativa Estado provincia dpto",dom-mc-cp "Código Postal",
  dom-mc-cp "Código Postal",dom-mc-pais "País",
  dom-mc-pais "País",dom-mc-tel "Teléfono en que se puede localizar con la extensión",
  dom-mc-tel "Teléfono en que se puede localizar con la extensión",dom-mc-ext "Extensión",
  dom-mc-ext "Extensión",dom-mc-val? "¿Los campos están completos?",
  dom-mc-val? "¿Los campos están completos?",dom-mc-falta "Ingresa el dato faltante",no
  dom-mc-falta "Ingresa el dato faltante",dom-mc-calle "Calle avenida o vía",
  dom-mc-val? "¿Los campos están completos?",dom-mc-guarda "Guarda la información",si
  dom-mc-guarda "Guarda la información",id-mc-header "DATOS DE IDENTIFICACIÓN IFE INE pasaporte etc",
  id-mc-header "DATOS DE IDENTIFICACIÓN IFE INE pasaporte etc",id-mc-nombre "Nombre del documento",
  id-mc-nombre "Nombre del documento",id-mc-num "Número que lo identifique",
  id-mc-num "Número que lo identifique",id-mc-aut "Autoridad que lo emite",
  id-mc-aut "Autoridad que lo emite",id-mc-val? "¿Los campos están completos?",
  id-mc-val? "¿Los campos están completos?",id-mc-falta "Ingresa el dato faltante",no
  id-mc-falta "Ingresa el dato faltante",id-mc-nombre "Nombre del documento",
  id-mc-val? "¿Los campos están completos?",id-mc-guarda "Guarda la información",si
  id-mc-guarda "Guarda la información",bc-header "BENEFICIARIO CONTROLADOR DUEÑO BENEFICIARIO",
  bc-header "BENEFICIARIO CONTROLADOR DUEÑO BENEFICIARIO",bc-sel? "Selección",
  bc-sel? "Selección",bc-agregar "Agregar Beneficiario controlador",Tengo conocimiento de beneficiario controlador diferente
  bc-agregar "Agregar Beneficiario controlador",bc-nombre "Nombre(s) Primer Apellido y Segundo Apellido",
  bc-nombre "Nombre(s) Primer Apellido y Segundo Apellido",bc-guarda "Guarda la información",
  bc-sel? "Selección",pep-header "PERSONA POLÍTICAMENTE EXPUESTA cliente o poderdante",No aplica
  bc-guarda "Guarda la información",pep-header "PERSONA POLÍTICAMENTE EXPUESTA cliente o poderdante",
  pep-header "PERSONA POLÍTICAMENTE EXPUESTA cliente o poderdante",pep-? "¿Usted desempeña o ha desempeñado funciones públicas destacadas PEP?",
  pep-? "¿Usted desempeña o ha desempeñado funciones públicas destacadas PEP?",pep-cargo "Qué cargo que ocupa",Sí
  pep-cargo "Qué cargo que ocupa",pep-familiar? "¿Es usted cónyuge o tiene parentesco hasta el segundo grado con PEP?",
  pep-? "¿Usted desempeña o ha desempeñado funciones públicas destacadas PEP?",pep-familiar? "¿Es usted cónyuge o tiene parentesco hasta el segundo grado con PEP?",No
  pep-familiar? "¿Es usted cónyuge o tiene parentesco hasta el segundo grado con PEP?",pep-fcargo "Qué cargo que ocupa",Sí
  pep-fcargo "Qué cargo que ocupa",pep-fnombre "Nombre completo del PEP",
  pep-fnombre "Nombre completo del PEP",pep-val? "¿Los campos están completos?",
  pep-familiar? "¿Es usted cónyuge o tiene parentesco hasta el segundo grado con PEP?",pep-val? "¿Los campos están completos?",No
  pep-val? "¿Los campos están completos?",pep-falta "Ingresa el dato faltante",no
  pep-falta "Ingresa el dato faltante",pep-? "¿Usted desempeña o ha desempeñado funciones públicas destacadas PEP?",
  pep-val? "¿Los campos están completos?",pep-guarda "Guarda la información",si
  pep-guarda "Guarda la información",fin-pf "Fin",
```

```toon
diagram: flow
notation: ansi-iso-5807
page: Persona Moral
direction: TD
nodes[3]{id,label,shape}:
  pm-inicio,Persona Moral,offpage
  pm-pendiente,Subflujo pendiente de definir,process
  fin-pm,Fin,terminator
edges[2]{from,to,label}:
  pm-inicio "Persona Moral",pm-pendiente "Subflujo pendiente de definir",
  pm-pendiente "Subflujo pendiente de definir",fin-pm "Fin",
```

```toon
diagram: flow
notation: ansi-iso-5807
page: Anexo 6-Bis
direction: TD
nodes[3]{id,label,shape}:
  a6-inicio,Anexo 6-Bis,offpage
  a6-pendiente,Subflujo pendiente de definir,process
  fin-a6,Fin,terminator
edges[2]{from,to,label}:
  a6-inicio "Anexo 6-Bis",a6-pendiente "Subflujo pendiente de definir",
  a6-pendiente "Subflujo pendiente de definir",fin-a6 "Fin",
```

```toon
diagram: flow
notation: ansi-iso-5807
page: Anexo 7
direction: TD
nodes[3]{id,label,shape}:
  a7-inicio,Anexo 7,offpage
  a7-pendiente,Subflujo pendiente de definir,process
  fin-a7,Fin,terminator
edges[2]{from,to,label}:
  a7-inicio "Anexo 7",a7-pendiente "Subflujo pendiente de definir",
  a7-pendiente "Subflujo pendiente de definir",fin-a7 "Fin",
```

```toon
diagram: flow
notation: ansi-iso-5807
page: Fideicomiso
direction: TD
nodes[3]{id,label,shape}:
  fid-inicio,Fideicomiso,offpage
  fid-pendiente,Subflujo pendiente de definir,process
  fin-fid,Fin,terminator
edges[2]{from,to,label}:
  fid-inicio "Fideicomiso",fid-pendiente "Subflujo pendiente de definir",
  fid-pendiente "Subflujo pendiente de definir",fin-fid "Fin",
```

## Casos alternos

- **Dato faltante**: cualquier apartado con campo vacío → el sistema señala el campo y el operador lo completa antes de guardar.
- **Beneficiario controlador**: si el operador declara tener conocimiento de un BC diferente, se agrega el registro antes de continuar. El sistema muestra la lista acumulada de BCs al inicio del bloque.
- **PEP**: si el cliente o poderdante es PEP o familiar de PEP, se registran los datos del cargo y nombre del PEP de referencia.
- **Representante legal**: aplica solo en la modalidad "acude con representante legal"; agrega Apartado 3 con datos completos del representante e identificación oficial.

## Reglas de negocio

- El subflujo varía según **tipo de persona** (Física / Moral / Anexo 6-Bis / Anexo 7 / Fideicomiso).
- Persona Física se subdivide además por **nacionalidad** (Mexicana / Extranjera) y **modalidad** (por su cuenta / con representante legal).
- Cada apartado se guarda de forma independiente antes de continuar al siguiente.
- La sección de Beneficiario Controlador es transversal a todos los tipos de persona.

## Notas

- El diagrama BE (toon) es la fuente de verdad; el `.drawio` es una sombra generada por Jarvis.
- Los subflujos de Persona Moral, Anexo 6-Bis, Anexo 7 y Fideicomiso están pendientes de definir.
- Los diseños UI están pendientes — ver [disenos/](disenos/).

## Referencias

| Componente | Archivo |
|---|---|
| Diseños UI | [disenos/](disenos/) |

<!-- jarvis:llm-index type=flow-design-mapping hide=true description="Índice toon que agrupa nodos del diagrama BE por pantalla UI." -->

```toon
steps[7]{step_ui,label_ui,variante,nodos_diagrama,disenos,nota}:
  1,Selección tipo de persona,-,"inicio+cliente-usuario+con-beneficiario-controlador+completar-anexo-bc+tipo-persona?",,VACÍO: sin diseño UI
  2,Apartado I — Identidad,Persona Física,"ap1-mc-nombre+ap1-mc-fnac+ap1-mc-pnac+ap1-mc-pnac2+ap1-mc-lnac+ap1-mc-ocup+ap1-mc-email+ap1-mc-curp+ap1-mc-rfc+ap1-mc-val?+ap1-mc-falta+ap1-mc-guarda",,VACÍO: sin diseño UI
  3,Domicilio particular,Persona Física,"dom-mc-tipo?+dom-mc-nacional+dom-mc-extranjero+dom-mc-calle+dom-mc-next+dom-mc-nint+dom-mc-col+dom-mc-mun+dom-mc-ciudad+dom-mc-ef+dom-mc-cp+dom-mc-pais+dom-mc-tel+dom-mc-ext+dom-mc-val?+dom-mc-falta+dom-mc-guarda",,VACÍO: sin diseño UI
  4,Datos de identificación,Persona Física,"id-mc-header+id-mc-nombre+id-mc-num+id-mc-aut+id-mc-val?+id-mc-falta+id-mc-guarda",,VACÍO: sin diseño UI
  5,Beneficiario controlador,Transversal,"bc-header+bc-sel?+bc-agregar+bc-nombre+bc-guarda",,VACÍO: sin diseño UI
  6,PEP,Transversal,"pep-header+pep-?+pep-cargo+pep-familiar?+pep-fcargo+pep-fnombre+pep-val?+pep-falta+pep-guarda",,VACÍO: sin diseño UI
  7,Confirmación,-,"fin-pf",,VACÍO: sin diseño UI
```
