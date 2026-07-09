# graph-service

> 엔드포인트별 Header · Request · Response 정의 — **버전별 전량 전개**. 소스는 실제 컨트롤러/DTO.
> 표기 — **굵은 필드 = 필수**, 스키마는 사용처마다 인라인으로 중복 전개. 버전 매칭은 정확 일치 (미등록/누락 버전 → common-core 의 API version 처리, 4xx).

Source: `/Users/jonghak/GitHub/Care&Co/graph-service`
Updated: 2026-07-09

`src/main/resources/shoes.json` 을 인메모리로 적재하는 러닝화 추천·카탈로그 서비스. 레거시 Neo4j 그래프 의존은 현재 런타임 경로에서 제거됐다. recommendation API 는 footprint/pressure/class 입력으로 신발을 랭킹하고, catalog API 는 검색 가능한 신발·브랜드 데이터를 노출한다. 컨트롤러는 3개 — `GraphSyncController`(v1.0.0), `GraphSyncV101Controller`(v1.0.1, GET shoes 전용), `ShoeQueryController`(catalog, v1.0.0).

**Servers**
- `https://dev.carencoinc.com` — dev gateway
- `https://app.carencoinc.com` — production gateway

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` \| `1.0.1` — 엔드포인트별 Header 표 참조 |
| `Content-Type` | `application/json` (요청 바디가 있을 때) |

**Security** &nbsp;서비스에 `SecurityConfig` · 메서드 시큐리티 애너테이션(`@PreAuthorize`/`@Secured`) 이 없다. 모든 엔드포인트가 사실상 permitAll 이며 인증은 gateway 에 위임된다. 서비스는 `userId` 를 검증 없이 그대로 recommendation 조회 키로 사용한다.

**Success 응답 틀** — 성공 응답은 두 종류다.
- catalog · stateless 계열은 common-core `CncResponse` 빌더를 직접 쓰되 `success` + `data` 만 세팅한다 (`timestamp` 미포함). → `{ "success": true, "data": ... }`
- `GET /api/recommendation/shoes` 는 **`CncResponse` 가 아니라 raw `LinkedHashMap`** 을 반환한다. 1.0.0 은 `{ success, data }`, 1.0.1 은 `{ success, data, full }`.

**Error 틀** — graph-service 는 common-core **0.0.30** (`carenco-platform 0.0.30`) 을 쓴다. 에러 코드 체계가 `CMN-XXX-XXX` 가 아니라 `E001`/`E002`… (`ErrorCodeV2`) 다.

| 원인 | 핸들러 | 바디 shape |
|---|---|---|
| 파라미터 타입 불일치 (`minPrice=abc` 등) | `RequestExceptionHandler` (HIGHEST_PRECEDENCE) | `{ success:false, error:{ message, error }, timestamp }` |
| 잘못된 JSON 바디 | `RequestExceptionHandler` | `{ success:false, error:{ message, error }, timestamp }` |
| 필수 쿼리 파라미터 누락 | `RequestExceptionHandler` | `{ success:false, error:{ message, error }, timestamp }` |
| 잘못된 `metric` 형식 (name 비어있음) | `MetricFilter.parse` → `CncException(E001)` → GlobalExceptionHandler | `{ success:false, code:"E001", message }` |
| 신발 미존재 (`match`, `getShoe`) | 컨트롤러 inline 404 | `{ success:false, message:"shoe not found: <id>" }` |

> `CncResponse` 는 `NON_NULL` 직렬화라 세팅한 필드만 나온다. 필드 집합 — `code` · `success` · `message` · `data` · `token` · `error` · `timestamp` · `userVerified` · `emailVerified`.

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning). 아래 "제공 버전" 중 하나를 보낸다. graph-service 는 unversioned 엔드포인트가 없다 (전부 `1.0.0`, GET shoes 만 `1.0.1` 추가).

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| GET | /api/recommendation/shoes | 1.0.0, 1.0.1 | 1.0.1 |
| POST | /api/recommendation/shoes/stateless | 1.0.0 | 1.0.0 |
| POST | /api/recommendation/shoes/stateless/full | 1.0.0 | 1.0.0 |
| POST | /api/recommendation/shoes/{productId}/match | 1.0.0 | 1.0.0 |
| GET | /api/recommendation/catalog/brands | 1.0.0 | 1.0.0 |
| GET | /api/recommendation/catalog/shoes | 1.0.0 | 1.0.0 |
| GET | /api/recommendation/catalog/shoes/{productId} | 1.0.0 | 1.0.0 |

---

## 1. `GET` /api/recommendation/shoes

`userId` (+선택 `recordId`) 로 프로필/측정 데이터를 조회해 개인화 추천을 생성한다. `CncResponse` 가 아니라 raw map 을 반환한다. **버전에 따라 응답 shape 이 다르다** — `1.0.0` 은 컴팩트 추천 리스트만(`data`), `1.0.1` 은 여기에 상세 매칭 payload(`full`) 를 더한다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` \| `1.0.1` | yes |

### Parameters (전 버전 공통)

| In | Name | Type | 필수 | 규칙 / 기본값 |
|---|---|---|---|---|
| query | `userId` | string | no | 프로필/측정 조회 키. 미전송 가능 |
| query | `recordId` | string | no | 측정 record id. 미전송 시 최신 record 사용 |
| query | `limit` | integer | no | 기본값 `5`, `@Min(1) @Max(50)` |

### `1.0.0` — Request

