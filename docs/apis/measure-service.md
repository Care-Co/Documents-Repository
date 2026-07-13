# measure-service

> 엔드포인트별 **Header · Request · Response** 정의 — 버전별 전량 전개, 소스는 실제 컨트롤러/DTO. 표기 — **굵은 필드 = 필수**, 규칙은 키워드만, 긴 설명은 각주(¹²³⁴⁵⁶).

| 항목 | 값 |
|---|---|
| Source | `/Users/jonghak/GitHub/Care&Co/measure-service` |
| Updated | 2026-07-13 |
| Server | `https://api.example.com` |
| Base path | 모든 엔드포인트 `/api/v2/users/{userId}/...` 하위 |

---

## 공통 규칙

- **버전** — 모든 엔드포인트 versioned (unversioned 없음). 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning), **정확 일치만** (미등록 버전 → `400 Invalid API version`).
- **인증** — 전 엔드포인트 `Authorization: Bearer <jwt>` (gateway-enforced). 메서드 시큐리티 애너테이션(`@PreAuthorize`/`@Secured`) 없음 — 인증 컨텍스트는 컨트롤러 내부에서 `AuthService.userGetData(userId)` 로 imperative 하게 해석·검증한다.
- **바디** — 요청 바디가 있으면 `Content-Type: application/json` 또는 `multipart/form-data` (record 생성).
- **응답 틀** — 모든 응답은 `Envelope` (아래 접기 참조) — 개별 섹션 샘플에서 반복하지 않음.
- **DTO calendar 이력** — `2026-01-13` (record DTO, V9 부터 `deviceId` / `deviceSerial` / `source` / `appVersion` 포함), `2026-02-02` (activity DTO).

**복수 버전 엔드포인트** — 나머지는 `1.0.0` 단일, 아래 4개만 복수. 버전 간 차이는 각 섹션 상단의 버전 표 참조.

