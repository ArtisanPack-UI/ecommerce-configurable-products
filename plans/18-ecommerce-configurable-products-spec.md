# Package Spec: `artisanpack-ui/ecommerce-configurable-products`

**Status:** Draft — ready to build. Issues are open in the [v1.0 milestone](https://github.com/ArtisanPack-UI/ecommerce-configurable-products/milestones).
**Owner:** Jacob Martella
**Last updated:** 2026-11-17
**Tracker:** [ArtisanPack-UI/ecommerce#56](https://github.com/ArtisanPack-UI/ecommerce/issues/56)
**Parent plan:** `12-ecommerce-package-plan.md` — §2 (product types), §3.3, §5.3 (products), §5.5 (cart `options` / `meta`), §6.1–§6.4 (contracts, registries, hooks, events)
**Engine spec:** [`13-ecommerce-engine-spec.md`](https://github.com/ArtisanPack-UI/ecommerce/blob/main/docs/plans/13-ecommerce-engine-spec.md) ("engine §N"). Engine guides: [`docs/contracts.md`](https://github.com/ArtisanPack-UI/ecommerce/blob/main/docs/contracts.md), [`docs/products.md`](https://github.com/ArtisanPack-UI/ecommerce/blob/main/docs/products.md), [`docs/satellites.md`](https://github.com/ArtisanPack-UI/ecommerce/blob/main/docs/satellites.md), [`docs/satellite-lifecycle.md`](https://github.com/ArtisanPack-UI/ecommerce/blob/main/docs/satellite-lifecycle.md)
**Siblings:** [`ecommerce-admin-livewire`](https://github.com/ArtisanPack-UI/ecommerce-admin-livewire) and [`ecommerce-storefront-livewire`](https://github.com/ArtisanPack-UI/ecommerce-storefront-livewire) build the UI for this package (D1).

## 0. Purpose

This is the build spec for the configurable-products satellite. A configurable product is one base product (a photo, a poster, a mug) that the shopper customizes by picking from **configurators** (size, color, finish, frame, …). Each choice can change the price, the SKU, the shipping weight and dimensions, and what downstream fulfillment receives.

It covers:

- the package identity;
- the data model for reusable configurator sets and per-product overrides;
- the `ConfiguratorType` contract and the three built-in types;
- the `configurable` product type: cart options, pricing, line identity, snapshots;
- shipping and fulfillment data per configured line;
- the REST and GraphQL surface the UI satellites consume;
- the engine changes it needs;
- the UI work, which lands in the admin and storefront satellites.

§16 breaks the build into issues.

When the parent plan and this spec disagree on configurable-product behavior, this spec wins. When this spec and the engine spec disagree on a contract, hook, table, or ability, the engine spec wins.

**Out of scope for v1.0:**

- Any UI in this package (D1). The admin and storefront Livewire UI is built in those packages; React and Vue follow in their own satellites.
- A live visual preview of the configured product (frame around the photo, canvas edge, puzzle overlay). Future Release.
- Generating print-ready files from the source media. Future Release.
- Stock tracking for physical components (frames, mats, boxes). Future Release.
- Built-in framing, canvas, or puzzle configurator types. Store owners model these with the Custom type (§6.3), and packages can register their own types (§6.1).
- Actual lab or print-on-demand integrations (Printful, Printify, local labs). They plug into the fulfillment router (§10.2) from their own satellites.

## 1. What the parent plan requires

| # | Deliverable | Section |
|---|---|---|
| 1 | A base product with pluggable configurators that modify price, variant identity, and downstream fulfillment (plan §3.3) | §5–§10 |
| 2 | Configurators are registered types (plan §3.3, §6.2) | §6 |
| 3 | Built on the engine's `ProductType` contract; validates the registry pattern under real complexity (tracker #56) | §7 |
| 4 | Contract-verified via `ecommerce:verify-satellite` (plan §15.2) | §14 |
| 5 | `ecosystem: ecommerce` issue set on this repo (tracker #56) | §16 |

## 2. Decisions

### 2.1 Locked by the parent plan and engine

- The product type implements `Contracts\ProductType` and registers through `ProductTypeRegistry` under the key `configurable`.
- Money is integer minor units through `moneyphp/money`; every currency has its own amounts, with the engine's `CurrencyRateProvider` fallback for missing ones (plan §5.1, §5.3).
- `validateCartOptions()` output is stored verbatim on `cart_items.options`, and `cart_items.options_hash` keeps different configurations of the same product on separate lines.
- `buildOrderSnapshot()` output is stored on `order_items.product_snapshot`, so historical orders survive edits to sets, choices, and prices.
- Registers with `SatelliteRegistry` and honors the uninstall lifecycle: when the package is removed, `configurable` products become read-only `MissingProductType` products and their orders keep rendering from snapshots.
- Day-1 locales are `en`, `es`, `fr`, `de`. Every user-facing string goes through `__()` / `trans_choice()`.
- New hooks use the `ap.` prefix and camelCase segments (`ap.ecommerce.configurableProducts.unitPrice`).

### 2.2 Decided for this package (confirmed 2026-11-17)

| # | Decision | Outcome | Why |
|---|---|---|---|
| D1 | UI location | **Headless only.** This package ships domain logic, REST, and GraphQL. The admin screens and the storefront configurator are issues on `ecommerce-admin-livewire` and `ecommerce-storefront-livewire`, in their **v2.0** (design-system) milestones, built on `livewire-ui-components` 3.0. One design issue in this repo lists every screen and component and blocks those build issues. | Keeps the engine/satellite split (plan §1.1): every UI family consumes the same API. |
| D2 | Configuration model | **Reusable configurator sets with per-product overrides.** A set ("Fine art prints") holds configurators and their choices and prices, and is attached to many products. A product can hide choices, change a choice's price, change defaults, and attach more than one set. | A photo catalog shares the same sizes and finishes across hundreds of products. Editing one set updates them all. |
| D3 | Configurator types | **Three built-ins:** Size (dimension choices that can also set weight and shipping dimensions), Color (swatches), and Custom (admin-defined choices). Plus a `ConfiguratorType` contract and `ConfiguratorTypeRegistry` so other packages add their own. | Size and Color cover the common cases; Custom covers framing, finish, paper, puzzle pieces, and anything else without code. |
| D4 | No variants | `configurable` products have **no `product_variants`**. A configuration is a map of configurator key → choice key on `cart_items.options`; prices are computed. | Variants explode combinatorially (8 sizes × 4 papers × 6 frames = 192 rows per photo). |
| D5 | Keys, not ids | Carts and APIs reference configurators and choices by stable **keys** (slugs), unique within their set and configurator. Database ids stay internal. | Keys survive set duplication and imports, and are readable in carts, webhooks, and fulfillment payloads. |
| D6 | Pricing | Unit price = **base price** (the product's `product_prices`) adjusted per chosen choice by a **fixed amount** per currency, a **percent** of the base, or a **set** price that replaces the base. Product overrides win over set prices. Details in §8. | Matches how print price lists are written: "+$40 for 16×20", "+15% for metallic paper". |
| D7 | Fulfillment | **A fulfillment routing contract in this package** (`FulfillmentRoute` + `FulfillmentRouteRegistry`), a fulfillment payload per configured order line, routing rules, and a default `manual` route. No lab integration ships here. | Lets Printful/Printify/lab satellites route configured lines without this package knowing about them. If the engine later gains a generic fulfillment-routing contract, the routes move there. |
| D8 | Inventory | The `configurable` type is **not inventory-tracked** in 1.0 (made to order). | `ProductType::isInventoryTracked()` is type-level; component inventory is Future Release. |
| D9 | Engine release | Requires `artisanpack-ui/ecommerce ^1.4` for the engine additions in §12. | All four are optional contracts or additive fields, so they land in a minor. |
| D10 | Tables | Prefix `ecommerce_configurable_`; migrations in `database/migrations/ecommerce-configurable-products/`. | Matches the other satellites and keeps uninstall clean. |

## 3. Package identity

| Item | Value |
|---|---|
| Composer name | `artisanpack-ui/ecommerce-configurable-products` |
| Repository | `github.com/ArtisanPack-UI/ecommerce-configurable-products` |
| Namespace | `ArtisanPackUI\EcommerceConfigurableProducts\` |
| Service provider | `EcommerceConfigurableProductsServiceProvider` |
| Facade | `ConfigurableProducts` |
| Config | `config/artisanpack/ecommerce-configurable-products.php` → `artisanpack.ecommerce-configurable-products.*` |
| Translations | JSON catalogues in `lang/` |
| Product type key | `configurable` |
| Meta namespaces | `cart_items.meta.configurable`, `order_items.meta.configurable` |
| License | MIT |

## 4. Dependencies

### 4.1 `require`

| Package | Constraint | Why |
|---|---|---|
| `php` | `^8.3` | Matches the other ecommerce satellites (Pest 4). |
| `illuminate/support`, `illuminate/database`, `illuminate/http` | `^12.0\|^13.0` | Matches the engine. |
| `artisanpack-ui/ecommerce` | `^1.4` | `ProductType`, registries, plus the §12 additions. |
| `artisanpack-ui/core`, `artisanpack-ui/hooks` | `^1.0` | Ecosystem baseline; hooks. |

### 4.2 `suggest` (detected at runtime)

| Package | Enables |
|---|---|
| `artisanpack-ui/media-library` | Color swatch images and choice images (`media_id` attributes) |
| `rebing/graphql-laravel` | GraphQL types (§11.2), when the engine's GraphQL is enabled |
| `artisanpack-ui/ecommerce-admin-livewire` (^2.0) | Admin screens for sets, the product panel, order line details (built there, D1) |
| `artisanpack-ui/ecommerce-storefront-livewire` (^2.0) | Storefront configurator and line summaries (built there, D1) |

### 4.3 `require-dev`

Pest 4 (+ `pest-plugin-laravel`), Orchestra Testbench, `artisanpack-ui/code-style`, `artisanpack-ui/code-style-pint`, `friendsofphp/php-cs-fixer`, `dealerdirect/phpcodesniffer-composer-installer`, `laravel/pint` (only for `code-style-pint`'s fixers; **never run Pint on this package**), `rebing/graphql-laravel`.

## 5. Data model

All tables use the `ecommerce_configurable_` prefix (shortened below to `…_`).

```sql
…_sets (
    id, key VARCHAR(100) UNIQUE, name VARCHAR(255), description TEXT NULL,
    status ENUM('active','archived') DEFAULT 'active',
    meta JSON NULL, created_at, updated_at, deleted_at
)

…_configurators (
    id, set_id FK, key VARCHAR(100),              -- unique per set
    type VARCHAR(60),                              -- ConfiguratorTypeRegistry key: size | color | custom | …
    label VARCHAR(255), help TEXT NULL,
    display VARCHAR(30),                           -- select | radio | swatch | cards (type decides which are allowed)
    is_required BOOLEAN DEFAULT TRUE,
    default_choice_key VARCHAR(100) NULL,
    position INT DEFAULT 0, settings JSON NULL,    -- type-specific, validated by the type
    created_at, updated_at,
    UNIQUE (set_id, key)
)

…_choices (
    id, configurator_id FK, key VARCHAR(100),      -- unique per configurator
    label VARCHAR(255), description TEXT NULL,
    is_active BOOLEAN DEFAULT TRUE, position INT DEFAULT 0,
    price_mode ENUM('none','add','percent','set') DEFAULT 'none',
    price_percent DECIMAL(7,4) NULL,               -- for percent
    sku_suffix VARCHAR(50) NULL,
    attributes JSON NULL,                          -- type-specific (size dimensions/weight, color hex/media_id, …)
    fulfillment JSON NULL,                         -- free-form data passed to fulfillment routes (§10)
    created_at, updated_at,
    UNIQUE (configurator_id, key)
)

…_choice_prices (                                  -- per-currency amounts for add / set
    id, choice_id FK, currency CHAR(3), amount BIGINT,
    UNIQUE (choice_id, currency)
)

…_rules (                                          -- §9
    id, set_id FK, kind ENUM('requires','excludes','shows'),
    when_configurator_key VARCHAR(100), when_choice_keys JSON,
    then_configurator_key VARCHAR(100), then_choice_keys JSON NULL,
    position INT DEFAULT 0, created_at, updated_at
)

…_product_sets (                                   -- attachment, ordered
    product_id FK, set_id FK, position INT DEFAULT 0,
    PRIMARY KEY (product_id, set_id)
)

…_product_overrides (                              -- per-product changes to a choice or configurator
    id, product_id FK, set_id FK, configurator_key VARCHAR(100), choice_key VARCHAR(100) NULL,
    is_hidden BOOLEAN NULL, is_default BOOLEAN NULL, label VARCHAR(255) NULL,
    price_mode ENUM('none','add','percent','set') NULL, price_percent DECIMAL(7,4) NULL,
    created_at, updated_at,
    UNIQUE (product_id, set_id, configurator_key, choice_key)
)

…_product_override_prices (
    id, override_id FK, currency CHAR(3), amount BIGINT,
    UNIQUE (override_id, currency)
)

…_fulfillment_rules (                              -- §10.3
    id, priority INT, route_key VARCHAR(60),
    match JSON,                                    -- { product_ids?, set_keys?, configurator_key?, choice_keys? }
    is_active BOOLEAN DEFAULT TRUE, created_at, updated_at
)
```

- Configurator keys must be unique across all sets attached to one product. Attaching a set that collides is a validation error naming both sets.
- Archiving a set hides it from new attachments but keeps it working on attached products. Deleting a set that is attached to products is refused; the error lists the products.
- Deleting a product removes its attachments and overrides (FK cascade).
- Every write goes through a service (§7.5) that records engine activity entries and busts the resolver cache (§7.2).

## 6. Configurator types

### 6.1 The contract

```php
interface ConfiguratorType
{
    public function key(): string;                                   // 'size'
    public function label(): string;                                 // translated
    public function icon(): ?string;
    public function displays(): array;                               // allowed display modes, first is the default
    public function settingsRules(): array;                          // Laravel rules for configurators.settings
    public function choiceAttributeRules(): array;                   // Laravel rules for choices.attributes
    public function describeChoice( Choice $choice ): string;        // "8 × 10 in", "Black"
    public function storefrontChoice( Choice $choice ): array;       // extra schema keys: swatch, image, hint (§7.4)
    public function shippingProfile( Choice $choice ): ?array;       // weight / dimensions, or null (§10.1)
    public function fulfillmentAttributes( Choice $choice ): array;  // merged into the fulfillment payload (§10.2)
}
```

`AbstractConfiguratorType` implements sensible defaults. Types register in a service provider's `boot()`:

```php
app( ConfiguratorTypeRegistry::class )->register( 'frame', FrameConfiguratorType::class );
```

Registration is idempotent and follows the engine registry conventions (throw in dev, warn in prod on a duplicate key). A configurator whose type is no longer registered keeps working for existing carts and orders (labels come from the choice rows and snapshots) but cannot be edited and is hidden from new configurations, with a logged warning. An abstract `ConfiguratorTypeContractTest` ships in `Testing\Contracts\` for other packages.

### 6.2 Size

- Choice attributes: `width`, `height` (decimal), `unit` (`in`, `cm`, `mm`), optional `weight` + `weight_unit`, optional shipping `length`/`width`/`height` + `dim_unit`.
- Configurator settings: `unit` default, `show_orientation_hint` (bool).
- Displays: `cards` (default), `select`, `radio`.
- `describeChoice()`: localized `8 × 10 in`. `shippingProfile()` returns the weight and package dimensions when set.

### 6.3 Custom

- Admin-defined choices with label, description, optional `media_id` image, and optional `weight`/`weight_unit` added to the line.
- Displays: `select` (default), `radio`, `swatch` (needs an image or hex on every choice), `cards`.
- Covers paper, finish, frame, mat, puzzle pieces, gift wrap.

### 6.4 Color

- Choice attributes: `hex` (validated `#RRGGBB`) and/or `media_id` (pattern or texture image).
- Displays: `swatch` (default), `select`, `radio`.
- `storefrontChoice()` adds `swatch: { hex, image_url, alt }`. The selected-state indicator and its contrast are the UI's job.

## 7. The `configurable` product type

### 7.1 Registration and behavior

- `ConfigurableProductType extends AbstractProductType implements ProvidesStorefrontOptions` plus the §12 contracts (`DescribesLineOptions`, `ProvidesLineShippingProfile`).
- `requiresFulfillment()` true; `isInventoryTracked()` false (D8).
- A `configurable` product with no attached sets behaves like a simple product with no options.
- Products cannot have variants; the product service rejects variant writes for this type with a `ProductWriteException`.

### 7.2 Resolver

`ProductConfigurationResolver::for( Product $product ): ProductConfiguration` merges the attached sets in order, applies the product's overrides, drops inactive and hidden choices, and returns an immutable value object: ordered configurators, each with its effective choices, defaults, prices per currency, and the set's rules. It is cached per product (tagged cache when available, versioned keys otherwise) and busted on any set, configurator, choice, rule, attachment, or override write. A configurator left with no visible choices is dropped, and the drop is logged.

### 7.3 Cart options

Cart options shape:

```json
{ "configuration": { "size": "8x10", "paper": "lustre", "frame": "black-oak" } }
```

`validateCartOptions()`:

1. resolves the product's configuration;
2. fills defaults for missing optional configurators;
3. rejects unknown configurator keys, unknown/inactive/hidden choices, and missing required choices;
4. applies the rules (§9): excluded choices, unmet requirements, and values for hidden configurators are errors (hidden ones are dropped, not errors, when they equal the default);
5. returns the canonical form: keys sorted by configurator position, so `options_hash` is stable for equal configurations.

Errors are `ValidationException`s keyed `options.configuration.{key}` with translated messages, so the storefront shows them next to the field.

### 7.4 Storefront options

`storefrontOptions()` returns one field per configurator, using the richer schema from engine §12 E3:

```php
[
    'type'    => 'radio',                       // or select / swatch per the configurator's display
    'name'    => 'configuration.size',
    'label'   => 'Size',
    'required'=> true,
    'default' => '8x10',
    'help'    => '…',
    'options' => [
        [ 'value' => '8x10', 'label' => '8 × 10 in', 'price_delta' => [ 'amount' => 0, 'currency' => 'USD' ], 'disabled' => false ],
        [ 'value' => '16x20', 'label' => '16 × 20 in', 'price_delta' => [ 'amount' => 4000, 'currency' => 'USD' ], 'image' => …, 'swatch' => … ],
    ],
    'visible_when' => [ … ],                    // from 'shows' rules
    'meta'    => [ 'configurator_type' => 'size', 'display' => 'cards', 'disables' => [ … ] ],  // excludes/requires map for client-side disabling
]
```

`price_delta` is relative to the configuration with every other configurator at its default and is in the storefront's current currency. Exact prices for a full configuration come from the price quote (§8.3).

### 7.5 Services

- `ConfiguratorSetService`: create, update, duplicate (new keys suffixed `-copy`), archive, restore, delete (guarded), reorder configurators and choices, and CRUD for configurators, choices, choice prices, and rules — every write validated by the configurator's type.
- `ProductConfigurationService`: attach, detach, reorder sets on a product; upsert and clear overrides.
- Both throw `ConfigurationWriteException` with `{ field, code, message }` rows (mirrors the engine's `ProductWriteException`), and write engine activity entries.

## 8. Pricing

### 8.1 Unit price

For cart currency `C`, with `base` = the product's resolved price in `C` (engine `ProductPriceResolver`, including sale windows):

1. If any chosen choice has mode `set`, start from the `set` amount of the **last** such configurator in position order; otherwise start from `base`.
2. Add every `add` amount.
3. Add every `percent` choice as `round( percent × base )` — always of the base, never compounding.
4. Pass through `ap.ecommerce.configurableProducts.unitPrice` `(Money $unit, Product $product, ProductConfiguration $configuration, array $selection, string $currency)`.
5. Clamp at zero.

`priceLine()` returns `unit × quantity`.

### 8.2 Currencies and overrides

- A product override's mode and amounts replace the set's for that choice.
- A missing `add`/`set` amount in `C` is converted from the store's base currency through the engine's `CurrencyRateProvider`, with the same rounding the engine uses for product prices.
- Rounding is half-up per step through moneyphp.

### 8.3 Price quote

Implements the engine's price-quote extension (engine §12 E4): given a product, options, quantity, and currency, return the unit price, line total, and a breakdown per configurator (`{ configurator, choice, mode, amount }`). Promotions are not applied (the quote is a list price).

## 9. Choice rules

Rules live on a set and reference configurator and choice keys inside it (rules across sets attached to one product are out of scope):

| Kind | Meaning | Example |
|---|---|---|
| `excludes` | When `when` has one of `when_choice_keys`, `then` may not be any of `then_choice_keys` | Size `4x6` excludes Frame `black-oak` |
| `requires` | When `when` has one of `when_choice_keys`, `then` must be one of `then_choice_keys` | Paper `canvas` requires Finish `gloss` |
| `shows` | `then` configurator is only shown (and only allowed) when `when` has one of `when_choice_keys` | Mat only when Frame ≠ `none` |

- Rule writes are validated: referenced keys exist; no rule makes a required configurator impossible to satisfy with the defaults; no `shows` cycles.
- The resolver exposes a `RuleEvaluator` used by cart validation (§7.3), the storefront schema (§7.4), and the quote (§8.3), so all three agree.
- Product overrides that hide choices are applied before rules; a rule that references a hidden choice is ignored for that product.

## 10. Shipping and fulfillment

### 10.1 Shipping weight and dimensions

Through the engine's per-line shipping profile contract (engine §12 E1): Size choices supply the package weight and dimensions when set (else the product's); Custom choices can add weight. The result goes to the engine's weight-based methods and to carrier rate providers.

### 10.2 Fulfillment payload

On `OrderPlaced`, `onOrderPlaced()` builds a `FulfillmentPayload` per configured line and stores it on `order_items.meta.configurable.fulfillment`:

```json
{
  "version": 1,
  "composite_sku": "PHOTO-123-16X20-LUS-BOAK",
  "quantity": 2,
  "product": { "id": 123, "name": "Sunrise over Lake Superior", "sku": "PHOTO-123", "media_id": 456 },
  "choices": [
    { "configurator": "size", "type": "size", "choice": "16x20", "label": "16 × 20 in",
      "attributes": { "width": 16, "height": 20, "unit": "in" }, "fulfillment": { "lab_code": "P1620" } }
  ],
  "route": { "key": "manual", "status": "pending", "reference": null }
}
```

- The composite SKU is the product SKU plus each choice's `sku_suffix`, in configurator order, joined by `-`; filter `ap.ecommerce.configurableProducts.compositeSku`.
- The payload passes through `ap.ecommerce.configurableProducts.fulfillmentPayload`.

### 10.3 Routing

```php
interface FulfillmentRoute
{
    public function key(): string;
    public function label(): string;
    public function accepts( FulfillmentPayload $payload ): bool;
    public function dispatch( Order $order, OrderItem $item, FulfillmentPayload $payload ): RouteResult;  // { status: submitted|failed|manual, reference?, message? }
    public function cancel( Order $order, OrderItem $item, FulfillmentPayload $payload ): RouteResult;
}
```

- `FulfillmentRouteRegistry` holds routes; the built-in `manual` route does nothing and marks the line for in-house production.
- Fulfillment rules (§5) pick a route per line: the first active rule by `priority` whose `match` fits and whose route `accepts()` the payload; otherwise `manual`.
- Dispatch runs in a queued job after `PaymentSucceeded` (never inside the placement transaction), is idempotent per order item, retries with backoff, and records the result on the payload plus an order note. Cancelling or fully refunding the order calls `cancel()`.
- Routes report later status changes with `ConfigurableProducts::updateRouteStatus( $orderItem, $status, $reference )`, which fires `ap.ecommerce.configurableProducts.routeStatusChanged`. Creating shipments stays with the engine's fulfillment service.
- An abstract `FulfillmentRouteContractTest` ships for route authors.

## 11. API surface

### 11.1 REST (`/api/ecommerce/v1`, engine auth, rate limits, problem responses, Idempotency-Key on writes)

| Method & path | Purpose | Ability |
|---|---|---|
| `GET/POST configurator-sets`, `GET/PATCH/DELETE configurator-sets/{set}` | Sets (id or key), `include=configurators.choices.prices,rules,products_count` | `configuratorSet.*` |
| `POST configurator-sets/{set}/duplicate`, `…/archive`, `…/restore` | Lifecycle | `configuratorSet.update` |
| nested `configurators`, `choices`, `rules` CRUD + `PUT …/order` | Editing | `configuratorSet.update` |
| `GET/PUT products/{product}/configuration` | Attached sets + overrides (admin) | `product.update` |
| `GET products/{product}/configuration/resolved` | Effective configuration (storefront, public for active products) | — |
| `POST products/{product}/quote` | Price quote (engine E4) | — |
| `GET configurator-types` | Registered types with their displays and attribute schemas | `configuratorSet.viewAny` |
| `GET/POST/PATCH/DELETE configurable-fulfillment-rules`, `GET fulfillment-routes` | Routing | `configurableFulfillment.*` |
| `POST orders/{order}/items/{item}/configurable-fulfillment/retry` | Re-dispatch a failed line | `configurableFulfillment.update` |

Cart add and order resources need no new endpoints: they carry `options.configuration` and the engine E2 descriptions.

### 11.2 GraphQL

When the engine's GraphQL is enabled: `ConfiguratorSet`, `Configurator`, `ConfiguratorChoice`, `ProductConfiguration` types; `Product.configuration` field; `configuratorSets` query; set/configurator/choice/rule/attachment mutations with `UserError`s; `productQuote` query (engine E4).

### 11.3 Abilities and policies

`ConfiguratorSetPolicy` and `ConfigurableFulfillmentPolicy` use the engine's policy base and ability naming (`configuratorSet.viewAny|view|create|update|delete`, `configurableFulfillment.viewAny|update`), registered with the engine's ability catalogue so the admin's roles screen lists them.

## 12. Engine changes (engine milestone v1.4)

Filed on the engine as ArtisanPack-UI/ecommerce#221–#224.

| # | Change | Used by |
|---|---|---|
| E1 ([#221](https://github.com/ArtisanPack-UI/ecommerce/issues/221)) | Optional `ProvidesLineShippingProfile` contract on product types: per-line weight and package dimensions, used by the built-in weight-based methods and passed to `ShippingRateProvider`s | §10.1 |
| E2 ([#222](https://github.com/ArtisanPack-UI/ecommerce/issues/222)) | Optional `DescribesLineOptions` contract: human-readable `{ label, value }` rows for a cart line's options; exposed on cart and order REST/GraphQL resources, copied into the order snapshot, and used by notification templates | §7, storefront + admin summaries |
| E3 ([#223](https://github.com/ArtisanPack-UI/ecommerce/issues/223)) | Richer add-to-cart field schema: per-option `price_delta`, `image`, `swatch`, `hint`, `disabled`; a `swatch` field type; `visible_when` conditions; documented in docs/products.md and validated by the storefront | §7.4 |
| E4 ([#224](https://github.com/ArtisanPack-UI/ecommerce/issues/224)) | Price quote: optional `QuotesPrice` contract (default falls back to `priceLine()`), `PriceQuoteService`, `POST products/{product}/quote`, GraphQL `productQuote` | §8.3 |

## 13. UI (built in the sibling satellites, D1)

One design issue in this repo lists every screen and component; build issues live in the UI repos' **v2.0** milestones and are blocked by the design issue and the APIs they need.

**ecommerce-admin-livewire** (registers only when this package is installed):

- Configurator sets index page (search, status filter, product counts, duplicate/archive).
- Configurator set editor page: configurators (reorder, type, display, required, default), per-type choice editors (Size dimensions/weight, Color swatches, Custom choices with images), per-currency prices, rules (reusing the admin's rule builder).
- Configurable product type panel (`ProductTypePanelRegistry`): attach and order sets, per-choice overrides (hide, default, label, price), price preview.
- Order detail: configured line summary (from the snapshot) and fulfillment route status, with retry.
- Fulfillment routing settings screen (rules and registered routes).

**ecommerce-storefront-livewire:**

- The generic add-to-cart options form gains the E3 schema: swatch and card choices, price deltas, disabled states, conditional fields, and a live price from the quote (E4). This serves configurable products and any other type that uses the schema.
- Configuration summaries on cart lines, the cart drawer, checkout, order confirmation, and account order pages (E2).

## 14. Testing

1. **Unit and feature tests (Pest 4)** for the resolver, rules, pricing (every mode, overrides, currency fallback, rounding), cart validation, snapshots, fulfillment routing, REST, and GraphQL. Target ~90% coverage (plan §15).
2. **Contract verification:** the engine's `ProductTypeContractTest` for `configurable`, this package's `ConfiguratorTypeContractTest` for the three built-ins and `FulfillmentRouteContractTest` for `manual`; `ecommerce:verify-satellite` on PRs; the release workflow signs the report with the org key.
3. **Performance check:** resolving and pricing a product with 3 sets, 10 configurators, and 200 choices stays within a fixed query budget (asserted), and cached resolution does no queries.
4. Browser tests belong to the UI satellites.

## 15. Future (not v1.0)

- Live visual preview (mockup) of a configuration.
- Print-file generation from the source media per order line.
- Component inventory (frames, mats, boxes) via the engine's `ExpandsInventory`.
- Rules across sets; price formulas per type (for example, Size priced by area).
- Moving `FulfillmentRoute` into the engine if a generic routing contract appears there (D7).

## 16. Issue breakdown

Milestone **v1.0** unless noted. "Depends on" means within this repo; "Blocked by" means cross-repo.

| # | Title | Group | Depends on |
|---|---|---|---|
| 1 | Package scaffold: composer, provider, config, code style, CI, SatelliteRegistry registration | M0 Foundation | — |
| 2 | Design the configurable-products admin and storefront UI | Design | — |
| 3 | Data model: configurator sets, configurators, choices, prices, rules, product attachments and overrides | M0 | 1 |
| 4 | `ConfiguratorType` contract, registry, and contract test | M0 | 1 |
| 5 | Built-in Custom configurator type | M1 Types | 3, 4 |
| 6 | Built-in Size configurator type | M1 | 3, 4 |
| 7 | Built-in Color configurator type | M1 | 3, 4 |
| 8 | Configurator set service: CRUD, ordering, duplicate, archive, deletion guards | M1 | 3, 4 |
| 9 | Product configuration service and resolver: attachments, overrides, caching | M2 Product type | 8 |
| 10 | Choice rules: requires, excludes, shows | M2 | 9 |
| 11 | `configurable` product type: registration, cart option validation, line identity | M2 | 9, 10 |
| 12 | Pricing: base price and per-choice modifiers across currencies | M2 | 11 |
| 13 | Storefront options schema for configurable products | M2 | 11, 12 · blocked by ArtisanPack-UI/ecommerce#223 |
| 14 | Price quote for a configuration | M2 | 12 · blocked by ArtisanPack-UI/ecommerce#224 |
| 15 | Line descriptions, composite SKU, and order snapshot | M2 | 11 · blocked by ArtisanPack-UI/ecommerce#222 |
| 16 | Shipping weight and dimensions from the configuration | M2 | 6, 11 · blocked by ArtisanPack-UI/ecommerce#221 |
| 17 | Fulfillment payload for configured order lines | M3 Fulfillment | 15 |
| 18 | Fulfillment routes: contract, registry, routing rules, manual route, dispatch | M3 | 17 |
| 19 | Abilities and policies | M4 API | 8 |
| 20 | REST API | M4 | 9, 10, 18, 19 |
| 21 | GraphQL schema | M4 | 20 |
| 22 | Promotion condition: cart contains a configuration choice | M4 | 11 |
| 23 | Hooks and filters: extension surface, documented and tested | M4 | 12, 17, 18 |
| 24 | Demo seeder: photography print catalog | M5 Release | 12 |
| 25 | Contract verification and verify-satellite workflow | M5 | 11, 18 |
| 26 | `es`, `fr`, `de` catalogues and translation lint | M5 | 5–23 |
| 27 | Documentation | M5 | 20, 21, 23 |
| 28 | v1.0 release | M5 | 24–27 |
| 29 | Live visual preview of a configuration | Future Release | — |
| 30 | Print-file generation from the source media | Future Release | 17 |
| 31 | Component inventory for configurable products | Future Release | 11 |
