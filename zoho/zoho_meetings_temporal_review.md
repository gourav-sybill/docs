# PR #2266 Review: Zoho Meetings Temporal Polling Workflow

## Table of Contents

- [Data Flow Overview](#data-flow-overview)
- [Phase 1: Polling Discovery](#phase-1-polling-discovery)
- [Phase 2: Recording Processing](#phase-2-recording-processing)
- [Phase 3: Upstream Processing](#phase-3-upstream-processing)
- [Phase 4: Call Processing](#phase-4-call-processing)
- [Component Deep Dives](#component-deep-dives)
- [Comparison with Gong and Other Upstream Providers](#comparison-with-gong-and-other-upstream-providers)
- [Test Coverage Analysis](#test-coverage-analysis)
- [Issues Found](#issues-found)
- [Things Done Correctly](#things-done-correctly)

---

## Data Flow Overview

```
┌───────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: POLLING DISCOVERY                                              │
│  ZohoMeetingsPollingWorkflow (infinite loop via continue_as_new)         │
│                                                                          │
│  FetchOrgsWithZohoMeetingsActivity                                       │
│    → organizations_dao.reader.list_org_ids_with_zoho_meetings_integration │
│    → Returns: [org_id_1, org_id_2, ...]                                  │
│                                                                          │
│  For each org (parallel, semaphore-bounded):                             │
│    CheckFeatureFlagActivity(ZOHO_MEETINGS_INTEGRATION)                   │
│      → Skip org if disabled                                              │
│    PollZohoRecordingsForOrgActivity                                      │
│      → OAuth token → ZohoMeetingsProvider → fetch_all_recordings()       │
│      → Filter by lookback window → Deduplicate via DB                    │
│      → Returns: [meeting_key_1, meeting_key_2, ...]                      │
│                                                                          │
│    For each new meeting_key (serial):                                    │
│      ProcessZohoRecordingActivity                                        │
│        → Re-fetch recording → Create UpstreamMeeting doc                 │
│        → Create MetadataPointer → Start UpstreamProcessingWorkflow       │
│                                                                          │
│  Sleep(polling_interval) → continue_as_new()                             │
└───────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: UPSTREAM PROCESSING                                            │
│  UpstreamProcessingWorkflow (per recording)                              │
│                                                                          │
│  CreateUpstreamParticipantsActivity                                      │
│    → Fetch participants from Zoho API                                    │
│    → Build UpstreamCallMetadata (topic, times, invitees, participants)   │
│    → Enrich with CRM person/company associations                        │
│                                                                          │
│  DoCrmMatchingActivity                                                   │
│    → SKIPPED for Zoho Meetings (no CRM object generation)               │
│    → should_ignore_recorder_prefs = True (no system identities)         │
│                                                                          │
│  StartUpstreamCallProcessingWorkflowsActivity                            │
│    → Starts CallProcessingWorkflow                                       │
└───────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: CALL PROCESSING                                                │
│  CallProcessingWorkflow (standard pipeline)                              │
│                                                                          │
│  Download recording → ASR/Transcription → NLP → FER → Core Insights     │
│  → POST_ML_COMPLETED → Vector ingestion → Deal sync                     │
│  → Call is now queryable in Sybill dashboard                             │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Polling Discovery

### Workflow: `ZohoMeetingsPollingWorkflow`

**File:** `src/sybill_py/runtimes/galactus/workflows/background/zoho_meetings_polling.py`

**Registration:** `src/sybill_py/runtimes/galactus/registry.py:725` (BACKGROUND_WORKFLOWS list)

**Input:** `ZohoMeetingsPollingWorkflowInput` (`src/sybill_py/components/schema/galactus.py:2431`)
- `org_ids: list[UUID]` — empty = fetch all orgs with active integration
- `parallel_orgs: int` — max concurrent org processing (default 5, range 1–20)
- `polling_interval: int` — sleep between polls in ms (default 5 min)
- `lookback_days: int` — lookback window for recordings (default 7, range 1–30)

**Execution flow:**

1. If `org_ids` is empty, executes `FetchOrgsWithZohoMeetingsActivity` to discover all orgs with active Zoho Meetings integrations.

2. Creates an `asyncio.Semaphore(parallel_orgs)` to bound concurrency.

3. For each org, calls `poll_recordings_for_org()` concurrently (semaphore-bounded):
   - Checks `ZOHO_MEETINGS_INTEGRATION` feature flag via `CheckFeatureFlagActivity`. Skips if disabled.
   - Executes `PollZohoRecordingsForOrgActivity` to discover new recordings.
   - For each new recording, executes `ProcessZohoRecordingActivity` serially.

4. Aggregates results across all orgs (recordings found, processed, errors).

5. Sleeps for `polling_interval` ms, then calls `workflow.continue_as_new()` with the same input parameters. This creates an infinite polling loop without history bloat.

**Error isolation:** One org's failure does not stop processing of other orgs. Errors are captured in the output list.

### Activity: `FetchOrgsWithZohoMeetingsActivity`

**File:** `src/sybill_py/runtimes/galactus/activities/background/zoho_meetings_polling.py:60`

**Retry policy:** `DurableAsyncActivity` (default retry with backoff)

Queries `organizations_dao.reader.list_org_ids_with_zoho_meetings_integration()` to find all org IDs that have an active `ZOHO_MEETINGS_TEAM` integration. Returns `FetchOrgsWithZohoMeetingsActivityOutput` containing a `list[UUID]`.

### Activity: `PollZohoRecordingsForOrgActivity`

**File:** `src/sybill_py/runtimes/galactus/activities/background/zoho_meetings_polling.py:75`

**Retry policy:** `NonRetriableAsyncActivity` with `NoRetryPolicy(should_pause=False)` — fails immediately, workflow continues with other orgs.

**Steps:**

1. **Fetch org integration info** — calls `integration_utils.fetch_org_integration_info()` to get the `ZohoMeetingsIntegrationInfoTeam` (dc_region, zsoid, identity) and the primary `MeetingProviderIntegrationTeam` (auth0_user_id for OAuth).

2. **Check connection health** — if the integration has a `connection_health_error`, returns error immediately (e.g., expired refresh token).

3. **Get OAuth access token** — calls `_get_zoho_meetings_access_token()` helper, which uses `ZohoMeetingsCredentialsManager` to decrypt the refresh token from Auth0 `app_metadata` and exchange it for a fresh access token.

4. **Fetch all recordings** — creates `ZohoMeetingsProvider` with dc_region, access_token, and zsoid. Calls `provider.fetch_all_recordings()` which hits `GET /meeting/api/v2/{zsoid}/recordings.json`.

5. **Filter by lookback window** — only keeps recordings where `start_time_in_ms >= (now - lookback_days)`.

6. **Deduplicate** — for each recent recording, calls `upstream_meetings_dao.reader.find_by_remote_id_and_org()` to check if an UpstreamMeeting already exists with the same `remote_id` (meeting_key), `org_id`, and `provider`. Only new recordings are returned.

7. **Return** — `PollZohoRecordingsForOrgActivityOutput` with `recording_ids` (list of meeting keys for new recordings), plus counts for `recordings_found`, `new_recordings_count`, `skipped_count`.

---

## Phase 2: Recording Processing

### Activity: `ProcessZohoRecordingActivity`

**File:** `src/sybill_py/runtimes/galactus/activities/background/zoho_meetings_polling.py:187`

**Retry policy:** `DurableAsyncActivity` (default retry with backoff)

**Steps:**

1. **Fetch org integration info** — same as poll activity.

2. **Get OAuth access token** — same as poll activity.

3. **Fetch full recording details** — creates `ZohoMeetingsProvider`, then calls `provider.fetch_all_recordings()` and finds the matching recording by `meeting_key`. Note: the single-recording endpoint (`GET /meeting/api/v2/{zsoid}/recordings/{meeting_key}.json`) returns incomplete data (missing meetingKey, startTimeinMs), so the list endpoint is used instead.

4. **Check download URL** — calls `provider.get_recording_download_url(recording)`. Returns error if no download URL (recording not yet available for download).

5. **Create UpstreamMeeting document** — builds the MongoDB document:
   ```
   {
     _id: <new UUID>,
     remoteId: <meeting_key>,
     provider: "ZOHO_MEETINGS",
     orgId: <org_id>,
     payload: <ZohoRecording.to_mongo()>,
     stateEvents: [{ event: "UPSTREAM_MEETING_RECEIVED", timestamp: <now> }],
     lastStateEvent: { ... }
   }
   ```
   Inserts into `runtime.app_db.upstreamMeetings`.

6. **Create MetadataPointer** — upserts into `runtime.app_db.metadataPointers`:
   ```
   { meetingId: <meeting_key>, upstreamMeetingId: <upstream_meeting_id> }
   ```
   This links the recording to the UpstreamMeeting for downstream CallProcessingWorkflow lookup.

7. **Start UpstreamProcessingWorkflow** — triggers via Temporal client:
   - Workflow ID: `"upstream_{upstream_meeting_id}"`
   - Input: `{ upstream_meeting_id, org_id, crm_matching: True }`
   - Task queue: inherited from current activity worker

---

## Phase 3: Upstream Processing

### Workflow: `UpstreamProcessingWorkflow`

**File:** `src/sybill_py/runtimes/galactus/workflows/upstream_processing.py`

This is the shared workflow used by all upstream providers (Gong, Chorus, Fathom, Zoho Meetings). It runs three activities in sequence:

### Activity: `CreateUpstreamParticipantsActivity`

**File:** `src/sybill_py/runtimes/galactus/activities/upstream_processing.py:140`

For `ZOHO_MEETINGS` provider (lines 196–228):

1. Fetches `ZohoMeetingsIntegrationInfoTeam` from org.
2. Gets OAuth access token via `ZohoMeetingsCredentialsManager`.
3. Creates `ZohoMeetingsProvider` instance.
4. Deserializes the stored recording from `upstream_meeting_info.payload` via `ZohoRecording.from_mongo()`.
5. Fetches participants from Zoho API: `provider.fetch_participants(remote_id)`.
6. Builds metadata: `provider.create_upstream_metadata(recording, participants, user_map)`.
7. Enriches with CRM person/company associations via `populate_person_and_company_associations_for_upstream_call_metadata()`.
8. Patches the UpstreamMeeting with the enriched `UpstreamCallMetadata`.

### Activity: `DoCrmMatchingActivity`

**File:** `src/sybill_py/runtimes/galactus/activities/upstream_processing.py:258`

For Zoho Meetings:
- **CRM matching is skipped** (line 290): `provider != MeetingProviders.ZOHO_MEETINGS`
- **Recorder preferences are always ignored** (lines 332–335): `should_ignore_recorder_prefs = True` for ZOHO_MEETINGS, because Zoho Meetings doesn't sync individual users with system identities. All recordings are processed.

### Activity: `StartUpstreamCallProcessingWorkflowsActivity`

**File:** `src/sybill_py/runtimes/galactus/activities/upstream_processing.py:378`

Starts `CallProcessingWorkflow` for the recording. If `STOP_PROC` state event was set by `DoCrmMatchingActivity`, processing is skipped.

---

## Phase 4: Call Processing

### Workflow: `CallProcessingWorkflow`

**File:** `src/sybill_py/runtimes/galactus/workflows/call_processing.py`

Standard pipeline shared by all providers. Downloads the recording, runs ASR/transcription, NLP analysis, FER (facial expression recognition if applicable), generates core insights, and stores results. On completion, fires `POST_ML_COMPLETED` state event which triggers vector ingestion and deal sync.

---

## Component Deep Dives

### ZohoMeetingsProvider

**File:** `src/sybill_py/components/upstream_providers/zoho_meetings/provider.py`

Registered via `@UpstreamProviderRegistry.register(MeetingProviders.ZOHO_MEETINGS)`.

Implements all required `BaseUpstreamProvider` methods:

| Method | Implementation |
|--------|---------------|
| `fetch_all_recordings()` | Uses `ZohoMeetingsClient` context manager, optionally sets zsoid, returns `list[ZohoRecording]` |
| `fetch_recording(meeting_key)` | Fetches single recording, returns `ZohoRecording | None` |
| `fetch_participants(meeting_key)` | Calls `client.get_all_participants()` with automatic pagination |
| `create_upstream_metadata()` | Transforms Zoho data to `UpstreamCallMetadata` with invitees, participants, meeting type |
| `get_recording_download_url()` | Returns `recording.download_url` |
| `get_recording_id()` | Returns `recording.meeting_key` |
| `get_recording_timestamp()` | Returns `recording.start_time_in_ms` |

Optional methods implemented:
- `fetch_meeting_details()` — returns session details dict
- `get_transcript_download_url()` — returns URL if `is_transcript_generated` is True
- `has_native_transcript()` — checks `is_transcript_generated` flag

**Metadata creation details:**
- Topic: uses `recording.topic` or "Untitled Meeting"
- Remote owner: first participant with `role == "presenter"`
- End time: uses `end_time_millis` if available, else `start_time_in_ms + duration`
- Meeting type: `EXTERNAL` if any participant not in Sybill user map, else `INTERNAL`
- Conference provider: `ConferenceProviders.ZOHO`
- Invitees: created from participants with valid emails; marked `is_internal`/`is_sybill_user` based on user map
- Participants: includes unresolved speaker IDs for LLM inference

### ZohoMeetingsClient

**File:** `src/sybill_py/components/upstream_providers/zoho_meetings/client.py`

Async HTTP client inheriting `SybillAsyncClient`. Supports all 8 DC regions with region-specific base URLs:

| Region | Base URL |
|--------|----------|
| US | `https://meeting.zoho.com` |
| EU | `https://meeting.zoho.eu` |
| IN | `https://meeting.zoho.in` |
| AU | `https://meeting.zoho.com.au` |
| UK | `https://meeting.zoho.com` (same as US) |
| CA | `https://meeting.zohocloud.ca` |
| JP | `https://meeting.zoho.jp` |
| SA | `https://meeting.zoho.sa` |

Authorization: `"Zoho-oauthtoken {access_token}"`

Key API methods:
- `get_user_details()` — `GET /api/v2/user.json` — fetches and caches zsoid
- `get_all_recordings()` — `GET /meeting/api/v2/{zsoid}/recordings.json`
- `get_recording(meeting_key)` — `GET /meeting/api/v2/{zsoid}/recordings/{meeting_key}.json`
- `get_meeting(meeting_key)` — `GET /api/v2/{zsoid}/sessions/{meeting_key}.json`
- `get_participants(meeting_key)` — `GET /api/v2/{zsoid}/participant/{meeting_key}.json` with pagination
- `get_all_participants(meeting_key)` — auto-paginates through all participants

Error handling via `_handle_response()`:
- 429 → `ZohoMeetingsRateLimitException`
- 401/403 → `ZohoMeetingsAuthException` (extends `ConnectionHealthException`)
- Other 4xx/5xx → `ZohoMeetingsAPIError`

### ZohoMeetingsCredentialsManager

**File:** `src/sybill_py/components/auth0/credentials/zoho_meetings.py`

Static method `get_zoho_meetings_cred_by_auth0_user_id()`:
1. Looks up Auth0 `connection_id` for the dc_region from `app_config.auth0_ref.zoho_meetings_connection_ids`
2. Gets the Zoho auth URL for the region via `OAuth2AsyncClient.ZOHO_AUTH_URL_V2(dc_region)`
3. Delegates to `OAuth2CredentialsManager.get_oauth2_cred_by_auth0_user_id()` for actual token refresh
4. Returns `OAuth2TokenInfo` containing `access_token: SecretStr`

### Schemas

**File:** `src/sybill_py/components/upstream_providers/zoho_meetings/schema.py`

| Class | Key Fields |
|-------|-----------|
| `ZohoRecording` | erecording_id, meeting_key, topic, download_url, play_url, share_url, transcription_download_url, is_transcript_generated, duration, start_time_in_ms, end_time_millis, status, is_meeting |
| `ZohoParticipant` | email, role (presenter/participant), join_time, leave_time, duration, member_id, source |
| `ZohoMeetingSession` | Meeting session metadata |
| `ZohoUserDetails` | zsoid (org ID), zuid (user ID), primary_email, is_admin |

---

## Comparison with Gong and Other Upstream Providers

### Architecture Differences

| Aspect | Gong | Chorus | Fathom | Zoho Meetings |
|--------|------|--------|--------|---------------|
| **Ingestion** | Historical polling (legacy) | Historical polling (legacy) | Historical polling (legacy) | Temporal polling workflow |
| **Provider Registry** | Not registered | Not registered | Not registered | Registered via `@UpstreamProviderRegistry.register` |
| **Auth** | Basic Auth (access_key + access_secret) | OAuth2 | OAuth2 | OAuth2 (multi-region) |
| **Multi-Region** | No | No | No | Yes (8 regions) |
| **Workspace Selection** | Yes (user selects workspace) | No | No | No (uses zsoid as remote_org_id) |
| **User Syncing** | Yes (per-user system identities) | Yes | Yes | No (org-level only) |
| **CRM Matching** | Yes (`create_gong_crm_objects`) | Yes (`create_chorus_crm_objects`) | Yes (`create_fathom_crm_objects`) | Skipped |
| **Recorder Preferences** | Configurable (`ignore_recorder_prefs`) | Standard | Configurable | Always ignored |
| **Health Monitoring** | No | No | No | Yes (`ConnectionHealthMixin`) |
| **Feature Flag** | No | No | No | Yes (`ZOHO_MEETINGS_INTEGRATION`) |
| **Deduplication** | Manual in transform function | Manual | Manual | DAL query (`find_by_remote_id_and_org`) |
| **Metadata Transform** | `create_gong_metadata()` utility | `create_chorus_metadata()` utility | `create_fathom_metadata()` utility | `provider.create_upstream_metadata()` method |
| **Conference Mapping** | `GONG_CONF_SYS_MAP` check | N/A | N/A | N/A (single provider) |
| **Native Transcript** | No | Yes (utterance-level) | No | Yes (if generated by Zoho) |

### How Gong Ingestion Works (for reference)

Gong uses a legacy polling approach without a dedicated Temporal workflow:

```
GongClient.get_historical_calls()
  → transform_raw_gong_call_safely()    [upstream_meeting_utils.py]
  → insert_new_upstream_meeting()        [creates UpstreamMeeting doc]
  → UpstreamProcessingWorkflow triggered
  → CreateUpstreamParticipantsActivity   [creates metadata via create_gong_metadata()]
  → DoCrmMatchingActivity               [creates CRM objects, checks recorder prefs]
  → StartUpstreamCallProcessingWorkflowsActivity
  → CallProcessingWorkflow
```

Key differences in `DoCrmMatchingActivity` (lines 258–376 of `upstream_processing.py`):
- Gong fetches `remote_users` from the integration, matches them to CRM users, and generates CRM objects.
- Gong checks `should_summarise_meeting()` against recorder preferences — if user hasn't opted in, recording gets `STOP_PROC`.
- Zoho Meetings skips both (no system identities, no CRM objects).

### What Zoho Meetings Intentionally Does Not Implement

These are **not bugs** — they are correct decisions based on Zoho Meetings' nature as an org-level integration without per-user sync:

1. **CRM object generation** — Zoho Meetings doesn't have the per-user workspace mapping needed to match recording owners to CRM users. Gong gets this via workspace user sync.

2. **Recorder preference filtering** — Zoho Meetings doesn't sync individual users as system identities, so there's no user-level opt-in/opt-out for recording processing. All recordings are processed.

3. **Webhook ingestion** — Zoho Meetings API doesn't support recording webhooks. Polling is the correct approach.

---

## Test Coverage Analysis

### Test Files

| File | Type | Tests |
|------|------|-------|
| `tests/sybill_py/runtimes/galactus/activities/background/test_zoho_meetings_polling.py` | Activity unit tests | 12 |
| `tests/sybill_py/runtimes/galactus/workflows/background/test_zoho_meetings_polling_workflow.py` | Workflow unit tests | 10 |
| `tests/sybill_py/runtimes/galactus/workflows/background/test_zoho_meetings_polling_integration.py` | Workflow integration tests (real Temporal + real MongoDB) | 6 |
| `tests/sybill_py/components/upstream_providers/zoho_meetings/test_client.py` | Client unit tests | ~15 |
| `tests/sybill_py/components/upstream_providers/zoho_meetings/test_provider.py` | Provider unit tests | ~20 |
| `tests/sybill_py/components/upstream_providers/zoho_meetings/test_schema.py` | Schema tests | ~10 |
| `tests/sybill_py/components/upstream_providers/zoho_meetings/test_exceptions.py` | Exception tests | ~5 |

### What IS Tested

**Activities (12 tests):**
- `FetchOrgsWithZohoMeetingsActivity`: success + empty result
- `PollZohoRecordingsForOrgActivity`: no integration, connection health error, token failure, success with deduplication (1 new + 1 existing)
- `ProcessZohoRecordingActivity`: no integration, recording not found, no download URL, success (creates UpstreamMeeting + MetadataPointer + starts workflow)
- `_get_zoho_meetings_access_token`: success + failure

**Workflow unit tests (10 tests):**
- Automatic org fetching when `org_ids` empty
- Using provided `org_ids` directly
- Early return when no orgs
- Result aggregation across multiple orgs
- Polling error handling (one org fails, others continue)
- Recording processing (2 recordings found, both processed)
- Processing error handling (1 success + 1 failure)
- `continue_as_new` triggered when `polling_interval > 0`
- Exception handling in `poll_recordings_for_org`

**Workflow integration tests (6 tests, real Temporal + real MongoDB):**
- Fetches orgs from actual database
- Discovers and processes new recordings (verifies UpstreamMeeting docs in MongoDB)
- Deduplicates against existing UpstreamMeeting docs
- Handles no orgs in database
- Handles auth errors gracefully
- Processes multiple orgs in parallel

**Client tests (~15):**
- Initialization for all 8 DC regions
- User details retrieval + zsoid caching
- `_ensure_zsoid` edge cases (cached, fetch, error)
- Recording listing (success + empty)
- Single recording fetch (success + 404 + empty list)
- Meeting details (success + 404)
- Participant listing with pagination parameters
- Participant 404 handling

**Provider tests (~20):**
- Initialization (with and without zsoid)
- `fetch_all_recordings`, `fetch_recording`, `fetch_participants`, `fetch_meeting_details`
- Metadata creation: external meeting, internal meeting, no presenter, untitled meeting, explicit end_time_millis
- Helper methods: download URL, recording ID, timestamp, transcript URL (available + not generated), `has_native_transcript`, dashboard URL (play_url priority, share_url fallback, none)
- Invitee creation: with emails (internal/external mapping), skipping no-email participants

### What IS NOT Tested

#### Priority 1 — Critical Gaps

1. **`ZohoMeetingsCredentialsManager` — 0% direct coverage**
   - Only tested via mocks in activity tests. No tests verify:
     - Connection ID lookup by region from `app_config`
     - OAuth2 token refresh delegation to `OAuth2CredentialsManager`
     - `RuntimeError` when `zoho_meetings_connection_ids` not configured
     - Multi-region connection routing

2. **Client error response handling — incomplete**
   - `_handle_response()` is not tested for:
     - 429 rate limit → `ZohoMeetingsRateLimitException`
     - 401/403 auth errors → `ZohoMeetingsAuthException`
     - 500/503 server errors → `ZohoMeetingsAPIError`
     - Malformed JSON responses
     - Network timeouts

3. **Feature flag disabled path — not tested in integration tests**
   - Workflow unit tests mock the flag, but no integration test verifies the workflow actually skips an org when the flag is disabled.

#### Priority 2 — Important Gaps

4. **`_create_participants` method — not directly tested**
   - Only tested indirectly via `create_upstream_metadata`. No tests for:
     - Unresolved speaker ID tracking
     - Participant without email getting placeholder name
     - Duplicate participant handling

5. **Activity DB failure paths**
   - No tests for exceptions during:
     - `runtime.app_db.upstreamMeetings.insert_one()`
     - `runtime.app_db.metadataPointers.find_one_and_update()`
     - `worker.client.start_workflow()`

6. **`get_all_participants` pagination — not tested**
   - Client's `get_all_participants()` auto-pagination logic (iterating through pages when `total_count > page_size`) has no test.

#### Priority 3 — Nice to Have

7. **Large-scale behavior** — no tests for >100 recordings per org, >100 participants per meeting.
8. **Workflow cancellation** — no tests for what happens when workflow is cancelled mid-execution.
9. **Race conditions** — no tests for concurrent processing of same recording by overlapping poll cycles.

### Coverage Summary

| Component | Positive Paths | Error Paths | Overall |
|-----------|---------------|-------------|---------|
| Activities | ~95% | ~60% | ~80% |
| Workflow | ~90% | ~70% | ~85% |
| Provider | ~85% | ~30% | ~65% |
| Client | ~75% | ~15% | ~50% |
| Credentials Manager | 0% (mocked only) | 0% | 0% |

---

## Issues Found

### MEDIUM: No Rate Limit Handling in Polling Loop

**Location:** `zoho_meetings_polling.py:134` (PollZohoRecordingsForOrgActivity)

The polling activity calls `provider.fetch_all_recordings()` which can return a `ZohoMeetingsRateLimitException` (429). This exception bubbles up as a generic error in the activity output. The workflow doesn't differentiate rate limits from other errors — it simply logs and moves on.

**Impact:** If Zoho's rate limit is hit while polling org N, the remaining orgs in the same poll cycle may also hit the limit. There's no backoff or delay between orgs after a rate limit.

**What Gong does differently:** Gong uses basic auth with higher rate limits and doesn't have a continuous polling loop.

**Recommendation:** Consider catching `ZohoMeetingsRateLimitException` specifically and adding a delay before processing subsequent orgs.

---

### MEDIUM: Recording Details Fetched via List Endpoint

**Location:** `zoho_meetings_polling.py:228–242` (ProcessZohoRecordingActivity)

The activity re-fetches ALL recordings via `provider.fetch_all_recordings()` to find a single recording by `meeting_key`. Comment at lines 229–230 explains: the single-recording endpoint returns incomplete data (missing meetingKey, startTimeinMs).

**Impact:** For orgs with many recordings, this fetches the entire list to find one recording. This is called once per recording, so N new recordings = N full list fetches.

**Recommendation:** This is a Zoho API limitation. Could cache the recording list from the poll phase and pass it to the processing activity, but this would require schema changes to the activity input. Current approach works but is inefficient at scale.

---

### LOW: `zsoid` Not Persisted After First Poll

**Location:** `zoho_meetings_polling.py:127–131`

When `ProcessZohoRecordingActivity` creates the `ZohoMeetingsProvider`, it passes `zsoid=integration.zsoid`. During the initial link, `zsoid` is fetched from the API and stored in the integration document. However, `PollZohoRecordingsForOrgActivity` doesn't update the stored `zsoid` if it was originally `None` — it relies on the provider's `get_user_details()` API call to fetch it each time.

**Impact:** Extra API call on first poll if zsoid wasn't stored during linking. After the first successful poll through `ProcessZohoRecordingActivity`, the zsoid is available in the integration. No functional issue, just a minor optimization opportunity.

---

## Things Done Correctly

### Infinite Polling with `continue_as_new`

The workflow correctly uses Temporal's `continue_as_new` pattern for infinite polling. This prevents workflow history from growing unbounded, which would eventually cause memory issues. Each poll cycle is a fresh workflow execution with clean history.

### Feature Flag Gating Per Org

The workflow checks `ZOHO_MEETINGS_INTEGRATION` feature flag **per org** (not globally). This allows incremental rollout — enabling Zoho Meetings for specific organizations before a full launch.

### Semaphore-Based Concurrency Control

The `asyncio.Semaphore(parallel_orgs)` correctly bounds the number of concurrent org polls. This prevents resource exhaustion when many orgs have the integration enabled.

### Error Isolation

One org's failure (auth error, API error, etc.) doesn't stop processing other orgs. Errors are captured in the workflow output and aggregated. This is production-critical for a background polling system.

### Deduplication via DAL

Recording deduplication uses the existing `upstream_meetings_dao.reader.find_by_remote_id_and_org()` DAL method. This is the correct approach — checking the database rather than maintaining in-memory state across poll cycles.

### Provider Abstraction

Zoho Meetings is the first provider to use the new `BaseUpstreamProvider` / `UpstreamProviderRegistry` abstraction. This is a clean foundation for migrating Gong, Chorus, and Fathom away from their legacy utility-function patterns.

### `should_ignore_recorder_prefs = True`

Correctly set for Zoho Meetings in `DoCrmMatchingActivity` (line 332–335). Zoho Meetings doesn't sync individual users as system identities, so there's no user-level opt-in/opt-out. All recordings should be processed without preference filtering.

### CRM Matching Correctly Skipped

`DoCrmMatchingActivity` (line 290) correctly skips CRM matching for `ZOHO_MEETINGS`. Without per-user workspace mapping, there's no reliable way to match recording owners to CRM users.

### Connection Health Integration

`ZohoMeetingsIntegrationInfoTeam` inherits `ConnectionHealthMixin`, and `PollZohoRecordingsForOrgActivity` checks `connection_health_error` before attempting API calls. `ZohoMeetingsAuthException` extends `ConnectionHealthException`, so auth failures are properly tracked.

### MetadataPointer Creation

`ProcessZohoRecordingActivity` correctly creates a `MetadataPointer` linking `meetingId` (meeting_key) to `upstreamMeetingId`. This is required for `CallProcessingWorkflow` to locate the upstream meeting during processing.

### Galactus Registry

All three activities and the workflow are properly registered in `src/sybill_py/runtimes/galactus/registry.py`:
- Workflow: `ZohoMeetingsPollingWorkflow` (line 725, `BACKGROUND_WORKFLOWS`)
- Activities: `FetchOrgsWithZohoMeetingsActivity`, `PollZohoRecordingsForOrgActivity`, `ProcessZohoRecordingActivity` (lines 750–752, `BACKGROUND_ACTIVITIES`)

---

## Summary Table

| Severity | Issue | Location | Status |
|----------|-------|----------|--------|
| **MEDIUM** | No rate limit handling in polling loop | `zoho_meetings_polling.py:134` | Open |
| **MEDIUM** | Full recording list re-fetched per recording | `zoho_meetings_polling.py:228` | Open (Zoho API limitation) |
| **LOW** | zsoid not persisted after first poll | `zoho_meetings_polling.py:127` | Open |
| — | ZohoMeetingsCredentialsManager: 0% direct test coverage | `credentials/zoho_meetings.py` | Open |
| — | Client `_handle_response` error paths: untested | `client.py:212` | Open |
| — | Feature flag disabled: untested in integration tests | `test_zoho_meetings_polling_integration.py` | Open |
