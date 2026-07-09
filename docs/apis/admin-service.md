# admin-service

> **DEPRECATED — 미사용 서비스.** 더 이상 운영/배포하지 않으며, 소스 레포(`admin-service`)도 로컬/원격에 존재하지 않는다. `/api/v1/instances` (Spring Boot Admin 인스턴스 모니터링) 기능은 현재 어떤 MSA 에도 존재하지 않는다(전 서비스 grep 결과 0건). 아래 내용은 삭제 직전 OpenAPI 스냅샷을 양식 B 로 재정리한 **과거 기록**이며 코드 대조가 불가능하다.
>
> 이 문서는 user-service 문서의 admin 섹션(§27~34, `/api/admin/users`·`/api/admin/audit`)과 **다른 도메인**이다. 그쪽은 회원 관리/감사 API, 이 문서는 MSA 인스턴스 헬스 대시보드 래퍼다. 중복이 아니므로 리다이렉트 스텁으로 축소하지 않는다.

Source: `admin-service` (레포 삭제됨 — 대조 불가)
Updated: 2026-07-09

Spring Boot Admin Server (v4.0.0) 래퍼. 현재 등록된 MSA 인스턴스 목록을 public REST 로 노출한다. Spring Boot Admin 대시보드 자체는 함께 마운트되나 여기서 다루지 않는다.

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` (default) \| `0.0.1` \| `2025-06-24` — 엔드포인트별 Header 표 참조 |

> 버전 협상은 header-versioned (`DateFormatApiVersionParser`) — 날짜형(`2025-06-24`)과 semver(`1.0.0`)를 함께 파싱한다. 미전송 시 엔드포인트별 default 로 폴백.

**Security** &nbsp;아래 모든 엔드포인트 `permitAll` — 인증 없음.

**Common response** &nbsp;공통 envelope 없이 스키마 배열/객체를 최상위로 직접 반환한다(`CncResponse` 미사용).

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (또는 날짜형). 아래 "제공 버전" 중 하나를 보낸다. 미전송 시 default.

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| GET | /api/v1/instances | 1.0.0, 2025-06-24 | 2025-06-24 |
| GET | /api/v1/instances/{instanceId} | 0.0.1, 2025-06-24 | 2025-06-24 |

---

## 1. `GET` /api/v1/instances

현재 등록된 MSA 인스턴스 목록. 응답 shape 이 `api-version` 에 따라 분기된다 — `1.0.0` 은 최소 뷰, `2025-06-24` 는 업데이트 공지(i18n) 포함 뷰.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (default) \| `2025-06-24` | no |

또한 선택 query 파라미터 — `status` (`UP`·`DOWN`·`OFFLINE`·`UNKNOWN`·`OUT_OF_SERVICE`·`RESTRICTED`, 대소문자 무시), `stage` (앱 이름의 stage 세그먼트 매칭, 예: `shared`·`dev`·`prod`).

### `1.0.0` — Request

바디 없음 — 선택 query `status` / `stage`.

### `1.0.0` — Response

**200 OK** — `InstanceView[]` (최상위 배열)

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

### `1.0.0` — Response 필드 정의 — `InstanceView`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`id`** | string | yes | SBA instance id. |
| **`name`** | string | yes | Application name. |
| **`status`** | string | yes | `UP`·`DOWN`·`OFFLINE`·`UNKNOWN`·`OUT_OF_SERVICE`·`RESTRICTED`. |
| **`lastUpdated`** | string (date-time, UTC) | yes | — |

### `2025-06-24` — Request

바디 없음 — 선택 query `status` / `stage`.

### `2025-06-24` — Response

**200 OK** — `MsaInstanceView[]` (최상위 배열, 업데이트 공지 필드 포함)

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

### `2025-06-24` — Response 필드 정의 — `MsaInstanceView`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`id`** | string | yes | SBA instance id. |
| **`name`** | string | yes | Application name. |
| **`status`** | string | yes | `InstanceView.status` 와 동일 enum. |
| `updateType` | string | no | DB 공급, nullable. |
| `lastUpdated` | string (date-time, UTC) | no | DB 공급, nullable. |
| `updateStartTime` | string (date-time, UTC) | no | 업데이트 윈도우 시작. |
| `updateEndTime` | string (date-time, UTC) | no | 업데이트 윈도우 종료. |
| `translations` | `MsaInstanceTranslation[]` | no | 업데이트 공지 i18n 문자열. |

`translations[]` 항목 (`MsaInstanceTranslation`).

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`languageCode`** | string | yes | ISO 639-1. |
| **`updateTitle`** | string | yes | — |
| **`updateReason`** | string | yes | — |

---

## 2. `GET` /api/v1/instances/{instanceId}

인스턴스 단건 조회. 응답 shape 이 `api-version` 에 따라 분기된다 — `0.0.1` 은 최소 뷰, `2025-06-24` 는 업데이트 공지 포함 뷰.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `0.0.1` (default) \| `2025-06-24` | no |

path 파라미터 `instanceId` (문자열, Spring Boot Admin instance id) 필수.

### `0.0.1` — Request

바디 없음 — path 의 `instanceId`.

### `0.0.1` — Response

**200 OK** — `InstanceView` (단건)

```json
{
  "id": "...",
  "name": "user-service-shared",
  "status": "UP",
  "lastUpdated": "2026-04-27T08:00:00Z"
}
```

### `0.0.1` — Response 필드 정의 — `InstanceView`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`id`** | string | yes | SBA instance id. |
| **`name`** | string | yes | Application name. |
| **`status`** | string | yes | `UP`·`DOWN`·`OFFLINE`·`UNKNOWN`·`OUT_OF_SERVICE`·`RESTRICTED`. |
| **`lastUpdated`** | string (date-time, UTC) | yes | — |

<details>
<summary><b>404 Not Found</b> — instanceId 미등록</summary>

```json
{}
```

</details>

### `2025-06-24` — Request

바디 없음 — path 의 `instanceId`.

### `2025-06-24` — Response

**200 OK** — `MsaInstanceView` (단건, 업데이트 공지 필드 포함)

```json
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
```

### `2025-06-24` — Response 필드 정의 — `MsaInstanceView`

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`id`** | string | yes | SBA instance id. |
| **`name`** | string | yes | Application name. |
| **`status`** | string | yes | `InstanceView.status` 와 동일 enum. |
| `updateType` | string | no | DB 공급, nullable. |
| `lastUpdated` | string (date-time, UTC) | no | DB 공급, nullable. |
| `updateStartTime` | string (date-time, UTC) | no | 업데이트 윈도우 시작. |
| `updateEndTime` | string (date-time, UTC) | no | 업데이트 윈도우 종료. |
| `translations` | `MsaInstanceTranslation[]` | no | 업데이트 공지 i18n 문자열 (`languageCode`·`updateTitle`·`updateReason`). |

<details>
<summary><b>404 Not Found</b> — instanceId 미등록</summary>

```json
{}
```

</details>
