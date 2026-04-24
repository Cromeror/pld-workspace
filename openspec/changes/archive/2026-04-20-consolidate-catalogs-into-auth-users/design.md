# Design: Consolidar `catalogs` en `auth-users` con Cache HTTP

## Technical Approach

Mover el único endpoint de `apps/catalogs` (`GET /beneficiario` con 3 labels hardcoded) a un módulo `catalogs/` dentro de `apps/auth-users`. Agregar `@Header('Cache-Control', ...)` en el controller. El interceptor `HttpErrorInterceptor` se copia localmente a `apps/auth-users/src/shared/` y `apps/cross/src/shared/`, eliminando el import desde `@pld-api/core` (regla persistida en `.claude/DECISIONES_MIGRACION.md`).

## Architecture Decisions

| Decisión | Elegida | Alternativa descartada | Rationale |
|---|---|---|---|
| Cache del lado cliente | Header `Cache-Control` en controller | `@nestjs/cache-manager` en servidor | Los datos son `const` en código. Server-side cache añadiría dependencia para cero ganancia. Navegador cachea 24h sin llamar. |
| TTL | `max-age=86400, immutable` | 300s, 1h, sin max-age | Labels cambian solo en redeploy. `immutable` evita revalidación. 24h es balance razonable entre frescura y ahorro. |
| Datos estáticos | `const CATALOGS_BENEFICIARIO` en `catalogs.data.ts` | Método en adapter (como hoy) | Separar datos de lógica. Facilita futuras extensiones (más catálogos). Tree-shaking friendly. |
| Ruta del endpoint | `GET /catalogs/beneficiario` bajo prefix `/pld-api/auth-users` | Preservar prefix `/pld-api/catalogs` | Auth-users ya tiene `globalPrefix=/pld-api/auth-users`. Cambiarlo rompería todos los demás endpoints. Se comunica al frontend. |
| Interceptor local | Copia en cada gateway (`shared/`) | Mantener import desde `libs/core` | Decisión del usuario (§Regla 2 + histórico 2026-04-20). Alinea con el principio "interceptors NestJS viven en la capa de transporte del gateway". |
| Swagger tag | Nuevo tag `catalogs` | Sin tag (queda en default) | Facilita el descubrimiento en la UI de Swagger ya disponible en auth-users. |

## Data Flow

```
Cliente HTTP                Gateway auth-users (NestJS)
    │                                │
    │ 1. GET /catalogs/beneficiario  │
    ├───────────────────────────────►│
    │                                │ CatalogsController.beneficiario()
    │                                │   ↓
    │                                │ return CATALOGS_BENEFICIARIO   (const en memoria)
    │                                │   ↓
    │                                │ @Header decorator agrega Cache-Control
    │                                │
    │ 2. 200 + JSON                  │
    │    Cache-Control: ...          │
    ◄───────────────────────────────┤
    │                                │
    │ 3. Segunda llamada (≤24h)      │
    │   (navegador NO llama servidor, sirve desde disk/memory cache)
```

## File Changes

