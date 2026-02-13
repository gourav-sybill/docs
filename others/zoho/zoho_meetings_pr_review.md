# PR #2266 Review: Zoho Meetings Integration

## Architecture Decision: Org-Level Integration

**Decision:** `ZOHO_MEETINGS_TEAM` stays as an **org-level integration**. No migration to user-level needed.

**Rationale:** All upstream meeting providers in the codebase (Gong, Chorus, Fathom) are org-level. Zoho Meetings fetches org-wide data scoped to a ZSOID, which follows the same pattern. The entire upstream processing pipeline is built around org-level context.

**Implication:** When User A in Org X connects Zoho Meetings, all recordings from that Zoho org appear for every member of Org X — even members who did not connect the integration themselves. This is identical to how Gong works.

---

## Table of Contents

- [End-to-End Connection Flow](#end-to-end-connection-flow)
- [PR Review: Comparison with Gong and Other Upstream Providers](#pr-review-comparison-with-gong-and-other-upstream-providers)
- [Issues Found](#issues-found)
- [Things Done Correctly](#things-done-correctly)

---

## End-to-End Connection Flow

Step-by-step trace from the moment a user clicks "Connect Zoho Meetings" on the dashboard to the backend completing the integration setup.

### Step 1: User Clicks "Connect Zoho Meetings" on the Dashboard

The frontend (flamingo-dashboard) presents a "Connect Zoho Meetings" button. When clicked, the user selects their **data center region** (US, EU, IN, AU, UK, CA, JP, SA). The frontend then redirects the user to Auth0's Universal Login page with these query parameters:

```
?syc_org_id=<org_uuid>
&meeting_provider=ZOHO_MEETINGS
&connection=zoho-meetings-oauth2        (or region variant like "eu-zoho-meetings-oauth2")
&force_reconnect=false
```

The `connection` value is region-specific -- e.g., `eu-zoho-meetings-oauth2` for EU, `in-zoho-meetings-oauth2` for India, etc. These map to `ConnectionNames` entries defined in `src/schema/main.py:1774-1781`.

---

### Step 2: Auth0 Redirects to Zoho OAuth2 Consent Screen

Auth0 sees the connection name (e.g., `zoho-meetings-oauth2`), looks up the configured Auth0 Social Connection for that connection, and redirects the user to Zoho's OAuth2 authorization endpoint for the appropriate region. The scopes requested are defined in `src/schema/integrations.py:496-503`:

```python
ZOHO_MEETINGS_SCOPES = [
    "ZohoMeeting.manageOrg.READ",
    "ZohoMeeting.meeting.READ",
    "ZohoMeeting.recording.READ",
    "ZohoMeeting.meetinguds.READ",
    "ZohoFiles.files.READ",
    "AaaServer.profile.READ",
]
```

The user sees Zoho's consent screen and grants access. Zoho redirects back to Auth0 with an authorization code. Auth0 exchanges it for access + refresh tokens and creates a new Auth0 user for this identity.

---

### Step 3: Auth0 Post-Login Action Fires (`login-combined-actions.js`)

After the OAuth completes, Auth0 runs the post-login action. The flow enters `onExecutePostLogin` at line 1586.

#### 3a. Integration Flow Branch

Since the URL had `syc_org_id` in the query, it hits the integration flow branch (`login-combined-actions.js:1608-1613`):

```javascript
if ("syc_user_id" in event.request.query || "syc_org_id" in event.request.query) {
    await onFirstTimeLogin(event, api, propsToBePersisted)
    await copyRefreshTokens(event, api, propsToBePersisted)
    await linkOnBackend(event, api, propsToBePersisted)
    return;
}
```

#### 3b. `storeTokens()` (line 1605)

Runs first. Encrypts the refresh token from Zoho and stores it in the Auth0 user's `app_metadata.refresh_tokens` under the key `oauth2|zoho-meetings-oauth2`.

#### 3c. `onFirstTimeLogin()`

Sends a Slack notification about the integration attempt.

#### 3d. `copyRefreshTokens()`

Finds the existing Auth0 user for the org (via `syc_org_id`) and copies the encrypted refresh tokens to it. This is how the primary Auth0 user (the Sybill meeting provider user) gets the Zoho refresh token merged into it.

#### 3e. `linkOnBackend()` (line 1123)

This is the key function. It:

1. Gets the identity: `provider: "oauth2"`, `connection: "zoho-meetings-oauth2"` (or region variant)
2. Matches the connection via `ZOHO_MEETINGS_CONNECTIONS.includes(identity.connection)` (line 1209)
3. Since `syc_org_id` exists, sets:
   ```javascript
   integrationType = "ZOHO_MEETINGS_TEAM";
   url = orgUrl;  // -> /organizations/{orgId}/linkAccount/{auth0UserId}?meetingProvider=...
   authKey = event.secrets.ORG_KEY;
   dcRegion = getZohoMeetingsDCRegionFromConnection(identity.connection);
   ```
4. The `getZohoMeetingsDCRegionFromConnection()` helper (line 542) maps the connection name to the dc region string using `ZOHO_MEETINGS_REGION_MAP` (lines 514-523), e.g., `"eu-zoho-meetings-oauth2"` -> `"eu"`.
5. Makes a POST to the backend:
   ```
   POST /organizations/{orgId}/linkAccount/{auth0UserId}?meetingProvider=ZOHO_MEETINGS&forceReconnect=false
   Body: {
       integrationType: "ZOHO_MEETINGS_TEAM",
       identity: { provider: "oauth2", userId: "...", connection: "zoho-meetings-oauth2" },
       email: "user@company.com",
       dcRegion: "us"
   }
   Authorization: Bearer <ORG_KEY>
   ```

---

### Step 4: Backend Router -- `link_org_integration_to_account` (`organizations.py:717`)

The POST hits `POST /{orgId}/linkAccount/{auth0UserId}` which maps to the `link_org_integration_to_account` function. Here's what happens step by step:

#### 4a. Cache Invalidation (line 729)

```python
await cache_utils.invalidate_cache_for_auth0_user_id_safely(auth0_user_id)
```

#### 4b. Fetch Org Integration Info (line 732)

```python
org_with_integration_info = await integration_utils.fetch_org_integration_info(
    org_id, request.app.mongodb, meeting_provider  # meeting_provider = ZOHO_MEETINGS
)
```

This queries MongoDB for the org document, looking specifically for a `ZOHO_MEETINGS_TEAM` integration (via `integration_utils.py:2349-2350`). On first connect, `zoho_meetings_integration_team` will be `None` on the result.

#### 4c. Resolve Connection Config (lines 795-803)

```python
elif link_request.identity.connection in ZOHO_MEETINGS_CONNECTIONS:
    connection_id = request.app.app_config.auth0_ref.zoho_meetings_connection_ids[link_request.dc_region]
    authorization_url = OAuth2AsyncClient.ZOHO_AUTH_URL_V2(link_request.dc_region)
    scopes = ZOHO_MEETINGS_SCOPES
    connection_name = ConnectionNames(link_request.identity.connection)
```

Maps the `dc_region` to the Auth0 connection ID and sets the Zoho authorization URL for that region.

#### 4d. Fetch OAuth Credentials (line 807)

```python
creds = await OAuth2CredentialsManager.get_oauth2_cred_by_auth0_user_id(
    auth0_user_id=auth0_user_id,
    auth0_connection_id=connection_id,
    scopes=scopes,
    auth_url=authorization_url,
    connection_name=connection_name,
    refresh_token_key=None,
)
```

Retrieves the OAuth2 access token by decrypting the refresh token from Auth0 `app_metadata` and exchanging it with Zoho.

#### 4e. Create Integration Object (lines 842-845)

```python
elif link_request.identity.connection in ZOHO_MEETINGS_CONNECTIONS:
    integration_obj = integration_utils.create_zoho_meetings_integration(
        org_id, link_request.identity, link_request.dc_region
    )
```

Calls `create_zoho_meetings_integration` in `integration_utils.py:2249`, which creates a `ZohoMeetingsIntegrationInfoTeam` with:

- `integration_type = ZOHO_MEETINGS_TEAM`
- `state_events = [INITIALIZING]`
- `identity` = the Auth0 identity
- `dc_region` = the Zoho data center region
- `zsoid = None` (will be fetched on first API call during polling)

#### 4f. Check for Existing Active Integration (lines 884-889)

```python
elif (
    link_request.identity.connection in ZOHO_MEETINGS_CONNECTIONS
    and org_with_integration_info.zoho_meetings_integration_team is not None
    and org_with_integration_info.zoho_meetings_integration_team.is_current
):
    existing_active_integration = True
```

If there's already an active Zoho Meetings integration and `force_reconnect=false`, it rejects the request with 400.

#### 4g. Persist Integration to MongoDB (lines 906-931)

First attempt -- push new integration if no `ZOHO_MEETINGS_TEAM` exists:

```python
await mongo["organizations"].update_one(
    {MONGODB_ID_FIELD: org_id, "integrations.integrationType": {"$ne": "ZOHO_MEETINGS_TEAM"}},
    {"$push": {"integrations": integration_obj.to_mongo()}},
)
```

If the integration type already exists (reconnect scenario), it replaces the existing one in-place with `$set`.

#### 4h. Link Auth0 Accounts (lines 946-953)

```python
await Auth0AsyncClient().link_user_account_by_auth0_user_id(
    meeting_provider_integration.auth0_user_id,  # primary Sybill user
    auth0_user_id,                                # new Zoho Meetings user
    link_request.identity.provider,               # "oauth2"
)
```

This links the new Zoho Meetings Auth0 identity into the primary Sybill Auth0 user. After this, the primary user has the Zoho refresh token as a linked identity, which is how `ZohoMeetingsCredentialsManager` later retrieves it during polling.

#### 4i. Post-Install: Initialize Integration (lines 1049-1064)

```python
elif link_request.identity.connection in ZOHO_MEETINGS_CONNECTIONS:
    initialized_event = IntegrationStateEvent(
        id=uuid4(), event_type=IntegrationEventType.INITIALIZED, timestamp=...
    )
    zoho_meetings_preference = UpstreamMeetingPreferencesPerProvider(
        provider=MeetingProviders.ZOHO_MEETINGS, remote_org_id=str(org_id)
    )
    await request.app.mongodb["organizations"].update_one(
        {MONGODB_ID_FIELD: org_id, f"integrations.{MONGODB_ID_FIELD}": integration_obj.id},
        {
            "$push": {"integrations.$.stateEvents": initialized_event.to_mongo()},
            "$addToSet": {
                "accountInfo.preferences.upstreamMeetingPreferences": zoho_meetings_preference.to_mongo()
            },
        },
    )
```

Two things happen atomically:

1. Pushes an `INITIALIZED` state event onto the integration (state goes from `INITIALIZING` -> `INITIALIZED`)
2. Adds an `UpstreamMeetingPreferencesPerProvider` entry to `accountInfo.preferences.upstreamMeetingPreferences` -- this is used later by the upstream processing workflow to resolve system identity prefixes

#### 4j. Audit Log + Response (lines 1066-1076)

```python
log_utils.get_and_publish_audit_log_org(...)
return {"sycOrgId": org_id}
```

---

### Step 5: Auth0 Action Returns to Frontend

Back in `linkOnBackend()` in `login-combined-actions.js`, the response is received (line 1253). A Slack notification is sent:

```
"ZOHO_MEETINGS_TEAM Integration via zoho-meetings-oauth2 Created at backend: User Name (user@company.com) (org_id)"
```

Auth0 completes the login flow and redirects the user back to the Sybill dashboard. The integration is now connected.

---

### Flow Diagram

```
User clicks "Connect Zoho Meetings" (selects region: e.g., US)
  |
  v
Frontend redirects to Auth0 with ?syc_org_id=...&connection=zoho-meetings-oauth2
  |
  v
Auth0 redirects to Zoho OAuth2 (region-specific URL, with ZOHO_MEETINGS_SCOPES)
  |
  v
User grants consent on Zoho -> Auth0 gets access+refresh tokens
  |
  v
Auth0 Post-Login Action fires (login-combined-actions.js)
  |-- storeTokens()       -> encrypts refresh token into app_metadata
  |-- copyRefreshTokens() -> copies tokens to primary Auth0 user
  +-- linkOnBackend()     -> POST /organizations/{orgId}/linkAccount/{auth0UserId}
       |                     body: { integrationType: "ZOHO_MEETINGS_TEAM", dcRegion: "us", identity: {...} }
       v
  Backend: link_org_integration_to_account()
    |-- Resolve Auth0 connection_id for region
    |-- Fetch OAuth2 creds via OAuth2CredentialsManager
    |-- Create ZohoMeetingsIntegrationInfoTeam (state: INITIALIZING)
    |-- Check no existing active integration
    |-- Persist integration to MongoDB organizations collection
    |-- Link Auth0 identities (merge Zoho identity into primary user)
    |-- Push INITIALIZED state event
    |-- Add UpstreamMeetingPreferencesPerProvider to org preferences
    +-- Return { sycOrgId: org_id }
```

After this, the `ZohoMeetingsPollingWorkflow` (running as a Temporal background workflow) will pick up this org on its next poll cycle because `list_org_ids_with_zoho_meetings_integration` will now return it.

---

## PR Review: Comparison with Gong and Other Upstream Providers

### Architecture Overview

The PR implements a complete Zoho Meetings integration following the upstream provider pattern (similar to Gong, Chorus, Fathom). Key differences from Gong:

| Aspect | Gong | Zoho Meetings |
|--------|------|---------------|
| **Auth Type** | Basic Auth (access_key + access_secret) | OAuth2 (multi-region) |
| **Multi-Region** | No | Yes (8 regions with separate Auth0 connections) |
| **Workspace Selection** | Yes (user selects from available workspaces) | No (uses org_id as remote_org_id) |
| **User Syncing** | Yes (per-user system identities) | No (org-level only) |
| **CRM Matching** | Yes | No (explicitly disabled) |
| **Recorder Preferences** | Yes | Always ignored |
| **Recording Ingestion** | Webhook-based | Polling (Temporal workflow) |
| **Connection Verification** | `verify_gong_connection()` with healthcheck | OAuth2 flow (no explicit verification) |
| **Health Monitoring** | No | Yes (`ConnectionHealthMixin`) |
| **Org ID** | Known at connection time (workspace selection) | Fetched on first API call (`zsoid`) |

---

## Issues Found

### CRITICAL: No Unlink/Disconnect Flow for `ZOHO_MEETINGS_TEAM`

**Location:** `src/routers/organizations.py:1091-1153`

**Impact:** There is no way to disconnect a Zoho Meetings integration once linked.

#### How Gong/Chorus Unlinking Works Today

A Gong/Chorus disconnect is triggered via either DELETE endpoint:
- `DELETE /{orgId}/linkAccount/m2m` (m2m auth, used by internal tools) — line 1157
- `DELETE /{accCN}/linkAccount` (user auth, used by dashboard) — line 1177

Both call `_unlink_team_integration` with:
```
integration_type = IntegrationType.MEETING_PROVIDER
meeting_provider = MeetingProviders.GONG  (or CHORUS / FATHOM)
```

The function performs 5 steps in order:

1. **Fetch the integration** (line 1105-1106) — `get_team_integration_by_org_id(org_id, MEETING_PROVIDER, meeting_provider=GONG)` finds the Gong integration in `organizations.integrations[]`.

2. **Unlink Auth0 identity** (line 1117-1122) — Guarded by `relevant_integration.integration_type != IntegrationType.MEETING_PROVIDER`. For Gong this is `False` (Gong uses API key auth, not OAuth2 linked identities), so Auth0 unlinking is **skipped**. For CRM/Slack/Zoom which use OAuth2, this branch unlinks the Auth0 identity from the primary user.

3. **Invalidate cache** (line 1124-1125) — Clears cached auth data for the primary Auth0 user.

4. **Append CANCELLED state event** (line 1127-1131) — Pushes `IntegrationEventType.CANCELLED` onto `stateEvents`. This makes `is_active` return `False`, so the polling workflow stops picking up this org.

5. **Clean up preferences** (line 1133-1136) — Since `MEETING_PROVIDER` is not in `[CRM_TEAM, SLACK_TEAM]`, it calls `on_team_integration_cancelled` (`preferences.py:151-174`), which:
   - `$unset`s `accountInfo.preferences.upstreamMeetingPreferences` **entirely** (removes all providers' preferences)
   - `$pull`s all `systemIdentities` from users whose principal starts with the provider name (e.g. removes all `GONG-*` system identities)

6. **Audit log + Slack notification** (lines 1137-1152)

#### What Blocks Zoho Meetings Unlinking

Two type restrictions prevent calling `_unlink_team_integration` for Zoho Meetings:

1. The `integration_type` parameter (line 1093-1099) is a `Literal` that excludes `ZOHO_MEETINGS_TEAM`:
   ```python
   integration_type: Literal[
       IntegrationType.CRM_TEAM,
       IntegrationType.SLACK_TEAM,
       IntegrationType.MEETING_PROVIDER,
       IntegrationType.VIDEO_CONFERENCE_TEAM,
       IntegrationType.DIALER_TEAM,
   ]
   ```

2. The `meeting_provider` parameter (line 1102) excludes `ZOHO_MEETINGS`:
   ```python
   meeting_provider: Literal[MeetingProviders.GONG, MeetingProviders.CHORUS, MeetingProviders.FATHOM] | None
   ```

3. Both DELETE endpoint signatures have the same restriction on `meeting_provider` (lines 1162, 1180).

#### What Zoho Meetings Unlinking Would Require

The core `_unlink_team_integration` flow is reusable. Key differences from Gong:

**Identical steps (work as-is):**
- Fetch the integration (using `ZOHO_MEETINGS_TEAM` type instead of `MEETING_PROVIDER` + discriminator)
- Invalidate cache
- Append CANCELLED state event
- Audit log + Slack notification

**Auth0 identity unlinking — works correctly:**
Zoho Meetings uses OAuth2 (unlike Gong's API key auth). The guard at line 1117 checks `integration_type != IntegrationType.MEETING_PROVIDER`. Since `ZOHO_MEETINGS_TEAM != MEETING_PROVIDER`, the Auth0 unlink branch **would fire correctly** — no change needed here.

**Preference cleanup — needs a new branch:**
`on_team_integration_cancelled` (`preferences.py:129`) has no `ZOHO_MEETINGS_TEAM` branch. The existing `MEETING_PROVIDER` branch (line 151) `$unset`s the **entire** `upstreamMeetingPreferences` array, which would also remove preferences for other providers (e.g. Gong) if both are connected simultaneously. Additionally, it removes system identities matching the provider prefix — but Zoho Meetings doesn't create system identities (`should_ignore_recorder_prefs=True`), so that step is a no-op.

#### Changes Needed to Implement

1. **`_unlink_team_integration`** (`organizations.py:1093-1099`): Add `IntegrationType.ZOHO_MEETINGS_TEAM` to the `Literal` type.

2. **`_unlink_team_integration`** (`organizations.py:1102`): Add `MeetingProviders.ZOHO_MEETINGS` to the `meeting_provider` `Literal` type.

3. **`unlink_org_integration_from_account_m2m`** (`organizations.py:1162`): Add `MeetingProviders.ZOHO_MEETINGS` to the endpoint's `meeting_provider` query param type.

4. **`unlink_org_integration_from_account`** (`organizations.py:1177-1191`): This endpoint currently does not accept a `meeting_provider` param — it may need one added, or the unlink logic needs to handle `ZOHO_MEETINGS_TEAM` without a `meeting_provider` discriminator (since `ZOHO_MEETINGS_TEAM` is its own integration type, not a variant of `MEETING_PROVIDER`).

5. **`on_team_integration_cancelled`** (`preferences.py:129`): Add a `ZOHO_MEETINGS_TEAM` branch that uses `$pull` to remove **only** the `ZOHO_MEETINGS` entry from `upstreamMeetingPreferences`, instead of `$unset`ing the whole array. System identity cleanup can be skipped (no system identities are created for Zoho Meetings).

6. **`_get_team_integration`** (`integration_utils.py:255`): Has an assertion `not ((meeting_provider is not None) ^ (type_hint == IntegrationType.MEETING_PROVIDER))`. When calling with `type_hint=ZOHO_MEETINGS_TEAM` and `meeting_provider=None`, this passes (both sides are `False`). But if `meeting_provider=ZOHO_MEETINGS` is passed alongside `type_hint=ZOHO_MEETINGS_TEAM`, the assertion **fails** (`True ^ False`). So the unlink call should pass `meeting_provider=None` when using `ZOHO_MEETINGS_TEAM` as the type hint, since the type itself is already unique (no discriminator needed).

---

### ~~MEDIUM: `remote_org_id` Uses Sybill `org_id` Instead of Zoho `zsoid`~~ — RESOLVED

**Status:** Not an issue. The current code at `src/routers/organizations.py:1066` correctly uses the Zoho `zsoid`:

```python
zoho_meetings_preference = UpstreamMeetingPreferencesPerProvider(
    provider=MeetingProviders.ZOHO_MEETINGS, remote_org_id=str(zoho_meetings_zsoid)
)
```

The `zsoid` is fetched from the Zoho API during linking (lines 822-829) via `ZohoMeetingsClient.get_user_details()` and stored in the integration record. The `system_ident_prefix` in `preferences.py:633-634` correctly generates `ZOHO_MEETINGS-{zsoid}`, consistent with how Gong uses its workspace ID.

---

## Things Done Correctly

### Auth0 login-combined-actions.js

Follows the exact Zoho CRM pattern:

- `ZOHO_MEETINGS_CONNECTIONS` list with 8 regions (matches Python-side)
- `ZOHO_MEETINGS_REGION_MAP` mapping connection -> dc_region
- `getProviderKey()` correctly returns `"zoho-meetings-oauth2"` for Zoho Meetings connections
- `linkOnBackend()` correctly handles `ZOHO_MEETINGS_TEAM` with `dcRegion` extraction, org-level linking, `ORG_KEY` auth

### No Calendar/Email Scope Validation for Zoho Meetings

Correct. Zoho Meetings is an org-level integration (not user-level calendar/email), so it doesn't need calendar scope validation in the Auth0 login flow. `MANDATORY_SCOPES` and `getScopesByAccessToken` intentionally have no handler for `"zoho-meetings-oauth2"`.

### `create_ci_admin_account` Doesn't Include Zoho Meetings

Correct. Gong/Chorus/Fathom use API key/basic auth via CI admin accounts. Zoho Meetings uses OAuth2 linking, so the different entry point (`linkAccount/{auth0UserId}`) is appropriate.

### Schema Additions Are Consistent

- `ZOHO_MEETINGS_REMOTE_ID_PREFIX = "ZOHO_MEETINGS-"` follows the pattern of `GONG_REMOTE_ID_PREFIX`, `CHORUS_REMOTE_ID_PREFIX`, `FATHOM_REMOTE_ID_PREFIX`
- `FeatureFlags.ZOHO_MEETINGS_INTEGRATION` -- proper feature gating
- 8-region `ConnectionNames` -- matches Zoho CRM pattern exactly
- `IntegrationType.ZOHO_MEETINGS_TEAM` -- follows existing team-level integration pattern
- `MeetingProviders.ZOHO_MEETINGS` -- properly added to enum
- `UpstreamMeetingPreferencesPerProvider` -- correctly extended with ZOHO_MEETINGS

### `dashboard_url` Validator Relaxation

`upstream_meeting_providers.py:83` -- Correct. Zoho Meetings doesn't have per-call dashboard URLs, same pattern as uploaded calls.

### CRM Matching Correctly Skipped

`upstream_processing.py:290` -- `provider != MeetingProviders.ZOHO_MEETINGS` prevents CRM matching, and `workspace_id` check (line 283) exempts Zoho Meetings. Logical since there's no direct Gong-style user-to-CRM mapping.

### `should_ignore_recorder_prefs=True`

`upstream_processing.py:332-335` -- Correct. Zoho Meetings doesn't sync individual users with system identities, so all recordings should be processed without preference filtering.

### Integration Info Classes

Well-structured:

- `BaseZohoMeetingsIntegrationInfo` correctly inherits `ConnectionHealthMixin` (enables health error tracking)
- `dc_region` field for multi-region support
- `zsoid` field for Zoho org ID caching
- Discriminated union with `integration_type` Literal works correctly

### Polling Workflow Pattern

Gong does NOT have a polling workflow (it uses webhooks/different mechanism). Zoho Meetings' Temporal polling workflow is a valid approach for a platform without webhook support.

### Provider Abstraction Layer

`base.py`, `registry.py` in `upstream_providers/` -- New infrastructure used only by Zoho Meetings. Existing providers (Gong, Chorus, Fathom) still use direct utility functions in `upstream_meeting_utils`. This is a good foundation for future provider migrations.

### `OrgWithIntegrationInfo.zoho_meetings_integration_team`

Correctly added to schema (`integrations.py:827`), extraction logic (`integration.py:243`), and construction (`integration.py:258`).

---

## Summary Table

| Severity | Issue | Location | Status |
|----------|-------|----------|--------|
| **CRITICAL** | No unlink/disconnect flow for `ZOHO_MEETINGS_TEAM` | `organizations.py:1091-1153`, `preferences.py:129` | Open — see detailed analysis above |
| ~~**MEDIUM**~~ | ~~`remote_org_id` uses Sybill org_id, not Zoho zsoid~~ | `organizations.py:1066` | Resolved — code correctly uses `zoho_meetings_zsoid` |
