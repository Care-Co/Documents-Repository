# payment-service

> 엔드포인트별 **Header · Request · Response** 정의 — 소스는 실제 컨트롤러/DTO. 표기 — **굵은 필드 = 필수**, 규칙은 키워드만.

| 항목 | 값 |
|---|---|
| Source | `/Users/jonghak/GitHub/Care&Co/payment-service` |
| Updated | 2026-07-13 |
| 배포 상태 | **dev 전용 (prod 미배포)** — 아래 스펙은 현행 구현이며 운영 계약으로 확정된 것이 아니다 |
| Server | `https://api.example.com` |
| Base path | 단일 base path 없음 — `/api/payment` · `/api/subscriptions` · `/api/transactions` · `/api/refunds` · `/api/internal` · `/api/customer-portal` · `/webhooks` 혼재 |

Paddle Billing API v2(sandbox) 연동 결제 서비스. customer/subscription/transaction/refund/webhook 이벤트를 PostgreSQL 에 영속하고, `b2b-service` 용 gRPC read-only license 조회를 노출한다. 로컬 개발에서는 Hookdeck 이 webhook 을 릴레이한다.

---

## 공통 규칙

- **버전** — 헤더 기반 API versioning 미사용, 모든 REST 엔드포인트가 unversioned. 경로의 `/api/v1` 등은 path prefix 일 뿐이다.
- **인증** — **기본 internal-only.** 대부분의 엔드포인트는 `InternalCallerInterceptor` 가 `X-Caller-Service` allowlist(`b2b-service` / `admin-service` / `payment-ops`)를 검증하며 gateway 라우팅에서도 제거되어 외부 직접 호출이 불가하다 — 외부는 **b2b-service `/api/v2/b2b/billing/**` 프록시(JWT 인증)** 를 경유한다. 동작 사용자는 `X-Acting-User-Id` 헤더로 감사 로그에 전파된다(권한 판단은 호출 측 책임). 공개(외부 직접 호출 가능)는 plan catalog(`GET /api/payment/plans`)와 webhook(`/webhooks/paddle`, `Paddle-Signature` HMAC-SHA256)뿐이다. 내부 전용 여부는 각 섹션 Security 참조.
- **바디** — 요청 바디가 있으면 `Content-Type: application/json`.
- **Money** — 모든 금액 필드는 정수 minor-units(`Long`). `currencyCode` 는 ISO 4217. 가격(통화 / 금액 / 국가 오버라이드)은 Paddle 이 소유하며 우리 DB 에 저장하지 않는다.
- **응답 틀** — 성공 응답은 envelope 없이 DTO 를 직접 반환. 에러는 RFC 9457 `ProblemDetail` (CncResponse envelope 아님) — 샘플과 상태별 원인은 아래 접기, 코드 전체는 문서 하단 [에러 코드 (ProblemDetail)](#에러-코드-problemdetail) 참조.

<details>
<summary><b>에러 응답</b> — ProblemDetail 샘플 · HTTP 상태별 원인</summary>

```json
{
  "type": "about:blank",
  "title": "Not Found",
  "status": 404,
  "detail": "Refund not found: ref_01h..."
}
```

| HTTP | 원인 |
|---|---|
| 400 | `MethodArgumentNotValidException` (request validation) |
| 401 | webhook signature 실패 |
| 403 | 알 수 없는 `X-Caller-Service` (cross-service 엔드포인트) |
| 404 | `Subscription` / `Transaction` / `Refund` / `Plan` / `Variant` not found |
| 409 | duplicate / state conflict (plan / variant / subscription) |

</details>

---


## plans

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/api/payment/plans`](#1-get-apipaymentplans) | — | 공개 | audience 별 활성 plan 목록과 그 variants(priceId × billingInt… |
| POST | [`/api/payment/plans`](#2-post-apipaymentplans) | — | 없음 (내부망 신뢰) | 신규 plan 을 메타 + variants 와 함께 한 번에 등록하는 운영/내부 도구 endpoi… |
| PATCH | [`/api/payment/plans/{planCode}`](#3-patch-apipaymentplansplancode) | — | 없음 (내부망 신뢰) | plan 메타(displayName / planSeats / enabled) 부분 수정… |
| POST | [`/api/payment/plans/{planCode}/variants`](#4-post-apipaymentplansplancodevariants) | — | 없음 (내부망 신뢰) | 기존 plan 에 variant 추가 — 신규 variant 의 `billingInterval`… |
| PATCH | [`/api/payment/plans/{planCode}/variants/{priceId}`](#5-patch-apipaymentplansplancodevariantspriceid) | — | 없음 (내부망 신뢰) | variant 의 `enabled` 토글 |

---

## 1. `GET` /api/payment/plans

audience 별 활성 plan 목록과 그 variants(priceId × billingInterval) 를 반환하는 공개 카탈로그. 가격은 포함되지 않음 — FE 가 Paddle.js `Paddle.PricePreview()` 로 country override 반영된 가격을 직접 받는다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | (allowlist 없음 — 공개 endpoint) | ❌ |

### Security

공개(외부 직접 호출 가능). caller allowlist 없음. 가격/plan 메타만 노출하며 가격은 Paddle 소유라 DB 미저장.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — query 파라미터만.

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`audience`** | enum | ✅ | `B2C` / `B2B`. 미등록 값이면 400 |

### Response

**200 OK** — 활성 plan 배열 (각 plan 의 활성 variants 포함)

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

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `planCode` | string | — | plan PK |
| `displayName` | string | — | 표시 이름 |
| `planSeats` | integer | — | 좌석 수 |
| `enabled` | boolean | — | 카탈로그 노출 여부 |
| `variants[]` | array | — | `(planCode, billingInterval)` 당 1개 |
| `variants[].priceId` | string | — | Paddle `pri_...` |
| `variants[].billingInterval` | string | — | `MONTHLY` / `YEARLY` |
| `variants[].enabled` | boolean | — | 노출 여부 |

</details>

---

## 2. `POST` /api/payment/plans

신규 plan 을 메타 + variants 와 함께 한 번에 등록하는 운영/내부 도구 endpoint.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `Content-Type` | `application/json` | ✅ |

### Security

권한 검증 없음 — cluster-internal 라우팅 신뢰. `InternalCallerInterceptor` 대상 경로(`/api/subscriptions`·`/api/refunds`·`/api/transactions`·`/api/customer-portal`)에 `/api/payment/**` 는 포함되지 않아 caller 가드가 걸리지 않는다. 외부 노출 시 별도 인증 레이어 필요.

<details>
<summary><b>Request · Response</b></summary>

### Request

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

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`planCode`** | string | ✅ | `@NotBlank` |
| **`audience`** | enum | ✅ | `@NotNull` — `B2C` / `B2B` |
| `displayName` | string | ❌ | — |
| **`planSeats`** | integer | ✅ | `@NotNull @PositiveOrZero` |
| `enabled` | boolean | ❌ | 미지정 시 서비스 기본값 |
| `variants[]` | array | ❌ | `@Valid` — 아래 필드 |
| **`variants[].priceId`** | string | ✅ | `@NotBlank` — Paddle `pri_...` |
| **`variants[].billingInterval`** | enum | ✅ | `@NotNull` — `MONTHLY` / `YEARLY` |
| `variants[].enabled` | boolean | ❌ | — |

### Response

**201 Created** — `Location: /api/payment/plans/{planCode}`. 바디는 `PlanResponse` (위 GET 과 동일 shape).

```json
{
  "planCode": "premium",
  "displayName": "Premium",
  "planSeats": 200,
  "enabled": true,
  "variants": [
    { "priceId": "pri_premium_month", "billingInterval": "MONTHLY", "enabled": true },
    { "priceId": "pri_premium_year",  "billingInterval": "YEARLY",  "enabled": true }
  ]
}
```

<details><summary><b>409 Conflict</b> — plan/variant 중복 또는 interval 충돌</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Plan already exists: premium" }
```

`PlanAlreadyExists` → `Plan already exists: {planCode}`. `VariantAlreadyExists` → `Variant already exists: priceId={priceId}`. `DuplicateVariantInterval` → `Plan {planCode} already has an active {interval} variant`. 응답 Content-Type `application/problem+json`.

</details>

</details>

---

## 3. `PATCH` /api/payment/plans/{planCode}

plan 메타(displayName / planSeats / enabled) 부분 수정 — `null` 필드는 기존 값 유지.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `Content-Type` | `application/json` | ✅ |

### Security

권한 검증 없음 — cluster-internal 라우팅 신뢰 (`/api/payment/**` 는 caller 가드 미적용).

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 + path 의 `planCode`.

```json
{ "displayName": "Premium B2B", "planSeats": 250, "enabled": false }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `displayName` | string | ❌ | null 이면 유지 |
| `planSeats` | integer | ❌ | `@PositiveOrZero`. null 이면 유지 |
| `enabled` | boolean | ❌ | null 이면 유지 |

path 파라미터. **`planCode`** (string) — 수정 대상 plan.

### Response

**200 OK** — 갱신된 `PlanResponse`.

<details><summary><b>404 Not Found</b> — plan 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Plan not found: premium" }
```

`PlanNotFound` → `Plan not found: {planCode}`. 응답 Content-Type `application/problem+json`.

</details>

</details>

---

## 4. `POST` /api/payment/plans/{planCode}/variants

기존 plan 에 variant 추가 — 신규 variant 의 `billingInterval` 은 기존 **활성** variant 와 충돌 불가.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `Content-Type` | `application/json` | ✅ |

### Security

권한 검증 없음 — cluster-internal 라우팅 신뢰.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 + path 의 `planCode`.

```json
{ "priceId": "pri_premium_quarter", "billingInterval": "MONTHLY", "enabled": true }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`priceId`** | string | ✅ | `@NotBlank` — Paddle `pri_...` |
| **`billingInterval`** | enum | ✅ | `@NotNull` — `MONTHLY` / `YEARLY`. 신규 값은 enum + `plan_variants` CHECK 마이그레이션 필요 |
| `enabled` | boolean | ❌ | — |

path 파라미터. **`planCode`** (string).

### Response

**201 Created** — 갱신된 `PlanResponse`.

<details><summary><b>404 Not Found</b> — plan 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Plan not found: premium" }
```

`PlanNotFound` → `Plan not found: {planCode}`.

</details>

<details><summary><b>409 Conflict</b> — variant 중복 또는 interval 충돌</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Variant already exists: priceId=pri_premium_quarter" }
```

`VariantAlreadyExists` → `Variant already exists: priceId={priceId}`. `DuplicateVariantInterval` → `Plan {planCode} already has an active {interval} variant`. 응답 Content-Type `application/problem+json`.

</details>

</details>

---

## 5. `PATCH` /api/payment/plans/{planCode}/variants/{priceId}

variant 의 `enabled` 토글.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `Content-Type` | `application/json` | ✅ |

### Security

권한 검증 없음 — cluster-internal 라우팅 신뢰.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 + path 의 `planCode`, `priceId`.

```json
{ "enabled": false }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `enabled` | boolean | ❌ | 현재는 이 필드만 의미 |

path 파라미터. **`planCode`** (string), **`priceId`** (string).

### Response

**200 OK** — 갱신된 `PlanResponse`.

<details><summary><b>404 Not Found</b> — variant 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Variant not found: priceId=pri_premium_quarter" }
```

`VariantNotFound` → `Variant not found: priceId={priceId}`. 응답 Content-Type `application/problem+json`.

</details>

</details>

---

## checkout · plan-change (b2b 연동)

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/api/payment/checkout-links`](#6-post-apipaymentcheckout-links) | — | internal — `b2b-service` | 결제 시작 — FE 의 `Paddle.Checkout.open()` 이 사용할 transactio… |
| POST | [`/api/payment/subscriptions/{paddleSubscriptionId}/link-organization`](#7-post-apipaymentsubscriptionspaddlesubscriptionidlink-organization) | — | internal — `b2b-service` | 백필 — 이미 존재하는 subscription 에 organization 매핑(및 선택 plan… |
| POST | [`/api/payment/plan-change`](#8-post-apipaymentplan-change) | — | internal — `b2b-service` | organization 의 active subscription 을 새 `priceId` 로 교체 |

---

## 6. `POST` /api/payment/checkout-links

결제 시작 — FE 의 `Paddle.Checkout.open()` 이 사용할 transaction reference 를 server-side 로 미리 생성. `custom_data` (organization_id / user_id / user_pool) 는 서버가 박아 FE 변조 불가.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` | ✅ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. 인라인 가드 — `X-Caller-Service` 화이트리스트가 **`b2b-service` 단독**. `InternalCallerInterceptor` 가 아니라 컨트롤러 내부에서 직접 검사. 불허 caller 는 403 ProblemDetail.

<details>
<summary><b>Request · Response</b></summary>

### Request

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

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`userPool`** | enum | ✅ | `@NotNull` — `CARENCO` / `B2B` |
| **`userId`** | string | ✅ | `@NotBlank` — 해당 풀 내 user id |
| **`organizationId`** | string | ✅ | `@NotBlank` — B2B 결제 대상 organization |
| **`priceId`** | string | ✅ | `@NotBlank` — `plan_variants` 에 등록된 Paddle `pri_...` |
| **`customerName`** | string | ✅ | `@NotBlank` |
| **`customerEmail`** | string | ✅ | `@NotBlank @Email` |

### Response

**200 OK** — Paddle transaction reference.

```json
{
  "transactionId": "txn_01abc...",
  "checkoutUrl": "https://buy.paddle.com/checkout/..."
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `transactionId` | string | — | Paddle `txn_...` (서버 pre-create). FE 가 `Paddle.Checkout.open({ transactionId })` 에 전달 |
| `checkoutUrl` | string | — | Paddle hosted checkout URL — redirect 로도 사용 |

<details><summary><b>403 Forbidden</b> — 허용되지 않은 caller</summary>

```json
{ "type": "about:blank", "title": "Forbidden", "status": 403, "detail": "Internal endpoint — caller unknown-svc not allowed" }
```

</details>

<details><summary><b>404 Not Found</b> — priceId 미등록</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Subscription plan not found: priceId=pri_xxx" }
```

`PlanNotFound` → `Subscription plan not found: priceId={priceId}`.

</details>

<details><summary><b>409 Conflict</b> — plan 비활성</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Subscription plan disabled: priceId=pri_xxx" }
```

`PlanDisabled` → `Subscription plan disabled: priceId={priceId}`.

</details>

</details>

---

## 7. `POST` /api/payment/subscriptions/{paddleSubscriptionId}/link-organization

백필 — 이미 존재하는 subscription 에 organization 매핑(및 선택 plan 메타) 보정 + b2b 캐시 invalidate.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` | ✅ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. 인라인 가드 — `X-Caller-Service` 는 `b2b-service` 단독. 성공 시 `b2bClient.invalidate(organizationId, "subscription.linked")` 호출(b2b 미배포면 skip).

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 + path 의 `paddleSubscriptionId`.

```json
{ "organizationId": "01HXORG...", "planSeats": 20, "planCode": "pro" }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`organizationId`** | string | ✅ | `@NotBlank` |
| `planSeats` | integer | ❌ | `@Min(0)`. null = `subscription_plans` 조회 또는 기존 값 유지 |
| `planCode` | string | ❌ | null = 기존 값 유지 |

path 파라미터. **`paddleSubscriptionId`** (string) — Paddle `sub_...`.

### Response

**200 OK** — 갱신된 `SubscriptionResponse` (아래 GET /api/subscriptions/{id} 와 동일 shape).

<details><summary><b>404 Not Found</b> — subscription 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Subscription not found: paddleId=sub_xxx" }
```

`SubscriptionNotFound` → `Subscription not found: paddleId={paddleSubscriptionId}`.

</details>

<details><summary><b>409 Conflict</b> — 다른 organization 에 이미 매핑됨</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Subscription already linked to organization=01HXORG_A, refused to relink to 01HXORG_B" }
```

`OrganizationConflict` → `Subscription already linked to organization={existing}, refused to relink to {requested}`.

</details>

<details><summary><b>400 Bad Request</b> — 입력 오류</summary>

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "Invalid organizationId: blank" }
```

`InvalidInput` → `Invalid {field}: {reason}`.

</details>

<details><summary><b>403 Forbidden</b> — 허용되지 않은 caller</summary>

```json
{ "type": "about:blank", "title": "Forbidden", "status": 403, "detail": "Internal endpoint — caller unknown-svc not allowed" }
```

</details>

</details>

---

## 8. `POST` /api/payment/plan-change

organization 의 active subscription 을 새 `priceId` 로 교체. 업/다운그레이드 판정으로 `proration_billing_mode` 자동 결정, 차액 정산은 Paddle 이 처리.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` | ✅ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. 인라인 가드 — `X-Caller-Service` 는 `b2b-service` 단독. **불허 caller 는 바디 없는 403** (`ResponseEntity.status(FORBIDDEN).build()` — 다른 endpoint 와 달리 ProblemDetail 본문 없음).

<details>
<summary><b>Request · Response</b></summary>

### Request

```json
{ "organizationId": "01HXORG...", "newPriceId": "pri_..." }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`organizationId`** | string | ✅ | `@NotBlank` — active subscription 자동 lookup |
| **`newPriceId`** | string | ✅ | `@NotBlank` |

### Response

**200 OK** — 변경된 subscription 의 핵심 메타(FE 즉시 표시용, DB/proration 은 webhook 으로 후속 sync).

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

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `paddleSubscriptionId` | string | — | 변경된 subscription |
| `newPriceId` | string | — | 적용된 price |
| `newPlanCode` | string | — | `plan_variants` join 으로 해석 |
| `newPlanSeats` | integer | — | `subscription_plans` 에서 해석 |
| `newBillingInterval` | string | — | `MONTHLY` / `YEARLY` |
| `prorationMode` | string | — | 서버 결정 (예: `prorated_immediately` / `full_next_billing_period`) |
| `status` | string | — | pre-webhook 스냅샷 상태 |

<details><summary><b>403 Forbidden</b> — 허용되지 않은 caller (바디 없음)</summary>

빈 본문. HTTP 403 상태만.

</details>

<details><summary><b>404 Not Found</b> — subscription 또는 plan variant 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "No subscription for organization: 01HXORG..." }
```

`SubscriptionNotFound` → `No subscription for organization: {organizationId}`. `NewPlanNotFound` → `Plan variant not found: {newPriceId}`. 응답 Content-Type `application/problem+json`.

</details>

<details><summary><b>409 Conflict</b> — plan 비활성 / 동일 price / 상태 불가</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Subscription is already on price: pri_..." }
```

`NewPlanDisabled` → `Plan variant disabled: {newPriceId}`. `SamePriceAsCurrent` → `Subscription is already on price: {priceId}`. `InvalidSubscriptionState` → `Subscription not changeable in state: {current}`.

</details>

</details>

---

## subscriptions

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/api/subscriptions/{id}`](#9-get-apisubscriptionsid) | — | internal — allowlist | 로컬 pk 로 subscription 단건 조회 |
| GET | [`/api/subscriptions`](#10-get-apisubscriptions) | — | internal — allowlist | customer 의 subscription 목록 |
| POST | [`/api/subscriptions/{id}/cancel`](#11-post-apisubscriptionsidcancel) | — | internal — allowlist | subscription 취소(다음 청구 주기에 반영) |
| POST | [`/api/subscriptions/{id}/pause`](#12-post-apisubscriptionsidpause) | — | internal — allowlist | subscription 일시 정지(status → `PAUSED`) |
| POST | [`/api/subscriptions/{id}/resume`](#13-post-apisubscriptionsidresume) | — | internal — allowlist | subscription 재개(status → `ACTIVE`) |
| POST | [`/api/subscriptions/{paddleSubscriptionId}/sync`](#14-post-apisubscriptionspaddlesubscriptionidsync) | — | internal — allowlist | Paddle 에서 subscription 강제 재동기화(운영용, 정상 경로는 webhook) |

---

## 9. `GET` /api/subscriptions/{id}

로컬 pk 로 subscription 단건 조회.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 가 `/api/subscriptions/**` 에 대해 `X-Caller-Service` allowlist(`b2b-service` / `admin-service` / `payment-ops`) 검증. `X-Acting-User-Id` 는 MDC 감사 로그로 전파(권한 판단은 호출 측). 불허 caller 는 403 ProblemDetail. 외부는 b2b `/api/v2/b2b/billing/**` 프록시 경유.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `id` (subscription 로컬 pk).

### Response

**200 OK** — `SubscriptionResponse`.

```json
{
  "id": "...",
  "paddleSubscriptionId": "sub_...",
  "customerId": "...",
  "organizationId": "01HXORG...",
  "priceId": "pri_...",
  "quantity": 1,
  "planSeats": 20,
  "planCode": "pro",
  "status": "ACTIVE",
  "nextBilledAt": "2026-08-01T00:00:00Z",
  "currentPeriodStart": "2026-07-01T00:00:00Z",
  "currentPeriodEnd": "2026-08-01T00:00:00Z",
  "canceledAt": null,
  "scheduledChangeAction": null,
  "scheduledChangeEffectiveAt": null,
  "scheduledChangeResumeAt": null,
  "createdAt": "2026-07-01T00:00:00Z"
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `id` | string | — | 로컬 pk |
| `paddleSubscriptionId` | string | — | `sub_...` |
| `customerId` | string | — | `customers.id` FK |
| `organizationId` | string \| null | — | B2B 대상 organization(비 B2B 는 null) |
| `priceId` | string | — | `pri_...` |
| `quantity` | integer | — | — |
| `planSeats` | integer | — | `subscription_plans` 스냅샷(license 조회용 비정규화) |
| `planCode` | string | — | `subscription_plans` 스냅샷 |
| `status` | string | — | `ACTIVE` `CANCELED` `PAST_DUE` `PAUSED` `TRIALING` `EXPIRED` |
| `nextBilledAt` | string(date-time) | — | — |
| `currentPeriodStart` | string(date-time) \| null | — | Paddle `current_billing_period.starts_at` |
| `currentPeriodEnd` | string(date-time) \| null | — | Paddle `current_billing_period.ends_at` |
| `canceledAt` | string(date-time) \| null | — | — |
| `scheduledChangeAction` | string \| null | — | `cancel` / `pause` / `resume`. null=예약 없음 |
| `scheduledChangeEffectiveAt` | string(date-time) \| null | — | scheduled change 적용 시점 |
| `scheduledChangeResumeAt` | string(date-time) \| null | — | pause 후 resume 예약 시점 |
| `createdAt` | string(date-time) | — | — |

<details><summary><b>404 Not Found</b> — subscription 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Subscription not found: {id}" }
```

`NotFound` → `Subscription not found: {id}`.

</details>

</details>

---

## 10. `GET` /api/subscriptions

customer 의 subscription 목록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증(`/api/subscriptions/**`).

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — query 파라미터만.

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`customerId`** | string | ✅ | 로컬 `customers.id` |

### Response

**200 OK** — `SubscriptionResponse` 배열(위 shape). 조회 결과 없으면 빈 배열.

</details>

---

## 11. `POST` /api/subscriptions/{id}/cancel

subscription 취소(다음 청구 주기에 반영).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `id` (subscription 로컬 pk).

### Response

**200 OK** — 갱신된 `SubscriptionResponse`.

<details><summary><b>404 Not Found</b> — subscription 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Subscription not found: {id}" }
```

</details>

<details><summary><b>409 Conflict</b> — 상태 전이 불가</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Cannot cancel subscription {id} in status CANCELED" }
```

`IllegalStateTransition` → `Cannot {requestedAction} subscription {id} in status {currentStatus}`.

</details>

</details>

---

## 12. `POST` /api/subscriptions/{id}/pause

subscription 일시 정지(status → `PAUSED`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `id`.

### Response

**200 OK** — 갱신된 `SubscriptionResponse`.

<details><summary><b>404 Not Found</b> — subscription 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Subscription not found: {id}" }
```

</details>

<details><summary><b>409 Conflict</b> — 상태 전이 불가</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Cannot pause subscription {id} in status CANCELED" }
```

</details>

</details>

---

## 13. `POST` /api/subscriptions/{id}/resume

subscription 재개(status → `ACTIVE`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `id`.

### Response

**200 OK** — 갱신된 `SubscriptionResponse`.

<details><summary><b>404 Not Found</b> — subscription 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Subscription not found: {id}" }
```

</details>

<details><summary><b>409 Conflict</b> — 상태 전이 불가</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Cannot resume subscription {id} in status CANCELED" }
```

</details>

</details>

---

## 14. `POST` /api/subscriptions/{paddleSubscriptionId}/sync

Paddle 에서 subscription 강제 재동기화(운영용, 정상 경로는 webhook).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `paddleSubscriptionId` (Paddle `sub_...`).

### Response

**200 OK** — `SubscriptionResponse`. Result 를 쓰지 않고 직접 반환 — Paddle↔DB 불일치 등 시스템 이상은 throw 되어 500(글로벌 핸들러)으로 노출.

</details>

---

## transactions

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/api/transactions/{id}`](#15-get-apitransactionsid) | — | internal — allowlist | 로컬 pk 로 transaction 단건 조회(항목·결제수단 포함) |
| GET | [`/api/transactions`](#16-get-apitransactions) | — | internal — allowlist | customer 의 transaction 목록 |
| GET | [`/api/transactions/subscription/{subscriptionId}`](#17-get-apitransactionssubscriptionsubscriptionid) | — | internal — allowlist | subscription 의 transaction 목록 |
| POST | [`/api/transactions/{paddleTransactionId}/sync`](#18-post-apitransactionspaddletransactionidsync) | — | internal — allowlist | Paddle 에서 transaction 강제 재동기화 |

---

## 15. `GET` /api/transactions/{id}

로컬 pk 로 transaction 단건 조회(항목·결제수단 포함).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 가 `/api/transactions/**` 검증. 외부는 b2b billing 프록시 경유.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `id` (transaction 로컬 pk).

### Response

**200 OK** — `TransactionResponse`. 금액 필드는 모두 정수 minor-units(`Long`).

```json
{
  "id": "...",
  "paddleTransactionId": "txn_...",
  "customerId": "...",
  "subscriptionId": "...",
  "status": "COMPLETED",
  "currencyCode": "KRW",
  "subtotal": 1000,
  "discountTotal": 0,
  "taxTotal": 100,
  "totalAmount": 1100,
  "billingPeriodStartsAt": "...",
  "billingPeriodEndsAt": "...",
  "billingCountryCode": "KR",
  "billingPostalCode": "...",
  "billingRegion": "...",
  "billingCity": "...",
  "billingAddressLine": "...",
  "items": [
    {
      "id": "...",
      "paddleItemId": "txnitm_...",
      "priceId": "pri_...",
      "productId": "pro_...",
      "productName": "...",
      "productDescription": "...",
      "quantity": 1,
      "unitPrice": 1000,
      "subtotal": 1000,
      "discountAmount": 0,
      "taxAmount": 100,
      "total": 1100,
      "taxRate": "10.0%"
    }
  ],
  "payments": [
    {
      "id": "...",
      "amount": 1100,
      "status": "captured",
      "paymentType": "card",
      "cardType": "visa",
      "cardLast4": "1111",
      "cardExpiryMonth": 12,
      "cardExpiryYear": 2030,
      "cardholderName": "...",
      "capturedAt": "..."
    }
  ],
  "createdAt": "..."
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `id` | string | — | 로컬 pk |
| `paddleTransactionId` | string | — | `txn_...` |
| `customerId` | string | — | — |
| `subscriptionId` | string \| null | — | one-off 면 null |
| `status` | string | — | `DRAFT` `READY` `BILLED` `PAID` `COMPLETED` `CANCELED` `PAST_DUE` |
| `currencyCode` | string | — | ISO 4217 |
| `subtotal` `discountTotal` `taxTotal` `totalAmount` | integer(long) | — | minor-units |
| `billingPeriodStartsAt` `billingPeriodEndsAt` | string(date-time) | — | — |
| `billingCountryCode` | string | — | ISO 3166-1 α-2 |
| `billingPostalCode` `billingRegion` `billingCity` `billingAddressLine` | string | — | — |
| `items[]` | array | — | 아래 항목 |
| `items[].id` | string | — | 로컬 pk |
| `items[].paddleItemId` | string | — | `txnitm_...` |
| `items[].priceId` | string | — | `pri_...` |
| `items[].productId` | string | — | `pro_...` |
| `items[].productName` `productDescription` | string | — | — |
| `items[].quantity` | integer | — | — |
| `items[].unitPrice` `subtotal` `discountAmount` `taxAmount` `total` | integer(long) | — | minor-units |
| `items[].taxRate` | string | — | 예 `"10.0%"` |
| `payments[]` | array | — | 아래 항목 |
| `payments[].id` | string | — | — |
| `payments[].amount` | integer(long) | — | minor-units |
| `payments[].status` | string | — | `captured` `pending` `failed` 등 |
| `payments[].paymentType` | string | — | `card` `apple_pay` 등 |
| `payments[].cardType` | string | — | `visa` `mastercard` 등(카드일 때) |
| `payments[].cardLast4` | string | — | — |
| `payments[].cardExpiryMonth` | integer | — | 1-12 |
| `payments[].cardExpiryYear` | integer | — | 4자리 |
| `payments[].cardholderName` | string | — | — |
| `payments[].capturedAt` | string(date-time) | — | — |
| `createdAt` | string(date-time) | — | — |

<details><summary><b>404 Not Found</b> — transaction 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Transaction not found: id=txn_xxx" }
```

`NotFound` → `Transaction not found: {identifierKind}={identifier}` (identifierKind 는 조회 키 종류).

</details>

</details>

---

## 16. `GET` /api/transactions

customer 의 transaction 목록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — query 파라미터만.

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`customerId`** | string | ✅ | 로컬 `customers.id` |

### Response

**200 OK** — `TransactionResponse` 배열(위 shape). 없으면 빈 배열.

</details>

---

## 17. `GET` /api/transactions/subscription/{subscriptionId}

subscription 의 transaction 목록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `subscriptionId` (로컬 pk).

### Response

**200 OK** — `TransactionResponse` 배열(위 shape). 없으면 빈 배열.

</details>

---

## 18. `POST` /api/transactions/{paddleTransactionId}/sync

Paddle 에서 transaction 강제 재동기화.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `paddleTransactionId` (Paddle `txn_...`).

### Response

**200 OK** — `TransactionResponse`. Result 미사용 — 직접 반환하며 시스템 이상은 throw → 500.

</details>

---

## refunds

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/api/refunds`](#19-post-apirefunds) | — | internal — allowlist | 환불 생성(전체 또는 부분) |
| GET | [`/api/refunds/{id}`](#20-get-apirefundsid) | — | internal — allowlist | 로컬 pk 로 refund 단건 조회 |
| GET | [`/api/refunds`](#21-get-apirefunds) | — | internal — allowlist | transaction 의 refund 목록 |

---

## 19. `POST` /api/refunds

환불 생성(전체 또는 부분). 성공 시 refund 는 `PENDING_APPROVAL` 로 시작.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. `InternalCallerInterceptor` 가 `/api/refunds/**` 검증. 외부는 b2b billing 프록시(거래 소유 검증 포함) 경유.

<details>
<summary><b>Request · Response</b></summary>

### Request

```json
{
  "transactionId": "...",
  "reason": "Customer request",
  "items": [{ "paddleItemId": "txnitm_...", "amount": "500" }]
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`transactionId`** | string | ✅ | `@NotBlank` — 로컬 pk |
| **`reason`** | string | ✅ | `@NotBlank` |
| `items[]` | array | ❌ | null/빈 배열 → 전체 환불. 값 있으면 부분 환불 |
| **`items[].paddleItemId`** | string | ✅ | `@NotBlank` — `TransactionItem.paddleItemId` (`txnitm_...`) |
| `items[].amount` | string | ❌ | null → 해당 항목 전체 환불. 값은 minor-units 문자열(예 `"500"`) |

### Response

**201 Created** — `Location: /api/refunds/{id}`. 바디는 `RefundResponse`. `amount` 는 minor-units(`Long`).

```json
{
  "id": "...",
  "paddleAdjustmentId": "adj_...",
  "transactionId": "...",
  "reason": "Customer request",
  "status": "PENDING_APPROVAL",
  "currencyCode": "KRW",
  "amount": 500,
  "createdAt": "..."
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `id` | string | — | 로컬 pk |
| `paddleAdjustmentId` | string | — | `adj_...` |
| `transactionId` | string | — | — |
| `reason` | string | — | — |
| `status` | string | — | `PENDING_APPROVAL` `APPROVED` `REJECTED` |
| `currencyCode` | string | — | ISO 4217 |
| `amount` | integer(long) | — | minor-units |
| `createdAt` | string(date-time) | — | — |

<details><summary><b>400 Bad Request</b> — 항목/금액 오류</summary>

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "Refund amount 900 exceeds item txnitm_x max 500" }
```

`InvalidItem` → `Invalid refund item {paddleItemId}: {reason}`. `AmountExceedsOriginal` → `Refund amount {requestedAmount} exceeds item {paddleItemId} max {maxAmount}`. `InvalidAmount` → `Invalid refund amount '{requestedAmount}' for item {paddleItemId}`. `DuplicateItem` → `Duplicate refund item: {paddleItemId}`.

</details>

<details><summary><b>404 Not Found</b> — refund/transaction 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Transaction not found: {transactionId}" }
```

`TransactionNotFound` → `Transaction not found: {transactionId}`. (`NotFound` → `Refund not found: {id}` 는 GET 경로에서.)

</details>

<details><summary><b>409 Conflict</b> — 환불 불가 상태</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "Transaction txn_x is not refundable in status DRAFT" }
```

`TransactionNotRefundable` → `Transaction {transactionId} is not refundable in status {currentStatus}`.

</details>

</details>

---

## 20. `GET` /api/refunds/{id}

로컬 pk 로 refund 단건 조회.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `id` (refund 로컬 pk).

### Response

**200 OK** — `RefundResponse`(위 shape).

<details><summary><b>404 Not Found</b> — refund 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Refund not found: {id}" }
```

`NotFound` → `Refund not found: {id}`.

</details>

</details>

---

## 21. `GET` /api/refunds

transaction 의 refund 목록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |

### Security

internal-only. `InternalCallerInterceptor` 검증. 이 목록 조회는 Result 미사용 — 항상 200 배열 반환.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — query 파라미터만.

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`transactionId`** | string | ✅ | 로컬 transaction pk |

### Response

**200 OK** — `RefundResponse` 배열(위 shape). 없으면 빈 배열.

</details>

---

## customer-portal

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/api/customer-portal/sessions`](#22-post-apicustomer-portalsessions) | — | internal — allowlist | Paddle 호스팅 Customer Portal 의 deep link 묶음 생성 — 각 URL 은… |

---

## 22. `POST` /api/customer-portal/sessions

Paddle 호스팅 Customer Portal 의 deep link 묶음 생성 — 각 URL 은 임시 인증 토큰 포함, 캐싱 금지.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `b2b-service` / `admin-service` / `payment-ops` | ✅ |
| `X-Acting-User-Id` | 감사용 acting user id | ❌ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. `InternalCallerInterceptor` 가 `/api/customer-portal/**` 검증. customerId 위조 방지는 호출 측(b2b) 책임 — b2b 가 OWNER 검증 후 organization 소유 구독에서 customerId 도출.

<details>
<summary><b>Request · Response</b></summary>

### Request

```json
{ "customerId": "..." }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`customerId`** | string | ✅ | `@NotBlank` — 본인 customerId |

### Response

**200 OK** — `CustomerPortalResponse`.

```json
{
  "overviewUrl": "https://customer-portal.paddle.com/...",
  "subscriptions": [
    {
      "subscriptionId": "sub_...",
      "cancelUrl": "https://customer-portal.paddle.com/.../cancel",
      "updatePaymentMethodUrl": "https://customer-portal.paddle.com/.../payment-method"
    }
  ]
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `overviewUrl` | string \| null | — | 결제 내역/송장/셀프서비스 메인 |
| `subscriptions[]` | array | — | sub 별 deep link |
| `subscriptions[].subscriptionId` | string | — | Paddle `sub_...` |
| `subscriptions[].cancelUrl` | string | — | 취소 deep link |
| `subscriptions[].updatePaymentMethodUrl` | string | — | 결제수단 변경 deep link |

<details><summary><b>404 Not Found</b> — customer 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Customer not found: ctm_xxx" }
```

`NotFound` → `Customer not found: {customerId}`.

</details>

<details><summary><b>403 Forbidden</b> — archived customer</summary>

```json
{ "type": "about:blank", "title": "Forbidden", "status": 403, "detail": "Customer archived: ctm_xxx" }
```

`Archived` → `Customer archived: {customerId}` (영구 비활성 — 접근 거부 의미의 403, 상태 충돌 409 아님).

</details>

</details>

---

## internal (운영 도구)

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/api/internal/plan-variants/{priceId}/promote`](#23-post-apiinternalplan-variantspriceidpromote) | — | internal — `admin-service` / `payment-ops` | 미매핑 priceId 를 `plan_variants` 로 단순 승격(활성 슬롯이 비어있을 때) |
| POST | [`/api/internal/plan-variants/{newPriceId}/swap`](#24-post-apiinternalplan-variantsnewpriceidswap) | — | internal — `admin-service` / `payment-ops` | 가격 정책 세대 교체 — 옛 priceId 비활성 + 새 priceId 활성 |
| POST | [`/api/internal/subscriptions/migrate-price`](#25-post-apiinternalsubscriptionsmigrate-price) | — | internal — `admin-service` / `payment-ops` | 옛 priceId → 새 priceId 일괄 마이그레이션 |
| GET | [`/api/internal/subscriptions/expiring`](#26-get-apiinternalsubscriptionsexpiring) | — | internal — `admin-service` / `payment-ops` | cancel 예약된 subscription 중 N 일 이내 만료 예정 목록(retention da… |

---

## 23. `POST` /api/internal/plan-variants/{priceId}/promote

미매핑 priceId 를 `plan_variants` 로 단순 승격(활성 슬롯이 비어있을 때).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `admin-service` / `payment-ops` | ✅ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. 인라인 가드 — `X-Caller-Service` 화이트리스트 `admin-service` / `payment-ops`. 1차엔 운영자 Postman 수동 호출, 추후 admin-service 자동화. 불허 caller 는 403 ProblemDetail.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 + path 의 `priceId`.

```json
{ "planCode": "pro", "billingInterval": "MONTHLY" }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`planCode`** | string | ✅ | `@NotBlank` |
| **`billingInterval`** | enum | ✅ | `@NotNull` — `MONTHLY` / `YEARLY` |

path 파라미터. **`priceId`** (string) — 승격 대상 Paddle `pri_...`.

### Response

**200 OK** — 새로 만들어진 variant 메타(`PromoteResult`).

```json
{
  "priceId": "pri_new...",
  "planCode": "pro",
  "billingInterval": "MONTHLY",
  "enabled": true
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `priceId` | string | — | 승격된 Paddle `pri_...` |
| `planCode` | string | — | 매핑된 plan (없으면 null) |
| `billingInterval` | string | — | `MONTHLY` / `YEARLY` |
| `enabled` | boolean | — | 활성 여부 |

<details><summary><b>404 Not Found</b> — pending priceId 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "Pending priceId not found: pri_xxx" }
```

`NotFound` → `Pending priceId not found: {priceId}`.

</details>

<details><summary><b>400 Bad Request</b> — planCode 없음</summary>

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "planCode not found: pro" }
```

`PlanNotFound` → `planCode not found: {planCode}`.

</details>

<details><summary><b>409 Conflict</b> — 이미 매핑됨 / 활성 슬롯 점유</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "priceId already mapped: pri_xxx → planCode=pro" }
```

`AlreadyMapped` → `priceId already mapped: {priceId} → planCode={existingPlanCode}`. `DuplicateMapping` → `(planCode={planCode}, billingInterval={billingInterval}) already has an active priceId — use swap to replace`.

</details>

</details>

---

## 24. `POST` /api/internal/plan-variants/{newPriceId}/swap

가격 정책 세대 교체 — 옛 priceId 비활성 + 새 priceId 활성. 승격/스왑은 같은 `PendingPromoteError` enum 을 공유.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `admin-service` / `payment-ops` | ✅ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. 인라인 가드 — `admin-service` / `payment-ops`. 불허 caller 는 403 ProblemDetail.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 + path 의 `newPriceId`.

```json
{ "replacePriceId": "pri_old...", "planCode": "pro", "billingInterval": "MONTHLY" }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`replacePriceId`** | string | ✅ | `@NotBlank` — 비활성화할 옛 priceId |
| **`planCode`** | string | ✅ | `@NotBlank` |
| **`billingInterval`** | enum | ✅ | `@NotNull` — `MONTHLY` / `YEARLY` |

path 파라미터. **`newPriceId`** (string) — 활성화할 신규 Paddle `pri_...`.

### Response

**200 OK** — 옛/새 priceId 짝(`SwapResult`).

```json
{
  "oldPriceId": "pri_old...",
  "newPriceId": "pri_new...",
  "planCode": "pro",
  "billingInterval": "MONTHLY",
  "newEnabled": true
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `oldPriceId` | string | — | 비활성된 옛 priceId |
| `newPriceId` | string | — | 활성된 새 priceId |
| `planCode` | string | — | — |
| `billingInterval` | string | — | `MONTHLY` / `YEARLY` |
| `newEnabled` | boolean | — | 새 variant 활성 여부 |

<details><summary><b>404 Not Found</b> — 교체 대상 없음</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "replacePriceId not in plan_variants: pri_old..." }
```

`ReplaceTargetNotFound` → `replacePriceId not in plan_variants: {replacePriceId}`.

</details>

<details><summary><b>400 Bad Request</b> — planCode 없음 / 매핑 불일치</summary>

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "planCode not found: pro" }
```

`PlanNotFound` → `planCode not found: {planCode}`. `ReplaceMismatch` → `replacePriceId {id} 의 실제 매핑 ({actualPlanCode}, {actualBillingInterval}) 이 요청 ({requestedPlanCode}, {requestedBillingInterval}) 과 다름`.

</details>

<details><summary><b>409 Conflict</b> — 옛 priceId 가 이미 비활성</summary>

```json
{ "type": "about:blank", "title": "Conflict", "status": 409, "detail": "replacePriceId already disabled (use promote): pri_old..." }
```

`ReplaceAlreadyDisabled` → `replacePriceId already disabled (use promote): {replacePriceId}`.

</details>

</details>

---

## 25. `POST` /api/internal/subscriptions/migrate-price

옛 priceId → 새 priceId 일괄 마이그레이션. `dryRun=true` 로 영향 평가 → announcement → `dryRun=false` 실행.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `admin-service` / `payment-ops` | ✅ |
| `Content-Type` | `application/json` | ✅ |

### Security

internal-only. 인라인 가드 — `admin-service` / `payment-ops`. 불허 caller 는 403 ProblemDetail. caller 값은 마이그레이션 감사에 기록.

<details>
<summary><b>Request · Response</b></summary>

### Request

```json
{
  "fromPriceId": "pri_old...",
  "toPriceId": "pri_new...",
  "prorationMode": "prorated_immediately",
  "dryRun": true
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`fromPriceId`** | string | ✅ | `@NotBlank` |
| **`toPriceId`** | string | ✅ | `@NotBlank` |
| **`prorationMode`** | string | ✅ | `@NotBlank` — Paddle 표준 5개(`do_not_bill` / `prorated_immediately` / `prorated_next_billing_period` / `full_immediately` / `full_next_billing_period`) |
| **`dryRun`** | boolean | ✅ | `@NotNull`. true 면 영향 평가만 |

### Response

**200 OK** — 마이그레이션 결과(`MigratePriceResult`).

```json
{
  "fromPriceId": "pri_old...",
  "toPriceId": "pri_new...",
  "prorationMode": "prorated_immediately",
  "dryRun": true,
  "totalCandidates": 12,
  "succeeded": 0,
  "failed": 0,
  "skippedScheduledChange": 2,
  "failedSubscriptionIds": [],
  "skippedSubscriptionIds": ["sub_a", "sub_b"],
  "durationMs": 143
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `fromPriceId` / `toPriceId` | string | — | — |
| `prorationMode` | string | — | 요청 반향 |
| `dryRun` | boolean | — | — |
| `totalCandidates` | integer | — | 대상 subscription 수 |
| `succeeded` / `failed` | integer | — | 실행 결과 (dryRun 이면 0) |
| `skippedScheduledChange` | integer | — | scheduled_change 있어 자동 제외된 수(Paddle 가 PATCH 거절) |
| `failedSubscriptionIds` | array\<string> | — | 실패한 subscription id |
| `skippedSubscriptionIds` | array\<string> | — | 제외된 subscription id |
| `durationMs` | integer(long) | — | 소요 ms |

<details><summary><b>400 Bad Request</b> — 동일 priceId / 잘못된 proration / plan·interval 불일치</summary>

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "fromPriceId == toPriceId — 의미 없는 호출: pri_x" }
```

`SamePriceId` → `fromPriceId == toPriceId — 의미 없는 호출: {priceId}`. `InvalidProrationMode` → `Invalid prorationMode (Paddle 표준 5개): {mode}`. `FromToMismatch` → `from ({fromPlanCode}, {fromInterval}) / to ({toPlanCode}, {toInterval}) plan/interval 불일치`.

</details>

<details><summary><b>404 Not Found</b> — priceId 미매핑</summary>

```json
{ "type": "about:blank", "title": "Not Found", "status": 404, "detail": "fromPriceId not mapped in plan_variants: pri_old..." }
```

`FromPriceNotMapped` → `fromPriceId not mapped in plan_variants: {fromPriceId}`. `ToPriceNotMapped` → `toPriceId not mapped — promote/swap 부터: {toPriceId}`.

</details>

</details>

---

## 26. `GET` /api/internal/subscriptions/expiring

cancel 예약된 subscription 중 N 일 이내 만료 예정 목록(retention dashboard 용).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `X-Caller-Service` | `admin-service` / `payment-ops` | ✅ |

### Security

internal-only. 인라인 가드 — `admin-service` / `payment-ops`. 불허 caller 는 403 ProblemDetail.

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — query 파라미터만.

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `days` | integer | ❌ | 기본값 `30`. 범위 1~365(벗어나면 400) |

### Response

**200 OK** — `SubscriptionResponse` 배열 (위 GET /api/subscriptions/{id} shape).

```json
[
  { "id": "...", "paddleSubscriptionId": "sub_...", "status": "ACTIVE", "scheduledChangeAction": "cancel", "scheduledChangeEffectiveAt": "2026-07-20T00:00:00Z" }
]
```

<details><summary><b>400 Bad Request</b> — days 범위 초과</summary>

```json
{ "type": "about:blank", "title": "Bad Request", "status": 400, "detail": "days must be 1~365: 500" }
```

</details>

<details><summary><b>403 Forbidden</b> — 허용되지 않은 caller</summary>

```json
{ "type": "about:blank", "title": "Forbidden", "status": 403, "detail": "Internal endpoint — caller unknown-svc not allowed" }
```

</details>

</details>

---

## webhooks

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/webhooks/paddle`](#27-post-webhookspaddle) | — | 공개 (HMAC 서명) | Paddle 웹훅 수신 — raw body 를 그대로 받아 HMAC 서명 검증 |

---

## 27. `POST` /webhooks/paddle

Paddle 웹훅 수신 — raw body 를 그대로 받아 HMAC 서명 검증. `event_id` 기준 idempotent(Paddle 재시도 dedup).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `Paddle-Signature` | `ts=<unix>;h1=<hmac-sha256>` | ✅ |
| `Content-Type` | `application/json` | ✅ |

### Security

공개(외부 직접 호출 가능). caller allowlist 없음 — 대신 `Paddle-Signature` HMAC-SHA256 검증으로 진위 확인. 개별 handler 오류는 로깅되지만 요청은 200 반환(재처리 회피).

<details>
<summary><b>Request · Response</b></summary>

### Request

raw bytes(서명 검증 위해 원본 보존). `PaddleWebhookEvent` 로 파싱.

```json
{
  "event_type": "subscription.created",
  "event_id": "evt_...",
  "occurred_at": "...",
  "notification_id": "ntf_...",
  "data": { }
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `event_type` | string | — | 이벤트 종류 |
| `event_id` | string | — | dedup 키 |
| `occurred_at` | string(date-time) | — | — |
| `notification_id` | string | — | — |
| `data` | object | — | 이벤트별 payload |

### Response

**200 OK** — 빈 본문. 성공/부분 handler 실패 모두 200.

<details><summary><b>401 Unauthorized</b> — 서명 검증 실패</summary>

`WebhookService.processEvent` 가 HMAC 불일치 시 예외를 throw 하며 글로벌 핸들러가 4xx(401)로 매핑. 컨트롤러 자체는 정상 흐름에서 200 만 반환.

</details>

</details>

---

## test

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/api/v1/test/payment/seed-active-subscription`](#28-post-apiv1testpaymentseed-active-subscription) | — | dev 전용 | dev 전용 — Paddle webhook 없이 customer + subscription 행을… |

---

## 28. `POST` /api/v1/test/payment/seed-active-subscription

dev 전용 — Paddle webhook 없이 customer + subscription 행을 직접 INSERT 해 license `ACTIVE` 시드. `@Profile("dev-k3s")` + `payment.test-api.enabled=true` 둘 다 충족해야 bean 등록(아니면 404).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `Content-Type` | `application/json` | ✅ |

### Security

dev 전용. caller 가드 없음(`InternalCallerInterceptor` 대상 경로 아님). 운영/staging 에서는 bean 미등록이라 404. 시드된 paddle id 는 `test_sub_*` / `test_txn_*` prefix — `PaddleApiClient` 가 이 prefix 를 만나면 실 Paddle 호출을 우회하고 stub 반환.

<details>
<summary><b>Request · Response</b></summary>

### Request

```json
{
  "organizationId": "01HXORG...",
  "userId": "01HXOWNER...",
  "planCode": "pro",
  "planSeats": 20,
  "withTransaction": true
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`organizationId`** | string | ✅ | `@NotBlank` |
| **`userId`** | string | ✅ | `@NotBlank` |
| **`planCode`** | string | ✅ | `@NotBlank` |
| **`planSeats`** | integer | ✅ | `@Min(1)` |
| `withTransaction` | boolean | ❌ | true 면 환불 정방향 테스트용 `PAID` 거래 1건(+항목) 도 시드. 기본 false |

### Response

**201 Created** — `SeedSubscriptionResponse`.

```json
{
  "subscriptionId": "...",
  "customerId": "...",
  "organizationId": "01HXORG...",
  "paddleSubscriptionId": "test_sub_...",
  "licenseState": "ACTIVE",
  "transactionId": "...",
  "paddleItemId": "test_txnitm_..."
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `subscriptionId` | string | — | 시드된 subscription 로컬 pk |
| `customerId` | string | — | 시드된 customer 로컬 pk |
| `organizationId` | string | — | 요청 반향 |
| `paddleSubscriptionId` | string | — | 합성 `test_sub_...` (Paddle 미호출) |
| `licenseState` | string | — | `ACTIVE` (b2b gRPC license 와 매칭) |
| `transactionId` | string \| null | — | `withTransaction=true` 일 때만, 아니면 null |
| `paddleItemId` | string \| null | — | 시드 항목 id(`test_txnitm_...`), 아니면 null |

</details>

---

## 에러 코드 (ProblemDetail)

패턴 A — 컨트롤러 inline `toResponse(error)` 또는 남은 throw 를 `*ExceptionHandler` 가 RFC 9457 `ProblemDetail` 로 매핑한다. `code` 필드는 없다 (status + title + detail).

| HTTP | title | 상황 |
|---|---|---|
| 400 | Bad Request | request validation (`MethodArgumentNotValidException`), 잘못된 enum/파라미터 |
| 401 | Unauthorized | webhook `Paddle-Signature` HMAC 검증 실패 (`WebhookService`) |
| 403 | Forbidden | 알 수 없는 `X-Caller-Service` (internal 엔드포인트). `plan-change` 만 예외적으로 **빈 body 403** (ProblemDetail 아님) |
| 404 | Not Found | Subscription / Transaction / Refund / Plan / Variant 미존재 |
| 409 | Conflict | duplicate / state conflict (plan · variant · subscription 상태 전이) |
| 500 | Internal Server Error | Paddle API 5xx·IO 실패, `sync` 경로의 시스템 이상 (throw) |
