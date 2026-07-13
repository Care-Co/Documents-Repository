# agent-service

> 엔드포인트별 **Header · Request · Response** 정의 — 버전별 전량 전개, 소스는 실제 컨트롤러/DTO. 표기 — **굵은 필드 = 필수**, 규칙은 키워드만.

| 항목 | 값 |
|---|---|
| Source | `/Users/jonghak/GitHub/Care&Co/agent-service` (`develop-k3s`) |
| Updated | 2026-07-13 |
| 배포 상태 | **prod 는 구버전 배포 (릴리즈 웨이브 대기)** — `GET /llm/usage` · `POST /llm/chat/records/{recordId}/overall-summary` 엔드포인트와 `diagnostics` 응답 필드는 아직 배포 전 |
| Server | `https://api.example.com` |
| Base path | 모든 엔드포인트 `/llm/...` 하위 |

대화형 LLM 게이트웨이. OpenAI chat/responses tool loop 를 구동하고 `fisica-mcp` 툴을 agent-service 내부에서 직접 실행한다. 세션/메시지/토큰 사용량을 PostgreSQL 에 저장하고, 툴 호출 결과(예: `shoe_recommend`)는 응답의 `meta` 로 노출한다.

---

## 공통 규칙

- **버전** — 모든 엔드포인트 `api-version: 1.0.0` 단일 버전 (`ChattingController` 클래스 레벨 `@RequestMapping(version="1.0.0")`). 버전 협상은 요청 헤더 `api-version: x.y.z` (Spring API versioning, `useRequestHeader("api-version")` + `addSupportedVersions("1.0.0")`).
- **인증** — 서비스 자체 Spring Security 없음 (`permitAll`) — 메서드 시큐리티 애너테이션 없고 JWT 필터도 없다. 호출자 신원은 요청의 `user_id` (body 또는 query) 로 전달되며, 모든 엔드포인트는 처리 전에 호출자가 Física user 로 존재하는지 검증한다.
- **CORS** — `*` origin / `GET POST PUT DELETE PATCH OPTIONS` (`WebMvcConfig`).
- **바디** — 요청 바디가 있으면 `Content-Type: application/json`.
- **Rate limit** — 채팅 요청은 사용자별로 rate-limit 된다 (**400 requests / day**, Asia/Seoul 캘린더 일 기준 — `UserDailyUsageService.DAILY_REQUEST_LIMIT`).
- **응답 틀** — 모든 응답은 `CncResponse` envelope (아래 접기 참조) — 성공은 `success` + `data` 만 채워지고, 에러는 commoncore `GlobalExceptionHandler` 가 렌더링한다.

<details>
<summary><b>응답 envelope</b> — 성공/에러 JSON · 필드 정의 · 핸들러별 매핑</summary>

**성공** (`CncResponse`, commoncore) — 모든 성공 응답의 바깥 틀. 컨트롤러는 `success` + `data` 만 채운다. `CncResponse` 는 클래스 레벨 `@JsonInclude(NON_NULL)` 이므로 null 필드(`code`/`message`/`error`/`timestamp` 등)는 직렬화에서 생략된다.

