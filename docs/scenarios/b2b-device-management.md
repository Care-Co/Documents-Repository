# B2B 센터 — 기기 관리 4단계 시나리오

> 갱신. 2026-06-01
> 대상. 프론트엔드 (B2B 관리자 콘솔)
> Base path. `/api/v2/b2b/organizations/{organizationId}/devices`
> 공통 헤더. `Authorization: Bearer {b2b_token}` + `api-version: 1.0.0`
> 응답 envelope. `CncResponse` — `{ success, code, message, data, error, timestamp }`

본 문서는 4단계 시나리오 (등록 / 조회 / 장비명 변경 / 삭제) 의 흐름 + endpoint + 요청/응답 예시 + 에러 케이스를 한 곳에 정리한다. 전체 명세는 [`b2b-service.md`](../apis/b2b-service.md), 백엔드 동작은 [`device-service.md`](../apis/device-service.md).

---

## 권한 요약

| 역할 | 등록 | 조회 | 장비명 변경 | 삭제 |
|---|---|---|---|---|
| OWNER | ✓ | ✓ | ✓ | ✓ |
| ADMIN | ✓ | ✓ | ✓ | ✓ |
| STAFF / TRAINER (멤버) | ✗ | ✓ | ✗ | ✗ |
| 비-멤버 | ✗ | ✗ | ✗ | ✗ |

권한 부족 시 응답.
- `403 AUTH-403-001 OrgRoleRequired` — OWNER/ADMIN 권한 부족.
- `403 AUTH-403-002 NotMember` — 시설 멤버가 아님.

---

## 1단계. 기기 등록

### 흐름

```
[기기 등록 버튼]
   ↓
[시리얼 번호 입력]
   ↓
GET /devices/preview?serial=... ────→ 장비번호 / 장비유형 / 펌웨어 미리보기
   ↓ (found=true && alreadyClaimed=false)
[장비명(별칭) 입력]
   ↓
POST /devices ────→ 센터 장비 등록 (REGISTERED)
```

### 1.1 시리얼로 미리보기

```bash
GET /api/v2/b2b/organizations/{orgId}/devices/preview?serial=CN1G00226KR1700001JP
Authorization: Bearer {token}
api-version: 1.0.0
```

권한. **OWNER / ADMIN**.

**응답 200 — 등록 가능**
```json
{
  "success": true, "code": "200", "message": "OK",
  "data": {
    "found": true,
    "alreadyClaimed": false,
    "hardwareSerial": "CN1G00226KR1700001JP",
    "deviceType": "SCALE2",
    "firmwareVersion": "1.0.0"
  }
}
```

**응답 200 — 이미 등록된 시리얼**
```json
{
  "data": { "found": true, "alreadyClaimed": true, ... }
}
```
→ UI 안내. "이미 다른 시설에 등록된 기기".

**응답 200 — 풀에 없는 시리얼**
```json
{
  "data": { "found": false, "alreadyClaimed": false, "hardwareSerial": null, ... }
}
```
→ UI 안내. "출고된 기기가 아닙니다".

**응답 403** — OWNER/ADMIN 권한 없음.

### 1.2 등록

```bash
POST /api/v2/b2b/organizations/{orgId}/devices
Authorization: Bearer {token}
api-version: 1.0.0
Content-Type: application/json

{
  "serialNumber": "CN1G00226KR1700001JP",
  "alias": "1번 인바디"
}
```

권한. **OWNER / ADMIN**. 라이센스 ACTIVE 또는 GRACE 필요.

**응답 201**
```json
{
  "success": true, "code": "201", "message": "Created",
  "data": {
    "deviceId": "dev-c3d4...",
    "organizationId": "org-A...",
    "serialNumber": "CN1G00226KR1700001JP",
    "alias": "1번 인바디",
    "status": "ACTIVE",
    "registeredAt": "2026-06-01T10:00:00Z",
    "registeredBy": "u-9f3e...",
    "deactivatedAt": null,
    "deactivatedBy": null,
    "deactivationReason": null,
    "lastUsedAt": null,
    "batteryLevel": null,
    "batteryReportedAt": null
  }
}
```

**에러**
- `403 AUTH-403-001` — OWNER/ADMIN 권한 부족.
- `403 LIC-403-001` — 라이센스 만료.
- `409 DEV-409-001` — 이미 다른 owner 가 등록한 시리얼.
- `404 DEV-404-001` — 풀에 없는 시리얼 (preview 통과 안 한 경우).

---

## 2단계. 기기 조회 (목록)

### 흐름

