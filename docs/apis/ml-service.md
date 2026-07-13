# ML-service

> 엔드포인트별 **Header · Request · Response** 정의 — 소스는 실제 Flask blueprint 컨트롤러 / Marshmallow 스키마 / 결과 빌드 코드. 표기 — **굵은 필드 = 필수**.

| 항목 | 값 |
|---|---|
| Source | `/Users/jonghak/GitHub/Care&Co/ML-service` |
| Framework | Flask (blueprint-based) |
| Updated | 2026-07-13 |
| Server | `https://ml.example.com` |
| Base path | URL prefix 없음 — 경로는 blueprint 에 정의된 그대로 |

Inference service for plantar pressure (footprint) and human pose estimation. Used by `measure-service` to produce per-record analysis bundles.

---

## 공통 규칙

- **버전** — ML-service 는 unversioned — Flask(blueprint) 기반이라 Spring API versioning 미사용. `api-version` 헤더 불필요, 버전 라벨 없이 Request / Response 로 전개한다.
- **인증** — none — 게이트웨이에서 guard. 서비스 자체 인증 없음.
- **바디** — JSON 계열은 `Content-Type: application/json`, 이미지 업로드 계열(`/measure`, `/pose`)은 `multipart/form-data`.
- **응답 틀** — JSON 을 반환하는 모든 엔드포인트의 바깥 틀은 `Envelope` (아래 접기 참조). probe 2종(`GET /check`, `GET /measure/test`)은 envelope 아님 (plain text).
- **에러 처리** — `handle_api_errors` 데코레이터가 예외를 상태 코드로 매핑한다. `PoseDetectionError` → 422, `ValueError` → 400, `FileNotFoundError` → 404, `RuntimeError` → 500, 그 외 `Exception` → 500. Marshmallow `ValidationError` 는 검증 데코레이터에서 400 으로 즉시 반환된다.
- **동시성** — 세마포어 초과 대기 30 s 타임아웃 → `503 Server is busy. Try again later.` (한도는 아래 접기 참조).

<details>
<summary><b>Architecture · Concurrency</b> — 모델 구성 · 세마포어 한도</summary>

**Architecture**
- Footprint: PyTorch JIT classifier on a 29×22 pressure grid (CPU default).
- Pose: ONNX runtime keypoint estimator (17 COCO joints + 5 computed spine joints) with 3D-cube face mosaic; CUDA + CPU fallback.
- Measurement: composes footprint + 2× pose (front/side), optionally body/age scoring.

**Concurrency** (semaphore timeout 30 s → `503 Server is busy. Try again later.`)

| Semaphore | Limit | Affects |
|---|---|---|
| `semaphore_req` | 15 | `/footprint`, `/measure` |
| `semaphore_req_pose` | 5 | `/pose` |

`/footprint/classification`, `/footprint/size-mask`, `/weight`, `/check`, `/measure/test` 는 세마포어 미적용 (경량 CPU 연산 또는 probe).

</details>

<details>
<summary><b>응답 envelope</b> (<code>Envelope</code>) — 성공 JSON · 필드 정의</summary>

`ApiResponse.create_response` 가 `success = 200 ≤ status < 300` 으로 계산하고 `data` 는 있을 때만 직렬화한다.

