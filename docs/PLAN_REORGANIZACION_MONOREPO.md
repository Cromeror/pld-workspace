# Plan de Reorganización a Monorepo Modular

**Proyecto**: pld-api
**Fecha inicial**: 2026-04-17
**Última actualización**: 2026-04-20
**Autor**: Cristobal Romero

---

## 1. Contexto y Objetivo

### Estado actual

El proyecto es un monorepo Nx 14 con **3 apps NestJS** (`auth-users`, `catalogs`, `cross`) y **11 libs** (originalmente 12; `persistencia-sql` eliminada durante la migración). Las libs usan decoradores de NestJS (`@Module`, `@Injectable`, `TypeOrmModule`) incluso cuando solo exportan tipos. No existe comunicación entre apps — cada una sirve un endpoint HTTP separado pero comparten la misma base de datos MySQL.

A la fecha de esta actualización ya se ejecutó la **Fase 0** (fundaciones) y pasos sueltos del camino. Las 3 apps siguen funcionando y el login responde HTTP 201.

### Objetivo

Reorganizar el código para:

1. **Paquetes de dominio puros** (sin NestJS) que ejecutan lógica de negocio y acceden a BD con el patrón actual (adapter / service / DTOs).
2. **Preparados para escalar** — si un dominio crece, se extrae a microservicio independiente sin reescribir la lógica.
3. **Módulos transversales como auditoría** capaces de suscribirse a eventos sin que los dominios sepan de ellos.
4. **Gateway HTTP como capa delgada** — el framework NestJS queda solo en la capa de transporte.

Las 3 apps actuales (`auth-users`, `catalogs`, `cross`) coexisten en paralelo mientras se migran los dominios.

### Principio arquitectónico

> El código del dominio **no debe saber que existe NestJS**. NestJS queda como capa de transporte HTTP que inyecta los dominios como contratos.

---

## 2. Arquitectura Objetivo

```
pld-api/
├── apps/
│   ├── auth-users/                   ← gateway NestJS principal
│   │   └── src/catalogs/             ← datos estáticos + Cache-Control (Fase 2)
│   └── cross/                        ← gateway NestJS actual (se mantiene)
│   (apps/catalogs se elimina en Fase 2)
│
├── packages/                         ← Lógica de negocio SIN NestJS
│   ├── domain-auth-users/            (Fase 4 — aquí vive el password hashing)
│   ├── domain-audit/                 (Fase 3 — bloqueado, ver §11)
│   ├── domain-participants/          (Fase 4)
│   ├── contracts/                    ✅ creado — interfaces públicas + EventPublisher
│   ├── shared-types/                 ✅ creado — scalars + enums + person types
│   ├── shared-errors/                ✅ creado — DomainError classes
│   └── persistence/                  ✅ creado — DataSource TypeORM + mysqlConfigFromEnv
│
└── libs/                             ← Se vacían gradualmente. Eliminar en Fase 5.
```

### Estructura interna de un paquete de dominio

```
packages/domain-users/
├── src/
│   ├── adapters/             (UsersAdapter — mapping + validación)
│   ├── services/             (UsersService — lógica de negocio pura)
│   ├── repositories/         (UsersRepository sobre DataSource)
│   ├── entities/             (TypeORM entities — no requieren NestJS)
│   ├── dtos/                 (class-validator, sin @ApiProperty)
│   ├── types/
│   └── index.ts              (factory pública: createUsersDomain(deps))
├── README.md
└── package.json
```

---

## 3. La Regla Clave: Frontera Extraíble

Cada paquete de dominio debe cumplir **tres condiciones** para poder extraerse a microservicio sin reescribir lógica:

### 3.1 Contrato explícito

Lo único que el gateway conoce del dominio es su contrato — una interfaz TypeScript.

```typescript
// packages/contracts/src/catalogs.contract.ts
export interface CatalogsContract {
  listBeneficiario(): Promise<BeneficiarioLabel[]>;
}
```

Hoy el contrato lo implementa una clase local. Mañana lo implementa un cliente HTTP. El controller no cambia.

### 3.2 Factory pura que recibe dependencias

```typescript
// packages/domain-catalogs/src/index.ts
export function createCatalogsDomain(deps: { ds: DataSource }): CatalogsContract {
  const repo = new CatalogsRepository(deps.ds);
  const service = new CatalogsService(repo);
  return new CatalogsAdapter(service);
}
```

Sin `@Injectable`, `@Module`, `@InjectRepository`. Solo TypeScript puro.