| 그룹 | 엔드포인트 | 최신 |
|---|---|---|
| records (측정 컨텍스트) | [`POST /records`](#2-post-apiv2usersuseridrecords) · [`GET /records`](#3-get-apiv2usersuseridrecords) · [`GET /records/{recordId}`](#4-get-apiv2usersuseridrecordsrecordid) | `1.0.1` |
| activities (응답 wrapping) | [`DELETE /activities`](#8-delete-apiv2usersuseridactivities) | `1.0.1` |

<details>
<summary><b>응답 envelope</b> — 성공/에러 JSON · 필드 정의</summary>

**성공** (`Envelope`) — 모든 응답 샘플의 바깥 틀. 아래 필드는 모든 엔드포인트 공통이라 개별 섹션에서 반복하지 않는다.

```json
{
  "success": true,
  "data": {},
  "timestamp": "2026-04-27T08:00:00Z"
}
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `success` | boolean | yes | — |
| `data` | object | no | endpoint-specific |
| `timestamp` | string (date-time, UTC) | yes | — |

**에러** (`ErrorResponse`)

```json
{
  "success": false,
  "code": "CMN-400-001",
  "message": "Invalid request parameter. Field: captureMode, Value: FOO",
  "timestamp": "2026-04-27T08:00:00Z"
}
```

| 필드 | 타입 | 필수 |
|---|---|---|
| `success` | boolean | yes |
| `code` | string | yes |
| `message` | string | yes |
| `timestamp` | string (date-time, UTC) | yes |

</details>

---

## 엔드포인트

| Method | Path (`/api/v2/users/{userId}` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| GET | [`/capture-settings`](#1-get--put-apiv2usersuseridcapture-settings) | 1.0.0 | Bearer | 측정 촬영 설정 조회 — 저장 행이 없어도 기본값으로 200 |
| PUT | [`/capture-settings`](#1-get--put-apiv2usersuseridcapture-settings) | 1.0.0 | Bearer | 측정 촬영 설정 전체 upsert (기기 간 동기화) |
| POST | [`/records`](#2-post-apiv2usersuseridrecords) | **1.0.0, 1.0.1** | Bearer | 측정 record 생성 (vision/footprint) |
| GET | [`/records`](#3-get-apiv2usersuseridrecords) | **1.0.0, 1.0.1** | Bearer | record 페이지 검색 |
| GET | [`/records/{recordId}`](#4-get-apiv2usersuseridrecordsrecordid) | **1.0.0, 1.0.1** | Bearer | 단건 record 조회 |
| DELETE | [`/records`](#5-delete-apiv2usersuseridrecords) | 1.0.0 | Bearer | ids · 시간범위로 record bulk 삭제 |
| POST | [`/activities`](#6-post-apiv2usersuseridactivities) | 1.0.0 | Bearer | 활동 로그 생성 |
| GET | [`/activities`](#7-get-apiv2usersuseridactivities) | 1.0.0 | Bearer | 활동 페이지 검색 |
| DELETE | [`/activities`](#8-delete-apiv2usersuseridactivities) | **1.0.0, 1.0.1** | Bearer | ids · 시간범위로 활동 bulk 삭제 |

---

## 1. `GET / PUT` /api/v2/users/{userId}/capture-settings

측정 촬영 설정 (기기 간 동기화). GET 은 저장 행이 없어도 **404 가 아니라 기본값으로 200**. PUT 은 4필드 전부 필수인 전체 upsert (사용자당 1행) — `rememberSettings=false` 여도 나머지 값은 저장돼 다음 선택 픽커의 기본값으로 쓰인다 ⁴.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` (PUT) | PUT만 |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

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

</details>

### Request 필드 정의 (공통 — PUT `CaptureSettingsUpdateRequest`)

응답(`CaptureSettingsResponse`)도 동일 4필드를 그대로 반환.

| 필드 | 값 | 기본값 | Validation |
|---|---|---|---|
| **`captureMode`** | `SELF` (혼자 촬영) · `ASSISTED` (타인 촬영) | `SELF` | `@NotNull @ValidEnum(CaptureMode)` |
| **`shutterMode`** | `COUNTDOWN` (타이머) · `MOTION` (동작인식 자동촬영) | `COUNTDOWN` | `@NotNull @ValidEnum(ShutterMode)` |
| **`countdownSeconds`** | 1~10 (`Short`) | `5` | `@NotNull @Min(1) @Max(10)` |
| **`rememberSettings`** | boolean ⁴ | `true` | `@NotNull` |

> 게이트웨이 — `/api/{v}/users/*/capture-settings` 는 measure-service 라우팅 (`GatewayConstants.Measure.CAPTURE_SETTINGS`). `/users/**` 하위 신규 리소스는 라우트 추가 없으면 user-service 로 흘러간다.

---

## 2. `POST` /api/v2/users/{userId}/records

측정 record 생성 (vision/footprint). 요청 바디는 `multipart/form-data`. `api-version` 헤더로 요청·응답 DTO 가 분기된다 — `1.0.0` 은 B2C 모바일 앱 자가 측정 코호트(서버에서 `source=SELF` 강제 매핑), `1.0.1` 은 측정 컨텍스트 추적(`macAddress` / `source` / `appVersion` / `firmwareVersion`) 을 추가.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | B2C 모바일 앱 자가 측정 코호트 — 서버에서 `source=SELF` 강제 매핑, 측정 컨텍스트 필드 미노출 |
| `1.0.1` **(최신)** | 측정 컨텍스트 추적 — 요청 `macAddress`/`source`/`appVersion`/`firmwareVersion` 추가, 응답 `deviceId`/`deviceSerial`/`deviceType`/`source`/`appVersion` 노출 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `multipart/form-data` | yes |

또한 쿼리 파라미터 `recordType` 필수 — `VISION` \| `FOOTPRINT` (`@ValidEnum(RecordType)`). path 의 `userId` 는 `@ValidUuid`.

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`multipart/form-data` parts (`CreateRecordRequestV1_0_0` — validation 애너테이션 없음).

| Part | 타입 | 필수 |
|---|---|---|
| **`measuredDataTime`** | date-time (UTC) | yes |
| `rawData` | string | no |
| `front` | binary (이미지) | no |
| `side` | binary (이미지) | no |

```bash
curl -X POST "https://api.example.com/api/v2/users/<uuid>/records?recordType=VISION" \
  -H "api-version: 1.0.0" -H "Authorization: Bearer ..." \
  -F "measuredDataTime=2026-04-27T01:23:45Z" \
  -F "front=@front.jpg" -F "side=@side.jpg"
```

### `1.0.0` — Response

**201 Created** — 측정 컨텍스트 필드(`deviceId`/`deviceSerial`/`deviceType`/`source`/`appVersion`) 미노출. 서버 내부엔 `source=SELF` 로 저장되지만 응답 DTO 에서 제거

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "userId": "uuid",
    "measuredDateTime": "2026-04-27T01:23:45Z",
    "timezone": "Asia/Seoul",
    "footprint": { "weight": 70.5, "height": 175.0, "age": 34 },
    "front": { "type": "FRONT", "imageUrl": "..." },
    "side": { "type": "SIDE", "imageUrl": "..." },
    "score": { "bodyScore": 80.0, "predictedAge": 30.0 }
  }
}
```

<details>
<summary><b>400 Bad Request</b> — recordType 누락·잘못된 값 ¹ · userId 형식 위반</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Invalid request parameter. Field: userId" }
```

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request (= 1.0.0 parts + `macAddress` · `source` · `appVersion` · `firmwareVersion`)

