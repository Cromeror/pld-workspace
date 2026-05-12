---
type: agent-reference
last-verified: 2026-04-28
affects: flujos/registro-auxiliar/
---

# Smoke test: registro de auxiliares (BE + FE)

> Checklist completo para validar end-to-end el flujo de registro de auxiliares. Ejecutar tras cualquier cambio que toque `pld-api/apps/auth-users/src/registration/auxiliaries/**`, `pld-web/src/pages/users/AuxiliaryRegistrationPage.tsx`, `pld-web/src/components/organisms/auxiliary/**` o las rutas relacionadas.

## Pre-requisitos

- [ ] **Stack arriba**: `/stack-up` (BE + Web).
- [ ] **BE ready**: `curl http://localhost:9001/pld-api/auth-users/docs` → HTTP 200.
- [ ] **Web ready**: `curl http://localhost:4200` → HTTP 200.
- [ ] **Migration aplicada**:
  ```bash
  docker compose -f pld-api/docker-compose.dev.yml exec -T mysql \
    mysql -uroot -psecret pld_api_bd \
    -e "SHOW CREATE TABLE auxiliary_profile\G"
  ```
  → debe mostrar la tabla con FKs `fk_auxiliary_profile_parent_user` y `fk_auxiliary_profile_address`.
- [ ] **Test users disponibles** (ver [test-users.md](test-users.md)):
  - NOTARY: `pedro.notario@example.mx` / `qNn7T0QHq6n4?0s&`
  - REAL_ESTATE: `contacto@inmobiliaria-test.mx` / `sl*0$e!2*GSWVOUa`
  - SUPERADMIN: `admin-test@pld.local` / `Test1234!@`

---

## Parte 1: Smokes BE (curl)

### 1.1 Login NOTARY → JWT

- [ ] ```bash
      JWT=$(curl -s -X POST http://localhost:9001/pld-api/auth-users/auth/login \
        -H 'Content-Type: application/json' \
        -d '{"email":"pedro.notario@example.mx","password":"qNn7T0QHq6n4?0s&"}' \
        | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
      echo "${#JWT}"  # debe imprimir ~256
      ```

### 1.2 Happy path → 201 con `{ user, temporaryPassword }`

- [ ] ```bash
      curl -s -w "\nHTTP %{http_code}\n" -X POST http://localhost:9001/pld-api/auth-users/registration/auxiliaries \
        -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
        -d '{
          "firstName":"Test","paternalSurname":"Smoke","maternalSurname":"User",
          "rfc":"TSMU010101A11","phone":"5550000000","email":"smoke.aux@example.mx",
          "address":{"state":"Morelos","postalCode":"62980","municipality":"Tlaquiltenango",
                     "neighborhood":"Centro","street":"Test","exteriorNumber":"1"}
        }' | python3 -m json.tool
      ```
  - [ ] HTTP 201
  - [ ] `user.role === "AUXILIARY"`, `user.profileType === "AUXILIARY"`, `user.mustChangePassword === true`
  - [ ] `temporaryPassword` viene en cleartext

### 1.3 Email duplicado → 409

- [ ] Repetir 1.2 con el mismo `email` → HTTP 409 `Email already registered`.

### 1.4 Role inválido → 403

- [ ] Login SUPERADMIN, repetir 1.2 con su JWT → HTTP 403 `Insufficient role`.

### 1.5 Sin JWT → 401

- [ ] Mismo POST sin header `Authorization` → HTTP 401.

### 1.6 DTO inválido → 400

- [ ] POST con `JWT` válido pero sin `rfc` → HTTP 400 con array de mensajes del validator.

### 1.7 Login del auxiliar recién creado

- [ ] Tomar el `temporaryPassword` del 1.2 y hacer login con `smoke.aux@example.mx` → HTTP 201 con JWT.
- [ ] `GET /auth/me` con ese JWT → `mustChangePassword: true`, `role: "AUXILIARY"`.

### 1.8 Regresión PF/PM (no rompimos `admin/registration`)

- [ ] Login SUPERADMIN, `POST /admin/registration` con `{profileType:"INDIVIDUAL", userRole:"NOTARY", rfc:"REGR111111X11"}` → HTTP 201.

### 1.9 Cleanup

- [ ] ```sql
      DELETE FROM auxiliary_profile WHERE rfc='TSMU010101A11';
      DELETE FROM users WHERE email='smoke.aux@example.mx';
      DELETE FROM reporting_entity_address WHERE postal_code='62980' AND street='Test';
      ```

---

## Parte 2: Smokes FE (browser, http://localhost:4200)

### 2.1 Build verde

- [ ] `cd pld-web && yarn tsc -b --noEmit` sin errores.
- [ ] `cd pld-web && yarn build` produce bundle sin warnings nuevos.

### 2.2 Acceso y redirect post-login

- [ ] Login con NOTARY (`pedro.notario@example.mx`) → debe redirigir a `/register` (NO a `/`).
  - **Nota**: en `/login`, tildar el checkbox de Aviso de Privacidad antes de hacer click; sin eso el botón no dispara la mutation.
- [ ] Login con REAL_ESTATE → redirect a `/register`.
- [ ] Login con SUPERADMIN → redirect a `/admin/reporting-entity/register` (no afectado).

### 2.3 Step 1 — Tipo de usuario (URL: `/register`)

- [ ] URL en barra: `/register`.
- [ ] **Breadcrumb**: solo "Crear usuario" (sin sufijo).
- [ ] Stepper lateral muestra: "Tipo de usuario / Datos del usuario / Revisión y validación".
- [ ] Botón "Siguiente" disabled.
- [ ] Click `Clientes` → aparece card "Próximamente"; "Siguiente" sigue disabled.
- [ ] Click `Auxiliar` → "Siguiente" habilita.
- [ ] Click "Siguiente" → URL cambia a `/register/auxiliary`.

