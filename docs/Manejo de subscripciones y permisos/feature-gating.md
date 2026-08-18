# Feature Gating — Subscripciones y Add-ons

## Arquitectura

Kipo usa un sistema híbrido **Entitlements + RBAC** para controlar el acceso a módulos y características según el plan contratado.

```
Plan del tenant (basico/emprendedor/pyme)
  + Add-ons activos (addon_taller, addon_salud, ...)
         ↓  recalculate_entitlements() — en cada webhook de Stripe
  tenants.entitlements JSONB  ← cache pre-computado por tenant
         ↓  @require_entitlement en cada endpoint
  tenant_users.permissions TEXT[]  ← permisos individuales por usuario (RBAC)
         ↓  @require_permission en cada endpoint
```

### Estructura del JSONB `entitlements`

```json
{
  "max_users": 3,
  "history_months": null,
  "modules": ["quotes", "recurring_invoices", "whatsapp_reminders", "reports", "multi_user"],
  "features": ["csf_ocr", "efos_alert", "rfc_realtime_validation", "rep_auto"],
  "addons": ["addon_taller"]
}
```

| Campo | Qué es |
|-------|--------|
| `modules` | Acceso completo a una sección del app (nav, CRUD completo) |
| `features` | Capacidad específica dentro de un módulo que existe en free |
| `addons` | Pack vertical activo (solo visible si está contratado) |
| `max_users` | Límite de usuarios del plan |
| `history_months` | Meses de historial disponibles (`null` = sin límite) |

---

## Planes y sus entitlements

| Tier (backend) | Tier (frontend) | Nombre UI | max_users |
|----------------|-----------------|-----------|-----------|
| `basico` | `free` | Básico | 1 |
| `emprendedor` | `pro` | Emprendedor | 1 |
| `pyme` | `enterprise` | PyME | 3 |

> `emprendedor === pro` y `pyme === enterprise` en código.

### Módulos por plan

| Módulo | Básico | Emprendedor | PyME |
|--------|--------|-------------|------|
| Facturas (CFDI básico) | ✓ | ✓ | ✓ |
| `quotes` — Cotizaciones | — | ✓ | ✓ |
| `recurring_invoices` — Facturas recurrentes | — | ✓ | ✓ |
| `whatsapp_reminders` — Recordatorios | — | ✓ | ✓ |
| `reports` — Reportes | — | ✓ | ✓ |
| `multi_user` — Multi-usuario | — | — | ✓ |

### Features por plan

| Feature | Básico | Emprendedor | PyME |
|---------|--------|-------------|------|
| `csf_ocr` — Escaneo CSF con OCR | — | ✓ | ✓ |
| `efos_alert` — Alerta EFOS/EDOS | — | ✓ | ✓ |
| `rfc_realtime_validation` — RFC en tiempo real | — | ✓ | ✓ |
| `advanced_search` — Búsqueda avanzada | — | ✓ | ✓ |
| `rep_auto` — REP automático | — | — | ✓ |
| `credit_note_advanced` — Nota de Crédito con CFDI original | — | — | ✓ |
| `guided_cancellation` — Cancelación guiada SAT | — | — | ✓ |
| `diot_report` — Reporte pre-DIOT | — | — | ✓ |
| `iva_dashboard` — Dashboard IVA | — | — | ✓ |
| `income_comparison` — Comparativa ingresos | — | — | ✓ |

### Add-ons disponibles (No requiere subscripcion activa)

| Add-on | Módulos | Features |
|--------|---------|----------|
| `addon_taller` — Pack Taller | `work_orders` | `taller_sat_catalog` |
| `addon_salud` — Pack Salud | `patient_catalog` | `salud_honorarios_complement` |
| `addon_educativo` — Pack Educativo | — | `educativo_curp`, `educativo_colegiaturas` |
| `addon_renta` — Pack Renta | `property_management` | `renta_isr_alert` |
| `addon_carta_porte` — Pack Carta Porte | `carta_porte` | — |

