# measure-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/measure-service`
Updated: 2026-04-27

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` |
| `Authorization` | `Bearer <jwt>` (gateway-enforced) |
| `Content-Type` | `application/json` or `multipart/form-data` |

**Security** &nbsp;Auth context resolved internally via `AuthService.userGetData(userId)`. No `@PreAuthorize` annotations.

DTO calendar versions: `2026-01-13` (record DTO), `2026-02-02` (activity DTO).

---

## `POST` /api/internal/sync-user-info

**Operation ID** &nbsp;`syncUserInfo`
**Tags** &nbsp;`internal`
**Security** &nbsp;internal-only (gateway)

One-shot sync of records/activities `user_id` against Auth Service; backfills UTC. Used pre-deployment of UTC migration.

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `Envelope` with `data: { syncedCount: integer, utcBackfillCount: integer }` |

#### 200 — example

```json
{ "success": true, "data": { "syncedCount": 10, "utcBackfillCount": 7 } }
```

---

## `POST` /api/v2/users/{userId}/records

**Operation ID** &nbsp;`createRecord`
**Tags** &nbsp;`record`

Create record (vision/footprint).

### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | — |
| path | `userId` | string (uuid) | yes | `@ValidUuid` |
| query | `recordType` | string (enum: `VISION` `FOOTPRINT`) | yes | `@ValidEnum(RecordType)` |

### Request body

`multipart/form-data` &nbsp;**Required**

**Schema** — [`CreateRecordRequest`](#createrecordrequest)

| Part | Type | Required |
|---|---|---|
| `measuredDataTime` | string (date-time, UTC) | yes |
| `rawData` | string | no |
| `front` | binary | no |
| `side` | binary | no |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | `Envelope` with `data:` [`Record`](#record) |
| **400** | `application/json` | [`ErrorResponse`](#errorresponse) |

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
    "front": { "type": "POSE", "imageUrl": "..." },
    "side": { "type": "POSE", "imageUrl": "..." },
    "score": { "bodyScore": 80.0, "predictedAge": 30.0 }
  }
}
```

```bash
curl -X POST "https://api.example.com/api/v2/users/<uuid>/records?recordType=VISION" \
  -H "api-version: 1.0.0" -H "Authorization: Bearer ..." \
  -F "measuredDataTime=2026-04-27T01:23:45Z" \
  -F "front=@front.jpg" -F "side=@side.jpg"
```

---

## `GET` /api/v2/users/{userId}/records

**Operation ID** &nbsp;`listRecords`
**Tags** &nbsp;`record`

Paginated record search.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | — |
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
| **200** | `application/json` | `Envelope` with `data:` [`Page_Record`](#page_record) |

---

## `GET` /api/v2/users/{userId}/records/{recordId}

**Operation ID** &nbsp;`getRecord`
**Tags** &nbsp;`record`

Single record. 404 if missing.

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes |
| path | `userId` | string (uuid) | yes |
| path | `recordId` | string (uuid) | yes |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `Envelope` with `data:` [`Record`](#record) |
| **404** | `application/json` | [`ErrorResponse`](#errorresponse) |

---

## `DELETE` /api/v2/users/{userId}/records

**Operation ID** &nbsp;`deleteRecords`
**Tags** &nbsp;`record`

Bulk delete by ids and/or time range.

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

---

## `DELETE` /api/v2/users/{userId}/activities

**Operation ID** &nbsp;`deleteActivities`
**Tags** &nbsp;`activity`

Bulk delete by ids and/or time range.

### Request body

`application/json` — [`ActivityDeleteRequest`](#activitydeleterequest)

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `Envelope` with `data: { result: { activityCount: integer }, total: integer }` |

---

## Schemas

### `Envelope`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | — |
| `data` | object | no | endpoint-specific |

### `Record`

(`RecordDtoV20260113`)

| Field | Type | Description |
|---|---|---|
| `id` | string (uuid) | — |
| `userId` | string (uuid) | — |
| `measuredDateTime` | string (date-time, UTC) | — |
| `timezone` | string (IANA zone id) | — |
| `footprint` | [`Footprint`](#footprint) | — |
| `front` | [`Vision`](#vision) | — |
| `side` | [`Vision`](#vision) | — |
| `score` | object — `{ bodyScore: number, predictedAge: number }` | — |

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

### `CreateRecordRequest`

See request-body table above.

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