`multipart/form-data` parts (`CreateRecordRequestV1_0_1`).

| Part | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`measuredDataTime`** | date-time (UTC) | yes | |
| `rawData` | string | no | |
| `front` | binary | no | |
| `side` | binary | no | |
| `macAddress` | string | no | `@ValidMacAddress` ² (null/blank 통과) |
| `source` | string | no | enum ³ — 미전송 시 `UNSPECIFIED` |
| `appVersion` | string | no | 예: `ios-2.3.1` (회귀 추적용) |
| `firmwareVersion` | string | no | device 보고 펌웨어. null/blank 면 device-service 가 기존 값 유지 |

```bash
curl -X POST "https://api.example.com/api/v2/users/<uuid>/records?recordType=FOOTPRINT" \
  -H "api-version: 1.0.1" -H "Authorization: Bearer ..." \
  -F "measuredDataTime=2026-05-29T01:23:45Z" \
  -F "macAddress=AA:BB:CC:DD:EE:FF" \
  -F "source=TRAINER_GUIDED" \
  -F "appVersion=ios-2.3.1"
```

### `1.0.1` — Response

**201 Created** — 1.0.0 shape + `deviceId` / `deviceSerial` / `deviceType` / `source` / `appVersion` ⁵

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "userId": "uuid",
    "measuredDateTime": "2026-05-29T01:23:45Z",
    "timezone": "Asia/Seoul",
    "footprint": { "weight": 70.5, "height": 175.0, "age": 34 },
    "front": { "type": "FRONT", "imageUrl": "..." },
    "side": { "type": "SIDE", "imageUrl": "..." },
    "score": { "bodyScore": 80.0, "predictedAge": 30.0 },
    "deviceId": "dev-uuid-from-device-service",
    "deviceSerial": "HW-SN-001234",
    "deviceType": "...",
    "source": "TRAINER_GUIDED",
    "appVersion": "ios-2.3.1"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — recordType ¹ · source enum ³ · macAddress 형식 ² · device lookup 실패</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Invalid request parameter. Field: macAddress" }
