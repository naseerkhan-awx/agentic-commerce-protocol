# Unified Product Object

This note defines a canonical product object for internal DB storage when a
catalog may be imported from Google Merchant product data, or when a merchant
converts another product feed into this unified shape. Once stored, the object
must contain enough data to export both Google Merchant product rows and ACP Feed
API products.

The unified object is intentionally **one stored shape**. It is not an ACP object
plus a Google object. Common commerce concepts are normalized into first-class
fields, while source-specific fields that do not map cleanly are preserved under
`extensions`.

## Overview

The unified product object stores one product group and its sellable variants.
It deliberately follows the ACP Feed API product/variant split, because Google
Merchant rows can be grouped into that structure with deterministic rules.

| Concept | Unified object | Notes |
|---------|----------------|-------|
| Product group | `UnifiedProduct` | One DB document or product row. Groups one or more sellable variants. |
| Sellable offer | `UnifiedVariant` | One purchasable item. Google Merchant rows import here. ACP `Variant` records import here. |
| Normalized fields | First-class fields | Shared concepts such as title, description, media, price, availability, categories, identifiers, seller, and options. |
| Source-specific fields | `extensions` | Preserves Google-only or future source-only fields without creating a second stored product shape. |
| Import/export notes | Projection sections | Rules for Google Merchant import, other catalog import, ACP export, and Google Merchant export are below the field reference. |

## UnifiedProduct

Top-level DB object for a product group.

