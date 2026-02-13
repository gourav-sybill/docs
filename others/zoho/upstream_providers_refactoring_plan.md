# Upstream Providers Refactoring Plan

This document outlines the plan to consolidate and standardize upstream meeting providers (Gong, Chorus, and future providers) using the `BaseUpstreamProvider` pattern established with Zoho Meetings.

---

## Executive Summary

The Zoho Meetings integration introduced a clean `BaseUpstreamProvider` abstraction that consolidates provider-specific logic into dedicated classes. Currently, Gong and Chorus have their logic scattered across multiple files. This refactoring will:

1. Create `GongProvider` and `ChorusProvider` classes implementing `BaseUpstreamProvider`
2. Consolidate scattered logic into single-responsibility classes
3. Enable registry-based provider lookup
4. Standardize testing patterns across all providers

---

## Current State

### Zoho Meetings (Reference Implementation)

**Architecture**: Clean, consolidated pattern

```
sybill_py/components/upstream_providers/
├── __init__.py                    # Public exports
├── base.py                        # BaseUpstreamProvider abstract class
├── registry.py                    # UpstreamProviderRegistry
└── zoho_meetings/
    ├── __init__.py
    ├── client.py                  # ZohoMeetingsClient (API client)
    ├── provider.py                # ZohoMeetingsProvider (implements BaseUpstreamProvider)
    ├── schema.py                  # ZohoRecording, ZohoParticipant, etc.
    └── exceptions.py              # Provider-specific exceptions
```

**Key Classes**:
- `ZohoMeetingsClient`: Handles API communication with Zoho
- `ZohoMeetingsProvider`: Implements `BaseUpstreamProvider`, handles metadata creation
- `ZohoRecording`, `ZohoParticipant`: Provider-specific schemas

**Test Coverage**: 171 tests across 8 files

---

### Gong (Current State - Scattered)

**Problem**: Logic spread across 5+ files with no dedicated provider class

| File | Gong-related Logic |
|------|-------------------|
| `src/common/upstream_meeting_utils.py` | Lines 210, 545-590 - Metadata creation for Gong |
| `src/routers/upstream_meetings.py` | Lines 343, 365, 431 - API handlers |
| `src/routers/helpers/upstream_meetings.py` | Lines 167, 181, 233 - Helper functions |
| `src/routers/org_utils.py` | Lines 579, 805 - Integration utilities |
| `src/dao/ci_reader_writer.py` | Line 23 - Database operations |

**Current Flow**:
```
API Request → upstream_meetings.py → upstream_meeting_utils.py → Database
                                   ↘ helpers/upstream_meetings.py ↗
```

**Issues**:
- No single source of truth for Gong logic
- Hard to test in isolation
- Difficult to understand full provider behavior
- Code duplication between providers

---

### Chorus (Current State - Scattered)

**Problem**: Same pattern as Gong - logic scattered across multiple files

| File | Chorus-related Logic |
|------|---------------------|
| `src/common/upstream_meeting_utils.py` | Lines 233, 555-638 - Metadata creation for Chorus |
| `src/routers/upstream_meetings.py` | Lines 345, 370, 433, 453 - API handlers |
| `src/routers/helpers/upstream_meetings.py` | Lines 184, 256 - Helper functions |
| `src/routers/org_utils.py` | Lines 588, 808 - Integration utilities |
| `src/dao/ci_reader_writer.py` | Line 27 - Database operations |

---

## Final State (Target Architecture)

### Directory Structure

```
sybill_py/components/upstream_providers/
├── __init__.py                    # Public exports
├── base.py                        # BaseUpstreamProvider (existing)
├── registry.py                    # UpstreamProviderRegistry (existing)
├── zoho_meetings/                 # ✅ Complete
│   ├── __init__.py
│   ├── client.py
│   ├── provider.py
│   ├── schema.py
│   └── exceptions.py
├── gong/                          # 🆕 New
│   ├── __init__.py
│   ├── client.py                  # GongClient (API communication)
│   ├── provider.py                # GongProvider (BaseUpstreamProvider impl)
│   ├── schema.py                  # GongRecording, GongParticipant
│   └── exceptions.py              # GongAPIError, etc.
└── chorus/                        # 🆕 New
    ├── __init__.py
    ├── client.py                  # ChorusClient
    ├── provider.py                # ChorusProvider (BaseUpstreamProvider impl)
    ├── schema.py                  # ChorusRecording, ChorusParticipant
    └── exceptions.py              # ChorusAPIError, etc.
```

### Test Structure

```
tests/sybill_py/components/upstream_providers/
├── zoho_meetings/                 # ✅ Complete (133 tests)
│   ├── test_client.py
│   ├── test_provider.py
│   ├── test_schema.py
│   └── test_exceptions.py
├── gong/                          # 🆕 New
│   ├── test_client.py
│   ├── test_provider.py
│   ├── test_schema.py
│   └── test_exceptions.py
└── chorus/                        # 🆕 New
    ├── test_client.py
    ├── test_provider.py
    ├── test_schema.py
    └── test_exceptions.py

tests/test_utils/
├── upstream_meetings.py           # 🆕 New - Shared test utilities
└── ...
```