### 3.3 No importar otros dominios directamente

Si `domain-users` necesita catálogos, recibe el **contrato** por constructor, no la implementación.

---

## 4. Patrón de Adaptador (el que ya existe, refinado)

Mantenemos la estructura actual `controller → adapter → service`. El cambio: **adapter + service + repository viven en el paquete puro**; el controller queda en la app NestJS.

### Después de migrar (puro)

```typescript
// packages/domain-users/src/repositories/users.repository.ts
export class UsersRepository {
  constructor(private readonly ds: DataSource) {}
  async findById(id: string) { return this.ds.getRepository(UserEntity).findOne({ where: { id } }); }
}

// packages/domain-users/src/services/users.service.ts
export class UsersService {
  constructor(private readonly repo: UsersRepository) {}
  async create(dto: CreateUserInput) { /* lógica */ }
}

// packages/domain-users/src/adapters/users.adapter.ts
export class UsersAdapter implements UsersContract {
  constructor(private readonly service: UsersService) {}
  async createNotario(input: CreateUserNotarioInput) { /* mapping + validación */ }
}

// packages/domain-users/src/index.ts
export function createUsersDomain(deps: { ds: DataSource }): UsersContract {
  return new UsersAdapter(new UsersService(new UsersRepository(deps.ds)));
}
```

### Wire-up en la app NestJS

```typescript
// apps/auth-users/src/users/users.module.ts
@Module({
  providers: [
    {
      provide: USERS_CONTRACT,
      useFactory: (ds: DataSource) => createUsersDomain({ ds }),
      inject: [DataSource],
    },
  ],
  controllers: [UsersController],
})
export class UsersModule {}
```

---

## 5. El Día que Extraigas un Dominio

**Solo cambias el provider**. Todo lo demás queda igual.

```typescript
// apps/<gateway>/src/modules/<dominio>/<dominio>.module.ts (después de extraer)
@Module({
  providers: [
    {
      provide: CATALOGS_CONTRACT,
      useFactory: () => new CatalogsHttpClient({ baseUrl: process.env.CATALOGS_SVC_URL }),
    },
  ],
  controllers: [CatalogsController],
})
export class CatalogsModule {}
```

Y `apps/svc-catalogs/` reutiliza el mismo `packages/domain-catalogs`.

---

## 6. Auditoría y Módulos Transversales

Auditoría se desacopla vía **EventPublisher** inyectado por constructor. Hoy implementación local; mañana Redis Streams / NATS.

```typescript
export class UsersService {
  constructor(private repo: UsersRepository, private events: EventPublisher) {}

  async create(input: CreateUserInput) {
    const user = await this.repo.save(input);
    await this.events.publish({ type: 'user.created', payload: user });
    return user;
  }
}
```

---

## 7. Beneficios y Tradeoffs

### Qué da
- **Testing**: paquetes se testean sin levantar NestJS.
- **Reuso**: CLIs, workers y scripts importan paquetes directo.
- **Extracción futura**: cambiar un provider, sin tocar lógica.
- **Independencia de crecimiento**: cada paquete evoluciona su API interna.

### Qué NO da (y está bien)
- Escalado horizontal independiente hoy (todo corre en el mismo proceso hasta extraer).
- Aislamiento de fallos entre dominios hoy (OOM derriba el proceso).
- Deploy independiente hoy.

---

## 8. Progreso Actual — Estado Real

### ✅ Fase 0 — Fundaciones (completada)

- Migración npm → pnpm workspaces.
- `packages/shared-types` creado (scalars + enums + person types).
- `packages/shared-errors` creado (DomainError + subclases).
- `packages/persistence` creado (createDataSource + mysqlConfigFromEnv con `autoLoadEntities: true`).
- `packages/contracts` creado (DomainEvent + EventPublisher + NoopEventPublisher).
- Aliases `@pld-api/*` registrados en `tsconfig.base.json`.
- README por cada paquete.

### ✅ Pasos adicionales ejecutados

- **Scalars** migrados desde `libs/core/lib/core.types.ts` a `shared-types` (re-export mantenido).
- **Enums** (`UserRole`, `Nacionalidad`, `TipoMoneda`, etc.) migrados desde `libs/catalogs/lib/catalogs.types.ts` a `shared-types` (re-export mantenido).
- **Types de persona** (`Domicilio`, `Contacto`, `TipoIdentificacion`, `Identificacion`) migrados desde `libs/address-contact-id` a `packages/shared-types/src/person.ts` (re-export mantenido).
- **`libs/persistencia-sql` eliminada** completamente. Apps migradas a `@pld-api/persistence`.
- **SDD inicializado** con backend `openspec`. Skill registry en `pld/.atl/skill-registry.md` (a nivel workspace, no dentro de `pld-api/`).

