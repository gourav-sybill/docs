# Zoho Meetings Integration Plan

## Executive Summary

This document analyzes two approaches for integrating Zoho Meetings: using **Unified.to** as an intermediary vs building a **Direct Client**. Based on the codebase analysis and requirements, we recommend a **hybrid approach** with a generic upstream provider architecture that can be extended to support future integrations (WebEx, Microsoft Teams native recordings, etc.).

---

## Table of Contents

1. [Background](#background)
   - [Why Org-Level (Team) Integration?](#why-org-level-team-integration)
2. [Unified.to Analysis](#unifiedto-analysis)
3. [Direct Client Analysis](#direct-client-analysis)
4. [Comparison Matrix](#comparison-matrix)
5. [Recommendation](#recommendation)
6. [Generic Upstream Provider Architecture](#generic-upstream-provider-architecture)
   - [Provider Interface](#provider-interface)
   - [Provider Registration Pattern](#provider-registration-pattern)
   - [Zoho Meetings Provider Implementation](#zoho-meetings-provider-implementation)
   - [Zoho Meetings Field Mappings](#zoho-meetings-field-mappings)
   - [Zoho Meetings Schemas](#zoho-meetings-schemas)
   - [Zoho Meetings Exceptions](#zoho-meetings-exceptions)
   - [API Client](#api-client-following-zoho-calendar-pattern)
   - [Polling Mechanism](#polling-mechanism-primary-approach)
   - [Supporting Activities](#supporting-activities)
   - [Integration Metadata Schema](#integration-metadata-schema)
   - [Frontend OAuth Flow](#frontend-oauth-flow)
   - [Temporal Schedule Setup](#temporal-schedule-setup)
   - [Error Handling & Recovery](#error-handling--recovery)
   - [Webhook Handler (Fallback)](#webhook-handler-fallback-if-available)
7. [Implementation Plan](#implementation-plan)
8. [Risk Assessment](#risk-assessment)
9. [Testing Strategy](#testing-strategy)
10. [Future Extensibility](#future-extensibility)
11. [Appendix](#appendix)
    - [A. Zoho Meetings API Reference (Validated)](#a-zoho-meetings-api-reference-validated)
    - [B. Existing Field Mappings Reference](#b-existing-field-mappings-reference)
    - [C. Related Files](#c-related-files)

---

## Background

### Current State
- **Bot provider (Recall)** does not support Zoho Meetings
- Zoho Meetings provides **native recordings** that can be pulled via API
- Current upstream meeting support: Gong, Chorus, Fathom, SDK Upload, Transcript Upload
- Zoho Calendar/CRM/Mail integrations already exist using direct clients
- Email integration uses Unified.to for Gmail/Outlook/Zoho Mail

### Requirements
- Pull meeting recordings from Zoho Meetings
- Create upstream meetings and trigger existing call processing pipeline
- Support OAuth at org-level (team integration, not user-level)
- Handle metadata: participants, invitees, timestamps, recording URL
- Support Zoho's multi-region architecture (US, EU, IN, AU, etc.)

### Why Org-Level (Team) Integration?

Zoho Meetings is a **team-level integration** (like Zoho CRM), not a user-level integration (like Zoho Calendar/Email):

| Aspect | User-Level (Calendar/Email) | Org-Level (Meetings/CRM) |
|--------|----------------------------|--------------------------|
| **Who connects** | Each individual user | One org admin |
| **Data access** | User's own data only | All org data |
| **Auth0 storage** | User's `app_metadata` | Org admin user linked to `sybill-admin-{org_id}` |
| **Example** | Zoho Calendar, Gmail | Zoho CRM, Zoho Meetings, Salesforce |

**Rationale:**
1. **Recordings belong to the org** - Zoho Meetings recordings are org-level resources, not tied to individual users
2. **Single admin connection** - One person (Zoho Meetings admin) connects, all org recordings become accessible
3. **Different admins possible** - The Zoho Meetings admin may be different from the Zoho CRM admin (different products)
4. **Consistent with CRM pattern** - Matches existing org-level integration patterns in the codebase

---

## Unified.to Analysis

### Capabilities
Based on codebase exploration (`sybill_py/components/unified_to/`):

| Feature | Status |
|---------|--------|
| Zoho Meeting in catalog | ✅ Available (under Calendar integrations) |
| Recording media field | ✅ Available |
| Webhook support | ✅ Available for created/updated events |
| OAuth handling | ✅ Managed by Unified.to |
| SDK available | ✅ `unified_python_sdk` |

### Current Unified.to Usage Pattern

```
Frontend → Auth0 → Unified.to OAuth → Backend Link Account API
                                    ↓
                            Create Unified Connection
                                    ↓
                            Set up Webhooks
                                    ↓
              Webhook Event → Temporal Workflow → Process Data
```

**Key files:**
- Connection: `sybill_py/components/unified_to/connection.py`
- Webhooks: `sybill_py/components/unified_to/webhook.py`
- Messaging: `sybill_py/components/messaging/reader_writer.py`

### Advantages
| Advantage | Impact |
|-----------|--------|
| Pre-built webhook infrastructure | Reduces development time |
| OAuth token refresh handled | No custom token management |
| Unified SDK abstractions | Consistent API patterns |
| Future provider support | May support more recording providers |

### Disadvantages
| Disadvantage | Impact | Severity |
|--------------|--------|----------|
| Early-stage company | Bugs may require coordination to fix | HIGH |
| No upstream recording precedent | Untested for this use case | MEDIUM |
| Limited Zoho Meetings docs | May have incomplete implementation | MEDIUM |
| Additional dependency | External service dependency | LOW |
| Unknown data mapping | Recording metadata structure unclear | MEDIUM |

### Unified.to Cost Factors
- API calls to Unified.to
- Webhook event processing
- Connection maintenance overhead

---

## Direct Client Analysis

### Existing Direct Client Patterns

The codebase has well-established patterns for direct API integrations:

**1. Zoho Calendar Client** (`common/zoho_calendar_utils.py`):
```python
class ZohoCalendarClient(SybillAsyncClient):
    def __init__(self, access_token: SecretStr, dc_region: ZohoDCRegions):
        base_url = get_zoho_calendar_base_url(dc_region)
        super().__init__(
            base_url=f"{base_url}/api/v1",
            headers={"Authorization": f"Zoho-oauthtoken {access_token.get_secret_value()}"},
        )
```

**2. CRM Clients** (`common/crm_network_utils.py`):
- `ZohoClient`: Full Zoho CRM integration
- `HubspotClient`: HubSpot with SDK wrapper
- `SalesforceClient`: Multi-region support

**3. Video Conference Clients** (`common/vc_network_utils.py`):
- `ZoomClient`: Meeting info, user management

### Zoho Meetings API (Validated)

See [Appendix A](#a-zoho-meetings-api-reference-validated) for complete API documentation.

**Key Endpoints:**
1. `GET /api/v2/user.json` - Get zsoid (org ID, required for all other calls)
2. `GET /meeting/api/v2/{zsoid}/recordings.json` - List all recordings
3. `GET /api/v2/{zsoid}/sessions/{meetingKey}.json` - Get meeting details
4. `GET /api/v2/{zsoid}/participant/{meetingKey}.json` - Get participant report

**Key Features:**
- Pre-signed download URLs for recordings
- Native transcript available if `isTranscriptGenerated: true`
- Native AI summary available if `isSummaryGenerated: true`

**Authentication:**
- OAuth 2.0 with refresh tokens
- Same flow as Zoho Calendar/CRM
- Multi-region support (8 regions)
- Required scopes: `ZohoMeeting.manageOrg.READ,ZohoMeeting.meeting.READ,ZohoMeeting.recording.READ,ZohoMeeting.meetinguds.READ,ZohoFiles.files.READ`

### Advantages
| Advantage | Impact |
|-----------|--------|
| Full control over implementation | Can optimize for specific needs |
| Consistent with Zoho Calendar/CRM | Reuse existing patterns |
| No external dependency | Self-contained |
| Direct debugging | Faster issue resolution |
| Multi-region already solved | Existing `ZohoDCRegions` enum |

### Disadvantages
| Disadvantage | Impact | Severity |
|--------------|--------|----------|
| Zoho API documentation quality | "Wild west" - may be inconsistent | HIGH |
| Webhook infrastructure needed | Must build polling or webhook handler | MEDIUM |
| OAuth connection setup | Need new Auth0 connection | LOW (pattern exists) |
| Maintenance burden | Own the code | LOW |

---

## Comparison Matrix

| Criteria | Unified.to | Direct Client | Winner |
|----------|------------|---------------|--------|
| **Development Speed (POC)** | Fast | Medium | Unified |
| **Development Speed (Production)** | Medium | Medium | Tie |
| **Reliability** | Unknown | Known patterns | Direct |
| **Debugging** | Limited | Full control | Direct |
| **Multi-region Support** | Unknown | Proven pattern | Direct |
| **Future Extensibility** | Limited to Unified catalog | Full flexibility | Direct |
| **Maintenance** | Shared with Unified | Self-maintained | Unified |
| **Zoho Ecosystem Fit** | Separate from CRM/Calendar | Consistent | Direct |
| **Timeline Risk** | HIGH (dependency on 3rd party) | LOW | Direct |
| **Cost** | API usage fees | None | Direct |

**Weighted Score:**
- Unified.to: 5/10
- Direct Client: 7/10

---

## Recommendation

### Primary Recommendation: **Direct Client with Generic Architecture**

**Rationale:**
1. **Timeline urgency** - Direct client avoids dependency on external bug fixes
2. **Existing patterns** - Zoho Calendar/CRM clients provide proven templates
3. **Consistency** - Same auth pattern (Auth0 + refresh tokens) as other Zoho integrations
4. **Extensibility** - Generic architecture supports future providers (WebEx, Teams, etc.)
5. **Known risks** - Zoho API quirks are manageable vs unknown Unified.to behavior

### Secondary Option: Quick POC with Unified.to

If timeline is extremely tight, start with Unified.to POC:
1. Verify Zoho Meetings data structure in Unified.to
2. Test webhook reliability
3. Validate recording URL accessibility
4. Fall back to direct client if issues arise

---

## Generic Upstream Provider Architecture

### Design Goals
1. **Provider-agnostic** - Support any recording provider (Zoho, WebEx, Teams, etc.)
2. **Pluggable** - Add new providers with minimal code changes
3. **Consistent** - Same processing pipeline for all providers
4. **Testable** - Easy to mock and test individual components

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           UPSTREAM PROVIDER LAYER                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  Zoho Meetings  │  │     WebEx       │  │  Teams Native   │  ...        │
│  │    Provider     │  │    Provider     │  │    Provider     │             │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘             │
│           │                    │                    │                       │
│           └────────────────────┼────────────────────┘                       │
│                                │                                            │
│                    ┌───────────▼───────────┐                               │
│                    │  BaseUpstreamProvider │  (Abstract Interface)         │
│                    └───────────┬───────────┘                               │
│                                │                                            │
└────────────────────────────────┼────────────────────────────────────────────┘
                                 │
┌────────────────────────────────┼────────────────────────────────────────────┐
│                                ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    UpstreamProviderManager                          │   │
│  │  - register_provider(provider_type, provider_class)                 │   │
│  │  - get_provider(provider_type) -> BaseUpstreamProvider              │   │
│  │  - process_webhook(provider_type, payload) -> UpstreamMeeting       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                │                                            │
│                    ORCHESTRATION LAYER                                      │
└────────────────────────────────┼────────────────────────────────────────────┘
                                 │
┌────────────────────────────────┼────────────────────────────────────────────┐
│                                ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │               Existing Upstream Processing Pipeline                  │   │
│  │                                                                      │   │
│  │  UpstreamMeeting → UpstreamProcessingWorkflow → CallProcessing      │   │
│  │       ↓                      ↓                        ↓              │   │
│  │  MongoDB          CreateParticipants            Transcription        │   │
│  │                   DoCrmMatching                 ML Processing        │   │
│  │                   StartCallProcessing           Insights             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│                    PROCESSING LAYER (Existing)                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Provider Interface

```python
# Location: sybill_py/components/upstream_providers/base.py

from abc import ABC, abstractmethod
from typing import TypeVar, Generic, Any
from uuid import UUID

from pydantic import SecretStr
from schema.main import MeetingProviders, MeetingTypes
from schema.upstream_meeting_providers import (
    UpstreamMeeting,
    UpstreamCallMetadata,
    UpstreamInvitee,
    UpstreamParticipant,
)

RecordingT = TypeVar("RecordingT")  # Provider-specific recording type
ParticipantT = TypeVar("ParticipantT")  # Provider-specific participant type


class BaseUpstreamProvider(ABC, Generic[RecordingT, ParticipantT]):
    """Abstract base class for upstream meeting providers.

    Each provider implementation handles:
    - Fetching recordings from the provider API
    - Fetching participant information
    - Transforming provider data into Sybill's UpstreamMeeting format
    - Optional webhook signature validation

    Generic types:
    - RecordingT: Provider-specific recording schema (e.g., ZohoRecording)
    - ParticipantT: Provider-specific participant schema (e.g., ZohoParticipant)
    """

    provider_type: MeetingProviders  # e.g., MeetingProviders.ZOHO_MEETINGS

    # ─────────────────────────────────────────────────────────────────
    # Abstract methods - must be implemented by each provider
    # ─────────────────────────────────────────────────────────────────

    @abstractmethod
    async def fetch_all_recordings(self) -> list[RecordingT]:
        """Fetch all available recordings for polling.

        Returns list of provider-specific recording objects.
        """
        pass

    @abstractmethod
    async def fetch_recording(self, meeting_id: str) -> RecordingT | None:
        """Fetch a specific recording by ID."""
        pass

    @abstractmethod
    async def fetch_participants(self, meeting_id: str) -> list[ParticipantT]:
        """Fetch participants for a meeting."""
        pass

    @abstractmethod
    async def create_upstream_metadata(
        self,
        recording: RecordingT,
        participants: list[ParticipantT],
        remote_to_sybill_user_map: dict[str, UUID],
    ) -> UpstreamCallMetadata:
        """Transform provider data into standard UpstreamCallMetadata.

        Args:
            recording: Provider-specific recording object
            participants: List of provider-specific participant objects
            remote_to_sybill_user_map: Map of remote email -> Sybill user ID

        Returns:
            UpstreamCallMetadata ready for UpstreamMeeting creation
        """
        pass

    @abstractmethod
    def get_recording_download_url(self, recording: RecordingT) -> str | None:
        """Extract download URL from recording object."""
        pass

    @abstractmethod
    def get_recording_id(self, recording: RecordingT) -> str:
        """Extract unique identifier from recording (used for deduplication)."""
        pass

    @abstractmethod
    def get_recording_timestamp(self, recording: RecordingT) -> int:
        """Extract start timestamp (epoch ms) from recording (used for ordering)."""
        pass

    # ─────────────────────────────────────────────────────────────────
    # Optional methods - override if provider supports feature
    # ─────────────────────────────────────────────────────────────────

    async def fetch_meeting_details(self, meeting_id: str) -> dict | None:
        """Fetch additional meeting details (optional).

        Some providers have separate meeting info from recording info.
        """
        return None

    def get_transcript_download_url(self, recording: RecordingT) -> str | None:
        """Extract native transcript URL if available."""
        return None

    def has_native_transcript(self, recording: RecordingT) -> bool:
        """Check if provider generated a transcript."""
        return False

    async def validate_webhook_signature(
        self,
        payload: bytes,
        signature: str,
        secret: str,
    ) -> bool:
        """Validate webhook signature (if webhooks supported).

        Default implementation returns False (webhooks not supported).
        """
        return False

    def get_dashboard_url(self, recording: RecordingT) -> str | None:
        """Get URL to view recording in provider's dashboard."""
        return None
```

### Provider Registration Pattern

```python
# Location: sybill_py/components/upstream_providers/registry.py

from typing import Type
from schema.main import MeetingProviders

class UpstreamProviderRegistry:
    """Registry for upstream meeting providers."""

    _providers: dict[MeetingProviders, Type[BaseUpstreamProvider]] = {}

    @classmethod
    def register(cls, provider_type: MeetingProviders):
        """Decorator to register a provider class."""
        def decorator(provider_class: Type[BaseUpstreamProvider]):
            cls._providers[provider_type] = provider_class
            return provider_class
        return decorator

    @classmethod
    def get_provider(
        cls,
        provider_type: MeetingProviders,
        **kwargs,
    ) -> BaseUpstreamProvider:
        """Get an instance of the provider."""
        if provider_type not in cls._providers:
            raise ValueError(f"Unknown provider: {provider_type}")
        return cls._providers[provider_type](**kwargs)

    @classmethod
    def supported_providers(cls) -> list[MeetingProviders]:
        """List all registered providers."""
        return list(cls._providers.keys())
```

### Zoho Meetings Provider Implementation

```python
# Location: sybill_py/components/upstream_providers/zoho_meetings/provider.py

from pydantic import SecretStr
from sybill_py.components.upstream_providers.base import BaseUpstreamProvider
from sybill_py.components.upstream_providers.registry import UpstreamProviderRegistry
from sybill_py.components.upstream_providers.zoho_meetings.client import ZohoMeetingsClient
from sybill_py.components.upstream_providers.zoho_meetings.schema import (
    ZohoMeetingPayload,
    ZohoRecording,
    ZohoParticipant,
)
from schema.main import MeetingProviders, ConferenceProviders, ZohoDCRegions
from schema.upstream_meeting_providers import (
    UpstreamCallMetadata,
    UpstreamInvitee,
    UpstreamParticipant,
)

# Field mappings: Zoho field -> Sybill field
ZOHO_MEETINGS_FIELD_MAPPINGS = {
    "meetingKey": "remote_id",
    "email": "email",
    "memberId": "remote_id",
}


@UpstreamProviderRegistry.register(MeetingProviders.ZOHO_MEETINGS)
class ZohoMeetingsProvider(BaseUpstreamProvider[ZohoRecording, ZohoParticipant]):
    """Zoho Meetings upstream provider implementation.

    Handles:
    - Polling for new recordings via GET /meeting/api/v2/{zsoid}/recordings.json
    - Fetching meeting details and participants
    - Downloading recordings and transcripts (if available)
    - Creating UpstreamMeeting objects for processing
    """

    provider_type = MeetingProviders.ZOHO_MEETINGS

    def __init__(self, dc_region: ZohoDCRegions, access_token: SecretStr):
        self.dc_region = dc_region
        self.access_token = access_token

    # ─────────────────────────────────────────────────────────────────
    # Required interface implementations
    # ─────────────────────────────────────────────────────────────────

    async def fetch_all_recordings(self) -> list[ZohoRecording]:
        """Fetch all recordings for the organization (for polling)."""
        async with ZohoMeetingsClient(self.access_token, self.dc_region) as client:
            recordings_data = await client.get_all_recordings()
            return [ZohoRecording.model_validate(r) for r in recordings_data]

    async def fetch_recording(self, meeting_key: str) -> ZohoRecording | None:
        """Fetch recording metadata including download URLs."""
        async with ZohoMeetingsClient(self.access_token, self.dc_region) as client:
            recording_data = await client.get_recording(meeting_key)
            if recording_data:
                return ZohoRecording.model_validate(recording_data)
            return None

    async def fetch_participants(self, meeting_key: str) -> list[ZohoParticipant]:
        """Fetch all participants for a meeting."""
        async with ZohoMeetingsClient(self.access_token, self.dc_region) as client:
            participants_data = await client.get_all_participants(meeting_key)
            return [ZohoParticipant.model_validate(p) for p in participants_data]

    async def create_upstream_metadata(
        self,
        recording: ZohoRecording,
        participants: list[ZohoParticipant],
        remote_to_sybill_user_map: dict[str, UUID],
    ) -> UpstreamCallMetadata:
        """Transform Zoho data into standard UpstreamCallMetadata."""
        invitees = self._create_invitees(participants, remote_to_sybill_user_map)
        upstream_participants, unresolved = self._create_participants(
            participants, remote_to_sybill_user_map
        )

        # Determine remote owner (presenter)
        presenter = next((p for p in participants if p.role == "presenter"), None)
        remote_owner_id = presenter.member_id if presenter else ""

        return UpstreamCallMetadata(
            topic=recording.topic or "Untitled Meeting",
            real_start_time=recording.start_time_in_ms,
            real_end_time=recording.start_time_in_ms + recording.duration,
            provider=MeetingProviders.ZOHO_MEETINGS,
            remote_owner_id=remote_owner_id,
            meeting_type=self._determine_meeting_type(invitees),
            dashboard_url=self.get_dashboard_url(recording),
            invitees=invitees,
            participants=upstream_participants,
            conference_provider=ConferenceProviders.ZOHO,
            unresolved_speaker_ids=unresolved,
        )

    def get_recording_download_url(self, recording: ZohoRecording) -> str | None:
        """Extract download URL from recording."""
        return recording.download_url

    def get_recording_id(self, recording: ZohoRecording) -> str:
        """Extract meeting key as unique identifier."""
        return recording.meeting_key

    def get_recording_timestamp(self, recording: ZohoRecording) -> int:
        """Extract start timestamp in epoch milliseconds."""
        return recording.start_time_in_ms

    # ─────────────────────────────────────────────────────────────────
    # Optional interface implementations
    # ─────────────────────────────────────────────────────────────────

    async def fetch_meeting_details(self, meeting_key: str) -> dict | None:
        """Fetch full meeting details from Zoho API."""
        async with ZohoMeetingsClient(self.access_token, self.dc_region) as client:
            return await client.get_meeting(meeting_key)

    def get_transcript_download_url(self, recording: ZohoRecording) -> str | None:
        """Extract native transcript URL if available."""
        if recording.is_transcript_generated:
            return recording.transcription_download_url
        return None

    def has_native_transcript(self, recording: ZohoRecording) -> bool:
        """Check if Zoho generated a transcript."""
        return recording.is_transcript_generated

    def get_dashboard_url(self, recording: ZohoRecording) -> str | None:
        """Get URL to view meeting in Zoho - requires fetching meeting details."""
        # Note: joinLink comes from meeting details, not recording
        # Return None here; actual URL fetched via fetch_meeting_details()
        return None

    async def validate_webhook_signature(
        self,
        payload: bytes,
        signature: str,
        secret: str,
    ) -> bool:
        """Validate Zoho webhook signature (if webhooks are available)."""
        import hmac
        import hashlib

        expected = hmac.new(
            secret.encode(),
            payload,
            hashlib.sha256,
        ).hexdigest()
        return hmac.compare_digest(signature, expected)

    def _create_invitees(
        self,
        participants: list[ZohoParticipant],
        remote_to_sybill_user_map: dict[str, UUID],
    ) -> list[UpstreamInvitee]:
        """Create invitees from participant list.

        Note: Zoho doesn't have a separate invitee concept for recordings.
        We derive invitees from participants.
        """
        invitees = []
        for p in participants:
            if not p.email:
                continue

            sybill_user_id = remote_to_sybill_user_map.get(p.email)
            invitees.append(
                UpstreamInvitee(
                    remote_id=p.member_id,
                    email=p.email,
                    name=p.email.split("@")[0],  # Best effort name extraction
                    is_internal=sybill_user_id is not None,
                    invitee_user_id=sybill_user_id,
                    is_sybill_user=sybill_user_id is not None,
                )
            )
        return invitees

    def _create_participants(
        self,
        participants: list[ZohoParticipant],
        remote_to_sybill_user_map: dict[str, UUID],
    ) -> tuple[list[UpstreamParticipant], list[str]]:
        """Create participants list and track unresolved speaker IDs."""
        upstream_participants = []
        unresolved_speaker_ids = []

        for i, p in enumerate(participants):
            sybill_user_id = remote_to_sybill_user_map.get(p.email) if p.email else None

            # If we can't resolve the participant name, mark as unresolved
            # This will trigger LLM speaker inference later
            name = p.email.split("@")[0] if p.email else None
            if not name:
                name = f"Speaker #{i + 1}"
                unresolved_speaker_ids.append(p.member_id)

            upstream_participants.append(
                UpstreamParticipant(
                    remote_id=p.member_id,
                    speaker_id=p.member_id,  # Used for transcript speaker matching
                    email=p.email,
                    name=name,
                    is_internal=sybill_user_id is not None,
                    invitee_user_id=sybill_user_id,
                    is_sybill_user=sybill_user_id is not None,
                )
            )

        return upstream_participants, unresolved_speaker_ids

    def _determine_meeting_type(self, invitees: list[UpstreamInvitee]) -> MeetingTypes:
        """Determine if meeting is internal or external based on participants.

        A meeting is EXTERNAL if any participant is not a Sybill user (external).
        Otherwise, it's INTERNAL.
        """
        from schema.main import MeetingTypes

        external_count = sum(1 for i in invitees if not i.is_internal)
        if external_count > 0:
            return MeetingTypes.EXTERNAL
        return MeetingTypes.INTERNAL
```

### Zoho Meetings Field Mappings

```python
# Location: sybill_py/components/upstream_providers/zoho_meetings/field_mappings.py

# Field mappings: Zoho API field -> Sybill schema field
ZOHO_MEETINGS_FIELD_MAPPINGS = {
    # Recording fields
    "meetingKey": "remote_id",
    "erecordingId": "erecording_id",
    "downloadUrl": "download_url",
    "transcriptionDownloadUrl": "transcription_download_url",
    "startTimeinMs": "start_time_in_ms",
    "isTranscriptGenerated": "is_transcript_generated",

    # Participant fields
    "memberId": "remote_id",
    "email": "email",
    "joinTime": "join_time",
    "leaveTime": "leave_time",

    # Meeting session fields
    "presenterEmail": "presenter_email",
    "presenterName": "presenter_name",
    "joinLink": "dashboard_url",
}

# Status values
RECORDING_STATUS_READY = "UPLOADED"  # Recording is ready for download
RECORDING_STATUS_PROCESSING = "PROCESSING"  # Still being processed by Zoho
```

### Zoho Meetings Schemas

```python
# Location: sybill_py/components/upstream_providers/zoho_meetings/schema.py

from pydantic import Field
from schema.main import SybillBaseModel


class ZohoRecording(SybillBaseModel):
    """Schema for Zoho recording data from API."""

    erecording_id: str = Field(alias="erecordingId")
    meeting_key: str = Field(alias="meetingKey")
    topic: str | None = None
    download_url: str | None = Field(None, alias="downloadUrl")
    transcription_download_url: str | None = Field(None, alias="transcriptionDownloadUrl")
    summary_download_url: str | None = Field(None, alias="summaryDownloadUrl")
    is_transcript_generated: bool = Field(False, alias="isTranscriptGenerated")
    is_summary_generated: bool = Field(False, alias="isSummaryGenerated")
    duration: int = 0  # Duration in milliseconds
    duration_in_mins: int = Field(0, alias="durationInMins")
    start_time_in_ms: int = Field(alias="startTimeinMs")
    file_size: str | None = Field(None, alias="fileSize")
    status: str = "UNKNOWN"  # e.g., "UPLOADED"


class ZohoParticipant(SybillBaseModel):
    """Schema for Zoho participant data from API."""

    email: str | None = None
    role: str = "participant"  # "presenter" or "participant"
    join_time: int = Field(alias="joinTime")  # Epoch milliseconds
    leave_time: int = Field(alias="leaveTime")  # Epoch milliseconds
    duration: int = 0  # Duration in milliseconds
    in_and_out_time: str | None = Field(None, alias="inAndOutTime")
    source: str | None = None  # "web", "mobile", etc.
    participant_avatar: str | None = Field(None, alias="participantAvatar")
    member_id: str = Field(alias="memberId")


class ZohoMeetingSession(SybillBaseModel):
    """Schema for Zoho meeting session details."""

    meeting_key: int = Field(alias="meetingKey")
    topic: str | None = None
    agenda: str | None = None
    presenter_email: str | None = Field(None, alias="presenterEmail")
    presenter_name: str | None = Field(None, alias="presenterName")
    start_time: str | None = Field(None, alias="startTime")  # Formatted string
    end_time: str | None = Field(None, alias="endTime")  # Formatted string
    duration: int = 0  # Duration in milliseconds
    timezone: str | None = None
    join_link: str | None = Field(None, alias="joinLink")


class ZohoMeetingPayload(SybillBaseModel):
    """Schema for webhook payload (if webhooks are supported)."""

    event_type: str | None = Field(None, alias="eventType")
    meeting_key: str | None = Field(None, alias="meetingKey")
    recording: ZohoRecording | None = None
    session: ZohoMeetingSession | None = None


class ZohoUserDetails(SybillBaseModel):
    """Schema for user details response."""

    zsoid: int  # Organization ID - required for all other APIs
    zuid: int  # User ID
    primary_email: str = Field(alias="primaryEmail")
    display_name: str | None = Field(None, alias="displayName")
    is_admin: bool = Field(False, alias="isAdmin")
```

### Zoho Meetings Exceptions

```python
# Location: sybill_py/components/upstream_providers/zoho_meetings/exceptions.py

from sybill_py.components.integration.exception import ConnectionHealthException


class ZohoMeetingsAPIError(Exception):
    """Base exception for Zoho Meetings API errors."""
    pass


class ZohoMeetingsAuthException(ConnectionHealthException):
    """Raised when authentication fails (401/403).

    This indicates the token may be invalid or expired.
    Should trigger token refresh or re-authentication.
    """
    pass


class ZohoMeetingsRateLimitException(ZohoMeetingsAPIError):
    """Raised when rate limit is exceeded (429).

    Caller should implement exponential backoff.
    """
    pass


class ZohoMeetingsNotFoundError(ZohoMeetingsAPIError):
    """Raised when a resource is not found (404)."""
    pass


class ZohoMeetingsPermissionError(ZohoMeetingsAPIError):
    """Raised when user lacks permission for an operation."""
    pass
```

---

### API Client (Following Zoho Calendar Pattern)

```python
# Location: sybill_py/components/upstream_providers/zoho_meetings/client.py

from pydantic import SecretStr
from common.network_utils import SybillAsyncClient
from schema.main import ZohoDCRegions

class ZohoMeetingsClient(SybillAsyncClient):
    """Async HTTP client for Zoho Meetings API v2.

    All meeting/recording endpoints require zsoid (organization ID).
    Call get_user_details() first to obtain the zsoid.
    """

    BASE_URLS = {
        ZohoDCRegions.US: "https://meeting.zoho.com",
        ZohoDCRegions.EU: "https://meeting.zoho.eu",
        ZohoDCRegions.IN: "https://meeting.zoho.in",
        ZohoDCRegions.AU: "https://meeting.zoho.com.au",
        ZohoDCRegions.UK: "https://meeting.zoho.uk",
        ZohoDCRegions.CA: "https://meeting.zoho.ca",
        ZohoDCRegions.JP: "https://meeting.zoho.jp",
        ZohoDCRegions.SA: "https://meeting.zoho.sa",
    }

    def __init__(
        self,
        access_token: SecretStr,
        dc_region: ZohoDCRegions = ZohoDCRegions.US,
    ):
        base_url = self.BASE_URLS.get(dc_region, self.BASE_URLS[ZohoDCRegions.US])
        super().__init__(
            base_url=base_url,
            headers={
                "Authorization": f"Zoho-oauthtoken {access_token.get_secret_value()}",
                "Accept": "application/json",
            },
        )
        self.dc_region = dc_region
        self._zsoid: int | None = None

    # ─────────────────────────────────────────────────────────────────
    # User/Organization APIs
    # ─────────────────────────────────────────────────────────────────

    async def get_user_details(self) -> dict:
        """Get user details including zsoid (organization ID).

        Returns:
            {
                "userDetails": {
                    "zsoid": 53758574,  # Required for all other APIs
                    "zuid": 53758790,
                    "primaryEmail": "user@company.com",
                    "displayName": "User Name",
                    "isAdmin": true
                }
            }
        """
        response = await self.get("/api/v2/user.json")
        self._handle_response(response, "get_user_details")
        data = response.json()
        # Cache zsoid for subsequent calls
        self._zsoid = data.get("userDetails", {}).get("zsoid")
        return data

    async def _ensure_zsoid(self) -> int:
        """Ensure zsoid is available, fetching if necessary."""
        if self._zsoid is None:
            await self.get_user_details()
        if self._zsoid is None:
            raise ZohoMeetingsAPIError("Could not obtain zsoid from user details")
        return self._zsoid

    # ─────────────────────────────────────────────────────────────────
    # Recording APIs (Core for Integration)
    # ─────────────────────────────────────────────────────────────────

    async def get_all_recordings(self) -> list[dict]:
        """Get all recordings for the organization.

        Returns list of recordings with:
            - erecordingId: Unique recording ID
            - meetingKey: Meeting identifier
            - topic: Meeting topic
            - downloadUrl: Pre-signed URL for video download
            - transcriptionDownloadUrl: URL for transcript (if available)
            - summaryDownloadUrl: URL for AI summary (if available)
            - isTranscriptGenerated: bool
            - isSummaryGenerated: bool
            - duration: Duration in milliseconds
            - startTimeinMs: Start time in epoch milliseconds
            - status: Recording status (e.g., "UPLOADED")
        """
        zsoid = await self._ensure_zsoid()
        response = await self.get(f"/meeting/api/v2/{zsoid}/recordings.json")
        self._handle_response(response, "get_all_recordings")
        return response.json().get("recordings", [])

    async def get_recording(self, meeting_key: str) -> dict | None:
        """Get specific recording by meeting key.

        Args:
            meeting_key: The meetingKey from recording list

        Returns:
            Recording dict or None if not found
        """
        zsoid = await self._ensure_zsoid()
        response = await self.get(f"/meeting/api/v2/{zsoid}/recordings/{meeting_key}.json")
        if response.status_code == 404:
            return None
        self._handle_response(response, "get_recording")
        data = response.json()
        recordings = data.get("recordings", [])
        return recordings[0] if recordings else None

    # ─────────────────────────────────────────────────────────────────
    # Meeting APIs
    # ─────────────────────────────────────────────────────────────────

    async def get_meeting(self, meeting_key: str) -> dict | None:
        """Get meeting details by meeting key.

        Returns:
            {
                "session": {
                    "meetingKey": 123456789,
                    "topic": "Monthly Marketing Meeting",
                    "agenda": "Points to discuss",
                    "presenterEmail": "presenter@company.com",
                    "presenterName": "Presenter Name",
                    "startTime": "Jun 19, 2020 07:00 PM IST",
                    "endTime": "Jun 19, 2020 08:00 PM IST",
                    "duration": 3600000,
                    "timezone": "Asia/Calcutta",
                    "joinLink": "https://meeting.zoho.com/join?key=123456789"
                }
            }
        """
        zsoid = await self._ensure_zsoid()
        response = await self.get(f"/api/v2/{zsoid}/sessions/{meeting_key}.json")
        if response.status_code == 404:
            return None
        self._handle_response(response, "get_meeting")
        return response.json().get("session")

    async def get_participants(
        self,
        meeting_key: str,
        index: int = 1,
        count: int = 100,
    ) -> list[dict]:
        """Get participant report for a meeting.

        Args:
            meeting_key: The meeting key
            index: Page number (1-indexed)
            count: Records per page

        Returns:
            List of participants with:
                - email: Participant email
                - role: "presenter" or "participant"
                - joinTime: Join time in epoch milliseconds
                - leaveTime: Leave time in epoch milliseconds
                - duration: Duration in milliseconds
                - source: "web", "mobile", etc.
                - memberId: Unique member ID
        """
        zsoid = await self._ensure_zsoid()
        params = {"index": index, "count": count}
        response = await self.get(
            f"/api/v2/{zsoid}/participant/{meeting_key}.json",
            params=params,
        )
        self._handle_response(response, "get_participants")
        return response.json().get("participants", [])

    async def get_all_participants(self, meeting_key: str) -> list[dict]:
        """Get all participants with pagination handling."""
        all_participants = []
        index = 1
        page_size = 100

        while True:
            participants = await self.get_participants(meeting_key, index, page_size)
            all_participants.extend(participants)
            if len(participants) < page_size:
                break
            index += 1

        return all_participants

    # ─────────────────────────────────────────────────────────────────
    # Error Handling
    # ─────────────────────────────────────────────────────────────────

    def _handle_response(self, response, operation: str):
        """Handle API response errors."""
        if response.status_code == 429:
            raise ZohoMeetingsRateLimitException(
                f"Rate limit exceeded for {operation}"
            )
        if response.status_code in [401, 403]:
            raise ZohoMeetingsAuthException(
                f"Authentication failed for {operation}: {response.text}"
            )
        if response.status_code >= 400:
            raise ZohoMeetingsAPIError(
                f"API error in {operation}: {response.status_code} - {response.text}"
            )
```

### Polling Mechanism (Primary Approach)

Since Zoho Meetings may not support webhooks for recording events, we implement a polling mechanism:

```python
# Location: sybill_py/runtimes/galactus/activities/zoho_meetings_polling.py

from temporalio import activity
from uuid import UUID
from pydantic import SecretStr

from sybill_py.components.upstream_providers.zoho_meetings.provider import ZohoMeetingsProvider
from sybill_py.components.upstream_providers.zoho_meetings.schema import ZohoRecording
from sybill_py.dal.queries.upstream_meetings import dao as upstream_meetings_dao
from schema.main import ZohoDCRegions


class PollZohoRecordingsActivityInput(SybillBaseModel):
    org_id: UUID
    access_token: SecretStr
    dc_region: ZohoDCRegions
    last_poll_time_ms: int | None = None


class PollZohoRecordingsActivityOutput(SybillBaseModel):
    new_recordings: list[ZohoRecording]
    latest_recording_time_ms: int | None = None
    errors: list[str] = Field(default_factory=list)


class PollZohoRecordingsActivity(DurableAsyncActivity):
    """Poll Zoho Meetings for new recordings."""

    @activity.defn(name="poll_zoho_recordings")
    async def run(
        self, input_: PollZohoRecordingsActivityInput
    ) -> PollZohoRecordingsActivityOutput:
        activity.logger.info(f"[OrgId={input_.org_id}] Polling Zoho recordings")

        provider = ZohoMeetingsProvider(
            dc_region=input_.dc_region,
            access_token=input_.access_token,
        )

        try:
            # Fetch all recordings
            all_recordings = await provider.fetch_all_recordings()

            # Filter to new recordings (after last poll time)
            new_recordings = []
            latest_time = input_.last_poll_time_ms or 0

            for recording in all_recordings:
                # Skip if already processed
                existing = await upstream_meetings_dao.reader.get_by_remote_id(
                    org_id=input_.org_id,
                    remote_id=recording.meeting_key,
                    provider=MeetingProviders.ZOHO_MEETINGS,
                )
                if existing:
                    continue

                # Skip if before last poll (unless first poll)
                if input_.last_poll_time_ms and recording.start_time_in_ms <= input_.last_poll_time_ms:
                    continue

                # Only process completed recordings
                if recording.status == "UPLOADED" and recording.download_url:
                    new_recordings.append(recording)
                    latest_time = max(latest_time, recording.start_time_in_ms)

            activity.logger.info(
                f"[OrgId={input_.org_id}] Found {len(new_recordings)} new recordings"
            )

            return PollZohoRecordingsActivityOutput(
                new_recordings=new_recordings,
                latest_recording_time_ms=latest_time if latest_time > 0 else None,
            )

        except Exception as e:
            activity.logger.error(
                f"[OrgId={input_.org_id}] Error polling recordings: {e}",
                exc_info=True,
            )
            return PollZohoRecordingsActivityOutput(
                new_recordings=[],
                errors=[str(e)],
            )
```

### Polling Workflow (Scheduled)

```python
# Location: sybill_py/runtimes/galactus/workflows/zoho_meetings_polling.py

from temporalio import workflow
from datetime import timedelta
from uuid import UUID
from pydantic import SecretStr, Field

from sybill_py.components.upstream_providers.zoho_meetings.schema import ZohoRecording
from schema.main import ZohoDCRegions, SybillBaseModel


# ─────────────────────────────────────────────────────────────────
# Workflow Input/Output Schemas
# ─────────────────────────────────────────────────────────────────

class ZohoMeetingsPollingWorkflowInput(SybillBaseModel):
    org_id: UUID
    access_token: SecretStr
    dc_region: ZohoDCRegions
    last_poll_time_ms: int | None = None


class ZohoMeetingsPollingWorkflowOutput(SybillBaseModel):
    success: bool
    error: str | None = None
    recordings_found: int = 0


class ProcessZohoRecordingWorkflowInput(SybillBaseModel):
    org_id: UUID
    recording: ZohoRecording
    access_token: SecretStr
    dc_region: ZohoDCRegions


class UpdateZohoIntegrationMetadataActivityInput(SybillBaseModel):
    org_id: UUID
    last_poll_time_ms: int | None = None
    zsoid: int | None = None


# ─────────────────────────────────────────────────────────────────
# Polling Workflow
# ─────────────────────────────────────────────────────────────────

@workflow.defn
class ZohoMeetingsPollingWorkflow(BaseWorkflow):
    """Scheduled workflow to poll for new Zoho recordings.

    Runs every 5-15 minutes per organization with active Zoho Meetings integration.
    """

    @workflow.run
    async def run(self, input_: ZohoMeetingsPollingWorkflowInput):
        workflow.logger.info(f"Starting Zoho polling for org {input_.org_id}")

        # 1. Poll for new recordings
        poll_output = await self.execute_activity(
            PollZohoRecordingsActivity,
            PollZohoRecordingsActivityInput(
                org_id=input_.org_id,
                access_token=input_.access_token,
                dc_region=input_.dc_region,
                last_poll_time_ms=input_.last_poll_time_ms,
            ),
        )

        if poll_output.errors:
            workflow.logger.error(f"Polling errors: {poll_output.errors}")
            return ZohoMeetingsPollingWorkflowOutput(
                success=False,
                error="; ".join(poll_output.errors),
            )

        # 2. Process each new recording
        for recording in poll_output.new_recordings:
            # Start child workflow for each recording
            await workflow.start_child_workflow(
                ProcessZohoRecordingWorkflow.run,
                ProcessZohoRecordingWorkflowInput(
                    org_id=input_.org_id,
                    recording=recording,
                    access_token=input_.access_token,
                    dc_region=input_.dc_region,
                ),
                id=f"zoho-recording-{input_.org_id}-{recording.meeting_key}",
            )

        # 3. Update last poll time in integration metadata
        await self.execute_activity(
            UpdateZohoIntegrationMetadataActivity,
            UpdateZohoIntegrationMetadataActivityInput(
                org_id=input_.org_id,
                last_poll_time_ms=poll_output.latest_recording_time_ms,
            ),
        )

        return ZohoMeetingsPollingWorkflowOutput(
            success=True,
            recordings_found=len(poll_output.new_recordings),
        )


# ─────────────────────────────────────────────────────────────────
# Recording Processing Workflow
# ─────────────────────────────────────────────────────────────────

@workflow.defn
class ProcessZohoRecordingWorkflow(BaseWorkflow):
    """Process a single Zoho recording into an UpstreamMeeting."""

    @workflow.run
    async def run(self, input_: ProcessZohoRecordingWorkflowInput):
        workflow.logger.info(
            f"Processing Zoho recording {input_.recording.meeting_key} for org {input_.org_id}"
        )

        # 1. Fetch participants for this recording
        participants_output = await self.execute_activity(
            FetchZohoParticipantsActivity,
            FetchZohoParticipantsActivityInput(
                meeting_key=input_.recording.meeting_key,
                access_token=input_.access_token,
                dc_region=input_.dc_region,
            ),
        )

        # 2. Download recording to S3
        download_output = await self.execute_activity(
            DownloadZohoRecordingActivity,
            DownloadZohoRecordingActivityInput(
                org_id=input_.org_id,
                recording=input_.recording,
            ),
        )

        if not download_output.s3_path:
            workflow.logger.error("Failed to download recording")
            return

        # 3. Create UpstreamMeeting document
        create_output = await self.execute_activity(
            CreateZohoUpstreamMeetingActivity,
            CreateZohoUpstreamMeetingActivityInput(
                org_id=input_.org_id,
                recording=input_.recording,
                participants=participants_output.participants,
                s3_recording_path=download_output.s3_path,
                s3_transcript_path=download_output.transcript_s3_path,
                access_token=input_.access_token,
                dc_region=input_.dc_region,
            ),
        )

        # 4. Trigger existing UpstreamProcessingWorkflow
        if create_output.upstream_meeting_id:
            await workflow.start_child_workflow(
                UpstreamProcessingWorkflow.run,
                UpstreamProcessingWorkflowInput(
                    upstream_meeting_id=create_output.upstream_meeting_id,
                    org_id=input_.org_id,
                ),
                id=f"upstream-processing-{create_output.upstream_meeting_id}",
            )

        workflow.logger.info(
            f"Successfully processed recording {input_.recording.meeting_key}"
        )
```

### Supporting Activities

```python
# Location: sybill_py/runtimes/galactus/activities/zoho_meetings.py

from temporalio import activity
from uuid import UUID
from pydantic import SecretStr, Field

from sybill_py.components.upstream_providers.zoho_meetings.provider import ZohoMeetingsProvider
from sybill_py.components.upstream_providers.zoho_meetings.schema import ZohoRecording, ZohoParticipant
from sybill_py.dal.queries.organizations import dao as orgs_dao
from schema.main import ZohoDCRegions, SybillBaseModel


# ─────────────────────────────────────────────────────────────────
# Activity Input/Output Schemas
# ─────────────────────────────────────────────────────────────────

class FetchZohoParticipantsActivityInput(SybillBaseModel):
    meeting_key: str
    access_token: SecretStr
    dc_region: ZohoDCRegions


class FetchZohoParticipantsActivityOutput(SybillBaseModel):
    participants: list[ZohoParticipant] = Field(default_factory=list)
    error: str | None = None


class DownloadZohoRecordingActivityInput(SybillBaseModel):
    org_id: UUID
    recording: ZohoRecording


class DownloadZohoRecordingActivityOutput(SybillBaseModel):
    s3_path: str | None = None
    transcript_s3_path: str | None = None
    error: str | None = None


class CreateZohoUpstreamMeetingActivityInput(SybillBaseModel):
    org_id: UUID
    recording: ZohoRecording
    participants: list[ZohoParticipant]
    s3_recording_path: str
    s3_transcript_path: str | None = None
    access_token: SecretStr
    dc_region: ZohoDCRegions


class CreateZohoUpstreamMeetingActivityOutput(SybillBaseModel):
    upstream_meeting_id: UUID | None = None
    error: str | None = None


# ─────────────────────────────────────────────────────────────────
# Activity Implementations
# ─────────────────────────────────────────────────────────────────

class FetchZohoParticipantsActivity(DurableAsyncActivity):
    """Fetch participants for a Zoho meeting."""

    @activity.defn(name="fetch_zoho_participants")
    async def run(
        self, input_: FetchZohoParticipantsActivityInput
    ) -> FetchZohoParticipantsActivityOutput:
        activity.logger.info(f"Fetching participants for meeting {input_.meeting_key}")

        try:
            provider = ZohoMeetingsProvider(
                dc_region=input_.dc_region,
                access_token=input_.access_token,
            )
            participants = await provider.fetch_participants(input_.meeting_key)

            return FetchZohoParticipantsActivityOutput(participants=participants)

        except Exception as e:
            activity.logger.error(f"Error fetching participants: {e}", exc_info=True)
            return FetchZohoParticipantsActivityOutput(error=str(e))


class DownloadZohoRecordingActivity(DurableAsyncActivity):
    """Download Zoho recording and optional transcript to S3."""

    @activity.defn(name="download_zoho_recording")
    async def run(
        self, input_: DownloadZohoRecordingActivityInput
    ) -> DownloadZohoRecordingActivityOutput:
        activity.logger.info(
            f"[OrgId={input_.org_id}] Downloading recording {input_.recording.meeting_key}"
        )

        try:
            # 1. Download video recording
            if not input_.recording.download_url:
                return DownloadZohoRecordingActivityOutput(
                    error="No download URL available"
                )

            s3_path = await self._download_to_s3(
                url=input_.recording.download_url,
                org_id=input_.org_id,
                meeting_key=input_.recording.meeting_key,
                file_type="video",
            )

            # 2. Download transcript if available
            transcript_s3_path = None
            if input_.recording.is_transcript_generated and input_.recording.transcription_download_url:
                transcript_s3_path = await self._download_to_s3(
                    url=input_.recording.transcription_download_url,
                    org_id=input_.org_id,
                    meeting_key=input_.recording.meeting_key,
                    file_type="transcript",
                )

            return DownloadZohoRecordingActivityOutput(
                s3_path=s3_path,
                transcript_s3_path=transcript_s3_path,
            )

        except Exception as e:
            activity.logger.error(f"Error downloading recording: {e}", exc_info=True)
            return DownloadZohoRecordingActivityOutput(error=str(e))

    async def _download_to_s3(
        self,
        url: str,
        org_id: UUID,
        meeting_key: str,
        file_type: str,
    ) -> str:
        """Stream download from URL to S3."""
        # Implementation uses existing S3 upload utilities
        # Similar to how other upstream providers handle downloads
        from common.s3_utils import upload_stream_to_s3

        s3_key = f"upstream_meetings/{org_id}/zoho/{meeting_key}/{file_type}"
        # Stream download to avoid memory issues with large files
        async with httpx.AsyncClient() as client:
            async with client.stream("GET", url) as response:
                response.raise_for_status()
                await upload_stream_to_s3(response.aiter_bytes(), s3_key)

        return s3_key


class CreateZohoUpstreamMeetingActivity(DurableAsyncActivity):
    """Create UpstreamMeeting document in MongoDB."""

    @activity.defn(name="create_zoho_upstream_meeting")
    async def run(
        self, input_: CreateZohoUpstreamMeetingActivityInput
    ) -> CreateZohoUpstreamMeetingActivityOutput:
        activity.logger.info(
            f"[OrgId={input_.org_id}] Creating upstream meeting for {input_.recording.meeting_key}"
        )

        try:
            provider = ZohoMeetingsProvider(
                dc_region=input_.dc_region,
                access_token=input_.access_token,
            )

            # 1. Build user mapping (email -> Sybill user ID)
            from sybill_py.dal.queries.users import dao as users_dao

            org_users = await users_dao.reader.get_users_by_org_id(input_.org_id)
            remote_to_sybill_user_map = {
                user.primary_email: user.id for user in org_users if user.primary_email
            }

            # 2. Create metadata
            metadata = await provider.create_upstream_metadata(
                recording=input_.recording,
                participants=input_.participants,
                remote_to_sybill_user_map=remote_to_sybill_user_map,
            )

            # 3. Create UpstreamMeeting document
            from sybill_py.dal.queries.upstream_meetings import dao as upstream_dao
            from schema.upstream_meeting_providers import (
                UpstreamMeeting,
                UpstreamMeetingState,
                StateEvents,
                UpstreamProcessingMetadata,
            )
            import time

            upstream_meeting = UpstreamMeeting(
                remote_id=input_.recording.meeting_key,
                provider=MeetingProviders.ZOHO_MEETINGS,
                org_id=input_.org_id,
                payload={
                    "recording": input_.recording.model_dump(),
                    "s3_recording_path": input_.s3_recording_path,
                    "s3_transcript_path": input_.s3_transcript_path,
                },
                metadata=metadata,
                state_events=[
                    UpstreamMeetingState(
                        event=StateEvents.UPSTREAM_MEETING_RECEIVED,
                        timestamp=int(time.time() * 1000),
                    )
                ],
                last_state_event=UpstreamMeetingState(
                    event=StateEvents.UPSTREAM_MEETING_RECEIVED,
                    timestamp=int(time.time() * 1000),
                ),
                processing_metadata=UpstreamProcessingMetadata(),
            )

            created = await upstream_dao.writer.create(upstream_meeting)

            return CreateZohoUpstreamMeetingActivityOutput(
                upstream_meeting_id=created.id
            )

        except Exception as e:
            activity.logger.error(f"Error creating upstream meeting: {e}", exc_info=True)
            return CreateZohoUpstreamMeetingActivityOutput(error=str(e))


class UpdateZohoIntegrationMetadataActivity(DurableAsyncActivity):
    """Update integration metadata (last_poll_time, zsoid, etc.)."""

    @activity.defn(name="update_zoho_integration_metadata")
    async def run(
        self, input_: UpdateZohoIntegrationMetadataActivityInput
    ) -> None:
        activity.logger.info(
            f"[OrgId={input_.org_id}] Updating Zoho integration metadata"
        )

        update_fields = {}
        if input_.last_poll_time_ms is not None:
            update_fields["integrations.zohoMeetings.lastPollTimeMs"] = input_.last_poll_time_ms
        if input_.zsoid is not None:
            update_fields["integrations.zohoMeetings.zsoid"] = input_.zsoid

        if update_fields:
            await orgs_dao.writer.update_org_fields(input_.org_id, update_fields)
```

---

### Integration Metadata Schema

The Zoho Meetings integration stores the following in the organization's `integrations` field:

```python
# In organization document: integrations.zohoMeetings
class ZohoMeetingsIntegrationMetadata(SybillBaseModel):
    """Stored in org.integrations.zohoMeetings"""

    # Auth0 identity for token management
    identity: str  # e.g., "oauth2|zoho-meetings|user123"

    # Zoho-specific
    zsoid: int  # Organization ID from Zoho (required for API calls)
    dc_region: ZohoDCRegions  # US, EU, IN, AU, etc.

    # Polling state
    last_poll_time_ms: int | None = None  # Timestamp of last successful poll
    poll_interval_minutes: int = 15  # Default polling interval

    # Integration health
    is_healthy: bool = True
    last_error: str | None = None
    error_count: int = 0
    last_successful_poll: int | None = None

    # Connection info
    connected_at: int  # Timestamp when integration was connected
    connected_by_user_id: UUID  # User who connected the integration
```

**MongoDB document example:**
```json
{
  "_id": "org-uuid",
  "integrations": {
    "zohoMeetings": {
      "identity": "oauth2|zoho-meetings|12345",
      "zsoid": 53758574,
      "dcRegion": "US",
      "lastPollTimeMs": 1697716087809,
      "pollIntervalMinutes": 15,
      "isHealthy": true,
      "lastError": null,
      "errorCount": 0,
      "connectedAt": 1697700000000,
      "connectedByUserId": "user-uuid"
    }
  }
}
```

---

### Auth0 Connection Configuration

Each Zoho region requires a **separate Auth0 connection** (same pattern as Zoho Calendar/CRM):

```
Auth0 Dashboard → Authentication → Social → Create Connection → Custom
```

**Connection Settings for US Region (`zoho-meetings-us`):**

| Field | Value |
|-------|-------|
| **Name** | `zoho-meetings-us` |
| **Authorization URL** | `https://accounts.zoho.com/oauth/v2/auth` |
| **Token URL** | `https://accounts.zoho.com/oauth/v2/token` |
| **Scope** | `ZohoMeeting.manageOrg.READ ZohoMeeting.meeting.READ ZohoMeeting.recording.READ ZohoMeeting.meetinguds.READ ZohoFiles.files.READ` |
| **Client ID** | (from Zoho API Console) |
| **Client Secret** | (from Zoho API Console) |

**Fetch User Profile Script** (reuse from Zoho Calendar):
```javascript
function fetchUserProfile(accessToken, context, callback) {
  const axios = require('axios');

  axios.get('https://meeting.zoho.com/api/v2/user.json', {
    headers: {
      'Authorization': 'Zoho-oauthtoken ' + accessToken,
      'Accept': 'application/json'
    }
  })
  .then(response => {
    const profile = response.data.userDetails;
    callback(null, {
      user_id: profile.zuid.toString(),
      email: profile.primaryEmail,
      name: profile.displayName,
      // Store zsoid in app_metadata for later use
      app_metadata: {
        zsoid: profile.zsoid,
        isAdmin: profile.isAdmin
      }
    });
  })
  .catch(err => {
    callback(err);
  });
}
```

**Multi-Region Support:**

| Region | Connection Name | Authorization URL |
|--------|----------------|-------------------|
| US | `zoho-meetings-us` | `https://accounts.zoho.com/oauth/v2/auth` |
| EU | `zoho-meetings-eu` | `https://accounts.zoho.eu/oauth/v2/auth` |
| IN | `zoho-meetings-in` | `https://accounts.zoho.in/oauth/v2/auth` |
| AU | `zoho-meetings-au` | `https://accounts.zoho.com.au/oauth/v2/auth` |

---

### login-combined-actions.js Updates

The Auth0 post-login action must be updated to handle the new integration type:

```javascript
// Location: sybill_js/auth0/login-combined-actions.js

// Add to the integration type handling section:
case 'zoho-meetings-us':
case 'zoho-meetings-eu':
case 'zoho-meetings-in':
case 'zoho-meetings-au':
  integrationType = 'ZOHO_MEETINGS_TEAM';
  // Extract region from connection name
  const region = event.connection.name.split('-').pop().toUpperCase();
  additionalData = {
    dcRegion: region,
    zsoid: event.user.app_metadata?.zsoid,
  };
  break;

// In linkOnBackendApi function, add handling:
if (integrationType === 'ZOHO_MEETINGS_TEAM') {
  apiPath = `/api/organizations/${sycOrgId}/link-account`;
  payload = {
    integrationType: integrationType,
    identity: event.user.user_id,
    ...additionalData,
  };
}
```

**Key Points:**
1. Connection name encodes the region (e.g., `zoho-meetings-us`)
2. `zsoid` is passed from the fetch user profile script
3. Calls org-level Link Account API (not user-level)

---

### Frontend OAuth Flow

The frontend initiates the OAuth flow for org-level integrations:

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐
│   Frontend  │     │    Auth0     │     │    Zoho     │     │   Backend   │
│ (Settings)  │     │              │     │   OAuth     │     │ Link API    │
└──────┬──────┘     └──────┬───────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                    │                   │
       │ 1. Click "Connect │                    │                   │
       │    Zoho Meetings" │                    │                   │
       │──────────────────>│                    │                   │
       │                   │                    │                   │
       │                   │ 2. Redirect to     │                   │
       │                   │    Zoho OAuth      │                   │
       │                   │───────────────────>│                   │
       │                   │                    │                   │
       │                   │                    │ 3. User approves  │
       │                   │                    │    (scopes)       │
       │                   │                    │                   │
       │                   │ 4. Callback with   │                   │
       │                   │    auth code       │                   │
       │                   │<───────────────────│                   │
       │                   │                    │                   │
       │                   │ 5. Exchange for    │                   │
       │                   │    tokens          │                   │
       │                   │───────────────────>│                   │
       │                   │<───────────────────│                   │
       │                   │                    │                   │
       │                   │ 6. Post-login hook │                   │
       │                   │    calls backend   │                   │
       │                   │────────────────────────────────────────>│
       │                   │                    │                   │
       │                   │                    │     7. Create     │
       │                   │                    │     integration,  │
       │                   │                    │     link accounts │
       │                   │                    │                   │
       │ 8. Redirect back  │                    │                   │
       │    with success   │                    │                   │
       │<──────────────────│                    │                   │
       │                   │                    │                   │
```

**Auth0 authorize URL construction:**
```javascript
// Frontend: initiate OAuth flow
const connectZohoMeetings = (orgId: string) => {
  const params = new URLSearchParams({
    client_id: AUTH0_CLIENT_ID,
    response_type: 'code',
    connection: 'zoho-meetings',  // Auth0 connection name
    redirect_uri: `${BACKEND_URL}/auth/callback`,
    scope: 'openid profile',
    state: JSON.stringify({ orgId, integrationType: 'ZOHO_MEETINGS_TEAM' }),
    // Pass org context to Auth0 post-login hook
    syc_org_id: orgId,
  });

  window.location.href = `https://${AUTH0_DOMAIN}/authorize?${params}`;
};
```

---

### Temporal Schedule Setup

Polling is scheduled per-organization using Temporal schedules:

```python
# Location: sybill_py/runtimes/galactus/schedules/zoho_meetings.py

from temporalio.client import Client, Schedule, ScheduleSpec, ScheduleIntervalSpec
from datetime import timedelta

async def create_zoho_polling_schedule(
    temporal_client: Client,
    org_id: UUID,
    poll_interval_minutes: int = 15,
) -> str:
    """Create a Temporal schedule for polling Zoho recordings.

    Called when:
    1. Org first connects Zoho Meetings integration
    2. Admin changes polling interval
    """
    schedule_id = f"zoho-polling-{org_id}"

    # Get integration details from org
    org = await orgs_dao.reader.get_org_by_id(org_id)
    zoho_integration = org.integrations.get("zohoMeetings")

    if not zoho_integration:
        raise ValueError(f"Org {org_id} does not have Zoho Meetings integration")

    # Get access token from Auth0
    access_token = await get_zoho_access_token(org_id)

    schedule = Schedule(
        action=ScheduleActionStartWorkflow(
            ZohoMeetingsPollingWorkflow.run,
            ZohoMeetingsPollingWorkflowInput(
                org_id=org_id,
                access_token=access_token,
                dc_region=ZohoDCRegions(zoho_integration.dc_region),
                last_poll_time_ms=zoho_integration.last_poll_time_ms,
            ),
            id=f"zoho-polling-{org_id}-{{workflow.now}}",
            task_queue="galactus-task-queue",
        ),
        spec=ScheduleSpec(
            intervals=[
                ScheduleIntervalSpec(
                    every=timedelta(minutes=poll_interval_minutes),
                    offset=timedelta(seconds=hash(str(org_id)) % 60),  # Jitter
                )
            ]
        ),
    )

    await temporal_client.create_schedule(schedule_id, schedule)
    return schedule_id


async def delete_zoho_polling_schedule(
    temporal_client: Client,
    org_id: UUID,
) -> None:
    """Delete polling schedule when integration is disconnected."""
    schedule_id = f"zoho-polling-{org_id}"
    handle = temporal_client.get_schedule_handle(schedule_id)
    await handle.delete()
```

---

### Error Handling & Recovery

```python
# Error handling strategy for polling workflow

class ZohoMeetingsPollingWorkflow(BaseWorkflow):
    @workflow.run
    async def run(self, input_: ZohoMeetingsPollingWorkflowInput):
        max_retries = 3
        retry_count = 0

        while retry_count < max_retries:
            try:
                poll_output = await self.execute_activity(
                    PollZohoRecordingsActivity,
                    input_,
                    start_to_close_timeout=timedelta(minutes=5),
                    retry_policy=RetryPolicy(
                        initial_interval=timedelta(seconds=10),
                        backoff_coefficient=2.0,
                        maximum_interval=timedelta(minutes=5),
                        maximum_attempts=3,
                        non_retryable_error_types=[
                            "ZohoMeetingsAuthException",  # Don't retry auth errors
                        ],
                    ),
                )

                if not poll_output.errors:
                    # Success - reset error count
                    await self._update_health(input_.org_id, healthy=True)
                    break

                # Handle specific errors
                for error in poll_output.errors:
                    if "rate limit" in error.lower():
                        # Wait and retry
                        await workflow.sleep(timedelta(minutes=5))
                        retry_count += 1
                    elif "auth" in error.lower():
                        # Auth error - mark unhealthy, don't retry
                        await self._update_health(
                            input_.org_id,
                            healthy=False,
                            error="Authentication failed - reconnection required",
                        )
                        return
                    else:
                        retry_count += 1

            except Exception as e:
                workflow.logger.error(f"Polling failed: {e}")
                retry_count += 1

        if retry_count >= max_retries:
            await self._update_health(
                input_.org_id,
                healthy=False,
                error=f"Polling failed after {max_retries} retries",
            )

    async def _update_health(
        self,
        org_id: UUID,
        healthy: bool,
        error: str | None = None,
    ):
        await self.execute_activity(
            UpdateZohoIntegrationMetadataActivity,
            UpdateZohoIntegrationMetadataActivityInput(
                org_id=org_id,
                is_healthy=healthy,
                last_error=error,
            ),
        )
```

---

### Webhook Handler (Fallback if Available)

```python
# Location: src/routers/upstream_meetings.py (new endpoint)

@router.post("/webhooks/zoho-meetings/{org_id}")
async def on_zoho_meetings_webhook(
    org_id: UUID,
    request: Request,
    background_tasks: BackgroundTasks,
):
    """Handle Zoho Meetings webhook events (if webhooks are supported).

    Note: This is a fallback. Primary ingestion uses polling.
    """

    # 1. Validate signature
    signature = request.headers.get("X-Zoho-Signature", "")
    raw_body = await request.body()

    provider = UpstreamProviderRegistry.get_provider(
        MeetingProviders.ZOHO_MEETINGS,
        dc_region=ZohoDCRegions.US,  # Get from org integration
    )

    if not await provider.validate_webhook_signature(
        raw_body, signature, org_webhook_secret
    ):
        raise HTTPException(status_code=401, detail="Invalid signature")

    # 2. Parse payload
    payload = await request.json()
    event_type = payload.get("event_type")

    # 3. Handle recording.completed event
    if event_type == "recording.completed":
        background_tasks.add_task(
            process_zoho_meeting_recording,
            org_id=org_id,
            payload=payload,
        )

    return {"status": "ok"}
```

### Directory Structure

```
sybill_py/
├── components/
│   └── upstream_providers/
│       ├── __init__.py
│       ├── base.py                    # BaseUpstreamProvider ABC
│       ├── registry.py                # UpstreamProviderRegistry
│       ├── exceptions.py              # Provider-specific exceptions
│       │
│       ├── zoho_meetings/
│       │   ├── __init__.py
│       │   ├── provider.py            # ZohoMeetingsProvider
│       │   ├── client.py              # ZohoMeetingsClient
│       │   ├── schema.py              # Zoho-specific schemas
│       │   └── field_mappings.py      # Data transformation
│       │
│       ├── webex/                     # Future: WebEx provider
│       │   └── ...
│       │
│       └── teams_native/              # Future: Teams native recordings
│           └── ...
│
├── runtimes/
│   └── galactus/
│       └── activities/
│           └── upstream_providers/    # Provider-specific activities
│               └── zoho_meetings.py
```

---

## Implementation Plan

### Phase 1: Foundation (Day 1)

| Task | Description |
|------|-------------|
| 1.1 | Add `ZOHO_MEETINGS` to `MeetingProviders` enum in `schema/main.py` |
| 1.2 | Add `ZOHO` to `ConferenceProviders` enum (if not exists) |
| 1.3 | Create `BaseUpstreamProvider` abstract class |
| 1.4 | Create `UpstreamProviderRegistry` |
| 1.5 | Set up directory structure: `sybill_py/components/upstream_providers/zoho_meetings/` |

### Phase 2: Zoho Meetings Client & Schemas (Day 1-2)

| Task | Description |
|------|-------------|
| 2.1 | Create `ZohoRecording`, `ZohoParticipant`, `ZohoMeetingSession` schemas |
| 2.2 | Create `ZohoMeetingsClient` with API v2 endpoints |
| 2.3 | Implement `get_user_details()` to fetch zsoid |
| 2.4 | Implement `get_all_recordings()` for polling |
| 2.5 | Implement `get_meeting()` and `get_all_participants()` |
| 2.6 | Add error handling: rate limits, auth errors, API errors |
| 2.7 | Test with Zoho sandbox account |

### Phase 3: Auth0 Integration (Day 2)

| Task | Description |
|------|-------------|
| 3.1 | Create Auth0 connection `zoho-meetings` with required scopes |
| 3.2 | Add integration type `ZOHO_MEETINGS_TEAM` to `IntegrationTypes` enum |
| 3.3 | Update `login-combined-actions.js` for new connection type |
| 3.4 | Implement `create_zoho_meetings_integration()` in `integration_utils.py` |
| 3.5 | Add handler to Link Account API in `organizations.py` |
| 3.6 | Store `zsoid` in integration metadata after first API call |

### Phase 4: Provider Implementation (Day 2-3)

| Task | Description |
|------|-------------|
| 4.1 | Implement `ZohoMeetingsProvider` class with registry registration |
| 4.2 | Implement `create_metadata()` with Zoho → UpstreamCallMetadata mapping |
| 4.3 | Implement `_create_invitees()` from participants (no separate invitee API) |
| 4.4 | Implement `_create_participants()` with speaker ID tracking |
| 4.5 | Handle native transcript: check `isTranscriptGenerated`, download if available |
| 4.6 | Handle native summary: check `isSummaryGenerated`, use if available |

### Phase 5: Polling & Processing (Day 3)

| Task | Description |
|------|-------------|
| 5.1 | Create `PollZohoRecordingsActivity` for fetching new recordings |
| 5.2 | Create `ProcessZohoRecordingActivity` for individual recording processing |
| 5.3 | Create `ZohoMeetingsPollingWorkflow` (scheduled, per-org) |
| 5.4 | Implement deduplication: check if `remote_id` already exists in `upstream_meetings` |
| 5.5 | Download recording to S3, create `UpstreamMeeting` document |
| 5.6 | Trigger `UpstreamProcessingWorkflow` for call processing |
| 5.7 | Set up Temporal schedule for polling (every 10-15 minutes) |

### Phase 6: Testing & Demo (Day 4)

| Task | Description |
|------|-------------|
| 6.1 | Unit tests for `ZohoMeetingsClient` with mocked responses |
| 6.2 | Unit tests for `ZohoMeetingsProvider` metadata creation |
| 6.3 | Integration tests with Temporal test environment |
| 6.4 | End-to-end demo: connect Zoho → poll → process → view in Sybill |
| 6.5 | Update documentation |

### Critical Path

```
Day 1: Foundation + Client basics
         ↓
Day 2: Auth0 + Client completion + Provider start
         ↓
Day 3: Provider completion + Polling workflow
         ↓
Day 4: Testing + Demo
```

### Demo Requirements (Monday)

For a working demo, minimum viable features:
1. ✅ OAuth flow working (Auth0 connection)
2. ✅ Can fetch recordings list from Zoho API
3. ✅ Can download at least one recording
4. ✅ Recording appears in Sybill as upstream meeting
5. ⭕ Transcript generated (can skip if time-constrained)
6. ⭕ Full call processing (insights, etc.)

---

## Risk Assessment

### High Risk

| Risk | Mitigation |
|------|------------|
| Zoho API undocumented edge cases | Test extensively with real account; API documented in Appendix A |
| Recording download URL expiry | Download immediately upon discovery; store in S3 |
| Large recording files | Stream download to S3; handle timeout appropriately |

### Medium Risk

| Risk | Mitigation |
|------|------------|
| Polling latency | 10-15 min polling interval acceptable for non-real-time use case |
| Multi-region complexity | Follow existing `ZohoCalendarClient` pattern; store region in integration |
| Token refresh issues | Use existing `OAuth2AsyncClient` pattern; test with long-running sessions |
| Rate limiting | Implement exponential backoff; polling is low-frequency |
| Native transcript format | May need parsing; fall back to own transcription if incompatible |

### Low Risk

| Risk | Mitigation |
|------|------------|
| Auth0 connection setup | Well-documented pattern exists; reuse Zoho Calendar scopes logic |
| Schema mapping | Follow Gong/Chorus patterns; field mappings documented |
| Temporal integration | Existing `UpstreamProcessingWorkflow` supports new providers |
| zsoid caching | Store in integration metadata after first API call |

### Mitigated Risks

| Original Risk | Resolution |
|---------------|------------|
| Webhook not available | Using polling mechanism (primary approach) |
| API documentation quality | API endpoints now validated and documented in Appendix A |
| Unknown data structure | Schemas defined: `ZohoRecording`, `ZohoParticipant`, etc. |

### Speaker Matching Considerations

From the transcript discussion, speaker matching is critical for transcription quality:

**When Zoho provides participant info:**
- Participant `email` and `memberId` are used to identify speakers
- Map to Sybill users via email lookup
- Unresolved speakers tracked in `unresolved_speaker_ids` for LLM inference

**Fallback: Calendar Invite Matching**

If Zoho's participant report is incomplete (missing emails), fall back to calendar invite matching:

```python
async def match_participants_from_calendar(
    org_id: UUID,
    recording: ZohoRecording,
    zoho_participants: list[ZohoParticipant],
) -> list[UpstreamParticipant]:
    """
    Fallback: Match participants using calendar invites if Zoho data is incomplete.

    1. Find calendar event matching the meeting time window
    2. Extract attendees from the calendar invite
    3. Match Zoho participants by join time overlap
    """
    # Find calendar events in the time window
    from sybill_py.dal.queries.calendar_events import dao as calendar_dao

    start_time = recording.start_time_in_ms
    end_time = recording.start_time_in_ms + recording.duration

    # Look for events within ±15 minutes of recording
    events = await calendar_dao.reader.find_events_in_range(
        org_id=org_id,
        start_time_ms=start_time - (15 * 60 * 1000),
        end_time_ms=end_time + (15 * 60 * 1000),
    )

    if not events:
        return []  # No matching calendar event

    # Find best matching event (closest start time)
    best_event = min(events, key=lambda e: abs(e.start_time_ms - start_time))

    # Use calendar attendees to fill in missing participant info
    attendee_emails = {a.email for a in best_event.attendees}

    enhanced_participants = []
    for p in zoho_participants:
        if p.email:
            enhanced_participants.append(p)
        elif len(attendee_emails) == len(zoho_participants):
            # Try to match by position/join order (heuristic)
            # This is imperfect but better than nothing
            pass

    return enhanced_participants
```

**When to trigger LLM speaker inference:**
- `unresolved_speaker_ids` is non-empty
- Participant names are generic ("Speaker #1", "Speaker #2")
- Existing `UpstreamProcessingWorkflow` handles this via speaker inference activity

---

## Testing Strategy

### Unit Tests

```python
# Location: tests/sybill_py/components/upstream_providers/zoho_meetings/test_client.py

import pytest
from unittest.mock import AsyncMock, patch
from pydantic import SecretStr

from sybill_py.components.upstream_providers.zoho_meetings.client import ZohoMeetingsClient
from sybill_py.components.upstream_providers.zoho_meetings.schema import ZohoRecording
from schema.main import ZohoDCRegions


@pytest.fixture
def mock_access_token():
    return SecretStr("test-access-token")


@pytest.fixture
def mock_recordings_response():
    return {
        "recordings": [
            {
                "erecordingId": "rec-123",
                "meetingKey": "meeting-456",
                "topic": "Test Meeting",
                "downloadUrl": "https://download.zoho.com/test",
                "isTranscriptGenerated": True,
                "transcriptionDownloadUrl": "https://files.zoho.com/transcript",
                "duration": 3600000,
                "startTimeinMs": 1697716087809,
                "status": "UPLOADED",
            }
        ],
        "meta": {"moreRecords": False, "count": 1},
    }


@pytest.fixture
def mock_user_response():
    return {
        "userDetails": {
            "zsoid": 53758574,
            "zuid": 53758790,
            "primaryEmail": "admin@company.com",
            "displayName": "Admin User",
            "isAdmin": True,
        }
    }


class TestZohoMeetingsClient:
    @pytest.mark.asyncio
    async def test_get_user_details(self, mock_access_token, mock_user_response):
        with patch.object(ZohoMeetingsClient, "get", new_callable=AsyncMock) as mock_get:
            mock_get.return_value.json.return_value = mock_user_response
            mock_get.return_value.status_code = 200

            async with ZohoMeetingsClient(mock_access_token, ZohoDCRegions.US) as client:
                result = await client.get_user_details()

            assert result["userDetails"]["zsoid"] == 53758574
            assert client._zsoid == 53758574

    @pytest.mark.asyncio
    async def test_get_all_recordings(
        self, mock_access_token, mock_user_response, mock_recordings_response
    ):
        with patch.object(ZohoMeetingsClient, "get", new_callable=AsyncMock) as mock_get:
            # First call: get_user_details, Second call: get_all_recordings
            mock_get.return_value.json.side_effect = [
                mock_user_response,
                mock_recordings_response,
            ]
            mock_get.return_value.status_code = 200

            async with ZohoMeetingsClient(mock_access_token, ZohoDCRegions.US) as client:
                recordings = await client.get_all_recordings()

            assert len(recordings) == 1
            assert recordings[0]["meetingKey"] == "meeting-456"

    @pytest.mark.asyncio
    async def test_rate_limit_exception(self, mock_access_token):
        with patch.object(ZohoMeetingsClient, "get", new_callable=AsyncMock) as mock_get:
            mock_get.return_value.status_code = 429
            mock_get.return_value.text = "Rate limit exceeded"

            async with ZohoMeetingsClient(mock_access_token, ZohoDCRegions.US) as client:
                with pytest.raises(ZohoMeetingsRateLimitException):
                    await client.get_user_details()


class TestZohoMeetingsProvider:
    @pytest.mark.asyncio
    async def test_create_upstream_metadata(self, mock_access_token):
        from sybill_py.components.upstream_providers.zoho_meetings.provider import (
            ZohoMeetingsProvider,
        )
        from sybill_py.components.upstream_providers.zoho_meetings.schema import (
            ZohoRecording,
            ZohoParticipant,
        )

        recording = ZohoRecording(
            erecording_id="rec-123",
            meeting_key="meeting-456",
            topic="Test Meeting",
            download_url="https://download.zoho.com/test",
            is_transcript_generated=True,
            duration=3600000,
            start_time_in_ms=1697716087809,
            status="UPLOADED",
        )

        participants = [
            ZohoParticipant(
                email="presenter@company.com",
                role="presenter",
                join_time=1697716087809,
                leave_time=1697719687809,
                member_id="member-1",
            ),
            ZohoParticipant(
                email="external@client.com",
                role="participant",
                join_time=1697716087809,
                leave_time=1697719687809,
                member_id="member-2",
            ),
        ]

        provider = ZohoMeetingsProvider(
            dc_region=ZohoDCRegions.US,
            access_token=mock_access_token,
        )

        # Mock user mapping - only internal user is mapped
        user_map = {"presenter@company.com": UUID("12345678-1234-5678-1234-567812345678")}

        metadata = await provider.create_upstream_metadata(
            recording=recording,
            participants=participants,
            remote_to_sybill_user_map=user_map,
        )

        assert metadata.topic == "Test Meeting"
        assert metadata.provider == MeetingProviders.ZOHO_MEETINGS
        assert metadata.meeting_type == MeetingTypes.EXTERNAL  # Has external participant
        assert len(metadata.participants) == 2
        assert len(metadata.invitees) == 2
```

### Workflow Tests

```python
# Location: tests/sybill_py/runtimes/galactus/workflows/test_zoho_meetings_polling.py

import pytest
from temporalio.testing import WorkflowEnvironment
from temporalio.worker import Worker
from uuid import UUID

from tests.sybill_py.runtimes.galactus.test_helpers.activity import MockActivityFactory


def create_zoho_polling_activities(
    recordings: list = None,
    poll_error: str | None = None,
):
    """Create mock activities for Zoho polling workflow tests."""
    activities = []

    # Poll activity
    activities.append(
        MockActivityFactory.create_mock_activity(
            "poll_zoho_recordings",
            return_value={
                "new_recordings": recordings or [],
                "latest_recording_time_ms": 1697716087809 if recordings else None,
                "errors": [poll_error] if poll_error else [],
            },
        )
    )

    # Update metadata activity
    activities.append(
        MockActivityFactory.create_mock_activity(
            "update_zoho_integration_metadata",
            return_value=None,
        )
    )

    return activities


@pytest.fixture
def workflow_id():
    return "test-zoho-polling-workflow"


@pytest.fixture
def workflow_input():
    return ZohoMeetingsPollingWorkflowInput(
        org_id=UUID("12345678-1234-5678-1234-567812345678"),
        access_token=SecretStr("test-token"),
        dc_region=ZohoDCRegions.US,
        last_poll_time_ms=None,
    )


@pytest.mark.asyncio
async def test_polling_workflow_no_recordings(workflow_id, workflow_input):
    """Test polling when no new recordings found."""
    env = await WorkflowEnvironment.start_local()
    try:
        async with Worker(
            env.client,
            task_queue="test-task-queue",
            workflows=[ZohoMeetingsPollingWorkflow],
            activities=create_zoho_polling_activities(recordings=[]),
        ):
            handle = await env.client.start_workflow(
                ZohoMeetingsPollingWorkflow.run,
                workflow_input,
                id=workflow_id,
                task_queue="test-task-queue",
            )
            result = await handle.result()

        assert result.success is True
        assert result.recordings_found == 0
    finally:
        await env.shutdown()


@pytest.mark.asyncio
async def test_polling_workflow_with_recordings(workflow_id, workflow_input):
    """Test polling when new recordings found."""
    mock_recording = ZohoRecording(
        erecording_id="rec-123",
        meeting_key="meeting-456",
        topic="Test",
        duration=3600000,
        start_time_in_ms=1697716087809,
        status="UPLOADED",
    )

    activities = create_zoho_polling_activities(recordings=[mock_recording])

    # Add ProcessZohoRecordingWorkflow mock
    # In real test, this would be a child workflow mock

    env = await WorkflowEnvironment.start_local()
    try:
        async with Worker(
            env.client,
            task_queue="test-task-queue",
            workflows=[ZohoMeetingsPollingWorkflow, ProcessZohoRecordingWorkflow],
            activities=activities,
        ):
            handle = await env.client.start_workflow(
                ZohoMeetingsPollingWorkflow.run,
                workflow_input,
                id=workflow_id,
                task_queue="test-task-queue",
            )
            result = await handle.result()

        assert result.success is True
        assert result.recordings_found == 1
    finally:
        await env.shutdown()


@pytest.mark.asyncio
async def test_polling_workflow_auth_error(workflow_id, workflow_input):
    """Test polling handles auth error gracefully."""
    activities = create_zoho_polling_activities(
        poll_error="Authentication failed: invalid token"
    )

    env = await WorkflowEnvironment.start_local()
    try:
        async with Worker(
            env.client,
            task_queue="test-task-queue",
            workflows=[ZohoMeetingsPollingWorkflow],
            activities=activities,
        ):
            handle = await env.client.start_workflow(
                ZohoMeetingsPollingWorkflow.run,
                workflow_input,
                id=workflow_id,
                task_queue="test-task-queue",
            )
            result = await handle.result()

        assert result.success is False
        assert "Authentication failed" in result.error
    finally:
        await env.shutdown()
```

### Integration Test with Mock Zoho Server

```python
# Location: tests/integration/test_zoho_meetings_e2e.py

import pytest
from httpx import ASGITransport, AsyncClient
from fastapi import FastAPI
from unittest.mock import patch, AsyncMock

# Create mock Zoho API server for testing
@pytest.fixture
def mock_zoho_api():
    """Mock Zoho API responses for E2E testing."""
    app = FastAPI()

    @app.get("/api/v2/user.json")
    async def get_user():
        return {
            "userDetails": {
                "zsoid": 12345,
                "zuid": 67890,
                "primaryEmail": "test@company.com",
                "isAdmin": True,
            }
        }

    @app.get("/meeting/api/v2/{zsoid}/recordings.json")
    async def get_recordings(zsoid: int):
        return {
            "recordings": [
                {
                    "erecordingId": "e2e-rec-1",
                    "meetingKey": "e2e-meeting-1",
                    "topic": "E2E Test Meeting",
                    "downloadUrl": "http://mock-zoho/download/video",
                    "isTranscriptGenerated": False,
                    "duration": 1800000,
                    "startTimeinMs": 1697716087809,
                    "status": "UPLOADED",
                }
            ],
            "meta": {"moreRecords": False},
        }

    @app.get("/api/v2/{zsoid}/participant/{meeting_key}.json")
    async def get_participants(zsoid: int, meeting_key: str, index: int = 1, count: int = 100):
        return {
            "participantsCount": 2,
            "participants": [
                {
                    "email": "host@company.com",
                    "role": "presenter",
                    "joinTime": 1697716087809,
                    "leaveTime": 1697717887809,
                    "memberId": "host-member-id",
                },
                {
                    "email": "guest@external.com",
                    "role": "participant",
                    "joinTime": 1697716087809,
                    "leaveTime": 1697717887809,
                    "memberId": "guest-member-id",
                },
            ],
        }

    return app


@pytest.mark.asyncio
async def test_e2e_zoho_recording_ingestion(mock_zoho_api):
    """End-to-end test: Zoho recording → UpstreamMeeting → Processing."""
    # This test verifies the complete flow using mocked Zoho API
    pass  # Implementation depends on test infrastructure
```

---

## Future Extensibility

### Adding a New Provider (e.g., WebEx)

1. **Create provider package:**
   ```
   sybill_py/components/upstream_providers/webex/
   ```

2. **Add to MeetingProviders enum:**
   ```python
   WEBEX_NATIVE = "WEBEX_NATIVE"
   ```

3. **Implement provider class:**
   ```python
   @UpstreamProviderRegistry.register(MeetingProviders.WEBEX_NATIVE)
   class WebExNativeProvider(BaseUpstreamProvider[WebExPayload]):
       ...
   ```

4. **Add Auth0 connection and Link Account handling**

5. **Add webhook endpoint:**
   ```python
   @router.post("/webhooks/webex/{org_id}")
   ```

The generic architecture ensures:
- Consistent data flow through existing `UpstreamProcessingWorkflow`
- Reusable OAuth/token patterns
- Standardized error handling
- Easy provider registration

---

## Appendix

### A. Zoho Meetings API Reference (Validated)

#### Base URLs

| Region | Base URL |
|--------|----------|
| US (default) | `https://meeting.zoho.com/api/v2` |
| EU | `https://meeting.zoho.eu/api/v2` |
| India | `https://meeting.zoho.in/api/v2` |
| Australia | `https://meeting.zoho.com.au/api/v2` |

---

#### 1. Authentication APIs

##### Get Authorization Code
| | |
|---|---|
| **URL** | `https://accounts.zoho.com/oauth/v2/auth` |
| **Method** | GET |
| **Docs** | [Authentication](https://www.zoho.com/meeting/api-integration/authentication.html) |

**Parameters:**
- `scope` - e.g., `ZohoMeeting.meeting.READ,ZohoMeeting.recording.READ`
- `client_id` - Your app's client ID
- `response_type` - `code`
- `redirect_uri` - Your callback URL
- `access_type` - `offline` (to get refresh token)
- `prompt` - `consent`

##### Get Access Token
| | |
|---|---|
| **URL** | `https://accounts.zoho.com/oauth/v2/token` |
| **Method** | POST |
| **Docs** | [Authentication](https://www.zoho.com/meeting/api-integration/authentication.html) |

**Parameters:**
- `code` - Authorization code from step 1
- `client_id` - Your app's client ID
- `client_secret` - Your app's client secret
- `redirect_uri` - Same as step 1
- `grant_type` - `authorization_code`

##### Refresh Access Token
| | |
|---|---|
| **URL** | `https://accounts.zoho.com/oauth/v2/token` |
| **Method** | POST |

**Parameters:**
- `refresh_token` - Stored refresh token
- `client_id` - Your app's client ID
- `client_secret` - Your app's client secret
- `grant_type` - `refresh_token`

---

#### 2. User & Organization APIs

##### Get User Details (Get zsoid)
| | |
|---|---|
| **URL** | `https://meeting.zoho.com/api/v2/user.json` |
| **Method** | GET |
| **Scope** | `ZohoMeeting.manageOrg.READ` |
| **Docs** | [Organization ID](https://www.zoho.com/meeting/api-integration/organization-id.html) |

**Purpose:** Get the `zsoid` (organization ID) required for all other API calls.

**Response:**
```json
{
  "userDetails": {
    "zsoid": 53758574,        // ⭐ Required for all other APIs
    "zuid": 53758790,
    "primaryEmail": "user@company.com",
    "displayName": "User Name",
    "isAdmin": true
  }
}
```

---

#### 3. Recording APIs (Core for Integration)

##### Get All Recordings
| | |
|---|---|
| **URL** | `https://meeting.zoho.com/meeting/api/v2/{zsoid}/recordings.json` |
| **Method** | GET |
| **Scope** | `ZohoMeeting.recording.READ` |
| **Docs** | [Get All Recordings](https://www.zoho.com/meeting/api-integration/recording-api/get-all-recordings.html) |

**Response:**
```json
{
  "recordings": [
    {
      "erecordingId": "abc123...",
      "meetingKey": "1087598854",
      "topic": "Team Meeting",
      "downloadUrl": "https://download.zoho.com/webdownload?...",
      "transcriptionDownloadUrl": "https://files.zoho.com/download?...",
      "summaryDownloadUrl": "https://files.zoho.com/download?...",
      "isTranscriptGenerated": true,
      "isSummaryGenerated": true,
      "duration": 77268,
      "durationInMins": 1,
      "startTimeinMs": 1697716087809,
      "fileSize": "1 MB",
      "status": "UPLOADED"
    }
  ],
  "meta": {
    "moreRecords": false,
    "count": 4
  }
}
```

##### Get Specific Recording by Meeting Key
| | |
|---|---|
| **URL** | `https://meeting.zoho.com/meeting/api/v2/{zsoid}/recordings/{meetingKey}.json` |
| **Method** | GET |
| **Scope** | `ZohoMeeting.recording.READ` |
| **Docs** | [Get Specific Recording](https://www.zoho.com/meeting/api-integration/recording-api/get-specific-recording.html) |

##### Download Recording
| | |
|---|---|
| **URL** | Use `downloadUrl` from recording response |
| **Method** | GET |
| **Scope** | `ZohoMeeting.meetinguds.READ`, `ZohoFiles.files.READ` |
| **Docs** | [Download Recording](https://www.zoho.com/meeting/api-integration/recording-api/download-recording.html) |

**Note:** The `downloadUrl` is a pre-signed URL returned in the recording list response.

---

#### 4. Meeting APIs

##### Get Meeting Details
| | |
|---|---|
| **URL** | `https://meeting.zoho.com/api/v2/{zsoid}/sessions/{meetingKey}.json` |
| **Method** | GET |
| **Scope** | `ZohoMeeting.meeting.READ` |
| **Docs** | [Get Meeting API](https://www.zoho.com/meeting/api-integration/meeting-api/get-meeting-api.html) |

**Response:**
```json
{
  "session": {
    "meetingKey": 123456789,
    "topic": "Monthly Marketing Meeting",
    "agenda": "Points to discuss",
    "presenterEmail": "presenter@company.com",
    "presenterName": "Presenter Name",
    "startTime": "Jun 19, 2020 07:00 PM IST",
    "endTime": "Jun 19, 2020 08:00 PM IST",
    "duration": 3600000,
    "timezone": "Asia/Calcutta",
    "joinLink": "https://meeting.zoho.com/join?key=123456789"
  }
}
```

##### Get Meeting Participant Report ⭐
| | |
|---|---|
| **URL** | `https://meeting.zoho.com/api/v2/{zsoid}/participant/{meetingKey}.json` |
| **Method** | GET |
| **Scope** | `ZohoMeeting.meeting.READ` |
| **Docs** | [Participant Report](https://www.zoho.com/meeting/api-integration/meeting-api/participant-report.html) |

**Parameters:**
- `index` - Page number (required)
- `count` - Records per page (required)

**Response:**
```json
{
  "participantsCount": 1,
  "participants": [
    {
      "email": "test@zoho.com",
      "role": "presenter",
      "joinTime": 1693903804737,
      "leaveTime": 1693903887527,
      "duration": 82790,
      "inAndOutTime": "02:20 PM - 02:21 PM",
      "source": "web",
      "participantAvatar": "https://contacts.zoho.com/file?...",
      "memberId": "438180043542297021"
    }
  ]
}
```

---

#### Required OAuth Scopes Summary

| Scope | Purpose |
|-------|---------|
| `ZohoMeeting.manageOrg.READ` | Get user details & zsoid |
| `ZohoMeeting.meeting.READ` | Get meeting details & participant report |
| `ZohoMeeting.recording.READ` | List and get recording metadata |
| `ZohoMeeting.meetinguds.READ` | Download recordings |
| `ZohoFiles.files.READ` | Download recordings & transcripts |

**Combined scope string for Auth0:**
```
ZohoMeeting.manageOrg.READ,ZohoMeeting.meeting.READ,ZohoMeeting.recording.READ,ZohoMeeting.meetinguds.READ,ZohoFiles.files.READ
```

---

#### Integration Flow Summary

```
1. Auth: Get access token via OAuth 2.0
         ↓
2. Get zsoid: GET /api/v2/user.json
         ↓
3. Poll recordings: GET /meeting/api/v2/{zsoid}/recordings.json
         ↓
4. For each new recording:
   a. Get meeting details: GET /api/v2/{zsoid}/sessions/{meetingKey}.json
   b. Get participants: GET /api/v2/{zsoid}/participant/{meetingKey}.json
   c. Download video: Use downloadUrl from recording (pre-signed URL)
   d. Download transcript (if isTranscriptGenerated=true): Use transcriptionDownloadUrl
         ↓
5. Create upstream_meeting in Sybill with:
   - Video file (uploaded to S3)
   - Transcript (if available, else run own ASR)
   - Participant info mapped to UpstreamParticipant
   - Metadata mapped to UpstreamCallMetadata
         ↓
6. Trigger UpstreamProcessingWorkflow → CallProcessingWorkflow
```

---

### B. Existing Field Mappings Reference

**Gong:**
```python
GONG_FIELD_MAPPINGS = {"id": "remote_id", "emailAddress": "email"}
```

**Chorus:**
```python
CHORUS_FIELD_MAPPINGS = {"id": "remote_id"}
```

**Fathom:**
```python
FATHOM_FIELD_MAPPINGS = {"id": "remote_id", "email": "email"}
```

**Zoho Meetings (New):**
```python
ZOHO_MEETINGS_FIELD_MAPPINGS = {
    "meetingKey": "remote_id",
    "memberId": "remote_id",  # For participants
    "email": "email",
}
```

### C. Related Files

| File | Purpose |
|------|---------|
| `schema/upstream_meeting_providers.py` | Core schemas |
| `common/upstream_meeting_utils.py` | Provider metadata creation |
| `routers/upstream_meetings.py` | API endpoints |
| `sybill_py/runtimes/galactus/workflows/upstream_processing.py` | Processing workflow |
| `sybill_py/runtimes/galactus/activities/upstream_processing.py` | Processing activities |
| `sybill_js/auth0/login-combined-actions.js` | Auth0 post-login actions |
| `routers/organizations.py` | Link Account API |
