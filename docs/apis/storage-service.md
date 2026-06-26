# storage-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/storage-service`
Updated: 2026-04-27

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` |

**Versioning**: header-based via Spring `version` attribute. Two API surfaces are exposed:

- `/api/v1/*` — **deprecated** (sunset `2026-12-31`). Each response carries `Deprecation: true`, `Sunset`, `Link: </api/v2/...>; rel="successor-version"`.
- `/api/v2/*` — current.

**Security** &nbsp;No `@PreAuthorize` — auth is enforced at the gateway / Keycloak (v26.0.7 dependency present).

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning). 아래 "제공 버전" 중 하나를 보낸다. `—` 는 unversioned.

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| DELETE | /api/v1/delete | 1.0.0 | 1.0.0 |
| GET | /api/v1/download | 1.0.0 | 1.0.0 |
| POST | /api/v1/upload | 1.0.0 | 1.0.0 |
| POST | /api/v2/confirm-upload | 1.0.0 | 1.0.0 |
| DELETE | /api/v2/delete | 1.0.0 | 1.0.0 |
| GET | /api/v2/download | 1.0.0 | 1.0.0 |
| GET | /api/v2/pre-sign/download | 1.0.0 | 1.0.0 |
| POST | /api/v2/pre-sign/upload | 1.0.0 | 1.0.0 |
| POST | /api/v2/upload | 1.0.0 | 1.0.0 |

---

## v1 (deprecated)

### `POST` /api/v1/upload

**Operation ID** &nbsp;`uploadV1`  &nbsp;**Tags** &nbsp;`v1`, `upload`

Server-mediated upload.

#### Request body

`multipart/form-data` &nbsp;**Required** — [`UploadParts`](#uploadparts)

#### Responses

| Status | Content-Type | Schema | Headers |
|---|---|---|---|
| **200** | `application/json` | [`CncResponse_FileMetadata`](#cncresponse_filemetadata) | `Deprecation`, `Sunset`, `Link` |

### `GET` /api/v1/download

**Operation ID** &nbsp;`downloadV1`  &nbsp;**Tags** &nbsp;`v1`

Streams object as attachment by `key` and/or `id`.

#### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| query | `key` | string | no | S3 object key |
| query | `id` | string | no | metadata pk (one of `key`/`id` should be present) |

#### Responses

| Status | Content-Type | Body | Headers |
|---|---|---|---|
| **200** | `*/*` (resolved from metadata) | binary | `Content-Disposition`, `Content-Length`, `Deprecation`, `Sunset`, `Link` |

### `DELETE` /api/v1/delete

**Operation ID** &nbsp;`deleteV1`  &nbsp;**Tags** &nbsp;`v1`

Removes the object + metadata. Same `key`/`id` parameters as download.

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`CncResponse_Empty`](#cncresponse_empty) |

---

## v2 (current)

### `POST` /api/v2/upload

**Operation ID** &nbsp;`uploadV2`  &nbsp;**Tags** &nbsp;`upload`

Server-mediated upload.

#### Request body

`multipart/form-data` &nbsp;**Required** — [`UploadParts`](#uploadparts)

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`CncResponse_FileMetadata`](#cncresponse_filemetadata) |

```bash
curl -X POST https://api.example.com/api/v2/upload \
  -H "api-version: 1.0.0" -H "Authorization: Bearer ..." \
  -F "file=@x.jpg" -F "folderCategory=PROFILE_PHOTO" \
  -F "identifier=<userId>" -F "isPublic=true"
