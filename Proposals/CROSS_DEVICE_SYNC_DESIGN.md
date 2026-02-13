# Cross-Device Chat Synchronization - Implementation Design

## Problem Statement
Enable real-time message synchronization across multiple devices:
- User types message on **DeviceA** → Immediately appears on **DeviceB**
- AI response loads on both devices simultaneously
- No page refresh required

## Current System Analysis

### Architecture Overview

📊 **[View Interactive Diagram](docs/diagrams/current-architecture.excalidraw)** - Open in Excalidraw

```
┌─────────────┐
│   MongoDB   │ ← Persistent storage (copilotChats collection)
│             │   - Chat documents with messages array
│             │   - lastUpdatedAt, latestMessageId
└─────────────┘
       ↑
       │ (DAL write)
       │
┌─────────────┐     ┌──────────────────┐
│ Redis Stream│ ←→  │  SSE Streaming   │
│  (4hr TTL)  │     │   (to client)    │
└─────────────┘     └──────────────────┘
  Per-run isolation      One client only
  {threadId}:{runId}
```

### Current Flow
1. **DeviceA** sends POST `/chat/v2`
2. **Backend** creates Redis stream for that specific `run_id`
3. **Agent executes**, events published to Redis + streamed via SSE
4. **Only DeviceA** receives the SSE stream
5. **MongoDB** updated with final messages
6. **DeviceB** has no knowledge of updates (must poll/refresh)

### Current Components
- **File**: `src/sybill_py/components/redis_streams/client.py`
  - `RedisStreamClient` - Pub/sub for SSE resilience
  - Run-specific streams: `ask_sybill:chat:{thread_id}:{run_id}`

- **File**: `src/sybill_py/runtimes/skynet/routers/chats/index.py`
  - POST `/chat/v2` - Single-device SSE streaming
  - GET `/chat/{threadId}/run/{runId}/resume` - Resume interrupted streams

- **File**: `src/sybill_py/dal/queries/copilot_chats/writer.py`
  - `upsert_chat_message()` - Writes messages to MongoDB

## Required Changes for Cross-Device Sync

### Option 1: Thread-Level Redis Pub/Sub (Recommended)

**Architecture:**

📊 **[View Interactive Diagram](docs/diagrams/proposed-pubsub-architecture.excalidraw)** - Open in Excalidraw

```
┌──────────────────────────────────────────────────────────────┐
│                     Thread-Level Channel                       │
│               Redis Pub/Sub: chat:thread:{threadId}            │
│                                                                │
│   Publisher (Backend)           Subscribers (All Devices)      │
│        ↓                              ↓        ↓        ↓     │
│   Agent Execution            DeviceA  DeviceB  DeviceC        │
└──────────────────────────────────────────────────────────────┘
```

**Implementation Steps:**

#### 1. **Create Thread-Level Pub/Sub Channel**

**New File**: `src/sybill_py/components/redis_pubsub/client.py`
```python
class RedisPubSubClient(metaclass=Singleton):
    """Redis Pub/Sub for cross-device chat synchronization."""

    def __init__(self):
        self._pool = redis.connection.ConnectionPool(
            host=runtime.app_config.redis_config.redis_host,
            port=runtime.app_config.redis_config.redis_port,
        )

    @property
    def conn(self) -> redis.Redis:
        return redis.Redis(connection_pool=self._pool)

    def _channel_name(self, thread_id: UUID) -> str:
        """Thread-level channel for all devices."""
        return f"chat:thread:{thread_id}"

    async def publish_message_event(
        self,
        thread_id: UUID,
        event_type: str,
        message: ChatMessage | None = None,
        run_id: UUID | None = None,
    ) -> None:
        """Publish chat event to all subscribers of this thread."""
        channel = self._channel_name(thread_id)

        event_data = {
            "eventType": event_type,
            "threadId": str(thread_id),
            "runId": str(run_id) if run_id else None,
            "timestamp": utils.get_current_ts(),
        }

        if message:
            event_data["message"] = message.to_mongo()

        async with self.conn as conn:
            await conn.publish(channel, json.dumps(event_data))

        _logger.info(f"Published {event_type} to channel {channel}")

    async def subscribe_to_thread(
        self,
        thread_id: UUID,
    ) -> AsyncGenerator[ChatThreadEvent, None]:
        """Subscribe to all updates for a specific thread."""
        channel = self._channel_name(thread_id)

        async with self.conn.pubsub() as pubsub:
            await pubsub.subscribe(channel)

            async for message in pubsub.listen():
                if message["type"] == "message":
                    data = json.loads(message["data"])
                    yield ChatThreadEvent.from_dict(data)
```

