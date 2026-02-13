# Redis Streams SSE Resume — API Reference

**Status:** Implemented
**Date:** February 2026
**Backend Source:** [flamingo-fer](https://github.com/sybill-ai-engineering/flamingo-fer) — `src/sybill_py/runtimes/skynet/routers/chats/`
**Design Doc:** [Redis Streams SSE Resilience](../others/redis_streams_sse_resilience.md)

---

## Table of Contents

1. [Overview](#1-overview)
2. [API Endpoints](#2-api-endpoints)
3. [SSE Event Format](#3-sse-event-format)
4. [Resume API Examples](#4-resume-api-examples)
5. [Error Responses](#5-error-responses)
6. [Assumptions & Constraints](#6-assumptions--constraints)
7. [Backend Source Reference](#7-backend-source-reference)

---

## 1. Overview

The backend dual-publishes every SSE event to both the client and a Redis Stream. Each SSE event now includes an `id` field containing the Redis Stream message ID.

A new resume endpoint allows clients to reconnect after an interruption and receive only the missed events.

### Key Concepts

- **Streams are run-scoped.** Each user message triggers a new `runId`. The Redis stream key is `ask_sybill:chat:{threadId}:{runId}`. Both `threadId` and `runId` are required for resume.
- **The `ack` event is the source of truth** for `threadId` and `runId`. These values are in the first SSE event of every stream.
- **The `id` field on every SSE event** is the Redis Stream message ID. Store it as an opaque string.
- **Feature flag:** `REDIS_STREAM_PERSISTENCE` (backend, per-user). When disabled, SSE events have no `id` field and the resume endpoint returns 404.

---

## 2. API Endpoints

### 2.1 `POST /agents/chat/v2` — Send Chat Message (Existing)

> **Source:** `routers/chats/index.py:169-181`

No changes to request format. When `REDIS_STREAM_PERSISTENCE` is enabled for the user, SSE events in the response now include an `id:` line.

#### Request

```http
POST /agents/chat/v2
Content-Type: application/json
Authorization: Bearer <token>
```

```json
{
  "message": { "content": "What is our Q4 revenue?" },
  "thread_id": "550e8400-e29b-41d4-a716-446655440000",
  "parent_message_id": "7f3b2a1c-e29b-41d4-a716-446655440000",
  "is_temporary": true,
  "launch_point": "chat_page",
  "web_search_enabled": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | `{ content: string }` | Yes | User's message |
| `thread_id` | UUID | No | Omit for new chat |
| `parent_message_id` | UUID | No | Message being replied to |
| `is_temporary` | bool | No | Default `true` |
| `launch_point` | string | No | Where the chat was initiated |
| `web_search_enabled` | bool | No | Enable web search |

#### Response

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

Example SSE output:

```
id: 1707000000000-0
event: ack
data: {"threadId":"550e8400-...","messageId":"7f3b2a1c-0001","runId":"9d4e5f6a-..."}

id: 1707000000001-0
event: message
data: {"content":"What is our Q4 revenue?"}

id: 1707000000002-0
event: delta
data: {"content":"Based on"}

id: 1707000000003-0
event: delta
data: {"content":" the data, our Q4 revenue was $10M."}

id: 1707000000004-0
event: citations
data: {"sources":[{"title":"Q4 Report","url":"/calls/abc123"}]}

id: 1707000000005-0
event: done
data: {"status":"complete","copilotStatus":"SUCCESS"}
```

The `id` is a Redis Stream message ID (`<timestamp_ms>-<sequence>`). Treat as an opaque string.

---

### 2.2 `GET /agents/chat/{threadId}/run/{runId}/resume` — Resume Stream (New)

> **Source:** `routers/chats/index.py:184-210`
> **Handler:** `routers/chats/chat_utils.py:1375-1414`
> **Generator:** `routers/chats/chat_utils.py:1327-1372`

Resumes an interrupted SSE stream. The server:
1. **Catch-up:** Reads buffered messages from Redis after the given offset
2. **Live:** If the stream is still active, subscribes to live updates via `XREAD BLOCK`

The transition between catch-up and live is invisible to the client.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `threadId` | UUID | Yes | Chat thread ID (from `ack` event) |
| `runId` | UUID | Yes | Run ID (from `ack` event) |

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `lastMessageId` | string | No | Last received Redis Stream message ID. Omit to replay from the beginning. |

#### Request Headers

```http
Authorization: Bearer <token>
```

Same Auth0 JWT as `POST /agents/chat/v2`.

#### Success Response: `200 OK`

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

Returns an SSE stream starting **after** the provided `lastMessageId`. Event format is identical to the original stream.

#### Error Response: `404 Not Found`

All error conditions return 404 (privacy through obscurity — prevents leaking resource existence to unauthorized users).

| Scenario | Body |
|----------|------|
| Chat not found or user unauthorized | `{"detail": "Chat not found"}` |
| Run ID not found in this chat | `{"detail": "Run not found in this chat"}` |
| Redis stream expired or missing | `{"detail": "Stream not found"}` |

#### Backend Validation Order

1. Authenticate user via Auth0 JWT
2. Look up chat by `threadId` + org — 404 if not found or user has no access
3. Verify `runId` exists in chat's `system_metadata.run_checkpoints` — 404 if not found
4. Check Redis stream key `ask_sybill:chat:{threadId}:{runId}` exists — 404 if expired/missing

---

## 3. SSE Event Format

> **Source:** `ask_sybill/schema/events.py`, `ask_sybill/schema/streaming.py`

Every SSE event follows this format:

```
id: <redis_stream_message_id>
event: <event_type>
data: <json_payload>
```

The `to_sse()` method in `streaming.py` formats each event:

```python
def to_sse(self, message_id: str | None = None) -> str:
    sse_parts = []
    if message_id:
        sse_parts.append(f"id: {message_id}")
    sse_parts.append(f"event: {self.event_type}")
    sse_parts.append(f"data: {json.dumps(self.data)}")
    return "\r\n".join(sse_parts) + "\r\n\r\n"
```

### Event Types (`AskSybillStreamingEventType` enum)

| Event Type | Description | Payload |
|------------|-------------|---------|
| `ack` | First event — request acknowledged | `{ threadId, messageId, runId }` |
| `message` | User's message echoed back | `{ content: string }` |
| `delta` | AI response token(s) | `{ content: string }` |
| `tool_debug` | Tool call lifecycle | `{ name, arguments, callId, phase }` |
| `execution_plan` | Planned task list | `{ tasks: [{ id, description, status }] }` |
| `task_progress` | Task status update | `{ taskId, status, description }` |
| `artifact` | Generated file/image | `{ type, name, url }` |
| `output` | Agent output | Varies |
| `structured_output` | Typed structured data | Varies |
| `chat_title` | Auto-generated title | `{ title: string }` |
| `citations` | Source citations | `{ sources: [{ title, url }] }` |
| `sources` | Source references | Varies |
| `suggestions` | Follow-up suggestions | `{ questions: string[] }` |
| `support` | Support/guidance info | Varies |
| `heartbeat` | Keep-alive (no payload) | `{}` |
| `done` | Final event — stream complete | `{ status, copilotStatus }` |

`copilotStatus` values: `"SUCCESS"`, `"FAILED"`, `"STOPPED"`

### Event Sequence (Typical)

```
ack → message → delta (×N) → [tool_debug → tool_debug]* → delta (×N) → citations → suggestions → done
```

Tool events, execution plans, artifacts, and other event types are interleaved based on the AI's behavior.

---

## 4. Resume API Examples

### Example 1: Network Error Resume (Partial Replay)

Client received events up to `id: 1707000000003-0`, then lost connection.

**Request:**
```http
GET /agents/chat/550e8400-.../run/9d4e5f6a-.../resume?lastMessageId=1707000000003-0
Authorization: Bearer <token>
```

**Response (200):**
```
id: 1707000000004-0
event: delta
data: {"content":" our Q4 revenue was $10M."}

id: 1707000000005-0
event: citations
data: {"sources":[{"title":"Q4 Report","url":"/calls/abc123"}]}

id: 1707000000006-0
event: done
data: {"status":"complete","copilotStatus":"SUCCESS"}
```

Events 0–3 are skipped. Only missed events are returned.

---

### Example 2: Page Refresh (Full Replay)

`lastMessageId` lost from memory. Call without it to replay the entire stream.

**Request:**
```http
GET /agents/chat/550e8400-.../run/9d4e5f6a-.../resume
Authorization: Bearer <token>
```

**Response (200):**
```
id: 1707000000000-0
event: ack
data: {"threadId":"550e8400-...","messageId":"7f3b2a1c-0001","runId":"9d4e5f6a-..."}

id: 1707000000001-0
event: message
data: {"content":"What is our Q4 revenue?"}

id: 1707000000002-0
event: delta
data: {"content":"Based on the data, our Q4 revenue was $10M."}

id: 1707000000003-0
event: citations
data: {"sources":[{"title":"Q4 Report","url":"/calls/abc123"}]}

id: 1707000000004-0
event: done
data: {"status":"complete","copilotStatus":"SUCCESS"}
```

Full stream from the beginning, including the user's original message.

---

### Example 3: Stream Still Active (Catch-up → Live Transition)

Client reconnects while the AI is still generating. Buffered events replay instantly, then live events arrive in real-time.

**Request:**
```http
GET /agents/chat/.../run/.../resume?lastMessageId=1707000000002-0
Authorization: Bearer <token>
```

**Response (200):**
```
id: 1707000000003-0
event: delta
data: {"content":" our Q4"}

id: 1707000000004-0
event: delta
data: {"content":" revenue"}

← Catch-up complete, now receiving live events →

id: 1707000000005-0
event: delta
data: {"content":" was approximately $10M."}

id: 1707000000006-0
event: done
data: {"status":"complete","copilotStatus":"SUCCESS"}
```

The transition is invisible — events arrive as one continuous stream.

---

### Example 4: Stream Completed Before Resume

AI finished responding. Server replays remaining buffered events and closes immediately (no XREAD BLOCK).

**Request:**
```http
GET /agents/chat/.../run/.../resume?lastMessageId=1707000000004-0
Authorization: Bearer <token>
```

**Response (200):**
```
id: 1707000000005-0
event: done
data: {"status":"complete","copilotStatus":"SUCCESS"}
```

---

### Example 5: Stream Expired / Not Found

**Request:**
```http
GET /agents/chat/old-thread/run/old-run/resume?lastMessageId=1707000000003-0
Authorization: Bearer <token>
```

**Response (404):**
```json
{"detail": "Stream not found"}
```

---

### Example 6: Resume with Tool Calls

**Request:**
```http
GET /agents/chat/.../run/.../resume?lastMessageId=1707000000002-0
Authorization: Bearer <token>
```

**Response (200):**
```
id: 1707000000003-0
event: tool_debug
data: {"name":"search_calls","arguments":"{\"query\":\"Acme Corp\"}","callId":"call-abc","phase":"invocation"}

id: 1707000000004-0
event: tool_debug
data: {"callId":"call-abc","result":"{\"calls\":[{\"title\":\"Acme Weekly Sync\"}]}","phase":"result"}

id: 1707000000005-0
event: delta
data: {"content":"Based on the Acme Corp call, the key takeaway was..."}

id: 1707000000006-0
event: citations
data: {"sources":[{"title":"Acme Weekly Sync","url":"/calls/xyz"}]}

id: 1707000000007-0
event: done
data: {"status":"complete","copilotStatus":"SUCCESS"}
```

---

## 5. Error Responses

### Resume Endpoint — All 404s

The resume endpoint returns 404 for every error scenario. The `detail` message varies but should not be shown to users:

| Scenario | `detail` |
|----------|----------|
| Chat doesn't exist or user has no access | `"Chat not found"` |
| `runId` not in chat's run checkpoints | `"Run not found in this chat"` |
| Redis stream key doesn't exist (expired or never created) | `"Stream not found"` |

### When 404 Means the Stream Expired

If the client successfully received SSE events with `id` fields earlier (meaning Redis persistence was active), a 404 on resume most likely means the stream TTL expired (4 hours). The client should fall back to loading the completed response from MongoDB.

### When 404 Means the Feature is Disabled

If the `REDIS_STREAM_PERSISTENCE` flag is off for the user, no Redis stream is created and the resume endpoint returns 404 with `"Stream not found"`. SSE events from `POST /agents/chat/v2` also won't have `id` fields.

---

## 6. Assumptions & Constraints

### Stream Lifecycle

| Property | Value |
|----------|-------|
| Stream TTL | **4 hours** — auto-expires |
| Max messages per stream | **1,000** (`MAXLEN ~1000` on XADD) |
| Stream scope | Per **run** (each user message = new run = new stream) |
| XREAD BLOCK timeout | **5 seconds** (how long the server waits for new messages during live subscription) |
| Read batch size | **100** messages per XREAD during catch-up |

### Redis Key Format

```
Stream:   ask_sybill:chat:{threadId}:{runId}
Metadata: ask_sybill:chat:{threadId}:{runId}:metadata
```

### Stream Metadata (Redis Hash)

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `active` → `completed` or `error` |
| `createdAt` | unix ms | When the stream was created |
| `updatedAt` | unix ms | Last update timestamp |
| `completedAt` | unix ms | Set when stream finishes |
| `totalMessages` | int | Incremented on each XADD |
| `errorMessage` | string | Set when stream errors |
| `context.thread_id` | string | Thread UUID |
| `context.run_id` | string | Run UUID |
| `context.user_id` | string | User UUID |
| `context.org_id` | string | Org UUID |

### Behavioral Guarantees

1. **No duplicate events on resume.** The server uses exclusive range queries (`XREAD` from the given offset, which returns messages strictly after it).

2. **Events continue publishing to Redis even after client disconnects.** The backend detached execution model means the AI agent keeps running and publishing to Redis when the SSE connection drops. This is what makes resume possible.

3. **Resume from active stream is seamless.** The server replays buffered messages first, then transitions to live subscription (`XREAD BLOCK`). There is no gap or overlap.

4. **Redis write failures don't break the primary SSE stream.** Redis publishing is fire-and-forget — if an XADD fails, the SSE event is still delivered to the connected client. The stream will just be incomplete for resume purposes.

5. **Same authentication.** The resume endpoint uses the same Auth0 JWT auth as the chat endpoint. The user must own the chat thread.

### Feature Flag

```
Flag: REDIS_STREAM_PERSISTENCE
Type: boolean, per-user/per-org
```

Checked in `chat_utils.py`:

```python
enable_redis_persistence = await config_utils.is_feature_enabled_for(
    FeatureFlags.REDIS_STREAM_PERSISTENCE, user_id=user_id, acc_cn=acc_cn
)
```

When **enabled**: SSE events include `id` fields, Redis streams are created, resume endpoint works.
When **disabled**: SSE events have no `id` field, no Redis stream, resume returns 404.

---

## 7. Backend Source Reference

Key files in the `flamingo-fer` repository:

| Component | File |
|-----------|------|
| Route definitions (chat + resume) | `src/sybill_py/runtimes/skynet/routers/chats/index.py` |
| Resume handler + validation | `src/sybill_py/runtimes/skynet/routers/chats/chat_utils.py` (L1375-1414) |
| Resume stream generator (catch-up + live) | `src/sybill_py/runtimes/skynet/routers/chats/chat_utils.py` (L1327-1372) |
| Dual-publish stream generator | `src/sybill_py/runtimes/skynet/routers/chats/chat_utils.py` (L635-779) |
| SSE event types enum | `src/sybill_py/runtimes/skynet/ask_sybill/schema/events.py` |
| SSE `to_sse()` formatting | `src/sybill_py/runtimes/skynet/ask_sybill/schema/streaming.py` |
| Redis Stream client (XADD/XREAD/subscribe) | `src/sybill_py/components/redis_streams/client.py` |
| Redis Stream schema (StreamMessage, StreamMetadata) | `src/sybill_py/components/redis_streams/schema.py` |
| Redis Stream config (TTL, max len, timeouts) | `src/sybill_py/components/redis_streams/config.py` |
| Auth dependencies | `src/routers/authentication.py` |
| Feature flags | `src/sybill_py/utils/config_utils.py` — `FeatureFlags.REDIS_STREAM_PERSISTENCE` |
| Redis client initialization | `src/sybill_py/runtimes/skynet/main.py` (L153-161) |
