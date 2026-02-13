# Redis Streams for SSE Connection Resilience

**Status:** Implemented  
**Author:** Engineering Team  
**Last Updated:** January 2026

---

## 1. Overview

This document describes the design for using Redis Streams to enable Server-Sent Events (SSE) connection resilience for Ask Sybill chat functionality. The feature allows clients to resume interrupted chat streams without losing messages.

---

## 2. Problem Statement

### Current Behavior

Ask Sybill SSE connections break during long-running chats (1-2 minutes) due to:

- **Load balancer connection draining**: AWS ALB/NLB closes idle connections
- **Client network changes**: IP address changes, WiFi to cellular switches
- **General network instability**: Temporary network outages

When a connection breaks, the frontend receives a network error, polls until the response is ready, and then receives all content at once—resulting in poor user experience.

### Impact

- ~5 network errors per day (from Amplitude metrics)
- Users lose real-time streaming experience
- Perceived latency increases significantly

### Desired Behavior

Frontend reconnects via a resume endpoint and receives only missed messages, enabling seamless continuation of the chat stream.

---

## 3. Goals and Non-Goals

### Goals

1. Enable clients to resume SSE streams after network interruption
2. Preserve message ordering during resume
3. Support both network-error recovery and page-refresh scenarios
4. Minimize Redis memory footprint with automatic TTL cleanup
5. Feature-flag controlled rollout

### Non-Goals

1. Persist streams beyond 4 hours (streams auto-expire)
2. Support cross-device resume (stream tied to original session context)
3. Replace MongoDB as the source of truth for chat history

---

## 4. Architecture

### System Diagram

```
┌─────────────┐     POST /agents/chats/v2    ┌──────────────────┐
│   Frontend  │ ──────────────────────────►  │  Skynet Server   │
│             │◄─────── SSE Stream ──────────│                  │
└─────────────┘                              └────────┬─────────┘
       │                                              │
       │  GET /agents/chat/{threadId}/resume          │ Dual publish
       │  ?lastMessageId=X                            ▼
       │                                     ┌──────────────────┐
       └────────────────────────────────────►│  Redis Streams   │
                Resume stream                │  (ElastiCache)   │
                                             └──────────────────┘
```

### Data Flow

#### Normal Operation
1. Client sends POST request to create/continue chat
2. Server creates Redis stream with unique ID (thread_id)
3. Each SSE event is **dual-published**: to client AND to Redis stream
4. Events include a `message_id` (Redis stream ID) for resumption
5. On completion, stream is marked complete in Redis metadata

#### On Network Error
1. Client detects connection loss
2. Client calls `GET /agents/chat/{threadId}/resume?lastMessageId=X`
3. Server reads Redis stream from offset X+1
4. Returns remaining events as SSE stream
5. If stream still active, subscribes to live updates

#### On Page Refresh
1. Client reloads and loses `lastMessageId` from memory
2. Client calls resume with `lastMessageId=null` to get full stream
3. Stream includes the user's message (via `MessageEvent`)
4. Complete conversation reconstructed from Redis

---

## 5. Stream Contents

The SSE stream includes **both user messages and server responses**. This ensures resume works correctly for all scenarios.

### Event Sequence

| Order | Event Type | Description |
|-------|------------|-------------|
| 1 | `ack` | Request acknowledgment with message/run/thread IDs |
| 2 | `message` | **User's message** echoed back |
| 3 | `delta` | AI response tokens (streamed incrementally) |
| 4 | `tool_call` | Tool invocations |
| 5 | `tool_result` | Tool responses |
| 6 | `task_progress` | Multi-step task progress updates |
| 7 | `artifact` | Generated files, images, etc. |
| 8 | `citations` | Source citations |
| 9 | `suggestions` | Follow-up question suggestions |
| 10 | `done` | Stream completion with status code |

### Key Design Decision

