# Google Merchant Center — Product data specification (reference)

**Source:** [Product data specification](https://support.google.com/merchants/answer/7052112?hl=en) (Google Merchant Center Help)

This document is an **external reference**. It is **not** part of the Agentic Commerce Protocol HTTP APIs. It summarizes how Google expects **product items** to be described in Merchant Center (text feeds, XML, Content API / Merchant API, and related surfaces).

**Item:** one sellable row or offer in your product data (for example, one line in a delimited feed or one product resource in an API). Attribute **names** use underscore-separated English identifiers (for example, `image_link`). Many **values** must also be English when Google defines a fixed vocabulary (for example, `condition`).

**Definitions (Google):**

- **Product:** what shoppers search for.
- **Item:** your submitted representation of a product in a feed or API.
- **Variant:** version of a product that differs by attributes such as size or color.

**Requirement legend:**

- **Required** — must be present for serving in the applicable programs.
- **Depends** — required only for certain products, countries, or programs.
- **Optional** — omit unless useful for performance or reporting.

Schemas below list **top-level product attributes** only. Composite attributes link to dedicated sections for sub-fields and formatting rules.

---

## Basic product data

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `id` | Required | Stable unique id per item (often SKU); max 50 characters. | — |
| `title` | Required* | Plain text; max 150 characters. | *Use **either** `title` **or** [`structured_title`](#structured_title) (not both for the same semantic). |
| `structured_title` | Required* | Gen-AI–sourced titles. | [Structured title](#structured_title) |
| `description` | Required* | Plain text; max 5000 characters. | *Use **either** `description` **or** [`structured_description`](#structured_description). |
| `structured_description` | Required* | Gen-AI–sourced descriptions. | [Structured description](#structured_description) |
| `link` | Required | Product landing page URL (`http`/`https`, encoded, verified domain). | — |
| `mobile_link` | Optional | Mobile PDP when it differs from `link`. | — |
| `image_link` | Required | Main image URL; crawlable; minimum size requirements apply (see Google’s article for dates). | — |
| `additional_image_link` | Optional | Extra images; repeatable; max 2000 characters per URL. | — |
| `video` | Optional | Product video URL (constraints in Google’s Product data specification article; Merchant API: `videoLinks`). | See [canonical article](https://support.google.com/merchants/answer/7052112?hl=en) and [video updates](https://support.google.com/merchants/answer/15216925?hl=en). |
| `virtual_model_link` | Optional | `.gltf` / `.glb` 3D model; US only in Google’s spec. | — |

---

## Price and availability

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `availability` | Required | Structured values such as `in_stock`, `out_of_stock`, `preorder`, `backorder`. | — |
| `availability_date` | Depends | Required when `availability` is `preorder` or `backorder` (ISO 8601). | — |
| `cost_of_goods_sold` | Optional | Numeric + ISO 4217 currency; reporting / margin insights. | — |
| `expiration_date` | Optional | ISO 8601; when the item should stop showing (< 30 days ahead per Google guidance). | — |
| `price` | Required | Numeric + ISO 4217; VAT/tax rules vary by country. | — |
| `sale_price` | Optional | Must pair with base `price` (non-sale). | — |
| `sale_price_effective_date` | Optional | ISO 8601 range; delimiter `/` between start and end. | — |
| `unit_pricing_measure` | Optional | Quantity + unit (per regulations). | — |
| `unit_pricing_base_measure` | Optional | Base quantity for unit price display (e.g. `100 ml`). | — |
| `installment` | Optional | Finance / lease display for eligible programs. | [Installment](#installment) |
| `subscription_cost` | Optional | Recurring billing for permitted categories/regions. | [Subscription cost](#subscription_cost) |
| `loyalty_program` | Optional | Member price, points, loyalty shipping linkage. | [Loyalty program](#loyalty_program) |
| `auto_pricing_min_price` | Optional | Floor for automated discounts / dynamic promotions. | — |
| `maximum_retail_price` | Depends | Example: India offerings; INR. | — |

---

## Product category

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `google_product_category` | Optional | Google taxonomy ID or full path text. | — |
| `product_type` | Optional | Merchant-defined breadcrumbs; max 750 characters first value used for some Ads features. | — |

---

## Product identifiers

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `brand` | Depends | Required for most new products; exceptions (e.g. books) per Google. | — |
| `gtin` | Depends | Strongly recommended when available; check digit and GS1 rules. | — |
| `mpn` | Depends | Required when no manufacturer GTIN. | — |
| `identifier_exists` | Optional | `yes` / `no` when UPIs are missing or partial. | — |

---

## Detailed product description

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `condition` | Depends | `new`, `refurbished`, `used`. | — |
| `adult` | Depends | `yes` / `no` for adult content. | — |
| `multipack` | Depends | Integer count for merchant-assembled multipacks. | — |
| `is_bundle` | Depends | `yes` / `no` when selling a merchant-defined bundle with a main product. | — |
| `certification` | Depends | Regulatory / energy labels (e.g. EPREL linkage in EU). | [Certification](#certification) |
| `energy_efficiency_class` | Depends | CH / NO / UK scope per Google’s 2025+ EU vs non-EU rules. | — |
| `min_energy_efficiency_class` | Depends | Combined with max + class for label range. | — |
| `max_energy_efficiency_class` | Depends | Same; note typo variants in legacy docs (`max_energy_efficiency`). | — |
| `age_group` | Depends | `newborn`, `infant`, `toddler`, `kids`, `adult`. | — |
| `color` | Depends | Variant distinguishing; apparel rules vary by country. | — |
| `gender` | Depends | `male`, `female`, `unisex`. | — |
| `material` | Depends | Slash-separated materials (primary / secondary). | — |
| `pattern` | Depends | Graphic / pattern description. | — |
| `size` | Depends | Apparel sizing where applicable. | — |
| `size_type` | Optional | Cut such as `petite`, `maternity`, `plus`. | — |
| `size_system` | Optional | `US`, `EU`, …; defaults by target country. | — |
| `item_group_id` | Depends | Groups variants that differ by supported axes. | — |
| `product_length` | Optional | Numeric + `in` or `cm`. | — |
| `product_width` | Optional | Same unit as other product dimensions. | — |
| `product_height` | Optional | Same unit as other product dimensions. | — |
| `product_weight` | Optional | `lb`, `oz`, `g`, `kg`. | — |
| `product_detail` | Optional | Structured name/value sections. | [Product detail](#product_detail) |
| `product_highlight` | Optional | Short bullets; repeatable. | — |

---

## Shopping campaigns and other configurations

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `ads_redirect` | Optional | Alternate final URL for ads; same registered domain as `link`. | — |
| `custom_label_0` … `custom_label_4` | Optional | Up to five campaign labels. | — |
| `promotion_id` | Depends | Match merchant promotion data; up to 10 per product. | — |
| `lifestyle_image_link` | Optional | Lifestyle imagery for eligible surfaces. | — |
| `short_title` | Optional | Demand Gen / short format; still supply `title`. | — |

---

## Marketplaces

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `external_seller_id` | Required | For multi-seller / marketplace accounts. | — |

---

## Destinations

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `excluded_destination` | Optional | Values such as `Shopping_ads`, `Free_listings`, … | — |
| `included_destination` | Optional | Same vocabulary; exception-based overrides. | — |
| `shopping_ads_excluded_country` | Optional | ISO 3166-1 alpha-2; Shopping ads country exclusions. | — |
| `pause` | Optional | Temporary pause (`ads`); re-approve by removing when stale. | — |

---

## Shipping and returns

| Attribute | Req | Notes | Details |
|-----------|-----|-------|---------|
| `shipping` | Depends | Overrides / augments account shipping (country, service, price, times). | [Shipping](#shipping) |
| `carrier_shipping` | Optional | Carrier-calculated pricing / speed configurations. | [Carrier shipping](#carrier_shipping) |
| `handling_cutoff_time` | Optional | Same-day cutoff by country/time zone. | [Handling cutoff](#handling_cutoff_time) |
| `minimum_order_value` | Optional | Minimum basket for purchase by country/surface. | [Minimum order value](#minimum_order_value) |
| `shipping_label` | Optional | Key into account shipping groups (not loyalty sub-field). | — |
| `shipping_weight` | Depends | For carrier-calculated / weight tiers at account level. | — |
| `shipping_length` | Depends | Package dimensions (`in`/`cm`). | — |
| `shipping_width` | Depends | Submit all three dimensions when any is used. | — |
| `shipping_height` | Depends | Submit all three dimensions when any is used. | — |
| `ships_from_country` | Optional | ISO alpha-2 ship-from. | — |
| `max_handling_time` | Optional | Business days upper bound (pairs with transit). | — |
| `min_handling_time` | Optional | Business days lower bound. | — |
| `shipping_transit_business_days` | Optional | In-transit days configuration. | [Shipping transit business days](#shipping_transit_business_days) |
| `shipping_handling_business_days` | Optional | Warehouse open days affecting handling SLAs. | [Shipping handling business days](#shipping_handling_business_days) |
| `free_shipping_threshold` | Optional | Order value for free shipping by country. | [Free shipping threshold](#free_shipping_threshold) |
| `return_policy_label` | Optional | Maps to Merchant Center return policy presets. | — |

---

## Composite attributes (nested fields)

Use these sections when an attribute bundles multiple labeled parts in one submission string or structured object.

### `structured_title`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `digital_source_type` | Optional | `default` or `trained_algorithmic_media`; defaults to `default`. |
| `content` | Required | Title text; max 150 characters. |

Example pattern (colon / quoted semantics per [Google’s attribute-value rules](https://support.google.com/merchants/answer/10668075)):

`trained_algorithmic_media:"Stride & Conquer: Example product title"`

---

### `structured_description`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `digital_source_type` | Optional | `default` or `trained_algorithmic_media`. |
| `content` | Required | Body text; max 5000 characters. |

Rules mirror plain `description` (no unrelated links, promotions, gimmick caps).

---

### `installment`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `credit_type` | Optional | `finance` (default) or `lease`; lease nuance applies to Vehicle Ads. |

Google describes a colon-separated composite for count, periodic amount, currency, optional down payment (vehicle / regional availability — see canonical article).

---

### `subscription_cost`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `period` | Required | `week` (physical goods subscriptions only where allowed), `month`, or `year`. |

Follow with amount + ISO 4217 per Google’s delimiter rules; landing page must state terms.

---

### `loyalty_program`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `program_label` | Optional | Matches Merchant Center program label; omit single-tier shorthand per Google. |
| `tier_label` | Optional | Tier within program. |
| `price` | Optional | Member price (never put member price in plain `sale_price`). |
| `cashback_for_future_use` | Optional | Reserved. |
| `loyalty_points` | Optional | Whole-number points earned. |
| `member_price_effective_date` | Optional | Interval member pricing applies. |
| `shipping_label` | Optional | Ties loyalty free shipping eligibility to labeled shipping setups. |

---

### `shipping`

Colon-separated composite; typical sub-fields (all optional unless Google requires shipping for your target):

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `country` | Required | ISO 3166 destination country. |
| `region` | Optional | Regional targeting. |
| `postal_code` | Optional | Postal rules / wildcards. |
| `location_id` | Optional | Merchant-defined location identifiers. |
| `location_group_name` | Optional | Named location groups. |
| `service` | Optional | Service class label. |
| `price` | Optional | Fixed price including VAT when required. |
| `min_handling_time` / `max_handling_time` | Optional | Handling SLA (numeric). |
| `min_transit_time` / `max_transit_time` | Optional | Transit SLA (numeric). |
| `shipping_transit_business_days` | Optional | See [Shipping transit business days](#shipping_transit_business_days). |
| `shipping_handling_business_days` | Optional | See [Shipping handling business days](#shipping_handling_business_days). |
| `loyalty_program_label` | Optional | Restricts rate to loyalty members. |
| `loyalty_tier_label` | Optional | Tier scoping for loyalty shipping. |

---

### `carrier_shipping`

Alternative to fully manual `shipping` rows when using carrier-calculated quotes.

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `flat_price` | One of | Manual carrier row price. |
| `carrier_price` | One of | Carrier-calculated selection; mutual exclusivity with `flat_price`. |
| `fixed_min_transit_time` / `fixed_max_transit_time` | Optional | Fixed transit window. |

Exact carrier enums and origin postal fields follow Google’s carrier list and Merchant API `CarrierShipping` documentation.

---

### `certification`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `certification_authority` | Required | e.g. `EC` / `European_Commission` for supported regulatory flows. |
| `certification_name` | Required | e.g. `EPREL` where applicable. |
| `certification_code` | Required | Certificate id extracted from registry URLs. |

EU energy-label graphical requirements increasingly prefer `certification` over legacy free-text-only classes — follow Google’s 2025+ guidance.

---

### `product_detail`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `section_name` | Required | Groups details; max 140 characters. |
| `attribute_name` | Required | Max 140 characters. |
| `attribute_value` | Required | Confirmed factual value; max 1000 characters. |

Avoid duplicating structured attributes (`price`, shipping, competitor references, spam).

---

### `handling_cutoff_time`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `country` | Optional | ISO country for cutoffs. |
| `cutoff_time` | Required | 24-hour `HH:MM` local meaning. |
| `cutoff_timezone` | Optional | IANA TZ; ties warehouse vs destination interpretation. |
| `disable_delivery_after_cutoff` | Optional | Boolean; defaults false. |

---

### `minimum_order_value`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `country` | Required | ISO country. |
| `service` | Optional | Named shipping service class. |
| `surface` | Optional | `online`, `local`, or `online_local` (default). |
| `price` | Required | Minimum spend threshold + currency matching offer currency. |

---

### `shipping_transit_business_days`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `country` | Required | ISO country (repeat attribute for multiple countries). |
| `business_days` | Required | `Mon-Fri`-style compressed grammar per Google (`M`, `Mon`, …). |

Up to ten configurations.

---

### `shipping_handling_business_days`

Single formatted value (optionally prefixed per country — see Google examples) listing operational weekdays for handling calculations; same day tokens as transit field.

---

### `free_shipping_threshold`

| Sub-field | Req | Notes |
|-----------|-----|-------|
| `country` | Required | ISO country code. |
| `price_threshold` | Required | Order subtotal granting free shipping. |

Currency must align with primary offer currency rules.

---

## Canonical sources

| Resource | Link |
|----------|------|
| Product data specification | https://support.google.com/merchants/answer/7052112?hl=en |
| Title / structured title | https://support.google.com/merchants/answer/6324415 |
| Description / structured description | https://support.google.com/merchants/answer/6324468 |
| Submit attributes & values | https://support.google.com/merchants/answer/10668075 |
| Product video links (policy / naming updates) | https://support.google.com/merchants/answer/15216925?hl=en |
| Merchant API `ProductAttributes` (camelCase field reference) | https://developers.google.com/merchant/api/reference/rest/products_v1/ProductAttributes |

**Disclaimer:** Requirements change by country, program, and time. Prefer Google’s live Help articles and Merchant Center diagnostics for authoritative enforcement dates and allowed values.
