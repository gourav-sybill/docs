# Redis Streams SSE Resilience Architecture

## Overview

This document describes the design and architecture for SSE (Server-Sent Events) connection resilience in Ask Sybill chat using Redis Streams. The feature enables clients to resume streaming responses after network interruptions without losing any events.

## Problem Statement

When a user sends a message to Ask Sybill, the response is streamed back via SSE. If the network connection drops during streaming (mobile network switching, brief connectivity issues, etc.), the client loses all subsequent events and cannot recover the response without re-requesting it.

### Goals

1. **Zero event loss**: Clients should be able to resume and receive all events they missed
2. **Minimal latency impact**: The feature should not significantly impact normal streaming performance
3. **Graceful degradation**: If Redis is unavailable, the system should fall back to normal SSE behavior
4. **Resource efficiency**: Streams should be automatically cleaned up after use

## Architecture

### High-Level Flow

```
┌─────────┐     SSE      ┌─────────────┐     Events    ┌─────────────┐
│  Client │◄────────────►│   Skynet    │◄─────────────►│  LangGraph  │
└─────────┘              │   Server    │               │    Agent    │
     │                   └──────┬──────┘               └─────────────┘
     │                          │
     │   Resume Request         │ Dual-publish
     │                          ▼
     │                   ┌─────────────┐
     └──────────────────►│    Redis    │
                         │   Streams   │
                         └─────────────┘
```

### Components

#### 1. Redis Streams Client (`sybill_py/components/redis_streams/`)

A reusable component for Redis Streams operations, designed for SSE resilience but extensible to other streaming use cases.

**Files:**
- `client.py` - Core `RedisStreamClient` singleton with stream operations
- `schema.py` - Pydantic models for stream data (`StreamMessage`, `StreamMetadata`, `ReadStreamResult`)
- `config.py` - Configuration constants and `StreamKeyPrefix` enum for namespacing

**Key Features:**
- Singleton pattern for connection pooling
- SSL/TLS support for production Redis
- Configurable TTL and max stream length
- Sequence numbers for client-side ordering
- Crash recovery with `get_or_create_stream`

#### 2. Chat Stream Generator (`chat_utils.py`)

The `_chat_message_stream_generator_detached` function handles dual-publishing:

```python
# Stream is scoped to both thread and run
stream_id = f"{thread_id}:{run_id}"

# Dual-publish: SSE queue + Redis stream
if redis_client is not None:
    await redis_client.publish(
        stream_id=stream_id,
        event_type=streaming_event.event_type,
        payload=streaming_event.data,
        prefix=StreamKeyPrefix.ASK_SYBILL_CHAT,
        sequence=sequence,
    )
    sequence += 1
```

#### 3. Resume Endpoint (`resume_chat_stream`)

Handles client reconnection requests:

1. Validates user access to the chat
2. Verifies the run exists in the chat's `run_checkpoints`
3. Checks that the Redis stream exists
4. Returns a `StreamingResponse` with catch-up + live events

## Data Model

### Stream Key Structure

```
ask_sybill:chat:{thread_id}:{run_id}
ask_sybill:chat:{thread_id}:{run_id}:metadata
```

Each run within a chat gets its own isolated stream, ensuring:
- Parallel runs don't interfere with each other
- Resume is scoped to a specific execution
- Clean separation of concerns

### StreamMessage Schema

```python
class StreamMessage(SybillBaseModel):
    message_id: str | None = None      # Redis-assigned ID (e.g., "1234567890123-0")
    event_type: str                     # Event type (e.g., "delta", "done")
    payload: dict[str, Any]             # Event data
    timestamp: UnixMilliseconds | None  # Creation timestamp
    sequence: int | None                # Client-side ordering number
```

### StreamMetadata Schema

```python
class StreamMetadata(SybillBaseModel):
    stream_id: str
    status: StreamStatus                # ACTIVE | COMPLETED | ERROR
    created_at: UnixMilliseconds
    updated_at: UnixMilliseconds | None
    completed_at: UnixMilliseconds | None
    total_messages: int
    error_message: str | None
    context: dict[str, Any]             # thread_id, run_id, user_id, org_id
```

## Resume Flow

### Client Reconnection

```
1. Client detects disconnection
2. Client calls: POST /chats/{thread_id}/runs/{run_id}/resume
   - Query param: last_message_id (optional, for resuming from specific point)
3. Server validates access and stream existence
4. Server returns StreamingResponse:
   - Phase 1: Catch-up (read all missed messages from Redis)
   - Phase 2: Live subscription (if stream still active)
```

### Resume Generator Implementation