**New Schema**: `src/sybill_py/components/schema/chat_events.py`
```python
class ChatThreadEventType(StrEnum):
    USER_MESSAGE_ADDED = "user_message_added"
    ASSISTANT_MESSAGE_ADDED = "assistant_message_added"
    MESSAGE_UPDATED = "message_updated"
    CHAT_TITLE_UPDATED = "chat_title_updated"
    RUN_STARTED = "run_started"
    RUN_COMPLETED = "run_completed"

class ChatThreadEvent(SybillBaseModel):
    event_type: ChatThreadEventType
    thread_id: UUID
    run_id: UUID | None = None
    message: ChatMessage | None = None
    timestamp: UnixMilliseconds
```

#### 2. **Modify Chat Persistence Layer to Broadcast**

**File**: `src/sybill_py/runtimes/skynet/ask_sybill/middleware/chat_persistence_layer.py`

```python
from sybill_py.components.redis_pubsub.client import RedisPubSubClient

class ChatPersistenceLayer(BaseGraphMiddleware):
    def __init__(self):
        super().__init__()
        self.pubsub_client = RedisPubSubClient()

    async def _add_user_message(self, ...):
        # Existing logic to add message to MongoDB
        chat, user_message = await super()._add_user_message(...)

        # NEW: Broadcast to all devices
        await self.pubsub_client.publish_message_event(
            thread_id=chat_context.thread_id,
            event_type=ChatThreadEventType.USER_MESSAGE_ADDED,
            message=user_message,
            run_id=chat_context.run_id,
        )

        return chat, user_message

    async def execute_and_stream(self, ...):
        # Existing streaming logic...

        async for event in self.inner_layer.execute_and_stream(...):
            # Publish to SSE stream (current behavior)
            yield event

            # NEW: If it's a message event, broadcast to all devices
            if isinstance(event, MessageEvent):
                await self.pubsub_client.publish_message_event(
                    thread_id=chat_context.thread_id,
                    event_type=ChatThreadEventType.ASSISTANT_MESSAGE_ADDED,
                    message=event.message,
                    run_id=chat_context.run_id,
                )
```

#### 3. **New Endpoint: Subscribe to Thread Updates**

**File**: `src/sybill_py/runtimes/skynet/routers/chats/index.py`

```python
@router.get(
    "/chat/{threadId}/subscribe",
    response_class=ChatEventStreamOpenAPIResponse,
)
async def subscribe_to_chat_thread(
    request: Request,
    thread_id: UUID = Path(..., alias="threadId"),
    user_id: UUID = Depends(get_user_id_for_user_logged_in),
    acc_cn: str = Depends(get_acc_cn_for_user_logged_in),
    _claims=Depends(check_org_read_access_token),
) -> StreamingResponse:
    """
    Subscribe to real-time updates for a chat thread across all devices.

    This endpoint creates a long-lived SSE connection that receives events when:
    - User sends a message from ANY device
    - AI responds to a message
    - Chat metadata changes (title, etc.)

    Use this for cross-device synchronization.
    """
    org_id = await get_org_id_from_user_id_acc_cn(user_id, acc_cn, request.app.mongodb)

    # Verify user has access to this chat
    chat = await copilot_chats_dao.reader.get_chat_for_user_or_viewer(
        thread_id, org_id, user_id, acc_cn
    )
    if chat is None:
        raise HTTPException(status_code=404, detail="Chat not found")

    async def event_generator():
        """Generate SSE events from Redis pub/sub."""
        pubsub_client = RedisPubSubClient()

        try:
            async for thread_event in pubsub_client.subscribe_to_thread(thread_id):
                # Convert to SSE format
                sse_event = StreamingEvent(
                    event_type=thread_event.event_type,
                    data=thread_event.to_mongo(),
                )
                yield sse_event.to_sse()

        except asyncio.CancelledError:
            _logger.info(f"Client disconnected from thread {thread_id}")
        except Exception as e:
            _logger.error(f"Error in thread subscription: {e}", exc_info=True)

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        },
    )
```

