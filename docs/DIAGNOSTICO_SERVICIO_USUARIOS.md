# Diagnóstico: Servicio de Usuarios (auth-users)

**Proyecto**: pld-api
**Fecha**: 2026-04-17
**Autor**: Cristobal Romero
**App analizada**: [pld-api/apps/auth-users](../pld-api/apps/auth-users)
**Objetivo**: ejecutar el servicio localmente y consumirlo desde un frontend.

---

## 1. Endpoints Disponibles

**Base URL**: `http://localhost:9001/pld-api/auth-users`

| Método | Ruta | Descripción | Auth |
|---|---|---|---|
| POST | `/auth/login` | Login con email+password → devuelve JWT | ❌ Pública |
| POST | `/system-users/notario-inmobiliario` | Crea usuario NOTARIO o INMOBILIARIA | ✅ JWT |
| POST | `/system-users/interno-externo-auxiliar` | Crea usuario AUXILIAR / USUARIO_INTERNO / USUARIO_EXTERNO | ✅ JWT |

**Swagger disponible en**: `http://localhost:9001/pld-api/auth-users/docs`

> Nota: el módulo también registra controllers de `participants-fisica`, `participants-moral`, `fideicomiso` y `anexo-7`, pero quedan fuera del alcance de este diagnóstico.

---

## 2. Cómo Ejecutar el Servicio

### Paso 1: Levantar MySQL
```bash
docker compose up -d mysql
```

### Paso 2: Verificar variables `.env`
Ya existe [.env](.env) con:
```
TYPEORM_HOST=127.0.0.1
TYPEORM_PORT=3306
TYPEORM_DB=pld_api_bd
TYPEORM_USER=root
TYPEORM_PASSWORD=secret
JWT_SECRET='holamundo'
JWT_EXPIRES_IN='1d'
```

### Paso 3: Instalar y levantar
```bash
npm install
npm run auth-users:debug   # modo watch + inspector
# o
npx nx serve auth-users
```

El servicio queda en `http://localhost:9001/pld-api/auth-users`.

---

## 3. Problemas Graves (Bloqueantes)

### 🔴 BLOQUEANTE 1 — No hay cómo crear el primer usuario
Ambos endpoints de creación están protegidos con `JwtAuthGuard`. Para obtener un JWT necesitas hacer login. Para hacer login necesitas un usuario. **No hay seed, ni endpoint público de registro, ni script de bootstrap.**

**Opciones de solución**:
- Crear un seed script que inserte un `SUPERADMIN` inicial.
- Insertar manualmente en BD con contraseña hasheada usando `hashPassword` de `@pld-api/core`.
- Agregar un flag temporal `@Public()` al primer endpoint de creación.

### 🔴 BLOQUEANTE 2 — No existen las tablas en BD
`autoLoadEntities: true` está activo pero **no hay `synchronize: true` ni migraciones ejecutables**. El archivo [query.sql](query.sql) contiene el DDL, pero nadie lo corre automáticamente.

**Arreglo rápido**:
- Activar `synchronize: true` en dev dentro de [persistencia-sql.config.ts](../pld-api/libs/persistencia-sql/src/lib/persistencia-sql.config.ts).
- O ejecutar `query.sql` manualmente al inicializar MySQL.
- Solución correcta a largo plazo: migraciones TypeORM versionadas.

### 🔴 BLOQUEANTE 3 — Inconsistencia en el schema SQL
[query.sql](query.sql) define el enum `role` **sin `USUARIO_EXTERNO`**, pero el enum de TypeScript en [pld-api/libs/catalogs/src/lib/catalogs.types.ts](../pld-api/libs/catalogs/src/lib/catalogs.types.ts) sí lo tiene. Si se intenta crear ese rol, MySQL rechaza el insert.

**Decisión pendiente**: agregar `USUARIO_EXTERNO` al SQL o eliminarlo del enum TS.

---

## 4. Bugs Funcionales

### 🟠 Nombres de métodos invertidos
En [users.adapter.ts](../pld-api/apps/auth-users/src/users/users.adapter.ts):
- `createInternoExterno` recibe `CreateUserNotarioInmobiliarioDto` (línea 14) — **tipo incorrecto**
- `createNotarioInmobiliario` recibe `CreateUserInternoExternoDto` (línea 64) — **tipo incorrecto**

Los métodos están **cruzados**. Funciona de casualidad porque las rutas HTTP llaman al método del nombre correcto pero con el DTO invertido. Efecto lateral: `rfc` (que solo existe en `CreateUserInternoExternoDto`) no se validará correctamente.