```python
async def _resume_stream_generator(stream_id: str, last_message_id: str | None):
    redis_client = RedisStreamClient()

    # Phase 1: Catch-up - read historical messages
    from_id = last_message_id or "0"
    while True:
        result = await redis_client.read_from(
            stream_id=stream_id,
            from_id=from_id,
            prefix=StreamKeyPrefix.ASK_SYBILL_CHAT,
        )
        for message in result.messages:
            yield StreamingEvent(...).to_sse()
        if not result.has_more:
            break
        from_id = result.last_message_id

    # Phase 2: Live subscription (if stream is still active)
    if result.is_active:
        async for message in redis_client.subscribe(
            stream_id=stream_id,
            from_id=current_id,
            prefix=StreamKeyPrefix.ASK_SYBILL_CHAT,
        ):
            yield StreamingEvent(...).to_sse()
```

## Frontend Client Integration

This section describes how frontend clients should integrate with the SSE streaming and resume APIs.

### API Endpoints

#### 1. Send Message (Initial Stream)

```
POST /api/skynet/chats
Content-Type: application/json
Authorization: Bearer <token>

{
  "message": { "text": "What are the key takeaways from my last call?" },
  "is_new_chat": true,
  "is_temporary": false
}

Response: SSE Stream (text/event-stream)
```

#### 2. Resume Stream

```
POST /api/skynet/chats/{thread_id}/runs/{run_id}/resume?last_message_id={id}
Authorization: Bearer <token>

Response: SSE Stream (text/event-stream)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `thread_id` | UUID | Yes | The chat thread ID |
| `run_id` | UUID | Yes | The specific run ID to resume |
| `last_message_id` | string | No | Redis message ID of last received event (e.g., "1706123456789-0") |

### SSE Event Format

Events are sent in standard SSE format:

```
event: <event_type>
data: <json_payload>

```

**Event Types:**
- `thinking_step` - Agent is processing/thinking
- `delta` - Incremental text content
- `message` - Complete message
- `tool_call` - Tool invocation
- `tool_result` - Tool response
- `heartbeat` - Keep-alive signal (every 15 seconds)
- `done` - Stream completed

**Example Events:**

```
event: thinking_step
data: {"step": "Searching your calls...", "sequence": 0}

event: delta
data: {"content": "Based on your last call", "sequence": 1}

event: delta
data: {"content": " with Acme Corp...", "sequence": 2}

event: done
data: {"status": "success", "sequence": 10}

```

### Client State Management

The client must track the following state for resume capability:

```typescript
interface StreamState {
  threadId: string;
  runId: string;
  lastMessageId: string | null;  // Redis message ID from last event
  lastSequence: number;          // Sequence number for ordering
  isActive: boolean;             // Whether stream is still active
}
```

### Scenarios and Handling

#### Scenario 1: Normal Flow (No Disconnection)

```
Client                          Server                         Redis
  │                                │                              │
  │──POST /chats (message)────────►│                              │
  │                                │──Create stream──────────────►│
  │◄──SSE: thinking_step (seq=0)───│◄─────────────────────────────│
  │◄──SSE: delta (seq=1)───────────│                              │
  │◄──SSE: delta (seq=2)───────────│                              │
  │◄──SSE: done (seq=10)───────────│──Mark completed─────────────►│
  │                                │                              │
```

**Client behavior:** Process events normally, update `lastMessageId` and `lastSequence` on each event.

#### Scenario 2: Brief Disconnection During Active Stream

```
Client                          Server                         Redis
  │                                │                              │
  │──POST /chats (message)────────►│                              │
  │◄──SSE: delta (seq=0-5)─────────│                              │
  │       ⚡ NETWORK DROP ⚡         │                              │
  │                                │──Continue publishing────────►│
  │                                │                              │
  │  (client detects disconnect)   │                              │
  │  (wait with backoff)           │                              │
  │                                │                              │
  │──POST /resume?last_message_id──►│                              │
  │                                │◄──Read from last_message_id──│
  │◄──SSE: delta (seq=6-8)─────────│  (catch-up phase)            │
  │                                │◄──Subscribe for new──────────│
  │◄──SSE: delta (seq=9)───────────│  (live phase)                │
  │◄──SSE: done (seq=10)───────────│                              │
  │                                │                              │
```

**Client behavior:**
1. Detect disconnection via `onerror` or `onclose` event
2. Check if stream is still active (not received `done` event)
3. Wait with exponential backoff
4. Call resume endpoint with `last_message_id`
5. Process catch-up events (may receive duplicates - use `sequence` to dedupe)
6. Continue with live events

#### Scenario 3: Disconnection After Stream Completed

```
Client                          Server                         Redis
  │                                │                              │
  │──POST /chats (message)────────►│                              │
  │◄──SSE: delta (seq=0-8)─────────│                              │
  │◄──SSE: done (seq=9)────────────│──Mark completed─────────────►│
  │       ⚡ NETWORK DROP ⚡         │                              │
  │                                │                              │
  │  (client already has done)     │                              │
  │  (no reconnection needed)      │                              │
  │                                │                              │
