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

## Arquitectura objetivo

| App | Prod | Staging |
|---|---|---|
| Landing (Astro) | `kipo.com.mx` / `www.kipo.com.mx` | Preview de Vercel por PR (ya funciona, no requiere dominio fijo) |
| Dashboard (Next.js) | `app.kipo.com.mx` | `staging.kipo.com.mx` |
| Backend (Flask) | `api.kipo.com.mx` | `staging-api.kipo.com.mx` |
| Supabase | proyecto cloud "kipo-prod" | proyecto cloud "kipo-staging" |
| PAC (timbrado CFDI) | credenciales de producción | credenciales de pruebas/sandbox del PAC |

Cada app sigue siendo **un proyecto de Vercel por app**, pero con un segundo
proyecto hermano para staging (`kipo-dashboard` + `kipo-dashboard-staging`,
`kipo-platform` + `kipo-platform-staging`). Es más simple de operar que usar
Custom Environments dentro de un solo proyecto, y es el patrón más parecido a
lo que ya hacen los workflows actuales.

---

## Paso 1 — Cuenta de Vercel

1. Crear cuenta / equipo (Team) en Vercel para la organización.
2. Generar un **token de acceso** (Account Settings → Tokens) — se usará como `VERCEL_TOKEN` en GitHub.
3. Anotar el `Team ID` (Org ID) — se usará como `VERCEL_ORG_ID`.
4. Crear 5 proyectos vacíos (se pueden crear al hacer el primer deploy con `vercel link`, o desde el dashboard):
   - `kipo-landing`
   - `kipo-dashboard`
   - `kipo-dashboard-staging`
   - `kipo-platform` (backend prod)
   - `kipo-platform-staging` (backend staging)
5. Anotar el **Project ID** de cada uno (Project → Settings → General) — son los `VERCEL_PROJECT_ID_*` que ya esperan los workflows, más los `_STAGING` nuevos.

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

| Registro | Apunta a |
|---|---|
| `kipo.com.mx` (apex) / `www` | proyecto `kipo-landing` (ya en uso — solo migrar si aún no apunta a Vercel) |
| `app.kipo.com.mx` | proyecto `kipo-dashboard` |
| `staging.kipo.com.mx` | proyecto `kipo-dashboard-staging` |
| `api.kipo.com.mx` | proyecto `kipo-platform` |
| `staging-api.kipo.com.mx` | proyecto `kipo-platform-staging` |

Agregar el custom domain primero en el proyecto de Vercel (Project → Settings → Domains), copiar el valor exacto que pide, y recién después crear el registro DNS.

## Paso 5 — GitHub Secrets y Environments

Usar **GitHub Environments** (`kipo-platform` → Settings → Environments) para separar `staging` y `production`, en vez de secrets planos a nivel repo — así el mismo nombre de secret (`DATABASE_URL`, `VERCEL_PROJECT_ID_BACKEND`, etc.) tiene un valor distinto según el entorno del job.

1. Crear environment `production` y `staging` en el repo.
2. En cada uno, cargar (Secrets o Variables según ya distinguen los workflows actuales):

   | Nombre | Tipo | Prod | Staging |
   |---|---|---|---|
   | `VERCEL_TOKEN` | secret | mismo token, o uno por entorno | |
   | `VERCEL_ORG_ID` | secret | mismo | |
   | `VERCEL_PROJECT_ID_DASHBOARD` | secret | proyecto `kipo-dashboard` | proyecto `kipo-dashboard-staging` |
   | `VERCEL_PROJECT_ID_BACKEND` | secret | proyecto `kipo-platform` | proyecto `kipo-platform-staging` |
   | `SUPABASE_PROJECT_REF` | secret | ref de `kipo-prod` | ref de `kipo-staging` |
   | `SUPABASE_DB_PASSWORD` | secret | password de `kipo-prod` | password de `kipo-staging` |
   | `SUPABASE_ACCESS_TOKEN` | secret | personal access token de Supabase | |
   | `DATABASE_URL` | secret | connection string `kipo-prod` | connection string `kipo-staging` |
   | `AUTH_KEY_SECRET` | secret | service role `kipo-prod` | service role `kipo-staging` |
   | `STORAGE_ACCESS_KEY` / `STORAGE_SECRET_KEY` | secret | de `kipo-prod` | de `kipo-staging` |
   | `PROJECT_URL` | variable | URL `kipo-prod` | URL `kipo-staging` |
   | `AUTH_KEY_PUBLISHABLE` | variable | anon key `kipo-prod` | anon key `kipo-staging` |
   | `STORAGE_URL` / `STORAGE_REGION` | variable | de `kipo-prod` | de `kipo-staging` |
   | `CORS_EXTRA_ORIGINS` / `CORS_WILDCARD_DOMAIN` | variable | `app.kipo.com.mx` | `staging.kipo.com.mx` |
   | `NEXT_PUBLIC_API_URL` | variable | `https://api.kipo.com.mx` | `https://staging-api.kipo.com.mx` |
   | `NEXT_PUBLIC_APP_DOMAIN` | variable | `app.kipo.com.mx` | `staging.kipo.com.mx` |

3. **Pendiente de código** (no es solo infraestructura, requiere tocar los workflows): hoy `deploy-backend.yml` y `deploy-dashboard.yml` no tienen `environment:` ni una rama/branch que dispare staging automáticamente — solo `workflow_dispatch` manual contra prod. Para que "staging" sea usable, hay que:
   - Agregar un input (`environment: staging|production`) al `workflow_dispatch`, o
   - Disparar por rama (`develop`/`staging` → staging, `main` → prod), similar al patrón que ya usa `deploy-landing.yml`.

   Esto es trabajo de implementación aparte; este documento cubre solo la infraestructura a preparar antes de ese cambio.

## Orden recomendado para hacerlo por primera vez

1. Vercel: cuenta + token + 5 proyectos vacíos (Paso 1).
2. Supabase: 2 proyectos + migraciones aplicadas (Paso 2).
3. GitHub Environments + secrets/vars cargados (Paso 5).
4. Deploy manual de staging primero (`workflow_dispatch` apuntando a los IDs de staging) para validar que todo conecta antes de tocar DNS.
5. DNS de subdominios de staging (Paso 4) para poder compartir el link con el equipo.
6. Repetir deploy contra prod, y solo al final mover/confirmar el DNS de `kipo.com.mx` / `app.kipo.com.mx` / `api.kipo.com.mx`.
7. PAC: confirmar sandbox antes de que cualquiera pruebe timbrado en staging (Paso 3).