### 🟠 Error handling traga el caso de errores internos
En [users.adapter.ts:28-31](../pld-api/apps/auth-users/src/users/users.adapter.ts#L28-L31), el `try/catch` alrededor de la validación de email/teléfono convierte CUALQUIER error interno en `BadRequestException(['Error checking existing user'])`. Si MySQL está caído, el cliente ve ese mensaje en vez del error real, dificultando el debugging.

### 🟠 Password devuelto en texto plano en el response
[users.adapter.ts:56](../pld-api/apps/auth-users/src/users/users.adapter.ts#L56) retorna `{ ...safeUser, password: currentPassword }`. Es intencional (para que el admin entregue la contraseña al nuevo usuario) pero **debe documentarse explícitamente** y evaluarse si se cambia a envío por email.

### 🟠 Import con path relativo frágil
[auth.adapter.ts:5](../pld-api/apps/auth-users/src/auth/auth.adapter.ts#L5):
```typescript
import { JwtService } from '../../../../libs/jwt/src/lib/jwt.service';
```
Salta el alias `@pld-api/jwt`. Si se mueve el archivo, se rompe. Debe importar desde `@pld-api/jwt`.

### 🟠 Configuración contradictoria de ValidationPipe
[main.ts:42](../pld-api/apps/auth-users/src/main.ts#L42): `{ whitelist: false, forbidNonWhitelisted: true }`. `forbidNonWhitelisted` solo tiene efecto si `whitelist: true`. Propiedades extra en el body NO son rechazadas hoy.

### 🟠 Regex de RFC con HTML escapado
[create-user.dto.ts:62](../pld-api/apps/auth-users/src/users/dto/create-user.dto.ts#L62):
```typescript
@Matches(/^([A-ZÑ&amp;]{3,4})(...)/)
```
El `&amp;` debe ser `&`. El regex nunca matcheará RFCs con `&`.

---

## 5. Problemas Menores (calidad / completitud)

- No hay endpoints `GET`, `PATCH`, `DELETE` de usuarios — solo creación.
- [UsersService](../pld-api/libs/users/src/lib/users.service.ts) tiene `listUsers`, `updateUser`, `deleteUser` implementados pero **no están expuestos** en ningún controller.
- No hay endpoint `/auth/me` para validar token y obtener el usuario actual.
- No hay refresh token — solo access token de 1 día.
- CORS abierto (`origin: '*'`) — está bien para dev, restringir en producción.
- `lastLoginAt` está en la entity pero nunca se actualiza al hacer login.

---

## 6. Contratos para el Frontend

### POST `/auth/login`
**Request:**
```json
{ "email": "admin@pld.com", "password": "Secr3tP@ssw0rd" }
```
**Response 200:**
```json
{ "token": "eyJhbGci..." }
```
**Response 401:** `{ "statusCode": 401, "message": "Credenciales inválidas" }`

### POST `/system-users/notario-inmobiliario`
**Headers:** `Authorization: Bearer <token>`
**Request:**
```json
{
  "nombre": "Carlos",
  "apellidoPaterno": "Ramírez",
  "apellidoMaterno": "Gómez",
  "email": "carlos@notaria.mx",
  "telefono": "5544332211",
  "role": "NOTARIO"
}
```
**Response 201 (contiene password generada):**
```json
{
  "id": "uuid",
  "nombre": "Carlos",
  "email": "carlos@notaria.mx",
  "role": "NOTARIO",
  "activo": true,
  "password": "Xy9!kL2mN#pQ"
}
```

### POST `/system-users/interno-externo-auxiliar`
Mismo shape, pero `role` debe ser `AUXILIAR`, `USUARIO_INTERNO` o `USUARIO_EXTERNO`. Acepta además `rfc` opcional.

### Respuestas de error (global)
El `HttpErrorInterceptor` (en [pld-api/libs/core](../pld-api/libs/core/src/interceptors/http.error.interceptor.ts)) estandariza errores. El frontend debe manejar:
- `400` — validación de DTO
- `401` — credenciales inválidas o JWT inválido/expirado
- `409` — email o teléfono duplicado

---

## 7. Checklist Mínimo para Consumir desde Frontend

Orden de prioridad para que sea realmente usable:

- [ ] **1.** Arreglar inconsistencia del enum SQL — agregar `USUARIO_EXTERNO` a [query.sql](query.sql) o quitarlo del enum TS.
- [ ] **2.** Crear seed/bootstrap de un `SUPERADMIN` inicial (script o migración).
- [ ] **3.** Ejecutar el DDL de `query.sql` o activar `synchronize: true` en dev.
- [ ] **4.** Corregir los DTOs cruzados en [users.adapter.ts](../pld-api/apps/auth-users/src/users/users.adapter.ts).
- [ ] **5.** Agregar endpoint `GET /auth/me` — el frontend lo necesita para saber quién está logueado.
- [ ] **6.** Agregar endpoint `GET /system-users` — listar usuarios.
- [ ] **7.** Arreglar el regex de RFC (`&amp;` → `&`).
- [ ] **8.** Cambiar import relativo de `JwtService` a `@pld-api/jwt`.
- [ ] **9.** Activar `whitelist: true` en el ValidationPipe global.
- [ ] **10.** Documentar que el password en response es intencional o cambiar a envío por email.
- [ ] **11.** Actualizar `lastLoginAt` en el flujo de login.

---

## 8. Flujo Esperado desde Frontend

Una vez resueltos los bloqueantes:

1. `POST /auth/login` → guardar token (localStorage, sessionStorage o cookie httpOnly).
2. Incluir `Authorization: Bearer <token>` en cada request autenticado.
3. Al expirar (1 día), re-login (no hay refresh token actualmente).
4. Manejar `401` redirigiendo a login.
5. Usar `/auth/me` (cuando exista) para validar sesión y obtener rol/perfil.

---

## 9. Notas para Iterar

Este documento es la base para decidir el orden de arreglos. Puntos abiertos para revisar contigo:

- ¿Prefieres seed script o endpoint público temporal de bootstrap?
- ¿Qué endpoints adicionales (además de `GET /auth/me` y `GET /system-users`) son imprescindibles para el primer frontend?
- ¿El password debe seguir viajando en el response del POST de creación, o se migra a envío por email?
- ¿Se activa `synchronize: true` para dev, o se invierte tiempo en configurar migraciones TypeORM ya?
- ¿Todos estos arreglos entran en la app actual, o se aprovecha para ir migrando a la estructura nueva propuesta en [PLAN_REORGANIZACION_MONOREPO.md](docs/PLAN_REORGANIZACION_MONOREPO.md)?
