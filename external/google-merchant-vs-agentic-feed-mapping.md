# Mapping: Google Merchant product data ↔ Agentic Feed API (`2026-04-17`)

This note relates:

- **[`google-merchant-product-data-spec.md`](./google-merchant-product-data-spec.md)** — summarized Google Merchant Center **flat** offer attributes (`id`, `link`, `image_link`, `item_group_id`, …).
- **[`api-docs/2026-04-17/feed.md`](../api-docs/2026-04-17/feed.md)** — Agentic Commerce Protocol **Feed API** catalog model (`Product` containing `variants[]`, plus nested envelopes).

Canonical machine-readable sources for ACP shapes: [`schema.feed.json`](../spec/2026-04-17/json-schema/schema.feed.json), [`openapi.feed.yaml`](../spec/2026-04-17/openapi/openapi.feed.yaml).

For a DB-backed canonical object that can ingest Google Merchant rows and project
to either ACP Feed API products or Google Merchant rows, see
[`unified-product-object.md`](../unified/unified-product-object.md).

**Feasibility:** A useful **directional mapping** exists for many merchandising concepts (identifiers, PDP URL, imagery, pricing, eligibility text). **It is not 1:1:** Google assumes many **sellable rows** keyed by **`[id]`** and links variants with **`item_group_id`**; ACP nests offers under **`Product.variants`** and requires a **non-empty `variants`** array on every **Product**. Integrations therefore need deterministic rules (e.g. one Google row → one `Variant`; `item_group_id` → stable `Product.id`).

---

## Structural alignment

| Concept | Google Merchant (flattened feed) | Agentic Feed API |
|--------|----------------------------------|------------------|
| Sellable SKU / offer row | Usually one **`id`** per line | **`Variant`** with required **`id`**, **`title`**, … |
| Variant family | Duplicate rows sharing **`item_group_id`** | One **`Product`** with many **`variants[]`** |
| PDP URL per offer | **`link`** (often item-specific URL) | `Variant.url` (variant PDP) or `Product.url` (group PDP) |
| Primary image | **`image_link`** (+ `additional_image_link`, …) | **`media[]`**; first variant item is treated as primary in docs |
| Description | **`description`** or **`structured_description`** | **`Description`** object: `plain`, `html`, `markdown` (≥1 field when description is supplied) |

---

## Google Merchant attribute → ACP Feed (typical destinations)

Rough targets only—payload validation is always against JSON Schema.

| Google attribute (see [Google summary](./google-merchant-product-data-spec.md)) | Typical ACP path | Notes |
|--------------------------------------------------------------------------------|------------------|--------|
| `id` | `Product.variants[*].id` | Often 1 Google row ⇒ 1 variant; SKU-level id aligns with schema example. |
| `item_group_id` | Derived `Product.id` | Concatenate/normalize externally; Merchant Center grouping has no typed `Product` object. |
| `title` | `Variant.title`; optional `Product.title` | **`Variant.title`** is **always required** once a variant exists (`feed.md`; schema). Google allows `title` *or* `structured_title`; ACP omits GA-specific structured titles. |
| `structured_title` | `Variant.title` (from `content`) | Lose `digital_source_type` unless mapped to extension metadata outside this spec. |
| `description` | `Variant.description.plain` (or `.html`/`.markdown`) | ACP **`Description`** requires ≥1 keyed field inside the object. |
| `structured_description` | Same as `description` | Same loss of GA metadata as structured title. |
| `link` | `Variant.url` | PDP for the SKU/offer. |
| `mobile_link` | `Variant.url` or omitted | Prefer explicit mobile URL handling in integrations if both exist. |
| `image_link` | `Variant.media[0]` with `type: "image"` | Add `additional_image_link` as further `Media` entries. |
| `video`, `virtual_model_link` | `Variant.media[*]` (`type`: `video`, `model`, …) | Google treats video/3D as separate attrs; unify under typed `Media` in ACP. |
| `brand` | `Seller.name`; or taxonomy extension | Feed schema has **no top-level `brand`** on Variant. Derive externally if needed. |
| `gtin`, `mpn`, `identifier_exists` | `Variant.barcodes[]` (+ business rules for missing ids) | ACP barcode is `{ type, value }`; derive `GTIN`/`MPN`/… from Google's columns (not listed as separate GMC keys in summary table). |
| `price`, `sale_price` | `Variant.price`; `Variant.list_price` | **Different semantics:** ACP **`Price.amount`** is **integer minor units** (`schema.feed.json`); Google submits decimal strings (`15.00 USD`). Convert carefully. Typical mapping: Google's non-sale **`price`** + **`sale_price`** ↔ ACP **`list_price`** + **`price`** **only after** aligning business meaning with your storefront. |
| `unit_pricing_measure`, `unit_pricing_base_measure` | `Variant.unit_price.measure` / `reference` | ACP **`UnitPrice`** also uses minor-unit `amount` + structured measures; Google's colon-separated composites need parsing. |
| `availability` | `Variant.availability.status` | Google uses string tokens (`in_stock`, …); ACP **`status`** is extensible (`feed.md`). |
| N/A explicitly | `Variant.availability.available` | Separate boolean in ACP; infer from Google's `availability` (`out_of_stock` ⇒ `available: false`, etc.). |
| `condition` | `Variant.condition` array | GMC: single enumerated value (`new`, …); ACP uses an array container (`feed.md`). |
| `google_product_category`, `product_type` | `Variant.categories[*].value` + `taxonomy` | e.g. `taxonomy: "google_product_category"` vs merchant path (`product_type`). |
| `color`, `size`, `gender`, `material`, `pattern`, `age_group`, `size_system`, … | `Variant.variant_options[*]` `{ name, value }` | Normalize axis names (“Color”, “Size”, …); Google columns become option pairs. |
| `seller` / marketplace context | `Variant.seller`; `Variant.marketplace` | `external_seller_id` overlaps **Marketplace seller id** loosely; typed policy **`Link`** payloads are richer than Google's flat id. |