```

</details>

<details>
<summary><b>4xx / 5xx</b> — device-service <code>DeviceLookup.LookupByMac</code> propagate (owner mismatch · device inactive 등)</summary>

`macAddress` 전송 시 controller 가 device-service `DeviceLookup.LookupByMac` gRPC 를 호출한다. lookup 실패 시 device-service 의 `ErrorCode` 가 `CncResponse` 로 그대로 propagate 된다. owner / status 검증·풀 자동 추적은 device-service 가 처리한다.

</details>

</details>

### Response 필드 정의 — `RecordResponseV1_0_0` / `RecordResponseV1_0_1`

`1.0.0` 은 아래 상단 8개 필드, `1.0.1` 은 + `deviceId`/`deviceSerial`/`deviceType`/`source`/`appVersion` ⁵. nested 타입(`Footprint`/`Vision`/`Score`)은 버전별로 독립 정의되지만 필드 동일.

| 필드 | 타입 | 버전 | 설명 |
|---|---|---|---|
| `id` | string (uuid) | 전 버전 | — |
| `userId` | string (uuid) | 전 버전 | — |
| `measuredDateTime` | string (date-time, UTC) | 전 버전 | — |
| `timezone` | string (IANA zone id) | 전 버전 | 측정 시점 사용자 zone snapshot |
| `footprint` | `Footprint` | 전 버전 | 아래 |
| `front` `side` | `Vision` | 전 버전 | 아래 |
| `score` | `Score` — `{ bodyScore: number, predictedAge: number }` | 전 버전 | — |
| `deviceId` | string (uuid) \| null | **1.0.1만** | device-service lookup. `allowed=true`+empty(미클레임/TMP)·macAddress 미전송 시 null |
| `deviceSerial` | string \| null | **1.0.1만** | 응답의 `hardwareSerial` |
| `deviceType` | string \| null | **1.0.1만** | device 타입 |
| `source` | enum `RecordSource` | **1.0.1만** | enum 이름 직렬화. 미전송 시 `UNSPECIFIED` |
| `appVersion` | string \| null | **1.0.1만** | 전송된 앱 버전 (`firmwareVersion` 은 응답 미노출) |

**`Footprint`**

| Field | Type |
|---|---|
| `id` | string |
| `firstClass` / `secondaryClass` / `thirdClass` | `{ type: number, accuracy: number }` |
| `leftFootDimension` / `rightFootDimension` | `{ length: number, width: number }` |
| `imageUrl` | string (uri) |
| `centerOfGravity` | `{ x: number, y: number }` |
| `leftPressure` / `rightPressure` | `{ top: number, middle: number, bottom: number, total: number }` |
| `weight` / `height` / `age` | number / number / integer |

**`Vision`** (`front` / `side`)

| Field | Type |
|---|---|
| `id` | string |
| `type` | string (enum `VisionType`) |
| `angleFace` / `anglePelvis` / `angleShoulder` | number |
| `classType` / `accuracy` | number / number |
| `imageUrl` | string (uri) |
| `keypoints` | object — 17 named keypoints, each `{ x: number, y: number, accuracy: number }` |

**`Score`** — `bodyScore`(number), `predictedAge`(number).

**`RecordSource`** (`source` enum, `EnumType.STRING` 저장)

| Value | Meaning |
|---|---|
| `UNSPECIFIED` | v1.0.1 호출자가 의도적으로 미전송. "정말 모름". NULL 회피용 명시 분류 |
| `SELF` | B2C 사용자가 본인 device 로 자가 측정 (B2B 시설 무관). v1.0.0 호출은 모두 이 case |
| `TRAINER_GUIDED` | B2B 시설의 device 로 측정 (트레이너 입회 가능) |
| `IMPORT` | 데이터 마이그레이션 / 일괄 인입 (수동 또는 batch) |

### Request 필드 정의 (공통)

| 필드 | 버전 | 비고 |
|---|---|---|
| `measuredDataTime` `rawData` `front` `side` | 전 버전 | 4개 part 공통 (validation 애너테이션 없음) |
| `macAddress` | **1.0.1만** | 전송 시 device-service lookup 수행 ² |
| `source` | **1.0.1만** | 미전송 → `UNSPECIFIED` ³ |
| `appVersion` `firmwareVersion` | **1.0.1만** | 추적용 · firmwareVersion 은 응답 미노출 |

**`recordType`** (쿼리 enum) — `VISION` \| `FOOTPRINT`.

---

## 3. `GET` /api/v2/users/{userId}/records

record 페이지 검색. `api-version` 헤더에 따라 응답 DTO 분기 (`1.0.0` → `RecordResponseV1_0_0`, `1.0.1` → `RecordResponseV1_0_1`). `1.0.1` 은 device 필터 쿼리 파라미터(`deviceId`/`deviceSerial`/`kind`) 도 추가.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 응답 `RecordResponseV1_0_0` — device 필드 미노출 |
| `1.0.1` **(최신)** | device 필터 쿼리(`deviceId`/`deviceSerial`/`kind`) 추가 + 응답 `RecordResponseV1_0_1` (device 필드 노출) |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 기본값 |
|---|---|---|
| `from` `to` | date-time (UTC) | — (범위 필터) |
| `ids` | array&lt;string&gt; | — |
| `size` | integer | `10` |
| `page` | integer | `0` |
| `sortBy` | string | `createdAt` |
| `direction` | `asc` \| `desc` | `desc` |

### `1.0.0` — Response

**200 OK** — `Page<RecordResponseV1_0_0>`. `deviceId` / `deviceSerial` / `deviceType` / `source` / `appVersion` 미노출

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid", "userId": "uuid",
        "measuredDateTime": "2026-05-29T01:23:45Z", "timezone": "Asia/Seoul",
        "footprint": { "weight": 70.5, "height": 175.0, "age": 34 },
        "front": { "type": "FRONT" }, "side": { "type": "SIDE" },
        "score": { "bodyScore": 80.0, "predictedAge": 30.0 }
      }
    ],
    "totalElements": 47, "totalPages": 5, "number": 0, "size": 10
  }
}
```

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request (= 1.0.0 쿼리 + `deviceId` · `deviceSerial` · `kind`)

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 기본값 |
|---|---|---|
| `from` `to` | date-time (UTC) | — |
| `ids` | array&lt;string&gt; | — |
| `deviceId` | string | — (device 필터, 1.0.1 신규) |
| `deviceSerial` | string | — (device 필터, 1.0.1 신규) |
| `kind` | `ALL` \| `FOOTPRINT_ONLY` \| `VISION` | `ALL` (1.0.1 신규) |
| `size` | integer | `10` |
| `page` | integer | `0` |
| `sortBy` | string | `createdAt` |
| `direction` | `asc` \| `desc` | `desc` |

