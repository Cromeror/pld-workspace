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
  - [flujos/inicio-sesion.md](flujos/inicio-sesion.md) — inicio de sesión + recuperación de contraseña
  - [flujos/post-login.md](flujos/post-login.md) — shell autenticada: redirect por rol, menú de usuario, switch de workspace
  - [flujos/registro-sujeto-obligado.md](flujos/registro-sujeto-obligado.md) — sujetos obligados (Notario/Inmobiliaria, PF/PM)
  - [flujos/registro-auxiliar.md](flujos/registro-auxiliar.md) — auxiliares por sujeto obligado (smoke: `registro-auxiliar-smoke.md`)
  - [flujos/actividad-secundaria.md](flujos/actividad-secundaria.md) — segunda actividad vulnerable; registro independiente enlazado al principal
  - [flujos/disenos/](flujos/disenos/) — todos los diseños UI organizados por flujo
- Inconsistencias conocidas (pendiente resolver)
  - **MoralIdentificationStep `birthCountry` sobrante**: el campo existe en FE pero no aparece en diseño ni diagrama BE — confirmar si eliminar
- Redes futuras
  - _(agregar aquí nuevos flujos, features o integraciones)_