```

### `POST` /api/v2/pre-sign/upload

**Operation ID** &nbsp;`presignUpload`  &nbsp;**Tags** &nbsp;`presign`

Issue a pre-signed `PUT` URL for client-direct S3 upload.

#### Request body

`multipart/form-data` &nbsp;**Required**

| Part | Type | Required | Validation |
|---|---|---|---|
| `folderCategory` | string (enum: `FOOT_PRINT` `POSE_ESTIMATION` `PROFILE_PHOTO`) | yes | `@NotNull` |
| `identifier` | string | yes | `@NotBlank` |
| `originalFilename` | string | yes | `@NotBlank` |

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `CncResponse` with `data: { uploadUrl: string, downloadUrl: string }` |

#### 200 — example

```json
{
  "success": true,
  "data": {
    "uploadUrl": "https://s3.../x.jpg?X-Amz-Signature=...",
    "downloadUrl": "https://api.example.com/api/v2/pre-sign/download?filename=..."
  }
}
```

### `GET` /api/v2/pre-sign/download

**Operation ID** &nbsp;`presignDownload`  &nbsp;**Tags** &nbsp;`presign`

Issue a time-limited pre-signed `GET` URL.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `filename` | string | yes |

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | `CncResponse` with `data: string` (URL) |

### `POST` /api/v2/confirm-upload

**Operation ID** &nbsp;`confirmUpload`  &nbsp;**Tags** &nbsp;`upload`

Persist metadata after a client-direct upload. Sets the S3 object public via tagging when applicable.

#### Request body

`multipart/form-data` &nbsp;**Required** — [`ConfirmUploadParts`](#confirmuploadparts)

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`CncResponse_Empty`](#cncresponse_empty) |

### `GET` /api/v2/download

**Operation ID** &nbsp;`downloadV2`  &nbsp;**Tags** &nbsp;`download`

Streams object as attachment by `key` and/or `id`.

#### Parameters

| In | Name | Type | Required |
|---|---|---|---|
| query | `key` | string | no |
| query | `id` | string | no |

#### Responses

| Status | Content-Type | Body | Headers |
|---|---|---|---|
| **200** | `*/*` | binary | `Content-Disposition`, `Content-Length` |

### `DELETE` /api/v2/delete

**Operation ID** &nbsp;`deleteV2`  &nbsp;**Tags** &nbsp;`delete`

Removes the object + metadata. Same `key`/`id` parameters as download.

#### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`CncResponse_Empty`](#cncresponse_empty) |

---

## Schemas

### `UploadParts`

| Part | Type | Required | Validation |
|---|---|---|---|
| `file` | binary | yes | `@NotNull` |
| `filename` | string | no | — |
| `folderCategory` | string (enum: `FOOT_PRINT` `POSE_ESTIMATION` `PROFILE_PHOTO`) | yes | `@NotNull` |
| `identifier` | string | yes | `@NotBlank` |
| `isPublic` | string (`"true"`/`"false"`) | no | — |

### `ConfirmUploadParts`

| Part | Type | Required | Validation |
|---|---|---|---|
| `fileName` | string | yes | `@NotBlank` |
| `fileType` | string | yes | `@NotBlank` |
| `fileSize` | number (double) | yes | `@PositiveOrZero` (KB) |
| `fileDownloadUri` | string | yes | `@NotBlank` |
| `bucketName` | string | yes | `@NotBlank` |
| `fileKey` | string | yes | `@NotBlank` |
| `filePath` | string | yes | `@NotBlank` |
| `uploadedBy` | string | yes | `@NotBlank` |
| `isPublic` | boolean | no | — |

### `FileMetadata`

(`FileMetadataDto`)

| Field | Type | Description |
|---|---|---|
| `id` | string | metadata pk |
| `fileName` | string | logical name |
| `fileType` | string | MIME |
| `fileSize` | number (double) | KB |
| `fileDownloadUri` | string | public/routed URI |
| `bucketName` | string | — |
| `fileKey` | string | full S3 key |
| `filePath` | string | object URL/path |
| `uploadedBy` | string | uploader identifier |
| `isPublic` | boolean | — |

### `CncResponse_FileMetadata` / `CncResponse_Empty`

`CncResponse` with `data` of [`FileMetadata`](#filemetadata) / `null` respectively.

### `CncResponse`

| Field | Type | Required |
|---|---|---|
| `success` | boolean | yes |
| `timestamp` | string (date-time, UTC) | yes |
| `data` | object | no |
| `code` | string | no |
| `message` | string | no |
| `error` | object | no |
| `token` | object | no |
| `userVerified` | boolean | no |
| `emailVerified` | boolean | no |

### `ErrorResponse`

```json
{ "success": false, "code": "STR-400-001", "message": "...", "timestamp": "..." }
```

| HTTP | Cause |
|---|---|
| 400 | Validation (missing required parts) |
| 404 | Object/metadata not found |
| 500 | S3/DB error |