```

**Client behavior:** Since the `done` event was received, mark `isActive = false` and don't attempt reconnection.

#### Scenario 4: Stream Expired (TTL exceeded)

```
Client                          Server                         Redis
  │                                │                              │
  │  (disconnect for > 4 hours)    │                              │
  │                                │              │──Stream TTL expires──│
  │                                │                              │
  │──POST /resume?last_message_id──►│                              │
  │                                │◄──Stream not found───────────│
  │◄──404 Not Found────────────────│                              │
  │                                │                              │
```

**Client behavior:**
1. Receive 404 response from resume endpoint
2. Display appropriate error message to user
3. User may need to re-send the message to get a fresh response

#### Scenario 5: Resume Without `last_message_id`

```
Client                          Server                         Redis
  │                                │                              │
  │  (disconnect, lost state)      │                              │
  │                                │                              │
  │──POST /resume (no last_id)────►│                              │
  │                                │◄──Read from beginning────────│
  │◄──SSE: delta (seq=0-10)────────│  (full replay)               │
  │◄──SSE: done───────────────────│                              │
  │                                │                              │
```

**Client behavior:** If `last_message_id` is not available, the resume endpoint returns ALL events from the beginning. Use `sequence` numbers to deduplicate if needed.

#### Scenario 6: Multiple Rapid Disconnections

```typescript
// Handle with exponential backoff and max attempts
const backoffDelays = [1000, 2000, 4000, 8000, 16000]; // ms

async function reconnectWithBackoff(attempt: number): Promise<void> {
  if (attempt >= backoffDelays.length) {
    throw new Error('Max reconnection attempts reached');
  }

  const delay = backoffDelays[attempt] + Math.random() * 1000; // Add jitter
  await sleep(delay);

  try {
    await resume();
  } catch (error) {
    await reconnectWithBackoff(attempt + 1);
  }
}
```

### Best Practices

#### 1. Always Track State

```typescript
// Store state persistently (localStorage, sessionStorage, or state management)
function saveStreamState(state: StreamState): void {
  sessionStorage.setItem(`stream_${state.threadId}_${state.runId}`, JSON.stringify(state));
}

function loadStreamState(threadId: string, runId: string): StreamState | null {
  const stored = sessionStorage.getItem(`stream_${threadId}_${runId}`);
  return stored ? JSON.parse(stored) : null;
}
```

#### 2. Deduplicate Events

```typescript
const processedSequences = new Set<number>();

function handleEvent(event: StreamEvent): void {
  if (event.sequence !== undefined) {
    if (processedSequences.has(event.sequence)) {
      console.log(`Skipping duplicate event with sequence ${event.sequence}`);
      return;
    }
    processedSequences.add(event.sequence);
  }

  // Process the event
  processEvent(event);
}
```

#### 3. Handle Partial Content Gracefully

```typescript
// Buffer incomplete SSE chunks
let buffer = '';

function processChunk(chunk: string): void {
  buffer += chunk;

  // SSE events are delimited by double newlines
  const events = buffer.split('\n\n');

  // Keep the last incomplete chunk in buffer
  buffer = events.pop() || '';

  for (const eventStr of events) {
    if (eventStr.trim()) {
      parseAndHandleEvent(eventStr);
    }
  }
}
```

#### 4. Provide User Feedback

```typescript
function onDisconnect(): void {
  showToast('Connection lost. Reconnecting...', 'warning');
}

function onReconnect(): void {
  showToast('Reconnected!', 'success');
}

