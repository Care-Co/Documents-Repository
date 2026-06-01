# device-service

> Updated: 2026-06-01 (proto v0.0.61)
> Style: OpenAPI 3.1 + code-backed endpoint catalog + schema tables
> Base path: `/api/v1/devices`, `/api/v1/admin/allowed-devices`
> Response envelope: `CncResponse` (common-core / Pattern B)

Source: `/Users/jonghak/GitHub/Care&Co/device-service`

---

## 1. Overview

`device-service` owns the shipped-device pool, user/B2B device registration scoped to that pool, owner-scoped device validation, lifecycle status changes, OTA firmware metadata, and cluster-internal MAC lookup for the measurement pipeline.

### 1.1 Domain responsibilities

| Domain | Responsibility |
|---|---|
| `allowed_device` | Shipped-device pool. Admin adds rows (single or xlsx bulk); register flow consumes them; `claim` / `unclaim` toggle is the binding to a real `device` row |
| `device` | Register physical devices to an owner using a pool serial, validate active ownership, change status, unregister (REVOKED + pool unclaim) |
| `firmware` | Store firmware by `deviceType + version`, mark one latest firmware per device type, serve upgrade checks |
| `grpc` | Four internal gRPC services — `DeviceLookup` (mac → serial for measure-service), `DeviceManagement` (B2B organization device asset lifecycle + serial preview), `DeviceLifecycle` (owner-scoped bulk unregister for user-service deletion worker), `DeviceMetrics` (battery report from measure-service after each record save) |

### 1.2 Runtime

| Item | Value |
|---|---|
| Java / Spring | Java 25, Spring Boot 4.0.5 |
| Persistence | PostgreSQL 17 + Flyway (`spring-boot-starter-flyway`) |
| HTTP port | `8080` |
| gRPC port | `9090` |
| Discovery/config | Spring Cloud Config via `CONFIG_URI`, profile `dev-k3s` / `prod-k3s` |
| Kubernetes service | `device.carenco.svc.cluster.local` (`http:8080`, `grpc:9090`) |
| Prometheus | `/actuator/prometheus` |

---

## 2. API Conventions

### 2.1 Common headers

| Header | Value | Required | Notes |
|---|---|---|---|
| `api-version` | `1.0.0` | yes | Spring header version selector |
| `Content-Type` | `application/json` | request body only | REST JSON endpoints |
| `X-Caller-Type` | `USER` \| `B2B_CENTER` | device owner endpoints | Injected by gateway/auth layer |
| `X-Caller-Id` | UUID string | device owner endpoints | `user.id` or B2B center id |
| `X-Caller-Role` | `ADMIN` (optional) | admin endpoints | Gateway-injected; non-`ADMIN` → `AllowedDeviceError.Forbidden` |

### 2.2 Response envelope

Success responses use common-core `CncResponse`.

```json
{
  "success": true,
  "data": {},
  "token": null,
  "error": null,
  "timestamp": "2026-05-29T01:00:00Z"
}
```

Binding errors that would otherwise fall through are mapped to `400`:

```json
{
  "success": false,
  "data": {
    "message": "Missing required header",
    "error": "Header 'X-Caller-Type' is required"
  }
}
```

### 2.3 Security notes

- Controllers currently do not declare `@PreAuthorize`; caller identity is passed through headers for owner-scoped device endpoints.
- Admin endpoints check `X-Caller-Role == "ADMIN"` inside the service (`AllowedDeviceService.requireAdmin`). Production gateways must populate this header from JWT roles.
- Firmware management endpoints are marked as operating endpoints in code; fine-grained auth is noted as follow-up work.
- Internal MSAs should call the gRPC adapters (`DeviceLookup`, `DeviceManagement`, `DeviceLifecycle`) instead of REST.

---

## 3. Endpoint Catalog

### 3.1 Device (user / B2C owner)

| # | Method | Path | Purpose | Caller headers |
|---|---|---|---|---|
| 1 | `POST` | `/api/v1/devices` | Register caller-owned device using a pool serial | required |
| 2 | `GET` | `/api/v1/devices/{id}` | Get device by id | no |
| 3 | `GET` | `/api/v1/devices` | List caller-owned devices | required |
| 4 | `DELETE` | `/api/v1/devices/{id}?reason=...` | Unregister caller-owned device (REVOKED + pool unclaim, idempotent) | required |
| 5 | `PATCH` | `/api/v1/devices/{id}/status` | Change device status | required |
| 6 | `POST` | `/api/v1/devices/validate` | Validate caller can use device | required |

### 3.2 Admin allowed-devices (shipped pool)

