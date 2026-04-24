# Domain Packages Specification

## Purpose

Definir el patrón arquitectónico para paquetes `domain-*` puros: sin dependencias de `@nestjs/*`, con inyección de DataSource vía factory y contrato de puerto expuesto al consumidor. Sirve como referencia canónica para `domain-auth-users` y para las subfases siguientes (`domain-participants`, etc.).

## Requirements

### Requirement: Paquete sin dependencias de framework

Un paquete `domain-*` MUST NOT importar ningún símbolo de `@nestjs/*` (incluyendo `@nestjs/common`, `@nestjs/typeorm`, `@nestjs/jwt`, `@nestjs/passport`). El paquete MUST compilar como TypeScript puro sin el runtime de NestJS.

#### Scenario: Compilación aislada sin NestJS

- GIVEN el directorio `packages/domain-*` con su `tsconfig.json` y `package.json`
- WHEN se compila el paquete de forma aislada (sin el contexto de la app)
- THEN la compilación termina sin errores
- AND ningún archivo fuente contiene `import ... from '@nestjs/...'`

#### Scenario: Decoradores TypeORM permitidos

- GIVEN una entidad usa decoradores de TypeORM (`@Entity`, `@Column`, etc.)
- WHEN se inspecciona el paquete
- THEN esos decoradores son aceptados porque `typeorm` no es `@nestjs/*`
- AND la entidad NO usa `@InjectRepository` ni decoradores de NestJS DI

### Requirement: Factory como único punto de creación

El paquete MUST exportar una función factory (p.ej. `createAuthUsersDomain`) que reciba un objeto de configuración con al menos `{ ds: DataSource }` y retorne un objeto que implementa el/los puertos del dominio.

#### Scenario: Factory retorna implementación del puerto

- GIVEN una instancia de `DataSource` de TypeORM ya conectada
- WHEN se llama `createAuthUsersDomain({ ds })`
- THEN retorna un objeto con los métodos definidos en los puertos del dominio
- AND el objeto satisface la interfaz del puerto en compile-time (tipado estático)

#### Scenario: Factory no lleva estado global

- GIVEN se llama `createAuthUsersDomain({ ds })` dos veces con instancias distintas de `DataSource`
- WHEN se usan ambas instancias retornadas
- THEN cada instancia opera sobre su propio `DataSource` sin interferir con la otra

### Requirement: Separación contrato / implementación

El paquete MUST exportar las interfaces (contratos de puerto) separadas de la implementación concreta. Los consumidores SHOULD depender de la interfaz, no de la clase concreta.

#### Scenario: Interfaz importable sin instanciar la implementación

- GIVEN un módulo consumidor importa solo el tipo del puerto (p.ej. `AuthPort`)
- WHEN se compila ese módulo sin llamar a la factory
- THEN compila sin error de tipos
- AND no hay efectos secundarios de inicialización de TypeORM

### Requirement: DataSource inyectado, no creado

El paquete MUST NOT crear ni conectar su propio `DataSource`. El consumidor (la app NestJS) MUST proveer una instancia ya conectada al llamar la factory.

#### Scenario: Factory recibe DataSource existente

- GIVEN la app NestJS tiene un `DataSource` conectado via `@pld-api/persistence`
- WHEN pasa ese `DataSource` a la factory del paquete domain
- THEN el paquete usa los repositorios de esa conexión sin crear una nueva

### Requirement: Alias registrado en tsconfig.base.json

Cada paquete `domain-*` nuevo MUST registrar su alias `@pld-api/<nombre>` en `tsconfig.base.json` para ser importable por las apps.

#### Scenario: Alias resuelve correctamente

- GIVEN `tsconfig.base.json` contiene el path `@pld-api/domain-auth-users`
- WHEN una app importa `import { ... } from '@pld-api/domain-auth-users'`
- THEN TypeScript resuelve el módulo sin error `Cannot find module`

### Requirement: Smoke test del patrón factory-adapter

El paquete `domain-auth-users` MUST incluir al menos 1 test que valide el flujo factory → adapter → operación de dominio usando una base de datos de prueba o un mock de `DataSource`.

#### Scenario: Smoke test pasa en CI

- GIVEN el paquete tiene un archivo `*.spec.ts` con el smoke test
- WHEN se ejecuta el runner de tests del paquete
- THEN el test corre (no es saltado por `passWithNoTests`)
- AND termina con exit code 0
