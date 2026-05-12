---
type: agent-reference
---

# Despliegue — PLD

## Contrato de entrega

- **`pld-api`** → imagen Docker publicada en Docker Hub (`zurcodesignio/pld-auth-users:<tag>`). Infra hace `docker pull` + `docker run` con sus propias env vars.
- **`pld-web`** → `.zip` con build estático (`vite build`). Infra lo despliega detrás de servidor estático / CDN.
- **Ningún secreto viaja en la imagen ni en el zip.** Todas las credenciales se inyectan por variables de entorno en el host.

## Variables de entorno — `pld-api`

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
| `MAIL_GATEWAY_URL` | requerida en prod | default `https://api-mails.zurco.com.mx` |
| `MAIL_GATEWAY_UID` | secreta | credencial gateway Zurco |
| `MAIL_GATEWAY_API_KEY` | secreta | credencial gateway Zurco |
| `MAIL_FROM` | requerida en prod | `Notificaciones <notificaciones@zurco.com.mx>` |
| `MAIL_REPLY_TO` | requerida en prod | `Soporte <support@zurco.com.mx>` |
| `MAIL_GATEWAY_TIMEOUT_MS` | opcional | default `10000` |
| `MAIL_DEV_OVERRIDE_TO` | dev-only | **prohibido en prod** |
| `MAIL_DEV_LOG_ONLY` | dev-only | **prohibido en prod** |
| `FRONTEND_LOGIN_URL` | requerida | URL absoluta del frontend para links de correo |

## Variables de entorno — `pld-web`

A definir. Variables `VITE_*` se incrustan en build-time. Si cambian entre ambientes hay que regenerar el zip. Alternativa: servir `/config.json` desde el host y leerlo al boot.

## Política de archivos `.env*`

```
.env.example    ← commiteado. Contrato de variables sin valores reales.
.env            ← gitignored. Defaults compartidos sin secretos fuertes.
.env.local      ← gitignored. Overrides personales con secretos locales.
.env.staging    ← NO en repo. Gestiona infra.
.env.production ← NO en repo. Gestiona infra.
```

⚠️ **Pendiente**: `pld-api/.env` está trackeado en git con `JWT_SECRET` y `TYPEORM_PASSWORD` históricos. Antes del primer push a Docker Hub:
1. `git rm --cached pld-api/.env`
2. Agregarlo a `pld-api/.gitignore`
3. Rotar los secretos comprometidos
4. Coordinar con el equipo

## Dockerfile prod — esqueleto

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

## .dockerignore recomendado

```
node_modules
dist
tmp
.nx
.git
.vscode
*.log
.env
.env.*
!.env.example
coverage
```

## Publicación

```bash
# desde pld-api/
docker build -t zurcodesignio/pld-auth-users:v1.0.0 -t zurcodesignio/pld-auth-users:latest .
docker push zurcodesignio/pld-auth-users:v1.0.0
docker push zurcodesignio/pld-auth-users:latest
```