---

## Implementation Plan

### Phase 1: Shared Test Utilities (1-2 hours)

**Goal**: Create reusable test utilities for upstream meeting testing

**Tasks**:
1. Create `tests/test_utils/upstream_meetings.py`
2. Extract `create_mock_upstream_meeting()` from existing tests
3. Add provider-agnostic fixtures for integration testing

**File to Create**: `tests/test_utils/upstream_meetings.py`

```python
"""Shared test utilities for upstream meeting providers."""

from uuid import UUID, uuid4
from common import utils
from schema.main import MeetingProviders
from schema.upstream_meeting_providers import UpstreamMeeting, UpstreamMeetingState, StateEvents


def create_mock_upstream_meeting(
    meeting_id: UUID | None = None,
    org_id: UUID | None = None,
    remote_id: str = "test-remote-id",
    provider: MeetingProviders = MeetingProviders.ZOHO_MEETINGS,
    with_state_events: bool = True,
) -> UpstreamMeeting:
    """Create a mock UpstreamMeeting for testing.

    Args:
        meeting_id: Optional UUID for the meeting
        org_id: Optional UUID for the organization
        remote_id: Remote ID from the provider
        provider: Meeting provider type
        with_state_events: Include default state events

    Returns:
        UpstreamMeeting instance ready for testing
    """
    meeting_id = meeting_id or uuid4()
    org_id = org_id or uuid4()
    now = utils.get_current_ts()

    state_events = []
    if with_state_events:
        state_event = UpstreamMeetingState(
            id=uuid4(),
            event=StateEvents.UPSTREAM_MEETING_RECEIVED,
            timestamp=now,
        )
        state_events = [state_event]

    return UpstreamMeeting(
        id=meeting_id,
        remote_id=remote_id,
        provider=provider,
        org_id=org_id,
        payload={},
        state_events=state_events,
        last_state_event=state_events[0] if state_events else None,
    )
```

---

### Phase 2: Gong Provider Implementation (4-6 hours)

**Goal**: Create `GongProvider` class and consolidate scattered Gong logic

#### 2.1 Create Gong Schemas

**File**: `src/sybill_py/components/upstream_providers/gong/schema.py`

Extract from existing code and create:
- `GongRecording` - Recording metadata from Gong API
- `GongParticipant` - Participant info from Gong
- `GongUserDetails` - User/org details

#### 2.2 Create Gong Client

**File**: `src/sybill_py/components/upstream_providers/gong/client.py`

Consolidate API calls from `upstream_meeting_utils.py`:
- `get_recordings()` - Fetch recordings list
- `get_recording(call_id)` - Fetch single recording
- `get_participants(call_id)` - Fetch call participants
- `download_recording(url)` - Download recording file

#### 2.3 Create Gong Provider

**File**: `src/sybill_py/components/upstream_providers/gong/provider.py`

```python
class GongProvider(BaseUpstreamProvider[GongRecording, GongParticipant]):
    """Gong upstream meeting provider implementation."""

    provider_type = MeetingProviders.GONG

    def __init__(self, access_token: SecretStr, org_id: UUID):
        self._client = GongClient(access_token=access_token)
        self._org_id = org_id

    async def fetch_all_recordings(self) -> list[GongRecording]:
        """Fetch all recordings from Gong API."""
        # Consolidate from upstream_meeting_utils.py
        ...

    async def fetch_recording(self, call_id: str) -> GongRecording | None:
        """Fetch specific recording."""
        ...

    async def fetch_participants(self, call_id: str) -> list[GongParticipant]:
        """Fetch participants for a call."""
        ...

    async def create_upstream_metadata(
        self,
        recording: GongRecording,
        participants: list[GongParticipant],
        remote_to_sybill_user_map: dict[str, UUID],
    ) -> UpstreamCallMetadata:
        """Transform Gong data to UpstreamCallMetadata.

        Consolidate logic from:
        - upstream_meeting_utils.py lines 545-590
        """
        ...

    def get_recording_download_url(self, recording: GongRecording) -> str | None:
        return recording.media_url

    def get_recording_id(self, recording: GongRecording) -> str:
        return recording.call_id

    def get_recording_timestamp(self, recording: GongRecording) -> int:
        return recording.started_at_ms
```

#### 2.4 Update Existing Code to Use GongProvider

**Files to Update**:
1. `src/routers/upstream_meetings.py` - Use `GongProvider` instead of inline logic
2. `src/common/upstream_meeting_utils.py` - Deprecate Gong-specific functions
3. `src/routers/helpers/upstream_meetings.py` - Delegate to provider

