# Feed API

**Specification version:** `2026-04-17`

This document describes the **Feed** HTTP API: creating feed resources, reading feed metadata and product catalogs, and partially updating products by `Product.id`. Shapes align with [`spec/2026-04-17/json-schema/schema.feed.json`](../../spec/2026-04-17/json-schema/schema.feed.json); path and HTTP details match [`spec/2026-04-17/openapi/openapi.feed.yaml`](../../spec/2026-04-17/openapi/openapi.feed.yaml).

**Base URL:** `https://merchant.example.com` (example; substitute your merchant host.)

For bulk offline ingestion, merchants may ship `metadata.json` ([FeedMetadata](#feedmetadata)) and `products.jsonl` (one [Product](#product) JSON object per line). File-based ingestion replaces the full product set; **partial** catalog changes use `PATCH /feeds/{id}/products` only.

**Examples:** Payloads use **only fields required by OpenAPI** (nested objects follow their own required arrays).

**Error response** ([`Error`](#error), required fields only):

```json
{
  "type": "invalid_request",
  "code": "example_error_code",
  "message": "Human-readable explanation."
}
```

---

## Overview

| Method | Path | Purpose |
|--------|------|--------|
| `POST` | `/feeds` | Create a feed; receive server-assigned metadata. |
| `GET` | `/feeds/{id}` | Read metadata for an existing feed. |
| `GET` | `/feeds/{id}/products` | Read the **full** current product list for the feed. |
| `PATCH` | `/feeds/{id}/products` | Upsert a **subset** of products; others stay unchanged. |

---

## POST `/feeds` — Create a feed

**Purpose:** Registers a new feed resource. The server returns stable identifiers and optional market and update timestamps.

**Request body (`application/json`):** [CreateFeedRequest](#createfeedrequest)

| Status | Body | Meaning |
|--------|------|---------|
| `201` | [FeedMetadata](#feedmetadata) | Feed created. |
| `400` | [Error](#error) | Invalid payload (e.g. bad `target_country`). |

**Example request** (`application/json`): [CreateFeedRequest](#createfeedrequest) has no required properties:

```json
{}
```

**Example response** (`201`, [FeedMetadata](#feedmetadata)):

```json
{
  "id": "feed_01EXAMPLE"
}
```

---

## GET `/feeds/{id}` — Retrieve feed metadata

**Purpose:** Returns server-managed metadata for the feed identified by `id`.

**Path parameters:**

| Name | Description |
|------|--------------|
| `id` | Feed resource identifier. |

**Responses:**

| Status | Body | Meaning |
|--------|------|---------|
| `200` | [FeedMetadata](#feedmetadata) | Success. |
| `404` | [Error](#error) | Feed not found. |

**Example response** (`200`, [FeedMetadata](#feedmetadata)):

```json
{
  "id": "feed_01EXAMPLE"
}
```

---

## GET `/feeds/{id}/products` — Retrieve products for a feed

**Purpose:** Returns the **complete** catalog snapshot for this feed.

**Path parameters:**

| Name | Description |
|------|--------------|
| `id` | Feed resource identifier. |

**Responses:**

| Status | Body | Meaning |
|--------|------|---------|
| `200` | [ProductsResponse](#productsresponse) | Full product set. |
| `404` | [Error](#error) | Feed not found. |

**Example response** (`200`, [ProductsResponse](#productsresponse)): [Product](#product) requires `id` and `variants`; each [Variant](#variant) requires `id` and `title`.

```json
{
  "products": [
    {
      "id": "prod_01EXAMPLE",
      "variants": [{ "id": "var_01EXAMPLE", "title": "Example variant" }]
    }
  ]
}
```

---

## PATCH `/feeds/{id}/products` — Upsert products for a feed

**Purpose:** Create or update products matched by [`Product.id`](#product). Products **not** included in the request are left as-is.

**Path parameters:**

| Name | Description |
|------|--------------|
| `id` | Feed resource identifier. |

**Request body (`application/json`):** [UpsertProductsRequest](#upsertproductsrequest)

**Responses:**

| Status | Body | Meaning |
|--------|------|---------|
| `200` | [UpsertProductsResponse](#upsertproductsresponse) | Upsert accepted (see OpenAPI schema for acknowledgement shape). |
| `400` | [Error](#error) | Invalid product payload. |
| `404` | [Error](#error) | Feed not found. |

**Example request** (`application/json`, [UpsertProductsRequest](#upsertproductsrequest)):

```json
{
  "products": [
    {
      "id": "prod_01EXAMPLE",
      "variants": [{ "id": "var_01EXAMPLE", "title": "Example variant" }]
    }
  ]
}
```

**Example response** (`200`, [UpsertProductsResponse](#upsertproductsresponse)):

```json
{
  "id": "feed_01EXAMPLE",
  "accepted": true
}
```

---

## Request and response envelopes (top level)

Schemas below show **only top-level keys**. Nested objects link to dedicated sections.

### CreateFeedRequest

Optional settings when creating a feed.

| Field | Type | Notes |
|-------|------|--------|
| `target_country` | string | Optional ISO 3166-1 alpha-2 market code (`^[A-Z]{2}$`). |

---

### FeedMetadata

Server-managed feed resource descriptor.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Stable feed identifier. |
| `target_country` | string | no | ISO 3166-1 alpha-2, when applicable. |
| `updated_at` | string (date-time) | no | Last update time on the feed. |

---

### ProductsResponse

Envelope for listing all products on a feed.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `products` | array | yes | Full list of [Product](#product) objects. |

---

### UpsertProductsRequest

Partial update payload keyed by product id.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `products` | array | yes | Subset of [Product](#product) to create or update. |

---

### UpsertProductsResponse

Acknowledgement after a successful `PATCH` (defined in OpenAPI alongside the Feed API routes).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Feed id that was updated. |
| `accepted` | boolean | yes | Whether the upsert payload was accepted for processing. |

---

### Error

Structured error returned for failed requests.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | High-level category. |
| `code` | string | yes | Machine-readable code. |
| `message` | string | yes | Human-readable explanation. |
| `param` | string | no | Related field or parameter, when relevant. |

---

## Catalog entities

### Product

A catalog grouping of one or more purchasable variants.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Stable product identifier. |
| `title` | string | no | Display title. |
| `description` | object | no | [Description](#description). |
| `url` | string (URI) | no | Product detail URL. |
| `media` | array | no | [Media](#media) items for the product. |
| `variants` | array | yes | [Variant](#variant) list for this product. |

---

### Variant

Purchasable offer under a product.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Stable variant identifier (e.g. SKU-level id). |
| `title` | string | yes | Variant display title. |
| `description` | object | no | [Description](#description). |
| `url` | string (URI) | no | Variant PDP URL. |
| `barcodes` | array | no | [Barcode](#barcode) entries. |
| `price` | object | no | Selling [Price](#price). |
| `list_price` | object | no | Reference or pre-discount [Price](#price). |
| `unit_price` | object | no | Normalized [UnitPrice](#unitprice). |
| `availability` | object | no | [Availability](#availability). |
| `categories` | array | no | [Category](#category) assignments. |
| `condition` | array | no | [Condition](#condition) labels. |
| `variant_options` | array | no | Option dimensions ([VariantOption](#variantoption)). |
| `media` | array | no | Variant-level [Media](#media); first item is primary. |
| `seller` | object | no | Merchant of record ([Seller](#seller)). |
| `marketplace` | object | no | Intermediary platform ([Seller](#seller) shape). |

---

### Description

Long-form description in one or more formats; **at least one** of the text fields must be present.

| Field | Type | Notes |
|-------|------|--------|
| `plain` | string | Plain text. |
| `html` | string | HTML. |
| `markdown` | string | Markdown. |

---

### Price

Money in minor units with ISO 4217 currency.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `amount` | integer | yes | Amount in minor units (≥ 0). |
| `currency` | string | yes | Three-letter ISO code (`^[A-Z]{3}$`). |

---

### UnitPrice

Unit-normalized price (e.g. per 100 ml). Requires measure context.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `amount` | integer | yes | Price in minor units (≥ 0). |
| `currency` | string | yes | ISO 4217 code. |
| `measure` | object | yes | Actual package [Measure](#measure). |
| `reference` | object | yes | Display reference [ReferenceMeasure](#referencemeasure). |

---

### Measure

Numerical quantity with a unit label (e.g. weight or volume).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `value` | number | yes | Measured quantity. |
| `unit` | string | yes | Unit label (e.g. `oz`, `ml`, `kg`). |

---

### ReferenceMeasure

Reference quantity for normalizing displayed unit pricing.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `value` | integer | yes | Reference amount. |
| `unit` | string | yes | Reference unit label. |

---

### Availability

Purchasability and fulfillment hint for a variant.

| Field | Type | Notes |
|-------|------|--------|
| `available` | boolean | Whether the variant can be purchased. |
| `status` | string | Fulfillment state; known examples include `in_stock`, `limited_stock`, `backorder`, `preorder`, `out_of_stock`, `discontinued` (extensible). |

---

### Barcode

Scheme plus raw value (GTIN, UPC, etc.).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Scheme name (e.g. `GTIN`). |
| `value` | string | yes | Identifier value as provided by merchant. |

---

### Media

Retrievable asset (image, video, model, etc.).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Kind (`image`, `video`, …). |
| `url` | string (URI) | yes | Canonical asset URL. |
| `alt_text` | string | no | Accessible description. |
| `width` | integer | no | Pixel width when known. |
| `height` | integer | no | Pixel height when known. |

---

### VariantOption

One selected option axis (size, color, …).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name` | string | yes | Axis label (`Color`, `Size`, …). |
| `value` | string | yes | Selected value. |

---

### Category

Classification within an optional taxonomy.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `value` | string | yes | Label or hierarchical path (e.g. `Apparel > Shirts`). |
| `taxonomy` | string | no | System name (`google_product_category`, `merchant`, …). |

---

### Condition

Array of merchant-defined condition labels (e.g. `["new"]`).

---

### Seller

Merchant or marketplace identity; optional policy links.

| Field | Type | Notes |
|-------|------|--------|
| `name` | string | Display name. |
| `links` | array | [Link](#link) objects (policies, FAQ, …). |

---

### Link

Policy or informational URL with a typed role.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | e.g. `privacy_policy`, `terms_of_service`, `shipping_policy`. |
| `title` | string | no | Human-readable label. |
| `url` | string (URI) | yes | Resource URL. |

---

## Canonical sources

| Artifact | Path |
|----------|------|
| JSON Schema bundle | [`spec/2026-04-17/json-schema/schema.feed.json`](../../spec/2026-04-17/json-schema/schema.feed.json) |
| OpenAPI (HTTP + `UpsertProductsResponse`) | [`spec/2026-04-17/openapi/openapi.feed.yaml`](../../spec/2026-04-17/openapi/openapi.feed.yaml) |
