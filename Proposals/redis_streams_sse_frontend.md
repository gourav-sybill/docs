# Redis Streams SSE — Implementation Plan & Architecture Overview

**Status:** Proposal
**Author:** Engineering Team
**Created:** February 2026
**Source:** [Redis Streams SSE Resilience Design](../Others/redis_streams_sse_resilience.md)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Implementation Plan](#3-implementation-plan)
4. [Phase 1 — Backend: Redis Stream Dual-Publishing](#phase-1--backend-redis-stream-dual-publishing)
5. [Phase 2 — Backend: Resume Endpoint](#phase-2--backend-resume-endpoint)
6. [Phase 3 — Frontend: SSE Client with Resume Support](#phase-3--frontend-sse-client-with-resume-support)
7. [Phase 4 — Frontend: Page Refresh Recovery](#phase-4--frontend-page-refresh-recovery)
8. [Phase 5 — Observability & Monitoring](#phase-5--observability--monitoring)
9. [Phase 6 — Rollout & Feature Flags](#phase-6--rollout--feature-flags)
10. [Risk Assessment & Mitigations](#4-risk-assessment--mitigations)
11. [Success Criteria](#5-success-criteria)
12. [Open Questions](#6-open-questions)

---

## 1. Executive Summary

Ask Sybill SSE connections break during long-running chats (~5 errors/day) due to load balancer draining, network changes, and general instability. When a connection drops, the frontend falls back to polling and delivers the entire response at once — degrading the real-time streaming experience.

**Solution:** Use Redis Streams as a durable message buffer. Every SSE event is dual-published (to the client AND to a Redis Stream). On disconnection, the frontend resumes from the last received message ID, receiving only missed events. Streams auto-expire after 4 hours.

**Key outcome:** Seamless reconnection with zero message loss and no perceptible interruption to the user.

---

## 2. Architecture Overview

### High-Level System Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                            FRONTEND                                  │
│                                                                      │
│  ┌─────────────┐    ┌──────────────────┐    ┌────────────────────┐  │
│  │  Chat UI     │◄──│  SSE Client       │◄──│  Resume Controller │  │
│  │  Component   │   │  (EventSource)    │   │  (retry + backoff) │  │
│  └─────────────┘    └──────────────────┘    └────────────────────┘  │
│        │                     │                        │              │
│        │              tracks lastMessageId      on error: resume     │
│        ▼                     ▼                        ▼              │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  State Manager (Redux / Zustand)              │   │
│  │  - lastMessageId: string | null                               │   │
│  │  - threadId: string                                           │   │
│  │  - streamStatus: 'idle' | 'streaming' | 'resuming' | 'done'  │   │
│  │  - accumulatedTokens: string                                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
                              │
              POST /agents/chats/v2 (initial)
              GET  /agents/chat/{threadId}/resume (reconnect)
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                            BACKEND                                   │
│                                                                      │
│  ┌────────────────────┐         ┌──────────────────────────┐        │
│  │  Chat SSE Handler  │────────►│  Stream Publisher         │        │
│  │  (POST /chats/v2)  │         │  - SSE to client          │        │
│  └────────────────────┘         │  - XADD to Redis Stream   │        │
│                                 └────────────┬─────────────┘        │
│  ┌────────────────────┐                      │                      │
│  │  Resume Handler    │◄─────────────────────┘                      │
│  │  (GET /resume)     │         ┌──────────────────────────┐        │
│  │  - XRANGE/XREAD    │────────►│  Redis (ElastiCache)     │        │
│  └────────────────────┘         │  - Stream: per thread     │        │
│                                 │  - Metadata: hash         │        │
│                                 │  - TTL: 4 hours           │        │
│                                 └──────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

| Scenario | Trigger | Endpoint | Redis Operation | Client Receives |
|----------|---------|----------|-----------------|-----------------|
| Normal streaming | User sends message | `POST /agents/chats/v2` | `XADD` per event | Real-time SSE events with `id` field |
| Network error resume | `EventSource.onerror` | `GET /resume?lastMessageId=X` | `XRANGE` from X+1 | Only missed events |
| Page refresh resume | Page load with active thread | `GET /resume` (no lastMessageId) | `XRANGE` from beginning | Full stream replay |
| Stream expired | Resume after 4h+ | `GET /resume` | Key not found | 404 → fall back to MongoDB |

### Event Lifecycle

```
  User sends message
        │
        ▼
  ┌─── ACK ──────────────────────────┐
  │  { threadId, messageId, runId }  │ ──► Redis XADD (id=0)
  └──────────────────────────────────┘
        │
        ▼
  ┌─── MESSAGE (user echo) ─────────┐
  │  { content: "user question" }   │ ──► Redis XADD (id=1)
  └──────────────────────────────────┘
        │
        ▼
  ┌─── DELTA (streamed tokens) ─────┐
  │  { content: "token..." }        │ ──► Redis XADD (id=2..N)
  └──────────────────────────────────┘  (repeated per token/chunk)
        │
        ▼
  ┌─── TOOL_CALL / TOOL_RESULT ─────┐
  │  (optional, interleaved)        │ ──► Redis XADD
  └──────────────────────────────────┘
        │
        ▼
  ┌─── CITATIONS / SUGGESTIONS ─────┐
  │  (optional)                     │ ──► Redis XADD
  └──────────────────────────────────┘
        │
        ▼
  ┌─── DONE ─────────────────────────┐
  │  { status: "complete" }         │ ──► Redis XADD + metadata.status = "complete"
  └──────────────────────────────────┘
```

### Redis Data Model

**Stream key:** `ask_sybill:chat:{thread_id}`

Each entry contains:
| Field | Type | Example |
|-------|------|---------|
| `event_type` | string | `"delta"` |
| `payload` | JSON string | `"{\"content\":\"Hello\"}"` |
| `timestamp` | int (ms) | `1706745600000` |
| `sequence` | int | `42` |

**Metadata key:** `ask_sybill:chat:{thread_id}:metadata` (Redis Hash)

| Field | Example |
|-------|---------|
| `status` | `"active"` / `"complete"` / `"error"` |
| `created_at` | `1706745600` |
| `thread_id` | `"uuid-..."` |
| `user_id` | `"uuid-..."` |

**TTL:** 4 hours on both keys. Max 10,000 messages per stream.

---

## 3. Implementation Plan

### Phase Overview

| Phase | Scope | Depends On | Estimated Effort |
|-------|-------|------------|------------------|
| **1** | Backend: Redis dual-publishing | — | Backend |
| **2** | Backend: Resume endpoint | Phase 1 | Backend |
| **3** | Frontend: SSE client with resume | Phase 2 | Frontend |
| **4** | Frontend: Page refresh recovery | Phase 3 | Frontend |
| **5** | Observability & monitoring | Phase 1 | Infra / Backend |
| **6** | Rollout & feature flags | Phase 3 | All |

---

### Phase 1 — Backend: Redis Stream Dual-Publishing

**Goal:** Every SSE event is written to a Redis Stream alongside being sent to the client. No frontend changes.

#### Tasks

1. **Redis connection setup**
   - Configure Redis client (ElastiCache) with connection pooling
   - Add health check for Redis connectivity
   - Ensure fallback: if Redis write fails, SSE still works (fire-and-forget)

2. **Stream creation on chat start**
   - On `POST /agents/chats/v2`, create Redis Stream key `ask_sybill:chat:{thread_id}`
   - Create metadata hash with `status=active`, `created_at`, `thread_id`, `user_id`
   - Set TTL of 4 hours on both keys

3. **Dual-publish every SSE event**
   - After writing each SSE event to the response stream, `XADD` to Redis
   - Include `event_type`, `payload` (JSON), `timestamp`, `sequence`
   - Use auto-generated Redis Stream IDs (timestamp-based) as `message_id`
   - Cap stream at 10,000 entries using `MAXLEN ~10000`

4. **Include `id` field in SSE events**
   - Each SSE event must include the `id:` field set to the Redis Stream message ID
   - This enables `EventSource.lastEventId` tracking on the client

5. **Mark stream complete on finish**
   - On `done` event, update metadata hash: `status=complete`
   - On error, update metadata hash: `status=error`

#### Acceptance Criteria
- All SSE events for a chat are stored in Redis Stream with correct ordering
- SSE events include `id` field with Redis Stream message ID
- Redis write failures do not break the SSE stream (graceful degradation)
- Streams auto-expire after 4 hours
- No observable latency increase on SSE delivery

---

### Phase 2 — Backend: Resume Endpoint

**Goal:** Expose `GET /agents/chat/{threadId}/resume` that replays missed events from Redis.

#### Tasks

1. **Create resume endpoint**
   ```
   GET /agents/chat/{threadId}/resume?lastMessageId={id}
   ```
   - Authenticate request (same auth as chat endpoints)
   - Validate `threadId` belongs to the requesting user
   - Return 404 for all error cases (privacy through obscurity)

2. **Implement replay logic**
   - If `lastMessageId` is provided: `XRANGE ask_sybill:chat:{thread_id} ({lastMessageId} +`
   - If `lastMessageId` is null: `XRANGE ask_sybill:chat:{thread_id} - +`
   - Stream results back as SSE events with correct `id`, `event`, `data` fields

3. **Handle active vs. complete streams**
   - If metadata `status=complete`: replay remaining messages, close connection
   - If metadata `status=active`: replay buffered messages, then `XREAD BLOCK` for live updates
   - If metadata `status=error` or key missing: return 404

4. **Set response headers**
   ```
   Content-Type: text/event-stream
   Cache-Control: no-cache
   Connection: keep-alive
   X-Accel-Buffering: no
   ```

#### Acceptance Criteria
- Resume from specific offset returns only events after that offset
- Resume with no offset returns the full stream from the beginning
- Active streams transition seamlessly from replay to live subscription
- 404 returned for expired, missing, or unauthorized streams
- Response format is identical to the original SSE stream

---

### Phase 3 — Frontend: SSE Client with Resume Support

**Goal:** Frontend automatically resumes on network error with zero message loss.

#### Tasks

1. **Track `lastMessageId` in state**
   ```typescript
   // On every SSE event received:
   if (event.lastEventId) {
     store.setLastMessageId(event.lastEventId);
   }
   ```

2. **Detect connection loss**
   - Listen for `EventSource.onerror` or `fetch` stream abort
   - Distinguish between recoverable network errors and terminal errors (e.g., 4xx)

3. **Implement resume with exponential backoff**
   ```
   Retry sequence: 1s → 2s → 4s → 8s → 16s (max)
   Max retries: 5
   ```
   - On network error, wait then call `GET /resume?lastMessageId={lastMessageId}`
   - On success (200): consume resumed stream, reset retry counter
   - On 404: stream expired — fall back to polling / MongoDB fetch
   - On other error: retry with backoff

4. **Deduplicate events on resume**
   - The server ensures no overlap (exclusive range query), but as a safety measure:
   - Track `lastMessageId` and skip any event with ID <= `lastMessageId`

5. **Update UI state during resume**
   - Set `streamStatus = 'resuming'` while reconnecting
   - Optionally show subtle "Reconnecting..." indicator
   - On successful resume, set `streamStatus = 'streaming'`
   - Append resumed tokens to existing `accumulatedTokens` (no duplicate content)

6. **Handle concurrent streams**
   - Ensure only one active SSE connection per thread at a time
   - Abort previous connection before starting resume

#### Acceptance Criteria
- Network disconnection mid-stream triggers automatic resume
- User sees continuous token streaming with no visible gap
- No duplicate tokens or events in the UI
- Maximum 5 retry attempts before falling back to polling
- "Reconnecting" state visible in UI during resume attempts

---

### Phase 4 — Frontend: Page Refresh Recovery

**Goal:** If the user refreshes the page while a stream is active, reconstruct the conversation from Redis.

#### Tasks

1. **Detect active stream on page load**
   - On chat page mount, check if the current thread has an active stream
   - Call `GET /agents/chat/{threadId}/resume` (no `lastMessageId`)

2. **Reconstruct conversation from stream**
   - If 200: consume the full stream
     - `message` event → render user message
     - `delta` events → accumulate into AI response
     - `tool_call` / `tool_result` → render tool interactions
     - `done` → mark stream complete
   - If 404: no active stream — load from MongoDB (existing behavior)

3. **Merge with MongoDB data**
   - On page load, fetch chat history from MongoDB in parallel
   - If resume stream is available, prefer stream data for the latest turn
   - Older turns come from MongoDB as before

4. **Handle edge case: stream completes during page load**
   - Between page load and resume call, the stream may complete and expire
   - Gracefully handle 404 by falling back to MongoDB

#### Acceptance Criteria
- Page refresh during active stream restores full conversation with streaming
- User sees their message + AI response building in real-time after refresh
- No duplicate messages when merging MongoDB history with stream data
- Graceful fallback when stream has already expired

---

### Phase 5 — Observability & Monitoring

**Goal:** Full visibility into stream health, resume usage, and Redis resource consumption.

#### Tasks

1. **Backend metrics** (DataDog / CloudWatch)

   | Metric | Type | Description |
   |--------|------|-------------|
   | `stream.created` | Counter | Streams created per minute |
   | `stream.completed` | Counter | Streams that reached `done` event |
   | `stream.error` | Counter | Streams that ended in error |
   | `stream.publish_latency_ms` | Histogram | Time to XADD per event |
   | `resume.requests` | Counter | Resume endpoint calls |
   | `resume.success` | Counter | Successful resumes (200) |
   | `resume.not_found` | Counter | 404 responses on resume |
   | `resume.replay_count` | Histogram | Number of events replayed per resume |
   | `redis.memory_usage_bytes` | Gauge | Memory used by stream keys |
   | `redis.active_streams` | Gauge | Count of active (non-expired) streams |

2. **Frontend metrics** (Amplitude)

   | Event | Properties |
   |-------|------------|
   | `sse_connection_lost` | `{ threadId, lastMessageId, errorType }` |
   | `sse_resume_attempted` | `{ threadId, lastMessageId, retryCount }` |
   | `sse_resume_success` | `{ threadId, resumedEventCount, latencyMs }` |
   | `sse_resume_failed` | `{ threadId, errorCode, retryCount }` |
   | `sse_resume_fallback` | `{ threadId, reason }` |
   | `page_refresh_resume` | `{ threadId, success, eventCount }` |

3. **Alerts**

   | Alert | Condition | Severity |
   |-------|-----------|----------|
   | Redis memory high | > 80% capacity | P2 |
   | Resume 404 rate high | > 10% of resume calls | P3 |
   | Stream creation failures | > 1% failure rate | P2 |
   | Redis publish latency | p99 > 50ms | P3 |

4. **Dashboard**
   - Real-time view: active streams, resume rate, Redis memory
   - Daily view: total streams, resume success rate, error breakdown
   - Comparison: pre/post feature SSE error rates from Amplitude

#### Acceptance Criteria
- All metrics emitting correctly in staging before production rollout
- Dashboard created and accessible to the team
- Alerts configured and tested with synthetic triggers

---

### Phase 6 — Rollout & Feature Flags (Informal plan)

**Goal:** Safe, incremental rollout with instant rollback capability.

#### Strategy

The backend (dual-publishing) has zero client-side impact — it runs for 100% of traffic from day one. The frontend (resume logic) rolls out gradually behind a feature flag.

#### Rollout Plan

| Step | Backend | Frontend | Duration | Gate |
|------|---------|----------|----------|------|
| **1. Backend deploy** | Enable dual-publish 100% | No changes | 1 week | Redis memory + latency stable |
| **2. Internal dogfood** | No changes | Enable for internal users | 1 week | No regressions, metrics healthy |
| **3. Beta** | No changes | Enable for beta cohort (~5%) | 1 week | Resume success rate > 90% |
| **4. Gradual rollout** | No changes | 10% → 25% → 50% → 100% | 2 weeks | Error rates stable or declining |

#### Feature Flag

```
Flag: ask_sybill_sse_resume_enabled
Type: boolean
Default: false
Targeting: user_id percentage rollout
```

#### Frontend flag usage

```typescript
const isResumeEnabled = useFeatureFlag('ask_sybill_sse_resume_enabled');

// In SSE error handler:
if (isResumeEnabled) {
  await resumeStream(threadId, lastMessageId);
} else {
  await fallbackToPoll(threadId);  // existing behavior
}
```

#### Rollback

| Scenario | Action | Impact |
|----------|--------|--------|
| Redis issues (memory, latency) | Disable dual-publish in backend | SSE works as before, resume returns 404 |
| Resume endpoint bugs | Frontend flag → 0% | Falls back to polling (existing behavior) |
| Frontend resume bugs | Frontend flag → 0% | Immediate, no deploy needed |

#### Acceptance Criteria
- Feature flag toggleable without deploy
- Rollback to 0% frontend resume takes < 5 minutes
- Backend can disable Redis publishing independently
- Each rollout step has documented go/no-go criteria

---

## 4. Risk Assessment & Mitigations

Claude generated possible risks / mitigations.

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Redis memory spike from large streams | Medium | High | MAXLEN ~10,000 cap per stream; 4h TTL; memory monitoring alert |
| XADD latency degrades SSE performance | Low | High | Fire-and-forget writes; benchmark shows <1ms p99 for XADD |
| Resume delivers duplicate tokens to UI | Medium | Medium | Server-side exclusive range query + client-side dedup guard |
| Stream expires before user resumes | Low | Low | 4h TTL is generous; 404 gracefully falls back to MongoDB |
| Redis cluster failover during active stream | Low | Medium | Stream data is transient; failover results in 404 → fallback |
| Race condition: resume called while stream still publishing | Medium | Medium | XREAD BLOCK handles this — transitions from replay to live |

---

## 5. Success Criteria

### Quantitative

| Metric | Current | Target |
|--------|---------|--------|
| SSE errors resulting in lost streaming UX | ~5/day | < 1/day |
| Resume success rate (when attempted) | N/A | > 95% |
| Added latency per SSE event (dual-publish) | 0ms | < 5ms p99 |
| Redis memory overhead | 0 | < 500MB peak |

### Qualitative

- Users experience uninterrupted streaming even on flaky networks
- Page refresh during active stream restores conversation seamlessly
- No user-reported issues related to duplicate or missing content
- Engineering team has full observability into stream health