**Migration Strategy**:
```python
# Before (scattered)
if MeetingProviders.GONG == provider:
    metadata = create_gong_metadata(recording, participants)  # inline logic

# After (consolidated)
if MeetingProviders.GONG == provider:
    gong_provider = GongProvider(access_token=token, org_id=org_id)
    metadata = await gong_provider.create_upstream_metadata(recording, participants, user_map)
```

#### 2.5 Create Gong Tests

**Files to Create**:
- `tests/sybill_py/components/upstream_providers/gong/test_client.py`
- `tests/sybill_py/components/upstream_providers/gong/test_provider.py`
- `tests/sybill_py/components/upstream_providers/gong/test_schema.py`

---

### Phase 3: Chorus Provider Implementation (4-6 hours)

**Goal**: Create `ChorusProvider` class following same pattern as Gong

#### 3.1 Create Chorus Schemas

**File**: `src/sybill_py/components/upstream_providers/chorus/schema.py`

- `ChorusRecording`
- `ChorusParticipant`
- `ChorusMeetingDetails`

#### 3.2 Create Chorus Client

**File**: `src/sybill_py/components/upstream_providers/chorus/client.py`

#### 3.3 Create Chorus Provider

**File**: `src/sybill_py/components/upstream_providers/chorus/provider.py`

```python
class ChorusProvider(BaseUpstreamProvider[ChorusRecording, ChorusParticipant]):
    """Chorus upstream meeting provider implementation."""

    provider_type = MeetingProviders.CHORUS

    # Same pattern as GongProvider
    ...
```

#### 3.4 Update Existing Code

Same migration strategy as Gong.

#### 3.5 Create Chorus Tests

Same structure as Gong tests.

---

### Phase 4: Registry Integration (1-2 hours)

**Goal**: Register all providers with `UpstreamProviderRegistry`

**Update**: `src/sybill_py/components/upstream_providers/registry.py`

```python
from .gong.provider import GongProvider
from .chorus.provider import ChorusProvider
from .zoho_meetings.provider import ZohoMeetingsProvider

# Auto-register providers
UpstreamProviderRegistry.register(MeetingProviders.GONG)(GongProvider)
UpstreamProviderRegistry.register(MeetingProviders.CHORUS)(ChorusProvider)
UpstreamProviderRegistry.register(MeetingProviders.ZOHO_MEETINGS)(ZohoMeetingsProvider)
```

**Usage**:
```python
# Get provider by type
provider_class = UpstreamProviderRegistry.get(MeetingProviders.GONG)
provider = provider_class(access_token=token, org_id=org_id)
```

---

### Phase 5: Cleanup & Documentation (2-3 hours)

**Goal**: Remove deprecated code and update documentation

#### 5.1 Deprecate Old Functions

Add deprecation warnings to old functions in `upstream_meeting_utils.py`:

```python
import warnings

def create_gong_metadata(...):
    warnings.warn(
        "create_gong_metadata is deprecated. Use GongProvider.create_upstream_metadata instead.",
        DeprecationWarning,
        stacklevel=2,
    )
    # Keep implementation for backwards compatibility
    ...
```

#### 5.2 Update Documentation

- Update `CLAUDE.md` with new provider pattern
- Add docstrings to all new classes
- Create migration guide for existing code

---

## Estimated Timeline

| Phase | Task | Estimated Time |
|-------|------|----------------|
| 1 | Shared Test Utilities | 1-2 hours |
| 2 | Gong Provider Implementation | 4-6 hours |
| 3 | Chorus Provider Implementation | 4-6 hours |
| 4 | Registry Integration | 1-2 hours |
| 5 | Cleanup & Documentation | 2-3 hours |
| **Total** | | **12-19 hours** |

---

## Success Criteria

1. **All providers implement `BaseUpstreamProvider`**
   - `ZohoMeetingsProvider` ✅
   - `GongProvider` 🆕
   - `ChorusProvider` 🆕

2. **Test coverage maintained or improved**
   - Unit tests for each provider component
   - Integration tests following Zoho Meetings pattern

3. **No regression in existing functionality**
   - All existing tests pass
   - API behavior unchanged

4. **Code consolidation achieved**
   - Gong logic in single provider class
   - Chorus logic in single provider class
   - Scattered code deprecated

5. **Registry enables dynamic provider lookup**
   - Can instantiate provider by `MeetingProviders` enum
   - Easy to add new providers

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Breaking existing Gong/Chorus functionality | High | Extensive testing, gradual migration with deprecation warnings |
| Missing edge cases in consolidation | Medium | Review all existing code paths before consolidation |
| Test coverage gaps | Medium | Maintain test parity with existing coverage |
| Performance regression | Low | Profile before/after, no architectural changes to hot paths |

---

## Future Providers

Once this refactoring is complete, adding new providers (e.g., Fathom, Fireflies) becomes straightforward:

1. Create `provider_name/` directory under `upstream_providers/`
2. Implement `schema.py`, `client.py`, `provider.py`, `exceptions.py`
3. Register with `UpstreamProviderRegistry`
4. Add tests following established patterns

Estimated time for new provider: **2-4 hours** (vs current ~8+ hours)