### 2.4 Step 2 — Datos del usuario (URL: `/register/auxiliary`)

- [ ] **Breadcrumb**: "Crear usuario → Auxiliar".
- [ ] Form vacío, "Siguiente" disabled.
- [ ] Llenar: nombre, primer/segundo apellido, RFC válido, teléfono, email.
- [ ] Seleccionar entidad federativa (ej. Morelos) — el catálogo carga 32 estados.
- [ ] Capturar CP (5 dígitos).
- [ ] Seleccionar localidad/municipio (cascada habilitada tras elegir entidad).
- [ ] Seleccionar o escribir colonia (editable; permite input libre si el catálogo está vacío).
- [ ] Calle, no. exterior; no. interior opcional.
- [ ] "Siguiente" habilita cuando todos los requeridos son válidos.
- [ ] Validación inline: RFC mal formado → mensaje de error; CP no 5 dígitos → mensaje.

### 2.5 Step 3 — Revisión

- [ ] Render read-only en 2 secciones: "Usuario" (tipo) y "Datos personales" (resto).
- [ ] Cada sección con icono lápiz.
- [ ] Click lápiz "Datos personales" → vuelve a step 2 con datos preservados.
- [ ] Click "Regresar" desde step 2 → URL vuelve a `/register`, breadcrumb pierde el sufijo.
- [ ] Avanzar de nuevo → URL `/register/auxiliary`, breadcrumb "Crear usuario → Auxiliar".

### 2.6 ConfirmDialog

- [ ] Click "Confirmar" en step 3 → abre modal "¿Estás seguro/a?" con texto y botones Cancelar/Confirmar.
- [ ] Cancelar cierra el modal sin enviar.
- [ ] Confirmar dispara `POST /registration/auxiliaries`. Botón muestra loading + queda disabled (anti doble click).

### 2.7 Modal de éxito

- [ ] HTTP 201 abre `SuccessFinalizeModal`:
  - [ ] Icono check verde.
  - [ ] Título "¡Usuario Creado con Éxito!".
  - [ ] Email del nuevo auxiliar visible.
  - [ ] Password con toggle Mostrar/Ocultar.
  - [ ] Botón "Copiar contraseña" → al click, label cambia a "Copiado" 2s y la password queda en clipboard.
  - [ ] Botón "Reenviar correo" disabled con tooltip "Próximamente".
  - [ ] Botón "Finalizar" → cierra modal, resetea wizard al step 1, navega a `/`.

### 2.8 Modal de error (genérico)

- [ ] Bajar BE: `docker compose -f pld-api/docker-compose.dev.yml stop auth-users`.
- [ ] Volver a llenar y confirmar → `ErrorFinalizeModal` con título "Algo salió mal" + botones Cancelar/Intentar de nuevo.
- [ ] Subir BE: `docker compose -f pld-api/docker-compose.dev.yml start auth-users`; esperar 5s.
- [ ] "Intentar de nuevo" → re-disparar el POST → SuccessFinalizeModal.
- [ ] "Cancelar" cierra el modal y deja al usuario en step 3 con Confirmar habilitado.

### 2.9 Modal de error 409 (email duplicado)

- [ ] Repetir el flujo con un email que ya exista en `users` → la mutation retorna 409.
- [ ] El comportamiento esperado (per spec INV-14): cerrar ConfirmDialog + mostrar mensaje específico de email duplicado.

### 2.10 Refresh resetea wizard (no persistencia)

- [ ] En cualquier step, F5.
- [ ] Si estabas en `/register` → arranca en step 1 limpio.
- [ ] Si estabas en `/register/auxiliary` → arranca en step 1 con `Auxiliar` autocompletado y "Siguiente" habilitado.

### 2.11 URL directa a `/register/auxiliary`

- [ ] Pegar `http://localhost:4200/register/auxiliary` en barra (logueado como NOTARY).
- [ ] Wizard arranca en step 1 con `Auxiliar` ya seleccionado.
- [ ] Click "Siguiente" → step 2.

### 2.12 Acceso denegado

- [ ] Logueado como SUPERADMIN, navegar a `/register` → redirect a `/`.
- [ ] Logueado como SUPERADMIN, navegar a `/register/auxiliary` → redirect a `/`.
- [ ] Sin sesión, navegar a `/register` → redirect a `/login`.

### 2.13 Cleanup post-test

- [ ] ```sql
      DELETE FROM auxiliary_profile WHERE rfc IN (lista de RFCs usados);
      DELETE FROM users WHERE email LIKE 'smoke%' OR email LIKE 'test%';
      DELETE FROM reporting_entity_address WHERE created_at > <hora-de-inicio>;
      ```

---

## Parte 3: Regresión

### 3.1 Login y rutas existentes

- [ ] Login SUPERADMIN → redirige a `/admin/reporting-entity/register`.
- [ ] Wizard de PF/PM funciona end-to-end (smoke abreviado).

### 3.2 Builds del monorepo

- [ ] `cd pld-api && pnpm exec nx run auth-users:build` verde.
- [ ] `cd pld-web && yarn build` verde.

---

## Resumen final

- [ ] Todos los items anteriores marcados como ✅.
- [ ] No quedaron filas de prueba en BD (verificar con `SELECT COUNT(*) FROM users WHERE role='AUXILIARY'`).
- [ ] Logs de auth-users no contienen passwords en cleartext: `docker compose -f pld-api/docker-compose.dev.yml logs auth-users | grep -iE 'password|temppassword'` → vacío o solo metadata.
