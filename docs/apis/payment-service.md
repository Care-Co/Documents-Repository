# payment-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/payment-service`
Updated: 2026-05-15

Payment integration service backed by **Paddle Billing API v2** (sandbox). Hookdeck relays webhooks for local development. Persists customers, subscriptions, transactions, refunds, and webhook events to PostgreSQL. Exposes a gRPC read-only license query for `b2b-service`.

**Servers**
- `https://api.example.com`

**Versioning** &nbsp;none — single deployed version.
**Security** &nbsp;none on REST endpoints (gate at the platform edge); webhook authenticated via `Paddle-Signature` HMAC-SHA256; cross-service calls authenticated via `X-Caller-Service` header (allowlist).
**Money** &nbsp;all monetary fields are integer minor-units (`Long`). `currencyCode` is ISO 4217. Pricing (currency / amount / country override) is owned by Paddle — not stored in our DB.

**Errors** &nbsp;RFC 9457 `ProblemDetail`.

| HTTP | Cause |
|---|---|
| 400 | `MethodArgumentNotValidException` (request validation) |
| 401 | webhook signature failure |
| 403 | unknown `X-Caller-Service` (cross-service endpoints) |
| 404 | `Subscription` / `Transaction` / `Refund` / `Plan` / `Variant` not found |
| 409 | duplicate / state conflict (plan / variant / subscription) |

---

## Plan catalog

### `GET` /api/payment/plans

**Operation ID** &nbsp;`listPlans`  &nbsp;**Tags** &nbsp;`plan`

