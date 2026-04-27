# graph-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables, request bodies, and response schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/graph-service`
Updated: 2026-04-27

Neo4j-backed shoe recommendation engine. Combines biomechanical fit (foot dimensions, pressure, gait class), collaborative filtering (similar users by primary class + foot dimensions ±5 cm), and brand preference (`PREFERS_BRAND`). User profile / latest measurement record are pulled via OpenFeign from `user-service` and `measure-service`.

**Servers**
- `https://api.example.com` — production gateway

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `0.0.1` |

---

## `GET` /api/recommendation/shoes

**Operation ID** &nbsp;`recommendShoes`
**Tags** &nbsp;`recommendation`
**Security** &nbsp;`permitAll` (no authentication)

Personalized shoe recommendation. Falls back to a random list when `userId` is blank or the user/measurement lookup fails.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `0.0.1`) | yes | Header-based API version selector. |
| query | `userId` | string | no | If blank, returns random recommendations. |
| query | `recordId` | string | no | If absent, the latest record is used. |
| query | `limit` | integer (`@Min(1) @Max(50)`) | no | Default `5`. |

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_RecommendedShoeList`](#cncresponse_recommendedshoelist) |
| **400** | `limit` out of range | `application/json` | [`ErrorResponse`](#errorresponse) |
| **502** | External service unreachable | `application/json` | [`ErrorResponse`](#errorresponse) |
| **500** | Neo4j query error | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 200 — example

```json
{
  "success": true,
  "data": [
    {
      "productId": "...",
      "shoeName": "...",
      "brand": "...",
      "category": "...",
      "price": "...",
      "year": 2025,
      "size": "270",
      "productUrl": "...",
      "imageUrl": "...",
      "score": 87.42,
      "brandMatch": 1,
      "attrMatchCount": 4,
      "matchedAttrs": [
        {
          "metric": "arch_support",
          "direction": "HIGH",
          "weight": 0.9,
          "target": null,
          "rawValue": 0.82,
          "classType": 1,
          "classRank": 1,
          "classRatio": 1.0
        }
      ]
    }
  ],
  "timestamp": "2026-04-27T08:00:00Z"
}
```

#### 400 — example

```json
{
  "success": false,
  "code": "CMN-400-001",
  "message": "Invalid request parameter. Field: limit, Value: 51",
  "timestamp": "2026-04-27T08:00:00Z"
}
```

---

## Schemas

### `CncResponse_RecommendedShoeList`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `true` on success. |
| `data` | [`RecommendedShoe`](#recommendedshoe)[] | yes | Ranked recommendations. |
| `timestamp` | string (date-time, UTC) | yes | — |

### `RecommendedShoe`

| Field | Type | Required | Description |
|---|---|---|---|
| `productId` | string | yes | Product identifier in the knowledge graph. |
| `shoeName` | string | yes | Model/product name. |
| `brand` | string | yes | Brand name. |
| `category` | string | no | Category name (via `IN_CATEGORY`). |
| `price` | string | no | Stored as string in the graph. |
| `year` | integer | no | Release year. |
| `size` | string | no | Shoe size. |
| `productUrl` | string | no | Gender-aware (`amazonMensUrl` / `amazonWomensUrl`) when present. |
| `imageUrl` | string | no | Product image URL. |
| `score` | number (double) | yes | `classScore × 10 + similarPositiveCnt × 2 + brandPrefScore × 1.5` (rounded 2dp). |
| `brandMatch` | integer (`0`/`1`) | yes | `1` if user has `PREFERS_BRAND` to brand. |
| `attrMatchCount` | integer | yes | Count of matched attrs (fit ≥ 0.75). |
| `matchedAttrs` | [`MatchedAttr`](#matchedattr)[] | yes | — |

### `MatchedAttr`

| Field | Type | Required | Description |
|---|---|---|---|
| `metric` | string | yes | Attribute name (e.g. `arch_support`). |
| `direction` | string (enum: `HIGH` `LOW` `MID`) | yes | Optimization direction. |
| `weight` | number (double) | no | Combined metric weight. |
| `target` | number (double) | no | Normalized target for `MID` direction. |
| `rawValue` | number (double) | no | Actual raw shoe metric value. |
| `classType` | integer | no | User foot class id this metric belongs to. |
| `classRank` | integer (`1` `2` `3`) | no | Class priority. |
| `classRatio` | number (double) | no | Relative accuracy vs. primary class. |

### `ErrorResponse`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `false`. |
| `code` | string | yes | Error code. |
| `message` | string | yes | Human-readable. |
| `error` | string | no | Underlying detail. |
| `timestamp` | string (date-time, UTC) | yes | — |

---

## Notes

- Endpoint is currently public — guard at the gateway.
- `GraphSyncService*` exists in code but is **not** exposed via HTTP yet.
- External calls (Feign): `AuthService.getUserInfo` (`api-version: 2025-07-11`), `MeasureService.getUserRecordInfo` / `getRecordById`.