```
[기기 목록 화면]
   ↓
GET /devices?sortBy=...&status=ACTIVE
   ↓
[테이블 표시 — 장비번호 / 장비명 / 등록일 / 최근 사용일시 / 배터리 잔량]
```

### 요청

```bash
GET /api/v2/b2b/organizations/{orgId}/devices?sortBy=battery&status=ACTIVE
Authorization: Bearer {token}
api-version: 1.0.0
```

권한. **시설 멤버** (OWNER/ADMIN/STAFF/TRAINER 모두).

#### Query 파라미터

| 이름 | 타입 | 설명 |
|---|---|---|
| `keyword` | string | `serialNumber` / `alias` 부분 일치 (선택) |
| `status` | string | `ACTIVE` / `INACTIVE` 필터 (선택) |
| `sortBy` | string | 정렬 기준 — 아래 표 참고 |

#### `sortBy` ↔ 사용자 정렬 기준

| 사용자 표기 | `sortBy` 값 | 정렬 |
|---|---|---|
| 최근 등록순 | `registered_at` (기본값) | `registeredAt` DESC |
| 최근 사용일순 | `last_used` | `lastUsedAt` DESC |
| 배터리 잔량 | `battery` | `batteryLevel` DESC, NULLS LAST |
| (보너스) 장비명 가나다순 | `name` | `alias` ASC |

### 응답 200

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "deviceId": "dev-c3d4...",
        "organizationId": "org-A...",
        "serialNumber": "CN1G00226KR1700001JP",
        "alias": "1번 인바디",
        "status": "ACTIVE",
        "registeredAt": "2026-05-13T10:00:00Z",
        "registeredBy": "u-9f3e...",
        "deactivatedAt": null,
        "deactivatedBy": null,
        "deactivationReason": null,
        "lastUsedAt": "2026-06-01T11:20:00Z",
        "batteryLevel": 87,
        "batteryReportedAt": "2026-06-01T11:20:00Z"
      }
    ]
  }
}
```

### UI 컬럼 매핑

| UI 표시 | 응답 필드 | 비고 |
|---|---|---|
| 장비번호 | `serialNumber` | 출고 시리얼 (예. `CN1G00226KR1700001JP`) |
| 장비명 | `alias` | 사용자 부여. null 가능 |
| 등록일 | `registeredAt` | ISO-8601 UTC. 클라이언트 timezone 변환 |
| 최근 사용일시 | `lastUsedAt` | 측정 이력 없으면 null |
| 배터리 잔량 | `batteryLevel` | 0-100. 측정 발생한 적 없으면 null — 화면에 "-" 표시 권장 |
| (참고) 마지막 배터리 보고 시각 | `batteryReportedAt` | tooltip 등 보조 표시용 |

### 동작 메모

- 배터리 값은 **측정이 발생할 때마다 자동 갱신** (별도 클라이언트 호출 불필요).
- `null` 인 디바이스는 측정 이력 자체가 없는 신규 등록 device.
- `sortBy=battery` 시 NULL 은 항상 마지막 페이지에 위치.

---

## 3단계. 장비명 변경

### 흐름

```
[조회된 장비 클릭]
   ↓
GET /devices/{deviceId} ────→ 상세 화면
   ↓
[장비명(alias) 수정 후 편집 버튼]
   ↓
PATCH /devices/{deviceId} {"alias": "..."}
```

### 3.1 상세 조회

```bash
GET /api/v2/b2b/organizations/{orgId}/devices/{deviceId}
```

권한. **시설 멤버**.

**응답 200** — `DeviceResponse` 단건 (2단계와 동일 shape, 1개 row).

#### UI 표시 ↔ 응답 필드

| UI 표시 | 응답 필드 |
|---|---|
| 장비명 | `alias` |
| 시리얼 넘버 | `serialNumber` |
| 장비번호 | `serialNumber` (현재 별도 ID 없음 — 같은 값 사용) |
| 장비유형 | _(현재 응답에 미포함. 등록 시 preview 의 `deviceType` 을 클라이언트 캐시하거나, 향후 backend PR 로 추가 가능)_ |
| 등록일 | `registeredAt` |

### 3.2 alias 변경

```bash
PATCH /api/v2/b2b/organizations/{orgId}/devices/{deviceId}
Authorization: Bearer {token}
Content-Type: application/json

