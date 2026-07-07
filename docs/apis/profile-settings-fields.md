# 프로필·측정 설정 API — 필드 정의 (2026-07-07)

> user-service register/GET/PATCH · measure-service capture-settings 의 요청/응답 필드 전정의.
> 소스: 실제 DTO (`UserRegisterRequest v1_1_1`, `UserUpdateRequest v1_1_1`, `UserResponse v1_1_1`, `CaptureSettingsUpdateRequest/Response`).

---

## 1. `POST /api/v2/auth/register` — 요청 필드

`api-version: 1.1.1` 기준. `1.0.0` 은 `weightUnit` 없음 (보내면 무시).

| 필드 | 타입 | 필수 | 검증 · 처리 규칙 |
|---|---|---|---|
| `username` | string | **yes** | `@ValidEmail` — 이메일 형식 |
| `password` | string | **yes** | `@ValidPassword` — 8자+, 영문/숫자/특수문자 |
| `firstName` | string | no | **B안 정규화 수용** — 숫자 허용, 이모지·기호 제거, 50 UTF-8 bytes 절단. 정규화 후 빈 값만 400 |
| `lastName` | string | no | firstName 과 동일 |
| `nickname` | string | no | `@ValidNickname` — 글자/숫자 시작, `'` `’` 공백 `.` `_` `-` 허용, 1~30자. 미제공 시 firstName 사용 |
| `phoneNumber` | string | no | `@ValidPhoneNumber` — E.164 (`+8210...`). 중복 시 409 |
| `gender` | string | no | `MALE` \| `FEMALE` \| `OTHER` \| `UNKNOWN` (미제공 → UNKNOWN) |
| `birthdate` | string | no | `@ValidBirthDate` — `YYYY-MM-DD`. **측정(footprint) 사용 전 필수** — 없으면 측정 400 |
| `height` | number | no | `@ValidHeight` (cm) |
| `weightUnit` | string | no | **1.1.1 전용** — `KG` \| `LB`. 미제공 시 `KG`. 표시 전용 (저장 체중은 항상 kg) |
| `countryCode` | string | no | ISO 3166-1 전량 — alpha-2(`KR`)/alpha-3(`KOR`)/별칭(`UK`,`UAE`) 수용, 저장은 alpha-3. 미제공 → UNKNOWN |
| `languageCode` | string | no | BCP-47 well-formed 수용, **저장은 지원 enum 화이트리스트** — 미지원 태그·미제공은 `UNKNOWN` |
| `timeZone` | string | no | `KOREA` 등 TimeZone enum |
| `zoneId` | string | no | `Asia/Seoul` 형식 — timeZone 과 택1 (zoneId 우선) |

---

## 2. `PATCH /api/v2/users/{userId}` — 요청 필드

모든 필드 `JsonNullable<T>` — **안 보냄(absent) = 변경 없음**, **명시적 `null` = clear 시도** (NOT NULL 필드는 400).

| 필드 | 타입 | null clear | 검증 · 처리 규칙 |
|---|---|---|---|
| `firstName` | string? | 400 | B안 정규화 수용 (register 와 동일) |
| `lastName` | string? | 400 | 〃 |
| `nickname` | string? | 허용 | `@ValidNickname` 1~30자 |
| `email` | string? | — | `@ValidEmail` |
| `phoneNumber` | string? | — | E.164. **타인 사용 중이면 409** |
| `gender` | string? | 400 | Gender enum |
| `birthdate` | string? | — | `YYYY-MM-DD` |
| `height` | number? | — | cm |
| `weightUnit` | string? | **400** | **1.1.1 전용** — `KG` \| `LB` |
| `countryCode` | string? | — | ISO 3166-1 (register 와 동일 수용 규칙) |
| `languageCode` | string? | — | BCP-47 → 화이트리스트 저장 |
| `timeZone` / `zoneId` | string? | — | TimeZone enum / IANA zone |

---

## 3. `GET /api/v2/users/{userId}` — 응답 필드 (1.1.1)

| 필드 | 타입 | 비고 |
|---|---|---|
| `id` | string (uuid) | |
| `userProviders` | array | `[{ provider: LOCAL\|GOOGLE\|APPLE, providerId }]` |
| `userEmail` | object | `{ emailFull, verified, verifiedAt }` |
| `firstName` / `lastName` / `nickname` | string | 정규화된 저장값 |
| `phone` | object | `{ e164, countryCode, national, formatted }` |
| `photoUrl` | string | |
| `gender` | enum | `MALE` \| `FEMALE` \| `OTHER` \| `UNKNOWN` |
| `birthdate` | date | `YYYY-MM-DD` |
| `height` | number | cm |
| `weightUnit` | enum | **1.1.1 전용** — `KG` \| `LB` (1.0.0/1.1.0 응답엔 없음) |
| `countryCode` | string | **1.1.0+: alpha-3** (`KOR`) / 1.0.0: enum 이름 |
| `languageCode` | string | **1.1.0+: BCP-47** (`ko-KR`) / 1.0.0: enum 이름 |
| `timeZone` | enum | `ASIA_SEOUL` 등 |
| `zoneId` | string | `Asia/Seoul` |
| `authorities` | array | `[{ role: USER\|ADMIN }]` |

---

## 4. `GET/PUT /api/v2/users/{userId}/capture-settings` — 필드

요청(PUT — 4필드 전부 필수)과 응답이 동일 구조.

| 필드 | 타입 | PUT 필수 | 값 · 규칙 |
|---|---|---|---|
| `captureMode` | string | **yes** | `SELF` (혼자 촬영) \| `ASSISTED` (다른 사람이 촬영) — 대소문자 무시 수용, 위반 400 |
| `shutterMode` | string | **yes** | `COUNTDOWN` (타이머) \| `MOTION` (동작인식 자동촬영) |
| `countdownSeconds` | integer | **yes** | **1~10** — 범위 밖 400. 기본 5 |
| `rememberSettings` | boolean | **yes** | `false` = 측정마다 선택 UI 노출 (값 자체는 저장되어 픽커 기본값으로 사용) |

GET 은 저장 행이 없어도 기본값 `{ SELF, COUNTDOWN, 5, true }` 로 200.

---

## 공통 에러 코드 (이 API 들에서 만나는 것)

| 코드 | HTTP | 상황 |
|---|---|---|
| `CMN-400-001` | 400 | 필드 형식 위반 · 이름 정규화 후 빈 값 · enum/범위 위반 |
| `CMN-400-002` | 400 | NOT NULL 필드 null clear 시도 |
| `CMN-409-001` | 409 | username(이메일)·전화번호 중복 |
| `AUTH-403-001` | 403 | 타인 리소스 접근 |
| `CMN-404-*` | 404 | 미존재 · 삭제 대기 계정 |
