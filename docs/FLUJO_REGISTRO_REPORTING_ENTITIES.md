# Flujo de registro de sujetos obligados (SUPERADMIN)

> Diagrama de referencia para el registro de sujetos obligados por parte de un `SUPERADMIN`. Dos tipos de sujeto obligado (Notarías, Inmobiliarias) y dos tipos de perfil (Persona Física, Persona Moral).

## Contexto y restricciones

- **Tipos de usuario del sistema**: `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR` (más `USUARIO_INTERNO`, `USUARIO_EXTERNO` existentes en el enum). Notarías e Inmobiliarias son tipos de usuario; `SUPERADMIN` y `AUXILIAR` son los perfiles administrativos que operan el alta.
- **Tipo de perfil** (aplica al sujeto obligado): `PERSONA_FISICA` o `PERSONA_MORAL`.
- **Registro largo con checkpoints**: el flujo se compone de pasos discretos (identificación → contacto → actividad vulnerable → …). Cada paso termina con una validación y un `Guardar información` que persiste parcialmente y permite continuar. Debe ser posible **retomar un registro incompleto** (diseño a resolver — ver sección "Gap detectado").
- **Campos de entrada (cajas azules en el diagrama)**: datos que se capturan. Los marcados con `*` son **requeridos**. La validación visual es responsabilidad del front; la validación autoritativa vive en el back en el DTO de cada request.
- **Acciones grises** (p. ej. `Datos de contacto`): marcan el inicio de un paso. Al final de cada paso, el decisor `¿Los campos están completos?` aplica validaciones del front. Para back, esas validaciones son el `class-validator` del DTO de cada endpoint.
- **`Ingresa el dato faltante`**: acción solo del front — el botón "Guardar/Continuar" queda deshabilitado. En back no hay equivalente: simplemente rechaza con HTTP 400 si el DTO no valida.
- **`Guardar información`**: cierra el paso y habilita el siguiente. Para back, cada acción `Guardar información` se mapea a **un endpoint distinto** (`POST` o `PATCH` sobre un sub-recurso del registro).
- **Independencia entre perfiles**: los flujos de Persona Física y Persona Moral son **ramas independientes** (no se entrelazan) y tienen sus propias entidades de persistencia.
- ⚠️ **Importante**: dentro del flujo de Persona Moral, si aparece un nodo "Persona Física" hace referencia al **responsable del cumplimiento de la Ley**, NO al tipo de perfil. No confundirlos.
- 📝 **Leyendas del paso "Actividad vulnerable\*"** (*"Para el caso de Notarías deberá decir: FE PÚBLICA (SERVIDORES PÚBLICOS...)"* y *"En caso de inmobiliarias deberá decir: TRANSMISIÓN DE BIENES INMUEBLES"*): son **etiquetas / copy del front** — el nombre de la sección que se muestra al usuario según el tipo de sujeto obligado. **No son valores del dominio a persistir**. El backend recibe y valida únicamente el valor del enum `ActividadVulnerable` que le envía el front.

## Referencias de código

### Enums canónicos (`packages/shared-types/src/enums.ts`)

