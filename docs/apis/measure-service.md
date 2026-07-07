# measure-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/measure-service`
Updated: 2026-07-07 (Errors 표 / api-version metadata 보강 audit)

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` (records · activities) / `1.0.1` (records POST + GET) |
| `Authorization` | `Bearer <jwt>` (gateway-enforced) |
| `Content-Type` | `application/json` or `multipart/form-data` |

**Security** &nbsp;Auth context resolved internally via `AuthService.userGetData(userId)`. No `@PreAuthorize` annotations.

DTO calendar versions: `2026-01-13` (record DTO — V9 부터 `deviceId` / `deviceSerial` / `source` / `appVersion` 포함), `2026-02-02` (activity DTO).

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning). 아래 "제공 버전" 중 하나를 보낸다. `—` 는 unversioned.

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| DELETE | /api/v2/users/{userId}/activities | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId}/capture-settings | 1.0.0 | 1.0.0 |
| PUT | /api/v2/users/{userId}/capture-settings | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId}/activities | 1.0.0 | 1.0.0 |
| POST | /api/v2/users/{userId}/activities | 1.0.0 | 1.0.0 |
| DELETE | /api/v2/users/{userId}/records | 1.0.0 | 1.0.0 |
| GET | /api/v2/users/{userId}/records | 1.0.0, 1.0.1 | 1.0.1 |
| POST | /api/v2/users/{userId}/records | 1.0.0, 1.0.1 | 1.0.1 |
| GET | /api/v2/users/{userId}/records/{recordId} | 1.0.0, 1.0.1 | 1.0.1 |

---

## `GET` /api/v2/users/{userId}/capture-settings

**Operation ID** &nbsp;`getCaptureSettings`  &nbsp;**Tags** &nbsp;`capture-settings`

측정 촬영 설정 조회 — 기기 간 동기화용. 저장된 행이 없으면 404 가 아니라 **기본값으로 200** 을 반환한다 (앱 분기 제거).

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `userId` | string (uuid) | yes |
| header | `api-version` | `1.0.0` | yes |

### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` with `data: CaptureSettingsResponse` |

#### 200 — example (기본값)

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

---

## `PUT` /api/v2/users/{userId}/capture-settings

**Operation ID** &nbsp;`upsertCaptureSettings`  &nbsp;**Tags** &nbsp;`capture-settings`

전체 upsert (사용자당 1행). `rememberSettings=false` 여도 나머지 값은 저장된다 — 다음 선택 픽커의 기본값으로 쓰인다.

### Request body

| Field | Type | Required | Validation |
|---|---|---|---|
| `captureMode` | string | yes | `SELF`(혼자 촬영) / `ASSISTED`(다른 사람이 촬영) |
| `shutterMode` | string | yes | `COUNTDOWN`(타이머) / `MOTION`(동작인식 자동촬영) |
| `countdownSeconds` | integer | yes | 1~10 |
| `rememberSettings` | boolean | yes | false = 측정마다 선택 UI 노출 |

### Responses

| Status | Schema |
|---|---|
| **200** | `CncResponse` with `data: CaptureSettingsResponse` (저장된 값) |
| **400** | enum·범위 위반 (`CMN-400-*`) |

#### 200 — example

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

#### 400 — example (enum 위반)

```json
{
  "success": false,
  "code": "CMN-400-001",
  "message": "Invalid request parameter. Field: captureMode, Value: FOO"
}
```

```bash
curl -X PUT https://api.example.com/api/v2/users/{userId}/capture-settings \
  -H "api-version: 1.0.0" -H "Authorization: Bearer <jwt>" -H "Content-Type: application/json" \
  -d '{"captureMode":"ASSISTED","shutterMode":"MOTION","countdownSeconds":3,"rememberSettings":false}'
```

> 게이트웨이 라우팅 — `/api/{v}/users/*/capture-settings` 는 measure-service 로 라우팅된다 (GatewayConstants.Measure.CAPTURE_SETTINGS). `/users/**` 하위 신규 리소스는 라우트 추가가 없으면 user-service 로 흘러간다.

---

## `POST` /api/v2/users/{userId}/records — `api-version: 1.0.0`

**Operation ID** &nbsp;`createRecord`
**Tags** &nbsp;`record`

Create record (vision/footprint). v1.0.0 는 B2C 모바일 앱 자가 측정 코호트 — 서버에서 `source` 를 `SELF` 로 강제 매핑.

### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | — |
| path | `userId` | string (uuid) | yes | `@ValidUuid` |
| query | `recordType` | string (enum: `VISION` `FOOTPRINT`) | yes | `@ValidEnum(RecordType)` |

### Request body

`multipart/form-data` &nbsp;**Required**

**Schema** — [`CreateRecordRequestV1_0_0`](#createrecordrequestv1_0_0)

| Part | Type | Required |
|---|---|---|
| `measuredDataTime` | string (date-time, UTC) | yes |
| `rawData` | string | no |
| `front` | binary | no |
| `side` | binary | no |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | `Envelope` with `data:` [`RecordV1_0_0`](#recordv1_0_0) |
| **400** | `application/json` | [`ErrorResponse`](#errorresponse) |

v1.0.0 응답은 측정 컨텍스트 필드 (`deviceId` / `deviceSerial` / `source` / `appVersion`) 를 노출하지 않는다. 서버 내부에는 `source=SELF` 로 저장되지만 응답 DTO 에서 제거.

#### 201 — example

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

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 400 | — | `recordType` 누락 / `VISION`·`FOOTPRINT` 외 값 (`ResponseStatusException`, message: `recordType must be vision or footprint`) |
| 400 | `CMN-400-002` | `userId` `@ValidUuid` 위반 (글로벌 핸들러) |
| 401 | `AUTH-401-*` | `AuthService.userGetData` 가 인증/유저 조회 실패 시 propagate |

```bash
curl -X POST "https://api.example.com/api/v2/users/<uuid>/records?recordType=VISION" \
  -H "api-version: 1.0.0" -H "Authorization: Bearer ..." \
  -F "measuredDataTime=2026-04-27T01:23:45Z" \
  -F "front=@front.jpg" -F "side=@side.jpg"
```

---

## `POST` /api/v2/users/{userId}/records — `api-version: 1.0.1`

**Operation ID** &nbsp;`createRecordV1_0_1`
**Tags** &nbsp;`record`

v1.0.0 + 측정 컨텍스트 추적 (`macAddress` / `source` / `appVersion`). 같은 path 에 `api-version` 헤더로 분기.

`macAddress` 전송 시 controller 가 device-service `DeviceLookup.LookupByMac` gRPC 호출 → 응답의 `deviceId` / `hardwareSerial` 을 record 에 함께 저장. owner / status 검증·풀 자동 추적은 device-service 가 처리. lookup 실패 시 device-service 의 `ErrorCode` 가 `CncResponse` 로 propagate.

`source` 미전송 시 [`RecordSource.UNSPECIFIED`](#recordsource) (의도적 "모름"). 유효하지 않은 값은 400.

### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.1`) | yes | — |
| path | `userId` | string (uuid) | yes | `@ValidUuid` |
| query | `recordType` | string (enum: `VISION` `FOOTPRINT`) | yes | `@ValidEnum(RecordType)` |

### Request body

`multipart/form-data` &nbsp;**Required**

**Schema** — [`CreateRecordRequestV1_0_1`](#createrecordrequestv1_0_1)

| Part | Type | Required | Validation | Description |
|---|---|---|---|---|
| `measuredDataTime` | string (date-time, UTC) | yes | — | — |
| `rawData` | string | no | — | — |
| `front` | binary | no | — | — |
| `side` | binary | no | — | — |
| `macAddress` | string | no | `@ValidMacAddress` (null/blank 통과) | 측정 device 의 MAC. 전송 시 device-service lookup 수행. 형식 오류는 controller 진입 단계에서 `INVALID_MAC` 으로 차단 |
| `source` | string (enum: `UNSPECIFIED` `SELF` `TRAINER_GUIDED` `IMPORT`) | no | — | 미전송 시 `UNSPECIFIED` |
| `appVersion` | string | no | — | 예: `ios-2.3.1` (회귀 추적용) |
| `firmwareVersion` | string | no | — | device 가 보고한 펌웨어 버전. null/blank 면 device-service 가 기존 값 유지 |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | `Envelope` with `data:` [`RecordV1_0_1`](#recordv1_0_1) |
| **400** | `application/json` | [`ErrorResponse`](#errorresponse) — `macAddress` 형식 오류 (`INVALID_MAC`) / `source` enum 위반 / device lookup 실패 (owner mismatch, device inactive 등) |

#### 201 — example

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
    "source": "TRAINER_GUIDED",
    "appVersion": "ios-2.3.1"
  }
}
```

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 400 | — | `recordType` 누락 / `VISION`·`FOOTPRINT` 외 값 (`ResponseStatusException`, message: `recordType must be vision or footprint`) |
| 400 | — | `source` 가 `UNSPECIFIED`·`SELF`·`TRAINER_GUIDED`·`IMPORT` 외 값 (`ResponseStatusException`, message: `source must be one of UNSPECIFIED / SELF / TRAINER_GUIDED / IMPORT`) |
| 400 | `CMN-400-002` | `userId` `@ValidUuid` 위반 / `macAddress` `@ValidMacAddress` 형식 위반 (글로벌 핸들러) |
| 401 | `AUTH-401-*` | `AuthService.userGetData` 인증/조회 실패 propagate |
| 4xx/5xx | device-service `ErrorCode` | `macAddress` 전송 시 device-service `DeviceLookup.LookupByMac` propagate (owner mismatch, device inactive 등) |

```bash
curl -X POST "https://api.example.com/api/v2/users/<uuid>/records?recordType=FOOTPRINT" \
  -H "api-version: 1.0.1" -H "Authorization: Bearer ..." \
  -F "measuredDataTime=2026-05-29T01:23:45Z" \
  -F "macAddress=AA:BB:CC:DD:EE:FF" \
  -F "source=TRAINER_GUIDED" \
  -F "appVersion=ios-2.3.1"
```

---

## `GET` /api/v2/users/{userId}/records

**Operation ID** &nbsp;`listRecords`
**Tags** &nbsp;`record`

Paginated record search. `api-version` 헤더에 따라 응답 DTO 분기 (v1.0.0 → `RecordV1_0_0`, v1.0.1 → `RecordV1_0_1`).

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0` `1.0.1`) | yes | — |
| path | `userId` | string (uuid) | yes | — |
| query | `from` | string (date-time, UTC) | no | range start |
| query | `to` | string (date-time, UTC) | no | range end |
| query | `ids` | array<string> | no | filter by ids |
| query | `size` | integer | no | default `10` |
| query | `page` | integer | no | default `0` |
| query | `sortBy` | string | no | default `createdAt` |
| query | `direction` | string (enum: `asc` `desc`) | no | default `desc` |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** (v1.0.0) | `application/json` | `Envelope` with `data:` `Page<`[`RecordV1_0_0`](#recordv1_0_0)`>` |
| **200** (v1.0.1) | `application/json` | `Envelope` with `data:` `Page<`[`RecordV1_0_1`](#recordv1_0_1)`>` |

#### 200 — example (v1.0.1)

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
        "deviceId": "dev-uuid", "deviceSerial": "HW-SN-001234",
        "source": "SELF", "appVersion": "ios-2.3.1"
      }
    ],
    "totalElements": 47, "totalPages": 5, "number": 0, "size": 10
  }
}
```

> v1.0.0 응답은 `deviceId` / `deviceSerial` / `source` / `appVersion` 미노출 — 그 외 동일.

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-*` | `AuthService.userGetData` 인증/조회 실패 propagate |

---

## `GET` /api/v2/users/{userId}/records/{recordId}

**Operation ID** &nbsp;`getRecord` (v1.0.0) / `getRecordV1_0_1` (v1.0.1)
**Tags** &nbsp;`record`

Single record. 404 if missing. `api-version` 헤더에 따라 응답 DTO 분기.

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0` `1.0.1`) | yes |
| path | `userId` | string (uuid) | yes |
| path | `recordId` | string (uuid) | yes |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** (v1.0.0) | `application/json` | `Envelope` with `data:` [`RecordV1_0_0`](#recordv1_0_0) |
| **200** (v1.0.1) | `application/json` | `Envelope` with `data:` [`RecordV1_0_1`](#recordv1_0_1) |
| **404** | `application/json` | [`ErrorResponse`](#errorresponse) — `Record not found` (`ResponseStatusException`) |

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 404 | — | record 미존재 (`ResponseStatusException`, message: `Record not found`) |
| 401 | `AUTH-401-*` | `AuthService.userGetData` 인증/조회 실패 propagate |

---

## `DELETE` /api/v2/users/{userId}/records

**Operation ID** &nbsp;`deleteRecords`
**Tags** &nbsp;`record`

Bulk delete by ids and/or time range.

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes |
| path | `userId` | string (uuid) | yes |

### Request body

`application/json` — [`RecordDeleteRequest`](#recorddeleterequest)

```json
{ "ids": ["..."], "from": "2026-04-01T00:00:00Z", "to": "2026-04-27T00:00:00Z" }
```

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `Envelope` with `data:` [`RecordDeleteResult`](#recorddeleteresult) |

#### 200 — example

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

---

## `POST` /api/v2/users/{userId}/activities

**Operation ID** &nbsp;`createActivity`
**Tags** &nbsp;`activity`

Create activity log.

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes |
| path | `userId` | string (uuid) | yes |

### Request body

`application/json` — [`CreateActivityRequest`](#createactivityrequest)

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

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | `Envelope` with `data:` [`Activity`](#activity) |

#### 201 — example

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

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 400 | — | `type` null / `0~4` 외 정수 / 알 수 없는 string (`IllegalArgumentException` → 400) |
| 401 | `AUTH-401-*` | `AuthService.userGetData` 인증/조회 실패 propagate |

---

## `GET` /api/v2/users/{userId}/activities

**Operation ID** &nbsp;`listActivities`
**Tags** &nbsp;`activity`

Paginated activity search. `from`/`to` are strings parsed against the user's `ZoneId`; `to` is set to end-of-day in that zone.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | — |
| path | `userId` | string (uuid) | yes | — |
| query | `from` | string | no | user-zone date/time |
| query | `to` | string | no | user-zone date/time → EOD |
| query | `size` | integer | no | default `10` |
| query | `page` | integer | no | default `0` |
| query | `sortBy` | string (enum: `startTime` `endTime`) | no | default `startTime` |
| query | `direction` | string (enum: `asc` `desc`) | no | default `desc` |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `Envelope` with `data:` [`Page_Activity`](#page_activity) |

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-*` | `AuthService.userGetData` 인증/조회 실패 propagate |

`sortBy` 가 `startTime`·`endTime` 외 값이면 silently `startTime` 으로 폴백 (400 아님).

---

## `DELETE` /api/v2/users/{userId}/activities

**Operation ID** &nbsp;`deleteActivities`
**Tags** &nbsp;`activity`

Bulk delete by ids and/or time range. `from`/`to` 는 UTC `Instant`.

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes |
| path | `userId` | string (uuid) | yes |

### Request body

`application/json` — [`ActivityDeleteRequest`](#activitydeleterequest)

```json
{ "ids": ["..."], "from": "2026-04-01T00:00:00Z", "to": "2026-04-27T00:00:00Z" }
```

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `Envelope` with `data: { result: { activityCount: integer }, total: integer }` |

**Errors**

| HTTP | 코드 | 케이스 |
|---|---|---|
| 401 | `AUTH-401-*` | `AuthService.userGetData` 인증/조회 실패 propagate |

---

## Schemas

### `Envelope`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | — |
| `data` | object | no | endpoint-specific |

### `RecordV1_0_0`

(`RecordDtoV1_0_0`) — v1.0.0 응답 DTO. 측정 컨텍스트 필드 (`deviceId` / `deviceSerial` / `source` / `appVersion`) 노출 안 함. nested 타입 (`Footprint` / `Vision` / `Score`) 은 `RecordV1_0_1` 와 동일 (재사용).

| Field | Type | Description |
|---|---|---|
| `id` | string (uuid) | — |
| `userId` | string (uuid) | — |
| `measuredDateTime` | string (date-time, UTC) | — |
| `timezone` | string (IANA zone id) | 측정 시점 사용자 zone snapshot |
| `footprint` | [`Footprint`](#footprint) | — |
| `front` | [`Vision`](#vision) | — |
| `side` | [`Vision`](#vision) | — |
| `score` | object — `{ bodyScore: number, predictedAge: number }` | — |

### `RecordV1_0_1`

(`RecordDtoV1_0_1` — V9, 이전 이름 `RecordDtoV20260113`)

| Field | Type | Description |
|---|---|---|
| `id` | string (uuid) | — |
| `userId` | string (uuid) | — |
| `measuredDateTime` | string (date-time, UTC) | — |
| `timezone` | string (IANA zone id) | 측정 시점 사용자 zone snapshot |
| `footprint` | [`Footprint`](#footprint) | — |
| `front` | [`Vision`](#vision) | — |
| `side` | [`Vision`](#vision) | — |
| `score` | object — `{ bodyScore: number, predictedAge: number }` | — |
| `deviceId` | string (uuid) \| null | device-service `DeviceLookup.LookupByMac` 응답. `allowed=true` + empty `deviceId` (미클레임/TMP) 는 null 로 정규화. macAddress 미전송 시 null |
| `deviceSerial` | string \| null | device-service 응답의 `hardwareSerial`. macAddress 미전송 시 null |
| `source` | [`RecordSource`](#recordsource) | 미전송 시 `UNSPECIFIED` |
| `appVersion` | string \| null | 전송된 앱 버전 |

### `Footprint`

| Field | Type |
|---|---|
| `id` | string |
| `firstClass` / `secondaryClass` / `thirdClass` | `{ type: number, accuracy: number }` |
| `leftFootDimension` / `rightFootDimension` | `{ length: number, width: number }` |
| `imageUrl` | string (uri) |
| `centerOfGravity` | `{ x: number, y: number }` |
| `leftPressure` / `rightPressure` | `{ top: number, middle: number, bottom: number, total: number }` |
| `weight` / `height` / `age` | number / number / integer |

### `Vision`

| Field | Type |
|---|---|
| `id` | string |
| `type` | string (enum `VisionType`) |
| `angleFace` / `anglePelvis` / `angleShoulder` | number |
| `classType` / `accuracy` | number / number |
| `imageUrl` | string (uri) |
| `keypoints` | object — 17 named keypoints, each `{ x: number, y: number, accuracy: number }` |

### `CreateRecordRequestV1_0_0`

See `POST /records — v1.0.0` request-body table.

### `CreateRecordRequestV1_0_1`

See `POST /records — v1.0.1` request-body table. v1.0.0 의 4개 part + `macAddress` (`@ValidMacAddress`) / `source` / `appVersion` / `firmwareVersion`.

### `RecordSource`

| Value | Meaning |
|---|---|
| `UNSPECIFIED` | v1.0.1 호출자가 의도적으로 미전송. "정말 모름". NULL 회피용 명시 분류 |
| `SELF` | B2C 사용자가 본인 device 로 자가 측정 (B2B 시설 무관). v1.0.0 호출은 모두 이 case |
| `TRAINER_GUIDED` | B2B 시설의 device 로 측정 (트레이너 입회 가능) |
| `IMPORT` | 데이터 마이그레이션 / 일괄 인입 (수동 또는 batch) |

DB 에는 `EnumType.STRING` 으로 저장 — 새 case 추가 시 schema 변경 없음.

### `RecordDeleteRequest`

| Field | Type | Required | Description |
|---|---|---|---|
| `ids` | array<string> | no | — |
| `from` | string (date-time, UTC) | no | — |
| `to` | string (date-time, UTC) | no | — |

### `RecordDeleteResult`

| Field | Type |
|---|---|
| `recordCount` | integer |
| `footprintCount` | integer |
| `frontVisionCount` | integer |
| `sideVisionCount` | integer |
| `deletedRecordIds` | array<string> |

### `Activity`

(`ActivityResultV20260202`)

| Field | Type | Description |
|---|---|---|
| `id` | string (uuid) | — |
| `userId` | string (uuid) | — |
| `type` | string (enum `ActivityType`) | `WORK_THERAPY` `ORIENTAL_TREATMENT` `PERSONAL_EXERCISE` `PROFESSIONAL_GUIDANCE` `MASSAGE_AND_PAIN_MANAGEMENT` |
| `startTime` | string (date-time, UTC) | — |
| `endTime` | string (date-time, UTC) | — |
| `paindata` | `{ painDecreased, samePain, mildPain, moderatePain, severePain : boolean }` | — |

### `CreateActivityRequest`

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | integer | yes | `0`=WORK_THERAPY `1`=ORIENTAL_TREATMENT `2`=PERSONAL_EXERCISE `3`=PROFESSIONAL_GUIDANCE `4`=MASSAGE_AND_PAIN_MANAGEMENT |
| `startTime` | string (date-time, UTC) | yes | — |
| `endTime` | string (date-time, UTC) | yes | — |
| `painDecreased` `samePain` `mildPain` `moderatePain` `severePain` | boolean | yes | — |

### `ActivityDeleteRequest`

| Field | Type | Required |
|---|---|---|
| `ids` | array<string> | no |
| `from` | string (date-time, UTC) | no |
| `to` | string (date-time, UTC) | no |

### `Page_Record` / `Page_Activity`

Spring Data `Page<T>`:

```json
{
  "content": [ /* T */ ],
  "totalElements": 0, "totalPages": 0,
  "number": 0, "size": 10,
  "first": true, "last": true,
  "sort": { "sorted": true, "unsorted": false }
}
```

### `ErrorResponse`

| Field | Type | Required |
|---|---|---|
| `success` | boolean | yes |
| `code` | string | yes |
| `message` | string | yes |
| `timestamp` | string (date-time, UTC) | yes |
