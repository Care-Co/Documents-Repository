# device-service

> Updated: 2026-05-14
> Style: OpenAPI 3.1 + code-backed endpoint catalog + schema tables
> Base path: `/api/v1/devices`
> Response envelope: `CncResponse` (common-core / Pattern B)

Source: `/Users/ijunseong/carenco/repos/device-service`

---

## 1. Overview

`device-service` owns physical device registration, owner-scoped device validation, lifecycle status changes, and OTA firmware metadata.

### 1.1 Domain responsibilities

| Domain | Responsibility |
|---|---|
| `device` | Register physical devices by `(macAddress, hardwareSerial)`, bind to owner, validate active ownership, track last use |
| `firmware` | Store firmware by `deviceType + version`, mark one latest firmware per device type, serve upgrade checks |
| `grpc` | Expose internal `DeviceValidator` gRPC for other MSAs to validate device usage without REST headers |

### 1.2 Runtime

| Item | Value |
|---|---|
| Java / Spring | Java 25, Spring Boot 4.0.5 |
| Persistence | PostgreSQL + Flyway |
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

### 2.2 Response envelope

Success responses use common-core `CncResponse`.

```json
{
  "success": true,
  "data": {},
  "token": null,
  "error": null,
  "timestamp": "2026-05-14T01:00:00Z"
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
- Firmware management endpoints are marked as operating endpoints in code; fine-grained auth is noted as follow-up work.
- Internal MSAs should prefer gRPC validation over REST `/validate`.

---

## 3. Endpoint Catalog

| # | Method | Path | Purpose | Caller headers |
|---|---|---|---|---|
| 1 | `POST` | `/api/v1/devices` | Register device | required |
| 2 | `GET` | `/api/v1/devices/{id}` | Get device by id | no |
| 3 | `GET` | `/api/v1/devices` | List caller-owned devices | required |
| 4 | `PATCH` | `/api/v1/devices/{id}/status` | Change device status | required |
| 5 | `POST` | `/api/v1/devices/validate` | Validate caller can use device | required |
| 6 | `GET` | `/api/v1/devices/firmware/upgrade` | Check latest firmware for a device | no |
| 7 | `POST` | `/api/v1/devices/firmware` | Register firmware | no |
| 8 | `GET` | `/api/v1/devices/firmware` | List firmware by device type | no |
| 9 | `GET` | `/api/v1/devices/firmware/{id}` | Get firmware by id | no |
| 10 | `PATCH` | `/api/v1/devices/firmware/{id}/promote` | Mark firmware as latest | no |
| 11 | `DELETE` | `/api/v1/devices/firmware/{id}` | Delete firmware | no |

---

## 4. Device API

### 4.1 `POST` /api/v1/devices

**Operation ID** `registerDevice`
**Tags** `device`
**Security** caller headers

Register a device to the caller. MAC addresses are normalized to lowercase colon format (`aa:bb:cc:dd:ee:ff`). New devices start with status `REGISTERED`.

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
  "macAddress": "AA-BB-CC-DD-EE-FF",
  "hardwareSerial": "SN-20260514-0001",
  "deviceType": "SCALE2",
  "model": "Scale2 Pro",
  "firmwareVersion": "1.0.0"
}
```

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `201` | [`CncResponse_DeviceResponse`](#cncresponse_deviceresponse) | Created |
| `400` | `CncResponse` error | validation / missing headers / enum mismatch |
| `409` | `CncResponse` error | duplicate `(macAddress, hardwareSerial)` |

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

List devices owned by the caller.

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

### 4.4 `PATCH` /api/v1/devices/{id}/status

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

### 4.5 `POST` /api/v1/devices/validate

**Operation ID** `validateDevice`
**Tags** `device`, `validation`
**Security** caller headers

Validate that `(macAddress, hardwareSerial)` belongs to the caller and has status `ACTIVE`. On success, the service updates `lastUsedAt` and `lastUsedIp`.

#### Request body

`application/json` — [`DeviceValidateRequest`](#devicevalidaterequest)

```json
{
  "macAddress": "aa:bb:cc:dd:ee:ff",
  "hardwareSerial": "SN-20260514-0001"
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

## 5. Firmware API

### 5.1 `GET` /api/v1/devices/firmware/upgrade

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

### 5.2 `POST` /api/v1/devices/firmware

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
  "releasedAt": "2026-05-14T00:00:00Z"
}
```

#### Responses

| Status | Schema | Notes |
|---|---|---|
| `201` | [`CncResponse_FirmwareResponse`](#cncresponse_firmwareresponse) | Created |
| `400` | `CncResponse` error | validation / enum mismatch |
| `409` | `CncResponse` error | duplicate `(deviceType, version)` |

### 5.3 `GET` /api/v1/devices/firmware

**Operation ID** `listFirmwareByType`
**Tags** `firmware`

List firmware records for a device type, sorted by `releasedAt` descending.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `deviceType` | [`DeviceType`](#devicetype) | yes |

#### Responses

| Status | Schema |
|---|---|
| `200` | [`CncResponse_FirmwareResponseList`](#cncresponse_firmwareresponselist) |
| `400` | `CncResponse` error |

### 5.4 `GET` /api/v1/devices/firmware/{id}

**Operation ID** `getFirmwareById`
**Tags** `firmware`

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `id` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| `200` | [`CncResponse_FirmwareResponse`](#cncresponse_firmwareresponse) |
| `404` | `CncResponse` error |

### 5.5 `PATCH` /api/v1/devices/firmware/{id}/promote

**Operation ID** `promoteFirmware`
**Tags** `firmware`, `operations`

Promote a firmware row to latest for its `deviceType`. Existing latest for the same type is unmarked.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `id` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| `200` | [`CncResponse_FirmwareResponse`](#cncresponse_firmwareresponse) |
| `404` | `CncResponse` error |

### 5.6 `DELETE` /api/v1/devices/firmware/{id}

**Operation ID** `deleteFirmware`
**Tags** `firmware`, `operations`

Delete a firmware row.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| path | `id` | string | yes |

#### Responses

| Status | Schema |
|---|---|
| `204` | no body |
| `404` | `CncResponse` error |

---

## 6. Internal gRPC

### `DeviceValidator.Validate`

Java implementation: `DeviceValidatorGrpcService extends DeviceValidatorGrpc.DeviceValidatorImplBase`.

This exposes the same validation logic as `POST /api/v1/devices/validate` for cluster-internal MSA calls.

#### Request

| Field | Type | Required | Notes |
|---|---|---|---|
| `macAddress` | string | yes | Normalized internally |
| `hardwareSerial` | string | yes | Device serial |
| `callerOwnerType` | string | yes | Must map to `OwnerType` enum |
| `callerOwnerId` | string | yes | Owner id |
| `callerIp` | string | no | Stored as `lastUsedIp` when allowed |

#### Response

| Field | Type | Notes |
|---|---|---|
| `allowed` | boolean | `true` only when device exists, owner matches, and status is `ACTIVE` |
| `deviceId` | string | Set on success |
| `rejectReason` | string | Set on invalid request or business rejection |

#### Failure behavior

Unlike REST, gRPC validation does not throw application errors to the caller. It returns:

```json
{
  "allowed": false,
  "rejectReason": "CMN-403-001: Access denied"
}
```

> Cross-repo note: some B2B code references a broader `DeviceManagementGrpc` API. The current `device-service` code exposes only `DeviceValidatorGrpcService`; treat management-client expectations as branch/proto drift until the service adds those RPCs.

---

## 7. Data Model

### 7.1 `devices`

| Column | Type | Required | Notes |
|---|---|---|---|
| `id` | `VARCHAR(36)` | yes | PK |
| `version` | `BIGINT` | yes | JPA optimistic lock |
| `mac_address` | `VARCHAR(17)` | yes | normalized MAC |
| `hardware_serial` | `VARCHAR(64)` | yes | factory serial |
| `device_type` | `VARCHAR(30)` | yes | [`DeviceType`](#devicetype) |
| `owner_type` | `VARCHAR(20)` | yes | [`OwnerType`](#ownertype) |
| `owner_id` | `VARCHAR(36)` | yes | cross-MSA id; no FK |
| `status` | `VARCHAR(20)` | yes | [`DeviceStatus`](#devicestatus) |
| `model` | `VARCHAR(64)` | no | free text |
| `firmware_version` | `VARCHAR(64)` | no | reported/registered version |
| `last_used_at` | `TIMESTAMP` | no | set on successful validation |
| `last_used_ip` | `VARCHAR(45)` | no | IPv4/IPv6 |
| `created_at`, `updated_at`, `deleted_at` | `TIMESTAMP` | no | `BaseTimeEntity` |

Indexes / constraints:

| Name | Columns |
|---|---|
| `pk_devices` | `id` |
| `uk_devices_mac_serial` | `mac_address`, `hardware_serial` |
| `idx_devices_owner` | `owner_type`, `owner_id` |
| `idx_devices_mac` | `mac_address` |
| `idx_devices_type` | `device_type` |

### 7.2 `device_firmwares`

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

## 8. Schemas

### `DeviceRegisterRequest`

| Field | Type | Required | Validation |
|---|---|---|---|
| `macAddress` | string | yes | `@NotBlank`, `@Size(max=17)`, domain MAC format validation |
| `hardwareSerial` | string | yes | `@NotBlank`, `@Size(max=64)` |
| `deviceType` | [`DeviceType`](#devicetype) | yes | `@NotNull` |
| `model` | string | no | `@Size(max=64)` |
| `firmwareVersion` | string | no | `@Size(max=64)` |

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

| Value |
|---|
| `SCALE2` |
| `SCALE_PUZZLE` |

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
| `DEACTIVATED` | temporarily disabled | no |
| `REVOKED` | permanently revoked; cannot be changed again | no |

### `CncResponse_DeviceResponse`

| Field | Type |
|---|---|
| `success` | boolean |
| `data` | [`DeviceResponse`](#deviceresponse) |
| `token` | object / null |
| `error` | object / null |
| `timestamp` | string date-time |

### `CncResponse_DeviceResponseList`

| Field | Type |
|---|---|
| `success` | boolean |
| `data` | [`DeviceResponse`](#deviceresponse)[] |

### `CncResponse_DeviceValidateResponse`

| Field | Type |
|---|---|
| `success` | boolean |
| `data` | [`DeviceValidateResponse`](#devicevalidateresponse) |

### `CncResponse_FirmwareResponse`

| Field | Type |
|---|---|
| `success` | boolean |
| `data` | [`FirmwareResponse`](#firmwareresponse) |

### `CncResponse_FirmwareResponseList`

| Field | Type |
|---|---|
| `success` | boolean |
| `data` | [`FirmwareResponse`](#firmwareresponse)[] |

### `CncResponse_FirmwareUpgradeResponse`

| Field | Type |
|---|---|
| `success` | boolean |
| `data` | [`FirmwareUpgradeResponse`](#firmwareupgraderesponse) |

---

## 9. Notes

- REST device validation and gRPC validation share `DeviceApplicationService.validate`.
- MAC input accepts raw 12 hex characters or colon/hyphen-separated pairs; storage normalizes to lowercase colon format.
- `changeStatus` currently allows direct enum assignment except that `REVOKED` cannot be changed again.
- Firmware `makeLatest` and `promote` both unmark the previous latest for the same `deviceType`.
- `DELETE /firmware/{id}` calls repository delete. Because entities extend `BaseTimeEntity`, confirm physical vs soft-delete behavior against common-core JPA implementation when relying on retention semantics.
