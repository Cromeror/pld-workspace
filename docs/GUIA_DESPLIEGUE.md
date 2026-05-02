# Guía de despliegue — PLD

> Documento vivo. Se completa a medida que se concretan ambientes (staging, prod) y se acuerdan convenciones con el equipo de infra.

## Resumen del contrato

- **`pld-api`** se entrega como **imagen Docker** publicada en Docker Hub (`zurcodesignio/pld-auth-users:<tag>`). El equipo de infra hace `docker pull` + `docker run` con sus propias env vars.
- **`pld-web`** se entrega como **`.zip`** con el build estático (resultado de `vite build`). El equipo de infra lo despliega detrás de un servidor estático / CDN.
- **Ningún secreto viaja en la imagen ni en el zip.** Todas las credenciales se inyectan por variables de entorno en el host de despliegue.

## Estado actual de los artefactos

| Recurso | Estado | Observaciones |
|---|---|---|
| `pld-api/Dockerfile.dev` | ✅ existe | Solo desarrollo local con `nx serve` |
| `pld-api/Dockerfile` (prod) | ❌ falta | Se debe crear antes de la primera publicación a Docker Hub |
| `pld-api/.dockerignore` | ⚠️ parcial | Excluye `.env.local` pero NO `.env`. Hay que ampliarlo |
| `pld-api/docker-compose.yml` | ✅ existe (dev) | Levanta MySQL + apps via `${VAR}` del `.env` del host |
| `pld-web/Dockerfile` | ❌ no aplica | El web se entrega como zip, no imagen |
| `pld-web/scripts de build` | ⚠️ verificar | Confirmar que `pnpm build` produce un `dist/` listo |

## Política de variables de entorno

### Principio

> El código entregado (imagen / zip) **debe poder cambiar de ambiente sin recompilar**. Toda variable que distinga dev de staging de prod va por env, NO se hardcodea.

### Variables actuales por servicio

#### `pld-api`

| Variable | Tipo | Notas |
|---|---|---|
| `NODE_ENV` | requerida | `production` activa fail-fast en validaciones |
| `TYPEORM_HOST` | requerida | host MySQL |
| `TYPEORM_PORT` | requerida | puerto MySQL |
| `TYPEORM_DB` | requerida | nombre de base |
| `TYPEORM_USER` | requerida | usuario MySQL |
| `TYPEORM_PASSWORD` | secreta | password MySQL |
| `JWT_SECRET` | secreta | secreto firma JWT — **rotar al deploy** |
| `JWT_EXPIRES_IN` | opcional | default `1d` |
| `MAIL_GATEWAY_URL` | requerida en prod | Default `https://api-mails.zurco.com.mx` |
| `MAIL_GATEWAY_UID` | secreta | credencial gateway Zurco |
| `MAIL_GATEWAY_API_KEY` | secreta | credencial gateway Zurco |
| `MAIL_FROM` | requerida en prod | `Notificaciones <notificaciones@zurco.com.mx>` u oficial |
| `MAIL_REPLY_TO` | requerida en prod | `Soporte <support@zurco.com.mx>` u oficial |
| `MAIL_GATEWAY_TIMEOUT_MS` | opcional | default `10000` |
| `MAIL_DEV_OVERRIDE_TO` | dev-only | **prohibido en prod** (Zod schema lo rechaza) |
| `MAIL_DEV_LOG_ONLY` | dev-only | **prohibido en prod** |
| `FRONTEND_LOGIN_URL` | requerida | URL absoluta del frontend para links de correo |

#### `pld-web`

> A definir en una iteración futura. El build de Vite empaqueta variables que comiencen con `VITE_*`. Las que dependen de ambiente deben inyectarse en build-time o resolverse en runtime via un `config.json` servido por el host.

## Política de archivos `.env*`

Para evitar que secretos se filtren en imágenes ni en git:

```
.env.example   ← commiteado. Contrato de qué variables existen, sin valores reales.
.env           ← gitignored. Defaults compartidos del equipo (sin secretos fuertes).
.env.local     ← gitignored. Overrides personales del developer (con secretos locales).
.env.staging   ← NO viven en el repo. Los gestiona el equipo de infra.
.env.production ← NO viven en el repo. Los gestiona el equipo de infra.
```

**Riesgo conocido (a cerrar antes de la primera publicación a Docker Hub)**:
el archivo `pld-api/.env` está actualmente trackeado en git con `JWT_SECRET` y `TYPEORM_PASSWORD` históricos. Acciones requeridas:

1. `git rm --cached pld-api/.env` (deja de trackearlo, queda local).
2. Agregarlo a `pld-api/.gitignore`.
3. **Rotar** los secretos comprometidos en cualquier ambiente que los use.
4. Coordinar con el equipo (otros devs van a tener que recrear su `.env` local).

## Build y publicación de `pld-api` a Docker Hub

> Procedimiento propuesto. A validar con el equipo de infra antes del primer push.

### Pre-requisitos

- `Dockerfile` de producción (multi-stage) creado y verificado.
- `.dockerignore` excluyendo todos los `.env*` (no solo `.env.local`).
- Tag de versión decidido (ej. `v1.0.0` + `latest`).

### `Dockerfile` propuesto (esqueleto)

```dockerfile
# build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN npx nx run auth-users:build --configuration=production --skip-nx-cache

# runtime stage
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist/apps/auth-users ./
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 9001
CMD ["node", "main.js"]
```

### `.dockerignore` recomendado

```
node_modules
dist
tmp
.nx
.git
.vscode
.idea
*.log
.env
.env.*
!.env.example
coverage
```

### Publicación

```bash
# desde pld-api/
docker build -t zurcodesignio/pld-auth-users:v1.0.0 -t zurcodesignio/pld-auth-users:latest .
docker push zurcodesignio/pld-auth-users:v1.0.0
docker push zurcodesignio/pld-auth-users:latest
```

### Despliegue por parte de infra

```bash
docker pull zurcodesignio/pld-auth-users:v1.0.0
docker run -d \
  --name pld-auth-users \
  --env-file /etc/pld/api.env \
  -p 9001:9001 \
  zurcodesignio/pld-auth-users:v1.0.0
```

El archivo `/etc/pld/api.env` lo gestiona infra (no se commitea ni viaja en la imagen).

## Build y entrega de `pld-web` (.zip)

> A completar cuando se acuerde el comando exacto y la estructura del zip.

### Procedimiento propuesto

```bash
# desde pld-web/
pnpm install --frozen-lockfile
pnpm build       # produce dist/
zip -r pld-web-v1.0.0.zip dist/
```

Entregar `pld-web-v1.0.0.zip` al equipo de infra. Ellos lo descomprimen detrás de un nginx / CDN.

### Variables del frontend

El build de Vite incrusta `VITE_*` en build-time. Si una variable cambia entre staging y prod, **hay que regenerar el zip**. Alternativa: servir un `/config.json` desde el mismo host y leerlo al boot del frontend (no requiere rebuild).

A decidir con infra.

## Checklist por ambiente

### Dev local

- [ ] `pld-api/.env.local` con credenciales personales del gateway.
- [ ] `pnpm install` en pld-api y pld-web.
- [ ] `docker compose up` para MySQL.
- [ ] `nx serve auth-users` en pld-api.
- [ ] `pnpm dev` en pld-web.

### Staging *(pendiente de definir)*

- [ ] Imagen pld-api publicada con tag `staging`.
- [ ] Variables de ambiente staging gestionadas por infra.
- [ ] URL del frontend de staging conocida (`FRONTEND_LOGIN_URL`).
- [ ] Gateway de mail con credenciales staging (¿son las mismas de prod?).

### Producción *(pendiente de definir)*

- [ ] Imagen pld-api publicada con tag versionado (`vX.Y.Z`).
- [ ] Variables de prod gestionadas por infra.
- [ ] Secretos rotados respecto al `.env` histórico.
- [ ] `MAIL_FROM` y `MAIL_REPLY_TO` con dominios oficiales (no `gmail.com`).
- [ ] DKIM/SPF/DMARC configurados con Zurco si el `MAIL_FROM` cambia de dominio.
- [ ] Backup de MySQL planificado.
- [ ] Logs centralizados (CloudWatch / Datadog / lo que use infra).

## Pendientes para completar esta guía

- [ ] Definir comando exacto de build de `pld-web` y estructura del zip.
- [ ] Crear `Dockerfile` de prod en `pld-api/` y verificar que la imagen levante.
- [ ] Confirmar credenciales del gateway de mail por ambiente.
- [ ] Acordar nomenclatura de tags Docker Hub (semver vs git sha).
- [ ] Documentar plan de rollback de imagen.
- [ ] Documentar política de migrations (TypeORM) en deploy.
- [ ] `MAIL_FROM`/`MAIL_REPLY_TO` oficiales de PLD.
- [ ] Decidir variables del web (build-time vs runtime config).
