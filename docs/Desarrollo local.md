# Correr Kipo en local

Guía para levantar el stack completo en tu máquina.

## Requisitos

| Herramienta | Versión mínima | Instalación |
|---|---|---|
| Node.js | >= 22 | [nodejs.org](https://nodejs.org) |
| pnpm | >= 10 | `npm install -g pnpm` |
| Python | >= 3.14 | [python.org](https://python.org) |
| uv | cualquiera | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Supabase CLI | cualquiera | `brew install supabase/tap/supabase` |
| Docker Desktop | cualquiera | [docker.com](https://www.docker.com/products/docker-desktop/) — debe estar corriendo |

---

## 1. Instalar dependencias

```bash
pnpm install        # dependencias de todos los workspaces (Node)
pnpm install:py     # dependencias de Python del backend (uv sync)
```

---

## 2. Variables de entorno

El proyecto usa tres archivos de variables de entorno distintos. Ninguno está en el repo — créalos manualmente.

### 2a. Dashboard — `src/apps/kipo-dashboard/.env.local`

Todas las variables de entorno las puedes encontrar en [[Variables de entorno]]

### 2b. Backend — `src/apps/kipo-platform/.env`

Primero levanta Supabase (paso 3) para obtener los valores de conexión, revisa [[Variables de entorno]]

### 2c. Raíz — `.env` (solo para providers de auth de Supabase)

Solo necesario si quieres SMS real o social login. Para desarrollo local con
los números de prueba del SAT y OTP fijo definidos en `supabase/config.toml`
puedes dejarlo vacío o no crearlo.

```env
# SMS (Twilio) — opcional en local
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# Social login — opcional en local
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
```

---

## 3. Levantar Supabase local

Supabase levanta contenedores Docker. Si hay un `.env` en la raíz, expórtalo primero:

```bash
# Con providers de auth configurados
export $(grep -v '^#' .env | xargs) && supabase start

# Sin providers (desarrollo básico)
supabase start
```

`supabase start` imprime las URLs y keys — cópialas al `.env` del backend (paso 2b).

Referencias útiles:
- Studio local: http://127.0.0.1:54323
- Mailpit (emails de auth): http://127.0.0.1:54324
- `supabase status` — para ver las keys de nuevo sin reiniciar
- `supabase stop` — apagar contenedores


---

## 4. Stripe CLI (para probar pagos)

Si vas a probar el flujo de checkout (suscripciones o timbres), necesitas la Stripe CLI para tunnelear webhooks a tu backend local:

```bash
brew install stripe/stripe-cli/stripe
stripe login   # solo la primera vez
```

> Ver guía completa en [stripe-local-testing.md](Desarrollo%20local%20(stripe).md).

---

## 5. Levantar las apps

Desde la raíz del repo, todo junto:

```bash
pnpm dev
```

O por separado:

```bash
pnpm dev:api        # Backend Flask en http://localhost:8000
pnpm dev:dashboard  # Dashboard Next.js en http://localhost:3000
pnpm dev:landing    # Landing Astro
```

> Si vas a probar pagos, corre también `stripe listen` en una terminal adicional (ver sección 4).

### Acceder al dashboard de un tenant

El dashboard usa subdominios por tenant. Ejemplo: si tu tenant tiene slug `mi-empresa`, accede en:

```
http://mi-empresa.localhost:3000
```

No hace falta configuración extra de DNS — los browsers resuelven `*.localhost` automáticamente.

---

## Scripts útiles

```bash
pnpm lint           # lint en todos los workspaces
pnpm typecheck      # typecheck en todos los workspaces
pnpm build          # build de packages + apps
pnpm storybook      # Storybook del design system (@kipo/ui-react)
```

Backend (desde `src/apps/kipo-platform` o vía filtro pnpm):

```bash
uv run pytest       # tests
uv run ruff check . # lint
```

---

## Troubleshooting

### Login redirige a `http://slug.:3000/dashboard` (hostname roto)

`NEXT_PUBLIC_APP_DOMAIN` no está seteado o el dev server no fue reiniciado después de agregarlo.

1. Verifica `src/apps/kipo-dashboard/.env.local` contiene `NEXT_PUBLIC_APP_DOMAIN=localhost`.
2. Reinicia el dev server (`Ctrl+C` y `pnpm dev:dashboard` de nuevo).

### Error CORS al hacer requests al backend desde el subdominio

`CORS_WILDCARD_DOMAIN` no está configurado en el backend.

1. Verifica `src/apps/kipo-platform/.env` contiene `CORS_WILDCARD_DOMAIN=localhost`.
2. Reinicia el backend (`Ctrl+C` y `pnpm dev:api` de nuevo).

### `supabase start` falla

- Docker Desktop no está corriendo — ábrelo primero.
- Puerto ocupado — revisa si ya hay contenedores corriendo con `supabase status`.
