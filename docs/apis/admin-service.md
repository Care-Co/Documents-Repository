# admin-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/admin-service`
Updated: 2026-04-27

Spring Boot Admin Server (v4.0.0) wrapper exposing a public REST view of currently registered MSA instances. The Spring Boot Admin dashboard is mounted alongside but not described here.

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value | Notes |
|---|---|---|
| `api-version` | `1.0.0` (default) / `2025-06-24` / `0.0.1` | header-versioned via `DateFormatApiVersionParser` |

**Security** &nbsp;`permitAll` for all endpoints below.

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning). 아래 "제공 버전" 중 하나를 보낸다. `—` 는 unversioned.

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| GET | /api/v1/instances | 1.0.0, 2025-06-24 | 2025-06-24 |
| GET | /api/v1/instances/{instanceId} | 0.0.1, 2025-06-24 | 2025-06-24 |

---

## `GET` /api/v1/instances

**Operation ID** &nbsp;`listInstances`
**Tags** &nbsp;`instances`

List currently-registered MSA instances. Response shape varies by `api-version`.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0` `2025-06-24`) | no | Default `1.0.0`. |
| query | `status` | string | no | Filter by `UP` `DOWN` `OFFLINE` `UNKNOWN` `OUT_OF_SERVICE` `RESTRICTED` (case-insensitive). |
| query | `stage` | string | no | Match by stage segment in app name (e.g. `shared`, `dev`, `prod`). |

### Responses

| Status | `api-version` | Content-Type | Schema |
|---|---|---|---|
| **200** | `1.0.0` | `application/json` | [`InstanceView`](#instanceview)[] |
| **200** | `2025-06-24` | `application/json` | [`MsaInstanceView`](#msainstanceview)[] |

#### 200 — `1.0.0` example

```json
[
  {
    "id": "...",
    "name": "user-service-shared",
    "status": "UP",
    "lastUpdated": "2026-04-27T08:00:00Z"
  }
]
```

#### 200 — `2025-06-24` example

```json
[
  {
    "id": "...",
    "name": "user-service-shared",
    "status": "UP",
    "updateType": "MAINTENANCE",
    "lastUpdated": "2026-04-27T08:00:00Z",
    "updateStartTime": "2026-04-30T00:00:00Z",
    "updateEndTime": "2026-04-30T01:00:00Z",
    "translations": [
      { "languageCode": "ko", "updateTitle": "...", "updateReason": "..." },
      { "languageCode": "en", "updateTitle": "...", "updateReason": "..." }
    ]
  }
]
```

---

## `GET` /api/v1/instances/{instanceId}

**Operation ID** &nbsp;`getInstance`
**Tags** &nbsp;`instances`

Retrieve a single MSA instance by id.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `0.0.1` `2025-06-24`) | no | Default `0.0.1`. |
| path | `instanceId` | string | yes | Spring Boot Admin instance id. |

### Responses

| Status | `api-version` | Content-Type | Schema |
|---|---|---|---|
| **200** | `0.0.1` | `application/json` | [`InstanceView`](#instanceview) |
| **200** | `2025-06-24` | `application/json` | [`MsaInstanceView`](#msainstanceview) |
| **404** | any | — | not found |

---

## Schemas

### `InstanceView`

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | SBA instance id. |
| `name` | string | yes | Application name. |
| `status` | string | yes | `UP` `DOWN` `OFFLINE` `UNKNOWN` `OUT_OF_SERVICE` `RESTRICTED`. |
| `lastUpdated` | string (date-time, UTC) | yes | — |

### `MsaInstanceView`

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | SBA instance id. |
| `name` | string | yes | Application name. |
| `status` | string | yes | See `InstanceView.status`. |
| `updateType` | string | no | DB-supplied; nullable. |
| `lastUpdated` | string (date-time, UTC) | no | DB-supplied; nullable. |
| `updateStartTime` | string (date-time, UTC) | no | Update window start. |
| `updateEndTime` | string (date-time, UTC) | no | Update window end. |
| `translations` | [`MsaInstanceTranslation`](#msainstancetranslation)[] | no | i18n strings for update notice. |

### `MsaInstanceTranslation`

| Field | Type | Required | Description |
|---|---|---|---|
| `languageCode` | string | yes | ISO 639-1. |
| `updateTitle` | string | yes | — |
| `updateReason` | string | yes | — |

