# device-service

> 갱신. 2026-06-01 (proto v0.0.62)
> 스타일. OpenAPI 3.1 + 코드 기반 endpoint 카탈로그 + 스키마 테이블
> Base path. `/api/v1/devices`, `/api/v1/admin/allowed-devices`
> 응답 envelope. `CncResponse` (common-core / Pattern B)

소스. `/Users/jonghak/GitHub/Care&Co/device-service`

---

## 1. 개요

`device-service` 는 다음을 소유한다.

- 출고된 기기 풀 (`allowed_devices`)
- 풀에 등록된 시리얼만 사용할 수 있는 owner 별 device 등록 (USER / B2B_CENTER)
- owner 검증 + lifecycle 상태 변경
- OTA 펌웨어 메타데이터
- 측정 파이프라인을 위한 cluster 내부 MAC lookup
- 측정 직후 battery / firmware 자동 보고

### 1.1 도메인 책임

| 도메인 | 책임 |
|---|---|
| `allowed_device` | 출고 기기 풀. 어드민이 단건 또는 xlsx 일괄 추가. register 흐름이 소비. `claim` / `unclaim` 으로 device row 와 단방향 binding |
| `device` | 풀의 시리얼로 owner 에 등록. owner 검증, 상태 변경, unregister (REVOKED + 풀 unclaim) |
| `firmware` | `deviceType + version` 단위 OTA 펌웨어 메타. type 당 최신 1개. device 측 polling endpoint 제공 |
| `grpc` | 4개 internal gRPC service. `DeviceLookup` (mac → serial, measure-service 용), `DeviceManagement` (B2B 자산 lifecycle + 등록 preview), `DeviceLifecycle` (탈퇴 worker 의 일괄 unregister), `DeviceMetrics` (측정 직후 battery + firmware 보고) |

### 1.2 런타임

| 항목 | 값 |
|---|---|
| Java / Spring | Java 25, Spring Boot 4.0.5 |
| Persistence | PostgreSQL 17 + Flyway (`spring-boot-starter-flyway`) |
| HTTP 포트 | `8080` |
| gRPC 포트 | `9090` |
| Discovery/config | Spring Cloud Config (`CONFIG_URI`), 프로필 `dev-k3s` / `prod-k3s` |
| Kubernetes 서비스 | `device.carenco.svc.cluster.local` (`http:8080`, `grpc:9090`) |
| Prometheus | `/actuator/prometheus` |

---

## 2. API 컨벤션

### 2.1 공통 헤더

| 헤더 | 값 | 필수 | 비고 |
|---|---|---|---|
| `api-version` | `1.0.0` | yes | Spring 의 header version selector |
| `Content-Type` | `application/json` | request body 시만 | REST JSON endpoint |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | device owner endpoint | gateway / auth layer 가 주입 |
| `X-Caller-Id` | UUID 문자열 | device owner endpoint | `user.id` 또는 B2B center id |
| `X-Caller-Role` | `ADMIN` (선택) | admin endpoint | gateway 가 JWT role 에서 주입. 없거나 `ADMIN` 이 아니면 `AllowedDeviceError.Forbidden` |

### 2.2 응답 envelope

성공 응답은 common-core 의 `CncResponse` 사용.

```json
{
  "success": true,
  "data": {},
  "token": null,
  "error": null,
  "timestamp": "2026-06-01T01:00:00Z"
}
```

binding 단계에서 빠지는 에러는 400 으로 매핑.

```json
{
  "success": false,
  "data": {
    "message": "Missing required header",
    "error": "Header 'X-Caller-Type' is required"
  }
}
```

### 2.3 보안 메모

- 컨트롤러는 현재 `@PreAuthorize` 사용하지 않음. owner 식별은 헤더로 통과.
- 어드민 endpoint 는 service 내부에서 `X-Caller-Role == "ADMIN"` 확인 (`AllowedDeviceService.requireAdmin`). 운영 gateway 가 JWT role 을 이 헤더로 변환해야 함.
- 펌웨어 운영 endpoint 는 코드에서 operations 표시만. fine-grained auth 는 follow-up.
- 다른 MSA 는 REST 가 아니라 gRPC adapter (`DeviceLookup` / `DeviceManagement` / `DeviceLifecycle` / `DeviceMetrics`) 사용.

---

## 3. Endpoint 카탈로그

### 3.1 Device (사용자 / B2C owner)

| # | Method | Path | 용도 | Caller 헤더 |
|---|---|---|---|---|
| 1 | `POST` | `/api/v1/devices` | 풀 시리얼로 caller 본인 device 등록 | 필수 |
| 2 | `GET` | `/api/v1/devices/{id}` | 단건 조회 | no |
| 3 | `GET` | `/api/v1/devices` | caller 본인 device 목록 | 필수 |
| 4 | `DELETE` | `/api/v1/devices/{id}?reason=...` | caller 본인 device unregister (REVOKED + 풀 unclaim, idempotent) | 필수 |
| 5 | `PATCH` | `/api/v1/devices/{id}/status` | 상태 변경 | 필수 |
| 6 | `POST` | `/api/v1/devices/validate` | caller 가 사용 가능한 device 인지 검증 | 필수 |

### 3.2 어드민 — 출고 풀 (allowed-devices)

