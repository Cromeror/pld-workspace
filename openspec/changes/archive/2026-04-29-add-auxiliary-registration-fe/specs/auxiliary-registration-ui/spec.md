# Auxiliary Registration UI Specification

## Purpose

Wizard frontend (`/auxiliary/register`) que permite a un usuario `NOTARY` o `REAL_ESTATE` registrar usuarios `AUXILIARY` asociados a su cuenta. Wizard de 3 pasos con stepper visual, validación inline, y modales de confirmación + éxito + error con reintentos. Consume `POST /registration/auxiliaries` (BE definido en SDD paralelo).

Capturas de referencia: [docs/designs/auxiliary-registration/](../../../../docs/designs/auxiliary-registration/).

## Behavioral Invariants

### INV-1: Acceso restringido a NOTARY y REAL_ESTATE

**Given** un usuario autenticado con rol distinto de NOTARY o REAL_ESTATE,
**When** intenta navegar a `/auxiliary/register`,
**Then** el sistema SHALL redirigir a `/` sin renderizar el wizard.

**Given** un usuario sin sesión,
**When** intenta navegar a `/auxiliary/register`,
**Then** el sistema SHALL redirigir a `/login`.

### INV-2: Stepper visual de 3 pasos

**Given** el wizard renderizado,
**When** se inspecciona la barra lateral,
**Then** el stepper SHALL mostrar exactamente 3 pasos: "Tipo de usuario", "Datos del usuario", "Revisión y validación".

**Given** un paso completado,
**When** el wizard avanza al siguiente,
**Then** el paso completado SHALL mostrar un check verde,
**AND** el paso actual SHALL tener indicador visual distintivo.

### INV-3: Step 1 — selección obligatoria

**Given** step 1 sin selección,
**When** se renderiza,
**Then** el botón "Siguiente" SHALL estar disabled.

**Given** step 1 con `Auxiliar` seleccionado,
**When** el usuario clickea "Siguiente",
**Then** el wizard SHALL avanzar a step 2 sin hacer ningún request al BE.

### INV-4: Step 1 — opción Clientes muestra placeholder

**Given** step 1 con `Clientes` seleccionado,
**When** se evalúa el estado del wizard,
**Then** el sistema SHALL mostrar un placeholder "Próximamente",
**AND** el botón "Siguiente" SHALL permanecer disabled,
**AND** el sistema NO SHALL hacer requests al BE.

### INV-5: Persistencia de selección entre pasos

**Given** el usuario está en step 2,
**When** clickea "Regresar",
**Then** el wizard SHALL volver a step 1 preservando la selección previa.

**Given** el usuario está en step 3,
**When** clickea el icono lápiz de una sección,
**Then** el wizard SHALL volver al step correspondiente preservando todos los datos capturados.

### INV-6: Step 2 — campos requeridos

**Given** step 2,
**When** se renderiza el formulario,
**Then** los siguientes campos SHALL ser requeridos: nombre(s), primer apellido, segundo apellido, RFC, teléfono, email, entidad federativa, código postal, localidad/municipio, colonia, calle, número exterior.

**Given** step 2,
**When** el usuario inspecciona el campo "Número interior",
**Then** SHALL estar marcado como opcional.

### INV-7: Step 2 — validación inline bloquea avance

**Given** step 2 con cualquier requerido vacío o inválido (RFC mal formado, email inválido, CP no 5 dígitos),
**When** se evalúa el estado del botón "Siguiente",
**Then** SHALL estar disabled,
**AND** el campo inválido SHALL mostrar mensaje de error inline.

**Given** step 2 con todos los requeridos válidos,
**When** el usuario clickea "Siguiente",
**Then** el wizard SHALL avanzar a step 3 sin hacer request al BE.

### INV-8: Step 2 — catálogos en cascada

**Given** step 2 sin entidad federativa seleccionada,
**When** se renderiza el select de localidad/municipio,
**Then** SHALL estar disabled.

**Given** step 2 con entidad seleccionada,
**When** se renderiza el select de localidad/municipio,
**Then** SHALL cargar opciones filtradas por la entidad.

**Given** step 2 con CP capturado,
**When** se renderiza el select de colonia,
**Then** SHALL cargar opciones filtradas por el CP.

**Given** un catálogo no responde,
**When** el usuario intenta seleccionar,
**Then** el sistema SHALL mostrar mensaje de error inline en el select,
**AND** SHALL permitir reintento.

### INV-9: Step 3 — render read-only