function onReconnectFailed(): void {
  showModal({
    title: 'Connection Lost',
    message: 'Unable to reconnect. Would you like to retry or start over?',
    actions: [
      { label: 'Retry', onClick: () => resume() },
      { label: 'Start Over', onClick: () => resendMessage() },
    ],
  });
}
```

#### 5. Clean Up State on Completion

```typescript
function onStreamComplete(state: StreamState): void {
  // Clear stored state after successful completion
  sessionStorage.removeItem(`stream_${state.threadId}_${state.runId}`);

  // Clear in-memory tracking
  processedSequences.clear();
}
```

### Error Responses

| Status Code | Meaning | Source | Client Action |
|-------------|---------|--------|---------------|
| 200 | Success | Endpoint | Process stream |
| 401 | Unauthorized | Auth middleware | Re-authenticate user |
| 403 | Forbidden | Auth middleware | User doesn't have organization access |
| 404 | Not Found | Endpoint | Chat not found, run not found, or stream expired - do not retry |
| 500 | Server Error | Any | Retry with backoff |

**Note:** The resume endpoint returns 404 for three distinct cases:
- `"Chat not found"` - The chat thread doesn't exist or user doesn't have access
- `"Run not found in this chat"` - The specific run ID is not part of this chat
- `"Stream not found"` - The Redis stream has expired (TTL exceeded) or was never created

## Feature Flag Control

The feature is controlled by the `REDIS_STREAM_PERSISTENCE` feature flag:

```python
enable_redis_persistence = await config_utils.is_feature_enabled_for(
    FeatureFlags.REDIS_STREAM_PERSISTENCE, user_id=user_id, acc_cn=acc_cn
)
```

This allows gradual rollout and easy disable if issues arise.

## Configuration

### Default Values (`config.py`)

| Constant | Value | Description |
|----------|-------|-------------|
| `DEFAULT_STREAM_TTL_SECONDS` | 4 hours | Stream lifetime after creation |
| `DEFAULT_STREAM_MAX_LEN` | 1000 | Max messages per stream (approximate) |
| `DEFAULT_BLOCK_TIMEOUT_MS` | 5000ms | XREAD block timeout |
| `DEFAULT_READ_COUNT` | 100 | Batch size for reading |
| `DEFAULT_MAX_STATUS_CHECK_ATTEMPTS` | 100 | Safety limit for subscribe loop |

### Stream Key Prefixes

```python
class StreamKeyPrefix(StrEnum):
    ASK_SYBILL_CHAT = "ask_sybill:chat:"
    WORKFLOW_PROGRESS = "workflow:progress:"  # Future use
```

## Failure Handling

### Redis Unavailable

If Redis connection fails during stream creation:
- Log error and set `redis_client = None`
- Continue with normal SSE streaming (graceful degradation)
- Resume endpoint returns 404 (stream not found)

### Publish Failures

If publishing to Redis fails:
- Log warning
- Continue streaming to client via SSE queue
- Partial stream data may be available for resume

### Stream Completion

When the agent finishes:
- On success/stopped: `complete_stream()` marks status as COMPLETED
- On error: `error_stream()` marks status as ERROR with message
- TTL ensures cleanup even if completion fails

### Subscribe Safety Exit

The subscribe loop has a safety mechanism to prevent infinite loops:
- Counter increments on each timeout (no messages received)
- Counter resets when a message is received
- After `max_status_check_attempts` consecutive timeouts, loop exits
- Prevents resource exhaustion if stream is never properly completed

## Security

### Access Control

The resume endpoint validates:
1. User authentication (via existing auth middleware)
2. User has access to the chat (`get_chat_for_user`)
3. Run exists in the chat's `run_checkpoints`
4. Stream exists in Redis

### Data Isolation

- Streams are namespaced by prefix (`ask_sybill:chat:`)
- Stream ID includes both `thread_id` and `run_id`
- Users can only resume streams for chats they own or have access to

## Testing

### Test Coverage

The implementation includes comprehensive tests:

1. **Unit Tests** (`test_client.py`) - 40 tests
   - Initialization and configuration
   - All CRUD operations
   - Subscribe functionality
   - Integration-style lifecycle tests

2. **E2E Integration Tests** (`test_chat_resume_e2e.py`) - 18 tests
   - Full resume flow with real Redis
   - Access control validation
   - Edge cases (archived chats, expired chats, etc.)

3. **Unit Tests for Resume** (`test_chat_resume.py`) - 12 tests
   - Mock-based tests for resume endpoint
   - Error handling scenarios

### Code Coverage

The Redis Streams component achieves **100% code coverage**.

## Future Considerations

### Potential Enhancements

1. **Client SDK**: Provide JavaScript/TypeScript utilities for automatic reconnection
2. **Compression**: Compress payloads for bandwidth efficiency
3. **Metrics**: Add observability for resume success rates, latency
4. **Multi-region**: Support for Redis cluster/replication

### Extensibility

The `StreamKeyPrefix` enum allows easy addition of new stream types:
```python
class StreamKeyPrefix(StrEnum):
    ASK_SYBILL_CHAT = "ask_sybill:chat:"
    WORKFLOW_PROGRESS = "workflow:progress:"  # For Temporal workflow progress
    AGENT_EXECUTION = "agent:execution:"      # Future agent types
```

## Related Files

### Source Code
- `src/sybill_py/components/redis_streams/client.py`
- `src/sybill_py/components/redis_streams/schema.py`
- `src/sybill_py/components/redis_streams/config.py`
- `src/sybill_py/runtimes/skynet/routers/chats/chat_utils.py`

### Tests
- `tests/sybill_py/components/redis_streams/test_client.py`
- `tests/sybill_py/runtimes/skynet/routers/chats/test_chat_resume.py`
- `tests/sybill_py/runtimes/skynet/routers/chats/test_chat_resume_e2e.py`
- `tests/conftest.py` (Redis fixtures)