| File | Action | Description |
|---|---|---|
| `apps/auth-users/src/catalogs/catalogs.module.ts` | Create | Módulo NestJS que registra controller |
| `apps/auth-users/src/catalogs/catalogs.controller.ts` | Create | `GET /catalogs/beneficiario` con `@Header` |
| `apps/auth-users/src/catalogs/catalogs.data.ts` | Create | `export const CATALOGS_BENEFICIARIO = [...] as const` |
| `apps/auth-users/src/catalogs/catalogs.controller.spec.ts` | Create | Test unitario Jest — primer test del proyecto |
| `apps/auth-users/src/shared/http-error.interceptor.ts` | Create | Copia literal de `libs/core/.../http.error.interceptor.ts` |
| `apps/auth-users/src/auth-users.module.ts` | Modify | Importar `CatalogsModule`, cambiar import de `HttpErrorInterceptor` a ruta local |
| `apps/auth-users/src/main.ts` | Modify | Cambiar import de `HttpErrorInterceptor` a ruta local |
| `apps/cross/src/shared/http-error.interceptor.ts` | Create | Copia literal (simétrico) |
| `apps/cross/src/app.module.ts` | Modify | Import de `HttpErrorInterceptor` local |
| `apps/cross/src/main.ts` | Modify | Import de `HttpErrorInterceptor` local |
| `apps/catalogs/` | Delete | Directorio completo |
| `docker-compose.yml` | Modify | Remover servicio `catalogs` |
| `docker-compose.dev.yml` | Modify | Remover bloque comentado |
| `package.json` | Modify | Remover `catalogs:debug`, ajustar `build` (quitar `catalogs:build`) |
| `tsconfig.base.json` | Modify | (Verificar) quitar alias si apuntara a app |

## Interfaces / Contracts

```typescript
// catalogs.data.ts
export interface CatalogEntry { key: string; label: string; }
export const CATALOGS_BENEFICIARIO: readonly CatalogEntry[] = [
  { key: 'beneficiario-1', label: 'Ejerce(n) el voto respecto de más del 25 % del capital social.' },
  { key: 'beneficiario-2', label: 'Tiene(n) derecho a recibir más del 25 % de los beneficios.' },
  { key: 'beneficiario-3', label: 'Posee(n), directa o indirectamente, más del 25 % de los derechos de propiedad.' },
] as const;

// catalogs.controller.ts
@Controller('catalogs')
export class CatalogsController {
  @Get('beneficiario')
  @Header('Cache-Control', 'public, max-age=86400, immutable')
  beneficiario() { return CATALOGS_BENEFICIARIO; }
}
```

## Testing Strategy

| Layer | Qué se prueba | Cómo |
|---|---|---|
| Unit | `CatalogsController.beneficiario()` retorna el array esperado | Jest spec en `apps/auth-users/src/catalogs/catalogs.controller.spec.ts` |
| Smoke (manual) | Endpoint responde 200 + shape correcto + header presente | `curl -i http://localhost:9001/pld-api/auth-users/catalogs/beneficiario` |
| Regresión | Login sigue 201 | `curl -X POST .../auth/login` con credenciales válidas |
| Compilación | Ambas apps restantes compilan | `nx run auth-users:build --skip-nx-cache` y `nx run cross:build --skip-nx-cache` |
| Test suite | Jest corre sin errores | `nx run auth-users:test --skip-nx-cache` |
| Comparación pre/post | Labels idénticos | Guardar response antes del cambio, `diff` contra después |

**Primer test unitario del proyecto**: se incluye como siembra (`catalogs.controller.spec.ts`) para validar que:
1. `CATALOGS_BENEFICIARIO` tiene exactamente 3 entradas con las keys correctas.
2. `controller.beneficiario()` retorna ese mismo array.

Es un test trivial pero establece la infraestructura de testing, reusable por futuros módulos. El proyecto ya tiene Jest funcional (verificado con `nx run auth-users:test` = exit 0 con `passWithNoTests: true`).

## Migration / Rollout

No hay migración de datos (todo estático). Rollout:

1. Rama dedicada (sugerida: `feat/consolidate-catalogs`).
2. Aplicar cambios, verificar manualmente con curl.
3. No requiere coordinación con frontend (confirmado por usuario: aún no consume el endpoint).
4. Merge → redeploy. La nueva URL disponible para cuando el frontend la integre: `http://<host>:9001/pld-api/auth-users/catalogs/beneficiario`.

Rollback: `git revert` + `docker compose up` restaura `apps/catalogs`.

## Open Questions

Ninguna. Respuestas del usuario:
- Test unitario: SÍ (Jest verificado funcional, se agrega como siembra).
- Frontend aún no consume `catalogs`: cambio de URL base sin impacto externo.
