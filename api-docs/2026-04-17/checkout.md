# Agentic Checkout API & webhooks

**Specification version:** `2026-04-17`

This document describes the **Agentic Checkout** merchant REST API (creating and managing checkout sessions, completing or canceling them) and the **order webhooks** surface merchants call on OpenAI’s receiver so ChatGPT stays aligned with order lifecycle truth.

- Data shapes align with [`spec/2026-04-17/json-schema/schema.agentic_checkout.json`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json).
- Merchant HTTP paths and component schemas match [`spec/2026-04-17/openapi/openapi.agentic_checkout.yaml`](../../spec/2026-04-17/openapi/openapi.agentic_checkout.yaml).
- Webhook path, signing rules, and [`WebhookEvent`](#webhookevent) / [`EventDataOrder`](#eventdataorder) are defined in [`spec/2026-04-17/openapi/openapi.agentic_checkout_webhook.yaml`](../../spec/2026-04-17/openapi/openapi.agentic_checkout_webhook.yaml).

**Merchant API base URL:** `https://merchant.example.com` (example; substitute your host.)

**Webhook receiver base URL:** `https://openai.example.com` (example; use the URL OpenAI provides for production.)

Security on the merchant API is **`bearerAuth`** (Bearer API key). Successful mutating responses may echo **`Idempotency-Key`**, **`Idempotent-Replayed`**, and **`Request-Id`** per OpenAPI.

**Examples:** JSON snippets under each operation include **only fields marked `required` in OpenAPI** (including nested objects). Arrays such as `fulfillment_options`, `messages`, or `links` appear empty when the schema allows zero items.

**Protocol-level error body** (any `4XX`/`5XX` that returns [`Error`](#error)) minimal shape:

```json
{
  "type": "invalid_request",
  "code": "example_error_code",
  "message": "Human-readable explanation."
}
```

---

## Overview — merchant REST API

| Method | Path | Purpose |
|--------|------|--------|
| `POST` | `/checkout_sessions` | Create a session; server returns authoritative cart state (`201`). |
| `POST` | `/checkout_sessions/{checkout_session_id}` | Update items, fulfillment details, selected fulfillment options, discounts (`200`). |
| `GET` | `/checkout_sessions/{checkout_session_id}` | Retrieve latest session state (`200`). |
| `POST` | `/checkout_sessions/{checkout_session_id}/complete` | Apply payment and create an order (`200` → [`CheckoutSessionWithOrder`](#checkoutsessionwithorder)). |
| `POST` | `/checkout_sessions/{checkout_session_id}/cancel` | Cancel if not already completed or canceled (`200`; `405` if not cancelable). |

---

## Common request headers (merchant API)

OpenAPI defines shared headers on operations. Merchants should implement at least:

| Header | Required | Notes |
|--------|----------|--------|
| `Authorization` | yes | `Bearer <api_key>` |
| `API-Version` | yes | Protocol version string (e.g. snapshot date). |
| `Content-Type` | on bodies | `application/json` where a body is sent. |
| `Idempotency-Key` | on `POST` | Opaque string (max 255 chars); required for all `POST` routes in this spec. |
| `Accept-Language`, `User-Agent`, `Request-Id`, `Signature`, `Timestamp` | optional | Correlation, localization, and optional signing hooks per OpenAPI. |

Idempotency failures use structured [`Error`](#error) responses (`400` missing key, `409` in-flight, `422` conflict); see OpenAPI `components/responses`.

---

## POST `/checkout_sessions` — Create a checkout session

**Purpose:** Initializes a session from [`CheckoutSessionCreateRequest`](#checkoutsessioncreaterequest). The server **must** return `201` with a rich, authoritative [`CheckoutSession`](#checkoutsession).

Agents may send [`AffiliateAttribution`](#affiliateattribution) with `touchpoint: "first"` for first-touch attribution.

**Request body (`application/json`):** [`CheckoutSessionCreateRequest`](#checkoutsessioncreaterequest)

| Status | Body | Meaning |
|--------|------|---------|
| `201` | [`CheckoutSession`](#checkoutsession) | Created. |
| `400` | [`Error`](#error) | Including idempotency key required (see OpenAPI). |
| `409` / `422` | [`Error`](#error) | Idempotency in-flight / payload mismatch. |
| `4XX` / `5XX` | [`Error`](#error) | Other client or server errors. |

**Example request** (`application/json`, required fields only):

```json
{
  "line_items": [{ "id": "item_001" }],
  "currency": "usd",
  "capabilities": {}
}
```

**Example response** (`201`, [`CheckoutSession`](#checkoutsession), required fields only):

```json
{
  "id": "cs_01EXAMPLE",
  "status": "incomplete",
  "currency": "usd",
  "line_items": [
    {
      "id": "li_001",
      "item": { "id": "item_001" },
      "quantity": 1,
      "totals": [
        { "type": "total", "display_text": "Total", "amount": 100 }
      ]
    }
  ],
  "totals": [{ "type": "total", "display_text": "Total", "amount": 100 }],
  "fulfillment_options": [],
  "messages": [],
  "links": [],
  "capabilities": {}
}
```

---

## POST `/checkout_sessions/{checkout_session_id}` — Update a checkout session

**Purpose:** Applies changes (items, fulfillment address, fulfillment option selection, discounts) and returns updated [`CheckoutSession`](#checkoutsession).

**Path parameters:**

| Name | Description |
|------|-------------|
| `checkout_session_id` | Session id returned at creation. |

**Request body (`application/json`):** [`CheckoutSessionUpdateRequest`](#checkoutsessionupdaterequest)

| Status | Body | Meaning |
|--------|------|---------|
| `200` | [`CheckoutSession`](#checkoutsession) | Updated. |
| `400` | [`Error`](#error) | Invalid payload or idempotency key issues. |
| `409` / `422` | [`Error`](#error) | Idempotency collisions. |
| `4XX` / `5XX` | [`Error`](#error) | Other errors. |

**Example request** (`application/json`): no fields are required on [`CheckoutSessionUpdateRequest`](#checkoutsessionupdaterequest); send an empty object when sending only headers:

```json
{}
```

**Example response** (`200`): same required-field shape as [create response](#post-checkout_sessions--create-a-checkout-session) (session ids and amounts reflect server state).

---

## GET `/checkout_sessions/{checkout_session_id}` — Retrieve a checkout session

**Purpose:** Returns the latest authoritative state.

**Path parameters:**

| Name | Description |
|------|-------------|
| `checkout_session_id` | Session id. |

| Status | Body | Meaning |
|--------|------|---------|
| `200` | [`CheckoutSession`](#checkoutsession) | Success. |
| `404` | [`Error`](#error) | Session not found. |

**Example response** (`200`, required fields only): same structure as the [create session example response](#post-checkout_sessions--create-a-checkout-session).

---

## POST `/checkout_sessions/{checkout_session_id}/complete` — Complete checkout

**Purpose:** Finalizes checkout with [`CheckoutSessionCompleteRequest`](#checkoutsessioncompleterequest); creates an **order** on success. Attribution sent here may use `touchpoint: "last"`; it is stored with the order and **not** echoed in the response.

**Request body (`application/json`):** [`CheckoutSessionCompleteRequest`](#checkoutsessioncompleterequest)

| Status | Body | Meaning |
|--------|------|---------|
| `200` | [`CheckoutSessionWithOrder`](#checkoutsessionwithorder) | Completed with embedded [`Order`](#order). |
| `400` | [`Error`](#error) | Validation / idempotency key issues. |
| `409` / `422` | [`Error`](#error) | Idempotency collisions. |
| `4XX` / `5XX` | [`Error`](#error) | Other errors. |

**Example request** (`application/json`): only [`payment_data`](#paymentdata) is required. OpenAPI allows either **`handler_id` + `instrument`** or **`purchase_order_number`** (mutually exclusive).

Handler + instrument (`instrument.type`, `instrument.credential.type`, `instrument.credential.token` required):

```json
{
  "payment_data": {
    "handler_id": "handler_001",
    "instrument": {
      "type": "card",
      "credential": { "type": "spt", "token": "tok_01EXAMPLE" }
    }
  }
}
```

Purchase order branch:

```json
{
  "payment_data": {
    "purchase_order_number": "PO-01EXAMPLE"
  }
}
```

**Example response** (`200`, [`CheckoutSessionWithOrder`](#checkoutsessionwithorder), required fields only): required [`CheckoutSession`](#checkoutsession) fields plus [`Order`](#order) fields `id`, `checkout_session_id`, and `permalink_url`:

```json
{
  "id": "cs_01EXAMPLE",
  "status": "completed",
  "currency": "usd",
  "line_items": [
    {
      "id": "li_001",
      "item": { "id": "item_001" },
      "quantity": 1,
      "totals": [
        { "type": "total", "display_text": "Total", "amount": 100 }
      ]
    }
  ],
  "totals": [{ "type": "total", "display_text": "Total", "amount": 100 }],
  "fulfillment_options": [],
  "messages": [],
  "links": [],
  "capabilities": {},
  "order": {
    "id": "ord_01EXAMPLE",
    "checkout_session_id": "cs_01EXAMPLE",
    "permalink_url": "https://merchant.example.com/orders/ord_01EXAMPLE"
  }
}
```

---

## POST `/checkout_sessions/{checkout_session_id}/cancel` — Cancel a session

**Purpose:** Cancels if not already completed or canceled. Optional [`CancelSessionRequest`](#cancelsessionrequest) may include an [`IntentTrace`](#intenttrace).

**Request body (`application/json`):** optional [`CancelSessionRequest`](#cancelsessionrequest)

| Status | Body | Meaning |
|--------|------|---------|
| `200` | [`CheckoutSession`](#checkoutsession) | Canceled. |
| `405` | [`Error`](#error) | Not cancelable (completed/canceled). |
| `400` / `409` / `422` | [`Error`](#error) | Bad request or idempotency errors. |
| `4XX` / `5XX` | [`Error`](#error) | Other errors. |

**Example request** (`application/json`): body is optional; omit entirely or send `{}`. When including only required [`IntentTrace`](#intenttrace) fields:

```json
{
  "intent_trace": {
    "reason_code": "other"
  }
}
```

**Example response** (`200`, [`CheckoutSession`](#checkoutsession), required fields only): same JSON shape as [create response](#post-checkout_sessions--create-a-checkout-session), with `status` such as `canceled` per server rules.

---

## POST `/agentic_checkout/webhooks/order_events` — Order lifecycle webhooks

**Host:** OpenAI’s webhook receiver (see webhook OpenAPI `servers`).

**Purpose:** Merchants **push** full [`Order`](#order) snapshots as [`WebhookEvent`](#webhookevent) payloads so agents stay in sync with fulfillment-grade truth. The `data` object **must** contain the full order (not deltas).

**Signing:** Requests **must** include header **`Merchant-Signature`**: `t=<unix_seconds>,v1=<64_hex>`. Signed payload is `timestamp + "." + raw_body`; **HMAC-SHA256** with a shared secret. Reject `401` if missing, malformed, outside timestamp window (~300s skew recommended), or verification fails.

**Migration note (webhook spec):** `EventDataOrder` composes the full checkout [`Order`](#order) schema. Legacy `refunds[]` is removed; use `adjustments[]` on `Order` for refunds and related post-order changes.

**Headers (webhook):**

| Header | Required | Notes |
|--------|----------|--------|
| `Merchant-Signature` | yes | Pattern `^t=\d+,v1=[a-fA-F0-9]{64}$` |
| `Content-Type` | yes | `application/json` |
| `Request-Id`, `Timestamp` | optional | Tracing / timing |

**Request body (`application/json`):** [`WebhookEvent`](#webhookevent)

| Status | Body | Meaning |
|--------|------|---------|
| `200` | `{ "received": boolean, "request_id"?: string }` | Event accepted (`received` required). |
| `400` | [`Error`](#error) | Bad payload. |
| `401` | [`Error`](#error) | Signature missing/invalid/expired window. |
| `429` | [`Error`](#error) | Rate limited. |
| `500` | [`Error`](#error) | Server error. |

**Example request** (`application/json`, [`WebhookEvent`](#webhookevent)): [`EventDataOrder`](#eventdataorder) requires `type`, `id`, `checkout_session_id`, `permalink_url`, and `status` on `data`:

```json
{
  "type": "order_create",
  "data": {
    "type": "order",
    "id": "ord_01EXAMPLE",
    "checkout_session_id": "cs_01EXAMPLE",
    "permalink_url": "https://merchant.example.com/orders/ord_01EXAMPLE",
    "status": "created"
  }
}
```

**Example response** (`200`, required fields only):

```json
{
  "received": true
}
```

---

## Request payloads (top level)

Schemas below emphasize **top-level keys**. Nested shapes link to sections or to the JSON Schema bundle for exhaustive detail.

### CheckoutSessionCreateRequest

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `line_items` | array | yes | Min 1 entry; each element is an [`Item`](#item). |
| `currency` | string | yes | ISO 4217 code. |
| `capabilities` | object | yes | [`Capabilities`](#capabilities) — payment handlers, interventions, extensions. |
| `buyer` | object | no | [`Buyer`](#buyer). |
| `fulfillment_details` | object | no | [`FulfillmentDetails`](#fulfillmentdetails). |
| `fulfillment_groups` | array | no | Groups [`FulfillmentGroup`](#fulfillmentgroup) (OpenAPI). |
| `affiliate_attribution` | object | no | [`AffiliateAttribution`](#affiliateattribution) (write-only in protocol). |
| `discounts` | object | no | [`DiscountsRequest`](#discountsrequest) — preferred for codes. |
| `coupons` | array of string | no | **Deprecated**; use `discounts.codes`. |
| `locale`, `timezone` | string | no | Localization hints. |
| `quote_id`, `metadata` | string / object | no | Quote linkage and merchant metadata. |

---

### CheckoutSessionUpdateRequest

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `buyer` | object | no | [`Buyer`](#buyer). |
| `line_items` | array | no | [`Item`](#item) entries to update the cart. |
| `fulfillment_details` | object | no | [`FulfillmentDetails`](#fulfillmentdetails). |
| `fulfillment_groups` | array | no | Updated grouping. |
| `selected_fulfillment_options` | array | no | Selected option(s); see OpenAPI `SelectedFulfillmentOption`. |
| `discounts` | object | no | [`DiscountsRequest`](#discountsrequest) — replaces prior codes if sent. |
| `coupons` | array of string | no | **Deprecated**; use `discounts.codes`. |

---

### CheckoutSessionCompleteRequest

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `payment_data` | object | yes | [`PaymentData`](#paymentdata). |
| `buyer` | object | no | [`Buyer`](#buyer). |
| `authentication_result` | object | no | 3DS result (OpenAPI `AuthenticationResult`). |
| `affiliate_attribution` | object | no | Last-touch attribution (write-only). |
| `risk_signals` | object | no | Fraud signals (OpenAPI `RiskSignals`). |
| `marketing_consents` | array | no | Entries aligned with session `marketing_consent_options`. |

---

### CancelSessionRequest

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `intent_trace` | object | no | [`IntentTrace`](#intenttrace). |

---

### CheckoutSession

Session responses use **`CheckoutSessionBase`** (OpenAPI `CheckoutSession`).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Session id. |
| `status` | string | yes | e.g. `incomplete`, `ready_for_payment`, `completed`, `canceled`, … (full enum in OpenAPI). |
| `currency` | string | yes | Settlement currency (ISO 4217). |
| `line_items` | array | yes | [`LineItem`](#lineitem) rows (authoritative cart). |
| `totals` | array | yes | [`Total`](#total) breakdown lines. |
| `fulfillment_options` | array | yes | Available options (`shipping`, `digital`, `pickup`, `local_delivery` discriminated unions). |
| `messages` | array | yes | `MessageInfo` \| `MessageWarning` \| `MessageError`. |
| `links` | array | yes | [`Link`](#link) entries (policies, support). |
| `capabilities` | object | yes | Effective [`Capabilities`](#capabilities) / extensions intersection. |
| `protocol` | object | no | `ProtocolVersion`. |
| `buyer` | object | no | [`Buyer`](#buyer). |
| `presentment_currency`, `exchange_rate`, `exchange_rate_timestamp` | various | no | Multi-currency presentation. |
| `locale`, `timezone` | string | no | Localization. |
| `fulfillment_details` | object | no | [`FulfillmentDetails`](#fulfillmentdetails). |
| `selected_fulfillment_options` | array | no | Buyer’s selection(s). |
| `fulfillment_groups` | array | no | Optional grouping. |
| `authentication_metadata` | object | no | Seller 3DS hints (`AuthenticationMetadata`). |
| `created_at`, `updated_at`, `expires_at` | string (date-time) | no | Lifecycle timestamps. |
| `continue_url` | string (URI) | no | Resume checkout URL. |
| `metadata` | object | no | Merchant-defined. |
| `quote_id`, `quote_expires_at` | string | no | Quote linkage. |
| `marketing_consent_options` | array | no | Channels requiring consent UI (`MarketingConsentOption`). |
| `discounts` | object | no | [`DiscountsResponse`](#discountsresponse) when `discount` extension active. |

---

### CheckoutSessionWithOrder

Extends [`CheckoutSession`](#checkoutsession) with:

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `order` | object | yes | [`Order`](#order) created by completion. |

---

### WebhookEvent

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | e.g. `order_create`, `order_update`; implementations should tolerate unknown values. |
| `data` | object | yes | [`EventDataOrder`](#eventdataorder). |

---

### EventDataOrder

Webhook `data` is the full **`Order`** schema with extra webhook requirements: **`type`**, **`checkout_session_id`**, **`permalink_url`**, **`status`** are required (see webhook OpenAPI `EventDataOrder`). Prefer including optional order fields (`line_items`, `fulfillments`, `adjustments`, `totals`) when available.

---

## Core entities

### Item

Agent → merchant request line shape (nested inside session `line_items` via OpenAPI; creation/update arrays use `Item`).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Stable line identifier. |
| `name` | string | no | Display name. |
| `unit_amount` | integer | no | Minor units per unit. |

---

### LineItem

Merchant → agent authoritative cart row.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Line id. |
| `item` | object | yes | [`Item`](#item). |
| `quantity` | integer | yes | ≥ 1. |
| `totals` | array | yes | [`Total`](#total) lines for this row. |
| `name`, `description`, `images` | string / array | no | Presentation. |
| `unit_amount` | integer | no | Minor units. |
| `product_id`, `sku`, `variant_id` | string | no | Catalog linkage. |
| `variant_options` | array | no | [`VariantOption`](#variantoption). |
| `weight`, `dimensions` | object | no | `WeightInfo`, `DimensionsInfo`. |
| `availability_status`, `available_quantity`, … | various | no | Inventory hints (OpenAPI). |
| `disclosures`, `custom_attributes` | array | no | Compliance / merchant extensions. |
| `marketplace_seller_details` | object | no | Third-party seller label. |
| `discount_details` | array | no | [`DiscountDetail`](#discountdetail). |

---

### Buyer

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `email` | string (email) | yes | Primary contact. |
| `first_name`, `last_name`, `full_name`, `phone_number` | string | no | Identity / reachability. |
| `customer_id` | string | no | Merchant customer id. |
| `account_type` | string | no | `guest`, `registered`, `business`. |
| `authentication_status` | string | no | `authenticated`, `guest`, `requires_signin`. |
| `company`, `loyalty`, `tax_exemption` | object | no | B2B / loyalty / tax (OpenAPI nested schemas). |

---

### FulfillmentDetails

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name`, `phone_number`, `email` | string | no | Contact; E.164 recommended for phone. |
| `address` | object | no | [`Address`](#address). |

---

### Address

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name`, `line_one`, `city`, `state`, `country`, `postal_code` | string | yes | `country` ISO 3166-1 alpha-2. |
| `line_two`, `company` | string | no | Optional lines. |

---

### PaymentData

Sent as [`CheckoutSessionCompleteRequest`](#checkoutsessioncomplete).`payment_data`. Describes how the buyer pays: either a **handler + instrument** (vault/tokenized credential) or a **purchase order**. **`additionalProperties` is false.**

Canonical definitions: [`$defs.PaymentData`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json), OpenAPI [`PaymentData`](../../spec/2026-04-17/openapi/openapi.agentic_checkout.yaml).

#### Required shape (`anyOf`)

Exactly **one** of these branches must be satisfied (schema **`anyOf`**):

| Branch | Required properties | Purpose |
|--------|---------------------|--------|
| **Handler + instrument** | `handler_id`, `instrument` | Use the seller-advertised [`PaymentHandler`](#paymenthandler); `handler_id` must match that handler’s `id`. |
| **Purchase order** | `purchase_order_number` | B2B-style PO reference without `handler_id` / `instrument`. |

Provide **either** (`handler_id` + `instrument`) **or** (`purchase_order_number`). Optional fields (`billing_address`, `payment_terms`, `due_date`, `approval_required`) may appear alongside whichever branch applies.

#### Top-level properties

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `handler_id` | string | **Branch: handler + instrument** | Payment handler id from [`Capabilities.payment.handlers`](#payment). |
| `instrument` | object | **Branch: handler + instrument** | See [Instrument](#instrument). |
| `purchase_order_number` | string | **Branch: PO** | Purchase order identifier. |
| `billing_address` | object | no | [`Address`](#address) for billing when needed. |
| `payment_terms` | string | no | B2B terms: `immediate`, `net_15`, `net_30`, `net_60`, `net_90`. |
| `due_date` | string (date-time) | no | RFC 3339 when payment is due. |
| `approval_required` | boolean | no | Whether payment needs approval before capture. |

#### Instrument

Used only with **`handler_id`** on the handler branch. Object with **`type`** and **`credential`** only (per schema).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Instrument category (e.g. `card`, `wallet_token`; open string in schema). |
| `credential` | object | yes | See [Credential](#credential). |

#### Credential

Nested under **`instrument`**.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Credential kind (e.g. `spt`, `wallet_token`). |
| `token` | string | yes | Opaque token value (vault token, PSP token, etc.). |

The bundled schema’s **`example`** on `PaymentData` is a flat `type` / `token` pair and does **not** match the normative `handler_id` + `instrument.credential` structure; follow the property tables and `anyOf` above.

---

### Capabilities

Top-level object exchanged on session **create**/**update** requests (agent) and in every [`CheckoutSession`](#checkoutsession) response (seller). **`additionalProperties` is false** — only the keys below are allowed. Shapes match [`$defs.Capabilities`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json) in the bundle.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `payment` | object | no | [`Payment`](#payment). When present, sellers advertise accepted handlers. |
| `interventions` | object | no | [`InterventionCapabilities`](#interventioncapabilities). Request vs response semantics differ (see that section). |
| `extensions` | array | no | **Union (`oneOf`):** either `string[]` (agent) or [`ExtensionDeclaration`](#extensiondeclaration) objects (seller). See [Extensions (oneOf)](#extensions-oneof). |

On **requests**, omit keys you are not declaring; `{}` is valid. On **responses**, the seller returns its supported payment handlers and the **intersection** of intervention support with what the agent offered.

#### Extensions (oneOf)

Exactly one array **shape** applies per message (schema `oneOf` over array item types):

| Variant | `items` element | When |
|---------|-----------------|------|
| **Agent request** | `string` | Extension ids the agent understands (e.g. `discount`, `affiliate_attribution`). **`uniqueItems`: true.** |
| **Seller response** | [`ExtensionDeclaration`](#extensiondeclaration) | Active extensions for this session (`name`, optional `extends` / `schema` / `spec`). **`uniqueItems`: true.** |

---

### Payment

Nested under `Capabilities.payment` ([`Capabilities`](#capabilities)).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `handlers` | array | yes | Each item is a [`PaymentHandler`](#paymenthandler). May be `[]` when no handlers are listed. |

Defined in [`$defs.Payment`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json).

---

### PaymentHandler

Each element of [`Payment.handlers`](#payment).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Seller-defined handler id (e.g. matches [`PaymentData`](#paymentdata) `handler_id`). |
| `name` | string | yes | Reverse-DNS handler name (e.g. `dev.acp.tokenized.card`). |
| `version` | string | yes | **`YYYY-MM-DD`** (`pattern` in schema). |
| `spec` | string (URI) | yes | URL of the handler specification. |
| `requires_delegate_payment` | boolean | yes | Whether delegate-payment flow is required. |
| `requires_pci_compliance` | boolean | yes | Whether the handler routes PCI-sensitive data. |
| `psp` | string | yes | Payment Service Provider identifier. |
| `config_schema` | string (URI) | yes | JSON Schema URL for `config`. |
| `instrument_schemas` | array of URI | yes | Schema URLs for accepted payment instruments (schema does not set `minItems`; `[]` is syntactically allowed). |
| `config` | object | yes | Handler-specific configuration object. |
| `display_name` | string | no | UI label (e.g. “Credit card”). |
| `display_order` | integer | no | Suggested ordering (lower first); agent may reorder. |

Defined in [`$defs.PaymentHandler`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json).

---

### InterventionCapabilities

Nested under `Capabilities.interventions` ([`Capabilities`](#capabilities)). Same object appears on agent requests and seller responses; **which fields are set depends on direction** (see [`$defs.InterventionCapabilities`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json)).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `supported` | array of string | no | Values: `3ds`, `biometric`, `address_verification`. **Request:** interventions the agent can run. **Response:** **intersection** with seller support. |
| `required` | array of string | no | Values: `3ds`, `biometric`. **Seller response only** — interventions required for this session. |
| `enforcement` | string | no | **Seller only:** `always`, `conditional`, `optional`. |
| `display_context` | string | no | **Agent only:** `native`, `webview`, `modal`, `redirect`. |
| `redirect_context` | string | no | **Agent only:** `in_app`, `external_browser`, `none`. |
| `max_redirects` | integer | no | **Agent only:** ≥ 0. |
| `max_interaction_depth` | integer | no | **Agent only:** ≥ 1. |

Defined in [`$defs.InterventionCapabilities`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json).

---

### ExtensionDeclaration

Used as **`Capabilities.extensions[]` entries on seller responses** when that array is not the agent’s plain-string form.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name` | string | yes | Extension id; matches schema `pattern` (simple slug or dotted reverse-DNS form; optional `@YYYY-MM-DD` suffix). |
| `extends` | array of string | no | JSONPaths of fields this extension adds (pattern `$.SchemaName.field`). **`uniqueItems`: true.** |
| `schema` | string (URI) | no | Extension JSON Schema URL. |
| `spec` | string (URI) | no | Extension human spec URL. |

Defined in [`$defs.ExtensionDeclaration`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json).

---

### DiscountsRequest

| Field | Type | Notes |
|-------|------|--------|
| `codes` | array of string | Case-insensitive codes; replaces prior set; empty array clears. |

---

### DiscountsResponse

| Field | Type | Notes |
|-------|------|--------|
| `codes` | array of string | Echo of submitted codes. |
| `applied` | array | `AppliedDiscount` entries. |
| `rejected` | array | `RejectedDiscount` entries with reasons/codes. |

---

### VariantOption

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name` | string | yes | Axis (e.g. `Size`). |
| `value` | string | yes | Selected value. |

---

### DiscountDetail

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | `percentage`, `fixed`, `bogo`, `volume`. |
| `amount` | integer | yes | Minor units. |
| `code`, `description`, `source` | string | no | `source`: `coupon`, `automatic`, `loyalty`. |

---

### Total

Cart or order breakdown row.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | e.g. `subtotal`, `tax`, `fulfillment`, `total`, `amount_refunded`, … |
| `display_text` | string | yes | Localized label. |
| `amount` | integer | yes | Minor units. |
| `presentment_amount`, `description`, `breakdown` | various | no | Presentation / tax breakdown (`TaxBreakdownItem`). |

---

### FulfillmentGroup

| Field | Type | Notes |
|-------|------|--------|
| `id` | string | Group id. |
| `item_ids` | array | Line/item ids in this group (OpenAPI). |

---

### Order

Persisted order returned on completion and in webhooks.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | no | Discriminator: `order` when present (webhooks). |
| `id` | string | yes | Order id. |
| `checkout_session_id` | string | yes | Originating session. |
| `permalink_url` | string (URI) | yes | Buyer-facing order URL. |
| `order_number` | string | no | Human-readable number. |
| `status` | string | no | Order lifecycle (`created`, `confirmed`, `processing`, `shipped`, `completed`, `canceled`, …); accept unknown values. |
| `line_items` | array | no | [`OrderLineItem`](#orderlineitem). |
| `fulfillments` | array | no | Ship/pickup/digital progress (`Fulfillment`). |
| `adjustments` | array | no | Refunds, credits, disputes (`Adjustment`). |
| `totals` | array | no | [`Total`](#total) lines (e.g. include `amount_refunded` when relevant). |
| `estimated_delivery`, `confirmation`, `support` | object | no | Post-purchase UX helpers (OpenAPI). |

---

### OrderLineItem

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Stable line id (referenced by fulfillments/adjustments). |
| `title` | string | yes | Product title. |
| `quantity` | object | yes | [`OrderLineItemQuantity`](#orderlineitemquantity). |
| `product_id`, `description`, `image_url`, `url` | various | no | Catalog/display. |
| `unit_price`, `subtotal` | integer | no | Minor units. |
| `totals` | array | no | Optional richer per-line breakdown. |
| `status` | string | no | Derived fulfillment progress (`processing`, `partial`, `fulfilled`, `removed`, …). |

---

### OrderLineItemQuantity

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `ordered` | integer | yes | Original quantity (≥ 1). |
| `current` | integer | yes | Active quantity after cancellations/returns (≥ 0). |
| `fulfilled` | integer | no | Fulfilled count (defaults per OpenAPI). |

---

### Link

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Link role (`terms_of_use`, `privacy_policy`, `return_policy`, `shipping_policy`, `contact_us`, `about_us`, `faq`, `support`). |
| `url` | string (URI) | yes | Destination URL. |
| `title` | string | no | Display text. |

---

### AffiliateAttribution

Write-only attribution envelope (session create / complete). Requires **`provider`** and either **`token`** or **`publisher_id`**. Optional: `campaign_id`, `creative_id`, `sub_id`, `source`, `issued_at`, `expires_at`, `metadata`, `touchpoint` (`first` \| `last`). Not returned on checkout responses.

---

### IntentTrace

Optional cancel/analytics payload.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `reason_code` | string | yes | Known reasons include `price_sensitivity`, `shipping_cost`, `timing_deferred`, `other`, …; servers should accept unknown values leniently. |
| `trace_summary` | string | no | Short narrative (max 500 chars). |
| `metadata` | object | no | Flat string/number/boolean values only. |

---

### Error

Protocol-level error when the server cannot return valid session state.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | `invalid_request`, `processing_error`, `service_unavailable`. |
| `code` | string | yes | Machine-readable (includes idempotency codes per OpenAPI). |
| `message` | string | yes | Human-readable. |
| `param` | string | no | JSONPath to offending field when applicable. |

Use **`MessageError`** inside `CheckoutSession.messages` for buyer-visible cart issues—not this envelope—for cases where a valid session still exists (see OpenAPI descriptions).

---

## Canonical sources

| Artifact | Path |
|----------|------|
| JSON Schema bundle | [`spec/2026-04-17/json-schema/schema.agentic_checkout.json`](../../spec/2026-04-17/json-schema/schema.agentic_checkout.json) |
| OpenAPI — merchant checkout HTTP API | [`spec/2026-04-17/openapi/openapi.agentic_checkout.yaml`](../../spec/2026-04-17/openapi/openapi.agentic_checkout.yaml) |
| OpenAPI — order webhooks receiver | [`spec/2026-04-17/openapi/openapi.agentic_checkout_webhook.yaml`](../../spec/2026-04-17/openapi/openapi.agentic_checkout_webhook.yaml) |
