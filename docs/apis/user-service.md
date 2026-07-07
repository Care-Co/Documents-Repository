# user-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/user-service`
Updated: 2026-07-07

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value | Notes |
|---|---|---|
| `api-version` | `1.0.0` | Required on every versioned endpoint. |
| `Authorization` | `Bearer <jwt>` | Required on protected endpoints. |
| `Content-Type` | `application/json` | For request bodies. |

**Security schemes**

| Scheme | Description |
|---|---|
| `permitAll` | No authentication. |
| `userBearer` | `Bearer` JWT with `ROLE_USER` (or `ROLE_GUEST` for terms-consent). Resource-owner check via `@tokenUtils.isResourceOwner(#userId)` is enforced where applicable. |
| `adminBearer` | `Bearer` JWT with `ROLE_ADMIN`. |

**Common response envelope** ([`CncResponse`](#cncresponse))

```json
{
  "success": true,
  "data": {},
  "token": null,
  "error": null,
  "timestamp": "2026-04-27T08:00:00Z"
}
```

**Error envelope** ([`ErrorResponse`](#errorresponse))

```json
{
  "success": false,
  "code": "CMN-400-001",
  "message": "Invalid request parameter. Field: email, Value: ...",
  "timestamp": "2026-04-27T08:00:00Z"
}
```

| Code | HTTP | Cause |
|---|---|---|
| `CMN-400-001` | 400 | Validation failure |
| `CMN-401-001` | 401 | Missing/invalid token |
| `CMN-403-001` | 403 | Resource owner / role mismatch |
| `CMN-404-001` | 404 | User not found |
| `USR-409-001` | 409 | Duplicate (email/nickname) |

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning). 아래 "제공 버전" 중 하나를 보낸다. `—` 는 unversioned(헤더 무시).

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| GET | /api/admin/audit/users/{userId} | — | unversioned |
| GET | /api/admin/audit?action= | — | unversioned |
| GET | /api/admin/audit?actorId= | — | unversioned |
| GET | /api/admin/users | — | unversioned |
| GET | /api/admin/users/{userId} | — | unversioned |
| PATCH | /api/admin/users/{userId}/status | — | unversioned |
| POST | /api/admin/users/{username}/unlock | — | unversioned |
| GET | /api/public/users/{userId} | 1.0.0 | 1.0.0 |
| DELETE | /api/public/users/{userId} | 1.0.0 | 1.0.0 |
| POST | /api/public/users/{userId}/revoke-tokens | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/apple | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/google | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/login | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/logout | 1.0.0 | 1.0.0 |
| GET | /api/v2/auth/oauth2/failure | — | unversioned |
| GET | /api/v2/auth/oauth2/register | — | unversioned |
| GET | /api/v2/auth/oauth2/success | — | unversioned |
| POST | /api/v2/auth/register | 1.0.0, 1.1.1 | 1.1.1 |
| POST | /api/v2/auth/resend-email | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/send-account-recovery | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/send-password-reset | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/token | 1.0.0 | 1.0.0 |
| POST | /api/v2/auth/verify-email | 1.0.0 | 1.0.0 |
| GET | /api/v2/availability | 1.0.0 | 1.0.0 |
| GET | /api/v2/exists | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId} | 1.0.0, 1.1.0, 1.1.1 | 1.1.1 |
| PATCH | /api/v2/users/{userId} | 1.0.0, 1.1.0, 1.1.1 | 1.1.1 |
| DELETE | /api/v2/users/{userId} | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId}/b2b-centers/{organizationId}/license-detail | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId}/login-history | 1.0.0 | 1.0.0 |
| PATCH | /api/v2/users/{userId}/password | 1.0.0 | 1.0.0 |
| POST | /api/v2/users/{userId}/password | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId}/sessions | 1.0.0 | 1.0.0 |
| DELETE | /api/v2/users/{userId}/sessions | 1.0.0 | 1.0.0 |
| DELETE | /api/v2/users/{userId}/sessions/{tokenId} | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId}/terms-consents | 1.0.0 | 1.0.0 |
| POST | /api/v2/users/{userId}/terms-consents | 1.0.0 | 1.0.0 |

