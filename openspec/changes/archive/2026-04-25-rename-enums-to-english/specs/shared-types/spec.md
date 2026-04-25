# Delta for shared-types (pld-api)

## MODIFIED Requirements

### Requirement: Exports de enums en packages/shared-types

`pld-api/packages/shared-types/src/enums.ts` MUST exportar exactamente 8 enums bajo sus nombres English. Los nombres Spanish previos MUST NOT existir como exports del paquete.

| Nombre anterior | Nombre nuevo | Valores nuevos (English) |
|----------------|-------------|--------------------------|
| `UserRole` (valores Spanish) | `UserRole` | `SUPERADMIN`, `NOTARY`, `REAL_ESTATE`, `AUXILIARY` |
| `TipoPersonaParticipante` | `ProfileType` | `INDIVIDUAL`, `LEGAL_ENTITY`, `TRUST` |
| `ActividadVulnerable` | `VulnerableActivity` | valores English correspondientes |
| `TipoMoneda` | `CurrencyType` | valores English correspondientes |
| `FormaPago` | `PaymentMethod` | valores English correspondientes |
| `TipoReporte` | `ReportType` | valores English correspondientes |
| `ModalidadAtencion` | `AttendanceMode` | valores English correspondientes |
| `Nacionalidad` | `Nationality` | `MEXICAN`, `FOREIGN` |

#### INV-6: Exports con nombres English solamente

**Given** el archivo `packages/shared-types/src/enums.ts` fue modificado,
**When** se compila el paquete y se inspeccionan sus exports,
**Then** SHALL exportar `UserRole`, `ProfileType`, `VulnerableActivity`, `CurrencyType`, `PaymentMethod`, `ReportType`, `AttendanceMode`, `Nationality`,
**AND** SHALL NOT exportar `TipoPersonaParticipante`, `ActividadVulnerable`, `TipoMoneda`, `FormaPago`, `TipoReporte`, `ModalidadAtencion` ni `Nacionalidad`.

#### INV-7: Valores de enum en English

**Given** los enums están exportados con nombres English,
**When** se consultan sus valores en runtime,
**Then** cada enum SHALL contener únicamente los valores English especificados en la tabla de la propuesta,
**AND** ningún valor Spanish (`'NOTARIO'`, `'PERSONA_FISICA'`, `'EFECTIVO'`, etc.) SHALL existir en ningún enum.

### Requirement: Consumers internos usan solo nombres nuevos

Todos los consumers BE (`apps/auth-users`, `apps/cross`, `libs/auth-profiles`, `libs/operacion`, `libs/reports`) MUST importar únicamente los nombres English del paquete `shared-types`.

#### INV-8: Consumers actualizados — imports con nombres English

**Given** la refactorización está completa,
**When** se compila `nx run auth-users:build` y `nx run cross:build`,
**Then** ambos comandos SHALL completar con código de salida 0,
**AND** ningún consumer SHALL referenciar los nombres Spanish anteriores.

#### INV-9: Grep de nombres Spanish retorna cero matches

**Given** la refactorización está completa,
**When** se ejecuta `grep -rn "TipoPersonaParticipante|ActividadVulnerable|TipoMoneda|FormaPago|TipoReporte|ModalidadAtencion|Nacionalidad" pld-api/{apps,libs,packages}/*/src`,
**Then** el comando SHALL retornar cero matches.

#### INV-10: Grep de valores Spanish retorna cero matches (excl. migrations)

**Given** la refactorización está completa,
**When** se ejecuta `grep -rn "'NOTARIO'|'INMOBILIARIA'|'AUXILIAR'|'PERSONA_FISICA'|'PERSONA_MORAL'|'FIDEICOMISO'|'EFECTIVO'|'TRANSFERENCIA'|'CHEQUE'|'TARJETA'|'MEXICANA'|'EXTRANJERA'" pld-api/{apps,libs,packages}/*/src` excluyendo archivos de migraciones SQL,
**Then** el comando SHALL retornar cero matches.

#### INV-11: Build limpio post-rename

**Given** todos los renames y cascadas están aplicados,
**When** se ejecuta `nx run auth-users:build` y `nx run cross:build`,
**Then** ambos SHALL completar con código de salida 0 y cero errores de compilación TypeScript.