#### 4. **Update DAL Writer to Broadcast Changes**

**File**: `src/sybill_py/dal/queries/copilot_chats/writer.py`

```python
from sybill_py.components.redis_pubsub.client import RedisPubSubClient

class CopilotChatsWriter(BaseWriter):
    def __init__(self):
        super().__init__()
        self.pubsub_client = RedisPubSubClient()

    async def update_chat_title_for_user(
        self, thread_id: UUID, org_id: UUID, user_id: UUID, title: str
    ) -> Chat | None:
        # Existing update logic
        chat = await super().update_chat_title_for_user(...)

        # NEW: Broadcast title change to all devices
        if chat:
            await self.pubsub_client.publish_message_event(
                thread_id=thread_id,
                event_type=ChatThreadEventType.CHAT_TITLE_UPDATED,
            )

        return chat
```

#### 5. **Frontend Integration**

**Client-Side Changes:**
```typescript
// NEW: Subscribe to thread-level updates
class ChatThreadSubscription {
  private eventSource: EventSource | null = null;

  subscribe(threadId: string) {
    // Close existing connection
    this.unsubscribe();

    // Create new SSE connection
    this.eventSource = new EventSource(
      `/api/chat/${threadId}/subscribe`,
      { withCredentials: true }
    );

    // Handle different event types
    this.eventSource.addEventListener('user_message_added', (event) => {
      const data = JSON.parse(event.data);
      // Update UI: add user message to chat
      this.onUserMessage(data.message);
    });

    this.eventSource.addEventListener('assistant_message_added', (event) => {
      const data = JSON.parse(event.data);
      // Update UI: add/update assistant message
      this.onAssistantMessage(data.message);
    });

    this.eventSource.addEventListener('chat_title_updated', (event) => {
      const data = JSON.parse(event.data);
      // Update UI: change chat title
      this.onTitleUpdated(data);
    });

    this.eventSource.onerror = (error) => {
      console.error('SSE connection error:', error);
      // Implement reconnection logic with exponential backoff
      this.reconnect();
    };
  }

  unsubscribe() {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
    }
  }
}

// Usage:
const subscription = new ChatThreadSubscription();

// When user opens a chat thread
subscription.subscribe(threadId);

// When user navigates away
subscription.unsubscribe();
```

### Option 2: WebSocket-Based (Alternative)

If SSE limitations become an issue (firewalls, proxies), use WebSockets:

**New File**: `src/sybill_py/runtimes/skynet/routers/websockets.py`
```python
from fastapi import WebSocket, WebSocketDisconnect

class ChatConnectionManager:
    """Manage WebSocket connections per thread."""

    def __init__(self):
        # Map of thread_id -> set of active connections
        self.active_connections: dict[UUID, set[WebSocket]] = {}

    async def connect(self, thread_id: UUID, websocket: WebSocket):
        await websocket.accept()
        if thread_id not in self.active_connections:
            self.active_connections[thread_id] = set()
        self.active_connections[thread_id].add(websocket)

    def disconnect(self, thread_id: UUID, websocket: WebSocket):
        if thread_id in self.active_connections:
            self.active_connections[thread_id].discard(websocket)

    async def broadcast(self, thread_id: UUID, message: dict):
        """Send message to all connections for this thread."""
        if thread_id in self.active_connections:
            dead_connections = set()

            for connection in self.active_connections[thread_id]:
                try:
                    await connection.send_json(message)
                except Exception:
                    dead_connections.add(connection)

            # Clean up dead connections
            self.active_connections[thread_id] -= dead_connections

manager = ChatConnectionManager()

@router.websocket("/ws/chat/{threadId}")
async def chat_websocket(
    websocket: WebSocket,
    thread_id: UUID,
):
    """WebSocket endpoint for real-time chat updates."""
    await manager.connect(thread_id, websocket)

    try:
        while True:
            # Keep connection alive and handle incoming messages
            data = await websocket.receive_json()

            # Process client -> server messages if needed
            # (e.g., typing indicators, read receipts)

    except WebSocketDisconnect:
        manager.disconnect(thread_id, websocket)
```

## Comparison: Pub/Sub vs WebSockets