### `1.0.1` — Response

**200 OK** — `Page<RecordResponseV1_0_1>`. 1.0.0 content + `deviceId` / `deviceSerial` / `deviceType` / `source` / `appVersion` ⁵

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid", "userId": "uuid",
        "measuredDateTime": "2026-05-29T01:23:45Z", "timezone": "Asia/Seoul",
        "footprint": { "weight": 70.5, "height": 175.0, "age": 34 },
        "front": { "type": "FRONT" }, "side": { "type": "SIDE" },
        "score": { "bodyScore": 80.0, "predictedAge": 30.0 },
        "deviceId": "dev-uuid", "deviceSerial": "HW-SN-001234", "deviceType": "...",
        "source": "SELF", "appVersion": "ios-2.3.1"
      }
    ],
    "totalElements": 47, "totalPages": 5, "number": 0, "size": 10
  }
}
```

<details>
<summary><b>401 Unauthorized</b> — 인증/유저 조회 실패</summary>

```json
{ "success": false, "code": "AUTH-401-001", "message": "..." }
```

`AuthService.userGetData` 가 인증/유저 조회에 실패하면 propagate 된다.

</details>

</details>

### Response 필드 정의 — `RecordResponseV1_0_0` / `RecordResponseV1_0_1`

`data` 는 Spring Data `Page<T>` (`content[]` + `totalElements` `totalPages` `number` `size` `first` `last` `sort`). `content[]` 항목.

| 필드 | 타입 | 버전 | 설명 |
|---|---|---|---|
| `id` | string (uuid) | 전 버전 | — |
| `userId` | string (uuid) | 전 버전 | — |
| `measuredDateTime` | string (date-time, UTC) | 전 버전 | — |
| `timezone` | string (IANA zone id) | 전 버전 | 측정 시점 사용자 zone snapshot |
| `footprint` | `Footprint` | 전 버전 | POST §2 참조 |
| `front` `side` | `Vision` | 전 버전 | POST §2 참조 |
| `score` | `Score` — `{ bodyScore, predictedAge : number }` | 전 버전 | — |
| `deviceId` | string (uuid) \| null | **1.0.1만** | device-service lookup. 미클레임/미전송 시 null |
| `deviceSerial` | string \| null | **1.0.1만** | 응답의 `hardwareSerial` |
| `deviceType` | string \| null | **1.0.1만** | device 타입 |
| `source` | enum `RecordSource` | **1.0.1만** | enum 이름 직렬화 (`UNSPECIFIED`/`SELF`/`TRAINER_GUIDED`/`IMPORT`) |
| `appVersion` | string \| null | **1.0.1만** | 전송된 앱 버전 |

---

## 4. `GET` /api/v2/users/{userId}/records/{recordId}

단건 record 조회. 미존재 시 404. `api-version` 헤더에 따라 응답 DTO 분기.

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 응답 `RecordResponseV1_0_0` — device 필드 미노출 |
| `1.0.1` **(최신)** | 응답 `RecordResponseV1_0_1` — `deviceId`/`deviceSerial`/`deviceType`/`source`/`appVersion` 추가 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid) · `recordId` (uuid) 만.

### `1.0.0` — Response

**200 OK** — `RecordResponseV1_0_0` (측정 컨텍스트 필드 미노출, POST §2 의 1.0.0 응답 shape 와 동일)

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "userId": "uuid",
    "measuredDateTime": "2026-04-27T01:23:45Z",
    "timezone": "Asia/Seoul",
    "footprint": { "weight": 70.5, "height": 175.0, "age": 34 },
    "front": { "type": "FRONT", "imageUrl": "..." },
    "side": { "type": "SIDE", "imageUrl": "..." },
    "score": { "bodyScore": 80.0, "predictedAge": 30.0 }
  }
}
```

