# Probar pagos de Stripe en local

Guía para testear el flujo completo de checkout con Stripe Elements en desarrollo local — sin cobros reales.

---
- [[#Requisitos previos|Requisitos previos]]
- [[#Setup: 5 terminales|Setup: 5 terminales]]
	- [[#Setup: 5 terminales#Webhook secrets temporales|Webhook secrets temporales]]
- [[#Variables de entorno necesarias|Variables de entorno necesarias]]
	- [[#Variables de entorno necesarias#`src/apps/kipo-dashboard/.env.local`|`src/apps/kipo-dashboard/.env.local`]]
	- [[#Variables de entorno necesarias#`src/apps/kipo-platform/.env`|`src/apps/kipo-platform/.env`]]
- [[#Flujo 1: Comprar una suscripción|Flujo 1: Comprar una suscripción]]
- [[#Flujo 2: Comprar paquete de timbres|Flujo 2: Comprar paquete de timbres]]
- [[#Tarjetas de prueba|Tarjetas de prueba]]
- [[#Ver eventos en tiempo real|Ver eventos en tiempo real]]
- [[#Troubleshooting|Troubleshooting]]
	- [[#Troubleshooting#El `PaymentElement` no aparece|El `PaymentElement` no aparece]]
	- [[#Troubleshooting#Error 400/500 al crear el payment intent|Error 400/500 al crear el payment intent]]
	- [[#Troubleshooting#El webhook no llega (Terminal 4 o 5 no muestra eventos)|El webhook no llega (Terminal 4 o 5 no muestra eventos)]]
	- [[#Troubleshooting#Pago 3DS se queda colgado|Pago 3DS se queda colgado]]

## Requisitos previos

Instala la Stripe CLI si no la tienes:

```bash
brew install stripe/stripe-cli/stripe
```

Autentícate con tu cuenta de Stripe (solo la primera vez):

```bash
stripe login
```

---

## Setup: 5 terminales

El stack completo requiere cinco procesos corriendo en paralelo. Suscripciones y timbres tienen endpoints de webhook separados, por lo que necesitan dos túneles de Stripe.

```bash
# Terminal 1 — Supabase (Docker debe estar corriendo)
supabase start

# Terminal 2 — Backend Flask
pnpm dev:api

# Terminal 3 — Dashboard Next.js
pnpm dev:dashboard

# Terminal 4 — Stripe webhook tunnel: suscripciones
stripe listen \
  --events customer.subscription.created,customer.subscription.updated,customer.subscription.deleted,invoice.payment_failed \
  --forward-to localhost:8000/api/v1/subscriptions/webhook

# Terminal 5 — Stripe webhook tunnel: timbres
stripe listen \
  --events checkout.session.completed \
  --forward-to localhost:8000/api/v1/stamp-packs/webhook
```

### Webhook secrets temporales

Al iniciar cada `stripe listen`, imprime una línea como esta:

```
> Ready! Your webhook signing secret is whsec_abc123...
```

Ambos túneles generan el **mismo** `whsec_` (ligado a la cuenta, no al proceso). Cópialo en el `.env` del backend:

```env
# src/apps/kipo-platform/.env
STRIPE_WEBHOOK_SECRET=whsec_abc123...       # de Terminal 4
STRIPE_STAMP_WEBHOOK_SECRET=whsec_abc123... # de Terminal 5 (mismo valor)
```

Reinicia el backend (`Ctrl+C` en Terminal 2 y `pnpm dev:api` de nuevo) para que tome el nuevo valor.

> **Nota:** `stripe listen` genera el mismo `whsec_` cada vez que lo corres con la misma cuenta. No tienes que actualizarlo en cada sesión.

---

## Variables de entorno necesarias

Antes de probar, confirma que tengas estos archivos:

### `src/apps/kipo-dashboard/.env.local`

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_APP_DOMAIN=localhost
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
```

### `src/apps/kipo-platform/.env`

```env
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...          # de stripe listen
STRIPE_STAMP_WEBHOOK_SECRET=whsec_...   # de stripe listen
STRIPE_PRICE_EMPRENDEDOR=price_1TxaieEC4QZg1nWpGOXEX67e
STRIPE_PRICE_PYME=price_1TxajIEC4QZg1nWpbT3mU8iB
```

---

## Flujo 1: Comprar una suscripción

1. Entra al dashboard en `http://<tu-slug>.localhost:3000`
2. Ve a **Configuración** → sección de plan
3. Haz clic en **Mejorar plan**
4. Selecciona **Emprendedor** o **PyME**
5. Aparece el formulario de pago con el `PaymentElement` de Stripe
6. Ingresa la tarjeta de prueba (ver abajo)
7. Haz clic en **Suscribirme**
8. El pago se confirma sin redirigir a Stripe

**Qué sucede en el backend:**
- Se crea una suscripción en Stripe con estado `incomplete`
- Al confirmar el pago, Stripe dispara `customer.subscription.updated` (estado `active`)
- El webhook llega vía Terminal 4 al endpoint `/api/v1/subscriptions/webhook`
- El backend actualiza el plan del tenant en Supabase

---

## Flujo 2: Comprar paquete de timbres

1. Entra al dashboard
2. Ve a **Configuración** → sección de timbres
3. Haz clic en **Comprar timbres**
4. Selecciona un paquete (25, 100 o 200 timbres)
5. Haz clic en **Comprar X timbres**
6. Aparece el formulario de pago
7. Ingresa la tarjeta de prueba
8. Haz clic en **Pagar**

**Qué sucede en el backend:**
- Se crea un `PaymentIntent` en Stripe con los metadatos del pack (`tenant_id`, `pack_id`, `qty`)
- Al confirmar, Stripe dispara `checkout.session.completed`
- El webhook llega vía Terminal 5 al endpoint `/api/v1/stamp-packs/webhook`
- El backend acredita los timbres al tenant en Supabase

---

## Tarjetas de prueba

| Tarjeta | Comportamiento |
|---|---|
| `4242 4242 4242 4242` | Pago exitoso sin fricción |
| `4000 0025 0000 3155` | Requiere autenticación 3D Secure |
| `4000 0000 0000 9995` | Pago rechazado (fondos insuficientes) |
| `4000 0000 0000 0002` | Pago rechazado (tarjeta declinada) |

Para todas: fecha de expiración `12/34`, CVC `123`, código postal `12345`.

---

## Ver eventos en tiempo real

Con `stripe listen` activo, cada evento que llega al backend se imprime en la Terminal 4:

```
2024-01-15 10:23:45   --> customer.subscription.updated [evt_1abc...]
2024-01-15 10:23:45  <-- [200] POST http://localhost:8000/api/v1/subscriptions/webhook
```

Para disparar eventos manualmente sin hacer un pago real:

```bash
# Simular suscripción activada
stripe trigger customer.subscription.updated

# Simular pago de timbres exitoso
stripe trigger checkout.session.completed
```

---

## Troubleshooting

### El `PaymentElement` no aparece

- Verifica que `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` esté en `.env.local` y que el dev server fue reiniciado.
- Abre la consola del browser — si hay error de Stripe será visible ahí.

### Error 400/500 al crear el payment intent

- El backend no tiene `STRIPE_SECRET_KEY` o `STRIPE_PRICE_EMPRENDEDOR`/`STRIPE_PRICE_PYME` en `.env`.
- Revisa los logs del backend en Terminal 2.

### El webhook no llega (Terminal 4 o 5 no muestra eventos)

- `stripe listen` no está corriendo o se cayó — reinicia la terminal correspondiente (Terminal 4 para suscripciones, Terminal 5 para timbres).
- `STRIPE_WEBHOOK_SECRET` o `STRIPE_STAMP_WEBHOOK_SECRET` en `.env` no coincide con el `whsec_` que imprimió `stripe listen` — cópialos de nuevo.

### Pago 3DS se queda colgado

Con `redirect: 'if_required'` (configuración actual), Stripe abre un popup para 3DS. En test mode usa la tarjeta `4000 0025 0000 3155` y en el popup haz clic en **Complete authentication**.
