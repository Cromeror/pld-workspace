---
name: stack-logs
description: Muestra logs del stack dev pld. BE via `docker compose logs`, Web desde `.stack/web.log`. Flags para filtrar por servicio y seguir en tiempo real.
---

# /stack-logs

Muestra logs del stack de desarrollo.

## Qué hace

1. BE: `docker compose -f pld-api/docker-compose.dev.yml logs [-f] [--tail N] [<servicio>]`.
2. Web: `tail [-f] [-n N] pld/.stack/web.log`.
3. Por default: últimas 100 líneas de BE + Web, sin follow.

## Uso

```bash
/stack-logs                    # últimas 100 líneas de BE + Web
/stack-logs -f                 # follow de ambos (BE en background, Web en foreground)
/stack-logs --be               # solo BE
/stack-logs --web              # solo Web
/stack-logs --be mysql         # solo el servicio mysql del BE
/stack-logs --be auth-users    # solo auth-users
/stack-logs --tail 500         # últimas 500 líneas
/stack-logs -f --web           # follow solo de Web
```

## Implementación

Ejecutar desde `/home/cristobal/work/pld/`:

```bash
set -euo pipefail

ROOT="$HOME/work/pld"
STACK_DIR="$ROOT/.stack"
WEB_LOG="$STACK_DIR/web.log"
BE_COMPOSE="$ROOT/pld-api/docker-compose.dev.yml"

SHOW_BE=1
SHOW_WEB=1
FOLLOW=""
TAIL_N="100"
BE_SERVICE=""

# Args: -f, --tail N, --be [svc], --web
while [ "$#" -gt 0 ]; do
  case "$1" in
    -f|--follow) FOLLOW="1" ;;
    --tail)      shift; TAIL_N="$1" ;;
    --be)
      SHOW_WEB=0
      # siguiente arg puede ser nombre de servicio o flag
      if [ "$#" -gt 1 ] && [[ ! "$2" =~ ^- ]]; then
        shift; BE_SERVICE="$1"
      fi
      ;;
    --web)       SHOW_BE=0 ;;
    *) echo "flag desconocida: $1" >&2; exit 2 ;;
  esac
  shift
done

cd "$ROOT"

# ─── Sin follow ───────────────────────────────────────────────
if [ -z "$FOLLOW" ]; then
  if [ "$SHOW_BE" = "1" ]; then
    echo "━━━ BE (últimas $TAIL_N líneas${BE_SERVICE:+, servicio=$BE_SERVICE}) ━━━"
    if [ -f "$BE_COMPOSE" ]; then
      docker compose -f "$BE_COMPOSE" logs --tail="$TAIL_N" $BE_SERVICE
    else
      echo "  ERROR: no existe $BE_COMPOSE"
    fi
    echo ""
  fi
  if [ "$SHOW_WEB" = "1" ]; then
    echo "━━━ Web (últimas $TAIL_N líneas) ━━━"
    if [ -f "$WEB_LOG" ]; then
      tail -n "$TAIL_N" "$WEB_LOG"
    else
      echo "  (sin log — Web no arrancado o recién iniciado)"
    fi
  fi
  exit 0
fi

# ─── Con follow ───────────────────────────────────────────────
# Si es BE-only o Web-only → follow directo foreground.
# Si ambos → BE en background, Web foreground; Ctrl-C mata ambos.
if [ "$SHOW_BE" = "1" ] && [ "$SHOW_WEB" = "0" ]; then
  exec docker compose -f "$BE_COMPOSE" logs -f --tail="$TAIL_N" $BE_SERVICE
fi
if [ "$SHOW_WEB" = "1" ] && [ "$SHOW_BE" = "0" ]; then
  if [ ! -f "$WEB_LOG" ]; then
    echo "  (sin log de Web — esperando a que se cree...)"
  fi
  exec tail -f -n "$TAIL_N" "$WEB_LOG"
fi

# Ambos en follow
echo "━━━ Follow BE + Web (Ctrl-C para salir) ━━━"
docker compose -f "$BE_COMPOSE" logs -f --tail="$TAIL_N" $BE_SERVICE &
BE_PID=$!
trap "kill $BE_PID 2>/dev/null || true" EXIT INT TERM
if [ -f "$WEB_LOG" ]; then
  tail -f -n "$TAIL_N" "$WEB_LOG"
else
  echo "(sin log de Web todavía)"
  wait "$BE_PID"
fi
```

## Notas

- El log de Web es `.stack/web.log` — se sobrescribe en cada `/stack-up` (por `nohup ... >log 2>&1`). Si querés acumular, cambiá a `>>` en stack-up.
- `docker compose logs` sin servicio muestra todos; con servicio (`mysql` o `auth-users`) filtra.
- En modo follow con ambos, `Ctrl-C` mata el tail de Web y via trap mata el `docker compose logs` de fondo.
