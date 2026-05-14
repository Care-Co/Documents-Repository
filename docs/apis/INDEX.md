# Care&Co Workspace API Index

Updated: 2026-05-14
Style: **OpenAPI 3.1 / Swagger** (markdown rendering of `openapi.yaml`)

## Documentation Standard

- One markdown file per service.
- Each operation block follows OpenAPI conventions:
  - `## METHOD /path`
  - **Operation ID** / **Tags** / **Security**
  - **Parameters** table (`In | Name | Type | Required | Description`)
  - **Request body** (content type + schema reference)
  - **Responses** table (`Status | Content-Type | Schema`) with example payloads
- Reusable types are collected in `## Schemas` at the end of each document.

## Service API Documents

| Doc | Routes | Versioning | Notes |
|---|---|---|---|
| [user-service](./user-service.md) | 48 endpoints (auth + user + admin + public + audit) | header `api-version: 1.0.0` | OAuth2 controller deprecated |
| [b2b-service](./b2b-service.md) | ~40 endpoints (auth + user + organization + membership + invite + device + measurement + feedback + license + billing) | unversioned | 별도 user pool (b2b_users), 다이어그램 + cross-MSA 흐름 + 코드 매핑 포함 |
| [measure-service](./measure-service.md) | 7 endpoints (record + activity + internal sync) | `api-version: 1.0.0` | record DTO `2026-01-13`, activity DTO `2026-02-02` |
| [noti-service](./noti-service.md) | 12 endpoints (FCM + device + topic + history) | `api-version: 1.0.0` | async job pattern |
| [storage-service](./storage-service.md) | 9 endpoints (v1 deprecated + v2 current) | `api-version: 1.0.0` | v1 sunset 2026-12-31 |
| [admin-service](./admin-service.md) | 4 endpoints (Spring Boot Admin instance view) | `api-version` `1.0.0`, `2025-06-24`, `0.0.1` | wraps SBA |
| [agent-service](./agent-service.md) | 2 endpoints (LLM chat + greeting) | `api-version: 1.0.0` | proxies Física Agent |
| [graph-service](./graph-service.md) | 1 endpoint (Neo4j shoe recommendation) | `api-version: 0.0.1` | random fallback |
| [payment-service](./payment-service.md) | 17 REST + 1 webhook | unversioned | Paddle Billing v2 |
| [ML-service](./ml-service.md) | 6 routes (footprint + pose + measurement) | `api-version: 1.0.0` (reserved, not branched) | Flask blueprints |
| [device-service](./device-service.md) | 11 REST + 1 gRPC | `api-version: 1.0.0` | device registry, owner validation, OTA firmware |

## Repositories Without API Controllers

- `gateway-service` — request routing
- `eureka-service` — service discovery
- `config-service` / `config-file` — externalized configuration
- `common-libs` — shared library
- `B2B_WEB` — Next.js frontend (no public REST surface)
- `Documents-Repository` — this repo

## Samples

`_samples/` contains the original style-comparison drafts (`user-register_style-B-github.md`, `user-register_style-E-openapi.md`) used to choose this style. Style E (OpenAPI) was adopted.
