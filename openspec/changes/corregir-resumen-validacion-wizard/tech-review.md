# Tech Review: corregir-resumen-validacion-wizard

## Resumen del Propose

**Problema:**

## Contexto

El wizard de registro de sujeto obligado (notarías/inmobiliarias) tiene un paso de Revisión y validación (ReviewStep) que muestra un resumen de todos los datos antes de confirmar. Se detectaron 4 problemas en este paso:

## Problemas a resolver

### 1. Alerta correo → usuario
El primer contacto agregado define el correo que se usará como credencial de acceso al sistema. El usuario no sabe esto. Se necesita mostrar una alerta/aviso visible en el paso de Contacto (o en el resumen) que diga que el correo del primer contacto será usado para crear el acceso al sistema.

### 2. Celular no aparece en resumen
En la sección 'Datos de Contacto' del resumen, solo se muestran: Clave lada, Número de teléfono, Correo electrónico. Falta mostrar el Número de celular. Aplica tanto para la actividad principal como para la secundaria (SecondActivityBlock).

Código actual (ReviewStep líneas 249-253 y 164-168):
```
const contactItems = state.contacts.flatMap((c) => [
  { title: 'Clave lada', value: c.countryCode ?? '' },
  { title: 'Numero de teléfono', value: c.phone ?? '' },
  { title: 'Correo electrónico', value: c.email ?? '' },
  // FALTA: cellphone
]);
```

### 3. División administrativa muestra fieldName (ID) en lugar de nombre legible
En la sección de domicilio de la actividad vulnerable, los campos de división administrativa (entidad federativa, municipio, colonia, etc.) muestran el `fieldName` técnico (ej: 'state', 'municipality', 'neighborhood') cuando no se encuentra el label en el mapa. El label debería venir del catálogo `useGetAdministrativeStructure`. El problema es que `VulnerableActivitySection` llama `useGetAdministrativeStructure(COUNTRY_CODE_MX)` internamente, pero el hook puede retornar `undefined` en el primer render. Aplica a actividad principal y secundaria.

Código actual (línea 88-99):
```
const labelMap = new Map((structure?.levels ?? []).map((l) => [l.fieldName, l.label]));
const divisionItems = (va.address?.divisions ?? [])
  .sort((a, b) => a.level - b.level)
  .map((d) => ({ title: labelMap.get(d.fieldName) ?? d.fieldName, value: d.name }));
```
Cuando `structure` es undefined, `labelMap` está vacío y el fallback es `d.fieldName` (el ID técnico).

### 4. Actividad secundaria duplica colonia
En `SecondActivityBlock`, la actividad vulnerable secundaria usa `VulnerableActivitySection` que llama `useGetAdministrativeStructure(COUNTRY_CODE_MX)` internamente. El problema con la colonia duplicada se investiga: posiblemente las `divisions` del estado de la actividad secundaria tienen una entrada duplicada para el nivel de colonia, o el componente renderiza la colonia dos veces (una desde `divisions` y otra hardcoded).

## Archivos afectados
- `pld-web/src/components/organisms/reporting-entity/ReviewStep/index.tsx`
- `pld-web/src/components/organisms/reporting-entity/ContactStep/index.tsx` (para la alerta del correo)
- `pld-web/src/components/organisms/reporting-entity/AddVulnerableActivityModal/index.tsx` (para verificar cómo se guardan las divisions de la actividad secundaria)

## Restricciones
- Solo FE, sin cambios en BE ni base de datos
- No rediseñar el componente, solo corregir los bugs y agregar el aviso

## Límites de la Solución (PM)

### Dentro del alcance

- 1. Agregar aviso en ContactStep: el correo del primer contacto se usará como credencial de acceso.
- 2. Agregar celular en sección Datos de Contacto del resumen (actividad principal y secundaria).
- 3. Corregir labels de divisiones administrativas en domicilio de actividad vulnerable (principal y secundaria).
- 4. Investigar y corregir colonia duplicada en actividad secundaria.
- Agregar aviso en ContactStep: el correo del primer contacto se usará como credencial de acceso
- Agregar celular en sección Datos de Contacto del resumen (actividad principal y secundaria)
- Corregir labels de divisiones administrativas en domicilio de actividad vulnerable (principal y secundaria)
- Corregir colonia duplicada en actividad secundaria

### Fuera del alcance

- C
- a
- m
- b
- i
- o
- s
-  
- e
- n
-  
- B
- E
-  
- o
-  
- b
- a
- s
- e
-  
- d
- e
-  
- d
- a
- t
- o
- s
- .
-  
- R
- e
- d
- i
- s
- e
- ñ
- o
-  
- d
- e
- l
-  
- c
- o
- m
- p
- o
- n
- e
- n
- t
- e
-  
- R
- e
- v
- i
- e
- w
- S
- t
- e
- p
- .
- B
- E
- ,
-  
- D
- B
- ,
-  
- r
- e
- d
- i
- s
- e
- ñ
- o
- .
- Cambios en BE o base de datos
- Rediseño del componente ReviewStep

### Restricciones de negocio

- S
- o
- l
- o
-  
- f
- r
- o
- n
- t
- e
- n
- d
- .
-  
- L
- o
- s
-  
- 4
-  
- f
- i
- x
- e
- s
-  
- v
- a
- n
-  
- e
- n
-  
- R
- e
- v
- i
- e
- w
- S
- t
- e
- p
- /
- i
- n
- d
- e
- x
- .
- t
- s
- x
-  
- y
-  
- C
- o
- n
- t
- a
- c
- t
- S
- t
- e
- p
- /
- i
- n
- d
- e
- x
- .
- t
- s
- x
- .
- S
- o
- l
- o
-  
- F
- E
- .
- Solo frontend
- Los 4 fixes van en ReviewStep/index.tsx y ContactStep/index.tsx

### Incógnitas PM

- [ ] undefined

## Estado

pm_tech_review
