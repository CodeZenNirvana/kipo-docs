# Preparar staging y prod para Kipo

Runbook para pasar de "nada provisionado" a tener **staging** y **producción**
funcionando para las 3 apps del monorepo `kipo-platform`
(`kipo-landing`, `kipo-dashboard`, `kipo-platform` backend) + Supabase + DNS.

## Punto de partida (2026-07-14)

- Dominio `kipo.com.mx` ya existe y ahí vive la landing (Astro).
- No hay cuenta/proyectos de Vercel creados todavía (no hay `.vercel/` linkeado en el repo).
- No hay proyecto de Supabase en la nube (solo Supabase local vía Docker, `project_id = "kipo-platform"` en `supabase/config.toml`).
- Ya existen 4 GitHub Actions en `kipo-platform/.github/workflows/` que **asumen un solo entorno "production"**:
  - `deploy-landing.yml` — único que ya tiene preview automático por PR (push a `main` = prod, PR = preview). Para landing, los PR previews de Vercel ya cubren el caso de "staging".
  - `deploy-dashboard.yml` — solo `workflow_dispatch`, solo prod, lee `secrets.VERCEL_PROJECT_ID_DASHBOARD` y `vars.NEXT_PUBLIC_API_URL` / `vars.NEXT_PUBLIC_APP_DOMAIN`.
  - `deploy-backend.yml` — solo `workflow_dispatch`, solo prod, lee `secrets.VERCEL_PROJECT_ID_BACKEND` + variables de Supabase/Storage/CORS.
  - `deploy-migrations.yml` — hace `supabase db push` contra `secrets.SUPABASE_PROJECT_REF` (un solo proyecto).
  - **Ninguno de los 4 tiene todavía el concepto de "staging"** — eso es trabajo de código pendiente (ver [Paso 5](#paso-5-github-secrets-y-environments)), no solo de infraestructura.
- Kipo es una plataforma de facturación CFDI 4.0 (SAT) — esto importa para el punto 4 (PAC): staging no debe timbrar contra el proveedor real.
- Kipo es multi-tenant: cada negocio tiene su propio subdominio (ej. `keeb-studio.kipo.com.mx` en prod, `keeb-studio.staging.kipo.com.mx` en staging) — esto requiere **Wildcard Domains** en Vercel.
- **Vercel: por ahora plan Hobby (gratis)**, no Pro todavía. Esto cambia cómo se arma "staging" — ver la nota debajo.

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

1. Crear environment `Production` y `Staging` en el repo (mayúscula inicial).
2. En cada uno, cargar (Secrets o Variables según ya distinguen los workflows actuales):

   | Nombre | Tipo | Prod | Staging |
   |---|---|---|---|
   | `VERCEL_TOKEN` | secret | mismo token, o uno por entorno | |
   | `VERCEL_ORG_ID` | secret | mismo | |
   | `VERCEL_PROJECT_ID_DASHBOARD` | secret | proyecto `kipo-dashboard` (mismo en ambos entornos) | |
   | `VERCEL_PROJECT_ID_BACKEND` | secret | proyecto `kipo-platform` (mismo en ambos entornos) | |
   | `SUPABASE_PROJECT_REF` | secret | ref de `kipo-prod` | ref de `kipo-staging` |
   | `SUPABASE_DB_PASSWORD` | secret | password de `kipo-prod` | password de `kipo-staging` |
   | `SUPABASE_ACCESS_TOKEN` | secret | personal access token de Supabase | |
   | `DATABASE_URL` | secret | connection string `kipo-prod` | connection string `kipo-staging` |
   | `AUTH_KEY_SECRET` | secret | service role `kipo-prod` | service role `kipo-staging` |
   | `STORAGE_ACCESS_KEY` / `STORAGE_SECRET_KEY` | secret | de `kipo-prod` | de `kipo-staging` |
   | `PROJECT_URL` | variable | URL `kipo-prod` | URL `kipo-staging` |
   | `AUTH_KEY_PUBLISHABLE` | variable | anon key `kipo-prod` | anon key `kipo-staging` |
   | `STORAGE_URL` / `STORAGE_REGION` | variable | de `kipo-prod` | de `kipo-staging` |
   | `CORS_EXTRA_ORIGINS` / `CORS_WILDCARD_DOMAIN` | variable | `*.kipo.com.mx` | `*.staging.kipo.com.mx` |
   | `NEXT_PUBLIC_API_URL` | variable | `https://api.kipo.com.mx` | `https://staging-api.kipo.com.mx` |
   | `NEXT_PUBLIC_APP_DOMAIN` | variable | `kipo.com.mx` | `staging.kipo.com.mx` |

   Como no hay Custom Environments en Hobby, estas variables **no se
   pueden scopear dentro de Vercel** por entorno nombrado — hay que
   seguir pasándolas como `--env`/`--build-env` en el propio comando de
   `vercel deploy` (igual que ya hacen `deploy-backend.yml` y
   `deploy-dashboard.yml` hoy para prod), leyéndolas del GitHub
   Environment correspondiente.

3. **Ya implementado** en los 3 workflows (`deploy-backend.yml`, `deploy-dashboard.yml`, `deploy-migrations.yml`): todos tienen ahora un input `environment` (`type: choice`, opciones `Production`/`Staging`, default `Production`) en su `workflow_dispatch`, y el job usa `environment: ${{ inputs.environment }}` para que el secret/variable correcto se resuelva según el Environment de GitHub elegido:
   - `deploy-backend.yml` / `deploy-dashboard.yml`: si `Production`, corre `vercel deploy --prod ...` (sin cambios respecto a antes); si `Staging`, corre `vercel deploy ...` (sin `--prod`, genera un Preview) y encadena los `vercel alias` (dominio fijo + wildcard en dashboard) descritos en [Arquitectura objetivo](#arquitectura-objetivo).
   - `deploy-migrations.yml`: mismo input, solo cambia a qué proyecto de Supabase (`Staging` o `Production`) se le hace `db push` — sigue siendo `workflow_dispatch` únicamente, sin trigger automático.
   - Para `Production`, el job sigue exigiendo `github.ref == 'refs/heads/main'`; `Staging` puede correr desde cualquier rama.
   - Pendiente real: en `deploy-backend.yml`, el paso de staging manda `FLASK_ENV=production` (no `staging`) a propósito — `shared/config.py` (`config_mapping`) todavía no tiene una entrada `"staging"`, solo `development`/`production`/`testing`, así que mandar `staging` ahí tronaría el backend con `KeyError`. Si más adelante se agrega esa entrada en el código del backend, este valor se puede cambiar a `staging`.

## Orden recomendado para hacerlo por primera vez

1. Vercel: cuenta Hobby + token + 3 proyectos vacíos (Paso 1).
2. Supabase: 2 proyectos + migraciones aplicadas (Paso 2).
3. GitHub Environments (`Production`/`Staging`, mayúscula) + secrets/vars cargados (Paso 5).
4. Los 3 workflows ya soportan `environment: Production|Staging` (Paso 5.3, implementado) — falta solo cargar los secrets/vars del punto anterior para que funcionen de punta a punta.
5. Deploy manual de staging primero (`workflow_dispatch` con `environment: Staging`) para validar que el alias conecta antes de tocar DNS del wildcard.
6. DNS de staging: CNAME de `staging.kipo.com.mx` + delegación de nameservers solo de esa subzona para el wildcard `*.staging.kipo.com.mx` (Paso 4).
7. Repetir deploy contra prod, y solo al final confirmar el DNS de `kipo.com.mx` / `app.kipo.com.mx` / `api.kipo.com.mx` (el wildcard `*.kipo.com.mx` de prod, al requerir delegar nameservers de todo el dominio, se evalúa aparte por su impacto en la landing).
8. PAC: confirmar sandbox antes de que cualquiera pruebe timbrado en staging (Paso 3).

## Apéndice — Si más adelante suben a Vercel Pro

Con Pro se puede simplificar el mecanismo de staging:
- Crear un **Custom Environment** `staging` en `kipo-dashboard` y `kipo-platform` (Project → Settings → Environments).
- Asignarle el dominio (`staging.kipo.com.mx` + wildcard) directo desde Vercel, sin `vercel alias` manual — cada deploy a ese entorno actualiza el dominio solo.
- Cambiar el comando de deploy de `vercel deploy` (sin flag) a `vercel deploy --target staging`, y las env vars de staging se pueden mover de GitHub a Project → Settings → Environment Variables scopeadas al Custom Environment.
- El resto del runbook (Supabase, PAC, GitHub Environments para secrets que no viven en Vercel) no cambia.
