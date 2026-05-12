---
type: agent-entrypoint
instruction: Read this file first at the start of any session involving pld-web UI, flows, or documentation. Navigate to the linked resources as needed — do not scan folders.
---

# PLD — Context Hub

- [arquitecturas/architecture.md](arquitecturas/architecture.md) — stack, puertos, contratos API→Web
- [arquitecturas/despliegue.md](arquitecturas/despliegue.md) — variables de entorno, Dockerfile prod, política .env
- [decisiones/](decisiones/) — historial de decisiones técnicas (ADRs, para LLM)
- [test-users.md](test-users.md) — credenciales y datos válidos por flujo
- [component-library.toon](component-library.toon) — componentes UI disponibles, props, cuándo usar cada uno
- [flujos/README.md](flujos/README.md) — índice de todos los flujos (BE + diseños UI)
  - [flujos/inicio-sesion/](flujos/inicio-sesion/) — inicio de sesión + recuperación de contraseña
  - [flujos/recuperacion-contrasena/](flujos/recuperacion-contrasena/) — diseños del subflujo de recuperación
  - [flujos/registro-sujeto-obligado/](flujos/registro-sujeto-obligado/) — sujetos obligados (Notario/Inmobiliaria, PF/PM)
    - [flujos/registro-sujeto-obligado/disenos/README.md](flujos/registro-sujeto-obligado/disenos/README.md) — mapa step→diseño→componente FE
  - [flujos/registro-auxiliar/](flujos/registro-auxiliar/) — auxiliares por sujeto obligado (smoke: `smoke.md`)
- Inconsistencias conocidas (pendiente resolver)
  - **Wizard PM falta Paso 5**: `ComplianceResponsibleStep` existe pero no está conectado al wizard (debe tener 6 pasos, no 5)
  - **MoralIdentificationStep `birthCountry` sobrante**: el campo existe en FE pero no aparece en diseño ni diagrama BE — confirmar si eliminar
- Redes futuras
  - _(agregar aquí nuevos flujos, features o integraciones)_
