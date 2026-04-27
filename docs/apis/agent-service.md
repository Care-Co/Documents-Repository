# agent-service

> OpenAPI 3.1 — rendered from `openapi.yaml`. Field tables and schemas mirror `components/schemas`.

Source: `/Users/jonghak/GitHub/Care&Co/agent-service`
Updated: 2026-04-27

Conversational LLM gateway. Wraps an external **Física Agent API** for chat + greeting flows, persists sessions/messages/usage in MySQL, and aggregates token usage. Tool-call output (e.g. `shoe_recommend`) is surfaced via `meta` on the response.

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

Send a user message; returns the agent reply (synchronously calls Física via Feign).

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
| **502** | External Física API failure | `application/json` | [`ErrorResponse`](#errorresponse) |

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
| **502** | External Física API failure | `application/json` | [`ErrorResponse`](#errorresponse) |

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

## External calls (informational, OpenFeign — base `${etc.agent}`)

| Method | Path | Notes |
|---|---|---|
| POST | `/api/chat` | main chat |
| POST | `/api/chat/greeting` | greeting |
| GET | `/api/v1/recommend/shoes?user_id=` | shoe recommendation |
| GET | `/session/{session_id}` | history retrieval |
