# agent-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/agent-service`
Updated: 2026-06-08

Conversational LLM gateway. Drives an OpenAI chat-completions tool loop and executes `fisica-mcp` tools directly inside agent-service, persists sessions/messages/usage in PostgreSQL, and aggregates token usage. Tool-call output (e.g. `shoe_recommend`) is surfaced via `meta` on the response.

Chat requests are rate-limited per user (15 requests / day, Asia/Seoul calendar day). Both chat and greeting endpoints validate that the caller exists as a Física user before processing.

**Servers**
- `https://api.example.com`

**Common headers**

| Header | Value |
|---|---|
| `api-version` | `1.0.0` |
| `Content-Type` | `application/json` |

**CORS** &nbsp;`*` origin / `GET POST PUT DELETE PATCH OPTIONS`. **Security** &nbsp;none (`permitAll`).

---

## `POST` /llm/chat

**Operation ID** &nbsp;`generateChat`
**Tags** &nbsp;`agent`
**Security** &nbsp;none

Send a user message; returns the agent reply (synchronously runs the OpenAI chat tool loop with native MCP tool execution).

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | — |

### Request body

`application/json` &nbsp;**Required**

**Schema** — [`AgentChatRequest`](#agentchatrequest)

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

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_AgentJobResponse`](#cncresponse_agentjobresponse) |
| **400** | Validation failed (`user_id` / `user_msg` blank) | `application/json` | [`ErrorResponse`](#errorresponse) |
| **404** | `user_id` not found in user-service (`Fisica user not found`) | `application/json` | [`ErrorResponse`](#errorresponse) |
| **429** | Daily chat request limit exceeded (15 / day, Asia/Seoul) | `application/json` | [`ErrorResponse`](#errorresponse) |
| **502** | User-service lookup failure or external LLM/MCP error | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 200 — example

```json
{
  "success": true,
  "data": {
    "session_id": "...",
    "response": "추천드릴 신발 목록입니다.",
    "created_at": "2026-04-27T08:00:00Z",
    "meta": { "shoes": ["..."] },
    "status": "COMPLETED",
    "processing_time": 0.42,
    "client_actions": null
  },
  "timestamp": "2026-04-27T08:00:00Z"
}
```

---

## `POST` /llm/chat/greeting

**Operation ID** &nbsp;`greeting`
**Tags** &nbsp;`agent`
**Security** &nbsp;none

Open a session with an agent greeting. `directives` is ignored. Note: the controller does **not** apply `@Valid`, so `user_msg` may be empty.

### Parameters

| In | Name | Type | Required | Description |
|---|---|---|---|---|
| header | `api-version` | string (enum: `1.0.0`) | yes | — |

### Request body

`application/json` &nbsp;**Required**

**Schema** — [`AgentChatRequest`](#agentchatrequest)

```json
{ "user_id": "u1" }
```

### Responses

| Status | Description | Content-Type | Schema |
|---|---|---|---|
| **200** | OK | `application/json` | [`CncResponse_AgentJobResponse`](#cncresponse_agentjobresponse) |
| **404** | `user_id` not found in user-service (`Fisica user not found`) | `application/json` | [`ErrorResponse`](#errorresponse) |
| **502** | User-service lookup failure | `application/json` | [`ErrorResponse`](#errorresponse) |

#### 200 — example

```json
{
  "success": true,
  "data": {
    "session_id": "f7d8...",
    "response": "안녕하세요! 무엇을 도와드릴까요?",
    "created_at": "2026-04-27T08:00:00Z",
    "meta": null,
    "status": "COMPLETED",
    "processing_time": 0.31,
    "client_actions": null
  },
  "timestamp": "2026-04-27T08:00:00Z"
}
```

---

## Schemas

### `AgentChatRequest`

| Field | Type | Required | Constraints | Description |
|---|---|---|---|---|
| `session_id` | string | no | — | New session is created when null. |
| `user_id` | string | yes | `@NotBlank` | — |
| `user_msg` | string | yes (chat) / no (greeting) | `@NotBlank` on `/llm/chat` | — |
| `directives` | [`Directives`](#directives) | no | — | — |

### `Directives`

| Field | Type | Required | Description |
|---|---|---|---|
| `tool` | string | no | Tool name (e.g. `shoe_recommend`). |
| `tool_args` | object (`Map<String, Object>`) | no | Tool arguments. |
| `current_screen` | string | no | UI context. |
| `screen_params` | object | no | Screen params. |
| `trigger` | string | no | Trigger event. |

### `CncResponse_AgentJobResponse`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | — |
| `data` | [`AgentJobResponse`](#agentjobresponse) | yes | — |
| `timestamp` | string (date-time, UTC) | yes | — |

### `AgentJobResponse`

| Field | Type | Required | JSON name | Description |
|---|---|---|---|---|
| `sessionId` | string | yes | `session_id` | — |
| `response` | string | yes | `response` | Agent text. |
| `createdAt` | string (date-time) | yes | `created_at` | — |
| `meta` | object | no | `meta` | Tool output; null when no tool was called. |
| `status` | string (enum: `COMPLETED` `FAILED` `PENDING`) | yes | `status` | — |
| `processingTime` | number (double) | no | `processing_time` | Seconds. |
| `clientActions` | array<object> | no | `client_actions` | UI actions. |

### `ErrorResponse`

| Field | Type | Required | Description |
|---|---|---|---|
| `success` | boolean | yes | Always `false`. |
| `code` | string | yes | Error code. |
| `message` | string | yes | Human-readable. |
| `timestamp` | string (date-time, UTC) | yes | — |

---

## Persistence (informational)

| Entity | Notes |
|---|---|
| `ChatSession` | `id`, `userId`, `lastActivityAt`, embedded `SessionUsage` |
| `ChatMessage` | role (`USER`/`AGENT`), content, status, embedded `ToolCall` & `ChatUsage` |
| `SessionUsage` | request count, total/today tokens, token limit |
| `ChatUsage` | per-message `inputTokens`, `outputTokens` |
| `ToolCall` | `toolName`, `status`, raw output, metadata |
| `fisica_agent_user_daily_usage` | `(user_id, usage_date)` 유니크, `request_count` — daily 15 cap counter |

## External calls (informational)

| Method | Path | Base | Notes |
|---|---|---|---|
| GET | `/api/public/users/{userId}` | user-service | existence check before chat / greeting |
| (LLM) | OpenAI chat-completions API | OpenAI | tool loop driven from `OpenAiChatToolClient` |
| (MCP) | `fisica-mcp` tools | internal | executed directly via `McpToolClient` (no remote `/api/chat` Feign call) |