---

## Auth (REST)

All endpoints below: **base** `/api/v2/auth`, `api-version: 1.0.0`, **security** `permitAll`.

### `POST` /api/v2/auth/register

계정 생성 + 즉시 토큰 발급.

```bash
curl -X POST https://api.example.com/api/v2/auth/register \
  -H "api-version: 1.1.1" -H "Content-Type: application/json" \
  -d '{
    "username": "u@example.com",
    "password": "P@ssw0rd!",
    "firstName": "김철수2",
    "lastName": "박",
    "gender": "MALE",
    "birthdate": "1995-04-27",
    "countryCode": "KR",
    "weightUnit": "LB"
  }'
```

**→ 201**

```json
{
  "success": true,
  "token": {
    "userId": "uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "accessTokenExpiresIn": 900
  }
}
```

**→ 400** — 이름 정규화 실패(전부 이모지·기호) · 필드 형식 위반

```json
{ "success": false, "code": "CMN-400-001", "error": "firstName is invalid after normalization." }
```

**→ 409** — username(이메일)·전화번호 중복

---

**규칙**

- 버전 — `1.0.0`(기존) / `1.1.1` = +`weightUnit`(선택, `KG`|`LB`, 미제공 시 `KG`). 응답 shape 동일.
- 이름 처리(B안, 2026-07-07) — `firstName`/`lastName` 은 거부 대신 정규화 수용: 숫자 허용, 이모지·기호 제거, 50 UTF-8 bytes 절단. 빈 결과만 400 + 서버 WARN 로그(마스킹).
- 전체 필드 정의 — [`UserRegisterRequest`](#userregisterrequest)

### `POST` /api/v2/auth/login

**Operation ID** &nbsp;`login`  &nbsp;**Tags** &nbsp;`auth`

#### Request body

`application/json` — [`UserLoginRequest`](#userloginrequest)

#### Responses

| Status | Schema |
|---|---|
| **200** | [`CncResponse_CncTokenDto`](#cncresponse_cnctokendto) |
| **400** / **401** | [`ErrorResponse`](#errorresponse) |

### `POST` /api/v2/auth/logout

**Operation ID** &nbsp;`logout`  &nbsp;**Tags** &nbsp;`auth`

Invalidate refresh token; record logout time.

#### Request body

`application/json` — `{ refreshToken: string @NotBlank }`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `POST` /api/v2/auth/token

**Operation ID** &nbsp;`refreshToken`  &nbsp;**Tags** &nbsp;`auth`

Reissue access + refresh token pair.

#### Request body

`application/json` — `{ refreshToken: string @NotBlank }`

#### Responses

| Status | Schema |
|---|---|
| **200** | [`CncResponse_CncTokenDto`](#cncresponse_cnctokendto) |
| **401** | [`ErrorResponse`](#errorresponse) |

### `POST` /api/v2/auth/google

**Operation ID** &nbsp;`googleOAuth`  &nbsp;**Tags** &nbsp;`auth`, `oauth`

Mobile Google OAuth.

#### Request body

`application/json` — [`GoogleOAuthRequest`](#googleoauthrequest)

#### Responses

| Status | Schema |
|---|---|
| **201** | [`OAuthTokenExchangeResponse`](#oauthtokenexchangeresponse) |

### `POST` /api/v2/auth/apple

**Operation ID** &nbsp;`appleOAuth`  &nbsp;**Tags** &nbsp;`auth`, `oauth`

Mobile Apple OAuth.

#### Request body

`application/json` — [`AppleOAuthRequest`](#appleoauthrequest)

#### Responses

| Status | Schema |
|---|---|
| **201** | [`OAuthTokenExchangeResponse`](#oauthtokenexchangeresponse) |

### `POST` /api/v2/auth/resend-email

**Operation ID** &nbsp;`resendVerificationEmail`  &nbsp;**Tags** &nbsp;`auth`, `email`

#### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| query | `email` | string | yes | `@ValidEmail(allowEmpty=false)` |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `POST` /api/v2/auth/verify-email

**Operation ID** &nbsp;`verifyEmailByCode`  &nbsp;**Tags** &nbsp;`auth`, `email`

#### Request body

`application/json` — `{ email: string @NotBlank, code: string @Size(min=6,max=6) }`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |
| **400** | [`ErrorResponse`](#errorresponse) — invalid code |

### `POST` /api/v2/auth/send-password-reset

**Operation ID** &nbsp;`sendPasswordResetEmail`  &nbsp;**Tags** &nbsp;`auth`, `email`

#### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| query | `email` | string | yes | `@ValidEmail` |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `POST` /api/v2/auth/send-account-recovery

**Operation ID** &nbsp;`sendAccountRecoveryEmail`  &nbsp;**Tags** &nbsp;`auth`, `email`

> Currently returns "Unsupported version" for `1.0.0`.

---

## Auth (Thymeleaf views)

These endpoints render HTML and are intended to be opened in a browser.

| Method | Path | Query | Description |
|---|---|---|---|
| GET | `/api/v2/auth/verify-email` | `token` | Verification result page |
| GET | `/api/v2/auth/reset-password` | `token` | Password reset form |
| POST | `/api/v2/auth/reset-password` | `token`, `newPassword`, `confirmPassword` | Submit reset |

---

## OAuth2 (Deprecated `/api/v2/auth/oauth2`)

`@Deprecated(since = "1.0.0")`. Not used in current routing.

| Method | Path | Description |
|---|---|---|
| GET | `/api/v2/auth/oauth2/register` | Returns Google authorization URL |
| GET | `/api/v2/auth/oauth2/success` | OAuth2 success — JWT issuance |
| GET | `/api/v2/auth/oauth2/failure` | 401 |

---

## Terms Consent

### `GET` /api/v2/users/{userId}/terms-consents

**Operation ID** &nbsp;`listTermsConsents`  &nbsp;**Tags** &nbsp;`terms-consent`
**Security** &nbsp;`adminBearer` OR (`userBearer`/`guestBearer` AND resource owner)

List the user's consent history (descending by `agreedAt`).

#### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | — |
| path | `userId` | string (uuid) | yes | `@ValidUuid` |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` with `data:` [`TermsConsentItem`](#termsconsentitem)[] |

### `POST` /api/v2/users/{userId}/terms-consents

**Operation ID** &nbsp;`recordTermsConsents`  &nbsp;**Tags** &nbsp;`terms-consent`
**Security** &nbsp;same as GET

Record consents (idempotent on `(termsType, termsVersion)`).

#### Request body

`application/json` — [`TermsConsentRecordRequest`](#termsconsentrecordrequest)

```json
{
  "items": [
    { "termsType": "SERVICE", "termsVersion": "1.0" },
    { "termsType": "PRIVACY", "termsVersion": "1.2" }
  ]
}
```

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |
| **400** | [`ErrorResponse`](#errorresponse) |

---

## User

All endpoints below: **base** `/api/v2`, `api-version: 1.0.0`, **security** `adminBearer` OR (`userBearer` AND resource owner) unless noted.

### `GET` /api/v2/users/{userId}

```bash
curl https://api.example.com/api/v2/users/{userId} \
  -H "api-version: 1.1.1" -H "Authorization: Bearer <jwt>"
```

**→ 200**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "firstName": "Jonghak", "lastName": "Lee", "nickname": "JH",
    "gender": "MALE", "birthdate": "1995-04-27", "height": 175.5,
    "countryCode": "KOR", "languageCode": "ko-KR",
    "weightUnit": "KG",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "userEmail": { "emailFull": "u@x.com", "verified": true },
    "userProviders": [{ "provider": "GOOGLE" }],
    "authorities": [{ "role": "USER" }]
  }
}
```

**→ 403 / 404** — 타인 리소스 · 미존재/삭제 대기 계정

---

**규칙**

- 버전 — `1.0.0`(legacy enum 표기) / `1.1.0`(country alpha-3·language BCP-47) / `1.1.1` = 1.1.0 + `weightUnit`. 하위 버전 응답에 `weightUnit` 은 노출되지 않는다.
- 응답 스키마 — [`UserResponse`](#userresponse)

### `PATCH` /api/v2/users/{userId}

부분 수정 — 모든 필드가 `JsonNullable<T>` 라 "안 보냄(absent)"과 "명시적 null" 을 구분한다.

```bash
curl -X PATCH https://api.example.com/api/v2/users/{userId} \
  -H "api-version: 1.1.1" -H "Authorization: Bearer <jwt>" -H "Content-Type: application/json" \
  -d '{"firstName":"John🔥","weightUnit":"KG"}'
```

**→ 200** — 정규화된 저장값이 그대로 반환된다

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "firstName": "John",
    "weightUnit": "KG",
    "countryCode": "KOR", "languageCode": "ko-KR"
  }
}
```

**→ 400** — null clear 시도 · 정규화 후 빈 이름

```json
{ "success": false, "code": "CMN-400-002", "error": "weightUnit cannot be null." }
```

---

**규칙**

- 버전 — `1.0.0`/`1.1.0`(불변) / `1.1.1` = 요청·응답에 `weightUnit`.
- 이름 처리(B안) — register 와 동일 규칙으로 정규화 수용. 임시 readonly(394b040) 해제 — 다시 수정 가능.
- NOT NULL 필드(gender·weightUnit 등)의 null clear 는 400.
- 전체 필드 정의 — [`UserUpdateRequest`](#userupdaterequest)

### `PATCH` /api/v2/users/{userId}/password

**Operation ID** &nbsp;`changePassword`  &nbsp;**Tags** &nbsp;`user`

#### Request body

`application/json` — `{ oldPassword: string @ValidPassword, newPassword: string @ValidPassword }`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |
| **400** / **401** | [`ErrorResponse`](#errorresponse) |

### `POST` /api/v2/users/{userId}/password

**Operation ID** &nbsp;`setPassword`  &nbsp;**Tags** &nbsp;`user`

Set password without an existing one (OAuth users).

#### Request body

`application/json` — `{ newPassword: string @ValidPassword }`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `DELETE` /api/v2/users/{userId}

**Operation ID** &nbsp;`deleteUser`  &nbsp;**Tags** &nbsp;`user`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `GET` /api/v2/availability

**Operation ID** &nbsp;`checkAvailability`  &nbsp;**Tags** &nbsp;`user`
**Security** &nbsp;`permitAll`

#### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| query | `field` | string (enum `DuplicationField`) | yes | `@ValidEnum` |
| query | `value` | string | yes | — |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` with `data: { field, value, available }` |

### `GET` /api/v2/exists

**Operation ID** &nbsp;`existsUser`  &nbsp;**Tags** &nbsp;`user`
**Security** &nbsp;`permitAll`

#### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| query | `field` | string (enum `ExistsField`) | yes | `@ValidEnum` |
| query | `value` | string | yes | — |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` with `data: { field, value, exists }` |

### `GET` /api/v2/users/{userId}/sessions

**Operation ID** &nbsp;`listSessions`  &nbsp;**Tags** &nbsp;`user`, `session`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` with `data:` [`Session`](#session)[] |

### `DELETE` /api/v2/users/{userId}/sessions

**Operation ID** &nbsp;`revokeAllSessions`  &nbsp;**Tags** &nbsp;`user`, `session`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `DELETE` /api/v2/users/{userId}/sessions/{tokenId}

**Operation ID** &nbsp;`revokeSession`  &nbsp;**Tags** &nbsp;`user`, `session`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `tokenId` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `GET` /api/v2/users/{userId}/login-history

**Operation ID** &nbsp;`getLoginHistory`  &nbsp;**Tags** &nbsp;`user`, `audit`

Pageable. Default sort `createdAt,desc`, size `20`.

#### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| query | `page` | integer | no | default `0` |
| query | `size` | integer | no | default `20` |
| query | `sort` | string | no | default `createdAt,desc` |

#### Responses

| Status | Schema |
|---|---|
| **200** | `Page<` [`LoginHistory`](#loginhistory) `>` |

---

## B2B Center (`/api/v2/users/{userId}/b2b-centers`)

B2C (carenco) 회원이 가입한 b2b 시설의 이용권 상세. b2b-service 의 `B2cMemberQuery` gRPC 합성 (0.0.78).

### `GET` /api/v2/users/{userId}/b2b-centers/{organizationId}/license-detail

**Operation ID** &nbsp;`getB2bCenterLicenseDetail`  &nbsp;**Tags** &nbsp;`b2b-center`
**Security** &nbsp;resource-owner (`ROLE_ADMIN` 또는 본인 `#userId`)

회원의 특정 시설 이용권 상세 — 멤버십 + 이용권 (구독 기간 / 갱신 취소 여부) + org 담당자 (공유자).
b2b-service 의 `B2cMemberQuery.GetLicenseDetail(carencoUserId, organizationId)` gRPC 를 호출해 합성.

**Path Parameters**

| In | Name | Type | Required |
|---|---|---|---|
| path | `userId` | string (uuid) | yes (`@ValidUuid`, 본인) |
| path | `organizationId` | string (uuid) | yes (`@ValidUuid`) |

**Responses**

| Status | Schema |
|---|---|
| **200** | `Envelope` with `data:` [`B2bCenterLicenseDetail`](#b2bcenterlicensedetail) |
| **401** | resource-owner 불일치 |
| **502** | b2b-service gRPC 실패 (다운 / deadline) |

#### 200 — example (가입 + 구독 ACTIVE)

```json
{
  "success": true,
  "data": {
    "registered": true,
    "organizationId": "4f97825c-6d9c-4438-b75f-a00f152ab6bb",
    "organizationName": "센터-1781487663697",
    "organizationType": "PILATES",
    "photoUrl": null,
    "membershipStatus": "ACTIVE",
    "memberNumber": "M260615-004",
    "joinedAt": "2026-06-15T01:41:05Z",
    "licenseState": "ACTIVE",
    "startedAt": "2026-06-15T01:41:21Z",
    "expiresAt": "2026-07-15T01:41:21Z",
    "canceledAt": null,
    "managers": [
      { "firstName": "박", "lastName": "관장", "role": "OWNER" }
    ]
  }
}
```

#### 200 — example (미가입)

```json
{ "success": true, "data": { "registered": false, "managers": [] } }
```

> `B2bCenterLicenseDetail`.
> - `registered` — `false` 면 해당 시설 멤버십 없음 (나머지 필드 null).
> - `licenseState` — `ACTIVE` / `GRACE` / `EXPIRED` / `SUSPENDED` / `NONE`.
> - `canceledAt` — 갱신 취소 예약 시각. `null` = 취소 예약 없음. 값 있으면 그 시점에 구독 종료 예정 (cycle-end cancel, 그 전까지 `ACTIVE`).
> - `managers` — org 담당자 (공유자). OWNER (`organizations.owner_user_id`) + ACTIVE 인 ADMIN staff 의 `firstName` / `lastName` / `role`.
> - 프리미엄 리포트 / Fisica AI 채팅 가용 여부는 추후 필드 추가 (현재 미노출).

가입 시설 **목록** 은 b2b-service 의 `GET /api/v2/b2b/b2c-members/{carencoUserId}/organizations` 직접 호출.

---

## Public (`/api/public`)

Service-to-service. **Security** &nbsp;`permitAll`.

### `GET` /api/public/users/{userId}

**Operation ID** &nbsp;`publicGetUser`  &nbsp;**Tags** &nbsp;`public`

#### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| path | `userId` | string (uuid) | yes | `@ValidUuid` |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` with `data:` [`UserResponse`](#userresponse) |

### `POST` /api/public/users/{userId}/revoke-tokens

**Operation ID** &nbsp;`publicRevokeTokens`  &nbsp;**Tags** &nbsp;`public`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `DELETE` /api/public/users/{userId}

**Operation ID** &nbsp;`publicDeleteUser`  &nbsp;**Tags** &nbsp;`public`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

---

## Admin (`/api/admin`) — `adminBearer`

### `GET` /api/admin/users

**Operation ID** &nbsp;`adminListUsers`  &nbsp;**Tags** &nbsp;`admin`

Pageable. Default sort `createdAt,desc`, size `20`.

#### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| query | `page` | integer | no | default `0` |
| query | `size` | integer | no | default `20` |
| query | `sort` | string | no | default `createdAt,desc` |

#### Responses

| Status | Schema |
|---|---|
| **200** | `Page<` [`AdminUser`](#adminuser) `>` |

### `GET` /api/admin/users/{userId}

**Operation ID** &nbsp;`adminGetUser`  &nbsp;**Tags** &nbsp;`admin`

#### Responses

| Status | Schema |
|---|---|
| **200** | [`AdminUser`](#adminuser) |

### `PATCH` /api/admin/users/{userId}/status

**Operation ID** &nbsp;`adminUpdateStatus`  &nbsp;**Tags** &nbsp;`admin`

#### Request body

`application/json` — `UpdateStatusRequest { status: string @NotBlank, reason?: string }`

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

### `POST` /api/admin/users/{username}/unlock

**Operation ID** &nbsp;`adminUnlockUser`  &nbsp;**Tags** &nbsp;`admin`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `username` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` (data: null) |

---

## Admin Audit (`/api/admin/audit`) — `adminBearer`

| Method | Path | Operation ID | Description |
|---|---|---|---|
| GET | `/api/admin/audit/users/{userId}` | `auditByUser` | by user (paged) |
| GET | `/api/admin/audit?action=` | `auditByAction` | by action type (paged) |
| GET | `/api/admin/audit?actorId=` | `auditByActor` | by actor (paged) |

All return `Page<` [`AuditEvent`](#auditevent) `>`. Default sort `createdAt,desc`, size `20`.

---

## Schemas

### `CncResponse`

| Field | Type | Required |
|---|---|---|
| `success` | boolean | yes |
| `data` | object | no |
| `token` | object | no |
| `error` | object | no |
| `timestamp` | string (date-time, UTC) | yes |

### `ErrorResponse`

```json
{ "success": false, "code": "...", "message": "...", "timestamp": "..." }
```

### `CncResponse_CncTokenDto`

`CncResponse` with `data: ` [`CncTokenDto`](#cnctokendto).

### `CncTokenDto`

| Field | Type | Required | Description |
|---|---|---|---|
| `userId` | string (uuid) | yes | — |
| `accessToken` | object | yes | JWT access token wrapper |
| `refreshToken` | object | yes | JWT refresh token wrapper |
| `expiresIn` | integer (int64) | yes | seconds |
| `refreshExpiresIn` | integer (int64) | yes | seconds |

### `UserRegisterRequest`

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `username` | string | no | `@ValidEmail(allowEmpty=false)` | login id (email) |
| `password` | string | yes | `@ValidPassword(allowEmpty=false)` | min 8, includes digit |
| `firstName` | string | no | `@ValidName(allowEmpty=false)` | — |
| `lastName` | string | no | `@ValidName(allowEmpty=false)` | — |
| `nickname` | string | no | `@ValidName(allowEmpty=false)` | — |
| `phoneNumber` | string | no | `@ValidPhoneNumber` | E.164 |
| `gender` | string | no | `@ValidEnum(Gender)` | `MALE` `FEMALE` `OTHER` `UNDISCLOSED` |
| `birthdate` | string (date) | no | `@ValidBirthDate` | not future |
| `height` | number (double) | no | `@ValidHeight` | 30-260 cm |
| `countryCode` | string | no | `@ValidEnum(CountryCode)` | ISO 3166-1 α-2 |
| `languageCode` | string | no | `@ValidEnum(LanguageCode)` | ISO 639-1 |
| `timeZone` | string | no | `@ValidEnum(TimeZone)` | enum |
| `zoneIdValue` | string | no | — | IANA zone id |

**`countryCode` 허용값** — `CountryCode` enum 이름 (ISO 3166-1 α-2).

| 값 | 국가 | 값 | 국가 |
|---|---|---|---|
| `UNKNOWN` | 알 수 없음 | `BE` | 벨기에 |
| `US` | 미국 | `SG` | 싱가포르 |
| `IT` | 이탈리아 | `JP` | 일본 |
| `FR` | 프랑스 | `CZ` | 체코 |
| `AE` | 아랍에미리트 | `CA` | 캐나다 |
| `UK` | 영국 | `AU` | 호주 |
| `KR` | 대한민국 | `CN` | 중국 (0.0.80 신규) |
| `NL` | 네덜란드 | `TW` | 대만 (0.0.80 신규) |
| `PT` | 포르투갈 | | |
| `DE` | 독일 | | |

> **입력** (common-libs 0.0.83~) — `countryCode` 입력은 alpha-2 외 **ISO 3166-1 alpha-3**(`KOR`·`USA`·`CHN`·`TWN`)·레거시 별칭도 허용. 모두 enum 으로 정규화.
>
> **저장·응답** (0.0.84~) —
> - **DB 저장**: ISO 3166-1 **alpha-3**(`KOR`). 0.0.83 까지 alpha-2 였고 V33 마이그레이션으로 전환.
> - **응답**: `api-version` 별 — **`1.0.0`** → alpha-2(`"KR"`, 기존 클라 무영향), **`1.1.0`** → alpha-3(`"KOR"`). gRPC·admin 은 alpha-2 유지.
>
> 레거시 별칭 (입력 전용).
>
> | alpha-2 | 레거시 별칭 |
> |---|---|
> | `AE` | `UAE` |
> | `NL` | `NETHERLANDS` |
> | `PT` | `PORTUGAL` |
> | `DE` | `GERMANY` |
> | `BE` | `BELGIUM` |
>
> 그 외 국가는 alpha-3 만 추가 허용. b2b-service 의 `country`(`@ValidCountryCode`)는 **strict alpha-2** 입력만 받되 DB 저장은 alpha-3(응답 `1.0.2` 노출).

**`languageCode` 허용값** — `LanguageCode` enum 이름. 값은 BCP 47 language tag 와 매핑되며 `LanguageCode.toLocale()` 로 `Locale` 변환.

| 값 | 언어 | BCP 47 |
|---|---|---|
| `UNKNOWN` | 알 수 없음 | — (`Locale.ROOT`) |
| `KR` | 한국어 | `ko-KR` |
| `IT` | 이탈리아어 | `it-IT` |
| `US` | 영어 (미국) | `en-US` |
| `SG` | 영어 (싱가포르) | `en-SG` |
| `JP` | 일본어 | `ja-JP` |
| `DE` | 독일어 | `de-DE` |
| `FR` | 프랑스어 | `fr-FR` |
| `CZ` | 체코어 | `cs-CZ` |
| `CA` | 영어 (캐나다) | `en-CA` |
| `AU` | 영어 (호주) | `en-AU` |
| `ZH_HANS` | 중국어 간체 (0.0.80 신규) | `zh-Hans` |
| `ZH_HANT` | 중국어 번체 (0.0.80 신규) | `zh-Hant` |

> 간체/번체는 국가가 아니라 문자(script) 구분이라 enum 이름이 국가코드가 아닌 BCP 47 script subtag(`ZH_HANS`/`ZH_HANT`) 기반.

> **입력** (common-libs 0.0.82~) — `languageCode` 는 enum 이름(`ZH_HANS`)·BCP 47 tag(`zh-Hans`·`ko-KR`) 모두 허용. `en` 같은 language-only 는 모호하여 비수용(full tag 만).
>
> **저장·응답** (0.0.84~) — **DB 저장**: **BCP 47**(`ko-KR`·`zh-Hans`, V33 마이그레이션). **응답**: `api-version` **`1.0.0`** → enum 이름(`"KR"`/`"ZH_HANS"`, 기존), **`1.1.0`** → BCP 47(`"ko-KR"`/`"zh-Hans"`). gRPC·admin 은 enum 이름 유지.

### `UserLoginRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `username` | string | yes | `@ValidEmail` |
| `password` | string | yes | `@ValidPassword` |

### `UserUpdateRequest`

Every field is `JsonNullable<T>`:

`firstName, lastName, nickname, email, phoneNumber, gender, birthdate, height, countryCode, languageCode, timeZone, zoneId`

Validation reuses the register validators (`@ValidName`, `@ValidEmail`, `@ValidPhoneNumber(allowNull=true)`, `@ValidEnum`, `@ValidBirthDate`, `@ValidHeight`).

### `GoogleOAuthRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `serverAuthCode` | string | yes | `@NotBlank` |
| `idToken` | string | yes | `@NotBlank` |

### `AppleOAuthRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `identityToken` | string | yes | `@NotBlank` |
| `authorizationCode` | string | yes | `@NotBlank` |

### `OAuthTokenExchangeResponse`

| Field | Type | Description |
|---|---|---|
| `accessToken` | string | provider access token |
| `idToken` | string | provider ID token |
| `refreshToken` | string | provider refresh token |
| `jwtToken` | [`CncTokenDto`](#cnctokendto) | CareNCo-issued tokens |
| `userInfo` | object | provider-specific |

### `TermsConsentItem`

| Field | Type |
|---|---|
| `termsType` | string (enum `TermsType`) |
| `termsVersion` | string |
| `agreedAt` | string (date-time, UTC) |

### `TermsConsentRecordRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `items` | [`ConsentItem`](#consentitem)[] | yes | `@NotEmpty`, `@Valid` |

### `ConsentItem`

| Field | Type | Required | Validation |
|---|---|---|---|
| `termsType` | string | yes | `@ValidEnum(TermsType)` |
| `termsVersion` | string | yes | `@NotBlank @Size(max=64)` |

### `UserResponse`

| Field | Type | Description |
|---|---|---|
| `id` | string (uuid) | — |
| `userProviders` | array<`{ provider: enum, providerId: string }`> | — |
| `userEmail` | `{ emailLocal, emailDomain, emailFull, verified, verifiedAt }` | — |
| `firstName` `lastName` `nickname` | string | — |
| `phone` | `{ e164, countryCode, national, formatted }` | — |
| `photoUrl` | string | — |
| `gender` | string (enum `Gender`) | — |
| `birthdate` | string (date) | — |
| `height` | number (double) | — |
| `countryCode` | string (enum `CountryCode`) | — |
| `languageCode` | string (enum `LanguageCode`) | — |
| `timeZone` | string (enum `TimeZone`) | — |
| `zoneId` | string (IANA) | — |
| `authorities` | array<`{ role: enum }`> | — |

### `Session`

| Field | Type | Description |
|---|---|---|
| `tokenId` | string | JWT id |
| `deviceId` | string | — |
| `deviceType` | string | `MOBILE` `WEB` etc. |
| `createdAt` | string (date-time, UTC) | — |
| `expiresAt` | string (date-time, UTC) | — |
| `current` | boolean | — |

### `LoginHistory`

| Field | Type | Description |
|---|---|---|
| `action` | string | `LOGIN` `LOGOUT` etc. |
| `ipAddress` | string | — |
| `description` | string | — |
| `timestamp` | string (date-time, UTC) | — |

### `AdminUser`

| Field | Type |
|---|---|
| `id` `email` `firstName` `lastName` `nickname` `phoneNumber` | string |
| `gender` | string |
| `birthdate` | string (date) |
| `height` | number (double) |
| `countryCode` `languageCode` `zoneId` | string |
| `deletionStatus` | string |
| `emailVerified` | boolean |
| `lastLoginTime` `lastLogoutTime` `createdAt` `updatedAt` | string (date-time, UTC) |

### `AuditEvent`

| Field | Type |
|---|---|
| `id` `actorId` `action` `entityType` `entityId` | string |
| `changes` | string (JSON) |
| `ipAddress` `traceId` `description` | string |
| `createdAt` | string (date-time, UTC) |