Most **Ads / GMC campaign** attributes (`ads_redirect`, `custom_label_*`, `promotion_id`, `shopping_ads_excluded_country`, `pause`, **shipping**\*, **loyalty_program**, **`certification`** composite, **`installment`** / **`subscription_cost`**, **`cost_of_goods_sold`**, **`expiration_date`**, …) **do not** have first-class fields on **Variant** in [`feed.md`](../api-docs/2026-04-17/feed.md)—map only via sidecar metadata, profiles, or future schema extensions unless your integration layer adds conventions.

\*Shipping in Google is expansive (many top-level composites). ACP has **Seller.links** typed URLs (e.g. `shipping_policy`) but no inline rate tables comparable to GMC `shipping` rows.

---

## ACP Feed → nearest Google Merchant column (inverse)

| ACP (`feed.md`) | Typical Google Merchant attribute |
|-----------------|-------------------------------------|
| `Product.id` | Derived from **`item_group_id`** (or fabricated when single-variant) |
| `Product.title`, `description`, `media` | Family-level grouping only; GMC often duplicates **`title`** on each row unless using feed rules in MC. |
| `Product.variants` | Repeated rows keyed by **`id`** + shared **`item_group_id`** |
| `Variant.title` | `title` (+ optional `structured_title` only if emitting GA format) |
| `Variant.description` | `description` (`plain` ⇒ default text export) |
| `Variant.url` | `link` |
| `Variant.price` | `sale_price` *or* `price` depending on pricing policy |
| `Variant.list_price` | Base `price` when using sale display (verify merchant rules). |
| `Variant.barcodes` | `gtin`, `mpn`, etc. expanded from `type`. |
| `Variant.media` | `image_link`, `additional_image_link`, optional video/model attrs |
| `Variant.categories` | `google_product_category` and/or `product_type` |
| `Variant.variant_options` | `color`, `size`, `gender`, … |
| `Variant.availability` | `availability` (+ `availability_date` for preorder/backorder) |
| `Variant.seller` / `marketplace` | Brand / seller fields (`brand`, possibly `external_seller_id`). |

---

## Variant required vs Google “required\*” titles

- **ACP `Variant`** requires **`id`** and **`title`** unconditionally ([`feed.md` § Variant](../api-docs/2026-04-17/feed.md)).
- Google requires a title **either** as `title` **or** `structured_title`; both are summarized in [`google-merchant-product-data-spec.md`](./google-merchant-product-data-spec.md) but interchangeably.

Mappings must supply **`Variant.title`** for every exported variant even when reusing Google **`structured_title`**.

---

## Required in ACP / JSON Schema but absent or materially different from [`google-merchant-product-data-spec.md`](./google-merchant-product-data-spec.md)

Below, **“missing”** means: **not modeled as named Google Merchant catalog attributes** in our external GMC summary doc (often because they are protocol-only, nesting-only, value-encoding-only, or ACP-added structure).

### 1. Product-level (catalog graph)

| ACP requirement | Why it is flagged |
|-----------------|-------------------|
| **`Product.variants`** (non-empty array) | Schema-required on every **`Product`** ([`schema.feed.json`](../spec/2026-04-17/json-schema/schema.feed.json) `required: ["id","variants"]`). Google conveys the same grouping via **repeated rows** + **`item_group_id`**, not a nested **`variants`** array attribute. Our GMC summary does **not** define such a nested container—**implementations must synthesize** `Product` + `variants` from GMC rows. |
| **`Product.id`** as stable group identifier | GMC exposes **`item_group_id`** optionally; SKU-level **`id`** is required per row instead. Exact string identity rules differ across merchants. |

### 2. JSON shapes & fields when objects are supplied

ACP fields required **when their parent object is present** (`schema.feed.json`). None of these exact keys appear as Google flat attribute names with the same structure in our GMC markdown:

| ACP nested object | Schema-required inner fields | GMC summary gap |
|-------------------|------------------------------|----------------|
| **`Description`** | At least **`plain`**, **`html`**, or **`markdown`** (`minProperties: 1`). | GMC has a single textual **`description`** (or structured GA form). Export must choose a channel (usually `plain`) for text feeds. HTML/Markdown parallel slots are **ACP-only.** |
| **`Price`**, **`list_price`** | **`amount`** (integer, minor units), **`currency`** (ISO 4217 `^[A-Z]{3}$`). | GMC **`price`** / **`sale_price`** strings combine major units + currency token in one grammar; minor-unit integers are **not** in the GMC attribute summary—and **ACP `list_price` has no GMC column rename** (`list_price` is an ACP semantic). |
| **`UnitPrice`** | **`amount`, `currency`, `measure.value`, `measure.unit`, `reference.value`, `reference.unit`** | GMC **`unit_pricing_measure`** / **`unit_pricing_base_measure`** are textual composites requiring parsing/normalization—not the same typed objects. |
| **`Barcode`** (each element) | **`type`**, **`value`** | GMC splits **`gtin`**, **`mpn`**, etc.; **explicit `Barcode.type`** is an ACP construct. **`identifier_exists`** has no counterpart in Feed JSON. |
| **`Media`** (each element) | **`type`**, **`url`**; optional **`alt_text`**, **`width`**, **`height`** | GMC uses parallel columns (`image_link`, video attrs, …); **typed `Media.type`** plus dimension fields are absent from GMC flat attribute names in our summary. |
| **`VariantOption`** | **`name`**, **`value`** | GMC uses dedicated columns (`color`, `size`, …) rather than an array of `{name,value}` pairs—**conceptually mappable**, structurally absent. |
| **`Category`** (each element) | **`value`**; optional **`taxonomy`** | GMC uses **`google_product_category`** vs **`product_type`** as parallel strings—not the unified `{value, taxonomy?}` carrier. |
| **`Link`** (under seller) | **`type`**, **`url`** required | GMC has no equivalent **typed policy link rows** list in product data summary (some policies live in Merchant Center account settings rather than per SKU). **`Seller.links[]`** remains **ACP-only** unless synthesized from storefront metadata. |

### 3. Boolean & container semantics absent from GMC summary rows

| ACP (`feed.md` / schema) | GMC summary |
|--------------------------|------------|
| **`Availability.available`** (boolean alongside `status`) | GMC **`availability`** conveys status mostly via enum-like strings—**boolean purchasability** is not a documented flat attribute alias in [`google-merchant-product-data-spec.md`](./google-merchant-product-data-spec.md). Infer or drop on import. |
| **`condition` as an array (`Condition`)** | GMC **`condition`** is a single enumerated submission per item (mapped to pluralized array semantics in exporters). |

### 4. HTTP / feed-resource envelopes (not merchant product attributes)

These are required by **`feed.md`** and/or OpenAPI for the **Feed API**, and **never** belong to GMC product rows:

| ACP artifact | Required fields (`feed.md`/OpenAPI/schema) |
|--------------|-------------------------------------------|
| **`ProductsResponse`**, **`UpsertProductsRequest`** | **`products`** array |
| **`FeedMetadata`** | **`id`**; optional **`target_country`**, **`updated_at`** |
| **`UpsertProductsResponse`** | **`id`**, **`accepted`** ([`openapi.feed.yaml`](../spec/2026-04-17/openapi/openapi.feed.yaml)) |
| **`Error`** | **`type`**, **`code`**, **`message`** |
| **`CreateFeedRequest`** | All fields optional |

Treat these strictly as transport concerns when comparing to Google `7052112`.

---

## Google-only buckets (little or no Feed `Variant`/`Product` coverage in `feed.md`)

The following regions of [`google-merchant-product-data-spec.md`](./google-merchant-product-data-spec.md) are **not mirrored** by first-class **`Product` / `Variant`** fields in **`2026-04-17`** `feed.md` (beyond possible future extensions or unstructured sidecars):

- **Shopping Ads configuration:** `ads_redirect`, `custom_label_0–4`, `promotion_id`, `lifestyle_image_link`, `short_title`
- **Destinations:** `excluded_destination`, `included_destination`, `shopping_ads_excluded_country`, `pause`
- **Marketplace identifiers:** `external_seller_id` (partial overlap with **`marketplace`** name only—not id typing)
- **Shipping & returns composites:** entire **Shipping** table (`shipping`, `carrier_shipping`, dimensional shipping weights, business-day matrices, thresholds, **`return_policy_label`**, …)
- **Merchant pricing programs:** `installment`, `subscription_cost`, `loyalty_program`, `auto_pricing_min_price`, `maximum_retail_price`, `cost_of_goods_sold`, `expiration_date`
- **Regulatory / labeling:** detailed **`certification`**, **`energy_efficiency_*`**, **`adult`**, multipack/bundle/eligibility quirks

When bi-directionally syncing catalogs, persist these in GMC-native feeds or ancillary storage rather than projecting them into unfinished ACP fields.

---

## Summary

Mapping **is possible as a pragmatic integration contract** once you nail **variants ↔ rows**, **pricing unit conversion**, and **taxonomy/option decomposition**. **`Product`/`variants`** nesting plus **typed `Money`/`Media`/`Barcode`/`Link`** models are **required by ACP** but **never appear as verbatim Google Merchant Center attribute names** in [`google-merchant-product-data-spec.md`](./google-merchant-product-data-spec.md)—those gaps are enumerated above **§ Required in ACP**.