```json
{
  "success": true,
  "data": {}
}
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `success` | boolean | yes | — |
| `data` | object | no | endpoint-specific |

**에러** (`CncResponse`, 에러 필드만 채워짐) — agent-service 는 별도 `ProblemDetail` 을 쓰지 않는다. commoncore `GlobalExceptionHandler` 가 auto-config (`CommonExceptionAutoConfiguration`) 로 등록되어 모든 예외를 `CncResponse` 로 렌더링한다. content-type `application/json`.

```json
{
  "success": false,
  "code": "E001",
  "message": "Fisica user not found",
  "error": "404 NOT_FOUND"
}
```

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `success` | boolean | yes | 에러 시 `false` |
| `code` | string \| null | no | `ErrorCodeV2` 코드(`E001`~`E019`) 또는 커스텀(`DAILY_CHAT_LIMIT_EXCEEDED`). `@Valid` 실패엔 `code` 없음 |
| `message` | string | yes | 에러 메시지 (resolved) |
| `error` | object \| string \| null | no | 상세 — `ResponseStatusException` 은 `statusCode.toString()`, `@Valid` 는 필드 에러 맵, 429 는 limit 상세 맵 |
| `timestamp` | string (date-time, UTC) \| null | no | 일부 핸들러(429)만 설정. `GlobalExceptionHandler` 경로는 미설정 → 생략 |

핸들러별 매핑 —

| 예외 | HTTP | `code` | 비고 |
|---|---|---|---|
| `ResponseStatusException` | 예외의 status (404/400/502) | `E001` (CHECK_PARAMETER 고정) | `message`=reason, `error`=`"<status>"`. **status 와 무관하게 code 는 항상 `E001`** ¹ |
| `MethodArgumentNotValidException` (`@Valid`) | 400 | (없음) | `message`=`"Not Valid"`, `error`=필드 에러 맵 |
| `CncException` (Feign 외부 호출 실패) | 502 | `E004` (EXTERNAL_SERVICE_ISSUE) | `CustomFeignErrorDecoder` 가 생성 |
| `DailyChatLimitExceededException` | 429 | `DAILY_CHAT_LIMIT_EXCEEDED` | 전용 `AgentExceptionHandler` (HIGHEST_PRECEDENCE), `Retry-After` 헤더 포함 |
| 그 외 `Exception` | 500 | `E003` (INTERNAL_SERVER_ERROR) | fallback |

> ¹ `GlobalExceptionHandler.handleResponseStatusException` 는 body `code` 를 `CHECK_PARAMETER.getCode()`(=`E001`) 로 하드코딩한다. HTTP status 는 예외의 실제 status 를 그대로 쓰지만 body 의 `code` 만 `E001` 로 고정된다.

</details>

---

| Method | Path (`/llm` 이하) | 버전 | 인증 | 설명 |
|---|---|---|---|---|
| POST | [`/chat`](#1-post-llmchat) | 1.0.0 | 공개 | 사용자 메시지를 보내고 agent 응답을 받는다 |
| POST | [`/chat/greeting`](#2-post-llmchatgreeting) | 1.0.0 | 공개 | agent 인사말로 세션을 연다 |
| POST | [`/chat/records/{recordId}/overall-summary`](#3-post-llmchatrecordsrecordidoverall-summary) | 1.0.0 | 공개 | 측정 record 에 대한 종합 요약을 생성한다 |
| GET | [`/usage`](#4-get-llmusage) | 1.0.0 | 공개 | 사용자의 당일 채팅 요청 사용량을 조회한다 |

---

## 1. `POST` /llm/chat

사용자 메시지를 보내고 agent 응답을 받는다 (OpenAI chat tool loop 를 native MCP 툴 실행과 함께 동기 실행). Física user 검증 후 **일일 요청 한도(400/day, Asia/Seoul)를 원자적으로 소비**한다 — 초과 시 429. 툴 호출 결과(`shoe_recommend` 등)는 `meta` 로 노출된다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`application/json` — `AgentChatRequest` (`@Valid` 적용 — `user_id` · `user_msg` `@NotBlank`).

```json
{
  "session_id": null,
  "user_id": "u1",
  "user_msg": "신발 추천해줘",
  "directives": {
    "tool": "shoe_recommend",
    "tool_args": {},
    "current_screen": "home",
    "screen_params": {},
    "trigger": "manual"
  }
}
```

### `1.0.0` — Response

**200 OK** — `AgentJobResponse`. 툴이 shoe 추천을 실행하면 `meta` 에 추천 목록 배열이 담긴다

```json
{
  "success": true,
  "data": {
    "session_id": "f7d8-...",
    "response": "추천드릴 신발 목록입니다.",
    "created_at": "2026-04-27T08:00:00Z",
    "meta": [ { "shoeId": "...", "name": "..." } ],
    "status": "COMPLETED",
    "processing_time": 0.42,
    "client_actions": null,
    "diagnostics": null
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>user_id</code> / <code>user_msg</code> blank (<code>@Valid</code>)</summary>

```json
{ "success": false, "message": "Not Valid", "error": { "user_msg": ["must not be blank"] } }
```

`@Valid` 실패는 `code` 없이 `message`=`"Not Valid"` + 필드 에러 맵.

</details>

<details>
<summary><b>404 Not Found</b> — <code>user_id</code> 가 user-service 에 없음</summary>

```json
{ "success": false, "code": "E001", "message": "Fisica user not found", "error": "404 NOT_FOUND" }
```

</details>

<details>
<summary><b>429 Too Many Requests</b> — 일일 채팅 한도 초과 (400/day, Asia/Seoul)</summary>

`Retry-After: <seconds>` 헤더 포함. 전용 `AgentExceptionHandler`.

```json
{
  "success": false,
  "code": "DAILY_CHAT_LIMIT_EXCEEDED",
  "message": "Daily chat request limit exceeded",
  "error": {
    "type": "daily_chat_request_limit",
    "limit": 400,
    "reset_at": "2026-07-10T00:00:00+09:00",
    "timezone": "Asia/Seoul",
    "retry_after_seconds": 57600
  },
  "timestamp": "2026-07-09T08:00:00Z"
}
```

</details>

<details>
<summary><b>502 Bad Gateway</b> — 외부 LLM/MCP 호출 실패 또는 user-service 조회 실패</summary>

```json
{ "success": false, "code": "E004", "message": "External service error. ServiceName: ..., Detail: ...", "error": "..." }
```

Feign 외부 호출 실패는 `CustomFeignErrorDecoder` → `CncException(EXTERNAL_SERVICE_ISSUE)` → 502 `E004`. user 검증 IO 실패는 `ResponseStatusException(BAD_GATEWAY)` → 502 `E001`.

</details>

</details>

### Request 필드 정의 — `AgentChatRequest`

| 필드 | 타입 | 필수 | Validation | 설명 |
|---|---|---|---|---|
| `session_id` | string | no | — | null 이면 새 세션 생성 |
| **`user_id`** | string | yes | `@NotBlank` | Física user id |
| **`user_msg`** | string | yes | `@NotBlank` | 사용자 메시지 |
| `directives` | `Directives` | no | — | 툴/화면 컨텍스트 (아래) |

**`Directives`** (모든 필드 optional, `@JsonInclude(NON_NULL)`)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `tool` | string | no | 툴 이름 (예: `shoe_recommend`) |
| `tool_args` | object (`Map<String,Object>`) | no | 툴 인자 |
| `current_screen` | string | no | UI 컨텍스트 |
| `screen_params` | object | no | 화면 파라미터 |
| `trigger` | string | no | 트리거 이벤트 |

### Response 필드 정의 — `AgentJobResponse`

`meta` / `processing_time` / `client_actions` / `diagnostics` 는 `@JsonInclude(NON_NULL)` — null 이면 생략.

| 필드 | 타입 | 설명 |
|---|---|---|
| `session_id` | string | — |
| `response` | string | agent 텍스트 |
| `created_at` | string (date-time) | — |
| `meta` | object \| array \| null | 툴 출력 (shoe 추천 목록 등). 툴 미호출 시 생략 |
| `status` | string (`COMPLETED` \| `FAILED` \| `PENDING`) | — |
| `processing_time` | number (double) \| null | 초 |
| `client_actions` | array\<object\> \| null | UI 액션 |
| `diagnostics` | object (`Map<String,Object>`) \| null | 진단 정보 (develop 신규) |

---

## 2. `POST` /llm/chat/greeting

agent 인사말로 세션을 연다. 컨트롤러가 **`@Valid` 를 적용하지 않으므로** `user_msg` 가 비어도 통과한다 (`greeting` 서비스가 null 이면 `""` 로 대체). `directives` 는 무시된다. 일일 한도를 소비하지 않아 429 가 없다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

`application/json` — `AgentChatRequest` (`@Valid` 미적용 — `user_msg` 생략 가능). 서비스는 `user_id` 만 필수로 사용한다.

```json
{ "user_id": "u1" }
```

### `1.0.0` — Response

**200 OK** — `AgentJobResponse` (`meta` 는 항상 null → 생략)

```json
{
  "success": true,
  "data": {
    "session_id": "f7d8-...",
    "response": "안녕하세요! 무엇을 도와드릴까요?",
    "created_at": "2026-04-27T08:00:00Z",
    "status": "COMPLETED",
    "processing_time": 0.31
  }
}
```

<details>
<summary><b>404 Not Found</b> — <code>user_id</code> 가 user-service 에 없음 (blank 포함)</summary>

```json
{ "success": false, "code": "E001", "message": "Fisica user not found", "error": "404 NOT_FOUND" }
```

`@Valid` 가 없어 blank `user_id` 도 400 이 아니라 user 검증에서 404 로 떨어진다.

</details>

<details>
<summary><b>502 Bad Gateway</b> — user-service 조회 실패 또는 외부 호출 실패</summary>

```json
{ "success": false, "code": "E001", "message": "Failed to validate user", "error": "502 BAD_GATEWAY" }
```

</details>

</details>

### Request 필드 정의 — `AgentChatRequest` (greeting)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `session_id` | string | no | null 이면 새 세션 생성 |
| **`user_id`** | string | yes | 서비스 필수 (Bean Validation 은 아님) |
| `user_msg` | string | no | greeting 은 생략 가능 (null → `""`) |
| `directives` | `Directives` | no | 무시됨 |

### Response 필드 정의 — `AgentJobResponse`

§1 와 동일 스키마. greeting 은 `meta` = null, `client_actions` = null (생략), `diagnostics` = 서버 응답에 있으면 노출.

| 필드 | 타입 | 설명 |
|---|---|---|
| `session_id` | string | — |
| `response` | string | 인사말 텍스트 |
| `created_at` | string (date-time) | — |
| `meta` | null | greeting 은 항상 생략 |
| `status` | string (`COMPLETED` \| `FAILED` \| `PENDING`) | 서버 status 대문자화, 없으면 `COMPLETED` |
| `processing_time` | number (double) \| null | 초 |
| `diagnostics` | object \| null | 진단 정보 |

---

## 3. `POST` /llm/chat/records/{recordId}/overall-summary

측정 record 에 대한 종합 요약을 생성한다. path 의 `recordId` 와 body 의 `user_id` 로 `fisicaAgentService.recordOverallSummary` 를 동기 호출한다. `recordId` blank → 400, Física user 검증 후 처리. 일일 한도를 소비하지 않는다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |
| `Content-Type` | `application/json` | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

path `recordId` (string) + `application/json` — `RecordReportRequest` (`@Valid` — `user_id` `@NotBlank`).

```json
{ "user_id": "u1" }
```

### `1.0.0` — Response

**200 OK** — `AgentJobResponse` (`session_id` 는 서버 응답에서, `meta` 는 null → 생략, `created_at` 은 서버 수신 시각)

```json
{
  "success": true,
  "data": {
    "session_id": "srv-session-...",
    "response": "측정 결과 종합 요약입니다.",
    "created_at": "2026-04-27T08:00:00Z",
    "status": "COMPLETED",
    "processing_time": 1.23
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>recordId</code> blank 또는 <code>user_id</code> blank (<code>@Valid</code>)</summary>

```json
{ "success": false, "code": "E001", "message": "recordId is required", "error": "400 BAD_REQUEST" }
```

`recordId` blank → `ResponseStatusException(BAD_REQUEST, "recordId is required")` (code `E001`). `user_id` blank → `@Valid` 실패 (code 없이 `"Not Valid"` + 필드 맵).

</details>

<details>
<summary><b>404 Not Found</b> — <code>user_id</code> 가 user-service 에 없음</summary>

```json
{ "success": false, "code": "E001", "message": "Fisica user not found", "error": "404 NOT_FOUND" }
```

</details>

<details>
<summary><b>502 Bad Gateway</b> — 외부 호출 실패 또는 user-service 조회 실패</summary>

```json
{ "success": false, "code": "E004", "message": "External service error. ServiceName: ..., Detail: ...", "error": "..." }
```

</details>

</details>

### Request 필드 정의 — `RecordReportRequest` (+ path)

| 필드 | 타입 | 필수 | Validation | 위치 |
|---|---|---|---|---|
| **`recordId`** | string | yes | blank → 400 | path |
| **`user_id`** | string | yes | `@NotBlank` | body |

### Response 필드 정의 — `AgentJobResponse`

§1 와 동일 스키마. `session_id` 는 서버(`fisicaAgentService`) 응답값, `created_at` 은 agent-service 수신 시각(`OffsetDateTime.now()`), `meta` 는 항상 생략.

| 필드 | 타입 | 설명 |
|---|---|---|
| `session_id` | string | 서버 응답의 sessionId |
| `response` | string | 요약 텍스트 |
| `created_at` | string (date-time) | agent-service 수신 시각 |
| `meta` | null | 항상 생략 |
| `status` | string (`COMPLETED` \| `FAILED` \| `PENDING`) | 서버 status 대문자화, 없으면 `COMPLETED` |
| `processing_time` | number (double) \| null | 초 |
| `client_actions` | array\<object\> \| null | UI 액션 |
| `diagnostics` | object \| null | 진단 정보 |

---

## 4. `GET` /llm/usage

사용자의 당일 채팅 요청 사용량을 조회한다. Asia/Seoul 캘린더 일 기준으로 `used` / `remaining` / `reset_at` 을 계산한다. `user_id` 쿼리 파라미터 필수. 조회 전에 Física user 검증을 수행한다.

### Header

| 헤더 | 값 | 필수 |
|---|---|---|
| `api-version` | `1.0.0` (단일 버전) | yes |

<details>
<summary><b><code>1.0.0</code> — Request · Response</b></summary>

### `1.0.0` — Request

바디 없음. 쿼리 파라미터 —

| 파라미터 | 타입 | 필수 | 규칙 |
|---|---|---|---|
| **`user_id`** | string | yes | blank 이면 400 (`user_id is required`) |

### `1.0.0` — Response

**200 OK** — `DailyUsageResponse`

```json
{
  "success": true,
  "data": {
    "user_id": "u1",
    "usage_date": "2026-07-09",
    "timezone": "Asia/Seoul",
    "limit": 400,
    "used": 12,
    "remaining": 388,
    "reset_at": "2026-07-10T00:00:00+09:00",
    "retry_after_seconds": 57600
  }
}
```

<details>
<summary><b>400 Bad Request</b> — <code>user_id</code> 누락/blank</summary>

```json
{ "success": false, "code": "E001", "message": "user_id is required", "error": "400 BAD_REQUEST" }
```

</details>

<details>
<summary><b>404 Not Found</b> — <code>user_id</code> 가 user-service 에 없음</summary>

```json
{ "success": false, "code": "E001", "message": "Fisica user not found", "error": "404 NOT_FOUND" }
```

</details>

<details>
<summary><b>502 Bad Gateway</b> — user-service 조회 실패 (NOT_FOUND 외 오류)</summary>

```json
{ "success": false, "code": "E001", "message": "Failed to validate user", "error": "502 BAD_GATEWAY" }
```

`UserApiClient.existsUser` 가 NOT_FOUND 외 응답/IO 오류 시 `ResponseStatusException(BAD_GATEWAY, "Failed to validate user")`.

</details>

</details>

### Request 필드 정의

| 필드 | 타입 | 필수 | 위치 |
|---|---|---|---|
| **`user_id`** | string | yes | query param |

### Response 필드 정의 — `DailyUsageResponse`

| 필드 | 타입 | 설명 |
|---|---|---|
| `user_id` | string | — |
| `usage_date` | string (date, Asia/Seoul) | 당일 (`yyyy-MM-dd`) |
| `timezone` | string | 항상 `Asia/Seoul` |
| `limit` | integer | 일일 한도 (현재 `400`) |
| `used` | integer | 당일 소비량 |
| `remaining` | integer | `max(0, limit - used)` |
| `reset_at` | string (date-time, offset) | 다음 날 00:00 Asia/Seoul (`+09:00`) |
| `retry_after_seconds` | integer (long) | 리셋까지 남은 초 |
