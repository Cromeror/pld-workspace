# Proposal: auxiliar-registro-workspace-admin

## Problema

Cuando un auxiliar se registra en la plataforma, debe quedar asociado automáticamente al workspace del usuario WORKSPACE_ADMIN que está logueado en ese momento. La asociación se resuelve usando la información del token JWT del usuario autenticado al momento del registro, sin que el auxiliar deba seleccionar o indicar el workspace manualmente.

Contexto adicional:

1. La creación de auxiliares está EXCLUSIVAMENTE permitida para usuarios con rol WORKSPACE_ADMIN. Ni el SUPERADMIN ni ningún otro rol pueden crear auxiliares ni asignarlos a un workspace.

2. El formulario de registro de auxiliar debe incluir un campo 'Nombre de usuario' (nickname) que será el identificador con el que se creará la cuenta del auxiliar al finalizar el registro. No se usará el email como identificador de login para estos usuarios — el nickname es el único identificador de acceso.

## Estado

pending_dev_review
