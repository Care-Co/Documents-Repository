# graph-service

> OpenAPI 3.1 style markdown. Field tables, request bodies, and response schemas mirror the current Spring controllers and DTO records.

Source: `/Users/ijunseong/carenco/repos/graph-service`
Updated: 2026-06-08

In-memory running-shoe recommendation and catalog service backed by `src/main/resources/shoes.json`. The legacy Neo4j graph dependency is no longer part of the current runtime path. Recommendation APIs rank shoes from footprint/pressure/class input; catalog APIs expose searchable shoe and brand data.

**Servers**
- `https://dev.carencoinc.com` — dev gateway
- `https://app.carencoinc.com` — production gateway

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` |

---

## API 버전 (endpoint별)

> 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning). 아래 "제공 버전" 중 하나를 보낸다. `—` 는 unversioned.

| Method | Path | 제공 버전 | 최신 |
|---|---|---|---|
| GET | /api/recommendation/catalog/brands | 1.0.0 | 1.0.0 |
| GET | /api/recommendation/catalog/shoes | 1.0.0 | 1.0.0 |
| GET | /api/recommendation/catalog/shoes/{productId} | 1.0.0 | 1.0.0 |
| GET | /api/recommendation/shoes | 1.0.0, 1.0.1 | 1.0.1 |
| POST | /api/recommendation/shoes/stateless | 1.0.0 | 1.0.0 |
| POST | /api/recommendation/shoes/stateless/full | 1.0.0 | 1.0.0 |
| POST | /api/recommendation/shoes/{productId}/match | 1.0.0 | 1.0.0 |

---

## `GET` /api/recommendation/shoes

**Operation ID** &nbsp;`recommendShoes`
**Tags** &nbsp;`recommendation`
**Security** &nbsp;`permitAll` (no authentication)

Personalized recommendation using `userId` and optionally a specific `recordId`. The current response is not wrapped by `CncResponse`; it returns the compact recommendation list in `data` and the detailed match payload in `full`.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | Header-based API version selector. |
| query | `userId` | string | no | User id used to fetch profile/measurement data. |
| query | `recordId` | string | no | Measurement record id. If absent, service uses the latest available record. |
| query | `limit` | integer (`@Min(1) @Max(50)`) | no | Default `5`. |

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`RecommendationGetResponse`](#recommendationgetresponse) |
| **400** | Invalid query parameter | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 200 — example

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
        {
          "metric": "shock_absorption_raw",
          "weight": 0.82,
          "rawValue": 75.0
        }
      ]
    }
  ],
  "full": {
    "needs": {
      "cushioning": 0.8,
      "stability": 0.6,
      "support": 0.7,
      "flexibility": 0.4,
      "responsiveness": 0.5
    },
    "requirementsText": {
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
          {
            "metric": "shock_absorption_raw",
            "weight": 0.82,
            "rawValue": 75.0
          }
        ]
      }
    ]
  }
}
```

---

## `POST` /api/recommendation/shoes/stateless

**Operation ID** &nbsp;`recommendShoesStateless`
**Tags** &nbsp;`recommendation`
**Security** &nbsp;`permitAll` (no authentication)

Returns compact shoe recommendations directly from the provided footprint-aligned measurement payload.

### Request Body

