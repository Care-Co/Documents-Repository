# device-service

> 엔드포인트별 Header · Request · Response 정의 — **버전별 전량 전개**. 소스는 실제 컨트롤러/DTO.
> 표기 — **굵은 필드 = 필수**. 모든 REST 엔드포인트는 `api-version: 1.0.0` 단일 버전 (미등록 버전 → `400 CMN-400-001`).

Source: `/Users/jonghak/GitHub/Care&Co/device-service`
Updated: 2026-07-09

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value | Notes |
|---|---|---|
| `api-version` | `1.0.0` | 모든 엔드포인트 필수 |
| `Content-Type` | `application/json` | 요청 바디가 있을 때 (bulk 는 `multipart/form-data`) |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | device 엔드포인트 caller 식별 |
| `X-Caller-Id` | caller owner id | device 엔드포인트 caller 식별 |
| `X-Caller-Role` | `ADMIN` | admin allowed-devices 엔드포인트 (누락/불일치 → 403) |

**Security** — `@PreAuthorize` 없음. 인증은 게이트웨이에서 강제. device 엔드포인트는 `X-Caller-Type`/`X-Caller-Id` 로 owner 를 식별하고, admin 엔드포인트는 `X-Caller-Role: ADMIN` 을 서비스 레이어에서 검사 (`AllowedDeviceError.Forbidden` → 403).

**Common response envelope** (`CncResponse`) — 모든 응답 샘플의 바깥 틀. 아래 필드는 모든 엔드포인트 공통이라 개별 섹션에서 반복하지 않는다.

```json
{
  "success": true,
  "data": {},
  "token": null,
  "error": null,
  "timestamp": "2026-07-09T08:00:00Z"
}
```

| 필드 | 타입 | 필수 |
|---|---|---|
| `success` | boolean | yes |
| `data` | object | no |
| `token` | object | no |
| `error` | object | no |
| `timestamp` | string (date-time, UTC) | yes |

**Error envelope** (`ErrorResponse`)

```json
{
  "success": false,
  "code": "CMN-404-001",
  "message": "Resource not found",
  "error": "Device not found: 0a2ad7f0-..."
}
```

에러 코드 전체는 문서 하단 [## 에러 코드](#에러-코드) 참조. `error` 는 도메인 sealed Error 의 `description()`.

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning). 아래 "제공 버전" 중 하나를 보낸다. device-service 는 전 엔드포인트가 `1.0.0` 단일 버전.

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| POST | /api/v1/devices | 1.0.0 | 1.0.0 |
| GET | /api/v1/devices/{id} | 1.0.0 | 1.0.0 |
| GET | /api/v1/devices | 1.0.0 | 1.0.0 |
| DELETE | /api/v1/devices/{id} | 1.0.0 | 1.0.0 |
| PATCH | /api/v1/devices/{id}/status | 1.0.0 | 1.0.0 |
| POST | /api/v1/devices/validate | 1.0.0 | 1.0.0 |
| POST | /api/v1/admin/allowed-devices | 1.0.0 | 1.0.0 |
| GET | /api/v1/admin/allowed-devices/{id} | 1.0.0 | 1.0.0 |
| GET | /api/v1/admin/allowed-devices | 1.0.0 | 1.0.0 |
| POST | /api/v1/admin/allowed-devices/{id}/unclaim | 1.0.0 | 1.0.0 |
| PATCH | /api/v1/admin/allowed-devices/{id}/serial | 1.0.0 | 1.0.0 |
| POST | /api/v1/admin/allowed-devices/bulk | 1.0.0 | 1.0.0 |
| GET | /api/v1/devices/firmware/upgrade | 1.0.0 | 1.0.0 |
| POST | /api/v1/devices/firmware | 1.0.0 | 1.0.0 |
| GET | /api/v1/devices/firmware | 1.0.0 | 1.0.0 |
| GET | /api/v1/devices/firmware/{id} | 1.0.0 | 1.0.0 |
| PATCH | /api/v1/devices/firmware/{id}/promote | 1.0.0 | 1.0.0 |
| DELETE | /api/v1/devices/firmware/{id} | 1.0.0 | 1.0.0 |

---

## 1. `POST` /api/v1/devices

caller 본인의 device 등록. 시리얼은 반드시 출고 풀(`allowed_devices`)에 존재해야 한다. 풀이 `mac`, `deviceType`, `firmwareVersion`, `model` 의 권위 — client 는 시리얼만 보내면 된다. 신규 device 는 `REGISTERED` 로 시작.

흐름 — ① `hardwareSerial` 로 풀 row lookup, 없거나 provisional(`TMP-` prefix)이면 `NotAllowed`. ② 다른 사용자가 이미 claim 했으면 `AlreadyClaimed`. ③ 같은 `(mac, serial)` 의 non-revoked device row 가 있으면 `DuplicateRegistration`. ④ 신규 device 저장 + 풀 row `claim(deviceId)`.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | yes |
| `X-Caller-Id` | caller owner id | yes |

### `1.0.0` — Request

```json
{
  "hardwareSerial": "CN1G002-2024-000123",
  "model": "Scale2 Pro",
  "firmwareVersion": "1.4.2"
}
```

`model` / `firmwareVersion` 은 선택. 생략 시 각각 풀의 `model` / `firmware_version`(출고 시 값)으로 자동 채워진다 — client 가 보낼 이유 거의 없음.

### `1.0.0` — Response

**201 Created** — `status=REGISTERED`

```json
{
  "success": true,
  "data": {
    "id": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceNumber": "FS2-001",
    "deviceType": "SCALE2",
    "ownerType": "USER",
    "ownerId": "5f8d0a2c-1b3e-4c6a-9d2f-7a1b2c3d4e5f",
    "status": "REGISTERED",
    "model": "Scale2 Pro",
    "firmwareVersion": "1.4.2",
    "lastUsedAt": null,
    "lastUsedIp": null,
    "createdAt": "2026-07-09T08:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — 시리얼이 풀에 없거나 provisional (`NotAllowed`)</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied", "error": "Serial not in shipped pool: CN1G002-2024-000123" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 풀이 다른 사용자에 점유됨 (`AlreadyClaimed`) 또는 활성 device row 존재 (`DuplicateRegistration`)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request", "error": "Already claimed by another user" }
```

