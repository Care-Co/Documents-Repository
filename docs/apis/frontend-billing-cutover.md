# 프론트 결제 라인 cutover 핸드오프 (FIS-440)

프론트엔드(B2B_WEB)가 payment-service 를 직접 호출하던 결제 조회를 b2b-service 경유로 일원화하기 위한 핸드오프. 백엔드(b2b `GET /api/v2/b2b/billing/plans`)는 dev 배포 완료. 프론트 전환 + 이후 gateway 축소만 남음.

---

## 1. plans 조회 전환

### 변경 대상 (프론트)

| 항목 | 지금 (직접 payment) | 전환 후 (b2b 경유) |
| --- | --- | --- |
| 요금제 조회 | `GET /api/payment/plans?audience=B2B` | `GET /api/v2/b2b/billing/plans` |

- b2b 엔드포인트는 `?organizationId=` optional. 미입력 시 공개 카탈로그 pass-through, 입력 시 해당 org 전용(엔터프라이즈 커스텀) 포함 + OWNER 검증.
- 응답은 b2b 표준 `CncResponse` 봉투(`{ success, code, message, data }`) — `data` 에 plans 배열. payment 직접 호출 시의 raw 배열과 다르므로 언랩 필요.

### 주의 — 인증 (결정 필요)

`GET /api/v2/b2b/billing/plans` 는 **인증(로그인) 필수**다. 반면 기존 `GET /api/payment/plans` 는 gateway 공개(비인증) 라우트였다.

- **로그인 후 화면에서만 plans 를 쓰면** → 그대로 전환 가능.
- **로그인 전(비인증) 요금제 노출 화면이 있으면** → b2b 엔드포인트로 못 옮긴다. 이 경우 (a) 공개 `/api/payment/plans` 를 그대로 두거나 (b) b2b 에 공개 plans 엔드포인트를 신설하는 결정이 선행돼야 한다.

→ **프론트팀 확인 필요. 로그인 전 요금제 화면 유무.**

---

## 2. `/api/customers` — 죽은 호출 제거

- 백엔드(payment / b2b) 어디에도 `/api/customers` 컨트롤러가 **없다**. gateway 라우트에도 없다.
- 프론트에 이 호출이 남아 있다면 항상 실패하는 죽은 코드 → **제거 대상**. (고객 정보가 필요하면 별도 엔드포인트 정의부터.)

---

## 3. gateway 축소 (cutover 완료 후, 백엔드/ops)

프론트가 plans 를 b2b 로 완전히 옮긴 뒤에만 안전. 공개 `/api/payment/plans` 외부 노출만 제거한다. `checkout-links` / `plan-change` 는 b2b 가 내부 호출로 계속 쓰므로 남긴다.

현재 `Payment.PAYMENT = "/api/payment/**"` 하나에 plans·checkout-links·plan-change 가 묶여 있어, 공개 plans 만 빼려면 상수를 쪼갠다. 3파일 (`gateway-service`):

1. **`config/constants/GatewayConstants.java`** (`Payment`) — `PAYMENT="/api/payment/**"` 를 세분화. 예. `PLANS_PUBLIC="/api/payment/plans"`, `CHECKOUT_LINKS="/api/payment/checkout-links/**"`, `PLAN_CHANGE="/api/payment/plan-change/**"`. cutover 완료 후 `PLANS_PUBLIC` 은 삭제.
2. **`config/GatewayRouteConfig.java`** (`"payment"` route `.path(...)`) — 세분 상수로 교체하고 제거 대상(공개 plans)은 목록에서 뺀다. 라우트에서 빠지면 외부 도달 불가 (이미 `TRANSACTIONS/SUBSCRIPTIONS/REFUNDS/CUSTOMER_PORTAL` 이 이 방식으로 내부전용 전환됨).
3. **`config/GatewaySecurityConfig.java`** (`buildAuthPolicies()`) — `auth(Payment.PAYMENT, false)` 를 세분 상수별 `auth(...)` 로 교체. 공개 plans 제거 시 해당 정책도 삭제. "구체적 경로 먼저" 순서 유지.

주의 — payment 의 `PlanCatalogController(/api/payment/plans)` 자체는 살려둔다. b2b/cs 는 org-scoped 카탈로그를 이미 클러스터 내부 `/api/internal/plans` 로 호출하므로, gateway 공개 라우트만 빼도 내부 조회는 영향 없다.

---

## 4. 참고 (백엔드 상태)

- b2b `GET /api/v2/b2b/billing/plans` — dev 배포 완료. `BillingController` / `BillingService.listPlans`.
- payment org-scoped 카탈로그 — `GET /api/internal/plans` (gateway 미라우팅, X-Caller 가드).
- 관련. `docs/apis/b2b-service.md` (billing plans 절), Documents-Repository PR #17(머지됨).