바디 없음 — 위 쿼리 파라미터만.

### `1.0.0` — Response

**200 OK** — `{ success, data }`. `data` 는 컴팩트 추천 리스트(`RecommendedShoeDto[]`). `full` 필드 **없음**.

```json
{
  "success": true,
  "data": [
    {
      "productId": "101",
      "shoeName": "Example Runner",
      "brand": "Example",
      "category": "Daily running",
      "price": "139.99",
      "year": 2026,
      "size": "270",
      "productUrl": "https://example.com/shoes/101",
      "imageUrl": "https://example.com/shoes/101.png",
      "score": 87.4,
      "brandMatch": 0,
      "attrMatchCount": 3,
      "matchedAttrs": [
        { "metric": "shock_absorption_raw", "weight": 0.82, "rawValue": 75.0 }
      ]
    }
  ]
}
```

### `1.0.1` — Request

바디 없음 — 위 쿼리 파라미터만 (1.0.0 과 동일).

### `1.0.1` — Response

**200 OK** — `{ success, data, full }`. `data` 는 1.0.0 과 동일한 컴팩트 리스트, `full` 은 needs + 상세 매칭(`RecommendationGetFullResponse`). `userId`/record 데이터가 없거나 유효하지 않으면 `full` 은 `{ needs:null, requirementsText:null, recommendations:[] }`.

