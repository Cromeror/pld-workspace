# Design: actualizar-wizard-registro-notarias-inmobiliarias

## 1. Decisiones de arquitectura

- **Cambio quirúrgico, no rediseño.** Sólo se ajustan validaciones (zod en FE, class-validator en BE) y la condicional de visibilidad del Paso 4. No se introducen nuevas entidades ni endpoints.
- **Single source of truth por capa.** FE valida con los schemas en `reporting-entity/schemas.ts`; BE valida con los DTOs en `admin/registration/dto/`. Ambas capas se actualizan en paralelo y deben quedar consistentes con el diseño Figma actualizado.
- **Compatibilidad backwards-only.** Todos los cambios de BE son *aflojamiento* de validaciones (de requerido a opcional) o agregado de campos opcionales nuevos. No se rompen payloads existentes.
- **Stepper como dato derivado.** La lista de pasos se calcula desde `state.profileType` en `ReportingEntityRegistrationPage`. Para Notario (siempre PF) y para Inmobiliaria PF, el Paso 4 se omite del array de pasos. No se duplica el wizard.
- **Nuevo campo `lugarNacimiento`** (PF, opcional) se agrega tanto a FE como a BE. En BE se mapea como columna nullable en `physical-person-profile.entity.ts`; la columna ya existe como `birthPlace` o requiere migración menor — se opta por agregar la propiedad opcional en DTO y entity sin migración bloqueante (TypeORM `synchronize` o migración aditiva trivial). Si la columna aún no existe, queda como sub-tarea en `tasks.md`.

## 2. FE — Cambios por archivo (pld-web)

### `src/components/organisms/reporting-entity/schemas.ts`
- `physicalIdentificationSchema`:
  - `firstName`, `paternalSurname`, `maternalSurname` siguen requeridos.
  - `birthDate`, `rfc`, `curp` → pasan a `.optional().or(z.literal(""))` manteniendo regex sólo cuando el valor está presente (usar `.refine` o `.regex` envuelto en optional).
  - Agregar `birthPlace: z.string().optional()` (alias FE `lugarNacimiento`).
- `contactSchema`:
  - `email` requerido (sin cambios).
  - `cellphone` → requerido con `.regex(/^\d{10}$/)`. Antes era opcional.
  - `phone` (numeroTelefono) → opcional (sin cambios).
  - `countryCode` (claveLada) → opcional (sin cambios).
- `vulnerableActivitySchema`:
  - `activity` requerido (sin cambios).
  - `startDate` ya opcional (sin cambios).
  - `address` → pasa a `addressSchema.partial().optional()` o se redefine como `addressSchema` con todos los campos `.optional()`. Se opta por un `addressOptionalSchema` paralelo para no romper otros usos de `addressSchema`.
  - `activityPerformedAtAddress` → pasa a opcional.

### `src/pages/admin/ReportingEntityRegistrationPage.tsx`
- Filtrar el step de `ComplianceResponsibleStep` cuando `state.profileType === ProfileType.INDIVIDUAL`. Reemplazar el `s.profileType !== ProfileType.LEGAL_ENTITY` actual (usado como `skip`) por exclusión total del paso en el array de steps, para que el stepper visual tampoco lo muestre.
- Ajustar el cómputo de índices y la navegación `next/back` para tolerar el array dinámico.

### `src/components/organisms/reporting-entity/PhysicalIdentificationStep/index.tsx`
- Agregar input `lugarNacimiento`.
- Quitar marcador "*" visual de `rfc`, `curp`, `fechaNacimiento`.

### `src/components/organisms/reporting-entity/ContactStep/index.tsx`
- Marcar `celular` como requerido (asterisco + `aria-required`).
- Quitar requerido visual de `numeroTelefono`.

### `src/components/organisms/reporting-entity/VulnerableActivityStep/index.tsx`
- Quitar asteriscos de domicilio y `activityPerformedAtAddress`.

## 3. BE — Cambios por archivo (pld-api)