---

## 9. Plan de Migración Pendiente

### Fase 2 — Consolidar `catalogs` en `auth-users` con cache HTTP

**Decisión arquitectónica**: `apps/catalogs` solo expone datos estáticos para poblar selects y checkboxes del frontend (35 líneas, sin BD). No justifica un proceso independiente. Se absorbe en `auth-users`.

**Objetivo**: el frontend sigue consumiendo `/pld-api/...catalogs/beneficiario` (y futuros catálogos), pero el servidor es uno solo y responde en <1 ms, con cache del lado del navegador de 24 h.

#### 2.1 Estrategia de datos y cache

- **Datos en memoria**: constantes TypeScript (`const CATALOGS = { ... } as const`). Fuente de verdad en código, alineada con enums de `@pld-api/shared-types`.
- **Cache HTTP**:
  - Sin cache server-side: los datos ya están en memoria, latencia sub-milisegundo.
  - Header `Cache-Control: public, max-age=86400, immutable` en cada endpoint. El navegador cachea 24 h sin revalidar y no vuelve a llamar al servidor hasta que expire.
  - Fuente de invalidación natural: redeploy = versión nueva del servidor = los clientes nuevos traen los datos actualizados al expirar TTL.

#### 2.2 Trabajo

- Crear módulo `catalogs/` dentro de `apps/auth-users`:
  - `catalogs.data.ts` — constantes (hoy: beneficiario; después: exponer enums de `shared-types` como labels).
  - `catalogs.controller.ts` — endpoints `GET` con `@Header('Cache-Control', ...)`.
  - `catalogs.module.ts`.
- Mantener ruta pública `/catalogs/beneficiario` para no romper el frontend (proxy/Traefik la rutea igual).
- Eliminar `apps/catalogs/` completa.
- Remover del `docker-compose.yml` y `docker-compose.dev.yml`.
- Actualizar README y scripts de `package.json` (`catalogs:debug`, etc.).

#### 2.3 Bloqueo documentado

Los vocabularios inconsistentes para "tipos de control de beneficiario" (ver §11.1) **no bloquean** la consolidación: los labels actuales se mueven tal cual. La unificación del enum canónico queda para Fase 6.

**Criterio de éxito**:
- `GET /pld-api/auth-users/catalogs/beneficiario` responde 200 con las 3 labels.
- Header `Cache-Control` presente en la respuesta.
- `apps/catalogs/` eliminada.
- Login HTTP 201 sin regresiones.

### Fase 3 — Dominio `audit` + EventPublisher — ⏸️ BLOQUEADA

**Bloqueo**: falta decisión del usuario sobre alcance de auditoría (A/B/C). Ver §11.

### Fase 4 — Migrar dominios de negocio

Orden sugerido (revisado):

1. **`domain-auth-users`** (recomendado arrancar aquí):
   - Incluye `domain-users` + `domain-auth` (hoy viven en la misma app).
   - Aquí vive el crypto (`hashPassword`, `verifyPassword`, `generateSecurePassword`).
   - Mover `UserEntity` desde `libs/users`.
   - Mover `JwtStrategy` / `JwtAuthGuard` al gateway; la lógica de firma/verificación al paquete.
   - **Riesgo medio**: es el primer paquete con BD real.

2. **`domain-participants`** (bloque atómico):
   - 5 sub-entidades con FKs cruzadas: fisica, moral, fideicomiso, anexo-7, beneficiario-controlador.
   - **No migrar por sub-entidad** (rompería FKs y el JSON `tiposControl`).
   - Migrar como un solo paquete con todas sus entities.

**Criterio de éxito por dominio**: lib vieja vacía (o puente), controller consume contrato, login sigue en HTTP 201.

### Fase 5 — Cleanup + Endurecimiento

- Eliminar `libs/` completo (los que hayan quedado vacíos o puente).
- Actualizar paths en `tsconfig.base.json`.
- Evaluar upgrade **Nx 14 → 22** (warning de peer deps detectado desde Fase 0).
- **Remover `accessToken` de Nx Cloud commiteado** en `nx.json:17` (secreto expuesto).
- Apagar apps viejas en `docker-compose.yml` si se decide consolidar.
- Actualizar README con la nueva estructura.
- **Setup de tests** mínimo: remover `passWithNoTests: true`, agregar al menos 1 test por dominio.

