# B2B Service API

> Updated: 2026-05-13
> Style: OpenAPI 3.1 + diagrams + code chain mapping + full request/response samples
> Base path: `/api/v2/b2b/...`
> 응답 envelope: [`CncResponse`](#71-cncresponse-envelope) (Pattern B)

---

## 1. 개요

운동 시설 (gym/pilates/yoga/PT_studio/crossfit/functional/boxing) B2B 관리 서비스. 관리자 계정 + 시설 + 멤버 + 디바이스 + 측정 데이터 노출 + 피드백 + 결제 진입점.

### 1.1 도메인

| 도메인 | 책임 |
|---|---|
| `auth` | 인증 / 세션 / JWT family rotation / OAuth2 (Google + Apple) |
| `user` | b2b_user (관장/스태프/트레이너) CRUD |
| `organization` | 시설 + 초대 코드 + 멤버십 lifecycle (master) + 회원목록 enrich |
| `feedback` | 트레이너 → 회원 피드백 (visibility=MEMBER/INTERNAL) |
| `external/license` | payment-service gRPC 소비 + Redis 캐시 + LicenseGuard + invalidate 수신 |
| `external/device` | device-service gRPC wrap |
| `external/measurement` | measure-service gRPC wrap |
| `external/user` | user-service gRPC wrap (회원목록 enrich) |
| `external/payment` | payment-service REST 호출 (결제 시작) |
| `billing` | 결제 진입 — OWNER 검증 후 payment 위임 |
| `archive` | account 탈퇴/멤버십 LEFT 의 outbox + Carenco Archive Collector gRPC publish |

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

    subgraph user[user-service]
        UB[UserBasic gRPC<br/>:9090]
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
    OC -- gRPC --> UB
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
    U->>B: GET /organizations/search?keyword=...
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

## 3. Endpoint Catalog

표기. 🔓 Public / 🔒 Auth / 👑 OWNER/ADMIN / 👥 OWNER/ADMIN/TRAINER / 🧍 self only.

| # | Method | Path | 권한 | 코드 매핑 |
|---|---|---|---|---|
| 1 | POST | `/api/v2/b2b/auth/login` | 🔓 | [§4.1](#41-post-apiv2b2bauthlogin) |
| 2 | POST | `/api/v2/b2b/auth/token` | 🔓 | [§4.2](#42-post-apiv2b2bauthtoken) |
| 3 | POST | `/api/v2/b2b/auth/logout` | 🔓 | [§4.3](#43-post-apiv2b2bauthlogout) |
| 4 | GET | `/api/v2/b2b/auth/me` | 🔒 | [§4.4](#44-get-apiv2b2bauthme) |
| 5 | POST | `/api/v2/b2b/auth/oauth2/google` | 🔓 | [§4.5](#45-post-apiv2b2bauthoauth2google) |
| 6 | POST | `/api/v2/b2b/auth/oauth2/apple` | 🔓 | [§4.6](#46-post-apiv2b2bauthoauth2apple) |
| 7 | POST | `/api/v2/b2b/users` | 🔓 | [§4.7](#47-post-apiv2b2busers) |
| 8 | GET | `/api/v2/b2b/users/{b2bUserId}` | 🧍 | [§4.8](#48-get-apiv2b2busersb2buserid) |
| 9 | PATCH | `/api/v2/b2b/users/{b2bUserId}` | 🧍 | [§4.9](#49-patch-apiv2b2busersb2buserid) |
| 10 | POST | `/api/v2/b2b/users/{b2bUserId}/password` | 🧍 | [§4.10](#410-post-apiv2b2busersb2buseridpassword) |
| 11 | DELETE | `/api/v2/b2b/users/{b2bUserId}` | 🧍 | [§4.11](#411-delete-apiv2b2busersb2buserid) |
| 12 | POST | `/api/v2/b2b/organizations` | 🔒 | [§4.12](#412-post-apiv2b2borganizations) |
| 13 | GET | `/api/v2/b2b/organizations/{id}` | 🔓 | [§4.13](#413-get-apiv2b2borganizationsid) |
| 14 | PATCH | `/api/v2/b2b/organizations/{id}` | 👑 | [§4.14](#414-patch-apiv2b2borganizationsid) |
| 15 | DELETE | `/api/v2/b2b/organizations/{id}` | 👑 | [§4.15](#415-delete-apiv2b2borganizationsid) |
| 16 | GET | `/api/v2/b2b/organizations/search` | 🔓 | [§4.16](#416-get-apiv2b2borganizationssearch) |
| 17 | GET | `/api/v2/b2b/organizations/mine` | 🔒 | [§4.17](#417-get-apiv2b2borganizationsmine) |
| 18 | POST | `/api/v2/b2b/organizations/{orgId}/join-requests` | 🔓 | [§4.18](#418-post-apiv2b2borganizationsorgidjoin-requests) |
| 19 | POST | `/api/v2/b2b/memberships/{id}/approve` | 👑 | [§4.19](#419-post-apiv2b2bmembershipsidapprove) |
| 20 | POST | `/api/v2/b2b/memberships/{id}/reject` | 👑 | [§4.20](#420-post-apiv2b2bmembershipsidreject) |
| 21 | POST | `/api/v2/b2b/memberships/{id}/suspend` | 👑 | [§4.21](#421-post-apiv2b2bmembershipsidsuspend) |
| 22 | POST | `/api/v2/b2b/memberships/{id}/leave` | 🔓 | [§4.22](#422-post-apiv2b2bmembershipsidleave) |
| 23 | POST | `/api/v2/b2b/organizations/{orgId}/staff` | 👑 | [§4.23](#423-post-apiv2b2borganizationsorgidstaff) |
| 24 | GET | `/api/v2/b2b/organizations/{orgId}/members` | 👥 | [§4.24](#424-get-apiv2b2borganizationsorgidmembers) |
| 25 | POST | `/api/v2/b2b/organizations/{orgId}/invite-codes` | 👑 | [§4.25](#425-post-apiv2b2borganizationsorgidinvite-codes) |
| 26 | GET | `/api/v2/b2b/organizations/{orgId}/invite-codes` | 🔒 | [§4.26](#426-get-apiv2b2borganizationsorgidinvite-codes) |
| 27 | POST | `/api/v2/b2b/invite-codes/{id}/revoke` | 👑 | [§4.27](#427-post-apiv2b2binvite-codesidrevoke) |
| 28 | POST | `/api/v2/b2b/invite-codes/redeem` | 🔓 | [§4.28](#428-post-apiv2b2binvite-codesredeem) |
| 29 | POST | `/api/v2/b2b/organizations/{orgId}/devices` | 👑 | [§4.29](#429-post-apiv2b2borganizationsorgiddevices) |
| 30 | GET | `/api/v2/b2b/organizations/{orgId}/devices` | 🔒 | [§4.30](#430-get-apiv2b2borganizationsorgiddevices) |
| 31 | GET | `/api/v2/b2b/organizations/{orgId}/devices/{deviceId}` | 🔒 | [§4.31](#431-get-apiv2b2borganizationsorgiddevicesdeviceid) |
| 32 | PATCH | `/api/v2/b2b/organizations/{orgId}/devices/{deviceId}` | 👑 | [§4.32](#432-patch-apiv2b2borganizationsorgiddevicesdeviceid) |
| 33 | POST | `/api/v2/b2b/organizations/{orgId}/devices/{deviceId}/deactivate` | 👑 | [§4.33](#433-post-apiv2b2borganizationsorgiddevicesdeviceiddeactivate) |
| 34 | GET | `…/members/{memberId}/measurements` | 👥 | [§4.34](#434-get-apiv2b2borganizationsorgidmembersmemberidmeasurements) |
| 35 | GET | `…/measurements/{recordId}` | 👥 | [§4.35](#435-get-apiv2b2borganizationsorgidmembersmemberidmeasurementsrecordid) |
| 36 | GET | `…/measurements/summary` | 👥 | [§4.36](#436-get-apiv2b2borganizationsorgidmembersmemberidmeasurementssummary) |
| 37 | POST | `/api/v2/b2b/feedbacks/memberships/{id}` | 👥 | [§4.37](#437-post-apiv2b2bfeedbacksmembershipsid) |
| 38 | PATCH | `/api/v2/b2b/feedbacks/{id}` | 작성자 | [§4.38](#438-patch-apiv2b2bfeedbacksid) |
| 39 | DELETE | `/api/v2/b2b/feedbacks/{id}` | 작성자 | [§4.39](#439-delete-apiv2b2bfeedbacksid) |
| 40 | GET | `/api/v2/b2b/feedbacks/memberships/{id}` | 🔒 | [§4.40](#440-get-apiv2b2bfeedbacksmembershipsid) |
| 41 | GET | `/api/v2/b2b/feedbacks/measurements/{recordId}` | 🔒 | [§4.41](#441-get-apiv2b2bfeedbacksmeasurementsrecordid) |
| 42 | GET | `/api/v2/b2b/license-summary` | 🔒 | [§4.42](#442-get-apiv2b2blicense-summary) |
| 43 | GET | `/api/v2/b2b/license-summary/{orgId}` | 🔒 | [§4.43](#443-get-apiv2b2blicense-summaryorgid) |
| 44 | POST | `/api/v2/b2b/billing/checkout-init` | 👑 | [§4.44](#444-post-apiv2b2bbillingcheckout-init) |

전 응답은 [`CncResponse`](#71-cncresponse-envelope) envelope 으로 감싸진다. 본 문서의 응답 샘플은 envelope 전체를 보여 준다.

---

## 4. Endpoint Detail

### 4.1 `POST /api/v2/b2b/auth/login`

로그인. `X-Client-Type` 헤더로 web 세션 / 모바일 토큰 분기.

**Auth** 🔓.

**Headers**
| 헤더 | 값 | 설명 |
|---|---|---|
| `X-Client-Type` | `web` 또는 미지정 | `web` → Spring Session Cookie 발급. 그 외 → mobile token 발급 (default). |

**Request body** — `LoginRequest`

```json
{
  "email": "owner@example.com",
  "password": "P@ssw0rd123",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

검증. `email` ValidEmail, `password` NotBlank. `deviceId`/`deviceType` optional (rotation 추적용).

**Response — 200 (mobile)** — `TokenPair`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "userId": "u-9f3e",
    "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
    "refreshToken": "rt_2f81a3..."
  }
}
```

**Response — 200 (web, `X-Client-Type: web`)**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "userId": "u-9f3e",
    "email": "owner@example.com",
    "sessionId": "F3D2..."
  }
}
```

추가로 `Set-Cookie: SESSION=...; HttpOnly` 가 발급된다.

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-003` | 비밀번호 불일치 / 미가입 |
| 403 | `AUTH-403-003` | 계정 SUSPENDED / DELETION_PENDING |

---

### 4.2 `POST /api/v2/b2b/auth/token`

refresh token rotation. 모바일에서 access token 만료 시 호출. family rotation — 같은 refresh 재사용 시 family 전체 폐기.

**Auth** 🔓 (refresh token 자체로 인증).

**Request body** — `RefreshRequest`

```json
{
  "refreshToken": "rt_2f81a3...",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**Response — 200** — `TokenPair` (새 access + rotated refresh)

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "userId": "u-9f3e",
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.NEW...",
    "refreshToken": "rt_NEW_a8b9c..."
  }
}
```

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-001` | refresh token 무효 / 만료 / 이미 rotated (family compromised) |

---

### 4.3 `POST /api/v2/b2b/auth/logout`

JTI revocation + (web 흐름) Session invalidate.

**Auth** Authorization 헤더 또는 Cookie 중 어느 한쪽.

**Headers**
| 헤더 | 값 |
|---|---|
| `Authorization` | `Bearer eyJ...` (모바일) — access 의 jti 를 revoke list 에 추가 |

**Request body** — `LogoutRequest` (optional — refresh family 까지 폐기하려면 전달)

```json
{ "refreshToken": "rt_2f81a3..." }
```

**Response — 200**

```json
{ "success": true, "code": "200", "message": "OK", "data": null }
```

---

### 4.4 `GET /api/v2/b2b/auth/me`

세션 sanity check. 인증 컨텍스트 노출.

**Auth** 🔒.

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "authenticated": true,
    "userId": "u-9f3e",
    "email": "owner@example.com",
    "roles": ["ROLE_USER"]
  }
}
```

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `CMN-401-001` | anonymous principal |

---

### 4.5 `POST /api/v2/b2b/auth/oauth2/google`

Google ID 토큰 검증 후 가입/로그인 후 JWT 발급.

**Auth** 🔓. 단, `b2b.oauth2.enabled=true` 설정 시에만 라우트 등록 (false 시 404).

**Request body** — `OAuth2LoginRequest`

```json
{
  "idToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEy...",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**Response — 200** — `TokenPair` (4.1 모바일 응답과 동일 shape).

**Errors** (`OAuth2Error`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-001` | InvalidIdToken — 서명/만료/issuer 불일치 |
| 401 | `AUTH-401-001` | InvalidAudience — aud 가 b2b clientId 와 다름 |
| 400 | `CMN-400-001` | EmailMissing — Google 가 이메일 미제공 |
| 403 | `AUTH-403-003` | AccountSuspended — DELETION_PENDING 등 |
| 502 | `CMN-502-001` | VerifierUnavailable — Google JWKS 다운 |

---

### 4.6 `POST /api/v2/b2b/auth/oauth2/apple`

Apple ID 토큰 검증 후 가입/로그인 후 JWT 발급. 4.5 와 동일 shape, 검증 verifier 만 Apple JWKS.

---

### 4.7 `POST /api/v2/b2b/users`

관리자 회원가입 (이메일 + 비밀번호 + 이름).

**Auth** 🔓.

**Request body** — `SignUpRequest`

```json
{
  "email": "owner@example.com",
  "password": "P@ssw0rd123",
  "firstName": "박",
  "lastName": "관장",
  "country": "KR",
  "phoneNumber": "+821011112222"
}
```

검증. ValidEmail / ValidPassword (영문/숫자/특수 1자 이상, 8자 이상) / ValidName (한글 또는 영문) / ValidCountryCode (ISO-3166 alpha-2 지원 셋) / ValidPhoneNumber (E.164).

**Response — 201** — `UserResponse`

```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "id": "u-9f3e",
    "email": "owner@example.com",
    "firstName": "박",
    "lastName": "관장",
    "phoneNumber": "+821011112222",
    "country": "KR",
    "profileImageUrl": null,
    "accountStatus": "ACTIVE",
    "lastLoginTime": null,
    "createdAt": "2026-05-13T08:00:00Z",
    "organizations": []
  }
}
```

가입 직후 `organizations: []`. 시설 생성/가입 시 채워짐.

**Errors** (`UserError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 409 | `CMN-409-001` | EmailDuplicate |
| 400 | `CMN-400-001` | InvalidPasswordFormat |

---

### 4.8 `GET /api/v2/b2b/users/{b2bUserId}`

본인 프로필 + 소속 organization (multi-org, license 포함).

**Auth** 🧍 — path 의 `b2bUserId` 가 인증된 principal 과 일치해야 함. 불일치 시 403.

**Response — 200** — `UserResponse` (with `organizations[]`)

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "id": "u-9f3e",
    "email": "owner@example.com",
    "firstName": "박",
    "lastName": "관장",
    "phoneNumber": "+821011112222",
    "country": "KR",
    "profileImageUrl": "https://cdn.carenco.com/u/9f3e.jpg",
    "accountStatus": "ACTIVE",
    "lastLoginTime": "2026-05-13T07:30:00Z",
    "createdAt": "2026-01-02T00:00:00Z",
    "organizations": [
      {
        "id": "org-A",
        "name": "강남점",
        "type": "PILATES",
        "role": "OWNER",
        "seatsUsed": 12,
        "planSeats": 50,
        "licenseState": "ACTIVE"
      },
      {
        "id": "org-B",
        "name": "분당점",
        "type": "PILATES",
        "role": "ADMIN",
        "seatsUsed": 7,
        "planSeats": 30,
        "licenseState": "GRACE"
      }
    ]
  }
}
```

**Errors** (`UserError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotSelf — path != principal |
| 404 | `USR-404-001` | NotFound |
| 403 | `AUTH-403-003` | AccountInactive — DELETION_PENDING |

**연쇄 호출**

```
UserController.get → ensureSelf(pathUserId, principal)
            → UserService.getMeWithOrganizations
  ├─ B2bUserRepository.findById
  ├─ OrganizationRepository.findByOwnerUserId (OWNER orgs)
  ├─ OrganizationMembershipRepository.findByMemberTypeAndMemberId(B2B, userId)
  └─ LicenseClient.batchGetLicense (한 번에 조회)
       ├─ Redis cache hits → 즉시
       └─ miss → payment-service gRPC BatchGetLicenseStatus
```

---

### 4.9 `PATCH /api/v2/b2b/users/{b2bUserId}`

프로필 부분 수정. JsonNullable 의 3-state semantics 사용.

**Auth** 🧍.

**Request body** — `UpdateProfileRequest`

| field | 동작 |
|---|---|
| undefined (키 없음) | 변경 없음 |
| `"field": null` | 컬럼 NULL 로 설정 (firstName/lastName/searchable 류는 거부) |
| `"field": value` | 적용 |

```json
{
  "phoneNumber": "+821033334444",
  "profileImageUrl": "https://cdn.carenco.com/u/9f3e-new.jpg",
  "country": "JP"
}
```

(firstName/lastName 은 ValidName + JsonNullableNonNull — null 로 설정 불가)

**Response — 200** — `UserResponse` (변경 후 전체 상태).

**Errors** (`UserError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotSelf |
| 404 | `USR-404-001` | NotFound |

---

### 4.10 `POST /api/v2/b2b/users/{b2bUserId}/password`

비밀번호 변경. 변경 성공 시 모든 session/token 폐기.

**Auth** 🧍.

**Request body** — `ChangePasswordRequest`

```json
{
  "currentPassword": "P@ssw0rd123",
  "newPassword": "NewP@ssw0rd456"
}
```

**Response — 204** (body 없음, envelope 없음).

**Errors** (`UserError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-003` | InvalidCurrentPassword |
| 400 | `CMN-400-001` | InvalidPasswordFormat |
| 400 | `CMN-400-001` | SamePasswordReuse |
| 403 | `AUTH-403-002` | NotSelf |

---

### 4.11 `DELETE /api/v2/b2b/users/{b2bUserId}`

soft 탈퇴 — `accountStatus=DELETION_PENDING` + 모든 세션 폐기. archive outbox 가 비동기로 Carenco Archive Collector 에 publish.

**Auth** 🧍.

**Response — 204**.

**Errors**. 4.9 와 동일 (NotSelf / NotFound).

---

### 4.12 `POST /api/v2/b2b/organizations`

시설 등록. 호출한 b2b_user 가 OWNER 로 자동 멤버십 생성 (`joinedVia=OWNER_INIT`).

**Auth** 🔒.

**Request body** — `CreateOrganizationRequest`

```json
{
  "name": "강남점",
  "type": "PILATES",
  "description": "강남구 신논현역 5분 거리",
  "phoneNumber": "+82215551234",
  "photoUrl": "https://cdn.carenco.com/org/gangnam.jpg",
  "country": "KR",
  "postalCode": "06000",
  "regionLevel1": "서울",
  "regionLevel2": "강남구",
  "addressLine1": "테헤란로 123",
  "addressLine2": "456호",
  "searchable": true,
  "inviteCodeEnabled": true,
  "approvalRequired": true
}
```

`OrganizationType` enum. `GYM | PILATES | YOGA | PT_STUDIO | CROSSFIT | FUNCTIONAL | BOXING | ETC`.

**Response — 201** — `OrganizationResponse`

```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "id": "org-A",
    "name": "강남점",
    "type": "PILATES",
    "description": "강남구 신논현역 5분 거리",
    "phoneNumber": "+82215551234",
    "photoUrl": "https://cdn.carenco.com/org/gangnam.jpg",
    "address": {
      "country": "KR",
      "postalCode": "06000",
      "regionLevel1": "서울",
      "regionLevel2": "강남구",
      "addressLine1": "테헤란로 123",
      "addressLine2": "456호"
    },
    "ownerUserId": "u-9f3e",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 0,
    "status": "ACTIVE"
  }
}
```

**Errors** (`OrganizationError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 400 | `CMN-400-001` | InvalidInput |

---

### 4.13 `GET /api/v2/b2b/organizations/{id}`

단건 조회 (공개).

**Auth** 🔓.

**Response — 200** — `OrganizationResponse` (4.12 와 동일 shape).

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |

---

### 4.14 `PATCH /api/v2/b2b/organizations/{id}`

부분 수정 (owner 만). JsonNullable 의 3-state semantics.

**Auth** 👑 (OWNER).

**Request body** — `UpdateOrganizationRequest`

```json
{
  "name": "강남점 리뉴얼",
  "description": null,
  "searchable": false,
  "inviteCodeEnabled": true,
  "approvalRequired": false
}
```

NotNullable. `name`, `type`, `searchable`, `inviteCodeEnabled`, `approvalRequired`.

**Response — 200** — `OrganizationResponse`.

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |
| 403 | `AUTH-403-002` | NotOwner |

---

### 4.15 `DELETE /api/v2/b2b/organizations/{id}`

soft 삭제 (owner 만). `status=DELETED` + 모든 멤버십 LEFT + 모든 invite code REVOKED cascade. archive outbox 비동기 publish.

**Auth** 👑 (OWNER).

**Response — 204**.

**Errors** (`OrganizationError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |
| 403 | `AUTH-403-002` | NotOwner |

---

### 4.16 `GET /api/v2/b2b/organizations/search`

`searchable=true` + `status=ACTIVE` 시설 검색.

**Auth** 🔓.

**Query**
| 이름 | 타입 | 기본 |
|---|---|---|
| `keyword` | string | 이름/주소 LIKE 매칭 |
| `type` | `OrganizationType` | optional |
| `page` | int | 0 |
| `size` | int | 20 |

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      {
        "id": "org-A",
        "name": "강남점",
        "type": "PILATES",
        "description": "...",
        "phoneNumber": "+82215551234",
        "photoUrl": "...",
        "address": { "country": "KR", "postalCode": "06000",
          "regionLevel1": "서울", "regionLevel2": "강남구",
          "addressLine1": "...", "addressLine2": "..." },
        "ownerUserId": "u-9f3e",
        "searchable": true, "inviteCodeEnabled": true, "approvalRequired": true,
        "seatsUsed": 12, "status": "ACTIVE"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "totalPages": 1,
    "hasNext": false
  }
}
```

---

### 4.17 `GET /api/v2/b2b/organizations/mine`

본인 소유 시설 목록.

**Auth** 🔒.

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      { "id": "org-A", "name": "강남점", "type": "PILATES", "... (OrganizationResponse 동일)": "" }
    ]
  }
}
```

---

### 4.18 `POST /api/v2/b2b/organizations/{orgId}/join-requests`

Carenco user 가 가입 신청. `searchable=true` 시설만 가능.

**Auth** 🔓 (body 의 `memberId` 가 carenco user uuid).

**Request body** — `MembershipController.JoinRequest`

```json
{
  "memberId": "carenco-user-uuid",
  "note": "5월 한 달 등록 희망합니다"
}
```

**Response — 201** — `MembershipResponse` (status=PENDING)

```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "id": "m-7c1d",
    "organizationId": "org-A",
    "memberType": "CARENCO",
    "memberId": "carenco-user-uuid",
    "role": "MEMBER",
    "status": "PENDING",
    "memberNumber": null,
    "joinedVia": "SEARCH_REQUEST",
    "inviteCodeId": null,
    "approvedBy": null,
    "approvedAt": null,
    "joinedAt": null,
    "leftAt": null,
    "note": "5월 한 달 등록 희망합니다"
  }
}
```

**Errors** (`MembershipFlowError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-001` | OrganizationNotSearchable |
| 409 | `CMN-409-001` | AlreadyMember |

---

### 4.19 `POST /api/v2/b2b/memberships/{membershipId}/approve`

PENDING → ACTIVE. 좌석 +1 + license guard.

**Auth** 👑.

**Response — 200** — `MembershipResponse`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "id": "m-7c1d",
    "organizationId": "org-A",
    "memberType": "CARENCO",
    "memberId": "carenco-user-uuid",
    "role": "MEMBER",
    "status": "ACTIVE",
    "memberNumber": "M-001",
    "joinedVia": "SEARCH_REQUEST",
    "approvedBy": "u-9f3e",
    "approvedAt": "2026-05-13T09:10:00Z",
    "joinedAt": "2026-05-13T09:10:00Z",
    "leftAt": null,
    "note": null,
    "inviteCodeId": null
  }
}
```

**Errors** (`MembershipFlowError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 409 | `CMN-409-001` | InvalidStateTransition |
| 409 | `CMN-409-001` | AlreadyMember |
| 403 | `LIC-403-001` | SeatConsumeFailed (NO_SEATS_REMAINING — license SUSPENDED / 만석) |

**연쇄 호출**

```
MembershipController.approve → MembershipFlowService.approve
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

### 4.20 `POST /api/v2/b2b/memberships/{membershipId}/reject`

PENDING → LEFT (좌석 차감 없음).

**Auth** 👑.

**Response — 200** — `MembershipResponse` (status=LEFT, leftAt 채워짐).

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 409 | `CMN-409-001` | InvalidStateTransition |

---

### 4.21 `POST /api/v2/b2b/memberships/{membershipId}/suspend`

ACTIVE → SUSPENDED.

**Auth** 👑.

**Request body** — `MembershipController.SuspendRequest` (optional)

```json
{ "reason": "회비 연체 3개월" }
```

**Response — 200** — `MembershipResponse` (status=SUSPENDED).

**Errors**. 4.20 와 동일.

---

### 4.22 `POST /api/v2/b2b/memberships/{membershipId}/leave`

본인 탈퇴. body 의 `requesterId` 와 멤버십의 `memberId` 가 일치해야 함. 좌석 −1 + archive outbox publish.

**Auth** 🔓 (body 자기검증).

**Request body** — `MembershipController.LeaveRequest`

```json
{ "requesterId": "carenco-user-uuid" }
```

**Response — 200** — `MembershipResponse` (status=LEFT).

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 409 | `CMN-409-001` | InvalidStateTransition |

---

### 4.23 `POST /api/v2/b2b/organizations/{orgId}/staff`

OWNER/ADMIN 이 기존 b2b_user 를 staff 로 지정. role 은 ADMIN 또는 TRAINER.

**Auth** 👑.

**Request body** — `MembershipController.AppointStaffRequest`

```json
{
  "b2bUserId": "u-staff-1",
  "role": "TRAINER",
  "memberNumber": "T-001"
}
```

**Response — 201** — `MembershipResponse`

```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "id": "m-staff-1",
    "organizationId": "org-A",
    "memberType": "B2B",
    "memberId": "u-staff-1",
    "role": "TRAINER",
    "status": "ACTIVE",
    "memberNumber": "T-001",
    "joinedVia": "DIRECT_INVITE",
    "inviteCodeId": null,
    "approvedBy": "u-9f3e",
    "approvedAt": "2026-05-13T09:30:00Z",
    "joinedAt": "2026-05-13T09:30:00Z",
    "leftAt": null,
    "note": null
  }
}
```

**Errors** (`MembershipFlowError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 400 | `CMN-400-001` | InvalidRoleAssignment (OWNER 부여 시도 등) |
| 409 | `CMN-409-001` | AlreadyMember |

---

### 4.24 `GET /api/v2/b2b/organizations/{orgId}/members`

회원 목록 (membership + user-service 정보 + measurement 요약). PDF 흐름 4.

**Auth** 👥. user/measure-service 장애 시 user/measurement 필드는 null — membership 정보만 노출 (graceful degradation).

**Query**
| 이름 | 타입 | 기본 |
|---|---|---|
| `query` | string | 이름/전화번호 부분일치 |
| `sort_by` | `NAME` / `REGISTERED_AT` / `LATEST_MEASUREMENT` | `REGISTERED_AT` |
| `status` | `MembershipStatus` | `ACTIVE` |
| `page` | int | 0 |
| `size` | int | 20 (max 100) |

**Response — 200** — `MemberListResponse`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      {
        "membershipId": "m-7c1d",
        "memberId": "carenco-user-uuid",
        "memberNumber": "M-001",
        "role": "MEMBER",
        "status": "ACTIVE",
        "joinedAt": "2026-05-13T09:10:00Z",

        "name": "김회원",
        "nickname": "kimkim",
        "email": "kim@example.com",
        "phoneNumber": "+821022223333",
        "photoUrl": "https://cdn.carenco.com/u/kim.jpg",
        "birthdate": "1995-03-15",
        "gender": "FEMALE",
        "countryCode": "KR",
        "height": 165.0,

        "totalMeasurements": 12,
        "currentMonthMeasurements": 3,
        "lastMeasuredAt": "2026-05-10T08:00:00Z",
        "lastWeight": 54.2,
        "lastBodyScore": 78.5
      }
    ],
    "totalCount": 47,
    "totalReports": 312,
    "hasNext": true
  }
}
```

**Errors** (`MemberListError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | RoleRequired |
| 404 | `CMN-404-001` | OrganizationNotFound |

---

### 4.25 `POST /api/v2/b2b/organizations/{orgId}/invite-codes`

초대 코드 발급. 헷갈리는 문자 (0/O, 1/l/I) 회피 generator. TTL 48h.

**Auth** 👑.

**Request body** — `IssueInviteCodeRequest`

```json
{
  "description": "5월 신규 회원용",
  "codeLength": 8
}
```

`codeLength` 6~12 (기본 8).

**Response — 201** — `InviteCodeResponse`

```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "id": "ic-a1b2",
    "organizationId": "org-A",
    "code": "Hk7mPqR3",
    "description": "5월 신규 회원용",
    "expiresAt": "2026-05-15T09:30:00Z",
    "usedAt": null,
    "status": "ACTIVE",
    "createdBy": "u-9f3e"
  }
}
```

**Errors** (`InviteCodeError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 403 | `AUTH-403-001` | InviteCodeDisabled (organization.inviteCodeEnabled=false) |
| 403 | `LIC-403-001` | LicenseBlocked (EXPIRED/SUSPENDED/NONE) |

---

### 4.26 `GET /api/v2/b2b/organizations/{orgId}/invite-codes`

ACTIVE 코드 목록.

**Auth** 🔒.

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      { "id": "ic-a1b2", "organizationId": "org-A", "code": "Hk7mPqR3",
        "description": "...", "expiresAt": "2026-05-15T09:30:00Z",
        "usedAt": null, "status": "ACTIVE", "createdBy": "u-9f3e" }
    ]
  }
}
```

---

### 4.27 `POST /api/v2/b2b/invite-codes/{inviteCodeId}/revoke`

수동 폐기 (ACTIVE → REVOKED).

**Auth** 👑.

**Response — 200** — `InviteCodeResponse` (status=REVOKED).

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | InviteCodeNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |

---

### 4.28 `POST /api/v2/b2b/invite-codes/redeem`

Carenco user 가 코드 입력하여 가입.

**Auth** 🔓. body 의 `memberId` 가 carenco userId.

**Request body** — `InviteCodeController.RedeemRequest`

```json
{
  "code": "Hk7mPqR3",
  "memberId": "carenco-user-uuid"
}
```

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "inviteCode": {
      "id": "ic-a1b2", "organizationId": "org-A", "code": "Hk7mPqR3",
      "description": "5월 신규 회원용",
      "expiresAt": "2026-05-15T09:30:00Z",
      "usedAt": "2026-05-13T09:50:00Z",
      "status": "USED",
      "createdBy": "u-9f3e"
    },
    "membershipId": "m-9d4e",
    "organizationId": "org-A",
    "status": "ACTIVE"
  }
}
```

**Errors** (`InviteCodeError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | InviteCodeNotFound |
| 410 | `INV-410-001` | InviteCodeExpired |
| 409 | `INV-409-001` | InviteCodeNotActive (USED / REVOKED) |
| 409 | `CMN-409-002` | AlreadyMember |
| 403 | `LIC-403-001` | LicenseBlocked (SeatsExhausted 포함) |

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

### 4.29 `POST /api/v2/b2b/organizations/{orgId}/devices`

InBody 등 측정 device 등록. device-service gRPC 위임.

**Auth** 👑. license state ACTIVE/GRACE 만.

**Request body** — `RegisterDeviceRequest`

```json
{
  "serialNumber": "INBODY-770-2024-0123",
  "alias": "1번 InBody"
}
```

**Response — 201** — `DeviceResponse`

```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "deviceId": "dev-c3d4",
    "organizationId": "org-A",
    "serialNumber": "INBODY-770-2024-0123",
    "alias": "1번 InBody",
    "status": "ACTIVE",
    "registeredAt": "2026-05-13T10:00:00Z",
    "registeredBy": "u-9f3e",
    "deactivatedAt": null,
    "deactivatedBy": null,
    "deactivationReason": null,
    "lastUsedAt": null
  }
}
```

**Errors** (`DeviceError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | OrgRoleRequired |
| 403 | `LIC-403-001` | LicenseDenied (EXPIRED/SUSPENDED) |

---

### 4.30 `GET /api/v2/b2b/organizations/{orgId}/devices`

device 목록.

**Auth** 🔒 (멤버).

**Query**
| 이름 | 타입 | 설명 |
|---|---|---|
| `keyword` | string | serialNumber / alias 부분일치 |
| `status` | `DeviceStatus` (`ACTIVE` / `INACTIVE`) | 필터 |
| `sortBy` | string | device-service 위임 (REGISTERED_AT DESC / ALIAS ASC 등) |

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      {
        "deviceId": "dev-c3d4",
        "organizationId": "org-A",
        "serialNumber": "INBODY-770-2024-0123",
        "alias": "1번 InBody",
        "status": "ACTIVE",
        "registeredAt": "2026-05-13T10:00:00Z",
        "registeredBy": "u-9f3e",
        "deactivatedAt": null,
        "deactivatedBy": null,
        "deactivationReason": null,
        "lastUsedAt": "2026-05-13T11:20:00Z"
      }
    ]
  }
}
```

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotMember |

---

### 4.31 `GET /api/v2/b2b/organizations/{orgId}/devices/{deviceId}`

device 단건.

**Auth** 🔒 (멤버).

**Response — 200** — `DeviceResponse` (4.29 와 동일).

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | DeviceNotInOrg |

---

### 4.32 `PATCH /api/v2/b2b/organizations/{orgId}/devices/{deviceId}`

alias 만 변경 가능. device-service gRPC 가 `alias=null` 을 "변경 없음" 으로 해석 → `null` 거부 (JsonNullableNonNull).

**Auth** 👑.

**Request body** — `UpdateDeviceRequest`

```json
{ "alias": "1번 InBody (대기실)" }
```

**Response — 200** — `DeviceResponse`.

---

### 4.33 `POST /api/v2/b2b/organizations/{orgId}/devices/{deviceId}/deactivate`

device 비활성화 (status=INACTIVE).

**Auth** 👑.

**Request body** — `DeactivateDeviceRequest` (optional)

```json
{ "reason": "수리 입고" }
```

**Response — 200** — `DeviceResponse` (status=INACTIVE, deactivatedAt/deactivatedBy 채워짐).

---

### 4.34 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements`

회원 측정 이력 — measure-service gRPC.

**Auth** 👥. 호출자 OWNER/ADMIN/TRAINER 이고 `memberId` 가 organization 의 ACTIVE 멤버.

**Query**
| 이름 | 타입 | 설명 |
|---|---|---|
| `from` | string (ISO date) | 측정일 ≥ from |
| `to` | string (ISO date) | 측정일 ≤ to |
| `page` | int | 0 |
| `size` | int | 20 |

**Response — 200** — `MeasurementClient.Page`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      {
        "recordId": "r-9e8d",
        "userId": "carenco-user-uuid",
        "measuredAt": "2026-05-10T08:00:00Z",
        "weight": 54.2,
        "height": 165.0,
        "age": 29,
        "bodyScore": 78.5,
        "predictedAge": 27.0,
        "deviceId": "dev-c3d4"
      }
    ],
    "totalCount": 12,
    "hasNext": false
  }
}
```

**Errors** (`MeasurementError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | RoleRequired |
| 404 | `CMN-404-001` | TargetNotActiveMember |

---

### 4.35 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/{recordId}`

단건 측정.

**Auth** 👥.

**Response — 200** — `MeasurementInfo` (4.34 의 한 item 과 동일).

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | RecordNotFound |

---

### 4.36 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/summary`

denormalized 요약 (measure-service `user_measurement_summary`).

**Auth** 👥.

**Response — 200** — `MeasurementSummary`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "userId": "carenco-user-uuid",
    "totalRecords": 12,
    "currentMonthRecords": 3,
    "lastMeasuredAt": "2026-05-10T08:00:00Z",
    "lastBodyScore": 78.5,
    "lastWeight": 54.2,
    "lastPredictedAge": 27.0
  }
}
```

이력 없으면 `lastMeasuredAt` 외 모두 null/0.

---

### 4.37 `POST /api/v2/b2b/feedbacks/memberships/{membershipId}`

피드백 작성 — TRAINER/ADMIN/OWNER 가 회원에 코멘트.

**Auth** 👥.

**Request body** — `CreateFeedbackRequest`

```json
{
  "body": "어깨 가동범위가 좋아졌습니다. 다음 주는 하체 위주로.",
  "visibility": "MEMBER",
  "measurementRecordId": "r-9e8d"
}
```

`visibility`. `MEMBER` (회원 앱 노출) / `INTERNAL` (스태프만). `measurementRecordId` 선택 — 특정 측정과 연관 시.

**Response — 201** — `FeedbackResponse`

```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "id": "f-1234",
    "organizationId": "org-A",
    "memberMembershipId": "m-7c1d",
    "authorUserId": "u-staff-1",
    "measurementRecordId": "r-9e8d",
    "body": "어깨 가동범위가 좋아졌습니다. 다음 주는 하체 위주로.",
    "visibility": "MEMBER",
    "createdAt": "2026-05-13T12:00:00Z",
    "editedAt": null
  }
}
```

**Errors** (`FeedbackError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotStaff |
| 400 | `CMN-400-001` | InvalidInput |

---

### 4.38 `PATCH /api/v2/b2b/feedbacks/{feedbackId}`

본문 수정 (작성자 본인만).

**Auth** 작성자.

**Request body** — `UpdateFeedbackRequest`

```json
{ "body": "오타 수정: 다음 주는 하체+코어 위주." }
```

**Response — 200** — `FeedbackResponse` (editedAt 채워짐).

**Errors**
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | FeedbackNotFound |
| 403 | `AUTH-403-002` | NotAuthor |

---

### 4.39 `DELETE /api/v2/b2b/feedbacks/{feedbackId}`

soft 삭제 (deletedAt 채움).

**Auth** 작성자.

**Response — 200** — `FeedbackResponse` (deletedAt 채워진 상태로 echo).

---

### 4.40 `GET /api/v2/b2b/feedbacks/memberships/{membershipId}`

회원 피드백 목록.

**Auth** 🔒.

**Query**. `page` (0), `size` (20).

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      { "id": "f-1234", "organizationId": "org-A",
        "memberMembershipId": "m-7c1d", "authorUserId": "u-staff-1",
        "measurementRecordId": "r-9e8d",
        "body": "...", "visibility": "MEMBER",
        "createdAt": "2026-05-13T12:00:00Z", "editedAt": null }
    ],
    "page": 0,
    "totalElements": 1,
    "hasNext": false
  }
}
```

---

### 4.41 `GET /api/v2/b2b/feedbacks/measurements/{recordId}`

특정 측정에 달린 피드백 목록 (회원이 본인 측정 결과를 볼 때).

**Auth** 🔒.

**Query / Response**. 4.40 와 동일 shape.

---

### 4.42 `GET /api/v2/b2b/license-summary`

본인 가입/소유 organization 들의 license + seat 요약 (한 번에).

**Auth** 🔒.

**Response — 200**

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      {
        "organizationId": "org-A",
        "organizationName": "강남점",
        "state": "ACTIVE",
        "planCode": "PILATES_BASIC",
        "planSeats": 50,
        "seatsUsed": 12,
        "seatsRemaining": 38,
        "startedAt": "2026-01-01T00:00:00Z",
        "expiresAt": "2027-01-01T00:00:00Z",
        "graceRemainingDays": 0,
        "subscriptionId": "sub_xxx"
      },
      {
        "organizationId": "org-B",
        "organizationName": "분당점",
        "state": "GRACE",
        "planCode": "PILATES_BASIC",
        "planSeats": 30,
        "seatsUsed": 7,
        "seatsRemaining": 23,
        "startedAt": "2025-12-01T00:00:00Z",
        "expiresAt": "2026-05-10T00:00:00Z",
        "graceRemainingDays": 4,
        "subscriptionId": "sub_yyy"
      }
    ]
  }
}
```

---

### 4.43 `GET /api/v2/b2b/license-summary/{organizationId}`

특정 organization 의 license 단건.

**Auth** 🔒 (해당 시설 멤버만).

**Response — 200** — `LicenseSummaryResponse` (4.42 의 item 과 동일).

**Errors** (`LicenseSummaryError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-002` | NotMember |

---

### 4.44 `POST /api/v2/b2b/billing/checkout-init`

결제 시작 진입점. b2b 가 권한 검증 후 payment-service 위임.

**Auth** 👑 (OWNER).

**Request body** — `CheckoutInitRequest`

```json
{
  "organizationId": "org-A",
  "priceId": "pri_01h5xxx"
}
```

**Response — 200** — `PaymentCheckoutLinkResponse` echo

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [{ "priceId": "pri_01h5xxx", "quantity": 1 }],
    "customer": { "email": "owner@example.com", "name": "박관장" },
    "customData": { "organization_id": "org-A" },
    "plan": {
      "planCode": "PILATES_BASIC",
      "planSeats": 50,
      "displayName": "필라테스 기본 (50석)",
      "amount": 99000,
      "currency": "KRW"
    },
    "paddleCustomerId": "ctm_xxx"
  }
}
```

**Errors** (`BillingError`)
| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 404 | `CMN-404-001` | PlanNotFound |
| 403 | `AUTH-403-002` | NotOwner |
| 403 | `AUTH-403-001` | OrganizationNotActive |
| 403 | `AUTH-403-001` | LicenseSuspended |
| 409 | `CMN-409-001` | AlreadyHasActiveSubscription |
| 502 | `CMN-502-001` | PaymentServiceUnavailable |

**연쇄 호출**

```
BillingController.checkoutInit → BillingService.initCheckout
  ├─ OrganizationRepository.findById
  ├─ LicenseClient.getLicense (Redis 캐시 우선)
  ├─ B2bUserRepository.findById (owner 본인)
  └─ PaymentClient.createCheckoutLink (HTTP → payment-service)
```

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
        L4[UserClient<br/>→ user.UserBasic] --> UB[user :9090]
        L5[PaymentClient.createCheckoutLink<br/>→ payment REST] --> CHK[payment :8080]
        L6[ArchiveCollectorClient<br/>→ archive.Collect] --> AR[archive :9090]
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
| b2b → user | UserClient | gRPC UserBasic.BatchGetUserBasic | 회원목록 enrich | fail-open (user 필드 null) |
| b2b → archive | ArchiveCollectorClient | gRPC Collect | account/membership LEFT 시 outbox 배달 | retry + dlq |
| payment → b2b | B2bLicenseInvalidateClient | gRPC LicenseInvalidate | webhook + 자정 batch | fail-open (b2b 가 5min TTL 자연 만료) |

---

## 7. Schemas

### 7.1 CncResponse envelope

```json
{
  "success": boolean,
  "code": "string (예: 200, CMN-404-001)",
  "message": "string (i18n resolved)",
  "data": <any>,
  "error": "string? (failure 시 description)",
  "token": "string? (TokenPair 등 일부 응답)"
}
```

성공 시 `success=true, code="200"|"201"|"204", data=<payload>`. 실패 시 `success=false, code=<ErrorCode.getCode()>, message=<i18n>, error=<description>`.

### 7.2 TokenPair

```json
{ "userId": "u-...", "accessToken": "...", "refreshToken": "..." }
```

### 7.3 UserResponse

```json
{
  "id": "u-...",
  "email": "...",
  "firstName": "...",
  "lastName": "...",
  "phoneNumber": "...",
  "country": "KR",
  "profileImageUrl": "...?",
  "accountStatus": "ACTIVE|SUSPENDED|BANNED|DELETION_PENDING|DELETED",
  "lastLoginTime": "...?",
  "createdAt": "...",
  "organizations": [
    { "id": "...", "name": "...", "type": "PILATES",
      "role": "OWNER|ADMIN|TRAINER|MEMBER",
      "seatsUsed": 12, "planSeats": 50,
      "licenseState": "ACTIVE|GRACE|EXPIRED|SUSPENDED|NONE" }
  ]
}
```

### 7.4 OrganizationResponse

```json
{
  "id": "...", "name": "...",
  "type": "GYM|PILATES|YOGA|PT_STUDIO|CROSSFIT|FUNCTIONAL|BOXING|ETC",
  "description": "...?",
  "phoneNumber": "...?",
  "photoUrl": "...?",
  "address": {
    "country": "KR", "postalCode": "...",
    "regionLevel1": "...", "regionLevel2": "...",
    "addressLine1": "...", "addressLine2": "..."
  },
  "ownerUserId": "...",
  "searchable": true,
  "inviteCodeEnabled": true,
  "approvalRequired": true,
  "seatsUsed": 12,
  "status": "ACTIVE|SUSPENDED|DELETED"
}
```

### 7.5 MembershipResponse

```json
{
  "id": "...",
  "organizationId": "...",
  "memberType": "B2B|CARENCO",
  "memberId": "...",
  "role": "OWNER|ADMIN|TRAINER|MEMBER",
  "status": "PENDING|ACTIVE|SUSPENDED|LEFT",
  "memberNumber": "...?",
  "joinedVia": "OWNER_INIT|SEARCH_REQUEST|INVITE_CODE|DIRECT_INVITE",
  "inviteCodeId": "...?",
  "approvedBy": "...?",
  "approvedAt": "...?",
  "joinedAt": "...?",
  "leftAt": "...?",
  "note": "...?"
}
```

### 7.6 InviteCodeResponse

```json
{
  "id": "...",
  "organizationId": "...",
  "code": "Hk7mPqR3",
  "description": "...?",
  "expiresAt": "...",
  "usedAt": "...?",
  "status": "ACTIVE|USED|EXPIRED|REVOKED",
  "createdBy": "..."
}
```

### 7.7 MemberListItemResponse

```json
{
  "membershipId": "...", "memberId": "...", "memberNumber": "...?",
  "role": "MEMBER", "status": "ACTIVE", "joinedAt": "...",

  "name": "...?", "nickname": "...?", "email": "...?",
  "phoneNumber": "...?", "photoUrl": "...?",
  "birthdate": "1990-01-01?", "gender": "FEMALE|MALE|OTHER?",
  "countryCode": "KR?", "height": 165.0,

  "totalMeasurements": 12, "currentMonthMeasurements": 3,
  "lastMeasuredAt": "...?", "lastWeight": 54.2, "lastBodyScore": 78.5
}
```

user-service / measure-service 다운 시 user/measurement 필드 모두 null.

### 7.8 DeviceResponse

```json
{
  "deviceId": "...", "organizationId": "...",
  "serialNumber": "...", "alias": "...?",
  "status": "ACTIVE|INACTIVE",
  "registeredAt": "...", "registeredBy": "...",
  "deactivatedAt": "...?", "deactivatedBy": "...?", "deactivationReason": "...?",
  "lastUsedAt": "...?"
}
```

### 7.9 MeasurementInfo

```json
{
  "recordId": "...", "userId": "...",
  "measuredAt": "...",
  "weight": 54.2, "height": 165.0, "age": 29,
  "bodyScore": 78.5, "predictedAge": 27.0,
  "deviceId": "...?"
}
```

### 7.10 MeasurementSummary

```json
{
  "userId": "...",
  "totalRecords": 12,
  "currentMonthRecords": 3,
  "lastMeasuredAt": "...?",
  "lastBodyScore": 78.5,
  "lastWeight": 54.2,
  "lastPredictedAge": 27.0
}
```

### 7.11 FeedbackResponse

```json
{
  "id": "...", "organizationId": "...",
  "memberMembershipId": "...", "authorUserId": "...",
  "measurementRecordId": "...?",
  "body": "...",
  "visibility": "MEMBER|INTERNAL",
  "createdAt": "...", "editedAt": "...?"
}
```

### 7.12 LicenseSummaryResponse

```json
{
  "organizationId": "...",
  "organizationName": "...",
  "state": "ACTIVE|GRACE|EXPIRED|SUSPENDED|NONE",
  "planCode": "...?",
  "planSeats": 50,
  "seatsUsed": 12,
  "seatsRemaining": 38,
  "startedAt": "...?",
  "expiresAt": "...?",
  "graceRemainingDays": 0,
  "subscriptionId": "...?"
}
```

`state=NONE` 시 `planCode`, `planSeats`, `seatsRemaining`, `subscriptionId` 모두 null. seatsUsed 는 organization master.

### 7.13 PaymentCheckoutLinkResponse

```json
{
  "items": [{ "priceId": "pri_xxx", "quantity": 1 }],
  "customer": { "email": "...", "name": "..." },
  "customData": { "organization_id": "..." },
  "plan": {
    "planCode": "...", "planSeats": 50,
    "displayName": "...", "amount": 99000, "currency": "KRW"
  },
  "paddleCustomerId": "ctm_xxx?"
}
```

---

## 8. Error Code (`ErrorCode` enum, common-core 기반)

| Code | HTTP | 사용처 |
|---|---|---|
| `CMN-400-001` | 400 | CHECK_PARAMETER (validation, role assignment 등) |
| `CMN-400-002` | 400 | VALIDATION_FAILED |
| `CMN-401-001` | 401 | Unauthorized (auth/me 의 anonymous principal) |
| `AUTH-401-001` | 401 | TOKEN_INVALID (refresh / oauth id_token / audience) |
| `AUTH-401-003` | 401 | AUTHENTICATION_FAILED (login 실패, InvalidCurrentPassword) |
| `AUTH-403-001` | 403 | ACCESS_DENIED (license/seat / NotSearchable) |
| `AUTH-403-002` | 403 | PERMISSION_DENIED (role: NotOwner / NotOwnerOrAdmin / NotStaff / NotAuthor / NotSelf / RoleRequired / OrgRoleRequired / NotMember) |
| `AUTH-403-003` | 403 | ACCOUNT_DISABLED (DELETION_PENDING / Suspended) |
| `CMN-404-001` | 404 | RESOURCE_NOT_FOUND (organization / membership / invite-code / device / plan / measurement) |
| `USR-404-001` | 404 | USER_NOT_FOUND |
| `CMN-409-001` | 409 | DUPLICATE_REQUEST (EmailDuplicate / InvalidStateTransition / AlreadyHasActiveSubscription) |
| `CMN-409-002` | 409 | ALREADY_CONNECTED (AlreadyMember) |
| `INV-409-001` | 409 | INVITE_CODE_USED (USED / REVOKED 상태) |
| `INV-410-001` | 410 | INVITE_CODE_EXPIRED |
| `LIC-403-001` | 403 | LICENSE_INACTIVE / NO_SEATS_REMAINING (LicenseBlocked / SeatConsumeFailed / LicenseDenied) |
| `CMN-502-001` | 502 | EXTERNAL_SERVICE_ISSUE (PaymentServiceUnavailable / VerifierUnavailable) |

응답에는 항상 i18n 으로 resolve 된 `message` 와 sealed Error record 가 채운 `error` (description) 가 들어간다.

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
    USER[user-service\n:9090 UserBasic]:::internal
    ARC[Archive Collector\n:9090 Collect]:::internal

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
    B2B -- gRPC --> USER
    B2B -- gRPC --> ARC
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
        VARCHAR status "ACTIVE/SUSPENDED/DELETED"
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
| Postman | `b2b-service/postman/B2B-Service-API.postman_collection.json` |

---

## 13. 미구현 / 후속 (roadmap.md 참조)

| 항목 | 상태 |
|---|---|
| 회원 검색 cross-MSA 이름/전화 enrich | ✅ user-service UserBasic gRPC 연동 완료 |
| 측정 상태 필터 (측정 필요/미측정/측정 완료) | ❌ |
| 리포트 scope 분기 (ACTIVE/GRACE → full, EXPIRED → basic + locked_items) | ❌ |
| WebSocket/SSE 실시간 동기화 (회원 가입/측정 푸시) | ❌ |
| 회원 history (재가입 추적) | ❌ |

전체 로드맵. [`b2b-service/docs/roadmap.md`](../../../b2b-service/docs/roadmap.md)