```json
{
  "success": true,
  "message": "Footprint processed successfully",
  "data": {}
}
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `success` | boolean | yes | `2xx` 이면 true |
| `message` | string | yes | 성공/실패 메시지 |
| `data` | object | no | endpoint-specific payload (`None` 이면 미직렬화) |

</details>

---

| Method | Path | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/footprint`](#1-post-footprint) | — | 공개 | 족저압 전체 분석 (분류 top-3 + 체중 + 구간별 압력 + 무게중심…) |
| POST | [`/footprint/classification`](#2-post-footprintclassification) | — | 공개 | 족저압 top-9 클래스 신뢰도만 반환 |
| POST | [`/footprint/size-mask`](#3-post-footprintsize-mask) | — | 공개 | 발 크기만 측정 (압력/분류/체중 없음) |
| POST | [`/weight`](#4-post-weight) | — | 공개 | 동일 hex payload 에서 체중만 디코딩 |
| POST | [`/pose`](#5-post-pose) | — | 공개 | 단일 이미지 포즈 추정 (multipart `file` 또는 `image_url`) |
| POST | [`/measure`](#6-post-measure) | — | 공개 | 족저압 + 전면/측면 포즈 2장 + 선택적 body/age 스코어링 합성 |
| GET | [`/check`](#7-get-check) | — | 공개 | 서버 상태 확인 probe (빈 본문 200) |
| GET | [`/measure/test`](#8-get-measuretest) | — | 공개 | 측정 blueprint 헬스 probe (plain text) |

---

## 1. `POST` /footprint

족저압 전체 분석 (분류 top-3 + 체중 + 구간별 압력 + 무게중심 + footprint 이미지 업로드). `semaphore_req` (15) 적용.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

`application/json` 바디 (`FootprintRequestSchema`).

```json
{
  "rawData": "0102....(1290 hex chars)",
  "uid": "user-123",
  "gender": "male",
  "version": "01"
}
```

### Response

**201 Created** — `Envelope` + `data` = footprint 결과 (`process_footprint`)

```json
{
  "success": true,
  "message": "Footprint processed successfully",
  "data": {
    "battery": "100",
    "first_class_type": 1, "first_accuracy": 95.0,
    "second_class_type": 2, "second_accuracy": 4.0,
    "third_class_type": 3, "third_accuracy": 1.0,
    "left_foot_width": 95.0, "left_foot_length": 250.0,
    "right_foot_width": 96.0, "right_foot_length": 251.0,
    "footprint_image_url": "https://.../presigned",
    "weight": 70.5,
    "leftTop": 42.5, "leftMiddle": 30.1, "leftBottom": 27.4,
    "rightTop": 41.0, "rightMiddle": 31.2, "rightBottom": 27.8,
    "leftTotal": 50.2, "rightTotal": 49.8,
    "cogX": 0.51, "cogY": 0.49
  }
}
```

<details>
<summary><b>400 Bad Request</b> — rawData 누락/형식 오류 (hex 아님·1290자 아님) 또는 스키마 검증 실패</summary>

```json
{ "success": false, "message": "Invalid plantar pressure data: expected 1290 hex chars (645 bytes)" }
```

</details>

<details>
<summary><b>500 Internal Server Error</b> — 처리 중 RuntimeError</summary>

```json
{ "success": false, "message": "Internal processing error: ..." }
```

</details>

<details>
<summary><b>503 Service Unavailable</b> — semaphore_req 타임아웃 (30 s)</summary>

```json
{ "success": false, "message": "Server is busy. Try again later." }
```

</details>

</details>

### Request 필드 정의

| 필드 | 타입 | 필수 | Validation | 설명 |
|---|---|---|---|---|
| **`rawData`** | string | yes | length ≥ 1, 디코딩 시 1290 hex chars (645 bytes) | 29×22 pressure grid + meta hex payload |
| `uid` | string | no | length 1–100 | 미지정 시 서버가 UUID 생성 |
| `gender` | string | no | `male` `female` `MALE` `FEMALE` `Male` `Female` | — |
| `version` | string | no | `01` `02`, 기본값 `01` | — |

### Response 필드 정의 (`data`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `battery` | string | 디바이스 배터리 (raw) |
| `first_class_type` | integer (0–9) | top-1 클래스 인덱스 |
| `first_accuracy` | number | 0–100 (%) |
| `second_class_type` / `second_accuracy` | int / number | top-2 |
| `third_class_type` / `third_accuracy` | int / number | top-3 |
| `left_foot_width` / `left_foot_length` | number | mm |
| `right_foot_width` / `right_foot_length` | number | mm |
| `footprint_image_url` | string (uri) | presigned URL |
| `weight` | number | kg |
| `leftTop` `leftMiddle` `leftBottom` | number | 좌발 forefoot/midfoot/heel 압력 비율 (%). 합 ≈ 100 |
| `rightTop` `rightMiddle` `rightBottom` | number | 우발 forefoot/midfoot/heel 비율 (%). 합 ≈ 100 |
| `leftTotal` `rightTotal` | number | 좌/우 발 전체 압력 비율 (%). 합 ≈ 100 |
| `cogX` `cogY` | number | 무게중심 (0–1 정규화) |

---

## 2. `POST` /footprint/classification

족저압 top-9 클래스 신뢰도만 반환. 세마포어 미적용.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

`application/json` 바디 (`FootprintRequestSchema` — §1 와 동일).

```json
{ "rawData": "0102....(1290 hex chars)", "uid": "user-123", "gender": "male", "version": "01" }
```

### Response

**201 Created** — `Envelope` + `data.measured_data` = 클래스 인덱스 → 신뢰도(%) 문자열 (top-9, `:.3f`)

```json
{
  "success": true,
  "message": "Footprint processed successfully",
  "data": { "measured_data": { "1": "85.320", "2": "5.100", "0": "3.200" } }
}
```

<details>
<summary><b>400 Bad Request</b> — rawData 누락/형식 오류</summary>

```json
{ "success": false, "message": "Invalid plantar pressure data: expected 1290 hex chars (645 bytes)" }
```

</details>

<details>
<summary><b>500 Internal Server Error</b> — 처리 중 RuntimeError</summary>

```json
{ "success": false, "message": "Internal processing error: ..." }
```

</details>

</details>

### Request 필드 정의

| 필드 | 타입 | 필수 | Validation | 설명 |
|---|---|---|---|---|
| **`rawData`** | string | yes | length ≥ 1, 1290 hex chars (645 bytes) | 29×22 pressure grid + meta |
| `uid` | string | no | length 1–100 | — |
| `gender` | string | no | `male` `female` (any case) | 사용 안 함 (분류만) |
| `version` | string | no | `01` `02`, 기본값 `01` | — |

### Response 필드 정의 (`data`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `measured_data` | object<string, string> | key = 클래스 인덱스(문자열), value = 신뢰도(%) `:.3f` 문자열. top-9 항목 |

---

## 3. `POST` /footprint/size-mask

발 크기만 측정 (압력/분류/체중 없음). 클라이언트가 측정 세션 동안 누적한 binary contact mask (셀이 한 번이라도 접촉했는지) 를 입력으로 받아, 좌/우 발의 width / length 와 quality / confidence 를 산출한다. 분류·체중 경로와 독립. 세마포어 미적용.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

`application/json` 바디 (`FootSizeMaskRequestSchema`).

```json
{
  "mask": [[0, 1, 1, 0], [1, 1, 1, 1]],
  "frameCount": 120,
  "durationMs": 4000
}
```

### Response

**201 Created** — `Envelope` + `data` = 발 크기 결과 (`calculate_foot_size_from_mask`)

```json
{
  "success": true,
  "message": "Footprint processed successfully",
  "data": {
    "left_foot_width": 95.0,
    "left_foot_length": 250.0,
    "right_foot_width": 96.0,
    "right_foot_length": 251.0,
    "quality": "complete",
    "confidence": 0.87,
    "left":  { "widthCm": 95.0, "lengthCm": 250.0, "quality": "complete", "confidence": 0.87, "rawCellCount": 130, "cellCount": 120, "removedCellCount": 10, "componentCount": 2, "keptComponentCount": 1, "minKeptComponentCells": 12, "rowTop": 3, "rowBottom": 24, "rowSpan": 22, "colMin": 1, "colMax": 9, "colSpan": 9, "touchesSensorEdge": false },
    "right": { "widthCm": 96.0, "lengthCm": 251.0, "quality": "complete", "confidence": 0.89, "rawCellCount": 132, "cellCount": 121, "removedCellCount": 11, "componentCount": 2, "keptComponentCount": 1, "minKeptComponentCells": 12, "rowTop": 3, "rowBottom": 25, "rowSpan": 23, "colMin": 1, "colMax": 9, "colSpan": 9, "touchesSensorEdge": false },
    "frameCount": 120,
    "durationMs": 4000
  }
}
```

> `frameCount` / `durationMs` 는 요청에서 제공된 경우에만 응답에 echo.

<details>
<summary><b>400 Bad Request</b> — mask 누락 또는 shape/값 오류 (정규화 실패 시 ValueError → 400)</summary>

```json
{ "success": false, "message": "mask is required" }
```

</details>

<details>
<summary><b>500 Internal Server Error</b> — 처리 중 오류</summary>

```json
{ "success": false, "message": "Internal server error: ..." }
```

</details>

</details>

### Request 필드 정의

| 필드 | 타입 | 필수 | Validation | 설명 |
|---|---|---|---|---|
| **`mask`** | array | yes | 29×22 grid 또는 flat length 638; 값은 0/1 또는 boolean (유한 수치) | 누적 binary contact mask. `> 0` 인 셀이 "한 번이라도 접촉" 으로 간주 |
| `frameCount` | integer | no | 1–10000 (`allow_none`) | 누적 프레임 수 (응답에 echo) |
| `durationMs` | integer | no | 1–600000 (`allow_none`) | 측정 세션 길이(ms) (응답에 echo) |

### Response 필드 정의 (`data`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `left_foot_width` / `left_foot_length` | number | cm. `left.widthCm` / `left.lengthCm` 와 동일 값 |
| `right_foot_width` / `right_foot_length` | number | cm |
| `quality` | string | overall — `complete` (양발 모두 complete) / `partial` (하나 이상 absent 아님) / `incomplete` (양발 모두 absent) |
| `confidence` | number | 0–1, absent 제외 side confidence 의 최솟값 (없으면 0.0) |
| `left` | object | 좌측 측정 + 통계 (아래 side 필드) |
| `right` | object | 우측 측정 + 통계 |
| `frameCount` | integer | 요청에 포함된 경우에만 echo |
| `durationMs` | integer | 요청에 포함된 경우에만 echo |

### Response `left` / `right` side 필드 정의

| 필드 | 타입 | 설명 |
|---|---|---|
| `widthCm` / `lengthCm` | number | side dimension (cm) |
| `quality` | string | `complete` (conf ≥ 0.78) / `partial` (≥ 0.45) / `incomplete` (< 0.45) / `absent` (접촉 없음) |
| `confidence` | number | 0–1 |
| `rawCellCount` / `cellCount` / `removedCellCount` | integer | clean 전/후 셀 개수 통계 |
| `componentCount` / `keptComponentCount` / `minKeptComponentCells` | integer | connected component 통계 |
| `rowTop` / `rowBottom` / `rowSpan` | integer / null | 접촉 row 범위 (absent 시 top/bottom null, span 0) |
| `colMin` / `colMax` / `colSpan` | integer / null | 접촉 col 범위 |
| `touchesSensorEdge` | boolean | 발이 센서 가장자리 (row 0/28, col 0/10) 에 닿았는지 |

> `left` / `right` 객체에는 위 외에 PCA/dimension 디버그 키가 함께 포함될 수 있다. 컨슈머는 알려진 키만 읽고 모르는 키는 무시한다.

---

## 4. `POST` /weight

동일 hex payload 에서 체중만 디코딩. 세마포어 미적용.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

`application/json` 바디 (`FootprintRequestSchema` — §1 와 동일, `rawData` 만 사용).

```json
{ "rawData": "0102....(1290 hex chars)" }
```

### Response

**201 Created** — `Envelope` + `data.weight`

```json
{ "success": true, "message": "Weight processed successfully", "data": { "weight": 70.5 } }
```

<details>
<summary><b>400 Bad Request</b> — rawData 누락/형식 오류</summary>

```json
{ "success": false, "message": "Invalid plantar pressure data: expected 1290 hex chars (645 bytes)" }
```

</details>

<details>
<summary><b>500 Internal Server Error</b> — 처리 중 RuntimeError</summary>

```json
{ "success": false, "message": "Internal server error: ..." }
```

</details>

</details>

### Request 필드 정의

| 필드 | 타입 | 필수 | Validation | 설명 |
|---|---|---|---|---|
| **`rawData`** | string | yes | length ≥ 1, 1290 hex chars (645 bytes) | 29×22 pressure grid + meta. 체중만 디코딩 |
| `uid` / `gender` / `version` | string | no | §1 참조 | 사용 안 함 |

### Response 필드 정의 (`data`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `weight` | number | kg |

---

## 5. `POST` /pose

단일 이미지 포즈 추정. `semaphore_req_pose` (5) 적용. 입력은 multipart `file` 또는 query `image_url` (MediaService 다운로드) 중 하나.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Content-Type` | `multipart/form-data` (file 전송 시) | file 사용 시 |

<details>
<summary><b>Request · Response</b></summary>

### Request

query params (`PoseRequestSchema`, source `args`) + 선택적 multipart `file`.

```bash
curl -X POST "https://ml.example.com/pose?uid=u1&mosaic=true" -F "file=@front.jpg"
```

### Response

**201 Created** — `Envelope` + `data` = 포즈 결과 (`process_pose_inference`)

```json
{
  "success": true,
  "message": "Pose processed successfully",
  "data": {
    "angleFace": 0.5,
    "anglePelvis": 1.0,
    "angleShoulder": 0.2,
    "url": "https://.../skeleton.jpg",
    "keypoints": {
      "nose": { "x": 0.5, "y": 0.2, "accuracy": 99.0 },
      "shoulderCenter": { "x": 0.5, "y": 0.4, "accuracy": 100 }
    }
  }
}
```

<details>
<summary><b>400 Bad Request</b> — file·image_url 모두 없음, filename 없음, 10 MB 초과, image_url 다운로드 실패, 얼굴 모자이크 키포인트 부족</summary>

```json
{ "success": false, "message": "Missing required file or image_url" }
```

</details>

<details>
<summary><b>422 Unprocessable Entity</b> — 사람(포즈) 미검출</summary>

```json
{
  "success": false,
  "message": "Person was not detected. Please retake the photo with the full body in frame.",
  "data": { "code": "PERSON_NOT_DETECTED", "retakeRequired": true }
}
```

</details>

<details>
<summary><b>503 Service Unavailable</b> — semaphore_req_pose 타임아웃 (30 s)</summary>

```json
{ "success": false, "message": "Server is busy. Try again later." }
```

</details>

</details>

### Request 필드 정의

| In | 이름 | 타입 | 필수 | Validation |
|---|---|---|---|---|
| query | `uid` | string | no | length 1–100 |
| query | `timestamp` | string | no | length 1–50 |
| query | `mosaic` | boolean | no | 기본값 `true` |
| query | `image_url` | string (url) | no | 유효 URL (`allow_none`). `file` 미전송 시 대체 입력 |
| form | `file` | binary (jpg/jpeg/png) | 조건부 | ≤ 10 MB. `image_url` 없으면 필수 |

`file` 과 `image_url` 이 **모두 없으면 400**. `mosaic=true` 인데 얼굴 키포인트(코·양쪽 귀)를 못 찾으면 모자이크 실패 → ValueError → 400.

### Response 필드 정의 (`data`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `angleFace` | number | 얼굴 각도 (deg) |
| `anglePelvis` | number | 골반 각도 (deg) |
| `angleShoulder` | number | 어깨 각도 (deg) |
| `url` | string (uri) | 스켈레톤/척추가 그려진 이미지 URL |
| `keypoints` | object<string, Keypoint> | 검출된 관절 맵 (아래) |

### Keypoint 필드 정의

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`x`** | number | yes | x 좌표 |
| **`y`** | number | yes | y 좌표 |
| **`accuracy`** | number (0–100) | yes | 신뢰도. computed spine joint 는 100 고정 |

Keypoint 키 (17 COCO) — `nose`, `leftEye`, `rightEye`, `leftEar`, `rightEar`, `leftShoulder`, `rightShoulder`, `leftElbow`, `rightElbow`, `leftWrist`, `rightWrist`, `leftHip`, `rightHip`, `leftKnee`, `rightKnee`, `leftAnkle`, `rightAnkle`.
Computed spine joint (양 어깨·엉덩이 검출 시 추가) — `shoulderCenter`, `hipCenter`, `upperChest`, `chest`, `spine`.

---

## 6. `POST` /measure

족저압 + 전면/측면 포즈 2장 + 선택적 body/age 스코어링을 합성. `semaphore_req` (15) 적용. 내부에서 포즈 미검출 시 `PoseDetectionError` → **422** 로 전파된다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |
| `Content-Type` | `multipart/form-data` | yes |

<details>
<summary><b>Request · Response</b></summary>

### Request

query params + `multipart/form-data` parts (`MeasurementRequestSchema`, source `mixed` — form + query 병합).

```bash
curl -X POST "https://ml.example.com/measure?uid=u1&gender=male&height=175.5&age=30" \
  -F "rawData=0102...." -F "front=@front.jpg" -F "side=@side.jpg"
```

### Response

**201 Created** — `Envelope` + `data` = `{ foot, pose, score? }`

```json
{
  "success": true,
  "message": "Measurement processed successfully",
  "data": {
    "foot": {
      "battery": "100", "weight": 70.5,
      "first_class_type": 1, "first_accuracy": 95.0,
      "second_class_type": 2, "second_accuracy": 4.0,
      "third_class_type": 3, "third_accuracy": 1.0,
      "left_foot_width": 95.0, "left_foot_length": 250.0,
      "right_foot_width": 96.0, "right_foot_length": 251.0,
      "footprint_image_url": "https://.../presigned",
      "leftTop": 42.5, "leftMiddle": 30.1, "leftBottom": 27.4,
      "rightTop": 41.0, "rightMiddle": 31.2, "rightBottom": 27.8,
      "leftTotal": 50.2, "rightTotal": 49.8,
      "cogX": 0.51, "cogY": 0.49
    },
    "pose": {
      "front": { "angleFace": 0.5, "anglePelvis": 1.0, "angleShoulder": 0.2, "url": "https://.../front.jpg", "keypoints": {} },
      "side":  { "angleFace": 0.7, "anglePelvis": 0.5, "angleShoulder": 0.2, "url": "https://.../side.jpg", "keypoints": {} }
    },
    "score": { "body_score": 80, "predicted_age": 32 }
  }
}
```

> `score` 는 `height` 와 `age` 가 **둘 다** 제공될 때만 존재한다.

<details>
<summary><b>400 Bad Request</b> — uid/gender/rawData 누락·형식 오류, front/side 파일 누락·10 MB 초과</summary>

```json
{ "success": false, "message": "Missing required files: front, side" }
```

</details>

<details>
<summary><b>422 Unprocessable Entity</b> — 전면 또는 측면에서 사람(포즈) 미검출</summary>

```json
{
  "success": false,
  "message": "Person was not detected. Please retake the photo with the full body in frame.",
  "data": { "code": "PERSON_NOT_DETECTED", "retakeRequired": true, "part": "front" }
}
```

</details>

<details>
<summary><b>500 Internal Server Error</b> — 처리 중 RuntimeError</summary>

```json
{ "success": false, "message": "Internal processing error: ..." }
```

</details>

<details>
<summary><b>503 Service Unavailable</b> — semaphore_req 타임아웃 (30 s)</summary>

```json
{ "success": false, "message": "Server is busy. Try again later." }
```

</details>

</details>

### Request 필드 정의

| In | 이름 | 타입 | 필수 | Validation |
|---|---|---|---|---|
| query | **`uid`** | string | yes | length 1–100 |
| query | **`gender`** | string | yes | `male` `female` (any case) |
| query | `timestamp` | string | no | length 1–50 (`allow_none`) |
| query | `height` | string (decimal) | no | `^\d+(\.\d+)?$`. score 계산에 필요 |
| query | `age` | string (integer) | no | `^\d+$`. score 계산에 필요 |
| query | `mosaic` | boolean | no | 기본값 `true` |
| form | **`rawData`** | string | yes | length ≥ 1 |
| form | **`front`** | binary (이미지) | yes | ≤ 10 MB |
| form | **`side`** | binary (이미지) | yes | ≤ 10 MB |

### Response 필드 정의 (`data`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `foot` | object | footprint 결과 — §1 의 `data` 와 동일 shape (battery / weight / class types / foot 크기 / 압력 비율 / cog / image url) |
| `pose.front` | object | 전면 포즈 결과 (아래 pose 필드) |
| `pose.side` | object | 측면 포즈 결과 |
| `score` | object | `height` + `age` 제공 시에만 존재 |
| `score.body_score` | number | 신체 점수 (정수 반올림) |
| `score.predicted_age` | number | 예측 나이 (정수) |

### pose 객체 필드 정의 (`pose.front` / `pose.side`)

| 필드 | 타입 | 설명 |
|---|---|---|
| `angleFace` | number | 얼굴 각도 (deg) |
| `anglePelvis` | number | 골반 각도 (deg) |
| `angleShoulder` | number | 어깨 각도 (deg) |
| `url` | string (uri) | 스켈레톤/척추가 그려진 이미지 URL |
| `keypoints` | object<string, Keypoint> | 17 COCO joint + 5 computed spine joint. §5 참조 |

---

## 7. `GET` /check

서버 상태 확인 probe. **빈 본문**을 200 으로 반환한다 (JSON envelope 아님).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디·파라미터 없음.

### Response

**200 OK** — 빈 문자열 본문 (`Content-Type: text/html`, 본문 길이 0)

```
```

</details>

---

## 8. `GET` /measure/test

측정 blueprint 헬스 probe. **plain text `measure test`** 를 200 으로 반환한다 (JSON envelope 아님).

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | — (unversioned) | no |

<details>
<summary><b>Request · Response</b></summary>

### Request

바디·파라미터 없음.

### Response

**200 OK** — plain text (`Content-Type: text/html`)

```
measure test
```

</details>