<details>
<summary><b>404 Not Found</b> — record 미존재</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Record not found" }
```

`ResponseStatusException(NOT_FOUND, "Record not found")`.

</details>

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

바디 없음 — path 의 `userId` (uuid) · `recordId` (uuid) 만.

### `1.0.1` — Response

**200 OK** — `RecordResponseV1_0_1` (1.0.0 shape + `deviceId` / `deviceSerial` / `deviceType` / `source` / `appVersion` ⁵)

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "userId": "uuid",
    "measuredDateTime": "2026-05-29T01:23:45Z",
    "timezone": "Asia/Seoul",
    "footprint": { "weight": 70.5, "height": 175.0, "age": 34 },
    "front": { "type": "FRONT", "imageUrl": "..." },
    "side": { "type": "SIDE", "imageUrl": "..." },
    "score": { "bodyScore": 80.0, "predictedAge": 30.0 },
    "deviceId": "dev-uuid", "deviceSerial": "HW-SN-001234", "deviceType": "...",
    "source": "TRAINER_GUIDED", "appVersion": "ios-2.3.1"
  }
}
```

<details>
<summary><b>404 Not Found</b> — record 미존재</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Record not found" }
```

</details>

</details>

### Response 필드 정의 — `RecordResponseV1_0_0` / `RecordResponseV1_0_1`

`1.0.0` 은 상단 8개 필드, `1.0.1` 은 + 5개 (`deviceId`/`deviceSerial`/`deviceType`/`source`/`appVersion`) ⁵. nested 타입 상세는 POST §2 참조.

| 필드 | 타입 | 버전 | 설명 |
|---|---|---|---|
| `id` | string (uuid) | 전 버전 | — |
| `userId` | string (uuid) | 전 버전 | — |
| `measuredDateTime` | string (date-time, UTC) | 전 버전 | — |
| `timezone` | string (IANA zone id) | 전 버전 | 측정 시점 사용자 zone snapshot |
| `footprint` | `Footprint` | 전 버전 | POST §2 참조 |
| `front` `side` | `Vision` | 전 버전 | POST §2 참조 |
| `score` | `Score` — `{ bodyScore, predictedAge : number }` | 전 버전 | — |
| `deviceId` | string (uuid) \| null | **1.0.1만** | device-service lookup. 미클레임/미전송 시 null |
| `deviceSerial` | string \| null | **1.0.1만** | 응답의 `hardwareSerial` |
| `deviceType` | string \| null | **1.0.1만** | device 타입 |
| `source` | enum `RecordSource` | **1.0.1만** | enum 이름 직렬화 (`UNSPECIFIED`/`SELF`/`TRAINER_GUIDED`/`IMPORT`) |
| `appVersion` | string \| null | **1.0.1만** | 전송된 앱 버전 |

---

## 5. `DELETE` /api/v2/users/{userId}/records

ids · 시간범위로 bulk 삭제.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "ids": ["..."], "from": "2026-04-01T00:00:00Z", "to": "2026-04-27T00:00:00Z" }
```

세 필드 모두 선택 (`Optional`). ids · from · to 조합으로 대상 지정.

### `1.0.0` — Response

**200 OK** — 삭제 집계 (`RecordDeleteResult`)

```json
{
  "success": true,
  "data": {
    "recordCount": 3,
    "footprintCount": 2,
    "frontVisionCount": 2,
    "sideVisionCount": 2,
    "deletedRecordIds": ["..."]
  }
}
```

</details>

### Request 필드 정의 (공통 — `RecordDeleteRequestV1_0_0`)

| 필드 | 타입 | 필수 |
|---|---|---|
| `ids` | array&lt;string&gt; (`Optional`) | no |
| `from` | date-time (UTC) (`Optional`) | no |
| `to` | date-time (UTC) (`Optional`) | no |

### Response 필드 정의 — `RecordDeleteResult`

| 필드 | 타입 |
|---|---|
| `recordCount` | integer (long) |
| `footprintCount` | integer (long) |
| `frontVisionCount` | integer (long) |
| `sideVisionCount` | integer (long) |
| `deletedRecordIds` | array&lt;string&gt; |

---

## 6. `POST` /api/v2/users/{userId}/activities

활동 로그 생성.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{
  "type": 2,
  "startTime": "2026-04-27T01:00:00Z",
  "endTime": "2026-04-27T02:00:00Z",
  "painDecreased": true,
  "samePain": false,
  "mildPain": false,
  "moderatePain": false,
  "severePain": false
}
```

### `1.0.0` — Response

**201 Created** — `type` 은 응답에서 enum 이름, pain 은 `paindata` 로 중첩

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "userId": "uuid",
    "type": "PERSONAL_EXERCISE",
    "startTime": "2026-04-27T01:00:00Z",
    "endTime": "2026-04-27T02:00:00Z",
    "paindata": {
      "painDecreased": true, "samePain": false,
      "mildPain": false, "moderatePain": false, "severePain": false
    }
  }
}
```