{
  "alias": "정문 인바디"
}
```

권한. **OWNER / ADMIN**.

**응답 200** — 갱신된 `DeviceResponse`.

**에러**
- `403 AUTH-403-001` — OWNER/ADMIN 권한 부족.
- `404 DEV-404-002` — device 가 시설 소속이 아님.

---

## 4단계. 장비 삭제

### 흐름

```
[조회된 장비 클릭]
   ↓
[삭제 버튼 클릭]
   ↓
DELETE /devices/{deviceId}?reason=...
   ↓
[목록에서 자동으로 사라짐]
```

### 요청

```bash
DELETE /api/v2/b2b/organizations/{orgId}/devices/{deviceId}?reason=lost
Authorization: Bearer {token}
api-version: 1.0.0
```

권한. **OWNER / ADMIN**.

| Query | 필수 | 설명 |
|---|---|---|
| `reason` | no | 사유 (예. `lost` / `discarded` / `replaced`). audit log 에 저장 |

### 응답 200

```json
{
  "success": true,
  "data": {
    "deviceId": "dev-c3d4...",
    "serialNumber": "CN1G00226KR1700001JP",
    "alias": "1번 인바디",
    "status": "INACTIVE",
    "deactivatedAt": "2026-06-01T12:00:00Z",
    "deactivatedBy": "u-9f3e...",
    "deactivationReason": "lost",
    "batteryLevel": 87,
    "batteryReportedAt": "2026-06-01T11:20:00Z"
  }
}
```

### 동작 메모

- 백엔드는 `status=REVOKED` 영구 전이 + 출고 풀의 row `unclaim`.
- 같은 시리얼이 다른 시설에서 재등록 가능해짐.
- `GET /devices?status=ACTIVE` 목록에서 자동으로 사라짐 — 클라이언트 추가 처리 불필요.
- **Idempotent** — 이미 삭제된 device 에 다시 호출해도 200 + 같은 row.

### 에러

- `403 AUTH-403-001` — OWNER/ADMIN 권한 부족.
- `404 DEV-404-002` — device 가 시설 소속이 아님.

---

## 부록 A. DeviceResponse 전체 스키마

```typescript
interface DeviceResponse {
  deviceId: string;             // UUID
  organizationId: string;       // UUID
  serialNumber: string;         // 출고 시리얼 (장비번호)
  alias: string | null;         // 장비명
  status: 'ACTIVE' | 'INACTIVE';
  registeredAt: string;         // ISO-8601 UTC
  registeredBy: string | null;  // b2b_users.id
  deactivatedAt: string | null;
  deactivatedBy: string | null;
  deactivationReason: string | null;
  lastUsedAt: string | null;    // 측정 발생 시 자동 갱신
  batteryLevel: number | null;  // 0-100. 측정 시 ML 응답의 battery 값
  batteryReportedAt: string | null;
}
```

## 부록 B. 에러 코드 표

| HTTP | code | 의미 | 시나리오 |
|---|---|---|---|
| 400 | `CMN-400-002` | Validation failed | 잘못된 alias 길이 등 |
| 401 | `AUTH-401-001` | Token invalid / expired | 재로그인 유도 |
| 403 | `AUTH-403-001` | OWNER/ADMIN 필요 | 시설 멤버이지만 권한 부족 |
| 403 | `AUTH-403-002` | NotMember | 시설 멤버가 아님 |
| 403 | `LIC-403-001` | 라이센스 만료 | 등록만 막힘, 조회/삭제는 가능 |
| 404 | `DEV-404-001` | 풀에 없는 시리얼 | preview 가 found=false 였는데 우회 등록 시도 |
| 404 | `DEV-404-002` | DeviceNotInOrg | URL 의 orgId / deviceId 가 매칭 안 됨 |
| 409 | `DEV-409-001` | 이미 등록된 시리얼 | 다른 시설이 점유 |
| 5xx | `CMN-500-001` | 서버 에러 | 재시도 후 안내 |

## 부록 C. 관련 백엔드 흐름 (참고)

- 등록 = `b2b-service` → device-service `DeviceManagement.RegisterDevice` gRPC + `OrganizationDeviceService.register` (풀에서 mac / deviceType / firmware / model 권위).
- 삭제 = device-service `DeviceManagement.UnregisterDevice` gRPC (0.0.59 신규) — REVOKED + 풀 unclaim.
- 배터리 = measure-service 가 측정마다 device-service `DeviceMetrics.ReportMetrics` (0.0.62 신규) fire-and-forget. 측정 응답 latency 영향 없음.
- 펌웨어 = 클라이언트가 측정 요청에 `firmwareVersion` 같이 보내면 ReportMetrics 가 자동 갱신 (앱 변경 후 동작).