| Concepto | Enum / tipo | Archivo | Valores |
|---|---|---|---|
| Tipo de usuario | `UserRole` | [pld-api/packages/shared-types/src/enums.ts:1](../pld-api/packages/shared-types/src/enums.ts#L1) | `SUPERADMIN`, `NOTARIO`, `INMOBILIARIA`, `AUXILIAR`, `USUARIO_INTERNO`, `USUARIO_EXTERNO` |
| Tipo de perfil | `TipoPersonaParticipante` | [pld-api/packages/shared-types/src/enums.ts:10](../pld-api/packages/shared-types/src/enums.ts#L10) | `PERSONA_FISICA`, `PERSONA_MORAL`, `FIDEICOMISO` |
| Actividad vulnerable | `ActividadVulnerable` | [pld-api/packages/shared-types/src/enums.ts:26](../pld-api/packages/shared-types/src/enums.ts#L26) | `TRANSMISION_DERECHOS_REALES_INMUEBLES`, `PODERES_IRREVOCABLES`, `CONSTITUCION_SOCIEDADES`, `FUSION`, `ESCISION`, `AUMENTO_CAPITAL`, `DISMINUCION_CAPITAL`, `TRANSMISION_ACCIONES_PARTES`, `FIDEICOMISOS`, `MUTUOS_PRESTAMOS_CREDITOS` |

> Los mismos enums están re-exportados desde [pld-api/libs/catalogs/src/lib/catalogs.types.ts](../pld-api/libs/catalogs/src/lib/catalogs.types.ts) con nota de deprecation — preferir `@pld-api/shared-types`.

### Entidades TypeORM

**Persona Física** — [pld-api/libs/participants/src/lib/fisica/participants.entity.ts](../pld-api/libs/participants/src/lib/fisica/participants.entity.ts):
- `ParticipantPFBasicaEntity` (tabla `participantsFisicaBasico`) — identificación base.
- `ParticipantPFDomicilioEntity` (tabla `participantsFisicaDomicilio`) — domicilio.
- Otras: `ParticipantPFIdentificacionEntity`, `ParticipantPFDocumentoINMEntity`, `ParticipantPFDomicilioNacionalEntity`, `ParticipantPFDatosIdentificacionEntity`, `ParticipantPFBeneficiarioControladorEntity`, `ParticipantRepresentanteEntity`, `ParticipantRepresentanteDomicilioEntity`, `ParticipantRepresentanteIdentificacionEntity`.

**Persona Moral** — [pld-api/libs/participants/src/lib/moral/participants.entity.ts](../pld-api/libs/participants/src/lib/moral/participants.entity.ts):
- `ParticipantePMBasicoEntity` (tabla `moralAmbosBasico`) — identificación base. Campo local `tipoPersonaMoral: 'mexicana' | 'extranjera'`.
- `ParticipantePMDomicilioEntity` (tabla `moralAmbosDomicilio`) — domicilio.
- Otras: `ParticipantePMRepresentanteEntity`, `ParticipantePMIdentificacionRepresentanteEntity`, `ParticipantePMPEPEntity`, `ParticipantePMDocumentoMigratorioEntity`, `ParticipantePMDomicilioTerritorialNacionalEntity`, `ParticipantePMBeneficiarioControladorEntity`.

**Usuario** — [pld-api/packages/domain-auth-users/src/entities/user.entity.ts](../pld-api/packages/domain-auth-users/src/entities/user.entity.ts):
- Columna `role: varchar` (sin constraint enum en BD; TypeORM valida en runtime contra `UserRole`).

### Tipo `Domicilio` (interfaz, no entity embebida)

- [pld-api/packages/shared-types/src/person.ts:3](../pld-api/packages/shared-types/src/person.ts#L3) — campos: `calle`, `numeroExterior`, `numeroInterior?`, `colonia`, `municipioDemarcacion`, `ciudadPoblacion`, `entidadFederativaEstado`, `codigoPostal`, `pais`, `tipoVialidad?`, `localidad?`. Cada tabla de domicilio en persistencia tiene nombres de columna ligeramente distintos — uniformar al integrar.

## Gaps detectados durante el análisis

1. **No existe tracking de completitud de registro**. Ninguna entidad de participante tiene columnas `estado`, `pasoActual`, `completado`, `registrationStep` ni similares. Para soportar la restricción "es posible que quede algún registro sin completar" hay que **agregar estado de registro** — se resuelve en el change SDD asociado (`add-registro-sujetos-obligados`).
2. **`UserEntity.role` es `varchar`, no `enum` en BD**. La migración a `enum` nativo de MySQL reforzaría la validación — fuera de scope del registro, pero queda anotado.

## Change SDD asociado

El plan de implementación vive en [openspec/changes/add-registro-sujetos-obligados/](../openspec/changes/add-registro-sujetos-obligados/) (proposal, specs, design, tasks). Resumen: endpoints paso-a-paso bajo `/admin/registration/*` que persisten cada checkpoint del diagrama y permiten retomar registros incompletos.

## Schema de base de datos

El diagrama ER y las tablas que dan soporte al registro viven en [REGISTRO_SCHEMA.md](REGISTRO_SCHEMA.md). Resumen:
- Tablas principales por perfil: `physical_person_profile`, `moral_person_profile`.
- Tablas auxiliares reusables: `reporting_entity_address`, `contact`.
- Solo PM: `compliance_responsible`.
- Tablas de proceso: `registration`, `vulnerable_activity`.

## Diagrama

Diagrama solo del lado **BE** — el wizard de UI (capturas, copys, validación visual) vive en [docs/designs/reporting-entity-registration/](designs/reporting-entity-registration/). Cada caja morada representa un endpoint del módulo `/admin/registration/*` y persiste un sub-recurso del registro.

```mermaid
flowchart TB
    Start["POST /admin/registration<br/>(JWT SUPERADMIN)<br/>{ profileType, userRole, rfc }"] --> Auth{¿role permitido?}
    Auth -->|No| E403[/"403 Forbidden"/]
    Auth -->|Sí| ValStart{¿DTO válido?}
    ValStart -->|No| E400[/"400 Bad Request"/]
    ValStart -->|Sí| Dedupe{"¿RFC sin draft<br/>IN_PROGRESS?"}
    Dedupe -->|No| E409[/"409 Conflict<br/>(id existente)"/]
    Dedupe -->|Sí| InsReg["INSERT registration<br/>(status=IN_PROGRESS,<br/>current_step=IDENTIFICATION,<br/>expires_at=NOW()+TTL)"]
    InsReg --> R201Draft[/"201 Created<br/>{ id }"/]

    R201Draft --> Step1["POST /:id/identification"]
    Step1 --> S1Branch{profile_type}
    S1Branch -->|INDIVIDUAL| InsPF["INSERT physical_person_profile"]
    S1Branch -->|LEGAL_ENTITY| InsPM["INSERT moral_person_profile"]
    InsPF --> UpdReg1["UPDATE registration.physical_profile_id<br/>+ current_step=CONTACT"]
    InsPM --> UpdReg1b["UPDATE registration.moral_profile_id<br/>+ current_step=CONTACT"]
    UpdReg1 --> Step2
    UpdReg1b --> Step2

    Step2["POST /:id/contact (1..N)"] --> InsContact["INSERT contact<br/>(phone, email, cellphone)"]
    InsContact --> UpdReg2["UPDATE current_step=VULNERABLE_ACTIVITY"]
    UpdReg2 --> Step3

    Step3["POST /:id/vulnerable-activity"] --> InsAddr["INSERT reporting_entity_address"]
    InsAddr --> InsAct["INSERT vulnerable_activity"]
    InsAct --> UpdReg3{profile_type}
    UpdReg3 -->|INDIVIDUAL| UpdReg3PF["UPDATE current_step=COMPLETED-ish"]
    UpdReg3 -->|LEGAL_ENTITY| UpdReg3PM["UPDATE current_step=COMPLIANCE_RESPONSIBLE"]
    UpdReg3PF --> Finalize
    UpdReg3PM --> Step4

    Step4["POST /:id/compliance-responsible<br/>(solo PM)"] --> InsCR["INSERT compliance_responsible<br/>+ UPDATE moral_person_profile<br/>.compliance_responsible_id"]
    InsCR --> Finalize

    Finalize["POST /:id/finalize"] --> ValFin{¿pasos requeridos<br/>completos?}
    ValFin -->|No| E422[/"422 Unprocessable"/]
    ValFin -->|Sí| TX{{"Transacción"}}
    TX --> Pwd["Generar + hashear password"]
    Pwd --> InsUser["INSERT users<br/>(role, profile_type,<br/>profile_id,<br/>must_change_password=true)"]
    InsUser --> UpdRegFinal["UPDATE registration.user_id,<br/>status=COMPLETED,<br/>current_step=COMPLETED"]
    UpdRegFinal -.->|feature posterior| Mail["📧 Enviar email<br/>(password en claro)"]
    UpdRegFinal --> R201Final[/"201 Created<br/>{ user, temporaryPassword }"/]
    Mail -.-> R201Final

    classDef ok fill:#a8e6a3,stroke:#4fa84a,color:#000;
    classDef err fill:#ffb3b3,stroke:#cc0000,color:#000;
    classDef accion fill:#cbb7ff,stroke:#8c68ff,color:#000;
    classDef decision fill:#fff2cc,stroke:#d6b656,color:#000;
    classDef tx fill:#b8e0f5,stroke:#5aa9d6,color:#000;

    class Start,Step1,Step2,Step3,Step4,Finalize,InsReg,InsPF,InsPM,UpdReg1,UpdReg1b,InsContact,UpdReg2,InsAddr,InsAct,UpdReg3PF,UpdReg3PM,InsCR,Pwd,InsUser,UpdRegFinal,Mail accion;
    class Auth,ValStart,Dedupe,S1Branch,UpdReg3,ValFin decision;
    class TX tx;
    class R201Draft,R201Final ok;
    class E400,E403,E409,E422 err;
```

> ⚠️ Envío de correo con la password queda **fuera de scope** en este flujo: no hay servicio de email implementado. La password temporal se devuelve en el response del `POST /:id/finalize` y la UI la muestra en modal copiable. Cuando exista servicio de email, se sumará en una feature posterior.
