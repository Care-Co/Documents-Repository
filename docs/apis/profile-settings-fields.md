# 프로필·측정 설정 API — 필드 정의 (2026-07-08)

> register / GET·PATCH users / capture-settings 의 Header · Request · Response 정의 — **버전별 전량 전개**. 소스는 실제 DTO.
> 표기 — **굵은 필드 = 필수**, 규칙은 키워드만, 설명은 각주(¹²³⁴). 버전 매칭은 정확 일치 (미등록 버전 → `400 Invalid API version`).

---

## 1. `POST /api/v2/auth/register`

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.1.1` | yes |
| `Content-Type` | `application/json` | yes |
| `Authorization` | — (공개 엔드포인트) | no |

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
  "countryCode": "KR",
  "languageCode": "ko-KR",
  "timeZone": "KOREA",
  "zoneId": "Asia/Seoul",
  "weightUnit": "LB"
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

### Request 필드 정의 (공통)

**계정 (필수)**

| 필드 | 값 | 규칙 |
|---|---|---|
| **`username`** | 이메일 | 중복 409 |
| **`password`** | 8자+ | 영문·숫자·특수문자 |

**프로필 (선택)**

| 필드 | 값 | 규칙 |
|---|---|---|
| `firstName` `lastName` | 문자열 | 정규화 수용 ¹ |
| `nickname` | 1~30자 | 미제공 시 firstName 사용 |
| `phoneNumber` | E.164 (`+8210...`) | 중복 409 |
| `gender` | `MALE` `FEMALE` `OTHER` | 미제공 → `UNKNOWN` |
| `birthdate` | `YYYY-MM-DD` | 측정 전 필수 ² |
| `height` | number (cm) | |

**로케일·단위 (선택 — 미제공 시 기본값)**

| 필드 | 값 | 기본값 | 버전 |
|---|---|---|---|
| `weightUnit` | `KG` `LB` | `KG` | **1.1.1만** (1.0.0에 보내면 무시) |
| `countryCode` | alpha-2/3 아무거나 ³ | `UNKNOWN` | 전 버전 |
| `languageCode` | BCP-47 ³ | `UNKNOWN` | 전 버전 |
| `timeZone` / `zoneId` | enum / `Asia/Seoul` (zoneId 우선) | | 전 버전 |

---

## 2. `PATCH /api/v2/users/{userId}`

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.1.0` \| `1.1.1` | yes |
| `Authorization` | `Bearer <jwt>` (본인만) | yes |
| `Content-Type` | `application/json` | yes |

부분 수정 — **안 보낸 필드 = 변경 없음**, **`null` = 지우기 시도**.

### `1.0.0` — Request

```json
{
  "firstName": "John🔥",
  "nickname": "JD",
  "height": 176.0
}
```

### `1.0.0` — Response

**200 OK** — country·language 는 **enum 이름** 표기

```json
{
  "success": true,
  "data": {
    "id": "3f2a...uuid",
    "firstName": "John",
    "lastName": "박",
    "nickname": "JD",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE" }],
    "gender": "MALE", "birthdate": "1995-04-27", "height": 176.0,
    "countryCode": "KR",
    "languageCode": "KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — `null` 지우기 거부 · 정규화 후 빈 이름</summary>

```json
{ "success": false, "code": "CMN-400-002", "error": "gender cannot be null." }
```

</details>

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
    "firstName": "John",
    "lastName": "박",
    "nickname": "JD",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE" }],
    "gender": "MALE", "birthdate": "1995-04-27", "height": 176.0,
    "countryCode": "KOR",
    "languageCode": "ko-KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — `null` 지우기 거부 · 정규화 후 빈 이름</summary>

```json
{ "success": false, "code": "CMN-400-002", "error": "gender cannot be null." }
```

</details>

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
    "firstName": "John",
    "lastName": "박",
    "nickname": "JD",
    "userEmail": { "emailFull": "u@x.com", "verified": true, "verifiedAt": "2026-07-01T00:00:00Z" },
    "phone": { "e164": "+821012345678", "countryCode": "KR", "national": "010-1234-5678", "formatted": "+82 10-1234-5678" },
    "userProviders": [{ "provider": "GOOGLE" }],
    "gender": "MALE", "birthdate": "1995-04-27", "height": 176.0,
    "weightUnit": "KG",
    "countryCode": "KOR", "languageCode": "ko-KR",
    "timeZone": "ASIA_SEOUL", "zoneId": "Asia/Seoul",
    "authorities": [{ "role": "USER" }]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — `null` 지우기 거부 · 정규화 후 빈 이름 · 잘못된 weightUnit</summary>

```json
{ "success": false, "code": "CMN-400-002", "error": "weightUnit cannot be null." }
```

</details>

### Request 필드 정의 (공통)

| 필드 | `null` 지우기 | 비고 |
|---|---|---|
| `firstName` `lastName` | ❌ 400 | 정규화 수용 ¹ |
| `gender` | ❌ 400 | |
| `weightUnit` | ❌ 400 | **1.1.1만** |
| `nickname` | ✅ 허용 | 1~30자 |
| `phoneNumber` | — | 타인 사용 중이면 409 |
| `email` `birthdate` `height` `countryCode` `languageCode` `timeZone` `zoneId` | — | register 와 동일 규칙 |

---

## 3. `GET /api/v2/users/{userId}`

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.1.0` \| `1.1.1` | yes |
| `Authorization` | `Bearer <jwt>` (본인만) | yes |

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
    "userProviders": [{ "provider": "GOOGLE" }],
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

