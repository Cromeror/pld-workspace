# Result Page — Post-finalize

Pantalla que se muestra UNA SOLA VEZ con el `tempPassword` retornado por `POST /finalize`.

Ruta: `/admin/reporting-entity/register/result`. La password viaja por `react-router state`, NO por URL ni localStorage.

## Screenshots a documentar

- `success-with-temp-password.png` — pantalla principal con tempPassword visible + botón Copiar + advertencia "Solo se mostrará una vez".
- `password-copied-feedback.png` — feedback visual tras click en Copiar (toast/checkmark).
- `confirm-button.png` — botón "Entendido" que regresa al home.
- `password-hidden-toggle.png` — si hay opción de mostrar/ocultar la password con un eye-icon.
