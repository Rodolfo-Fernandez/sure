# Arquitectura de Sure

Guía en español de la arquitectura del proyecto. Complementa (no reemplaza) la
documentación técnica original en inglés:

- [Arquitectura y convenciones](llm-guides/architecture.md)
- [Guías de tareas](llm-guides/README.md)
- [Guía de desarrollo](llm-guides/development.md)
- [Guía de testing](llm-guides/testing.md)
- [Sistema de diseño](llm-guides/design-system.md)

## Qué es Sure

Sure es una aplicación de finanzas personales full-stack. Es un fork comunitario
del proyecto Maybe Finance (abandonado en 2023), mantenido bajo licencia AGPLv3.
Permite registrar cuentas bancarias, inversiones, criptomonedas, propiedades,
vehículos y deudas; sincronizar movimientos mediante proveedores externos;
importar CSV; generar reportes de patrimonio y flujo de caja; y consultar un
asistente de IA.

## Stack tecnológico

| Capa | Tecnología |
| --- | --- |
| Backend | Ruby on Rails 8.1 (Ruby 3.4.9) |
| Base de datos | PostgreSQL (>9.3) |
| Jobs / colas | Sidekiq + Redis (>5.4) |
| Frontend | Hotwire (Turbo + Stimulus), ViewComponent |
| Estilos | Tailwind CSS con tokens del sistema de diseño (`sure-design-system.css`) |
| JS/CSS linter | Biome 1.9 |
| Desktop | Tauri + Vite + TypeScript (`desktop/`) |
| Mobile | Flutter (`mobile/`) |
| API | REST `/api/v1` con OAuth (Doorkeeper) y `X-Api-Key` |
| Documentación API | rswag → `docs/api/openapi.yaml` |

## Estructura del monorepo

```
sure/
├── app/                  # Código Rails
│   ├── models/           # Lógica de dominio (models, concerns, POROs)
│   ├── controllers/      # Controladores delgados
│   ├── components/       # ViewComponents, primitivas DS::
│   ├── javascript/       # Stimulus controllers
│   ├── jobs/             # Sidekiq jobs
│   └── views/            # ERB server-rendered
├── config/               # Rutas, initializers, schedule.yml (Sidekiq)
├── db/                   # Migraciones y schema.rb
├── desktop/              # Cliente escritorio (Tauri + Vite + TS)
├── mobile/               # Cliente móvil (Flutter)
├── lib/                  # Money y librerías compartidas
├── test/                 # Minitest (espeja app/), fixtures
├── spec/                 # Solo specs rswag (documentación OpenAPI)
├── docs/                 # Documentación y guías
└── bin/                  # Scripts (setup, dev, tokens.mjs)
```

## Modelo de dominio

### Familias, usuarios y acceso

- `Family` es la raíz de tenencia: posee cuentas, usuarios, suscripciones y
  preferencias.
- `User` pertenece a una familia; `Session` pertenece a un usuario. Roles:
  guest, member, admin, super admin.
- Las cuentas pueden tener dueño opcional y reglas de compartición, por lo que
  la membresía de familia no describe por sí sola el acceso de cada usuario.
- **Regla obligatoria:** usar `Current.user` y `Current.family`, nunca
  `current_user` ni `current_family`.

### Cuentas y saldos

- `Account` usa delegación a un `accountable` polimórfico. Tipos de activo:
  `Depository`, `Investment`, `Crypto`, `Property`, `Vehicle`, `OtherAsset`.
  Tipos de pasivo: `CreditCard`, `Loan`, `OtherLiability`.
- Cada cuenta tiene su propio saldo y moneda. Los saldos diarios (`Balance`)
  registran el valor del activo o la deuda pendiente. Los `Holding` registran
  cantidad y precio de un `Security` por fecha.
- `Balance::Materializer` materializa holdings (vía `Holding::Materializer`)
  antes de calcular el historial de saldos.

### Entradas (Entry)

`Entry` delega a uno de tres tipos `Entryable`, con fecha, monto y moneda:

- `Valuation` — valor absoluto de la cuenta o deuda en una fecha.
- `Transaction` — movimiento de efectivo; puede tener categoría, comercio y
  etiquetas; las reglas pueden clasificarla.
- `Trade` — movimiento de valores con cantidad y precio (compras/ventas).

**Convención de signo (crítica):** en movimientos de efectivo, **los montos
negativos son entradas de dinero y los positivos salidas**. Una transacción
negativa en una cuenta corriente aumenta el efectivo; negativa en una tarjeta
de crédito es un pago que reduce la deuda. No aplicar signos de movimiento a
valuaciones absolutas.

### Transferencias

- `Transfer` empareja una entrada (inflow) y una salida (outflow) entre cuentas
  distintas de la misma familia. Montos en la misma moneda deben ser iguales y
  opuestos.
- El emparejamiento automático (`Family::AutoTransferMatchable`) busca en una
  ventana de cuatro días y soporta candidatos entre monedas usando tasas de
  cambio fechadas con tolerancia. Las transferencias confirmadas permiten hasta
  30 días de diferencia.
- El destino define el tipo de salida: pago de préstamo, pago de tarjeta,
  aporte a inversión o movimiento ordinario. Solo los dos primeros cuentan
  como gastos en la clasificación presupuestaria.

