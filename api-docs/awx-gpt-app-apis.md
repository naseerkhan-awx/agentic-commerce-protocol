# AWX GPT App — MCP checkout APIs

## MCP tools

| Tool name | Request | Response |
|-----------|---------|----------|
| `create_checkout` | [`CheckoutSessionRequest`](#checkoutsessionrequest) — do **not** send `id` | [`CheckoutSession`](#checkoutsession) |
| `update_checkout` | [`CheckoutSessionRequest`](#checkoutsessionrequest) — same body; **`id` required** (session from prior response) | [`CheckoutSession`](#checkoutsession) |
| `get_checkout` | [`CheckoutSessionIdRequest`](#checkoutsessionidrequest) | [`CheckoutSession`](#checkoutsession) |
| `complete_checkout` | [`CheckoutSessionIdRequest`](#checkoutsessionidrequest) | [`CheckoutSessionWithOrder`](#checkoutsessionwithorder) |
| `cancel_checkout` | [`CheckoutSessionIdRequest`](#checkoutsessionidrequest) | [`CheckoutSession`](#checkoutsession) |
| `get_customer_payment` | Same body as [Airwallex List payment methods](https://www.airwallex.com/docs/api/payments/payment_methods/list) **without `customer_id`** ([details](#payment-form)) | Same as that API’s response |

---

## Payment Form

### `get_customer_payment`

Returns the customer’s saved payment methods for rendering the payment step.

**Request and response** use the same JSON shape as Airwallex’s [List payment methods](https://www.airwallex.com/docs/api/payments/payment_methods/list) API.

The request **must not** include **`customer_id`**. The server derives the customer from the **auth token** and supplies the identifier when calling the underlying service.

---

## Frontend (FE) integration

### When to call `create_checkout` vs `update_checkout`

Both tools use the same request object, [`CheckoutSessionRequest`](#checkoutsessionrequest). **`create_checkout`** is used when there is no session yet (**omit `id`**). **`update_checkout`** updates an existing session (**include `id`** from the last [`CheckoutSession`](#checkoutsession) response).

**1. `create_checkout` — new session**

After **items are selected** and the FE has collected **`buyer`** and **`fulfillment_details`** (for example via checkout forms), call **`create_checkout`**:

- Set **`line_items`**, **`currency`**, **`buyer`**, and **`fulfillment_details`** from the UI state.
- **Do not send `id`**.
- **Omit `fulfillment_groups`** unless your flow must send grouping before options are known.

Store the returned **`id`** from [`CheckoutSession`](#checkoutsession) for **`update_checkout`**.

**2. Present fulfillment options**

Use **`fulfillment_options`** from the response to drive the next UI step. Each element is one of `shipping`, `digital`, `pickup`, or `local_delivery` (see [Fulfillment options](#fulfillment-options)); use **`id`**, **`title`**, **`description`**, type-specific fields (carrier, windows, pickup `location`, etc.), and per-option **`totals`**. Always render **`messages`** (info / warning / error) so the buyer sees stock, policy, or validation issues before confirming fulfillment choices.

**3. `update_checkout` — fulfillment grouping**

After the buyer chooses **which line items** are fulfilled **with which** option (partition the cart by fulfillment path), call **`update_checkout`**:

- Set **`id`** to the session **`id`** from step 1.
- Resend **`line_items`**, **`currency`**, **`buyer`**, and **`fulfillment_details`** consistent with what the session should reflect (same as before, or edited values if the UI allowed changes).
- Populate **`fulfillment_groups`**: each [`FulfillmentGroup`](#fulfillmentgroup) lists **`item_ids`** that share one fulfillment path (identifiers must match the stable line ids used in request [`Item.id`](#item) and echoed on response [`LineItem`](#lineitem)), with **`destination_type`** aligned to the chosen option’s `type`, and any optional group fields (e.g. **`location_id`** for pickup / local delivery, per OpenAPI).

The response returns an updated [`CheckoutSession`](#checkoutsession) (refreshed **`totals`**, **`status`**, **`fulfillment_options`**, **`messages`**, **`links`**) for the next UI state (e.g. payment).

---

## Object shapes

### `CheckoutSessionRequest`

JSON body for **`create_checkout`** and **`update_checkout`**.

**`id`:** omit for **`create_checkout`**; required for **`update_checkout`** (value is [`CheckoutSession.id`](#checkoutsession) from a previous successful call).

**All other fields** are required except **`fulfillment_groups`**, which is optional until the buyer has chosen how each line is fulfilled (see [Frontend integration](#frontend-fe-integration)).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | Update only | **Create:** omit. **Update:** required. Session id from the last response. |
| `line_items` | array | yes | Min 1 entry; each element is an [`Item`](#item). |
| `currency` | string | yes | ISO 4217 code. |
| `buyer` | object | yes | [`Buyer`](#buyer). |
| `fulfillment_details` | object | yes | [`FulfillmentDetails`](#fulfillmentdetails). |
| `fulfillment_groups` | array | no | [`FulfillmentGroup`](#fulfillmentgroup) entries. Typically omitted on **`create_checkout`**; sent on **`update_checkout`** once the buyer maps lines to fulfillment choices. |

---

### `CheckoutSessionIdRequest`

JSON body for **`get_checkout`**, **`complete_checkout`**, and **`cancel_checkout`**. Identifies the session by id only.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Checkout session id (same as [`CheckoutSession.id`](#checkoutsession)). |

---

### `CheckoutSession`

Response for **`create_checkout`**, **`update_checkout`**, **`get_checkout`**, and **`cancel_checkout`**. Slim session: only the fields below (protocol session fields from `capabilities` through `discounts` are not returned). **`complete_checkout`** returns [`CheckoutSessionWithOrder`](#checkoutsessionwithorder) instead (same session fields plus **`order`**).

**Included fields** — all are **required** on this response:

| Field | Type | Notes |
|-------|------|--------|
| `id` | string | Session id. |
| `status` | string | Allowed values: [checkout session status](#checkout-session-status-values). |
| `currency` | string | Settlement currency (ISO 4217). |
| `line_items` | array | [`LineItem`](#lineitem) rows (authoritative cart). |
| `totals` | array | [`Total`](#total) breakdown lines. |
| `fulfillment_options` | array | Each element is one [fulfillment option](#fulfillment-options) variant (`shipping`, `digital`, `pickup`, `local_delivery`). |
| `messages` | array | Each element is [`MessageInfo`](#messageinfo) \| [`MessageWarning`](#messagewarning) \| [`MessageError`](#messageerror) ([session messages](#session-messages)). |
| `links` | array | [`Link`](#link) entries (policies, support). |

#### Checkout session `status` values

Per OpenAPI `CheckoutSessionBase`, `status` MUST be one of:

- `incomplete`
- `not_ready_for_payment`
- `requires_escalation`
- `authentication_required`
- `ready_for_payment`
- `pending_approval`
- `complete_in_progress`
- `completed`
- `canceled`
- `in_progress`
- `expired`

---

### `CheckoutSessionWithOrder`

Response for **`complete_checkout`**. Everything in [`CheckoutSession`](#checkoutsession), plus the persisted **`order`** created by completion.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `order` | object | yes | [`Order`](#order). |

All fields from [`CheckoutSession`](#checkoutsession) (`id`, `status`, `currency`, `line_items`, `totals`, `fulfillment_options`, `messages`, `links`) are present with the same meanings as after a successful completion (for example `status` is typically `completed`).

---

### Nested schemas

Unless stated, **`additionalProperties`** follow OpenAPI (`openapi.agentic_checkout.yaml`, often `false`).

#### `Item`

Request line shape (used in **`CheckoutSessionRequest`.`line_items`**).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Stable line identifier. |
| `name` | string | no | Display name. |
| `unit_amount` | integer | no | Minor units per unit. |

---

#### `LineItem`

Authoritative cart row (used in **`CheckoutSession`.`line_items`**).

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
| `availability_status` | string | no | Seller’s stock signal for this line: `in_stock` (can fulfill now), `low_stock` (still purchasable but constrained—pair with `available_quantity` / messaging), `out_of_stock` (cannot fulfill at current quantity), `backorder` (may purchase but ships when restocked), `pre_order` (sale before ready to ship). Agents should treat this with `available_quantity` and session `messages[]` when the seller surfaces warnings or blocks. |
| `available_quantity` | integer | no | Units the seller reports as purchasable **now** (≥ 0). Use with `quantity` on the line to avoid oversell: if present and less than requested quantity, expect `not_ready_for_payment` / corrective `MessageError` or warning codes such as `low_stock` / `quantity_exceeded` in `messages[]`. |
| `max_quantity_per_order` | integer | no | Per-order cap for this SKU/line (≥ 1 when present). Buyers or agents must not exceed this without a cart update; exceeding may yield validation-style session messages. |
| `fulfillable_on` | string (date-time) | no | RFC 3339 timestamp when the seller expects the item to be fulfillable (e.g. pre-order / backorder readiness). Useful for setting buyer expectations when `availability_status` is `pre_order` or `backorder`. |
| `disclosures`, `custom_attributes` | array | no | Compliance / merchant extensions. |
| `marketplace_seller_details` | object | no | Third-party seller label. |
| `discount_details` | array | no | [`DiscountDetail`](#discountdetail). |

---

#### `Order`

Persisted order returned on **`complete_checkout`** (nested under [`CheckoutSessionWithOrder`](#checkoutsessionwithorder)).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | no | Discriminator: `order` when present (e.g. webhooks). |
| `id` | string | yes | Order id. |
| `checkout_session_id` | string | yes | Originating session. |
| `permalink_url` | string (URI) | yes | Buyer-facing order URL. |
| `order_number` | string | no | Human-readable number. |
| `status` | string | no | Order lifecycle (`created`, `confirmed`, `processing`, `shipped`, `completed`, `canceled`, …); accept unknown values. |
| `line_items` | array | no | [`OrderLineItem`](#orderlineitem). |
| `fulfillments` | array | no | Ship / pickup / digital progress (`Fulfillment` in OpenAPI). |
| `adjustments` | array | no | Refunds, credits, disputes (`Adjustment` in OpenAPI). |
| `totals` | array | no | [`Total`](#total) lines (e.g. include `amount_refunded` when relevant). |
| `estimated_delivery` | object | no | [`EstimatedDelivery`](#estimateddelivery). Order-level delivery window for post-purchase UI (“arrives between …”). |
| `confirmation` | object | no | [`OrderConfirmation`](#orderconfirmation). Confirmation identifiers, receipt link, and whether confirmation outreach was sent. |
| `support` | object | no | [`SupportInfo`](#supportinfo). Merchant support contacts and help URL for the order / account. |

##### `EstimatedDelivery`

Present when the seller can project an order-wide delivery range (often derived from fulfillments or the dominant shipping method).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `earliest` | string (date-time) | yes | RFC 3339 start of the expected delivery window. |
| `latest` | string (date-time) | yes | RFC 3339 end of the expected delivery window. |

Use **`earliest`** / **`latest`** in thank-you and order-status screens; pair with per-fulfillment data in **`fulfillments`** when line-level dates differ.

##### `OrderConfirmation`

Carrier for “what to show the buyer immediately after payment”: proof of purchase, receipt access, and optional invoice reference.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `confirmation_number` | string | no | Merchant confirmation reference (may match or complement **`order_number`**). |
| `confirmation_email_sent` | boolean | no | Whether a confirmation email was dispatched; useful for UI copy (“Check your inbox …”). |
| `receipt_url` | string (URI) | no | Link to downloadable or web receipt. |
| `invoice_number` | string | no | Invoice id when B2B or tax invoicing produces one. |

All fields are optional in the payload; FE should render only what is present.

##### `SupportInfo`

Post-purchase support surface: contact channels and self-serve help for order issues, returns, or tracking questions.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `email` | string (email) | no | Support email (e.g. `mailto` target). |
| `phone` | string | no | Support phone (display and click-to-call where supported). |
| `hours` | string | no | Human-readable hours of operation. |
| `help_center_url` | string (URI) | no | Merchant help / FAQ / order-help landing page. |

---

#### `OrderLineItem`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Stable line id (referenced by fulfillments / adjustments). |
| `title` | string | yes | Product title. |
| `quantity` | object | yes | [`OrderLineItemQuantity`](#orderlineitemquantity). |
| `product_id`, `description`, `image_url`, `url` | various | no | Catalog / display. |
| `unit_price`, `subtotal` | integer | no | Minor units. |
| `totals` | array | no | Optional richer per-line breakdown. |
| `status` | string | no | Derived fulfillment progress (`processing`, `partial`, `fulfilled`, `removed`, …). |

---

#### `OrderLineItemQuantity`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `ordered` | integer | yes | Original quantity (≥ 1). |
| `current` | integer | yes | Active quantity after cancellations / returns (≥ 0). |
| `fulfilled` | integer | no | Fulfilled count (defaults per OpenAPI). |

---

#### `VariantOption`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name` | string | yes | Axis (e.g. `Size`). |
| `value` | string | yes | Selected value. |

---

#### `DiscountDetail`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | `percentage`, `fixed`, `bogo`, `volume`. |
| `amount` | integer | yes | Minor units. |
| `code`, `description`, `source` | string | no | `source`: `coupon`, `automatic`, `loyalty`. |

---

#### `Buyer`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `email` | string (email) | yes | Primary contact. |
| `first_name`, `last_name`, `full_name`, `phone_number` | string | no | Identity / reachability. |
| `customer_id` | string | no | Merchant customer id. |
| `account_type` | string | no | `guest`, `registered`, `business`. |
| `authentication_status` | string | no | `authenticated`, `guest`, `requires_signin`. |
| `company`, `loyalty`, `tax_exemption` | object | no | B2B / loyalty / tax (OpenAPI nested schemas). |

---

#### `FulfillmentDetails`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name`, `phone_number`, `email` | string | no | Contact; E.164 recommended for phone. |
| `address` | object | no | [`Address`](#address). |

---

#### `Address`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `name`, `line_one`, `city`, `state`, `country`, `postal_code` | string | yes | `country` ISO 3166-1 alpha-2. |
| `line_two`, `company` | string | no | Optional lines. |

---

#### `FulfillmentGroup`

Groups line items that share the same fulfillment method and destination (used in **`fulfillment_groups`** after the buyer selects options).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `id` | string | yes | Unique group id (client- or server-defined per integration contract). |
| `item_ids` | array of string | yes | Line item ids in this group; must match [`Item.id`](#item) / [`LineItem.id`](#lineitem) for the cart. |
| `destination_type` | string | yes | `shipping`, `pickup`, `local_delivery`, or `digital` — must align with the `type` of the fulfillment option this group corresponds to. |
| `fulfillment_details` | object | no | Override or supplement contact/address for this group ([`FulfillmentDetails`](#fulfillmentdetails)). |
| `location_id` | string | no | Pickup or local-delivery location identifier when the seller models discrete locations. |
| `instructions` | string | no | Special fulfillment instructions for this group. |

---

#### `Total`

Cart or order breakdown row.

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | e.g. `subtotal`, `tax`, `fulfillment`, `total`, `amount_refunded`, … |
| `display_text` | string | yes | Localized label. |
| `amount` | integer | yes | Minor units. |
| `presentment_amount`, `description`, `breakdown` | various | no | Presentation / tax breakdown (`TaxBreakdownItem`). |

---

#### `Link`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Link role; OpenAPI enum: `terms_of_use`, `privacy_policy`, `return_policy`, `shipping_policy`, `contact_us`, `about_us`, `faq`, `support`. |
| `url` | string (URI) | yes | Destination URL. |
| `title` | string | no | Display text. |

---

#### Fulfillment options

Each element of **`CheckoutSession`.`fulfillment_options`** is one of four objects; discriminate on **`type`**:

| `type` constant | Schema | OpenAPI |
|-----------------|--------|---------|
| `shipping` | [`FulfillmentOptionShipping`](#fulfillmentoptionshipping) | `FulfillmentOptionShipping` |
| `digital` | [`FulfillmentOptionDigital`](#fulfillmentoptiondigital) | `FulfillmentOptionDigital` |
| `pickup` | [`FulfillmentOptionPickup`](#fulfillmentoptionpickup) | `FulfillmentOptionPickup` |
| `local_delivery` | [`FulfillmentOptionLocalDelivery`](#fulfillmentoptionlocaldelivery) | `FulfillmentOptionLocalDelivery` |

##### `FulfillmentOptionShipping`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Constant `shipping`. |
| `id` | string | yes | Option id. |
| `title` | string | yes | e.g. “Standard Shipping”. |
| `description` | string | no | Extra detail. |
| `carrier` | string | no | e.g. `USPS`, `FedEx`. |
| `earliest_delivery_time`, `latest_delivery_time` | string (date-time) | no | RFC 3339 delivery bounds. |
| `totals` | array | yes | [`Total`](#total) lines for this option. |

##### `FulfillmentOptionDigital`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Constant `digital`. |
| `id` | string | yes | Option id. |
| `title` | string | yes | Display title. |
| `description` | string | no | Digital delivery notes. |
| `totals` | array | yes | [`Total`](#total) lines for this option. |

##### `FulfillmentOptionPickup`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Constant `pickup`. |
| `id` | string | yes | Option id. |
| `title` | string | yes | Display title. |
| `description` | string | no | Extra detail. |
| `location` | object | yes | `name` (string), `address` ([`Address`](#address)), optional `phone`, `instructions`. |
| `pickup_type` | string | no | `in_store`, `curbside`, `locker`. |
| `ready_by`, `pickup_by` | string (date-time) | no | RFC 3339. |
| `totals` | array | yes | [`Total`](#total) lines for this option. |

##### `FulfillmentOptionLocalDelivery`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Constant `local_delivery`. |
| `id` | string | yes | Option id. |
| `title` | string | yes | Display title. |
| `description` | string | no | Extra detail. |
| `delivery_window` | object | no | `start`, `end` (date-time), both required if present. |
| `service_area` | object | no | Optional `radius_miles`, `center_postal_code`. |
| `totals` | array | yes | [`Total`](#total) lines for this option. |

---

#### Session messages

Each element of **`CheckoutSession`.`messages`** is exactly one of the following. Discriminate on **`type`**: `info`, `warning`, or `error`.

##### `MessageInfo`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Constant `info`. |
| `content_type` | string | yes | `plain` or `markdown` (CommonMark; no raw HTML). |
| `content` | string | yes | Message body. |
| `severity` | string | no | `info`, `low`, `medium`, `high`, `critical`. |
| `resolution` | string | no | `recoverable`, `requires_buyer_input`, `requires_buyer_review`. |
| `param` | string | no | RFC 9535 JSONPath. |

##### `MessageWarning`

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Constant `warning`. |
| `code` | string | yes | OpenAPI enum: `low_stock`, `high_demand`, `shipping_delay`, `price_change`, `expiring_promotion`, `limited_availability`, `discount_code_expired`, `discount_code_invalid`, `discount_code_already_applied`, `discount_code_combination_disallowed`, `discount_code_minimum_not_met`, `discount_code_user_not_logged_in`, `discount_code_user_ineligible`, `discount_code_usage_limit_reached`. |
| `content_type` | string | yes | `plain` or `markdown`. |
| `content` | string | yes | Warning body. |
| `severity` | string | no | `info`, `low`, `medium`, `high`, `critical`. |
| `resolution` | string | no | `recoverable`, `requires_buyer_input`, `requires_buyer_review`. |
| `param` | string | no | RFC 9535 JSONPath. |

##### `MessageError`

In-session business errors when the session is still valid (contrast with HTTP-level [`Error`](#error)).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | Constant `error`. |
| `code` | string | yes | OpenAPI enum: `missing`, `invalid`, `out_of_stock`, `payment_declined`, `requires_sign_in`, `requires_3ds`, `low_stock`, `quantity_exceeded`, `coupon_invalid`, `coupon_expired`, `minimum_not_met`, `maximum_exceeded`, `region_restricted`, `age_verification_required`, `approval_required`, `unsupported`, `not_found`, `conflict`, `rate_limited`, `expired`, `intervention_required`. |
| `content_type` | string | yes | `plain` or `markdown`. |
| `content` | string | yes | Error body. |
| `severity` | string | no | `info`, `low`, `medium`, `high`, `critical`. |
| `resolution` | string | no | `recoverable`, `requires_buyer_input`, `requires_buyer_review`. |
| `param` | string \| null | no | RFC 9535 JSONPath (nullable in schema). |

---

#### `Error`

Protocol-level error when the server cannot return valid session state (HTTP error responses, not `messages[]`).

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `type` | string | yes | `invalid_request`, `processing_error`, `service_unavailable`. |
| `code` | string | yes | Machine-readable (includes idempotency codes per OpenAPI). |
| `message` | string | yes | Human-readable. |
| `param` | string | no | JSONPath to offending field when applicable. |

Use **`MessageError`** inside `CheckoutSession.messages` for buyer-visible cart issues when a valid session still exists.

---

*Further sections (MCP wire format; payment payload wiring for completion if required by implementation) will be added in later revisions.*
