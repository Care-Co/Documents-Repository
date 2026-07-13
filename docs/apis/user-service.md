# user-service

> 엔드포인트별 **Header · Request · Response** 정의 — 버전별 전량 전개, 소스는 실제 컨트롤러/DTO. 표기 — **굵은 필드 = 필수**, 규칙은 키워드만, 긴 설명은 각주(¹²³).

| 항목 | 값 |
|---|---|
| Source | `/Users/jonghak/GitHub/Care&Co/user-service` |
| Updated | 2026-07-13 |
| Server | `https://api.example.com` |
| Base path | 일반 `/api/v2/...` · 내부 `/api/public/...` · 관리자 `/api/admin/...` |

---

## 공통 규칙

- **버전** — versioned 엔드포인트는 `api-version: x.y.z` 헤더 필수, **정확 일치만** (미등록 버전 → `400 Invalid API version`). admin·deprecated 계열은 unversioned — 헤더 무시.
- **인증** — 보호 엔드포인트는 `Authorization: Bearer <jwt>` 필수. scheme 상세는 아래 접기 참조.
- **바디** — 요청 바디가 있으면 `Content-Type: application/json`.
- **응답 틀** — 모든 응답은 `CncResponse` envelope (아래 접기 참조) — 개별 섹션 샘플에서 반복하지 않음. 에러 코드 전체는 문서 하단 [에러 코드](#에러-코드) 참조.

**복수 버전 엔드포인트** — 대부분 `1.0.0` 단일, 아래 3개만 복수. 버전 간 차이는 각 섹션 상단의 버전 표 참조.

| 그룹 | 엔드포인트 | 최신 |
|---|---|---|
| auth (weightUnit) | [`POST /auth/register`](#1-post-apiv2authregister) | `1.1.1` |
| users (로케일 표기·weightUnit) | [`GET`](#11-get-apiv2usersuserid)/[`PATCH /users/{userId}`](#12-patch-apiv2usersuserid) | `1.1.1` |

<details>
<summary><b>Security scheme</b> — permitAll · userBearer · adminBearer</summary>

| Scheme | Description |
|---|---|
| `permitAll` | 인증 불필요 (`@PreAuthorize("permitAll()")`) |
| `userBearer` | `ROLE_USER` (terms 는 `ROLE_GUEST` 도 허용) JWT. 리소스 오너 검증은 `@tokenUtils.isResourceOwner(#userId)` |
| `adminBearer` | `ROLE_ADMIN` JWT |

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

| 필드 | 타입 | 필수 |
|---|---|---|
| `success` | boolean | yes |
| `data` | object | no |
| `token` | object | no |
| `error` | object | no |
| `timestamp` | string (date-time, UTC) | yes |

**에러** (`ErrorResponse`) — 코드 전체는 문서 하단 [에러 코드](#에러-코드) 참조

```json
{
  "success": false,
  "code": "CMN-400-001",
  "message": "Invalid request parameter. Field: email, Value: ...",
  "timestamp": "2026-04-27T08:00:00Z"
}
```

</details>

---

## auth

| Method | Path (`/api/v2` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/auth/register`](#1-post-apiv2authregister) | **1.0.0, 1.1.1** | 공개 | 계정 생성 + 즉시 토큰 발급 |
| POST | [`/auth/resend-email`](#2-post-apiv2authresend-email) | 1.0.0 | 공개 | 이메일 인증 메일 재발송 |
| POST | [`/auth/verify-email`](#3-post-apiv2authverify-email) | 1.0.0 | 공개 | 인증 코드(6자리)로 이메일 검증 |
| POST | [`/auth/login`](#4-post-apiv2authlogin) | 1.0.0 | 공개 | 이메일·비밀번호 로그인 → 토큰 발급 |
| POST | [`/auth/google`](#5-post-apiv2authgoogle) | 1.0.0 | 공개 | 모바일 Google OAuth → CareNCo 토큰 발급 |
| POST | [`/auth/apple`](#6-post-apiv2authapple) | 1.0.0 | 공개 | 모바일 Apple OAuth → CareNCo 토큰 발급 |
| POST | [`/auth/token`](#7-post-apiv2authtoken) | 1.0.0 | 공개 | access + refresh 토큰 쌍 재발급 |
| POST | [`/auth/logout`](#8-post-apiv2authlogout) | 1.0.0 | 공개 | refresh token 무효화 + 로그아웃 시각 기록 |
| POST | [`/auth/send-password-reset`](#9-post-apiv2authsend-password-reset) | 1.0.0 | 공개 | 비밀번호 재설정 메일 발송 |
| POST | [`/auth/send-account-recovery`](#10-post-apiv2authsend-account-recovery) | 1.0.0 (미구현) | 공개 | 계정 복구 메일 발송 — 현재 1.0.0 미구현 |

---

## 1. `POST` /api/v2/auth/register

계정 생성 + 즉시 토큰 발급. 버전 간 응답 shape 동일.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 기본 필드 |
| `1.1.1` **(최신)** | = `1.0.0` 필드 + `weightUnit` (선택, `KG`\|`LB`, 미제공 시 `KG`) |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.1.1` | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "username": "u@example.com",
  "password": "P@ssw0rd!",
  "firstName": "김철수2",
  "lastName": "박",
  "nickname": "철수",
  "phoneNumber": "+821012345678",
  "gender": "MALE",
  "birthdate": "1995-04-27",
  "height": 175.5,
  "countryCode": "KR",
  "languageCode": "ko-KR",
  "timeZone": "KOREA",
  "zoneId": "Asia/Seoul"
}
```

### `1.0.0` — Response

**201 Created**

```json
{
  "success": true,
  "token": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "accessTokenExpiresIn": 900
  },
  "timestamp": "2026-07-08T09:00:00Z"
}
```

<details>
<summary><b>400 Bad Request</b> — 형식 위반 · 이름 정규화 후 빈 값 ¹</summary>

```json
{ "success": false, "code": "CMN-400-001", "error": "firstName is invalid after normalization." }
```

</details>

<details>
<summary><b>409 Conflict</b> — 이메일·전화번호 중복</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request. Field: username" }
```

</details>

</details>

<details>
<summary><b><code>1.1.1</code> — Request · Response</b></summary>

### `1.1.1` — Request (= 1.0.0 필드 + `weightUnit`)

```json
{
  "username": "u@example.com",
  "password": "P@ssw0rd!",
  "firstName": "김철수2",
  "lastName": "박",
  "nickname": "철수",
  "phoneNumber": "+821012345678",
  "gender": "MALE",
  "birthdate": "1995-04-27",
  "height": 175.5,
  "weightUnit": "LB",
  "countryCode": "KR",
  "languageCode": "ko-KR",
  "timeZone": "KOREA",
  "zoneId": "Asia/Seoul"
}
```

### `1.1.1` — Response

**201 Created**

```json
{
  "success": true,
  "token": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "accessTokenExpiresIn": 900
  },
  "timestamp": "2026-07-08T09:00:00Z"
}
```

<details>
<summary><b>400 Bad Request</b> — 형식 위반 · 이름 정규화 후 빈 값 ¹ · 잘못된 weightUnit</summary>

```json
{ "success": false, "code": "CMN-400-001", "error": "Invalid request parameter. Field: weightUnit, Value: STONE" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 이메일·전화번호 중복</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request. Field: username" }
```

</details>

</details>

### Response 필드 정의 — `CncTokenDto`

응답의 `token`. 필드 `@JsonInclude(NON_NULL)`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string (uuid) | — |
| `accessToken` | object | JWT access token wrapper |
| `refreshToken` | object | JWT refresh token wrapper |
| `expiresIn` | integer (int64) | access 만료 (초) |
| `refreshExpiresIn` | integer (int64) | refresh 만료 (초) |

> 예시 JSON 의 `accessTokenExpiresIn`(초) 는 클라이언트 관점 표기이며, 실제 DTO 필드는 `expiresIn`/`refreshExpiresIn`.

### Request 필드 정의 (공통)

**계정 (필수)**

| 필드 | 값 | 규칙 |
|---|---|---|
| **`username`** | 이메일 | `@ValidEmail` · 중복 409 |
| **`password`** | 8자+ | `@ValidPassword` · 영문·숫자·특수문자 |

**프로필 (선택)**

| 필드 | 값 | 규칙 |
|---|---|---|
| `firstName` `lastName` | 문자열 | 정규화 수용 ¹ (validation 애너테이션 없음) |
| `nickname` | 1~30자 | `@ValidNickname` · 미제공 시 firstName 사용 |
| `phoneNumber` | E.164 (`+8210...`) | `@ValidPhoneNumber` · 중복 409 |
| `gender` | `MALE` `FEMALE` `OTHER` `UNDISCLOSED` | 미제공 → `UNKNOWN`/`UNDISCLOSED` |
| `birthdate` | `YYYY-MM-DD` | `@ValidBirthDate` · 측정 전 필수 ² |
| `height` | number (cm) | `@ValidHeight` |

**로케일·단위 (선택 — 미제공 시 기본값)**

| 필드 | 값 | 기본값 | 버전 |
|---|---|---|---|
| `weightUnit` | `KG` `LB` | `KG` | **1.1.1만** (1.0.0에 보내면 무시) |
| `countryCode` | alpha-2/3/별칭 ³ | `UNKNOWN` | 전 버전 |
| `languageCode` | BCP-47 / enum 이름 ³ | `UNKNOWN` | 전 버전 |
| `timeZone` / `zoneId` | enum / `Asia/Seoul` (JSON key `zoneId` → 필드 `zoneIdValue`) | | 전 버전 |

**필드별 검증 애너테이션 (`UserRegisterRequest`)**

| Field (JSON key) | Type | Required | Validation |
|---|---|---|---|
| `username` | string | no | `@ValidEmail(allowEmpty=false, allowNull=false)` |
| `password` | string | yes | `@ValidPassword(allowEmpty=false, allowNull=false)` |
| `firstName` | string | no | (없음 — 정규화 ¹) |
| `lastName` | string | no | (없음 — 정규화 ¹) |
| `nickname` | string | no | `@ValidNickname(allowNull=true, min=1, max=30)` |
| `phoneNumber` | string | no | `@ValidPhoneNumber(allowNull=true)` |
| `gender` | string | no | `@ValidEnum(Gender)` — `MALE` `FEMALE` `OTHER` `UNDISCLOSED` |
| `birthdate` | string (date) | no | `@ValidBirthDate(allowNull=true)` |
| `height` | number (double) | no | `@ValidHeight(allowNull=true)` |
| `weightUnit` | string | no | **1.1.1만** · `@ValidEnum(WeightUnit)` — `KG` `LB` |
| `countryCode` | string | no | `@ValidEnum(CountryCode)` |
| `languageCode` | string | no | `@ValidLanguageTag(allowEmpty=false, allowNull=true)` |
| `timeZone` | string | no | `@ValidEnum(TimeZone)` |
| `zoneId` | string | no | Java 필드 `zoneIdValue` (`@JsonProperty("zoneId")`) — IANA zone id |

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

> **입력** (common-libs 0.0.83~) — alpha-2 외 **ISO 3166-1 alpha-3**(`KOR`·`USA`·`CHN`·`TWN`)·레거시 별칭도 허용. 모두 enum 으로 정규화.
>
> **저장·응답** (0.0.84~) —
> - **DB 저장**: ISO 3166-1 **alpha-3**(`KOR`). 0.0.83 까지 alpha-2, V33 마이그레이션으로 전환.
> - **응답**: `api-version` 별 — **`1.0.0`** → alpha-2(`"KR"`), **`1.1.0`** → alpha-3(`"KOR"`). gRPC·admin 은 alpha-2 유지.
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
> b2b-service 의 `country`(`@ValidCountryCode`)는 **strict alpha-2** 입력만 받되 DB 저장은 alpha-3(응답 `1.0.2` 노출).

**`languageCode` 허용값** — `LanguageCode` enum 이름. BCP 47 language tag 와 매핑.

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

> 간체/번체는 문자(script) 구분이라 enum 이름이 국가코드 아닌 BCP 47 script subtag 기반.
>
> **입력** (0.0.82~) — enum 이름(`ZH_HANS`)·BCP 47 tag(`zh-Hans`·`ko-KR`) 모두 허용. `en` 같은 language-only 는 비수용(full tag 만).
> **저장·응답** (0.0.84~) — DB 는 **BCP 47**(V33). 응답은 `1.0.0` → enum 이름, `1.1.0` → BCP 47. gRPC·admin 은 enum 이름 유지.

---

## 2. `POST` /api/v2/auth/resend-email

이메일 인증 메일 재발송.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 `email` (필수, `@ValidEmail(allowEmpty=false)`).

```
POST /api/v2/auth/resend-email?email=u@example.com
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## 3. `POST` /api/v2/auth/verify-email

인증 코드(6자리)로 이메일 검증.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "email": "u@example.com", "code": "123456" }
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

<details>
<summary><b>400 Bad Request</b> — 코드 무효/불일치</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid verification code" }
```

</details>

</details>

### Request 필드 정의 (공통)

`VerifyEmailByCodeRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`email`** | 문자열 | `@NotBlank` |
| **`code`** | 6자 | `@NotBlank @Size(min=6, max=6)` |

---

## 4. `POST` /api/v2/auth/login

이메일·비밀번호 로그인 → 토큰 발급.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "username": "u@example.com", "password": "P@ssw0rd!" }
```

### `1.0.0` — Response

**200 OK** — `token` 에 `CncTokenDto`

```json
{
  "success": true,
  "token": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "accessTokenExpiresIn": 900
  }
}
```

<details>
<summary><b>400 Bad Request · 401 Unauthorized</b> — 형식 위반 · 자격 증명 실패</summary>

```json
{ "success": false, "code": "AUTH-401-003", "message": "Authentication failed" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — 계정 잠김 (로그인 시도 초과)</summary>

```json
{ "success": false, "code": "AUTH-403-004", "message": "Your account is locked" }
```

</details>

</details>

### Response 필드 정의 — `CncTokenDto`

응답의 `token`. 필드 `@JsonInclude(NON_NULL)`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string (uuid) | — |
| `accessToken` | object | JWT access token wrapper |
| `refreshToken` | object | JWT refresh token wrapper |
| `expiresIn` | integer (int64) | access 만료 (초) |
| `refreshExpiresIn` | integer (int64) | refresh 만료 (초) |

> 예시 JSON 의 `accessTokenExpiresIn`(초) 는 클라이언트 관점 표기이며, 실제 DTO 필드는 `expiresIn`/`refreshExpiresIn`.

### Request 필드 정의 (공통)

| 필드 | 값 | 규칙 |
|---|---|---|
| **`username`** | 이메일 | `@ValidEmail` |
| **`password`** | 문자열 | `@ValidPassword` |

---

## 5. `POST` /api/v2/auth/google

모바일 Google OAuth → CareNCo 토큰 발급.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "serverAuthCode": "4/0Ab...", "idToken": "eyJ..." }
```

### `1.0.0` — Response

**201 Created** — CareNCo 발급 토큰 (`jwtToken` → `CncTokenDto`)

```json
{
  "success": true,
  "data": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "accessTokenExpiresIn": 900
  }
}
```

</details>

### Response 필드 정의 — `CncTokenDto`

응답의 `data` (`jwtToken`). 필드 `@JsonInclude(NON_NULL)`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string (uuid) | — |
| `accessToken` | object | JWT access token wrapper |
| `refreshToken` | object | JWT refresh token wrapper |
| `expiresIn` | integer (int64) | access 만료 (초) |
| `refreshExpiresIn` | integer (int64) | refresh 만료 (초) |

> 예시 JSON 의 `accessTokenExpiresIn`(초) 는 클라이언트 관점 표기이며, 실제 DTO 필드는 `expiresIn`/`refreshExpiresIn`. 컨트롤러는 `s.value().jwtToken()` 만 반환한다.

### Request 필드 정의 (공통)

`GoogleOAuthRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`serverAuthCode`** | 문자열 | `@NotBlank` |
| **`idToken`** | 문자열 | `@NotBlank` |

---

## 6. `POST` /api/v2/auth/apple

모바일 Apple OAuth → CareNCo 토큰 발급.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "identityToken": "eyJ...", "authorizationCode": "c1a..." }
```

### `1.0.0` — Response

**201 Created** — CareNCo 발급 토큰 (`jwtToken` → `CncTokenDto`)

```json
{
  "success": true,
  "data": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "accessTokenExpiresIn": 900
  }
}
```

</details>

### Response 필드 정의 — `CncTokenDto`

응답의 `data` (`jwtToken`). 필드 `@JsonInclude(NON_NULL)`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string (uuid) | — |
| `accessToken` | object | JWT access token wrapper |
| `refreshToken` | object | JWT refresh token wrapper |
| `expiresIn` | integer (int64) | access 만료 (초) |
| `refreshExpiresIn` | integer (int64) | refresh 만료 (초) |

> 예시 JSON 의 `accessTokenExpiresIn`(초) 는 클라이언트 관점 표기이며, 실제 DTO 필드는 `expiresIn`/`refreshExpiresIn`. 컨트롤러는 `s.value().jwtToken()` 만 반환한다.

### Request 필드 정의 (공통)

`AppleOAuthRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`identityToken`** | 문자열 | `@NotBlank` |
| **`authorizationCode`** | 문자열 | `@NotBlank` |

---

## 7. `POST` /api/v2/auth/token

access + refresh 토큰 쌍 재발급.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "refreshToken": "eyJ..." }
```

### `1.0.0` — Response

**200 OK** — `token` 에 새 `CncTokenDto`

```json
{
  "success": true,
  "token": {
    "userId": "3f2a...uuid",
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "accessTokenExpiresIn": 900
  }
}
```

<details>
<summary><b>401 Unauthorized</b> — refresh token 무효/만료</summary>

```json
{ "success": false, "code": "AUTH-401-001", "message": "Invalid or malformed token" }
```

</details>

</details>

### Response 필드 정의 — `CncTokenDto`

응답의 `token`. 필드 `@JsonInclude(NON_NULL)`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `userId` | string (uuid) | — |
| `accessToken` | object | JWT access token wrapper |
| `refreshToken` | object | JWT refresh token wrapper |
| `expiresIn` | integer (int64) | access 만료 (초) |
| `refreshExpiresIn` | integer (int64) | refresh 만료 (초) |

> 예시 JSON 의 `accessTokenExpiresIn`(초) 는 클라이언트 관점 표기이며, 실제 DTO 필드는 `expiresIn`/`refreshExpiresIn`.

### Request 필드 정의 (공통)

`ReissueTokenRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`refreshToken`** | 문자열 | `@NotBlank` |

---

## 8. `POST` /api/v2/auth/logout

refresh token 무효화 + 로그아웃 시각 기록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "refreshToken": "eyJ..." }
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

### Request 필드 정의 (공통)

`UserLogoutRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`refreshToken`** | 문자열 | `@NotBlank` |

---

## 9. `POST` /api/v2/auth/send-password-reset

비밀번호 재설정 메일 발송.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 `email` (필수, `@ValidEmail`).

```
POST /api/v2/auth/send-password-reset?email=u@example.com
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## 10. `POST` /api/v2/auth/send-account-recovery

계정 복구 메일 발송. **현재 1.0.0 미구현** — `ResponseHelper.unsupportedVersion()` 반환.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 `email` (`@ValidEmail`).

### `1.0.0` — Response

현재 미지원 버전 응답 반환 (구현 시 `data: null` 200 예정).

</details>

---

## users

| Method | Path (`/api/v2` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/users/{userId}`](#11-get-apiv2usersuserid) | **1.0.0, 1.1.0, 1.1.1** | Bearer | 프로필 조회 — 버전별 로케일 표기가 다르다 |
| PATCH | [`/users/{userId}`](#12-patch-apiv2usersuserid) | **1.0.0, 1.1.0, 1.1.1** | Bearer | 부분 수정 — absent=변경 없음, 명시적 `null`=지우기 시도 |
| PATCH | [`/users/{userId}/password`](#13-patch-apiv2usersuseridpassword) | 1.0.0 | Bearer | 기존 비밀번호 확인 후 변경 |
| POST | [`/users/{userId}/password`](#14-post-apiv2usersuseridpassword) | 1.0.0 | Bearer | 기존 비밀번호 없이 설정 (OAuth 가입 사용자용) |
| DELETE | [`/users/{userId}`](#15-delete-apiv2usersuserid) | 1.0.0 | Bearer | 회원 탈퇴 (삭제 대기 전환) |

---

## 11. `GET` /api/v2/users/{userId}

프로필 조회. 버전별 로케일 표기가 다르다.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | country·language 를 **enum 이름** 으로 표기 (`KR`) |
| `1.1.0` | country **alpha-3** (`KOR`) · language **BCP-47** (`ko-KR`) |
| `1.1.1` **(최신)** | = 1.1.0 + `weightUnit` |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.1.0` \| `1.1.1` | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid) 만.

### `1.0.0` — Response

**200 OK** — country·language 는 **enum 이름** (`KR`), 미지원 locale 은 `UNKNOWN`

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "firstName": "Jonghak", "lastName": "Lee", "nickname": "JH",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE", "providerId": "..." }],
    "photoUrl": null,
    "gender": "MALE", "birthdate": "1995-04-27", "height": 175.5,
    "countryCode": "KR",
    "languageCode": "KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — 타인 리소스 · 미존재/삭제 대기 계정</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

<details>
<summary><b><code>1.1.0</code> — Request · Response</b></summary>

### `1.1.0` — Request

바디 없음 — path 의 `userId` (uuid) 만.

### `1.1.0` — Response

**200 OK** — country=**alpha-3** (`KOR`), language=**BCP-47** (`ko-KR`)

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "firstName": "Jonghak", "lastName": "Lee", "nickname": "JH",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE", "providerId": "..." }],
    "photoUrl": null,
    "gender": "MALE", "birthdate": "1995-04-27", "height": 175.5,
    "countryCode": "KOR",
    "languageCode": "ko-KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — 타인 리소스 · 미존재/삭제 대기 계정</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

<details>
<summary><b><code>1.1.1</code> — Request · Response</b></summary>

### `1.1.1` — Request

바디 없음 — path 의 `userId` (uuid) 만.

### `1.1.1` — Response

**200 OK** — 1.1.0 shape + `weightUnit`

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "firstName": "Jonghak", "lastName": "Lee", "nickname": "JH",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE", "providerId": "..." }],
    "photoUrl": null,
    "gender": "MALE", "birthdate": "1995-04-27", "height": 175.5,
    "weightUnit": "KG",
    "countryCode": "KOR", "languageCode": "ko-KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>403 Forbidden · 404 Not Found</b> — 타인 리소스 · 미존재/삭제 대기 계정</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

### Response 필드 정의 — `UserResponse`

응답 바디의 `data`. 필드 순서 = 선언 순서.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `userProviders` | array<`UserProvider`> | `provider`(enum `Provider`), `providerId`(string) |
| `userEmail` | `UserEmail` | `emailLocal` `emailDomain` `emailFull`(string), `verified`(boolean), `verifiedAt`(Instant) |
| `firstName` `lastName` `nickname` | string | — |
| `phone` | `Phone` | `e164`, `countryCode`(v1.0.0 enum / v1.1.0·1.1.1 string), `national`, `formatted` |
| `photoUrl` | string | — |
| `gender` | string (enum `Gender`) | — |
| `birthdate` | string (date) | — |
| `height` | number (double) | — |
| `weightUnit` | string (enum `WeightUnit`) | **1.1.1만** |
| `countryCode` | v1.0.0 enum `CountryCode` / v1.1.0·1.1.1 string (alpha-3) | — |
| `languageCode` | string (v1.0.0 enum 이름 / v1.1.0·1.1.1 BCP-47) | — |
| `timeZone` | string (enum `TimeZone`) | — |
| `zoneId` | string (IANA) | — |
| `authorities` | array<`Authority`> | `role`(enum `Role`) |

> `@JsonIgnore` — `Authority`/`UserEmail`/`UserProvider` 의 `id`·`userId` 는 직렬화되지 않는다.

### 응답 필드 — 버전 대조

| 필드 | 1.0.0 | 1.1.0 | 1.1.1 |
|---|---|---|---|
| `countryCode` | enum 이름 (`KR`) | alpha-3 (`KOR`) | alpha-3 (`KOR`) |
| `languageCode` | enum 이름 (`KR`·`UNKNOWN`) | BCP-47 (`ko-KR`) | BCP-47 (`ko-KR`) |
| `phone.countryCode` | enum 이름 (`KR`) | string (`KR`) | string (`KR`) |
| `weightUnit` | — | — | `KG` \| `LB` |
| 나머지 (신원·연락처·프로필) | 동일 | 동일 | 동일 |

---

## 12. `PATCH` /api/v2/users/{userId}

부분 수정 — 모든 필드가 `JsonNullable<T>` 라 "안 보냄(absent) = 변경 없음", "명시적 `null` = 지우기 시도". 응답 로케일 표기는 GET §11 과 동일한 버전 규칙.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 요청 DTO v1_0_0 · 응답 로케일 **enum 이름** 표기 |
| `1.1.0` | 요청 DTO 는 `1.0.0` 과 동일(v1_0_0) · 응답 alpha-3/BCP-47 |
| `1.1.1` **(최신)** | = 1.1.0 + `weightUnit` |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.1.0` \| `1.1.1` | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "firstName": "John🔥",
  "nickname": "JD",
  "height": 176.0
}
```

### `1.0.0` — Response

**200 OK** — country·language 는 **enum 이름** 표기 (정규화된 저장값 반환)

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "firstName": "John", "lastName": "박", "nickname": "JD",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE", "providerId": "..." }],
    "gender": "MALE", "birthdate": "1995-04-27", "height": 176.0,
    "countryCode": "KR",
    "languageCode": "KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>null</code> 지우기 거부 · 정규화 후 빈 이름</summary>

```json
{ "success": false, "code": "CMN-400-002", "error": "gender cannot be null." }
```

</details>

</details>

<details>
<summary><b><code>1.1.0</code> — Request · Response</b></summary>

### `1.1.0` — Request

```json
{
  "firstName": "John🔥",
  "nickname": "JD",
  "height": 176.0
}
```

### `1.1.0` — Response

**200 OK** — country=**alpha-3**, language=**BCP-47**

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "firstName": "John", "lastName": "박", "nickname": "JD",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE", "providerId": "..." }],
    "gender": "MALE", "birthdate": "1995-04-27", "height": 176.0,
    "countryCode": "KOR",
    "languageCode": "ko-KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>null</code> 지우기 거부 · 정규화 후 빈 이름</summary>

```json
{ "success": false, "code": "CMN-400-002", "error": "gender cannot be null." }
```

</details>

</details>

<details>
<summary><b><code>1.1.1</code> — Request · Response</b></summary>

### `1.1.1` — Request (= + `weightUnit`)

```json
{
  "firstName": "John🔥",
  "nickname": "JD",
  "height": 176.0,
  "weightUnit": "KG"
}
```

### `1.1.1` — Response

**200 OK** — 1.1.0 shape + `weightUnit`

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "firstName": "John", "lastName": "박", "nickname": "JD",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE", "providerId": "..." }],
    "gender": "MALE", "birthdate": "1995-04-27", "height": 176.0,
    "weightUnit": "KG",
    "countryCode": "KOR", "languageCode": "ko-KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>null</code> 지우기 거부 · 정규화 후 빈 이름 · 잘못된 weightUnit</summary>

```json
{ "success": false, "code": "CMN-400-002", "error": "weightUnit cannot be null." }
```

</details>

</details>

### Request 필드 정의 (공통)

`UserUpdateRequest` — 모든 필드 `JsonNullable<T>` (null → `JsonNullable.undefined()` 정규화). 안 보냄=변경 없음, `null`=지우기 시도. 검증은 register validator 재사용.

| 필드 | `null` 지우기 | 비고 |
|---|---|---|
| `firstName` `lastName` | ❌ 400 | 정규화 수용 ¹ (validation 애너테이션 없음) |
| `gender` | ❌ 400 | `@ValidEnum(Gender)` |
| `weightUnit` | ❌ 400 | **1.1.1만** · `@ValidEnum(WeightUnit)` |
| `nickname` | ✅ 허용 | `@ValidNickname` 1~30자 |
| `phoneNumber` | — | `@ValidPhoneNumber(allowNull=true)` · 타인 사용 중이면 409 |
| `email` `birthdate` `height` `countryCode` `languageCode` `timeZone` `zoneId` | — | register 와 동일 규칙 (`@ValidEmail`·`@ValidBirthDate`·`@ValidHeight`·`@ValidLanguageTag` 등) |

### Response 필드 정의 — `UserResponse`

응답 바디의 `data` (GET §11 과 동일 DTO). 필드 순서 = 선언 순서.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `userProviders` | array<`UserProvider`> | `provider`(enum `Provider`), `providerId`(string) |
| `userEmail` | `UserEmail` | `emailLocal` `emailDomain` `emailFull`(string), `verified`(boolean), `verifiedAt`(Instant) |
| `firstName` `lastName` `nickname` | string | — |
| `phone` | `Phone` | `e164`, `countryCode`(v1.0.0 enum / v1.1.0·1.1.1 string), `national`, `formatted` |
| `photoUrl` | string | — |
| `gender` | string (enum `Gender`) | — |
| `birthdate` | string (date) | — |
| `height` | number (double) | — |
| `weightUnit` | string (enum `WeightUnit`) | **1.1.1만** |
| `countryCode` | v1.0.0 enum `CountryCode` / v1.1.0·1.1.1 string (alpha-3) | — |
| `languageCode` | string (v1.0.0 enum 이름 / v1.1.0·1.1.1 BCP-47) | — |
| `timeZone` | string (enum `TimeZone`) | — |
| `zoneId` | string (IANA) | — |
| `authorities` | array<`Authority`> | `role`(enum `Role`) |

> `@JsonIgnore` — `Authority`/`UserEmail`/`UserProvider` 의 `id`·`userId` 는 직렬화되지 않는다.

---

## 13. `PATCH` /api/v2/users/{userId}/password

기존 비밀번호 확인 후 변경.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "oldPassword": "OldP@ss1!", "newPassword": "NewP@ss2!" }
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

<details>
<summary><b>400 Bad Request · 401 Unauthorized</b> — 형식 위반 · 기존 비밀번호 불일치</summary>

```json
{ "success": false, "code": "AUTH-401-003", "message": "Authentication failed" }
```

</details>

</details>

### Request 필드 정의 (공통)

`UserPasswordUpdateRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`oldPassword`** | 문자열 | `@ValidPassword` |
| **`newPassword`** | 문자열 | `@ValidPassword` |

---

## 14. `POST` /api/v2/users/{userId}/password

기존 비밀번호 없이 설정 (OAuth 가입 사용자용).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "newPassword": "NewP@ss2!" }
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

### Request 필드 정의 (공통)

`UserPasswordSetRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`newPassword`** | 문자열 | `@ValidPassword` |

---

## 15. `DELETE` /api/v2/users/{userId}

회원 탈퇴 (삭제 대기 전환).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid) 만.

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## lookup

| Method | Path (`/api/v2` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/availability`](#16-get-apiv2availability) | 1.0.0 | 공개 | 필드 값의 사용 가능 여부(중복 아님) 확인 |
| GET | [`/exists`](#17-get-apiv2exists) | 1.0.0 | 공개 | 필드 값의 존재 여부 확인 |

---

## 16. `GET` /api/v2/availability

필드 값의 사용 가능 여부(중복 아님) 확인.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 `field`(필수, `@ValidEnum(DuplicationField)`) · `value`(필수).

```
GET /api/v2/availability?field=EMAIL&value=u@example.com
```

### `1.0.0` — Response

**200 OK**

```json
{
  "success": true,
  "data": { "field": "EMAIL", "value": "u@example.com", "available": true }
}
```

</details>

### Response 필드 정의 — `AvailabilityResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `field` | enum (`DuplicationField`) | 조회 대상 |
| `value` | string | 조회 값 |
| `available` | boolean | 사용 가능(중복 아님) 여부 |

---

## 17. `GET` /api/v2/exists

필드 값의 존재 여부 확인.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (공개 엔드포인트) | no |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 `field`(필수, `@ValidEnum(ExistsField)`) · `value`(필수).

```
GET /api/v2/exists?field=EMAIL&value=u@example.com
```

### `1.0.0` — Response

**200 OK**

```json
{
  "success": true,
  "data": { "field": "EMAIL", "value": "u@example.com", "exists": true }
}
```

</details>

### Response 필드 정의 — `ExistsResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `field` | enum (`ExistsField`) | 조회 대상 |
| `value` | string | 조회 값 |
| `exists` | boolean | 존재 여부 |

---

## sessions

| Method | Path (`/api/v2` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/users/{userId}/sessions`](#18-get-apiv2usersuseridsessions) | 1.0.0 | Bearer | 활성 세션 목록 |
| DELETE | [`/users/{userId}/sessions/{tokenId}`](#19-delete-apiv2usersuseridsessionstokenid) | 1.0.0 | Bearer | 특정 세션 무효화 |
| DELETE | [`/users/{userId}/sessions`](#20-delete-apiv2usersuseridsessions) | 1.0.0 | Bearer | 모든 세션 무효화 |
| GET | [`/users/{userId}/login-history`](#21-get-apiv2usersuseridlogin-history) | 1.0.0 | Bearer | 로그인 이력 페이지 — 기본 정렬 `createdAt,desc`, size 20 |

---

## 18. `GET` /api/v2/users/{userId}/sessions

활성 세션 목록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid) 만.

### `1.0.0` — Response

**200 OK** — `data:` `Session[]`

```json
{
  "success": true,
  "data": [
    {
      "tokenId": "jti-...",
      "deviceId": "...",
      "deviceType": "MOBILE",
      "createdAt": "2026-07-01T00:00:00Z",
      "expiresAt": "2026-07-08T00:00:00Z",
      "current": true
    }
  ]
}
```

</details>

### Response 필드 정의 — `Session`

| 필드 | 타입 | 설명 |
|---|---|---|
| `tokenId` | string | JWT id |
| `deviceId` | string | — |
| `deviceType` | string | `MOBILE` `WEB` 등 |
| `createdAt` | string (date-time, UTC) | — |
| `expiresAt` | string (date-time, UTC) | — |
| `current` | boolean | — |

> `Session` 응답 shape 은 `SessionService` 가 생성 — 전용 record DTO 가 아니므로 필드는 예시(코드 미검증). 정확한 필드는 `com.carenco.userservice.user.application.SessionService` 반환 타입 참조.

---

## 19. `DELETE` /api/v2/users/{userId}/sessions/{tokenId}

특정 세션 무효화.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid) · `tokenId` (문자열).

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## 20. `DELETE` /api/v2/users/{userId}/sessions

모든 세션 무효화.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid) 만.

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## 21. `GET` /api/v2/users/{userId}/login-history

로그인 이력 페이지. 기본 정렬 `createdAt,desc`, size `20`.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path `userId` (uuid) + Pageable 쿼리 (`page`=0, `size`=20, `sort`=`createdAt,desc`).

### `1.0.0` — Response

**200 OK** — `Page<LoginHistory>`

```json
{
  "success": true,
  "data": {
    "content": [
      { "action": "LOGIN", "ipAddress": "1.2.3.4", "description": "...", "timestamp": "2026-07-01T00:00:00Z" }
    ],
    "totalElements": 5, "totalPages": 1, "number": 0, "size": 20
  }
}
```

</details>

### Response 필드 정의 — `LoginHistory`

| 필드 | 타입 | 설명 |
|---|---|---|
| `action` | string | `LOGIN` `LOGOUT` 등 |
| `ipAddress` | string | — |
| `description` | string | — |
| `timestamp` | string (date-time, UTC) | — |

> `LoginHistory` 응답 shape 은 `SessionService` 가 생성 — 전용 record DTO 가 아니므로 필드는 예시(코드 미검증). 정확한 필드는 `com.carenco.userservice.user.application.SessionService` 반환 타입 참조.

---

## terms

| Method | Path (`/api/v2` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/users/{userId}/terms-consents`](#22-post-apiv2usersuseridterms-consents) | 1.0.0 | Bearer | 동의 기록 (`(termsType, termsVersion)` 멱등) |
| GET | [`/users/{userId}/terms-consents`](#23-get-apiv2usersuseridterms-consents) | 1.0.0 | Bearer | 약관 동의 이력 (`agreedAt` 내림차순) |

---

## 22. `POST` /api/v2/users/{userId}/terms-consents

동의 기록 (`(termsType, termsVersion)` 멱등).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 USER/GUEST 또는 ADMIN) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "items": [
    { "termsType": "SERVICE", "termsVersion": "1.0" },
    { "termsType": "PRIVACY", "termsVersion": "1.2" }
  ]
}
```

### `1.0.0` — Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

<details>
<summary><b>400 Bad Request</b> — items 비어있음 · termsType enum 위반</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: items[0].termsType" }
```

</details>

</details>

### Request 필드 정의 (공통)

`TermsConsentRecordRequest` — `items` 는 `ConsentItem[]`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`items`** | 배열 | `@NotEmpty @Valid` |
| **`items[].termsType`** | enum | `@ValidEnum(TermsType)` |
| **`items[].termsVersion`** | 문자열 | `@NotBlank @Size(max=64)` |

---

## 23. `GET` /api/v2/users/{userId}/terms-consents

약관 동의 이력 (`agreedAt` 내림차순). ADMIN 또는 (USER/**GUEST** 본인).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 USER/GUEST 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid) 만.

### `1.0.0` — Response

**200 OK** — `data:` `TermsConsentItem[]`

```json
{
  "success": true,
  "data": [
    { "termsType": "SERVICE", "termsVersion": "1.0", "agreedAt": "2026-07-01T00:00:00Z" },
    { "termsType": "PRIVACY", "termsVersion": "1.2", "agreedAt": "2026-07-01T00:00:00Z" }
  ]
}
```

</details>

### Response 필드 정의 — `TermsConsentItem` (`TermsConsentItemResponse`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `termsType` | string (enum `TermsType`) | — |
| `termsVersion` | string | — |
| `agreedAt` | string (date-time, UTC) | — |

---

## b2b-centers

| Method | Path (`/api/v2` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/users/{userId}/b2b-centers/{organizationId}/license-detail`](#24-get-apiv2usersuseridb2b-centersorganizationidlicense-detail) | 1.0.0 | Bearer | B2C 회원이 가입한 b2b 시설의 이용권 상세 |

---

## 24. `GET` /api/v2/users/{userId}/b2b-centers/{organizationId}/license-detail

B2C 회원이 가입한 b2b 시설의 이용권 상세. b2b-service `B2cMemberQuery.GetLicenseDetail(carencoUserId, organizationId)` gRPC 를 합성 (0.0.78).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인 또는 ADMIN) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid, `@ValidUuid`) · `organizationId` (uuid, `@ValidUuid`).

### `1.0.0` — Response

**200 OK** — 가입 + 구독 ACTIVE 예시

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

<details>
<summary><b>200 OK</b> — 미가입</summary>

```json
{ "success": true, "data": { "registered": false, "managers": [] } }
```

</details>

<details>
<summary><b>401 Unauthorized · 502 Bad Gateway</b> — resource-owner 불일치 · b2b-service gRPC 실패</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

502 = b2b-service 다운/deadline.

</details>

</details>

### Response 필드 정의 — `B2bCenterLicenseDetail`

b2b-service gRPC 합성 (로컬 record 아님).

| 필드 | 타입 | 설명 |
|---|---|---|
| `registered` | boolean | `false` 면 멤버십 없음 (나머지 필드 null) |
| `organizationId` | string (uuid) | — |
| `organizationName` | string | — |
| `organizationType` | string | `PILATES` 등 |
| `photoUrl` | string \| null | — |
| `membershipStatus` | string | `ACTIVE` 등 |
| `memberNumber` | string | — |
| `joinedAt` | string (date-time, UTC) | — |
| `licenseState` | string | `ACTIVE` / `GRACE` / `EXPIRED` / `SUSPENDED` / `NONE` |
| `startedAt` `expiresAt` | string (date-time, UTC) | 구독 기간 |
| `canceledAt` | string (date-time, UTC) \| null | 갱신 취소 예약 시각(cycle-end cancel, 그전까지 ACTIVE). null=취소 없음 |
| `managers` | array<`{ firstName, lastName, role }`> | OWNER + ACTIVE ADMIN staff (공유자) |

> 가입 시설 **목록** 은 b2b-service `GET /api/v2/b2b/b2c-members/{carencoUserId}/organizations` 직접 호출.

---

## public

`permitAll` — 내부 service-to-service 호출용.

| Method | Path (`/api/public` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/users/{userId}`](#25-get-apipublicusersuserid) | 1.0.0 | permitAll(내부) | Service-to-service 유저 조회 |
| POST | [`/users/{userId}/revoke-tokens`](#26-post-apipublicusersuseridrevoke-tokens) | 1.0.0 | permitAll(내부) | Service-to-service 토큰 무효화 |
| DELETE | [`/users/{userId}`](#27-delete-apipublicusersuserid) | 1.0.0 | permitAll(내부) | Service-to-service 유저 삭제 |

---

## 25. `GET` /api/public/users/{userId}

Service-to-service 유저 조회.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (`permitAll`, 내부 호출용) | no |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `userId` (uuid, `@ValidUuid`) 만.

### Response

**200 OK** — `data:` `UserResponse` (v1_0_0, enum 이름 표기)

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "userProviders": [{ "provider": "GOOGLE", "providerId": "..." }],
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "firstName": "Jonghak", "lastName": "Lee", "nickname": "JH",
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "photoUrl": null,
    "gender": "MALE", "birthdate": "1995-04-27", "height": 175.5,
    "countryCode": "KR", "languageCode": "KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

</details>

### Response 필드 정의 — `UserResponse`

응답 바디의 `data` (GET §11 과 동일 DTO, v1_0_0 enum 이름 표기). 필드 순서 = 선언 순서.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `userProviders` | array<`UserProvider`> | `provider`(enum `Provider`), `providerId`(string) |
| `userEmail` | `UserEmail` | `emailLocal` `emailDomain` `emailFull`(string), `verified`(boolean), `verifiedAt`(Instant) |
| `firstName` `lastName` `nickname` | string | — |
| `phone` | `Phone` | `e164`, `countryCode`(enum), `national`, `formatted` |
| `photoUrl` | string | — |
| `gender` | string (enum `Gender`) | — |
| `birthdate` | string (date) | — |
| `height` | number (double) | — |
| `countryCode` | string (enum `CountryCode`) | public 은 v1_0_0 표기 |
| `languageCode` | string (enum `LanguageCode`) | — |
| `timeZone` | string (enum `TimeZone`) | — |
| `zoneId` | string (IANA) | — |
| `authorities` | array<`Authority`> | `role`(enum `Role`) |

> `@JsonIgnore` — `Authority`/`UserEmail`/`UserProvider` 의 `id`·`userId` 는 직렬화되지 않는다.

---

## 26. `POST` /api/public/users/{userId}/revoke-tokens

Service-to-service 토큰 무효화.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (`permitAll`) | no |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `userId` (uuid) 만.

### Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## 27. `DELETE` /api/public/users/{userId}

Service-to-service 유저 삭제.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | — (`permitAll`) | no |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `userId` (uuid) 만.

### Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## admin

전부 **unversioned** — `api-version` 헤더 무시.

| Method | Path (`/api/admin` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/users`](#28-get-apiadminusers) | — | Bearer(ADMIN) | 회원 목록 페이지 — 기본 정렬 `createdAt,desc`, size 20 |
| GET | [`/users/{userId}`](#29-get-apiadminusersuserid) | — | Bearer(ADMIN) | 회원 단건 조회 |
| PATCH | [`/users/{userId}/status`](#30-patch-apiadminusersuseridstatus) | — | Bearer(ADMIN) | 회원 상태 변경 |
| POST | [`/users/{username}/unlock`](#31-post-apiadminusersusernameunlock) | — | Bearer(ADMIN) | 로그인 잠금 해제 |
| GET | [`/audit/users/{userId}`](#32-get-apiadminauditusersuserid) | — | Bearer(ADMIN) | 특정 유저의 감사 로그 페이지 |
| GET | [`/audit?action=`](#33-get-apiadminauditaction) | — | Bearer(ADMIN) | action 유형별 감사 로그 페이지 |
| GET | [`/audit?actorId=`](#34-get-apiadminauditactorid) | — | Bearer(ADMIN) | actor 별 감사 로그 페이지 |

---

## 28. `GET` /api/admin/users

회원 목록 페이지. **unversioned** (api-version 헤더 무시). 기본 정렬 `createdAt,desc`, size `20`.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Authorization` | `Bearer <jwt>` (`ROLE_ADMIN`) | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — Pageable 쿼리 (`page`=0, `size`=20, `sort`=`createdAt,desc`).

### Response

**200 OK** — `Page<AdminUser>`

```json
{
  "success": true,
  "data": {
    "content": [ { "id": "...", "email": "u@x.com", "deletionStatus": "NONE" } ],
    "totalElements": 100, "totalPages": 5, "number": 0, "size": 20
  }
}
```

</details>

### Response 필드 정의 — `AdminUser` (`AdminUserResponse`)

`content[]` 항목.

| 필드 | 타입 |
|---|---|
| `id` `email` `firstName` `lastName` `nickname` `phoneNumber` | string |
| `gender` | string |
| `birthdate` | string (date) |
| `height` | number (double) |
| `countryCode` `languageCode` `zoneId` | string |
| `deletionStatus` | string |
| `emailVerified` | boolean |
| `lastLoginTime` `lastLogoutTime` `createdAt` `updatedAt` | string (date-time, UTC) |

---

## 29. `GET` /api/admin/users/{userId}

회원 단건 조회. **unversioned**.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Authorization` | `Bearer <jwt>` (`ROLE_ADMIN`) | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `userId` (문자열, `@ValidUuid` 없음).

### Response

**200 OK** — `AdminUser`

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid", "email": "u@x.com",
    "firstName": "Jonghak", "lastName": "Lee", "nickname": "JH", "phoneNumber": "+821012345678",
    "gender": "MALE", "birthdate": "1995-04-27", "height": 175.5,
    "countryCode": "KOR", "languageCode": "ko-KR", "zoneId": "Asia/Seoul",
    "deletionStatus": "NONE", "emailVerified": true,
    "lastLoginTime": "2026-07-01T00:00:00Z", "lastLogoutTime": "2026-07-01T01:00:00Z",
    "createdAt": "2026-01-01T00:00:00Z", "updatedAt": "2026-07-01T00:00:00Z"
  }
}
```

</details>

### Response 필드 정의 — `AdminUser` (`AdminUserResponse`)

| 필드 | 타입 |
|---|---|
| `id` `email` `firstName` `lastName` `nickname` `phoneNumber` | string |
| `gender` | string |
| `birthdate` | string (date) |
| `height` | number (double) |
| `countryCode` `languageCode` `zoneId` | string |
| `deletionStatus` | string |
| `emailVerified` | boolean |
| `lastLoginTime` `lastLogoutTime` `createdAt` `updatedAt` | string (date-time, UTC) |

---

## 30. `PATCH` /api/admin/users/{userId}/status

회원 상태 변경. **unversioned**.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Authorization` | `Bearer <jwt>` (`ROLE_ADMIN`) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

```json
{ "status": "SUSPENDED", "reason": "policy violation" }
```

### Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

### Request 필드 정의 (공통)

`UpdateStatusRequest`.

| 필드 | 값 | 규칙 |
|---|---|---|
| **`status`** | 문자열 | `@NotBlank` |
| `reason` | 문자열 | 선택 |

---

## 31. `POST` /api/admin/users/{username}/unlock

로그인 잠금 해제. **unversioned**.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Authorization` | `Bearer <jwt>` (`ROLE_ADMIN`) | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path 의 `username` (문자열).

### Response

**200 OK** — `data: null`

```json
{ "success": true, "data": null }
```

</details>

---

## 32. `GET` /api/admin/audit/users/{userId}

특정 유저의 감사 로그 페이지. **unversioned**.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Authorization` | `Bearer <jwt>` (`ROLE_ADMIN`) | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — path `userId` + 쿼리 `page`(기본 0) · `pageSize`(기본 20).

### Response

**200 OK** — `AuditPageResponse`

```json
{
  "success": true,
  "data": {
    "items": [ { "id": "...", "actorId": "...", "action": "UPDATE", "createdAt": "2026-07-01T00:00:00Z" } ],
    "page": 0, "size": 20, "totalElements": 3, "totalPages": 1, "hasNext": false
  }
}
```

</details>

### Response 필드 정의 — `AuditPageResponse`

| 필드 | 타입 |
|---|---|
| `items` | `AuditRecordResponse[]` |
| `page` | integer |
| `size` | integer |
| `totalElements` | integer (int64) |
| `totalPages` | integer |
| `hasNext` | boolean |

> `AuditRecordResponse` 항목 필드(예시, 세부 미검증) — `id` `actorId` `action` `entityType` `entityId` `changes`(JSON) `ipAddress` `traceId` `description` `createdAt`.

---

## 33. `GET` /api/admin/audit?action=

action 유형별 감사 로그 페이지. **unversioned**. `/api/admin/audit` 경로에 `params=action` 셀렉터로 구분.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Authorization` | `Bearer <jwt>` (`ROLE_ADMIN`) | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — 쿼리 `action`(필수) · `page`(기본 0) · `pageSize`(기본 20).

### Response

**200 OK** — `AuditPageResponse` (§32 와 동일 shape)

```json
{
  "success": true,
  "data": {
    "items": [ { "id": "...", "actorId": "...", "action": "UPDATE", "createdAt": "2026-07-01T00:00:00Z" } ],
    "page": 0, "size": 20, "totalElements": 3, "totalPages": 1, "hasNext": false
  }
}
```

</details>

### Response 필드 정의 — `AuditPageResponse`

| 필드 | 타입 |
|---|---|
| `items` | `AuditRecordResponse[]` |
| `page` | integer |
| `size` | integer |
| `totalElements` | integer (int64) |
| `totalPages` | integer |
| `hasNext` | boolean |

> `AuditRecordResponse` 항목 필드(예시, 세부 미검증) — `id` `actorId` `action` `entityType` `entityId` `changes`(JSON) `ipAddress` `traceId` `description` `createdAt`.

---

## 34. `GET` /api/admin/audit?actorId=

actor 별 감사 로그 페이지. **unversioned**. `params=actorId` 셀렉터로 구분.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Authorization` | `Bearer <jwt>` (`ROLE_ADMIN`) | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디 없음 — 쿼리 `actorId`(필수) · `page`(기본 0) · `pageSize`(기본 20).

### Response

**200 OK** — `AuditPageResponse` (§32 와 동일 shape)

```json
{
  "success": true,
  "data": {
    "items": [ { "id": "...", "actorId": "...", "action": "UPDATE", "createdAt": "2026-07-01T00:00:00Z" } ],
    "page": 0, "size": 20, "totalElements": 3, "totalPages": 1, "hasNext": false
  }
}
```

</details>

### Response 필드 정의 — `AuditPageResponse`

| 필드 | 타입 |
|---|---|
| `items` | `AuditRecordResponse[]` |
| `page` | integer |
| `size` | integer |
| `totalElements` | integer (int64) |
| `totalPages` | integer |
| `hasNext` | boolean |

> `AuditRecordResponse` 항목 필드(예시, 세부 미검증) — `id` `actorId` `action` `entityType` `entityId` `changes`(JSON) `ipAddress` `traceId` `description` `createdAt`.

---

## OAuth2 (Deprecated `/api/v2/auth/oauth2`)

`@Deprecated(since = "1.0.0")`. 현재 라우팅에서 사용 안 함. **unversioned**, `permitAll`.

| Method | Path | Response |
|---|---|---|
| GET | `/api/v2/auth/oauth2/register` | 200 — Google authorization URL 문자열 |
| GET | `/api/v2/auth/oauth2/success` | 200 — 토큰 발급 / 401 |
| GET | `/api/v2/auth/oauth2/failure` | 401 |

---

## Auth (Thymeleaf views)

`@Controller` (REST 아님) — 브라우저에서 열리는 HTML 렌더링. security/version 애너테이션 없음.

| Method | Path | Query | Description |
|---|---|---|---|
| GET | `/api/v2/auth/verify-email` | `token` | 인증 결과 페이지 |
| GET | `/api/v2/auth/reset-password` | `token` | 비밀번호 재설정 폼 |
| POST | `/api/v2/auth/reset-password` | `token`, `newPassword`, `confirmPassword` | 재설정 제출 |

---

## 각주

> ¹ **이름 정규화(B안, 2026-07-07)** — `firstName`/`lastName` 은 거부하지 않고 정규화 수용. 숫자 유지, 이모지·기호 제거, 50 UTF-8 bytes 절단. 전부 제거되는 입력만 400 + 서버 WARN 로그(마스킹). PATCH 의 임시 readonly(394b040) 는 해제됨.
> ² birthdate 없으면 측정(footprint) 생성 시 400.
> ³ **입력** — `countryCode` 는 alpha-2/alpha-3(`KOR`)/레거시 별칭 허용(0.0.83~). `languageCode` 는 enum 이름(`ZH_HANS`)·BCP-47(`ko-KR`) 허용(0.0.82~). **저장·응답** (0.0.84~) — DB 는 alpha-3/BCP-47(V33). 응답은 api-version 별 (`1.0.0` enum 이름, `1.1.0` alpha-3/BCP-47). gRPC·admin 은 enum/alpha-2 유지. 상세 표는 §1 register 의 `countryCode`/`languageCode` 허용값 참조.

## 에러 코드

> 소스 — common-core `ErrorCode` enum (`error/dto/v1_0_0/ErrorCode.java`). 코드·HTTP status·i18n 키가 enum 에서 자동 연결.

| 코드 | HTTP | 상황 |
|---|---|---|
| `CMN-400-001` | 400 | 잘못된 요청 형식·누락 파라미터·enum 위반·이름 정규화 후 빈 값 ¹·미등록 api-version (`CHECK_PARAMETER`) |
| `CMN-400-002` | 400 | bean validation 위반 — `@ValidUuid`, NOT NULL 필드 `null` 지우기 (`VALIDATION_FAILED`) |
| `AUTH-401-001` | 401 | 토큰 누락/무효 (`TOKEN_INVALID`) |
| `AUTH-401-002` | 401 | 토큰 만료 (`TOKEN_EXPIRED`) |
| `AUTH-401-003` | 401 | 인증 실패 (`AUTHENTICATION_FAILED`) |
| `AUTH-403-001` | 403 | 리소스 오너/권한 불일치 (`ACCESS_DENIED`) |
| `AUTH-403-004` | 403 | 계정 잠김 (`ACCOUNT_LOCKED`) |
| `CMN-404-001` | 404 | 리소스 미존재·삭제 대기 계정 (`RESOURCE_NOT_FOUND`) |
| `USR-404-001` | 404 | 유저 미존재 (`USER_NOT_FOUND`) |
| `CMN-409-001` | 409 | 이메일·닉네임·전화번호 중복 (`DUPLICATE_REQUEST`) |