| # | Method | Path | Purpose |
|---|---|---|---|
| 7 | `POST` | `/api/v1/admin/allowed-devices` | Add single pool row |
| 8 | `GET` | `/api/v1/admin/allowed-devices/{id}` | Get pool row by id |
| 9 | `GET` | `/api/v1/admin/allowed-devices?query&claimed&page&size` | List pool rows with filter |
| 10 | `POST` | `/api/v1/admin/allowed-devices/{id}/unclaim` | Force-unclaim + revoke the bound device (loss/swap ops) |
| 11 | `PATCH` | `/api/v1/admin/allowed-devices/{id}/serial` | Confirm real serial for a provisional (`TMP-...`) row |
| 12 | `POST` | `/api/v1/admin/allowed-devices/bulk` | xlsx bulk upload (multipart/form-data) |

### 3.3 Firmware

| # | Method | Path | Purpose |
|---|---|---|---|
| 13 | `GET` | `/api/v1/devices/firmware/upgrade` | Check latest firmware for a device |
| 14 | `POST` | `/api/v1/devices/firmware` | Register firmware |
| 15 | `GET` | `/api/v1/devices/firmware` | List firmware by device type |
| 16 | `GET` | `/api/v1/devices/firmware/{id}` | Get firmware by id |
| 17 | `PATCH` | `/api/v1/devices/firmware/{id}/promote` | Mark firmware as latest |
| 18 | `DELETE` | `/api/v1/devices/firmware/{id}` | Delete firmware |

---

## 4. Device API

### 4.1 `POST` /api/v1/devices

**Operation ID** `registerDevice`
**Tags** `device`
**Security** caller headers

Register a device to the caller using a serial that exists in the shipped-device pool. The pool is authoritative for `mac`, `deviceType`, and `firmwareVersion` — the client does not send them. New devices start with status `REGISTERED`.

Flow:
1. Look up the pool row by `hardwareSerial`. Missing or provisional (`TMP-` prefix) → `DeviceError.NotAllowed`.
2. If already claimed by another user → `DeviceError.AlreadyClaimed`.
3. If a non-revoked `devices` row with the same `(mac, serial)` already exists → `DeviceError.DuplicateRegistration`.
4. Persist the new device. The pool row gets `claim(deviceId)`.

#### Headers

| Name | Required | Description |
|---|---|---|
| `api-version` | yes | `1.0.0` |
| `X-Caller-Type` | yes | `USER` or `B2B_CENTER` |
| `X-Caller-Id` | yes | caller owner id |

#### Request body

`application/json` — [`DeviceRegisterRequest`](#deviceregisterrequest)

```json
{
  "hardwareSerial": "CN1G00226KR1700001JP",
  "model": "Scale2 Pro",
  "firmwareVersion": "1.0.0"
}
```

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `201` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) | Created |
| `400` | `CncResponse` error | validation / missing headers / enum mismatch |
| `403` | `CncResponse` error | `NotAllowed` — serial not in pool / provisional |
| `409` | `CncResponse` error | `AlreadyClaimed` (pool taken) or `DuplicateRegistration` (active device row exists) |

### 4.2 `GET` /api/v1/devices/{id}

**Operation ID** `getDeviceById`
**Tags** `device`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `id` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| `200` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) |
| `404` | `CncResponse` error |

### 4.3 `GET` /api/v1/devices

**Operation ID** `listMyDevices`
**Tags** `device`
**Security** caller headers

List devices owned by the caller (including `REVOKED` rows — they persist for audit).

#### Headers

| Name | Required | Description |
|---|---|---|
| `api-version` | yes | `1.0.0` |
| `X-Caller-Type` | yes | `USER` or `B2B_CENTER` |
| `X-Caller-Id` | yes | caller owner id |

#### Responses

| Status | Schema |
|---|---|
| `200` | [`CncResponse_DeviceResponseList`](#cncresponse_deviceresponselist) |
| `400` | `CncResponse` error |

### 4.4 `DELETE` /api/v1/devices/{id}

**Operation ID** `unregisterDevice`
**Tags** `device`
**Security** caller headers

Unregister a caller-owned device. Sets `status=REVOKED` and calls `AllowedDevice.unclaim()` on the pool row, making the same serial re-registerable by another user. Idempotent — calling again on an already-revoked device returns 200 with the existing row.

#### Parameters

| In | Name | Type | Required | Notes |
|---|---|---|---|---|
| path | `id` | string | yes | device id |
| query | `reason` | string | no | Audit reason; stored on `device.revocation_reason` |

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `200` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) | `status=REVOKED` |
| `403` | `CncResponse` error | `NotOwner` — caller is not the device owner |
| `404` | `CncResponse` error | `NotFound` |

