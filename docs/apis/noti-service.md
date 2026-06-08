# noti-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/noti-service`
Updated: 2026-04-27

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` |
| `Content-Type` | `application/json` |

**Security** &nbsp;No `@PreAuthorize` annotations — admin endpoints (`/admin/topics/*`, `/devices/cleanup`) must be guarded at the gateway.

---

## `POST` /api/v1/jobs

**Operation ID** &nbsp;`createFcmJob`  &nbsp;**Tags** &nbsp;`fcm-job`

Enqueue an async FCM send job.

### Request body

`application/json` &nbsp;**Required** — [`FcmJobCreateRequest`](#fcmjobcreaterequest)

```json
{ "type": "SEND_USER", "payload": { "userId": "...", "title": "hi", "body": "..." } }
```

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **202** | `application/json` | [`FcmJobEnqueueResponse`](#fcmjobenqueueresponse) |
| **400** | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 202 — example

```json
{ "jobId": "..." }
```

```bash
curl -X POST https://api.example.com/api/v1/jobs \
  -H "api-version: 1.0.0" -H "Content-Type: application/json" \
  -d '{"type":"SEND_USER","payload":{"userId":"...","title":"hi","body":"..."}}'
```

---

## `GET` /api/v1/jobs/{jobId}

**Operation ID** &nbsp;`getFcmJob`  &nbsp;**Tags** &nbsp;`fcm-job`

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `jobId` | string | yes |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`FcmJobResponse`](#fcmjobresponse) |
| **404** | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 200 — example

```json
{
  "id": "...", "jobType": "SEND_USER",
  "status": "PROCESSING",
  "successCount": 3, "failureCount": 1,
  "lastError": null, "nextRunAt": null,
  "deferredReason": null,
  "startedAt": "...", "finishedAt": null,
  "createdAt": "...", "updatedAt": "..."
}
```

---

## `PUT` /api/v1/devices

**Operation ID** &nbsp;`upsertDevice`  &nbsp;**Tags** &nbsp;`device`

Upsert a device (matches on `userId + platform + fcmToken`).

### Request body

`application/json` &nbsp;**Required** — [`RegisterDeviceRequest`](#registerdevicerequest)

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`Device`](#device) |
| **400** | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 200 — example

```json
{
  "id": "...", "userId": "...",
  "platform": "ANDROID", "pushEnabled": true,
  "appVersion": "...", "osVersion": "...", "deviceModel": "...",
  "locale": "ko-KR", "timezone": "Asia/Seoul",
  "status": "ACTIVE",
  "tokenUpdatedAt": "...", "lastSeenAt": "...",
  "createdAt": "...", "updatedAt": "..."
}
```

---

## `PATCH` /api/v1/devices/deactivate

**Operation ID** &nbsp;`deactivateDevice`  &nbsp;**Tags** &nbsp;`device`

Logout (deactivate) device by FCM token.

### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| query | `fcmToken` | string | yes | `@NotBlank` |

### Responses

| Status | Body |
|---|---|
| **204** | (empty) |

---

## `POST` /api/v1/users/{userId}/topics/{topicCode}

**Operation ID** &nbsp;`subscribeTopic`  &nbsp;**Tags** &nbsp;`topic`

Subscribe user to a notification topic.

### Parameters

| In | Name | Type | Required | Validation |
|---|---|---|---|---|
| path | `userId` | string | yes | `@NotBlank` |
| path | `topicCode` | string | yes | `@NotBlank` |

### Responses

| Status | Body |
|---|---|
| **200** | (empty) |

---

## `DELETE` /api/v1/users/{userId}/topics/{topicCode}

**Operation ID** &nbsp;`unsubscribeTopic`  &nbsp;**Tags** &nbsp;`topic`

Unsubscribe user from a notification topic. Same parameters/response as subscribe.

---

## `POST` /api/v1/admin/topics

**Operation ID** &nbsp;`createTopic`  &nbsp;**Tags** &nbsp;`topic-admin`
**Security** &nbsp;admin (gateway)

Create topic. `sortOrder` is auto-assigned to `max + 1`.

### Request body

`application/json` &nbsp;**Required** — [`NotificationTopicCreateRequest`](#notificationtopiccreaterequest)

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | [`NotificationTopic`](#notificationtopic) |

---

## `DELETE` /api/v1/admin/topics/{id}

**Operation ID** &nbsp;`deleteTopic`  &nbsp;**Tags** &nbsp;`topic-admin`
**Security** &nbsp;admin (gateway)

Delete and renumber `sortOrder` for remaining rows.

### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `id` | string | yes |

### Responses

| Status | Body |
|---|---|
| **204** | (empty) |

---

## `POST` /api/v1/devices/cleanup

**Operation ID** &nbsp;`cleanupDevices`  &nbsp;**Tags** &nbsp;`device-admin`
**Security** &nbsp;admin / internal (gateway)

Manually delete `INACTIVE` / `INVALID_TOKEN` devices past retention.

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `{ deletedCount: integer }` |

---

## Push History (`FcmHistoryController`)

| Method | Path | Operation ID | Description |
|---|---|---|---|
| GET | `/api/v1/histories/{historyId}` | `getHistory` | Single record (404 if missing) |
| GET | `/api/v1/users/{userId}/histories` | `listHistoriesByUser` | All for a user |
| GET | `/api/v1/histories/status/{status}` | `listHistoriesByStatus` | `status` ∈ `SUCCESS` `FAIL` |
| GET | `/api/v1/histories/target/{targetType}` | `listHistoriesByTarget` | `targetType` ∈ `TOKEN` `TOPIC` `USER` |
| GET | `/api/v1/histories` | `listHistories` | All histories (no pagination) |

### Responses

`200 application/json` → [`PushSendHistory`](#pushsendhistory) (single) or array.

#### example item

```json
{
  "id": "...", "targetType": "USER", "targetValue": "userId-or-masked-token",
  "userId": "...", "devicePk": "...", "topicPk": null,
  "title": "...", "body": "...",
  "sendStatus": "SUCCESS",
  "firebaseMessageId": "projects/.../messages/...",
  "errorCode": null,
  "sentAt": "..."
}
```

---

## Schemas

### `FcmJobCreateRequest`

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `type` | string (enum: `SEND_TOKEN` `SEND_USER` `SEND_TOPIC` `SEND_BULK`) | yes | `@NotNull` | — |
| `payload` | object | yes | `@NotNull` | shape matches `SendTo*Request` for the chosen `type` |

### `FcmJobEnqueueResponse`

| Field | Type | Required |
|---|---|---|
| `jobId` | string | yes |

### `FcmJobResponse`

| Field | Type | Description |
|---|---|---|
| `id` | string | — |
| `jobType` | string (enum `FcmJobType`) | — |
| `status` | string (enum: `PENDING` `DEFERRED` `PROCESSING` `SUCCESS` `PARTIAL_SUCCESS` `FAIL`) | — |
| `successCount` | integer | — |
| `failureCount` | integer | — |
| `lastError` | string | — |
| `nextRunAt` | string (date-time, UTC) | — |
| `deferredReason` | string | — |
| `startedAt` | string (date-time, UTC) | — |
| `finishedAt` | string (date-time, UTC) | — |
| `createdAt` | string (date-time, UTC) | — |
| `updatedAt` | string (date-time, UTC) | — |

### `RegisterDeviceRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `userId` | string | no | anonymous if omitted |
| `platform` | string (enum: `ANDROID` `IOS` `WEB`) | yes | `@NotNull` |
| `fcmToken` | string | yes | `@NotBlank` |
| `pushEnabled` | boolean | yes | `@NotNull` |
| `appVersion` | string | no | — |
| `osVersion` | string | no | — |
| `deviceModel` | string | no | — |
| `locale` | string | no | `@ValidLocale` |
| `timezone` | string | no | `@ValidTimezone` |

### `Device`

| Field | Type | Description |
|---|---|---|
| `id` | string | — |
| `userId` | string | — |
| `platform` | string (enum) | — |
| `pushEnabled` | boolean | — |
| `appVersion` `osVersion` `deviceModel` `locale` `timezone` | string | — |
| `status` | string (enum: `ACTIVE` `INACTIVE` `INVALID_TOKEN`) | — |
| `tokenUpdatedAt` `lastSeenAt` `createdAt` `updatedAt` | string (date-time, UTC) | — |

### `NotificationTopicCreateRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `topicCode` | string | yes | `@NotBlank` (unique) |
| `topicName` | string | yes | `@NotBlank` |
| `displayName` | string | yes | `@NotBlank` |
| `description` | string | no | — |
| `enabled` | boolean | yes | `@NotNull` |
| `subscribable` | boolean | yes | `@NotNull` |
| `allowUserUnsubscribe` | boolean | yes | `@NotNull` |
| `defaultEnabled` | boolean | yes | `@NotNull` |

### `NotificationTopic`

`NotificationTopicCreateRequest` fields plus `id`, `sortOrder` (integer), `createdAt`, `updatedAt`.

### `PushSendHistory`

| Field | Type | Description |
|---|---|---|
| `id` | string | — |
| `targetType` | string (enum: `TOKEN` `TOPIC` `USER`) | — |
| `targetValue` | string | userId or masked token |
| `userId` | string | — |
| `devicePk` | string | — |
| `topicPk` | string | — |
| `title` | string | — |
| `body` | string | — |
| `sendStatus` | string (enum: `SUCCESS` `FAIL`) | — |
| `firebaseMessageId` | string | populated on success |
| `errorCode` | string | populated on failure |
| `sentAt` | string (date-time, UTC) | — |

### `ErrorResponse`

| Field | Type | Required |
|---|---|---|
| `success` | boolean | yes |
| `code` | string | yes |
| `message` | string | yes |
| `timestamp` | string (date-time, UTC) | yes |
