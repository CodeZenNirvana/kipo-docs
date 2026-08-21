Cómo está estructurado el backend (`src/apps/kipo-platform`, `name: kipo-api`):
**Flask + Python**, gestionado con **`uv`**, organizado con **Screaming
Architecture** por dominio, **Clean Architecture** en capas y **DDD diluido** —
en estilo **funcional**, sin clases de servicio. Parte de [[Arquitectura del
Monorepo]]. Es el espejo backend de [[Arquitectura del Frontend]].

- [[#La estructura grita el dominio|La estructura grita el dominio]]
- [[#Anatomía de un dominio|Anatomía de un dominio]]
- [[#El flujo — command → execute → operation → repository|El flujo — command → execute → operation → repository]]
- [[#Value objects|Value objects]]
- [[#Capa API|Capa API]]
- [[#shared y providers|shared y providers]]
- [[#Ejecución|Ejecución]]

---

## La estructura grita el dominio

Bajo la raíz del backend, los directorios de primer nivel **son los dominios**,
no capas técnicas. **No hay sufijos** `_service`, `_model`, `_repository` a nivel
carpeta ni un `utils.py` cajón-de-sastre: la estructura grita *qué hace el
negocio*.

```
src/apps/kipo-platform/
  auth/         customer/     invoice/      emisor/
  tenant/       tenant_addons/ subscriptions/ stamp_packs/
  catalog/      api/          shared/       tests/
  app.py  run.py  pyproject.toml
```

---

## Anatomía de un dominio

Cada dominio es **plano** —los archivos raíz son el corazón del dominio; solo
`operations/`, `infrastructure/` y (a veces) `value_objects/` son subcarpetas.
Ejemplo `invoice/`:

```
invoice/
  invoice.py            ← Entidad de dominio (inmutable)
  invoice_concept.py    ← Entidad hija (concepto/partida)
  commands.py           ← Commands y Queries (dataclasses frozen)
  execute.py            ← Dispatcher: enruta command → operation
  repository.py         ← Interfaz IInvoiceRepository (ABC)
  operations/           ← Una acción de negocio por archivo
    create.py  list.py  cancel.py  delete.py  stamp.py
    dashboard_stats.py  billing_activity.py
  infrastructure/
    supabase_repository.py  ← Implementación concreta del repo
```

Capas Clean, de adentro hacia afuera:

- **Dominio**: `invoice.py`, `invoice_concept.py`, `value_objects/` —entidades
  inmutables + reglas puras.
- **Aplicación**: `commands.py`, `execute.py`, `operations/` —casos de uso.
- **Infraestructura**: `infrastructure/supabase_repository.py` —implementa la
  interfaz del dominio contra Supabase/Postgres.

El dominio define la **interfaz** del repositorio; la infraestructura la
implementa. Las operaciones dependen de la interfaz, nunca de Supabase directo.

---

## El flujo — command → execute → operation → repository

El backend evita clases de servicio. Un caso de uso es una **función pura** que
recibe el repo (inyectado) más datos primitivos. La orquestación es un
**dispatcher con pattern matching**.

**1. Command / Query** — intención inmutable (`@dataclass(frozen=True)`):

```python
# invoice/commands.py
@dataclass(frozen=True)
class CreateInvoiceCommand:
    schema_name: str
    voucher_type: str
    # …
    receiver: dict
    concepts: list[dict]
```

**2. `execute` (dispatcher)** — hace `match` del command y delega a la operation,
resolviendo los repos vía `providers`:

```python
# invoice/execute.py
@log_operation("command")
def execute(command: Any) -> Any:
    repo = get_invoice_repo()
    match command:
        case CreateInvoiceCommand(schema_name, ..., receiver, concepts):
            return create.execute(repo, get_emisor_repo(), schema_name, ...)
        case ListInvoicesQuery(schema_name, limit, offset, history_months):
            return list_.execute(repo, schema_name, limit, offset, history_months)
        case _:
            raise BusinessRuleViolation(f"Comando desconocido: {type(command).__name__}")
```

**3. Operation** — la lógica de negocio, una por archivo, recibe el repo por
parámetro (nunca lo importa) y devuelve una entidad:

```python
# invoice/operations/create.py
def execute(repo: IInvoiceRepository, emisor_repo: IEmisorRepository,
            schema_name: str, ...) -> Invoice:
    folio_num, series = emisor_repo.next_folio(schema_name)
    # cálculo de subtotal, IVA, total, construcción de la entidad Invoice…
    return repo.save(invoice, schema_name)
```

**4. Repository** — interfaz en el dominio, implementación en infraestructura:

```python
# invoice/repository.py
class IInvoiceRepository(ABC):
    @abstractmethod
    def save(self, invoice: Invoice, schema_name: str) -> Invoice: ...
    @abstractmethod
    def find_all(self, schema_name, limit, offset, since=None) -> list[Invoice]: ...
    # …
```

> **Multi-tenant por schema**: casi todas las firmas cargan `schema_name`. Cada
> tenant es un schema de Postgres; el repo opera contra ese schema. El
> `schema_name` se resuelve desde el tenant del usuario autenticado en la capa
> API (ver abajo).

---

## Value objects

Los dominios con validación de valores usan `value_objects/`: subclases de `str`
que validan en `__new__` y lanzan `BusinessRuleViolation` si el valor es
inválido. Encapsulan "qué es un valor legal" en un solo lugar.

```python
# customer/value_objects/tax_id.py
_RFC_RE = re.compile(r"^[A-ZÑ&]{3,4}[0-9]{6}[A-Z0-9]{3}$")

class TaxId(str):
    def __new__(cls, value: str) -> "TaxId":
        v = (value or "").strip().upper()
        if not _RFC_RE.match(v):
            raise BusinessRuleViolation("El RFC debe ser válido (ej. XAXX010101000).")
        return super().__new__(cls, v)
```

Ejemplos existentes: `CustomerId`, `LegalName`, `TaxId`, `TaxRegime`,
`ZipCode`. Es el equivalente Python de los *branded types* del front.

---

## Capa API

`api/` es la **capa de entrega HTTP**, versionada y por dominio. No contiene
lógica de negocio: valida input, resuelve el tenant, arma el command y llama a
`execute`.

```
api/
  __init__.py            ← register_blueprints(app)
  v1/
    endpoints/
      health.py  profile.py  session.py  dashboard.py
      customers.py  subscriptions.py  stamp_packs.py  tenants.py  catalogs.py
      invoices/          ← endpoint por acción
        create.py  list.py  cancel.py  delete.py  stamp.py  stats.py
      emisor/
        get.py  upsert.py  csd.py  logo.py  manifiesto.py  pdf_customization.py
```

Un endpoint típico: autentica, resuelve el tenant → `schema_name`, valida,
despacha, serializa:

```python
# api/v1/endpoints/invoices/create.py
@invoices_bp.route("", methods=["POST"])
@require_auth
def create_invoice():
    tenant = get_tenant_repo().find_by_auth_id(g.user_id)
    if not tenant:
        return jsonify({"error": TENANT_NOT_FOUND}), 404
    # validación de RFC / C.P. del receptor…
    try:
        invoice = invoice_execute(CreateInvoiceCommand(
            schema_name=tenant.schema_name, ...))
        return jsonify(invoice_requester.serialize_invoice(invoice)), 201
    except BusinessRuleViolation as err:
        return jsonify({"error": str(err)}), 400
```

La autenticación y el gating por plan/permiso se aplican con **decoradores**
(`@require_auth`, y `@require_entitlement`/`@require_permission` para features de
plan; ver la doc de subscripciones y permisos).

---

## shared y providers

`shared/` es la infraestructura transversal (no un cajón de utilidades sueltas):

```
shared/
  config.py            ← config por entorno (config_mapping)
  supabase.py database.py db_admin.py   ← acceso a datos
  providers.py         ← inyección de dependencias (factories de repos)
  auth_decorators.py auth_errors.py     ← autenticación
  cors.py request_logging.py logger.py  ← middleware / observabilidad
  payment_gateway.py exceptions.py api_errors.py
  infrastructure/
    stripe_gateway.py            ← IPaymentGateway → Stripe
    facturapi_pac_client.py      ← cliente del PAC de timbrado CFDI
    facturapi_pdf_options_mapper.py
```

**`providers.py`** es el contenedor de DI: factories `get_xxx_repo()` que cachean
la instancia en el objeto `g` de Flask (una por request) y devuelven la
implementación concreta (`Supabase*Repository`, `StripePaymentGateway`) detrás de
la interfaz del dominio. Es donde se "cablea" infraestructura a los contratos.

```python
def get_invoice_repo() -> SupabaseInvoiceRepository:
    if "invoice_repo" not in g:
        g.invoice_repo = SupabaseInvoiceRepository(supabase.get_client())
    return g.invoice_repo
```

---

## Ejecución

- **App factory**: `app.py` → `create_app()` construye Flask, carga config por
  `FLASK_ENV`, inicializa logging/CORS/request-logging y registra los blueprints
  (`register_blueprints`).
- **Dev**: `pnpm dev:api` → `uv run flask run --debug --port 8000`.
- **Deps Python**: `pyproject.toml` + `uv.lock` (no `package.json`); el
  `package.json` solo envuelve comandos `uv` para integrarse a pnpm.
- **Lint/test**: `uv run ruff check .` · `uv run pytest` (tests en `tests/`).

El detalle de despliegue (staging vs prod, dominios, Supabase, PAC, secrets)
está en [[Infraestructura]] y [[Variables de entorno]].
