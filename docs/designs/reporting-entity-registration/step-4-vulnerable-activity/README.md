# Step 4 — Actividad vulnerable

Form con dropdown de actividad + fecha inicial + domicilio completo + descripción de actividad.

## Endpoint en `onNext`

`POST /admin/registration/:id/vulnerable-activity` (upsert — sobreescribe en re-POST).
Body:
```ts
{
  activity: ActividadVulnerable,  // enum del catálogo
  startDate?: string (YYYY-MM-DD),
  address: {
    street, exteriorNumber, interiorNumber?, neighborhood,
    municipality, city, state, postalCode, country,
    roadType?, locality?
  },
  activityPerformedAtAddress: string
}
```

## Catálogo

El dropdown de `activity` consume `GET /catalogs/vulnerable-activities` (Q8 — change separado `add-vulnerable-activities-catalog`). Hasta que ese change esté implementado, fallback a constante TS local.

## Screenshots a documentar

- `empty.png` — al entrar.
- `dropdown-open.png` — dropdown de actividad abierto con las 10 opciones del catálogo.
- `with-activity-selected.png` — actividad seleccionada, sección de domicilio visible.
- `with-data-valid.png` — todos los campos llenos.
- `error-required.png` — campos obligatorios sin llenar.
- `loading.png`, `error-server.png`.

## ⚠ Pendiente

El `FLUJO_REGISTRO_SUPERADMIN.md` menciona que el título de la sección cambia según userRole:
- Notarías: "FE PÚBLICA (SERVIDORES PÚBLICOS...)"
- Inmobiliarias: "TRANSMISIÓN DE BIENES INMUEBLES"

Confirmar si el diseño efectivamente cambia el título o si es solo prosa de la ley.
