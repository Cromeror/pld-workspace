# Flujos

Documentación de flujos de negocio del sistema PLD. Cada carpeta contiene la lógica BE y los diseños UI de un flujo.

## Cómo navegar

- `flujo.md` — diagrama BE en formato toon (fuente de verdad). Leer esto primero para entender endpoints, tablas y validaciones autoritativas.
- `disenos/` — capturas UI organizadas por step. Leer si trabajas en FE o en diseño de pantallas.
- Los flujos se relacionan entre sí — ver columna "Notas" para dependencias.

## Índice

| Flujo | Actores | Estado BE | Estado UI | Notas |
|---|---|---|---|---|
| [inicio-sesion.md](inicio-sesion.md) | WORKSPACE_ADMIN, SUPERADMIN | ✅ login + recovery<br>🚧 `POST /auth/select-profile` pendiente | ❌ sin diseños aprobados | Subflujo de recuperación de contraseña incluido en el mismo `flujo.md`. SUPERADMIN omite selección de perfil. |
| [post-login.md](post-login.md) | Todos los roles | 🚧 shell implementada, switch-workspace con bug conocido | ⚠️ diseños pendientes | Shell autenticada transversal: redirect por rol, menú de usuario, switch de workspace. |
| [actividad-secundaria.md](actividad-secundaria.md) | SUPERADMIN | 🚧 en progreso | ⚠️ diseños parciales | Se dispara desde [registro-sujeto-obligado.md](registro-sujeto-obligado.md) paso 4 cuando el usuario agrega una segunda actividad vulnerable. |
| [registro-sujeto-obligado.md](registro-sujeto-obligado.md) | SUPERADMIN | 🚧 en progreso | ⚠️ diseños parciales | Registro de Notarios e Inmobiliarias (PF y PM). Punto de entrada al flujo de actividad secundaria. |
| [registro-auxiliar.md](registro-auxiliar.md) | WORKSPACE_ADMIN | ✅ implementado | ⚠️ diseños parciales | Registro de auxiliares por sujeto obligado. El `workspaceId` se infiere del JWT — el front no lo manda. |
| [registro-cliente.md](registro-cliente.md) | WORKSPACE_ADMIN | ❌ sin implementar | ❌ sin diseños | Registro de clientes asociados a un sujeto obligado. |

## Relaciones entre flujos

```
inicio-sesion
  └── post-login (shell autenticada)
        ├── SUPERADMIN → registro-sujeto-obligado
        │                     └── (paso 4) → actividad-secundaria
        ├── WORKSPACE_ADMIN → registro-auxiliar
        │                   → registro-cliente
        └── AUXILIARY → (pendiente)
```

## Convenciones de estado

| Símbolo | Significado |
|---|---|
| ✅ | Implementado y en producción |
| 🚧 | En progreso — parcialmente implementado |
| ⚠️ | Existe pero incompleto |
| ❌ | No existe aún |
