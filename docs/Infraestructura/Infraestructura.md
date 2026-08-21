# Kipo staging y prod

Runbook para pasar de "nada provisionado" a tener **staging** y **producción**
funcionando para las 3 apps del monorepo `kipo-platform`
(`kipo-landing`, `kipo-dashboard`, `kipo-platform` backend) + Supabase + DNS. [[Diagrama Infraestructura]]

> Para cómo está armado el repo por dentro (pnpm workspaces, apps, packages y la
> arquitectura de front y back), ver [[Arquitectura del Monorepo]].

- [[#Nota sobre plan de Vercel (Hobby vs Pro)|Nota sobre plan de Vercel (Hobby vs Pro)]]
- [[#Arquitectura objetivo|Arquitectura objetivo]]
- [[#Paso 1 — Cuenta de Vercel|Paso 1 — Cuenta de Vercel]]
- [[#Paso 2 — Supabase cloud (2 proyectos)|Paso 2 — Supabase cloud (2 proyectos)]]
- [[#Paso 3 — PAC de timbrado CFDI (sandbox vs producción)|Paso 3 — PAC de timbrado CFDI (sandbox vs producción)]]
- [[#Paso 4 — DNS en kipo.com.mx|Paso 4 — DNS en kipo.com.mx]]
- [[#Paso 5 — GitHub Secrets y Environments|Paso 5 — GitHub Secrets y Environments]]
- [[#Orden recomendado para hacerlo por primera vez|Orden recomendado para hacerlo por primera vez]]
- [[#Apéndice — Si más adelante suben a Vercel Pro|Apéndice — Si más adelante suben a Vercel Pro]]

### Nota sobre plan de Vercel (Hobby vs Pro)

Verificado contra la documentación oficial de Vercel:

| Feature | Hobby (gratis) | Pro |
|---|---|---|
| Preview deployments | ✅ Sí, sin restricción de feature (límite de 100 deploys/día) | ✅ Sí (6,000/día) |
| Wildcard domains (`*.dominio.com`) | ✅ Sí, en todos los planes — requiere delegar el dominio/subdominio a los nameservers de Vercel | ✅ Sí |
| Custom Environments (entorno nombrado "staging" con su propio dominio/env vars, gestionado desde el dashboard) | ❌ No disponible | ✅ 1 environment (Enterprise: hasta 12) |
| Logs de runtime | 1 hora de retención | 1 día |
| **Uso comercial** | ⚠️ Los [fair use guidelines de Vercel](https://vercel.com/docs/plans/hobby) restringen Hobby a "non-commercial, personal use only" | Sin esa restricción |

Dado que Kipo es un SaaS comercial, el plan Hobby es válido para levantar y
probar la infraestructura de staging ahora, pero conviene tener en mente
que el uso comercial real (clientes reales, timbrado real) debería vivir en
Pro. Este documento describe el camino con **Hobby ahora** (sin Custom
Environments) y deja una nota al final sobre qué cambia al subir a Pro.

## Arquitectura objetivo

| App | Prod | Staging |
|---|---|---|
| Landing (Astro) | `kipo.com.mx` / `www.kipo.com.mx` | Preview de Vercel por PR (ya funciona, no requiere dominio fijo) |
| Dashboard (Next.js) | `app.kipo.com.mx` + `*.kipo.com.mx` (tenants) | `staging.kipo.com.mx` + `*.staging.kipo.com.mx` (tenants) |
| Backend (Flask) | `api.kipo.com.mx` | `staging-api.kipo.com.mx` |
| Supabase | proyecto cloud "kipo-prod" | proyecto cloud "kipo-staging" |
| PAC (timbrado CFDI) | credenciales de producción | credenciales de pruebas/sandbox del PAC |

**Un solo proyecto de Vercel por app** (`kipo-dashboard`, `kipo-platform`) —
no hace falta un proyecto hermano `*-staging`. En el flujo del equipo le
llamamos **"staging"**, pero técnicamente lo que se manda es un
**Preview deployment de Vercel** (no un Custom Environment, porque eso
requiere Pro) con un dominio fijo pegado encima vía alias:

```bash
# Deploy normal (sin --prod) → genera una URL de preview
DEPLOY_URL=$(vercel deploy --token "$VERCEL_TOKEN" --yes)

# Fijar ese deployment al dominio de staging (y su wildcard de tenants)
vercel alias "$DEPLOY_URL" staging.kipo.com.mx --token "$VERCEL_TOKEN"
vercel alias "$DEPLOY_URL" "*.staging.kipo.com.mx" --token "$VERCEL_TOKEN"
```

`vercel alias` no está gateado por plan, así que este mecanismo funciona
completo en Hobby. La diferencia contra tener un Custom Environment real es
que **el alias hay que re-ejecutarlo en cada deploy de staging** — no hay
nada en Vercel que automáticamente apunte `staging.kipo.com.mx` al último
preview como sí pasa con Production. Por eso este paso debe vivir dentro
del propio job de CI (ver [Paso 5](#paso-5-github-secrets-y-environments)),
no como algo manual de una sola vez.

> Nota: confirmar al implementar que `vercel alias` acepta el dominio
> wildcard tal cual (`*.staging.kipo.com.mx`) como target — es el
> comportamiento esperado, pero vale la pena validarlo en el primer deploy
> real antes de automatizarlo.

---

## Paso 1 — Cuenta de Vercel

1. Crear cuenta / equipo (Team) en Vercel para la organización — **Hobby está bien para empezar**, no hace falta pagar Pro todavía (ver la nota de arriba).
2. Generar un **token de acceso** (Account Settings → Tokens) — se usará como `VERCEL_TOKEN` en GitHub.
3. Anotar el `Team ID` (Org ID) — se usará como `VERCEL_ORG_ID`.
4. Crear 3 proyectos vacíos (se pueden crear al hacer el primer deploy con `vercel link`, o desde el dashboard):
   - `kipo-landing`
   - `kipo-dashboard`
   - `kipo-platform` (backend)
5. Anotar el **Project ID** de cada uno (Project → Settings → General) — son los `VERCEL_PROJECT_ID_*` que ya esperan los workflows. El mismo ID sirve para prod y staging, ya que ambos entornos viven en el mismo proyecto (prod = deployment de Production, staging = Preview deployment aliasado).

## Paso 2 — Supabase cloud (2 proyectos)

1. Crear proyecto **`kipo-staging`** en https://supabase.com/dashboard.
2. Crear proyecto **`kipo-prod`**.
3. Para cada uno, guardar de **Project Settings → API**:
   - `Project URL` → `PROJECT_URL`
   - `anon` / `publishable` key → `AUTH_KEY_PUBLISHABLE`
   - `service_role` / `secret` key → `AUTH_KEY_SECRET`
   - `Project Settings → Database` → connection string → `DATABASE_URL`
   - `Project Settings → Storage` (S3-compatible) → `STORAGE_URL`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`, `STORAGE_REGION`
   - `Project Settings → General` → **Reference ID** → `SUPABASE_PROJECT_REF`
   - Password de la base (la que se define al crear el proyecto) → `SUPABASE_DB_PASSWORD`
4. En **Authentication → Providers** de cada proyecto, configurar los mismos providers que hoy están en `.env` local (Google, Facebook, Apple, SMS/Twilio) con sus credenciales reales — hoy solo existen como placeholders para desarrollo local.
5. Generar un **Personal Access Token** de Supabase (Account → Access Tokens) → `SUPABASE_ACCESS_TOKEN` (usado por `deploy-migrations.yml`).
6. Correr las migraciones existentes (`supabase/migrations`) contra cada proyecto antes del primer deploy de la app (ver `deploy-migrations.yml` como referencia de comando: `supabase link` + `supabase db push`).

## Paso 3 — PAC de timbrado CFDI (sandbox vs producción)

Kipo timbra CFDI 4.0 contra un PAC autorizado por el SAT. Antes de exponer
staging a nadie:
1. Confirmar con el PAC elegido si ya se tienen credenciales de **pruebas/sandbox** (no deben generar CFDI legales reales) y credenciales de **producción**.
2. Guardar ambos sets de credenciales — se agregan como secrets/vars por entorno igual que el resto (no existe aún una variable estándar para esto en `.env.example`; agregarla cuando se integre el PAC en el backend).
3. **Nunca** apuntar staging a las credenciales de producción del PAC — el riesgo es timbrar (y cobrar/gastar folios) facturas de prueba.

## Paso 4 — DNS en kipo.com.mx

En el proveedor donde está registrado/administrado `kipo.com.mx`, agregar los registros de la tabla de abajo. Para dominios normales, Vercel te da un CNAME (`cname.vercel-dns.com`) o el A/ALIAS que indique para el apex. **Para wildcards, Vercel requiere que le delegues los nameservers** de esa zona — no basta un CNAME.

| Registro | Apunta a | Deployment en Vercel | Tipo de registro |
|---|---|---|---|
| `kipo.com.mx` (apex) / `www` | proyecto `kipo-landing` (ya en uso — solo migrar si aún no apunta a Vercel) | Production | A/ALIAS o CNAME |
| `app.kipo.com.mx` | proyecto `kipo-dashboard` | Production | CNAME |
| `*.kipo.com.mx` (wildcard, tenants prod) | proyecto `kipo-dashboard` | Production | Delegar nameservers de `kipo.com.mx` a Vercel — **impacta también a la landing**, evaluarlo con cuidado cuando llegue el momento (no es parte de este runbook de staging) |
| `staging.kipo.com.mx` | proyecto `kipo-dashboard` | Preview (alias manual, ver arriba) | CNAME |
| `*.staging.kipo.com.mx` (wildcard, tenants staging) | proyecto `kipo-dashboard` | Preview (alias manual) | Delegar nameservers **solo de la subzona `staging.kipo.com.mx`** (registro NS en el proveedor actual apuntando a los nameservers de Vercel) — así no se toca el DNS del resto de `kipo.com.mx` |
| `api.kipo.com.mx` | proyecto `kipo-platform` | Production | CNAME |
| `staging-api.kipo.com.mx` | proyecto `kipo-platform` | Preview (alias manual) | CNAME |

Agregar el domain primero en el proyecto de Vercel (Project → Settings →
Domains), copiar el valor exacto que pide (CNAME o nameservers), y recién
después crear el registro en el proveedor DNS. Los wildcards son el
mecanismo para los subdominios por negocio (`keeb-studio.kipo.com.mx`,
`keeb-studio.staging.kipo.com.mx`) — la app lee el subdominio en runtime
para resolver el tenant, no hace falta un registro DNS por cliente.

## Paso 5 — GitHub Secrets y Environments

Usar **GitHub Environments** (`kipo-platform` → Settings → Environments) para separar `Staging` y `Production`, en vez de secrets planos a nivel repo — así el mismo nombre de secret (`DATABASE_URL`, `VERCEL_PROJECT_ID_BACKEND`, etc.) tiene un valor distinto según el entorno del job. **Los nombres deben ser exactamente `Staging` y `Production`** (con mayúscula) — es como ya están creados en GitHub, y el workflow_dispatch de los 3 workflows abajo manda ese mismo valor (`inputs.environment`) al campo `environment:` del job, que hace match exacto y sensible a mayúsculas contra el nombre del Environment.

Nota: `VERCEL_PROJECT_ID_DASHBOARD` y `VERCEL_PROJECT_ID_BACKEND` **no
cambian entre entornos** (mismo proyecto para prod y staging) — lo que
distingue el deploy es si se manda con `--prod` (producción) o sin él +
alias manual (staging, ver [Arquitectura objetivo](#arquitectura-objetivo)).

[[Variables de entorno]]