<details>
<summary><b>400 Bad Request</b> — type null · 0~4 외 정수 · 알 수 없는 값</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "..." }
```

`ActivityType.parse(int)` 가 `IllegalArgumentException` → 400.

</details>

</details>

### Request 필드 정의 (공통 — `CreateActivityRequestV1_0_0`)

| 필드 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`type`** | integer | yes | `0`=WORK_THERAPY `1`=ORIENTAL_TREATMENT `2`=PERSONAL_EXERCISE `3`=PROFESSIONAL_GUIDANCE `4`=MASSAGE_AND_PAIN_MANAGEMENT |
| **`startTime`** | date-time (UTC) | yes | |
| **`endTime`** | date-time (UTC) | yes | |
| **`painDecreased`** `samePain` `mildPain` `moderatePain` `severePain` | boolean | yes | 5개 pain flag (요청은 primitive `boolean`) |

### Response 필드 정의 — `Activity` (`ActivityResponseV1_0_0`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `userId` | string (uuid) | — |
| `type` | string (enum `ActivityType`) | 응답은 enum 이름 (`WORK_THERAPY` … `MASSAGE_AND_PAIN_MANAGEMENT`) |
| `startTime` | string (date-time, UTC) | — |
| `endTime` | string (date-time, UTC) | — |
| `paindata` | `{ painDecreased, samePain, mildPain, moderatePain, severePain : boolean }` | 필드명 lowercase `paindata` · 응답은 wrapper `Boolean` |

---

## 7. `GET` /api/v2/users/{userId}/activities

활동 페이지 검색. `from`/`to` 는 사용자 `ZoneId` 기준으로 파싱되며 `to` 는 그 zone 의 end-of-day 로 확장.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Authorization` | `Bearer <jwt>` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 기본값 |
|---|---|---|
| `from` | string (user-zone date/time) | — |
| `to` | string (user-zone date/time → EOD) | — |
| `size` | integer | `10` |
| `page` | integer | `0` |
| `sortBy` | `startTime` \| `endTime` | `startTime` ⁶ |
| `direction` | `asc` \| `desc` | `desc` |

### `1.0.0` — Response

