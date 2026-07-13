# storage-service

> 엔드포인트별 **Header · Request · Response** 정의 — 버전별 전량 전개, 소스는 실제 컨트롤러/DTO. 표기 — **굵은 파트 = 필수**, 규칙은 키워드만.

| 항목 | 값 |
|---|---|
| Source | `/Users/jonghak/GitHub/Care&Co/storage-service` |
| Updated | 2026-07-13 |
| Server | `https://api.example.com` |
| Base path | `/api/v2/*` (current) · `/api/v1/*` (**deprecated**) |

---

## 공통 규칙

- **버전** — 모든 엔드포인트 Spring API `version=1.0.0` 단일 — `api-version: 1.0.0` 헤더 (v1·v2 공통). `/api/v1` 와 `/api/v2` 는 URL 경로 구분일 뿐 버전 협상값이 아니다.
- **인증** — `@PreAuthorize` 없음. 인증은 게이트웨이 / Keycloak 에서 강제. 컨트롤러에 `@RequestHeader` 없음 — 커스텀 필수 헤더 없음.
- **바디** — 모든 POST 는 `Content-Type: multipart/form-data` (`@ModelAttribute` form 바인딩, JSON 아님).
- **두 API surface** — `/api/v2/*` 가 current (pre-sign / confirm-upload 추가, deprecation 헤더 없음). `/api/v1/*` 는 **deprecated** — 여전히 존재하며 각 응답에 deprecation 헤더 부착 (아래 접기 참조).
- **응답 틀** — 성공은 `CncResponse` envelope, 에러는 `ErrorResponse` (아래 접기 참조) — 개별 섹션 샘플에서 반복하지 않음. 에러 코드는 common-core `ErrorCode`(**`CMN-*`** prefix) — storage 전용 `STR-*` 네임스페이스는 존재하지 않는다. 전체는 문서 하단 [에러 코드](#에러-코드) 참조.

<details>
<summary><b>v1 deprecation 상세</b> — 헤더 값 · 부착 방식</summary>

`/api/v1/*` 는 `@Deprecated(since=2026-03-26, forRemoval=false)` 상태로 여전히 존재한다. 각 응답에 deprecation 헤더 부착 — `Deprecation: true`, `Sunset: Wed, 31 Dec 2026 23:59:59 GMT`, `Link: </api/v2/upload>; rel="successor-version"`. 헤더는 v1 컨트롤러의 private helper `applyV1DeprecationHeaders(...)` 가 메서드별로 부착 (인터셉터/필터 아님).

</details>

<details>
<summary><b>응답 envelope</b> — 성공/에러 JSON · 필드 정의</summary>

**성공** (`CncResponse`) — 성공 응답의 바깥 틀. `ResponseHelper.success(HttpStatus.OK, data, null)` 는 null 필드를 생략하므로 성공 시 `token`/`code`/`message`/`error` 는 빠진다.

```json
{
  "success": true,
  "data": {},
  "timestamp": "2026-07-09T08:00:00Z"
}
```

| 필드 | 타입 | 필수 |
|---|---|---|
| `success` | boolean | yes |
| `data` | object \| string \| null | no |
| `timestamp` | string (date-time, UTC) | yes |

**에러** (`ErrorResponse`) — `CncErrorResponseBuilder.toResponseEntity(error)`. HTTP status 는 `ErrorCode` 에서. throw 경로(`GlobalExceptionHandler`)와 byte-identical.

```json
{
  "success": false,
  "code": "CMN-404-001",
  "message": "FileMetadata not found",
  "error": "FileMetadata not found: id=file-abc"
}
```

</details>

---

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/api/v2/upload`](#1-post-apiv2upload) | 1.0.0 | 게이트웨이 | 서버 경유 업로드 (current) |
| POST | [`/api/v2/pre-sign/upload`](#2-post-apiv2pre-signupload) | 1.0.0 | 게이트웨이 | client-direct S3 업로드용 pre-signed PUT URL 발급 (TTL 5분) |
| POST | [`/api/v2/confirm-upload`](#3-post-apiv2confirm-upload) | 1.0.0 | 게이트웨이 | client-direct 업로드 후 metadata 영속화 |
| GET | [`/api/v2/pre-sign/download`](#4-get-apiv2pre-signdownload) | 1.0.0 | 게이트웨이 | 시간 제한 pre-signed GET URL 발급 (TTL 5분) |
| GET | [`/api/v2/download`](#5-get-apiv2download) | 1.0.0 | 게이트웨이 | `key` / `id` 로 object 를 attachment 스트리밍 (current) |
| DELETE | [`/api/v2/delete`](#6-delete-apiv2delete) | 1.0.0 | 게이트웨이 | object + metadata 삭제 (current) |
| POST | [`/api/v1/upload`](#7-post-apiv1upload-deprecated) | 1.0.0 | 게이트웨이 | **deprecated** — 서버 경유 업로드, deprecation 헤더 부착 |
| GET | [`/api/v1/download`](#8-get-apiv1download-deprecated) | 1.0.0 | 게이트웨이 | **deprecated** — object attachment 스트리밍 |
| DELETE | [`/api/v1/delete`](#9-delete-apiv1delete-deprecated) | 1.0.0 | 게이트웨이 | **deprecated** — object + metadata 삭제 |

---

## 1. `POST` /api/v2/upload

서버 경유 업로드 (current).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `multipart/form-data` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`multipart/form-data` — `@Valid @ModelAttribute FileUploadRequest` (§7 의 `AwsUploadRequest` 와 필드 동일).

| Part | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`file`** | binary | yes | `@NotNull` |
| `filename` | string | no | — |
| **`folderCategory`** | enum `FolderCategory` — `FOOT_PRINT` \| `POSE_ESTIMATION` \| `PROFILE_PHOTO` | yes | `@NotNull` |
| **`identifier`** | string | yes | `@NotBlank` |
| `isPublic` | string (`"true"`/`"false"`) | no | 대소문자 무관 |

```bash
curl -X POST https://api.example.com/api/v2/upload \
  -H "api-version: 1.0.0" \
  -F 'file=@/tmp/pose.jpg' -F 'folderCategory=POSE_ESTIMATION' \
  -F 'identifier=user-123' -F 'isPublic=false'
```

### `1.0.0` — Response

**200 OK** — `data:` `FileMetadata`

```json
{
  "success": true,
  "data": {
    "id": "file-def",
    "fileName": "pose.jpg",
    "fileType": "image/jpeg",
    "fileSize": 220.0,
    "fileDownloadUri": "https://cdn.example.com/pose_estimation/user-123/pose.jpg",
    "bucketName": "carenco-prod",
    "fileKey": "pose_estimation/user-123/pose.jpg",
    "filePath": "https://s3.../pose.jpg",
    "uploadedBy": "user-123",
    "isPublic": false
  }
}
```

<details>
<summary><b>400 Bad Request</b> — 필수 part 누락 · folderCategory enum 위반</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 동일 <code>uploadedBy</code>+<code>fileName</code> 존재 (`DuplicateFileName`)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request" }
```

</details>

</details>

### Response 필드 정의 — `FileMetadata` (`FileMetadataDto`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | metadata pk |
| `fileName` | string | 논리 파일명 |
| `fileType` | string | MIME |
| `fileSize` | number (double) | KB |
| `fileDownloadUri` | string | public/routed URI |
| `bucketName` | string | — |
| `fileKey` | string | 전체 S3 key |
| `filePath` | string | object URL/path |
| `uploadedBy` | string | 업로더 식별자 |
| `isPublic` | boolean | — |

---

## 2. `POST` /api/v2/pre-sign/upload

client-direct S3 업로드용 pre-signed `PUT` URL 발급 (TTL 5분). 업로드 전 파일명 중복 pre-check.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `multipart/form-data` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`multipart/form-data` — `@Valid @ModelAttribute PreSignedUrlRequest` (**JSON 아님**).

| Part | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`folderCategory`** | enum `FolderCategory` — `FOOT_PRINT` \| `POSE_ESTIMATION` \| `PROFILE_PHOTO` | yes | `@NotNull` |
| **`identifier`** | string | yes | `@NotBlank` |
| **`originalFilename`** | string | yes | `@NotBlank` |

```bash
curl -X POST https://api.example.com/api/v2/pre-sign/upload \
  -H "api-version: 1.0.0" \
  -F 'folderCategory=PROFILE_PHOTO' -F 'identifier=user-123' -F 'originalFilename=avatar.png'
```

### `1.0.0` — Response

**200 OK** — `data:` 는 정확히 2개 키를 가진 map (`uploadUrl`=presigned PUT, `downloadUrl`=동반 request URL)

```json
{
  "success": true,
  "data": {
    "uploadUrl": "https://s3.../avatar.png?X-Amz-Signature=...",
    "downloadUrl": "https://api.example.com/api/v2/pre-sign/download?filename=avatar.png"
  }
}
```

<details>
<summary><b>409 Conflict</b> — 동일 <code>uploadedBy</code>+<code>fileName</code> 존재 (`DuplicateFileName`)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request" }
```

</details>

</details>

### Response 필드 정의 — `data` (`Map<String,String>`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `uploadUrl` | string | presigned PUT URL (TTL 5분) |
| `downloadUrl` | string | 동반 download request URL |

---

## 3. `POST` /api/v2/confirm-upload

client-direct 업로드 후 metadata 영속화. 해당하면 S3 object 를 tagging 으로 public 설정.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `multipart/form-data` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`multipart/form-data` — `@Valid @ModelAttribute ConfirmUploadRequest` (**JSON 아님, form 바인딩**).

| Part | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`fileName`** | string | yes | `@NotBlank` |
| **`fileType`** | string | yes | `@NotBlank` · MIME |
| **`fileSize`** | number (double) | yes | `@PositiveOrZero` · KB |
| **`fileDownloadUri`** | string | yes | `@NotBlank` |
| **`bucketName`** | string | yes | `@NotBlank` |
| **`fileKey`** | string | yes | `@NotBlank` · S3 object key |
| **`filePath`** | string | yes | `@NotBlank` |
| **`uploadedBy`** | string | yes | `@NotBlank` |
| `isPublic` | boolean (primitive) | no | — |

```bash
curl -X POST https://api.example.com/api/v2/confirm-upload \
  -H "api-version: 1.0.0" \
  -F 'fileName=avatar.png' -F 'fileType=image/png' -F 'fileSize=42.5' \
  -F 'fileDownloadUri=https://cdn/.../avatar.png' -F 'bucketName=carenco-prod' \
  -F 'fileKey=profile_photo/user-123/avatar.png' -F 'filePath=https://s3/.../avatar.png' \
  -F 'uploadedBy=user-123' -F 'isPublic=true'
```

### `1.0.0` — Response

**200 OK** — `data` 없는 envelope

```json
{ "success": true }
```

<details>
<summary><b>400 Bad Request</b> — 필수 part 누락</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 동일 <code>uploadedBy</code>+<code>fileName</code> 존재 (`DuplicateFileName`)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request" }
```

</details>

</details>

---

## 4. `GET` /api/v2/pre-sign/download

시간 제한 pre-signed `GET` URL 발급 (TTL 5분).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 `filename` (필수, `@RequestParam` — 누락 시 400).

```
GET /api/v2/pre-sign/download?filename=avatar.png
```

### `1.0.0` — Response

**200 OK** — `data:` 는 **문자열** presigned GET URL (map 아님)

```json
{
  "success": true,
  "data": "https://s3.../avatar.png?X-Amz-Signature=..."
}
```

<details>
<summary><b>404 Not Found</b> — 해당 filename metadata 없음 (`Metadata.NotFound`)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

### Response 필드 정의 — `data` (string)

| 필드 | 타입 | 설명 |
|---|---|---|
| `data` | string | presigned GET URL (TTL 5분) |

---

## 5. `GET` /api/v2/download

`key` / `id` 로 object 를 attachment 스트리밍 (current, deprecation 헤더 없음).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 (`key`/`id` 중 하나 이상).

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `key` | string | no | S3 object key |
| `id` | string | no | metadata pk (`fileId`) |

### `1.0.0` — Response

**200 OK** — 바디는 **raw 파일 바이트**. 응답 헤더 — `Content-Type` · `Content-Disposition` · `Content-Length` (deprecation 헤더 없음).

<details>
<summary><b>400 Bad Request</b> — <code>key</code>·<code>id</code> 둘 다 없음 (`MissingSelector`)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>404 Not Found</b> — metadata 없음 (`Metadata.NotFound`)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## 6. `DELETE` /api/v2/delete

object + metadata 삭제 (current). download 와 동일한 `key`/`id` 셀렉터.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 (`key`/`id` 중 하나 이상).

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `key` | string | no | S3 object key |
| `id` | string | no | metadata pk (`fileId`) |

### `1.0.0` — Response

**200 OK** — `data` 없는 envelope

```json
{ "success": true }
```

<details>
<summary><b>400 Bad Request</b> — <code>key</code>·<code>id</code> 둘 다 없음 (`MissingSelector`)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>404 Not Found</b> — metadata 없음 (`Metadata.NotFound`) · 삭제 대상 0건 (`DeleteAffectedNothing`)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## 7. `POST` /api/v1/upload (deprecated)

서버 경유 업로드. 응답에 deprecation 헤더 부착.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |
| `Content-Type` | `multipart/form-data` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`multipart/form-data` — `@Valid @ModelAttribute AwsUploadRequest`.

| Part | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`file`** | binary | yes | `@NotNull` |
| `filename` | string | no | — |
| **`folderCategory`** | enum `FolderCategory` — `FOOT_PRINT` \| `POSE_ESTIMATION` \| `PROFILE_PHOTO` | yes | `@NotNull` |
| **`identifier`** | string | yes | `@NotBlank` |
| `isPublic` | string (`"true"`/`"false"`) | no | 대소문자 무관 |

```bash
curl -X POST https://api.example.com/api/v1/upload \
  -H "api-version: 1.0.0" \
  -F 'file=@/tmp/foot.png' -F 'folderCategory=FOOT_PRINT' \
  -F 'identifier=user-123' -F 'filename=foot.png' -F 'isPublic=true'
```

### `1.0.0` — Response

**200 OK** — `data:` `FileMetadata`. 응답 헤더 `Deprecation: true` · `Sunset` · `Link`.

```json
{
  "success": true,
  "data": {
    "id": "file-abc",
    "fileName": "foot.png",
    "fileType": "image/png",
    "fileSize": 128.4,
    "fileDownloadUri": "https://cdn.example.com/foot_print/user-123/foot.png",
    "bucketName": "carenco-prod",
    "fileKey": "foot_print/user-123/foot.png",
    "filePath": "https://s3.../foot.png",
    "uploadedBy": "user-123",
    "isPublic": true
  }
}
```

<details>
<summary><b>400 Bad Request</b> — 필수 part 누락 · folderCategory enum 위반</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>409 Conflict</b> — 동일 <code>uploadedBy</code>+<code>fileName</code> 존재 (`DuplicateFileName`)</summary>

```json
{ "success": false, "code": "CMN-409-001", "message": "Duplicate request", "error": "fileName already exists: foot.png" }
```

</details>

</details>

### Response 필드 정의 — `FileMetadata` (`FileMetadataDto`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | metadata pk |
| `fileName` | string | 논리 파일명 |
| `fileType` | string | MIME |
| `fileSize` | number (double) | KB |
| `fileDownloadUri` | string | public/routed URI |
| `bucketName` | string | — |
| `fileKey` | string | 전체 S3 key |
| `filePath` | string | object URL/path |
| `uploadedBy` | string | 업로더 식별자 |
| `isPublic` | boolean | — |

---

## 8. `GET` /api/v1/download (deprecated)

`key` / `id` 로 object 를 attachment 스트리밍. 응답에 deprecation 헤더 부착.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 (`key`/`id` 중 하나 이상).

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `key` | string | no | S3 object key |
| `id` | string | no | metadata pk (`fileId` 로 매핑) |

```bash
curl -OJ 'https://api.example.com/api/v1/download?id=file-abc'
```

### `1.0.0` — Response

**200 OK** — 바디는 **raw 파일 바이트** (JSON envelope 아님). 응답 헤더 — `Content-Type`(metadata `fileType`) · `Content-Disposition: attachment; filename="<fileName>"` · `Content-Length` · `Deprecation`/`Sunset`/`Link`.

<details>
<summary><b>400 Bad Request</b> — <code>key</code>·<code>id</code> 둘 다 없음 (`MissingSelector`)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter", "error": "Both key and id are null" }
```

</details>

<details>
<summary><b>404 Not Found</b> — metadata 없음 (`Metadata.NotFound`)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found" }
```

</details>

</details>

---

## 9. `DELETE` /api/v1/delete (deprecated)

object + metadata 삭제. download 와 동일한 `key`/`id` 셀렉터. 응답에 deprecation 헤더 부착.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음 — 쿼리 파라미터 (`key`/`id` 중 하나 이상).

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `key` | string | no | S3 object key |
| `id` | string | no | metadata pk (`fileId`) |

```bash
curl -X DELETE 'https://api.example.com/api/v1/delete?key=foot_print/user-123/foot.png'
```

### `1.0.0` — Response

**200 OK** — `data` 없는 envelope

```json
{ "success": true }
```

<details>
<summary><b>400 Bad Request</b> — <code>key</code>·<code>id</code> 둘 다 없음 (`MissingSelector`)</summary>

```json
{ "success": false, "code": "CMN-400-001", "message": "Check parameter" }
```

</details>

<details>
<summary><b>404 Not Found</b> — metadata 없음 (`Metadata.NotFound`) · 삭제 대상 0건 (`DeleteAffectedNothing`)</summary>

```json
{ "success": false, "code": "CMN-404-001", "message": "Resource not found", "error": "Bulk delete affected 0 rows: fileKey=..." }
```

</details>

</details>

---

## 에러 코드

> 소스 — common-core `ErrorCode` enum. 코드·HTTP status·i18n 키가 enum 에서 자동 연결. **storage 전용 `STR-*` 코드는 없다.** 외부 S3 IO / `.join()` 실패는 throw → 500 (시스템 오류, Result 아님).

| 코드 | HTTP | 상황 |
|---|---|---|
| `CMN-400-001` | 400 | 필수 part 누락·folderCategory enum 위반·`MissingSelector` (`CHECK_PARAMETER`) |
| `CMN-404-001` | 404 | metadata 없음 (`Metadata.NotFound`) · 삭제 대상 0건 (`DeleteAffectedNothing`) (`RESOURCE_NOT_FOUND`) |
| `CMN-409-001` | 409 | 동일 `uploadedBy`+`fileName` 존재 (`DuplicateFileName`) (`DUPLICATE_REQUEST`) |
| `CMN-500-001` | 500 | S3 / DB IO 실패 (throw, 시스템 오류) |

### 도메인 sealed Error → HTTP

| Error case | HTTP | `description()` |
|---|---|---|
| `FileStorageError.MissingSelector()` | 400 | `Both key and id are null` |
| `FileStorageError.DuplicateFileName(fileName)` | 409 | `fileName already exists: …` |
| `FileStorageError.Metadata(MetadataError.NotFound)` | 404 | `FileMetadata not found: field=value` |
| `FileStorageError.Metadata(MetadataError.DeleteAffectedNothing)` | 404 | `Bulk delete affected 0 rows: fileKey=…` |
