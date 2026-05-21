# Proposal: implementar-menu-switch-workspace-disenio

## Problema

Queremos implementar el diseño del menú de usuario para el switch de workspace en pld-web. Actualmente el menú tiene un item plano 'Cambiar a Notario/Inmobiliaria', pero el diseño pide:
1. Ítem 'Ver perfil' (actualmente ausente)
2. Submenú colapsable 'Seleccionar tipo de perfil' con chevron (actualmente es item plano directo)
3. Workspace activo visible con checkmark (actualmente no se muestra)
4. Ambas opciones visibles simultáneamente (NOTARY y REAL_ESTATE)
5. Label directo vs submenú anidado

Problema adicional: el usuario menciona que para WORKSPACE_ADMIN hay '3 actividades vulnerables', pero en el código ActivityType solo tiene NOTARY y REAL_ESTATE. El tercer caso podría ser AUXILIARY — un WORKSPACE_ADMIN podría tener registrations de tipo AUXILIARY además de NOTARY/REAL_ESTATE, y el endpoint GET /auth/me/workspaces devuelve TODAS las registrations COMPLETED sin filtrar por activityType. Hay que entender si esto puede devolver tipos no esperados en el menú.

Dudas que necesitamos resolver antes de implementar:
- ¿Qué significa '3 actividades vulnerables' para WORKSPACE_ADMIN? ¿Puede un usuario tener NOTARY + REAL_ESTATE + algo más?
- ¿El submenú debe ser colapsable por defecto o expandido?
- ¿'Ver perfil' debe navegar a alguna ruta existente?
- ¿El diseño del submenú es compatible con PrimeReact Menu component o necesitamos un componente custom?
- Si un usuario tiene ambos workspaces (NOTARY y REAL_ESTATE), ¿el actual lleva checkmark y el otro es clickeable, o hay otra interacción?

Queremos implementar el diseño del menú de usuario para el switch de workspace en pld-web. Actualmente el menú tiene un item plano 'Cambiar a Notario/Inmobiliaria', pero el diseño pide:

1. Ítem 'Ver perfil' (actualmente ausente)
2. Submenú colapsable 'Seleccionar tipo de perfil' con chevron (actualmente es item plano directo)
3. Workspace activo visible con checkmark (actualmente no se muestra)
4. Ambas opciones visibles simultáneamente (NOTARY y REAL_ESTATE)
5. Label directo vs submenú anidado

Contexto adicional de Figma:
- El archivo de diseño (tR7Y1QD0Ic0aY9OOpcJ9ww) tiene páginas: SUPER ADMIN, ADMIN (NOTARIA), ADMIN (INMOBILIARIA)
- El board de flujos (NPqPS0d6k9pH9Z5g5tyltV) tiene secciones 'Inicio de Sesión', 'Usuarios', etc. por workspace
- El flujo cambio-workspace.md ya documenta el flujo end-to-end

Dudas abiertas:
- ¿Qué significa '3 actividades vulnerables' para WORKSPACE_ADMIN? ¿Puede un usuario tener NOTARY + REAL_ESTATE + algo más?
- ¿El submenú debe ser colapsable por defecto o expandido?
- ¿'Ver perfil' debe navegar a alguna ruta existente?
- ¿El diseño del submenú es compatible con PrimeReact Menu component o necesitamos un componente custom?
- Si un usuario tiene ambos workspaces (NOTARY y REAL_ESTATE), ¿el actual lleva checkmark y el otro es clickeable, o hay otra interacción?

## Estado

pm_tech_review