### `apps/auth-users/src/admin/registration/dto/physical-identification.dto.ts`
- `birthDate`: `@IsDateString()` → envolver con `@IsOptional()`. Quitar `!`.
- `rfc`: agregar `@IsOptional()` antes de `@IsString()`; mantener `@Length(13,13)` y `@Matches` para cuando esté presente.
- `curp`: agregar `@IsOptional()`; mantener `@Length(18,18)` y `@Matches`.
- Agregar campo `birthPlace?: string` con `@ApiPropertyOptional`, `@IsOptional`, `@IsString`.

### `apps/auth-users/src/admin/registration/dto/contact.dto.ts`
- `cellphone`: quitar `@IsOptional()`; queda requerido con `@Matches(/^\d{10}$/)`. Cambiar `cellphone?: string` → `cellphone!: string` y `@ApiPropertyOptional` → `@ApiProperty`.
- `phone` (numeroTelefono): mantener `@IsOptional()`.

### `apps/auth-users/src/admin/registration/dto/vulnerable-activity.dto.ts`
- `address`: agregar `@IsOptional()` antes de `@ValidateNested()`. Cambiar `address!` → `address?`.
- `activityPerformedAtAddress`: agregar `@IsOptional()`; quitar `@IsNotEmpty()`. Cambiar a opcional.

### `apps/auth-users/src/admin/registration/entities/physical-person-profile.entity.ts`
- Asegurar que las columnas `birthDate`, `rfc`, `curp` son `nullable: true`. Si ya lo son, no hay cambio. Agregar columna `birthPlace` nullable si no existe (migración aditiva).

### `apps/auth-users/src/admin/registration/registration.adapter.ts` / `registration.service.ts`
- Manejar `birthPlace` y los campos ahora opcionales sin asumir presencia (null-safe).

## 4. Compatibilidad

- Todos los cambios de BE son *relajaciones* o *adiciones opcionales*: registros y payloads existentes siguen validando.
- `cellphone` pasa de opcional a requerido en BE: revisar si hay registros previos con `cellphone = NULL`. Como la columna sigue siendo nullable a nivel DB, no afecta datos históricos; sólo afecta nuevos POSTs al endpoint de contacto. Se acepta como ruptura controlada coherente con el rediseño.
- FE y BE deben deployarse juntos para evitar que un FE viejo envíe `cellphone` vacío contra un BE nuevo.

---

## Design Created
**Change**: actualizar-wizard-registro-notarias-inmobiliarias
### Decisions
- Ajuste quirúrgico de validaciones, sin entidades ni endpoints nuevos.
- FE valida con zod (`reporting-entity/schemas.ts`); BE con class-validator en `admin/registration/dto/`.
- Stepper deriva de `profileType`: el Paso 4 se excluye del array cuando es PF (Notario o Inmobiliaria PF).
- `lugarNacimiento` (`birthPlace`) se agrega opcional en FE/BE; migración aditiva en `physical-person-profile.entity.ts` si la columna no existe.
- Cambios BE son backwards-compatible salvo `cellphone` que pasa a requerido; FE+BE deployan juntos.
### Files to touch
- FE:
  - `pld-web/src/components/organisms/reporting-entity/schemas.ts`
  - `pld-web/src/pages/admin/ReportingEntityRegistrationPage.tsx`
  - `pld-web/src/components/organisms/reporting-entity/PhysicalIdentificationStep/index.tsx`
  - `pld-web/src/components/organisms/reporting-entity/ContactStep/index.tsx`
  - `pld-web/src/components/organisms/reporting-entity/VulnerableActivityStep/index.tsx`
- BE:
  - `pld-api/apps/auth-users/src/admin/registration/dto/physical-identification.dto.ts`
  - `pld-api/apps/auth-users/src/admin/registration/dto/contact.dto.ts`
  - `pld-api/apps/auth-users/src/admin/registration/dto/vulnerable-activity.dto.ts`
  - `pld-api/apps/auth-users/src/admin/registration/entities/physical-person-profile.entity.ts`
  - `pld-api/apps/auth-users/src/admin/registration/registration.adapter.ts` (null-safety)
