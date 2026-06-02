# Tech Review: separar-domain-infra-auth-users

## Resumen del Propose

**Problema:**

El paquete `@pld-api/domain-auth-users` mezcla dominio puro con infraestructura (TypeORM, jsonwebtoken, bcrypt/crypto). Esto genera problemas arquitecturales:

1. `UserEntity` tiene decoradores TypeORM (`@Entity`, `@Column`, etc.) mezclados con campos de dominio
2. Los adapters (`UsersAdapter`, `AuthAdapter`) mezclan lógica de negocio con queries SQL/TypeORM
3. `jwt/sign.ts` es wrapper de `jsonwebtoken` — infraestructura exportada como dominio
4. `jti-store.ts` mezcla concepto de dominio (token único) con implementación in-memory
5. `factory.ts` acopla directamente a `DataSource` de TypeORM
6. Múltiples módulos de la app (`registration`, `auxiliaries`, `admin`) importan `UserEntity` directamente como si fuera un tipo de dominio

La intención es separar correctamente: el dominio define interfaces y tipos puros, la infraestructura implementa con TypeORM/JWT. El paquete `domain-auth-users` debería quedar solo con dominio puro (ports, tipos, reglas de negocio). Los adapters e entidades TypeORM deberían vivir fuera o en una subcapa de infraestructura dentro del mismo paquete.

Contexto adicional:
- La app `auth-users` es un monolito NestJS (no microservicios)
- Se usa Ports & Adapters (Hexagonal) como patrón objetivo
- El trabajo de `listWorkspaceAdminsPaginated` fue revertido y guardado en stash porque no había un lugar correcto para esa query de backoffice — la refactor debe resolver dónde va ese tipo de query
- No existe todavía un `domain-registration` ni `domain-admin`; esa deuda también debe considerarse en el scope o en los límites de esta propuesta

## Límites de la Solución (PM)

### Dentro del alcance

- Reorganizar el paquete domain-auth-users separando dominio puro de infraestructura dentro del mismo paquete (no crear paquetes nuevos)
- Definir tipo de dominio User puro (sin decoradores ORM) separado de UserEntity
- Mover adapters e infraestructura a subcapa infrastructure/ dentro del paquete
- Actualizar consumidores en apps/auth-users/src/ que importan UserEntity directamente
- Definir donde debe vivir una query de backoffice tipo listWorkspaceAdminsPaginated en la nueva estructura
- Reorganizar domain-auth-users creando subcapa infrastructure/ para adapters, entities, factory y jwt/sign
- Mover InMemoryJtiStore a infrastructure/ — JtiStore (interfaz) queda en dominio
- Actualizar index.ts para re-exportar los mismos simbolos publicos desde paths nuevos
- Actualizar imports en apps/auth-users/src/ que apuntan a UserEntity o adapters directamente
- Documentar donde va listWorkspaceAdminsPaginated post-refactor (AdminAdapter en apps/auth-users/src/admin/)

### Fuera del alcance

- Crear paquetes nuevos (domain-registration, domain-admin) — segunda fase
- Cambiar la API HTTP expuesta
- Cambiar el esquema de base de datos
- Reimplementar listWorkspaceAdminsPaginated — esta en stash, se reimplementa despues de esta refactor
- Crear tipo User puro separado de UserEntity — no vale la pena en monolito (mapeo doble sin beneficio real)
- Crear paquetes nuevos domain-registration o domain-admin
- Cambiar contratos de API HTTP
- Reimplementar listWorkspaceAdminsPaginated

### Restricciones de negocio

- El stack debe seguir compilando y arrancando al final de cada paso — no hay pasos rotos intermedios
- La refactor es interna — no cambia contratos de API
- Compilacion y arranque OK en cada paso incremental
- Mismos simbolos exportados por el paquete — sin breaking changes para consumidores

### Incógnitas PM

- [x] User puro separado de UserEntity vale la pena en monolito?
- [x] Donde va listWorkspaceAdminsPaginated post-refactor?
- [x] JtiStore ya tiene interfaz separada de InMemoryJtiStore?

## Revisión Técnica (DEV)

### Validación de límites

- undefined `undefined`

### Límites técnicos adicionales

- Dentro del mismo paquete domain-auth-users se crea subcapa infrastructure/ para: adapters/, entities/, factory.ts, jwt/sign.ts, jwt/jti-store.ts (InMemoryJtiStore)
- El dominio (ports/, crypto/, jwt/types.ts, jwt/jti-store.ts interfaz JtiStore) no importa typeorm ni jsonwebtoken
- UserEntity queda en infrastructure/entities/ — no se crea tipo User puro separado
- factory.ts se mueve a infrastructure/ porque requiere DataSource
- index.ts re-exporta los mismos simbolos publicos desde los nuevos paths — sin breaking changes
- listWorkspaceAdminsPaginated no va en UsersPort — su lugar es AdminAdapter en apps/auth-users/src/admin/ (fuera del paquete domain)

### Approach propuesto

Reorganizacion incremental en 4 pasos: 1) Crear infrastructure/ y mover archivos infra, 2) Actualizar imports internos dentro del paquete, 3) Actualizar index.ts, 4) Actualizar imports en apps/auth-users/src/. Cada paso debe compilar antes de continuar.

### Estimación


## Estado

finalized