The `MessageEvent` containing the user's message is part of the stream. This means:
- Resuming from the beginning returns the complete conversation
- Page refresh works correctly—user sees their message + AI response
- No inconsistency between normal flow and resume flow

---

## 6. API Design

### Resume Endpoint

```
GET /agents/chat/{threadId}/resume?lastMessageId={id}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `threadId` | UUID (path) | Yes | The chat thread ID |
| `lastMessageId` | string (query) | No | Last received Redis stream message ID |

**Response:** `text/event-stream` (SSE)

### Response Headers

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
```

### Error Responses

All errors return **404 Not Found** to follow the "privacy through obscurity" pattern:

| Scenario | Detail Message |
|----------|----------------|
| Feature disabled | "Stream resumption not available" |
| Chat not found / unauthorized | "Chat not found" |
| Stream expired or missing | "Stream not found" |

**Rationale:** Returning 404 for all error cases prevents leaking information about whether a resource exists to unauthorized users.

---

## 7. Resume Scenarios

| Scenario | `lastMessageId` | Resume Behavior | Result |
|----------|-----------------|-----------------|--------|
| Network error (app running) | In memory | From specific offset | Only missed events |
| Page refresh | Lost | From beginning (`null`) | Full stream |
| Browser close/reopen | Lost | From beginning (`null`) | Full stream |
| Stream completed | N/A | Returns 404 | Use MongoDB data |
| Stream expired (>4 hours) | N/A | Returns 404 | Use MongoDB data |

### Network Error Flow

```
Client                                     Server
  │                                          │
  │◄─── SSE: id=1, delta ────────────────────│
  │◄─── SSE: id=2, delta ────────────────────│
  │                                          │
  │  ✖ NETWORK ERROR ✖                       │
  │                                          │
  │  GET /resume?lastMessageId=2             │
  │─────────────────────────────────────────►│
  │                                          │
  │◄─── SSE: id=3, delta ────────────────────│  (messages 1-2 skipped)
  │◄─── SSE: id=4, done ─────────────────────│
  │                                          │
```

### Page Refresh Flow

```
Client                                     Server
  │                                          │
  │  [Page Refresh - loses lastMessageId]    │
  │                                          │
  │  GET /resume (no lastMessageId)          │
  │─────────────────────────────────────────►│
  │                                          │
  │◄─── SSE: id=0, ack ──────────────────────│
  │◄─── SSE: id=1, message (user's input) ───│
  │◄─── SSE: id=2, delta ────────────────────│
  │◄─── SSE: id=3, delta ────────────────────│
  │◄─── SSE: id=4, done ─────────────────────│
  │                                          │
  │  ✓ Complete conversation restored        │
```

---

## 8. Frontend Integration

### Tracking Message IDs

The frontend must track the `id` field from each SSE event for resume capability:

```typescript
let lastMessageId: string | null = null;

eventSource.onmessage = (event) => {
  if (event.lastEventId) {
    lastMessageId = event.lastEventId;
  }
  handleEvent(JSON.parse(event.data));
};
```

### Resume on Network Error

```typescript
async function resumeStream(threadId: string, lastMessageId: string | null) {
  const url = `/agents/chat/${threadId}/resume` +
    (lastMessageId ? `?lastMessageId=${lastMessageId}` : '');
  
  const response = await fetch(url);
  
  if (response.status === 404) {
    // Stream expired - fall back to polling or MongoDB
    return fallbackToPoll();
  }
  
  // Resume successful - consume stream
  await consumeStream(response.body);
}
```

### Resume on Page Refresh

```typescript
async function loadChat(threadId: string) {
  // 1. Load chat metadata from MongoDB
  const chat = await fetchChat(threadId);
  
  // 2. Try to resume any active stream (with lastMessageId=null)
  try {
    const response = await fetch(`/agents/chat/${threadId}/resume`);
    if (response.ok) {
      // Active stream found - consume it
      await consumeStream(response.body);
    }
  } catch (e) {
    // 404 or error - no active stream, use MongoDB data
  }
  
  return chat;
}
```

