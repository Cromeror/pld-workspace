# Tasks: Add auxiliary registration endpoint (BE)

## Phase 1: Infrastructure (schema + shared types)

- [x] 1.1 [BE] Crear `pld-api/packages/persistence/migrations/20260428000000-create-auxiliary-profile.sql` con `CREATE TABLE auxiliary_profile` (id CHAR(36) PK, parent_user_id CHAR(36) NOT NULL FK→users(id) ON DELETE RESTRICT, rfc VARCHAR(13) NOT NULL, address_id CHAR(36) NOT NULL FK→reporting_entity_address(id) ON DELETE RESTRICT, created_at, updated_at) + `CREATE INDEX idx_auxiliary_profile_parent (parent_user_id)`.
- [x] 1.2 [BE] Crear migration down `20260428000000-create-auxiliary-profile.down.sql` con `DROP TABLE IF EXISTS auxiliary_profile`.
- [x] 1.3 [BE] Verificar up/down en local: aplicar up, inspeccionar schema (`SHOW CREATE TABLE auxiliary_profile`), aplicar down, confirmar drop limpio.
- [x] 1.4 [BE] Agregar `AUXILIARY = 'AUXILIARY'` al enum `ProfileType` en `pld-api/packages/shared-types/src/enums.ts`.
- [x] 1.5 [BE] Reemplazar shape de `AuxiliaryRegistration` en `pld-api/libs/auth-profiles/src/lib/auth-profiles.types.ts` con la forma completa del DTO (firstName, paternalSurname, maternalSurname, rfc, phone, email, address{...}).

## Phase 2: Core Implementation (entity + DTOs + adapter + service + controller)

- [x] 2.1 [BE] Crear `apps/auth-users/src/registration/auxiliaries/entities/auxiliary-profile.entity.ts` siguiendo el patrón de `physical-person-profile.entity.ts` (uuid, parent_user_id, rfc, address_id, timestamps).
- [x] 2.2 [BE] Crear `apps/auth-users/src/registration/auxiliaries/dto/create-auxiliary.dto.ts` con `CreateAuxiliaryDto` (firstName, paternalSurname, maternalSurname, rfc, phone, email) + `AuxiliaryAddressDto` anidado (state, postalCode, municipality, neighborhood, street, exteriorNumber, interiorNumber?). Reusar regex RFC de `physical-identification.dto.ts`.
- [x] 2.3 [BE] Crear `apps/auth-users/src/registration/auxiliaries/dto/auxiliary-response.dto.ts` con `AuxiliaryRegistrationResponse { user: UserResponseDto; temporaryPassword: string }`.
- [x] 2.4 [BE] Crear `apps/auth-users/src/registration/auxiliaries/auxiliaries.adapter.ts` con métodos: `findUserById(id, qr?)`, `insertAddress(qr, dto, country='México')`, `insertAuxiliaryProfile(qr, parentUserId, rfc, addressId)`, `createUser(qr, payload)` con catch `ER_DUP_ENTRY` → throw `ConflictException`.
- [x] 2.5 [BE] Crear `apps/auth-users/src/registration/auxiliaries/auxiliaries.service.ts` con `createAuxiliary(dto, currentUser)`: validar `parent_user.role IN (NOTARY, REAL_ESTATE)` (else `ForbiddenException`), ejecutar `dataSource.transaction(qr => …)` con 3 INSERTs, `generateSecurePassword(16)` + `hashPassword` (importados de `@pld-api/domain-auth-users`), mapper de respuesta excluyendo `passwordHash`.
- [x] 2.6 [BE] Crear `apps/auth-users/src/registration/auxiliaries/auxiliaries.controller.ts` con `@Controller('registration/auxiliaries')`, `@UseGuards(JwtAuthGuard, RolesGuard)`, `@Roles(UserRole.NOTARY, UserRole.REAL_ESTATE)`, `@Post() @HttpCode(201)`, decorators `@ApiResponse` 201/400/401/403/409, `@CurrentUser()` injection.

## Phase 3: Wiring

- [x] 3.1 [BE] Crear `apps/auth-users/src/registration/auxiliaries/auxiliaries.module.ts` con `TypeOrmModule.forFeature([AuxiliaryProfileEntity, ReportingEntityAddressEntity, UserEntity])`, providers `[AuxiliariesService, AuxiliariesAdapter, RolesGuard]`, controllers `[AuxiliariesController]`.
- [x] 3.2 [BE] Registrar `AuxiliariesModule` en `apps/auth-users/src/auth-users.module.ts` (el módulo raíz se llama así, no `app.module.ts`).
- [x] 3.3 [BE] Inspección de logging: el único interceptor global (`HttpErrorInterceptor`) solo loguea errores, no body/response — sin riesgo de filtración. Sin acción requerida.

## Phase 4: Build & Verification (smoke manual)

- [x] 4.1 [BE] `pnpm exec nx run auth-users:lint` — sin errores/warnings nuevos respecto al baseline (3 errores pre-existentes en archivos no tocados).
- [x] 4.2 [BE] `pnpm exec nx run auth-users:build` — verde (tras `sudo rm -rf dist/apps/auth-users/` para destrabar permisos de docker build previo).
- [x] 4.3 [BE] `pnpm exec nx run cross:build` — verde.
- [x] 4.4 [BE] Migration up aplicada en BD local; schema confirmado vía `SHOW CREATE TABLE`.
- [x] 4.5 [BE] Smoke happy path → HTTP 201 con `{ user, temporaryPassword }`. Auxiliar creado: `gerson.aux01@example.mx` (luego limpiado).
- [x] 4.6 [BE] Smoke email duplicado → HTTP 409 `Email already registered`.
- [x] 4.7 [BE] Smoke JWT SUPERADMIN → HTTP 403 `Insufficient role`.
- [x] 4.8 [BE] Smoke sin Authorization → HTTP 401 `Unauthorized`.
- [x] 4.9 [BE] Smoke DTO sin `rfc` → HTTP 400 con array de mensajes del validator.
- [x] 4.10 [BE] Smoke login del auxiliar con `temporaryPassword` → HTTP 201 + JWT con `role: AUXILIARY`. `GET /auth/me` confirma `mustChangePassword: true`.
- [x] 4.11 [BE] Regresión `POST /admin/registration` con SUPERADMIN → HTTP 201 (PF/PM intacto).
- [x] 4.12 [BE] Regresión login NOTARY existente → HTTP 201.
