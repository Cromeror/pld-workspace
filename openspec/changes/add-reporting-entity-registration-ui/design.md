# Design: Reporting Entity Registration UI

Documenta las decisiones técnicas. Cada sección referencia el ID del Q o R relevante del thread de jarvis `5a231957-c89e-46ef-9122-82f482247a67`.

## Eventos del Wizard (Q1)

`onNext` / `onPrevious` / `onFinish`. Nombres React-idiomáticos consistentes con el resto del codebase.

API extendida del Wizard:

```ts
type WizardStep<S> = {
  key: string;
  title: string;
  label?: string;
  render: (ctx: WizardContext<S>) => ReactNode;
  isValid?: (state: S) => boolean;
  shouldSkip?: (state: S) => boolean;
  onNext?: (ctx: WizardContext<S>) => Promise<boolean | void>;
  onPrevious?: (ctx: WizardContext<S>) => void | Promise<void>;
};

type WizardContext<S> = {
  state: S;
  setState: (patch: Partial<S>) => void;
  goNext: () => void;
  goBack: () => void;
  goTo: (key: string) => void;
  isTransitioning: boolean;
  transitionError: Error | null;
};
```

Implementación clave:
- Si `onNext` retorna `false` o lanza, no avanza. Error guardado en `transitionError`.
- Botón Siguiente con `loading={isTransitioning}` y `disabled={isTransitioning || nextDisabled}`.
- Botón Regresar con `disabled={isFirst || isTransitioning}`.
- Banner inline en card cuando `transitionError !== null`.

## Persistencia paso a paso (Q2)

Cada `Siguiente` dispara el POST correspondiente. El BE persiste checkpoint. El FE solo avanza si 2xx.

## Step 1 con RFC en step 2 (Q11)

El step 1 es completamente local (sin POST). El RFC se captura en step 2 (form de identificación). El primer POST al BE sucede en `onNext` del step 2:

```ts
onNext: async ({ state, setState }) => {
  // Crear draft si no existe
  if (!state.registrationId) {
    try {
      const { id } = await registrationService.startDraft({
        profileType: state.profileType,
        userRole: state.userRole,
        rfc: state.identification.rfc,
      });
      setState({ registrationId: id });
      localStorage.setItem(LOCAL_STORAGE_KEYS.ACTIVE_REGISTRATION_ID, id);
    } catch (err) {
      if (err.status === 409) {
        // Modal de resume
        setResumeModal({ open: true, existingId: err.body.registrationId });
        return false;
      }
      throw err;
    }
  }
  await registrationService.saveIdentification(state.registrationId, state.identification);
};
```

## Múltiples contactos (Q3)

Step 3 con array dinámico. Mín. 1 contacto requerido para `isValid`. Botón "Agregar contacto" suma fila vacía. Cada fila tiene icono basurero.

`onNext` itera la lista y dispara `POST /:id/contact` por cada contacto **no persistido aún** (trackeo via `persistedAt: Date | null` en cada fila):

```ts
onNext: async ({ state, setState }) => {
  for (let i = 0; i < state.contacts.length; i++) {
    const c = state.contacts[i];
    if (!c.persistedAt) {
      await registrationService.appendContact(state.registrationId, {
        countryCode: c.countryCode,
        phone: c.phone,
        email: c.email,
        cellphone: c.phone, // Q3 hack: cellphone = phone hasta que diseño lo agregue
      });
      const updated = [...state.contacts];
      updated[i] = { ...c, persistedAt: new Date() };
      setState({ contacts: updated });
    }
  }
};
```

## Resume vía localStorage (Q4)

Una sola key: `pld:activeRegistrationId`. Lifecycle:

1. Setear al crear draft (`POST /` exitoso en step 2).
2. Limpiar al `POST /finalize` exitoso (cierre del modal de éxito).
3. Limpiar al `DELETE /:id` exitoso (cambio de profileType en step 1).
4. Limpiar en logout del usuario.

Al cargar la página:

```ts
useEffect(() => {
  const activeId = localStorage.getItem(LOCAL_STORAGE_KEYS.ACTIVE_REGISTRATION_ID);
  if (!activeId) return;
  registrationService.getRegistration(activeId)
    .then(draft => {
      // R1: si COMPLETED → silent clean (race condition de finalize)
      if (draft.status === 'COMPLETED') {
        localStorage.removeItem(LOCAL_STORAGE_KEYS.ACTIVE_REGISTRATION_ID);
        return;
      }
      // Si IN_PROGRESS, mostrar modal de resume
      setResumeModal({ open: true, draft });
    })
    .catch(err => {
      // 404, 410, network: clear y arrancar limpio
      localStorage.removeItem(LOCAL_STORAGE_KEYS.ACTIVE_REGISTRATION_ID);
    });
}, []);
```

