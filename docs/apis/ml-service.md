# ML-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/ML-service`
Framework: Flask (blueprint-based)
Updated: 2026-06-08

Inference service for plantar pressure (footprint) and human pose estimation. Used by `measure-service` to produce per-record analysis bundles.

**Servers**
- `https://ml.example.com`

**Architecture**
- Footprint: PyTorch JIT classifier on a 29×22 pressure grid (CPU default).
- Pose: ONNX runtime keypoint estimator (17 COCO joints) with 3D-cube face mosaic; CUDA + CPU fallback.
- Measurement: composes footprint + 2× pose (front/side), optionally body/age scoring.

**Concurrency** (semaphore timeout 30 s → `503 Server is busy. Try again later.`)

| Semaphore | Limit | Affects |
|---|---|---|
| `semaphore_req` | 15 | `/footprint`, `/footprint/classification`, `/weight`, `/measure` |
| `semaphore_req_pose` | 5 | `/pose` |

`/footprint/size-mask` 는 세마포어 미적용 (경량 CPU 연산).

**Security** &nbsp;none — guard at the gateway.

---

## API 버전 (endpoint별)

> ML-service 는 Flask(blueprint) 기반 — Spring API versioning 미사용. 모든 endpoint 가 unversioned(`—`). 경로는 정의된 그대로.

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| `GET` | /check | — | unversioned |
| `POST` | /footprint | — | unversioned |
| `POST` | /footprint/classification | — | unversioned |
| `POST` | /footprint/size-mask | — | unversioned |
| `GET` | /measure/test | — | unversioned |
| `POST` | /measure | — | unversioned |
| `POST` | /pose | — | unversioned |
| `POST` | /weight | — | unversioned |

---

## `GET` /check

**Operation ID** &nbsp;`healthCheck`
**Tags** &nbsp;`server`