</details>

### Request 필드 정의 — `DeviceRegisterRequest`

| 필드 | 타입 | 규칙 |
|---|---|---|
| **`hardwareSerial`** | string | `@NotBlank @Size(max=64)` |
| `model` | string | `@Size(max=64)`. 미입력 시 `allowed_devices.model` fallback |
| `firmwareVersion` | string | `@Size(max=64)`. 미입력 시 `allowed_devices.firmware_version` fallback |

> `macAddress` / `deviceType` 은 더 이상 받지 않음 — 풀이 권위.

### Response 필드 정의 — `DeviceResponse`

응답 `data`. record component 선언 순서 (14 필드).

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `macAddress` | string | 정규화된 lowercase 콜론 |
| `hardwareSerial` | string | 시리얼 |
| `deviceNumber` | string \| null | owner 별 type 별 친화 번호 (예. `FS2-001`). UNCLAIMED row 는 null |
| `deviceType` | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | — |
| `ownerType` | enum `OwnerType` — `USER` \| `B2B_CENTER` \| `UNCLAIMED` | — |
| `ownerId` | string (uuid) | — |
| `status` | enum `DeviceStatus` — `REGISTERED` \| `ACTIVE` \| `DEACTIVATED` \| `REVOKED` | — |
| `model` | string \| null | — |
| `firmwareVersion` | string \| null | 현재 펌웨어 |
| `lastUsedAt` | string (date-time, UTC) \| null | — |
| `lastUsedIp` | string \| null | — |
| `createdAt` | string (date-time, UTC) \| null | — |
| `updatedAt` | string (date-time, UTC) \| null | — |

---

## 2. `GET` /api/v1/devices/{id}

device 단건 조회.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 의 `id` (device id).

### `1.0.0` — Response

**200 OK** — `DeviceResponse` (§1 과 동일 shape)

```json
{
  "success": true,
  "data": {
    "id": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceNumber": "FS2-001",
    "deviceType": "SCALE2",
    "ownerType": "USER",
    "ownerId": "5f8d0a2c-1b3e-4c6a-9d2f-7a1b2c3d4e5f",
    "status": "ACTIVE",
    "model": "Scale2 Pro",
    "firmwareVersion": "1.4.2",
    "lastUsedAt": "2026-07-08T12:00:00Z",
    "lastUsedIp": "1.2.3.4",
    "createdAt": "2026-07-01T00:00:00Z",
    "updatedAt": "2026-07-08T12:00:00Z"
  }
}
```

<details>
<summary><b>404 Not Found</b> — device 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Response 필드 정의 — `DeviceResponse`

응답 `data`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `macAddress` | string | 정규화된 lowercase 콜론 |
| `hardwareSerial` | string | 시리얼 |
| `deviceNumber` | string \| null | owner 별 type 별 친화 번호 |
| `deviceType` | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | — |
| `ownerType` | enum `OwnerType` — `USER` \| `B2B_CENTER` \| `UNCLAIMED` | — |
| `ownerId` | string (uuid) | — |
| `status` | enum `DeviceStatus` — `REGISTERED` \| `ACTIVE` \| `DEACTIVATED` \| `REVOKED` | — |
| `model` | string \| null | — |
| `firmwareVersion` | string \| null | — |
| `lastUsedAt` | string (date-time, UTC) \| null | — |
| `lastUsedIp` | string \| null | — |
| `createdAt` | string (date-time, UTC) \| null | — |
| `updatedAt` | string (date-time, UTC) \| null | — |

---

## 3. `GET` /api/v1/devices

caller 가 소유한 device 목록 (REVOKED 포함 — 감사용 보존).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | yes |
| `X-Caller-Id` | caller owner id | yes |

### `1.0.0` — Request

바디 없음 — caller 헤더로 owner 식별.

### `1.0.0` — Response

**200 OK** — `data:` `DeviceResponse[]`

```json
{
  "success": true,
  "data": [
    {
      "id": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999",
      "macAddress": "aa:bb:cc:dd:ee:ff",
      "hardwareSerial": "CN1G002-2024-000123",
      "deviceNumber": "FS2-001",
      "deviceType": "SCALE2",
      "ownerType": "USER",
      "ownerId": "5f8d0a2c-1b3e-4c6a-9d2f-7a1b2c3d4e5f",
      "status": "ACTIVE",
      "model": "Scale2 Pro",
      "firmwareVersion": "1.4.2",
      "lastUsedAt": "2026-07-08T12:00:00Z",
      "lastUsedIp": "1.2.3.4",
      "createdAt": "2026-07-01T00:00:00Z",
      "updatedAt": "2026-07-08T12:00:00Z"
    }
  ]
}
```

### Response 필드 정의 — `DeviceResponse[]`

배열 각 항목이 §1 의 `DeviceResponse`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `macAddress` | string | 정규화된 lowercase 콜론 |
| `hardwareSerial` | string | 시리얼 |
| `deviceNumber` | string \| null | 친화 번호 |
| `deviceType` | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | — |
| `ownerType` | enum `OwnerType` — `USER` \| `B2B_CENTER` \| `UNCLAIMED` | — |
| `ownerId` | string (uuid) | — |
| `status` | enum `DeviceStatus` — `REGISTERED` \| `ACTIVE` \| `DEACTIVATED` \| `REVOKED` | — |
| `model` | string \| null | — |
| `firmwareVersion` | string \| null | — |
| `lastUsedAt` | string (date-time, UTC) \| null | — |
| `lastUsedIp` | string \| null | — |
| `createdAt` | string (date-time, UTC) \| null | — |
| `updatedAt` | string (date-time, UTC) \| null | — |

