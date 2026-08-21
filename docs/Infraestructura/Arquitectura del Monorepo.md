Cómo está organizado `kipo-platform`: un **monorepo pnpm** con las apps de
producto (landing, dashboard, backend) y los packages compartidos del design
system. Este archivo es el índice de la arquitectura; el detalle de cada capa
vive en documentos aparte. [[Infraestructura]]

- [[#pnpm workspaces|pnpm workspaces]]
- [[#Apps|Apps]]
- [[#Packages|Packages]]
- [[#Scripts raíz|Scripts raíz]]
- [[#Arquitectura por app|Arquitectura por app]]
- [[Arquitectura del Frontend]]
- [[Arquitectura del Backend]]

---

## pnpm workspaces

El repo usa **pnpm workspaces** (no Turborepo/Nx). La declaración de qué es un
paquete del workspace vive en `pnpm-workspace.yaml`:

```yaml
packages:
  - 'src/apps/*'
  - 'src/packages/*'
allowBuilds:
  '@vercel/speed-insights': true
  esbuild: true
  sharp: true
  unrs-resolver: false
```

Todo directorio con un `package.json` bajo `src/apps/*` o `src/packages/*` es un
paquete del workspace. Se referencian entre sí con el protocolo `workspace:*`
—no por versión publicada—, así que un cambio en un package se ve al instante en
las apps que lo consumen:

```jsonc
// src/apps/kipo-dashboard/package.json
"dependencies": {
  "@kipo/fonts":    "workspace:*",
  "@kipo/theme":    "workspace:*",
  "@kipo/ui-react": "workspace:*"
}
```

Requisitos (raíz `package.json`): **Node ≥ 22**, **pnpm ≥ 10**. El repo es
`"type": "module"` y `private`.

`allowBuilds` es la allowlist de paquetes a los que pnpm permite ejecutar
scripts de post-install (compilar binarios nativos como `sharp`/`esbuild`); es
el reemplazo de pnpm 10 al viejo `.npmrc` de `node-linker`.

---

## Apps

Las 3 apps de producto viven en `src/apps/`. El nombre del directorio y el
`name` del `package.json` **no siempre coinciden** —los filtros de pnpm usan el
`name`, no la carpeta:

| Directorio | `name` (para `--filter`) | Stack | Rol |
|---|---|---|---|
| `src/apps/kipo-landing` | `Kipo-landing` | **Astro 4** + Tailwind 3 | Sitio público / marketing (`kipo.com.mx`) |
| `src/apps/kipo-dashboard` | `kipo-dashboard-ui` | **Next.js 16** App Router, React 19 | App del cliente (`app.kipo.com.mx` + tenants por subdominio) |
| `src/apps/kipo-platform` | `kipo-api` | **Flask** (Python, `uv`) | Backend / API (`api.kipo.com.mx`) |

Notas:

- El **backend es Python** dentro del monorepo JS: su `package.json` solo envuelve
  comandos `uv` (`uv sync`, `uv run flask run`), de modo que `pnpm --filter
  kipo-api dev` funcione igual que las apps JS. El código y dependencias reales
  viven en `pyproject.toml` + `uv.lock`.
- El mapeo de cada app a su dominio/entorno de despliegue (prod vs staging,
  wildcards de tenants) está en [[Infraestructura]].

---

## Packages

Paquetes compartidos en `src/packages/`. Son el **design system propio** que
consumen landing y dashboard:

| Package | Qué expone |
|---|---|
| `@kipo/ui-react` | Componentes React del design system (build con `pnpm build:packages`; tiene Storybook) |
| `@kipo/theme` | CSS variables / tokens de tema (`var(--brand)`, `var(--surface-card)`, …) |
| `@kipo/tokens` | Tokens crudos de diseño (fuente de los del tema) |
| `@kipo/fonts` | Fuentes empaquetadas |
| `@kipo/fonts-kipo` | Fuentes de marca Kipo |

`@kipo/ui-react` es el único que **requiere build previo** antes de compilar las
apps —por eso casi todos los scripts de build lo anteponen (`pnpm
build:packages && …`).

---

## Scripts raíz

Orquestados desde el `package.json` de la raíz con filtros de pnpm:

```jsonc
"dev":            "pnpm -r --parallel dev",              // todas las apps a la vez
"dev:dashboard":  "pnpm --filter kipo-dashboard-ui dev",
"dev:landing":    "pnpm --filter Kipo-landing dev",
"dev:api":        "pnpm --filter kipo-api dev",

"build":          "pnpm build:packages && pnpm --parallel --filter './src/apps/**' build",
"build:packages": "pnpm --filter @kipo/ui-react build",  // el UI se compila primero

"lint":           "pnpm -r --parallel lint",
"typecheck":      "pnpm -r --parallel typecheck",
"install:py":     "pnpm --filter kipo-api sync",         // uv sync del backend
"test:e2e":       "playwright test --config e2e/playwright.config.ts"
```

- `-r` = recursivo sobre todos los paquetes; `--parallel` = en simultáneo.
- Los **E2E con Playwright** viven en `e2e/` en la raíz (cruzan apps), no dentro
  de una app.
- Hooks de git con **husky** + **lint-staged** (`prepare: husky`).

---

## Arquitectura por app

Las dos apps con lógica de negocio —dashboard (front) y backend— comparten la
**misma filosofía**: **Clean Architecture** + **Screaming Architecture** + una
versión **diluida de DDD**. La estructura de carpetas grita *qué hace el negocio*
(dominios: `invoice`, `customer`, `billing`, `subscriptions`…) y no *qué
herramienta se usó*; dentro de cada dominio las capas apuntan hacia el centro
(dominio ← aplicación ← infraestructura).

Cada lado aplica esa filosofía con los idiomas de su stack:

- **Front** (Next.js, TypeScript, funcional): DDD por dominio con
  `core/` (application·domain·infrastructure) + `ui/`. → [[Arquitectura del Frontend]]
- **Back** (Flask, Python, funcional, sin clases de servicio): dominios planos
  con `commands` → `execute` (dispatcher) → `operations` → `repository`. →
  [[Arquitectura del Backend]]

La landing (Astro) es un sitio de contenido: no sigue este patrón de dominios,
solo consume `@kipo/ui-react` y `@kipo/theme`.
