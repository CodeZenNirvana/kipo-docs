# Variables de entorno

Referencia completa de todas las variables que necesita el stack. Los valores cambian por entorno (local, staging, prod) pero los nombres y su función son siempre los mismos.

---

## Dashboard — `src/apps/kipo-dashboard/.env.local`

| Variable | Para qué | Local | Staging | Prod |
|---|---|---|---|---|
| `NEXT_PUBLIC_API_URL` | URL base del backend que consume el dashboard | `http://localhost:8000` | `https://staging-api.kipo.com.mx` | `https://api.kipo.com.mx` |
| `NEXT_PUBLIC_APP_DOMAIN` | Dominio base para routing por subdominio de tenant | `localhost` | `staging.kipo.com.mx` | `kipo.com.mx` |

> **`NEXT_PUBLIC_APP_DOMAIN` es obligatoria.** Sin ella el login redirige a una URL rota (`slug.:puerto/dashboard`).  
> **Reiniciar el dev server** después de cambiar `.env.local` — Next.js no hot-reloads este archivo.

---

## Backend — `src/apps/kipo-platform/.env`

### Supabase

| Variable | Para qué |
|---|---|
| `PROJECT_URL` | URL del proyecto Supabase. El backend la usa para llamar a GoTrue (auth) y Storage |
| `DATABASE_URL` | Connection string de PostgreSQL. Formato: `postgresql://user:pass@host:port/db` |
| `AUTH_KEY_PUBLISHABLE` | Clave `anon` / `publishable` de Supabase. Para operaciones autenticadas como el usuario |
| `AUTH_KEY_SECRET` | Clave `service_role` / `secret` de Supabase. Para operaciones administrativas (bypass RLS) |
| `STORAGE_URL` | URL del endpoint S3-compatible de Supabase Storage |
| `STORAGE_ACCESS_KEY` | Access key S3 de Storage |
| `STORAGE_SECRET_KEY` | Secret key S3 de Storage |
| `STORAGE_REGION` | Región de Storage (`local` en dev, la región real en prod) |

**En local** estos valores los da `supabase status` una vez levantado Docker:

```bash
supabase status
# Copia los valores que imprime a tu .env
```

Valores típicos de Supabase local:

```env
PROJECT_URL=http://127.0.0.1:54321
DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:54322/postgres
AUTH_KEY_PUBLISHABLE=<anon key>
AUTH_KEY_SECRET=<service_role key>
STORAGE_URL=http://127.0.0.1:54321/storage/v1/s3
STORAGE_ACCESS_KEY=<S3 Access Key>
STORAGE_SECRET_KEY=<S3 Secret Key>
STORAGE_REGION=local
```

---

### Stripe (pagos y suscripciones)

| Variable | Para qué |
|---|---|
| `STRIPE_SECRET_KEY` | Clave secreta de Stripe para crear sesiones de checkout y consultar suscripciones |
| `STRIPE_WEBHOOK_SECRET` | Signing secret del webhook de Stripe para eventos de suscripción (upgrades, cancelaciones) |
| `STRIPE_STAMP_WEBHOOK_SECRET` | Signing secret del webhook de Stripe para eventos de compra de timbres |
| `STRIPE_PRICE_EMPRENDEDOR` | Price ID en Stripe del plan Emprendedor (`price_xxx`) |
| `STRIPE_PRICE_PYME` | Price ID en Stripe del plan PyME (`price_xxx`) |

En local puedes usar las claves de **test mode** de Stripe (`sk_test_...`) — no cobra real.  
En staging, también test mode. En prod, claves live (`sk_live_...`).

---

### CORS

| Variable | Para qué |
|---|---|
| `CORS_WILDCARD_DOMAIN` | Dominio base para permitir subdominios vía regex. Ejemplo: `localhost` permite `*.localhost`; `kipo.com.mx` permite `*.kipo.com.mx` |
| `CORS_EXTRA_ORIGINS` | Orígenes adicionales específicos separados por coma. Ej: `http://localhost:3001,https://mi-dominio.com` |

`CORS_WILDCARD_DOMAIN` es la var clave para que el dashboard en subdominio pueda hablar con el backend.  
Si no está, el browser bloquea las requests con error CORS.

---

### Flask

| Variable | Para qué |
|---|---|
| `FLASK_ENV` | Modo de ejecución: `development` (debug, sin validaciones estrictas) o `production` (sin debug, exige `DATABASE_URL`). Default: `development` |

---

## Raíz — `.env` (providers de auth de Supabase)

Solo necesario para SMS real y social login. En local puedes omitirlo y usar los números de prueba con OTP fijo definidos en `supabase/config.toml` bajo `[auth.sms.test_otp]`.

| Variable | Para qué |
|---|---|
| `TWILIO_ACCOUNT_SID` | Account SID de Twilio para enviar SMS de OTP |
| `TWILIO_AUTH_TOKEN` | Auth token de Twilio |
| `TWILIO_PHONE_NUMBER` | Número de teléfono remitente en Twilio |
| `GOOGLE_CLIENT_ID` | Client ID de la app de Google para login social |
| `GOOGLE_CLIENT_SECRET` | Client secret de Google |

Este `.env` lo lee el CLI de Supabase al hacer `supabase start`, no las apps directamente. Hay que exportarlo antes:

```bash
export $(grep -v '^#' .env | xargs) && supabase start
```

---

## Resumen por entorno

| Variable | Local | Staging | Prod |
|---|---|---|---|
| `NEXT_PUBLIC_API_URL` | `http://localhost:8000` | `https://staging-api.kipo.com.mx` | `https://api.kipo.com.mx` |
| `NEXT_PUBLIC_APP_DOMAIN` | `localhost` | `staging.kipo.com.mx` | `kipo.com.mx` |
| `PROJECT_URL` | `http://127.0.0.1:54321` (Supabase local) | URL proyecto `kipo-staging` | URL proyecto `kipo-prod` |
| `DATABASE_URL` | `postgresql://postgres:postgres@127.0.0.1:54322/postgres` | Connection string `kipo-staging` | Connection string `kipo-prod` |
| `AUTH_KEY_PUBLISHABLE` | anon key Supabase local | anon key `kipo-staging` | anon key `kipo-prod` |
| `AUTH_KEY_SECRET` | service_role key Supabase local | service_role key `kipo-staging` | service_role key `kipo-prod` |
| `STORAGE_URL` | `http://127.0.0.1:54321/storage/v1/s3` | Storage `kipo-staging` | Storage `kipo-prod` |
| `STORAGE_REGION` | `local` | región real | región real |
| `STRIPE_SECRET_KEY` | `sk_test_...` | `sk_test_...` | `sk_live_...` |
| `STRIPE_PRICE_EMPRENDEDOR` | price test | price test staging | price live prod |
| `CORS_WILDCARD_DOMAIN` | `localhost` | `staging.kipo.com.mx` | `kipo.com.mx` |
| `FLASK_ENV` | `development` | `production` | `production` |
