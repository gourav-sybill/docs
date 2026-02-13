# Zoho Meetings PR Split Plan

## Overview

The `zoho-meetings-schema-changes` branch contains 14 commits that can be cleanly separated into two PRs: one for the backend router linking flow and one for the Temporal polling workflow.

## Commit Categorization

| # | Commit | Message | Category |
|---|--------|---------|----------|
| 1 | `fddf95e72` | Schema definitions | Schema/Foundation |
| 2 | `203e8c766` | Base class + registry | Schema/Foundation |
| 3 | `7bc92694b` | API client + schemas | Schema/Foundation |
| 4 | `8f174bc29` | Manual test script | Schema/Foundation |
| 5 | `efab58eef` | Unit tests for module | Schema/Foundation |
| 6 | `b2ad16f55` | Auth0 / router linking (Phase 3) | Backend Router |
| 7 | `496d67095` | ZohoMeetingsProvider (Phase 4) | Temporal/Provider |
| 8 | `71e9326b9` | Polling workflow + DAL (Phase 5) | Temporal/Provider |
| 9 | `db650d9c4` | Polling integration tests (Phase 6.3) | Temporal/Provider |
| 10 | `ca6763c02` | E2E fixes (mixed) | **BOTH** |
| 11 | `62c015346` | Fix polling test failures | Temporal/Provider |
| 12 | `feafee9e6` | Feature flag gating | Temporal (tiny schema touch) |
| 13 | `1924503e3` | DAL query e2e tests | Temporal/Provider |
| 14 | `6cd6a90f4` | Router linking tests | Backend Router |

## PR 1: Schema + Foundation + Backend Router Linking

### Commits

| Commit | Description |
|--------|-------------|
| `fddf95e72` | Schema definitions (enums, preferences, integration types) |
| `203e8c766` | Base upstream provider class + registry |
| `7bc92694b` | Zoho Meetings API client + Pydantic schemas |
| `8f174bc29` | Manual test script for API client |
| `efab58eef` | Unit tests for client, schema, exceptions |
| `b2ad16f55` | Auth0 credentials manager, router linking/unlinking handlers |
| **partial `ca6763c02`** | Router fixes only: `integration_utils.py`, `organizations.py`, `integration.py` |
| `6cd6a90f4` | Unit + E2E tests for router linking flow |

### Files

- `src/schema/*` (enums, preferences, upstream providers)
- `src/sybill_py/components/schema/galactus.py`
- `src/sybill_py/components/upstream_providers/` (base, registry, zoho_meetings client/schema/exceptions)
- `src/routers/integration_utils.py`, `src/routers/organizations.py`
- `src/sybill_js/auth0/login-combined-actions.js`
- `src/sybill_py/components/auth0/credentials/zoho_meetings.py`
- `src/sybill_py/components/integration/integration.py`
- `tests/routers/organizations/test_zoho_meetings_link_account*.py`
- `tests/sybill_py/components/upstream_providers/zoho_meetings/test_*.py`
- `scripts/test_zoho_client.py`

### What this PR delivers

Self-contained linking flow: a user can connect their Zoho Meetings account via Auth0, the backend stores the integration in MongoDB, and the connection can be force-reconnected or unlinked. Includes the reusable API client and all shared schemas.

---

## PR 2: Temporal Polling Workflow (depends on PR 1)

### Commits

| Commit | Description |
|--------|-------------|
| `496d67095` | ZohoMeetingsProvider implementation |
| `71e9326b9` | Polling workflow, activities, DAL queries, galactus registry |
| `db650d9c4` | Polling integration tests |
| **partial `ca6763c02`** | Temporal fixes only: `provider.py`, `zoho_meetings_polling.py`, `upstream_processing.py` |
| `62c015346` | Fix polling test failures |
| `feafee9e6` | Feature flag gating (adds `ZOHO_MEETINGS_INTEGRATION` to `FeatureFlags` enum) |
| `1924503e3` | DAL e2e tests |

### Files

- `src/sybill_py/components/upstream_providers/zoho_meetings/provider.py`
- `src/sybill_py/runtimes/galactus/activities/background/zoho_meetings_polling.py`
- `src/sybill_py/runtimes/galactus/activities/upstream_processing.py`
- `src/sybill_py/runtimes/galactus/workflows/background/zoho_meetings_polling.py`
- `src/sybill_py/runtimes/galactus/registry.py`
- `src/sybill_py/dal/queries/organizations/reader.py`
- `src/sybill_py/dal/queries/upstream_meetings/reader.py`
- `src/schema/main.py` (feature flag enum addition)
- All corresponding `tests/` files

### What this PR delivers