**200 OK** — `Page<Activity>`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid", "userId": "uuid",
        "type": "PERSONAL_EXERCISE",
        "startTime": "2026-04-27T01:00:00Z", "endTime": "2026-04-27T02:00:00Z",
        "paindata": {
          "painDecreased": true, "samePain": false,
          "mildPain": false, "moderatePain": false, "severePain": false
        }
      }
    ],
    "totalElements": 12, "totalPages": 2, "number": 0, "size": 10
  }
}
```

<details>
<summary><b>401 Unauthorized</b> — 인증/유저 조회 실패</summary>

```json
{ "success": false, "code": "AUTH-401-001", "message": "..." }
```

</details>

</details>

### Response 필드 정의 — `Activity` (`ActivityResponseV1_0_0`)

`data` 는 Spring Data `Page<T>` (`content[]` + `totalElements` `totalPages` `number` `size` …). `content[]` 항목.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `userId` | string (uuid) | — |
| `type` | string (enum `ActivityType`) | 응답은 enum 이름 |
| `startTime` | string (date-time, UTC) | — |
| `endTime` | string (date-time, UTC) | — |
| `paindata` | `{ painDecreased, samePain, mildPain, moderatePain, severePain : boolean }` | 필드명 lowercase `paindata` · wrapper `Boolean` |

---

## 8. `DELETE` /api/v2/users/{userId}/activities

ids · 시간범위로 bulk 삭제. `from`/`to` 는 UTC `Instant`. 요청 shape 는 두 버전 동일 — **응답 wrapping 만 다르다** (`1.0.0` 은 중첩 `{ result, total }`, `1.0.1` 은 flat `{ activityCount }`).

### 버전

| 버전 | 차이 |
|---|---|
| `1.0.0` | 응답 중첩 `{ result, total }` (`ActivityDeleteResponse`) |
| `1.0.1` **(최신)** | 응답 flat `{ activityCount }` (`ActivityDeleteResult`) — 요청 shape 는 두 버전 동일 |

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |
| `Authorization` | `Bearer <jwt>` | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

```json
{ "ids": ["..."], "from": "2026-04-01T00:00:00Z", "to": "2026-04-27T00:00:00Z" }
```

### `1.0.0` — Response

**200 OK** — 중첩 `ActivityDeleteResponse` (`result` + `total`)

```json
{
  "success": true,
  "data": {
    "result": { "activityCount": 3 },
    "total": 3
  }
}
```

</details>

<details>
<summary><b><code>1.0.1</code> — Request · Response</b></summary>

### `1.0.1` — Request

```json
{ "ids": ["..."], "from": "2026-04-01T00:00:00Z", "to": "2026-04-27T00:00:00Z" }
```

(요청 shape 는 1.0.0 과 동일 — `ActivityDeleteRequestV1_0_0`.)

### `1.0.1` — Response

**200 OK** — flat `ActivityDeleteResult` (`activityCount` 만)

```json
{
  "success": true,
  "data": { "activityCount": 3 }
}
```

<details>
<summary><b>401 Unauthorized</b> — 인증/유저 조회 실패</summary>

```json
{ "success": false, "code": "AUTH-401-001", "message": "..." }
```

</details>

</details>

### Request 필드 정의 (공통 — `ActivityDeleteRequestV1_0_0`, 1.0.0·1.0.1 공용)

| 필드 | 타입 | 필수 |
|---|---|---|
| `ids` | array&lt;string&gt; (`Optional`) | no |
| `from` | date-time (UTC) (`Optional`) | no |
| `to` | date-time (UTC) (`Optional`) | no |

컴팩트 생성자가 null → `Optional.empty()` 로 정규화.

### Response 필드 정의 — 버전별

**`ActivityDeleteResponse`** (1.0.0)

| 필드 | 타입 |
|---|---|
| `result` | `ActivityDeleteResult` (`{ activityCount: long }`) |
| `total` | integer (long) |

**`ActivityDeleteResult`** (1.0.1)

| 필드 | 타입 |
|---|---|
| `activityCount` | integer (long) |

---

## 각주

> ¹ `recordType` 누락 또는 `VISION`·`FOOTPRINT` 외 값 → `ResponseStatusException` 400, message `recordType must be vision or footprint`.
> ² `macAddress` 는 `@ValidMacAddress` — null/blank 는 통과. 형식 오류는 controller 진입 단계에서 `INVALID_MAC` (전역 핸들러 `CMN-400-002`) 으로 차단. 전송 시 device-service `DeviceLookup.LookupByMac` gRPC 호출 → 응답의 `deviceId` / `hardwareSerial` 을 record 에 함께 저장.
> ³ `source` 는 free-form string 을 controller 가 `RecordSource` 로 파싱. 유효 값은 `UNSPECIFIED` / `SELF` / `TRAINER_GUIDED` / `IMPORT`. 그 외는 `ResponseStatusException` 400 (message `source must be one of UNSPECIFIED / SELF / TRAINER_GUIDED / IMPORT`). blank/null → `UNSPECIFIED`.
> ⁴ `rememberSettings=false` = 측정마다 선택 UI 노출. 값 자체는 저장돼 다음 픽커의 기본값으로 쓰인다.
> ⁵ `deviceId` — device-service lookup 응답. `allowed=true` + empty `deviceId`(미클레임/TMP) 는 null 로 정규화. macAddress 미전송 시 null. `deviceSerial` — 응답의 `hardwareSerial`. `deviceType` — device 타입. `source` 는 `RecordSource` enum 이름으로 직렬화. `firmwareVersion` 은 요청에만 있고 응답 DTO 에는 없다.
> ⁶ `sortBy` 가 `startTime`·`endTime` 외 값이면 silently `startTime` 으로 폴백 (400 아님).

## 에러 코드

| 코드 | HTTP | 상황 |
|---|---|---|
| `CMN-400-001` | 400 | validation 위반 (capture-settings enum·범위, activity type) |
| `CMN-400-002` | 400 | `@ValidUuid`(userId) · `@ValidMacAddress`(macAddress) 형식 위반 (글로벌 핸들러) |
| — | 400 | `recordType` ¹ · `source` ³ 잘못된 값 (`ResponseStatusException`) |
| — | 404 | record 미존재 (`ResponseStatusException`, `Record not found`) |
| `AUTH-401-*` | 401 | `AuthService.userGetData` 인증/유저 조회 실패 propagate |
| device-service `ErrorCode` | 4xx/5xx | `macAddress` 전송 시 `DeviceLookup.LookupByMac` propagate (owner mismatch · device inactive 등) |
