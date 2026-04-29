# Proposal: Add auxiliary registration wizard (FE)

## Intent

Implementar el wizard de 3 pasos en `pld-web` para que un usuario `NOTARY` / `REAL_ESTATE` pueda dar de alta auxiliares asociados a su cuenta. Las capturas en [docs/designs/auxiliary-registration/](../../../docs/designs/auxiliary-registration/) son la fuente de verdad de campos, copys y comportamiento. Consume el endpoint del SDD paralelo `add-auxiliary-registration-be` (`POST /registration/auxiliaries`).

Sub-repo afectado: **pld-web** únicamente (con type mirror del DTO BE).

## Scope

### In Scope
- Ruta nueva `/auxiliary/register` (path final a confirmar) protegida por guard `role IN (NOTARY, REAL_ESTATE)`.
- Wizard 3 pasos con stepper lateral:
  1. Tipo de usuario (`Auxiliar` | `Clientes`). Solo `Auxiliar` avanza.
  2. Datos del usuario (nombre, apellidos, RFC, teléfono, email, entidad, CP, localidad/municipio, colonia, calle, no. ext, no. int opcional).
  3. Revisión + modal "¿Estás seguro/a?" antes del POST.
- Modales de resultado: éxito (password copiable + finalizar) y error (cancelar / reintentar).
- Mirror del DTO BE en `pld-web/src/types/`.
- Validación visual reusando schemas Yup existentes del wizard de reporting entities.
- Estado del wizard en Zustand (no persistir entre sesiones); mutación con React Query.
- Placeholder "Próximamente" al elegir `Clientes` (botón Siguiente bloqueado, sin endpoint).

### Out of Scope
- Endpoint BE (cubierto en `add-auxiliary-registration-be`).
- Flujo real para `Clientes` (solo placeholder).
- Botón "Reenviar correo" funcional — visible pero deshabilitado hasta que exista servicio de email.
- Listado / edición / borrado de auxiliares.
- Login del auxiliar (ya existe).
- Cambios en la dashboard del notario/inmobiliaria más allá de un punto de entrada al wizard.

## Approach

Replicar la estructura del wizard de reporting-entity-registration ya implementado en `pld-web/src/features/`: un container con stepper, un componente por paso, hooks de validación con Yup + React Hook Form, store Zustand para el estado del wizard. Catálogos (entidades, municipios, colonias) se obtienen vía endpoints `/admin/catalogs/*` ya existentes (verificar en sdd-design). El submit dispara una mutation de React Query; success abre el modal de password copiable, error abre el modal genérico con retry.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `pld-web/src/features/auxiliary-registration/**` | New | Pages, steps, store, schemas. |
| `pld-web/src/services/auxiliaries.ts` | New | Cliente HTTP + tipos. |
| `pld-web/src/types/auxiliary.ts` | New | Mirror del DTO BE. |
| `pld-web/src/router/**` | Modified | Nueva ruta protegida. |
| Dashboard del notario/inmobiliaria | Modified | Punto de entrada al wizard (ubicación a definir en sdd-design). |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Drift de contrato con BE | Med | Mirror del DTO compartido vía `pld-api/packages/shared-types`; verificar en sdd-verify. |
| Catálogos no exponen los campos del form | Med | sdd-design valida endpoints disponibles antes de implementar. |
| Password expuesta en logs/network tab | Low | No loguear response; modal cierra al click "Finalizar"; advertir al usuario. |
| Confusión con la opción `Clientes` | Low | Placeholder claro "Próximamente"; bloquear "Siguiente"; copy revisado con diseño. |

## Rollback Plan

1. Revertir el PR (cambios FE-only, sin migraciones ni cambios de schema).
2. Si el feature flag se introduce, apagarlo en lugar de revertir.
3. No hay datos persistidos por el FE — la limpieza la hace el rollback del BE si fuera necesario.

## Dependencies

- **`add-auxiliary-registration-be`** debe estar mergeado (o accesible vía branch) para integrar contra el endpoint real.
- Endpoints de catálogos existentes (`/admin/catalogs/*`) deben exponer entidades, municipios y colonias.

## Success Criteria

- [ ] Usuario con role `NOTARY` o `REAL_ESTATE` puede entrar al wizard; otros roles obtienen 403/redirect.
- [ ] Wizard avanza y retrocede entre pasos preservando el estado.
- [ ] Paso 1 con `Clientes` muestra "Próximamente" y no avanza.
- [ ] Paso 2 valida campos requeridos visualmente y bloquea "Siguiente" si faltan.
- [ ] Paso 3 muestra exactamente lo capturado y permite editar por sección.
- [ ] Submit exitoso muestra modal con password copiable y al click "Finalizar" cierra y resetea wizard.
- [ ] Submit con error muestra modal genérico con "Intentar de nuevo".
- [ ] `yarn build` (tsc -b && vite build) verde, sin warnings nuevos.
