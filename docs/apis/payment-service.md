# payment-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/payment-service`
Updated: 2026-04-27

Payment integration service backed by **Paddle Billing API v2** (sandbox). Hookdeck relays webhooks for local development. Persists customers, subscriptions, transactions, refunds, and webhook events to PostgreSQL.

**Servers**
- `https://api.example.com`

**Versioning** &nbsp;none — single deployed version. **Security** &nbsp;none on REST endpoints (gate at the platform edge); webhook authenticated via `Paddle-Signature` HMAC-SHA256.

**Money** &nbsp;all monetary fields are integer cents (`Long`). `currencyCode` is ISO 4217.

**Errors** &nbsp;RFC 9457 `ProblemDetail`.

| HTTP | Cause |
|---|---|
| 400 | `MethodArgumentNotValidException` |
| 401 | webhook signature failure |
| 404 | `Customer/Subscription/Transaction/Refund NotFoundException` |

---

## Customers

### `POST` /api/customers

**Operation ID** &nbsp;`createCustomer`  &nbsp;**Tags** &nbsp;`customer`

Create local + Paddle customer.

#### Request body

`application/json` &nbsp;**Required** — [`CreateCustomerRequest`](#createcustomerrequest)

#### Responses

| Status | Content-Type | Schema | Headers |
|---|---|---|---|
| **201** | `application/json` | [`Customer`](#customer) | `Location: /api/customers/{id}` |
| **400** | `application/problem+json` | [`ProblemDetail`](#problemdetail) | — |

#### 201 — example

```json
{
  "id": "...",
  "userId": "...",
  "paddleCustomerId": "ctm_...",
  "name": "Jonghak",
  "email": "u@example.com",
  "status": "ACTIVE",
  "createdAt": "2026-04-27T08:00:00Z"
}
```

### `GET` /api/customers/{id}

**Operation ID** &nbsp;`getCustomer`  &nbsp;**Tags** &nbsp;`customer`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `id` | string | yes |

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`Customer`](#customer) |
| **404** | `application/problem+json` | [`ProblemDetail`](#problemdetail) |

### `GET` /api/customers/user/{userId}

**Operation ID** &nbsp;`getCustomerByUserId`  &nbsp;**Tags** &nbsp;`customer`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `userId` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Customer`](#customer) |
| **404** | [`ProblemDetail`](#problemdetail) |

### `POST` /api/customers/{paddleCustomerId}/sync

**Operation ID** &nbsp;`syncCustomer`  &nbsp;**Tags** &nbsp;`customer`

Force-sync from Paddle.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `paddleCustomerId` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| **200** | [`Customer`](#customer) |

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
  "quantity": 1, "status": "ACTIVE",
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

Cancel (effective at next billing).

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

Force-sync from Paddle.

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
  "status": "COMPLETED", "currencyCode": "USD",
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

## Schemas

### `CreateCustomerRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `userId` | string | yes | `@NotBlank` |
| `name` | string | yes | `@NotBlank` |
| `email` | string | yes | `@NotBlank @Email` |

### `Customer`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk |
| `userId` | string | — |
| `paddleCustomerId` | string | `ctm_...` |
| `name` | string | — |
| `email` | string | — |
| `status` | string (enum: `ACTIVE` `ARCHIVED`) | — |
| `createdAt` | string (date-time, UTC) | — |

### `Subscription`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk |
| `paddleSubscriptionId` | string | `sub_...` |
| `customerId` | string | — |
| `priceId` | string | `pri_...` |
| `quantity` | integer | — |
| `status` | string (enum: `ACTIVE` `CANCELED` `PAST_DUE` `PAUSED` `TRIALING`) | — |
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
| `subtotal` `discountTotal` `taxTotal` `totalAmount` | integer (`Long`) | cents |
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
| `unitPrice` `subtotal` `discountAmount` `taxAmount` `total` | integer (`Long`) | cents |
| `taxRate` | string (e.g. `"10.0%"`) | — |

### `TransactionPayment`

| Field | Type | Description |
|---|---|---|
| `id` | string | — |
| `amount` | integer (`Long`) | cents |
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
| `amount` | string (cents) | no | — | item full refund when null |

### `Refund`

| Field | Type | Description |
|---|---|---|
| `id` | string | local pk |
| `paddleAdjustmentId` | string | `adj_...` |
| `transactionId` | string | — |
| `reason` | string | — |
| `status` | string (enum: `PENDING_APPROVAL` `APPROVED` `REJECTED`) | — |
| `currencyCode` | string (ISO 4217) | — |
| `amount` | integer (`Long`) | cents |
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
- CORS: `*` origin / `GET POST PUT DELETE PATCH OPTIONS` / max age 3600 s.
