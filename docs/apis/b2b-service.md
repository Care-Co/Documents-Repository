# B2B Service API

> 갱신. 2026-06-23
> Base path. `/api/v2/b2b/...`
> 응답 envelope. `CncResponse` (Pattern B)

---

## 1. 목차

표기. `Public`, `Auth`, `OWNER · ADMIN`, `OWNER · ADMIN · TRAINER`, `Self only`, `Author only`.

| 도메인          | endpoint 수 | 주요 권한       | 섹션                                          |
|-----------------|-------------|-----------------|-----------------------------------------------|
| Auth            | 5           | Public · Auth  | [2.1 ~ 2.6](#21-post-apiv2b2bauthlogin)     |
| User            | 5           | Self only      | [2.7 ~ 2.11](#27-post-apiv2b2busers)        |
| Organization    | 5           | Public · Auth · OWNER · ADMIN | [2.12 ~ 2.16](#212-post-apiv2b2borganizations) |
| Membership      | 7           | Public · OWNER · ADMIN | [2.18 ~ 2.24](#218-post-apiv2b2borganizationsorgidjoin-requests) |
| Invite Code     | 4           | Public · Auth · OWNER · ADMIN | [2.25 ~ 2.28](#225-post-apiv2b2borganizationsorgidinvite-codes) |
| Device          | 7           | Auth · OWNER · ADMIN | [2.29 ~ 2.33.2](#229-post-apiv2b2borganizationsorgiddevices) |
| Measurement     | 3           | OWNER · ADMIN · TRAINER | [2.34 ~ 2.36](#234-get-apiv2b2borganizationsorgidmembersmemberidmeasurements) |
| Feedback        | 5           | Auth · OWNER · ADMIN · TRAINER · Author only | [2.37 ~ 2.41](#237-post-apiv2b2bfeedbacksmembershipsid) |
| License         | 2           | Auth           | [2.42 ~ 2.43](#242-get-apiv2b2blicense-summary) |
| Billing         | 3           | OWNER · ADMIN  | [2.44 ~ 2.46](#244-post-apiv2b2bbillingcheckout-init) |
| B2C / 가용성     | 2           | Public         | [2.47 ~ 2.48](#247-get-apiv2b2bavailability) |

총 48 endpoint. 번호 결번 (2.4 `GET /auth/me`, 2.17 `GET /organizations/mine`) 은 코드에서 제거된 슬롯 — 추적 편의상 유지.

---

## 2. Endpoint Detail

### 2.1 `POST /api/v2/b2b/auth/login`

로그인. `X-Client-Type` 헤더로 web 세션 / 모바일 토큰 분기.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/auth/login | Public | no | no |

**Headers**

| 헤더 | 값 | 설명 |
|---|---|---|
| `X-Client-Type` | `web` 또는 미지정 | `web` → Spring Session Cookie 발급. 그 외 → mobile token 발급 (default). |

**Request Body** — `LoginRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `email` | string | yes | ValidEmail |
| `password` | string | yes | NotBlank |
| `deviceId` | string | no | rotation 추적용 |
| `deviceType` | string | no | rotation 추적용 (e.g. `iOS` / `Android`) |

```json
{
  "email": "owner@example.com",
  "password": "P@ssw0rd123",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**Response — 200 (mobile)** — `TokenPair`

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string | 유저 PK |
| `accessToken` | string | JWT access token |
| `refreshToken` | string | refresh token (rotation 대상) |

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

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string | 유저 PK |
| `email` | string | identifier |
| `sessionId` | string | Spring Session id |

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

### 2.2 `POST /api/v2/b2b/auth/token`

refresh token rotation. 모바일에서 access token 만료 시 호출. family rotation — 같은 refresh 재사용 시 family 전체 폐기.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/auth/token | Public (refresh token 자체로 인증) | no | no |

**Request Body** — `RefreshRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `refreshToken` | string | yes | NotBlank |
| `deviceId` | string | no | rotation 추적용 |
| `deviceType` | string | no | rotation 추적용 |

```json
{
  "refreshToken": "rt_2f81a3...",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**Response — 200** — `TokenPair` (새 access + rotated refresh)

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string | 유저 PK |
| `accessToken` | string | 새 JWT access token |
| `refreshToken` | string | rotated refresh token |

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

### 2.3 `POST /api/v2/b2b/auth/logout`

JTI revocation + (web 흐름) Session invalidate.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/auth/logout | Auth (Authorization 헤더 또는 Cookie 중 어느 한쪽) | yes | no |

**Headers**

| 헤더 | 값 |
|---|---|
| `Authorization` | `Bearer eyJ...` (모바일) — access 의 jti 를 revoke list 에 추가 |

**Request Body** — `LogoutRequest` (optional — refresh family 까지 폐기하려면 전달)

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `refreshToken` | string | no | family 폐기 대상 |

```json
{ "refreshToken": "rt_2f81a3..." }
```

**Response — 200**

`data` 는 null. envelope 만.

```json
{ "success": true, "code": "200", "message": "OK", "data": null }
```

---

### 2.5 `POST /api/v2/b2b/auth/oauth2/google`

Google ID 토큰 검증 후 가입/로그인 후 JWT 발급.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/auth/oauth2/google | Public (단 `b2b.oauth2.enabled=true` 시에만 라우트 등록, false 시 404) | no | no |

**Request Body** — `OAuth2LoginRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `idToken` | string | yes | Google ID Token (JWT, RS256) |
| `deviceId` | string | no | rotation 추적용 |
| `deviceType` | string | no | rotation 추적용 |

```json
{
  "idToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEy...",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**Response — 200** — `TokenPair`

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string | 유저 PK |
| `accessToken` | string | JWT access token |
| `refreshToken` | string | refresh token (rotation 대상) |

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "userId": "u-9f3e",
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.OAUTH_GOOGLE...",
    "refreshToken": "rt_google_b4d2..."
  }
}
```

**Errors** (`OAuth2Error`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-001` | InvalidIdToken — 서명/만료/issuer 불일치 |
| 401 | `AUTH-401-001` | InvalidAudience — aud 가 b2b clientId 와 다름 |
| 400 | `CMN-400-001` | EmailMissing — Google 가 이메일 미제공 |
| 403 | `AUTH-403-003` | AccountSuspended — DELETION_PENDING 등 |
| 502 | `CMN-502-001` | VerifierUnavailable — Google JWKS 다운 |

---

### 2.6 `POST /api/v2/b2b/auth/oauth2/apple`

Apple ID 토큰 검증 후 가입/로그인 후 JWT 발급. verifier 는 Apple JWKS — `aud` 는 `com.carenco.b2b.web` (웹) 또는 모바일 client id.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/auth/oauth2/apple | Public (단 `b2b.oauth2.enabled=true` 시에만 라우트 등록, false 시 404) | no | no |

**Request Body** — `OAuth2LoginRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `idToken` | string | yes | Apple ID Token (JWT, RS256) |
| `deviceId` | string | no | rotation 추적용 |
| `deviceType` | string | no | rotation 추적용 |

```json
{
  "idToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IkFwcGxlS2lk...",
  "deviceId": "iphone-15-uuid",
  "deviceType": "iOS"
}
```

**Response — 200** — `TokenPair`

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string | 유저 PK |
| `accessToken` | string | JWT access token |
| `refreshToken` | string | refresh token (rotation 대상) |

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "userId": "u-9f3e",
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.OAUTH_APPLE...",
    "refreshToken": "rt_apple_c5e3..."
  }
}
```

**Errors** (`OAuth2Error`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-001` | InvalidIdToken — 서명/만료/issuer 불일치 (Apple JWKS) |
| 401 | `AUTH-401-001` | InvalidAudience — aud 가 `com.carenco.b2b.web` (또는 모바일 client id) 와 다름 |
| 400 | `CMN-400-001` | EmailMissing — Apple 가 이메일 미제공 (private relay 거부 등) |
| 403 | `AUTH-403-003` | AccountSuspended — DELETION_PENDING 등 |
| 502 | `CMN-502-001` | VerifierUnavailable — Apple JWKS 다운 |

---

### 2.7 `POST /api/v2/b2b/users`

관리자 회원가입 (이메일 + 비밀번호 + 이름).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/users | Public | no | no |

**Request Body** — `SignUpRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `email` | string | yes | ValidEmail |
| `password` | string | yes | ValidPassword (영문/숫자/특수 1자 이상, 8자 이상) |
| `firstName` | string | yes | ValidName (한글 또는 영문) |
| `lastName` | string | yes | ValidName |
| `country` | string | yes | ValidCountryCode (ISO-3166 alpha-2 지원 셋) |
| `phoneNumber` | string | yes | ValidPhoneNumber (E.164) |

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

**Response — 201** — `UserResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | 유저 PK |
| `email` | string | identifier |
| `firstName` | string | 이름 |
| `lastName` | string | 성 |
| `phoneNumber` | string | E.164 |
| `country` | string | ISO-3166 alpha-2 |
| `profileImageUrl` | string \| null | 프로필 이미지 URL |
| `accountStatus` | string | `ACTIVE` / `SUSPENDED` / `DELETION_PENDING` |
| `lastLoginTime` | string (date-time, UTC) \| null | 최근 로그인 시각 |
| `createdAt` | string (date-time, UTC) | 가입 시각 |
| `organizations` | array<object> | 소속 시설 (가입 직후 빈 배열) |

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

### 2.8 `GET /api/v2/b2b/users/{b2bUserId}`

본인 프로필 + 소속 organization (multi-org, license 포함).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/users/{b2bUserId} | Self only — path 의 `b2bUserId` 가 인증된 principal 과 일치해야 함. 불일치 시 403 | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `b2bUserId` | string (uuid) | 본인 |

**Response — 200** — `UserResponse` (with `organizations[]`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | 유저 PK |
| `email` | string | identifier |
| `firstName` | string | 이름 |
| `lastName` | string | 성 |
| `phoneNumber` | string | E.164 |
| `country` | string | ISO-3166 alpha-2 |
| `profileImageUrl` | string \| null | 프로필 이미지 URL |
| `accountStatus` | string | 계정 상태 |
| `lastLoginTime` | string (date-time, UTC) \| null | 최근 로그인 |
| `createdAt` | string (date-time, UTC) | 가입 시각 |
| `organizations` | array<object> | 소속 시설 목록 (아래 schema) |
| `organizations[].id` | string | organization PK |
| `organizations[].name` | string | 시설명 |
| `organizations[].type` | string | `OrganizationType` |
| `organizations[].role` | string | `OWNER` / `ADMIN` / `TRAINER` / `MEMBER` |
| `organizations[].seatsUsed` | integer | 사용 중 좌석 |
| `organizations[].planSeats` | integer | 플랜 좌석 한도 |
| `organizations[].licenseState` | string | `ACTIVE` / `GRACE` / `EXPIRED` / `SUSPENDED` / `NONE` |

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

### 2.9 `PATCH /api/v2/b2b/users/{b2bUserId}`

프로필 부분 수정. JsonNullable 의 3-state semantics 사용.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| PATCH | /api/v2/b2b/users/{b2bUserId} | Self only | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `b2bUserId` | string (uuid) | 본인 |

**Request Body** — `UpdateProfileRequest`

| field | 동작 |
|---|---|
| undefined (키 없음) | 변경 없음 |
| `"field": null` | 컬럼 NULL 로 설정 (firstName/lastName/searchable 류는 거부) |
| `"field": value` | 적용 |

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `firstName` | JsonNullable<string> | no | ValidName. null 거부 (UpdateProfileValidator) |
| `lastName` | JsonNullable<string> | no | ValidName. null 거부 |
| `phoneNumber` | JsonNullable<string> | no | ValidPhoneNumber |
| `profileImageUrl` | JsonNullable<string> | no | URL, null 로 clear 가능 |
| `country` | JsonNullable<string> | no | ValidCountryCode |

```json
{
  "phoneNumber": "+821033334444",
  "profileImageUrl": "https://cdn.carenco.com/u/9f3e-new.jpg",
  "country": "JP"
}
```

(firstName/lastName 은 ValidName + UpdateProfileValidator — null 로 설정 불가)

**Response — 200** — `UserResponse` (변경 후 전체 상태, 2.7 schema 참조).

**Errors** (`UserError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotSelf |
| 404 | `USR-404-001` | NotFound |
| 400 | `CMN-400-002` | FieldCannotBeCleared (firstName/lastName 을 null 로 시도) |

---

### 2.10 `POST /api/v2/b2b/users/{b2bUserId}/password`

비밀번호 변경. 변경 성공 시 모든 session/token 폐기.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/users/{b2bUserId}/password | Self only | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `b2bUserId` | string (uuid) | 본인 |

**Request Body** — `ChangePasswordRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `currentPassword` | string | yes | NotBlank, 현재 비밀번호 확인 |
| `newPassword` | string | yes | ValidPassword |

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

### 2.11 `DELETE /api/v2/b2b/users/{b2bUserId}`

soft 탈퇴 — `accountStatus=DELETION_PENDING` + 모든 세션 폐기 + archive outbox publish + **cleanup outbox event enqueue** + **Redis 차단 키 (30일)**. archive 송신 완료 후 5초 cycle 워커가 cascade 진행 → 최종 hard delete.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| DELETE | /api/v2/b2b/users/{b2bUserId} | Self only | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `b2bUserId` | string (uuid) | 본인 |

**Response — 204**.

**Errors** (`UserError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotSelf |
| 404 | `USR-404-001` | NotFound |

**Cleanup cascade (v0.0.76 — V17 마이그 신규)**

`UserService.deleteAccount` 가 soft delete 트랜잭션 안에서.

1. `accountStatus=DELETION_PENDING` 마킹.
2. `ArchiveOutboxWriter` 가 `B2bUser` snapshot 을 `archive_outbox` 에 PENDING 으로 enqueue.
3. `AuthService.revokeAllSessions` 가 refresh token + session 폐기.
4. `B2bUserOutboxStateService.enqueue` 가 `b2b_user_outbox_events` 에 `USER_DELETION_REQUESTED` (INIT) 추가.
5. `TokenRevocationService.blockUser(userId, "DELETION_PENDING", 30일)` 가 Redis `b2b:blocked:user:{userId}` 키 publish — 인증 필터가 매 요청 검사하여 access token 차단.

`B2bUserOutboxWorker` (`@Scheduled fixedDelay=5s`, `b2b.user-cleanup.enabled` 로 토글) 가 INIT / retry-due event 를 claim → `B2bUserOutboxDeletionProcessor` 호출. cascade step 별로 payload flag (`orgSubsCancelled.{orgId}` / `orgDevicesRevoked.{orgId}` / `orgDeleted.{orgId}` / `membershipsLeftMarked` / `inviteCodesRevoked` / `feedbacksAnonymized` / `userHardDeleted`) 누적 — retry 시 완료된 step 은 skip.

| 순서 | 처리 | 비고 |
|---|---|---|
| 1 | OWNER organization cascade delete | payment cancel (404/409 idempotent) + device unregister (gRPC `UnregisterAllByOwner(B2B_CENTER)`) → feedbacks / invite_codes / memberships / organization hard delete |
| 2 | 본인 (OWNER 아닌) memberships → `status=LEFT` 마킹 (`leftAt` 채움) | 시설은 그대로 유지 |
| 3 | 본인 발급 active invite codes → `status=REVOKED` | 사용 차단 |
| 4 | 본인 작성 feedbacks → `authorUserId=null` 익명화 | 본문 보존 (멤버에 제공된 정보) |
| 5 | `archive_outbox` 의 entityType=B2bUser entityId=userId 가 `SENT` 인지 확인 후 → `b2b_users` hard delete | SENT 아니면 다음 tick 까지 보류 |

cascade 완료 후 같은 email 로 재가입 가능 (unique 제약 해제).

**Cross-MSA 호출 (cleanup worker)**

| 대상 | 호출 | 동작 |
|---|---|---|
| payment-service | `POST /api/subscriptions/{id}/cancel` | OWNER organization 의 active subscription 을 Paddle 경유 취소. `LicenseClient.getLicense(orgId).subscriptionId()` 로 lookup. 404 / 409 는 idempotent skip |
| device-service | `DeviceLifecycle.UnregisterAllByOwner(B2B_CENTER, orgId)` gRPC | OWNER organization 의 모든 device REVOKED + 풀 unclaim |
| archive-service | (간접) | 기존 `ArchiveOutboxWorker` 가 SENT 처리. cleanup 은 `archive_outbox.status=SENT` 만 확인 |

retry 정책. exponential backoff (5s × 2^n, max 1h), MAX_RETRY=10 회 후 FAILED 마킹.

---

### 2.12 `POST /api/v2/b2b/organizations`

시설 등록. 호출한 b2b_user 가 OWNER 로 자동 멤버십 생성 (`joinedVia=OWNER_INIT`).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/organizations | Auth | no | yes |

**Request Body (v1.0.0, default)** — `CreateOrganizationRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `name` | string | yes | NotBlank |
| `type` | string | yes | `OrganizationType` enum |
| `description` | string | no | 시설 소개 |
| `phoneNumber` | string | no | E.164 |
| `photoUrl` | string | no | URL |
| `country` | string | no | ValidCountryCode (ISO-3166 alpha-2). v1.0.0 default null |
| `postalCode` | string | no | 우편번호 |
| `regionLevel1` | string | no | 시/도 |
| `regionLevel2` | string | no | 구/군 |
| `addressLine1` | string | no | 도로명 주소 |
| `addressLine2` | string | no | 상세 주소 |
| `searchable` | boolean | no | 시설 검색 노출 여부. service 가 null → default true |
| `inviteCodeEnabled` | boolean | no | 초대 코드 발급 가능 여부. service 가 null → default true |
| `approvalRequired` | boolean | no | join request 승인 필요 여부. service 가 null → default true |

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

**Request Body (v1.0.1, header `api-version: 1.0.1`)** — `CreateOrganizationRequestV1_0_1`

v1.0.0 의 모든 필드 + `businessRegistrationNumber` 필수 (0.0.64 신규). country 별 정규식 검증 (`BusinessRegistrationNumberValidator`).

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `name` | string | yes | NotBlank, ≤ 200 |
| `type` | string | yes | `OrganizationType` enum |
| `description` | string | no | ≤ 65535 |
| `phoneNumber` | string | no | E.164 (nullable 허용) |
| `photoUrl` | string | no | ≤ 500 |
| `country` | string | yes | `@NotBlank @ValidCountryCode`, 2자 |
| `postalCode` | string | no | ≤ 20 |
| `regionLevel1` | string | no | ≤ 100 |
| `regionLevel2` | string | no | ≤ 100 |
| `addressLine1` | string | no | ≤ 255 |
| `addressLine2` | string | no | ≤ 255 |
| `businessRegistrationNumber` | string | **yes** | `@NotBlank`, ≤ 40, country 별 정규식 (0.0.64 신규) |
| `searchable` | boolean | no | 검색 노출. service 가 null → default true |
| `inviteCodeEnabled` | boolean | no | 초대 코드 발급. service 가 null → default true |
| `approvalRequired` | boolean | no | 가입 승인. service 가 null → default true |

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
  "businessRegistrationNumber": "123-45-67890",
  "searchable": true,
  "inviteCodeEnabled": true,
  "approvalRequired": true
}
```

`OrganizationType` enum. `GYM | PILATES | YOGA | PT_STUDIO | CROSSFIT | FUNCTIONAL | BOXING | ETC`.

`businessRegistrationNumber` — country 별 형식 (`BusinessRegistrationNumberValidator`). 매칭 우선순위. ① country 전용 정규식 → ② EU VAT → ③ fallback. `country` 는 `@ValidCountryCode` (ISO 3166-1 α-2, 중국 `CN` · 대만 `TW` 포함).

| country | 형식 | 정규식 |
|---|---|---|
| KR | 사업자등록번호 10자리 | `^(\d{3}-?\d{2}-?\d{5})$` |
| US | EIN 9자리 | `^(\d{2}-?\d{7})$` |
| JP | 법인번호 13자리 | `^\d{13}$` |
| CN | 통일사회신용코드 18자리 | `^[0-9A-HJ-NPQRTUWXY]{18}$` |
| TW | 統一編號 8자리 (대만, 신규 — CountryCode.TW 대응) | `^\d{8}$` |
| VN | 사업자번호 10자리 (+선택 `-`3자리 지점) | `^\d{10}(-?\d{3})?$` |
| EU* | VAT | `^[A-Z]{2}\d{8,12}$` |
| 그 외 | fallback (영숫자 + dash, 5~40자) | `^[A-Za-z0-9-]{5,40}$` |

EU* = DE FR IT ES NL BE AT PL SE FI DK IE PT GR CZ HU RO BG HR SI SK EE LV LT MT CY LU.

> **`country` 저장·응답 형식 (carencoPlatformVersion 0.0.84~)** —
> - **입력**: alpha-2 (`@ValidCountryCode`, 2자 strict). 변동 없음.
> - **DB 저장**: ISO 3166-1 **alpha-3**(`KOR`). V18 마이그레이션으로 기존 alpha-2 → alpha-3 백필 (`b2b_organizations.country` VARCHAR(3)).
> - **응답**: `api-version` 별 — **`1.0.0`/`1.0.1`** → alpha-2(`"KR"`, 기존 호환), 신규 **`1.0.2`**(create/get) → alpha-3(`"KOR"`). v1.0.2 는 v1.0.1 과 동일 shape, `address.country` 만 alpha-3.

**Response — 201 (v1.0.0, default)** — `OrganizationResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | organization PK |
| `name` | string | 시설명 |
| `type` | string | `OrganizationType` |
| `description` | string \| null | 시설 소개 |
| `phoneNumber` | string \| null | E.164 |
| `photoUrl` | string \| null | 시설 사진 URL |
| `address` | object | 주소 (country/postalCode/regionLevel1/regionLevel2/addressLine1/addressLine2) |
| `ownerUserId` | string (uuid) | OWNER 유저 |
| `searchable` | boolean | 검색 노출 |
| `inviteCodeEnabled` | boolean | 초대 코드 발급 가능 |
| `approvalRequired` | boolean | 가입 승인 필요 |
| `seatsUsed` | integer | 사용 좌석 |
| `status` | string | `ACTIVE` / `DELETED` |

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

**Response — 201 (v1.0.1, header `api-version: 1.0.1`)** — `OrganizationResponseV1_0_1`

v1.0.0 의 모든 필드 + `businessRegistrationNumber` (0.0.64) + `stats` (0.0.63). 모두 nullable.

| 필드 | 타입 | 설명 |
|---|---|---|
| (v1.0.0 의 모든 필드) | — | 위 v1.0.0 응답 동일 |
| `businessRegistrationNumber` | string \| null | 사업자등록번호 (country 별 정규식) |
| `stats` | object — `{ totalMembers, activeMembers, byRole, suspended }` | 멤버 통계 (LEFT 제외). 생성 직후는 OWNER 1 만 |

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
    "businessRegistrationNumber": "123-45-67890",
    "ownerUserId": "u-9f3e",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 1,
    "status": "ACTIVE",
    "stats": {
      "totalMembers": 1,
      "activeMembers": 1,
      "byRole": { "OWNER": 1, "ADMIN": 0, "TRAINER": 0, "MEMBER": 0 },
      "suspended": 0
    }
  }
}
```

**Errors** (`OrganizationError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 400 | `CMN-400-001` | InvalidInput |
| 400 | `CMN-400-001` | InvalidInput (v1.0.1 의 country 별 정규식 위반 포함 — `OrganizationError.InvalidInput`) |

---

### 2.13 `GET /api/v2/b2b/organizations/{id}`

단건 조회 (공개).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{id} | Public | yes | yes |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | organization PK |

**Response — 200 (v1.0.0, default)** — `OrganizationResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | organization PK |
| `name` | string | 시설명 |
| `type` | string | `OrganizationType` |
| `description` | string \| null | 시설 소개 |
| `phoneNumber` | string \| null | E.164 |
| `photoUrl` | string \| null | 시설 사진 URL |
| `address` | object | 주소 (country/postalCode/regionLevel1/regionLevel2/addressLine1/addressLine2) |
| `ownerUserId` | string (uuid) | OWNER 유저 |
| `searchable` | boolean | 검색 노출 |
| `inviteCodeEnabled` | boolean | 초대 코드 발급 가능 |
| `approvalRequired` | boolean | 가입 승인 필요 |
| `seatsUsed` | integer | 사용 좌석 |
| `status` | string | `ACTIVE` / `DELETED` |

```json
{
  "success": true, "code": "200", "message": "OK",
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
    "seatsUsed": 12,
    "status": "ACTIVE"
  }
}
```

**Response — 200 (v1.0.1, header `api-version: 1.0.1`)** — `OrganizationResponseV1_0_1`

v1.0.0 + `businessRegistrationNumber` (0.0.64) + `stats` (0.0.63).

```json
{
  "success": true, "code": "200", "message": "OK",
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
    "businessRegistrationNumber": "123-45-67890",
    "ownerUserId": "u-9f3e",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 12,
    "status": "ACTIVE",
    "stats": {
      "totalMembers": 14,
      "activeMembers": 12,
      "byRole": { "OWNER": 1, "ADMIN": 2, "TRAINER": 5, "MEMBER": 4 },
      "suspended": 2
    }
  }
}
```

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |

---

### 2.14 `PATCH /api/v2/b2b/organizations/{id}`

부분 수정 (owner 만). JsonNullable 의 3-state semantics.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| PATCH | /api/v2/b2b/organizations/{id} | OWNER | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | organization PK |

**Request Body** — `UpdateOrganizationRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `name` | JsonNullable<string> | no | null 거부 (UpdateOrganizationValidator) |
| `type` | JsonNullable<string> | no | null 거부 |
| `description` | JsonNullable<string> | no | null 로 clear 가능 |
| `phoneNumber` | JsonNullable<string> | no | E.164 |
| `photoUrl` | JsonNullable<string> | no | null 로 clear 가능 |
| `country` | JsonNullable<string> | no | ValidCountryCode |
| `postalCode` | JsonNullable<string> | no | |
| `regionLevel1` | JsonNullable<string> | no | |
| `regionLevel2` | JsonNullable<string> | no | |
| `addressLine1` | JsonNullable<string> | no | |
| `addressLine2` | JsonNullable<string> | no | |
| `searchable` | JsonNullable<boolean> | no | null 거부 |
| `inviteCodeEnabled` | JsonNullable<boolean> | no | null 거부 |
| `approvalRequired` | JsonNullable<boolean> | no | null 거부 |

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

**Response — 200** — `OrganizationResponse` (2.12 schema 참조).

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |
| 403 | `AUTH-403-002` | NotOwner |
| 400 | `CMN-400-002` | FieldCannotBeCleared |

---

### 2.15 `DELETE /api/v2/b2b/organizations/{id}`

soft 삭제 (owner 만). `status=DELETED` + 모든 멤버십 LEFT + 모든 invite code REVOKED cascade. archive outbox 비동기 publish.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| DELETE | /api/v2/b2b/organizations/{id} | OWNER | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | organization PK |

**Response — 204**.

**Errors** (`OrganizationError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | NotFound |
| 403 | `AUTH-403-002` | NotOwner |

---

### 2.16 `GET /api/v2/b2b/organizations/search`

`searchable=true` + `status=ACTIVE` 시설 검색.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/search | Public | yes | no |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `keyword` | string | no | 이름/주소 LIKE 매칭 |
| `type` | `OrganizationType` | no | optional |
| `page` | int | no | 기본 0 |
| `size` | int | no | 기본 20 |

**Response — 200**

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`OrganizationResponse`> | 2.12 schema 의 organization 목록 |
| `page` | integer | 현재 페이지 |
| `size` | integer | 페이지 크기 |
| `totalElements` | integer | 전체 개수 |
| `totalPages` | integer | 전체 페이지 수 |
| `hasNext` | boolean | 다음 페이지 존재 여부 |

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

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/organizations/{orgId}/join-requests | Public (body 의 `memberId` 가 carenco user uuid) | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Request Body** — `MembershipController.JoinRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `memberId` | string (uuid) | yes | carenco user uuid |
| `note` | string | no | 가입 신청 메모 |

```json
{
  "memberId": "carenco-user-uuid",
  "note": "5월 한 달 등록 희망합니다"
}
```

**Response — 201** — `MembershipResponse` (status=PENDING)

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | membership PK |
| `organizationId` | string (uuid) | 소속 시설 |
| `memberType` | string | `CARENCO` / `B2B` |
| `memberId` | string (uuid) | 멤버 user id |
| `role` | string | `OWNER` / `ADMIN` / `TRAINER` / `MEMBER` |
| `status` | string | `PENDING` / `ACTIVE` / `SUSPENDED` / `LEFT` |
| `memberNumber` | string \| null | 회원 번호 |
| `joinedVia` | string | `SEARCH_REQUEST` / `INVITE_CODE` / `DIRECT_INVITE` / `OWNER_INIT` |
| `inviteCodeId` | string \| null | 초대 코드 PK (해당 시) |
| `approvedBy` | string \| null | 승인자 b2bUserId |
| `approvedAt` | string (date-time, UTC) \| null | 승인 시각 |
| `joinedAt` | string (date-time, UTC) \| null | 활성 전환 시각 |
| `leftAt` | string (date-time, UTC) \| null | 탈퇴 시각 |
| `note` | string \| null | 가입 신청 메모 |

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
| 409 | `INV-409-002` | AlreadyMember (`ALREADY_CONNECTED`) |

---

### 2.19 `POST /api/v2/b2b/memberships/{membershipId}/approve`

PENDING → ACTIVE. 좌석 +1 + license guard.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/memberships/{membershipId}/approve | OWNER · ADMIN | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `membershipId` | string (uuid) | membership PK |

**Response — 200** — `MembershipResponse` (2.18 schema 참조)

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

**Errors** (`MembershipFlowError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 409 | `CMN-409-001` | InvalidStateTransition |
| 409 | `INV-409-002` | AlreadyMember (`ALREADY_CONNECTED`) |
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

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/memberships/{membershipId}/reject | OWNER · ADMIN | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `membershipId` | string (uuid) | membership PK |

**Response — 200** — `MembershipResponse` (status=LEFT, leftAt 채워짐, 2.18 schema 참조).

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 409 | `CMN-409-001` | InvalidStateTransition |

---

### 2.21 `POST /api/v2/b2b/memberships/{membershipId}/suspend`

ACTIVE → SUSPENDED.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/memberships/{membershipId}/suspend | OWNER · ADMIN | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `membershipId` | string (uuid) | membership PK |

**Request Body** — `MembershipController.SuspendRequest` (optional)

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `reason` | string | no | 정지 사유 |

```json
{ "reason": "회비 연체 3개월" }
```

**Response — 200** — `MembershipResponse` (status=SUSPENDED, 2.18 schema 참조).

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 409 | `CMN-409-001` | InvalidStateTransition |

---

### 2.22 `POST /api/v2/b2b/memberships/{membershipId}/leave`

본인 탈퇴. body 의 `requesterId` 와 멤버십의 `memberId` 가 일치해야 함. 좌석 −1 + archive outbox publish.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/memberships/{membershipId}/leave | Public (body 자기검증) | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `membershipId` | string (uuid) | membership PK |

**Request Body** — `MembershipController.LeaveRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `requesterId` | string (uuid) | yes | 본인 검증용 — 멤버십의 `memberId` 와 일치 필요 |

```json
{ "requesterId": "carenco-user-uuid" }
```

**Response — 200** — `MembershipResponse` (status=LEFT, 2.18 schema 참조).

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | MembershipNotFound |
| 409 | `CMN-409-001` | InvalidStateTransition |

---

### 2.23 `POST /api/v2/b2b/organizations/{orgId}/staff`

OWNER/ADMIN 이 기존 b2b_user 를 staff 로 지정. role 은 ADMIN 또는 TRAINER.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/organizations/{orgId}/staff | OWNER · ADMIN | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Request Body** — `MembershipController.AppointStaffRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `b2bUserId` | string (uuid) | yes | staff 로 지정할 기존 b2b_user |
| `role` | string | yes | `ADMIN` 또는 `TRAINER` (OWNER 부여 불가) |
| `memberNumber` | string | no | 회원/스태프 번호 |

```json
{
  "b2bUserId": "u-staff-1",
  "role": "TRAINER",
  "memberNumber": "T260608-001"
}
```

**Response — 201** — `MembershipResponse` (2.18 schema 참조)

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

**Errors** (`MembershipFlowError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |
| 400 | `CMN-400-001` | InvalidRoleAssignment (OWNER 부여 시도 등) |
| 409 | `INV-409-002` | AlreadyMember (`ALREADY_CONNECTED`) |

---

### 2.24 `GET /api/v2/b2b/organizations/{orgId}/members`

회원 목록 (membership + user-service 정보 + measurement 요약). PDF 흐름 4.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/members | OWNER · ADMIN · TRAINER (user/measure-service 장애 시 user/measurement 필드는 null — membership 정보만 노출, graceful degradation) | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `query` | string | no | 이름 부분일치 (user-service `SearchUsers` 4가지 결합 매칭 — firstName+lastName / lastName+firstName / firstName / lastName, 공백 무시) |
| `include_nickname` | boolean | no | 기본 `false`. true 시 `query` 매칭에 nickname 포함 |
| `sort_by` | `NAME` / `REGISTERED_AT` / `LATEST_MEASUREMENT` | no | 기본 `REGISTERED_AT` |
| `status` | `MembershipStatus` | no | 기본 `ACTIVE` |
| `page` | int | no | 기본 0 |
| `size` | int | no | 기본 20 (max 100) |

> 0.0.52 에서 phone_number 매칭은 user-service 측에서 제거됨 — 이름/nickname 만 검색 가능.

**Response — 200** — `MemberListResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<object> | 회원 항목 (아래 schema) |
| `items[].membershipId` | string (uuid) | membership PK |
| `items[].memberId` | string (uuid) | carenco user PK |
| `items[].memberNumber` | string \| null | 회원 번호 |
| `items[].role` | string | `MEMBER` / `TRAINER` / `ADMIN` / `OWNER` |
| `items[].status` | string | `MembershipStatus` |
| `items[].joinedAt` | string (date-time, UTC) | 가입 시각 |
| `items[].name` | string \| null | user-service 에서 가져온 표시 이름 |
| `items[].nickname` | string \| null | nickname |
| `items[].email` | string \| null | |
| `items[].phoneNumber` | string \| null | |
| `items[].photoUrl` | string \| null | |
| `items[].birthdate` | string (date) \| null | |
| `items[].gender` | string \| null | `MALE` / `FEMALE` 등 |
| `items[].countryCode` | string \| null | |
| `items[].height` | number \| null | cm |
| `items[].totalMeasurements` | integer \| null | 누적 측정 횟수 |
| `items[].currentMonthMeasurements` | integer \| null | 이번 달 측정 횟수 |
| `items[].lastMeasuredAt` | string (date-time, UTC) \| null | 최근 측정 시각 |
| `items[].lastWeight` | number \| null | 최근 체중 |
| `items[].lastBodyScore` | number \| null | 최근 body score |
| `totalCount` | integer | 전체 회원 수 |
| `totalReports` | integer | 전체 측정 리포트 수 |
| `hasNext` | boolean | 다음 페이지 |

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

**Errors** (`MemberListError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | RoleRequired |
| 404 | `CMN-404-001` | OrganizationNotFound |

---

### 2.25 `POST /api/v2/b2b/organizations/{orgId}/invite-codes`

초대 코드 발급. 헷갈리는 문자 (0/O, 1/l/I) 회피 generator. TTL 48h.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/organizations/{orgId}/invite-codes | OWNER · ADMIN | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Request Body** — `IssueInviteCodeRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `description` | string | no | 초대 코드 설명 |
| `codeLength` | integer | no | 6~12 (기본 8) |

```json
{
  "description": "5월 신규 회원용",
  "codeLength": 8
}
```

`codeLength` 6~12 (기본 8).

**Response — 201** — `InviteCodeResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | invite code PK |
| `organizationId` | string (uuid) | 발급 시설 |
| `code` | string | 초대 코드 문자열 |
| `description` | string \| null | 설명 |
| `expiresAt` | string (date-time, UTC) | 만료 시각 (TTL 48h) |
| `usedAt` | string (date-time, UTC) \| null | 사용 시각 |
| `status` | string | `ACTIVE` / `USED` / `REVOKED` / `EXPIRED` |
| `createdBy` | string (uuid) | 발급자 b2bUserId |

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

### 2.26 `GET /api/v2/b2b/organizations/{orgId}/invite-codes`

ACTIVE 코드 목록.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/invite-codes | Auth | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Response — 200**

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`InviteCodeResponse`> | 2.25 schema 의 invite code 목록 |

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

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/invite-codes/{inviteCodeId}/revoke | OWNER · ADMIN | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `inviteCodeId` | string (uuid) | invite code PK |

**Response — 200** — `InviteCodeResponse` (status=REVOKED, 2.25 schema 참조).

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | InviteCodeNotFound |
| 403 | `AUTH-403-002` | NotOwnerOrAdmin |

---

### 2.28 `POST /api/v2/b2b/invite-codes/redeem`

Carenco user 가 코드 입력하여 가입.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/invite-codes/redeem | Public (body 의 `memberId` 가 carenco userId) | no | no |

**Request Body** — `InviteCodeController.RedeemRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `code` | string | yes | NotBlank, 발급된 invite code |
| `memberId` | string (uuid) | yes | carenco user uuid |

```json
{
  "code": "Hk7mPqR3",
  "memberId": "carenco-user-uuid"
}
```

**Response — 200**

| 필드 | 타입 | 설명 |
|---|---|---|
| `inviteCode` | `InviteCodeResponse` | 2.25 schema (status=USED, usedAt 채워짐) |
| `membershipId` | string (uuid) | 생성된 membership PK |
| `organizationId` | string (uuid) | 가입 시설 |
| `status` | string | membership status (ACTIVE) |

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
| 409 | `INV-409-002` | AlreadyMember (`ALREADY_CONNECTED`) |
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

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/organizations/{orgId}/devices | OWNER · ADMIN (license state ACTIVE/GRACE 만) | no | yes |

**Headers**

| 헤더 | 값 | 필수 | 설명 |
|---|---|---|---|
| `api-version` | `1.0.0` (default) / `1.0.1` | no | `1.0.1` 시 `deviceType` + `deviceNumber` 포함 응답 |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Request Body** — `RegisterDeviceRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `serialNumber` | string | yes | NotBlank, 하드웨어 시리얼 |
| `alias` | string | no | ≤ 255. 표시명 (nullable / blank 허용) |

```json
{
  "serialNumber": "INBODY-770-2024-0123",
  "alias": "1번 InBody"
}
```

**Response — 201 (v1.0.0, default)** — `DeviceResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `deviceId` | string (uuid) | device PK |
| `organizationId` | string (uuid) | 소유 시설 |
| `serialNumber` | string | 하드웨어 시리얼 |
| `alias` | string | 표시명 |
| `status` | string | `ACTIVE` / `INACTIVE` / `REVOKED` |
| `registeredAt` | string (date-time, UTC) | 등록 시각 |
| `registeredBy` | string (uuid) | 등록자 |
| `deactivatedAt` | string (date-time, UTC) \| null | 비활성화 시각 |
| `deactivatedBy` | string (uuid) \| null | 비활성화 자 |
| `deactivationReason` | string \| null | 비활성화 사유 |
| `lastUsedAt` | string (date-time, UTC) \| null | 최근 사용 시각 |
| `batteryLevel` | integer \| null | 배터리 % (0.0.61+) |
| `batteryReportedAt` | string (date-time, UTC) \| null | 배터리 보고 시각 |

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
    "lastUsedAt": null,
    "batteryLevel": null,
    "batteryReportedAt": null
  }
}
```

**Response — 201 (v1.0.1, header `api-version: 1.0.1`)** — `DeviceResponseV1_0_1`

v1.0.0 의 모든 필드 + `deviceType` (0.0.63) + `deviceNumber` (0.0.73).

| 필드 | 타입 | 설명 |
|---|---|---|
| (v1.0.0 의 모든 필드) | — | 상기 schema 참조 |
| `deviceType` | string | `DeviceType` enum (e.g. `SCALE2`) |
| `deviceNumber` | string \| null | 풀에서 부여된 디바이스 번호 (e.g. `FS2-001`) |

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
    "lastUsedAt": null,
    "batteryLevel": null,
    "batteryReportedAt": null,
    "deviceType": "SCALE2",
    "deviceNumber": "FS2-001"
  }
}
```

**Errors** (`DeviceError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | OrgRoleRequired |
| 403 | `LIC-403-001` | LicenseDenied (EXPIRED/SUSPENDED) |

---

### 2.30 `GET /api/v2/b2b/organizations/{orgId}/devices`

device 목록.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/devices | Auth (멤버) | yes | yes |

**Headers**

| 헤더 | 값 | 필수 | 설명 |
|---|---|---|---|
| `api-version` | `1.0.0` (default) / `1.0.1` | no | `1.0.1` 시 각 item 에 `deviceType` + `deviceNumber` 포함 |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `keyword` | string | no | serialNumber / alias 부분일치 |
| `status` | `DeviceStatus` (`ACTIVE` / `INACTIVE`) | no | 필터 |
| `sortBy` | string | no | `name` / `registered_at` (default DESC) / `last_used` / **`battery`** (0.0.61, NULLS LAST) |

**Response — 200 (v1.0.0, default)**

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`DeviceResponse`> | 2.29 v1.0.0 schema 의 device 목록 |

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

**Response — 200 (v1.0.1, header `api-version: 1.0.1`)**

각 item 에 `deviceType` (0.0.63) + `deviceNumber` (0.0.73) 추가. 그 외 동일.

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`DeviceResponseV1_0_1`> | 2.29 v1.0.1 schema 의 device 목록 |

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
        "batteryReportedAt": "2026-05-13T11:20:00Z",
        "deviceType": "SCALE2",
        "deviceNumber": "FS2-001"
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

### 2.31 `GET /api/v2/b2b/organizations/{orgId}/devices/{deviceId}`

device 단건.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/devices/{deviceId} | Auth (멤버) | yes | yes |

**Headers**

| 헤더 | 값 | 필수 | 설명 |
|---|---|---|---|
| `api-version` | `1.0.0` (default) / `1.0.1` | no | `1.0.1` 시 `deviceType` + `deviceNumber` 포함 |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |
| `deviceId` | string (uuid) | device PK |

**Response — 200 (v1.0.0, default)** — `DeviceResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `deviceId` | string (uuid) | device PK |
| `organizationId` | string (uuid) | 소유 시설 |
| `serialNumber` | string | 하드웨어 시리얼 |
| `alias` | string | 표시명 |
| `status` | string | `ACTIVE` / `INACTIVE` / `REVOKED` |
| `registeredAt` | string (date-time, UTC) | 등록 시각 |
| `registeredBy` | string (uuid) | 등록자 |
| `deactivatedAt` | string (date-time, UTC) \| null | 비활성화 시각 |
| `deactivatedBy` | string (uuid) \| null | 비활성화 자 |
| `deactivationReason` | string \| null | 비활성화 사유 |
| `lastUsedAt` | string (date-time, UTC) \| null | 최근 사용 시각 |
| `batteryLevel` | integer \| null | 배터리 % (0.0.61+) |
| `batteryReportedAt` | string (date-time, UTC) \| null | 배터리 보고 시각 |

```json
{
  "success": true, "code": "200", "message": "OK",
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
    "lastUsedAt": "2026-05-13T11:20:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-05-13T11:20:00Z"
  }
}
```

**Response — 200 (v1.0.1, header `api-version: 1.0.1`)** — `DeviceResponseV1_0_1`

v1.0.0 의 모든 필드 + `deviceType` (0.0.63) + `deviceNumber` (0.0.73).

| 필드 | 타입 | 설명 |
|---|---|---|
| (v1.0.0 의 모든 필드) | — | 상기 schema 참조 |
| `deviceType` | string | `DeviceType` enum (e.g. `SCALE2`) |
| `deviceNumber` | string \| null | 풀에서 부여된 디바이스 번호 (e.g. `FS2-001`) |

```json
{
  "success": true, "code": "200", "message": "OK",
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
    "lastUsedAt": "2026-05-13T11:20:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-05-13T11:20:00Z",
    "deviceType": "SCALE2",
    "deviceNumber": "FS2-001"
  }
}
```

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | DeviceNotInOrg |

---

### 2.32 `PATCH /api/v2/b2b/organizations/{orgId}/devices/{deviceId}`

alias 만 변경 가능. device-service gRPC 가 `alias=null` 을 "변경 없음" 으로 해석 → `null` 거부 (UpdateDeviceValidator).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| PATCH | /api/v2/b2b/organizations/{orgId}/devices/{deviceId} | OWNER · ADMIN | yes | yes |

**Headers**

| 헤더 | 값 | 필수 | 설명 |
|---|---|---|---|
| `api-version` | `1.0.0` (default) / `1.0.1` | no | `1.0.1` 시 `deviceType` + `deviceNumber` 포함 응답 |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |
| `deviceId` | string (uuid) | device PK |

**Request Body** — `UpdateDeviceRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `alias` | JsonNullable<string> | no | null 거부 (UpdateDeviceValidator) |

```json
{ "alias": "1번 InBody (대기실)" }
```

**Response — 200 (v1.0.0, default)** — `DeviceResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `deviceId` | string (uuid) | device PK |
| `organizationId` | string (uuid) | 소유 시설 |
| `serialNumber` | string | 하드웨어 시리얼 |
| `alias` | string | 표시명 (변경된 값) |
| `status` | string | `ACTIVE` / `INACTIVE` / `REVOKED` |
| `registeredAt` | string (date-time, UTC) | 등록 시각 |
| `registeredBy` | string (uuid) | 등록자 |
| `deactivatedAt` | string (date-time, UTC) \| null | 비활성화 시각 |
| `deactivatedBy` | string (uuid) \| null | 비활성화 자 |
| `deactivationReason` | string \| null | 비활성화 사유 |
| `lastUsedAt` | string (date-time, UTC) \| null | 최근 사용 시각 |
| `batteryLevel` | integer \| null | 배터리 % |
| `batteryReportedAt` | string (date-time, UTC) \| null | 배터리 보고 시각 |

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "deviceId": "dev-c3d4",
    "organizationId": "org-A",
    "serialNumber": "INBODY-770-2024-0123",
    "alias": "1번 InBody (대기실)",
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
}
```

**Response — 200 (v1.0.1, header `api-version: 1.0.1`)** — `DeviceResponseV1_0_1`

v1.0.0 의 모든 필드 + `deviceType` + `deviceNumber`.

| 필드 | 타입 | 설명 |
|---|---|---|
| (v1.0.0 의 모든 필드) | — | 상기 schema 참조 |
| `deviceType` | string | `DeviceType` enum (e.g. `SCALE2`) |
| `deviceNumber` | string \| null | 풀에서 부여된 디바이스 번호 |

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "deviceId": "dev-c3d4",
    "organizationId": "org-A",
    "serialNumber": "INBODY-770-2024-0123",
    "alias": "1번 InBody (대기실)",
    "status": "ACTIVE",
    "registeredAt": "2026-05-13T10:00:00Z",
    "registeredBy": "u-9f3e",
    "deactivatedAt": null,
    "deactivatedBy": null,
    "deactivationReason": null,
    "lastUsedAt": "2026-05-13T11:20:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-05-13T11:20:00Z",
    "deviceType": "SCALE2",
    "deviceNumber": "FS2-001"
  }
}
```

**Errors** (`DeviceError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | OrgRoleRequired |
| 404 | `CMN-404-001` | DeviceNotInOrg |
| 400 | `CMN-400-002` | FieldCannotBeCleared (alias 를 null 로 시도) |

---

### 2.33 `POST /api/v2/b2b/organizations/{orgId}/devices/{deviceId}/deactivate`

device 비활성화 (status=INACTIVE).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/organizations/{orgId}/devices/{deviceId}/deactivate | OWNER · ADMIN | yes | yes |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |
| `deviceId` | string (uuid) | device PK |

**Request Body** — `DeactivateDeviceRequest` (optional)

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `reason` | string | no | 비활성화 사유 |

```json
{ "reason": "수리 입고" }
```

**Response — 200** — `DeviceResponse` (status=INACTIVE, deactivatedAt/deactivatedBy 채워짐, 2.29 schema 참조). 헤더 `api-version: 1.0.1` 시 v1.0.1 응답 (`deviceType` + `deviceNumber` 추가).

**Errors** (`DeviceError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | OrgRoleRequired |
| 404 | `CMN-404-001` | DeviceNotInOrg |

---

### 2.33.1 `GET .../devices/preview` (0.0.61 신규)

시리얼만으로 풀 조회 — 등록 전 미리보기. claim 안 함, write 없음. UI 의 "기기 등록" 흐름에서 serial 입력 후 alias 입력 전 호출.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/devices/preview | OWNER · ADMIN | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `serial` | string | yes | 풀에서 찾을 `hardware_serial` |

**Response — 200** — `PreviewDeviceResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `found` | boolean | 풀에 있고 provisional 아닌 경우 true |
| `alreadyClaimed` | boolean | 이미 다른 owner 가 등록함 |
| `hardwareSerial` | string | found=true 일 때만 |
| `deviceType` | string | `DeviceType` 문자열 (e.g. `SCALE2`) |
| `firmwareVersion` | string \| null | 풀 시점 펌웨어 |

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `OrgRoleRequired` | caller 가 OWNER/ADMIN 아님 |

---

### 2.33.2 `DELETE .../devices/{deviceId}` (0.0.61 신규)

영구 삭제 — `status=REVOKED` + 풀 row unclaim. 일시 정지 ([2.33](#233-post-apiv2b2borganizationsorgiddevicesdeviceiddeactivate)) 와 다름. 같은 `(mac, serial)` 가 다른 organization 에 재등록 가능해짐.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| DELETE | /api/v2/b2b/organizations/{orgId}/devices/{deviceId} | OWNER · ADMIN | yes | yes |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |
| `deviceId` | string (uuid) | device PK |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `reason` | string | no | audit (예. `lost` / `discarded` / `replaced`) |

**Response — 200** — `DeviceResponse` (status=REVOKED, deactivatedAt/deactivatedBy/deactivationReason 채워짐, batteryLevel 마지막 값 유지, 2.29 schema 참조). 헤더 `api-version: 1.0.1` 시 v1.0.1 응답 (`deviceType` + `deviceNumber` 추가).

**Idempotent**. 이미 REVOKED 인 device 에 다시 호출해도 200 + 같은 row 반환.

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `OrgRoleRequired` | caller 권한 부족 |
| 404 | `DeviceNotInOrg` | device 가 해당 organization 소유가 아님 |

---

### 2.34 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements`

회원 측정 이력 — measure-service gRPC.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements | OWNER · ADMIN · TRAINER (호출자 권한 + `memberId` 가 organization 의 ACTIVE 멤버) | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |
| `memberId` | string (uuid) | 회원 user PK |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `from` | string (ISO date) | no | 측정일 ≥ from |
| `to` | string (ISO date) | no | 측정일 ≤ to |
| `page` | int | no | 기본 0 |
| `size` | int | no | 기본 20 |

**Response — 200** — `MeasurementClient.Page`

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`MeasurementInfo`> | 측정 항목 (아래 schema) |
| `items[].recordId` | string (uuid) | record PK |
| `items[].userId` | string (uuid) | 측정 user |
| `items[].measuredAt` | string (date-time, UTC) | 측정 시각 |
| `items[].weight` | number | 체중 (kg) |
| `items[].height` | number | 키 (cm) |
| `items[].age` | integer | 나이 |
| `items[].bodyScore` | number | body score |
| `items[].predictedAge` | number | 예측 신체 나이 |
| `items[].deviceId` | string (uuid) | 측정 device |
| `totalCount` | integer | 전체 측정 수 |
| `hasNext` | boolean | 다음 페이지 |

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

### 2.35 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/{recordId}`

단건 측정.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/{recordId} | OWNER · ADMIN · TRAINER | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |
| `memberId` | string (uuid) | 회원 user PK |
| `recordId` | string (uuid) | measurement record PK |

**Response — 200** — `MeasurementInfo` (2.34 의 한 item 과 동일).

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | RecordNotFound |

---

### 2.36 `GET /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/summary`

denormalized 요약 (measure-service `user_measurement_summary`).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/organizations/{orgId}/members/{memberId}/measurements/summary | OWNER · ADMIN · TRAINER | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `orgId` | string (uuid) | organization PK |
| `memberId` | string (uuid) | 회원 user PK |

**Response — 200** — `MeasurementSummary`

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string (uuid) | 회원 user |
| `totalRecords` | integer | 누적 측정 수 |
| `currentMonthRecords` | integer | 이번 달 측정 수 |
| `lastMeasuredAt` | string (date-time, UTC) \| null | 최근 측정 시각 |
| `lastBodyScore` | number \| null | 최근 body score |
| `lastWeight` | number \| null | 최근 체중 |
| `lastPredictedAge` | number \| null | 최근 예측 신체 나이 |

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

**Errors** (`MeasurementError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 403 | `AUTH-403-002` | RoleRequired |
| 404 | `CMN-404-001` | TargetNotActiveMember |

---

### 2.37 `POST /api/v2/b2b/feedbacks/memberships/{membershipId}`

피드백 작성 — TRAINER/ADMIN/OWNER 가 회원에 코멘트.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/feedbacks/memberships/{membershipId} | OWNER · ADMIN · TRAINER | no | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `membershipId` | string (uuid) | 회원 membership PK |

**Request Body** — `CreateFeedbackRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `body` | string | yes | NotBlank, 피드백 본문 |
| `visibility` | string | yes | `MEMBER` (회원 앱 노출) / `INTERNAL` (스태프만) |
| `measurementRecordId` | string (uuid) | no | 특정 측정과 연관 시 |

```json
{
  "body": "어깨 가동범위가 좋아졌습니다. 다음 주는 하체 위주로.",
  "visibility": "MEMBER",
  "measurementRecordId": "r-9e8d"
}
```

`visibility`. `MEMBER` (회원 앱 노출) / `INTERNAL` (스태프만). `measurementRecordId` 선택 — 특정 측정과 연관 시.

**Response — 201** — `FeedbackResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | feedback PK |
| `organizationId` | string (uuid) | 시설 |
| `memberMembershipId` | string (uuid) | 회원 membership |
| `authorUserId` | string (uuid) | 작성자 b2bUserId |
| `measurementRecordId` | string (uuid) \| null | 연관 측정 record |
| `body` | string | 본문 |
| `visibility` | string | `MEMBER` / `INTERNAL` |
| `createdAt` | string (date-time, UTC) | 작성 시각 |
| `editedAt` | string (date-time, UTC) \| null | 수정 시각 |

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

### 2.38 `PATCH /api/v2/b2b/feedbacks/{feedbackId}`

본문 수정 (작성자 본인만).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| PATCH | /api/v2/b2b/feedbacks/{feedbackId} | Author only | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `feedbackId` | string (uuid) | feedback PK |

**Request Body** — `UpdateFeedbackRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `body` | string | yes | NotBlank, 새 본문 |

```json
{ "body": "오타 수정: 다음 주는 하체+코어 위주." }
```

**Response — 200** — `FeedbackResponse` (editedAt 채워짐, 2.37 schema 참조).

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | FeedbackNotFound |
| 403 | `AUTH-403-002` | NotAuthor |

---

### 2.39 `DELETE /api/v2/b2b/feedbacks/{feedbackId}`

soft 삭제 (deletedAt 채움).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| DELETE | /api/v2/b2b/feedbacks/{feedbackId} | Author only | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `feedbackId` | string (uuid) | feedback PK |

**Response — 200** — `FeedbackResponse` (deletedAt 채워진 상태로 echo, 2.37 schema 참조).

**Errors** (`FeedbackError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | FeedbackNotFound |
| 403 | `AUTH-403-002` | NotAuthor |

---

### 2.40 `GET /api/v2/b2b/feedbacks/memberships/{membershipId}`

회원 피드백 목록.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/feedbacks/memberships/{membershipId} | Auth | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `membershipId` | string (uuid) | 회원 membership PK |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `page` | int | no | 기본 0 |
| `size` | int | no | 기본 20 |

**Response — 200**

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`FeedbackResponse`> | 2.37 schema 의 feedback 목록 |
| `page` | integer | 현재 페이지 |
| `totalElements` | integer | 전체 개수 |
| `hasNext` | boolean | 다음 페이지 |

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

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/feedbacks/measurements/{recordId} | Auth | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `recordId` | string (uuid) | measurement record PK |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `page` | int | no | 기본 0 |
| `size` | int | no | 기본 20 |

**Response — 200**

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`FeedbackResponse`> | 2.37 schema 의 feedback 목록 |
| `page` | integer | 현재 페이지 |
| `totalElements` | integer | 전체 개수 |
| `hasNext` | boolean | 다음 페이지 |

```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "items": [
      { "id": "f-1234", "organizationId": "org-A",
        "memberMembershipId": "m-7c1d", "authorUserId": "u-staff-1",
        "measurementRecordId": "r-9e8d",
        "body": "어깨 가동범위가 좋아졌습니다. 다음 주는 하체 위주로.",
        "visibility": "MEMBER",
        "createdAt": "2026-05-13T12:00:00Z", "editedAt": null }
    ],
    "page": 0,
    "totalElements": 1,
    "hasNext": false
  }
}
```

---

### 2.42 `GET /api/v2/b2b/license-summary`

본인 가입/소유 organization 들의 license + seat 요약 (한 번에).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/license-summary | Auth | yes | no |

**Response — 200**

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<object> | organization 별 license 요약 (아래 schema) |
| `items[].organizationId` | string (uuid) | 시설 PK |
| `items[].organizationName` | string | 시설명 |
| `items[].state` | string | `ACTIVE` / `GRACE` / `EXPIRED` / `SUSPENDED` / `NONE` |
| `items[].planCode` | string \| null | plan code (e.g. `PILATES_BASIC`) |
| `items[].planSeats` | integer | 플랜 좌석 한도 |
| `items[].seatsUsed` | integer | 사용 좌석 |
| `items[].seatsRemaining` | integer | 남은 좌석 |
| `items[].startedAt` | string (date-time, UTC) \| null | license 시작 |
| `items[].expiresAt` | string (date-time, UTC) \| null | 만료 시각 |
| `items[].graceRemainingDays` | integer | grace period 잔여 일수 |
| `items[].graceRemainingSeconds` | integer | grace period 잔여 초 |
| `items[].subscriptionId` | string \| null | Paddle subscription id |

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

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/license-summary/{organizationId} | Auth (해당 시설 멤버만) | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `organizationId` | string (uuid) | organization PK |

**Response — 200** — `LicenseSummaryResponse` (2.42 의 item 과 동일).

**Errors** (`LicenseSummaryError`)

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | `CMN-404-001` | OrganizationNotFound |
| 403 | `AUTH-403-002` | NotMember |

---

### 2.44 `POST /api/v2/b2b/billing/checkout-init`

결제 시작 진입점. b2b 가 권한 검증 후 payment-service 위임.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/billing/checkout-init | OWNER | no | no |

**Request Body** — `CheckoutInitRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `organizationId` | string (uuid) | yes | ValidUuid, 결제 대상 시설 |
| `priceId` | string | yes | NotBlank, Paddle price id |

```json
{
  "organizationId": "org-A",
  "priceId": "pri_01h5xxx"
}
```

**Response — 200** — `PaymentCheckoutLinkResponse` echo

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<object> | line items (priceId/quantity) |
| `items[].priceId` | string | Paddle price id |
| `items[].quantity` | integer | 수량 |
| `customer` | object | 결제자 정보 (email/name) |
| `customer.email` | string | owner email |
| `customer.name` | string | owner 표시명 |
| `customData` | object | Paddle custom_data (organization_id 포함) |
| `customData.organization_id` | string | 시설 PK |
| `plan` | object | plan 정보 |
| `plan.planCode` | string | plan code |
| `plan.planSeats` | integer | 좌석 수 |
| `plan.displayName` | string | 표시명 |
| `plan.amount` | integer | 금액 (최소 단위) |
| `plan.currency` | string | 통화 (e.g. `KRW`) |
| `paddleCustomerId` | string | Paddle customer id |

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

### 2.45 `POST /api/v2/b2b/billing/plan-change`

기존 구독의 plan 변경 (price 변경). b2b 가 OWNER 검증 후 payment-service REST 위임. Paddle PATCH 즉시 반영 + 후속 webhook 으로 DB 정합 확정.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| POST | /api/v2/b2b/billing/plan-change | OWNER | no | no |

**Request Body** — `ChangePlanRequest`

| 필드 | 타입 | 필수 | 검증 / 설명 |
|---|---|---|---|
| `organizationId` | string (uuid) | yes | ValidUuid |
| `newPriceId` | string | yes | NotBlank, 새 Paddle price id |

```json
{
  "organizationId": "org-A",
  "newPriceId": "pri_01h6_pro"
}
```

**Response — 200** — `PaymentPlanChangeResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `paddleSubscriptionId` | string | Paddle subscription id |
| `newPriceId` | string | 변경된 price id |
| `newPlanCode` | string | 새 plan code |
| `newPlanSeats` | integer | 새 좌석 수 |
| `newBillingInterval` | string | `month` / `year` 등 |
| `prorationMode` | string | `prorated_immediately` 등 |
| `status` | string | subscription status (e.g. `active`) |

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

**Errors** (`BillingError`)

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

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/billing/organizations/{organizationId}/transactions | OWNER | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `organizationId` | string (uuid) | organization PK |

**Response — 200** — `List<PaymentTransactionView>`

`data` 는 transaction 배열. 각 transaction schema.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | b2b-service 내부 PK |
| `paddleTransactionId` | string | Paddle transaction id |
| `customerId` | string | Paddle customer id |
| `subscriptionId` | string \| null | Paddle subscription id |
| `status` | string | `completed` / `billed` / `paid` / `canceled` 등 |
| `currencyCode` | string | ISO 4217 |
| `subtotal` | integer | 세전 (최소 단위) |
| `discountTotal` | integer | 할인 |
| `taxTotal` | integer | 세금 |
| `totalAmount` | integer | 총액 |
| `billingPeriodStartsAt` | string (date-time, UTC) \| null | 청구 시작 |
| `billingPeriodEndsAt` | string (date-time, UTC) \| null | 청구 종료 |
| `billingCountryCode` | string \| null | 청구 국가 |
| `billingPostalCode` | string \| null | 우편번호 |
| `billingRegion` | string \| null | 시/도 |
| `billingCity` | string \| null | 구/군 |
| `billingAddressLine` | string \| null | 도로명 |
| `items` | array<object> | line items (id/paddleItemId/priceId/productId/productName/productDescription/quantity/unitPrice/subtotal/discountAmount/taxAmount/total/taxRate) |
| `payments` | array<object> | 결제 시도 (id/amount/status/paymentType/cardType/cardLast4/cardExpiryMonth/cardExpiryYear/cardholderName/capturedAt) |
| `createdAt` | string (date-time, UTC) | 생성 시각 |

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

**Errors** (`BillingError`)

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

---

### 2.47 `GET /api/v2/b2b/availability`

b2b user 가입 전 email 가용성 검사. user-service `/api/v2/availability` 와 같은 shape, b2b namespace.

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/availability | Public | yes | no |

**Query Parameters**

| 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `field` | string | yes | `DuplicationField` enum. 현재 `EMAIL` 만 지원 |
| `value` | string | yes | 검사할 값 |

**Response — 200** — `AvailabilityResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `field` | string | 검사 대상 (`EMAIL`) |
| `value` | string | 검사한 값 |
| `available` | boolean | `true` = 사용 가능 (중복 없음), `false` = 이미 점유 |

```json
{
  "data": { "field": "EMAIL", "value": "owner@example.com", "available": true },
  "success": true,
  "timestamp": "2026-06-10T09:12:36Z"
}
```

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 400 | `CMN-400-002` | `field` 가 `DuplicationField` enum 외 값 (`@ValidEnum`) |

---

### 2.48 `GET /api/v2/b2b/b2c-members/{carencoUserId}/organizations`

B2C (carenco) 회원이 가입한 b2b 시설 목록. b2c app 이 user-service jwt 로 호출 (gateway 검증 가정).

| Method | Path | 권한 | Idempotent | Versioned |
|---|---|---|---|---|
| GET | /api/v2/b2b/b2c-members/{carencoUserId}/organizations | Public | yes | no |

**Path Parameters**

| 이름 | 타입 | 설명 |
|---|---|---|
| `carencoUserId` | string (uuid) | carenco (B2C) 회원 id |

**Response — 200** — `B2cMemberOrganizationView[]`

`memberType=CARENCO` + `memberId=carencoUserId` 인 멤버십 전부 (LEFT 이력 포함). `joinedAt` DESC.

| 필드 | 타입 | 설명 |
|---|---|---|
| `organizationId` | string (uuid) | 시설 id |
| `name` | string | 시설명 |
| `type` | string | `OrganizationType` |
| `photoUrl` | string \| null | 시설 사진 |
| `role` | string | 보통 `MEMBER` |
| `status` | string | `ACTIVE` / `PENDING` / `SUSPENDED` / `LEFT` |
| `memberNumber` | string \| null | 시설 회원번호 (예. `M260609-008`) |
| `joinedAt` | string (date-time) \| null | 가입 시각 |
| `leftAt` | string (date-time) \| null | 탈퇴 시각 (LEFT 일 때) |

```json
{
  "data": [
    {
      "organizationId": "c07e9099-39bc-4a5d-b559-5f90507fdd9d",
      "name": "케어엔코", "type": "ETC", "photoUrl": null,
      "role": "MEMBER", "status": "ACTIVE", "memberNumber": "M260609-008",
      "joinedAt": "2026-06-09T08:58:26Z", "leftAt": null
    }
  ],
  "success": true,
  "timestamp": "2026-06-10T09:13:06Z"
}
```

이용권 상세 (구독 기간 / 갱신 취소 / 공유자) 는 user-service 의
`GET /api/v2/users/{userId}/b2b-centers/{organizationId}/license-detail` 가 b2b gRPC 합성으로 제공.