(본문 예시 없음)

</details>

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
    "userProviders": [{ "provider": "GOOGLE" }],
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

(본문 예시 없음)

</details>

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
    "userProviders": [{ "provider": "GOOGLE" }],
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

(본문 예시 없음)

</details>

### 응답 필드 — 버전 대조

| 필드 | 1.0.0 | 1.1.0 | 1.1.1 |
|---|---|---|---|
| `countryCode` | enum 이름 (`KR`) | alpha-3 (`KOR`) | alpha-3 (`KOR`) |
| `languageCode` | enum 이름 (`KR`·`UNKNOWN`) | BCP-47 (`ko-KR`) | BCP-47 (`ko-KR`) |
| `weightUnit` | — | — | `KG` \| `LB` |
| 나머지 (신원·연락처·프로필) | 동일 | 동일 | 동일 |

---

## 4. `GET / PUT /api/v2/users/{userId}/capture-settings`

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` (본인만) | yes |
| `Content-Type` | `application/json` (PUT) | PUT만 |

### `1.0.0` — Request (GET)

바디 없음 — path 의 `userId` (uuid) 만.

### `1.0.0` — Response (GET)

**200 OK** — 저장 행이 없으면 기본값

```json
{
  "success": true,
  "data": {
    "captureMode": "SELF",
    "shutterMode": "COUNTDOWN",
    "countdownSeconds": 5,
    "rememberSettings": true
  }
}
```

### `1.0.0` — Request (PUT — 4필드 전부 필수, 전체 upsert)

```json
{
  "captureMode": "ASSISTED",
  "shutterMode": "MOTION",
  "countdownSeconds": 3,
  "rememberSettings": false
}
```

| 필드 | 값 | 기본값 |
|---|---|---|
| **`captureMode`** | `SELF` (혼자 촬영) · `ASSISTED` (타인 촬영) | `SELF` |
| **`shutterMode`** | `COUNTDOWN` (타이머) · `MOTION` (동작인식) | `COUNTDOWN` |
| **`countdownSeconds`** | 1~10 | `5` |
| **`rememberSettings`** | boolean ⁴ | `true` |

### `1.0.0` — Response (PUT)

**200 OK** — 저장된 값 반환

```json
{
  "success": true,
  "data": {
    "captureMode": "ASSISTED",
    "shutterMode": "MOTION",
    "countdownSeconds": 3,
    "rememberSettings": false
  }
}
```

<details>
<summary><b>400 Bad Request</b> — enum·범위 위반</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Invalid request parameter. Field: captureMode, Value: FOO" }
```

</details>

---

## 각주

> ¹ **이름 정규화(B안)** — 거부하지 않음. 숫자 유지, 이모지·기호 제거, 50 UTF-8 bytes 절단. 전부 제거되는 입력만 400 + 서버 WARN 로그(마스킹).
> ² birthdate 없으면 측정(footprint) 생성 시 400.
> ³ 저장 표준 — country는 alpha-3(`KOR`)로, language는 지원 목록만(`ko-KR` 등, 미지원 태그·미제공 → `UNKNOWN`).
> ⁴ `false` = 측정마다 선택 UI 노출. 값 자체는 저장되어 다음 픽커의 기본값으로 쓰인다.

## 에러 코드

| 코드 | HTTP | 상황 |
|---|---|---|
| `CMN-400-001` | 400 | 형식 위반 · 이름 정규화 후 빈 값 · enum/범위 위반 · 미등록 api-version |
| `CMN-400-002` | 400 | NOT NULL 필드 `null` 지우기 시도 |
| `CMN-409-001` | 409 | 이메일·전화번호 중복 |
| `AUTH-403-001` | 403 | 타인 리소스 접근 |
| `CMN-404-*` | 404 | 미존재 · 삭제 대기 계정 |
