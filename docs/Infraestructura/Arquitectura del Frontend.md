Cómo está estructurado el dashboard (`src/apps/kipo-dashboard`, `name:
kipo-dashboard-ui`): **Next.js 16 App Router + React 19**, organizado con
**Screaming Architecture** por dominio, **Clean Architecture** en capas y una
versión **diluida de DDD**. Parte de [[Arquitectura del Monorepo]].

- [[#La estructura grita el dominio|La estructura grita el dominio]]
- [[#Anatomía de un dominio|Anatomía de un dominio]]
- [[#Capa core — Clean + DDD diluido|Capa core — Clean + DDD diluido]]
- [[#Capa ui|Capa ui]]
- [[#Reglas de custom hooks|Reglas de custom hooks]]
- [[#Tipo UI vs entidad de dominio|Tipo UI vs entidad de dominio]]
- [[#shared|shared]]

---

## La estructura grita el dominio

Bajo `src/` los directorios de primer nivel **son los dominios de negocio**, no
capas técnicas. Al abrir el proyecto se lee *qué hace Kipo*, no *con qué está
hecho*:

```
src/
  auth/        billing/      catalogs/     customers/
  dashboard/   onboarding/   settings/     stamp-packs/
  subscriptions/             shared/
```

Cada dominio se parte en dos mitades: **`core/`** (lógica de negocio, sin React)
y **`ui/`** (React). Es la frontera de Clean Architecture: la UI depende del
core, nunca al revés.

---

## Anatomía de un dominio

Estructura completa (ejemplo `billing`):

```
src/billing/
  core/
    application/
      dtos/          ← DTOs de entrada/salida hacia el API
      use-cases/     ← Casos de uso (lógica de aplicación)
    domain/
      entities/      ← Entidades de dominio (inmutables, tipadas)
      value-objects/ ← Branded types con validación (Email, TaxId, …)
      repositories/  ← Interfaces IXxxRepository
      exceptions/    ← Errores de dominio tipados
    infrastructure/
      mappers/       ← respuesta del API → entidad de dominio
      repositories/  ← Implementaciones HTTP de los repositorios
  ui/
    components/      ← Componentes React del dominio
    hooks/           ← Custom hooks (una responsabilidad c/u)
    data/            ← Catálogos estáticos, mocks (NO hooks)
    views/           ← Vistas completas montadas en las rutas
```

Las **rutas** (`app/`, App Router) son delgadas: montan una *view* del dominio.
La lógica no vive en `app/`.

---

## Capa core — Clean + DDD diluido

Flujo de dependencias hacia adentro: `infrastructure → application → domain`.

- **domain/** — el centro, sin dependencias externas.
  - `entities/`: entidades **inmutables** (`Readonly<{…}>`), con value objects y
    lógica pura.
  - `value-objects/`: **branded types** con validación (p. ej. `Email`, `TaxId`)
    —encapsulan la regla de "qué es un valor válido".
  - `repositories/`: solo **interfaces** `IXxxRepository` (contrato, no
    implementación).
  - `exceptions/`: errores de dominio tipados.
- **application/** — orquesta el dominio.
  - `use-cases/`: casos de uso que combinan entidades + repositorios.
  - `dtos/`: forma de los datos que cruzan hacia/desde el API.
- **infrastructure/** — los detalles reemplazables.
  - `repositories/`: implementación **HTTP** de las interfaces del dominio.
  - `mappers/`: traducen la respuesta cruda del API → entidad de dominio.

"DDD diluido" = se toman los patrones útiles (entidades, value objects,
repositorios como interfaz, casos de uso) **sin** el aparato pesado (aggregates,
domain events, bounded contexts formales). Suficiente para aislar la lógica,
sin sobre-ingeniería.

---

## Capa ui

Cada componente vive en su **propia subcarpeta** con `index.tsx` como entrada.
El componente *orquestador* del dominio (`components/invoices/`,
`components/customers/`) compone los hooks y renderiza a los hijos.

```
ui/components/
  InvoiceRow/
    index.tsx      ← SOLO JSX
    types.ts       ← interface InvoiceRowProps
  InvoiceDetailSheet/
    index.tsx
    types.ts
    constants.ts   ← labels, tonos por estado, formatMXN…
  invoices/
    index.tsx      ← Orquestador: hooks + hijos
  shared/
    types.ts       ← Tipos UI del dominio (UIInvoice, InvoiceStatus)
```

Reglas internas del componente:

- **`index.tsx`** — solo JSX. Sin interfaces ni constantes no triviales inline.
- **`types.ts`** — props y tipos locales (crear si hay ≥ 1 interface).
- **`constants.ts`** — arrays de config, mapas de labels/colores, helpers puros
  (crear si hay ≥ 1 constante no trivial).
- **`components/shared/`** — tipos/utilidades que aplican a todo el dominio.

Otros patrones de UI:

- `"use client"` explícito en todo componente/hook con estado o efectos.
- **Sheets y modales** vía `createPortal(…, document.body)`; reciben `isOpen` o
  el dato a mostrar y retornan `null` si no aplica.
- Comunicación **padre → lista** con `forwardRef` + `useImperativeHandle` (p. ej.
  agregar un item a la lista tras crearlo desde un sheet).
- **Skeleton loaders** por dominio en `src/shared/ui/components/dashboard/
  skeletons.tsx`, replicando la forma visual real.
- **Solo Tailwind**, nada de `style={{ }}` en UI nueva —incluidos colores/tokens
  del design system (`bg-card`, `text-muted-foreground`, …), mapeados en
  `app/globals.css`.

---

## Reglas de custom hooks

**Una responsabilidad por hook**, declarada en su nombre. La lista y sus
mutaciones **no** van juntas:

| Hook | Qué hace |
|---|---|
| `useXxxList` | Solo estado de la lista: `items[]`, `isLoading`, `selectedItem` |
| `useAddXxx` | Solo agregar |
| `useCancelXxx` | Solo cancelar / pasar a cancelled |
| `useDeleteXxx` | Solo eliminar |
| `useXxxForm` | Todo el estado del formulario + `buildXxx()` |
| `useXxxSearch` | Búsqueda/filtrado |

Se **componen en el orquestador** del dominio:

```tsx
const { invoices, setInvoices, isLoading, selectedInvoice } = useInvoiceList()
const addInvoice    = useAddInvoice(setInvoices)
const cancelInvoice = useCancelInvoice(setInvoices)
const deleteInvoice = useDeleteInvoice(setInvoices)
```

Los **catálogos SAT** y datos mock **no son hooks** → van en `ui/data/
catalogs.ts`, nunca en `ui/hooks/`.

---

## Tipo UI vs entidad de dominio

Son distintos a propósito. El **tipo UI** (`ui/components/shared/types.ts`) es
plano y orientado a mostrar; la **entidad de dominio**
(`core/domain/entities/`) es inmutable, con value objects y lógica.

```ts
// UI: plano, strings ya formateados
export interface UIInvoice {
  id: string; folio: string; status: InvoiceStatus
  issuedAt: string; receiverName: string; total: number
}

// dominio: inmutable, value objects
export type Invoice = Readonly<{
  id: InvoiceId; issuedAt: Date
  receiverId: ReceiverId; items: readonly InvoiceConcept[]
}>
```

Los **mappers** de `core/infrastructure/mappers/` convierten respuesta del API →
entidad; la view aplana la entidad → tipo UI para pintar.

---

## shared

`src/shared/` reúne lo transversal a todos los dominios:

```
src/shared/
  domain/          ← tipos/reglas de dominio compartidas
  infrastructure/  ← cliente HTTP, adaptadores comunes
  host/            ← resolución de tenant por subdominio (host)
  ui/              ← componentes UI compartidos (skeletons, layout de dashboard)
```

El backend que consume este front está documentado en [[Arquitectura del
Backend]]; el gating por plan/permiso vive en la doc de subscripciones.