| # | Method | Path | 용도 |
|---|---|---|---|
| 7 | `POST` | `/api/v1/admin/allowed-devices` | 풀 단건 추가 |
| 8 | `GET` | `/api/v1/admin/allowed-devices/{id}` | 풀 단건 조회 |
| 9 | `GET` | `/api/v1/admin/allowed-devices?query&claimed&page&size` | 풀 목록 |
| 10 | `POST` | `/api/v1/admin/allowed-devices/{id}/unclaim` | 강제 unclaim + 바인딩된 device REVOKED (분실/교체 운영) |
| 11 | `PATCH` | `/api/v1/admin/allowed-devices/{id}/serial` | 임시 serial (`TMP-...`) 을 진짜 serial 로 확정 |
| 12 | `POST` | `/api/v1/admin/allowed-devices/bulk` | xlsx 일괄 업로드 (multipart/form-data) |

### 3.3 펌웨어

| # | Method | Path | 용도 |
|---|---|---|---|
| 13 | `GET` | `/api/v1/devices/firmware/upgrade` | device 별 최신 펌웨어 확인 |
| 14 | `POST` | `/api/v1/devices/firmware` | 펌웨어 등록 |
| 15 | `GET` | `/api/v1/devices/firmware` | deviceType 별 펌웨어 목록 |
| 16 | `GET` | `/api/v1/devices/firmware/{id}` | 단건 조회 |
| 17 | `PATCH` | `/api/v1/devices/firmware/{id}/promote` | 최신으로 promote |
| 18 | `DELETE` | `/api/v1/devices/firmware/{id}` | 삭제 |

---

## 4. Device API

### 4.1 `POST` /api/v1/devices

**Operation ID** `registerDevice`
**Tags** `device`
**Security** caller headers

caller 본인의 device 등록. 시리얼은 반드시 출고 풀에 존재해야 한다. 풀이 `mac`, `deviceType`, `firmwareVersion`, `model` 의 권위 — client 는 시리얼만 보내면 된다. 신규 device 는 `REGISTERED` 로 시작.

흐름.
1. `hardwareSerial` 로 풀 row lookup. 없거나 provisional (`TMP-` prefix) 이면 `DeviceError.NotAllowed`.
2. 이미 다른 사용자가 claim 했으면 `DeviceError.AlreadyClaimed`.
3. 같은 `(mac, serial)` 의 non-revoked device row 가 있으면 `DeviceError.DuplicateRegistration`.
4. 신규 device 저장 + 풀 row 에 `claim(deviceId)`.

#### 헤더

| 이름 | 필수 | 설명 |
|---|---|---|
| `api-version` | yes | `1.0.0` |
| `X-Caller-Type` | yes | `USER` 또는 `B2B_CENTER` |
| `X-Caller-Id` | yes | caller owner id |

#### Request body

`application/json` — [`DeviceRegisterRequest`](#deviceregisterrequest)

```json
{
  "hardwareSerial": "CN1G00226KR1700001JP"
}
```

`model` / `firmwareVersion` 은 optional. 생략 시 `model` 은 풀의 `model` 값으로, `firmwareVersion` 은 풀의 `firmware_version` (출고 시 값) 으로 자동 채워진다. client 가 보낼 이유 거의 없음.

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `201` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) | Created |
| `400` | `CncResponse` error | validation / 헤더 누락 / enum 불일치 |
| `403` | `CncResponse` error | `NotAllowed` — 시리얼이 풀에 없거나 provisional |
| `409` | `CncResponse` error | `AlreadyClaimed` (풀이 점유됨) 또는 `DuplicateRegistration` (활성 device row 존재) |

### 4.2 `GET` /api/v1/devices/{id}

**Operation ID** `getDeviceById`
**Tags** `device`

#### 파라미터

| 위치 | 이름 | 타입 | 필수 |
|---|---|---|---|
| path | `id` | string | yes |

#### 응답

| 상태 | 스키마 |
|---|---|
| `200` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) |
| `404` | `CncResponse` error |

### 4.3 `GET` /api/v1/devices

**Operation ID** `listMyDevices`
**Tags** `device`
**Security** caller headers

caller 가 소유한 device 목록 (REVOKED 포함 — 감사용 보존).

#### 헤더

| 이름 | 필수 | 설명 |
|---|---|---|
| `api-version` | yes | `1.0.0` |
| `X-Caller-Type` | yes | `USER` 또는 `B2B_CENTER` |
| `X-Caller-Id` | yes | caller owner id |

#### 응답

| 상태 | 스키마 |
|---|---|
| `200` | [`CncResponse_DeviceResponseList`](#cncresponse_deviceresponselist) |
| `400` | `CncResponse` error |

### 4.4 `DELETE` /api/v1/devices/{id}

**Operation ID** `unregisterDevice`
**Tags** `device`
**Security** caller headers

caller 본인 device unregister. `status=REVOKED` 로 전이 + `AllowedDevice.unclaim()` 호출 → 동일 시리얼이 다른 사용자에 의해 재등록 가능해짐. Idempotent — 이미 REVOKED 인 device 에 다시 호출해도 200 + 기존 row 반환.

#### 파라미터

| 위치 | 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|---|
| path | `id` | string | yes | device id |
| query | `reason` | string | no | 감사 사유. `device.revocation_reason` 저장 |

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `200` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) | `status=REVOKED` |
| `403` | `CncResponse` error | `NotOwner` |
| `404` | `CncResponse` error | `NotFound` |

### 4.5 `PATCH` /api/v1/devices/{id}/status

**Operation ID** `updateDeviceStatus`
**Tags** `device`
**Security** caller headers