**Given** step 3 con datos válidos del step 2,
**When** se renderiza,
**Then** el sistema SHALL mostrar todos los campos en read-only,
**AND** SHALL agruparlos en secciones "Usuario" y "Datos personales",
**AND** cada sección SHALL tener un icono lápiz para regresar al step correspondiente.

### INV-10: Confirmar abre modal antes de submit

**Given** step 3,
**When** el usuario clickea "Confirmar",
**Then** el sistema SHALL abrir un ConfirmDialog "¿Estás seguro/a?",
**AND** SHALL no ejecutar `POST /registration/auxiliaries` todavía.

**Given** ConfirmDialog abierto,
**When** el usuario clickea "Cancelar",
**Then** el sistema SHALL cerrar el modal,
**AND** el wizard SHALL permanecer en step 3 con "Confirmar" habilitado.

### INV-11: Submit con loading bloquea doble click

**Given** ConfirmDialog abierto,
**When** el usuario clickea "Confirmar",
**Then** el sistema SHALL ejecutar `POST /registration/auxiliaries`,
**AND** el botón "Confirmar" SHALL mostrar estado loading,
**AND** SHALL estar disabled durante la mutation.

### INV-12: Modal de éxito muestra password copiable

**Given** la mutation retorna 201,
**When** se abre el SuccessModal,
**Then** SHALL mostrar título "¡Usuario Creado con Éxito!",
**AND** SHALL mostrar el `email` del user creado,
**AND** SHALL mostrar `temporaryPassword` en input read-only,
**AND** SHALL incluir botón "Copiar contraseña",
**AND** SHALL incluir botón "Reenviar correo" disabled con tooltip "Próximamente",
**AND** SHALL incluir botón "Finalizar".

**Given** SuccessModal abierto,
**When** el usuario clickea "Copiar contraseña",
**Then** el sistema SHALL copiar al clipboard,
**AND** SHALL mostrar feedback visual ("Copiado" o toast).

**Given** SuccessModal abierto,
**When** el usuario clickea "Finalizar",
**Then** el sistema SHALL cerrar el modal,
**AND** SHALL resetear el wizard al step 1 con state vacío,
**AND** SHALL redirigir al dashboard del notario/inmobiliaria.

### INV-13: Modal de error con reintento

**Given** la mutation retorna 4xx/5xx,
**When** se abre el ErrorModal,
**Then** SHALL mostrar título "Algo salió mal",
**AND** SHALL mostrar mensaje "Ocurrió un problema al procesar tu solicitud. Por favor, verifica tu conexión e inténtalo de nuevo.",
**AND** SHALL incluir botones "Cancelar" e "Intentar de nuevo".

**Given** ErrorModal abierto,
**When** el usuario clickea "Intentar de nuevo",
**Then** el sistema SHALL re-disparar `POST /registration/auxiliaries` con el mismo payload,
**AND** SHALL no resetear el wizard.

**Given** ErrorModal abierto,
**When** el usuario clickea "Cancelar",
**Then** el sistema SHALL cerrar el modal,
**AND** el wizard SHALL permanecer en step 3 con "Confirmar" habilitado.

### INV-14: 409 email duplicado regresa a step 2

**Given** la mutation retorna 409 con mensaje de email duplicado,
**When** se procesa el error,
**Then** el sistema SHALL mostrar un alert/toast indicando email duplicado,
**AND** SHALL regresar al step 2 con el campo email marcado como inválido.

### INV-15: Estado del wizard sin persistencia

**Given** el usuario está en cualquier step con datos capturados,
**When** refresca la página,
**Then** el wizard SHALL resetear al step 1 sin avisos,
**AND** los datos previos NO SHALL persistir.

**Given** el usuario navega fuera del wizard sin completar,
**When** vuelve a `/auxiliary/register`,
**Then** el wizard SHALL arrancar en step 1 limpio,
**AND** NO SHALL existir draft persistido en BE ni localStorage.

### INV-16: Header de auth en cada request

**Given** el usuario dispara el submit,
**When** el cliente HTTP envía el POST,
**Then** el request SHALL incluir el header `Authorization: Bearer <jwt>` automáticamente vía interceptor existente.

### INV-17: Build limpio

**Given** la implementación es completa,
**When** se ejecuta `yarn tsc -b --noEmit`,
**Then** SHALL pasar con cero errores.

**When** se ejecuta `yarn build`,
**Then** SHALL producir un bundle de producción con cero errores.
