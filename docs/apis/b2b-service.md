# b2b-service

> 엔드포인트별 **Header · Request · Response** 정의 — 버전별 전량 전개, 소스는 실제 컨트롤러/DTO. 표기 — **굵은 필드 = 필수**, 규칙은 키워드만.

| 항목 | 값 |
|---|---|
| Source | `/Users/jonghak/GitHub/Care&Co/b2b-service` (`origin/develop-k3s`) |
| Updated | 2026-07-10 |
| 배포 상태 | **dev 전용 (prod 미배포)** — `develop-k3s` 현행 구현, 운영 API 계약 미확정 |
| Server | `https://api.example.com` |
| Base path | 모든 엔드포인트 `/api/v2/b2b/...` 하위 |

---

## 공통 규칙

- **버전** — 모든 엔드포인트 versioned. `api-version: x.y.z` 헤더 필수, **정확 일치만** (미등록 → `400 Invalid API version`).
- **인증** — `Authorization: Bearer <jwt>` (모바일) → `B2B-SESSION` 쿠키 (웹) → 익명 순 분기. 역할(OWNER · ADMIN · TRAINER)·본인 확인은 application 레이어 검증 — 실패 시 `403 PERMISSION_DENIED`.
- **바디** — 요청 바디가 있으면 `Content-Type: application/json`.
- **응답 틀** — 모든 응답은 `CncResponse` envelope (아래 접기 참조) — 개별 섹션 샘플에서 반복하지 않음.

**복수 버전 엔드포인트** — 대부분 `1.0.0` 단일, 아래 11개만 복수. 버전 간 차이는 각 섹션 상단의 버전 표 참조.

| 그룹 | 엔드포인트 | 최신 |
|---|---|---|
| users (country 표기) | [`POST /users`](#6-post-apiv2b2busers) · [`GET`](#7-get-apiv2b2busersb2buserid)/[`PATCH /users/{b2bUserId}`](#8-patch-apiv2b2busersb2buserid) | `1.0.1` |
| organizations (BRN·stats·country) | [`POST /organizations`](#11-post-apiv2b2borganizations) · [`GET /organizations/{id}`](#13-get-apiv2b2borganizationsid) | `1.0.2` |
| devices (deviceType·deviceNumber) | [`등록`](#23-post-apiv2b2borganizationsorganizationiddevices) · [`목록`](#24-get-apiv2b2borganizationsorganizationiddevices) · [`단건`](#25-get-apiv2b2borganizationsorganizationiddevicesdeviceid) · [`수정`](#26-patch-apiv2b2borganizationsorganizationiddevicesdeviceid) · [`deactivate`](#27-post-apiv2b2borganizationsorganizationiddevicesdeviceiddeactivate) · [`삭제`](#28-delete-apiv2b2borganizationsorganizationiddevicesdeviceid) | `1.0.1` |

<details>
<summary><b>공개 경로</b> (인증 불필요) — 그 외 전부 인증 필수</summary>

| 공개 경로 | 비고 |
|---|---|
| `/api/v2/b2b/auth/**` | 로그인 · 토큰 회전 · OAuth2 |
| `GET /api/v2/b2b/organizations/search` · `GET /api/v2/b2b/organizations/{id}` | 시설 검색/열람 (가입 전) |
| `POST /api/v2/b2b/organizations/{id}/join-requests` · `POST /api/v2/b2b/invite-codes/redeem` · `POST /api/v2/b2b/memberships/{id}/leave` | Carenco 사용자 가입 흐름 (body 의 memberId 로 식별) |
| `POST /api/v2/b2b/users` | 관리자 회원가입 |
| `GET /api/v2/b2b/availability` | 가입 전 가용성 검사 |
| `GET /api/v2/b2b/b2c-members/{carencoUserId}/organizations` | Carenco 사용자 가입 현황 조회 |

</details>

<details>
<summary><b>응답 envelope</b> — 성공/에러 JSON · 필드 정의</summary>

**성공** (`CncResponse`)

```json
{
  "success": true,
  "data": {},
  "token": null,
  "error": null,
  "timestamp": "2026-04-27T08:00:00Z"
}
```

**에러** (`ErrorResponse`) — 코드 전체는 문서 하단 [에러 코드](#에러-코드) 참조

```json
{
  "success": false,
  "code": "CMN-400-001",
  "message": "Invalid request parameter. Field: email, Value: ...",
  "timestamp": "2026-04-27T08:00:00Z"
}
```

| 필드 | 타입 | 필수 |
|---|---|---|
| `success` | boolean | yes |
| `data` | object | no — endpoint-specific |
| `token` | object | no — 인증 계열 응답에서만 (`accessToken`/`refreshToken`) |
| `error` | object | no |
| `code` · `message` | string | 에러 응답에서만 |
| `timestamp` | string (date-time, UTC) | yes |

</details>


---

## auth

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/auth/login`](#1-post-apiv2b2bauthlogin) | 1.0.0 | 공개 | 이메일·비밀번호 로그인 |
| POST | [`/auth/oauth2/google`](#2-post-apiv2b2bauthoauth2google) | 1.0.0 | 공개 | 모바일 클라이언트가 네이티브 SDK 로 받은 Google `idToken` 을 검증하고 Care… |
| POST | [`/auth/oauth2/apple`](#3-post-apiv2b2bauthoauth2apple) | 1.0.0 | 공개 | 모바일 클라이언트가 네이티브 SDK 로 받은 Apple `idToken` 을 검증하고 CareN… |
| POST | [`/auth/token`](#4-post-apiv2b2bauthtoken) | 1.0.0 | 공개 | refresh token 회전 → 새 access + refresh 토큰 쌍 재발급 |
| POST | [`/auth/logout`](#5-post-apiv2b2bauthlogout) | 1.0.0 | 공개 | 로그아웃 |

---

## 1. `POST` /api/v2/b2b/auth/login

이메일·비밀번호 로그인. `X-Client-Type` 헤더로 흐름이 갈린다 — `web` 이면 세션(쿠키) 로그인이라 응답에 `userId`·`email`·`sessionId` 를, 그 외(모바일 기본) 이면 `TokenPair` 를 반환한다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `X-Client-Type` | `web` (세션 흐름) \| 그 외/미전송 (모바일 토큰 흐름) | no |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

인증 불필요. 자격 증명 검증은 Spring Security `AuthenticationManager` 가 수행하며 실패 시 throw → GlobalExceptionHandler 가 매핑한다.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "email": "owner@example.com",
  "password": "P@ssw0rd!",
  "deviceId": "device-uuid",
  "deviceType": "IOS"
}
```

### `1.0.0` — Response

**200 OK** — 모바일 흐름 (기본). `data` 에 `TokenPair`

```json
{
  "success": true,
  "data": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ..."
  }
}
```

**200 OK** — 웹 세션 흐름 (`X-Client-Type: web`). `data` 에 세션 식별 정보, 토큰은 쿠키

```json
{
  "success": true,
  "data": {
    "userId": "3f2a...uuid",
    "email": "owner@example.com",
    "sessionId": "A1B2C3..."
  }
}
```

<details>
<summary><b>400 Bad Request</b> — email 형식 위반 · password 누락</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: email" }
```

</details>

<details>
<summary><b>401 Unauthorized</b> — 자격 증명 불일치</summary>

```json
{ "success": false, "code": "AUTH-401-003", "message": "Authentication failed" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — 계정 잠김/비활성 (로그인 시도 초과·정지 등)</summary>

```json
{ "success": false, "code": "AUTH-403-004", "message": "Your account is locked" }
```

</details>

</details>

### Request 필드 정의 — `LoginRequest`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`email`** | string | yes | `@ValidEmail(allowEmpty=false, allowNull=false)` |
| **`password`** | string | yes | `@NotBlank` |
| `deviceId` | string | no | 토큰 흐름의 세션 식별용 (검증 없음) |
| `deviceType` | string | no | 예 `IOS` · `ANDROID` (검증 없음) |

### Response 필드 정의 — 모바일 `TokenPair`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `userId` | string (uuid) | yes | — |
| `accessToken` | string (JWT) | yes | — |
| `refreshToken` | string (opaque) | yes | — |

### Response 필드 정의 — 웹 세션 (`Map`)

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `userId` | string (uuid) | yes | — |
| `email` | string | yes | — |
| `sessionId` | string | yes | Spring Session id (쿠키로도 전달) |

---

## 2. `POST` /api/v2/b2b/auth/oauth2/google

모바일 클라이언트가 네이티브 SDK 로 받은 Google `idToken` 을 검증하고 CareNCo JWT 를 발급한다. `b2b.oauth2.enabled=true` 일 때만 등록되며, 미설정/false 면 엔드포인트 자체가 없어 404.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

인증 불필요. 단, 컨트롤러가 `@ConditionalOnProperty(b2b.oauth2.enabled=true)` 로 조건부 등록 — 비활성 시 404.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "idToken": "eyJ...",
  "deviceId": "device-uuid",
  "deviceType": "IOS"
}
```

### `1.0.0` — Response

**200 OK** — `data` 에 CareNCo 발급 `TokenPair`

```json
{
  "success": true,
  "data": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ..."
  }
}
```

<details>
<summary><b>400 Bad Request</b> — idToken 누락 (<code>@NotBlank</code>)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: idToken" }
```

</details>

<details>
<summary><b>400 Bad Request</b> — provider 가 email 미반환 (<code>EmailMissing</code>)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter" }
```

</details>

<details>
<summary><b>401 Unauthorized</b> — idToken 검증 실패 · audience 불일치 (<code>InvalidIdToken</code> / <code>InvalidAudience</code>)</summary>

```json
{ "success": false, "code": "AUTH-401-001", "message": "Invalid or malformed token" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — 계정 정지 (<code>AccountSuspended</code>)</summary>

```json
{ "success": false, "code": "AUTH-403-003", "message": "Your account is disabled" }
```

</details>

<details>
<summary><b>502 Bad Gateway</b> — OAuth2 검증기 사용 불가 (<code>VerifierUnavailable</code>)</summary>

```json
{ "success": false, "code": "CMN-502-001", "message": "External service error" }
```

</details>

</details>

### Request 필드 정의 — `OAuth2LoginRequest`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`idToken`** | string | yes | `@NotBlank` — 네이티브 SDK 발급 Google id token |
| `deviceId` | string | no | 검증 없음 |
| `deviceType` | string | no | 검증 없음 |

### Response 필드 정의 — `TokenPair`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `userId` | string (uuid) | yes | — |
| `accessToken` | string (JWT) | yes | — |
| `refreshToken` | string (opaque) | yes | — |

---

## 3. `POST` /api/v2/b2b/auth/oauth2/apple

모바일 클라이언트가 네이티브 SDK 로 받은 Apple `idToken` 을 검증하고 CareNCo JWT 를 발급한다. Google 과 동일 DTO·에러 체계. `b2b.oauth2.enabled=true` 일 때만 등록 (미설정 시 404).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

인증 불필요. `@ConditionalOnProperty(b2b.oauth2.enabled=true)` 로 조건부 등록 — 비활성 시 404.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "idToken": "eyJ...",
  "deviceId": "device-uuid",
  "deviceType": "IOS"
}
```

### `1.0.0` — Response

**200 OK** — `data` 에 CareNCo 발급 `TokenPair`

```json
{
  "success": true,
  "data": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ..."
  }
}
```

<details>
<summary><b>400 Bad Request</b> — idToken 누락 · email 미반환</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: idToken" }
```

</details>

<details>
<summary><b>401 Unauthorized</b> — idToken 검증 실패 · audience 불일치</summary>

```json
{ "success": false, "code": "AUTH-401-001", "message": "Invalid or malformed token" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — 계정 정지</summary>

```json
{ "success": false, "code": "AUTH-403-003", "message": "Your account is disabled" }
```

</details>

<details>
<summary><b>502 Bad Gateway</b> — OAuth2 검증기 사용 불가</summary>

```json
{ "success": false, "code": "CMN-502-001", "message": "External service error" }
```

</details>

</details>

### Request 필드 정의 — `OAuth2LoginRequest`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`idToken`** | string | yes | `@NotBlank` — 네이티브 SDK 발급 Apple id token |
| `deviceId` | string | no | 검증 없음 |
| `deviceType` | string | no | 검증 없음 |

### Response 필드 정의 — `TokenPair`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `userId` | string (uuid) | yes | — |
| `accessToken` | string (JWT) | yes | — |
| `refreshToken` | string (opaque) | yes | — |

---

## 4. `POST` /api/v2/b2b/auth/token

refresh token 회전 → 새 access + refresh 토큰 쌍 재발급.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

인증 불필요. refresh token 유효성은 `JwtTokenService.rotateRefresh` 가 검증하며 실패 시 throw.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "refreshToken": "eyJ...",
  "deviceId": "device-uuid",
  "deviceType": "IOS"
}
```

### `1.0.0` — Response

**200 OK** — `data` 에 새 `TokenPair`

```json
{
  "success": true,
  "data": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ..."
  }
}
```

<details>
<summary><b>400 Bad Request</b> — refreshToken 누락</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: refreshToken" }
```

</details>

<details>
<summary><b>401 Unauthorized</b> — refresh token 무효/만료/재사용</summary>

```json
{ "success": false, "code": "AUTH-401-001", "message": "Invalid or malformed token" }
```

</details>

</details>

### Request 필드 정의 — `RefreshRequest`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`refreshToken`** | string | yes | `@NotBlank` |
| `deviceId` | string | no | 회전 시 세션 식별용 (검증 없음) |
| `deviceType` | string | no | 검증 없음 |

### Response 필드 정의 — `TokenPair`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `userId` | string (uuid) | yes | — |
| `accessToken` | string (JWT) | yes | — |
| `refreshToken` | string (opaque) | yes | — |

---

## 5. `POST` /api/v2/b2b/auth/logout

로그아웃. 모바일 흐름은 바디의 `refreshToken` 패밀리를 무효화하고, 웹 흐름은 세션을 무효화한다. `Authorization` Bearer 의 access jti 도 함께 revoke 한다. 바디는 선택.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | no (바디 선택) |
| `Authorization` | `Bearer <jwt>` (access jti revoke 용, 선택) | no |

### Security

인증 강제 없음. 전달된 refreshToken / Authorization jti 가 있으면 그에 대해 revoke 를 수행한다.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 선택 (`required=false`). 모바일 흐름은 refreshToken 전달, 웹 흐름은 빈 바디 가능.

```json
{ "refreshToken": "eyJ..." }
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

### Request 필드 정의 — `LogoutRequest`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `refreshToken` | string | no | 모바일 흐름에서 패밀리 무효화 대상. 웹 흐름은 생략 가능 (검증 없음) |

---

## users

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/users`](#6-post-apiv2b2busers) | **1.0.0, 1.0.1** | 공개 | 관리자(b2b_user) 회원가입 |
| GET | [`/users/{b2bUserId}`](#7-get-apiv2b2busersb2buserid) | **1.0.0, 1.0.1** | Bearer | 본인 프로필 상세 + 소속 organization 목록(멀티-org, license 요약 포함)… |
| PATCH | [`/users/{b2bUserId}`](#8-patch-apiv2b2busersb2buserid) | **1.0.0, 1.0.1** | Bearer | 본인 프로필 부분 수정 |
| POST | [`/users/{b2bUserId}/password`](#9-post-apiv2b2busersb2buseridpassword) | 1.0.0 | Bearer | 기존 비밀번호 확인 후 변경 |
| DELETE | [`/users/{b2bUserId}`](#10-delete-apiv2b2busersb2buserid) | 1.0.0 | Bearer | 본인 계정 소프트 탈퇴 — `DELETION_PENDING` 으로 전환하고 전 세션을 revok… |

---

## 6. `POST` /api/v2/b2b/users

관리자(b2b_user) 회원가입. 성공 시 생성된 프로필을 반환하며, 가입 직후라 `organizations` 는 빈 배열.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | country 를 표시용 **alpha-2** 로 반환 (저장 alpha-3 를 역변환) |
| `1.0.1` **(최신)** | country 를 **alpha-3 저장값 그대로** 반환 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` (country 표기 차이) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

공개 가입 엔드포인트. 별도 `@PreAuthorize` 없음.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "email": "owner@example.com",
  "password": "P@ssw0rd!",
  "firstName": "종학",
  "lastName": "이",
  "country": "KR",
  "phoneNumber": "+821012345678"
}
```

### `1.0.0` — Response

**201 Created** — `data` 에 `UserResponse` (`organizations` 는 빈 배열). country 는 표시용 **alpha-2** (저장 alpha-3 를 역변환)

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "email": "owner@example.com",
    "firstName": "종학",
    "lastName": "이",
    "phoneNumber": "+821012345678",
    "country": "KR",
    "profileImageUrl": null,
    "accountStatus": "ACTIVE",
    "lastLoginTime": null,
    "createdAt": "2026-07-09T09:00:00Z",
    "organizations": []
  }
}
```

<details>
<summary><b>400 Bad Request</b> — email/password/name/country/phone 형식 위반 (<code>@Valid</code>)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: email" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 이메일 중복 (<code>UserError.EmailDuplicate</code>)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request (요청 shape 동일 — country 수용 규칙만 확대)

```json
{
  "email": "owner@example.com",
  "password": "P@ssw0rd!",
  "firstName": "종학",
  "lastName": "이",
  "country": "UK",
  "phoneNumber": "+821012345678"
}
```

### `1.0.1` — Response

**201 Created** — country 를 **alpha-3 저장값 그대로** 반환

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "email": "owner@example.com",
    "firstName": "종학",
    "lastName": "이",
    "phoneNumber": "+821012345678",
    "country": "GBR",
    "profileImageUrl": null,
    "accountStatus": "ACTIVE",
    "lastLoginTime": null,
    "createdAt": "2026-07-10T09:00:00Z",
    "organizations": []
  }
}
```

400/409 는 `1.0.0` 과 동일.

</details>

### Request 필드 정의 — `SignUpRequest`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`email`** | string | yes | `@ValidEmail(allowEmpty=false, allowNull=false)` · 중복 시 409 |
| **`password`** | string | yes | `@ValidPassword(allowEmpty=false, allowNull=false)` — 영문·숫자·특수문자 8자+ |
| **`firstName`** | string | yes | `@ValidName(allowEmpty=false, allowNull=false)` |
| **`lastName`** | string | yes | `@ValidName(allowEmpty=false, allowNull=false)` |
| `country` | string | no | `@ValidEnum(CountryCode)` — alpha-2/alpha-3/별칭(`UK`·`UAE`) 수용 · 저장은 alpha-3 · **미제공 → `UNKNOWN`** |
| `phoneNumber` | string | no | `@ValidPhoneNumber(allowNull=true)` — E.164 |

### Response 필드 정의 — `UserResponse`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `id` | string (uuid) | yes | — |
| `email` | string | yes | — |
| `firstName` `lastName` | string | yes | — |
| `phoneNumber` | string \| null | no | — |
| `country` | string | yes | **버전별** — `1.0.0`: alpha-2(`KR`, 역변환) / `1.0.1`: alpha-3(`KOR`) · 미제공 `UNKNOWN` |
| `profileImageUrl` | string \| null | no | — |
| `accountStatus` | string (enum `AccountStatus`) | yes | `ACTIVE` `SUSPENDED` `BANNED` `DELETION_PENDING` `DELETED` |
| `lastLoginTime` | string (date-time, UTC) \| null | no | — |
| `createdAt` | string (date-time, UTC) | yes | — |
| `organizations` | array<`UserOrganizationSummaryView`> | yes | 가입 직후 빈 배열. 상세는 GET §users/{id} 참조 |

---

## 7. `GET` /api/v2/b2b/users/{b2bUserId}

본인 프로필 상세 + 소속 organization 목록(멀티-org, license 요약 포함) 조회.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | country 를 표시용 **alpha-2** 로 반환 (저장 alpha-3 를 역변환) |
| `1.0.1` **(최신)** | country 를 **alpha-3 저장값 그대로** 반환 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` (country 표기 차이) | yes |
| `Authorization` | `Bearer <jwt>` (본인) | yes |