---

## Regla de UX — Qué mostrar y qué ocultar

### Módulos de plan: siempre visibles, bloqueados si no aplica

Los módulos de plan (`quotes`, `reports`, `multi_user`, etc.) **siempre aparecen en la navegación**, aunque el usuario no tenga el plan requerido. Si no tiene acceso, el módulo muestra un estado de upgrade — no desaparece.

El modelo es el de Vercel y otras apps SaaS modernas: el usuario puede ver que la feature existe, lo que incentiva el upgrade.

```
┌─────────────────────────────────────────────┐
│  Cotizaciones                               │
│                                             │
│  [Contenido bloqueado / preview]            │
│                                             │
│  Esta función está disponible en el         │
│  plan Emprendedor.          [Upgrade →]     │
└─────────────────────────────────────────────┘
```

### Add-ons: ocultos si no están contratados

Los add-ons (`addon_taller`, `addon_salud`, etc.) **no se muestran en el dashboard** a menos que el tenant los tenga activos. Son módulos verticales que solo son relevantes para ciertos giros de negocio.

> Regla simple: si `entitlements.addons` no incluye el addon, no renderices nada relacionado a él.

---

## Cómo implementar un nuevo módulo de plan

### 1. Agregar al catálogo (`plan_catalog.py`)

```python
# src/apps/kipo-platform/tenant/plan_catalog.py

PLAN_CATALOG = {
    "emprendedor": {
        "modules": (
            "quotes",
            "recurring_invoices",
            "mi_nuevo_modulo",   # ← agregar aquí
        ),
        ...
    },
}
```

Si el módulo también necesita features granulares dentro de él:

```python
"features": (
    "mi_nuevo_modulo:feature_avanzada",
),
```

### 2. Proteger los endpoints del backend

```python
# src/apps/kipo-platform/api/v1/endpoints/mi_modulo.py

from shared.auth_decorators import require_auth, require_entitlement, require_permission

@bp.route("/mi-modulo", methods=["POST"])
@require_auth
@require_entitlement("mi_nuevo_modulo")       # bloquea con 402 si el plan no incluye el módulo
@require_permission("mi_modulo:create")       # bloquea con 403 si el usuario no tiene permiso
def crear_item():
    ...
```

Respuestas de error:
- `402 Payment Required` + `{"error": "FEATURE_NOT_AVAILABLE", "required": "mi_nuevo_modulo"}` — plan insuficiente
- `403 Forbidden` + `{"error": "PERMISSION_DENIED", "required": "mi_modulo:create"}` — permiso insuficiente

### 3. Agregar el permiso al catálogo de permisos

En `api/v1/endpoints/tenants.py`, agregar el nuevo permiso a `_ALL_PERMISSIONS`:

```python
_ALL_PERMISSIONS = [
    ...
    "mi_modulo:view",
    "mi_modulo:create",
    "mi_modulo:delete",
]
```

### 4. Implementar el gate en el frontend

**Opción A — Componente `<FeatureGate>` (recomendado para módulos completos):**

```tsx
// En la vista o en el layout del módulo
import { FeatureGate } from "@/src/subscriptions/ui/components/FeatureGate"
import { UpgradeBanner } from "@/src/shared/ui/components/UpgradeBanner"

export function MiModuloView() {
  return (
    <FeatureGate
      module="mi_nuevo_modulo"
      fallback={
        <UpgradeBanner
          feature="Mi Nuevo Módulo"
          requiredPlan="Emprendedor"
        />
      }
    >
      {/* contenido del módulo */}
    </FeatureGate>
  )
}
```

**Opción B — Hook `useEntitlement` (para features dentro de módulos existentes):**

```tsx
import { useEntitlement } from "@/src/subscriptions/ui/hooks/useEntitlement"

export function InvoiceRow({ invoice }: Props) {
  const hasAdvancedSearch = useEntitlement("advanced_search")

  return (
    <div>
      {hasAdvancedSearch && <SearchBar />}
    </div>
  )
}
```