| Content-Type | Schema |
|---|---|
| `application/json` | [`StatelessRecommendRequest`](#statelessrecommendrequest) |

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_RecommendedShoeList`](#cncresponse_recommendedshoelist) |
| **400** | Invalid request body or `limit` out of range | `application/json` | [`ErrorResponse`](#errorresponse) |

---

## `POST` /api/recommendation/shoes/stateless/full

**Operation ID** &nbsp;`recommendShoesStatelessFull`
**Tags** &nbsp;`recommendation`
**Security** &nbsp;`permitAll` (no authentication)

Returns detailed needs and full shoe detail objects from the provided footprint-aligned measurement payload.

### Request Body

| Content-Type | Schema |
|---|---|
| `application/json` | [`StatelessRecommendRequest`](#statelessrecommendrequest) |

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_RecommendationFull`](#cncresponse_recommendationfull) |
| **400** | Invalid request body or `limit` out of range | `application/json` | [`ErrorResponse`](#errorresponse) |

---

## `POST` /api/recommendation/shoes/{productId}/match

**Operation ID** &nbsp;`matchSingleShoe`
**Tags** &nbsp;`recommendation`
**Security** &nbsp;`permitAll` (no authentication)

Scores one catalog shoe against the provided footprint-aligned measurement payload.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| path | `productId` | string | yes | Numeric shoe id from `shoes.json`. |

### Request Body

| Content-Type | Schema |
|---|---|
| `application/json` | [`StatelessRecommendRequest`](#statelessrecommendrequest) |

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_SingleShoeMatch`](#cncresponse_singleshoematch) |
| **400** | Invalid request body or `limit` out of range | `application/json` | [`ErrorResponse`](#errorresponse) |
| **404** | Shoe not found | `application/json` | [`CncResponse_ErrorMessage`](#cncresponse_errormessage) |

---

## `GET` /api/recommendation/catalog/brands

**Operation ID** &nbsp;`listBrands`
**Tags** &nbsp;`catalog`
**Security** &nbsp;`permitAll` (no authentication)

Returns brand names and product counts from the loaded shoe catalog.

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_BrandList`](#cncresponse_brandlist) |

---

## `GET` /api/recommendation/catalog/shoes

**Operation ID** &nbsp;`listShoes`
**Tags** &nbsp;`catalog`
**Security** &nbsp;`permitAll` (no authentication)

Searches catalog shoes with optional brand/category/name/price filters and metric filters. Although the request accepts `size` up to `100`, the service currently caps returned page size to `3`.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| query | `brand` | string | no | Exact case-insensitive brand match. |
| query | `category` | string | no | Exact case-insensitive category match. |
| query | `minPrice` | number (double) | no | Inclusive lower price bound. |
| query | `maxPrice` | number (double) | no | Exclusive upper price bound. |
| query | `name` | string | no | Case-insensitive substring match against shoe name. |
| query | `metric` | string[] | no | Repeatable metric filter in `name:min:max` format. Empty min/max are allowed, e.g. `weight_g::250`. |
| query | `sortBy` | string | no | Metric name to sort by. Sorting only applies when the same metric also appears in `metric`. |
| query | `direction` | string | no | `asc` for ascending; any other value defaults to descending when metric sorting applies. |
| query | `page` | integer (`@Min(0)`) | no | Default `0`. |
| query | `size` | integer (`@Min(1) @Max(100)`) | no | Default `20`; effective response size is capped to `3`. |

### Supported Metric Names

`weight_g`, `drop_lab_mm`, `heel_stack_lab_mm`, `forefoot_stack_lab_mm`, `midsole_softness_new`, `shock_absorption_raw`, `energy_return_raw`, `torsional_rigidity_raw`, `heel_counter_stiffness_raw`, `stiffness_n_new`, `breathability_raw`, `traction_raw`, `size_rating_raw`, `forefoot_width_mm`

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_ShoePage`](#cncresponse_shoepage) |
| **400** | Invalid parameter type, malformed `metric`, or invalid page/size | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 200 — example

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
        "metrics": {
          "weight_g": 238.0,
          "shock_absorption_raw": 75.0
        }
      }
    ],
    "page": 0,
    "size": 3,
    "totalElements": 128
  }
}
```

---

## `GET` /api/recommendation/catalog/shoes/{productId}

**Operation ID** &nbsp;`getShoe`
**Tags** &nbsp;`catalog`
**Security** &nbsp;`permitAll` (no authentication)

Returns the full catalog detail for one shoe.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| path | `productId` | string | yes | Numeric shoe id from `shoes.json`. |

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_ShoeDetail`](#cncresponse_shoedetail) |
| **404** | Shoe not found | `application/json` | [`CncResponse_ErrorMessage`](#cncresponse_errormessage) |

---

## Schemas

### `RecommendationGetResponse`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`RecommendedShoe`](#recommendedshoe)[] | yes | Compact ranked recommendations. |
| `full` | [`RecommendationGetFull`](#recommendationgetfull) | yes | Detailed needs and per-shoe match data. |

### `RecommendationGetFull`

| Field | Type | Required | Description |
|---|---|---|---|
| `needs` | [`Needs`](#needs) | yes | User need scores inferred from measurement data. |
| `requirementsText` | [`LocalizedText`](#localizedtext) | no | Class-based natural language summary of requirements. `null` when no class signal available. |
| `recommendations` | [`RecommendedShoeMatch`](#recommendedshoematch)[] | yes | Detailed match data keyed by product id. |

### `CncResponse_RecommendedShoeList`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`RecommendedShoe`](#recommendedshoe)[] | yes | Ranked recommendations. |
| `timestamp` | string (date-time) | no | Included by common response wrapper. |

### `CncResponse_RecommendationFull`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`RecommendationFull`](#recommendationfull) | yes | Detailed recommendation response. |
| `timestamp` | string (date-time) | no | Included by common response wrapper. |

### `CncResponse_SingleShoeMatch`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`SingleShoeMatch`](#singleshoematch) | yes | Single-shoe match result. |
| `timestamp` | string (date-time) | no | Included by common response wrapper. |

### `CncResponse_BrandList`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`Brand`](#brand)[] | yes | Brand counts. |
| `timestamp` | string (date-time) | no | Included by common response wrapper. |

### `CncResponse_ShoePage`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`PagedResponse_Shoe`](#pagedresponse_shoe) | yes | Paged catalog result. |
| `timestamp` | string (date-time) | no | Included by common response wrapper. |

### `CncResponse_ShoeDetail`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`ShoeDetail`](#shoedetail) | yes | Full shoe detail. |
| `timestamp` | string (date-time) | no | Included by common response wrapper. |

### `CncResponse_ErrorMessage`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `false`. |
| `message` | string | yes | Human-readable error message. |
| `timestamp` | string (date-time) | no | Included by common response wrapper when applicable. |

### `RecommendedShoe`

| Field | Type | Required | Description |
|---|---|---|---|
| `productId` | string | yes | Shoe id. |
| `shoeName` | string | yes | Product name. |
| `brand` | string | no | Brand name. |
| `category` | string | no | Category name. |
| `price` | string | no | Price formatted as a string in recommendation DTO. |
| `year` | integer | no | Release year when available. |
| `size` | string | no | Recommended size string. |
| `productUrl` | string | no | Product URL when available. |
| `imageUrl` | string | no | Product image URL. |
| `score` | number (double) | yes | Recommendation score. |
| `brandMatch` | integer | yes | Brand match flag. |
| `attrMatchCount` | integer | yes | Count of matched attributes. |
| `matchedAttrs` | [`MatchedAttr`](#matchedattr)[] | yes | Matched metric summary. |

### `MatchedAttr`

| Field | Type | Required | Description |
|---|---|---|---|
| `metric` | string | yes | Metric name. |
| `weight` | number (double) | no | Metric weight. |
| `rawValue` | number (double) | no | Raw shoe metric value. |

### `RecommendationFull`

| Field | Type | Required | Description |
|---|---|---|---|
| `needs` | [`Needs`](#needs) | yes | User need scores. |
| `requirementsText` | [`LocalizedText`](#localizedtext) | no | Class-based natural language summary of requirements. `null` when no class signal available. |
| `recommendations` | [`RecommendedShoeFull`](#recommendedshoefull)[] | yes | Full shoe recommendation objects. |

### `SingleShoeMatch`

| Field | Type | Required | Description |
|---|---|---|---|
| `needs` | [`Needs`](#needs) | yes | User need scores. |
| `requirementsText` | [`LocalizedText`](#localizedtext) | no | Class-based natural language summary of requirements. `null` when no class signal available. |
| `recommendation` | [`RecommendedShoeFull`](#recommendedshoefull) | yes | Match result for one shoe. |

### `LocalizedText`

| Field | Type | Required | Description |
|---|---|---|---|
| `en` | string | no | English text. Other locales may be added later. |

### `RecommendedShoeFull`

| Field | Type | Required | Description |
|---|---|---|---|
| `shoe` | [`ShoeDetail`](#shoedetail) | yes | Full catalog shoe detail. |
| `matchScore` | number (double) | yes | Match score. |
| `matchReasons` | string[] | yes | Human-readable match reasons. |
| `recommendedSize` | integer | no | Recommended shoe size. |
| `sizeFit` | string | no | Size fit label. |
| `cushioningScore` | number (double) | no | Per-dimension score. |
| `stabilityScore` | number (double) | no | Per-dimension score. |
| `supportScore` | number (double) | no | Per-dimension score. |
| `flexibilityScore` | number (double) | no | Per-dimension score. |
| `responsivenessScore` | number (double) | no | Per-dimension score. |
| `matchedAttrs` | [`MatchedAttr`](#matchedattr)[] | yes | Matched metric summary. |

### `RecommendedShoeMatch`

| Field | Type | Required | Description |
|---|---|---|---|
| `productId` | string | yes | Shoe id. |
| `matchScore` | number (double) | yes | Match score. |
| `matchReasons` | string[] | yes | Human-readable match reasons. |
| `recommendedSize` | integer | no | Recommended shoe size. |
| `sizeFit` | string | no | Size fit label. |
| `cushioningScore` | number (double) | no | Per-dimension score. |
| `stabilityScore` | number (double) | no | Per-dimension score. |
| `supportScore` | number (double) | no | Per-dimension score. |
| `flexibilityScore` | number (double) | no | Per-dimension score. |
| `responsivenessScore` | number (double) | no | Per-dimension score. |
| `matchedAttrs` | [`MatchedAttr`](#matchedattr)[] | yes | Matched metric summary. |

### `Needs`

| Field | Type | Required | Description |
|---|---|---|---|
| `cushioning` | number (double) | no | Need score. |
| `stability` | number (double) | no | Need score. |
| `support` | number (double) | no | Need score. |
| `flexibility` | number (double) | no | Need score. |
| `responsiveness` | number (double) | no | Need score. |

### `StatelessRecommendRequest`

| Field | Type | Required | Description |
|---|---|---|---|
| `firstClassType` | integer | no | Primary foot/spine class code. |
| `firstAccuracy` | number (double) | no | Primary class accuracy percent. |
| `secondClassType` | integer | no | Secondary class code. |
| `secondAccuracy` | number (double) | no | Secondary class accuracy percent. |
| `thirdClassType` | integer | no | Third class code. |
| `thirdAccuracy` | number (double) | no | Third class accuracy percent. |
| `leftFootLengthCm` | number (double) | no | Left foot length in cm. |
| `rightFootLengthCm` | number (double) | no | Right foot length in cm. |
| `leftFootWidthCm` | number (double) | no | Left foot width in cm. |
| `rightFootWidthCm` | number (double) | no | Right foot width in cm. |
| `leftTopPct` | number (double) | no | Left top pressure percent. |
| `leftMidPct` | number (double) | no | Left mid pressure percent. |
| `leftBotPct` | number (double) | no | Left bottom pressure percent. |
| `rightTopPct` | number (double) | no | Right top pressure percent. |
| `rightMidPct` | number (double) | no | Right mid pressure percent. |
| `rightBotPct` | number (double) | no | Right bottom pressure percent. |
| `leftTotalPct` | number (double) | no | Total left foot pressure percent. |
| `rightTotalPct` | number (double) | no | Total right foot pressure percent. |
| `cogX` | number (double) | no | Center of gravity x coordinate, normalized 0..1. |
| `cogY` | number (double) | no | Center of gravity y coordinate, normalized 0..1. |
| `gender` | string | no | Reserved; no current recommendation effect. |
| `limit` | integer (`@Min(1) @Max(50)`) | no | Recommendation limit. |

### `Brand`

| Field | Type | Required | Description |
|---|---|---|---|
| `brandName` | string | yes | Brand name. |
| `productCount` | integer (int64) | yes | Number of shoes for that brand. |

### `PagedResponse_Shoe`

| Field | Type | Required | Description |
|---|---|---|---|
| `content` | [`Shoe`](#shoe)[] | yes | Current page content. |
| `page` | integer | yes | Requested page number. |
| `size` | integer | yes | Effective page size. |
| `totalElements` | integer (int64) | yes | Total matched records before paging. |

### `Shoe`

| Field | Type | Required | Description |
|---|---|---|---|
| `productId` | string | yes | Shoe id. |
| `name` | string | yes | Product name. |
| `slug` | string | no | Product slug. |
| `brand` | string | no | Brand name. |
| `category` | string | no | Category. |
| `price` | number (double) | no | Numeric price from catalog. |
| `imageUrl` | string | no | Product image URL. |
| `archSupport` | string | no | Arch support label. |
| `metrics` | object | no | Requested metric names mapped to numeric values. `null` when no `metric` query is provided. |

### `ShoeDetail`

| Field | Type | Required | Description |
|---|---|---|---|
| `productId` | string | yes | Shoe id. |
| `name` | string | yes | Product name. |
| `slug` | string | no | Product slug. |
| `brand` | string | no | Brand name. |
| `category` | string | no | Category. |
| `price` | number (double) | no | Numeric price from catalog. |
| `imageUrl` | string | no | Product image URL. |
| `archSupport` | string | no | Arch support label. |
| `weightG` | number (double) | no | Weight in grams. |
| `dropLabMm` | number (double) | no | Lab-measured drop in mm. |
| `heelStackLabMm` | number (double) | no | Heel stack in mm. |
| `forefootStackLabMm` | number (double) | no | Forefoot stack in mm. |
| `midsoleSoftnessLabel` | string | no | Midsole softness label. |
| `midsoleSoftnessNew` | number (double) | no | Midsole softness value. |
| `shockAbsorptionLabel` | string | no | Shock absorption label. |
| `shockAbsorptionRaw` | number (double) | no | Shock absorption raw value. |
| `energyReturnLabel` | string | no | Energy return label. |
| `energyReturnRaw` | number (double) | no | Energy return raw value. |
| `torsionalRigidityRaw` | number (double) | no | Torsional rigidity raw value. |
| `heelCounterStiffnessRaw` | number (double) | no | Heel counter stiffness raw value. |
| `stiffnessLabel` | string | no | Stiffness label. |
| `stiffnessNNew` | number (double) | no | Stiffness value. |
| `breathabilityLabel` | string | no | Breathability label. |
| `breathabilityRaw` | number (double) | no | Breathability raw value. |
| `tractionRaw` | number (double) | no | Traction raw value. |
| `sizeRatingLabel` | string | no | Size rating label. |
| `sizeRatingRaw` | number (double) | no | Size rating raw value. |
| `forefootWidthMm` | number (double) | no | Forefoot width in mm. |
| `outsoleDurabilityMm` | number (double) | no | Outsole durability value. |
| `toeboxDurabilityRaw` | number (double) | no | Toebox durability raw value. |
| `overallScore` | number (double) | no | Overall score. |
| `expertScore` | number (double) | no | Expert score. |
| `userScore` | number (double) | no | User score. |
| `terrain` | string | no | Terrain label. |
| `strikePattern` | string | no | Strike pattern label. |
| `description` | string | no | Product description. |
| `orthoticFriendly` | boolean | no | Orthotic-friendly flag. |
| `removableInsole` | boolean | no | Removable insole flag. |
| `isLightweight` | boolean | no | Lightweight flag. |
| `hasPlate` | boolean | no | Plate flag. |
| `hasRocker` | boolean | no | Rocker flag. |
| `bestFor` | string[] | no | Best-use tags. |

### `ErrorResponse`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `false`. |
| `code` | string | no | Error code from common-core when available. |
| `message` | string | yes | Human-readable message. |
| `error` | string | no | Underlying detail. |
| `timestamp` | string (date-time) | no | Error timestamp when supplied by common response handling. |
