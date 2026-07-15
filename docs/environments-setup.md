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
- Kipo es multi-tenant: cada negocio tiene su propio subdominio (ej. `keeb-studio.kipo.com.mx` en prod, `keeb-studio.staging.kipo.com.mx` en staging) — esto requiere **Wildcard Domains** en Vercel, no solo un dominio fijo por entorno.

## Arquitectura objetivo

| App | Prod | Staging |
|---|---|---|
| Landing (Astro) | `kipo.com.mx` / `www.kipo.com.mx` | Preview de Vercel por PR (ya funciona, no requiere dominio fijo) |
| Dashboard (Next.js) | `app.kipo.com.mx` + `*.kipo.com.mx` (tenants) | `staging.kipo.com.mx` + `*.staging.kipo.com.mx` (tenants) |
| Backend (Flask) | `api.kipo.com.mx` | `staging-api.kipo.com.mx` |
| Supabase | proyecto cloud "kipo-prod" | proyecto cloud "kipo-staging" |
| PAC (timbrado CFDI) | credenciales de producción | credenciales de pruebas/sandbox del PAC |

**Un solo proyecto de Vercel por app** (`kipo-dashboard`, `kipo-platform`) —
no hace falta un proyecto hermano `*-staging`. Vercel permite definir un
**Custom Environment** (ej. `staging`) dentro del mismo proyecto, con su
propio dominio (incluyendo wildcard) y su propio set de variables de
entorno, independiente de Production. Como `deploy-dashboard.yml` y
`deploy-backend.yml` no usan la integración Git de Vercel (todo corre vía
CLI en `workflow_dispatch`), no dependemos de "asignar un dominio a una
rama" — basta con pasarle el nombre del entorno al deploy:

```bash
vercel deploy --target staging --token "$VERCEL_TOKEN"
```

Esto aplica automáticamente el dominio y las env vars que se hayan
configurado para el Custom Environment `staging` en ese proyecto. Nota:
Custom Environments y Wildcard Domains son features de plan **Pro o
Enterprise** (no están en Hobby) — de cualquier forma Hobby no aplica para
un SaaS comercial como Kipo, así que esto no debería ser una sorpresa al
dar de alta la cuenta en el Paso 1.

---

## Paso 1 — Cuenta de Vercel

1. Crear cuenta / equipo (Team) en Vercel para la organización, en plan **Pro** (o Enterprise) — requerido para Wildcard Domains y Custom Environments, no disponibles en Hobby.
2. Generar un **token de acceso** (Account Settings → Tokens) — se usará como `VERCEL_TOKEN` en GitHub.
3. Anotar el `Team ID` (Org ID) — se usará como `VERCEL_ORG_ID`.
4. Crear 3 proyectos vacíos (se pueden crear al hacer el primer deploy con `vercel link`, o desde el dashboard):
   - `kipo-landing`
   - `kipo-dashboard`
   - `kipo-platform` (backend)
5. Anotar el **Project ID** de cada uno (Project → Settings → General) — son los `VERCEL_PROJECT_ID_*` que ya esperan los workflows. El mismo ID sirve para prod y staging, ya que ambos entornos viven en el mismo proyecto.
6. En `kipo-dashboard` y `kipo-platform` (Project → Settings → Environments), crear el Custom Environment `staging` junto al de `Production` ya existente.

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

En el proveedor donde está registrado/administrado `kipo.com.mx`, agregar (Vercel te da los valores exactos al agregar el "Custom Domain" en cada proyecto — normalmente CNAME a `cname.vercel-dns.com` para subdominios, o los A/ALIAS que indique Vercel para el apex):

| Registro | Apunta a | Environment en Vercel |
|---|---|---|
| `kipo.com.mx` (apex) / `www` | proyecto `kipo-landing` (ya en uso — solo migrar si aún no apunta a Vercel) | Production |
| `app.kipo.com.mx` | proyecto `kipo-dashboard` | Production |
| `*.kipo.com.mx` (wildcard, tenants) | proyecto `kipo-dashboard` | Production |
| `staging.kipo.com.mx` | proyecto `kipo-dashboard` | Custom Environment `staging` |
| `*.staging.kipo.com.mx` (wildcard, tenants) | proyecto `kipo-dashboard` | Custom Environment `staging` |
| `api.kipo.com.mx` | proyecto `kipo-platform` | Production |
| `staging-api.kipo.com.mx` | proyecto `kipo-platform` | Custom Environment `staging` |