Background polling workflow that discovers new Zoho Meetings recordings across all linked orgs, creates UpstreamMeeting documents, and triggers the existing upstream processing pipeline. Gated behind the `ZOHO_MEETINGS_INTEGRATION` feature flag.

---

## Splitting Commit `ca6763c02`

This is the only commit that crosses both domains. It must be split during cherry-pick:

```bash
# For PR 1 branch:
git cherry-pick ca6763c02 --no-commit
git add src/routers/integration_utils.py src/routers/organizations.py src/sybill_py/components/integration/integration.py
git commit -m "fix: End-to-end fixes for Zoho Meetings router linking"

# For PR 2 branch:
git cherry-pick ca6763c02 --no-commit
git add src/sybill_py/components/upstream_providers/zoho_meetings/provider.py
git add src/sybill_py/runtimes/galactus/activities/background/zoho_meetings_polling.py
git add src/sybill_py/runtimes/galactus/activities/upstream_processing.py
git commit -m "fix: End-to-end fixes for Zoho Meetings upstream processing pipeline"
```

---

## Review Notes

### PR 1 Review Focus
- Schema field defaults removed from `ZohoMeetingsIntegrationInfoTeam` Literal fields (fixes serialization bug with `exclude_defaults=True`)
- Auth0 account linking logic (same-user skip, force reconnect)
- `integration_utils.py` changes for Zoho Meetings connection handling
- E2E tests verify real MongoDB state after linking

### PR 2 Review Focus
- Polling workflow uses `continue_as_new` for infinite polling without history bloat
- Semaphore-bounded parallelism across orgs (`parallel_orgs` config)
- Feature flag check per org before polling
- `ProcessZohoRecordingActivity` re-fetches full recording list per recording (known inefficiency, documented in temporal review doc)
- No rate limit handling in polling loop (documented in temporal review doc)
- CRM matching is intentionally skipped for Zoho Meetings (line 290 in `upstream_processing.py`)

### Related Review Docs
- [Backend Router Review](./zoho_meetings_pr_review.md)
- [Temporal Workflow Review](./zoho_meetings_temporal_review.md)

---

## Refactor: Remove ZOHO_MEETINGS_TEAM, Reuse MEETING_PROVIDER

### Rationale

`MEETING_PROVIDER` is a singleton-per-org integration type. An org uses one recording source at a time (Sybill, Gong, Chorus, Fathom). Zoho Meetings serves the same role — it replaces Sybill as the recording source, not supplements it. There is no reason for a separate `IntegrationType`.

The current `ZOHO_MEETINGS_TEAM` approach causes:
- Awkward fallback logic in `prepare_org_with_integration_info`
- A separate `zoho_meetings_integration_team` field on `OrgWithIntegrationInfo`
- Zoho Meetings routed through `_deserialize_simple_integration` instead of `_deserialize_meeting_provider_integration`
- Special-case `$ne` override in `fetch_org_integration_info`

### Changes

1. **`src/schema/integrations.py`**: Remove `ZOHO_MEETINGS_TEAM` enum value, change `ZohoMeetingsIntegrationInfoTeam.integration_type` to `Literal[IntegrationType.MEETING_PROVIDER]`, add to `TeamMeetingProviderTypeVar` and `MeetingProviderIntegrationInfo`, remove `ZohoMeetingsIntegrationTypeVar`, remove `zoho_meetings_integration_team` from `OrgWithIntegrationInfo`, add `MEETING_PROVIDER` to `LinkRequest` validator
2. **`src/sybill_py/components/integration/integration.py`**: Add `ZOHO_MEETINGS` to `_deserialize_meeting_provider_integration`, remove from `_deserialize_simple_integration`, remove fallback hack and separate extraction in `prepare_org_with_integration_info`
3. **`src/routers/integration_utils.py`**: Use `MEETING_PROVIDER` in `create_zoho_meetings_integration`, remove Zoho override in `fetch_org_integration_info`
4. **`src/routers/organizations.py`**: Change `ZOHO_MEETINGS_TEAM` to `MEETING_PROVIDER` in post-link handler, remove separate `zoho_meetings_integration_team` force_reconnect check
5. **`src/sybill_py/dal/queries/organizations/reader.py`**: Query by `integrationType=MEETING_PROVIDER` AND `provider=ZOHO_MEETINGS`
6. **`src/sybill_js/auth0/login-combined-actions.js`**: Change `"ZOHO_MEETINGS_TEAM"` to `"MEETING_PROVIDER"`
7. **Temporal activities**: Replace `zoho_meetings_integration_team` with `meeting_provider_integration_team` (cast to `ZohoMeetingsIntegrationInfoTeam`)
8. **All tests**: Mechanical update of enum values and field names