| Feature | Redis Pub/Sub + SSE | WebSockets |
|---------|---------------------|------------|
| **Complexity** | Low (SSE built-in) | Medium (need WS manager) |
| **Bi-directional** | No (SSE is one-way) | Yes |
| **Browser Support** | Excellent (HTTP) | Good (requires WS) |
| **Proxy/Firewall** | Better (HTTP/1.1) | Can be blocked |
| **Reconnection** | Auto-handled by browser | Manual implementation |
| **Load Balancing** | Works seamlessly | Needs sticky sessions |
| **Typing Indicators** | Need separate endpoint | Built-in |

**Recommendation**: Start with **Option 1 (Redis Pub/Sub + SSE)** for simplicity. Upgrade to WebSockets only if you need bi-directional features (typing indicators, read receipts).

## Data Flow Diagrams

📊 **[View Interactive Diagram](docs/diagrams/before-after-comparison.excalidraw)** - Open in Excalidraw

### Before (Current System)
```
DeviceA: User types → POST /chat/v2 → Agent → SSE → DeviceA ✓
DeviceB:                                              ✗ (no update)
```

### After (Cross-Device Sync)
```
DeviceA: User types → POST /chat/v2 ─┐
                                      ├→ Agent → Redis Pub/Sub ─┬→ DeviceA ✓
DeviceB: GET /chat/{id}/subscribe ───┘                          └→ DeviceB ✓
```

## Performance Considerations

### Redis Pub/Sub Scaling
- **Memory**: Pub/Sub is lightweight (no message persistence)
- **Connections**: Each subscribed device = 1 Redis connection
- **Throughput**: Redis can handle 100K+ pub/sub messages/sec

### Connection Limits
- **Assumption**: ~1000 concurrent active chats
- **Per chat**: 2-3 devices subscribed on average
- **Total**: ~3000 Redis pub/sub connections (negligible)

### Fallback Strategy
If Redis pub/sub becomes a bottleneck:
1. Use Redis Cluster for horizontal scaling
2. Implement connection pooling per thread
3. Add WebSocket gateway (Socket.io, Centrifugo)

## Migration Path

### Phase 1: Infrastructure (Week 1)
- [ ] Create `RedisPubSubClient` class
- [ ] Add `ChatThreadEvent` schema
- [ ] Add `ChatThreadEventType` enum
- [ ] Unit tests for pub/sub client

### Phase 2: Backend Integration (Week 2)
- [ ] Modify `ChatPersistenceLayer` to broadcast events
- [ ] Update DAL writers to publish changes
- [ ] Create `/chat/{threadId}/subscribe` endpoint
- [ ] Integration tests for cross-device scenarios

### Phase 3: Frontend (Week 3)
- [ ] Implement `ChatThreadSubscription` service
- [ ] Connect to SSE endpoint on chat open
- [ ] Handle incoming events (add/update messages)
- [ ] Auto-reconnection logic
- [ ] E2E tests with multiple browser tabs

### Phase 4: Monitoring & Optimization (Week 4)
- [ ] Add metrics: active subscriptions, pub/sub throughput
- [ ] Error handling and retry logic
- [ ] Performance testing with 1000+ concurrent connections
- [ ] Document for team

## Security Considerations

### Authentication for Subscribe Endpoint
```python
# Verify user has access BEFORE allowing subscription
chat = await copilot_chats_dao.reader.get_chat_for_user_or_viewer(
    thread_id, org_id, user_id, acc_cn
)
if chat is None:
    raise HTTPException(status_code=403, detail="Access denied")
```

### Authorization Checks
- User must be **owner** or **viewer** of chat
- Org-level isolation (can't subscribe to other org's chats)
- Rate limiting on subscription endpoint

## Testing Strategy

### Unit Tests
```python
@pytest.mark.asyncio
async def test_pubsub_broadcast():
    """Test message is broadcast to all subscribers."""
    client = RedisPubSubClient()
    thread_id = uuid4()

    # Subscribe 2 devices
    events_device1 = []
    events_device2 = []

    async def collect_events(device_events):
        async for event in client.subscribe_to_thread(thread_id):
            device_events.append(event)
            if len(device_events) >= 1:
                break

    # Start subscribers
    task1 = asyncio.create_task(collect_events(events_device1))
    task2 = asyncio.create_task(collect_events(events_device2))

    await asyncio.sleep(0.1)  # Let subscribers connect

    # Publish message
    await client.publish_message_event(
        thread_id=thread_id,
        event_type=ChatThreadEventType.USER_MESSAGE_ADDED,
        message=mock_message,
    )

    # Wait for events
    await asyncio.gather(task1, task2)

    # Both devices should receive the event
    assert len(events_device1) == 1
    assert len(events_device2) == 1
    assert events_device1[0].event_type == ChatThreadEventType.USER_MESSAGE_ADDED
```