Agregar el domain primero en el proyecto de Vercel (Project → Settings →
Domains), asignándolo explícitamente al Environment correspondiente
(Production o el Custom Environment `staging`), copiar el valor exacto que
pide, y recién después crear el registro DNS. Los wildcards (`*.kipo.com.mx`,
`*.staging.kipo.com.mx`) son el mecanismo para los subdominios por negocio
(`keeb-studio.kipo.com.mx`, `keeb-studio.staging.kipo.com.mx`) — la app lee
el subdominio en runtime para resolver el tenant, no hace falta un registro
DNS por cliente.

## Paso 5 — GitHub Secrets y Environments

Usar **GitHub Environments** (`kipo-platform` → Settings → Environments) para separar `staging` y `production`, en vez de secrets planos a nivel repo — así el mismo nombre de secret (`DATABASE_URL`, `VERCEL_PROJECT_ID_BACKEND`, etc.) tiene un valor distinto según el entorno del job.

Nota: `VERCEL_PROJECT_ID_DASHBOARD` y `VERCEL_PROJECT_ID_BACKEND` ya **no
cambian entre entornos** (mismo proyecto para prod y staging) — lo único
que distingue el deploy es el flag `--target` y, dentro de Vercel, las env
vars scopeadas al Custom Environment `staging` (configuradas directo en
Project → Settings → Environment Variables, no necesariamente como GitHub
secret). Aun así conviene mantener `staging`/`production` como GitHub
Environments para separar lo que sí cambia por entorno: la referencia a
Supabase, la URL pública, y CORS.

1. Crear environment `production` y `staging` en el repo.
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

   `DATABASE_URL`, `PROJECT_URL`, `AUTH_KEY_*`, `STORAGE_*`,
   `NEXT_PUBLIC_*` también se pueden cargar directo en Vercel (Project →
   Settings → Environment Variables, scopeadas a Production vs. Custom
   Environment `staging`) en vez de pasarlas por GitHub — Vercel las
   inyecta solas según el `--target` del deploy. Dejarlas en GitHub
   Environments tiene sentido solo si además las necesitas en el propio
   job de CI (por ejemplo, para correr las migraciones de Supabase antes
   del deploy).

3. **Pendiente de código** (no es solo infraestructura, requiere tocar los workflows): hoy `deploy-backend.yml` y `deploy-dashboard.yml` no tienen `environment:` ni una forma de disparar staging — solo `workflow_dispatch` manual contra prod, con `vercel deploy --prod` fijo. Para que "staging" sea usable, hay que:
   - Agregar un input (`environment: staging|production`) al `workflow_dispatch`.
   - Cambiar el comando de deploy para que use `--target "$ENVIRONMENT"` (con `production`/`staging`) en vez de `--prod` fijo — `vercel deploy --target production` es equivalente a `--prod`, y `--target staging` apunta al Custom Environment.
   - Seleccionar el GitHub Environment (`environment: ${{ inputs.environment }}`) del job para que tome el secret/variable correcto de Supabase/CORS/URL según el paso 2 de arriba.

   Esto es trabajo de implementación aparte; este documento cubre solo la infraestructura a preparar antes de ese cambio.

## Orden recomendado para hacerlo por primera vez

1. Vercel: cuenta Pro + token + 3 proyectos vacíos + Custom Environment `staging` en `kipo-dashboard` y `kipo-platform` (Paso 1).
2. Supabase: 2 proyectos + migraciones aplicadas (Paso 2).
3. GitHub Environments + secrets/vars cargados, y env vars scopeadas en Vercel por entorno (Paso 5).
4. Ajustar los workflows para aceptar `--target staging|production` (Paso 5.3) — sin esto, no hay forma de disparar un deploy de staging.
5. Deploy manual de staging primero (`workflow_dispatch` con `environment: staging`) para validar que todo conecta antes de tocar DNS.
6. DNS + wildcard de staging (Paso 4) para poder compartir el link con el equipo, incluyendo subdominios de tenant.
7. Repetir deploy contra prod, y solo al final mover/confirmar el DNS de `kipo.com.mx` / `app.kipo.com.mx` / `api.kipo.com.mx` / `*.kipo.com.mx`.
8. PAC: confirmar sandbox antes de que cualquiera pruebe timbrado en staging (Paso 3).
