# Care&Co Workspace API Index

Updated: 2026-07-13
Style: **예시-우선 양식 B** — 엔드포인트별 Header · Request · Response, 버전별 전량 전개

## Documentation Standard (양식 B)

- 서비스당 markdown 파일 1개. 소스는 실제 컨트롤러/DTO.
- 엔드포인트 블록 구조는 아래 순서.
  - ``## N. `METHOD` /path`` — 도메인 묶음 순으로 번호 부여
  - `### Header` 표 (`헤더 | 값 | 필수`) + `### Security` (권한/본인 확인 규칙)
  - `### 버전` 표 — **복수 버전 엔드포인트만**, 버전 간 차이를 행별로
  - 버전마다 ``### `x.y.z` — Request`` / ``### `x.y.z` — Response`` — **동일해도 전량 전개**, 샘플 JSON 포함
  - 성공 응답은 `**200 OK**` + JSON 펼침, 실패 응답은 `<details><summary><b>400 Bad Request</b> — 사유</summary>` 접기
  - 필드 정의 그룹표 — **굵은 필드 = 필수**, 세부 규칙은 각주 ¹²³
- 재사용 타입의 `## Schemas` 부록 **금지** — 필드 정의는 해당 엔드포인트에 인라인.
- 규모가 큰 문서(b2b-service)는 추가로 — 도메인 헤더 + 도메인별 엔드포인트 표(설명·인증·섹션 링크)를 그룹 시작 앞에 배치, 버전별 Request · Response 는 `<details>` 접기.

## Service API Documents

| Doc | Routes | Versioning | Notes |
|---|---|---|---|
| [user-service](./user-service.md) | 34 endpoints (auth + user + b2b-center + admin + public + audit) | header `api-version` — `1.0.0` / `1.0.2` / `1.1.0` / `1.1.1` (weightUnit·capture-settings 는 `1.1.x`) | OAuth2 controller deprecated. b2b-center license-detail 는 b2b gRPC 합성 (0.0.78) |
| [b2b-service](./b2b-service.md) | 57 endpoints (auth + user + organization + membership + invite + device + measurement + feedback + license + billing + B2C/availability) | **전부 versioned, 정확 일치만** — users `1.0.0`/`1.0.1` (country alpha-3), organizations `1.0.0`~`1.0.2`, devices `1.0.0`/`1.0.1`, 나머지 `1.0.0` 단일 | 별도 user pool (b2b_users). country alpha-3 + `UNKNOWN` sentinel (V19). memberNumber `M260608-001` (V16). B2cMemberQuery gRPC (0.0.78) |
| [measure-service](./measure-service.md) | 8 endpoints (record + activity + internal sync) | `api-version: 1.0.0` / `1.0.1` (POST records) | record DTO `2026-01-13` (V9: `deviceId` / `deviceSerial` / `source` / `appVersion` 노출), activity DTO `2026-02-02` |
| [noti-service](./noti-service.md) | 9 endpoints (FCM + device + topic + history) | `api-version: 1.0.0` | async job pattern |
| [storage-service](./storage-service.md) | 9 endpoints (v1 deprecated + v2 current) | `api-version: 1.0.0` | v1 sunset 2026-12-31 |
| [admin-service](./admin-service.md) **(deprecated)** | 2 endpoints (Spring Boot Admin instance view) | `api-version` `1.0.0`, `2025-06-24`, `0.0.1` | **미사용 서비스** — wraps SBA |
| [agent-service](./agent-service.md) | 4 endpoints (LLM chat + greeting) | `api-version: 1.0.0` | proxies Física Agent |
| [graph-service](./graph-service.md) | 7 endpoints (shoe recommendation + catalog) | `api-version: 1.0.0` | in-memory `shoes.json` |
| [payment-service](./payment-service.md) | 28 endpoints (REST + webhook) | unversioned | Paddle Billing v2 |
| [ML-service](./ml-service.md) | 8 routes (footprint + pose + measurement) | `api-version: 1.0.0` (reserved, not branched) | Flask blueprints |
| [device-service](./device-service.md) | 18 REST + 4 gRPC | `api-version: 1.0.0` (proto v0.0.73) | 출고 풀 / owner 별 등록 / measure MAC lookup / B2B 자산 + 등록 preview / 탈퇴 worker 일괄 unregister / battery+firmware metrics / OTA / deviceNumber `FS2-001` (V15) |

## Repositories Without API Controllers

- `gateway-service` — request routing
- `eureka-service` — service discovery
- `config-service` / `config-file` — externalized configuration
- `common-libs` — shared library
- `B2B_WEB` — Next.js frontend (no public REST surface)
- `Documents-Repository` — this repo