Health probe.

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **200** | `application/json` | [`Envelope`](#envelope) |

#### 200 — example

```json
{ "success": true, "message": "ok" }
```

---

## `POST` /footprint

**Operation ID** &nbsp;`footprint`
**Tags** &nbsp;`footprint`

Full footprint analysis (classification, weight, regional pressure, image upload).

### Request body

`application/json` &nbsp;**Required**

**Schema** — [`FootprintRequest`](#footprintrequest)

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | [`Envelope_FootprintResult`](#envelope_footprintresult) |
| **400** | `application/json` | [`Envelope`](#envelope) |
| **500** | `application/json` | [`Envelope`](#envelope) |
| **503** | `application/json` | [`Envelope`](#envelope) — semaphore timeout |

#### 201 — example

```json
{
  "success": true,
  "message": "Footprint processed successfully",
  "data": {
    "battery": "100",
    "weight": 70.5,
    "first_class_type": 1, "first_accuracy": 95.0,
    "second_class_type": 2, "second_accuracy": 4.0,
    "third_class_type": 3, "third_accuracy": 1.0,
    "left_foot_width": 95.0, "left_foot_length": 250.0,
    "right_foot_width": 96.0, "right_foot_length": 251.0,
    "footprint_image_url": "https://.../presigned",
    "leftTop": 0.1, "leftMiddle": 0.4, "leftBottom": 0.5,
    "rightTop": 0.1, "rightMiddle": 0.4, "rightBottom": 0.5,
    "leftTotal": 1.0, "rightTotal": 1.0,
    "cogX": 0.51, "cogY": 0.49
  }
}
```

---

## `POST` /footprint/classification

**Operation ID** &nbsp;`footprintClassification`
**Tags** &nbsp;`footprint`

Top-9 class confidence only.

### Request body

`application/json` &nbsp;**Required** — [`FootprintRequest`](#footprintrequest)

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | `Envelope` with `data: { measured_data: { "0": "string", … "8": "string" } }` |
| **400** / **500** / **503** | `application/json` | [`Envelope`](#envelope) |

#### 201 — example

```json
{
  "success": true,
  "message": "Footprint processed successfully",
  "data": { "measured_data": { "0": "85.32", "1": "5.10", "2": "3.20" } }
}
```

---

## `POST` /weight

**Operation ID** &nbsp;`weight`
**Tags** &nbsp;`footprint`

Decode weight only from the same hex payload.

### Request body

`application/json` &nbsp;**Required** — [`FootprintRequest`](#footprintrequest)

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | `Envelope` with `data: { weight: number }` |

#### 201 — example

```json
{ "success": true, "message": "Weight processed successfully", "data": { "weight": 70.5 } }
```

---

## `POST` /footprint/size-mask

**Operation ID** &nbsp;`footprintSizeMask`
**Tags** &nbsp;`footprint`

발 크기만 측정 (압력/분류/체중 없음). 클라이언트가 측정 세션 동안 누적한 binary contact mask (셀이 한 번이라도 접촉했는지) 를 입력으로 받아, 좌/우 발의 width / length 와 quality / confidence 를 산출한다. 분류·체중 경로와 독립.

### Request body

`application/json` &nbsp;**Required**

**Schema** — [`FootSizeMaskRequest`](#footsizemaskrequest)

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | [`Envelope_FootSizeMaskResult`](#envelope_footsizemaskresult) |
| **400** | `application/json` | [`Envelope`](#envelope) — mask shape/값 오류 |
| **500** | `application/json` | [`Envelope`](#envelope) |

#### 201 — example

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
    "left":  { "widthCm": 95.0, "lengthCm": 250.0, "quality": "complete", "confidence": 0.87 },
    "right": { "widthCm": 96.0, "lengthCm": 251.0, "quality": "complete", "confidence": 0.89 },
    "frameCount": 120,
    "durationMs": 4000
  }
}
```

> `frameCount` / `durationMs` 는 요청에서 제공된 경우에만 응답에 echo. `left` / `right` 객체에는 측정 통계 (`rowSpan`, `colSpan`, `cellCount`, `touchesSensorEdge` 등) 가 함께 담긴다.

---

## `POST` /pose

**Operation ID** &nbsp;`poseInference`
**Tags** &nbsp;`pose`

Single-image pose estimation.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| query | `uid` | string (length 1-100) | no | — |
| query | `timestamp` | string (max 50) | no | — |
| query | `mosaic` | boolean | no | Default `true`. |

### Request body

`multipart/form-data` &nbsp;**Required**

| Part | Type | Required | Constraints |
|---|---|---|---|
| `file` | binary (jpg/jpeg/png) | yes | ≤ 10 MB |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | [`Envelope_PoseResult`](#envelope_poseresult) |
| **400** | `application/json` | missing/invalid file |
| **404** | `application/json` | no pose detected |
| **503** | `application/json` | semaphore timeout |

#### 201 — example

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
      "nose": { "x": 0.5, "y": 0.2, "accuracy": 99.0 }
    }
  }
}
```

```bash
curl -X POST "https://ml.example.com/pose?uid=u1&mosaic=true" -F "file=@front.jpg"
```

---

## `GET` /measure/test

Plain text `measure test`. **Status** 200.

---

## `POST` /measure

**Operation ID** &nbsp;`measure`
**Tags** &nbsp;`measurement`

Footprint + dual pose + optional scoring.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| query | `uid` | string | yes | User id. |
| query | `gender` | string (`male` / `female`, any case) | yes | — |
| query | `timestamp` | string | no | — |
| query | `height` | string (decimal) | no | e.g. `175.5`. Required for `score`. |
| query | `age` | string (integer) | no | Required for `score`. |
| query | `mosaic` | boolean | no | Default `true`. |

### Request body

`multipart/form-data` &nbsp;**Required**

| Part | Type | Required | Constraints |
|---|---|---|---|
| `rawData` | string | yes | hex 645 bytes |
| `front` | binary | yes | ≤ 10 MB |
| `side` | binary | yes | ≤ 10 MB |

### Responses

| Status | Content-Type | Schema |
|---|---|---|
| **201** | `application/json` | [`Envelope_MeasureResult`](#envelope_measureresult) |
| **400** / **500** / **503** | `application/json` | [`Envelope`](#envelope) |

#### 201 — example

```json
{
  "success": true,
  "message": "Measurement processed successfully",
  "data": {
    "foot": { "weight": 70.5 },
    "pose": {
      "front": { "angleFace": 0.5, "anglePelvis": 1.0, "angleShoulder": 0.2, "url": "...", "keypoints": {} },
      "side":  { "angleFace": 0.7, "anglePelvis": 0.5, "angleShoulder": 0.2, "url": "...", "keypoints": {} }
    },
    "score": { "bodyScore": 80.0, "predictedAge": 30.0 }
  }
}
```

> `score` is only present when both `height` and `age` are provided.

---

## Schemas

### `Envelope`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | — |
| `message` | string | yes | — |
| `data` | object | no | Endpoint-specific payload. |

### `FootprintRequest`

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `rawData` | string | yes | hex 645 bytes | 29×22 pressure grid + meta. |
| `uid` | string | no | length 1-100 | — |
| `gender` | string | no | enum: `male` `female` (any case) | — |
| `version` | string | no | enum: `01` `02` | Default `01`. |

### `FootSizeMaskRequest`

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `mask` | array | yes | 29×22 grid 또는 flat length 638; 값은 0/1 또는 boolean (유한 수치) | 클라이언트가 측정 세션 동안 누적한 binary contact mask. `> 0` 인 셀이 "한 번이라도 접촉" 으로 간주. |
| `frameCount` | integer | no | 1-10000 | 누적에 사용된 프레임 수 (응답에 echo). |
| `durationMs` | integer | no | 1-600000 | 측정 세션 길이(ms) (응답에 echo). |

### `Envelope_FootprintResult`

`Envelope` with `data` =

| Field | Type | Description |
|---|---|---|
| `battery` | string | — |
| `weight` | number | kg |
| `first_class_type` | integer (0-9) | — |
| `first_accuracy` | number | 0-100 |
| `second_class_type` / `second_accuracy` | int / number | — |
| `third_class_type` / `third_accuracy` | int / number | — |
| `left_foot_width` / `left_foot_length` | number | mm |
| `right_foot_width` / `right_foot_length` | number | mm |
| `footprint_image_url` | string (uri) | presigned URL |
| `leftTop` `leftMiddle` `leftBottom` `leftTotal` | number | normalized |
| `rightTop` `rightMiddle` `rightBottom` `rightTotal` | number | normalized |
| `cogX` `cogY` | number | center of gravity |

### `Envelope_FootSizeMaskResult`

`Envelope` with `data` =

| Field | Type | Description |
|---|---|---|
| `left_foot_width` / `left_foot_length` | number | cm. `widthCm` / `lengthCm` 와 동일 값. |
| `right_foot_width` / `right_foot_length` | number | cm |
| `quality` | string | overall — `complete` (양발 모두 complete) / `partial` (한쪽 이상이 absent 가 아님) / `incomplete` (양발 모두 absent) |
| `confidence` | number | 0-1, 양 side confidence 의 최솟값 (absent 제외). |
| `left` | object | 좌측 측정 + 통계 (아래 [`FootSizeMaskSide`](#footsizemaskside)) |
| `right` | object | 우측 측정 + 통계 |
| `frameCount` | integer | 요청에 포함된 경우에만 echo |
| `durationMs` | integer | 요청에 포함된 경우에만 echo |

### `FootSizeMaskSide`

| Field | Type | Description |
|---|---|---|
| `widthCm` / `lengthCm` | number | side dimension (cm) |
| `quality` | string | `complete` / `partial` / `incomplete` / `absent` |
| `confidence` | number | 0-1 |
| `rawCellCount` / `cellCount` / `removedCellCount` | integer | clean 전/후 셀 개수 통계 |
| `componentCount` / `keptComponentCount` / `minKeptComponentCells` | integer | connected component 통계 |
| `rowTop` / `rowBottom` / `rowSpan` | integer / null | 접촉 row 범위 |
| `colMin` / `colMax` / `colSpan` | integer / null | 접촉 col 범위 |
| `touchesSensorEdge` | boolean | 발이 센서 가장자리 (29 row 끝, 11 col 끝) 에 닿았는지 |

> `left` / `right` 객체에는 위 필드 외에 PCA/dimension 디버그 키가 함께 포함될 수 있다. 응답 컨슈머는 알려진 키만 읽고 모르는 키는 무시한다.

### `Envelope_PoseResult`

`Envelope` with `data` =

| Field | Type | Description |
|---|---|---|
| `angleFace` | number (deg, -90 to 90) | — |
| `anglePelvis` | number (deg) | — |
| `angleShoulder` | number (deg) | — |
| `url` | string (uri) | image with skeleton drawn |
| `keypoints` | object<[`Keypoint`](#keypoint)> | 17 detected joints + computed spine joints (`shoulderCenter`, `hipCenter`, `upperChest`, `chest`, `spine`) |

Keypoint names: `nose`, `leftEye`, `rightEye`, `leftEar`, `rightEar`, `leftShoulder`, `rightShoulder`, `leftElbow`, `rightElbow`, `leftWrist`, `rightWrist`, `leftHip`, `rightHip`, `leftKnee`, `rightKnee`, `leftAnkle`, `rightAnkle`.

### `Keypoint`

| Field | Type | Required | Description |
|---|---|---|---|
| `x` | number | yes | — |
| `y` | number | yes | — |
| `accuracy` | number (0-100) | yes | — |

### `Envelope_MeasureResult`

`Envelope` with `data` =

| Field | Type | Description |
|---|---|---|
| `foot` | `FootprintResult` | — |
| `pose.front` | `PoseResult` | — |
| `pose.side` | `PoseResult` | — |
| `score` | object | only present when `height` and `age` provided |
| `score.bodyScore` | number | — |
| `score.predictedAge` | number | — |