```json
{
  "success": true,
  "data": [
    {
      "productId": "101",
      "shoeName": "Example Runner",
      "brand": "Example",
      "category": "Daily running",
      "price": "139.99",
      "year": 2026,
      "size": "270",
      "productUrl": "https://example.com/shoes/101",
      "imageUrl": "https://example.com/shoes/101.png",
      "score": 87.4,
      "brandMatch": 0,
      "attrMatchCount": 3,
      "matchedAttrs": [
        { "metric": "shock_absorption_raw", "weight": 0.82, "rawValue": 75.0 }
      ]
    }
  ],
  "full": {
    "needs": {
      "cushioning": 0.8, "stability": 0.6, "support": 0.7,
      "flexibility": 0.4, "responsiveness": 0.5
    },
    "requirementsText": {
      "ko": "쿠셔닝과 지지력을 중시합니다.",
      "en": "Looks for cushioning and support."
    },
    "recommendations": [
      {
        "productId": "101",
        "matchScore": 87.4,
        "matchReasons": ["High cushioning match"],
        "recommendedSize": 270,
        "sizeFit": "Standard Fit",
        "cushioningScore": 0.86,
        "stabilityScore": 0.61,
        "supportScore": 0.72,
        "flexibilityScore": 0.43,
        "responsivenessScore": 0.54,
        "matchedAttrs": [
          { "metric": "shock_absorption_raw", "weight": 0.82, "rawValue": 75.0 }
        ]
      }
    ]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>limit</code> 범위 위반 · <code>limit</code> 타입 불일치</summary>

```json
{
  "success": false,
  "error": { "message": "Invalid parameter type", "error": "Parameter 'limit' has invalid value 'abc' (expected int)" },
  "timestamp": "2026-04-27T08:00:00Z"
}
```

</details>

### Response 필드 정의 — `data[]` (`RecommendedShoeDto`, 전 버전 공통)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`productId`** | string | yes | 신발 id |
| **`shoeName`** | string | yes | 제품명 |
| `brand` | string | no | 브랜드명 |
| `category` | string | no | 카테고리 |
| `price` | string | no | 추천 DTO 에서 가격은 문자열 |
| `year` | integer | no | 출시 연도 |
| `size` | string | no | 추천 사이즈 문자열 |
| `productUrl` | string | no | 제품 URL |
| `imageUrl` | string | no | 제품 이미지 URL |
| **`score`** | number (double) | yes | 추천 점수 |
| **`brandMatch`** | integer | yes | 브랜드 매치 플래그 |
| **`attrMatchCount`** | integer | yes | 매치된 속성 수 |
| **`matchedAttrs`** | `MatchedAttr[]` | yes | 매치 메트릭 요약 (아래) |

**`MatchedAttr`**

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`metric`** | string | yes | 메트릭 이름 |
| `weight` | number (double) | no | 메트릭 가중치 |
| `rawValue` | number (double) | no | 신발의 raw 메트릭 값 |

### Response 필드 정의 — `full` (**1.0.1만**, `RecommendationGetFullResponse`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `needs` | `Needs` | no | 측정 데이터에서 추론한 need 점수. 데이터 없으면 null |
| `requirementsText` | `LocalizedText` | no | class 기반 요구사항 요약. class 신호 없으면 null |
| **`recommendations`** | `RecommendedShoeMatch[]` | yes | 상세 매칭 데이터 (`productId` 기준). 데이터 없으면 빈 배열 |

**`Needs`**

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `cushioning` `stability` `support` `flexibility` `responsiveness` | number (double) | no | need 점수 5종 |

**`LocalizedText`** — 로케일별 텍스트. 세팅된 로케일만 직렬화.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `ko` `ja` `en` `it` `fr` `de` `cs` | string | no | 로케일별 텍스트 |
| `zh-hans` | string | no | JSON 키 `zh-hans` (`@JsonProperty`) |
| `zh-hant` | string | no | JSON 키 `zh-hant` (`@JsonProperty`) |

**`RecommendedShoeMatch`** — `full.recommendations[]` 항목. 전체 `shoe` 객체 대신 `productId` 만 담는다.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`productId`** | string | yes | 신발 id |
| **`matchScore`** | number (double) | yes | 매치 점수 |
| **`matchReasons`** | string[] | yes | 매치 사유 |
| `recommendedSize` | integer | no | 추천 사이즈 |
| `sizeFit` | string | no | 사이즈 핏 라벨 |
| `cushioningScore` `stabilityScore` `supportScore` `flexibilityScore` `responsivenessScore` | number (double) | no | 차원별 점수 |
| **`matchedAttrs`** | `MatchedAttr[]` | yes | 매치 메트릭 요약 |

---

## 2. `POST` /api/recommendation/shoes/stateless

전달된 footprint-정렬 측정 payload 로부터 컴팩트 추천 리스트를 바로 반환한다. `CncResponse{ success, data }` 로 감싼다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |

### `1.0.0` — Request

`StatelessRecommendRequest` (`@Valid`). 모든 필드 선택. `limit` 만 `@Min(1) @Max(50)`.

```json
{
  "firstClassType": 3,
  "firstAccuracy": 82.5,
  "leftFootLengthCm": 26.8,
  "rightFootLengthCm": 26.9,
  "leftFootWidthCm": 10.2,
  "rightFootWidthCm": 10.1,
  "leftTopPct": 30.0, "leftMidPct": 35.0, "leftBotPct": 35.0,
  "rightTopPct": 31.0, "rightMidPct": 34.0, "rightBotPct": 35.0,
  "leftTotalPct": 49.0, "rightTotalPct": 51.0,
  "cogX": 0.51, "cogY": 0.48,
  "limit": 5
}
```

### `1.0.0` — Response

**200 OK** — `data` 는 컴팩트 추천 리스트(`RecommendedShoeDto[]`).

```json
{
  "success": true,
  "data": [
    {
      "productId": "101",
      "shoeName": "Example Runner",
      "brand": "Example",
      "category": "Daily running",
      "price": "139.99",
      "year": 2026,
      "size": "270",
      "productUrl": "https://example.com/shoes/101",
      "imageUrl": "https://example.com/shoes/101.png",
      "score": 87.4,
      "brandMatch": 0,
      "attrMatchCount": 3,
      "matchedAttrs": [
        { "metric": "shock_absorption_raw", "weight": 0.82, "rawValue": 75.0 }
      ]
    }
  ]
}
```

<details>
<summary><b>400 Bad Request</b> — 잘못된 JSON 바디 · <code>limit</code> 범위 위반</summary>

```json
{
  "success": false,
  "error": { "message": "Malformed JSON or incompatible payload", "error": "..." },
  "timestamp": "2026-04-27T08:00:00Z"
}
```

</details>

### Request 필드 정의 — `StatelessRecommendRequest` (공통)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `firstClassType` | integer | no | 1차 발/척추 class 코드 |
| `firstAccuracy` | number (double) | no | 1차 class 정확도 % |
| `secondClassType` | integer | no | 2차 class 코드 |
| `secondAccuracy` | number (double) | no | 2차 class 정확도 % |
| `thirdClassType` | integer | no | 3차 class 코드 |
| `thirdAccuracy` | number (double) | no | 3차 class 정확도 % |
| `leftFootLengthCm` `rightFootLengthCm` | number (double) | no | 발 길이(cm). 둘 다 있으면 평균으로 추천 사이즈 산출 |
| `leftFootWidthCm` `rightFootWidthCm` | number (double) | no | 발 너비(cm). width 체크는 넓은 발 기준 |
| `leftTopPct` `leftMidPct` `leftBotPct` | number (double) | no | 왼발 부위별 압력 % |
| `rightTopPct` `rightMidPct` `rightBotPct` | number (double) | no | 오른발 부위별 압력 % |
| `leftTotalPct` `rightTotalPct` | number (double) | no | 발별 총 압력 점유 % (좌+우 ≈ 100) |
| `cogX` `cogY` | number (double) | no | 무게중심 (0..1 정규화 좌표) |
| `gender` | string | no | 예약 필드 — 현재 추천에 영향 없음 |
| `limit` | integer | no | `@Min(1) @Max(50)` |

### Response 필드 정의 — `data[]` (`RecommendedShoeDto`)

§1 의 `RecommendedShoeDto` 와 동일.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`productId`** | string | yes | 신발 id |
| **`shoeName`** | string | yes | 제품명 |
| `brand` | string | no | 브랜드명 |
| `category` | string | no | 카테고리 |
| `price` | string | no | 가격 문자열 |
| `year` | integer | no | 출시 연도 |
| `size` | string | no | 추천 사이즈 문자열 |
| `productUrl` | string | no | 제품 URL |
| `imageUrl` | string | no | 제품 이미지 URL |
| **`score`** | number (double) | yes | 추천 점수 |
| **`brandMatch`** | integer | yes | 브랜드 매치 플래그 |
| **`attrMatchCount`** | integer | yes | 매치된 속성 수 |
| **`matchedAttrs`** | `MatchedAttr[]` | yes | `{ metric(필수), weight, rawValue }` |

---

## 3. `POST` /api/recommendation/shoes/stateless/full

전달된 footprint-정렬 측정 payload 로부터 needs + **전체 신발 상세(`ShoeDetail`)** 를 포함한 추천을 반환한다. `CncResponse{ success, data }` 로 감싼다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |

### `1.0.0` — Request

`StatelessRecommendRequest` (`@Valid`) — §2 와 동일 (모든 필드 선택, `limit` 만 `@Min(1) @Max(50)`).

```json
{
  "firstClassType": 3,
  "firstAccuracy": 82.5,
  "leftFootLengthCm": 26.8,
  "rightFootLengthCm": 26.9,
  "leftTotalPct": 49.0, "rightTotalPct": 51.0,
  "cogX": 0.51, "cogY": 0.48,
  "limit": 3
}
```

### `1.0.0` — Response

**200 OK** — `data` 는 `RecommendationFullResponse` (`needs` + `requirementsText` + `recommendations[]`). `recommendations[]` 항목은 전체 `shoe` 상세를 담는 `RecommendedShoeFullDto`.

```json
{
  "success": true,
  "data": {
    "needs": {
      "cushioning": 0.8, "stability": 0.6, "support": 0.7,
      "flexibility": 0.4, "responsiveness": 0.5
    },
    "requirementsText": {
      "ko": "쿠셔닝과 지지력을 중시합니다.",
      "en": "Looks for cushioning and support."
    },
    "recommendations": [
      {
        "shoe": {
          "productId": "101",
          "name": "Example Runner",
          "slug": "example-runner",
          "brand": "Example",
          "category": "Daily running",
          "price": 139.99,
          "imageUrl": "https://example.com/shoes/101.png",
          "archSupport": "Neutral",
          "weightG": 238.0,
          "shockAbsorptionRaw": 75.0,
          "bestFor": ["daily", "long-run"]
        },
        "matchScore": 87.4,
        "matchReasons": ["High cushioning match"],
        "recommendedSize": 270,
        "sizeFit": "Standard Fit",
        "cushioningScore": 0.86,
        "stabilityScore": 0.61,
        "supportScore": 0.72,
        "flexibilityScore": 0.43,
        "responsivenessScore": 0.54,
        "matchedAttrs": [
          { "metric": "shock_absorption_raw", "weight": 0.82, "rawValue": 75.0 }
        ]
      }
    ]
  }
}
```

<details>
<summary><b>400 Bad Request</b> — 잘못된 JSON 바디 · <code>limit</code> 범위 위반</summary>

```json
{
  "success": false,
  "error": { "message": "Malformed JSON or incompatible payload", "error": "..." },
  "timestamp": "2026-04-27T08:00:00Z"
}
```

</details>

### Request 필드 정의 — `StatelessRecommendRequest`

§2 와 동일.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `firstClassType` `secondClassType` `thirdClassType` | integer | no | 발/척추 class 코드 3종 |
| `firstAccuracy` `secondAccuracy` `thirdAccuracy` | number (double) | no | class 정확도 % 3종 |
| `leftFootLengthCm` `rightFootLengthCm` | number (double) | no | 발 길이(cm) |
| `leftFootWidthCm` `rightFootWidthCm` | number (double) | no | 발 너비(cm) |
| `leftTopPct` `leftMidPct` `leftBotPct` | number (double) | no | 왼발 부위별 압력 % |
| `rightTopPct` `rightMidPct` `rightBotPct` | number (double) | no | 오른발 부위별 압력 % |
| `leftTotalPct` `rightTotalPct` | number (double) | no | 발별 총 압력 % |
| `cogX` `cogY` | number (double) | no | 무게중심 (0..1) |
| `gender` | string | no | 예약 필드 |
| `limit` | integer | no | `@Min(1) @Max(50)` |

### Response 필드 정의 — `data` (`RecommendationFullResponse`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `needs` | `Needs` | no | need 점수 |
| `requirementsText` | `LocalizedText` | no | class 기반 요약. 없으면 null |
| **`recommendations`** | `RecommendedShoeFull[]` | yes | 전체 신발 상세 추천 객체 |

**`Needs`** — `cushioning` `stability` `support` `flexibility` `responsiveness` (number, no).

**`LocalizedText`** — `ko` `ja` `en` `it` `fr` `de` `cs` `zh-hans`(`@JsonProperty`) `zh-hant`(`@JsonProperty`), 전부 string/no. 세팅된 로케일만 직렬화.

**`RecommendedShoeFull`** — `recommendations[]` 항목.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`shoe`** | `ShoeDetail` | yes | 전체 카탈로그 신발 상세 (아래) |
| **`matchScore`** | number (double) | yes | 매치 점수 |
| **`matchReasons`** | string[] | yes | 매치 사유 |
| `recommendedSize` | integer | no | 추천 사이즈 |
| `sizeFit` | string | no | 사이즈 핏 라벨 |
| `cushioningScore` `stabilityScore` `supportScore` `flexibilityScore` `responsivenessScore` | number (double) | no | 차원별 점수 |
| **`matchedAttrs`** | `MatchedAttr[]` | yes | `{ metric(필수), weight, rawValue }` |

**`ShoeDetail`** (`shoe`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`productId`** | string | yes | 신발 id |
| **`name`** | string | yes | 제품명 |
| `slug` | string | no | 제품 slug |
| `brand` | string | no | 브랜드명 |
| `category` | string | no | 카테고리 |
| `price` | number (double) | no | 카탈로그 숫자 가격 |
| `imageUrl` | string | no | 제품 이미지 URL |
| `archSupport` | string | no | 아치 서포트 라벨 |
| `weightG` | number (double) | no | 무게(g) |
| `dropLabMm` | number (double) | no | 랩 측정 드롭(mm) |
| `heelStackLabMm` | number (double) | no | 힐 스택(mm) |
| `forefootStackLabMm` | number (double) | no | 전족 스택(mm) |
| `midsoleSoftnessLabel` | string | no | 미드솔 소프트니스 라벨 |
| `midsoleSoftnessNew` | number (double) | no | 미드솔 소프트니스 값 |
| `shockAbsorptionLabel` | string | no | 충격 흡수 라벨 |
| `shockAbsorptionRaw` | number (double) | no | 충격 흡수 raw 값 |
| `energyReturnLabel` | string | no | 에너지 리턴 라벨 |
| `energyReturnRaw` | number (double) | no | 에너지 리턴 raw 값 |
| `torsionalRigidityRaw` | number (double) | no | 비틀림 강성 raw 값 |
| `heelCounterStiffnessRaw` | number (double) | no | 힐 카운터 강성 raw 값 |
| `stiffnessLabel` | string | no | 강성 라벨 |
| `stiffnessNNew` | number (double) | no | 강성 값 |
| `breathabilityLabel` | string | no | 통기성 라벨 |
| `breathabilityRaw` | number (double) | no | 통기성 raw 값 |
| `tractionRaw` | number (double) | no | 접지력 raw 값 |
| `sizeRatingLabel` | string | no | 사이즈 평가 라벨 |
| `sizeRatingRaw` | number (double) | no | 사이즈 평가 raw 값 |
| `forefootWidthMm` | number (double) | no | 전족 너비(mm) |
| `outsoleDurabilityMm` | number (double) | no | 아웃솔 내구도 값 |
| `toeboxDurabilityRaw` | number (double) | no | 토박스 내구도 raw 값 |
| `overallScore` | number (double) | no | 종합 점수 |
| `expertScore` | number (double) | no | 전문가 점수 |
| `userScore` | number (double) | no | 사용자 점수 |
| `terrain` | string | no | 지형 라벨 |
| `strikePattern` | string | no | 착지 패턴 라벨 |
| `description` | string | no | 제품 설명 |
| `orthoticFriendly` | boolean | no | 교정기 호환 플래그 |
| `removableInsole` | boolean | no | 탈착 인솔 플래그 |
| `isLightweight` | boolean | no | 경량 플래그 |
| `hasPlate` | boolean | no | 플레이트 플래그 |
| `hasRocker` | boolean | no | 로커 플래그 |
| `bestFor` | string[] | no | 용도 태그 |

---

## 4. `POST` /api/recommendation/shoes/{productId}/match

카탈로그의 신발 1개를 전달된 footprint-정렬 측정 payload 로 채점한다. 미존재 시 404. `CncResponse{ success, data }` 로 감싼다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |

### Parameters

| In | Name | Type | 필수 | 설명 |
|---|---|---|---|---|
| path | `productId` | string | yes | `shoes.json` 의 숫자 신발 id (숫자 파싱 실패 시 404) |

### `1.0.0` — Request

`StatelessRecommendRequest` (`@Valid`) — §2 와 동일.

```json
{
  "firstClassType": 3,
  "firstAccuracy": 82.5,
  "leftFootLengthCm": 26.8,
  "rightFootLengthCm": 26.9,
  "cogX": 0.51, "cogY": 0.48
}
```

### `1.0.0` — Response

**200 OK** — `data` 는 `SingleShoeMatchResponse` (`needs` + `requirementsText` + 단건 `recommendation`). `recommendation` 은 전체 `shoe` 상세를 담는 `RecommendedShoeFullDto`.

```json
{
  "success": true,
  "data": {
    "needs": {
      "cushioning": 0.8, "stability": 0.6, "support": 0.7,
      "flexibility": 0.4, "responsiveness": 0.5
    },
    "requirementsText": {
      "ko": "쿠셔닝과 지지력을 중시합니다.",
      "en": "Looks for cushioning and support."
    },
    "recommendation": {
      "shoe": {
        "productId": "101",
        "name": "Example Runner",
        "slug": "example-runner",
        "brand": "Example",
        "category": "Daily running",
        "price": 139.99,
        "imageUrl": "https://example.com/shoes/101.png",
        "archSupport": "Neutral",
        "weightG": 238.0,
        "shockAbsorptionRaw": 75.0,
        "bestFor": ["daily", "long-run"]
      },
      "matchScore": 87.4,
      "matchReasons": ["High cushioning match"],
      "recommendedSize": 270,
      "sizeFit": "Standard Fit",
      "cushioningScore": 0.86,
      "stabilityScore": 0.61,
      "supportScore": 0.72,
      "flexibilityScore": 0.43,
      "responsivenessScore": 0.54,
      "matchedAttrs": [
        { "metric": "shock_absorption_raw", "weight": 0.82, "rawValue": 75.0 }
      ]
    }
  }
}
```

<details>
<summary><b>404 Not Found</b> — 신발 미존재</summary>

```json
{ "success": false, "message": "shoe not found: 999" }
```

컨트롤러가 `ResponseEntity.status(404)` 로 직접 반환. `code`·`timestamp` 없음.

</details>

<details>
<summary><b>400 Bad Request</b> — 잘못된 JSON 바디 · <code>limit</code> 범위 위반</summary>

```json
{
  "success": false,
  "error": { "message": "Malformed JSON or incompatible payload", "error": "..." },
  "timestamp": "2026-04-27T08:00:00Z"
}
```

</details>

### Request 필드 정의 — `StatelessRecommendRequest`

§2 와 동일.

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `firstClassType` `secondClassType` `thirdClassType` | integer | no | class 코드 3종 |
| `firstAccuracy` `secondAccuracy` `thirdAccuracy` | number (double) | no | class 정확도 % |
| `leftFootLengthCm` `rightFootLengthCm` | number (double) | no | 발 길이(cm) |
| `leftFootWidthCm` `rightFootWidthCm` | number (double) | no | 발 너비(cm) |
| `leftTopPct` `leftMidPct` `leftBotPct` | number (double) | no | 왼발 부위별 압력 % |
| `rightTopPct` `rightMidPct` `rightBotPct` | number (double) | no | 오른발 부위별 압력 % |
| `leftTotalPct` `rightTotalPct` | number (double) | no | 발별 총 압력 % |
| `cogX` `cogY` | number (double) | no | 무게중심 (0..1) |
| `gender` | string | no | 예약 필드 |
| `limit` | integer | no | `@Min(1) @Max(50)` |

### Response 필드 정의 — `data` (`SingleShoeMatchResponse`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `needs` | `Needs` | no | need 점수 |
| `requirementsText` | `LocalizedText` | no | class 기반 요약. 없으면 null |
| **`recommendation`** | `RecommendedShoeFull` | yes | 단건 신발 매치 결과 |

**`Needs`** — `cushioning` `stability` `support` `flexibility` `responsiveness` (number, no).

**`LocalizedText`** — `ko` `ja` `en` `it` `fr` `de` `cs` `zh-hans`(`@JsonProperty`) `zh-hant`(`@JsonProperty`), 전부 string/no.

**`RecommendedShoeFull`** (`recommendation`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`shoe`** | `ShoeDetail` | yes | 전체 카탈로그 신발 상세 (아래) |
| **`matchScore`** | number (double) | yes | 매치 점수 |
| **`matchReasons`** | string[] | yes | 매치 사유 |
| `recommendedSize` | integer | no | 추천 사이즈 |
| `sizeFit` | string | no | 사이즈 핏 라벨 |
| `cushioningScore` `stabilityScore` `supportScore` `flexibilityScore` `responsivenessScore` | number (double) | no | 차원별 점수 |
| **`matchedAttrs`** | `MatchedAttr[]` | yes | `{ metric(필수), weight, rawValue }` |

**`ShoeDetail`** (`shoe`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`productId`** | string | yes | 신발 id |
| **`name`** | string | yes | 제품명 |
| `slug` | string | no | 제품 slug |
| `brand` | string | no | 브랜드명 |
| `category` | string | no | 카테고리 |
| `price` | number (double) | no | 카탈로그 숫자 가격 |
| `imageUrl` | string | no | 제품 이미지 URL |
| `archSupport` | string | no | 아치 서포트 라벨 |
| `weightG` | number (double) | no | 무게(g) |
| `dropLabMm` | number (double) | no | 랩 측정 드롭(mm) |
| `heelStackLabMm` | number (double) | no | 힐 스택(mm) |
| `forefootStackLabMm` | number (double) | no | 전족 스택(mm) |
| `midsoleSoftnessLabel` | string | no | 미드솔 소프트니스 라벨 |
| `midsoleSoftnessNew` | number (double) | no | 미드솔 소프트니스 값 |
| `shockAbsorptionLabel` | string | no | 충격 흡수 라벨 |
| `shockAbsorptionRaw` | number (double) | no | 충격 흡수 raw 값 |
| `energyReturnLabel` | string | no | 에너지 리턴 라벨 |
| `energyReturnRaw` | number (double) | no | 에너지 리턴 raw 값 |
| `torsionalRigidityRaw` | number (double) | no | 비틀림 강성 raw 값 |
| `heelCounterStiffnessRaw` | number (double) | no | 힐 카운터 강성 raw 값 |
| `stiffnessLabel` | string | no | 강성 라벨 |
| `stiffnessNNew` | number (double) | no | 강성 값 |
| `breathabilityLabel` | string | no | 통기성 라벨 |
| `breathabilityRaw` | number (double) | no | 통기성 raw 값 |
| `tractionRaw` | number (double) | no | 접지력 raw 값 |
| `sizeRatingLabel` | string | no | 사이즈 평가 라벨 |
| `sizeRatingRaw` | number (double) | no | 사이즈 평가 raw 값 |
| `forefootWidthMm` | number (double) | no | 전족 너비(mm) |
| `outsoleDurabilityMm` | number (double) | no | 아웃솔 내구도 값 |
| `toeboxDurabilityRaw` | number (double) | no | 토박스 내구도 raw 값 |
| `overallScore` | number (double) | no | 종합 점수 |
| `expertScore` | number (double) | no | 전문가 점수 |
| `userScore` | number (double) | no | 사용자 점수 |
| `terrain` | string | no | 지형 라벨 |
| `strikePattern` | string | no | 착지 패턴 라벨 |
| `description` | string | no | 제품 설명 |
| `orthoticFriendly` | boolean | no | 교정기 호환 플래그 |
| `removableInsole` | boolean | no | 탈착 인솔 플래그 |
| `isLightweight` | boolean | no | 경량 플래그 |
| `hasPlate` | boolean | no | 플레이트 플래그 |
| `hasRocker` | boolean | no | 로커 플래그 |
| `bestFor` | string[] | no | 용도 태그 |

---

## 5. `GET` /api/recommendation/catalog/brands

로드된 신발 카탈로그에서 브랜드명과 제품 수를 반환한다. 브랜드명 오름차순(`TreeMap`) 정렬.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |

### `1.0.0` — Request

바디·파라미터 없음.

### `1.0.0` — Response

**200 OK** — `data` 는 `Brand[]`.

```json
{
  "success": true,
  "data": [
    { "brandName": "Example", "productCount": 12 },
    { "brandName": "Sample", "productCount": 8 }
  ]
}
```

### Response 필드 정의 — `data[]` (`BrandDto`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`brandName`** | string | yes | 브랜드명 |
| **`productCount`** | integer (int64) | yes | 해당 브랜드 신발 수 |

---

## 6. `GET` /api/recommendation/catalog/shoes

브랜드/카테고리/이름/가격 및 메트릭 필터로 카탈로그를 검색한다. 요청은 `size` 를 최대 `100` 까지 받지만 **서비스가 반환 페이지 크기를 `3` 으로 캡한다** (`MAX_SHOE_RESULTS=3`). 응답의 `size` 는 이 effective size(=min(size,3)) 다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |

### Parameters

| In | Name | Type | 필수 | 규칙 / 기본값 |
|---|---|---|---|---|
| query | `brand` | string | no | 대소문자 무시 정확 일치 |
| query | `category` | string | no | 대소문자 무시 정확 일치 |
| query | `minPrice` | number (double) | no | 하한 포함 (inclusive) |
| query | `maxPrice` | number (double) | no | 상한 제외 (exclusive) |
| query | `name` | string | no | 신발명 대소문자 무시 부분 일치 |
| query | `metric` | string[] | no | 반복 가능. `name:min:max` 형식. min/max 생략 가능 (예: `weight_g::250`) |
| query | `sortBy` | string | no | 정렬 기준 메트릭명. 같은 메트릭이 `metric` 에도 있어야 정렬 적용 |
| query | `direction` | string | no | **`LOW`(대소문자 무시) → 오름차순. 그 외 값(미전송 포함) → 내림차순** (메트릭 정렬이 적용될 때) |
| query | `page` | integer | no | 기본값 `0`, `@Min(0)` |
| query | `size` | integer | no | 기본값 `20`, `@Min(1) @Max(100)`. 실제 반환은 `3` 으로 캡 |

### 지원 메트릭 이름 (`metric` · `sortBy`)

`weight_g`, `drop_lab_mm`, `heel_stack_lab_mm`, `forefoot_stack_lab_mm`, `midsole_softness_new`, `shock_absorption_raw`, `energy_return_raw`, `torsional_rigidity_raw`, `heel_counter_stiffness_raw`, `stiffness_n_new`, `breathability_raw`, `traction_raw`, `size_rating_raw`, `forefoot_width_mm`

### `1.0.0` — Request

바디 없음 — 위 쿼리 파라미터만.

### `1.0.0` — Response

**200 OK** — `data` 는 `PagedResponse<Shoe>`. `metrics` 는 `metric` 쿼리에 요청된 이름만 매핑되며, `metric` 미전송 시 `null`.

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "productId": "101",
        "name": "Example Runner",
        "slug": "example-runner",
        "brand": "Example",
        "category": "Daily running",
        "price": 139.99,
        "imageUrl": "https://example.com/shoes/101.png",
        "archSupport": "Neutral",
        "metrics": { "weight_g": 238.0, "shock_absorption_raw": 75.0 }
      }
    ],
    "page": 0,
    "size": 3,
    "totalElements": 128
  }
}
```

