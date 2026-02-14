# Plan: Migrate `routers/users.py` to DAL Layer

## Context
The `src/routers/users.py` file (~3023 lines) has ~47 direct MongoDB accesses via `request.app.mongodb[collection]` that bypass the established DAL layer (`sybill_py/dal/queries/`). This migration moves all direct DB accesses to the DAL, improving consistency, testability, and separation of concerns.

## Commit Plan (9 commits)

---

### Commit 1: Replace simple user reads with existing DAL reader methods

**What**: Replace ~20 direct `find_one` calls that already have DAL equivalents.

**Router changes** (`src/routers/users.py`):
| Line(s) | Current | Replacement |
|---------|---------|-------------|
| 494 | `mongodb["users"].find_one({_id})` → `User.from_mongo(...)` | `users_dao.reader.get_user_by_id(_id)` |
| 502 | `mongodb["usersExtra"].find_one({_id})` → `UserExtra.from_mongo(...)` | `users_extra_dao.reader.get_user_extra_by_id(_id)` |
| 513 | `mongodb["users"].find_one({"emails.email": email_id})` | `users_dao.reader.get_user_by_email(email_id)` |
| 582-585 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 729 | `mongo["users"].find_one({"emails.email": primary_email})` | `users_dao.reader.get_user_by_email(primary_email)` |
| 807 | Same pattern | `users_dao.reader.get_user_by_email(primary_email)` |
| 1337-1341 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 1667-1668 | `mongodb["users"].find_one({_id}, projection=USER_PROJECTION)` | `users_dao.reader.get_user_by_id(user_id)` |
| 1797-1798 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 1826-1828 | `mongodb["users"].find_one({"emails.email": ...})` | `users_dao.reader.get_user_by_email(...)` |
| 1857 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 1948-1949 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 2011-2012 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 2095-2096 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 2150-2154 | Two `find_one` with `isSybillUser` filter | `users_dao.reader.get_user_by_id(id, is_sybill_user=True)` |
| 2288-2289 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 2339-2342 | `mongodb["users"].find_one({_id: user_id})` | `users_dao.reader.get_user_by_id(user_id)` |
| 2352-2354 | `mongodb["users"].find_one({"emails.email": ...})` | `users_dao.reader.get_user_by_email(...)` |
| 1900-1904 | `mongodb["usersExtra"].find_one(...)` for onboarding | `users_extra_dao.reader.onboarding_info_by_user_id(user_id)` |

**Status**: DONE (commit `2726e5312`)

**Tests** (`tests/routers/users/test_simple_reads.py`):
- Test `show_user` returns user via DAL
- Test `show_user_extra_info` returns user extra via DAL
- Test `get_user_by_email_id` returns user via DAL
- Test `get_dashboard_user_ref` / `get_additional_dashboard_user_info_by_email`
- Test `get_onboarding_info` uses `users_extra_dao`
- Test `omniscient_access` with two user lookups
- Verify DAL methods are called (not raw MongoDB)

**Shadow comparison tests** (`tests/sybill_py/dal/queries/users/test_commit1_shadow.py`):
- 16 tests running original inline MongoDB queries alongside new DAL methods against the same data
- **Pattern A** (`get_user_by_id`, no projection → `USER_PROJECTION`): proves `extendedInfo` is the ONLY excluded field; all other fields (name, emails, integrations, identities, validatedExtendedInfo, userAccountInfo, affiliateInfo, stateEvents) are byte-for-byte identical
- **Pattern A variant** (already had `USER_PROJECTION`): `to_mongo()` output identical (covers `refresh_user_integrations`)
- **Pattern B** (`get_user_by_id` + `isSybillUser` filter): fully equivalent; also tests non-sybill users return `None` from both (covers `omniscient_access`)
- **Pattern C** (`get_user_by_email` + `.lower()`): documents both deltas; includes test proving raw query FAILS with mixed-case email while DAL succeeds (improvement, not regression)
- **Pattern D** (`get_user_extra_by_id`): fully equivalent (covers `show_user_extra_info`)
- **Pattern E** (`onboarding_info_by_user_id` + fallback): all 3 cases (data exists, field missing, doc missing) produce identical results despite extra round-trip; includes partial-fields edge case
- Can be deleted once validated in production