### Security

self-check — path 의 `b2bUserId` 가 인증 principal 의 id 와 일치해야 한다. 불일치 시 `403 PERMISSION_DENIED` (`UserError.NotSelf`). (향후 admin 역할 도입 시 확장 예정, 현재는 본인만.)

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `b2bUserId` (uuid) 만.

### `1.0.0` — Response

**200 OK** — `data` 에 `UserResponse` (`organizations` 채워짐). country 는 표시용 **alpha-2** (역변환)

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "email": "owner@example.com",
    "firstName": "종학",
    "lastName": "이",
    "phoneNumber": "+821012345678",
    "country": "KR",
    "profileImageUrl": null,
    "accountStatus": "ACTIVE",
    "lastLoginTime": "2026-07-08T09:00:00Z",
    "createdAt": "2026-07-01T09:00:00Z",
    "organizations": [
      {
        "id": "org-uuid",
        "name": "강남 필라테스",
        "type": "PILATES",
        "role": "OWNER",
        "seatsUsed": 12,
        "planSeats": 30,
        "licenseState": "ACTIVE"
      }
    ]
  }
}
```

<details>
<summary><b>403 Forbidden</b> — path b2bUserId 가 principal 과 불일치 (<code>UserError.NotSelf</code>)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 사용자 미존재 (<code>UserError.NotFound</code>)</summary>

```json
{ "success": false, "code": "USR-404-001", "message": "User not found" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

바디 없음 — path 의 `b2bUserId` (uuid) 만.

### `1.0.1` — Response

**200 OK** — `1.0.0` 과 동일 shape, country 만 **alpha-3 저장값 그대로**

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "email": "owner@example.com",
    "firstName": "종학",
    "lastName": "이",
    "phoneNumber": "+821012345678",
    "country": "KOR",
    "profileImageUrl": null,
    "accountStatus": "ACTIVE",
    "lastLoginTime": "2026-07-08T09:00:00Z",
    "createdAt": "2026-07-01T09:00:00Z",
    "organizations": [
      {
        "id": "org-uuid",
        "name": "강남 필라테스",
        "type": "PILATES",
        "role": "OWNER",
        "seatsUsed": 12,
        "planSeats": 30,
        "licenseState": "ACTIVE"
      }
    ]
  }
}
```

403/404 는 `1.0.0` 과 동일.

</details>

### Request 필드 정의

바디 없음. path `b2bUserId` (uuid) 만.

### Response 필드 정의 — `UserResponse` (§POST /users 와 동일 shape)

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `id` | string (uuid) | yes | — |
| `email` | string | yes | — |
| `firstName` `lastName` | string | yes | — |
| `phoneNumber` | string \| null | no | — |
| `country` | string | yes | **버전별** — `1.0.0`: alpha-2(`KR`) / `1.0.1`: alpha-3(`KOR`) · 미제공 `UNKNOWN` |
| `profileImageUrl` | string \| null | no | — |
| `accountStatus` | string (enum `AccountStatus`) | yes | — |
| `lastLoginTime` | string (date-time, UTC) \| null | no | — |
| `createdAt` | string (date-time, UTC) | yes | — |
| `organizations` | array<`UserOrganizationSummaryView`> | yes | 아래 nested |

### Response nested 정의 — `UserOrganizationSummaryView`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `id` | string (uuid) | yes | organization id |
| `name` | string | yes | 시설명 |
| `type` | string (enum `OrganizationType`) | yes | `GYM` `PILATES` `YOGA` `PT_STUDIO` `CROSSFIT` `FUNCTIONAL` `BOXING` `ETC` |
| `role` | string (enum `MembershipRole`) | yes | `OWNER` `ADMIN` `TRAINER` `MEMBER` (owner 매칭 우선) |
| `seatsUsed` | integer | yes | 사용 좌석 수 |
| `planSeats` | integer \| null | no | 플랜 총 좌석 |
| `licenseState` | string | no | license 상태 (예 `ACTIVE`) |

---

## 8. `PATCH` /api/v2/b2b/users/{b2bUserId}

본인 프로필 부분 수정. 모든 필드가 `JsonNullable<T>` — "안 보냄(absent)=변경 없음", "명시적 `null`=지우기 시도". `firstName`·`lastName` 은 가입 필수라 clear 불가 (`UpdateProfileValidator` 가 400).

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | country 를 표시용 **alpha-2** 로 반환 (저장 alpha-3 를 역변환) |
| `1.0.1` **(최신)** | country 를 **alpha-3 저장값 그대로** 반환 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` (country 표기 차이) | yes |
| `Authorization` | `Bearer <jwt>` (본인) | yes |
| `Content-Type` | `application/json` | yes |

### Security

self-check — path `b2bUserId` 가 principal id 와 불일치 시 `403 PERMISSION_DENIED` (`UserError.NotSelf`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "firstName": "종학",
  "phoneNumber": "+821099998888",
  "profileImageUrl": "https://cdn.example.com/p.jpg",
  "country": "KR"
}
```

### `1.0.0` — Response

**200 OK** — `data` 에 갱신된 `UserResponse`. country 는 표시용 **alpha-2** (역변환)

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "email": "owner@example.com",
    "firstName": "종학",
    "lastName": "이",
    "phoneNumber": "+821099998888",
    "country": "KR",
    "profileImageUrl": "https://cdn.example.com/p.jpg",
    "accountStatus": "ACTIVE",
    "lastLoginTime": "2026-07-08T09:00:00Z",
    "createdAt": "2026-07-01T09:00:00Z",
    "organizations": []
  }
}
```

<details>
<summary><b>400 Bad Request</b> — firstName/lastName 을 <code>null</code> 로 clear 시도 (<code>UserError.FieldCannotBeCleared</code>)</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Not Valid" }
```

</details>

<details>
<summary><b>400 Bad Request</b> — name/phone/country 형식 위반 · profileImageUrl 500자 초과</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Not Valid" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — path b2bUserId 가 principal 과 불일치</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 사용자 미존재</summary>

```json
{ "success": false, "code": "USR-404-001", "message": "User not found" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request (요청 shape 동일 — country 수용 규칙만 확대)

```json
{
  "firstName": "종학",
  "phoneNumber": "+821099998888",
  "profileImageUrl": "https://cdn.example.com/p.jpg",
  "country": "UK"
}
```

### `1.0.1` — Response

**200 OK** — country 를 **alpha-3 저장값 그대로** 반환

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "email": "owner@example.com",
    "firstName": "종학",
    "lastName": "이",
    "phoneNumber": "+821099998888",
    "country": "GBR",
    "profileImageUrl": "https://cdn.example.com/p.jpg",
    "accountStatus": "ACTIVE",
    "lastLoginTime": "2026-07-08T09:00:00Z",
    "createdAt": "2026-07-01T09:00:00Z",
    "organizations": []
  }
}
```

400/403/404 는 `1.0.0` 과 동일.

</details>

### Request 필드 정의 — `UpdateProfileRequest` (모든 필드 `JsonNullable<T>`)

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `firstName` | `JsonNullable<String>` | no | `@ValidName(allowEmpty=false, allowNull=true)` (type-arg) · `null` clear 거부 → 400 |
| `lastName` | `JsonNullable<String>` | no | `@ValidName(allowEmpty=false, allowNull=true)` (type-arg) · `null` clear 거부 → 400 |
| `phoneNumber` | `JsonNullable<String>` | no | `@ValidPhoneNumber(allowNull=true)` · clear 허용 |
| `profileImageUrl` | `JsonNullable<String>` | no | `@Size(max=500)` (type-arg) · clear 허용 |
| `country` | `JsonNullable<String>` | no | `@ValidEnum(CountryCode)` — alpha-2/alpha-3/별칭 수용 · `null` clear 시 **`UNKNOWN` 으로 리셋** |

### Response 필드 정의 — `UserResponse`

§GET /users/{b2bUserId} 응답과 동일 shape (`organizations` 포함). country 표기도 버전 규칙 동일 — `1.0.0`: alpha-2 / `1.0.1`: alpha-3.

---

## 9. `POST` /api/v2/b2b/users/{b2bUserId}/password

기존 비밀번호 확인 후 변경. 변경 성공 시 전 세션을 revoke 한다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인) | yes |
| `Content-Type` | `application/json` | yes |

### Security

self-check — path `b2bUserId` 가 principal id 와 불일치 시 `403 PERMISSION_DENIED` (`UserError.NotSelf`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "currentPassword": "OldP@ss1!",
  "newPassword": "NewP@ss2!"
}
```

### `1.0.0` — Response

**204 No Content** — 바디 없음.

<details>
<summary><b>400 Bad Request</b> — newPassword 형식 위반 · 기존과 동일 재사용 (<code>SamePasswordReuse</code>)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter" }
```

</details>

<details>
<summary><b>401 Unauthorized</b> — 기존(current) 비밀번호 불일치 (<code>InvalidCurrentPassword</code>)</summary>

```json
{ "success": false, "code": "AUTH-401-003", "message": "Authentication failed" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — path b2bUserId 가 principal 과 불일치</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 사용자 미존재</summary>

```json
{ "success": false, "code": "USR-404-001", "message": "User not found" }
```

</details>

</details>

### Request 필드 정의 — `ChangePasswordRequest`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`currentPassword`** | string | yes | `@NotBlank` — 불일치 시 401 |
| **`newPassword`** | string | yes | `@ValidPassword(allowEmpty=false, allowNull=false)` — 기존과 동일하면 400 |

---

## 10. `DELETE` /api/v2/b2b/users/{b2bUserId}

본인 계정 소프트 탈퇴 — `DELETION_PENDING` 으로 전환하고 전 세션을 revoke 한다. `reason` 은 감사용(선택).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인) | yes |

### Security