---

## 4. `DELETE` /api/v1/devices/{id}

caller 본인 device unregister. `status=REVOKED` 전이 + `AllowedDevice.unclaim()` → 동일 시리얼을 다른 사용자가 재등록 가능해짐. Idempotent — 이미 REVOKED 여도 200 + 기존 row 반환.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | yes |
| `X-Caller-Id` | caller owner id | yes |

### `1.0.0` — Request

바디 없음 — path 의 `id` (device id) + 선택 쿼리 `reason`(감사 사유, `device.revocation_reason` 저장).

```
DELETE /api/v1/devices/{id}?reason=lost
```

### `1.0.0` — Response

**200 OK** — `status=REVOKED` 인 `DeviceResponse`

```json
{
  "success": true,
  "data": {
    "id": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceNumber": "FS2-001",
    "deviceType": "SCALE2",
    "ownerType": "USER",
    "ownerId": "5f8d0a2c-1b3e-4c6a-9d2f-7a1b2c3d4e5f",
    "status": "REVOKED",
    "model": "Scale2 Pro",
    "firmwareVersion": "1.4.2",
    "lastUsedAt": "2026-07-08T12:00:00Z",
    "lastUsedIp": "1.2.3.4",
    "createdAt": "2026-07-01T00:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — caller 가 owner 아님 (`NotOwner`)</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

<details>
<summary><b>404 Not Found</b> — device 없음 (`NotFound`)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Response 필드 정의 — `DeviceResponse`

§1 의 `DeviceResponse` 와 동일. `status` 는 `REVOKED`.

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` `macAddress` `hardwareSerial` `deviceNumber` | string | `deviceNumber` nullable |
| `deviceType` | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | — |
| `ownerType` | enum `OwnerType` — `USER` \| `B2B_CENTER` \| `UNCLAIMED` | — |
| `ownerId` | string (uuid) | — |
| `status` | enum `DeviceStatus` — `REGISTERED` \| `ACTIVE` \| `DEACTIVATED` \| `REVOKED` | — |
| `model` `firmwareVersion` `lastUsedIp` | string \| null | — |
| `lastUsedAt` `createdAt` `updatedAt` | string (date-time, UTC) \| null | — |

---

## 5. `PATCH` /api/v1/devices/{id}/status

caller 본인 device 의 상태 변경. `REVOKED` 는 terminal — 더 이상 변경 불가.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | yes |
| `X-Caller-Id` | caller owner id | yes |

### `1.0.0` — Request

```json
{ "status": "DEACTIVATED" }
```

### `1.0.0` — Response

**200 OK** — 갱신된 `DeviceResponse`