---

## 9. Redis Data Model

### Stream Key Format

```
ask_sybill:chat:{thread_id}           # Main stream (Redis Stream)
ask_sybill:chat:{thread_id}:metadata  # Stream metadata (Redis Hash)
```

### Stream Message Format

Each message in the Redis stream contains:

| Field | Type | Description |
|-------|------|-------------|
| `event_type` | string | Event type (ack, delta, done, etc.) |
| `payload` | JSON string | Event payload |
| `timestamp` | int | Unix timestamp in milliseconds |
| `sequence` | int | Monotonic sequence number |

### Metadata Hash

| Field | Value |
|-------|-------|
| `status` | `active` / `complete` / `error` |
| `created_at` | Unix timestamp |
| `thread_id` | Chat thread UUID |
| `user_id` | User UUID |

### TTL Configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| Stream TTL | 4 hours | Sufficient for long chats, auto-cleanup |
| Max messages | 10,000 | Safety limit per stream |

---

## 10. Rollout Strategy

### Approach

Since dual-publishing to Redis has no side effects on the client experience (SSE stream works identically), we use a **backend-first, frontend-gradual** rollout:

1. **Backend**: Enable Redis publishing for 100% of requests
2. **Frontend**: Gradual rollout of resume functionality

This simplifies rollout because:
- No backend feature flag complexity
- Frontend controls when to use resume
- Easy rollback (stop calling resume endpoint)
- Backend is always ready

### Phases

| Phase | Backend | Frontend |
|-------|---------|----------|
| **1. Backend Deploy** | Enable Redis publishing 100% | No changes |
| **2. Monitor** | Monitor Redis memory/latency | No changes |
| **3. Beta** | No changes | Enable resume for beta users |
| **4. Gradual Rollout** | No changes | 10% → 50% → 100% |

### Rollback

- **Backend issue**: Disable Redis publishing (SSE still works)
- **Frontend issue**: Disable resume calls (falls back to polling)

---

## 11. Monitoring

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Stream creation rate | Streams created per minute | Baseline + 50% |
| Resume call rate | Resume endpoint calls | N/A (informational) |
| Resume 404 rate | Percentage of 404 responses | > 10% |
| Redis memory usage | Memory for stream keys | > 80% capacity |
| Stream TTL before completion | Time until stream completes | > 30 min avg |

### Alerts

- Redis memory > 80% capacity
- Resume 404 rate > 10% (may indicate TTL too short)
- Stream creation failure rate > 1%

---

## 12. Alternatives Considered

### Alternative 1: Client-Side Message Buffer

Store messages in IndexedDB/localStorage on the client.

**Pros:** No server changes needed  
**Cons:** Doesn't survive browser close, storage limits, sync complexity

### Alternative 2: WebSocket with Reconnection

Replace SSE with WebSocket protocol.

**Pros:** Better reconnection support built-in  
**Cons:** Large migration effort, SSE works well for our use case

### Alternative 3: Server-Side Message Log in MongoDB

Store each message in MongoDB as it's generated.

**Pros:** Single source of truth  
**Cons:** Higher write latency, MongoDB not optimized for streaming reads

**Decision:** Redis Streams chosen for optimal balance of performance, reliability, and implementation complexity.

---

## 13. Future Enhancements

1. **Active stream detection API**: Endpoint to check if stream is active without consuming it
2. **Cross-device resume**: Store checkpoint in MongoDB for device handoff
3. **Stream compression**: Reduce Redis memory for large responses
4. **Configurable TTL**: Per-chat TTL based on expected duration
5. **Temporal workflow streaming**: Reuse infrastructure for other streaming use cases

---

## 14. References

- [Redis Streams Documentation](https://redis.io/docs/latest/develop/data-types/streams/)
- [Server-Sent Events Spec](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- Internal: Ask Sybill Chat Architecture Doc