**Audit Findings** (post-implementation — all 19 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Projection: `None` → `USER_PROJECTION`** (`{extendedInfo: False}`). Raw calls fetched full docs; DAL excludes `extendedInfo`. | 14 of 19 call sites | **None** — `extendedInfo` (raw scraped data) is not used by any affected endpoint. `validatedExtendedInfo` (the processed version) IS included. This actually reduces data transfer from MongoDB. | Safe, net positive |
| **Email `.lower()` normalization**. DAL's `get_user_by_email` calls `.lower()` on input; raw calls did not. | 4 call sites | **None** — emails should be case-insensitive per RFC. This is a correctness improvement. | Safe, improvement |
| **Extra DB round-trip in `get_onboarding_info`**. `onboarding_info_by_user_id` returns `None` for both "doc missing" and "field missing", so a second `get_user_extra_by_id` query is needed to distinguish them for 404 vs empty response. | 1 call site | **Low** — only on the "no onboarding data" path, which is rare and not latency-sensitive. Comment added to code explaining the reason. | Acceptable |
| **Fully equivalent** (same query + same projection). | 3 call sites | **None** | Identical |

---

### Commit 2: Add new users DAL reader methods + replace router accesses

**New DAL methods** (`src/sybill_py/dal/queries/users/reader.py`):

```python
async def get_user_by_system_identity_principal(self, principal: str) -> User | None:
    """Find user by systemIdentities principal (line 642-647)."""

async def check_user_invite_eligibility(self, email: str) -> dict | None:
    """Lightweight check returning {is_sybill_user, user_account_info} (lines 332, 375, 1104)."""

async def count_users_by_acc_cn(self, acc_cn: str) -> int:
    """Count users in an organization (line 258)."""
```

**Router changes**:
| Line(s) | Current | Replacement |
|---------|---------|-------------|
| 332-333 | `mongo["users"].find_one({"emails.email": ...}, projection={...})` | `users_dao.reader.check_user_invite_eligibility(email)` |
| 375-376 | Same pattern | `users_dao.reader.check_user_invite_eligibility(email)` |
| 643-647 | `mongodb["users"].find_one({"systemIdentities": {$elemMatch: ...}})` | `users_dao.reader.get_user_by_system_identity_principal(principal)` |
| 258-260 | `mongodb["users"].count_documents(...)` | `users_dao.reader.count_users_by_acc_cn(acc_cn)` |
| 1104-1106 | `mongo["users"].find_one(...)` invite check | `users_dao.reader.check_user_invite_eligibility(email)` |

**Status**: DONE

**Tests** (`tests/sybill_py/dal/queries/users/test_reader.py`):
- Test `get_user_by_system_identity_principal` with matching/non-matching principal and revoked principal
- Test `check_user_invite_eligibility` for new, existing sybill, invited users, and case-insensitive email
- Test `count_users_by_acc_cn` with 0, multiple users, and cross-org isolation

**Tests** (`tests/routers/users/test_reader_replacements.py`):
- Test `get_user_by_principal` endpoint found/not found
- Test `validate_invite_emails` endpoint for new, active, and invited users
- Test `get_eligible_orgs` endpoint returns correct member count and handles missing user

**Shadow comparison tests** (`tests/sybill_py/dal/queries/users/test_reader_shadow.py`):
- Each test runs the original inline MongoDB query and the new DAL method against the same data, asserts identical results
- Covers all 3 methods across found/not-found/edge cases (9 shadow tests total)
- Can be deleted once validated in production

**Audit Findings** (post-implementation — all 5 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent query + projection** for `get_user_by_system_identity_principal`. Constructs identical `Owner` object, same `$elemMatch`, same `USER_PROJECTION`. | 1 call site | **None** | Identical |
| **Removed `USER_PROJECTION` import** from router. No longer needed after `get_user_by_principal` endpoint migrated to DAL (which handles projection internally). | Import only | **None** | Cleanup |
| **Double `.lower()` on email** in `check_user_invite_eligibility`. Router already lowercases email before calling; DAL also calls `.lower()` internally. `"foo".lower().lower() == "foo".lower()` — idempotent. | 3 call sites | **None** | Safe, idempotent |
| **Fully equivalent query + projection** for `count_users_by_acc_cn`. Same `count_documents` with same filter `{"userAccountInfo.accCN": acc_cn}`. | 1 call site | **None** | Identical |

---

### Commit 3: Add users DAL writer methods (part 1 - auth/account) + replace

**New DAL methods** (`src/sybill_py/dal/queries/users/writer.py`):

```python
async def set_user_account_info(self, user_id: UUID, user_account_info: UserAccountInfo) -> User | None:
    """Set userAccountInfo for a user (line 446-455)."""

async def add_identities_by_email(self, email: str, identities: list) -> User | None:
    """Add identities to user found by email (lines 701-705)."""

async def finalize_user_setup(self, user_id: UUID, sybill_system_identity: Owner) -> User | None:
    """Set integration/preferences userId and add system identity (lines 939-946)."""
```

**Router changes**:
| Line(s) | Replacement |
|---------|-------------|
| 446-455 | `users_dao.writer.set_user_account_info(user_id, user_account_info)` |
| 701-705 | `users_dao.writer.add_identities_by_email(primary_email, identities)` |
| 939-946 | `users_dao.writer.finalize_user_setup(user_id, sybill_system_identity)` |

**Tests** (`tests/sybill_py/dal/queries/users/test_writer_auth.py`):
- Test `set_user_account_info` sets and returns updated user
- Test `add_identities_by_email` adds identities correctly
- Test `finalize_user_setup` sets userId on integrations/preferences and adds system identity

---

### Commit 4: Add users DAL writer methods (part 2 - profile/integrations) + replace

**New DAL methods** (`src/sybill_py/dal/queries/users/writer.py`):

```python
async def update_linkedin_profile(self, user_id: UUID, linkedin_url: str, updated_by: UUID, has_preferences: bool) -> bool:
    """Update LinkedIn URL in validatedExtendedInfo (lines 619-628)."""

async def update_onboarding_validated_fields(self, user_id: UUID, fields: dict) -> bool:
    """Set validated extended info fields from onboarding (lines 1867-1883)."""

async def push_integration(self, user_id: UUID, integration_doc: dict) -> bool:
    """Push a new integration to user's integrations array (lines 1399-1401)."""

async def revoke_identity(self, user_id: UUID, identity_user_id: str, identity_provider: str) -> bool:
    """Mark an identity as revoked (lines 1919-1923)."""
```

**Router changes**:
| Line(s) | Replacement |
|---------|-------------|
| 619-628 | `users_dao.writer.update_linkedin_profile(...)` |
| 1867-1883 | `users_dao.writer.update_onboarding_validated_fields(user_id, fields)` |
| 1399-1401 | `users_dao.writer.push_integration(user_id, integration_doc)` |
| 1919-1923 | `users_dao.writer.revoke_identity(user_id, identity_user_id, identity_provider)` |

**Tests** (`tests/sybill_py/dal/queries/users/test_writer_profile.py`):
- Test `update_linkedin_profile` with and without preferences
- Test `update_onboarding_validated_fields` sets correct fields
- Test `push_integration` appends to array
- Test `revoke_identity` sets revoked=True on correct identity

---

### Commit 5: Add users DAL writer methods (part 3 - complex operations) + replace

**New DAL methods** (`src/sybill_py/dal/queries/users/writer.py`):

```python
async def migrate_google_social_identity(self, primary_email: str, callback_identities: list, new_google_identity, identity_user_id: str) -> User | None:
    """Migrate Google Social to Custom connection (lines 1493-1522)."""

async def upsert_or_replace_integration(self, user_id: UUID, integration_type: str, integration_doc: dict) -> tuple[bool, dict | None]:
    """Push integration if type doesn't exist, else replace existing (lines 2666-2682)."""

async def push_integration_initialized_state_event(self, user_id: UUID, integration_id: UUID, state_event) -> bool:
    """Push INITIALIZED state event to integration (lines 2524-2537)."""

async def update_affiliate_link(self, user_id: UUID, affiliate_link: str | None, paypal_email: str | None) -> User | None:
    """Update affiliate link and paypal email (lines 2319-2323)."""

async def update_affiliate_paypal_email(self, user_id: UUID, paypal_email: str) -> User | None:
    """Update affiliate PayPal email (lines 2469-2473)."""
```

**Router changes**:
| Line(s) | Replacement |
|---------|-------------|
| 1493-1522 | `users_dao.writer.migrate_google_social_identity(...)` |
| 2666-2682 | `users_dao.writer.upsert_or_replace_integration(...)` |
| 2524-2537 | `users_dao.writer.push_integration_initialized_state_event(...)` |
| 2319-2323 | `users_dao.writer.update_affiliate_link(...)` |
| 2469-2473 | `users_dao.writer.update_affiliate_paypal_email(...)` |

**Tests** (`tests/sybill_py/dal/queries/users/test_writer_complex.py`):
- Test `migrate_google_social_identity` updates identity and integration
- Test `upsert_or_replace_integration` for push (new type) and replace (existing type)
- Test `push_integration_initialized_state_event` with active and cancelled integration
- Test `update_affiliate_link` and `update_affiliate_paypal_email`

---

### Commit 6: users_extra writer + org reader additions + replace

**New DAL methods**:

`src/sybill_py/dal/queries/users_extra/writer.py`:
```python
async def set_onboarding_info(self, user_id: UUID, onboarding_info) -> bool:
    """Upsert onboarding info in usersExtra (lines 1861-1865)."""
```

`src/sybill_py/dal/queries/organizations/reader.py`:
```python
async def list_joinable_orgs_by_email_domain(self, domain: str) -> list[dict]:
    """Find non-invite-only orgs matching email domain (lines 246-253)."""
```

**Router changes**:
| Line(s) | Replacement |
|---------|-------------|
| 1861-1865 | `users_extra_dao.writer.set_onboarding_info(user_id, onboarding_info)` |
| 246-253 | `org_dao.reader.list_joinable_orgs_by_email_domain(domain)` |
| 418-420 | `org_dao.reader.get_org_info_by_org_id(org_id)` (already exists) |
| 737-742 | `org_dao.reader.get_org_info_by_acc_cn(acc_cn)` (already exists) |
| 815-820 | Same as above |
| 823-826 | `org_dao.reader.list_joinable_orgs_by_email_domain(domain)` or similar |

**Tests**:
- `tests/sybill_py/dal/queries/users_extra/test_writer.py` - test `set_onboarding_info`
- `tests/sybill_py/dal/queries/organizations/test_reader.py` - test `list_joinable_orgs_by_email_domain`

---

### Commit 7: New referrals DAL module + replace

**New files**:
- `src/sybill_py/dal/queries/referrals/__init__.py`
- `src/sybill_py/dal/queries/referrals/reader.py`
- `src/sybill_py/dal/queries/referrals/writer.py`

**DAL methods**:
```python
# reader.py
async def list_referrals_by_affiliate_id(self, affiliate_id: str) -> list[Referral]:
    """List referrals for an affiliate (lines 2239-2244)."""

# writer.py
async def upsert_referral_with_email_event(self, affiliate_id: str, entity_id: UUID, referral, email_event) -> bool:
    """Upsert referral and push email send event (lines 2426-2434)."""
```

**Router changes**:
| Line(s) | Replacement |
|---------|-------------|
| 2239-2244 | `referrals_dao.reader.list_referrals_by_affiliate_id(affiliate_id)` |
| 2426-2434 | `referrals_dao.writer.upsert_referral_with_email_event(...)` |

**Tests** (`tests/sybill_py/dal/queries/referrals/`):
- `test_reader.py` - list referrals with 0 and multiple results
- `test_writer.py` - upsert creates new and appends to existing

---

### Commit 8: New copilot_public_registrations DAL module + replace

**New files**:
- `src/sybill_py/dal/queries/copilot_public_registrations/__init__.py`
- `src/sybill_py/dal/queries/copilot_public_registrations/writer.py`

**DAL methods**:
```python
# writer.py
async def upsert_registration(self, email: str, registration, contact_for_sales: bool | None) -> bool:
    """Upsert public copilot registration (lines 2832-2839)."""
```

**Router changes**:
| Line(s) | Replacement |
|---------|-------------|
| 2832-2839 | `copilot_registrations_dao.writer.upsert_registration(...)` |

**Tests** (`tests/sybill_py/dal/queries/copilot_public_registrations/test_writer.py`):
- Test upsert creates new registration
- Test upsert increments usage count on existing

---

### Commit 9: Cleanup + remove unused imports + E2E tests

**Cleanup**:
- Remove `from motor.core import AgnosticDatabase` (no longer needed)
- Remove `from pymongo.database import Collection` (no longer needed)
- Remove `from pymongo import ReturnDocument` if fully unused
- Remove unused `request.app.mongodb` references in function signatures where `mongo` was passed around
- Simplify function signatures that no longer need `mongo` parameter (e.g., `_handle_auth0_login`, `_handle_legacy_auth0_login`, `_process_invite_emails`, `_finalize_user_setup`)

**E2E tests** (`tests/routers/users/test_users_e2e.py`):
- Full `join_organization` flow (create new org + join existing org)
- Full `on_user_login_via_auth0` flow (new user + existing user)
- Full `link_calendar` flow
- Full `link_integration_to_account` flow
- Full referral flow (`add_referral_info` → `send_referral_email` → `get_referrals`)

---

## Key Files to Modify

| File | Purpose |
|------|---------|
| `src/routers/users.py` | Main router - replace all direct MongoDB accesses |
| `src/sybill_py/dal/queries/users/reader.py` | Add 3 new reader methods |
| `src/sybill_py/dal/queries/users/writer.py` | Add ~12 new writer methods |
| `src/sybill_py/dal/queries/users_extra/writer.py` | Add 1 new writer method |
| `src/sybill_py/dal/queries/organizations/reader.py` | Add 1 new reader method |
| `src/sybill_py/dal/queries/referrals/` (new) | New DAL module (reader + writer) |
| `src/sybill_py/dal/queries/copilot_public_registrations/` (new) | New DAL module (writer) |

## Existing Utilities to Reuse

- `users_dao` from `sybill_py.dal.queries.users` - existing reader/writer
- `users_extra_dao` from `sybill_py.dal.queries.users_extra` - existing reader/writer
- `org_dao` from `sybill_py.dal.queries.organizations` - existing reader
- `BaseReader` / `BaseWriter` from `sybill_py.dal.queries.base` - base classes for new DAL modules
- `create_mock_user_doc`, `create_mock_user_in_db` from `tests/test_utils/users.py`
- `create_mock_org_doc`, `insert_mock_org_in_db` from `tests/test_utils/organizations.py`
- `mock_request` fixture from `tests/conftest.py` for DAL tests
- `AsyncClient` + `ASGITransport` pattern from existing router tests

## Follow-ups

Items discovered during implementation. Can be addressed in Commit 9 (cleanup) or as a dedicated follow-up commit.

### 1. Remove redundant `.lower()` at call sites

DAL reader methods (`get_user_by_email`, `check_user_invite_eligibility`) already normalize emails to lowercase internally. Several router call sites still do `.lower()` before calling the DAL, making the lowering redundant.

**Call sites to clean up** (`src/routers/users.py`):
| Line | Current | After cleanup |
|------|---------|---------------|
| ~327-329 | `email_lower = email.lower()` → `check_user_invite_eligibility(email_lower)` | `check_user_invite_eligibility(email)` |
| ~369-370 | `email_lower = email.lower()` → `check_user_invite_eligibility(email_lower)` | `check_user_invite_eligibility(email)` |
| ~1086-1088 | `invitee_email_lower = invitee_email.lower()` → `check_user_invite_eligibility(invitee_email_lower)` | `check_user_invite_eligibility(invitee_email)` |

**Note**: Some of these `_lower` variables are also used downstream (e.g., passed to `UserRequest(primary_email=invitee_email_lower)`), so each site needs individual review — only remove the variable if it's exclusively used for the DAL call.

### 2. Delete shadow comparison tests after production validation

`tests/sybill_py/dal/queries/users/test_reader_shadow.py` (9 tests) exists solely to prove mechanical equivalence. Delete once migration is validated in production.

## Verification

1. **Unit tests**: Run `pytest tests/sybill_py/dal/queries/users/ -v` after each DAL commit
2. **Router tests**: Run `pytest tests/routers/users/ -v` after each router commit
3. **Full suite**: Run `pytest tests/ -x` after final commit to ensure no regressions
4. **Lint/format**: Run pre-commit hooks (ruff) before each commit
5. **Type check**: Verify no type errors introduced via DAL method signatures