self-check — path `b2bUserId` 가 principal id 와 불일치 시 `403 PERMISSION_DENIED` (`UserError.NotSelf`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `b2bUserId` (uuid) + 선택 쿼리 파라미터 `reason`.

| 파라미터 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `reason` | string (enum `DeletionReason`) | no | `PRICE` `LOW_USAGE` `MISSING_FEATURE` `SERVICE_ISSUE` `SWITCHING_TO_OTHER` `OTHER` — 감사용 |

### `1.0.0` — Response

**204 No Content** — 바디 없음.

<details>
<summary><b>403 Forbidden</b> — path b2bUserId 가 principal 과 불일치</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 사용자 미존재</summary>

```json
{ "success": false, "code": "USR-404-001", "message": "User not found" }
```

</details>

</details>

---

## organizations

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/organizations`](#11-post-apiv2b2borganizations) | **1.0.0, 1.0.1, 1.0.2** | Bearer | 시설(organization) 등록 — 인증된 b2b_user 가 owner 로 자동 지정된다 |
| GET | [`/organizations/search`](#12-get-apiv2b2borganizationssearch) | 1.0.0 | 공개 | `searchable=true` + `status=ACTIVE` 시설 검색 (Carenco 사용… |
| GET | [`/organizations/{id}`](#13-get-apiv2b2borganizationsid) | **1.0.0, 1.0.1, 1.0.2** | 공개 | 단건 시설 조회 |
| PATCH | [`/organizations/{id}`](#14-patch-apiv2b2borganizationsid) | 1.0.0 | Bearer | 부분 수정 — 모든 필드가 `JsonNullable<T>` (absent = 변경 없음, `nu… |
| DELETE | [`/organizations/{id}`](#15-delete-apiv2b2borganizationsid) | 1.0.0 | Bearer | 시설 소프트 삭제 — owner 만 |

---

## 11. `POST` /api/v2/b2b/organizations

시설(organization) 등록 — 인증된 b2b_user 가 owner 로 자동 지정된다. 버전별로 요청/응답이 다르다. `1.0.1`·`1.0.2` 는 `country` + `businessRegistrationNumber` 를 **필수**로 요구하고 응답에 BRN·멤버 통계(stats)를 추가한다. `1.0.2` 는 `1.0.1` 과 동일 shape 이되 응답 `country` 만 ISO 3166-1 alpha-3 로 노출한다.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | country·BRN **선택** — 응답에 BRN·stats 없음, `address.country` alpha-2 |
| `1.0.1` | country·BRN **필수** — 응답에 BRN·`stats` 추가 |
| `1.0.2` **(최신)** | `1.0.1` 과 동일 shape — 응답 country 만 **alpha-3** |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` \| `1.0.2` | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

인증된 b2b_user JWT 필요 (SecurityConfig `anyRequest().authenticated()`). 별도 role 체크 없음 — 호출한 `principal.getUser().getId()` 가 그대로 새 organization 의 `ownerUserId` 로 저장된다. `@PreAuthorize` 애너테이션은 없다.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "name": "강남 필라테스",
  "type": "PILATES",
  "description": "1:1 리포머 전문",
  "phoneNumber": "+821012345678",
  "photoUrl": "https://cdn.example.com/org/1.jpg",
  "country": "KR",
  "postalCode": "06000",
  "regionLevel1": "서울",
  "regionLevel2": "강남구",
  "addressLine1": "테헤란로 1",
  "addressLine2": "3층",
  "searchable": true,
  "inviteCodeEnabled": true,
  "approvalRequired": true
}
```

### `1.0.0` — Response

**201 Created** — `OrganizationResponse` (BRN·stats 미노출, `address.country` 는 표시용 alpha-2)

```json
{
  "success": true,
  "data": {
    "id": "org-uuid",
    "name": "강남 필라테스",
    "type": "PILATES",
    "description": "1:1 리포머 전문",
    "phoneNumber": "+821012345678",
    "photoUrl": "https://cdn.example.com/org/1.jpg",
    "address": {
      "country": "KR",
      "postalCode": "06000",
      "regionLevel1": "서울",
      "regionLevel2": "강남구",
      "addressLine1": "테헤란로 1",
      "addressLine2": "3층"
    },
    "ownerUserId": "b2b-user-uuid",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 0,
    "status": "ACTIVE"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — country 소문자 · BRN 형식 위반 (InvalidInput)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid country: must be ISO-3166 alpha-2 uppercase" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request (= 1.0.0 필드 + `country` 필수 + `businessRegistrationNumber` 필수)

```json
{
  "name": "강남 필라테스",
  "type": "PILATES",
  "description": "1:1 리포머 전문",
  "phoneNumber": "+821012345678",
  "photoUrl": "https://cdn.example.com/org/1.jpg",
  "country": "KR",
  "postalCode": "06000",
  "regionLevel1": "서울",
  "regionLevel2": "강남구",
  "addressLine1": "테헤란로 1",
  "addressLine2": "3층",
  "businessRegistrationNumber": "123-45-67890",
  "searchable": true,
  "inviteCodeEnabled": true,
  "approvalRequired": true
}
```

### `1.0.1` — Response

**201 Created** — `OrganizationResponseV1_0_1` (BRN + `stats` 추가, `address.country` 표시용 alpha-2). 신규 org 라 stats 는 전부 0.

```json
{
  "success": true,
  "data": {
    "id": "org-uuid",
    "name": "강남 필라테스",
    "type": "PILATES",
    "description": "1:1 리포머 전문",
    "phoneNumber": "+821012345678",
    "photoUrl": "https://cdn.example.com/org/1.jpg",
    "address": {
      "country": "KR",
      "postalCode": "06000",
      "regionLevel1": "서울",
      "regionLevel2": "강남구",
      "addressLine1": "테헤란로 1",
      "addressLine2": "3층"
    },
    "businessRegistrationNumber": "123-45-67890",
    "ownerUserId": "b2b-user-uuid",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 0,
    "status": "ACTIVE",
    "stats": {
      "totalMembers": 0,
      "activeMembers": 0,
      "byRole": { "OWNER": 0, "ADMIN": 0, "TRAINER": 0, "MEMBER": 0 },
      "suspended": 0
    }
  }
}
```

<details>
<summary><b>400 Bad Request</b> — country/BRN 누락 (@NotBlank) · BRN 형식 위반</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid businessRegistrationNumber: invalid format for country=KR" }
```

</details>

</details>

<details>
<summary><b><code>1.0.2</code> — Request · Response</b></summary>

### `1.0.2` — Request

`1.0.1` 과 **동일** (`CreateOrganizationRequestV1_0_1` 재사용). `country` + `businessRegistrationNumber` 필수.

```json
{
  "name": "강남 필라테스",
  "type": "PILATES",
  "country": "KR",
  "businessRegistrationNumber": "123-45-67890",
  "searchable": true,
  "inviteCodeEnabled": true,
  "approvalRequired": true
}
```

### `1.0.2` — Response

**201 Created** — `1.0.1` 과 동일 shape, `address.country` 만 ISO 3166-1 **alpha-3** (`KOR`)

```json
{
  "success": true,
  "data": {
    "id": "org-uuid",
    "name": "강남 필라테스",
    "type": "PILATES",
    "address": { "country": "KOR", "postalCode": "06000" },
    "businessRegistrationNumber": "123-45-67890",
    "ownerUserId": "b2b-user-uuid",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 0,
    "status": "ACTIVE",
    "stats": {
      "totalMembers": 0, "activeMembers": 0,
      "byRole": { "OWNER": 0, "ADMIN": 0, "TRAINER": 0, "MEMBER": 0 },
      "suspended": 0
    }
  }
}
```

</details>

### Request 필드 정의 — 버전 대조

| 필드 | 타입 | 1.0.0 | 1.0.1 / 1.0.2 | Validation |
|---|---|---|---|---|
| **`name`** | string | 필수 | 필수 | `@NotBlank @Size(max=200)` |
| **`type`** | enum `OrganizationType` | 필수 | 필수 | `@NotNull` — `GYM` `PILATES` `YOGA` `PT_STUDIO` `CROSSFIT` `FUNCTIONAL` `BOXING` `ETC` |
| `description` | string | 선택 | 선택 | `@Size(max=65535)` |
| `phoneNumber` | string | 선택 | 선택 | `@ValidPhoneNumber(allowNull=true)` (E.164) |
| `photoUrl` | string | 선택 | 선택 | `@Size(max=500)` |
| `country` | string | 선택 | **필수** | 1.0.0 `@Size(min=2,max=2)` / 1.0.1+ `@NotBlank @ValidCountryCode @Size(min=2,max=2)` (alpha-2 대문자) |
| `postalCode` | string | 선택 | 선택 | `@Size(max=20)` |
| `regionLevel1` `regionLevel2` | string | 선택 | 선택 | `@Size(max=100)` |
| `addressLine1` `addressLine2` | string | 선택 | 선택 | `@Size(max=255)` |
| **`businessRegistrationNumber`** | string | **없음** | **필수 (신규)** | `@NotBlank @Size(max=40)` + `BusinessRegistrationNumberValidator.isValid(country, brn)` (country 별 정규식) |
| `searchable` | boolean | 선택 | 선택 | null → `true` (기본) |
| `inviteCodeEnabled` | boolean | 선택 | 선택 | null → `true` (기본) |
| `approvalRequired` | boolean | 선택 | 선택 | null → `true` (기본) |

> `country` 는 저장 시 alpha-3 로 정규화되지만, 해석 불가/UNKNOWN 은 대문자 원본 유지. 응답 표기는 버전별 (1.0.0·1.0.1 = alpha-2, 1.0.2 = alpha-3).

### Response 필드 정의 — 버전 대조

| 필드 | 타입 | 1.0.0 | 1.0.1 | 1.0.2 |
|---|---|---|---|---|
| `id` `name` `type` `description` `phoneNumber` `photoUrl` `ownerUserId` | — | ✅ | ✅ | ✅ |
| `address` | `Address` (`country` `postalCode` `regionLevel1` `regionLevel2` `addressLine1` `addressLine2`) | ✅ (country alpha-2) | ✅ (country alpha-2) | ✅ (country **alpha-3**) |
| `searchable` `inviteCodeEnabled` `approvalRequired` | boolean | ✅ | ✅ | ✅ |
| `seatsUsed` | int | ✅ | ✅ | ✅ |
| `status` | enum `OrganizationStatus` (`ACTIVE`·`SUSPENDED`·`DELETED`) | ✅ | ✅ | ✅ |
| `businessRegistrationNumber` | string \| null | ❌ | ✅ | ✅ |
| `stats` | `Stats` (`totalMembers` `activeMembers` `byRole` `suspended`) | ❌ | ✅ | ✅ |

---

## 12. `GET` /api/v2/b2b/organizations/search

`searchable=true` + `status=ACTIVE` 시설 검색 (Carenco 사용자 가입 전 단계). `type`·`keyword` 옵션 필터, 페이징. 정렬은 `createdAt` DESC 고정.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

**공개** (`permitAll` — `GET /api/v2/b2b/organizations/search`). 인증 불필요.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 기본값 | 비고 |
|---|---|---|---|
| `keyword` | string | — | blank/null 이면 미적용 |
| `type` | enum `OrganizationType` | — | 옵션 필터 |
| `page` | int | `0` | 음수는 0 으로 보정 |
| `size` | int | `20` | ≤0 또는 >100 이면 20 으로 보정 |

### `1.0.0` — Response

**200 OK** — 커스텀 페이지 Map. `items[]` 는 `OrganizationResponse` (v1.0.0 shape)

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "org-uuid",
        "name": "강남 필라테스",
        "type": "PILATES",
        "address": { "country": "KR", "postalCode": "06000" },
        "ownerUserId": "b2b-user-uuid",
        "searchable": true,
        "inviteCodeEnabled": true,
        "approvalRequired": true,
        "seatsUsed": 3,
        "status": "ACTIVE"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 42,
    "totalPages": 3,
    "hasNext": true
  }
}
```

</details>

### Response 필드 정의

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`OrganizationResponse`> | v1.0.0 shape (BRN·stats 없음, country alpha-2) |
| `page` | int | 현재 페이지 (0-base) |
| `size` | int | 페이지 크기 |
| `totalElements` | long | 필터 후 전체 건수 |
| `totalPages` | int | 전체 페이지 수 |
| `hasNext` | boolean | 다음 페이지 존재 |

---

## 13. `GET` /api/v2/b2b/organizations/{id}

단건 시설 조회. `DELETED` 상태는 404. 버전별로 `1.0.0`(기본), `1.0.1`(+ BRN·stats, country alpha-2), `1.0.2`(+ BRN·stats, country **alpha-3**).

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 응답 — BRN·stats 미노출, `address.country` alpha-2 |
| `1.0.1` | BRN·`stats` 노출 |
| `1.0.2` **(최신)** | `1.0.1` 과 동일 shape — 응답 country 만 **alpha-3** |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` \| `1.0.2` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

**공개** (`permitAll` — `GET /api/v2/b2b/organizations/*`). 인증·권한 체크 없음.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `id` 만.

### `1.0.0` — Response

**200 OK** — `OrganizationResponse` (BRN·stats 미노출, country alpha-2)

```json
{
  "success": true,
  "data": {
    "id": "org-uuid",
    "name": "강남 필라테스",
    "type": "PILATES",
    "description": "1:1 리포머 전문",
    "phoneNumber": "+821012345678",
    "photoUrl": null,
    "address": { "country": "KR", "postalCode": "06000", "regionLevel1": "서울", "regionLevel2": "강남구", "addressLine1": "테헤란로 1", "addressLine2": "3층" },
    "ownerUserId": "b2b-user-uuid",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 3,
    "status": "ACTIVE"
  }
}
```