### Integration Tests
```python
@pytest.mark.asyncio
async def test_cross_device_message_sync(client: TestClient):
    """E2E test: message sent from device1 appears on device2."""
    thread_id = await create_test_chat()

    # Device 2 subscribes first
    device2_events = []
    async with client.stream(
        "GET", f"/chat/{thread_id}/subscribe"
    ) as response:
        # Device 1 sends message
        await client.post(
            "/chat/v2",
            json={"threadId": thread_id, "message": "Hello"}
        )

        # Device 2 should receive event
        async for line in response.aiter_lines():
            if line.startswith("data:"):
                event = json.loads(line[6:])
                device2_events.append(event)
                break

    assert len(device2_events) == 1
    assert device2_events[0]["eventType"] == "user_message_added"
```

## Monitoring & Observability

### Metrics to Track
```python
# In RedisPubSubClient
await runtime.metrics_client.increment(
    "chat.pubsub.message_published",
    tags={"event_type": event_type, "thread_id": str(thread_id)}
)

await runtime.metrics_client.gauge(
    "chat.pubsub.active_subscriptions",
    value=len(active_subscriptions),
)
```

### Logging
```python
_logger.info(
    f"[ThreadId={thread_id}] Published {event_type} to {subscriber_count} subscribers"
)
```

### Alerts
- Alert if Redis pub/sub latency > 100ms
- Alert if subscription error rate > 1%
- Alert if Redis connection pool exhausted

## Rollout Plan

### Beta Testing
1. Enable for internal team only (feature flag)
2. Test with 10-20 concurrent users
3. Monitor Redis metrics and error rates

### Gradual Rollout
1. 10% of users (1 week)
2. 50% of users (1 week)
3. 100% of users

### Rollback Strategy
- Feature flag to disable pub/sub broadcasting
- Fallback: clients poll GET `/chat/{threadId}` every 5 seconds
- No data loss (MongoDB remains source of truth)

## Cost Analysis

### Redis Usage
- **Current**: Redis used for streams only (4hr TTL)
- **New**: Add pub/sub channels (no persistence, minimal memory)
- **Estimated Increase**: < 5% Redis memory, < 10% CPU

### Infrastructure
- No additional servers needed
- Existing Redis cluster can handle pub/sub
- Consider Redis Cluster if scaling beyond 10K concurrent chats

## FAQ

### Q: What if Redis goes down?
**A**: Clients fall back to polling. MongoDB is source of truth, so no data loss.

### Q: What about message ordering?
**A**: Redis pub/sub preserves order within a channel. Clients should use `sequence` field for definitive ordering.

### Q: How do we handle network interruptions?
**A**: SSE auto-reconnects. On reconnect, client should:
1. Re-subscribe to thread
2. Fetch latest messages from MongoDB to fill gap
3. Resume receiving live updates

### Q: What about chat history sync?
**A**: Pub/sub only handles real-time updates. For history:
- Initial load: GET `/chat/{threadId}` (full MongoDB fetch)
- Updates: SSE subscription

### Q: Can we use this for typing indicators?
**A**: Yes! Add new event types:
```python
ChatThreadEventType.USER_TYPING_STARTED = "user_typing_started"
ChatThreadEventType.USER_TYPING_STOPPED = "user_typing_stopped"
```

## Summary

**Recommended Approach**: Redis Pub/Sub + SSE (Option 1)

**Key Changes:**
1. Thread-level pub/sub channel per chat
2. Broadcast all message/update events
3. New `/chat/{threadId}/subscribe` endpoint
4. Frontend subscribes on chat open

**Benefits:**
- ✅ True real-time sync across devices
- ✅ Low complexity (built on existing Redis)
- ✅ No additional infrastructure
- ✅ Graceful degradation (falls back to polling)

**Effort Estimate:**
- Backend: 2-3 weeks
- Frontend: 1-2 weeks
- Testing: 1 week
- **Total**: ~6 weeks for production-ready implementation
