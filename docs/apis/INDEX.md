# Care&Co Workspace API Index

Updated: 2026-06-08
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
| [user-service](./user-service.md) | 49 endpoints (auth + user + b2b-center + admin + public + audit) | header `api-version: 1.0.0` | OAuth2 controller deprecated. b2b-center license-detail 는 b2b gRPC 합성 (0.0.78) |
| [b2b-service](./b2b-service.md) | 48 endpoints (auth + user + organization + membership + invite + device + measurement + feedback + license + billing + B2C/availability) | mostly unversioned, device 응답 `api-version: 1.0.0` / `1.0.1` (deviceType 0.0.63 + deviceNumber 0.0.73) | 별도 user pool (b2b_users). memberNumber `M260608-001` (V16). B2cMemberQuery gRPC (0.0.78) — B2C 회원 이용권 상세 |
| [measure-service](./measure-service.md) | 8 endpoints (record + activity + internal sync) | `api-version: 1.0.0` / `1.0.1` (POST records) | record DTO `2026-01-13` (V9: `deviceId` / `deviceSerial` / `source` / `appVersion` 노출), activity DTO `2026-02-02` |
| [noti-service](./noti-service.md) | 12 endpoints (FCM + device + topic + history) | `api-version: 1.0.0` | async job pattern |
| [storage-service](./storage-service.md) | 9 endpoints (v1 deprecated + v2 current) | `api-version: 1.0.0` | v1 sunset 2026-12-31 |
| [admin-service](./admin-service.md) **(deprecated)** | 4 endpoints (Spring Boot Admin instance view) | `api-version` `1.0.0`, `2025-06-24`, `0.0.1` | **미사용 서비스** — wraps SBA |
| [agent-service](./agent-service.md) | 2 endpoints (LLM chat + greeting) | `api-version: 1.0.0` | proxies Física Agent |
| [graph-service](./graph-service.md) | 7 endpoints (shoe recommendation + catalog) | `api-version: 1.0.0` | in-memory `shoes.json` |
| [payment-service](./payment-service.md) | 17 REST + 1 webhook | unversioned | Paddle Billing v2 |
| [ML-service](./ml-service.md) | 6 routes (footprint + pose + measurement) | `api-version: 1.0.0` (reserved, not branched) | Flask blueprints |
| [device-service](./device-service.md) | 18 REST + 4 gRPC | `api-version: 1.0.0` (proto v0.0.73) | 출고 풀 / owner 별 등록 / measure MAC lookup / B2B 자산 + 등록 preview / 탈퇴 worker 일괄 unregister / battery+firmware metrics / OTA / deviceNumber `FS2-001` (V15) |

## Repositories Without API Controllers

- `gateway-service` — request routing
- `eureka-service` — service discovery
- `config-service` / `config-file` — externalized configuration
- `common-libs` — shared library
- `B2B_WEB` — Next.js frontend (no public REST surface)
- `Documents-Repository` — this repo

## Samples

`_samples/` contains the original style-comparison drafts (`user-register_style-B-github.md`, `user-register_style-E-openapi.md`) used to choose this style. Style E (OpenAPI) was adopted.
- [profile-settings-fields.md](profile-settings-fields.md) — 2026-07 프로필·측정 설정 API 필드 전정의 (register/PATCH/GET users · capture-settings)