<details>
<summary><b>404 Not Found</b> — 미존재 또는 DELETED (NotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Organization not found: org-uuid" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

바디 없음 — path 의 `id` 만.

### `1.0.1` — Response

**200 OK** — `OrganizationResponseV1_0_1` (v1.0.0 필드 + `businessRegistrationNumber` + `stats`, country alpha-2). stats 는 실제 멤버 집계.

```json
{
  "success": true,
  "data": {
    "id": "org-uuid",
    "name": "강남 필라테스",
    "type": "PILATES",
    "address": { "country": "KR", "postalCode": "06000" },
    "businessRegistrationNumber": "123-45-67890",
    "ownerUserId": "b2b-user-uuid",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": true,
    "seatsUsed": 3,
    "status": "ACTIVE",
    "stats": {
      "totalMembers": 5,
      "activeMembers": 4,
      "byRole": { "OWNER": 1, "ADMIN": 1, "TRAINER": 1, "MEMBER": 1 },
      "suspended": 1
    }
  }
}
```

<details>
<summary><b>404 Not Found</b> — 미존재 또는 DELETED</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Organization not found: org-uuid" }
```

</details>

</details>

<details>
<summary><b><code>1.0.2</code> — Request · Response</b></summary>

### `1.0.2` — Request

바디 없음 — path 의 `id` 만.

### `1.0.2` — Response

**200 OK** — `1.0.1` 과 동일 shape, `address.country` 만 ISO 3166-1 **alpha-3** (`KOR`)

```json
{
  "success": true,
  "data": {
    "id": "org-uuid",
    "name": "강남 필라테스",
    "type": "PILATES",
    "address": { "country": "KOR", "postalCode": "06000" },
    "businessRegistrationNumber": "123-45-67890",
    "ownerUserId": "b2b-user-uuid",
    "searchable": true, "inviteCodeEnabled": true, "approvalRequired": true,
    "seatsUsed": 3, "status": "ACTIVE",
    "stats": { "totalMembers": 5, "activeMembers": 4, "byRole": { "OWNER": 1, "ADMIN": 1, "TRAINER": 1, "MEMBER": 1 }, "suspended": 1 }
  }
}
```

</details>

### Response 필드 정의 — 버전 대조 / `stats`

§POST /organizations 의 Response 필드 대조표와 동일. `stats` 세부.

| 필드 | 타입 | 설명 |
|---|---|---|
| `totalMembers` | int | PENDING + ACTIVE + SUSPENDED (LEFT 제외) |
| `activeMembers` | int | ACTIVE 만 |
| `byRole` | Map<`MembershipRole`, int> | ACTIVE 기준 role 별 (OWNER/ADMIN/TRAINER/MEMBER 키 항상 존재, 0 포함) |
| `suspended` | int | SUSPENDED 카운트 |

---

## 14. `PATCH` /api/v2/b2b/organizations/{id}

부분 수정 — 모든 필드가 `JsonNullable<T>` (absent = 변경 없음, `null` = clear 시도, value = 적용). **owner 만** 가능. `name`·`type`·`searchable`·`inviteCodeEnabled`·`approvalRequired` 는 clear 불가.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

인증 필요 + **owner only** — 서비스가 `org.ownerUserId == principal.getUser().getId()` 를 검증, 불일치 시 `NotOwner` (403). `@PreAuthorize` 없음.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "description": "리뉴얼 오픈",
  "phoneNumber": "+821099998888",
  "approvalRequired": false
}
```

### `1.0.0` — Response

**200 OK** — `OrganizationResponse` (v1.0.0 shape, BRN·stats 없음, country 표시용 alpha-2)

```json
{
  "success": true,
  "data": {
    "id": "org-uuid",
    "name": "강남 필라테스",
    "type": "PILATES",
    "description": "리뉴얼 오픈",
    "phoneNumber": "+821099998888",
    "address": { "country": "KR" },
    "ownerUserId": "b2b-user-uuid",
    "searchable": true,
    "inviteCodeEnabled": true,
    "approvalRequired": false,
    "seatsUsed": 3,
    "status": "ACTIVE"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — clear 불가 필드에 null (FieldCannotBeCleared) · country 소문자 (InvalidInput)</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Field cannot be cleared: name" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — owner 아님 (NotOwner)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User b2b-user-x is not the owner of organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — organization 미존재 (NotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Organization not found: org-uuid" }
```

</details>

</details>

### Request 필드 정의 (`UpdateOrganizationRequest`)

모든 필드 `JsonNullable<T>`. clear(`null`) 가능 여부.

| 필드 | 타입 | clear(`null`) | 규칙 |
|---|---|---|---|
| `name` | string | ❌ 400 | `@Size(max=200)` |
| `type` | enum `OrganizationType` | ❌ 400 | — |
| `searchable` | boolean | ❌ 400 | — |
| `inviteCodeEnabled` | boolean | ❌ 400 | — |
| `approvalRequired` | boolean | ❌ 400 | — |
| `description` | string | ✅ | — |
| `phoneNumber` | string | ✅ | `@ValidPhoneNumber(allowNull=true)` |
| `photoUrl` | string | ✅ | `@Size(max=500)` |
| `country` | string | ✅ | `@ValidCountryCode(allowNull=true)` + 소문자면 InvalidInput 400 |
| `postalCode` | string | ✅ | `@Size(max=20)` |
| `regionLevel1` `regionLevel2` | string | ✅ | `@Size(max=100)` |
| `addressLine1` `addressLine2` | string | ✅ | `@Size(max=255)` |

> clear 거부 5개 필드는 `UpdateOrganizationValidator` 가 진입 첫 줄에서 fail-fast (`JsonNullableChecks.isClearAttempt`). `businessRegistrationNumber` 는 PATCH 대상이 아니다 (요청 DTO 에 없음).

---

## 15. `DELETE` /api/v2/b2b/organizations/{id}

시설 소프트 삭제 — **owner 만**. `status=DELETED` + 모든 멤버십(PENDING/ACTIVE/SUSPENDED) → `LEFT` + 모든 ACTIVE invite code → `REVOKED` cascade. 구독 취소는 outbox 로 payment-service 에 비동기 위임.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |

### Security

인증 필요 + **owner only** (`org.ownerUserId == requesterId`, 불일치 403 `NotOwner`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `id` 만.

### `1.0.0` — Response

**204 No Content** — `data` 없음

```json
{ "success": true, "data": null }
```

<details>
<summary><b>403 Forbidden</b> — owner 아님 (NotOwner)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User b2b-user-x is not the owner of organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 미존재 또는 이미 DELETED (NotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Organization not found: org-uuid" }
```

</details>

</details>

---

## organizations — 멤버/가입

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/organizations/{organizationId}/staff`](#16-post-apiv2b2borganizationsorganizationidstaff) | 1.0.0 | Bearer | OWNER/ADMIN 이 b2b_user 를 `ADMIN` 또는 `TRAINER` 로 직접 등록… |
| POST | [`/organizations/{organizationId}/join-requests`](#17-post-apiv2b2borganizationsorganizationidjoin-requests) | 1.0.0 | 공개 | Carenco 사용자가 검색 결과 시설에 가입 신청 (pull 모델) |
| GET | [`/organizations/{organizationId}/members`](#18-get-apiv2b2borganizationsorganizationidmembers) | 1.0.0 | Bearer | 회원 목록 조회·검색 (트레이너 대시보드) |
| GET | [`/organizations/{organizationId}/members/{memberId}/measurements`](#19-get-apiv2b2borganizationsorganizationidmembersmemberidmeasurements) | 1.0.0 | Bearer | 조직 멤버의 측정 record 페이지 |
| GET | [`/organizations/{organizationId}/members/{memberId}/measurements/summary`](#20-get-apiv2b2borganizationsorganizationidmembersmemberidmeasurementssummary) | 1.0.0 | Bearer | 멤버 측정 요약 (누적 / 당월 / 마지막 측정 스냅샷) |
| GET | [`/organizations/{organizationId}/members/{memberId}/measurements/{recordId}`](#21-get-apiv2b2borganizationsorganizationidmembersmemberidmeasurementsrecordid) | 1.0.0 | Bearer | 멤버 측정 record 단건 (전체 footprint / vision 상세) |

---

## 16. `POST` /api/v2/b2b/organizations/{organizationId}/staff

OWNER/ADMIN 이 b2b_user 를 `ADMIN` 또는 `TRAINER` 로 직접 등록 (B2B 전용 role, 좌석 미소비). `MEMBER` role 은 거부 (search/invite 경로 사용).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

인증 필요 + **OWNER 또는 ADMIN** — `MembershipRoleResolver.resolve(organizationId, requesterId)` 로 해석한 role 에 OWNER/ADMIN 포함 필요. 불일치 시 `NotOwnerOrAdmin` (403).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "b2bUserId": "b2b-user-uuid",
  "role": "TRAINER",
  "memberNumber": "T-0001"
}
```

### `1.0.0` — Response

**201 Created** — `MembershipResponse` (status=ACTIVE, joinedVia=DIRECT_INVITE)

```json
{
  "success": true,
  "data": {
    "id": "mbr-uuid",
    "organizationId": "org-uuid",
    "memberType": "B2B",
    "memberId": "b2b-user-uuid",
    "role": "TRAINER",
    "status": "ACTIVE",
    "memberNumber": "T-0001",
    "joinedVia": "DIRECT_INVITE",
    "inviteCodeId": null,
    "approvedBy": "requester-b2b-uuid",
    "approvedAt": "2026-07-09T09:00:00Z",
    "joinedAt": "2026-07-09T09:00:00Z",
    "leftAt": null,
    "note": null
  }
}
```

<details>
<summary><b>400 Bad Request</b> — role=MEMBER (InvalidRoleAssignment)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Role MEMBER cannot be assigned to memberType B2B (TRAINER/ADMIN/OWNER are B2B-only)" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님 (NotOwnerOrAdmin)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User req-x lacks owner/admin role for organization org-uuid" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 이미 멤버 (AlreadyMember)</summary>

```json
{ "success": false, "code": "INV-409-002", "message": "User b2b-user-uuid already a member of organization org-uuid" }
```

</details>

</details>

### Request 필드 정의 (`AppointStaffRequest`)

| 필드 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`b2bUserId`** | string (uuid) | yes | `@ValidUuid` |
| **`role`** | enum `MembershipRole` | yes | `@NotNull` — `ADMIN` \| `TRAINER` (MEMBER 거부, OWNER 는 등록 불가) |
| `memberNumber` | string | no | blank/null 이면 `MemberNumberGenerator` 가 role 기반 자동 생성 |

---

## 17. `POST` /api/v2/b2b/organizations/{organizationId}/join-requests

Carenco 사용자가 검색 결과 시설에 가입 신청 (pull 모델). organization 이 `searchable=true` 여야 한다. 결과는 항상 `PENDING` (후속 승인 필요).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개, memberId 는 본문으로 식별) | no |
| `Content-Type` | `application/json` | yes |

### Security

**공개** (`permitAll` — `POST /api/v2/b2b/organizations/*/join-requests`). b2b JWT 없음. gateway 가 user-service JWT 를 검증한다는 가정하에 본문 `memberId` 로 식별. memberType 은 서버가 `CARENCO` 로 강제.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "memberId": "carenco-user-uuid", "note": "PT 상담 희망" }
```

### `1.0.0` — Response

**201 Created** — `MembershipResponse` (status=PENDING, role=MEMBER, joinedVia=SEARCH_REQUEST)

```json
{
  "success": true,
  "data": {
    "id": "mbr-uuid",
    "organizationId": "org-uuid",
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
    "note": "PT 상담 희망"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — organization 이 searchable=false (OrganizationNotSearchable)</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Organization not accepting search-based join: org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — organization 미존재 (OrganizationNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Organization not found: org-uuid" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 이미 ACTIVE/PENDING/SUSPENDED 멤버 (AlreadyMember)</summary>

```json
{ "success": false, "code": "INV-409-002", "message": "User carenco-user-uuid already a member of organization org-uuid" }
```

</details>

</details>

### Request 필드 정의 (`JoinRequest`)

| 필드 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`memberId`** | string (uuid) | yes | `@ValidUuid` — carenco user-service ID |
| `note` | string | no | — |

---

## 18. `GET` /api/v2/b2b/organizations/{organizationId}/members

회원 목록 조회·검색 (트레이너 대시보드). membership + user-service 사용자 정보 + measure-service 측정 요약 통합 (enriched view). upstream 장애 시 해당 필드 null (graceful degradation).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |

### Security

인증 필요 + **OWNER / ADMIN / TRAINER** 중 하나 — 미보유 시 `RoleRequired` (403). organization 미존재 시 404.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 기본값 | 비고 |
|---|---|---|---|
| `query` | string | — | 이름(firstName/lastName)·전화번호 부분일치 (user-service 위임) |
| `sort_by` | enum `SortBy` | `REGISTERED_AT` | `NAME` \| `REGISTERED_AT` \| `LATEST_MEASUREMENT` |
| `status` | enum `MembershipStatus` | `ACTIVE` | `PENDING`·`ACTIVE`·`SUSPENDED`·`LEFT` |
| `include_nickname` | boolean | `false` | 검색 시 nickname 포함 여부 |
| `page` | int | `0` | 음수는 0 보정 |
| `size` | int | `20` | ≤0 또는 >100 이면 20 보정 |

> scope 는 `CARENCO` memberType 멤버만 (B2B 스태프 제외).

### `1.0.0` — Response

**200 OK** — `MemberListResponse`

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "membershipId": "mbr-uuid",
        "memberId": "carenco-user-uuid",
        "memberNumber": "M-0001",
        "role": "MEMBER",
        "status": "ACTIVE",
        "joinedAt": "2026-07-01T00:00:00Z",
        "name": "홍길동",
        "nickname": "길동",
        "email": "u@example.com",
        "phoneNumber": "+821012345678",
        "photoUrl": null,
        "birthdate": "1990-01-01",
        "gender": "MALE",
        "countryCode": "KR",
        "height": 175.5,
        "totalMeasurements": 12,
        "currentMonthMeasurements": 3,
        "lastMeasuredAt": "2026-07-08T00:00:00Z",
        "lastWeight": 70.5,
        "lastBodyScore": 82.0
      }
    ],
    "totalCount": 42,
    "totalReports": 130,
    "hasNext": true
  }
}
```

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN/TRAINER 아님 (RoleRequired)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "member list requires OWNER/ADMIN/TRAINER" }
```

</details>

<details>
<summary><b>404 Not Found</b> — organization 미존재 (OrganizationNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Organization not found: org-uuid" }
```

</details>

</details>

### Response 필드 정의 (`MemberListResponse`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `items` | array<`MemberListItemResponse`> | 아래 |
| `totalCount` | int | 필터 후 전체 회원 수 |
| `totalReports` | int | organization 전체(scope) 측정 건수 |
| `hasNext` | boolean | 다음 페이지 존재 |

**`MemberListItemResponse`** — membership 필드는 항상 존재, user/measurement 필드는 upstream 장애·미존재 시 null.

| 필드 | 타입 | 그룹 |
|---|---|---|
| `membershipId` `memberId` `memberNumber` | string | membership (always) |
| `role` | enum `MembershipRole` | membership |
| `status` | enum `MembershipStatus` | membership |
| `joinedAt` | Instant | membership |
| `name` `nickname` `email` `phoneNumber` `photoUrl` `gender` `countryCode` | string \| null | B2C user info |
| `birthdate` | LocalDate \| null | B2C user info |
| `height` | double \| null | B2C user info |
| `totalMeasurements` `currentMonthMeasurements` | int \| null | measurement summary |
| `lastMeasuredAt` | Instant \| null | measurement summary |
| `lastWeight` `lastBodyScore` | double \| null | measurement summary |

---

## 19. `GET` /api/v2/b2b/organizations/{organizationId}/members/{memberId}/measurements

조직 멤버의 측정 record 페이지. TRAINER / OWNER / ADMIN 만 조회. measure-service gRPC 위임. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

요청자는 **OWNER / ADMIN / TRAINER** (`RoleRequired`), 대상 `memberId` 는 해당 org 의 **ACTIVE 멤버** (`CARENCO` 타입, 아니면 `TargetNotActiveMember`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 필수 | 기본값 | 규칙/설명 |
|---|---|---|---|---|
| `from` | string (ISO instant) | no | — | 파싱 실패 시 null (무제한). 예: `2026-07-01T00:00:00Z` |
| `to` | string (ISO instant) | no | — | 파싱 실패 시 null |
| `page` | integer | no | `0` | |
| `size` | integer | no | `20` | |

### `1.0.0` — Response

**200 OK** — `{ items: MeasurementInfo[], totalCount, hasNext }`

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "recordId": "rec-uuid",
        "userId": "user-uuid",
        "measuredDateTime": "2026-07-05T04:00:00Z",
        "timezone": "Asia/Seoul",
        "footprint": { "id": "fp-uuid", "weight": 70.5, "height": 175.0, "age": 34 },
        "front": { "id": "vis-uuid", "type": "FRONT" },
        "side": { "id": "vis-uuid", "type": "SIDE" },
        "score": { "bodyScore": 80.0, "predictedAge": 30.0 },
        "deviceId": "dev-uuid",
        "source": "TRAINER_GUIDED"
      }
    ],
    "totalCount": 47,
    "hasNext": true
  }
}
```

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN/TRAINER 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "requires OWNER/ADMIN/TRAINER role" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 대상이 organization 의 ACTIVE 멤버 아님</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "member-uuid is not an ACTIVE member of organization org-uuid" }
```

</details>

</details>

### Response 필드 정의 — `MeasurementInfo` (page item)

measure-service 의 `RecordDtoV20260113` 과 1:1 정렬 (recordId 만 예외). page 는 gRPC `MeasurementClient.Page` (`items` + `totalCount` + `hasNext`).

| 필드 | 타입 | 설명 |
|---|---|---|
| `recordId` | string | record row id |
| `userId` | string | 측정 대상 user |
| `measuredDateTime` | string (date-time, UTC) | 측정 시각 |
| `timezone` | string (IANA zone id) \| null | 사용자 ZoneId (예: `Asia/Seoul`) |
| `footprint` | `MeasurementFootprintInfo` \| null | 아래 §measurements/{recordId} 참조 |
| `front` `side` | `MeasurementVisionInfo` \| null | 아래 참조 |
| `score` | `{ bodyScore: number, predictedAge: number }` \| null | — |
| `deviceId` | string \| null | 측정 기기 ID |
| `source` | string | `UNSPECIFIED` \| `SELF` \| `TRAINER_GUIDED` \| `IMPORT` |

---

## 20. `GET` /api/v2/b2b/organizations/{organizationId}/members/{memberId}/measurements/summary

멤버 측정 요약 (누적 / 당월 / 마지막 측정 스냅샷). 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

list 와 동일 — 요청자 OWNER/ADMIN/TRAINER + 대상 ACTIVE 멤버.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` · `memberId` 만.

### `1.0.0` — Response

**200 OK** — `MeasurementSummary`

```json
{
  "success": true,
  "data": {
    "userId": "user-uuid",
    "totalRecords": 47,
    "currentMonthRecords": 3,
    "lastMeasuredAt": "2026-07-05T04:00:00Z",
    "lastBodyScore": 80.0,
    "lastWeight": 70.5,
    "lastPredictedAge": 30.0
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — 권한 부족 · 대상 비-ACTIVE 멤버</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

### Response 필드 정의 — `MeasurementSummary`

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string | carenco user uuid |
| `totalRecords` | integer | 누적 측정 횟수 |
| `currentMonthRecords` | integer | 당월 측정 횟수 (month 불일치 시 0) |
| `lastMeasuredAt` | string \| null | 마지막 측정 시각. 이력 없으면 null |
| `lastBodyScore` | number \| null | 마지막 body score |
| `lastWeight` | number \| null | 마지막 체중 |
| `lastPredictedAge` | number \| null | 마지막 predicted age |

---

## 21. `GET` /api/v2/b2b/organizations/{organizationId}/members/{memberId}/measurements/{recordId}

멤버 측정 record 단건 (전체 footprint / vision 상세). 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

list 와 동일 — 요청자 OWNER/ADMIN/TRAINER + 대상 ACTIVE 멤버. record 미존재 시 `RecordNotFound` (404).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` · `memberId` · `recordId`.

### `1.0.0` — Response

**200 OK** — `MeasurementInfo` (footprint / vision keypoints 전체 전개)

```json
{
  "success": true,
  "data": {
    "recordId": "rec-uuid",
    "userId": "user-uuid",
    "measuredDateTime": "2026-07-05T04:00:00Z",
    "timezone": "Asia/Seoul",
    "footprint": {
      "id": "fp-uuid",
      "firstClass": { "type": 1.0, "accuracy": 0.92 },
      "secondaryClass": { "type": 2.0, "accuracy": 0.71 },
      "thirdClass": { "type": 3.0, "accuracy": 0.55 },
      "leftFootDimension": { "length": 255.0, "width": 98.0 },
      "rightFootDimension": { "length": 257.0, "width": 99.0 },
      "imageUrl": "https://.../footprint.png",
      "centerOfGravity": { "x": 0.5, "y": 0.48 },
      "leftPressure": { "top": 10.0, "middle": 20.0, "bottom": 30.0, "total": 60.0 },
      "rightPressure": { "top": 11.0, "middle": 21.0, "bottom": 31.0, "total": 63.0 },
      "weight": 70.5,
      "height": 175.0,
      "age": 34
    },
    "front": {
      "id": "vis-uuid-1",
      "type": "FRONT",
      "angleFace": 1.2, "anglePelvis": 0.8, "angleShoulder": 1.0,
      "classType": 2.0, "accuracy": 0.9,
      "imageUrl": "https://.../front.jpg",
      "keypoints": { "nose": { "x": 0.5, "y": 0.1, "accuracy": 0.99 } }
    },
    "side": { "id": "vis-uuid-2", "type": "SIDE" },
    "score": { "bodyScore": 80.0, "predictedAge": 30.0 },
    "deviceId": "dev-uuid",
    "source": "TRAINER_GUIDED"
  }
}
```

<details>
<summary><b>404 Not Found</b> — record 미존재</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Measurement record not found: rec-uuid" }
```

</details>

</details>

### Response 필드 정의 — nested 타입

**`MeasurementFootprintInfo`** (`footprint`, null 가능)

| 필드 | 타입 |
|---|---|
| `id` | string |
| `firstClass` / `secondaryClass` / `thirdClass` | `{ type: number, accuracy: number }` |
| `leftFootDimension` / `rightFootDimension` | `{ length: number, width: number }` |
| `imageUrl` | string |
| `centerOfGravity` | `{ x: number, y: number }` |
| `leftPressure` / `rightPressure` | `{ top, middle, bottom, total : number }` |
| `weight` / `height` / `age` | number / number / integer |

**`MeasurementVisionInfo`** (`front` / `side`, null 가능)

| 필드 | 타입 |
|---|---|
| `id` | string |
| `type` | string (`FRONT` \| `SIDE`) |
| `angleFace` / `anglePelvis` / `angleShoulder` | number |
| `classType` / `accuracy` | number |
| `imageUrl` | string |
| `keypoints` | `VisionKeypoints` — COCO 17 (nose, leftEye, …, rightAnkle), 각 `{ x, y, accuracy : number }`. null 가능 |

---

## organizations — devices

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/organizations/{organizationId}/devices/preview`](#22-get-apiv2b2borganizationsorganizationiddevicespreview) | 1.0.0 | Bearer | 시리얼 입력 후 등록 전 미리보기 (풀 조회만, claim 안 함) |
| POST | [`/organizations/{organizationId}/devices`](#23-post-apiv2b2borganizationsorganizationiddevices) | **1.0.0, 1.0.1** | Bearer | 시리얼로 device 를 organization 에 등록 (claim) |
| GET | [`/organizations/{organizationId}/devices`](#24-get-apiv2b2borganizationsorganizationiddevices) | **1.0.0, 1.0.1** | Bearer | organization 의 device 목록 |
| GET | [`/organizations/{organizationId}/devices/{deviceId}`](#25-get-apiv2b2borganizationsorganizationiddevicesdeviceid) | **1.0.0, 1.0.1** | Bearer | device 단건 상세 |
| PATCH | [`/organizations/{organizationId}/devices/{deviceId}`](#26-patch-apiv2b2borganizationsorganizationiddevicesdeviceid) | **1.0.0, 1.0.1** | Bearer | device alias 부분 수정 |
| POST | [`/organizations/{organizationId}/devices/{deviceId}/deactivate`](#27-post-apiv2b2borganizationsorganizationiddevicesdeviceiddeactivate) | **1.0.0, 1.0.1** | Bearer | device 일시 정지 (측정 불가) |
| DELETE | [`/organizations/{organizationId}/devices/{deviceId}`](#28-delete-apiv2b2borganizationsorganizationiddevicesdeviceid) | **1.0.0, 1.0.1** | Bearer | 영구 삭제 — REVOKED + 풀 unclaim (일시 정지 deactivate 와 다름) |

---

## 22. `GET` /api/v2/b2b/organizations/{organizationId}/devices/preview

시리얼 입력 후 등록 전 미리보기 (풀 조회만, claim 안 함). UI 의 기기 등록 흐름에서 alias 입력 전 호출. **단일 버전 (`1.0.0`)**.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER / ADMIN** 만 (`OrgRoleRequired`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음. 쿼리 파라미터 `serial` (필수).

```
GET /api/v2/b2b/organizations/{organizationId}/devices/preview?serial=FS2-SN-001234
```

### `1.0.0` — Response

**200 OK** — `PreviewDeviceResponse`. `found=false` 면 `hardwareSerial` 외 모두 null

```json
{
  "success": true,
  "data": {
    "found": true,
    "alreadyClaimed": false,
    "hardwareSerial": "FS2-SN-001234",
    "deviceType": "SCALE2",
    "firmwareVersion": "1.4.2"
  }
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `found` | boolean | 풀에 존재 여부 |
| `alreadyClaimed` | boolean | 이미 다른 조직이 클레임했는지 |
| `hardwareSerial` | string | 조회한 시리얼 |
| `deviceType` | string \| null | 기기 유형 |
| `firmwareVersion` | string \| null | 보고 펌웨어 |

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "device action requires OWNER/ADMIN" }
```

</details>

</details>

---

## 23. `POST` /api/v2/b2b/organizations/{organizationId}/devices

시리얼로 device 를 organization 에 등록 (claim). device-service `DeviceManagement` gRPC 위임. 응답 shape 이 버전별로 다르다 — `1.0.1` 은 `deviceType` · `deviceNumber` 추가.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 응답 |
| `1.0.1` **(최신)** | 응답에 `deviceType` · `deviceNumber` 추가 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` (gateway-enforced) | yes |
| `Content-Type` | `application/json` | yes |

### Security

`@AuthenticationPrincipal CustomB2bUserDetails` (게이트웨이 JWT). 메서드 시큐리티 애너테이션(`@PreAuthorize`) 없음 — 권한은 `OrgDeviceService` 가 `MembershipRoleResolver` 로 imperative 검증. 등록은 **OWNER / ADMIN** 만 (아니면 `OrgRoleRequired`), 그리고 license 가 `allowDeviceRegister` 통과해야 함 (ACTIVE / GRACE).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "serialNumber": "FS2-SN-001234",
  "alias": "1번 인바디"
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`serialNumber`** | string | **yes** | `@NotBlank @Size(max=64)` — 하드웨어 시리얼 |
| `alias` | string | no | `@Size(max=255)` — 조직이 부여한 별명 |

### `1.0.0` — Response

**201 Created** — `DeviceResponse` (deviceType / deviceNumber 미노출)

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-uuid",
    "organizationId": "org-uuid",
    "serialNumber": "FS2-SN-001234",
    "alias": "1번 인바디",
    "status": "ACTIVE",
    "registeredAt": "2026-07-01T00:00:00Z",
    "registeredBy": "b2buser-uuid",
    "deactivatedAt": null,
    "deactivatedBy": null,
    "deactivationReason": null,
    "lastUsedAt": null,
    "batteryLevel": null,
    "batteryReportedAt": null
  }
}
```

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "device action requires OWNER/ADMIN" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — license 비활성 (write 불가)</summary>

```json
{ "success": false, "code": "LIC-403-001", "message": "License is not active" }
```

</details>

<details>
<summary><b>4xx / 5xx</b> — device-service gRPC propagate (serial 형식 오류 <code>DEV-400-001</code> · 중복 <code>DEV-409-001</code> 등)</summary>

`deviceClient.registerDevice` 는 device-service `DeviceManagement` gRPC 호출. 시리얼 검증 · 중복 · 풀 클레임 실패는 device-service 의 `ErrorCode` 가 propagate 된다.

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request (= 1.0.0 과 동일 body)

```json
{
  "serialNumber": "FS2-SN-001234",
  "alias": "1번 인바디"
}
```

동일한 `RegisterDeviceRequest`. 요청 필드 변화 없음.

### `1.0.1` — Response

**201 Created** — 1.0.0 shape + `deviceType` + `deviceNumber` (`DeviceResponseV1_0_1`)

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-uuid",
    "organizationId": "org-uuid",
    "serialNumber": "FS2-SN-001234",
    "alias": "1번 인바디",
    "status": "ACTIVE",
    "registeredAt": "2026-07-01T00:00:00Z",
    "registeredBy": "b2buser-uuid",
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

</details>

### Response 필드 정의 — `DeviceResponse` / `DeviceResponseV1_0_1`

| 필드 | 타입 | 버전 | 설명 |
|---|---|---|---|
| `deviceId` | string | 전 버전 | device-service PK |
| `organizationId` | string | 전 버전 | 소유 organization |
| `serialNumber` | string | 전 버전 | 하드웨어 시리얼 |
| `alias` | string \| null | 전 버전 | 조직 부여 별명 |
| `status` | enum | 전 버전 | `ACTIVE` \| `INACTIVE` (b2b 는 2 상태로 단순화) |
| `registeredAt` | string (date-time) | 전 버전 | 등록 시각 |
| `registeredBy` | string | 전 버전 | 등록한 b2b_users.id |
| `deactivatedAt` | string \| null | 전 버전 | 비활성 시각 (null = 활성) |
| `deactivatedBy` | string \| null | 전 버전 | 비활성 처리자 |
| `deactivationReason` | string \| null | 전 버전 | 비활성 사유 |
| `lastUsedAt` | string \| null | 전 버전 | 마지막 측정 시각 |
| `batteryLevel` | integer \| null | 전 버전 | 배터리 잔량 |
| `batteryReportedAt` | string \| null | 전 버전 | 배터리 보고 시각 |
| `deviceType` | string \| null | **1.0.1만** | DeviceType enum 문자열 (예: `SCALE2`). 빈 문자열 → null |
| `deviceNumber` | string \| null | **1.0.1만** | 조직 친화 번호 (예: `FS2-001`). 빈 문자열 → null |

---

## 24. `GET` /api/v2/b2b/organizations/{organizationId}/devices

organization 의 device 목록. 응답 item 이 버전별로 다르다 (`1.0.1` = deviceType / deviceNumber 추가).

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 응답 |
| `1.0.1` **(최신)** | 응답에 `deviceType` · `deviceNumber` 추가 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

멤버(role 이 하나라도 있음)만 조회 가능. role 이 비면 `NotMember`. `@PreAuthorize` 없음 — 서비스에서 imperative 검증.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `keyword` | string | no | serial / alias 부분 일치 |
| `status` | enum | no | `ACTIVE` \| `INACTIVE` (미지정 = 전체) |
| `sortBy` | string | no | `name` \| `registered_at` \| `last_used` (기본 `registered_at` desc) |

### `1.0.0` — Response

**200 OK** — `{ items: DeviceResponse[] }`

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "deviceId": "dev-uuid",
        "organizationId": "org-uuid",
        "serialNumber": "FS2-SN-001234",
        "alias": "1번 인바디",
        "status": "ACTIVE",
        "registeredAt": "2026-07-01T00:00:00Z",
        "registeredBy": "b2buser-uuid",
        "deactivatedAt": null,
        "deactivatedBy": null,
        "deactivationReason": null,
        "lastUsedAt": "2026-07-05T04:00:00Z",
        "batteryLevel": 87,
        "batteryReportedAt": "2026-07-05T04:00:00Z"
      }
    ]
  }
}
```

<details>
<summary><b>403 Forbidden</b> — organization 멤버 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "not a member of organization" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

바디 없음. 쿼리 파라미터는 1.0.0 과 동일 (`keyword` · `status` · `sortBy`).

### `1.0.1` — Response

**200 OK** — `{ items: DeviceResponseV1_0_1[] }`. 각 item 에 `deviceType` · `deviceNumber` 추가

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "deviceId": "dev-uuid",
        "organizationId": "org-uuid",
        "serialNumber": "FS2-SN-001234",
        "alias": "1번 인바디",
        "status": "ACTIVE",
        "registeredAt": "2026-07-01T00:00:00Z",
        "registeredBy": "b2buser-uuid",
        "deactivatedAt": null,
        "deactivatedBy": null,
        "deactivationReason": null,
        "lastUsedAt": "2026-07-05T04:00:00Z",
        "batteryLevel": 87,
        "batteryReportedAt": "2026-07-05T04:00:00Z",
        "deviceType": "SCALE2",
        "deviceNumber": "FS2-001"
      }
    ]
  }
}
```

item 필드 정의는 POST §device 응답 표 참조.

</details>

---

## 25. `GET` /api/v2/b2b/organizations/{organizationId}/devices/{deviceId}

device 단건 상세. organization scope 벗어난 deviceId 는 404. `1.0.1` 은 deviceType / deviceNumber 포함.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 응답 |
| `1.0.1` **(최신)** | 응답에 `deviceType` · `deviceNumber` 추가 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

멤버만 조회 (`NotMember`). deviceId 가 해당 org 소속 아니면 `DeviceNotInOrg` (404).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` · `deviceId` 만.

### `1.0.0` — Response

**200 OK** — `DeviceResponse` (POST §device 1.0.0 shape 와 동일)

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-uuid",
    "organizationId": "org-uuid",
    "serialNumber": "FS2-SN-001234",
    "alias": "1번 인바디",
    "status": "ACTIVE",
    "registeredAt": "2026-07-01T00:00:00Z",
    "registeredBy": "b2buser-uuid",
    "deactivatedAt": null,
    "deactivatedBy": null,
    "deactivationReason": null,
    "lastUsedAt": "2026-07-05T04:00:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-07-05T04:00:00Z"
  }
}
```

<details>
<summary><b>404 Not Found</b> — device 가 organization scope 에 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Device not found in organization scope: dev-uuid" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — organization 멤버 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

바디 없음 — path 만.

### `1.0.1` — Response

**200 OK** — 1.0.0 shape + `deviceType` + `deviceNumber` (`DeviceResponseV1_0_1`)

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-uuid",
    "organizationId": "org-uuid",
    "serialNumber": "FS2-SN-001234",
    "alias": "1번 인바디",
    "status": "ACTIVE",
    "registeredAt": "2026-07-01T00:00:00Z",
    "registeredBy": "b2buser-uuid",
    "deactivatedAt": null,
    "deactivatedBy": null,
    "deactivationReason": null,
    "lastUsedAt": "2026-07-05T04:00:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-07-05T04:00:00Z",
    "deviceType": "SCALE2",
    "deviceNumber": "FS2-001"
  }
}
```

</details>

---

## 26. `PATCH` /api/v2/b2b/organizations/{organizationId}/devices/{deviceId}

device alias 부분 수정. `alias` 만 변경 가능. `1.0.1` 은 deviceType / deviceNumber 포함 응답.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 응답 |
| `1.0.1` **(최신)** | 응답에 `deviceType` · `deviceNumber` 추가 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

**OWNER / ADMIN** 만 (`OrgRoleRequired`). scope 밖 device 는 `DeviceNotInOrg`.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`alias` 는 `JsonNullable<String>` — 안 보냄(absent) = 변경 없음, 명시적 `null` = 지우기 시도(거부). device-service gRPC 가 `alias=null` 을 "변경 없음" 으로 해석하므로 clear 미지원.

```json
{ "alias": "2번 인바디" }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `alias` | `JsonNullable<String>` | no | `@Size(max=255)` (type-argument 자리). `null` = clear 시도 → `FieldCannotBeCleared` (400). `undefined` = 변경 없음 |

### `1.0.0` — Response

**200 OK** — `DeviceResponse` (alias 변경 후 상태, 또는 undefined 면 현재 상태 그대로)

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-uuid",
    "organizationId": "org-uuid",
    "serialNumber": "FS2-SN-001234",
    "alias": "2번 인바디",
    "status": "ACTIVE",
    "registeredAt": "2026-07-01T00:00:00Z",
    "registeredBy": "b2buser-uuid",
    "deactivatedAt": null,
    "deactivatedBy": null,
    "deactivationReason": null,
    "lastUsedAt": "2026-07-05T04:00:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-07-05T04:00:00Z"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>alias: null</code> 지우기 거부</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Not Valid", "error": "Field cannot be cleared: alias" }
```

`UpdateDeviceValidator` 가 `JsonNullableChecks.isClearAttempt(alias)` 로 진입 첫 줄에서 차단.

</details>

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

<details>
<summary><b>404 Not Found</b> — device 가 organization scope 에 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

동일 `UpdateDeviceRequest` (`alias` JsonNullable). 요청 변화 없음.

### `1.0.1` — Response

**200 OK** — 1.0.0 shape + `deviceType` + `deviceNumber`.

</details>

---

## 27. `POST` /api/v2/b2b/organizations/{organizationId}/devices/{deviceId}/deactivate

device 일시 정지 (측정 불가). 영구 삭제 DELETE 와 다름 — claim 유지. `1.0.1` 은 deviceType / deviceNumber 포함 응답.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 응답 |
| `1.0.1` **(최신)** | 응답에 `deviceType` · `deviceNumber` 추가 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | body 전송 시 |

### Security

**OWNER / ADMIN** 만. scope 밖 device 는 `DeviceNotInOrg`.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 선택 (`@RequestBody(required = false)`).

```json
{ "reason": "배터리 교체 대기" }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `reason` | string | no | `@Size(max=500)` — 비활성 사유 |

### `1.0.0` — Response

**200 OK** — deactivate 후 `DeviceResponse` (status = INACTIVE, deactivatedAt/By/Reason 채워짐)

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-uuid",
    "organizationId": "org-uuid",
    "serialNumber": "FS2-SN-001234",
    "alias": "1번 인바디",
    "status": "INACTIVE",
    "registeredAt": "2026-07-01T00:00:00Z",
    "registeredBy": "b2buser-uuid",
    "deactivatedAt": "2026-07-09T00:00:00Z",
    "deactivatedBy": "b2buser-uuid",
    "deactivationReason": "배터리 교체 대기",
    "lastUsedAt": "2026-07-05T04:00:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-07-05T04:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — OWNER/ADMIN 아님 · scope 밖 device</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

바디 선택 (`DeactivateDeviceRequest`). 1.0.0 과 동일.

### `1.0.1` — Response

**200 OK** — 1.0.0 shape + `deviceType` + `deviceNumber`.

</details>

---

## 28. `DELETE` /api/v2/b2b/organizations/{organizationId}/devices/{deviceId}

영구 삭제 — REVOKED + 풀 unclaim (일시 정지 deactivate 와 다름). 동일 (mac, serial) 을 다른 조직이 재등록 가능해짐. `1.0.1` 은 deviceType / deviceNumber 포함 응답.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 응답 |
| `1.0.1` **(최신)** | 응답에 `deviceType` · `deviceNumber` 추가 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER / ADMIN** 만. scope 밖 device 는 `DeviceNotInOrg`.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음. 쿼리 파라미터 `reason` (선택).

```
DELETE /api/v2/b2b/organizations/{organizationId}/devices/{deviceId}?reason=폐기
```

### `1.0.0` — Response

**200 OK** — unregister 후 최종 `DeviceInfo` 를 `DeviceResponse` 로 반환

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-uuid",
    "organizationId": "org-uuid",
    "serialNumber": "FS2-SN-001234",
    "alias": "1번 인바디",
    "status": "INACTIVE",
    "registeredAt": "2026-07-01T00:00:00Z",
    "registeredBy": "b2buser-uuid",
    "deactivatedAt": "2026-07-09T00:00:00Z",
    "deactivatedBy": "b2buser-uuid",
    "deactivationReason": "폐기",
    "lastUsedAt": "2026-07-05T04:00:00Z",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-07-05T04:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — OWNER/ADMIN 아님 · scope 밖 device</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

바디 없음. 쿼리 파라미터 `reason` (선택). 1.0.0 과 동일.

### `1.0.1` — Response

**200 OK** — 1.0.0 shape + `deviceType` + `deviceNumber`.

</details>

---

## memberships / b2c

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/availability`](#29-get-apiv2b2bavailability) | 1.0.0 | 공개 | b2b 가입 전 이메일 등 필드 값의 사용 가능 여부(중복 아님)를 확인한다 |
| GET | [`/b2c-members/{carencoUserId}/organizations`](#30-get-apiv2b2bb2c-memberscarencouseridorganizations) | 1.0.0 | 공개 | B2C(carenco) 회원이 가입한 b2b 시설 목록 조회 |
| POST | [`/memberships/{membershipId}/approve`](#31-post-apiv2b2bmembershipsmembershipidapprove) | 1.0.0 | Bearer | PENDING 멤버십을 OWNER/ADMIN 이 승인 → 좌석 점유(license/seat gu… |
| POST | [`/memberships/{membershipId}/reject`](#32-post-apiv2b2bmembershipsmembershipidreject) | 1.0.0 | Bearer | PENDING 멤버십을 OWNER/ADMIN 이 거절 → `LEFT` (history 보존, a… |
| POST | [`/memberships/{membershipId}/suspend`](#33-post-apiv2b2bmembershipsmembershipidsuspend) | 1.0.0 | Bearer | ACTIVE 멤버십을 OWNER/ADMIN 이 일시 정지 → `SUSPENDED` (좌석 점유 … |
| POST | [`/memberships/{membershipId}/leave`](#34-post-apiv2b2bmembershipsmembershipidleave) | 1.0.0 | 공개 | 멤버 본인 탈퇴 → 좌석 해제 후 `LEFT` |

---

## 29. `GET` /api/v2/b2b/availability

b2b 가입 전 이메일 등 필드 값의 사용 가능 여부(중복 아님)를 확인한다. user-service 의 `/api/v2/availability` 와 동일 shape.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개 엔드포인트) | no |

### Security

`@PreAuthorize("permitAll()")` — 가입 전 미인증 호출.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 `field`(필수, `@ValidEnum(DuplicationField)`) · `value`(필수).

```
GET /api/v2/b2b/availability?field=EMAIL&value=owner@example.com
```

| 파라미터 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`field`** | string (enum `DuplicationField`) | yes | `@ValidEnum(DuplicationField)` — 예 `EMAIL`. 서버가 `toUpperCase()` 후 `valueOf` |
| **`value`** | string | yes | 조회 값 |

### `1.0.0` — Response

**200 OK** — `data` 에 `AvailabilityResponse`

```json
{
  "success": true,
  "data": {
    "field": "EMAIL",
    "value": "owner@example.com",
    "available": true
  }
}
```

<details>
<summary><b>400 Bad Request</b> — field 가 DuplicationField enum 외 값</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: field" }
```

</details>

</details>

### Response 필드 정의 — `AvailabilityResponse`

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `field` | string (enum `DuplicationField`) | yes | 조회 대상 |
| `value` | string | yes | 조회 값 |
| `available` | boolean | yes | true = 사용 가능(중복 없음), false = 이미 점유 |

---

## 30. `GET` /api/v2/b2b/b2c-members/{carencoUserId}/organizations

B2C(carenco) 회원이 가입한 b2b 시설 목록 조회. b2c 앱이 user-service 토큰으로 직접 호출하며, 각 시설의 role / status / memberNumber 를 반환한다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (현재 permitAll, internal MSA 호출 가정) | no |

### Security

`@PreAuthorize("permitAll()")` — 토큰 sub 와 path `carencoUserId` 일치 검증은 gateway 또는 후속 작업으로 위임 (현재는 internal MSA 호출 환경 가정).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `carencoUserId` (uuid, `@ValidUuid`) 만.

### `1.0.0` — Response

**200 OK** — `data` 에 `B2cMemberOrganizationView[]`

```json
{
  "success": true,
  "data": [
    {
      "organizationId": "org-uuid",
      "name": "강남 필라테스",
      "type": "PILATES",
      "photoUrl": null,
      "role": "MEMBER",
      "status": "ACTIVE",
      "memberNumber": "M-000123",
      "joinedAt": "2026-06-01T09:00:00Z",
      "leftAt": null
    }
  ]
}
```

<details>
<summary><b>400 Bad Request</b> — carencoUserId 가 uuid 형식 위반 (<code>@ValidUuid</code>)</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Not Valid" }
```

</details>

</details>

### Request 필드 정의

바디 없음. path `carencoUserId` (uuid, `@ValidUuid`) 만.

### Response 필드 정의 — `B2cMemberOrganizationView` (배열 element)

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| `organizationId` | string (uuid) | yes | — |
| `name` | string | yes | 시설명 |
| `type` | string (enum `OrganizationType`) | yes | `GYM` `PILATES` `YOGA` `PT_STUDIO` `CROSSFIT` `FUNCTIONAL` `BOXING` `ETC` |
| `photoUrl` | string \| null | no | 시설 사진 |
| `role` | string (enum `MembershipRole`) | yes | `OWNER` `ADMIN` `TRAINER` `MEMBER` |
| `status` | string (enum `MembershipStatus`) | yes | `PENDING` `ACTIVE` `SUSPENDED` `LEFT` |
| `memberNumber` | string | no | 시설 내 회원번호 |
| `joinedAt` | string (date-time, UTC) | yes | 가입 시각 |
| `leftAt` | string (date-time, UTC) \| null | no | 탈퇴 시각 (미탈퇴 시 null) |

---

## 31. `POST` /api/v2/b2b/memberships/{membershipId}/approve

PENDING 멤버십을 OWNER/ADMIN 이 승인 → 좌석 점유(license/seat guard) 후 `ACTIVE`. memberNumber 미부여 시 자동 생성.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |

### Security

인증 필요 + **OWNER/ADMIN** — 컨트롤러가 membership 의 organizationId 를 조회해 `MembershipRoleResolver` 로 role 해석, 서비스가 OWNER/ADMIN 검증. 미보유 시 `NotOwnerOrAdmin` (403).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `membershipId` 만.

### `1.0.0` — Response

**200 OK** — `MembershipResponse` (status=ACTIVE, approvedBy/approvedAt/joinedAt 채워짐)

```json
{
  "success": true,
  "data": {
    "id": "mbr-uuid",
    "organizationId": "org-uuid",
    "memberType": "CARENCO",
    "memberId": "carenco-user-uuid",
    "role": "MEMBER",
    "status": "ACTIVE",
    "memberNumber": "M-0001",
    "joinedVia": "SEARCH_REQUEST",
    "inviteCodeId": null,
    "approvedBy": "approver-b2b-uuid",
    "approvedAt": "2026-07-09T09:00:00Z",
    "joinedAt": "2026-07-09T09:00:00Z",
    "leftAt": null,
    "note": null
  }
}
```

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님 (NotOwnerOrAdmin)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User req-x lacks owner/admin role for organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — membership 미존재 (MembershipNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Membership not found: mbr-uuid" }
```

</details>

<details>
<summary><b>409 Conflict</b> — PENDING 아님 (InvalidStateTransition)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Membership mbr-uuid cannot transition ACTIVE → ACTIVE" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 좌석 부족/라이선스 비활성 (SeatConsumeFailed)</summary>

```json
{ "success": false, "code": "LIC-409-001", "message": "Seat consume failed: ..." }
```

</details>

</details>

---

## 32. `POST` /api/v2/b2b/memberships/{membershipId}/reject

PENDING 멤버십을 OWNER/ADMIN 이 거절 → `LEFT` (history 보존, archive outbox enqueue).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |

### Security

인증 필요 + **OWNER/ADMIN** (approve 와 동일 해석). 미보유 시 `NotOwnerOrAdmin` (403).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `membershipId` 만.

### `1.0.0` — Response

**200 OK** — `MembershipResponse` (status=LEFT, leftAt 채워짐)

```json
{
  "success": true,
  "data": {
    "id": "mbr-uuid",
    "organizationId": "org-uuid",
    "memberType": "CARENCO",
    "memberId": "carenco-user-uuid",
    "role": "MEMBER",
    "status": "LEFT",
    "joinedVia": "SEARCH_REQUEST",
    "approvedBy": null,
    "approvedAt": null,
    "joinedAt": null,
    "leftAt": "2026-07-09T09:00:00Z",
    "note": null
  }
}
```

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님 (NotOwnerOrAdmin)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User req-x lacks owner/admin role for organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — membership 미존재 (MembershipNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Membership not found: mbr-uuid" }
```

</details>

<details>
<summary><b>409 Conflict</b> — PENDING 아님 (InvalidStateTransition)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Membership mbr-uuid cannot transition ACTIVE → LEFT" }
```

</details>

</details>

---

## 33. `POST` /api/v2/b2b/memberships/{membershipId}/suspend

ACTIVE 멤버십을 OWNER/ADMIN 이 일시 정지 → `SUSPENDED` (좌석 점유 유지). optional `reason` 을 note 로 기록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |
| `Content-Type` | `application/json` | reason 전송 시 |

### Security

인증 필요 + **OWNER/ADMIN**. 미보유 시 `NotOwnerOrAdmin` (403).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 선택 (`required=false`). reason 없으면 바디 생략 가능.

```json
{ "reason": "미납" }
```

### `1.0.0` — Response

**200 OK** — `MembershipResponse` (status=SUSPENDED)

```json
{
  "success": true,
  "data": {
    "id": "mbr-uuid",
    "organizationId": "org-uuid",
    "memberType": "CARENCO",
    "memberId": "carenco-user-uuid",
    "role": "MEMBER",
    "status": "SUSPENDED",
    "memberNumber": "M-0001",
    "joinedVia": "SEARCH_REQUEST",
    "approvedBy": "approver-b2b-uuid",
    "approvedAt": "2026-07-01T00:00:00Z",
    "joinedAt": "2026-07-01T00:00:00Z",
    "leftAt": null,
    "note": "미납"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님 (NotOwnerOrAdmin)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User req-x lacks owner/admin role for organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — membership 미존재 (MembershipNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Membership not found: mbr-uuid" }
```

</details>

<details>
<summary><b>409 Conflict</b> — ACTIVE 아님 (InvalidStateTransition)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Membership mbr-uuid cannot transition PENDING → SUSPENDED" }
```

</details>

</details>

### Request 필드 정의 (`SuspendRequest`)

| 필드 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| `reason` | string | no | 바디 자체 optional. note 로 저장 |

---

## 34. `POST` /api/v2/b2b/memberships/{membershipId}/leave

멤버 본인 탈퇴 → 좌석 해제 후 `LEFT`. 본문 `requesterId` 가 membership 의 `memberId` 와 일치해야 한다 (본인만, 강제 탈퇴 미지원).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개, requesterId 는 본문으로 식별) | no |
| `Content-Type` | `application/json` | yes |

### Security

**공개** (`permitAll` — `POST /api/v2/b2b/memberships/*/leave`). 본문 `requesterId != membership.memberId` 이면 `NotOwnerOrAdmin` (403) — 즉 **본인만** 탈퇴 가능.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "requesterId": "carenco-user-uuid" }
```

### `1.0.0` — Response

**200 OK** — `MembershipResponse` (status=LEFT, leftAt 채워짐)

```json
{
  "success": true,
  "data": {
    "id": "mbr-uuid",
    "organizationId": "org-uuid",
    "memberType": "CARENCO",
    "memberId": "carenco-user-uuid",
    "role": "MEMBER",
    "status": "LEFT",
    "joinedVia": "SEARCH_REQUEST",
    "leftAt": "2026-07-09T09:00:00Z",
    "note": null
  }
}
```

<details>
<summary><b>403 Forbidden</b> — 본인 아님 (NotOwnerOrAdmin)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User other-uuid lacks owner/admin role for organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — membership 미존재 (MembershipNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Membership not found: mbr-uuid" }
```

</details>

</details>

### Request 필드 정의 (`LeaveRequest`)

| 필드 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`requesterId`** | string (uuid) | yes | `@ValidUuid` — membership.memberId 와 일치해야 함 |

---

## invite-codes

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/organizations/{organizationId}/invite-codes`](#35-post-apiv2b2borganizationsorganizationidinvite-codes) | 1.0.0 | Bearer | 초대 코드 발급 — OWNER/ADMIN |
| GET | [`/organizations/{organizationId}/invite-codes`](#36-get-apiv2b2borganizationsorganizationidinvite-codes) | 1.0.0 | Bearer | organization 의 `ACTIVE` 초대 코드 목록 |
| POST | [`/invite-codes/redeem`](#37-post-apiv2b2binvite-codesredeem) | 1.0.0 | 공개 | Carenco 사용자가 코드를 입력하여 가입 |
| POST | [`/invite-codes/{inviteCodeId}/revoke`](#38-post-apiv2b2binvite-codesinvitecodeidrevoke) | 1.0.0 | Bearer | 초대 코드 폐기 — OWNER/ADMIN |

---

## 35. `POST` /api/v2/b2b/organizations/{organizationId}/invite-codes

초대 코드 발급 — OWNER/ADMIN. organization 의 `inviteCodeEnabled=true` + license(ACTIVE/GRACE) 필요. TTL 48시간.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

인증 필요 + **OWNER/ADMIN** — `MembershipRoleResolver` 로 해석, 미보유 시 `NotOwnerOrAdmin` (403).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "description": "7월 프로모션", "codeLength": 8 }
```

### `1.0.0` — Response

**201 Created** — `InviteCodeResponse` (status=ACTIVE)

```json
{
  "success": true,
  "data": {
    "id": "code-uuid",
    "organizationId": "org-uuid",
    "code": "A1B2C3D4",
    "description": "7월 프로모션",
    "expiresAt": "2026-07-11T09:00:00Z",
    "usedAt": null,
    "status": "ACTIVE",
    "createdBy": "b2b-user-uuid"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — inviteCodeEnabled=false (InviteCodeDisabled)</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Invite code disabled for organization: org-uuid" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님 (NotOwnerOrAdmin)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User req-x lacks owner/admin role for organization org-uuid" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — license 비활성 (LicenseBlocked)</summary>

```json
{ "success": false, "code": "LIC-403-001", "message": "License blocks invite code op on organization=org-uuid: ..." }
```

</details>

<details>
<summary><b>404 Not Found</b> — organization 미존재 (OrganizationNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Organization not found: org-uuid" }
```

</details>

</details>

### Request 필드 정의 (`IssueInviteCodeRequest`)

| 필드 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| `description` | string | no | `@Size(max=200)` |
| `codeLength` | int | no | `@Min(6) @Max(12)` — null 이면 기본 `8` |

---

## 36. `GET` /api/v2/b2b/organizations/{organizationId}/invite-codes

organization 의 `ACTIVE` 초대 코드 목록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |

### Security

인증 필요 (`anyRequest().authenticated()` — `/organizations/*/invite-codes` 는 공개 목록에 미포함). 다만 컨트롤러/서비스에 **role 체크 없음** — 인증된 b2b_user 면 누구나 조회 가능.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — `items[]` 는 `InviteCodeResponse`

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "code-uuid",
        "organizationId": "org-uuid",
        "code": "A1B2C3D4",
        "description": "7월 프로모션",
        "expiresAt": "2026-07-11T09:00:00Z",
        "usedAt": null,
        "status": "ACTIVE",
        "createdBy": "b2b-user-uuid"
      }
    ]
  }
}
```

</details>

### Response 필드 정의 (`InviteCodeResponse`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `organizationId` | string (uuid) | — |
| `code` | string | 초대 코드 문자열 |
| `description` | string \| null | — |
| `expiresAt` | Instant | 만료 시각 (발급 + 48h) |
| `usedAt` | Instant \| null | 사용 시각 |
| `status` | enum `InviteCodeStatus` | `ACTIVE`·`USED`·`EXPIRED`·`REVOKED` |
| `createdBy` | string (uuid) | 발급자 b2b_user |

---

## 37. `POST` /api/v2/b2b/invite-codes/redeem

Carenco 사용자가 코드를 입력하여 가입. 코드 ACTIVE + 미만료 검증 → membership PENDING 생성 → 좌석 점유 시 ACTIVE → 코드 USED 처리.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개, memberId 는 본문으로 식별) | no |
| `Content-Type` | `application/json` | yes |

### Security

**공개** (`permitAll` — `POST /api/v2/b2b/invite-codes/redeem`). 본문 `memberId` 로 식별, memberType 은 `CARENCO` 고정.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "code": "A1B2C3D4", "memberId": "carenco-user-uuid" }
```

### `1.0.0` — Response

**200 OK** — 커스텀 Map (`inviteCode` + membership 요약)

```json
{
  "success": true,
  "data": {
    "inviteCode": {
      "id": "code-uuid",
      "organizationId": "org-uuid",
      "code": "A1B2C3D4",
      "description": "7월 프로모션",
      "expiresAt": "2026-07-11T09:00:00Z",
      "usedAt": "2026-07-09T09:00:00Z",
      "status": "USED",
      "createdBy": "b2b-user-uuid"
    },
    "membershipId": "mbr-uuid",
    "organizationId": "org-uuid",
    "status": "ACTIVE"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — 좌석/라이선스 실패 (LicenseBlocked, membership 은 PENDING 유지)</summary>

```json
{ "success": false, "code": "LIC-403-001", "message": "License blocks invite code op on organization=org-uuid: ..." }
```

</details>

<details>
<summary><b>404 Not Found</b> — 코드 미존재 (InviteCodeNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Invite code not found: A1B2C3D4" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 코드 비활성 USED/REVOKED (InviteCodeNotActive)</summary>

```json
{ "success": false, "code": "INV-409-001", "message": "Invite code A1B2C3D4 not ACTIVE (current=USED)" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 이미 멤버 (AlreadyMember)</summary>

```json
{ "success": false, "code": "INV-409-002", "message": "User carenco-user-uuid already a member of organization org-uuid" }
```

</details>

<details>
<summary><b>410 Gone</b> — 코드 만료 (InviteCodeExpired)</summary>

```json
{ "success": false, "code": "INV-410-001", "message": "Invite code expired: A1B2C3D4" }
```

</details>

</details>

### Request 필드 정의 (`RedeemRequest`)

| 필드 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`code`** | string | yes | `@NotBlank` |
| **`memberId`** | string (uuid) | yes | `@ValidUuid` — carenco user-service ID |

### Response 필드 정의 (커스텀 Map)

| 필드 | 타입 | 설명 |
|---|---|---|
| `inviteCode` | `InviteCodeResponse` | USED 로 갱신된 코드 |
| `membershipId` | string (uuid) | 생성된 멤버십 ID |
| `organizationId` | string (uuid) | — |
| `status` | enum `MembershipStatus` | 좌석 점유 성공 시 `ACTIVE` |

---

## 38. `POST` /api/v2/b2b/invite-codes/{inviteCodeId}/revoke

초대 코드 폐기 — OWNER/ADMIN. `ACTIVE → REVOKED` (좌석 영향 없음).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <b2b-jwt>` | yes |

### Security

인증 필요 + **OWNER/ADMIN** — 컨트롤러가 코드의 organizationId 를 조회해 `MembershipRoleResolver` 로 role 해석, 서비스가 OWNER/ADMIN 검증. 미보유 시 `NotOwnerOrAdmin` (403).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `inviteCodeId` 만.

### `1.0.0` — Response

**200 OK** — `InviteCodeResponse` (status=REVOKED)

```json
{
  "success": true,
  "data": {
    "id": "code-uuid",
    "organizationId": "org-uuid",
    "code": "A1B2C3D4",
    "description": "7월 프로모션",
    "expiresAt": "2026-07-11T09:00:00Z",
    "usedAt": null,
    "status": "REVOKED",
    "createdBy": "b2b-user-uuid"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — OWNER/ADMIN 아님 (NotOwnerOrAdmin)</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "User req-x lacks owner/admin role for organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 코드 미존재 (InviteCodeNotFound)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Invite code not found: code-uuid" }
```

</details>

<details>
<summary><b>409 Conflict</b> — ACTIVE 아님 (InviteCodeNotActive)</summary>

```json
{ "success": false, "code": "INV-409-001", "message": "Invite code A1B2C3D4 not ACTIVE (current=REVOKED)" }
```

</details>

</details>

---

## billing

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/billing/plans`](#39-get-apiv2b2bbillingplans) | 1.0.0 | Bearer | B2B plan 카탈로그 (planCode × variants) |
| POST | [`/billing/checkout-init`](#40-post-apiv2b2bbillingcheckout-init) | 1.0.0 | Bearer | 신규/갱신 체크아웃 시작 |
| GET | [`/billing/organizations/{organizationId}/subscription`](#41-get-apiv2b2bbillingorganizationsorganizationidsubscription) | 1.0.0 | Bearer | organization 현재 구독 상세 (scheduledChange 포함) |
| POST | [`/billing/plan-change`](#42-post-apiv2b2bbillingplan-change) | 1.0.0 | Bearer | active 구독의 plan 교체 (업/다운그레이드) |
| POST | [`/billing/organizations/{organizationId}/subscription/pause`](#43-post-apiv2b2bbillingorganizationsorganizationidsubscriptionpause) | 1.0.0 | Bearer | 구독 일시정지 |
| POST | [`/billing/organizations/{organizationId}/subscription/resume`](#44-post-apiv2b2bbillingorganizationsorganizationidsubscriptionresume) | 1.0.0 | Bearer | 일시정지/취소 예약 구독 재개 |
| POST | [`/billing/organizations/{organizationId}/subscription/cancel`](#45-post-apiv2b2bbillingorganizationsorganizationidsubscriptioncancel) | 1.0.0 | Bearer | 구독 취소 (cycle-end) |
| GET | [`/billing/organizations/{organizationId}/transactions`](#46-get-apiv2b2bbillingorganizationsorganizationidtransactions) | 1.0.0 | Bearer | organization 구독의 결제 내역 |
| POST | [`/billing/organizations/{organizationId}/refunds`](#47-post-apiv2b2bbillingorganizationsorganizationidrefunds) | 1.0.0 | Bearer | 거래 환불 생성 (전체 또는 부분) |
| GET | [`/billing/organizations/{organizationId}/refunds`](#48-get-apiv2b2bbillingorganizationsorganizationidrefunds) | 1.0.0 | Bearer | 거래의 환불 목록 |
| GET | [`/billing/organizations/{organizationId}/refunds/{refundId}`](#49-get-apiv2b2bbillingorganizationsorganizationidrefundsrefundid) | 1.0.0 | Bearer | 환불 단건 조회 |
| POST | [`/billing/organizations/{organizationId}/portal-session`](#50-post-apiv2b2bbillingorganizationsorganizationidportal-session) | 1.0.0 | Bearer | Paddle 호스팅 고객 포털 세션 생성 (overview / 송장 / 결제수단 deep link) |

---

## 39. `GET` /api/v2/b2b/billing/plans

B2B plan 카탈로그 (planCode × variants). audience 는 서버가 `B2B` 로 고정. optional `organizationId` 쿼리를 붙이면 그 org 전용 커스텀(enterprise) 가격을 공개 카탈로그에 병합해 조회한다. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**인증된 b2b 유저** 필요. `organizationId` 유무로 분기.

- **미지정** — 공개 카탈로그 (OWNER · organization 불필요, 카탈로그 read). payment-service `GET /api/payment/plans?audience=B2B` 프록시.
- **지정** — 그 org 전용 커스텀(enterprise) variant 를 병합. **요청자가 그 org 의 OWNER** 여야 한다 (payment `GET /api/internal/plans?audience=B2B&organizationId=` 경유). 미등록 org → `OrganizationNotFound` (404), 비OWNER → `NotOwner` (403). org 전용 커스텀 가격은 공개 `/api/payment/plans` 에는 노출되지 않는다 (다른 고객사에 협상가 유출 방지).

이전엔 프론트가 payment-service 를 직접 호출했으나 결제 라인 일원화로 b2b 경유로 전환.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음. audience 는 서버가 `B2B` 로 고정한다.

| 쿼리 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `organizationId` | string (UUID) | optional | 지정 시 그 org 전용 커스텀(enterprise) variant 를 공개 카탈로그에 병합 (OWNER 필수). 미지정 시 공개 카탈로그만. |

- 공개. `GET /api/v2/b2b/billing/plans`
- org 지정. `GET /api/v2/b2b/billing/plans?organizationId=<org-uuid>` → 공개 + 그 org 커스텀 (요청자 OWNER 여야 함)

### `1.0.0` — Response

**200 OK** — `PaymentPlanView[]` (payment-service echo). 활성 plan 없으면 `[]`. 활성 variant 없는 plan 은 `variants: []`. `organizationId` 지정 시 그 org 전용 `enterprise` plan 의 variant 가 병합돼 내려온다 (미지정 시 `enterprise` 는 `variants: []`).

```json
{
  "success": true,
  "data": [
    {
      "planCode": "pro",
      "displayName": "Fisica_B2B_Pro",
      "planSeats": 20,
      "enabled": true,
      "variants": [
        { "priceId": "pri_01kvf67a25pv2f211vypfexc8r", "billingInterval": "MONTHLY", "enabled": true },
        { "priceId": "pri_01kvf6a87dpvh61vzfjhb8xvpe", "billingInterval": "YEARLY", "enabled": true }
      ]
    }
  ]
}
```

<details>
<summary><b>4xx / 502</b> — payment-service propagate</summary>

`organizationId` 지정 시 — 미등록 org → `OrganizationNotFound` (404), 요청자가 OWNER 아님 → `NotOwner` (403).

5xx/network 는 `PaymentServiceUnavailable` → `CMN-502-001` (502).

```json
{ "success": false, "code": "CMN-502-001", "message": "External service error", "error": "Payment service unavailable: ..." }
```

</details>

</details>

### Response 필드 정의 — `PaymentPlanView`

| 필드 | 타입 | 출처 |
|---|---|---|
| `planCode` | string | 비즈니스 plan 식별자 (예: `pro`) |
| `displayName` | string | 표시명 (예: `Fisica_B2B_Pro`) |
| `planSeats` | number | plan 좌석 수 |
| `enabled` | boolean | plan 활성 여부 (활성만 조회되므로 항상 `true`) |
| `variants` | array | 활성 가격 변형 (priceId × billingInterval) |
| `variants[].priceId` | string | Paddle priceId — checkout / plan-change 요청에 사용 |
| `variants[].billingInterval` | string | `MONTHLY` / `YEARLY` |
| `variants[].enabled` | boolean | variant 활성 여부 |

> 가격(금액)은 응답에 없다 — 프론트가 Paddle.js 로 국가별 가격을 직접 수신하고, `priceId` 만 checkout 에 전달한다.

---

## 40. `POST` /api/v2/b2b/billing/checkout-init

신규/갱신 체크아웃 시작. OWNER 만. payment-service `PaymentClient.createCheckoutLink` 위임 (Paddle). 단일 버전 (`1.0.0`). 응답은 프런트의 `Paddle.Checkout.open()` 에 직접 전달.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

권한 검증은 b2b-service 전담. organization ACTIVE (`OrganizationNotActive`) + **OWNER** (`NotOwner`) + license 가 ACTIVE 아님(ACTIVE 면 `AlreadyHasActiveSubscription`, SUSPENDED 면 `LicenseSuspended`) 이어야 함. NONE/EXPIRED/GRACE 만 허용.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "organizationId": "org-uuid",
  "priceId": "pri_01h..."
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`organizationId`** | string (uuid) | **yes** | `@ValidUuid` |
| **`priceId`** | string | **yes** | `@NotBlank` — Paddle priceId (payment-service subscription_plans 에 등록돼야 함) |

### `1.0.0` — Response

**200 OK** — `PaymentCheckoutLinkResponse` (payment-service 응답 그대로 echo)

```json
{
  "success": true,
  "data": {
    "transactionId": "txn_01h...",
    "checkoutUrl": "https://checkout.paddle.com/..."
  }
}
```

| 필드 | 타입 | 출처 | 설명 |
|---|---|---|---|
| `transactionId` | string \| null | payment-service | Paddle 미리 생성 transaction ref |
| `checkoutUrl` | string \| null | payment-service | 대체 리다이렉트 URL |

<details>
<summary><b>403 Forbidden</b> — OWNER 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "User b2buser-uuid is not the owner of organization org-uuid — only OWNER can initiate billing" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — organization 비-ACTIVE 또는 license SUSPENDED</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied", "error": "Organization org-uuid license is SUSPENDED — contact admin to release before new payment" }
```

`OrganizationNotActive` · `LicenseSuspended` 모두 `ACCESS_DENIED` (403).

</details>

<details>
<summary><b>404 Not Found</b> — organization 미존재</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Organization not found: org-uuid" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 이미 ACTIVE 구독 존재</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request", "error": "Organization org-uuid already has ACTIVE subscription — plan change is a separate flow" }
```

</details>

<details>
<summary><b>4xx / 502</b> — payment-service propagate</summary>

payment-service 호출 실패는 `mapPaymentFailure` 로 변환. 4xx 는 `PaymentRejected` — status code 보존 (404 → `CMN-404-001`, 409 → `CMN-409-001`, 그 외 → `CMN-400-001`), detail 에 payment-service 원문. 5xx/network 는 `PaymentServiceUnavailable` → `CMN-502-001` (502).

```json
{ "success": false, "code": "CMN-502-001", "message": "External service error", "error": "Payment service unavailable: ..." }
```

</details>

</details>

---

## 41. `GET` /api/v2/b2b/billing/organizations/{organizationId}/subscription

organization 현재 구독 상세 (scheduledChange 포함). OWNER 만. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** + license.subscriptionId 존재 필요 (없으면 `NoActiveSubscription`, 404).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — `PaymentSubscriptionView` (payment-service echo)

```json
{
  "success": true,
  "data": {
    "id": "sub-uuid",
    "paddleSubscriptionId": "sub_01h...",
    "customerId": "ctm_01h...",
    "organizationId": "org-uuid",
    "priceId": "pri_01h...",
    "quantity": 1,
    "planSeats": 20,
    "planCode": "GYM_PRO",
    "status": "active",
    "nextBilledAt": "2026-08-01T00:00:00Z",
    "currentPeriodStart": "2026-07-01T00:00:00Z",
    "currentPeriodEnd": "2026-08-01T00:00:00Z",
    "canceledAt": null,
    "scheduledChangeAction": null,
    "scheduledChangeEffectiveAt": null,
    "scheduledChangeResumeAt": null,
    "createdAt": "2026-06-01T00:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — OWNER 아님 · organization/구독 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Organization org-uuid has no active subscription — start checkout instead" }
```

</details>

<details>
<summary><b>4xx / 502</b> — payment-service propagate</summary>

`mapPaymentFailure` — §checkout-init 참조.

</details>

</details>

### Response 필드 정의 — `PaymentSubscriptionView`

전 필드 payment-service SubscriptionResponse mirror. `scheduledChangeAction` (`cancel`/`pause`/`resume`, null=예약 없음) 로 예약 변경 가시화. `customerId` 는 portal-session 에서 재사용.

---

## 42. `POST` /api/v2/b2b/billing/plan-change

active 구독의 plan 교체 (업/다운그레이드). OWNER 만, license 가 ACTIVE 여야 함. payment-service 위임. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

organization ACTIVE + **OWNER** + license `state == ACTIVE` (아니면 `NoActiveSubscription`, 404). proration/cycle reset 은 payment-service 가 처리.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "organizationId": "org-uuid",
  "newPriceId": "pri_02h..."
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`organizationId`** | string (uuid) | **yes** | `@ValidUuid` |
| **`newPriceId`** | string | **yes** | `@NotBlank` — 변경할 Paddle priceId |

### `1.0.0` — Response

**200 OK** — `PaymentPlanChangeResponse` (payment-service echo)

```json
{
  "success": true,
  "data": {
    "paddleSubscriptionId": "sub_01h...",
    "newPriceId": "pri_02h...",
    "newPlanCode": "GYM_PRO",
    "newPlanSeats": 20,
    "newBillingInterval": "month",
    "prorationMode": "prorated_immediately",
    "status": "active"
  }
}
```

| 필드 | 타입 | 출처 |
|---|---|---|
| `paddleSubscriptionId` | string | payment-service |
| `newPriceId` | string | payment-service |
| `newPlanCode` | string | payment-service |
| `newPlanSeats` | integer | payment-service |
| `newBillingInterval` | string | payment-service |
| `prorationMode` | string | payment-service |
| `status` | string | payment-service |

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — OWNER 아님 · organization 미존재 · active 구독 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Organization org-uuid has no active subscription — start checkout instead" }
```

</details>

<details>
<summary><b>4xx / 502</b> — payment-service propagate (PaymentRejected / PaymentServiceUnavailable)</summary>

`mapPaymentFailure` 매핑 — §checkout-init 참조.

</details>

</details>

---

## 43. `POST` /api/v2/b2b/billing/organizations/{organizationId}/subscription/pause

구독 일시정지. OWNER 만. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** + 구독 존재.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — 변경 후 `PaymentSubscriptionView` (`scheduledChangeAction: "pause"`). shape 은 §subscription/cancel 과 동일.

<details>
<summary><b>403 Forbidden · 404 Not Found · 4xx/502</b> — OWNER 아님 · 구독 없음 · payment-service propagate</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## 44. `POST` /api/v2/b2b/billing/organizations/{organizationId}/subscription/resume

일시정지/취소 예약 구독 재개. OWNER 만. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** + 구독 존재.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — 변경 후 `PaymentSubscriptionView` (`scheduledChangeAction: "resume"` 또는 null 로 정상화). shape 은 §subscription/cancel 과 동일.

<details>
<summary><b>403 Forbidden · 404 Not Found · 4xx/502</b> — OWNER 아님 · 구독 없음 · payment-service propagate</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## 45. `POST` /api/v2/b2b/billing/organizations/{organizationId}/subscription/cancel

구독 취소 (cycle-end). OWNER 만. payment-service 위임. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** + 구독 존재 (`NoActiveSubscription`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — 변경 후 `PaymentSubscriptionView` (§GET subscription 과 동일 shape, `scheduledChangeAction: "cancel"` 등 반영)

```json
{
  "success": true,
  "data": {
    "id": "sub-uuid",
    "paddleSubscriptionId": "sub_01h...",
    "customerId": "ctm_01h...",
    "organizationId": "org-uuid",
    "priceId": "pri_01h...",
    "quantity": 1,
    "planSeats": 20,
    "planCode": "GYM_PRO",
    "status": "active",
    "nextBilledAt": "2026-08-01T00:00:00Z",
    "currentPeriodStart": "2026-07-01T00:00:00Z",
    "currentPeriodEnd": "2026-08-01T00:00:00Z",
    "canceledAt": null,
    "scheduledChangeAction": "cancel",
    "scheduledChangeEffectiveAt": "2026-08-01T00:00:00Z",
    "scheduledChangeResumeAt": null,
    "createdAt": "2026-06-01T00:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found · 4xx/502</b> — OWNER 아님 · 구독 없음 · payment-service propagate</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## 46. `GET` /api/v2/b2b/billing/organizations/{organizationId}/transactions

organization 구독의 결제 내역. OWNER 만. license.subscriptionId 로 payment-service 조회. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** 만 (`NotOwner`). organization 미존재 시 `OrganizationNotFound`. license 에 subscriptionId 없으면(구독 이력 없음) 빈 배열 (에러 아님).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — `PaymentTransactionView[]` (payment-service echo). 구독 없으면 `[]`

```json
{
  "success": true,
  "data": [
    {
      "id": "txn-uuid",
      "paddleTransactionId": "txn_01h...",
      "customerId": "ctm_01h...",
      "subscriptionId": "sub_01h...",
      "status": "completed",
      "currencyCode": "USD",
      "subtotal": 10000,
      "discountTotal": 0,
      "taxTotal": 1000,
      "totalAmount": 11000,
      "billingPeriodStartsAt": "2026-06-01T00:00:00Z",
      "billingPeriodEndsAt": "2026-07-01T00:00:00Z",
      "billingCountryCode": "KR",
      "billingPostalCode": "06000",
      "billingRegion": "Seoul",
      "billingCity": "Seoul",
      "billingAddressLine": "...",
      "items": [
        {
          "id": "it-uuid", "paddleItemId": "txnitm_...", "priceId": "pri_01h...",
          "productId": "pro_...", "productName": "Gym Pro", "productDescription": "...",
          "quantity": 1, "unitPrice": 10000, "subtotal": 10000,
          "discountAmount": 0, "taxAmount": 1000, "total": 11000, "taxRate": "0.1"
        }
      ],
      "payments": [
        {
          "id": "pay-uuid", "amount": 11000, "status": "captured", "paymentType": "card",
          "cardType": "visa", "cardLast4": "4242", "cardExpiryMonth": 12, "cardExpiryYear": 2028,
          "cardholderName": "HONG GILDONG", "capturedAt": "2026-06-01T00:00:05Z"
        }
      ],
      "createdAt": "2026-06-01T00:00:00Z"
    }
  ]
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — OWNER 아님 · organization 미존재</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

</details>

### Response 필드 정의 — `PaymentTransactionView`

전 필드 payment-service Transaction 응답 mirror (b2b 가공 없음). 금액은 모두 정수 minor unit (센트). nested `items[]` (`Item`) · `payments[]` (`Payment`) 은 위 예시 키 참조.

---

## 47. `POST` /api/v2/b2b/billing/organizations/{organizationId}/refunds

거래 환불 생성 (전체 또는 부분). OWNER 이고 대상 거래가 organization 구독 소속일 때만. payment-service 위임. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

**OWNER** + 구독 존재 + `transactionId` 가 구독 거래 목록에 있어야 함 (아니면 `RefundTransactionNotOwned`, 403).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "transactionId": "txn_01h...",
  "reason": "고객 요청 환불",
  "items": [
    { "paddleItemId": "txnitm_01h...", "amount": "5000" }
  ]
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`transactionId`** | string | **yes** | `@NotBlank` — 환불 대상 거래 |
| **`reason`** | string | **yes** | `@NotBlank` |
| `items` | array | no | null/빈 리스트 → 전체 환불. 값 있으면 부분 환불 |
| `items[].paddleItemId` | string | items 있을 때 yes | `@NotBlank` |
| `items[].amount` | string \| null | no | null = 항목 전체, 값 = 부분 금액 (센트 문자열) |

### `1.0.0` — Response

**200 OK** — `PaymentRefundView` (payment-service echo)

```json
{
  "success": true,
  "data": {
    "id": "rf-uuid",
    "paddleAdjustmentId": "adj_01h...",
    "transactionId": "txn_01h...",
    "reason": "고객 요청 환불",
    "status": "pending_approval",
    "currencyCode": "USD",
    "amount": 5000,
    "createdAt": "2026-07-09T00:00:00Z"
  }
}
```

| 필드 | 타입 | 출처 |
|---|---|---|
| `id` | string | payment-service |
| `paddleAdjustmentId` | string | payment-service |
| `transactionId` | string | payment-service |
| `reason` | string | payment-service |
| `status` | string | payment-service |
| `currencyCode` | string | payment-service |
| `amount` | integer (long) | payment-service (센트) |
| `createdAt` | string (date-time) | payment-service |

<details>
<summary><b>403 Forbidden</b> — OWNER 아님 · 거래가 organization 소속 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "Transaction txn_01h... does not belong to organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — organization/구독 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

<details>
<summary><b>4xx / 502</b> — payment-service propagate (PaymentRejected / PaymentServiceUnavailable)</summary>

`mapPaymentFailure` — §checkout-init 참조.

</details>

</details>

---

## 48. `GET` /api/v2/b2b/billing/organizations/{organizationId}/refunds

거래의 환불 목록. `transactionId` 쿼리 필수. OWNER + 거래 소유 검증. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** + 구독 존재 + `transactionId` 가 구독 거래 소속 (`RefundTransactionNotOwned`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음. 쿼리 파라미터 `transactionId` (필수).

```
GET /api/v2/b2b/billing/organizations/{organizationId}/refunds?transactionId=txn_01h...
```

### `1.0.0` — Response

**200 OK** — `PaymentRefundView[]` (payment-service echo)

```json
{
  "success": true,
  "data": [
    {
      "id": "rf-uuid",
      "paddleAdjustmentId": "adj_01h...",
      "transactionId": "txn_01h...",
      "reason": "고객 요청 환불",
      "status": "approved",
      "currencyCode": "USD",
      "amount": 5000,
      "createdAt": "2026-07-09T00:00:00Z"
    }
  ]
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found · 4xx/502</b> — OWNER/거래소유 아님 · 구독 없음 · payment-service propagate</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

</details>

---

## 49. `GET` /api/v2/b2b/billing/organizations/{organizationId}/refunds/{refundId}

환불 단건 조회. 조회 후 그 환불의 거래가 organization 소유인지 검증. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** + 구독 존재. payment-service 에서 환불 조회 후 `refund.transactionId` 가 구독 거래 소속인지 검증 (`RefundTransactionNotOwned`).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` · `refundId`.

### `1.0.0` — Response

**200 OK** — `PaymentRefundView` (§POST refunds 응답과 동일 shape)

```json
{
  "success": true,
  "data": {
    "id": "rf-uuid",
    "paddleAdjustmentId": "adj_01h...",
    "transactionId": "txn_01h...",
    "reason": "고객 요청 환불",
    "status": "approved",
    "currencyCode": "USD",
    "amount": 5000,
    "createdAt": "2026-07-09T00:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found · 4xx/502</b> — OWNER/거래소유 아님 · 구독 없음 · payment-service propagate</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## 50. `POST` /api/v2/b2b/billing/organizations/{organizationId}/portal-session

Paddle 호스팅 고객 포털 세션 생성 (overview / 송장 / 결제수단 deep link). OWNER 만. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

**OWNER** + 구독 존재. 구독의 `customerId` 를 서버에서 도출해 호출 — 프런트의 customerId 위조 차단.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — `PaymentCustomerPortalView` (payment-service echo)

```json
{
  "success": true,
  "data": {
    "overviewUrl": "https://customer-portal.paddle.com/...",
    "subscriptions": [
      {
        "subscriptionId": "sub_01h...",
        "cancelUrl": "https://customer-portal.paddle.com/.../cancel",
        "updatePaymentMethodUrl": "https://customer-portal.paddle.com/.../update"
      }
    ]
  }
}
```

| 필드 | 타입 | 출처 |
|---|---|---|
| `overviewUrl` | string | payment-service |
| `subscriptions` | array | payment-service |
| `subscriptions[].subscriptionId` | string | payment-service |
| `subscriptions[].cancelUrl` | string | payment-service |
| `subscriptions[].updatePaymentMethodUrl` | string | payment-service |

<details>
<summary><b>403 Forbidden · 404 Not Found · 4xx/502</b> — OWNER 아님 · 구독 없음 · payment-service propagate</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## feedbacks

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/feedbacks/memberships/{membershipId}`](#51-post-apiv2b2bfeedbacksmembershipsmembershipid) | 1.0.0 | Bearer | 멤버(membership)에 피드백 작성 |
| GET | [`/feedbacks/memberships/{membershipId}`](#52-get-apiv2b2bfeedbacksmembershipsmembershipid) | 1.0.0 | Bearer | membership 의 피드백 목록 (soft-delete 제외) |
| GET | [`/feedbacks/measurements/{recordId}`](#53-get-apiv2b2bfeedbacksmeasurementsrecordid) | 1.0.0 | Bearer | 특정 측정 record 에 연결된 피드백 목록 (soft-delete 제외) |
| PATCH | [`/feedbacks/{feedbackId}`](#54-patch-apiv2b2bfeedbacksfeedbackid) | 1.0.0 | Bearer | 피드백 본문 수정 |
| DELETE | [`/feedbacks/{feedbackId}`](#55-delete-apiv2b2bfeedbacksfeedbackid) | 1.0.0 | Bearer | 피드백 soft-delete (`deletedAt` 세팅) |

---

## 51. `POST` /api/v2/b2b/feedbacks/memberships/{membershipId}

멤버(membership)에 피드백 작성. 스태프(OWNER/ADMIN/TRAINER)만. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

대상 membership 의 organization 에서 작성자가 **OWNER / ADMIN / TRAINER** 여야 함 (`NotStaff`). membership 미존재 시 `MembershipNotFound`.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "body": "자세 교정 필요. 어깨 비대칭 관찰됨.",
  "visibility": "MEMBER",
  "measurementRecordId": "rec-uuid"
}
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`body`** | string | **yes** | `@NotBlank @Size(max=65535)` — 저장 시 trim |
| `visibility` | enum | no | `MEMBER` \| `INTERNAL`. 미전송 시 `MEMBER` |
| `measurementRecordId` | string | no | `@Size(max=36)` — 특정 측정과 연결 |

### `1.0.0` — Response

**201 Created** — `FeedbackResponse`

```json
{
  "success": true,
  "data": {
    "id": "fb-uuid",
    "organizationId": "org-uuid",
    "memberMembershipId": "membership-uuid",
    "authorUserId": "b2buser-uuid",
    "measurementRecordId": "rec-uuid",
    "body": "자세 교정 필요. 어깨 비대칭 관찰됨.",
    "visibility": "MEMBER",
    "createdAt": "2026-07-09T00:00:00Z",
    "editedAt": null
  }
}
```

<details>
<summary><b>400 Bad Request</b> — body 공백</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter", "error": "Invalid body: must not be blank" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — 스태프 role 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

<details>
<summary><b>404 Not Found</b> — 대상 membership 미존재</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Target membership not found: membership-uuid" }
```

</details>

</details>

### Response 필드 정의 — `FeedbackResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | feedback id |
| `organizationId` | string | 소속 organization |
| `memberMembershipId` | string | 대상 membership id |
| `authorUserId` | string | 작성자 b2b_users.id |
| `measurementRecordId` | string \| null | 연결된 측정 record |
| `body` | string | 본문 |
| `visibility` | enum | `MEMBER` \| `INTERNAL` |
| `createdAt` | string (date-time) | 생성 시각 |
| `editedAt` | string \| null | 수정 시각 (미수정 시 null) |

---

## 52. `GET` /api/v2/b2b/feedbacks/memberships/{membershipId}

membership 의 피드백 목록 (soft-delete 제외). 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

이 핸들러는 `@AuthenticationPrincipal` 을 받지 않음 — 게이트웨이 JWT 인증 외 per-request 권한 검증 없음(멤버십/스태프 role 체크 안 함). 인증된 b2b 사용자면 누구나 조회 가능.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 필수 | 기본값 | 규칙 |
|---|---|---|---|---|
| `page` | integer | no | `0` | 음수는 0 으로 |
| `size` | integer | no | `20` | ≤0 또는 >100 이면 20 |

### `1.0.0` — Response

**200 OK** — `{ items: FeedbackResponse[], page, totalElements, hasNext }`

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "fb-uuid",
        "organizationId": "org-uuid",
        "memberMembershipId": "membership-uuid",
        "authorUserId": "b2buser-uuid",
        "measurementRecordId": "rec-uuid",
        "body": "자세 교정 필요.",
        "visibility": "MEMBER",
        "createdAt": "2026-07-09T00:00:00Z",
        "editedAt": null
      }
    ],
    "page": 0,
    "totalElements": 5,
    "hasNext": false
  }
}
```

item 필드는 §POST feedbacks 응답 표와 동일.

</details>

---

## 53. `GET` /api/v2/b2b/feedbacks/measurements/{recordId}

특정 측정 record 에 연결된 피드백 목록 (soft-delete 제외). 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

`@AuthenticationPrincipal` 미수신 — §GET feedbacks/memberships 와 동일하게 per-request 권한 검증 없음.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음. 쿼리 파라미터 `page` (기본 0) · `size` (기본 20, ≤0/>100 이면 20).

### `1.0.0` — Response

**200 OK** — `{ items: FeedbackResponse[], page, totalElements, hasNext }` (§GET feedbacks/memberships 와 동일 shape)

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "fb-uuid",
        "organizationId": "org-uuid",
        "memberMembershipId": "membership-uuid",
        "authorUserId": "b2buser-uuid",
        "measurementRecordId": "rec-uuid",
        "body": "자세 교정 필요.",
        "visibility": "MEMBER",
        "createdAt": "2026-07-09T00:00:00Z",
        "editedAt": null
      }
    ],
    "page": 0,
    "totalElements": 2,
    "hasNext": false
  }
}
```

</details>

---

## 54. `PATCH` /api/v2/b2b/feedbacks/{feedbackId}

피드백 본문 수정. 작성자 본인만. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

### Security

작성자(`authorUserId`)만 수정 가능 (`NotAuthor`). 미존재/삭제됨 시 `FeedbackNotFound`.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "body": "수정된 피드백 본문." }
```

| 필드 | 타입 | 필수 | 규칙/설명 |
|---|---|---|---|
| **`body`** | string | **yes** | `@NotBlank @Size(max=65535)` — 저장 시 trim |

### `1.0.0` — Response

**200 OK** — `FeedbackResponse` (editedAt 갱신)

```json
{
  "success": true,
  "data": {
    "id": "fb-uuid",
    "organizationId": "org-uuid",
    "memberMembershipId": "membership-uuid",
    "authorUserId": "b2buser-uuid",
    "measurementRecordId": "rec-uuid",
    "body": "수정된 피드백 본문.",
    "visibility": "MEMBER",
    "createdAt": "2026-07-09T00:00:00Z",
    "editedAt": "2026-07-09T01:00:00Z"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — body 공백</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter", "error": "Invalid body: must not be blank" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — 작성자 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "User b2buser-uuid is not the author of feedback fb-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — feedback 미존재/삭제됨</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Feedback not found: fb-uuid" }
```

</details>

</details>

---

## 55. `DELETE` /api/v2/b2b/feedbacks/{feedbackId}

피드백 soft-delete (`deletedAt` 세팅). 작성자 또는 OWNER/ADMIN. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

작성자 본인 **또는** 해당 organization 의 OWNER/ADMIN 만 (아니면 `NotAuthor`). 미존재/이미 삭제 시 `FeedbackNotFound`.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `feedbackId` 만.

### `1.0.0` — Response

**200 OK** — soft-delete 직전 상태의 `FeedbackResponse`

```json
{
  "success": true,
  "data": {
    "id": "fb-uuid",
    "organizationId": "org-uuid",
    "memberMembershipId": "membership-uuid",
    "authorUserId": "b2buser-uuid",
    "measurementRecordId": "rec-uuid",
    "body": "자세 교정 필요.",
    "visibility": "MEMBER",
    "createdAt": "2026-07-09T00:00:00Z",
    "editedAt": null
  }
}
```

<details>
<summary><b>403 Forbidden</b> — 작성자·OWNER·ADMIN 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission" }
```

</details>

<details>
<summary><b>404 Not Found</b> — feedback 미존재/삭제됨</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## license-summary

| Method | Path (`/api/v2/b2b` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/license-summary`](#56-get-apiv2b2blicense-summary) | 1.0.0 | Bearer | 로그인 사용자가 소유(OWNER) 또는 ACTIVE B2B 멤버인 모든 organization … |
| GET | [`/license-summary/{organizationId}`](#57-get-apiv2b2blicense-summaryorganizationid) | 1.0.0 | Bearer | 특정 organization 의 license 단건 |

---

## 56. `GET` /api/v2/b2b/license-summary

로그인 사용자가 소유(OWNER) 또는 ACTIVE B2B 멤버인 모든 organization 의 license + seat 종합. 단일 버전 (`1.0.0`). license 는 payment-service 에서 조회 (`LicenseClient.batchGetLicense`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

인증 principal 의 소속 organization 만 자동 스코핑 — 별도 실패 케이스 없음 (소속 없으면 빈 목록). `@PreAuthorize` 없음.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 인증 컨텍스트만.

### `1.0.0` — Response

**200 OK** — `{ items: LicenseSummaryResponse[] }` (소속 organization 없으면 빈 배열)

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "organizationId": "org-uuid",
        "organizationName": "필라테스 강남점",
        "state": "ACTIVE",
        "planCode": "PILATES_BASIC",
        "planSeats": 10,
        "seatsUsed": 4,
        "seatsRemaining": 6,
        "startedAt": "2026-06-01T00:00:00Z",
        "expiresAt": "2026-07-01T00:00:00Z",
        "graceRemainingDays": 0,
        "graceRemainingSeconds": 0,
        "subscriptionId": "sub-uuid",
        "cancelable": true,
        "upgradable": true
      }
    ]
  }
}
```

필드 정의는 아래 §GET license-summary/{organizationId} 참조.

</details>

---

## 57. `GET` /api/v2/b2b/license-summary/{organizationId}

특정 organization 의 license 단건. 멤버만 조회. 단일 버전 (`1.0.0`).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

### Security

`MembershipRoleResolver.resolve` role 이 비면 `NotMember` (403). organization row 없으면 `OrganizationNotFound` (404).

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `organizationId` 만.

### `1.0.0` — Response

**200 OK** — `LicenseSummaryResponse`

```json
{
  "success": true,
  "data": {
    "organizationId": "org-uuid",
    "organizationName": "필라테스 강남점",
    "state": "GRACE",
    "planCode": "GYM_PRO",
    "planSeats": 20,
    "seatsUsed": 12,
    "seatsRemaining": 8,
    "startedAt": "2026-05-01T00:00:00Z",
    "expiresAt": "2026-06-01T00:00:00Z",
    "graceRemainingDays": 5,
    "graceRemainingSeconds": 432000,
    "subscriptionId": "sub-uuid",
    "cancelable": false,
    "upgradable": true
  }
}
```

<details>
<summary><b>403 Forbidden</b> — organization 멤버 아님</summary>

```json
{ "success": false, "code": "AUTH-403-002", "message": "Insufficient permission", "error": "not a member of organization org-uuid" }
```

</details>

<details>
<summary><b>404 Not Found</b> — organization 미존재</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Organization not found: org-uuid" }
```

</details>

</details>

### Response 필드 정의 — `LicenseSummaryResponse`

`state`/`planSeats`/`startedAt`/`expiresAt`/`grace*`/`subscriptionId` 는 payment-service license snapshot 에서 유래. `seatsUsed`/`seatsRemaining` 은 b2b organization 의 `seatsUsed` 와 조합.

| 필드 | 타입 | 규칙/설명 |
|---|---|---|
| `organizationId` | string | b2b organization id |
| `organizationName` | string | organization 이름 |
| `state` | enum | `ACTIVE` \| `GRACE` \| `EXPIRED` \| `SUSPENDED` \| `NONE` |
| `planCode` | string \| null | 플랜 식별자 (예: `PILATES_BASIC`). NONE 이면 null |
| `planSeats` | integer \| null | 플랜 좌석 수. `state == NONE` 이면 null |
| `seatsUsed` | integer | 현재 사용 좌석 (b2b organization.seatsUsed) |
| `seatsRemaining` | integer \| null | `max(0, planSeats - seatsUsed)`. planSeats null 이면 null |
| `startedAt` | string \| null | 구독 시작일. NONE 이면 null |
| `expiresAt` | string \| null | 현재 주기 종료일. NONE 이면 null |
| `graceRemainingDays` | integer | GRACE 남은 일수 (0~7). 그 외 0 |
| `graceRemainingSeconds` | integer (long) | GRACE 남은 초. 그 외 0 |
| `subscriptionId` | string \| null | payment-service subscription PK |
| `cancelable` | boolean | `state == ACTIVE` |
| `upgradable` | boolean | `state == ACTIVE \|\| GRACE` |

---

## 에러 코드

application 레이어 sealed Error → `ErrorCode` → HTTP/code 매핑. 아래는 도메인 전반의 대표 코드. 인증(login/token/logout)·Spring Security 계열은 throw → `GlobalExceptionHandler` 가 매핑한다.

| 코드 | HTTP | 상황 |
|---|---|---|
| `CMN-400-001` | 400 | 일반 validation 위반 (`@Valid` body, enum·형식) |
| `CMN-400-002` | 400 | `@ValidUuid`/`@ValidCountryCode` 등 형식 위반 · `FieldCannotBeCleared` (PATCH 로 필수 필드 clear 시도) |
| `USR-404-001` | 404 | b2b_user 미존재 (`UserError.NotFound`) |
| `CMN-404-001` | 404 | organization / membership / device / feedback / invite-code 미존재 |
| `CMN-409-001` | 409 | 상태 전이 거절 (`InvalidStateTransition`) |
| `INV-409-001` | 409 | invite code 비활성 (`InviteCodeNotActive`) |
| `INV-409-002` | 409 | 이미 조직 멤버 (`AlreadyMember`) |
| `INV-410-001` | 410 | invite code 만료 (`InviteCodeExpired`) |
| `LIC-409-001` | 409 | seat 소진 실패 (`SeatConsumeFailed`) |
| `LIC-403-001` | 403 | license 차단 (`LicenseBlocked`) |
| `AUTH-403-001` | 403 | 조직 비공개·invite 비활성 등 접근 거절 (`OrganizationNotSearchable` / `InviteCodeDisabled`) |
| `AUTH-403-002` | 403 | 권한 없음 — owner/admin/self 아님 (`NotOwner` / `NotOwnerOrAdmin` / `RoleRequired` / `NotSelf`) |
| `AUTH-401-001` / `AUTH-401-003` | 401 | 토큰 무효·자격 증명 실패 (Spring Security throw) |
| `CMN-502-001` | 502 | OAuth2 verifier 미가용 (`VerifierUnavailable`) |
| device-service `ErrorCode` | 4xx/5xx | device 등록/조회 gRPC 위임 시 propagate |
| payment-service `ProblemDetail` | 4xx/5xx | billing 프록시 시 payment-service 에러 propagate |
