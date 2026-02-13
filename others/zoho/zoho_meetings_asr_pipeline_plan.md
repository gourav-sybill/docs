# Zoho Meetings ASR Pipeline Plan

## Goal

Enable end-to-end processing of Zoho Meetings recordings through the ASR pipeline (AssemblyAI) for proper diarization and timestamps, instead of using Zoho's native plain-text transcripts (which lack speaker labels and timestamps).

## Current State

The pipeline currently progresses through:
1. `ZohoMeetingsPollingWorkflow` — polls for new recordings
2. `ProcessZohoRecordingActivity` — creates UpstreamMeeting + MetadataPointer in MongoDB
3. `UpstreamProcessingWorkflow` — creates participants, skips CRM matching, starts CallProcessingWorkflow
4. `CallProcessingWorkflow` — **fails at `DownloadASRActivity`** with `ValueError: Unsupported meeting_provider: ZOHO_MEETINGS`

The recording video is hosted on Zoho's servers. The existing pipeline expects either:
- A transcript from the upstream provider (Gong/Chorus/Fathom), OR
- An audio/video file already in S3 (UPLOADED_CALL)

ZOHO_MEETINGS has neither — the video needs to be downloaded from Zoho first.

## Pipeline Flow (Proposed)

```
ZohoMeetingsPollingWorkflow
  └─ ProcessZohoRecordingActivity
       ├─ Create UpstreamMeeting in MongoDB
       ├─ Create MetadataPointer
       ├─ [NEW] Download recording from Zoho API → upload to S3
       ├─ [NEW] Add media asset path to MetadataPointer
       └─ Start UpstreamProcessingWorkflow
            └─ ... → CallProcessingWorkflow
                 ├─ CreateCoreInsightActivity  ← needs fixes (workspace_id, call_utils)
                 ├─ CheckUploadedMeetingTypeActivity → returns UPLOADED_MEDIA
                 ├─ UploadASRActivity (upstream=True) → sends audio to AssemblyAI
                 ├─ Wait for ASR webhook signal
                 ├─ DownloadASRActivity → downloads transcript from AssemblyAI
                 └─ ... rest of pipeline (NLP, FER, results, etc.)
```

---

## Changes Required

### 1. Download Recording to S3 in `ProcessZohoRecordingActivity`

**File**: `src/sybill_py/runtimes/galactus/activities/background/zoho_meetings_polling.py`
**Lines**: ~244-290 (inside `ProcessZohoRecordingActivity.run()`)

After creating the upstream meeting and metadata pointer, download the Zoho recording and upload to S3:

```python
# After metadata pointer creation...

# Download recording from Zoho and upload to S3
download_url = provider.get_recording_download_url(recording)
s3_key = f"v2/{upstream_meeting_id}/{upstream_meeting_id}.mp4"

# Stream download from Zoho → upload to S3
recording_bytes = await provider.download_recording(download_url)
await aws_utils.upload_bytes_to_s3(
    recording_bytes,
    s3_key,
    runtime.app_config.s3_data_bucket,
    content_type="video/mp4",
)

# Add media asset path to MetadataPointer
media_metadata_path = MetadataPath(
    asset_cls=AssetClass.RAW_DATA,
    asset_type=AssetType.GALLERY_RECORDING_MP4,
    path=s3_key,
)
await runtime.app_db.metadataPointers.find_one_and_update(
    {"meetingId": upstream_meeting_id},
    {"$push": {"metadataPaths": media_metadata_path.to_mongo()}},
)
```

**New method needed on `ZohoMeetingsClient`**: `download_recording(url: str) -> bytes`
- Streams the recording file from the Zoho download URL
- Uses the OAuth token for authentication
- Returns the raw bytes

**File**: `src/sybill_py/components/upstream_providers/zoho_meetings/client.py`

```python
async def download_recording(self, download_url: str) -> bytes:
    """Download recording file from Zoho."""
    response = await self._client.get(
        download_url,
        headers=self._auth_headers(),
        follow_redirects=True,
    )
    self._handle_response(response, "download_recording")
    return response.content
```