```json
{
  "success": true,
  "data": {
    "id": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceNumber": "FS2-001",
    "deviceType": "SCALE2",
    "ownerType": "USER",
    "ownerId": "5f8d0a2c-1b3e-4c6a-9d2f-7a1b2c3d4e5f",
    "status": "DEACTIVATED",
    "model": "Scale2 Pro",
    "firmwareVersion": "1.4.2",
    "lastUsedAt": "2026-07-08T12:00:00Z",
    "lastUsedIp": "1.2.3.4",
    "createdAt": "2026-07-01T00:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — 비-owner / REVOKED / 사용 불가 상태 규칙</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

<details>
<summary><b>404 Not Found</b> — device 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Request 필드 정의 — `DeviceUpdateStatusRequest`

| 필드 | 타입 | 규칙 |
|---|---|---|
| **`status`** | enum `DeviceStatus` — `REGISTERED` \| `ACTIVE` \| `DEACTIVATED` \| `REVOKED` | `@NotNull` |

### Response 필드 정의 — `DeviceResponse`

§1 과 동일 (14 필드).

---

## 6. `POST` /api/v1/devices/validate

`(macAddress, hardwareSerial)` 가 caller 소유 + `ACTIVE` 인지 검증. 통과 시 `lastUsedAt` / `lastUsedIp`(요청 remote addr) 갱신. v1.0.1 측정 파이프라인은 이 endpoint 대신 `DeviceLookup.LookupByMac` gRPC(mac-only) 사용.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | yes |
| `X-Caller-Id` | caller owner id | yes |

### `1.0.0` — Request

```json
{
  "macAddress": "AA:BB:CC:DD:EE:FF",
  "hardwareSerial": "CN1G002-2024-000123"
}
```

### `1.0.0` — Response

**200 OK** — 성공 시 `allowed=true`

```json
{
  "success": true,
  "data": {
    "allowed": true,
    "deviceId": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — caller 가 owner 아니거나 `ACTIVE` 아님</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

<details>
<summary><b>404 Not Found</b> — device 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Request 필드 정의 — `DeviceValidateRequest`

| 필드 | 타입 | 규칙 |
|---|---|---|
| **`macAddress`** | string | `@NotBlank @Size(max=17)` · 도메인 MAC 형식 검증 |
| **`hardwareSerial`** | string | `@NotBlank @Size(max=64)` |

### Response 필드 정의 — `DeviceValidateResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `allowed` | boolean | REST 성공 시 항상 `true` (실패는 예외) |
| `deviceId` | string (uuid) | matched device id |

---

## 7. `POST` /api/v1/admin/allowed-devices

출고 풀에 단건 추가. MAC 은 lowercase 콜론 형식으로 정규화, `hardwareSerial` unique. `hardwareSerial` 을 비워 보내면 서비스가 `TMP-{MAC}` provisional 시리얼을 자동 부여(어드민이 나중에 `/serial` 로 확정).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |
| `X-Caller-Role` | `ADMIN` | 값 검사 (누락/불일치 → 403) |

### `1.0.0` — Request

```json
{
  "macAddress": "AA:BB:CC:DD:EE:FF",
  "hardwareSerial": "CN1G002-2024-000123",
  "deviceType": "SCALE2",
  "firmwareVersion": "1.4.2",
  "productCode": "CN1G002",
  "productionCountry": "KR",
  "productionWeek": 12,
  "productionYear": 2024,
  "serialNumber": 123,
  "shippingCountry": "KR",
  "manufacturedAt": "2024-03-20T09:00:00Z"
}
```

`hardwareSerial` 을 생략하거나 빈 문자열로 보내면 서비스가 `TMP-{MAC}` provisional 시리얼을 부여한다.

### `1.0.0` — Response

**201 Created** — `AllowedDeviceResponse`

```json
{
  "success": true,
  "data": {
    "id": "9c1e...uuid",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceType": "SCALE2",
    "firmwareVersion": "1.4.2",
    "productCode": "CN1G002",
    "productionCountry": "KR",
    "productionWeek": 12,
    "productionYear": 2024,
    "serialNumber": 123,
    "shippingCountry": "KR",
    "manufacturedAt": "2024-03-20T09:00:00Z",
    "claimed": false,
    "claimedByDeviceId": null,
    "claimedAt": null,
    "provisional": false,
    "createdAt": "2026-07-09T08:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — MAC 형식 위반 (`InvalidMacFormat`) · validation</summary>

```json
{ "success": false, "code": "CMN-400-002", "message": "Validation failed", "error": "Invalid MAC format: ..." }
```

</details>

<details>
<summary><b>403 Forbidden</b> — `X-Caller-Role` 이 ADMIN 아님 (`Forbidden`)</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

<details>
<summary><b>409 Conflict</b> — mac/serial 충돌 (`Duplicate`) 또는 시리얼 충돌 (`SerialConflict`)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request" }
```

</details>

### Request 필드 정의 — `AllowedDeviceRegisterRequest`

| 필드 | 타입 | 규칙 |
|---|---|---|
| **`macAddress`** | string | `@NotBlank @Size(max=17)` · MAC 형식 |
| `hardwareSerial` | string | `@Size(max=64)`. **blank → `TMP-{MAC}` 자동 부여** |
| **`deviceType`** | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | `@NotNull` |
| `firmwareVersion` | string | `@Size(max=64)` |
| `productCode` | string | `@Size(max=32)` · xlsx `제품코드` |
| `productionCountry` | string | `@Size(max=8)` · ISO-2 |
| `productionWeek` | integer | — |
| `productionYear` | integer | 2자리 |
| `serialNumber` | integer | 주차 내 카운터 |
| `shippingCountry` | string | `@Size(max=8)` · ISO-2 |
| `manufacturedAt` | string (date-time, UTC) | — |

### Response 필드 정의 — `AllowedDeviceResponse`

응답 `data`. record component 선언 순서 (18 필드).

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `macAddress` | string | 정규화된 lowercase 콜론 |
| `hardwareSerial` | string | provisional 이면 `TMP-{MAC}` |
| `deviceType` | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | — |
| `firmwareVersion` | string \| null | 출고 펌웨어 |
| `productCode` | string \| null | — |
| `productionCountry` | string \| null | — |
| `productionWeek` | integer \| null | — |
| `productionYear` | integer \| null | — |
| `serialNumber` | integer \| null | — |
| `shippingCountry` | string \| null | — |
| `manufacturedAt` | string (date-time, UTC) \| null | — |
| `claimed` | boolean | device 에 바인딩됨 여부 |
| `claimedByDeviceId` | string (uuid) \| null | claim 한 device |
| `claimedAt` | string (date-time, UTC) \| null | — |
| `provisional` | boolean | `TMP-` 시리얼 미확정 플래그 |
| `createdAt` | string (date-time, UTC) \| null | — |
| `updatedAt` | string (date-time, UTC) \| null | — |

---

## 8. `GET` /api/v1/admin/allowed-devices/{id}

풀 단건 조회.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `X-Caller-Role` | `ADMIN` | 값 검사 (누락/불일치 → 403) |

### `1.0.0` — Request

바디 없음 — path 의 `id` (풀 row id).

### `1.0.0` — Response

**200 OK** — `AllowedDeviceResponse` (§7 과 동일 shape)

```json
{
  "success": true,
  "data": {
    "id": "9c1e...uuid",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceType": "SCALE2",
    "firmwareVersion": "1.4.2",
    "productCode": "CN1G002",
    "productionCountry": "KR",
    "productionWeek": 12,
    "productionYear": 2024,
    "serialNumber": 123,
    "shippingCountry": "KR",
    "manufacturedAt": "2024-03-20T09:00:00Z",
    "claimed": true,
    "claimedByDeviceId": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999",
    "claimedAt": "2026-07-01T00:00:00Z",
    "provisional": false,
    "createdAt": "2026-06-01T00:00:00Z",
    "updatedAt": "2026-07-01T00:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — `Forbidden`</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

<details>
<summary><b>404 Not Found</b> — `NotFound`</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Response 필드 정의 — `AllowedDeviceResponse`

§7 과 동일 (18 필드).

---

## 9. `GET` /api/v1/admin/allowed-devices

풀 목록 (페이지).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `X-Caller-Role` | `ADMIN` | 값 검사 (누락/불일치 → 403) |

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| `query` | string | no | — | `mac` / `serial` 부분 일치 |
| `claimed` | boolean | no | — | `true`=바인딩, `false`=미바인딩, 생략=전체 |
| `page` | int | no | `0` | — |
| `size` | int | no | `20` | — |

```
GET /api/v1/admin/allowed-devices?query=CN1G002&claimed=false&page=0&size=20
```

### `1.0.0` — Response

**200 OK** — `AllowedDevicePageResponse`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "9c1e...uuid",
        "macAddress": "aa:bb:cc:dd:ee:ff",
        "hardwareSerial": "CN1G002-2024-000123",
        "deviceType": "SCALE2",
        "firmwareVersion": "1.4.2",
        "productCode": "CN1G002",
        "productionCountry": "KR",
        "productionWeek": 12,
        "productionYear": 2024,
        "serialNumber": 123,
        "shippingCountry": "KR",
        "manufacturedAt": "2024-03-20T09:00:00Z",
        "claimed": false,
        "claimedByDeviceId": null,
        "claimedAt": null,
        "provisional": false,
        "createdAt": "2026-06-01T00:00:00Z",
        "updatedAt": "2026-06-01T00:00:00Z"
      }
    ],
    "totalElements": 1,
    "totalPages": 1,
    "page": 0,
    "size": 20,
    "hasNext": false
  }
}
```

<details>
<summary><b>403 Forbidden</b> — `Forbidden`</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

### Response 필드 정의 — `AllowedDevicePageResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `content` | array<`AllowedDeviceResponse`> | §7 의 18-필드 항목 |
| `totalElements` | integer (int64) | 총 row 수 |
| `totalPages` | integer | 총 페이지 수 |
| `page` | integer | 현재 페이지 |
| `size` | integer | 페이지 크기 |
| `hasNext` | boolean | — |

---

## 10. `POST` /api/v1/admin/allowed-devices/{id}/unclaim

풀 row 강제 unclaim + 바인딩된 device 가 있으면 REVOKED. 분실/교체 운영용. Idempotent.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `X-Caller-Role` | `ADMIN` | 값 검사 (누락/불일치 → 403) |

### `1.0.0` — Request

바디 없음 — path 의 `id` (풀 row id) + 선택 쿼리 `reason`(감사 사유).

```
POST /api/v1/admin/allowed-devices/{id}/unclaim?reason=replacement
```

### `1.0.0` — Response

**200 OK** — `claimed=false` 인 `AllowedDeviceResponse`

```json
{
  "success": true,
  "data": {
    "id": "9c1e...uuid",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceType": "SCALE2",
    "firmwareVersion": "1.4.2",
    "productCode": "CN1G002",
    "productionCountry": "KR",
    "productionWeek": 12,
    "productionYear": 2024,
    "serialNumber": 123,
    "shippingCountry": "KR",
    "manufacturedAt": "2024-03-20T09:00:00Z",
    "claimed": false,
    "claimedByDeviceId": null,
    "claimedAt": null,
    "provisional": false,
    "createdAt": "2026-06-01T00:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>403 Forbidden</b> — `Forbidden`</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

<details>
<summary><b>404 Not Found</b> — `NotFound`</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Response 필드 정의 — `AllowedDeviceResponse`

§7 과 동일 (18 필드).

---

## 11. `PATCH` /api/v1/admin/allowed-devices/{id}/serial

provisional row(`provisional=true`, 보통 `TMP-{MAC}` 형식)의 진짜 시리얼 확정. 이미 device 가 claim 했으면 그 device 의 `hardware_serial` 도 동기화.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |
| `X-Caller-Role` | `ADMIN` | 값 검사 (누락/불일치 → 403) |

### `1.0.0` — Request

```json
{ "hardwareSerial": "CN1G002-2024-000123" }
```

### `1.0.0` — Response

**200 OK** — `provisional=false` 인 `AllowedDeviceResponse`

```json
{
  "success": true,
  "data": {
    "id": "9c1e...uuid",
    "macAddress": "aa:bb:cc:dd:ee:ff",
    "hardwareSerial": "CN1G002-2024-000123",
    "deviceType": "SCALE2",
    "firmwareVersion": "1.4.2",
    "productCode": "CN1G002",
    "productionCountry": "KR",
    "productionWeek": 12,
    "productionYear": 2024,
    "serialNumber": 123,
    "shippingCountry": "KR",
    "manufacturedAt": "2024-03-20T09:00:00Z",
    "claimed": false,
    "claimedByDeviceId": null,
    "claimedAt": null,
    "provisional": false,
    "createdAt": "2026-06-01T00:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — 이미 확정된 row (`NotProvisional`)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter", "error": "Serial already confirmed" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — `Forbidden`</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

<details>
<summary><b>404 Not Found · 409 Conflict</b> — row 없음 (`NotFound`) · 대상 시리얼 이미 존재 (`SerialConflict`)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request", "error": "Serial already exists" }
```

</details>

### Request 필드 정의 — `ConfirmSerialRequest`

| 필드 | 타입 | 규칙 |
|---|---|---|
| **`hardwareSerial`** | string | `@NotBlank @Size(max=64)` |

### Response 필드 정의 — `AllowedDeviceResponse`

§7 과 동일 (18 필드).

---

## 12. `POST` /api/v1/admin/allowed-devices/bulk

xlsx 일괄 업로드. 첫 sheet 파싱, row 당 register 1회 호출. header=row 1, 데이터=row 2 부터. 예상 컬럼(한글) — `고유번호`, `MAC주소`, `펌웨어버전`, `제품코드`, `년도`, `생산국가`, `생산주차`, `배송국가`, `일련번호`, `등록일시`. `제품코드` → `DeviceType` 매핑은 `AdminAllowedDeviceController.mapDeviceType` inline switch (현재 `CN1G002` → `SCALE2` 만 매핑). 비-atomic — 한 row 실패해도 나머지 진행, duplicate 는 `skipped`, 그 외 실패는 `failed`. 첫 row 부터 `Forbidden` 이면 즉시 중단(403).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `multipart/form-data` | yes |
| `X-Caller-Role` | `ADMIN` | 값 검사 (누락/불일치 → 403) |

### `1.0.0` — Request

`multipart/form-data`.

| Part | 타입 | 필수 | 비고 |
|---|---|---|---|
| **`file`** | binary (xlsx) | yes | 출고 풀 시트 |

```bash
curl -X POST https://api.example.com/api/v1/admin/allowed-devices/bulk \
  -H "api-version: 1.0.0" -H "X-Caller-Role: ADMIN" \
  -F "file=@shipped_pool.xlsx"
```

### `1.0.0` — Response

**200 OK** — row 별 카운터 (per-row 실패가 있어도 200)

```json
{
  "success": true,
  "data": {
    "addedCount": 48,
    "skippedCount": 2,
    "failedCount": 1,
    "skipped": [
      { "rowNumber": 5, "hardwareSerial": "CN1G002-2024-000005", "macAddress": "aa:bb:cc:00:00:05", "reason": "Duplicate" }
    ],
    "failed": [
      { "rowNumber": 12, "hardwareSerial": "CN1G002-2024-000012", "macAddress": "aa:bb:cc:00:00:12", "reason": "Invalid MAC format" }
    ]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — xlsx 파싱 실패</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>403 Forbidden</b> — `Forbidden` (첫 row 부터 즉시 중단)</summary>

```json
{ "success": false, "code": "AUTH-403-001", "message": "Access denied" }
```

</details>

### Response 필드 정의 — `AllowedDeviceBulkResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `addedCount` | integer | 신규 insert row 수 |
| `skippedCount` | integer | duplicate |
| `failedCount` | integer | 그 외 실패 |
| `skipped` | array<`RowResult`> | duplicate row |
| `failed` | array<`RowResult`> | 실패 row |

`RowResult` —

| 필드 | 타입 | 설명 |
|---|---|---|
| `rowNumber` | integer | xlsx row 번호 |
| `hardwareSerial` | string | — |
| `macAddress` | string | — |
| `reason` | string | 실패/skip 사유 |

---

## 13. `GET` /api/v1/devices/firmware/upgrade

device 측 polling endpoint. `currentVersion` 이 최신과 다르면 최신 펌웨어 반환, 이미 최신이거나 등록된 최신 없으면 `204 No Content`.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`deviceType`** | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | yes | — |
| **`currentVersion`** | string | yes | device 가 보고하는 펌웨어 버전 |

```
GET /api/v1/devices/firmware/upgrade?deviceType=SCALE2&currentVersion=1.4.2
```

### `1.0.0` — Response

**200 OK** — 업그레이드 있음 (`FirmwareUpgradeResponse`)

```json
{
  "success": true,
  "data": {
    "latestVersion": "1.5.0",
    "downloadUrl": "https://cdn.carenco.com/fw/scale2/1.5.0.bin"
  }
}
```

**204 No Content** — 이미 최신이거나 등록된 최신 없음 (바디 없음)

<details>
<summary><b>400 Bad Request</b> — enum / 파라미터 오류</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

### Response 필드 정의 — `FirmwareUpgradeResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `latestVersion` | string | 최신 버전 |
| `downloadUrl` | string | 펌웨어 바이너리 URL |

---

## 14. `POST` /api/v1/devices/firmware

펌웨어 메타 등록. `makeLatest=true` 면 같은 `deviceType` 의 기존 latest 를 먼저 unmark.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `application/json` | yes |

### `1.0.0` — Request

```json
{
  "deviceType": "SCALE2",
  "version": "1.5.0",
  "downloadUrl": "https://cdn.carenco.com/fw/scale2/1.5.0.bin",
  "makeLatest": true,
  "notes": "OTA rollout",
  "releasedAt": "2026-06-01T00:00:00Z"
}
```

### `1.0.0` — Response

**201 Created** — `FirmwareResponse`

```json
{
  "success": true,
  "data": {
    "id": "f1a2...uuid",
    "deviceType": "SCALE2",
    "version": "1.5.0",
    "downloadUrl": "https://cdn.carenco.com/fw/scale2/1.5.0.bin",
    "isLatest": true,
    "notes": "OTA rollout",
    "releasedAt": "2026-06-01T00:00:00Z",
    "createdAt": "2026-07-09T08:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>400 Bad Request</b> — validation / enum 불일치</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>409 Conflict</b> — <code>(deviceType, version)</code> 중복</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request" }
```

</details>

> 펌웨어 엔드포인트(§13~18)는 Result/sealed Error 패턴 미적용 — 실패는 throw 되어 GlobalExceptionHandler 가 처리한다.

### Request 필드 정의 — `FirmwareRegisterRequest`

| 필드 | 타입 | 규칙 |
|---|---|---|
| **`deviceType`** | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | `@NotNull` |
| **`version`** | string | `@NotBlank @Size(max=64)` |
| **`downloadUrl`** | string | `@NotBlank @Size(max=2000)` |
| `makeLatest` | boolean (primitive) | 생략 시 `false` |
| `notes` | string | `@Size(max=500)` |
| `releasedAt` | string (date-time, UTC) | null 시 now default |

### Response 필드 정의 — `FirmwareResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string (uuid) | — |
| `deviceType` | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | — |
| `version` | string | — |
| `downloadUrl` | string | — |
| `isLatest` | boolean | — |
| `notes` | string \| null | — |
| `releasedAt` | string (date-time, UTC) \| null | — |
| `createdAt` | string (date-time, UTC) \| null | — |
| `updatedAt` | string (date-time, UTC) \| null | — |

---

## 15. `GET` /api/v1/devices/firmware

deviceType 별 펌웨어 목록. `releasedAt` 내림차순.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터.

| 파라미터 | 타입 | 필수 |
|---|---|---|
| **`deviceType`** | enum `DeviceType` — `SCALE2` \| `SCALE_PUZZLE` | yes |

```
GET /api/v1/devices/firmware?deviceType=SCALE2
```

### `1.0.0` — Response

**200 OK** — `data:` `FirmwareResponse[]`

```json
{
  "success": true,
  "data": [
    {
      "id": "f1a2...uuid",
      "deviceType": "SCALE2",
      "version": "1.5.0",
      "downloadUrl": "https://cdn.carenco.com/fw/scale2/1.5.0.bin",
      "isLatest": true,
      "notes": "OTA rollout",
      "releasedAt": "2026-06-01T00:00:00Z",
      "createdAt": "2026-06-01T00:00:00Z",
      "updatedAt": "2026-06-01T00:00:00Z"
    }
  ]
}
```

<details>
<summary><b>400 Bad Request</b> — enum / 파라미터 오류</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

### Response 필드 정의 — `FirmwareResponse[]`

배열 각 항목이 §14 의 `FirmwareResponse` (9 필드).

---

## 16. `GET` /api/v1/devices/firmware/{id}

펌웨어 단건 조회.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 의 `id` (펌웨어 id).

### `1.0.0` — Response

**200 OK** — `FirmwareResponse`

```json
{
  "success": true,
  "data": {
    "id": "f1a2...uuid",
    "deviceType": "SCALE2",
    "version": "1.5.0",
    "downloadUrl": "https://cdn.carenco.com/fw/scale2/1.5.0.bin",
    "isLatest": true,
    "notes": "OTA rollout",
    "releasedAt": "2026-06-01T00:00:00Z",
    "createdAt": "2026-06-01T00:00:00Z",
    "updatedAt": "2026-06-01T00:00:00Z"
  }
}
```

<details>
<summary><b>404 Not Found</b> — 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Response 필드 정의 — `FirmwareResponse`

§14 와 동일 (9 필드).

---

## 17. `PATCH` /api/v1/devices/firmware/{id}/promote

선택한 펌웨어를 해당 `deviceType` 의 latest 로 promote. 기존 latest 는 unmark.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 의 `id` (펌웨어 id).

### `1.0.0` — Response

**200 OK** — `isLatest=true` 인 `FirmwareResponse`

```json
{
  "success": true,
  "data": {
    "id": "f1a2...uuid",
    "deviceType": "SCALE2",
    "version": "1.5.0",
    "downloadUrl": "https://cdn.carenco.com/fw/scale2/1.5.0.bin",
    "isLatest": true,
    "notes": "OTA rollout",
    "releasedAt": "2026-06-01T00:00:00Z",
    "createdAt": "2026-06-01T00:00:00Z",
    "updatedAt": "2026-07-09T08:00:00Z"
  }
}
```

<details>
<summary><b>404 Not Found</b> — 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

### Response 필드 정의 — `FirmwareResponse`

§14 와 동일 (9 필드).

---

## 18. `DELETE` /api/v1/devices/firmware/{id}

펌웨어 row 삭제 (soft — `deletedAt` 마킹).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

### `1.0.0` — Request

바디 없음 — path 의 `id` (펌웨어 id).

### `1.0.0` — Response

**204 No Content** — 삭제 완료 (바디 없음)

<details>
<summary><b>404 Not Found</b> — 없음</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

---

## 내부 gRPC

`device-service` 는 4개 gRPC service 를 `9090` 에 노출 (게이트웨이 미라우팅, k3s 클러스터 내부 호출 전용). proto 는 `common-libs/common-grpc` (`com.carenco.grpc.device.v1`), 본 release 기준 `0.0.73`.

### `DeviceLookup.LookupByMac`

`measure-service` (v1.0.1 측정 파이프라인) 가 raw MAC 으로 표준 device record 를 resolve. MAC 이 풀에 없으면 자동으로 `TMP-{MAC}` provisional row 를 만들고 `allowed=false, rejectReason=NOT_REGISTERED` 응답 → 측정 layer 가 `device_serial=TMP-{MAC}` 로 저장 후 추후 reconciliation.

**Request**

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `mac_address` | string | yes | 내부에서 정규화 |
| `caller_owner_type` | string | yes | `USER` \| `B2B_CENTER` |
| `caller_owner_id` | string | yes | owner id |
| `caller_ip` | string | no | 허용 시 `lastUsedIp` 저장 |

**Response**

| 필드 | 타입 | 비고 |
|---|---|---|
| `allowed` | boolean | device 존재 + owner 일치 + `ACTIVE` 인 경우만 `true` |
| `device_id` | string | 성공 시 |
| `hardware_serial` | string | 항상 (실 시리얼 또는 provisional 의 `TMP-{MAC}`) |
| `owner_type` | string | 성공 시 |
| `owner_id` | string | 성공 시 |
| `device_type` | string | `DeviceType` 이름 |
| `auto_provisioned` | boolean | 이번 호출에서 `TMP-` row 를 새로 만들었는지 |
| `reject_reason` | string | `NOT_REGISTERED` / `INVALID_MAC` / `NOT_OWNER` / `DEVICE_INACTIVE` |

### `DeviceManagement` (B2B organization device 자산)

`b2b-service` 가 organization 의 device 자산을 관리. USER device 등록과는 별도 — `owner_type=B2B_CENTER` + `alias`, `registered_by`, `deactivated_by` 등.

| RPC | 비고 |
|---|---|
| `RegisterDevice` | 풀 기반 등록 (organizationId, serialNumber, alias, registeredBy). model 미입력 시 풀 model fallback |
| `GetDevice` | 단건 |
| `ListDevices` | organization 별 페이지 + 상태 필터 + 정렬 (`NAME`/`REGISTERED_AT`/`LAST_USED`/`BATTERY`(0.0.61)) |
| `UpdateDevice` | alias 변경 |
| `DeactivateDevice` | 일시 정지 (DEACTIVATED). 가역적 |
| `UnregisterDevice` | **0.0.59 신규.** 영구 해제 (REVOKED + 풀 unclaim). REVOKED 는 terminal |
| `PreviewBySerial` | **0.0.61 신규.** 시리얼만으로 풀 조회 — claim/write 없음. b2b UI 등록 preview. `{found, already_claimed, hardware_serial, device_type, firmware_version}` |

`Device` proto 필드 이력 — 0.0.61 `battery_level`(int32, `-1`=unknown) / `battery_reported_at`(ISO-8601, `""`=없음); 0.0.63 `device_type`(string, `""`=unknown); 0.0.73 `device_number`(string, `""`=미부여, 예 `FS2-001`).

`OrganizationDeviceError` → gRPC `Status` 매핑.

| Error case | gRPC Status |
|---|---|
| `InvalidOrganization`, `InvalidSerial` | `INVALID_ARGUMENT` |
| `NotAllowed`, `NotB2BDevice`, `RevokedForbidden` | `PERMISSION_DENIED` |
| `AlreadyClaimed`, `DuplicateRegistration` | `ALREADY_EXISTS` |
| `NotFound` | `NOT_FOUND` |

### `DeviceLifecycle.UnregisterAllByOwner` (0.0.59 신규)

`user-service` 의 탈퇴 outbox worker (`UserOutboxDeletionRetryExecutor.unregisterAllDevices`) 가 사용자 hard-delete 직전에 호출. owner 의 모든 device 를 REVOKED + 풀 unclaim. Idempotent — 이미 REVOKED 는 skip.

**Request**

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `owner_type` | string | yes | `USER` \| `B2B_CENTER` — invalid → `INVALID_ARGUMENT` |
| `owner_id` | string | yes | blank → `INVALID_ARGUMENT` |
| `revoked_by` | string | no | 기본 `"user-service:deletion-worker"` |
| `reason` | string | no | 기본 `"user_deletion"` |

**Response**

| 필드 | 타입 | 비고 |
|---|---|---|
| `unregistered_count` | int | 이번 호출에서 신규로 REVOKED 전이된 device 수 |
| `active_before` | int | 전이 전 owner 의 active device 수 |

### `DeviceMetrics.ReportMetrics` (0.0.62 신규, battery + firmware 통합)

`measure-service` 가 footprint / vision record 저장 직후 fire-and-forget 호출. `device.battery_level` / `battery_reported_at` / `firmware_version` / `firmware_reported_at` + `allowed_devices.current_firmware_version` 동시 갱신. 실패는 caller 측에서 삼킴 → 측정 응답 지연 0. `battery_level=-1` 또는 `firmware_version=""` 시 해당 필드 no-op.

**Request — `ReportMetricsRequest`**

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `device_id` | string | yes | 없으면 NOT_FOUND |
| `battery_level` | int32 | no | 0-100. `-1`=미보고. 범위 밖 → INVALID_ARGUMENT |
| `firmware_version` | string | no | 빈 문자열=미보고 |
| `reported_at` | string | no | ISO-8601 (UTC). 빈 문자열 → server now |
| `reported_by` | string | no | 감사 (예 `measure-service:record`) |

**Response — `ReportMetricsResponse`**

| 필드 | 타입 | 비고 |
|---|---|---|
| `battery_level` | int32 | 저장된 현재 값 (no-op 시 기존 값) |
| `battery_reported_at` | string | ISO-8601 |
| `firmware_version` | string | 저장된 현재 값 |
| `firmware_reported_at` | string | ISO-8601 |

기존 `ReportBattery`(0.0.61)는 호환 유지되나 0.0.62 부터 `ReportMetrics` 권장. 제거된 gRPC — `DeviceValidator.Validate` (common-libs 0.0.58 에서 제거, 측정은 `DeviceLookup.LookupByMac`, B2B 는 `DeviceManagement.GetDevice` + 상태 체크로 분리).

---

## 에러 코드

> 소스 — common-core `ErrorCode` enum. 코드·HTTP status·i18n 키가 enum 에서 자동 연결. `error` 필드는 도메인 sealed Error 의 `description()`.

| 코드 | HTTP | 상황 |
|---|---|---|
| `CMN-400-001` | 400 | 잘못된 파라미터·enum 위반·미등록 api-version·xlsx 파싱 실패·`NotProvisional` (`CHECK_PARAMETER`) |
| `CMN-400-002` | 400 | bean validation 위반·`InvalidMacFormat`·`InvalidBatteryLevel` (`VALIDATION_FAILED`) |
| `AUTH-403-001` | 403 | `NotAllowed`·`NotOwner`·`Forbidden` (`ACCESS_DENIED`) |
| `CMN-404-001` | 404 | device / 풀 row / 펌웨어 미존재 (`RESOURCE_NOT_FOUND`) |
| `CMN-409-001` | 409 | `AlreadyClaimed`·`DuplicateRegistration`·`Duplicate`·`SerialConflict` (`DUPLICATE_REQUEST`) |

### 도메인 sealed Error → HTTP

| Error case | HTTP | 비고 |
|---|---|---|
| `DeviceError.NotAllowed(hardwareSerial)` | 403 | 시리얼이 출고 풀에 없음/provisional |
| `DeviceError.AlreadyClaimed(hardwareSerial)` | 409 | 다른 사용자가 claim |
| `DeviceError.DuplicateRegistration(hardwareSerial)` | 409 | devices 테이블에 활성 row |
| `DeviceError.NotFound(deviceId)` | 404 | — |
| `DeviceError.NotOwner(deviceId)` | 403 | — |
| `DeviceError.InvalidBatteryLevel(level)` | 400 | — |
| `AllowedDeviceError.Forbidden(callerRole)` | 403 | `X-Caller-Role` 불일치 |
| `AllowedDeviceError.NotFound(id)` | 404 | — |
| `AllowedDeviceError.Duplicate(mac, serial)` | 409 | — |
| `AllowedDeviceError.InvalidMacFormat(raw)` | 400 | — |
| `AllowedDeviceError.SerialConflict(hardwareSerial)` | 409 | 확정 대상 시리얼 이미 존재 |
| `AllowedDeviceError.NotProvisional(id)` | 400 | 이미 확정된 row 를 재확정 시도 |

> 펌웨어 컨트롤러는 sealed Error 미참여 — 실패는 throw → GlobalExceptionHandler.
