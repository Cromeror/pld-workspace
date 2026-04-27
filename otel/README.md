# Claude Code Telemetry — PLD

Stack local de OpenTelemetry para persistir y visualizar el consumo de
tokens de Claude Code en este workspace. Todo corre en `localhost`,
filtrado por el atributo de recurso `project=pld`.

## Arquitectura

```
Claude Code  ──OTLP/HTTP──▶  Prometheus 3.x  ◀──query──  Grafana 11.x
                            (127.0.0.1:9091)            (127.0.0.1:3002)
```

- **Prometheus** recibe las métricas vía OTLP nativo (`--web.enable-otlp-receiver`)
  y promueve `project`, `service.name` y `host.name` a labels para poder filtrar.
- **Grafana** se auto-provisiona al arranque: datasource + dashboard ya cargados,
  sin clicks.
- Ambos puertos bindeados a `127.0.0.1` — nada externo puede escribir ni leer.

> Los puertos elegidos (`9091` y `3002`) no chocan con los del stack
> equivalente en Cash Management (`9090` y `3001`), así que podés correr
> ambos a la vez.

## Prerequisitos

- Docker + Docker Compose v2
- `.claude/settings.local.json` con el bloque `env` (ver [Setup en otra máquina](#setup-en-otra-máquina))

## Comandos

Desde la raíz del workspace (`pld/`):

```bash
# Levantar
docker compose -f docker-compose.otel.yml up -d

# Ver estado
docker compose -f docker-compose.otel.yml ps

# Logs
docker compose -f docker-compose.otel.yml logs -f prometheus
docker compose -f docker-compose.otel.yml logs -f grafana

# Bajar (preserva datos)
docker compose -f docker-compose.otel.yml down

# Bajar y borrar TODO el histórico
docker compose -f docker-compose.otel.yml down -v
```

> Importante: después de levantar el stack hay que **reiniciar Claude Code**
> para que tome las env vars OTEL. Las métricas aparecen en el dashboard
> ~10 s después del primer turno.

## URLs y credenciales

| Servicio   | URL                          | Usuario | Password |
| ---------- | ---------------------------- | ------- | -------- |
| Prometheus | http://localhost:9091        | —       | —        |
| Grafana    | http://localhost:3002        | admin   | admin    |

Anonymous viewer está habilitado en Grafana, así que para sólo mirar el
dashboard no hace falta loguearse.

Dashboard pre-cargado: **PLD → Claude Code — PLD**.

## Setup en otra máquina

El compose y las configs de `otel/` están versionadas. Lo único que **no**
viaja por git es el bloque `env` de Claude Code (vive en
`.claude/settings.local.json`, gitignored).

En la máquina nueva, después del `git clone`:

1. Levantar el stack:
   ```bash
   docker compose -f docker-compose.otel.yml up -d
   ```
2. Agregar este bloque al inicio de `.claude/settings.local.json`
   (creando el archivo si no existe):
   ```json
   {
     "env": {
       "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
       "OTEL_METRICS_EXPORTER": "otlp",
       "OTEL_LOGS_EXPORTER": "none",
       "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
       "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:9091/api/v1/otlp",
       "OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE": "cumulative",
       "OTEL_RESOURCE_ATTRIBUTES": "project=pld",
       "OTEL_METRIC_EXPORT_INTERVAL": "10000"
     }
   }
   ```
3. Reiniciar Claude Code.

## Métricas exportadas

Claude Code emite estas métricas (tras pasar por el translator de Prometheus):

| Métrica                              | Tipo    | Descripción                                |
| ------------------------------------ | ------- | ------------------------------------------ |
| `claude_code_token_usage_tokens_total` | counter | Tokens consumidos (label `type`, `model`)  |
| `claude_code_cost_usage_USD_total`     | counter | Costo en USD (label `model`)               |
| `claude_code_session_count_total`      | counter | Sesiones iniciadas (label `start_type`)    |
| `claude_code_active_time_total_seconds_total` | counter | Segundos de actividad             |
| `claude_code_lines_of_code_count_total` | counter | Líneas de código modificadas (label `type`) |
| `claude_code_pull_request_count_total`  | counter | Pull requests creados                      |
| `claude_code_commit_count_total`        | counter | Commits creados                            |
| `claude_code_code_edit_tool_decision_total` | counter | Decisiones sobre tools de edición      |

Toda métrica lleva los labels estándar: `project`, `model`, `session_id`,
`user_email`, `terminal_type`, etc.

## Queries útiles (PromQL)

```promql
# Tokens consumidos hoy, separados por tipo
sum by (type) (increase(claude_code_token_usage_tokens_total{project="pld"}[24h]))

# Costo acumulado del último día
sum(increase(claude_code_cost_usage_USD_total{project="pld"}[24h]))

# Tasa de tokens por modelo (últimos 5 min)
sum by (model) (rate(claude_code_token_usage_tokens_total{project="pld"}[5m]))

# Sesiones iniciadas en la última semana
sum(increase(claude_code_session_count_total{project="pld"}[7d]))

# Costo proyectado mensual (basado en última hora)
sum(rate(claude_code_cost_usage_USD_total{project="pld"}[1h])) * 60 * 60 * 24 * 30
```

## Troubleshooting

**El dashboard está vacío**

1. ¿El stack está arriba? `docker compose -f docker-compose.otel.yml ps`
2. ¿Reiniciaste Claude Code después de levantar el stack?
3. ¿Las env vars están cargadas? Desde Claude Code corré:
   ```
   ! env | grep OTEL
   ```
   Tienen que aparecer `OTEL_EXPORTER_OTLP_ENDPOINT`, etc.
4. ¿Prometheus está recibiendo? Probá:
   ```bash
   curl -s http://localhost:9091/api/v1/query?query=claude_code_token_usage_tokens_total | jq .
   ```
   Si devuelve `data.result: []`, todavía no llegó nada — esperá 10–30 s
   después de mandar un mensaje y reintentá.

**Quiero borrar todo el histórico**

```bash
docker compose -f docker-compose.otel.yml down -v
```

El flag `-v` elimina los volúmenes `prom-data` y `grafana-data`.

**Cambiar la retención de Prometheus**

Editar `docker-compose.otel.yml`, flag `--storage.tsdb.retention.time` (default 90d).

## Datos y privacidad

- Los datos viven en Docker named volumes (`prom-data`, `grafana-data`),
  fuera del repo. Nada de telemetría se commitea jamás.
- `.gitignore` tiene entradas defensivas (`otel/**/data/`, `otel/**/*.db`)
  por si alguien cambia los volúmenes a bind mounts.
- Los servicios bindean en `127.0.0.1` — no son alcanzables desde la red.
- El bloque `env` con la config OTEL vive en `.claude/settings.local.json`
  (gitignored), no se sube nunca.
