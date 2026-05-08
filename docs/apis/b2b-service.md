# B2B Service API

> Updated: 2026-05-06
> Style: OpenAPI 3.1 + diagrams + code chain mapping
> Base path: `/api/v2/b2b/...`
> 응답 envelope: [`CncResponse`](#cncresponse) (Pattern B)

---

## 1. 개요

운동 시설 (gym/pilates/yoga/PT_studio/crossfit/functional/boxing) B2B 관리 서비스. 관리자 계정 + 시설 + 멤버 + 디바이스 + 측정 데이터 노출 + 피드백 + 결제 진입점.

### 1.1 도메인

| 도메인 | 책임 |
|---|---|
| `auth` | 인증 / 세션 / JWT family rotation / OAuth2 (Google + Apple) |
| `user` | b2b_user (관장/스태프/트레이너) CRUD |
| `organization` | 시설 + 초대 코드 + 멤버십 lifecycle (master) |
| `feedback` | 트레이너 → 회원 피드백 (1:N 코멘트) |
| `external/license` | payment-service gRPC 소비 + Redis 캐시 + LicenseGuard + invalidate 수신 |
| `external/device` | device-service gRPC wrap |
| `external/measurement` | measure-service gRPC wrap |
| `external/payment` | payment-service REST 호출 (결제 시작) |
| `billing` | 결제 진입 — OWNER 검증 후 payment 위임 |

### 1.2 외부 의존

```mermaid
flowchart LR
    subgraph b2b[b2b-service]
        BC[BillingController]
        OC[Organization/Membership/Invite/Device]
        MC[MeasurementController]
        FC[FeedbackController]
        AC[AuthController/UserController]
        LIG[LicenseInvalidate gRPC<br/>:9090]
    end

    subgraph payment[payment-service]
        SQS[SubscriptionQuery gRPC<br/>:9090]
        CHK[/api/payment/checkout-links/]
    end

    subgraph device[device-service]
        DM[DeviceManagement gRPC<br/>:9090]
    end

    subgraph measure[measure-service]
        MQ[MeasurementQuery gRPC<br/>:9090]
    end

    REDIS[(Redis<br/>b2b:license:*<br/>5min TTL)]
    PG[(PostgreSQL<br/>b2b schema)]

    BC -- HTTP --> CHK
    OC -.consult.-> LC[LicenseClient]
    LC -- gRPC --> SQS
    LC <--> REDIS
    payment -- gRPC notify --> LIG
    LIG --> REDIS
    OC -- gRPC --> DM
    MC -- gRPC --> MQ
    b2b <--> PG
```

---

## 2. Architecture — Cross-MSA 데이터 흐름

### 2.1 결제 시작 (Billing)

```mermaid
sequenceDiagram
    autonumber
    participant W as B2B_WEB
    participant B as b2b-service<br/>BillingController
    participant L as LicenseClient
    participant P as payment-service
    participant Pad as Paddle

    W->>B: POST /api/v2/b2b/billing/checkout-init<br/>{ organizationId, priceId }
    Note over B: 1) Org 존재 + ACTIVE<br/>2) ownerUserId == principal<br/>3) license state ≠ ACTIVE/SUSPENDED
    B->>L: getLicense(orgId)
    L-->>B: LicenseInfo (NONE/EXPIRED/GRACE 만 통과)
    B->>P: POST /api/payment/checkout-links<br/>X-Caller-Service: b2b-service
    Note over P: 4) caller whitelist 검증<br/>5) priceId in subscription_plans<br/>6) Customer 보장 (Paddle)<br/>7) customData = { organization_id }
    P-->>B: CheckoutLinkResponse
    B-->>W: 응답 echo
    W->>Pad: Paddle.Checkout.open(config)
```

### 2.2 회원 가입 + 좌석 점유 (검색 + 승인)

```mermaid
sequenceDiagram
    autonumber
    participant U as Carenco User
    participant B as b2b-service
    participant O as OWNER
    participant L as LicenseClient
    U->>B: POST /organizations/search?keyword=...
    B-->>U: items[]
    U->>B: POST /organizations/{orgId}/join-requests<br/>{ memberId, note }
    Note over B: membership PENDING + joinedVia=SEARCH_REQUEST
    O->>B: POST /memberships/{id}/approve
    B->>L: getLicense(orgId)
    L-->>B: LicenseInfo
    B->>B: LicenseGuard.allowSeatConsumption(license, seatsUsed)
    Note over B: organization.seatsUsed += 1<br/>(@Version optimistic lock)<br/>membership.status = ACTIVE
    B-->>O: 200 ACTIVE
```

### 2.3 초대 코드 1회성 가입

```mermaid
sequenceDiagram
    autonumber
    participant O as OWNER
    participant U as Carenco User
    participant B as b2b-service
    participant S as MembershipSeatService
    O->>B: POST /organizations/{orgId}/invite-codes
    Note over B: 헷갈리는 문자 회피 generator<br/>+48h TTL, status=ACTIVE
    B-->>O: { code: "Hk7mPqR3" }
    O->>U: 코드 공유
    U->>B: POST /invite-codes/redeem<br/>{ code, memberId }
    Note over B: code 매칭 + 만료 검증
    B->>S: consumeSeat(membershipId)
    S-->>B: ACTIVE (또는 SeatsExhausted/LicenseBlocked)
    Note over B: invite.status=USED + audit
    B-->>U: { membershipId, status: ACTIVE }
```

### 2.4 측정 데이터 + 피드백 (트레이너)

```mermaid
sequenceDiagram
    autonumber
    participant T as TRAINER
    participant B as b2b-service
    participant M as MeasurementClient
    participant ME as measure-service
    T->>B: GET /organizations/{org}/members/{memberId}/measurements
    Note over B: roleResolver — OWNER/ADMIN/TRAINER<br/>memberId 가 ACTIVE 멤버
    B->>M: listByUser(memberId, from, to, page, size)
    M->>ME: ListMeasurementsByUser [gRPC]
    ME-->>M: items + totalCount
    M-->>B: Page<MeasurementInfo>
    B-->>T: 200 { items }
    T->>B: POST /feedbacks/memberships/{id}<br/>{ body, visibility, measurementRecordId }
    B-->>T: 201 { feedbackId }
```

### 2.5 License invalidate (양방향 sync)

```mermaid
sequenceDiagram
    Paddle->>P payment-service: subscription.canceled
    P->>P: subscriptions.status=CANCELED
    P->>+B b2b-service: InvalidateLicense(orgId) [gRPC]
    B->>Redis: DEL b2b:license:{orgId}
    B-->>-P: { invalidated: true }
    Note over Paddle,B: 다음 b2b 의 getLicense 호출 시<br/>새 상태 즉시 반영
```

### 2.6 자정 batch (PAST_DUE → EXPIRED)

```mermaid
sequenceDiagram
    Note over P: 매일 00:05 KST cron<br/>+ ShedLock JDBC 분산락
    P->>P: findByStatusAndNextBilledAtBefore(PAST_DUE, now-7d)
    P->>P: 모두 status=EXPIRED 로 update
    P->>+B b2b-service: InvalidateMany([orgIds]) [gRPC]
    B->>Redis: DEL multiple keys
    B-->>-P: { invalidatedCount, notCachedCount }
```

---

## 3. Endpoint Catalog (Domain x HTTP Method)

표기: 🔓 Public / 🔒 Auth / 👑 OWNER/ADMIN / 👥 OWNER/ADMIN/TRAINER

### 3.1 Auth

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/auth/login` | 🔓 | `AuthController.login` → `AuthService.authenticateForToken/authenticateForSession` |
| POST | `/api/v2/b2b/auth/token` | 🔓 | `AuthController.refresh` → `JwtTokenService.rotateRefresh` |
| POST | `/api/v2/b2b/auth/logout` | 🔓 | `AuthController.logout` → `TokenRevocationService.revokeJti` |
| GET | `/api/v2/b2b/auth/me` | 🔒 | `AuthController.me` (간단 — `UserController.me` 가 detail) |
| POST | `/api/v2/b2b/auth/oauth2/google` | 🔓 | `OAuth2Controller.googleLogin` → `OAuth2LoginService` + `GoogleIdTokenVerifier` |
| POST | `/api/v2/b2b/auth/oauth2/apple` | 🔓 | `OAuth2Controller.appleLogin` → `OAuth2LoginService` + `AppleIdTokenVerifier` |

### 3.2 User

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/users` | 🔓 | `UserController.signUp` → `UserService.signUp` |
| GET | `/api/v2/b2b/users/{b2bUserId}` | 🔒 (self) | `UserController.get` → `UserService.getMeWithOrganizations` (organization+license enrich) |
| PATCH | `/api/v2/b2b/users/{b2bUserId}` | 🔒 (self) | `UserController.update` → `UserService.updateProfile` |
| POST | `/api/v2/b2b/users/{b2bUserId}/password` | 🔒 (self) | `UserController.changePassword` → `UserService.changePassword` + `AuthService.revokeAllSessions` |
| DELETE | `/api/v2/b2b/users/{b2bUserId}` | 🔒 (self) | `UserController.delete` → `UserService.deleteAccount` |

**`b2bUserId` 검증**: path 의 ID 가 인증된 principal 과 일치해야 통과 (`UserController.ensureSelf`). 불일치 시 `403 PERMISSION_DENIED` ([`UserError.NotSelf`](#userresponse))

### 3.3 Organization

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/organizations` | 🔒 | `OrganizationController.create` → `OrganizationService.create` |
| GET | `/api/v2/b2b/organizations/{id}` | 🔓 | `OrganizationController.get` → `OrganizationService.get` |
| PATCH | `/api/v2/b2b/organizations/{id}` | 👑 | `OrganizationController.update` → `OrganizationService.update` (owner check) |
| GET | `/api/v2/b2b/organizations/search` | 🔓 | `OrganizationController.search` → `OrganizationRepository.searchActive` |
| GET | `/api/v2/b2b/organizations/mine` | 🔒 | `OrganizationController.mine` → `OrganizationService.listByOwner` |

### 3.4 Membership

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/organizations/{orgId}/join-requests` | 🔓 | `MembershipController.requestJoin` → `MembershipFlowService.requestJoin` |
| POST | `/api/v2/b2b/memberships/{id}/approve` | 👑 | `MembershipController.approve` → `MembershipFlowService.approve` → `MembershipSeatService.consumeSeat` |
| POST | `/api/v2/b2b/memberships/{id}/reject` | 👑 | `MembershipController.reject` → `MembershipFlowService.reject` |
| POST | `/api/v2/b2b/memberships/{id}/suspend` | 👑 | `MembershipController.suspend` → `MembershipFlowService.suspend` |
| POST | `/api/v2/b2b/memberships/{id}/leave` | 🔓 | `MembershipController.leave` → `MembershipFlowService.leave` (body의 requesterId 자기검증) |
| POST | `/api/v2/b2b/organizations/{orgId}/staff` | 👑 | `MembershipController.appointStaff` → `MembershipFlowService.appointB2bRole` |
| GET | `/api/v2/b2b/organizations/{orgId}/members?status=` | 🔒 | `MembershipController.listMembers` → `MembershipFlowService.listByOrganization` |

### 3.5 Invite Code

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/organizations/{orgId}/invite-codes` | 👑 | `InviteCodeController.issue` → `InviteCodeService.issue` + `InviteCodeGenerator` |
| GET | `/api/v2/b2b/organizations/{orgId}/invite-codes` | 🔒 | `InviteCodeController.list` → `InviteCodeService.listActive` |
| POST | `/api/v2/b2b/invite-codes/redeem` | 🔓 | `InviteCodeController.redeem` → `InviteCodeService.redeem` → seat cascade |
| POST | `/api/v2/b2b/invite-codes/{id}/revoke` | 👑 | `InviteCodeController.revoke` → `InviteCodeService.revoke` |

### 3.6 Device

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/organizations/{orgId}/devices` | 👑 | `DeviceController.register` → `DeviceClient.registerDevice` (gRPC → device-service) |
| GET | `/api/v2/b2b/organizations/{orgId}/devices` | 🔒 | `DeviceController.list` → `DeviceClient.listByOrganization` |
| GET | `/api/v2/b2b/organizations/{orgId}/devices/{deviceId}` | 🔒 | `DeviceController.get` → `DeviceClient.getDevice` |
| PATCH | `/api/v2/b2b/organizations/{orgId}/devices/{deviceId}` | 👑 | `DeviceController.update` → `DeviceClient.updateDevice` |
| POST | `/api/v2/b2b/organizations/{orgId}/devices/{deviceId}/deactivate` | 👑 | `DeviceController.deactivate` → `DeviceClient.deactivateDevice` |

### 3.7 Measurement

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| GET | `.../members/{memberId}/measurements` | 👥 | `MeasurementController.list` → `MeasurementClient.listByUser` (gRPC) |
| GET | `.../measurements/{recordId}` | 👥 | `MeasurementController.get` → `MeasurementClient.get` |
| GET | `.../measurements/summary` | 👥 | `MeasurementController.summary` → `MeasurementClient.getSummary` |

(전체 path: `/api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/...`)

### 3.8 Feedback

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/feedbacks/memberships/{id}` | 👥 | `FeedbackController.create` → `FeedbackService.create` |
| PATCH | `/api/v2/b2b/feedbacks/{id}` | 작성자 | `FeedbackController.edit` → `FeedbackService.edit` |
| DELETE | `/api/v2/b2b/feedbacks/{id}` | 작성자 또는 👑 | `FeedbackController.delete` → `FeedbackService.delete` (soft) |
| GET | `/api/v2/b2b/feedbacks/memberships/{id}` | 🔒 | `FeedbackController.listForMember` → `FeedbackService.listForMember` |
| GET | `/api/v2/b2b/feedbacks/measurements/{recordId}` | 🔒 | `FeedbackController.listForMeasurement` |

### 3.9 License Summary

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| GET | `/api/v2/b2b/license-summary` | 🔒 | `LicenseSummaryController.mine` → 본인 owner+ACTIVE 멤버 organization 들 |
| GET | `/api/v2/b2b/license-summary/{orgId}` | 🔒 (멤버) | `LicenseSummaryController.get` → `LicenseClient.getLicense` + `MembershipRoleResolver` |

### 3.10 Billing

| Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|
| POST | `/api/v2/b2b/billing/checkout-init` | 👑 (OWNER) | `BillingController.checkoutInit` → `BillingService.initCheckout` → `RestPaymentClient.createCheckoutLink` |

---

## 4. Endpoint Detail (선택 항목)

### 4.1 `POST /api/v2/b2b/billing/checkout-init`

> 결제 시작 진입점. b2b 가 권한 검증 후 payment-service 위임.

**Auth**: Bearer (b2b JWT) — OWNER 권한

**Request body**

```json
{ "organizationId": "org-uuid", "priceId": "pri_xxx" }
```

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [{ "priceId": "pri_xxx", "quantity": 1 }],
    "customer": { "email": "owner@example.com", "name": "박관장" },
    "customData": { "organization_id": "org-uuid" },
    "plan": {
      "planCode": "PILATES_BASIC", "planSeats": 50,
      "displayName": "필라테스 기본 (50석)",
      "amount": 99000, "currency": "KRW"
    },
    "paddleCustomerId": "ctm_xxx"
  }
}
```

**Errors** (`BillingError` sealed)

| HTTP | Error Code | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-002` | NotOwner |
| 403 | `AUTH-403-001` | OrganizationNotActive / LicenseSuspended |
| 409 | `CMN-409-001` | AlreadyHasActiveSubscription (state=ACTIVE) |
| 502 | `CMN-502-001` | PaymentServiceUnavailable |

**연쇄 호출**

```
BillingController → BillingService.initCheckout
  ├─ OrganizationRepository.findById
  ├─ LicenseClient.getLicense (Redis 캐시 우선)
  ├─ B2bUserRepository.findById (owner 본인)
  └─ PaymentClient.createCheckoutLink (HTTP → payment-service)
```

---

### 4.2 `POST /api/v2/b2b/memberships/{id}/approve`

> 가입 신청 PENDING → ACTIVE 전이 + 좌석 +1.

**Auth**: Bearer — OWNER/ADMIN

**Response — 200** — `MembershipResponse` (status=ACTIVE, joinedAt 채워짐)

**Errors** (`MembershipFlowError` sealed)

| HTTP | 케이스 |
|---|---|
| 404 | MembershipNotFound |
| 403 | NotOwnerOrAdmin |
| 409 | InvalidStateTransition / AlreadyMember |
| 403 | SeatConsumeFailed (license SUSPENDED / 만석) |

**연쇄 호출**

```
MembershipController → MembershipFlowService.approve
  ├─ MembershipRoleResolver.resolve (역할 확인)
  ├─ MembershipFlowService 내부 검증
  └─ MembershipSeatService.consumeSeat
       ├─ OrganizationRepository.findById
       ├─ LicenseClient.getLicense
       ├─ LicenseGuard.allowSeatConsumption
       ├─ organization.setSeatsUsed (+1, optimistic lock)
       └─ membership.setStatus(ACTIVE)
```

---

### 4.3 `POST /api/v2/b2b/invite-codes/redeem`

> Carenco user 가 코드로 1회성 가입.

**Auth**: 🔓 Public (body memberId 가 carenco userId)

**Request body**

```json
{ "code": "Hk7mPqR3", "memberId": "carenco-user-uuid" }
```

**Response — 200**

```json
{
  "success": true,
  "data": {
    "inviteCode": { "id": "...", "code": "Hk7mPqR3", "status": "USED",
                    "usedAt": "...", "usedByMemberId": "...", "usedMembershipId": "..." },
    "membershipId": "m-uuid",
    "organizationId": "org-uuid",
    "status": "ACTIVE"
  }
}
```

**Errors** (`InviteCodeError` sealed)

| HTTP | 케이스 |
|---|---|
| 404 | InviteCodeNotFound |
| 403 | InviteCodeExpired / InviteCodeNotActive |
| 409 | AlreadyMember |
| 403 | LicenseBlocked (SeatsExhausted 포함) |

**연쇄 호출**

```
InviteCodeController.redeem → InviteCodeService.redeem
  ├─ InviteCodeRepository.findByCode
  ├─ 만료 검증 (expiresAt < now)
  ├─ 중복 멤버 검증
  ├─ Membership PENDING row 생성
  └─ MembershipSeatService.consumeSeat (license guard + seats+1)
       └─ 성공 시 InviteCode.status=USED + audit
```

---

### 4.4 `GET /api/v2/b2b/users/{b2bUserId}`

> 본인 프로필 + 소속 organization (multi-org) + license 상태.
>
> `b2bUserId` 가 인증된 principal 과 일치해야 함 (불일치 시 403). user-service `/api/v2/users/{userId}` 와 일관 패턴.

**Response — 200**

```json
{
  "data": {
    "id": "u-uuid", "email": "owner@example.com",
    "firstName": "박", "lastName": "관장", "country": "KR",
    "accountStatus": "ACTIVE",
    "organizations": [
      { "id": "org-A", "name": "강남점", "type": "PILATES",
        "role": "OWNER", "seatsUsed": 12, "planSeats": 50, "licenseState": "ACTIVE" },
      { "id": "org-B", "name": "분당점", "type": "PILATES",
        "role": "ADMIN", "seatsUsed": 7, "planSeats": 30, "licenseState": "GRACE" }
    ]
  }
}
```

**연쇄 호출**

```
UserController.get → ensureSelf(pathUserId, principal)   // 403 if mismatch
            → UserService.getMeWithOrganizations
  ├─ B2bUserRepository.findById
  ├─ OrganizationRepository.findByOwnerUserId (OWNER orgs)
  ├─ OrganizationMembershipRepository.findByMemberTypeAndMemberId(B2B, userId)
  └─ LicenseClient.batchGetLicense (한 번에 조회)
       ├─ Redis cache hits → 즉시
       └─ miss → payment-service gRPC BatchGetLicenseStatus
```

---

(다른 endpoint detail 은 코드 매핑 표 (3절) 참조 — 모두 동일 Pattern B 응답 구조)

---

## 5. License Guard 매트릭스

`external/license/LicenseGuard.java` — 4종 결정.

| Operation | ACTIVE | GRACE | EXPIRED | SUSPENDED | NONE |
|---|---|---|---|---|---|
| 좌석 점유 (membership 가입) | ✅ (seats < plan) | ❌ | ❌ | ❌ | ❌ |
| 코드 발급 | ✅ | ✅ | ❌ | ❌ | ❌ |
| Device 등록 | ✅ | ✅ | ❌ | ❌ | ❌ |
| 일반 read (조회) | ✅ | ✅ | ✅ | ❌ | ✅ |

`getLicense` 실패 (payment-service 다운) → `LicenseInfo.none()` 반환 → 위 표의 NONE 컬럼 적용 → 좌석/코드/device write 차단, read 통과.

---

## 6. Cross-MSA 호출 정리

```mermaid
flowchart TD
    subgraph in[b2b 외부 호출]
        L1[LicenseClient.getLicense<br/>→ payment.SubscriptionQuery] --> SQS[payment :9090]
        L2[DeviceClient<br/>→ device.DeviceManagement] --> DM[device :9090]
        L3[MeasurementClient<br/>→ measure.MeasurementQuery] --> MQ[measure :9090]
        L4[PaymentClient.createCheckoutLink<br/>→ payment REST] --> CHK[payment :8080]
    end

    subgraph out[b2b 외부 수신]
        LIG[LicenseInvalidate gRPC :9090<br/>← payment 가 webhook/batch 후 호출]
    end
```

| 채널 | 주체 | endpoint | 빈도 | failure 정책 |
|---|---|---|---|---|
| b2b → payment (read) | LicenseClient | gRPC SubscriptionQuery.GetLicenseStatus | 모든 license guard 호출 (캐시 hit 시 미발생) | fail-open → NONE |
| b2b → payment (write) | PaymentClient | REST POST /checkout-links | billing/checkout-init 시 1회 | fail-fast → BillingError |
| b2b → device | DeviceClient | gRPC DeviceManagement | device CRUD 시 | read fail-open / write fail-fast |
| b2b → measure | MeasurementClient | gRPC MeasurementQuery | trainer 측정 조회 시 | fail-open (빈 결과) |
| payment → b2b | B2bLicenseInvalidateClient | gRPC LicenseInvalidate | webhook + 자정 batch | fail-open (b2b 가 5min TTL 자연 만료) |

---

## 7. Schemas

### CncResponse

```json
{
  "success": boolean,
  "code": "string (예: 200, CMN-404-001)",
  "message": "string (i18n resolved)",
  "data": <any>,
  "error": "string? (failure 시 description)",
  "token": "string? (TokenPair 등)"
}
```

### TokenPair

```json
{ "accessToken": "...", "refreshToken": "...", "expiresIn": 900 }
```

### OrganizationResponse

```json
{
  "id": "...", "name": "...", "type": "PILATES|GYM|YOGA|...",
  "description": "...", "phoneNumber": "...", "photoUrl": "...",
  "address": {
    "country": "KR", "postalCode": "06000",
    "regionLevel1": "서울", "regionLevel2": "강남구",
    "addressLine1": "...", "addressLine2": "..."
  },
  "ownerUserId": "...",
  "searchable": true, "inviteCodeEnabled": true, "approvalRequired": true,
  "seatsUsed": 12, "status": "ACTIVE"
}
```

### MembershipResponse

```json
{
  "id": "...", "organizationId": "...",
  "memberType": "B2B|CARENCO", "memberId": "...",
  "role": "OWNER|ADMIN|TRAINER|MEMBER",
  "status": "PENDING|ACTIVE|SUSPENDED|LEFT",
  "memberNumber": "M-001",
  "joinedVia": "OWNER_INIT|SEARCH_REQUEST|INVITE_CODE|DIRECT_INVITE",
  "inviteCodeId": "...?",
  "approvedBy": "...?", "approvedAt": "...?",
  "joinedAt": "...?", "leftAt": "...?",
  "note": "..."
}
```

### LicenseSummaryResponse

```json
{
  "organizationId": "...", "organizationName": "...",
  "state": "ACTIVE|GRACE|EXPIRED|SUSPENDED|NONE",
  "planCode": "PILATES_BASIC", "planSeats": 50,
  "seatsUsed": 12, "seatsRemaining": 38,
  "startedAt": "...", "expiresAt": "...",
  "graceRemainingDays": 0,
  "subscriptionId": "..."
}
```

### CheckoutLinkResponse (`PaymentCheckoutLinkResponse` echo)

```json
{
  "items": [{ "priceId": "pri_xxx", "quantity": 1 }],
  "customer": { "email": "...", "name": "..." },
  "customData": { "organization_id": "..." },
  "plan": { "planCode": "...", "planSeats": 50, "displayName": "...", "amount": 99000, "currency": "KRW" },
  "paddleCustomerId": "ctm_xxx?"
}
```

---

## 8. Error Code (`ErrorCode` enum, common-core)

| Code | HTTP | 사용처 |
|---|---|---|
| `CMN-400-001` | 400 | CHECK_PARAMETER (validation) |
| `CMN-400-002` | 400 | VALIDATION_FAILED |
| `AUTH-401-001` | 401 | TOKEN_INVALID |
| `AUTH-401-003` | 401 | AUTHENTICATION_FAILED (login fail, oauth2) |
| `AUTH-403-001` | 403 | ACCESS_DENIED (license/seat) |
| `AUTH-403-002` | 403 | PERMISSION_DENIED (role) |
| `AUTH-403-003` | 403 | ACCOUNT_DISABLED (DELETION_PENDING) |
| `CMN-404-001` | 404 | RESOURCE_NOT_FOUND |
| `USR-404-001` | 404 | USER_NOT_FOUND |
| `CMN-409-001` | 409 | DUPLICATE_REQUEST (이미 멤버 / 좌석 만석 / 상태 충돌) |
| `CMN-502-001` | 502 | EXTERNAL_SERVICE_ISSUE (payment 다운 등) |

---

## 9. Redis Namespace

```
b2b:revoked:jti:{jti}           access TTL (15min)   — JWT JTI revocation
b2b:blocked:user:{userId}       24h or longer        — 비밀번호 변경/탈퇴 시
b2b:license:{orgId}             5min                 — LicenseClient 캐시
spring:session:b2b:*            30min                — Spring Session
```

---

## 10. 인프라 / 외부 의존 요약

```mermaid
flowchart LR
    classDef internal fill:#dff,stroke:#06f
    classDef external fill:#fed,stroke:#f60
    classDef infra fill:#dfd,stroke:#080

    B2B[b2b-service\n:8080 REST\n:9090 LicenseInvalidate gRPC]:::internal
    PAY[payment-service\n:8080 REST\n:9090 SubscriptionQuery]:::internal
    DEV[device-service\n:9090 DeviceManagement]:::internal
    MEAS[measure-service\n:9090 MeasurementQuery]:::internal
    USER[user-service\nfuture batch\nGetUserBasic]:::internal

    REDIS[(Redis\nb2b:* prefix)]:::infra
    PG[(PostgreSQL\nb2b schema)]:::infra
    CONFIG[(Spring Config Server\nconfig.carencoinc.com)]:::infra

    PADDLE[Paddle Billing API\nwebhook + REST]:::external
    GOOGLE[Google JWKS\noauth2.googleapis.com]:::external
    APPLE[Apple JWKS\nappleid.apple.com]:::external

    B2B <--> REDIS
    B2B <--> PG
    B2B <-- gRPC --> PAY
    B2B -- REST --> PAY
    B2B -- gRPC --> DEV
    B2B -- gRPC --> MEAS
    PAY -- gRPC --> B2B
    PAY <--> PADDLE
    B2B -- JWKS --> GOOGLE
    B2B -- JWKS --> APPLE
    B2B <-- config --> CONFIG
```

---

## 11. 데이터 모델 (b2b schema, PostgreSQL)

```mermaid
erDiagram
    b2b_users ||--o{ b2b_user_providers : "1:N"
    b2b_users ||--o{ b2b_organizations : "owns (1:N)"
    b2b_organizations ||--o{ b2b_organization_invite_codes : "1:N"
    b2b_organizations ||--o{ b2b_organization_memberships : "1:N"
    b2b_organizations ||--o{ b2b_organization_feedbacks : "1:N"
    b2b_organization_memberships ||--o{ b2b_organization_feedbacks : "target (1:N)"
    b2b_users ||--o{ b2b_jwt_tokens : "1:N (family rotation)"

    b2b_users {
        VARCHAR id PK
        VARCHAR email UK
        VARCHAR password "BCrypt, social-only NULL"
        VARCHAR first_name
        VARCHAR last_name
        VARCHAR phone_number
        VARCHAR country "ISO-3166 (V4)"
        VARCHAR account_status "ACTIVE/SUSPENDED/BANNED/DELETION_PENDING/DELETED"
    }

    b2b_user_providers {
        VARCHAR id PK
        VARCHAR user_id FK
        VARCHAR provider_type "LOCAL/GOOGLE/APPLE"
        VARCHAR provider_user_id "social sub"
    }

    b2b_organizations {
        VARCHAR id PK
        VARCHAR name
        VARCHAR type "PILATES/GYM/YOGA/..."
        VARCHAR country "ISO-3166"
        VARCHAR address_line_1_2
        VARCHAR owner_user_id FK
        BOOL searchable
        BOOL invite_code_enabled
        BOOL approval_required
        INT seats_used "b2b master"
        VARCHAR status "ACTIVE/SUSPENDED"
        BIGINT version "optimistic lock"
    }

    b2b_organization_invite_codes {
        VARCHAR id PK
        VARCHAR organization_id FK
        VARCHAR code UK "6~12자리 alphanum"
        DATETIME expires_at "now+48h"
        DATETIME used_at "1회성"
        VARCHAR used_by_member_id "audit"
        VARCHAR used_membership_id "audit"
        VARCHAR status "ACTIVE/USED/EXPIRED/REVOKED"
    }

    b2b_organization_memberships {
        VARCHAR id PK
        VARCHAR organization_id FK
        VARCHAR member_type "B2B/CARENCO discriminator"
        VARCHAR member_id
        VARCHAR role "OWNER/ADMIN/TRAINER/MEMBER"
        VARCHAR status "PENDING/ACTIVE/SUSPENDED/LEFT"
        VARCHAR member_number "센터 회원번호 unique"
        VARCHAR joined_via "OWNER_INIT/SEARCH_REQUEST/INVITE_CODE/DIRECT_INVITE"
    }

    b2b_organization_feedbacks {
        VARCHAR id PK
        VARCHAR organization_id FK
        VARCHAR member_membership_id FK
        VARCHAR author_user_id FK
        VARCHAR measurement_record_id "optional"
        TEXT body
        VARCHAR visibility "MEMBER/INTERNAL"
        DATETIME deleted_at "soft delete"
    }

    b2b_jwt_tokens {
        VARCHAR id PK
        VARCHAR user_id FK
        VARCHAR family_id "rotation chain"
        CHAR refresh_token_hash "sha256"
        DATETIME expires_at
        DATETIME absolute_expires_at "+90d"
        VARCHAR revoked_by "rotation/logout/family_compromised"
    }
```

---

## 12. Build / Deploy

| 항목 | 값 |
|---|---|
| Java | 25 |
| Spring Boot | 4.0.6 |
| common-libs | `carencoPlatformVersion=0.0.46` |
| DB | PostgreSQL (TIMESTAMP(6) / VARCHAR / payment-service 와 같은 인스턴스, DB 분리) |
| Flyway | V1 init / V2 jwt / V3 feedback / V4 user country |
| Build | `./gradlew compileJava` (config-server 필요 시 dev profile + CONFIG_URI) |
| Postman | `b2b-service/postman/B2B-Service-API.postman_collection.json` (7 시나리오) |

---

## 13. 미구현 / 후속 (roadmap.md 참조)

| 항목 | 상태 |
|---|---|
| 회원 검색 cross-MSA 이름/전화 enrich | b2b 자체 데이터만 (member_number, joined_at) ✅ / user-service batch 미구현 ❌ |
| 측정 상태 필터 (측정 필요/미측정/측정 완료) | ❌ |
| 리포트 scope 분기 (ACTIVE/GRACE → full, EXPIRED → basic + locked_items) | ❌ |
| WebSocket/SSE 실시간 동기화 (회원 가입/측정 푸시) | ❌ |
| 회원 history (재가입 추적) | ❌ |

전체 로드맵: [`b2b-service/docs/roadmap.md`](../../../b2b-service/docs/roadmap.md)
