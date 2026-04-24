---
name: stack-up
description: Levanta el stack dev de pld. Por default arranca BE (mysql + auth-users dockerizados via pld-api/docker-compose.dev.yml) y Web (pld-web con `yarn dev` en background). Flags para levantar parcial.
---

# /stack-up

Arranca el stack de desarrollo del workspace pld.

## Qué hace

1. Valida que esté parado en el root del workspace (`/home/cristobal/work/pld/`).
2. Por default levanta BE + Web.
3. BE: `docker compose -f pld-api/docker-compose.dev.yml up -d [--build]`. Levanta `mysql` y `auth-users` según el compose. El servicio `cross` queda comentado en el compose (si lo necesitás, editá el YAML manualmente).
4. Web: ejecuta `yarn dev` dentro de `pld-web/` como proceso background. PID va a `pld/.stack/web.pid`, stdout+stderr a `pld/.stack/web.log`.
5. Si el proceso Web ya estaba corriendo (PID válido), no arranca otro — avisa y sigue.

## Uso

```bash
/stack-up              # BE + Web (default)
/stack-up --no-web     # solo BE
/stack-up --no-be      # solo Web
/stack-up --rebuild    # BE con --build (rebuild de imágenes), Web sin cambios
```

## Implementación

Ejecutar desde `/home/cristobal/work/pld/`:

```bash
set -euo pipefail

ROOT="$HOME/work/pld"
STACK_DIR="$ROOT/.stack"
WEB_PID="$STACK_DIR/web.pid"
WEB_LOG="$STACK_DIR/web.log"
BE_COMPOSE="$ROOT/pld-api/docker-compose.dev.yml"

# Args
WITH_BE=1
WITH_WEB=1
REBUILD=""
for arg in "$@"; do
  case "$arg" in
    --no-be)    WITH_BE=0 ;;
    --no-web)   WITH_WEB=0 ;;
    --rebuild)  REBUILD="--build" ;;
    *) echo "flag desconocida: $arg" >&2; exit 2 ;;
  esac
done

cd "$ROOT"
mkdir -p "$STACK_DIR"

# ─── BE ───────────────────────────────────────────────────────
if [ "$WITH_BE" = "1" ]; then
  if [ ! -f "$BE_COMPOSE" ]; then
    echo "ERROR: no existe $BE_COMPOSE" >&2
    exit 1
  fi
  echo "▶ BE: docker compose up -d $REBUILD"
  docker compose -f "$BE_COMPOSE" up -d $REBUILD
fi

# ─── Web ──────────────────────────────────────────────────────
if [ "$WITH_WEB" = "1" ]; then
  if [ -f "$WEB_PID" ] && kill -0 "$(cat "$WEB_PID")" 2>/dev/null; then
    echo "▶ Web: ya corriendo (PID $(cat "$WEB_PID")). Usá /stack-down o /stack-status."
  else
    rm -f "$WEB_PID"
    echo "▶ Web: arrancando yarn dev (log: $WEB_LOG)"
    cd "$ROOT/pld-web"
    nohup yarn dev >"$WEB_LOG" 2>&1 &
    echo $! > "$WEB_PID"
    cd "$ROOT"
    sleep 1
    if kill -0 "$(cat "$WEB_PID")" 2>/dev/null; then
      echo "  PID $(cat "$WEB_PID") — listo."
    else
      echo "  ERROR: el proceso murió. Ver $WEB_LOG"
      rm -f "$WEB_PID"
      exit 1
    fi
  fi
fi

echo ""
echo "✓ stack-up completo. Ver estado con /stack-status."
```

## Notas

- El compose del BE define `pld-api-dev-network` y los volumenes `pld-api-dev-mysql-data` / `pld-api-dev-mysql-config`. Persisten entre `up`/`down` (a menos que uses `docker compose down -v`).
- El servicio `auth-users` monta `pld-api/` como volumen → hot-reload de `npx nx serve` funciona sin rebuild.
- Vite corre en el puerto default (`5173`) salvo que `pld-web/vite.config.ts` lo cambie.
- BE expone auth-users en `http://localhost:9001/pld-api/auth-users/*` (ver `docs/architecture.md`).
- No crea bloques YAML para ambientes — por ahora solo existe `development` local.
