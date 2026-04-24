---
name: stack-down
description: Baja el stack dev de pld. Por default detiene BE (docker compose down del pld-api) y Web (mata el proceso Vite). Flags para acotar.
---

# /stack-down

Detiene el stack de desarrollo del workspace pld.

## Qué hace

1. Valida que esté parado en el root del workspace.
2. Por default baja BE + Web.
3. BE: `docker compose -f pld-api/docker-compose.dev.yml down [-v]`.
4. Web: lee `pld/.stack/web.pid`, hace `kill <pid>`, espera hasta 5s, si sigue vivo `kill -9`. Borra el pidfile.

## Uso

```bash
/stack-down              # baja BE + Web
/stack-down --no-web     # solo BE
/stack-down --no-be      # solo Web
/stack-down --volumes    # BE con -v (borra mysql data + mysql config)
```

⚠ `--volumes` borra la base de datos local. El schema de [pld-api/query.sql](../../../pld-api/query.sql) se recarga en el siguiente `/stack-up` pero los registros de usuarios/entidades que creaste a mano se pierden.

## Implementación

Ejecutar desde `/home/cristobal/work/pld/`:

```bash
set -euo pipefail

ROOT="$HOME/work/pld"
STACK_DIR="$ROOT/.stack"
WEB_PID="$STACK_DIR/web.pid"
BE_COMPOSE="$ROOT/pld-api/docker-compose.dev.yml"

WITH_BE=1
WITH_WEB=1
VOLUMES=""
for arg in "$@"; do
  case "$arg" in
    --no-be)    WITH_BE=0 ;;
    --no-web)   WITH_WEB=0 ;;
    --volumes)  VOLUMES="-v" ;;
    *) echo "flag desconocida: $arg" >&2; exit 2 ;;
  esac
done

cd "$ROOT"

# ─── Web ──────────────────────────────────────────────────────
if [ "$WITH_WEB" = "1" ]; then
  if [ -f "$WEB_PID" ]; then
    PID="$(cat "$WEB_PID")"
    if kill -0 "$PID" 2>/dev/null; then
      echo "▶ Web: kill $PID"
      kill "$PID" 2>/dev/null || true
      # esperá hasta 5s
      for i in 1 2 3 4 5; do
        kill -0 "$PID" 2>/dev/null || break
        sleep 1
      done
      if kill -0 "$PID" 2>/dev/null; then
        echo "  Sigue vivo, kill -9"
        kill -9 "$PID" 2>/dev/null || true
      fi
      # vite fork: matar hijos de yarn si quedaron
      pkill -P "$PID" 2>/dev/null || true
      pkill -f "vite" 2>/dev/null || true
    else
      echo "▶ Web: PID $PID ya no existe (proceso muerto). Limpiando."
    fi
    rm -f "$WEB_PID"
  else
    echo "▶ Web: no había PID guardado."
  fi
fi

# ─── BE ───────────────────────────────────────────────────────
if [ "$WITH_BE" = "1" ]; then
  if [ ! -f "$BE_COMPOSE" ]; then
    echo "ERROR: no existe $BE_COMPOSE" >&2
    exit 1
  fi
  echo "▶ BE: docker compose down $VOLUMES"
  docker compose -f "$BE_COMPOSE" down $VOLUMES
fi

echo ""
echo "✓ stack-down completo."
```

## Notas

- Si Vite se colgó y el `kill` no funciona, `pkill -f "vite"` captura cualquier proceso sobreviviente.
- `docker compose down` sin `-v` preserva los volúmenes de MySQL → los datos sobreviven al restart.
