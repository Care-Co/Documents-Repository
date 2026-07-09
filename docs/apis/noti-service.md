# noti-service

> 엔드포인트별 Header · Request · Response 정의 — 전 엔드포인트 `1.0.0` 단일 버전. 소스는 실제 컨트롤러/DTO.
> 표기 — **굵은 필드 = 필수**.

Source: `/Users/jonghak/GitHub/Care&Co/noti-service`
Updated: 2026-07-09

> **내부 전용 (2026-07 기준)** — 게이트웨이가 noti 라우팅을 의도적으로 비활성 상태로 유지한다 (`/api/v1/devices/**` 가 device-service 경로와 충돌 + 외부 노출 설계 미결). 외부(app.carencoinc.com)에서 호출 불가, **k3s 클러스터 내부에서만** 호출 가능하다.

**Servers**
- `http://noti.carenco.svc.cluster.local:8080` (클러스터 내부)

**Common headers**

| Header | Value | Notes |
|---|---|---|
| `api-version` | `1.0.0` | 전 엔드포인트 단일 버전 |
| `Content-Type` | `application/json` | 바디 있는 요청 |

**Response envelope** — 없음. `CncResponse` 를 쓰지 않고 DTO 를 최상위로 직접 반환한다. 검증 실패는 400.

---

## 1. `PUT` /api/v1/devices

FCM 디바이스 토큰 upsert — 같은 토큰이면 갱신, 없으면 등록.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |

### `1.0.0` — Request

```json
{
  "userId": "3f2a...uuid",
  "platform": "ANDROID",
  "fcmToken": "fcm-token...",
  "pushEnabled": true,
  "appVersion": "1.4.0",
  "osVersion": "14",
  "deviceModel": "SM-S926N",
  "locale": "ko-KR"
}
```

| 필드 | 값 | 규칙 |
|---|---|---|
| **`userId`** | uuid | |
| **`platform`** | `ANDROID` \| `IOS` | |
| **`fcmToken`** | string | upsert 키 |
| `pushEnabled` | boolean | 기본 true |
| `appVersion` `osVersion` `deviceModel` `locale` | string | 선택 메타 |

### `1.0.0` — Response

**200 OK** — 저장된 디바이스

```json
{
  "id": "device-pk",
  "userId": "3f2a...uuid",
  "platform": "ANDROID",
  "pushEnabled": true,
  "appVersion": "1.4.0",
  "osVersion": "14",
  "deviceModel": "SM-S926N",
  "locale": "ko-KR"
}
```

<details>
<summary><b>400 Bad Request</b> — 필수 필드 누락 · enum 위반</summary>

(Bean Validation 400 — envelope 없음)

</details>

---

## 2. `PATCH` /api/v1/devices/deactivate

푸시 비활성화 (토큰 기준).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음. 쿼리 파라미터:

| 파라미터 | 타입 | 필수 |
|---|---|---|
| **`fcmToken`** | string | yes |

### `1.0.0` — Response

**200 OK** — 바디 없음.

---

## 3. `POST` /api/v1/users/{userId}/topics/{topicCode} · `DELETE` 동일 경로

토픽 구독(POST) / 구독 해제(DELETE).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 의 `userId` (uuid), `topicCode` (string).

### `1.0.0` — Response

**200 OK** — 바디 없음.

---

## 4. `POST` /api/v1/jobs

FCM 발송 잡 등록 (비동기 — 워커가 1초 폴링으로 처리).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |

### `1.0.0` — Request

```json
{
  "type": "SEND_USER",
  "payload": { "userId": "3f2a...uuid", "title": "제목", "body": "내용", "imageUrl": null }
}
```

| 필드 | 값 | 규칙 |
|---|---|---|
| **`type`** | `SEND_TOKEN` \| `SEND_USER` \| `SEND_TOPIC` | |
| **`payload`** | object | type 별 요청 shape — SEND_TOKEN: `{tokens[], title, body, imageUrl}` / SEND_USER: `{userId, title, body, imageUrl}` / SEND_TOPIC: `{topic, title, body, imageUrl}` |

### `1.0.0` — Response

**202 Accepted** — 잡 ID 발급 (처리는 비동기)

```json
{ "jobId": "job-uuid" }
```

---

## 5. `GET` /api/v1/jobs/{jobId}

잡 상태 조회.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 의 `jobId`.

### `1.0.0` — Response

**200 OK**

```json
{
  "id": "job-uuid",
  "jobType": "SEND_USER",
  "status": "SUCCESS",
  "successCount": 2,
  "failureCount": 0,
  "lastError": null,
  "nextRunAt": null,
  "deferredReason": null
}
```

| 필드 | 값 |
|---|---|
| `status` | `PENDING` \| `DEFERRED` \| `PROCESSING` \| `SUCCESS` \| `PARTIAL_SUCCESS` \| `FAILED` |
| `successCount` / `failureCount` | 대상 디바이스별 발송 결과 수 |
| `deferredReason` `nextRunAt` | DEFERRED 시 사유·재시도 시각 |

---

## 6. `GET` /api/v1/histories/{historyId} · `/users/{userId}/histories` · `/histories/status/{status}` · `/histories/target/{targetType}` · `/histories`

발송 이력 조회 5종 — 단건 / 사용자별 / 상태별(`SUCCESS`·`FAILED`) / 대상타입별(`TOKEN`·`TOPIC`) / 전체. 응답은 단건 또는 배열.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 파라미터만 (전체 조회는 파라미터 없음, 페이징 없음).

### `1.0.0` — Response

**200 OK** — `PushSendHistoryResponse` (배열 엔드포인트는 `[...]`)

```json
{
  "id": "history-uuid",
  "targetType": "TOKEN",
  "targetValue": "fcm-token...",
  "userId": "3f2a...uuid",
  "devicePk": "device-pk",
  "topicPk": null,
  "title": "제목",
  "body": "내용"
}
```

---

## 7. `POST` /api/v1/admin/topics

알림 토픽 생성 (admin).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |

### `1.0.0` — Request

```json
{
  "topicCode": "MARKETING",
  "topicName": "marketing",
  "displayName": "마케팅 알림",
  "description": "이벤트·혜택 소식",
  "enabled": true,
  "subscribable": true,
  "allowUserUnsubscribe": true,
  "defaultEnabled": false
}
```

| 필드 | 값 | 규칙 |
|---|---|---|
| **`topicCode`** | string | 구독 API 의 path 키 |
| **`topicName`** | string | FCM 토픽명 |
| `displayName` `description` | string | 표시용 |
| `enabled` `subscribable` `allowUserUnsubscribe` `defaultEnabled` | boolean | 정책 플래그 |

### `1.0.0` — Response

**201 Created** — 생성된 토픽 (`NotificationTopicResponse` — 요청 필드 + `id`)

```json
{
  "id": "topic-pk",
  "topicCode": "MARKETING",
  "topicName": "marketing",
  "displayName": "마케팅 알림",
  "description": "이벤트·혜택 소식",
  "enabled": true,
  "subscribable": true,
  "allowUserUnsubscribe": true
}
```

---

## 8. `DELETE` /api/v1/admin/topics/{id}

토픽 삭제 (admin).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 의 `id` (topic PK).

### `1.0.0` — Response

**200 OK** — 바디 없음.

---

## 9. `POST` /api/v1/devices/cleanup

비활성/무효 디바이스 수동 정리 (admin — 자동 정리는 매일 03:00 cron).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음.

### `1.0.0` — Response

**200 OK**

```json
{ "deletedCount": 12 }
```
