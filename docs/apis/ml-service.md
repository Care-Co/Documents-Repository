# ML-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/ML-service`
Framework: Flask (blueprint-based)
Updated: 2026-04-27

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

**Security** &nbsp;none — guard at the gateway.

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

---

## Notes

- `api-version` header is reserved (default `1.0.0`) but not branched on.
- `/measure` is the canonical entry point used by `measure-service` when persisting a record.
- Models auto-download via `gdown` when missing; CUDA EP + CPU EP, 2 GB GPU memory cap, 3-session pool.
