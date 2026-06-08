# B2B Service API

> 갱신. 2026-06-08
> Base path. `/api/v2/b2b/...`
> 응답 envelope. `CncResponse` (Pattern B)

---

## 1. 목차

표기. `Public`, `Auth`, `OWNER · ADMIN`, `OWNER · ADMIN · TRAINER`, `Self only`, `Author only`.

| 도메인          | endpoint 수 | 주요 권한       | 섹션                                          |
|-----------------|-------------|-----------------|-----------------------------------------------|
| Auth            | 5           | Public · Auth  | [§2.1 ~ §2.6](#21-post-apiv2b2bauthlogin)     |
| User            | 5           | Self only      | [§2.7 ~ §2.11](#27-post-apiv2b2busers)        |
| Organization    | 5           | Public · Auth · OWNER · ADMIN | [§2.12 ~ §2.16](#212-post-apiv2b2borganizations) |
| Membership      | 7           | Public · OWNER · ADMIN | [§2.18 ~ §2.24](#218-post-apiv2b2borganizationsorgidjoin-requests) |
| Invite Code     | 4           | Public · Auth · OWNER · ADMIN | [§2.25 ~ §2.28](#225-post-apiv2b2borganizationsorgidinvite-codes) |
| Device          | 7           | Auth · OWNER · ADMIN | [§2.29 ~ §2.33.2](#229-post-apiv2b2borganizationsorgiddevices) |
| Measurement     | 3           | OWNER · ADMIN · TRAINER | [§2.34 ~ §2.36](#234-get-apiv2b2borganizationsorgidmembersmemberidmeasurements) |
| Feedback        | 5           | Auth · OWNER · ADMIN · TRAINER · Author only | [§2.37 ~ §2.41](#237-post-apiv2b2bfeedbacksmembershipsid) |
| License         | 2           | Auth           | [§2.42 ~ §2.43](#242-get-apiv2b2blicense-summary) |
| Billing         | 3           | OWNER · ADMIN  | [§2.44 ~ §2.46](#244-post-apiv2b2bbillingcheckout-init) |

총 46 endpoint. § 번호 결번 (§2.4 `GET /auth/me`, §2.17 `GET /organizations/mine`) 은 코드에서 제거된 슬롯 — 추적 편의상 유지.

---

## 2. Endpoint Detail

### 2.1 `POST /api/v2/b2b/auth/login`

로그인. `X-Client-Type` 헤더로 web 세션 / 모바일 토큰 분기.

**권한**. Public

**Headers**

| 헤더 | 값 | 설명 |
|---|---|---|
| `X-Client-Type` | `web` 또는 미지정 | `web` → Spring Session Cookie 발급. 그 외 → mobile token 발급 (default). |

**요청 바디** — `LoginRequest`

```json
{
  "email": "owner@example.com",
  "password": "P@ssw0rd123",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

검증. `email` ValidEmail, `password` NotBlank. `deviceId`/`deviceType` optional (rotation 추적용).

**응답 — 200 (mobile)** — `TokenPair`

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

**응답 — 200 (web, `X-Client-Type: web`)**

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

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-003` | 비밀번호 불일치 / 미가입 |
| 403 | `AUTH-403-003` | 계정 SUSPENDED / DELETION_PENDING |

---

### 2.2 `POST /api/v2/b2b/auth/token`

refresh token rotation. 모바일에서 access token 만료 시 호출. family rotation — 같은 refresh 재사용 시 family 전체 폐기.

**권한**. Public (refresh token 자체로 인증)

**요청 바디** — `RefreshRequest`

```json
{
  "refreshToken": "rt_2f81a3...",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**응답 — 200** — `TokenPair` (새 access + rotated refresh)

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

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-001` | refresh token 무효 / 만료 / 이미 rotated (family compromised) |

---

### 2.3 `POST /api/v2/b2b/auth/logout`

JTI revocation + (web 흐름) Session invalidate.

**권한**. Auth (Authorization 헤더 또는 Cookie 중 어느 한쪽)

**Headers**

| 헤더 | 값 |
|---|---|
| `Authorization` | `Bearer eyJ...` (모바일) — access 의 jti 를 revoke list 에 추가 |

**요청 바디** — `LogoutRequest` (optional — refresh family 까지 폐기하려면 전달)

```json
{ "refreshToken": "rt_2f81a3..." }
```

**응답 — 200**

```json
{ "success": true, "code": "200", "message": "OK", "data": null }
```

---

### 2.5 `POST /api/v2/b2b/auth/oauth2/google`

Google ID 토큰 검증 후 가입/로그인 후 JWT 발급.

**권한**. Public. 단, `b2b.oauth2.enabled=true` 설정 시에만 라우트 등록 (false 시 404)

**요청 바디** — `OAuth2LoginRequest`

```json
{
  "idToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEy...",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**응답 — 200** — `TokenPair` (4.1 모바일 응답과 동일 shape).

**에러** (`OAuth2Error`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-001` | InvalidIdToken — 서명/만료/issuer 불일치 |
| 401 | `AUTH-401-001` | InvalidAudience — aud 가 b2b clientId 와 다름 |
| 400 | `CMN-400-001` | EmailMissing — Google 가 이메일 미제공 |
| 403 | `AUTH-403-003` | AccountSuspended — DELETION_PENDING 등 |
| 502 | `CMN-502-001` | VerifierUnavailable — Google JWKS 다운 |

---

### 2.6 `POST /api/v2/b2b/auth/oauth2/apple`

Apple ID 토큰 검증 후 가입/로그인 후 JWT 발급. §2.5 와 동일 shape, 검증 verifier 만 Apple JWKS.

---

### 2.7 `POST /api/v2/b2b/users`

관리자 회원가입 (이메일 + 비밀번호 + 이름).

**권한**. Public

**요청 바디** — `SignUpRequest`

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

**응답 — 201** — `UserResponse`

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

**에러** (`UserError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 409 | `CMN-409-001` | EmailDuplicate |
| 400 | `CMN-400-001` | InvalidPasswordFormat |

---

### 2.8 `GET /api/v2/b2b/users/{b2bUserId}`

본인 프로필 + 소속 organization (multi-org, license 포함).

**권한**. Self only — path 의 `b2bUserId` 가 인증된 principal 과 일치해야 함. 불일치 시 403

**응답 — 200** — `UserResponse` (with `organizations[]`)

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

**에러** (`UserError`)

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

### 2.9 `PATCH /api/v2/b2b/users/{b2bUserId}`

프로필 부분 수정. JsonNullable 의 3-state semantics 사용.

**권한**. Self only

**요청 바디** — `UpdateProfileRequest`

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

**응답 — 200** — `UserResponse` (변경 후 전체 상태).

**에러** (`UserError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotSelf |
| 404 | `USR-404-001` | NotFound |

---

### 2.10 `POST /api/v2/b2b/users/{b2bUserId}/password`

비밀번호 변경. 변경 성공 시 모든 session/token 폐기.

**권한**. Self only

**요청 바디** — `ChangePasswordRequest`

```json
{
  "currentPassword": "P@ssw0rd123",
  "newPassword": "NewP@ssw0rd456"
}
```

**응답 — 204** (body 없음, envelope 없음).

**에러** (`UserError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-003` | InvalidCurrentPassword |
| 400 | `CMN-400-001` | InvalidPasswordFormat |
| 400 | `CMN-400-001` | SamePasswordReuse |
| 403 | `AUTH-403-002` | NotSelf |

---

### 2.11 `DELETE /api/v2/b2b/users/{b2bUserId}`

soft 탈퇴 — `accountStatus=DELETION_PENDING` + 모든 세션 폐기. archive outbox 가 비동기로 Carenco Archive Collector 에 publish.

**권한**. Self only

**응답 — 204**.

**에러**. §2.9 와 동일 (NotSelf / NotFound).

---

### 2.12 `POST /api/v2/b2b/organizations`

시설 등록. 호출한 b2b_user 가 OWNER 로 자동 멤버십 생성 (`joinedVia=OWNER_INIT`).

**권한**. Auth

**요청 바디** — `CreateOrganizationRequest`

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

**응답 — 201** — `OrganizationResponse`

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

**에러** (`OrganizationError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 400 | `CMN-400-001` | InvalidInput |

---

### 2.13 `GET /api/v2/b2b/organizations/{id}`

단건 조회 (공개).

**권한**. Public

**응답 — 200** — `OrganizationResponse` (4.12 와 동일 shape).

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |

---

### 2.14 `PATCH /api/v2/b2b/organizations/{id}`

부분 수정 (owner 만). JsonNullable 의 3-state semantics.

**권한**. OWNER

**요청 바디** — `UpdateOrganizationRequest`

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

**응답 — 200** — `OrganizationResponse`.

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |
| 403 | `AUTH-403-002` | NotOwner |

---

### 2.15 `DELETE /api/v2/b2b/organizations/{id}`

soft 삭제 (owner 만). `status=DELETED` + 모든 멤버십 LEFT + 모든 invite code REVOKED cascade. archive outbox 비동기 publish.

**권한**. OWNER

**응답 — 204**.

**에러** (`OrganizationError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |
| 403 | `AUTH-403-002` | NotOwner |

---

### 2.16 `GET /api/v2/b2b/organizations/search`

`searchable=true` + `status=ACTIVE` 시설 검색.

**권한**. Public

**Query 파라미터**

| 이름 | 타입 | 기본 |
|---|---|---|
| `keyword` | string | 이름/주소 LIKE 매칭 |
| `type` | `OrganizationType` | optional |
| `page` | int | 0 |
| `size` | int | 20 |

**응답 — 200**

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

### 2.18 `POST /api/v2/b2b/organizations/{orgId}/join-requests`

Carenco user 가 가입 신청. `searchable=true` 시설만 가능.

**권한**. Public (body 의 `memberId` 가 carenco user uuid)

**요청 바디** — `MembershipController.JoinRequest`

```json
{
  "memberId": "carenco-user-uuid",
  "note": "5월 한 달 등록 희망합니다"
}
```

**응답 — 201** — `MembershipResponse` (status=PENDING)

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

**에러** (`MembershipFlowError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-001` | OrganizationNotSearchable |
| 409 | `CMN-409-001` | AlreadyMember |

---

### 2.19 `POST /api/v2/b2b/memberships/{membershipId}/approve`

PENDING → ACTIVE. 좌석 +1 + license guard.

**권한**. OWNER · ADMIN

**응답 — 200** — `MembershipResponse`

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
    "memberNumber": "M260608-001",
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

**에러** (`MembershipFlowError`)

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

### 2.20 `POST /api/v2/b2b/memberships/{membershipId}/reject`

PENDING → LEFT (좌석 차감 없음).

**권한**. OWNER · ADMIN

**응답 — 200** — `MembershipResponse` (status=LEFT, leftAt 채워짐).

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 409 | `CMN-409-001` | InvalidStateTransition |

---

### 2.21 `POST /api/v2/b2b/memberships/{membershipId}/suspend`

ACTIVE → SUSPENDED.

**권한**. OWNER · ADMIN

**요청 바디** — `MembershipController.SuspendRequest` (optional)

```json
{ "reason": "회비 연체 3개월" }
```

**응답 — 200** — `MembershipResponse` (status=SUSPENDED).

**에러**. §2.20 와 동일.

---

### 2.22 `POST /api/v2/b2b/memberships/{membershipId}/leave`

본인 탈퇴. body 의 `requesterId` 와 멤버십의 `memberId` 가 일치해야 함. 좌석 −1 + archive outbox publish.

**권한**. Public (body 자기검증)

**요청 바디** — `MembershipController.LeaveRequest`

```json
{ "requesterId": "carenco-user-uuid" }
```

**응답 — 200** — `MembershipResponse` (status=LEFT).

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 409 | `CMN-409-001` | InvalidStateTransition |

---

### 2.23 `POST /api/v2/b2b/organizations/{orgId}/staff`

OWNER/ADMIN 이 기존 b2b_user 를 staff 로 지정. role 은 ADMIN 또는 TRAINER.

**권한**. OWNER · ADMIN

**요청 바디** — `MembershipController.AppointStaffRequest`

```json
{
  "b2bUserId": "u-staff-1",
  "role": "TRAINER",
  "memberNumber": "T260608-001"
}
```

**응답 — 201** — `MembershipResponse`

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
    "memberNumber": "T260608-001",
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

**에러** (`MembershipFlowError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 400 | `CMN-400-001` | InvalidRoleAssignment (OWNER 부여 시도 등) |
| 409 | `CMN-409-001` | AlreadyMember |

---

### 2.24 `GET /api/v2/b2b/organizations/{orgId}/members`

회원 목록 (membership + user-service 정보 + measurement 요약). PDF 흐름 4.

**권한**. OWNER · ADMIN · TRAINER. user/measure-service 장애 시 user/measurement 필드는 null — membership 정보만 노출 (graceful degradation)

**Query 파라미터**

| 이름 | 타입 | 기본 |
|---|---|---|
| `query` | string | 이름 부분일치 (user-service `SearchUsers` 4가지 결합 매칭 — firstName+lastName / lastName+firstName / firstName / lastName, 공백 무시) |
| `include_nickname` | boolean | `false` (true 시 `query` 매칭에 nickname 포함) |
| `sort_by` | `NAME` / `REGISTERED_AT` / `LATEST_MEASUREMENT` | `REGISTERED_AT` |
| `status` | `MembershipStatus` | `ACTIVE` |
| `page` | int | 0 |
| `size` | int | 20 (max 100) |

> 0.0.52 에서 phone_number 매칭은 user-service 측에서 제거됨 — 이름/nickname 만 검색 가능.

**응답 — 200** — `MemberListResponse`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      {
        "membershipId": "m-7c1d",
        "memberId": "carenco-user-uuid",
        "memberNumber": "M260608-001",
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

**에러** (`MemberListError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | RoleRequired |
| 404 | `CMN-404-001` | OrganizationNotFound |

---

### 2.25 `POST /api/v2/b2b/organizations/{orgId}/invite-codes`

초대 코드 발급. 헷갈리는 문자 (0/O, 1/l/I) 회피 generator. TTL 48h.

**권한**. OWNER · ADMIN

**요청 바디** — `IssueInviteCodeRequest`

```json
{
  "description": "5월 신규 회원용",
  "codeLength": 8
}
```

`codeLength` 6~12 (기본 8).

**응답 — 201** — `InviteCodeResponse`

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

**에러** (`InviteCodeError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 403 | `AUTH-403-001` | InviteCodeDisabled (organization.inviteCodeEnabled=false) |
| 403 | `LIC-403-001` | LicenseBlocked (EXPIRED/SUSPENDED/NONE) |

---

### 2.26 `GET /api/v2/b2b/organizations/{orgId}/invite-codes`

ACTIVE 코드 목록.

**권한**. Auth

**응답 — 200**

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

### 2.27 `POST /api/v2/b2b/invite-codes/{inviteCodeId}/revoke`

수동 폐기 (ACTIVE → REVOKED).

**권한**. OWNER · ADMIN

**응답 — 200** — `InviteCodeResponse` (status=REVOKED).

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | InviteCodeNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |

---

### 2.28 `POST /api/v2/b2b/invite-codes/redeem`

Carenco user 가 코드 입력하여 가입.

**권한**. Public. body 의 `memberId` 가 carenco userId

**요청 바디** — `InviteCodeController.RedeemRequest`

```json
{
  "code": "Hk7mPqR3",
  "memberId": "carenco-user-uuid"
}
```

**응답 — 200**

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

**에러** (`InviteCodeError`)

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

### 2.29 `POST /api/v2/b2b/organizations/{orgId}/devices`

InBody 등 측정 device 등록. device-service gRPC 위임.

**권한**. OWNER · ADMIN. license state ACTIVE/GRACE 만

**요청 바디** — `RegisterDeviceRequest`

```json
{
  "serialNumber": "INBODY-770-2024-0123",
  "alias": "1번 InBody"
}
```

**응답 — 201** — `DeviceResponse`

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

**에러** (`DeviceError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | OrgRoleRequired |
| 403 | `LIC-403-001` | LicenseDenied (EXPIRED/SUSPENDED) |

---

### 2.30 `GET /api/v2/b2b/organizations/{orgId}/devices`

device 목록.

**권한**. Auth (멤버)

**Query 파라미터**

| 이름 | 타입 | 설명 |
|---|---|---|
| `keyword` | string | serialNumber / alias 부분일치 |
| `status` | `DeviceStatus` (`ACTIVE` / `INACTIVE`) | 필터 |
| `sortBy` | string | `name` / `registered_at` (default DESC) / `last_used` / **`battery`** (0.0.61, NULLS LAST) |

**응답 — 200**

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
        "lastUsedAt": "2026-05-13T11:20:00Z",
        "batteryLevel": 87,
        "batteryReportedAt": "2026-05-13T11:20:00Z"
      }
    ]
  }
}
```

`batteryLevel` / `batteryReportedAt` 는 0.0.61 추가. 측정 record 가 저장될 때마다 ML 응답의 battery 가 fire-and-forget 으로 갱신됨. 측정 이력 없는 device 는 둘 다 `null`.

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotMember |

---

### 2.31 `GET /api/v2/b2b/organizations/{orgId}/devices/{deviceId}`

device 단건.

**권한**. Auth (멤버)

**응답 — 200** — `DeviceResponse` (4.29 와 동일).

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | DeviceNotInOrg |

---

### 2.32 `PATCH /api/v2/b2b/organizations/{orgId}/devices/{deviceId}`

alias 만 변경 가능. device-service gRPC 가 `alias=null` 을 "변경 없음" 으로 해석 → `null` 거부 (JsonNullableNonNull).

**권한**. OWNER · ADMIN

**요청 바디** — `UpdateDeviceRequest`

```json
{ "alias": "1번 InBody (대기실)" }
```

**응답 — 200** — `DeviceResponse`.

---

### 2.33 `POST /api/v2/b2b/organizations/{orgId}/devices/{deviceId}/deactivate`

device 비활성화 (status=INACTIVE).

**권한**. OWNER · ADMIN

**요청 바디** — `DeactivateDeviceRequest` (optional)

```json
{ "reason": "수리 입고" }
```

**응답 — 200** — `DeviceResponse` (status=INACTIVE, deactivatedAt/deactivatedBy 채워짐).

---

### 2.33.1 `GET .../devices/preview` (0.0.61 신규)

시리얼만으로 풀 조회 — 등록 전 미리보기. claim 안 함, write 없음. UI 의 "기기 등록" 흐름에서 serial 입력 후 alias 입력 전 호출.

**권한**. OWNER · ADMIN

**Query 파라미터**

| 이름 | 필수 | 비고 |
|---|---|---|
| `serial` | yes | 풀에서 찾을 `hardware_serial` |

**응답 — 200** — `PreviewDeviceResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `found` | boolean | 풀에 있고 provisional 아닌 경우 true |
| `alreadyClaimed` | boolean | 이미 다른 owner 가 등록함 |
| `hardwareSerial` | string | found=true 일 때만 |
| `deviceType` | string | `DeviceType` 문자열 (e.g. `SCALE2`) |
| `firmwareVersion` | string | 풀 시점 펌웨어, nullable |

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `OrgRoleRequired` | caller 가 OWNER/ADMIN 아님 |

---

### 2.33.2 `DELETE .../devices/{deviceId}` (0.0.61 신규)

영구 삭제 — `status=REVOKED` + 풀 row unclaim. 일시 정지 ([§2.33](#233-post-apiv2b2borganizationsorgiddevicesdeviceiddeactivate)) 와 다름. 같은 `(mac, serial)` 가 다른 organization 에 재등록 가능해짐.

**권한**. OWNER · ADMIN

**Query 파라미터**

| 이름 | 필수 | 비고 |
|---|---|---|
| `reason` | no | audit (예. `lost` / `discarded` / `replaced`) |

**응답 — 200** — `DeviceResponse` (status=INACTIVE, deactivatedAt/deactivatedBy/deactivationReason 채워짐, batteryLevel 마지막 값 유지).

**Idempotent**. 이미 REVOKED 인 device 에 다시 호출해도 200 + 같은 row 반환.

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `OrgRoleRequired` | caller 권한 부족 |
| 404 | `DeviceNotInOrg` | device 가 해당 organization 소유가 아님 |

---

### 2.34 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements`

회원 측정 이력 — measure-service gRPC.

**권한**. OWNER · ADMIN · TRAINER. 호출자 OWNER/ADMIN/TRAINER 이고 `memberId` 가 organization 의 ACTIVE 멤버

**Query 파라미터**

| 이름 | 타입 | 설명 |
|---|---|---|
| `from` | string (ISO date) | 측정일 ≥ from |
| `to` | string (ISO date) | 측정일 ≤ to |
| `page` | int | 0 |
| `size` | int | 20 |

**응답 — 200** — `MeasurementClient.Page`

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

**에러** (`MeasurementError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | RoleRequired |
| 404 | `CMN-404-001` | TargetNotActiveMember |

---

### 2.35 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/{recordId}`

단건 측정.

**권한**. OWNER · ADMIN · TRAINER

**응답 — 200** — `MeasurementInfo` (4.34 의 한 item 과 동일).

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | RecordNotFound |

---

### 2.36 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/summary`

denormalized 요약 (measure-service `user_measurement_summary`).

**권한**. OWNER · ADMIN · TRAINER

**응답 — 200** — `MeasurementSummary`

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

### 2.37 `POST /api/v2/b2b/feedbacks/memberships/{membershipId}`

피드백 작성 — TRAINER/ADMIN/OWNER 가 회원에 코멘트.

**권한**. OWNER · ADMIN · TRAINER

**요청 바디** — `CreateFeedbackRequest`

```json
{
  "body": "어깨 가동범위가 좋아졌습니다. 다음 주는 하체 위주로.",
  "visibility": "MEMBER",
  "measurementRecordId": "r-9e8d"
}
```

`visibility`. `MEMBER` (회원 앱 노출) / `INTERNAL` (스태프만). `measurementRecordId` 선택 — 특정 측정과 연관 시.

**응답 — 201** — `FeedbackResponse`

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

**에러** (`FeedbackError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotStaff |
| 400 | `CMN-400-001` | InvalidInput |

---

### 2.38 `PATCH /api/v2/b2b/feedbacks/{feedbackId}`

본문 수정 (작성자 본인만).

**권한**. Author only

**요청 바디** — `UpdateFeedbackRequest`

```json
{ "body": "오타 수정: 다음 주는 하체+코어 위주." }
```

**응답 — 200** — `FeedbackResponse` (editedAt 채워짐).

**에러**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | FeedbackNotFound |
| 403 | `AUTH-403-002` | NotAuthor |

---

### 2.39 `DELETE /api/v2/b2b/feedbacks/{feedbackId}`

soft 삭제 (deletedAt 채움).

**권한**. Author only

**응답 — 200** — `FeedbackResponse` (deletedAt 채워진 상태로 echo).

---

### 2.40 `GET /api/v2/b2b/feedbacks/memberships/{membershipId}`

회원 피드백 목록.

**권한**. Auth

**Query 파라미터**. `page` (0), `size` (20).

**응답 — 200**

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

### 2.41 `GET /api/v2/b2b/feedbacks/measurements/{recordId}`

특정 측정에 달린 피드백 목록 (회원이 본인 측정 결과를 볼 때).

**권한**. Auth

**Query / Response**. §2.40 와 동일 shape.

---

### 2.42 `GET /api/v2/b2b/license-summary`

본인 가입/소유 organization 들의 license + seat 요약 (한 번에).

**권한**. Auth

**응답 — 200**

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
        "graceRemainingSeconds": 0,
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
        "graceRemainingSeconds": 345600,
        "subscriptionId": "sub_yyy"
      }
    ]
  }
}
```

---

### 2.43 `GET /api/v2/b2b/license-summary/{organizationId}`

특정 organization 의 license 단건.

**권한**. Auth (해당 시설 멤버만)

**응답 — 200** — `LicenseSummaryResponse` (4.42 의 item 과 동일).

**에러** (`LicenseSummaryError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-002` | NotMember |

---

### 2.44 `POST /api/v2/b2b/billing/checkout-init`

결제 시작 진입점. b2b 가 권한 검증 후 payment-service 위임.

**권한**. OWNER

**요청 바디** — `CheckoutInitRequest`

```json
{
  "organizationId": "org-A",
  "priceId": "pri_01h5xxx"
}
```

**응답 — 200** — `PaymentCheckoutLinkResponse` echo

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

**에러** (`BillingError`)

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

### 2.45 `POST /api/v2/b2b/billing/plan-change`

기존 구독의 plan 변경 (price 변경). b2b 가 OWNER 검증 후 payment-service REST 위임. Paddle PATCH 즉시 반영 + 후속 webhook 으로 DB 정합 확정.

**권한**. OWNER

**요청 바디** — `ChangePlanRequest`

```json
{
  "organizationId": "org-A",
  "newPriceId": "pri_01h6_pro"
}
```

검증. `organizationId` ValidUuid, `newPriceId` NotBlank.

**응답 — 200** — `PaymentPlanChangeResponse`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "paddleSubscriptionId": "sub_xxx",
    "newPriceId": "pri_01h6_pro",
    "newPlanCode": "PILATES_PRO",
    "newPlanSeats": 100,
    "newBillingInterval": "month",
    "prorationMode": "prorated_immediately",
    "status": "active"
  }
}
```

**에러** (`BillingError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 404 | `CMN-404-001` | PlanNotFound — 새 priceId 가 subscription_plans 에 없음 |
| 404 | `CMN-404-001` | NoActiveSubscription — 변경할 구독이 없음 |
| 403 | `AUTH-403-002` | NotOwner |
| 409 | `CMN-409-001` | SamePlanAsCurrent — 현재와 동일한 priceId |
| 4xx | `CMN-400-001`/`CMN-404-001`/`CMN-409-001` | PaymentRejected — payment-service 4xx (statusCode 별 매핑) |
| 502 | `CMN-502-001` | PaymentServiceUnavailable |

**연쇄 호출**

```
BillingController.planChange → BillingService.changePlan
  ├─ OrganizationRepository.findById
  ├─ owner 검증 (organization.ownerUserId == principal)
  └─ PaymentClient.changePlan (HTTP → payment-service)
       └─ payment-service: Paddle PATCH /subscriptions/{id} + subscription_plan_history 기록
```

---

### 2.46 `GET /api/v2/b2b/billing/organizations/{organizationId}/transactions`

organization 의 결제 transaction 이력. payment-service REST 응답을 그대로 echo.

**권한**. OWNER

**응답 — 200** — `List<PaymentTransactionView>`

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": [
    {
      "id": "txn-9c2a",
      "paddleTransactionId": "txn_01h_xxx",
      "customerId": "ctm_xxx",
      "subscriptionId": "sub_xxx",
      "status": "completed",
      "currencyCode": "KRW",
      "subtotal": 99000,
      "discountTotal": 0,
      "taxTotal": 9900,
      "totalAmount": 108900,
      "billingPeriodStartsAt": "2026-05-01T00:00:00Z",
      "billingPeriodEndsAt": "2026-06-01T00:00:00Z",
      "billingCountryCode": "KR",
      "billingPostalCode": "06000",
      "billingRegion": "서울",
      "billingCity": "강남구",
      "billingAddressLine": "테헤란로 123",
      "items": [
        {
          "id": "ti-1",
          "paddleItemId": "txnitm_xxx",
          "priceId": "pri_01h5xxx",
          "productId": "pro_xxx",
          "productName": "PILATES_BASIC (50석)",
          "productDescription": "월 99,000원 / 50명 수용",
          "quantity": 1,
          "unitPrice": 99000,
          "subtotal": 99000,
          "discountAmount": 0,
          "taxAmount": 9900,
          "total": 108900,
          "taxRate": "0.10"
        }
      ],
      "payments": [
        {
          "id": "pay-1",
          "amount": 108900,
          "status": "captured",
          "paymentType": "card",
          "cardType": "visa",
          "cardLast4": "4242",
          "cardExpiryMonth": 12,
          "cardExpiryYear": 2028,
          "cardholderName": "OWNER PARK",
          "capturedAt": "2026-05-01T00:05:00Z"
        }
      ],
      "createdAt": "2026-05-01T00:05:00Z"
    }
  ]
}
```

이력이 없으면 `data: []`.

**에러** (`BillingError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-002` | NotOwner |
| 502 | `CMN-502-001` | PaymentServiceUnavailable |

**연쇄 호출**

```
BillingController.listTransactions → BillingService.listTransactions
  ├─ OrganizationRepository.findById
  ├─ owner 검증
  └─ PaymentClient.listTransactions(organizationId) (HTTP → payment-service)
```