#### 5.1 Simplificar prefix global de rutas

Una vez terminada la reorganización, con una sola app sirviendo todos los dominios, el prefix global `/pld-api/auth-users` pierde sentido semántico (un endpoint de `catalogs` no debería vivir bajo `/auth-users/`).

**Acción**: cambiar `prefixUrl: '/pld-api/auth-users'` → `prefixUrl: '/pld-api'` en `apps/auth-users/src/configs/common.configs.ts`.

**Impacto en URLs públicas**:
- Antes: `POST /pld-api/auth-users/auth/login`
- Después: `POST /pld-api/auth/login`

**Restricción**: reconfirmar al llegar a Fase 5 qué endpoints consume el frontend y coordinar el cambio.

**Timing**: hacer **después** de consolidar todos los dominios (Fase 4 completa). Hacerlo antes implicaría comunicar al frontend 2 cambios de URL en lugar de uno.

#### 5.2 Renombrar `auth-users` → `gateway` (opcional, evaluar timing)

Una vez que `auth-users` haya absorbido `catalogs` (Fase 2), consuma `domain-auth-users` + `domain-audit` + `domain-participants` (Fases 3-4) y deje de ser exclusivo de autenticación/usuarios, su rol pasa a ser **API gateway del sistema**. Renombrarla para reflejarlo.

**Alcance**:
- Directorio: `apps/auth-users/` → `apps/gateway/`.
- Config Nx (`project.json`) y scripts de `package.json` (`auth-users:debug` → `gateway:debug`, etc.).
- Docker images/services: `pld-api-dev-auth-users` → `pld-api-dev-gateway`.
- Variables de entorno y documentación (README, docker-compose).

**Restricción crítica — compatibilidad con frontend**:
- La **ruta pública HTTP NO cambia**. El frontend seguirá llamando a `POST /pld-api/auth-users/auth/login` sin modificaciones.
- El `globalPrefix` del NestJS (`apps/gateway/src/configs/common.configs.ts`) se mantiene como `/pld-api/auth-users`.
- Solo cambia el **nombre del proyecto, directorio y container**, no el contrato HTTP.

**Criterio de éxito**:
- `curl http://localhost:9001/pld-api/auth-users/auth/login` sigue funcionando (HTTP 201).
- `docker compose -f docker-compose.dev.yml up` levanta `pld-api-dev-gateway`.
- Frontend no requiere ningún cambio.

**Por qué va en Fase 5 y no antes**: renombrar un directorio toca muchos paths (imports, configs, CI, Docker). Conviene hacerlo cuando la estructura interna ya esté estable, no en medio de migraciones. Si se hace antes, cada fase siguiente duplica conflictos de merge.

### Fase 6 — Resolución de Bloqueos (ver §11)

Se ataca después del resto. Requiere investigación adicional antes de tocar código.

---

## 10. Decisiones Resueltas

1. **Gestor**: ✅ pnpm workspaces (migrado). Nx 14 mantenido hasta Fase 5.
2. **DTOs y Swagger**: ✅ split confirmado. Paquetes puros con `class-validator`, apps con `@ApiProperty`.
3. **Base de datos**: ✅ una sola BD MySQL indefinidamente. No se separan schemas por ahora.
4. **Arranque**: ✅ Fase 0 + pasos sueltos ejecutados. Apps no unificadas.

---

## 11. Bloqueos y Revisiones Pendientes (Fase 6)

Todo lo siguiente requiere investigación o decisión **antes** de tocar código asociado.

### 11.1 Unificar 3 vocabularios para "tipos de control de beneficiario"

Después de consolidar `catalogs` en `auth-users` (Fase 2), los labels se mueven tal cual. Lo pendiente es unificar el vocabulario disperso:

- `apps/catalogs` (ahora en `auth-users`) expone keys `beneficiario-1/2/3`.
- `TipoControlEjercido` (DTOs de participantes): `impone`, `ejerce`, `dirige`.
- `PMBeneficiarioControlTipoControl` (`libs/users/users.types.ts`): `IMPONE_DECISIONES`, `DIRIGE_ADMINISTRACION`, `VOTA_25_PORCIENTO`.
- BD: JSON arrays con strings libres.