| Field | Type | Required | Notes | Details |
|-------|------|----------|-------|---------|
| `object` | string | no | Optional discriminator for mixed-document stores. Use `unified_product` when present. | — |
| `id` | string | yes | Canonical DB identifier for the product group. May equal ACP `Product.id`, Google `item_group_id`, or a deterministic single-row group id. | [Identifier rules](#identifier-rules) |
| `identifiers` | object | no | External and projection-specific product identifiers. | [ProductIdentifiers](#productidentifiers) |
| `market_scope` | object | no | Market, locale, storefront, or catalog partition context for scoped catalogs. | [MarketScope](#marketscope) |
| `title` | string | no | Shared product title when known. Variant titles remain authoritative for sellable offers. | — |
| `description` | object | no | Shared product description. | [Description](#description) |
| `url` | string (URI) | no | Shared product detail URL. Variant URL may override it. | — |
| `media` | array | no | Shared product media. Variant media may override it for offer-specific imagery. | [Media](#media) |
| `variants` | array | yes | Non-empty list of sellable variants grouped under this product. | [UnifiedVariant](#unifiedvariant) |
| `extensions` | object | no | Source-specific fields that are part of the one stored shape but not normalized first-class fields. | [UnifiedProductExtensions](#unifiedproductextensions) |
| `source` | object | no | Import source and source timestamps. | [SourceMetadata](#sourcemetadata) |
| `projection` | object | no | Warnings and projection bookkeeping. | [ProjectionMetadata](#projectionmetadata) |

## UnifiedVariant

Sellable offer under a unified product. This is the normal destination for one
Google Merchant row and for one ACP Feed API `Variant`.

| Field | Type | Required | Notes | Details |
|-------|------|----------|-------|---------|
| `id` | string | yes | Canonical DB identifier for the sellable offer. Should be checkout-compatible when possible. | [Identifier rules](#identifier-rules) |
| `identifiers` | object | no | External and projection-specific variant identifiers. | [VariantIdentifiers](#variantidentifiers) |
| `title` | string | yes | Variant display title. Required because ACP `Variant.title` is required. | — |
| `description` | object | yes | Variant-specific description. Required so Google `description` can be emitted. | [Description](#description) |
| `url` | string (URI) | yes | Variant detail URL. Required so Google `link` can be emitted. | — |
| `barcodes` | array | no | Typed identifiers such as GTIN, UPC, EAN, or MPN. | [Barcode](#barcode) |
| `price` | object | yes | Active selling price in minor units. Required so Google `price` can be emitted. | [Price](#price) |
| `list_price` | object | no | Reference or pre-discount price in minor units. | [Price](#price) |
| `unit_price` | object | no | Normalized unit price for measured goods. | [UnitPrice](#unitprice) |
| `availability` | object | yes | Purchasability and fulfillment state. Required so Google `availability` can be emitted. | [Availability](#availability) |
| `categories` | array | no | Variant-level category assignments. Google Merchant and ACP both emit categories on sellable items, so categories live only on variants. | [Category](#category) |
| `condition` | array | no | Condition labels such as `new`, `used`, or `refurbished`. | — |
| `variant_options` | array | no | Option axes such as color, size, gender, material, pattern, or age group. | [VariantOption](#variantoption) |
| `media` | array | yes | Variant-specific media. Must include at least one `image` item so Google `image_link` can be emitted. | [Media](#media) |
| `seller` | object | no | Merchant of record. | [Seller](#seller) |
| `marketplace` | object | no | Marketplace or intermediary platform. | [Seller](#seller) |
| `extensions` | object | no | Source-specific row or offer fields not normalized first-class fields. | [UnifiedVariantExtensions](#unifiedvariantextensions) |
| `source` | object | no | Import source and source timestamps. | [SourceMetadata](#sourcemetadata) |
| `projection` | object | no | Warnings and projection bookkeeping for this variant. | [ProjectionMetadata](#projectionmetadata) |

## Requiredness Model

The unified object is Google-first and export-ready. Required fields are chosen
so each stored variant can produce both a Google Merchant product row and an ACP
Feed API `Variant` without needing source-specific fallbacks.

| Requirement class | Required fields | Notes |
|-------------------|-----------------|-------|
| Unified product storage | `UnifiedProduct.id`, non-empty `UnifiedProduct.variants` | Guarantees a stable product group for DB storage and ACP `Product.id`. |
| Unified variant storage | `UnifiedVariant.id`, `title`, `description`, `url`, `price`, `availability`, `media` | Covers Google's universal row requirements and ACP's required `Variant.id` / `Variant.title`. |
| Nested required fields | `Description` has at least one text field; `Price.amount`, `Price.currency`; `Availability.available`, `Availability.status`; every `Media.type`, `Media.url`; at least one variant media item with `type: "image"` | Ensures export code can create Google `description`, `price`, `availability`, and `image_link`. |
| ACP export requirements | `Product.id`, `Product.variants`, every `Variant.id`, every `Variant.title` | Covered by unified storage requirements. Other unified fields are extra context ACP can accept when mapped to ACP-supported fields. |
| Google universal requirements | row `id`, `title` or `structured_title`, `description` or `structured_description`, `link`, `image_link`, `availability`, `price` | Covered by unified variant requirements after import normalization. |
| Google conditional requirements | Context-dependent fields such as `brand`, `gtin`, `mpn`, `condition`, apparel option axes, `item_group_id`, `availability_date`, `external_seller_id`, shipping fields, and regulatory fields | Validate separately based on target country, destination, product type, marketplace account, and Google program. |

Missing unified storage-required fields are validation errors. Missing Google
conditional fields are validation errors only when the applicable Google
condition is known to apply.

## Nested Fields

### ProductIdentifiers

External and projection-specific identifiers for a product group.

| Field | Type | Notes |
|-------|------|-------|
| `agentic_product_id` | string | ACP Feed API `Product.id` when distinct from the canonical DB `id`. |
| `google_item_group_id` | string | Google Merchant `item_group_id`, used to group variant rows. |
| `merchant_product_id` | string | Merchant catalog product id. |
| `source_product_id` | string | Source-system product id when distinct from merchant or ACP ids. |

### VariantIdentifiers

External and projection-specific identifiers for a sellable offer.

| Field | Type | Notes |
|-------|------|-------|
| `agentic_variant_id` | string | ACP Feed API `Variant.id` when distinct from the canonical DB `id`. |
| `google_id` | string | Google Merchant row `id`. |
| `sku` | string | Merchant SKU. |
| `gtin` | string | Global Trade Item Number when represented as a simple identifier. |
| `mpn` | string | Manufacturer part number. |
| `merchant_variant_id` | string | Merchant catalog variant id. |
| `source_variant_id` | string | Source-system variant id when distinct from merchant or ACP ids. |

### Identifier Rules

`id` is the canonical DB identity for the product or variant. The `identifiers`
object records external and projection-specific identifiers.

| Source | Product `id` | Variant `id` | Identifier notes |
|--------|--------------|--------------|------------------|
| Google Merchant with `item_group_id` | `item_group_id` | Google row `id` | Also store `google_item_group_id` and `google_id`. |
| Google Merchant without `item_group_id` | Google row `id` or deterministic group id | Google row `id` | Create a single-variant product. |
| Merchant catalog | Merchant product id or generated group id | Merchant variant id or SKU | Store source-specific ids under `identifiers`. |

### Description

Long-form content in one or more formats. At least one text field is required.
For Google export, prefer `plain`; when importing Google `structured_description`,
copy its `content` into `plain` and preserve structured metadata in extensions.

| Field | Type | Notes |
|-------|------|-------|
| `plain` | string | Plain text. Default destination for Google `description`. |
| `html` | string | HTML content. Renderers must sanitize before display. |
| `markdown` | string | Markdown content. Renderers must sanitize before display. |

### Price

Money in minor units with ISO 4217 currency. This should match the ACP `Price`
shape.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `amount` | integer | yes | Amount in currency minor units, for example cents for USD. |
| `currency` | string | yes | ISO 4217 currency code such as `USD`. |

### UnitPrice

Unit-normalized price for products sold by weight, volume, or other measure.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `amount` | integer | yes | Normalized price in minor units. |
| `currency` | string | yes | ISO 4217 currency code. |
| `measure` | object | yes | Actual package measure. See [Measure](#measure). |
| `reference` | object | yes | Reference measure for display. See [ReferenceMeasure](#referencemeasure). |

### Measure

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `value` | number | yes | Measured quantity. |
| `unit` | string | yes | Unit label such as `oz`, `ml`, `g`, or `kg`. |

### ReferenceMeasure

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `value` | integer | yes | Reference quantity. |
| `unit` | string | yes | Reference unit label. |

### Availability

Purchasability and fulfillment state.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `available` | boolean | yes | Whether the variant can currently be purchased. Infer from Google `availability` when importing Google rows. |
| `status` | string | yes | Fulfillment state such as `in_stock`, `limited_stock`, `backorder`, `preorder`, `out_of_stock`, or `discontinued`. Required so Google `availability` can be emitted. |

### Barcode

Typed machine-readable identifier.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `type` | string | yes | Identifier scheme such as `GTIN`, `UPC`, `EAN`, or `MPN`. |
| `value` | string | yes | Raw identifier value. |

### Media

Retrievable product or variant asset. Every media item requires `type` and
`url`. Each `UnifiedVariant.media` array must include at least one item with
`type: "image"` so Google `image_link` can be emitted.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `type` | string | yes | Asset kind such as `image`, `video`, or `model`. |
| `url` | string (URI) | yes | Canonical asset URL. |
| `alt_text` | string | no | Accessible description. |
| `width` | integer | no | Pixel width when known. |
| `height` | integer | no | Pixel height when known. |

### VariantOption

One selected option axis on a variant.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | Axis label such as `Color`, `Size`, `Gender`, `Material`, or `Age group`. |
| `value` | string | yes | Selected option value. |

### Category

Classification in a named taxonomy.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `value` | string | yes | Category label, taxonomy id, or hierarchical path. |
| `taxonomy` | string | no | Taxonomy name such as `google_product_category`, `product_type`, or `merchant`. |

### Seller

Merchant or marketplace identity.

| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Display name. |
| `links` | array | Policy or information links. See [Link](#link). |

### Link

Policy or informational URL.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `type` | string | yes | Link role such as `privacy_policy`, `terms_of_service`, `shipping_policy`, or `refund_policy`. |
| `title` | string | no | Human-readable label. |
| `url` | string (URI) | yes | Canonical URL. |

### MarketScope

Market, locale, storefront, or catalog partition scope for this stored product.

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | Scope id or catalog partition id. |
| `target_country` | string | ISO 3166-1 alpha-2 country code. |
| `locale` | string | Locale such as `en-US`. |
| `storefront` | string | Storefront, channel, seller partition, or other merchant-defined scope. |

### UnifiedProductExtensions

Container for product-level source-specific fields.

| Field | Type | Notes |
|-------|------|-------|
| `googleMerchant` | object | Google Merchant fields that apply to the product group or can be inherited by exported rows. See [GoogleMerchantProductExtension](#googlemerchantproductextension). |

### UnifiedVariantExtensions

Container for variant-level source-specific fields.

| Field | Type | Notes |
|-------|------|-------|
| `googleMerchant` | object | Google Merchant row fields that do not have first-class normalized fields. See [GoogleMerchantVariantExtension](#googlemerchantvariantextension). |

### GoogleMerchantProductExtension

Google Merchant fields that apply to a product group or can be inherited by each
exported Google row.

| Field | Type | Notes |
|-------|------|-------|
| `raw` | object | Original source payload or unmodeled fields for diagnostics and round-trip fidelity. |
| `structured_title` | object | Gen-AI title metadata. See [GoogleStructuredText](#googlestructuredtext). |
| `structured_description` | object | Gen-AI description metadata. See [GoogleStructuredText](#googlestructuredtext). |
| `custom_label_0` ... `custom_label_4` | string | Shopping campaign labels. |
| `included_destination` | array | Google destination inclusions. |
| `excluded_destination` | array | Google destination exclusions. |
| `shopping_ads_excluded_country` | array | Country exclusions for Shopping ads. |

### GoogleMerchantVariantExtension

Google Merchant row fields that are not normalized into first-class unified
fields.

| Field group | Fields | Notes |
|-------------|--------|-------|
| Raw source | `raw` | Original source payload or unmodeled row fields. |
| Links and ads | `mobile_link`, `ads_redirect`, `lifestyle_image_link`, `short_title`, `promotion_id`, `pause` | Google-specific merchandising and ads fields. |
| Marketplace | `external_seller_id` | Google marketplace seller id. |
| Identifiers | `identifier_exists`, `brand`, `mpn` | Preserved when not represented in canonical identifiers or barcodes. |
| Categories | `google_product_category`, `product_type` | Original Google category values when preserving exact source format. Normalized copies live in variant `categories`. |
| Detailed product flags | `adult`, `multipack`, `is_bundle` | Google-specific eligibility and product description fields. |
| Price programs | `cost_of_goods_sold`, `auto_pricing_min_price`, `maximum_retail_price`, `installment`, `subscription_cost`, `loyalty_program` | Google-specific pricing and program fields. |
| Dates | `availability_date`, `expiration_date`, `sale_price_effective_date` | Google-specific date or date-range fields. |
| Product dimensions | `product_length`, `product_width`, `product_height`, `product_weight` | Google product dimensions and weight. |
| Shipping and returns | `shipping`, `carrier_shipping`, `shipping_label`, `shipping_weight`, `shipping_length`, `shipping_width`, `shipping_height`, `ships_from_country`, `min_handling_time`, `max_handling_time`, `return_policy_label`, `free_shipping_threshold` | Google shipping and return policy fields. |
| Regulatory | `certification`, `energy_efficiency_class`, `min_energy_efficiency_class`, `max_energy_efficiency_class` | Regulatory and labeling fields. |

### GoogleStructuredText

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `content` | string | yes | Text content. |
| `digital_source_type` | string | no | Usually `default` or `trained_algorithmic_media`. |

### SourceMetadata

| Field | Type | Notes |
|-------|------|-------|
| `source` | string | Import source, for example `google_merchant`, `merchant_catalog`, or a merchant PIM/feed name. |
| `imported_at` | string (date-time) | Time this object was imported. |
| `updated_at` | string (date-time) | Source update time when known. |

### ProjectionMetadata

| Field | Type | Notes |
|-------|------|-------|
| `warnings` | array | Projection warnings. See [ProjectionWarning](#projectionwarning). |
| `generated_fields` | array | Field paths generated during import or export. |
| `lossy_fields` | array | Field paths that could not be represented exactly in a target projection. |

### ProjectionWarning

| Field | Type | Notes |
|-------|------|-------|
| `code` | string | Machine-readable warning code. |
| `message` | string | Human-readable warning. |
| `path` | string | Optional field path related to the warning. |

## Import and Export Logic

### Google Merchant Import

Google Merchant product data is row-based. Import each row as a
`UnifiedVariant`, then group variants into `UnifiedProduct` objects. Rows that
cannot populate the unified storage-required fields should fail validation unless
a documented normalization rule supplies the missing value.

1. Group rows by `item_group_id` when present.
2. If `item_group_id` is absent, create a single-variant product for the row.
3. Set the variant `id` from Google `id`.
4. Set the product `id` from `item_group_id`, or from the row `id` for
   single-variant products.
5. Map `title` or `structured_title.content` to `UnifiedVariant.title`.
   Preserve `structured_title` metadata in `extensions.googleMerchant`.
6. Map `description` or `structured_description.content` to
   `description.plain`. Preserve `structured_description` metadata in
   `extensions.googleMerchant`.
7. Map `link` to `url`; keep `mobile_link` in the Google extension unless the
   application wants mobile URLs to override canonical URLs.
8. Map `image_link`, `additional_image_link`, `video`, and
   `virtual_model_link` to typed `media[]`. The Google `image_link` must become
   the first `image` media item.
9. Map Google `price` to ACP-style `Price` in minor units. When `sale_price` is
   active, map `sale_price` to `price` and the base Google `price` to
   `list_price`.
10. Map `availability` to `availability.status` and infer
    `availability.available`.
11. Map `gtin`, `mpn`, and similar identifiers to `barcodes[]` or
    `identifiers`.
12. Map `google_product_category` and `product_type` to variant `categories[]`.
13. Map option axes such as `color`, `size`, `gender`, `material`, `pattern`,
    `age_group`, `size_system`, and `size_type` to `variant_options[]`.
14. Store Google-only row fields in `extensions.googleMerchant`.

### Other Catalog Import

Merchants that do not already have Google Merchant product data should convert
their catalog into this unified shape directly. The converted product must
provide all unified storage-required fields, plus any Google conditional fields
that apply to the merchant's target country, destination, product type,
marketplace account, or Google program.

1. Create one `UnifiedProduct` per product group.
2. Create one `UnifiedVariant` per sellable offer.
3. Provide `title`, `description`, `url`, at least one image `media` item,
   `price`, and `availability` for every variant.
4. Put Google-only fields that do not have normalized first-class fields in
   `extensions.googleMerchant`.
5. Validate universal unified requirements before storage and validate
   Google-conditional requirements before Google submission.

### ACP Feed Export

Exporting to ACP ignores Google-specific extensions except where they have
already been normalized into canonical fields. Because unified variants require
more fields than ACP does, ACP export can always satisfy ACP's required
`Product.id`, `Product.variants`, `Variant.id`, and `Variant.title` fields.

```ts
function toAgenticProduct(product: UnifiedProduct): AgenticProduct {
  return {
    id: product.identifiers?.agentic_product_id ?? product.id,
    title: product.title,
    description: product.description,
    url: product.url,
    media: product.media,
    variants: product.variants.map((variant) => ({
      id: variant.identifiers?.agentic_variant_id ?? variant.id,
      title: variant.title,
      description: variant.description,
      url: variant.url,
      barcodes: variant.barcodes,
      price: variant.price,
      list_price: variant.list_price,
      unit_price: variant.unit_price,
      availability: variant.availability,
      categories: variant.categories,
      condition: variant.condition,
      variant_options: variant.variant_options,
      media: variant.media,
      seller: variant.seller,
      marketplace: variant.marketplace
    }))
  };
}
```

The exported object must satisfy the ACP Feed API schema: `Product.id` and
`Product.variants` are required, and every `Variant` requires `id` and `title`.

### Google Merchant Export

Exporting to Google emits one row per `UnifiedVariant`. Universal Google row
fields should always be available from storage-required unified fields;
conditional Google fields still need context-specific validation before final
submission.

For each variant:

- Emit Google `id` from `variant.identifiers.google_id ?? variant.id`.
- Emit `item_group_id` from
  `product.identifiers.google_item_group_id ?? product.id` when the product has
  multiple variants or variant options.
- Emit `title` from `variant.title`.
- Emit `description` from `variant.description.plain`.
- Emit `link` from `variant.url`.
- Emit `image_link` from the first image in `variant.media`.
- Emit `additional_image_link` from remaining image media.
- Emit `video` and `virtual_model_link` from typed media where available.
- Convert ACP minor-unit `Price` objects into Google money strings such as
  `"19.99 USD"`.
- Emit `availability` from `variant.availability.status`.
- Emit `condition` from the first condition value when present.
- Expand `barcodes[]` and identifiers into `gtin`, `mpn`, and related Google
  columns.
- Expand variant `categories[]` into `google_product_category` and
  `product_type`.
- Expand `variant_options[]` into Google option columns where names match known
  axes.
- Merge in explicit Google extension fields after canonical fields, unless doing
  so would contradict a normalized canonical value.

## Worked Example

Two Google Merchant rows:

```json
[
  {
    "id": "sku-red-s",
    "item_group_id": "classic-tee",
    "title": "Classic Tee - Red / Small",
    "description": "A classic cotton tee.",
    "link": "https://merchant.example/products/classic-tee?variant=sku-red-s",
    "image_link": "https://cdn.example/classic-tee-red-s.jpg",
    "price": "24.99 USD",
    "sale_price": "19.99 USD",
    "availability": "in_stock",
    "condition": "new",
    "google_product_category": "Apparel & Accessories > Clothing > Shirts & Tops",
    "color": "Red",
    "size": "S",
    "gtin": "00012345600012"
  },
  {
    "id": "sku-red-m",
    "item_group_id": "classic-tee",
    "title": "Classic Tee - Red / Medium",
    "description": "A classic cotton tee.",
    "link": "https://merchant.example/products/classic-tee?variant=sku-red-m",
    "image_link": "https://cdn.example/classic-tee-red-m.jpg",
    "price": "24.99 USD",
    "sale_price": "19.99 USD",
    "availability": "out_of_stock",
    "condition": "new",
    "google_product_category": "Apparel & Accessories > Clothing > Shirts & Tops",
    "color": "Red",
    "size": "M",
    "gtin": "00012345600029"
  }
]
```

Unified DB object. This is the minimum export-ready shape: every variant has a
title, description, URL, image media, price, and availability status.

```json
{
  "object": "unified_product",
  "id": "classic-tee",
  "identifiers": {
    "google_item_group_id": "classic-tee"
  },
  "description": {
    "plain": "A classic cotton tee."
  },
  "variants": [
    {
      "id": "sku-red-s",
      "identifiers": {
        "google_id": "sku-red-s",
        "gtin": "00012345600012"
      },
      "title": "Classic Tee - Red / Small",
      "description": {
        "plain": "A classic cotton tee."
      },
      "url": "https://merchant.example/products/classic-tee?variant=sku-red-s",
      "barcodes": [
        {
          "type": "GTIN",
          "value": "00012345600012"
        }
      ],
      "price": {
        "amount": 1999,
        "currency": "USD"
      },
      "list_price": {
        "amount": 2499,
        "currency": "USD"
      },
      "availability": {
        "available": true,
        "status": "in_stock"
      },
      "categories": [
        {
          "taxonomy": "google_product_category",
          "value": "Apparel & Accessories > Clothing > Shirts & Tops"
        }
      ],
      "condition": ["new"],
      "variant_options": [
        {
          "name": "Color",
          "value": "Red"
        },
        {
          "name": "Size",
          "value": "S"
        }
      ],
      "media": [
        {
          "type": "image",
          "url": "https://cdn.example/classic-tee-red-s.jpg"
        }
      ]
    },
    {
      "id": "sku-red-m",
      "identifiers": {
        "google_id": "sku-red-m",
        "gtin": "00012345600029"
      },
      "title": "Classic Tee - Red / Medium",
      "description": {
        "plain": "A classic cotton tee."
      },
      "url": "https://merchant.example/products/classic-tee?variant=sku-red-m",
      "barcodes": [
        {
          "type": "GTIN",
          "value": "00012345600029"
        }
      ],
      "price": {
        "amount": 1999,
        "currency": "USD"
      },
      "list_price": {
        "amount": 2499,
        "currency": "USD"
      },
      "availability": {
        "available": false,
        "status": "out_of_stock"
      },
      "categories": [
        {
          "taxonomy": "google_product_category",
          "value": "Apparel & Accessories > Clothing > Shirts & Tops"
        }
      ],
      "condition": ["new"],
      "variant_options": [
        {
          "name": "Color",
          "value": "Red"
        },
        {
          "name": "Size",
          "value": "M"
        }
      ],
      "media": [
        {
          "type": "image",
          "url": "https://cdn.example/classic-tee-red-m.jpg"
        }
      ]
    }
  ],
  "source": {
    "source": "google_merchant"
  }
}
```

ACP Feed API product projection:

```json
{
  "id": "classic-tee",
  "description": {
    "plain": "A classic cotton tee."
  },
  "variants": [
    {
      "id": "sku-red-s",
      "title": "Classic Tee - Red / Small",
      "description": {
        "plain": "A classic cotton tee."
      },
      "url": "https://merchant.example/products/classic-tee?variant=sku-red-s",
      "barcodes": [
        {
          "type": "GTIN",
          "value": "00012345600012"
        }
      ],
      "price": {
        "amount": 1999,
        "currency": "USD"
      },
      "list_price": {
        "amount": 2499,
        "currency": "USD"
      },
      "availability": {
        "available": true,
        "status": "in_stock"
      },
      "categories": [
        {
          "taxonomy": "google_product_category",
          "value": "Apparel & Accessories > Clothing > Shirts & Tops"
        }
      ],
      "condition": ["new"],
      "variant_options": [
        {
          "name": "Color",
          "value": "Red"
        },
        {
          "name": "Size",
          "value": "S"
        }
      ],
      "media": [
        {
          "type": "image",
          "url": "https://cdn.example/classic-tee-red-s.jpg"
        }
      ]
    },
    {
      "id": "sku-red-m",
      "title": "Classic Tee - Red / Medium",
      "description": {
        "plain": "A classic cotton tee."
      },
      "url": "https://merchant.example/products/classic-tee?variant=sku-red-m",
      "barcodes": [
        {
          "type": "GTIN",
          "value": "00012345600029"
        }
      ],
      "price": {
        "amount": 1999,
        "currency": "USD"
      },
      "list_price": {
        "amount": 2499,
        "currency": "USD"
      },
      "availability": {
        "available": false,
        "status": "out_of_stock"
      },
      "categories": [
        {
          "taxonomy": "google_product_category",
          "value": "Apparel & Accessories > Clothing > Shirts & Tops"
        }
      ],
      "condition": ["new"],
      "variant_options": [
        {
          "name": "Color",
          "value": "Red"
        },
        {
          "name": "Size",
          "value": "M"
        }
      ],
      "media": [
        {
          "type": "image",
          "url": "https://cdn.example/classic-tee-red-m.jpg"
        }
      ]
    }
  ]
}
```

## SQL

### PostgreSQL Product Table

This simple relational storage model keeps one row per `UnifiedProduct` and
stores nested objects as JSONB. Variants remain inside the product row because
the unified object is one stored shape.

```sql
CREATE TABLE unified_products (
  id                      TEXT            PRIMARY KEY,
  object                  TEXT            DEFAULT 'unified_product',
  identifiers             JSONB,
  market_scope            JSONB,
  title                   TEXT,
  description             JSONB,
  url                     TEXT,
  media                   JSONB,
  variants                JSONB           NOT NULL,
  extensions              JSONB,
  source                  JSONB,
  projection              JSONB,
  created_at              TIMESTAMPTZ     NOT NULL DEFAULT now(),
  updated_at              TIMESTAMPTZ     NOT NULL DEFAULT now(),
  search_vector           TSVECTOR        GENERATED ALWAYS AS (
    setweight(
      to_tsvector(
        'english',
        coalesce(
          nullif(jsonb_path_query_array(variants, '$[*].title') #>> '{}', '[]'),
          title,
          ''
        )
      ),
      'A'
    ) ||
    setweight(
      to_tsvector(
        'english',
        coalesce(
          nullif(
            jsonb_path_query_array(variants, '$[*].description.plain') #>> '{}',
            '[]'
          ),
          description ->> 'plain',
          ''
        )
      ),
      'B'
    )
  ) STORED,

  CHECK (jsonb_typeof(variants) = 'array' AND jsonb_array_length(variants) > 0)
);

CREATE INDEX unified_products_search_idx
  ON unified_products USING GIN (search_vector);
```

The generated `search_vector` searches effective title and description text
only. Variant `title` and `description.plain` values are indexed first because
variants are authoritative for sellable offers; product-level `title` and
`description.plain` are fallback text when variant values are absent.

## DB Indexing

For a document database or JSONB table, index:

- `id`
- `market_scope.id`
- `market_scope.target_country`
- `identifiers.google_item_group_id`
- `variants.id`
- `variants.identifiers.google_id`
- `variants.identifiers.sku`
- `variants.barcodes.value`
- `variants.availability.status`
- `variants.price.amount`
- `variants.price.currency`
- variant category values and taxonomies
- variant option names and values

For a fully normalized relational model, store product and variant canonical
fields in separate tables and keep `extensions` as JSONB columns.

## Validation and Lossiness Rules

Validation errors should stop storage or target export when required data is
missing.

- Missing unified storage-required fields are errors: product `id`, non-empty
  `variants`, variant `id`, `title`, `description`, `url`, `price`,
  `availability`, or `media`.
- Missing nested required fields are errors: `Price.amount`, `Price.currency`,
  `Availability.available`, `Availability.status`, every `Media.type`, every
  `Media.url`, and at least one variant media item with `type: "image"`.
- Missing Google universal row fields after normalization are errors:
  `id`, title, description, `link`, `image_link`, `availability`, and `price`.
- Missing Google conditional fields are errors only when the relevant condition
  applies. Examples include `brand`, `gtin`, `mpn`, `condition`, apparel option
  axes, `item_group_id`, `availability_date`, `external_seller_id`, shipping
  fields, and regulatory fields.
- Google price strings that cannot be converted into minor units are errors.

Projection code should emit warnings, not errors, when data is valid but the
target representation is less expressive:

- A Google row lacks `item_group_id` but appears to be part of a variant family.
- ACP export omits Google-only data such as shipping, ads labels, loyalty, or
  certification fields.
- Google export has to flatten multiple categories, media entries, or condition
  values into single-value Google columns.
- Structured title or description metadata is preserved in extensions but
  projected to plain ACP fields.

Warnings belong in `projection.warnings` so import/export jobs can inspect data
quality without changing the canonical object shape.
