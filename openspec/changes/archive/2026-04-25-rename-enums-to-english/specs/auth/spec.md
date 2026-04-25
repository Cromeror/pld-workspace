# Delta for Auth

## MODIFIED Requirements

### Requirement: UserRole ENUM — columna users.role

La columna `users.role` en MySQL MUST aceptar exactamente los valores `'SUPERADMIN'`, `'NOTARY'`, `'REAL_ESTATE'`, `'AUXILIARY'` al finalizar la migración. Los valores previos en español MUST NOT ser aceptados por el ENUM después de la restricción final.

#### INV-1: Columna role acepta solo valores English post-migración

**Given** la migración up fue ejecutada a completitud,
**When** se inspecciona la definición del ENUM en `INFORMATION_SCHEMA`,
**Then** los valores permitidos SHALL ser exactamente `('SUPERADMIN','NOTARY','REAL_ESTATE','AUXILIARY')`,
**AND** los valores `'NOTARIO'`, `'INMOBILIARIA'`, `'AUXILIAR'` SHALL NOT existir en la definición.

#### INV-2: Filas existentes migradas a valores English

**Given** la migración up fue ejecutada,
**When** se consulta `SELECT role FROM users`,
**Then** ninguna fila SHALL contener `'NOTARIO'`, `'INMOBILIARIA'` ni `'AUXILIAR'`,
**AND** todas las filas con `'NOTARIO'` previo SHALL tener `role = 'NOTARY'`,
**AND** todas las filas con `'INMOBILIARIA'` previo SHALL tener `role = 'REAL_ESTATE'`,
**AND** todas las filas con `'AUXILIAR'` previo SHALL tener `role = 'AUXILIARY'`,
**AND** las filas con `'SUPERADMIN'` previo SHALL permanecer sin cambios.

#### INV-5: Down migration restaura valores Spanish

**Given** la migración up fue ejecutada,
**When** se ejecuta la migración down,
**Then** la definición del ENUM SHALL ser restaurada a los valores Spanish originales,
**AND** todas las filas SHALL haber sido revertidas a sus valores Spanish correspondientes.

### Requirement: JWT role claim — valores English

`POST /auth/login` MUST emitir el JWT con el claim `role` usando los nuevos valores English. No MUST NOT existir lógica de compatibilidad hacia atrás que decodifique valores Spanish del claim `role`.

#### INV-3: Login exitoso emite JWT con role en English

**Given** un usuario con `role = 'NOTARY'` en BD (post-migración),
**When** el cliente envía `POST /auth/login` con credenciales válidas,
**Then** el servidor SHALL responder HTTP 201 con `{ token: string }`,
**AND** el claim `role` del JWT SHALL ser `"NOTARY"` (o `"REAL_ESTATE"` / `"AUXILIARY"` / `"SUPERADMIN"` según corresponda),
**AND** el claim `role` SHALL NOT contener valores Spanish.

#### INV-4: GET /auth/me retorna role en English

**Given** un JWT válido cuyo `sub` referencia un usuario activo post-migración,
**When** el cliente envía `GET /auth/me`,
**Then** el servidor SHALL responder HTTP 200,
**AND** el campo `role` en el body SHALL ser uno de `"SUPERADMIN"`, `"NOTARY"`, `"REAL_ESTATE"`, `"AUXILIARY"`.