**Alternative (streaming for large files)**: Use `httpx` streaming to avoid loading entire video into memory. Write to a temp file, then upload to S3 using multipart upload.

---

### 2. Return `UPLOADED_MEDIA` for ZOHO_MEETINGS

**File**: `src/sybill_py/runtimes/galactus/activities/call_processing.py`
**Lines**: 3970-3978 (inside `CheckUploadedMeetingTypeActivity.run()`)

Add ZOHO_MEETINGS to the UPLOADED_MEDIA branch:

```python
elif provider == MeetingProviders.UPLOADED_CALL:
    upload_type = MeetingUploadType.UPLOADED_MEDIA
elif provider == MeetingProviders.SDK_UPLOAD:
    upload_type = MeetingUploadType.UPLOADED_MEDIA
elif provider == MeetingProviders.ZOHO_MEETINGS:        # NEW
    upload_type = MeetingUploadType.UPLOADED_MEDIA       # NEW
else:
    upload_type = MeetingUploadType.NOT_UPLOADED
```

This ensures the CallProcessingWorkflow routes ZOHO_MEETINGS through the ASR upload → wait for signal → download ASR path (same as uploaded calls).

---

### 3. Handle ZOHO_MEETINGS in `DownloadASRActivity`

**File**: `src/sybill_py/runtimes/galactus/activities/call_processing.py`
**Lines**: 1236-1293 (inside the `else:` branch for upstream meetings)

Add a ZOHO_MEETINGS branch before the final `else: raise ValueError`:

```python
elif upstream_meeting.provider == MeetingProviders.ZOHO_MEETINGS:
    sentences_obj = await asr_client.get_transcript(transcript_id)
    if sentences_obj is not None:
        if ASRVendors.ASSEMBLY == asr_vendor:
            transcript = RawTranscript.from_assembly_ai(
                sentences_obj, input_.meeting_id, utils.get_current_ts()
            )
        elif ASRVendors.GLADIA == asr_vendor:
            transcript = RawTranscript.from_gladia(
                sentences_obj, input_.meeting_id, utils.get_current_ts()
            )
        if transcript is not None:
            # Create participants from ASR diarization (same as UPLOADED_CALL)
            upstream_participants, unresolved_speaker_ids = (
                upstream_meeting_utils.create_participants_from_transcript(transcript)
            )
            updated_metadata = UpstreamCallMetadata(
                participants=upstream_participants,
                unresolved_speaker_ids=unresolved_speaker_ids,
                **upstream_meeting.metadata.model_dump(
                    exclude={"participants", "unresolved_speaker_ids"}
                ),
            )
            upstream_meeting = await upstream_meeting_client.patch_call_metadata(
                updated_metadata
            )
    asr_vendor = transcript.vendor
    await asr_client.aclose()
```

This mirrors the UPLOADED_CALL handling (lines 1236-1291):
- Gets transcript from ASR vendor
- Parses it into RawTranscript
- Creates participants from transcript diarization (speaker A, B, C → participant objects)
- Patches the upstream meeting metadata with the ASR-derived participants

---

### 4. Fix `create_core_insights_from_upstream_meeting` in `call_utils.py`

**File**: `src/common/call_utils.py`
**Lines**: 902-916

Add ZOHO_MEETINGS to the list of providers that use the simple path (like UPLOADED_CALL):

```python
elif upstream_meeting.provider in [
    MeetingProviders.MINDTICKLE,
    MeetingProviders.UPLOADED_CALL,
    MeetingProviders.UPLOADED_TRANSCRIPT,
    MeetingProviders.SDK_UPLOAD,
    MeetingProviders.AVOMA,
    MeetingProviders.FATHOM,
    MeetingProviders.ZOHO_MEETINGS,        # NEW
]:
    is_private = False
    event_id = None
    conf_provider = None
    user_owner_prefix = None
    additional_party_info = []
```

---

### 5. Fix `_create_core_insight_from_upstream_meeting` workspace_id call

**File**: `src/sybill_py/runtimes/galactus/activities/call_processing.py`
**Lines**: 639-644

The `get_remote_org_id` call uses `IntegrationType.MEETING_PROVIDER` which doesn't exist for ZOHO_MEETINGS. Add a guard:

```python
if upstream_meeting.provider == MeetingProviders.ZOHO_MEETINGS:
    workspace_id = None
else:
    workspace_id = await org_client.get_remote_org_id(
        upstream_meeting.org_id,
        IntegrationType.MEETING_PROVIDER,
        meeting_provider=upstream_meeting.provider,
    )
```

---

### 6. Handle `processing_metadata` being None for `remote_crm_owner_id`

**File**: `src/sybill_py/runtimes/galactus/activities/call_processing.py`
**Line**: 645

The `processing_metadata` may be None for newly created ZOHO_MEETINGS upstream meetings (we already fixed the workflow ID check but this is a separate access):

```python
remote_crm_owner_id = (
    upstream_meeting.processing_metadata.remote_crm_owner_id
    if upstream_meeting.processing_metadata
    else None
)
```

---

## Files Modified (Summary)

| File | Change | Reason |
|------|--------|--------|
| `activities/background/zoho_meetings_polling.py` | Download recording → S3, add media path to MetadataPointer | Recording must be in S3 for ASR |
| `upstream_providers/zoho_meetings/client.py` | Add `download_recording()` method | Download video from Zoho API |
| `activities/call_processing.py:3970` | `CheckUploadedMeetingTypeActivity` returns UPLOADED_MEDIA for ZOHO_MEETINGS | Route through ASR pipeline |
| `activities/call_processing.py:1236` | `DownloadASRActivity` ZOHO_MEETINGS branch | Parse ASR transcript |
| `activities/call_processing.py:639` | `_create_core_insight_from_upstream_meeting` workspace_id guard | Avoid IntegrationType.MEETING_PROVIDER lookup |
| `activities/call_processing.py:645` | `processing_metadata` None guard for `remote_crm_owner_id` | Avoid NoneType attribute error |
| `common/call_utils.py:902` | Add ZOHO_MEETINGS to simple provider list | Avoid ValueError in core insight creation |

## Considerations

### Large File Downloads
- Zoho recordings can be hundreds of MB
- Consider streaming download + multipart S3 upload via temp file instead of loading full bytes into memory
- The `ProcessZohoRecordingActivity` runs as a Temporal activity with heartbeat — long downloads should send heartbeats

### ASR Webhook
- `UploadASRActivity` submits audio to AssemblyAI with a webhook URL
- The webhook callback is handled by the existing ASR signal infrastructure
- The `CallProcessingWorkflow` waits for the ASR signal before proceeding to `DownloadASRActivity`
- No changes needed for the webhook/signal flow

### Existing Fixes (Already Applied)
These fixes from the current session are prerequisites and already in place:
- `upstream_processing.py`: ZOHO_MEETINGS workspace_id exclusion in `DoCrmMatchingActivity`
- `upstream_processing.py`: Skip CRM matching for ZOHO_MEETINGS
- `upstream_processing.py`: `should_ignore_recorder_prefs = True` for ZOHO_MEETINGS
- `upstream_processing.py`: `processing_metadata` None guard for workflow ID check
- `zoho_meetings_polling.py`: Use `fetch_all_recordings()` instead of broken single-recording endpoint
- `zoho_meetings_polling.py`: Create MetadataPointer after upstream meeting insert

### What About Zoho's Native Transcript?
Zoho provides a plain-text transcript (`transcription_download_url`) but it lacks:
- Per-sentence timestamps
- Speaker diarization (who said what)
- Structured format

The ASR route via AssemblyAI provides all of these, making it the preferred approach. The native transcript could be used as a fallback if ASR fails, but that's a future enhancement.

## Testing Plan

1. Restart Temporal worker to pick up code changes
2. Clean up existing upstream meetings / metadata pointers for test org
3. Start `ZohoMeetingsPollingWorkflow` on localhost Temporal
4. Verify the recording is downloaded to S3
5. Verify `UploadASRActivity` submits to AssemblyAI successfully
6. Wait for ASR webhook signal
7. Verify `DownloadASRActivity` downloads and parses transcript
8. Verify rest of pipeline completes (ProcessTranscript, GenerateResults, etc.)