caller 본인 device 의 상태 변경. `REVOKED` 는 더 이상 변경 불가.

#### 파라미터

| 위치 | 이름 | 타입 | 필수 |
|---|---|---|---|
| path | `id` | string | yes |

#### Request body

`application/json` — [`DeviceUpdateStatusRequest`](#deviceupdatestatusrequest)

```json
{
  "status": "ACTIVE"
}
```

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `200` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) | Updated |
| `403` | `CncResponse` error | 비-owner / revoked / unusable 상태 규칙 |
| `404` | `CncResponse` error | device 없음 |

### 4.6 `POST` /api/v1/devices/validate

**Operation ID** `validateDevice`
**Tags** `device`, `validation`
**Security** caller headers

`(macAddress, hardwareSerial)` 가 caller 소유 + `ACTIVE` 인지 검증. 통과 시 `lastUsedAt` / `lastUsedIp` 갱신. v1.0.1 측정 파이프라인은 이 endpoint 대신 `DeviceLookup.LookupByMac` gRPC (mac-only) 를 사용.

#### Request body

`application/json` — [`DeviceValidateRequest`](#devicevalidaterequest)

```json
{
  "macAddress": "aa:bb:cc:dd:ee:ff",
  "hardwareSerial": "CN1G00226KR1700001JP"
}
```

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `200` | [`CncResponse_DeviceValidateResponse`](#cncresponse_devicevalidateresponse) | `allowed=true` |
| `403` | `CncResponse` error | caller 가 owner 아니거나 `ACTIVE` 아님 |
| `404` | `CncResponse` error | device 없음 |

#### 200 예시

```json
{
  "success": true,
  "data": {
    "allowed": true,
    "deviceId": "0a2ad7f0-0e0f-44d8-ae21-b9966f31d999"
  }
}
```

---

## 5. 어드민 — Allowed-Devices API

모든 endpoint 가 `X-Caller-Role: ADMIN` 필요. 없거나 다른 값 → `AllowedDeviceError.Forbidden` (403).

### 5.1 `POST` /api/v1/admin/allowed-devices

**Operation ID** `registerAllowedDevice`
**Tags** `admin`, `allowed-device`

풀에 단건 추가. MAC 은 lowercase 콜론 형식으로 정규화. `hardwareSerial` unique. `TMP-{MAC}` 형식 시리얼을 일부러 보내면 provisional 으로 표시 (어드민이 나중에 `/serial` 로 확정).

#### Request body

`application/json` — [`AllowedDeviceRegisterRequest`](#alloweddeviceregisterrequest)

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `201` | [`CncResponse_AllowedDeviceResponse`](#cncresponse_alloweddeviceresponse) | Created |
| `400` | `CncResponse` error | `InvalidMacFormat` / validation |
| `403` | `CncResponse` error | `Forbidden` |
| `409` | `CncResponse` error | `Duplicate` (mac 또는 serial 충돌) |

### 5.2 `GET` /api/v1/admin/allowed-devices/{id}

풀 단건 조회. 없으면 `404`.

### 5.3 `GET` /api/v1/admin/allowed-devices

풀 목록.

#### 파라미터

| 위치 | 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|---|
| query | `query` | string | no | `mac` / `serial` 부분 일치 |
| query | `claimed` | boolean | no | `true` = 바인딩된 것, `false` = 미바인딩, 생략 = 전체 |
| query | `page` | int | no | 기본 `0` |
| query | `size` | int | no | 기본 `20` |

[`CncResponse_AllowedDevicePageResponse`](#cncresponse_alloweddevicepageresponse) 반환.

### 5.4 `POST` /api/v1/admin/allowed-devices/{id}/unclaim

풀 row 강제 unclaim + 바인딩된 device 가 있으면 REVOKED. 분실 / 교체 운영용. Idempotent.

#### 파라미터

| 위치 | 이름 | 타입 | 필수 | 비고 |
|---|---|---|---|---|
| path | `id` | string | yes | 풀 row id |
| query | `reason` | string | no | 감사 사유 |

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `200` | [`CncResponse_AllowedDeviceResponse`](#cncresponse_alloweddeviceresponse) | `claimed=false` |
| `403` | `CncResponse` error | `Forbidden` |
| `404` | `CncResponse` error | `NotFound` |

### 5.5 `PATCH` /api/v1/admin/allowed-devices/{id}/serial

provisional row (`is_provisional=true`, 보통 `TMP-{MAC}` 형식) 의 진짜 시리얼 확정. 이미 device 가 claim 했으면 그 device 의 `hardware_serial` 도 같이 동기화.

#### Request body

`application/json` — [`ConfirmSerialRequest`](#confirmserialrequest)

```json
{
  "hardwareSerial": "CN1G00226KR1700019JP"
}
```

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `200` | [`CncResponse_AllowedDeviceResponse`](#cncresponse_alloweddeviceresponse) | `is_provisional=false` |
| `400` | `CncResponse` error | `NotProvisional` (이미 확정된 row) |
| `403` | `CncResponse` error | `Forbidden` |
| `404` | `CncResponse` error | `NotFound` |
| `409` | `CncResponse` error | `Duplicate` — 대상 시리얼이 이미 존재 |

### 5.6 `POST` /api/v1/admin/allowed-devices/bulk

xlsx 일괄 업로드. 첫 sheet 파싱, row 당 `register` 1회 호출. header 가 row 1, 데이터는 row 2 부터. 예상 컬럼 (한글). `고유번호`, `MAC주소`, `펌웨어버전`, `제품코드`, `년도`, `생산국가`, `생산주차`, `배송국가`, `일련번호`, `등록일시`.

`제품코드` → `DeviceType` 매핑은 `AdminAllowedDeviceController.mapDeviceType` 의 inline switch. 새 product code 는 거기 추가.

비-atomic — 한 row 실패해도 나머지 진행. duplicate 는 `skipped`, 그 외 실패는 `failed`. 첫 row 부터 `Forbidden` 이면 즉시 중단.

#### Request body

`multipart/form-data`

| Part | 필수 | 비고 |
|---|---|---|
| `file` | yes | xlsx 파일 |

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `200` | [`CncResponse_AllowedDeviceBulkResponse`](#cncresponse_alloweddevicebulkresponse) | row 별 카운터 |
| `400` | `CncResponse` error | xlsx 파싱 실패 |
| `403` | `CncResponse` error | `Forbidden` |

---

## 6. 펌웨어 API

### 6.1 `GET` /api/v1/devices/firmware/upgrade

**Operation ID** `checkFirmwareUpgrade`
**Tags** `firmware`

device 측 polling endpoint. `currentVersion` 이 최신과 다르면 최신 펌웨어 반환. 이미 최신이거나 등록된 최신이 없으면 `204 No Content`.

#### 파라미터

| 위치 | 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|---|
| query | `deviceType` | [`DeviceType`](#devicetype) | yes | device type |
| query | `currentVersion` | string | yes | device 가 보고하는 펌웨어 버전 |

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `200` | [`CncResponse_FirmwareUpgradeResponse`](#cncresponse_firmwareupgraderesponse) | 업그레이드 있음 |
| `204` | body 없음 | 최신 또는 최신 등록 없음 |
| `400` | `CncResponse` error | enum / 파라미터 오류 |

### 6.2 `POST` /api/v1/devices/firmware

**Operation ID** `registerFirmware`
**Tags** `firmware`, `operations`

펌웨어 메타 등록. `makeLatest=true` 면 같은 `deviceType` 의 기존 latest 를 먼저 unmark.

#### Request body

`application/json` — [`FirmwareRegisterRequest`](#firmwareregisterrequest)

```json
{
  "deviceType": "SCALE2",
  "version": "1.1.0",
  "downloadUrl": "https://cdn.example.com/firmware/scale2/1.1.0.bin",
  "makeLatest": true,
  "notes": "Stability improvements",
  "releasedAt": "2026-06-01T00:00:00Z"
}
```

#### 응답

| 상태 | 스키마 | 비고 |
|---|---|---|
| `201` | [`CncResponse_FirmwareResponse`](#cncresponse_firmwareresponse) | Created |
| `400` | `CncResponse` error | validation / enum 불일치 |
| `409` | `CncResponse` error | `(deviceType, version)` 중복 |

### 6.3 `GET` /api/v1/devices/firmware

deviceType 별 펌웨어 목록. `releasedAt` 내림차순.

#### 파라미터

| 위치 | 이름 | 타입 | 필수 |
|---|---|---|---|
| query | `deviceType` | [`DeviceType`](#devicetype) | yes |

[`CncResponse_FirmwareResponseList`](#cncresponse_firmwareresponselist) 반환.

### 6.4 `GET` /api/v1/devices/firmware/{id}

단건 조회. 없으면 `404`.

### 6.5 `PATCH` /api/v1/devices/firmware/{id}/promote

선택한 펌웨어를 해당 `deviceType` 의 latest 로 promote. 기존 latest 는 unmark.

### 6.6 `DELETE` /api/v1/devices/firmware/{id}

펌웨어 row 삭제. 성공 시 `204`.

---

## 7. 내부 gRPC

`device-service` 는 4개 gRPC service 를 `9090` 에 노출. proto 는 `common-libs/common-grpc` (`com.carenco.grpc.device.v1`) 에 있고, 본 release 기준 `0.0.62`.

### 7.1 `DeviceLookup.LookupByMac`

Java. `DeviceLookupGrpcService extends DeviceLookupGrpc.DeviceLookupImplBase`.

`measure-service` (v1.0.1 측정 파이프라인) 가 raw MAC 으로 표준 device record 를 resolve 할 때 사용. MAC 이 풀에 없으면 자동으로 `TMP-{MAC}` provisional row 를 만들고 `allowed=false, rejectReason=NOT_REGISTERED` 응답 → 측정 layer 가 `device_serial=TMP-{MAC}` 로 저장 후 추후 reconciliation.

#### Request

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `mac_address` | string | yes | 내부에서 정규화 |
| `caller_owner_type` | string | yes | `USER` 또는 `B2B_CENTER` |
| `caller_owner_id` | string | yes | owner id |
| `caller_ip` | string | no | 허용 시 `lastUsedIp` 저장 |

#### Response

| 필드 | 타입 | 비고 |
|---|---|---|
| `allowed` | boolean | device 존재 + owner 일치 + `ACTIVE` 인 경우만 `true` |
| `device_id` | string | 성공 시 |
| `hardware_serial` | string | 항상 (실 시리얼 또는 provisional 의 `TMP-{MAC}`) — 측정 추적용 |
| `owner_type` | string | 성공 시 |
| `owner_id` | string | 성공 시 |
| `device_type` | string | `DeviceType` 이름 |
| `auto_provisioned` | boolean | 이번 호출에서 `TMP-` row 를 새로 만들었는지 |
| `reject_reason` | string | 실패 시 `NOT_REGISTERED` / `INVALID_MAC` / `NOT_OWNER` / `DEVICE_INACTIVE` |

### 7.2 `DeviceManagement` (B2B organization device 자산)

Java. `DeviceManagementGrpcService extends DeviceManagementGrpc.DeviceManagementImplBase`.

`b2b-service` 가 organization 의 device 자산을 관리할 때 사용. USER device 등록과는 별도 — `owner_type=B2B_CENTER` + `alias`, `registered_by`, `deactivated_by` 등.

| RPC | 비고 |
|---|---|
| `RegisterDevice` | 풀 기반 등록 (organizationId, serialNumber, alias, registeredBy). model 미입력 시 풀의 model fallback |
| `GetDevice` | 단건 |
| `ListDevices` | organization 별 페이지 + 상태 필터 + 정렬 (`NAME`/`REGISTERED_AT`/`LAST_USED`/`BATTERY` (0.0.61)) |
| `UpdateDevice` | alias 변경 |
| `DeactivateDevice` | 일시 정지 (DEACTIVATED). 가역적 |
| `UnregisterDevice` | **0.0.59 신규.** 영구 해제 (REVOKED + 풀 unclaim). 일시 정지와 다름 — REVOKED 는 terminal |
| `PreviewBySerial` | **0.0.61 신규.** 시리얼만으로 풀 조회 — claim 안 함, write 없음. b2b UI 등록 preview 단계. `{found, already_claimed, hardware_serial, device_type, firmware_version}` 반환 |

0.0.61 에서 `Device` proto 에 추가된 필드. `battery_level` (int32, `-1` = unknown), `battery_reported_at` (ISO-8601, `""` = 없음).

`OrganizationDeviceError` → gRPC `Status` 매핑.

| Error case | gRPC Status |
|---|---|
| `InvalidOrganization`, `InvalidSerial` | `INVALID_ARGUMENT` |
| `NotAllowed`, `NotB2BDevice`, `RevokedForbidden` | `PERMISSION_DENIED` |
| `AlreadyClaimed`, `DuplicateRegistration` | `ALREADY_EXISTS` |
| `NotFound` | `NOT_FOUND` |

### 7.3 `DeviceLifecycle.UnregisterAllByOwner` (0.0.59 신규)

Java. `DeviceLifecycleGrpcService extends DeviceLifecycleGrpc.DeviceLifecycleImplBase`.

`user-service` 의 탈퇴 outbox worker (`UserOutboxDeletionRetryExecutor.unregisterAllDevices`) 가 사용자 hard-delete 직전에 호출. owner 의 모든 device 를 REVOKED + 풀 unclaim. Idempotent — 이미 REVOKED 인 device 는 skip.

#### Request

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `owner_type` | string | yes | `USER` 또는 `B2B_CENTER` — invalid → `INVALID_ARGUMENT` |
| `owner_id` | string | yes | blank → `INVALID_ARGUMENT` |
| `revoked_by` | string | no | 감사 (기본 `"user-service:deletion-worker"`) |
| `reason` | string | no | 감사 (기본 `"user_deletion"`) |

#### Response

| 필드 | 타입 | 비고 |
|---|---|---|
| `unregistered_count` | int | 이번 호출에서 신규로 REVOKED 전이된 device 수 |
| `active_before` | int | 전이 전 owner 가 가지고 있던 active device 수 |

### 7.4 `DeviceMetrics.ReportMetrics` (0.0.62 신규, battery + firmware 통합)

Java. `DeviceMetricsGrpcService extends DeviceMetricsGrpc.DeviceMetricsImplBase`.

`measure-service` 가 footprint / vision record 저장 직후 fire-and-forget 호출 (`RecordServiceImpl.createFootprint` / `createByVision`). `device.battery_level` / `battery_reported_at` / `firmware_version` / `firmware_reported_at` + `allowed_devices.current_firmware_version` 을 동시 갱신. 실패 (NOT_FOUND, INVALID_ARGUMENT, deadline exceeded) 는 caller 측에서 삼킴 → 측정 응답 지연 영향 0.

`battery_level=-1` 또는 `firmware_version=""` 시 해당 필드 no-op (기존 값 유지). 사용자가 firmware 안 보내고 battery 만 보고하거나 그 반대 가능.

#### Request — `ReportMetricsRequest`

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `device_id` | string | yes | 없으면 NOT_FOUND |
| `battery_level` | int32 | no | 0-100. `-1` = 미보고 (기존 값/timestamp 유지). 범위 밖 → INVALID_ARGUMENT |
| `firmware_version` | string | no | 빈 문자열 = 미보고 |
| `reported_at` | string | no | ISO-8601 (UTC). 빈 문자열 → server now |
| `reported_by` | string | no | 감사 (예. `measure-service:record`) |

#### Response — `ReportMetricsResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `battery_level` | int32 | 저장된 현재 값 (no-op 시 기존 값) |
| `battery_reported_at` | string | ISO-8601 |
| `firmware_version` | string | 저장된 현재 값 |
| `firmware_reported_at` | string | ISO-8601 |

#### 기존 `ReportBattery` (0.0.61)

호환을 위해 유지되지만 0.0.62 부터는 `ReportMetrics` 사용 권장 (battery + firmware 한 호출에 통합).

dev 검증 (2026-06-01). newman `device-mac-lookup-flow` 1회 footprint 측정 → `DeviceApplicationService.reportMetrics args=[deviceId, 99, null, ...]` 성공 33ms. battery=99 정상 갱신, firmware=null 이라 firmware 컬럼은 no-op.

### 7.5 제거된 gRPC

- `DeviceValidator.Validate` — common-libs `0.0.58` 에서 제거. 기능 분리. 측정은 `DeviceLookup.LookupByMac`, B2B 는 `DeviceManagement.GetDevice` + 상태 체크.

---

## 8. 데이터 모델

### 8.1 `allowed_devices` (출고 기기 풀)

| 컬럼 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `id` | `VARCHAR(36)` | yes | PK. `gen_random_uuid()` default |
| `version` | `BIGINT` | yes | JPA optimistic lock |
| `mac_address` | `VARCHAR(17)` | yes | 정규화된 lowercase 콜론 |
| `hardware_serial` | `VARCHAR(64)` | yes | 출고 시리얼 (예. `CN1G00226KR1700001JP`). provisional 동안 `TMP-{MAC}` |
| `device_type` | `VARCHAR(30)` | yes | [`DeviceType`](#devicetype) |
| `firmware_version` | `VARCHAR(64)` | no | **출고 시 펌웨어 (factory, immutable)**. OTA 후에도 변경 X |
| `current_firmware_version` | `VARCHAR(64)` | no | **OTA 갱신 시 최신 펌웨어** (V11, 0.0.62). `ReportMetrics` 시 동기 |
| `model` | `VARCHAR(64)` | no | 출고 시 모델명 (V11, 0.0.62). 사용자 register 시 model 미입력하면 fallback |
| `product_code` | `VARCHAR(32)` | no | xlsx `제품코드` (예. `CN1G002`) |
| `production_country` | `VARCHAR(8)` | no | ISO-2 |
| `production_week` | `INTEGER` | no | xlsx `생산주차` |
| `production_year` | `INTEGER` | no | 2자리 (예. `26` → 2026) |
| `serial_number` | `INTEGER` | no | 주차 내 카운터 |
| `shipping_country` | `VARCHAR(8)` | no | ISO-2 |
| `manufactured_at` | `TIMESTAMP` | no | 공장 등록 시각 |
| `claimed` | `BOOLEAN` | yes | 사용자 등록 시 `true`, unregister 시 `false` |
| `claimed_by_device_id` | `VARCHAR(36)` | no | weak FK to `devices.id` |
| `claimed_at` | `TIMESTAMP` | no | 첫 claim 시각 |
| `is_provisional` | `BOOLEAN` | yes | `true` 동안 시리얼이 `TMP-{MAC}` (DeviceLookup auto-provision) |
| `created_at`, `updated_at`, `deleted_at` | `TIMESTAMP` | no | `BaseTimeEntity` |

인덱스 / 제약.

| 이름 | 컬럼 |
|---|---|
| `pk_allowed_devices` | `id` |
| `uk_allowed_devices_mac_serial` | `mac_address`, `hardware_serial` |
| `uk_allowed_devices_serial` | `hardware_serial` |
| `uk_allowed_devices_mac` | `mac_address` |
| `idx_allowed_devices_claimed` | `claimed` |

Seed. V5 Flyway 가 legacy `mac_database.xlsx` 의 35 row seed. (V5 첫 row 누락 anomaly 는 follow-up I.)

### 8.2 `devices`

| 컬럼 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `id` | `VARCHAR(36)` | yes | PK |
| `version` | `BIGINT` | yes | JPA optimistic lock |
| `mac_address` | `VARCHAR(17)` | yes | 정규화된 MAC. 등록 시 풀에서 복사 |
| `hardware_serial` | `VARCHAR(64)` | yes | 출고 시리얼. 등록 시 풀에서 복사 |
| `device_type` | `VARCHAR(30)` | yes | [`DeviceType`](#devicetype). 풀에서 복사 |
| `owner_type` | `VARCHAR(20)` | yes | [`OwnerType`](#ownertype) |
| `owner_id` | `VARCHAR(36)` | yes | cross-MSA id. FK 없음 |
| `organization_id` | `VARCHAR(36)` | no | B2B 등록 시 set |
| `alias` | `VARCHAR(64)` | no | B2B alias |
| `status` | `VARCHAR(20)` | yes | [`DeviceStatus`](#devicestatus) |
| `model` | `VARCHAR(64)` | no | free text. 미입력 시 `allowed_devices.model` fallback |
| `firmware_version` | `VARCHAR(64)` | no | **현재 펌웨어** (OTA 시 갱신). `ReportMetrics` 가 갱신 |
| `registered_by` | `VARCHAR(64)` | no | B2B 감사 |
| `deactivated_at` | `TIMESTAMP` | no | DEACTIVATED 전이 시 |
| `deactivated_by` | `VARCHAR(64)` | no | 감사 |
| `deactivation_reason` | `VARCHAR(255)` | no | 감사 |
| `last_used_at` | `TIMESTAMP` | no | validation / lookup 통과 시 |
| `last_used_ip` | `VARCHAR(45)` | no | IPv4/IPv6 |
| `battery_level` | `INTEGER` | no | 0-100. `ReportMetrics` 가 갱신 (V10) |
| `battery_reported_at` | `TIMESTAMP` | no | 마지막 battery 보고 시각 (V10) |
| `firmware_reported_at` | `TIMESTAMP` | no | 마지막 firmware 보고 시각 (V11, 0.0.62) |
| `created_at`, `updated_at`, `deleted_at` | `TIMESTAMP` | no | `BaseTimeEntity` |

인덱스 / 제약.

| 이름 | 컬럼 | 비고 |
|---|---|---|
| `pk_devices` | `id` | — |
| `uk_devices_mac_serial_active` | `mac_address`, `hardware_serial` | **partial unique** `WHERE status != 'REVOKED'` (V9). 같은 `(mac, serial)` 가 revoke 후 재등록 가능 |
| `idx_devices_owner` | `owner_type`, `owner_id` | — |
| `idx_devices_mac` | `mac_address` | — |
| `idx_devices_type` | `device_type` | — |
| `idx_devices_organization` | `organization_id` | B2B lookup |
| `idx_devices_battery_level` | `battery_level` | battery 정렬 (V10) |

### 8.3 `device_firmwares`

| 컬럼 | 타입 | 필수 | 비고 |
|---|---|---|---|
| `id` | `VARCHAR(36)` | yes | PK |
| `version_lock` | `BIGINT` | yes | JPA optimistic lock |
| `device_type` | `VARCHAR(30)` | yes | [`DeviceType`](#devicetype) |
| `version` | `VARCHAR(64)` | yes | Semver / 날짜 / free text |
| `download_url` | `VARCHAR(2000)` | yes | 펌웨어 바이너리 URL |
| `is_latest` | `BOOLEAN` | yes | 기본 `false` |
| `notes` | `VARCHAR(500)` | no | release note |
| `released_at` | `TIMESTAMP` | no | service 에서 `Instant.now()` default |
| `created_at`, `updated_at`, `deleted_at` | `TIMESTAMP` | no | `BaseTimeEntity` |

인덱스 / 제약.

| 이름 | 컬럼 |
|---|---|
| `pk_device_firmwares` | `id` |
| `uk_firmwares_type_version` | `device_type`, `version` |
| `idx_firmwares_type` | `device_type` |
| `uk_firmwares_one_latest_per_type` | `device_type` 의 partial unique `WHERE is_latest = TRUE` |

---

## 9. 스키마

### `DeviceRegisterRequest`

| 필드 | 타입 | 필수 | validation |
|---|---|---|---|
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |
| `model` | string | no | `@Size(max=64)`. 미입력 시 `allowed_devices.model` fallback |
| `firmwareVersion` | string | no | `@Size(max=64)`. 미입력 시 `allowed_devices.firmware_version` fallback |

> `macAddress` / `deviceType` 은 더 이상 받지 않음 — 풀이 권위.

### `DeviceUpdateStatusRequest`

| 필드 | 타입 | 필수 |
|---|---|---|
| `status` | [`DeviceStatus`](#devicestatus) | yes |

### `DeviceValidateRequest`

| 필드 | 타입 | 필수 | validation |
|---|---|---|---|
| `macAddress` | string | yes | `@NotBlank`, `@Size(max=17)`, 도메인 MAC 형식 validation |
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |

### `DeviceResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `id` | string | UUID |
| `macAddress` | string | 정규화된 lowercase 콜론 |
| `hardwareSerial` | string | 시리얼 |
| `deviceType` | [`DeviceType`](#devicetype) | — |
| `ownerType` | [`OwnerType`](#ownertype) | — |
| `ownerId` | string | UUID |
| `status` | [`DeviceStatus`](#devicestatus) | — |
| `model` | string | nullable |
| `firmwareVersion` | string | nullable. 현재 펌웨어 |
| `lastUsedAt` | string date-time | nullable |
| `lastUsedIp` | string | nullable |
| `createdAt` | string date-time | nullable |
| `updatedAt` | string date-time | nullable |

### `DeviceValidateResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `allowed` | boolean | REST 성공 시 항상 `true` |
| `deviceId` | string | matched device id |

### `AllowedDeviceRegisterRequest`

| 필드 | 타입 | 필수 | validation |
|---|---|---|---|
| `macAddress` | string | yes | `@NotBlank`, MAC 형식 |
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |
| `deviceType` | [`DeviceType`](#devicetype) | yes | — |
| `firmwareVersion` | string | no | — |
| `productCode` | string | no | xlsx `제품코드` |
| `productionCountry` | string | no | ISO-2 |
| `productionWeek` | integer | no | — |
| `productionYear` | integer | no | 2자리 |
| `serialNumber` | integer | no | 주차 내 카운터 |
| `shippingCountry` | string | no | ISO-2 |
| `manufacturedAt` | string date-time | no | — |

### `ConfirmSerialRequest`

| 필드 | 타입 | 필수 | validation |
|---|---|---|---|
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |

### `AllowedDeviceResponse`

`allowed_devices` 컬럼 + `claimed`, `claimedByDeviceId`, `claimedAt`, `isProvisional` 미러.

### `AllowedDevicePageResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `content` | [`AllowedDeviceResponse`](#alloweddeviceresponse)[] | page content |
| `page` | int | 현재 페이지 |
| `size` | int | 페이지 크기 |
| `totalElements` | long | 총 row 수 |
| `totalPages` | int | 총 페이지 수 |
| `hasNext` | boolean | — |

### `AllowedDeviceBulkResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `added` | int | 신규 insert row 수 |
| `skippedCount` | int | duplicate |
| `failedCount` | int | 그 외 실패 |
| `skipped` | `RowResult[]` | `{ row, hardwareSerial, macAddress, reason }` |
| `failed` | `RowResult[]` | 동일 shape |

### `FirmwareRegisterRequest`

| 필드 | 타입 | 필수 | validation |
|---|---|---|---|
| `deviceType` | [`DeviceType`](#devicetype) | yes | `@NotNull` |
| `version` | string | yes | `@NotBlank`, `@Size(max=64)` |
| `downloadUrl` | string | yes | `@NotBlank`, `@Size(max=2000)` |
| `makeLatest` | boolean | yes | primitive. JSON mapper 가 생략하면 `false` |
| `notes` | string | no | `@Size(max=500)` |
| `releasedAt` | string date-time | no | null 시 now default |

### `FirmwareResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `id` | string | UUID |
| `deviceType` | [`DeviceType`](#devicetype) | — |
| `version` | string | — |
| `downloadUrl` | string | — |
| `isLatest` | boolean | record component 의 JSON property |
| `notes` | string | nullable |
| `releasedAt` | string date-time | nullable |
| `createdAt` | string date-time | nullable |
| `updatedAt` | string date-time | nullable |

### `FirmwareUpgradeResponse`

| 필드 | 타입 | 비고 |
|---|---|---|
| `latestVersion` | string | 최신 버전 |
| `downloadUrl` | string | 펌웨어 바이너리 URL |

### `DeviceType`

| 값 | 비고 |
|---|---|
| `SCALE2` | — |
| `SCALE_PUZZLE` | — |

### `OwnerType`

| 값 | 의미 |
|---|---|
| `USER` | 개인 사용자 (`user-service.user.id`) |
| `B2B_CENTER` | B2B 센터 / organization owner id |

### `DeviceStatus`

| 값 | 의미 | validate 사용 가능? |
|---|---|---|
| `REGISTERED` | 등록만 됨. 비활성 | no |
| `ACTIVE` | 사용 가능 | yes |
| `DEACTIVATED` | 일시 정지 — 가역적 | no |
| `REVOKED` | 영구 해제. row 는 감사 보존, 풀 unclaim 은 동시에 발생 | no |

### CncResponse envelope 들

| 스키마 | data shape |
|---|---|
| `CncResponse_DeviceResponse` | [`DeviceResponse`](#deviceresponse) |
| `CncResponse_DeviceResponseList` | [`DeviceResponse`](#deviceresponse)[] |
| `CncResponse_DeviceValidateResponse` | [`DeviceValidateResponse`](#devicevalidateresponse) |
| `CncResponse_AllowedDeviceResponse` | [`AllowedDeviceResponse`](#alloweddeviceresponse) |
| `CncResponse_AllowedDevicePageResponse` | [`AllowedDevicePageResponse`](#alloweddevicepageresponse) |
| `CncResponse_AllowedDeviceBulkResponse` | [`AllowedDeviceBulkResponse`](#alloweddevicebulkresponse) |
| `CncResponse_FirmwareResponse` | [`FirmwareResponse`](#firmwareresponse) |
| `CncResponse_FirmwareResponseList` | [`FirmwareResponse`](#firmwareresponse)[] |
| `CncResponse_FirmwareUpgradeResponse` | [`FirmwareUpgradeResponse`](#firmwareupgraderesponse) |

모든 envelope shape. `{ success, data, token, error, timestamp }`.

---

## 10. 메모

- 출고 풀 (`allowed_devices`) 이 `(mac, deviceType, firmwareVersion, model)` 의 권위. 사용자 등록은 `hardwareSerial` 만 보낸다.
- MAC 정규화 (lowercase 콜론) 는 domain service 에서 — persist + 풀 lookup 전에 적용.
- `DELETE /api/v1/devices/{id}` 는 idempotent — 이미 REVOKED 인 device 에 다시 호출해도 200 + 기존 row, 풀 추가 side-effect 없음.
- 풀의 partial unique (`status != 'REVOKED'`) 가 같은 `(mac, serial)` 의 재등록을 허용 (이전 owner 가 unregister 한 후).
- provisional 흐름. 모르는 MAC 으로 `DeviceLookup.LookupByMac` 호출이 오면 `hardware_serial=TMP-{MAC}` + `is_provisional=true` 인 풀 row 를 만들고 `NOT_REGISTERED` 로 reject. 측정 layer 는 그래도 `device_serial=TMP-{MAC}` 저장 → 어드민이 `PATCH /admin/allowed-devices/{id}/serial` 로 확정 가능.
- `user-service` 의 탈퇴 outbox worker 가 activities / records 정리와 OAuth / hard-delete 사이 단계에서 `DeviceLifecycle.UnregisterAllByOwner` 를 호출. 실패 시 exponential backoff 로 최대 2회 retry.
- gRPC client 는 `etc.<svc>-grpc=device.carenco.svc.cluster.local:9090` config + deadline interceptor (기본 5000 ms) 사용.
- 펌웨어 `makeLatest` / `promote` 는 같은 `deviceType` 의 기존 latest 를 unmark.
- `DELETE /firmware/{id}` 는 repository delete. `BaseTimeEntity` 의 physical vs soft-delete 동작을 common-core JPA 구현 기준으로 확인 후 retention 의도 결정.
- `ReportMetrics` 는 fire-and-forget. measure-service 가 매 측정마다 호출. battery 또는 firmware 한쪽만 보내면 다른 쪽 컬럼은 그대로 유지. client (앱) 가 firmware 를 보내기 시작하면 자동으로 `device.firmware_version` + `allowed_devices.current_firmware_version` 갱신.
