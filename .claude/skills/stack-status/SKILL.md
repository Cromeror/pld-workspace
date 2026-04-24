---
name: stack-status
description: Muestra estado del stack dev pld. BE via `docker compose ps`, Web via PID + puerto + healthcheck HTTP.
---

# /stack-status

Reporta el estado del stack de desarrollo.

## Qué hace

1. BE: `docker compose -f pld-api/docker-compose.dev.yml ps` — lista servicios con su estado (up/exit/healthy).
2. Web:
   - Lee `pld/.stack/web.pid`. Si existe y el proceso vive, muestra PID.
   - Chequea `http://localhost:5173` con `curl` → Vite responde si está listo.
3. BE healthcheck extra: `curl http://localhost:9001/pld-api/auth-users/docs` para confirmar que auth-users está sirviendo Swagger.

## Uso

```bash
/stack-status        # reporte completo
/stack-status --be   # solo BE
/stack-status --web  # solo Web
```

## Implementación

Ejecutar desde `/home/cristobal/work/pld/`:

```bash
set -euo pipefail

ROOT="$HOME/work/pld"
STACK_DIR="$ROOT/.stack"
WEB_PID="$STACK_DIR/web.pid"
BE_COMPOSE="$ROOT/pld-api/docker-compose.dev.yml"

SHOW_BE=1
SHOW_WEB=1
for arg in "$@"; do
  case "$arg" in
    --be)  SHOW_WEB=0 ;;
    --web) SHOW_BE=0 ;;
    *) echo "flag desconocida: $arg" >&2; exit 2 ;;
  esac
done

cd "$ROOT"

# ─── BE ───────────────────────────────────────────────────────
if [ "$SHOW_BE" = "1" ]; then
  echo "━━━ BE (pld-api dev) ━━━"
  if [ -f "$BE_COMPOSE" ]; then
    docker compose -f "$BE_COMPOSE" ps
  else
    echo "  ERROR: no existe $BE_COMPOSE"
  fi
  echo ""
  echo "  auth-users Swagger:"
  code="$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 http://localhost:9001/pld-api/auth-users/docs || echo 000)"
  case "$code" in
    200) echo "    ✓ HTTP 200 — http://localhost:9001/pld-api/auth-users/docs" ;;
    000) echo "    ✗ sin respuesta (servicio caído o arrancando)" ;;
    *)   echo "    ? HTTP $code" ;;
  esac
  echo ""
fi

# ─── Web ──────────────────────────────────────────────────────
if [ "$SHOW_WEB" = "1" ]; then
  echo "━━━ Web (pld-web) ━━━"
  if [ -f "$WEB_PID" ]; then
    PID="$(cat "$WEB_PID")"
    if kill -0 "$PID" 2>/dev/null; then
      echo "  Proceso: PID $PID (vivo)"
    else
      echo "  Proceso: PID $PID guardado pero NO EXISTE (pidfile stale)"
    fi
  else
    echo "  Proceso: no arrancado (sin pidfile)"
  fi
  code="$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 http://localhost:5173 || echo 000)"
  case "$code" in
    200) echo "  HTTP: ✓ 200 — http://localhost:5173" ;;
    000) echo "  HTTP: ✗ sin respuesta" ;;
    *)   echo "  HTTP: ? $code" ;;
  esac
  echo ""
fi
```

## Notas

- El puerto de Vite asumido es `5173` (default). Si lo cambiás en `pld-web/vite.config.ts`, hay que ajustar esta skill y `stack-logs`.
- El timeout de `curl` es 3s — si el stack acaba de arrancar puede dar `000` unos segundos hasta que Vite y NestJS inicien.
