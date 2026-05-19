# Test users (dev local)

Usuarios creados para smoke / testing del stack local. **No usar en otros entornos.**

Todos viven en la DB de `pld-api-dev-mysql` (puerto `13306`, db `pld_api_bd`). Las contraseñas temporales se generaron en finalize del wizard (o seed manual) y NO rotan automáticamente.

Login en `http://localhost:4200/login` o vía API:

```bash
curl -X POST http://localhost:9001/pld-api/auth-users/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"<email>","password":"<password>"}'
```

## SUPERADMIN (seed manual)

| Email | Password | Notas |
|---|---|---|
| `admin-test@pld.local` | `Test1234!@` | Insertado directo en DB con hash scrypt. Usar para acceder al wizard `/admin/reporting-entity/register`. |

## Notario — Persona física (Rol: WORKSPACE_ADMIN · activityType: NOTARY · vía wizard 2026-04-27)

| Campo | Valor |
|---|---|
| Email | `pedro.notario@example.mx` |
| Password (temp) | `qNn7T0QHq6n4?0s&` |
| Nombre | Pedro Hernández Vargas |
| RFC | `HEVP800502ABC` |
| CURP | `HEVP800502HDFRGN01` |
| Fecha nac. | 1980-05-02 |
| Rol | WORKSPACE_ADMIN |
| activityType | NOTARY |
| Perfil | INDIVIDUAL |

## Notario — Persona física #2 (Rol: WORKSPACE_ADMIN · activityType: NOTARY · datos smoke pre-selección actividad vulnerable)

| Campo | Valor |
|---|---|
| RFC | `HEVP800502XYZ` |
| Nombre | Pedro |
| Apellido paterno | Hernández |
| Apellido materno | Vargas |
| Fecha nac. | 1980-05-02 |
| CURP | `HEVP800502HDFRGN02` |
| Email contacto | `pedro2.notario@example.mx` |
| Celular | `5550001234` |
| Actividad vulnerable | Fe pública (Corredores y Notarios) — pre-seleccionada por rol |

## Inmobiliaria — Persona moral (Rol: WORKSPACE_ADMIN · activityType: REAL_ESTATE · vía wizard 2026-04-27)

| Campo | Valor |
|---|---|
| Email | `contacto@inmobiliaria-test.mx` |
| Password (temp) | `sl*0$e!2*GSWVOUa` |
| Razón social | Inmobiliaria Test S.A. de C.V. |
| RFC | `ITE000101AB1` |
| Rol | WORKSPACE_ADMIN |
| activityType | REAL_ESTATE |
| Perfil | LEGAL_ENTITY |
| Responsable cumplimiento | María Hernández Soto (RFC `HESM800101ABC`, CURP `HESM800101MDFRSN01`) |

## Inmobiliaria — Persona moral (Rol: WORKSPACE_ADMIN · activityType: REAL_ESTATE · datos reales Métrica Inmobiliaria)

> Extraído de documento SHCP "Detalle de Alta Métrica". Usar solo en dev local.

**Step 1**

| Campo | Valor |
|---|---|
| Tipo de usuario | Inmobiliaria |
| Tipo de perfil | Persona moral |

**Step 2 — Identificación**

| Campo | Valor |
|---|---|
| Razón social | METRICA INMOBILIARIA |
| Fecha de constitución | 12/07/2006 |
| RFC | `MIN0607127R0` |
| País de nacionalidad | México |

**Step 3 — Contacto (2 contactos)**

| # | Email | Teléfono | Celular | Clave lada |
|---|---|---|---|---|
| 1 | `mballi@metricainmobiliaria.com` | 55067575 | 5555067575 | 55 |
| 2 | `rballi@metricainmobiliaria.com` | 54029142 | 5554029142 | 55 |

**Step 4 — Actividad vulnerable**

| Campo | Valor |
|---|---|
| Actividad | Transmisión de bienes inmuebles — pre-seleccionada por rol |
| Fecha inicial | 27/03/2024 |
| CP | 05100 |
| Entidad federativa | Ciudad de México (Distrito Federal) |
| Municipio | Cuajimalpa de Morelos |
| Localidad | Cuajimalpa de Morelos |
| Colonia | Lomas de Vista Hermosa |
| Tipo de vialidad | Avenida |
| Calle | Loma de Vista Hermosa |
| Núm. exterior | 149 |
| Núm. interior | 401 |
| Actividad en domicilio | Transmisión de bienes inmuebles |

**Step PM — Responsable de cumplimiento**

| Campo | Valor |
|---|---|
| Nombre(s) | Maria Angelica |
| Apellido paterno | Cervantes |
| Apellido materno | Vera |
| Fecha de nacimiento | 14/06/1976 |
| RFC | `CEVA760614BR1` |
| CURP | `CEVA760614MDFRRN08` |
| País de nacionalidad | México |
| Fecha de designación | 19/04/2024 |

## Cleanup

Para borrar todos estos test users:

```sql
DELETE FROM users WHERE email IN (
  'admin-test@pld.local',
  'pedro.notario@example.mx',
  'contacto@inmobiliaria-test.mx'
);
-- Para los registrations asociados, borrar primero por FK:
-- contact, physical_person_profile/moral_person_profile, vulnerable_activity,
-- reporting_entity_address, compliance_responsible, registration.
```

## Notas

- Los users creados via wizard tienen `must_change_password = true` (regla del BE: usuarios nuevos deben rotar la temp password en el primer login). Para tests rápidos, podés bypassearlo con UPDATE manual a `false`.
- `mustChangePassword` se expone en la respuesta de `GET /auth/me`, FE puede mostrar un toast/redirect a cambio de password (no implementado todavía).
- `lastLoginAt` se actualiza en cada login exitoso desde el change `add-current-user-endpoint`.

## Capturas del flow

Ver `.stack/screenshots/` (no commiteado, regenerable):
- `01-login.png` — pantalla de login
- `02-wizard-step1-closed.png` — wizard step 1 inicial
- `03-notario-only-individual.png` — Notario expandido (solo persona física)
- `04-inmobiliarias-both-options.png` — Inmobiliarias expandido (ambas opciones)
- `05-notario-creado.png` — modal de éxito tras crear notario
- `06-inmobiliaria-creada.png` — modal de éxito tras crear inmobiliaria