## RoleProtectedRoute con JWT decode (Q5 + V2)

Verificado en BE: el JWT incluye `role` en payload. NO existe `GET /me`. Decodificar JWT sin verificar firma (BE valida cada request).

```ts
// src/hooks/useCurrentUserRole.ts
import { jwtDecode } from 'jwt-decode';

export const useCurrentUserRole = (): UserRole | null => {
  const token = localStorage.getItem(LOCAL_STORAGE_KEYS.AUTH_TOKEN);
  if (!token) return null;
  try {
    const payload = jwtDecode<{ role: UserRole }>(token);
    return payload.role;
  } catch {
    return null;
  }
};
```

```tsx
// src/routes/RoleProtectedRoute.tsx
const RoleProtectedRoute: FC<{ requiredRoles: UserRole[]; children: ReactNode }> = ({
  requiredRoles, children,
}) => {
  const role = useCurrentUserRole();
  if (!role || !requiredRoles.includes(role)) {
    return <Navigate to={RoutesUrl.HOME} replace />;
  }
  return <ProtectedRoute>{children}</ProtectedRoute>;
};
```

## Sin botón Cancelar (Q6)

Solo Regresar / Siguiente / Confirmar. Cancelación implícita: cerrar pestaña.

## Ruta + redirect post-login (Q7)

Ruta `/admin/reporting-entity/register`. Mapping configurable:

```ts
// src/config/postLoginRedirect.ts
export const POST_LOGIN_REDIRECT_BY_ROLE: Partial<Record<UserRole, string>> = {
  SUPERADMIN: RoutesUrl.REPORTING_ENTITY_REGISTER,
  // NOTARIO, INMOBILIARIA, AUXILIAR: fallback DEFAULT_REDIRECT
};
export const DEFAULT_REDIRECT = RoutesUrl.HOME;
export const getPostLoginRedirect = (role: UserRole | undefined): string =>
  (role && POST_LOGIN_REDIRECT_BY_ROLE[role]) ?? DEFAULT_REDIRECT;
```

## PF + PM en flujo único (Q9)

`shouldSkip: s => s.profileType !== 'PERSONA_MORAL'` en el step `compliance-responsible`. El stepper renderiza 5 (PF) o 6 (PM) automáticamente. El step de identificación renderiza condicionalmente `<PhysicalIdentificationStep>` o `<MoralIdentificationStep>` según `state.profileType`.

## Modales

### ConfirmDialog (Q13)

Mover `ui/Dialog` → `atoms/Dialog`. Crear `molecules/ConfirmDialog`:

```tsx
type Props = {
  open: boolean;
  title: string;
  description: string;
  cancelLabel?: string;
  confirmLabel?: string;
  onCancel: () => void;
  onConfirm: () => void;
  isConfirming?: boolean;
};
```

### SuccessFinalizeModal (Q14)

Sobre el step 5. Botones: Copiar contraseña, **Reenviar correo (maquetado sin función — Q12)**, Finalizar. Click en Finalizar:
- Clear localStorage.
- Reset state al `initialState`.
- Cerrar modal.
- Wizard vuelve al step 1 limpio en la misma URL.

### ErrorFinalizeModal con reintentos (R1)

Contador interno de reintentos (máx 3). Botones: Cancelar, Intentar de nuevo. Tras 3 fallos: botón "Intentar de nuevo" disabled con mensaje "Se alcanzó el número máximo de reintentos. Si el problema persiste, contactá soporte."

`registrationId` se mantiene en localStorage durante los reintentos (al refresh, contador se resetea).

## Step 4: campos del domicilio (Q15)

Form incluye los campos visibles en el diseño + `country` (default "México") + `city` (Ciudad o población). Country: dropdown con catálogo, visualmente activo pero no interactivo (R4).

## Catálogos multi-país (Q16)

Componente `<AdministrativeDivisionsField />` (molecule) recibe `countryCode` + `value` + `onChange`. Internamente:

1. `useGetAdministrativeStructure(countryCode)` para conocer los niveles del país.
2. Renderiza N dropdowns en cascada (uno por nivel).
3. Cada dropdown usa `useGetAdministrativeEntries(countryCode, level, parentCode)`.
4. Al cambiar nivel N, los niveles N+1, N+2, ... se resetean.

Para v1, MX tiene 3 niveles. El nivel 3 (`neighborhood`) en MX no tiene entries cargadas → si la entries query devuelve array vacío, fallback a `<InputText />` libre.

## Stepper dinámico (Q17)

El Wizard ya tiene la lógica via `shouldSkip`. No requiere cambios.

## Botón Confirmar + flow del modal (Q18)