**Pendiente**: investigar cómo el frontend consume esos keys, unificar en un enum canónico (probablemente `TipoControlBeneficiario` en `shared-types`), decidir migración de datos en BD.

### 11.2 Auditoría — alcance A/B/C

Falta elegir entre:

- **A**: solo cambios de estado (create/update/delete).
- **B**: A + acciones de auth (login OK/KO, logout).
- **C**: B + lecturas sensibles (ver participante, generar reporte).

Define el diseño de `domain-audit` y el volumen esperado.

### 11.3 Scaffolding vacío de Nx

16 archivos (`*.service.ts` vacío + `*.module.ts` vacío) en 8 libs. Placeholders de Nx sin lógica. **Decisión del usuario**: mantenerlos como posibles placeholders futuros. Evaluar en Fase 4 por cada dominio migrado: si sigue vacío → eliminar; si se llenará → desarrollar.

### 11.4 Libs puente

- `libs/catalogs` — solo re-exporta enums desde `shared-types`. Consumidores: 6 archivos.
- `libs/address-contact-id` — solo re-exporta types desde `shared-types`. Consumidores: 1 archivo.

Evaluar en Fase 5 si se migran los imports directo a `@pld-api/shared-types` y se eliminan las libs puente.

### 11.5 Libs de código muerto (0 consumidores en apps)

`anexos`, `auth-profiles`, `operacion`, `risk`, `reports`. Contienen solo types/interfaces de dominio que se usarán cuando se migren `domain-operacion`, `domain-reports`, etc. **No tocar hasta Fase 4** correspondiente.

### 11.6 Bugs conocidos de `apps/auth-users` — diagnóstico pendiente de fix

Ver [docs/DIAGNOSTICO_SERVICIO_USUARIOS.md](DIAGNOSTICO_SERVICIO_USUARIOS.md):

- DTOs cruzados en `users.adapter.ts` (método `createInternoExterno` recibe tipo `Notario...` y viceversa).
- Regex de RFC con `&amp;` HTML-escapado.
- `forbidNonWhitelisted: true` sin `whitelist: true` (configuración contradictoria).
- Import relativo frágil de `JwtService` (debería usar `@pld-api/jwt`).
- `lastLoginAt` nunca se actualiza.

Corregir durante Fase 4 al migrar `domain-auth-users`.

### 11.7 Ausencia de tests

Jest configurado con `passWithNoTests: true` en todos los proyectos — oculta que no existen tests. Abordar en Fase 5.

---

## 12. Referencias de Código

| Componente actual | Destino |
|---|---|
| [pld-api/libs/core/services/password.function.ts](../pld-api/libs/core/src/services/password.function.ts) | `packages/domain-auth-users` (Fase 4) |
| [pld-api/libs/core/interceptors/http.error.interceptor.ts](../pld-api/libs/core/src/interceptors/http.error.interceptor.ts) | `apps/*/src/shared/` (NestJS, no va a paquete puro) |
| [pld-api/libs/users/](../pld-api/libs/users/) | `packages/domain-auth-users` (Fase 4) |
| [pld-api/libs/participants/](../pld-api/libs/participants/) | `packages/domain-participants` (Fase 4) |
| [pld-api/libs/jwt/](../pld-api/libs/jwt/) | Lógica pura a `domain-auth-users`; guard/strategy a `apps/*/src/shared/` |
| [pld-api/apps/auth-users/src/auth/](../pld-api/apps/auth-users/src/auth/) | Controller se queda; adapter/service → `domain-auth-users` |
| [pld-api/apps/auth-users/src/users/](../pld-api/apps/auth-users/src/users/) | Controller se queda; adapter/service → `domain-auth-users` |
| [pld-api/apps/catalogs/](../pld-api/apps/catalogs/) | Absorbido en `apps/auth-users/src/catalogs/` (Fase 2). La app se elimina. |
| [pld-api/apps/cross/](../pld-api/apps/cross/) | Controller se queda; adapter/service → `domain-participants` |

---

## 13. Referencia a artefactos vivos

- [.claude/CHECKLIST_MIGRACION.md](../.claude/CHECKLIST_MIGRACION.md) — checklist interno de tareas (gitignored).
- [openspec/config.yaml](../openspec/config.yaml) — reglas SDD del proyecto.
- [docs/DIAGNOSTICO_SERVICIO_USUARIOS.md](DIAGNOSTICO_SERVICIO_USUARIOS.md) — bugs pendientes en auth-users.