<details>
<summary><b>400 Bad Request</b> — 잘못된 <code>metric</code> 형식 (name 비어있음, 예 <code>:250</code>)</summary>

```json
{ "success": false, "code": "E001", "message": "..." }
```

`MetricFilter.parse` 가 `CncException(ErrorCodeV2.CHECK_PARAMETER)` 를 던지고 common-core GlobalExceptionHandler 가 `E001`/400 으로 매핑.

</details>

<details>
<summary><b>400 Bad Request</b> — <code>minPrice</code>/<code>maxPrice</code>/<code>page</code>/<code>size</code> 타입 불일치</summary>

```json
{
  "success": false,
  "error": { "message": "Invalid parameter type", "error": "Parameter 'minPrice' has invalid value 'abc' (expected Double)" },
  "timestamp": "2026-04-27T08:00:00Z"
}
```

</details>

### Response 필드 정의 — `data` (`PagedResponse<Shoe>`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`content`** | `Shoe[]` | yes | 현재 페이지 내용 |
| **`page`** | integer | yes | 요청 페이지 번호 |
| **`size`** | integer | yes | effective 페이지 크기 (min(size,3)) |
| **`totalElements`** | integer (int64) | yes | 페이징 전 총 매치 건수 |

**`Shoe`** (`content[]`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`productId`** | string | yes | 신발 id |
| **`name`** | string | yes | 제품명 |
| `slug` | string | no | 제품 slug |
| `brand` | string | no | 브랜드명 |
| `category` | string | no | 카테고리 |
| `price` | number (double) | no | 카탈로그 숫자 가격 |
| `imageUrl` | string | no | 제품 이미지 URL |
| `archSupport` | string | no | 아치 서포트 라벨 |
| `metrics` | object (map string→double) | no | 요청 메트릭명→값. `metric` 미전송 시 null |

---

## 7. `GET` /api/recommendation/catalog/shoes/{productId}

신발 1개의 전체 카탈로그 상세를 반환한다. `CncResponse{ success, data }` 로 감싼다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |

### Parameters

| In | Name | Type | 필수 | 설명 |
|---|---|---|---|---|
| path | `productId` | string | yes | `shoes.json` 의 숫자 신발 id (blank·숫자 파싱 실패 시 404) |

### `1.0.0` — Request

바디 없음 — path 의 `productId` 만.

### `1.0.0` — Response

**200 OK** — `data` 는 `ShoeDetail` 전체.

```json
{
  "success": true,
  "data": {
    "productId": "101",
    "name": "Example Runner",
    "slug": "example-runner",
    "brand": "Example",
    "category": "Daily running",
    "price": 139.99,
    "imageUrl": "https://example.com/shoes/101.png",
    "archSupport": "Neutral",
    "weightG": 238.0,
    "dropLabMm": 8.0,
    "heelStackLabMm": 38.0,
    "forefootStackLabMm": 30.0,
    "midsoleSoftnessLabel": "Balanced",
    "midsoleSoftnessNew": 28.5,
    "shockAbsorptionLabel": "High",
    "shockAbsorptionRaw": 75.0,
    "energyReturnLabel": "High",
    "energyReturnRaw": 63.0,
    "torsionalRigidityRaw": 4.0,
    "heelCounterStiffnessRaw": 3.0,
    "stiffnessLabel": "Moderate",
    "stiffnessNNew": 18.0,
    "breathabilityLabel": "Good",
    "breathabilityRaw": 4.0,
    "tractionRaw": 4.0,
    "sizeRatingLabel": "True to size",
    "sizeRatingRaw": 0.0,
    "forefootWidthMm": 98.0,
    "outsoleDurabilityMm": 1.2,
    "toeboxDurabilityRaw": 4.0,
    "overallScore": 88.0,
    "expertScore": 90.0,
    "userScore": 86.0,
    "terrain": "Road",
    "strikePattern": "Heel",
    "description": "...",
    "orthoticFriendly": true,
    "removableInsole": true,
    "isLightweight": false,
    "hasPlate": false,
    "hasRocker": true,
    "bestFor": ["daily", "long-run"]
  }
}
```

<details>
<summary><b>404 Not Found</b> — 신발 미존재</summary>

```json
{ "success": false, "message": "shoe not found: 999" }
```

컨트롤러가 `ResponseEntity.status(404)` 로 직접 반환. `code`·`timestamp` 없음.

</details>

### Response 필드 정의 — `data` (`ShoeDetail`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| **`productId`** | string | yes | 신발 id |
| **`name`** | string | yes | 제품명 |
| `slug` | string | no | 제품 slug |
| `brand` | string | no | 브랜드명 |
| `category` | string | no | 카테고리 |
| `price` | number (double) | no | 카탈로그 숫자 가격 |
| `imageUrl` | string | no | 제품 이미지 URL |
| `archSupport` | string | no | 아치 서포트 라벨 |
| `weightG` | number (double) | no | 무게(g) |
| `dropLabMm` | number (double) | no | 랩 측정 드롭(mm) |
| `heelStackLabMm` | number (double) | no | 힐 스택(mm) |
| `forefootStackLabMm` | number (double) | no | 전족 스택(mm) |
| `midsoleSoftnessLabel` | string | no | 미드솔 소프트니스 라벨 |
| `midsoleSoftnessNew` | number (double) | no | 미드솔 소프트니스 값 |
| `shockAbsorptionLabel` | string | no | 충격 흡수 라벨 |
| `shockAbsorptionRaw` | number (double) | no | 충격 흡수 raw 값 |
| `energyReturnLabel` | string | no | 에너지 리턴 라벨 |
| `energyReturnRaw` | number (double) | no | 에너지 리턴 raw 값 |
| `torsionalRigidityRaw` | number (double) | no | 비틀림 강성 raw 값 |
| `heelCounterStiffnessRaw` | number (double) | no | 힐 카운터 강성 raw 값 |
| `stiffnessLabel` | string | no | 강성 라벨 |
| `stiffnessNNew` | number (double) | no | 강성 값 |
| `breathabilityLabel` | string | no | 통기성 라벨 |
| `breathabilityRaw` | number (double) | no | 통기성 raw 값 |
| `tractionRaw` | number (double) | no | 접지력 raw 값 |
| `sizeRatingLabel` | string | no | 사이즈 평가 라벨 |
| `sizeRatingRaw` | number (double) | no | 사이즈 평가 raw 값 |
| `forefootWidthMm` | number (double) | no | 전족 너비(mm) |
| `outsoleDurabilityMm` | number (double) | no | 아웃솔 내구도 값 |
| `toeboxDurabilityRaw` | number (double) | no | 토박스 내구도 raw 값 |
| `overallScore` | number (double) | no | 종합 점수 |
| `expertScore` | number (double) | no | 전문가 점수 |
| `userScore` | number (double) | no | 사용자 점수 |
| `terrain` | string | no | 지형 라벨 |
| `strikePattern` | string | no | 착지 패턴 라벨 |
| `description` | string | no | 제품 설명 |
| `orthoticFriendly` | boolean | no | 교정기 호환 플래그 |
| `removableInsole` | boolean | no | 탈착 인솔 플래그 |
| `isLightweight` | boolean | no | 경량 플래그 |
| `hasPlate` | boolean | no | 플레이트 플래그 |
| `hasRocker` | boolean | no | 로커 플래그 |
| `bestFor` | string[] | no | 용도 태그 |
