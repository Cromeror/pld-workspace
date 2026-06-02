# Proposal: separar-domain-infra-auth-users

## Problema

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

## Estado

pm_tech_review