Lists active plans (`enabled=true`) for the given audience, with their active variants. Prices (currency / amount / country override) are resolved on the frontend via Paddle.js `Paddle.PricePreview()` — this endpoint returns catalog metadata only.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `audience` | enum `B2C` / `B2B` | yes |

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`PlanResponse`](#planresponse)[] |

#### 200 — example

```json
[
  {
    "planCode": "pro",
    "displayName": "Pro",
    "planSeats": 20,
    "enabled": true,
    "variants": [
      { "priceId": "pri_01kpmjn40qe4pqjjn0sehkmk0a", "billingInterval": "MONTHLY", "enabled": true },
      { "priceId": "pri_01kpmjpmzb9ce7w4pg7yw2da8w", "billingInterval": "YEARLY",  "enabled": true }
    ]
  }
]
```

---

## Plan catalog management (Admin)

Internal-only. No auth — trust cluster-internal routing. External exposure must add an auth layer.

### `POST` /api/payment/plans

**Operation ID** &nbsp;`createPlan`  &nbsp;**Tags** &nbsp;`plan-admin`

Create a new plan with optional variants in one request.

#### Request body

`application/json` &nbsp;**Required** — [`CreatePlanRequest`](#createplanrequest)

```json
{
  "planCode": "premium",
  "audience": "B2B",
  "displayName": "Premium",
  "planSeats": 200,
  "enabled": true,
  "variants": [
    { "priceId": "pri_premium_month", "billingInterval": "MONTHLY", "enabled": true },
    { "priceId": "pri_premium_year",  "billingInterval": "YEARLY",  "enabled": true }
  ]
}
```

#### Responses

| Status | Content-Type | Schema | Headers |
|---|---|---|---|
| **201** | `application/json` | [`PlanResponse`](#planresponse) | `Location: /api/payment/plans/{planCode}` |
| **409** | `application/problem+json` | [`ProblemDetail`](#problemdetail) | `PlanAlreadyExists` / `VariantAlreadyExists` / `DuplicateVariantInterval` |

### `PATCH` /api/payment/plans/{planCode}

**Operation ID** &nbsp;`updatePlan`  &nbsp;**Tags** &nbsp;`plan-admin`

Partial update. Fields: `displayName`, `planSeats`, `enabled`. `null` fields are kept.

#### Request body

```json
{ "displayName": "Premium B2B", "planSeats": 250, "enabled": false }
```

#### Responses

| Status | Schema |
|---|---|
| **200** | [`PlanResponse`](#planresponse) |
| **404** | [`ProblemDetail`](#problemdetail) (`PlanNotFound`) |

### `POST` /api/payment/plans/{planCode}/variants

**Operation ID** &nbsp;`createVariant`  &nbsp;**Tags** &nbsp;`plan-admin`

Add a variant to an existing plan. The new variant's `billingInterval` must not collide with an existing **enabled** variant.

#### Request body

```json
{ "priceId": "pri_premium_quarter", "billingInterval": "QUARTERLY", "enabled": true }
```

> `BillingInterval` enum currently allows `MONTHLY`, `YEARLY`. New values require enum + `plan_variants` CHECK constraint migration.

#### Responses

| Status | Schema |
|---|---|
| **201** | [`PlanResponse`](#planresponse) |
| **404** | [`ProblemDetail`](#problemdetail) (`PlanNotFound`) |
| **409** | [`ProblemDetail`](#problemdetail) (`VariantAlreadyExists` / `DuplicateVariantInterval`) |

### `PATCH` /api/payment/plans/{planCode}/variants/{priceId}

**Operation ID** &nbsp;`updateVariant`  &nbsp;**Tags** &nbsp;`plan-admin`

Toggle variant `enabled`.

#### Request body

```json
{ "enabled": false }
```

#### Responses

| Status | Schema |
|---|---|
| **200** | [`PlanResponse`](#planresponse) |
| **404** | [`ProblemDetail`](#problemdetail) (`VariantNotFound`) |

---

## Checkout

### `POST` /api/payment/checkout-links

**Operation ID** &nbsp;`createCheckoutLink`  &nbsp;**Tags** &nbsp;`checkout`

Builds the config payload that the frontend passes to `Paddle.Checkout.open(...)`. Ensures a `customer` row + Paddle customer exist for `(userPool, userId)` (creates lazily). Stamps `customData.organization_id` server-side so the frontend cannot tamper with it.

**Caller** — `b2b-service` only (`X-Caller-Service: b2b-service` required).

#### Request body

`application/json` &nbsp;**Required** — [`CreateCheckoutLinkRequest`](#createcheckoutlinkrequest)

```json
{
  "userPool": "B2B",
  "userId": "01HXOWNER...",
  "organizationId": "01HXORG...",
  "priceId": "pri_01kpmjn40qe4pqjjn0sehkmk0a",
  "customerName": "홍길동",
  "customerEmail": "owner@example.com"
}
```

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`CheckoutLinkResponse`](#checkoutlinkresponse) |
| **403** | — | unknown caller |
| **404** | `application/problem+json` | [`ProblemDetail`](#problemdetail) (`PlanNotFound`) |
| **409** | `application/problem+json` | [`ProblemDetail`](#problemdetail) (`PlanDisabled`) |

#### 200 — example

```json
{
  "items": [{ "priceId": "pri_01kpmjn40qe4pqjjn0sehkmk0a", "quantity": 1 }],
  "customer": { "email": "owner@example.com", "name": "홍길동" },
  "customData": { "organization_id": "01HXORG..." },
  "plan": {
    "planCode": "pro",
    "planSeats": 20,
    "displayName": "Pro",
    "billingInterval": "MONTHLY"
  },
  "paddleCustomerId": "ctm_01abc..."
}
```

### `POST` /api/payment/subscriptions/{paddleSubscriptionId}/link-organization

**Operation ID** &nbsp;`linkOrganization`  &nbsp;**Tags** &nbsp;`checkout`

Backfill — attaches `organization_id` (and optional plan metadata) to an existing subscription whose webhook arrived without `custom_data.organization_id`. Cross-MSA call (`b2b → payment`); `X-Caller-Service: b2b-service` required.

#### Request body

`application/json` &nbsp;**Required** — [`LinkOrganizationRequest`](#linkorganizationrequest)

```json
{ "organizationId": "01HXORG...", "planSeats": 20, "planCode": "pro" }
```

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Subscription`](#subscription) |
| **404** | [`ProblemDetail`](#problemdetail) (`SubscriptionNotFound`) |
| **409** | [`ProblemDetail`](#problemdetail) (`OrganizationConflict` — already mapped to a different organization) |

---

## Plan change

### `POST` /api/payment/plan-change

**Operation ID** &nbsp;`changePlan`  &nbsp;**Tags** &nbsp;`subscription`

Replaces the items of an organization's active subscription with a new `priceId`. Tier is compared (`plan_seats` first, then `billing_interval` where YEARLY > MONTHLY) to pick `proration_billing_mode` automatically — `prorated_immediately` for upgrade, `full_next_billing_period` for downgrade.

**Caller** — `b2b-service` only (`X-Caller-Service: b2b-service` required).

#### Request body

`application/json` &nbsp;**Required** — [`PlanChangeRequest`](#planchangerequest)

```json
{ "organizationId": "01HXORG...", "newPriceId": "pri_..." }
```

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`PlanChangeResponse`](#planchangeresponse) |
| **403** | — | unknown caller |
| **404** | `application/problem+json` | [`ProblemDetail`](#problemdetail) (`SubscriptionNotFound` / `NewPlanNotFound`) |
| **409** | `application/problem+json` | [`ProblemDetail`](#problemdetail) (`NewPlanDisabled` / `SamePriceAsCurrent` / `InvalidSubscriptionState`) |

#### 200 — example

```json
{
  "paddleSubscriptionId": "sub_xxx",
  "newPriceId": "pri_01kpmjqrasy0xae9hhfpj6717g",
  "newPlanCode": "bis",
  "newPlanSeats": 50,
  "newBillingInterval": "MONTHLY",
  "prorationMode": "prorated_immediately",
  "status": "ACTIVE"
}
```

DB state (`subscriptions.priceId / plan_code / plan_seats / next_billed_at`) and proration transaction rows are populated asynchronously by Paddle webhooks (`subscription.updated`, `transaction.created`, `transaction.completed`).

---

## Customers

No external REST API. Customer rows are created/updated by two internal flows only.

- **Pre-payment** — `POST /api/payment/checkout-links` invokes `CheckoutLinkService`, which calls `CustomerService.createCustomer` to ensure a Paddle + local customer exists for `(userPool, userId)`.
- **Post-payment** — `POST /webhooks/paddle` `customer.created` / `customer.updated` events are handled by `CustomerWebhookHandler` to keep metadata fresh across all rows that share the same `paddleCustomerId`.

`customers.paddle_customer_id` is **not unique** — a single Paddle customer (one email) can be referenced by multiple rows when the same person has both a `B2B` (b2b-service) and a `CARENCO` (user-service) identity. The `Customer` schema is still referenced inside `CheckoutLink` / `Subscription` / `Transaction` payloads (see schemas section).

---

## Subscriptions

### `GET` /api/subscriptions/{id}

**Operation ID** &nbsp;`getSubscription`  &nbsp;**Tags** &nbsp;`subscription`

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Subscription`](#subscription) |
| **404** | [`ProblemDetail`](#problemdetail) |

#### 200 — example

```json
{
  "id": "...", "paddleSubscriptionId": "sub_...",
  "customerId": "...", "priceId": "pri_...",
  "quantity": 1,
  "organizationId": "01HXORG...",
  "planSeats": 20, "planCode": "pro",
  "status": "ACTIVE",
  "nextBilledAt": "...", "canceledAt": null,
  "createdAt": "..."
}
```

### `GET` /api/subscriptions

**Operation ID** &nbsp;`listSubscriptions`  &nbsp;**Tags** &nbsp;`subscription`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `customerId` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Subscription`](#subscription)[] |

### `POST` /api/subscriptions/{id}/cancel

**Operation ID** &nbsp;`cancelSubscription`  &nbsp;**Tags** &nbsp;`subscription`

Cancel (effective at next billing period).

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Subscription`](#subscription) |

### `POST` /api/subscriptions/{id}/pause

**Operation ID** &nbsp;`pauseSubscription`  &nbsp;**Tags** &nbsp;`subscription`

Status → `PAUSED`.

### `POST` /api/subscriptions/{id}/resume

**Operation ID** &nbsp;`resumeSubscription`  &nbsp;**Tags** &nbsp;`subscription`

Status → `ACTIVE`.

### `POST` /api/subscriptions/{paddleSubscriptionId}/sync

**Operation ID** &nbsp;`syncSubscription`  &nbsp;**Tags** &nbsp;`subscription`

Force-sync from Paddle (ops use). Webhook is the normal path.

---

## Transactions

### `GET` /api/transactions/{id}

**Operation ID** &nbsp;`getTransaction`  &nbsp;**Tags** &nbsp;`transaction`

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Transaction`](#transaction) |
| **404** | [`ProblemDetail`](#problemdetail) |

#### 200 — example

```json
{
  "id": "...", "paddleTransactionId": "txn_...",
  "customerId": "...", "subscriptionId": "...",
  "status": "COMPLETED", "currencyCode": "KRW",
  "subtotal": 1000, "discountTotal": 0,
  "taxTotal": 100, "totalAmount": 1100,
  "billingPeriodStartsAt": "...", "billingPeriodEndsAt": "...",
  "billingCountryCode": "KR",
  "billingPostalCode": "...", "billingRegion": "...",
  "billingCity": "...", "billingAddressLine": "...",
  "items": [
    {
      "id": "...", "paddleItemId": "txnitm_...",
      "priceId": "pri_...", "productId": "pro_...",
      "productName": "...", "productDescription": "...",
      "quantity": 1, "unitPrice": 1000,
      "subtotal": 1000, "discountAmount": 0,
      "taxAmount": 100, "total": 1100, "taxRate": "10.0%"
    }
  ],
  "payments": [
    {
      "id": "...", "amount": 1100, "status": "captured",
      "paymentType": "card",
      "cardType": "visa", "cardLast4": "1111",
      "cardExpiryMonth": 12, "cardExpiryYear": 2030,
      "cardholderName": "...", "capturedAt": "..."
    }
  ],
  "createdAt": "..."
}
```

### `GET` /api/transactions

**Operation ID** &nbsp;`listTransactions`  &nbsp;**Tags** &nbsp;`transaction`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `customerId` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Transaction`](#transaction)[] |

### `GET` /api/transactions/subscription/{subscriptionId}

**Operation ID** &nbsp;`listTransactionsBySubscription`  &nbsp;**Tags** &nbsp;`transaction`

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Transaction`](#transaction)[] |

### `POST` /api/transactions/{paddleTransactionId}/sync

**Operation ID** &nbsp;`syncTransaction`  &nbsp;**Tags** &nbsp;`transaction`

Force-sync from Paddle.

---

## Refunds

### `POST` /api/refunds

**Operation ID** &nbsp;`createRefund`  &nbsp;**Tags** &nbsp;`refund`

Create refund (full or partial).

#### Request body

`application/json` &nbsp;**Required** — [`CreateRefundRequest`](#createrefundrequest)

```json
{
  "transactionId": "...",
  "reason": "Customer request",
  "items": [{ "paddleItemId": "txnitm_...", "amount": "500" }]
}
```

#### Responses

| Status | Content-Type | Schema | Headers |
|---|---|---|---|
| **201** | `application/json` | [`Refund`](#refund) | `Location: /api/refunds/{id}` |
| **400** | `application/problem+json` | [`ProblemDetail`](#problemdetail) | — |

### `GET` /api/refunds/{id}

**Operation ID** &nbsp;`getRefund`  &nbsp;**Tags** &nbsp;`refund`

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Refund`](#refund) |
| **404** | [`ProblemDetail`](#problemdetail) |

### `GET` /api/refunds

**Operation ID** &nbsp;`listRefunds`  &nbsp;**Tags** &nbsp;`refund`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `transactionId` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Refund`](#refund)[] |

---

## Webhook

### `POST` /webhooks/paddle

**Operation ID** &nbsp;`receivePaddleWebhook`  &nbsp;**Tags** &nbsp;`webhook`
**Security** &nbsp;`Paddle-Signature` HMAC-SHA256 (`ts=<unix>;h1=<hash>`)

Receive a Paddle webhook event. Idempotent on `event_id` (Paddle retries are deduplicated). Per-handler errors are logged but the request still returns `200`.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| header | `Paddle-Signature` | string | yes |

#### Request body

raw bytes (preserved for signature verification). Parsed as [`PaddleWebhookEvent`](#paddlewebhookevent).

```json
{
  "event_type": "subscription.created",
  "event_id": "evt_...",
  "occurred_at": "...",
  "notification_id": "ntf_...",
  "data": { }
}
```

**Supported `event_type`**

- Customer: `customer.created` `customer.updated`
- Subscription: `subscription.created` `subscription.activated` `subscription.updated` `subscription.canceled` `subscription.paused` `subscription.resumed` `subscription.past_due`
- Transaction: `transaction.completed` `transaction.payment_failed` `transaction.canceled` `transaction.billed`
- Adjustment / Refund: `adjustment.created` `adjustment.updated`

#### Responses

| Status | Body |
|---|---|
| **200** | (empty) |
| **401** | invalid signature |

---

## gRPC — `SubscriptionQuery` (internal)

Read-only license query exposed for `b2b-service`. Proto: `com.carenco.grpc.payment.v1.SubscriptionQuery`. Server port `9090` inside the cluster — not exposed via the edge gateway.

### `getLicenseStatus(GetLicenseStatusRequest) → LicenseStatus`

Looks up the latest `subscription` for the given `organization_id` and maps its status to a `LicenseState`.

**Mapping** — Paddle subscription status → `LicenseState`

| Paddle status | `LicenseState` |
|---|---|
| `ACTIVE`, `TRIALING` | `LICENSE_STATE_ACTIVE` |
| `PAST_DUE` | `LICENSE_STATE_GRACE` (grace period = `nextBilledAt + 7d`) |
| `PAUSED` | `LICENSE_STATE_SUSPENDED` |
| `CANCELED`, `EXPIRED` | `LICENSE_STATE_EXPIRED` |
| (no row) | `LICENSE_STATE_NONE` |

**Errors** — `INVALID_ARGUMENT` (missing `organization_id`), `INTERNAL` (unexpected).

### `batchGetLicenseStatus(BatchGetLicenseStatusRequest) → BatchGetLicenseStatusResponse`

Same lookup for a list of `organization_ids`. Returns one `LicenseStatus` per requested id (no row → `LICENSE_STATE_NONE`).

---

## Schemas

### `CreatePlanRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `planCode` | string | yes | `@NotBlank` |
| `audience` | enum (`B2C` / `B2B`) | yes | `@NotNull` |
| `displayName` | string | no | — |
| `planSeats` | integer | yes | `@NotNull @PositiveOrZero` |
| `enabled` | boolean | no | default `true` |
| `variants` | array<[`VariantInput`](#variantinput)> | no | — |

### `VariantInput`

| Field | Type | Required | Validation |
|---|---|---|---|
| `priceId` | string | yes | `@NotBlank` |
| `billingInterval` | enum (`MONTHLY` / `YEARLY`) | yes | `@NotNull` |
| `enabled` | boolean | no | default `true` |

### `UpdatePlanRequest`

Partial update — `null` fields keep current value.

| Field | Type | Validation |
|---|---|---|
| `displayName` | string | — |
| `planSeats` | integer | `@PositiveOrZero` |
| `enabled` | boolean | — |

### `CreateVariantRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `priceId` | string | yes | `@NotBlank` |
| `billingInterval` | enum (`MONTHLY` / `YEARLY`) | yes | `@NotNull` |
| `enabled` | boolean | no | default `true` |

### `UpdateVariantRequest`

| Field | Type |
|---|---|
| `enabled` | boolean |

### `PlanResponse`

| Field | Type | Description |
|---|---|---|
| `planCode` | string | plan PK |
| `displayName` | string | label |
| `planSeats` | integer | seats |
| `enabled` | boolean | catalog visibility |
| `variants` | array<[`PlanVariantSummary`](#planvariantsummary)> | one per `(plan_code, billing_interval)` |

### `PlanVariantSummary`

| Field | Type | Description |
|---|---|---|
| `priceId` | string | Paddle `pri_...` |
| `billingInterval` | enum (`MONTHLY` / `YEARLY`) | — |
| `enabled` | boolean | catalog visibility (admin response includes disabled rows; public catalog response does not) |

### `CreateCheckoutLinkRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `userPool` | enum (`CARENCO` / `B2B`) | yes | `@NotNull` |
| `userId` | string | yes | `@NotBlank` |
| `organizationId` | string | yes | `@NotBlank` (B2B subscription target) |
| `priceId` | string | yes | `@NotBlank` (must exist in `plan_variants`) |
| `customerName` | string | yes | `@NotBlank` |
| `customerEmail` | string | yes | `@NotBlank @Email` |

### `CheckoutLinkResponse`

| Field | Type | Description |
|---|---|---|
| `items` | array<[`CheckoutItem`](#checkoutitem)> | passed verbatim to `Paddle.Checkout.open({ items })` |
| `customer` | [`CheckoutCustomer`](#checkoutcustomer) | for `Paddle.Checkout.open({ customer })` when no `paddleCustomerId` |
| `customData` | object | server-stamped — always `{ organization_id }` |
| `plan` | [`PlanInfo`](#planinfo) | display-only metadata |
| `paddleCustomerId` | string \| null | when set, frontend uses `customer: { id }` instead — reuses card vault |

### `CheckoutItem`

| Field | Type |
|---|---|
| `priceId` | string |
| `quantity` | integer |

### `CheckoutCustomer`

| Field | Type |
|---|---|
| `email` | string |
| `name` | string |

### `PlanInfo`

| Field | Type | Description |
|---|---|---|
| `planCode` | string | — |
| `planSeats` | integer | — |
| `displayName` | string | — |
| `billingInterval` | enum (`MONTHLY` / `YEARLY`) | — |

> Pricing intentionally omitted — frontend resolves it via `Paddle.PricePreview()`.

### `LinkOrganizationRequest`

| Field | Type | Validation |
|---|---|---|
| `organizationId` | string | `@NotBlank` |
| `planSeats` | integer | `@Min(0)` (null = keep existing or fall back to `plan_variants` lookup) |
| `planCode` | string | nullable |

### `PlanChangeRequest`

| Field | Type | Validation |
|---|---|---|
| `organizationId` | string | `@NotBlank` |
| `newPriceId` | string | `@NotBlank` |

### `PlanChangeResponse`

| Field | Type | Description |
|---|---|---|
| `paddleSubscriptionId` | string | the subscription that was updated |
| `newPriceId` | string | applied price |
| `newPlanCode` | string | resolved from `plan_variants` join |
| `newPlanSeats` | integer | resolved from `subscription_plans` |
| `newBillingInterval` | enum (`MONTHLY` / `YEARLY`) | — |
| `prorationMode` | enum (`prorated_immediately` / `full_next_billing_period`) | server-decided from tier comparison |
| `status` | enum (`ACTIVE` `CANCELED` `PAST_DUE` `PAUSED` `TRIALING` `EXPIRED`) | subscription status (pre-webhook snapshot) |

### `Customer`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk (UUID) |
| `userPool` | enum (`CARENCO` / `B2B`) | nullable when row created by webhook before `createCustomer` |
| `userId` | string | the user id within that pool; nullable along with `userPool` |
| `paddleCustomerId` | string | `ctm_...` — not unique (one Paddle customer can map to multiple rows across pools) |
| `name` | string | — |
| `email` | string | — |
| `status` | string (enum: `ACTIVE` `ARCHIVED`) | — |
| `createdAt` | string (date-time, UTC) | — |

### `Subscription`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk |
| `paddleSubscriptionId` | string | `sub_...` |
| `customerId` | string | FK to `customers.id` |
| `priceId` | string | `pri_...` |
| `quantity` | integer | — |
| `organizationId` | string \| null | B2B target organization (null for non-B2B) |
| `planSeats` | integer | snapshot from `subscription_plans` (denormalized for license queries) |
| `planCode` | string | snapshot from `subscription_plans` |
| `status` | enum (`ACTIVE` `CANCELED` `PAST_DUE` `PAUSED` `TRIALING` `EXPIRED`) | — |
| `nextBilledAt` | string (date-time, UTC) | — |
| `canceledAt` | string (date-time, UTC) | nullable |
| `createdAt` | string (date-time, UTC) | — |

### `Transaction`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk |
| `paddleTransactionId` | string | `txn_...` |
| `customerId` | string | — |
| `subscriptionId` | string | nullable |
| `status` | string (enum: `DRAFT` `READY` `BILLED` `PAID` `COMPLETED` `CANCELED` `PAST_DUE`) | — |
| `currencyCode` | string (ISO 4217) | — |
| `subtotal` `discountTotal` `taxTotal` `totalAmount` | integer (`Long`) | minor units |
| `billingPeriodStartsAt` `billingPeriodEndsAt` | string (date-time, UTC) | — |
| `billingCountryCode` | string (ISO 3166-1 α-2) | — |
| `billingPostalCode` `billingRegion` `billingCity` `billingAddressLine` | string | — |
| `items` | [`TransactionItem`](#transactionitem)[] | — |
| `payments` | [`TransactionPayment`](#transactionpayment)[] | — |
| `createdAt` | string (date-time, UTC) | — |

### `TransactionItem`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk |
| `paddleItemId` | string | `txnitm_...` |
| `priceId` | string | `pri_...` |
| `productId` | string | `pro_...` |
| `productName` | string | — |
| `productDescription` | string | — |
| `quantity` | integer | — |
| `unitPrice` `subtotal` `discountAmount` `taxAmount` `total` | integer (`Long`) | minor units |
| `taxRate` | string (e.g. `"10.0%"`) | — |

### `TransactionPayment`

| Field | Type | Description |
|---|---|---|
| `id` | string | — |
| `amount` | integer (`Long`) | minor units |
| `status` | string | `captured` `pending` `failed` etc. |
| `paymentType` | string | `card` `apple_pay` `google_pay` `paypal` `alipay` etc. |
| `cardType` | string | `visa` `mastercard` `amex` … (when card) |
| `cardLast4` | string | — |
| `cardExpiryMonth` | integer (1-12) | — |
| `cardExpiryYear` | integer (4-digit) | — |
| `cardholderName` | string | — |
| `capturedAt` | string (date-time, UTC) | — |

### `CreateRefundRequest`

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `transactionId` | string | yes | `@NotBlank` | local pk |
| `reason` | string | yes | `@NotBlank` | — |
| `items` | array<[`RefundItemRequest`](#refunditemrequest)> | no | — | full refund when omitted/empty |

### `RefundItemRequest`

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `paddleItemId` | string | yes | `@NotBlank` | `txnitm_...` |
| `amount` | string (minor units) | no | — | item full refund when null |

### `Refund`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk |
| `paddleAdjustmentId` | string | `adj_...` |
| `transactionId` | string | — |
| `reason` | string | — |
| `status` | string (enum: `PENDING_APPROVAL` `APPROVED` `REJECTED`) | — |
| `currencyCode` | string (ISO 4217) | — |
| `amount` | integer (`Long`) | minor units |
| `createdAt` | string (date-time, UTC) | — |

### `PaddleWebhookEvent`

| Field | Type | Description |
|---|---|---|
| `event_type` | string | — |
| `event_id` | string | dedup key |
| `occurred_at` | string (date-time, UTC) | — |
| `notification_id` | string | — |
| `data` | object | event-specific payload |

### `ProblemDetail`

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "Validation failure",
  "errors": [{ "field": "email", "message": "must be a well-formed email address" }]
}
```

---

## Notes

- Local IDs and Paddle IDs are stored side-by-side; sync endpoints reconcile drift.
- Webhook is the source of truth for subscription / transaction state — REST sync endpoints exist mainly for ops.
- Plan / variant catalog is the source of truth for "which plan is sellable"; pricing itself lives in Paddle (country-specific overrides included).
- `customers.paddle_customer_id` is **non-unique** by design (multi-pool customers share Paddle's email-keyed customer).
- `customData.organization_id` is server-stamped in `CheckoutLinkService` and never trusted from the frontend.
- CORS: `*` origin / `GET POST PUT DELETE PATCH OPTIONS` / max age 3600 s.