`finalButtonLabel="Confirmar"`. El `onFinish` del Wizard NO dispara `POST /finalize` directo: solo abre el `ConfirmDialog`. El `onConfirm` del modal dispara el POST.

```tsx
const [confirmOpen, setConfirmOpen] = useState(false);
const [isFinalizing, setIsFinalizing] = useState(false);
const [retryCount, setRetryCount] = useState(0);
const finalize = useFinalizeRegistration();

const handleConfirm = async () => {
  setIsFinalizing(true);
  try {
    const result = await finalize.mutateAsync(state.registrationId);
    setConfirmOpen(false);
    setSuccessModal({ open: true, ...result });
  } catch (err) {
    setRetryCount(c => c + 1);
    setErrorModal({ open: true });
  } finally {
    setIsFinalizing(false);
  }
};
```

## Loading + errores (Q19)

- Wizard ya cubre el spinner por step en `onNext`.
- Error inline en banner cuando `onNext` falla.
- Modal de error post-finalize con contador R1.

## Mapping resume (Q20)

```ts
const BE_STEP_TO_FE_KEY: Record<string, string> = {
  IDENTIFICATION: STEP_KEYS.IDENTIFICATION,
  CONTACT: STEP_KEYS.CONTACT,
  VULNERABLE_ACTIVITY: STEP_KEYS.VULNERABLE_ACTIVITY,
  COMPLIANCE_RESPONSIBLE: STEP_KEYS.COMPLIANCE_RESPONSIBLE,
  COMPLETED: STEP_KEYS.REVIEW,
};
```

Al reanudar: pre-llena el state desde el draft del BE + `goTo(targetKey)`.

## Cambio de profileType con draft activo (Q21)

Si el usuario está en step 1 con `registrationId !== undefined` y cambia profileType o userRole:

1. Mostrar `ConfirmDialog`: "Cambiar el tipo descartará el borrador actual y se creará uno nuevo. ¿Continuar?".
2. Si confirma:
   - `await registrationService.discardRegistration(registrationId)`.
   - `localStorage.removeItem(...)`.
   - `setState({ ...initialState, userRole: nextUserRole, profileType: nextProfileType })`.
3. Si cancela: revertir la selección visual al original.

## Schemas con zod

Cada step content-component exporta su schema:

```ts
// schemas.ts
export const physicalIdentificationSchema = z.object({
  firstName: z.string().min(1, 'Nombre requerido'),
  paternalSurname: z.string().min(1, 'Primer apellido requerido'),
  // ... etc
  rfc: z.string().regex(/^[A-ZÑ&]{4}\d{6}[A-Z0-9]{3}$/i, 'RFC inválido'),
  curp: z.string().regex(/^[A-Z]{4}\d{6}[HM][A-Z]{5}[A-Z0-9]{2}$/i, 'CURP inválido'),
});
// idem moralSchema, contactSchema, vulnerableActivitySchema, complianceResponsibleSchema
```

`isValid` de cada step usa `schema.safeParse(s.slice).success`.

## Step 1 al refrescar (R2)

Si el usuario refresca durante step 1 SIN haber pasado a step 2: pierde la selección. Aceptado, documentado.

## Hooks en `catalogsQueries.ts` existente (R3)

Los hooks de catálogos del change `add-reference-catalogs` viven en `catalogsQueries.ts` junto a los existentes. NO carpeta nueva. `staleTime: 24h`.

## Dropdown país visualmente activo pero sin click (R4)

Implementación con PrimeReact:

```tsx
<Dropdown
  value={selectedCountry}
  options={countries}
  optionLabel="name"
  optionValue="code"
  pt={{
    input: { onClick: (e) => e.preventDefault() },
    trigger: { className: "pointer-events-none" },
  }}
/>
```

Si PrimeReact no soporta esto cleanly, fallback: `<Dropdown disabled />` con override de estilos para que NO se vea disabled.

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Modificar el Wizard organism puede romper CreateUserPage legacy | Medio | Test manual de CreateUserPage tras agregar `onNext`/`onPrevious` (deben ser opcionales) |
| jwt-decode no instalado | Bajo | Tarea Phase 0 instala lib |
| Shape exacto de `GET /:id` (V3) | Medio | Tarea Phase 0 verifica shape leyendo BE adapter |
| Dataset municipios MX (~150KB) | Bajo (BE bundle) | Aceptable; optimización futura |
| Race condition al hacer 2 POSTs encadenados en step 2 | Bajo | El primero (POST /) debe completar antes del segundo (POST /:id/identification) — secuencial, no paralelo |

## Open questions

- ¿Toast de éxito post-cierre del SuccessFinalizeModal? Decisión durante implementación.
- ¿Componente del listado merece change separado? Sí, futuro `add-reporting-entity-list`.