### Moneda y conversión

- `Money` (en `lib/money.rb`) maneja operaciones y formato monetario. Cada
  cuenta, entrada, saldo y holding retiene su moneda. La moneda preferida de la
  familia solo normaliza reportes.
- **`Money` nunca convierte automáticamente:** sumar, restar, dividir u ordenar
  montos en monedas distintas lanza `Money::CurrencyMismatchError` (en
  desarrollo y test). Convertir primero con `exchange_to(currency, date:)` en la
  fecha que corresponda, usando `ExchangeRate` y su concern `Provided`.

## Sincronización y proveedores

- El concern `Syncable` programa syncs en background; `Sync` registra estado,
  jerarquía y errores.
- `Account::Syncer` importa datos de mercado, materializa historial (reverso
  para cuentas vinculadas, directo para manuales), aplica overrides de saldos
  del proveedor y empareja transferencias.
- `Family::Syncer` programa todos los `*_items` elegibles más cuentas manuales,
  luego empareja transferencias y aplica reglas activas.
- Conexiones de proveedor (`PlaidItem` y otras) normalizan payloads mediante
  `Account::ProviderImportAdapter`; `AccountProvider` vincula cuentas internas
  con registros del proveedor.
- `Provider::Registry` registra proveedores intercambiables en runtime; las
  interfaces viven en `app/models/provider/*_concept.rb`. Los proveedores
  devuelven `Provider::Response` (`success?`, `data`, `error`) mediante
  `with_provider_response`. Los modelos eligen proveedores por sus concerns
  `Provided`, no llamando al registry directamente.
- Fallas relevantes para soporte se capturan con `DebugLogEntry.capture(...)`.

## Frontend y sistema de diseño

- Interacción vía Turbo/Stimulus con vistas server-rendered; estado en la URL.
- Tokens funcionales de `app/assets/tailwind/sure-design-system.css`
  (`bg-container`, `text-primary`, `border-primary`, `text-destructive`...).
  Prohibidos colores Tailwind crudos o hex literales.
- Usar primitivas existentes en `app/components/DS/` (alertas, badges, botones,
  dialogs, inputs). Si una forma hecha a mano aparece dos veces sin equivalente
  DS, proponer una primitiva nueva antes de duplicar.
- Iconos con el helper `icon`, nunca `lucide_icon` directo ni SVG crudo.
- Textos visibles con `t()` (i18n). Escalas de tokens en vez de píxeles
  arbitrarios.

## API

- Endpoints en `app/controllers/api/v1/`, autenticación dual: OAuth con
  Doorkeeper **y** `X-Api-Key` (claves opacas, no JWTs). El base controller
  fija el contexto actual, desactiva CSRF de sesión y aplica rate limits.
- Toda modificación de endpoints exige tests de comportamiento en Minitest y
  specs rswag **solo documentales** en `spec/requests/api/v1/`. Regenerar
  `docs/api/openapi.yaml` con
  `RAILS_ENV=test bundle exec rake rswag:specs:swaggerize`.

## Testing

- Minitest con fixtures (Mocha y VCR donde haga falta). Los tests espejan
  `app/` bajo `test/`. RSpec/rswag existe únicamente para documentación
  OpenAPI.
- Ejecutar `bin/rails test` en verde antes de pushear. Antes de abrir un PR:
  tests completos, tests de sistema aplicables, lint Ruby/ERB, Biome y
  Brakeman.

## Modos de aplicación

`Rails.application.config.app_mode` soporta `managed` y `self_hosted`. La
disponibilidad de proveedores puede diferir entre instalaciones; las variables
de entorno activan integraciones específicas (ver `.env.local.example`).

## Despliegue

- Self-hosting con Docker: [docs/hosting/docker.md](hosting/docker.md).
- One-click en Render (tres variantes: sin IA, IA simple, IA externa),
  PikaPods y Hostim — botones en el [README](../README.md).
- Clientes disponibles: navegador, app de escritorio macOS (Tauri), app móvil
  (Flutter), clientes API y agentes LLM. Ver [docs/clients.md](clients.md).

## Setup de desarrollo

```sh
cd sure
cp .env.local.example .env.local
bin/setup
bin/dev

# Opcional: datos demo
rake demo_data:default
```

Credenciales demo: `user@example.com` / `Password1!`. Requisitos: Ruby 3.4.9
(ver `.ruby-version`), PostgreSQL >9.3, Redis >5.4.

## Convenciones de código y commits

- Controladores delgados; lógica en modelos, concerns y POROs (sin directorios
  de service objects). Preferir `account.balance_series` antes de
  `AccountSeries.new(account).call`.
- Ruby: indentación de dos espacios, `snake_case` para métodos/variables,
  `CamelCase` para clases. JavaScript: `lowerCamelCase` para variables,
  `PascalCase` para clases, siguiendo Biome. Sin lógica de dominio en ERB.
- Nunca commitear secretos; usar variables de entorno y `.env.local`.
- Commits: sujeto imperativo de máximo 72 caracteres, con racional y
  referencias a issues donde aplique. PRs pequeños y cohesivos hacia `main`.