### 4.5 `PATCH` /api/v1/devices/{id}/status

**Operation ID** `updateDeviceStatus`
**Tags** `device`
**Security** caller headers

Change a caller-owned device status. `REVOKED` devices cannot be changed again.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `id` | string | yes |

#### Request body

`application/json` — [`DeviceUpdateStatusRequest`](#deviceupdatestatusrequest)

```json
{
  "status": "ACTIVE"
}
```

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `200` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) | Updated |
| `403` | `CncResponse` error | not owner, revoked, unusable status rule |
| `404` | `CncResponse` error | device not found |

### 4.6 `POST` /api/v1/devices/validate

**Operation ID** `validateDevice`
**Tags** `device`, `validation`
**Security** caller headers

Validate that `(macAddress, hardwareSerial)` belongs to the caller and has status `ACTIVE`. On success, the service updates `lastUsedAt` and `lastUsedIp`. The v1.0.1 measurement pipeline prefers `DeviceLookup.LookupByMac` gRPC (mac-only) over this endpoint.

#### Request body

`application/json` — [`DeviceValidateRequest`](#devicevalidaterequest)

```json
{
  "macAddress": "aa:bb:cc:dd:ee:ff",
  "hardwareSerial": "CN1G00226KR1700001JP"
}
```

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `200` | [`CncResponse_DeviceValidateResponse`](#cncresponse_devicevalidateresponse) | `allowed=true` |
| `403` | `CncResponse` error | caller is not owner or status is not `ACTIVE` |
| `404` | `CncResponse` error | device not found |

#### 200 example

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

## 5. Admin Allowed-Devices API

All endpoints require `X-Caller-Role: ADMIN`. Missing or non-`ADMIN` → `AllowedDeviceError.Forbidden` (403).

### 5.1 `POST` /api/v1/admin/allowed-devices

**Operation ID** `registerAllowedDevice`
**Tags** `admin`, `allowed-device`

Add a single row to the shipped pool. MAC is normalized to lowercase colon format. `hardwareSerial` must be unique; passing a `TMP-{MAC}` style serial intentionally marks the row provisional (admin confirms later via `/serial`).

#### Request body

`application/json` — [`AllowedDeviceRegisterRequest`](#alloweddeviceregisterrequest)

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `201` | [`CncResponse_AllowedDeviceResponse`](#cncresponse_alloweddeviceresponse) | Created |
| `400` | `CncResponse` error | `InvalidMacFormat` / validation |
| `403` | `CncResponse` error | `Forbidden` |
| `409` | `CncResponse` error | `Duplicate` (mac or serial collision) |

### 5.2 `GET` /api/v1/admin/allowed-devices/{id}

Get pool row by id. `404` when missing.

### 5.3 `GET` /api/v1/admin/allowed-devices

List pool rows.

#### Parameters

| In | Name | Type | Required | Notes |
|---|---|---|---|---|
| query | `query` | string | no | substring match on `mac` / `serial` |
| query | `claimed` | boolean | no | `true` = bound to a device, `false` = available, omit = all |
| query | `page` | int | no | default `0` |
| query | `size` | int | no | default `20` |

Returns [`CncResponse_AllowedDevicePageResponse`](#cncresponse_alloweddevicepageresponse).

### 5.4 `POST` /api/v1/admin/allowed-devices/{id}/unclaim

Force-unclaim a pool row and revoke the bound `device` if any. Used by ops for loss/swap. Idempotent.

#### Parameters

| In | Name | Type | Required | Notes |
|---|---|---|---|---|
| path | `id` | string | yes | pool row id |
| query | `reason` | string | no | audit reason |

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `200` | [`CncResponse_AllowedDeviceResponse`](#cncresponse_alloweddeviceresponse) | row with `claimed=false` |
| `403` | `CncResponse` error | `Forbidden` |
| `404` | `CncResponse` error | `NotFound` |

### 5.5 `PATCH` /api/v1/admin/allowed-devices/{id}/serial

Confirm the real serial for a provisional row (`is_provisional=true`, typically `TMP-{MAC}` shape). If the row is already claimed by a device, that device's `hardware_serial` is synced as well.

#### Request body

`application/json` — [`ConfirmSerialRequest`](#confirmserialrequest)

```json
{
  "hardwareSerial": "CN1G00226KR1700019JP"
}
```

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `200` | [`CncResponse_AllowedDeviceResponse`](#cncresponse_alloweddeviceresponse) | `is_provisional=false` |
| `400` | `CncResponse` error | `NotProvisional` (row is already confirmed) |
| `403` | `CncResponse` error | `Forbidden` |
| `404` | `CncResponse` error | `NotFound` |
| `409` | `CncResponse` error | `Duplicate` — target serial already exists |

### 5.6 `POST` /api/v1/admin/allowed-devices/bulk

xlsx bulk upload. The first sheet is parsed; one `register` per data row. Header row at row 1, data from row 2. Expected columns (Korean): `고유번호`, `MAC주소`, `펌웨어버전`, `제품코드`, `년도`, `생산국가`, `생산주차`, `배송국가`, `일련번호`, `등록일시`.

`제품코드` is mapped to `DeviceType` inline (`CN1G002` → `SCALE2`). New product codes must be added to `AdminAllowedDeviceController.mapDeviceType`.

Non-atomic by default — one row failing does not abort the rest. Duplicates are reported in `skipped`; other failures in `failed`. `Forbidden` from the first row aborts.

#### Request body

`multipart/form-data`

| Part | Required | Notes |
|---|---|---|
| `file` | yes | xlsx file |

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `200` | [`CncResponse_AllowedDeviceBulkResponse`](#cncresponse_alloweddevicebulkresponse) | per-row counters |
| `400` | `CncResponse` error | xlsx parse failure |
| `403` | `CncResponse` error | `Forbidden` |

---

## 6. Firmware API

### 6.1 `GET` /api/v1/devices/firmware/upgrade

**Operation ID** `checkFirmwareUpgrade`
**Tags** `firmware`

Device-side polling endpoint. Returns the current latest firmware for the device type when `currentVersion` differs from latest. Returns `204 No Content` when already latest or no latest firmware exists.

#### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| query | `deviceType` | [`DeviceType`](#devicetype) | yes | Device type |
| query | `currentVersion` | string | yes | Firmware version reported by device |

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `200` | [`CncResponse_FirmwareUpgradeResponse`](#cncresponse_firmwareupgraderesponse) | Upgrade available |
| `204` | no body | Latest or no registered latest firmware |
| `400` | `CncResponse` error | invalid enum / missing parameter |

### 6.2 `POST` /api/v1/devices/firmware

**Operation ID** `registerFirmware`
**Tags** `firmware`, `operations`

Register firmware metadata. If `makeLatest=true`, the previous latest firmware for the same `deviceType` is unmarked first.

#### Request body

`application/json` — [`FirmwareRegisterRequest`](#firmwareregisterrequest)

```json
{
  "deviceType": "SCALE2",
  "version": "1.1.0",
  "downloadUrl": "https://cdn.example.com/firmware/scale2/1.1.0.bin",
  "makeLatest": true,
  "notes": "Stability improvements",
  "releasedAt": "2026-05-29T00:00:00Z"
}
```

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `201` | [`CncResponse_FirmwareResponse`](#cncresponse_firmwareresponse) | Created |
| `400` | `CncResponse` error | validation / enum mismatch |
| `409` | `CncResponse` error | duplicate `(deviceType, version)` |

### 6.3 `GET` /api/v1/devices/firmware

List firmware records for a device type, sorted by `releasedAt` descending.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `deviceType` | [`DeviceType`](#devicetype) | yes |

Returns [`CncResponse_FirmwareResponseList`](#cncresponse_firmwareresponselist).

### 6.4 `GET` /api/v1/devices/firmware/{id}

Get firmware by id. `404` when missing.

### 6.5 `PATCH` /api/v1/devices/firmware/{id}/promote

Promote a firmware row to latest for its `deviceType`. Existing latest for the same type is unmarked.

### 6.6 `DELETE` /api/v1/devices/firmware/{id}

Delete a firmware row. `204` on success.

---

## 7. Internal gRPC

`device-service` exposes three gRPC services on port `9090`. Protos live in `common-libs/common-grpc` (`com.carenco.grpc.device.v1`) and are bumped at `0.0.59` (this release).

### 7.1 `DeviceLookup.LookupByMac`

Java: `DeviceLookupGrpcService extends DeviceLookupGrpc.DeviceLookupImplBase`.

Cluster-internal endpoint used by `measure-service` (v1.0.1 measurement pipeline) to resolve a raw MAC into the canonical device record. If the MAC is not in the pool, the service auto-provisions a `TMP-{MAC}` row (provisional) and replies `allowed=false` with `rejectReason=NOT_REGISTERED` so that the measurement layer can store `device_serial=TMP-{MAC}` for later reconciliation.

#### Request

| Field | Type | Required | Notes |
|---|---|---|---|
| `mac_address` | string | yes | Normalized internally |
| `caller_owner_type` | string | yes | `USER` or `B2B_CENTER` |
| `caller_owner_id` | string | yes | Owner id |
| `caller_ip` | string | no | Stored as `lastUsedIp` when allowed |

#### Response

| Field | Type | Notes |
|---|---|---|
| `allowed` | boolean | `true` only when device exists, owner matches, and status is `ACTIVE` |
| `device_id` | string | set on success |
| `hardware_serial` | string | always set (real or `TMP-{MAC}` for provisional) — measurement layer stores this for traceability |
| `owner_type` | string | set on success |
| `owner_id` | string | set on success |
| `device_type` | string | `DeviceType` name; set on success |
| `auto_provisioned` | boolean | `true` when the service just created a `TMP-` pool row in this call |
| `reject_reason` | string | one of `NOT_REGISTERED`, `INVALID_MAC`, `NOT_OWNER`, `DEVICE_INACTIVE`; set on failure |

### 7.2 `DeviceManagement` (B2B organization device asset)

Java: `DeviceManagementGrpcService extends DeviceManagementGrpc.DeviceManagementImplBase`.

Used by `b2b-service` to manage device assets attached to organizations. Distinct from user (`USER`) device registration — these devices have `owner_type=B2B_CENTER` and carry `alias`, `registered_by`, `deactivated_by`, etc.

| RPC | Notes |
|---|---|
| `RegisterDevice` | Pool-backed register (organizationId, serialNumber, alias, registeredBy) |
| `GetDevice` | By device id |
| `ListDevices` | Paged by organization, with status filter + sort (`NAME`/`REGISTERED_AT`/`LAST_USED`/**`BATTERY`** new in 0.0.61) |
| `UpdateDevice` | Update alias |
| `DeactivateDevice` | Soft-deactivate (DEACTIVATED). Reversible |
| `UnregisterDevice` | **New in 0.0.59.** Permanent revoke (REVOKED + pool unclaim). Distinct from `DeactivateDevice` — REVOKED rows are terminal |
| `PreviewBySerial` | **New in 0.0.61.** Lookup pool row by `hardware_serial` only — no claim, no write. b2b UI registration preview step. Returns `{found, already_claimed, hardware_serial, device_type, firmware_version}` |

`Device` proto fields added in 0.0.61: `battery_level` (int32, `-1` = unknown), `battery_reported_at` (ISO-8601, `""` = none).

`OrganizationDeviceError` cases map to gRPC `Status`:

| Error case | gRPC Status |
|---|---|
| `InvalidOrganization`, `InvalidSerial` | `INVALID_ARGUMENT` |
| `NotAllowed`, `NotB2BDevice`, `RevokedForbidden` | `PERMISSION_DENIED` |
| `AlreadyClaimed`, `DuplicateRegistration` | `ALREADY_EXISTS` |
| `NotFound` | `NOT_FOUND` |

### 7.3 `DeviceLifecycle.UnregisterAllByOwner` (new in 0.0.59)

Java: `DeviceLifecycleGrpcService extends DeviceLifecycleGrpc.DeviceLifecycleImplBase`.

Called by `user-service` deletion outbox worker (`UserOutboxDeletionRetryExecutor.unregisterAllDevices`) during user-account hard-delete. Walks all owner devices and applies REVOKED + pool unclaim. Idempotent — devices already REVOKED are skipped.

#### Request

| Field | Type | Required | Notes |
|---|---|---|---|
| `owner_type` | string | yes | `USER` or `B2B_CENTER` — invalid → `INVALID_ARGUMENT` |
| `owner_id` | string | yes | blank → `INVALID_ARGUMENT` |
| `revoked_by` | string | no | audit (default `"user-service:deletion-worker"`) |
| `reason` | string | no | audit (default `"user_deletion"`) |

#### Response

| Field | Type | Notes |
|---|---|---|
| `unregistered_count` | int | devices newly transitioned to REVOKED in this call |
| `active_before` | int | active devices observed before transition |

### 7.4 `DeviceMetrics.ReportBattery` (new in 0.0.61)

Java: `DeviceMetricsGrpcService extends DeviceMetricsGrpc.DeviceMetricsImplBase`.

Called by `measure-service` fire-and-forget after every footprint / vision record save (`RecordServiceImpl.createFootprint` / `createByVision`). Updates `device.battery_level` + `battery_reported_at`. Failures (NOT_FOUND, INVALID_ARGUMENT, deadline exceeded) are swallowed at the caller so they cannot delay measurement responses.

#### Request

| Field | Type | Required | Notes |
|---|---|---|---|
| `device_id` | string | yes | NOT_FOUND if missing |
| `battery_level` | int32 | yes | 0-100; INVALID_ARGUMENT if out of range |
| `reported_at` | string | no | ISO-8601 (UTC). Empty → server now |
| `reported_by` | string | no | Audit (e.g. `measure-service:record`) |

#### Response

| Field | Type | Notes |
|---|---|---|
| `battery_level` | int32 | Echo of stored value |
| `battery_reported_at` | string | ISO-8601 of stored timestamp |

Dev verification (2026-06-01): one footprint upload via newman `device-mac-lookup-flow` → `DeviceApplicationService.reportBattery args=[deviceId, 99, ...]` succeeded in 12ms.

### 7.5 Removed gRPC

- `DeviceValidator.Validate` — removed in `common-libs` `0.0.58`. Its functionality is split: measurement uses `DeviceLookup.LookupByMac`; B2B uses `DeviceManagement.GetDevice` + status check.

---

## 8. Data Model

### 8.1 `allowed_devices` (shipped-device pool)

| Column | Type | Required | Notes |
|---|---|---|---|
| `id` | `VARCHAR(36)` | yes | PK; `gen_random_uuid()` default |
| `version` | `BIGINT` | yes | JPA optimistic lock |
| `mac_address` | `VARCHAR(17)` | yes | normalized lowercase colon |
| `hardware_serial` | `VARCHAR(64)` | yes | factory serial (e.g. `CN1G00226KR1700001JP`); `TMP-{MAC}` while provisional |
| `device_type` | `VARCHAR(30)` | yes | [`DeviceType`](#devicetype) |
| `firmware_version` | `VARCHAR(64)` | no | seeded from factory record |
| `product_code` | `VARCHAR(32)` | no | xlsx `제품코드` (e.g. `CN1G002`) |
| `production_country` | `VARCHAR(8)` | no | ISO-2 |
| `production_week` | `INTEGER` | no | xlsx `생산주차` |
| `production_year` | `INTEGER` | no | 2-digit (e.g. `26` for 2026) |
| `serial_number` | `INTEGER` | no | within-week counter |
| `shipping_country` | `VARCHAR(8)` | no | ISO-2 |
| `manufactured_at` | `TIMESTAMP` | no | factory record timestamp |
| `claimed` | `BOOLEAN` | yes | flips `true` on user register, `false` on unregister/unclaim |
| `claimed_by_device_id` | `VARCHAR(36)` | no | weak FK to `devices.id` |
| `claimed_at` | `TIMESTAMP` | no | first claim time |
| `is_provisional` | `BOOLEAN` | yes | `true` while serial is `TMP-{MAC}` auto-provisioned by `DeviceLookup` |
| `created_at`, `updated_at`, `deleted_at` | `TIMESTAMP` | no | `BaseTimeEntity` |

Indexes / constraints:

| Name | Columns |
|---|---|
| `pk_allowed_devices` | `id` |
| `uk_allowed_devices_mac_serial` | `mac_address`, `hardware_serial` |
| `uk_allowed_devices_serial` | `hardware_serial` |
| `uk_allowed_devices_mac` | `mac_address` |
| `idx_allowed_devices_claimed` | `claimed` |

Seed data: V5 Flyway migration seeds 35 rows from the legacy `mac_database.xlsx` (first-row anomaly tracked under follow-up I).

### 8.2 `devices`

| Column | Type | Required | Notes |
|---|---|---|---|
| `id` | `VARCHAR(36)` | yes | PK |
| `version` | `BIGINT` | yes | JPA optimistic lock |
| `mac_address` | `VARCHAR(17)` | yes | normalized MAC, copied from pool on register |
| `hardware_serial` | `VARCHAR(64)` | yes | factory serial, copied from pool on register |
| `device_type` | `VARCHAR(30)` | yes | [`DeviceType`](#devicetype), copied from pool |
| `owner_type` | `VARCHAR(20)` | yes | [`OwnerType`](#ownertype) |
| `owner_id` | `VARCHAR(36)` | yes | cross-MSA id; no FK |
| `organization_id` | `VARCHAR(36)` | no | set for B2B-owned rows (`DeviceManagement.RegisterDevice`) |
| `alias` | `VARCHAR(64)` | no | B2B alias |
| `status` | `VARCHAR(20)` | yes | [`DeviceStatus`](#devicestatus) |
| `model` | `VARCHAR(64)` | no | free text |
| `firmware_version` | `VARCHAR(64)` | no | reported/registered version |
| `registered_by` | `VARCHAR(64)` | no | B2B audit |
| `deactivated_at` | `TIMESTAMP` | no | set on `DEACTIVATED` transition |
| `deactivated_by` | `VARCHAR(64)` | no | audit |
| `deactivation_reason` | `VARCHAR(255)` | no | audit |
| `revoked_at` | `TIMESTAMP` | no | set on `REVOKED` transition |
| `revoked_by` | `VARCHAR(64)` | no | audit |
| `revocation_reason` | `VARCHAR(255)` | no | audit |
| `last_used_at` | `TIMESTAMP` | no | set on successful validation/lookup |
| `last_used_ip` | `VARCHAR(45)` | no | IPv4/IPv6 |
| `battery_level` | `INTEGER` | no | 0-100; updated by `DeviceMetrics.ReportBattery` (V10) |
| `battery_reported_at` | `TIMESTAMP` | no | timestamp of last battery report (V10) |
| `created_at`, `updated_at`, `deleted_at` | `TIMESTAMP` | no | `BaseTimeEntity` |

Indexes / constraints:

| Name | Columns | Notes |
|---|---|---|
| `pk_devices` | `id` | — |
| `uk_devices_mac_serial_active` | `mac_address`, `hardware_serial` | **partial unique** `WHERE status != 'REVOKED'` (V9). Lets the same `(mac, serial)` be re-registered after revoke |
| `idx_devices_owner` | `owner_type`, `owner_id` | — |
| `idx_devices_mac` | `mac_address` | — |
| `idx_devices_type` | `device_type` | — |
| `idx_devices_organization` | `organization_id` | B2B lookups |
| `idx_devices_battery_level` | `battery_level` | sort by battery (V10) |

### 8.3 `device_firmwares`

| Column | Type | Required | Notes |
|---|---|---|---|
| `id` | `VARCHAR(36)` | yes | PK |
| `version_lock` | `BIGINT` | yes | JPA optimistic lock |
| `device_type` | `VARCHAR(30)` | yes | [`DeviceType`](#devicetype) |
| `version` | `VARCHAR(64)` | yes | Semver/date/free text |
| `download_url` | `VARCHAR(2000)` | yes | Firmware binary URL |
| `is_latest` | `BOOLEAN` | yes | default `false` |
| `notes` | `VARCHAR(500)` | no | release notes |
| `released_at` | `TIMESTAMP` | no | defaults to `Instant.now()` in service |
| `created_at`, `updated_at`, `deleted_at` | `TIMESTAMP` | no | `BaseTimeEntity` |

Indexes / constraints:

| Name | Columns |
|---|---|
| `pk_device_firmwares` | `id` |
| `uk_firmwares_type_version` | `device_type`, `version` |
| `idx_firmwares_type` | `device_type` |
| `uk_firmwares_one_latest_per_type` | partial unique on `device_type` where `is_latest = TRUE` |

---

## 9. Schemas

### `DeviceRegisterRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |
| `model` | string | no | `@Size(max=64)` |
| `firmwareVersion` | string | no | `@Size(max=64)` |

> `macAddress` / `deviceType` are no longer accepted — the pool is authoritative.

### `DeviceUpdateStatusRequest`

| Field | Type | Required |
|---|---|---|
| `status` | [`DeviceStatus`](#devicestatus) | yes |

### `DeviceValidateRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `macAddress` | string | yes | `@NotBlank`, `@Size(max=17)`, domain MAC format validation |
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |

### `DeviceResponse`

| Field | Type | Notes |
|---|---|---|
| `id` | string | UUID |
| `macAddress` | string | normalized lowercase colon format |
| `hardwareSerial` | string | serial |
| `deviceType` | [`DeviceType`](#devicetype) | — |
| `ownerType` | [`OwnerType`](#ownertype) | — |
| `ownerId` | string | UUID string |
| `status` | [`DeviceStatus`](#devicestatus) | — |
| `model` | string | nullable |
| `firmwareVersion` | string | nullable |
| `lastUsedAt` | string date-time | nullable |
| `lastUsedIp` | string | nullable |
| `createdAt` | string date-time | nullable |
| `updatedAt` | string date-time | nullable |

### `DeviceValidateResponse`

| Field | Type | Notes |
|---|---|---|
| `allowed` | boolean | Always `true` on REST success |
| `deviceId` | string | matched device id |

### `AllowedDeviceRegisterRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `macAddress` | string | yes | `@NotBlank`, MAC format |
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |
| `deviceType` | [`DeviceType`](#devicetype) | yes | — |
| `firmwareVersion` | string | no | — |
| `productCode` | string | no | xlsx `제품코드` |
| `productionCountry` | string | no | ISO-2 |
| `productionWeek` | integer | no | — |
| `productionYear` | integer | no | 2-digit |
| `serialNumber` | integer | no | within-week counter |
| `shippingCountry` | string | no | ISO-2 |
| `manufacturedAt` | string date-time | no | — |

### `ConfirmSerialRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |

### `AllowedDeviceResponse`

Mirror of `allowed_devices` columns plus `claimed`, `claimedByDeviceId`, `claimedAt`, `isProvisional`.

### `AllowedDevicePageResponse`

| Field | Type | Notes |
|---|---|---|
| `content` | [`AllowedDeviceResponse`](#alloweddeviceresponse)[] | page content |
| `page` | int | current page |
| `size` | int | page size |
| `totalElements` | long | total |
| `totalPages` | int | total pages |
| `hasNext` | boolean | — |

### `AllowedDeviceBulkResponse`

| Field | Type | Notes |
|---|---|---|
| `added` | int | newly inserted row count |
| `skippedCount` | int | duplicates (already in pool) |
| `failedCount` | int | other failures |
| `skipped` | `RowResult[]` | `{ row, hardwareSerial, macAddress, reason }` |
| `failed` | `RowResult[]` | same shape |

### `FirmwareRegisterRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `deviceType` | [`DeviceType`](#devicetype) | yes | `@NotNull` |
| `version` | string | yes | `@NotBlank`, `@Size(max=64)` |
| `downloadUrl` | string | yes | `@NotBlank`, `@Size(max=2000)` |
| `makeLatest` | boolean | yes | primitive, default `false` when omitted by JSON mapper |
| `notes` | string | no | `@Size(max=500)` |
| `releasedAt` | string date-time | no | defaults to now when null |

### `FirmwareResponse`

| Field | Type | Notes |
|---|---|---|
| `id` | string | UUID |
| `deviceType` | [`DeviceType`](#devicetype) | — |
| `version` | string | — |
| `downloadUrl` | string | — |
| `isLatest` | boolean | JSON property from record component |
| `notes` | string | nullable |
| `releasedAt` | string date-time | nullable |
| `createdAt` | string date-time | nullable |
| `updatedAt` | string date-time | nullable |

### `FirmwareUpgradeResponse`

| Field | Type | Notes |
|---|---|---|
| `latestVersion` | string | latest firmware version |
| `downloadUrl` | string | firmware binary URL |

### `DeviceType`

| Value | Notes |
|---|---|
| `SCALE2` | — |
| `SCALE_PUZZLE` | — |

### `OwnerType`

| Value | Meaning |
|---|---|
| `USER` | personal user (`user-service.user.id`) |
| `B2B_CENTER` | B2B center / organization owner id |

### `DeviceStatus`

| Value | Meaning | Usable by validate |
|---|---|---|
| `REGISTERED` | registered but not active | no |
| `ACTIVE` | usable | yes |
| `DEACTIVATED` | temporarily disabled — reversible | no |
| `REVOKED` | permanently revoked; row persists for audit but pool unclaim happens at the same time | no |

### CncResponse envelopes

| Schema | Data shape |
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

All envelopes share the shape `{ success, data, token, error, timestamp }`.

---

## 10. Notes

- The shipped-device pool (`allowed_devices`) is authoritative for `(mac, deviceType, firmwareVersion)`. End-user registration only carries `hardwareSerial`.
- MAC normalization to lowercase colon happens in domain service before persistence and pool lookup.
- `DELETE /api/v1/devices/{id}` is idempotent: a second call on an already-REVOKED device returns 200 with the existing row, and no extra pool side-effect.
- The pool partial unique (`status != 'REVOKED'`) lets a user re-register the same `(mac, serial)` after a previous owner unregistered.
- The provisional flow: `DeviceLookup.LookupByMac` for an unknown MAC inserts an `allowed_devices` row with `hardware_serial=TMP-{MAC}` and `is_provisional=true`, then rejects the measurement with `NOT_REGISTERED`. The measurement layer still records `device_serial=TMP-{MAC}` so analytics can later reconcile via `PATCH /admin/allowed-devices/{id}/serial`.
- `user-service` deletion outbox worker calls `DeviceLifecycle.UnregisterAllByOwner` between the activities/records cleanup steps and the OAuth/hard-delete steps. Failures are retried twice with exponential backoff.
- gRPC clients should use `etc.<svc>-grpc=device.carenco.svc.cluster.local:9090` config with a deadline interceptor (default 5000 ms).
- Firmware `makeLatest` and `promote` both unmark the previous latest for the same `deviceType`.
- `DELETE /firmware/{id}` calls repository delete. Because entities extend `BaseTimeEntity`, confirm physical vs soft-delete behavior against common-core JPA implementation when relying on retention semantics.