**Importante:** No uses `useEntitlement` para ocultar el módulo completo del nav. El módulo siempre aparece; solo el contenido interno se reemplaza por el `UpgradeBanner`.

### 5. Para add-ons: mostrar solo si está activo

```tsx
import { useEntitlement } from "@/src/subscriptions/ui/hooks/useEntitlement"

// En el nav o sidebar
function Sidebar() {
  const hasTaller = useEntitlement("work_orders")  // módulo del addon_taller

  return (
    <nav>
      <NavItem href="/invoices">Facturas</NavItem>
      <NavItem href="/quotes">Cotizaciones</NavItem>
      {hasTaller && <NavItem href="/work-orders">Órdenes de Trabajo</NavItem>}
    </nav>
  )
}
```

Los add-ons no tienen `UpgradeBanner` — simplemente no aparecen si no están contratados.

---

## Cómo implementar una nueva feature dentro de un módulo existente

Mismo flujo, pero usando `features` en lugar de `modules` en el catálogo:

```python
# plan_catalog.py
"emprendedor": {
    "features": (
        "csf_ocr",
        "mi_nueva_feature",   # ← agregar
    ),
}
```

Backend:
```python
@require_entitlement("mi_nueva_feature")
```

Frontend:
```tsx
const hasFeature = useEntitlement("mi_nueva_feature")
// usar para mostrar/ocultar un botón o sección dentro de un módulo existente
```

---

## Cómo implementar un nuevo add-on

### 1. Agregar al catálogo

```python
# plan_catalog.py
ADDON_CATALOG = {
    "addon_nuevo": {
        "min_plan": "emprendedor",
        "modules": ("nuevo_modulo",),
        "features": ("nueva_feature",),
        "stripe_price_id": os.environ.get("STRIPE_PRICE_ADDON_NUEVO"),
    },
}
```

### 2. Agregar la variable de entorno

En todos los entornos:
```
STRIPE_PRICE_ADDON_NUEVO=price_xxxxx
```

Ver [environment-variables.md](environment-variables.md) para el proceso de agregar variables.

### 3. Crear el Price en Stripe

El add-on se vende como un ítem adicional en la suscripción de Stripe (subscription item). Cuando el usuario lo compra, Stripe dispara `customer.subscription.updated` con el nuevo ítem — el webhook lo detecta, actualiza `tenant_addons`, y recalcula `entitlements`.

### 4. Frontend: solo mostrar si activo

```tsx
const hasAddon = useEntitlement("nuevo_modulo")
{hasAddon && <NuevoModuloNavItem />}
```

---

## Archivos clave de referencia

| Archivo | Propósito |
|---------|-----------|
| `tenant/plan_catalog.py` | Fuente de verdad: qué incluye cada plan y add-on |
| `shared/auth_decorators.py` | `@require_entitlement` y `@require_permission` |
| `tenant/operations/recalculate_entitlements.py` | Recalcula y persiste entitlements JSONB |
| `subscriptions/operations/sync_from_webhook.py` | Procesa webhook Stripe → actualiza addons → recalcula |
| `api/v1/endpoints/tenants.py` | `GET /api/v1/tenants/me` — retorna entitlements + permisos |
| `src/subscriptions/ui/hooks/useEntitlement.ts` | Hook frontend para chequear módulo/feature |
| `src/subscriptions/ui/hooks/usePermission.ts` | Hook frontend para chequear permiso RBAC |
| `src/subscriptions/ui/components/FeatureGate/` | Componente declarativo de gating |

---

## DB: tablas relacionadas

```sql
-- Entitlements pre-computados del tenant
tenants.entitlements JSONB

-- Permisos individuales del usuario dentro del tenant (RBAC)
tenant_users.permissions TEXT[]

-- Add-ons activos por tenant
tenant_addons (tenant_id, addon_id, stripe_subscription_item_id, status)
```

Las migraciones están en `supabase/migrations/20260818000001-000003`.
