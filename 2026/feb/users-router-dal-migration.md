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

**Status**: DONE

**Tests** (`tests/sybill_py/dal/queries/users/test_writer_auth.py`):

- Test `set_user_account_info` sets/returns user, returns None for nonexistent, overwrites existing
- Test `add_identities_by_email` adds identity, returns None for nonexistent email, deduplicates via $addToSet
- Test `finalize_user_setup` sets userId on integrations/preferences + adds system identity, returns None for nonexistent, deduplicates

**Shadow comparison tests** (`tests/sybill_py/dal/queries/users/test_commit3_shadow.py`):

- 8 tests running original inline MongoDB queries alongside new DAL writer methods against the same data
- `set_user_account_info`: equivalent result + both return None for nonexistent user
- `add_identities_by_email`: equivalent identity set after addition + both return None for nonexistent email + both deduplicate via $addToSet
- `finalize_user_setup`: equivalent integrations userId/preferences userId/systemIdentities + both return None + idempotent $addToSet
- Can be deleted once validated in production

**Audit Findings** (post-implementation — all 3 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent query** for `set_user_account_info`. Same `find_one_and_update` with `$set userAccountInfo`, same `ReturnDocument.AFTER`. DAL returns `User` directly instead of raw doc. | 1 call site | **None** | Identical |
| **Fully equivalent query** for `add_identities_by_email`. Same `find_one_and_update` with `$addToSet identities.$each`. DAL returns `User` directly. | 1 call site | **None** | Identical |
| **Fully equivalent query** for `finalize_user_setup`. Same `find_one_and_update` with `$set` + `$addToSet systemIdentities`. DAL returns `User` directly. | 1 call site | **None** | Identical |
| **Removed unused `Collection` import** from router. No longer needed after guest login migration. | Import only | **None** | Cleanup |

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

**Status**: DONE

**Tests** (`tests/sybill_py/dal/queries/users/test_writer_profile.py`):

- Test `update_linkedin_profile` with/without preferences, nonexistent user
- Test `update_onboarding_validated_fields` sets fields, empty dict returns false, nonexistent user
- Test `push_integration` appends to array, nonexistent user
- Test `revoke_identity` revokes correct identity, nonexistent user, no matching identity

**Shadow comparison tests** (`tests/sybill_py/dal/queries/users/test_commit4_shadow.py`):

- 7 tests running original inline MongoDB queries alongside new DAL writer methods
- `update_linkedin_profile`: equivalent with and without preferences
- `update_onboarding_validated_fields`: equivalent result + empty fields no-op
- `push_integration`: equivalent push result
- `revoke_identity`: equivalent revocation + both return zero for nonexistent user
- Can be deleted once validated in production

**Audit Findings** (post-implementation — all 4 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent** for `update_linkedin_profile`. Same `$set` fields, conditional preferences branch preserved via `has_preferences` parameter. | 1 call site | **None** | Identical |
| **Fully equivalent** for `update_onboarding_validated_fields`. Same `$set` with dynamic fields dict. DAL adds early-return guard for empty dict (matches router's `if user_set_query:` guard). | 1 call site | **None** | Identical |
| **Fully equivalent** for `push_integration`. Same `$push integrations` with `to_mongo()` doc. Return type changed from `UpdateResult` to `bool`. | 1 call site | **None** | Identical |
| **Fully equivalent** for `revoke_identity`. Same `$set identities.$[idt].revoked` with same array_filters. | 1 call site | **None** | Identical |

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

**Status**: DONE

**Tests** (`tests/sybill_py/dal/queries/users/test_writer_complex.py`):

- Test `migrate_google_social_identity`: migrates identity, nonexistent user, no matching Google Social identity
- Test `upsert_or_replace_integration`: push new, replace existing, nonexistent user
- Test `push_integration_initialized_state_event`: pushes event, nonexistent user
- Test `update_affiliate_link`: updates link+email, nonexistent user
- Test `update_affiliate_paypal_email`: updates email, nonexistent user

**Shadow comparison tests** (`tests/sybill_py/dal/queries/users/test_commit5_shadow.py`):

- 6 tests running original inline MongoDB queries alongside new DAL writer methods
- `migrate_google_social_identity`: equivalent identity set after migration
- `upsert_or_replace_integration`: equivalent push-new and replace-existing behavior
- `push_integration_initialized_state_event`: equivalent state event count
- `update_affiliate_link`: equivalent affiliateInfo fields
- `update_affiliate_paypal_email`: equivalent paypal email field
- Can be deleted once validated in production

**Audit Findings** (post-implementation — all 5 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent** for `migrate_google_social_identity`. Same complex `$addToSet` + `$set` with array_filters. | 1 call site | **None** | Identical |
| **Fully equivalent** for `upsert_or_replace_integration`. Same two-step push/replace pattern. Returns `(bool, list or None)` instead of raw `UpdateResult` + raw doc. | 1 call site | **None** | Identical |
| **Fully equivalent** for `push_integration_initialized_state_event`. Same `$elemMatch` + `$push stateEvents`. Uses `runtime.app_db` internally (same DB). | 1 call site | **None** | Identical |
| **Fully equivalent** for `update_affiliate_link`. Same `$set` both fields. Returns `User` instead of raw doc. | 1 call site | **None** | Identical |
| **Fully equivalent** for `update_affiliate_paypal_email`. Same `$set` single field. Returns `User` instead of raw doc. | 1 call site | **None** | Identical |

---

### Commit 6: users_extra writer + org reader additions + replace

**New DAL methods**:

`src/sybill_py/dal/queries/users_extra/writer.py`:

```python
async def set_onboarding_info(self, user_id: UUID, onboarding_info: OnboardingInfo) -> None:
    """Upsert onboarding info in usersExtra (set_onboarding_info)."""
```

`src/sybill_py/dal/queries/organizations/reader.py`:

```python
async def list_joinable_orgs_by_email_domain(self, domain: str) -> list[OrgInfo]:
    """Find non-invite-only orgs matching email domain (get_eligible_orgs)."""

async def get_org_by_email_domains(self, domains: list[str]) -> OrgInfo | None:
    """Find first org matching any of the given email domains (_handle_legacy_auth0_login)."""
```

**Router changes**:

| Line(s) | Replacement |
|---------|-------------|
| 1773-1777 | `users_extra_dao.writer.set_onboarding_info(user_id, onboarding_info)` |
| 244-251 | `org_dao.reader.list_joinable_orgs_by_email_domain(domain)` |
| 410-418 | `org_dao.reader.get_org_info_by_org_id(org_id)` (already exists) |
| 692-697 | `org_dao.reader.get_org_info_by_acc_cn(acc_cn)` (already exists) |
| 770-775 | Same as above |
| 778-790 | `org_dao.reader.get_org_by_email_domains(domains)` |

**Status**: DONE

**Tests** (`tests/sybill_py/dal/queries/users_extra/test_writer.py`):

- Test `set_onboarding_info` creates new doc via upsert, updates existing, and handles partial info

**Tests** (`tests/sybill_py/dal/queries/organizations/test_reader.py`):

- Test `list_joinable_orgs_by_email_domain` with matching/no-match/invite-only/multiple orgs/email domains in result
- Test `get_org_by_email_domains` with match/no-match/multi-domain/$in semantics

**Shadow comparison tests** (`tests/sybill_py/dal/queries/users_extra/test_commit6_shadow.py`):

- 6 tests running original inline MongoDB queries alongside new DAL methods
- `set_onboarding_info`: equivalent upsert creates doc + updates existing doc
- `list_joinable_orgs_by_email_domain`: equivalent results + both exclude invite-only
- `get_org_by_email_domains`: equivalent find + both return None for no match
- Can be deleted once validated in production

**Audit Findings** (post-implementation — all 6 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent** for `set_onboarding_info`. Same `update_one` with `$set` + `$setOnInsert` + `upsert=True`. | 1 call site | **None** | Identical |
| **Projection widened** for `list_joinable_orgs_by_email_domain`. Same query filter; returns `OrgInfo` objects instead of raw docs. Result includes `email_domains` and `account_info` — same fields used by caller. | 1 call site | **None** | Identical |
| **Projection widened** for `get_org_info_by_org_id`. Was `ACCOUNT_INFO_PROJECTION` (accountInfo + emailDomains); now default (`{integrations.users: 0}`). Slightly more data returned but all needed fields present. | 1 call site | **None** — extra fields unused | Safe, negligible overhead |
| **Projection widened** for `get_org_info_by_acc_cn` (2 sites). Was `{"accountInfo": 1}`; now default. Slightly more data but only `account_info` is used. | 2 call sites | **None** — extra fields unused | Safe, negligible overhead |
| **Projection change** for `get_org_by_email_domains`. Was `{"accountInfo": 1}`; now `ACCOUNT_INFO_PROJECTION` (adds `emailDomains`). Needed for `OrgInfo` model parsing (requires `email_domains`). | 1 call site | **None** — `emailDomains` was already in the doc | Safe |
| **Removed unused `ACCOUNT_INFO_PROJECTION` import** from router. No longer needed after all org queries migrated to DAL. | Import only | **None** | Cleanup |

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
    """List referrals for an affiliate (get_referrals)."""

# writer.py
async def upsert_referral_with_email_event(
    self, affiliate_id: str, referred_entity_id: UUID,
    new_referral: Referral, email_send_event: EmailSendEvent
) -> None:
    """Upsert referral and push email send event (send_referral_email)."""
```

**Router changes**:

| Line(s) | Replacement |
|---------|-------------|
| 2119-2125 | `referrals_dao.reader.list_referrals_by_affiliate_id(affiliate_id)` |
| 2289-2299 | `referrals_dao.writer.upsert_referral_with_email_event(...)` |

**Status**: DONE

**Tests** (`tests/sybill_py/dal/queries/referrals/`):

- `test_reader.py` - list referrals with match/no-match/cross-affiliate isolation (3 tests)
- `test_writer.py` - upsert creates new + appends to existing (2 tests)

**Shadow comparison tests** (`tests/sybill_py/dal/queries/referrals/test_commit7_shadow.py`):

- 3 tests running original inline MongoDB queries alongside new DAL methods
- `list_referrals_by_affiliate_id`: equivalent result set
- `upsert_referral_with_email_event`: equivalent create + equivalent append
- Can be deleted once validated in production

**Audit Findings** (post-implementation — all 2 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent** for `list_referrals_by_affiliate_id`. Same `find` query, same `Referral.from_mongo()` parsing. | 1 call site | **None** | Identical |
| **Timestamp source change** for `upsert_referral_with_email_event`. Original used a shared `now` for both `email_send_event.sent_at` and `updatedAt`. DAL uses `email_send_event.sent_at` for `updatedAt` — same value since caller creates the event with `utils.get_current_ts()`. | 1 call site | **None** — same timestamp flows through | Identical |

---

### Commit 8: New copilot_public_registrations DAL module + replace

**New files**:

- `src/sybill_py/dal/queries/copilot_public_registrations/__init__.py`
- `src/sybill_py/dal/queries/copilot_public_registrations/writer.py`

**DAL methods**:

```python
# writer.py
async def upsert_registration(
    self, email: str, registration: PublicRegistration, contact_for_sales: bool | None = None
) -> None:
    """Upsert public copilot registration (register_user_copilot)."""
```

**Router changes**:

| Line(s) | Replacement |
|---------|-------------|
| 2668-2681 | `copilot_registrations_dao.writer.upsert_registration(...)` |

**Status**: DONE

**Tests** (`tests/sybill_py/dal/queries/copilot_public_registrations/test_writer.py`):

- Test upsert creates new registration, increments usages on existing, sets contactForSales, omits contactForSales when None (4 tests)

**Shadow comparison tests** (`tests/sybill_py/dal/queries/copilot_public_registrations/test_commit8_shadow.py`):

- 2 tests running original inline MongoDB query alongside new DAL method
- Equivalent upsert creates doc + equivalent upsert with contactForSales
- Can be deleted once validated in production

**Audit Findings** (post-implementation — 1 replacement verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent** for `upsert_registration`. Same `update_one` with `$set/$inc/$setOnInsert` + `upsert=True`. `contact_for_sales` conditionally added to `$set` same as original. | 1 call site | **None** | Identical |

---

### Commit 9: Cleanup + remove unused imports

**Cleanup performed**:

- Removed unused `mongo` parameter from `_finalize_user_setup` (migrated to DAL in Commit 3)
- Updated 2 call sites that passed `request.app.mongodb` to `_finalize_user_setup`

**Could NOT remove** (still used by remaining raw accesses):

- `from motor.core import AgnosticDatabase` — still used at line 2442 (`mongo: AgnosticDatabase = request.app.mongodb`)
- `from pymongo import ReturnDocument` — still used at lines 711, 782, 820 (in `upsert_new_user_by_email` calls)
- `_handle_auth0_login`, `_handle_legacy_auth0_login`, `_process_invite_emails` — still need `mongo` for `upsert_new_user_by_email` which uses `extract_integration()` on raw docs

**Status**: DONE (commit `8cc56bcb1`)

**Tests**: Full test suite passed — 284 passed, 10 skipped. No new tests needed (cleanup-only commit).

**Audit Findings** (post-implementation — verified all changes, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Removed unused `mongo` param** from `_finalize_user_setup`. This function was fully migrated to DAL in Commit 3 but the signature wasn't cleaned up. | 1 function + 2 call sites | **None** | Cleanup |
| **4 remaining direct `users` collection accesses** at lines 711/782/820, 2232, 2442/2606. These use `extract_integration()` which expects raw `list[dict]` from MongoDB, not typed DAL objects. | 4 access patterns | **None** — deferred to follow-up | See Follow-up #3 |
| **`AgnosticDatabase` + `ReturnDocument` imports retained**. Still needed by remaining raw accesses. | 2 imports | **None** | Deferred |

**E2E tests**: Not implemented in this commit — the full-flow integration tests (join_organization, on_user_login_via_auth0, link_calendar, link_integration_to_account, referral flow) require extensive mocking of Auth0, Stripe, and external services. Deferred to follow-up #4.

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

### 1. ~~Remove redundant `.lower()` at call sites~~ — RESOLVED (no-op)

Investigation confirmed all `.lower()` variables are **used downstream** (e.g., passed to `UserRequest(primary_email=invitee_email_lower)`, domain extraction for org creation), not just for DAL calls. The DAL's internal `.lower()` is a safety net, not a replacement for the router's normalization. No changes needed.

### 2. Delete shadow comparison tests after production validation

All shadow test files exist solely to prove mechanical equivalence. Delete once migration is validated in production:

- `tests/sybill_py/dal/queries/users/test_commit1_shadow.py` (16 tests)
- `tests/sybill_py/dal/queries/users/test_reader_shadow.py` (9 tests)
- `tests/sybill_py/dal/queries/users/test_commit3_shadow.py` (8 tests)
- `tests/sybill_py/dal/queries/users/test_commit4_shadow.py` (7 tests)
- `tests/sybill_py/dal/queries/users/test_commit5_shadow.py` (6 tests)
- `tests/sybill_py/dal/queries/users_extra/test_commit6_shadow.py` (6 tests)
- `tests/sybill_py/dal/queries/referrals/test_commit7_shadow.py` (3 tests)
- `tests/sybill_py/dal/queries/copilot_public_registrations/test_commit8_shadow.py` (2 tests)
- `tests/sybill_py/dal/queries/users/test_writer_insert.py` — shadow tests in `TestInsertUserShadow` + `TestGetOrCreateUserByEmailShadow` (2 tests)
- `tests/sybill_py/components/integration/test_extract_adapter.py` — shadow tests in `TestExtractIntegrationFromUserShadow` (3 tests)

### 3. Migrate remaining direct `users` collection accesses — DONE

**Phase 1** (follow-up commits `d878eac03` + `b479529a8`):

| Line | Pattern | Solution |
|------|---------|----------|
| 782 | `find_one_and_update` get-or-create temp user | `users_dao.writer.get_or_create_user_by_email(email, user)` |
| 2232 | `insert_one(recipient_user.to_mongo())` | `users_dao.writer.insert_user(recipient_user)` |
| 2442+2454 | `find_one` + `extract_integration(user_doc.get("integrations",...))` | `users_dao.reader.get_user_by_id()` + `extract_integration_from_user()` adapter |
| 2606-2608 | Same pattern | Same solution |

**New adapter**: `extract_integration_from_user(user, type_hint, ...)` in `sybill_py/components/integration/integration.py` — converts `User.integrations` to raw dicts via `.to_mongo()` for the existing `extract_integration` function.

**Cleanup**: Removed `from motor.core import AgnosticDatabase` import (was only used at line 2442). Removed unused `MONGODB_ID_FIELD` import.

**Tests**: 8 unit + 2 shadow tests for new writer methods; 7 unit + 3 shadow tests for adapter function.

**Audit Findings — Phase 1** (all 4 replacements verified):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent** for `insert_user`. Same `insert_one(user.to_mongo())`. | 1 call site | **None** | Identical |
| **Fully equivalent** for `get_or_create_user_by_email`. Same `find_one_and_update` with `$setOnInsert` + `upsert=True` + `ReturnDocument.AFTER`. | 1 call site | **None** | Identical |
| **Projection: `None` → `USER_PROJECTION`** for link/unlink endpoints. Was raw `find_one` (full doc); now DAL `get_user_by_id` which uses `USER_PROJECTION` (excludes `extendedInfo`). | 2 call sites | **None** — `extendedInfo` not used by these endpoints | Safe, net positive |
| **Removed try/except deserialization guard** in `link_integration_to_account`. Was `try: User.from_mongo(user_doc) except:`. DAL does the same internally; if deserialization fails, the error propagates naturally. | 1 call site | **None** — error surfaces same way | Safe |
| **Added null check** in `unlink_integration_from_account`. Original code had no null check after `find_one`; now properly returns 404. | 1 call site | **None** — improvement | Safe, improvement |

**Phase 2 — `upsert_new_user_by_email`** (branch `chore/dal-upsert-by-email`, 3 commits):

Migrated the 120-line `upsert_new_user_by_email` utility from `creation_helpers.py` to `users_dao.writer.upsert_by_email`. This function was used by 15 call sites across 12 files (routers, components, runtimes, backfill scripts). All call sites now go through the DAL instead of receiving a raw `mongo` parameter.

| Commit | Hash | Description |
|--------|------|-------------|
| 1 | `39dd5a535` | Add `upsert_by_email` to `UsersWriter` + 7 unit tests + 3 shadow tests |
| 2 | `a5f167969` | Migrate all 15 call sites across 12 files |
| 3 | (pending) | Remove old function + `_augment_lower_case_emails` from `creation_helpers.py`, convert shadow tests to integration tests |

**15 call sites migrated**:

| File | Calls | `return_document` |
|------|-------|--------------------|
| `src/routers/users.py` | 4 | 2 AFTER, 2 BEFORE |
| `src/routers/organizations.py` | 1 | AFTER |
| `src/routers/calls.py` | 1 | AFTER |
| `src/routers/admin.py` | 1 | AFTER |
| `src/dao/invite_reader_writer.py` | 1 | AFTER |
| `src/sybill_py/components/sharing/helpers.py` | 1 | AFTER |
| `src/sybill_py/components/messaging/helpers.py` | 1 | AFTER |
| `src/sybill_py/components/upstream_meetings/helpers.py` | 1 | AFTER |
| `src/sybill_py/components/people/user_helpers.py` | 1 | AFTER |
| `src/sybill_py/runtimes/galactus/activities/call_processing/meeting_info.py` | 1 | AFTER |
| `src/cs_scripts/backfill/script_backfill_message_members.py` | 1 | AFTER |
| `src/cs_scripts/backfill/script_backfill_crm_activities_to_messages.py` | 1 | AFTER |

**Cleanup**: Removed `upsert_new_user_by_email` + `_augment_lower_case_emails` from `creation_helpers.py`. Removed unused imports (`HTTPException`, `AsyncIOMotorDatabase`, `ReturnDocument`, `DuplicateKeyError`, `MONGODB_ID_FIELD`, `USER_PROJECTION`). Kept `create_user_from_request` and `is_valid_calendar_identity`.

**Tests**: 7 unit + 3 integration tests for `upsert_by_email`.

**Audit Findings — Phase 2** (all 15 replacements verified, no behavioral regressions):

| Finding | Affected | Risk | Verdict |
|---------|----------|------|---------|
| **Fully equivalent** for all call sites. Same `count_documents` → `insert_one` / `find_one_and_update` logic with `DuplicateKeyError` handling. | 15 call sites | **None** | Identical |
| **Circular import resolved** via lazy import. `is_valid_calendar_identity` imported inside method body to break `writer.py` → `creation_helpers` → `routers.integration_utils` → `calendar.creds` → `users.dao` cycle. | DAL only | **None** | Safe |
| **Removed `mongo` parameter** from all callers. No more passing `request.app.mongodb` / `runtime.app_db` — DAL uses `self._collection` internally. | 15 call sites | **None** | Cleanup, improvement |
| **Fixed broken imports** in backfill scripts. `script_backfill_message_members.py` imported from `routers.router_utils` which didn't export `upsert_new_user_by_email`. | 1 call site | **None** | Bug fix |
| **Removed `ReturnDocument` import** from `routers/users.py`. No longer needed — only the DAL uses it now. Lines 709/812 pass `ReturnDocument.BEFORE` as a parameter to the DAL method. | 1 import | **None** | Cleanup |

**All direct `users` collection accesses in `routers/users.py` are now fully migrated to the DAL layer.**

### 4. E2E integration tests — DONE

**Phase 1** (commits ending at `9780ce915`):

Added comprehensive E2E and unit test coverage for the users router, achieving **91% coverage** (up from 42%). Total: **177 tests** across 13 test files.

#### Test Files Created/Enhanced

**E2E Integration Tests** (HTTP-level via `AsyncClient`):

1. **`test_e2e_referrals.py`** (10 tests): `add_referral_info`, `get_referrals`, `send_referral_email` — new recipient creation, existing recipient handling, affiliate validation, referral doc creation
2. **`test_e2e_join_org.py`** (8 tests): `join_organization` — create new org (admin role), join existing org (member role), invite validation, domain matching, already-in-org guard
3. **`test_e2e_integrations.py`** (12 tests): `link_calendar`, `link_integration_to_account`, `unlink_integration_from_account` — Google Calendar linking, email account linking, credential validation, CANCELLED state event append
4. **`test_e2e_auth0_login.py`** (5 tests): `on_user_login_via_auth0` — multi-org/legacy routing, cache invalidation, org background task orchestration, response shape
5. **`test_e2e_account_mgmt.py`** (9 tests): `suspend_user_account`, `activate_user_account` — account suspension/activation flows, CRM/calendar cleanup, state event tracking, integration status changes
6. **`test_e2e_affiliate_extended.py`** (12 tests): `update_affiliate_link`, `update_affiliate_paypal_email`, `register_user_copilot` — affiliate program setup, PayPal email updates, copilot public registration with referral attribution
7. **`test_e2e_auth_misc.py`** (35 tests): `set_user_email`, `sync_user_calendar_events`, `desktop_app_heartbeat` — email setting logic, Google/Outlook/Zoho calendar sync, desktop app presence tracking, suspended user handling
8. **`test_e2e_simple_endpoints.py`** (19 tests): `get_user_info`, `get_user_stats`, `get_wrapped_2025`, `get_extended_info`, `get_desktop_app_heartbeat`, `find_users_by_system_ident_prefix`, `get_calendar_last_synced_at`, `set_onboarding_info`, `validate_invite_emails`, `get_additional_dashboard_user_info_by_email` — simple GET/POST endpoints with various edge cases

**Integration Logic Tests** (component/DAL-level):

1. **`test_refresh_integrations.py`** (13 tests): `refresh_user_integrations` — Google Calendar, Outlook Calendar, Stripe integration refresh logic, expiration checks, MS Graph credential handling, setup retry on failure
2. **`test_reader_replacements.py`** (7 tests): Direct reader method tests — `get_user_by_principal`, `validate_invite_emails` eligibility checks, `get_eligible_orgs` with member counts
3. **`test_simple_reads.py`** (12 tests): Simple read operations — `show_user`, `show_user_extra_info`, `get_user_by_email`, `get_dashboard_user_ref`, `get_dashboard_user_info_by_email`, `get_onboarding_info`
4. **`test_set_email.py`** (2 tests): `set_user_email` — set email if unset, handle already-set case

**Unit Tests for Helper Functions**:

1. **`test_unit_helpers.py`** (33 tests): Isolated unit tests for complex helper functions:
    - `_handle_auth0_login`: new user without org, existing user with invite, existing user with missing org
    - `_finalize_user_setup`: integration/preference userId setup, system identity addition
    - `_setup_user_integrations`: Google Calendar, Stripe integration setup
    - `_setup_org_background_tasks`: task enqueueing
    - `_process_invite_emails`: new user invites, existing Sybill user skip, already-invited skip
    - `_initialize_integrations_after_org_join`: Google/Outlook/Zoho calendar initialization, email integration setup, already-initialized skip
    - `_suggested_messages_for_onboarding`: Google Mail message filtering, domain skipping, author validation
    - `_handle_legacy_auth0_login`: new user creates org+user, existing user with invite, joins existing org, unsupported identity rejection
    - `_handle_existing_calendar_cleanup`: stops active calendar watcher, skips if no active calendar
    - `_sync_user_calendar_common`: Google/Outlook/Zoho calendar resync, user not found, suspended user handling, creds not found, watcher update failure retry
    - `_sync_proxycurl_and_upsert_person`: success with/without person
    - `_suggested_messages_google_mail`: filtered message returns, same-domain skip, missing author skip
    - `_setup_unified_to_entities_and_trigger_sync`: success, existing integration deletion

#### Test Infrastructure

- **Pattern**: HTTP-level tests via `AsyncClient` + `ASGITransport`, real MongoDB (Docker), external services mocked (`mocker.patch`)
- **Fixtures**: Extended [`conftest.py`](tests/routers/users/conftest.py) with `api_tokens`, `email_client`, `rate_limiter`, mock factories for Auth0/Stripe/calendar services
- **Helper utilities**: `create_test_user_with_account`, `create_test_org`, `create_test_integration`, `create_test_calendar_integration` in `conftest.py`
- **Execution time**: Full suite runs in ~17 minutes (1038 seconds)

#### Coverage Achievement

- **Starting coverage**: 42%
- **Final coverage**: 91%
- **Tests passing**: 177 / 177 (0 failures)
- **Test files**: 13 files totaling ~5,800 lines of test code

#### Git Commits

| Commit | Hash | Description |
|--------|------|-------------|
| 1 | `b707874ee` | Add E2E tests for referral flow endpoints |
| 2 | `d67a111b6` | Add E2E tests for join-org endpoint + fix referral UTC import |
| 3 | `a3b935d7b` | Add E2E tests for link/unlink integration endpoints |
| 4 | `4048daa72` | Add E2E tests for Auth0 login callback endpoint |
| 5 | `9780ce915` | Add comprehensive E2E and unit tests for users router (42% → 91% coverage) |

## Verification

1. **Unit tests**: Run `pytest tests/sybill_py/dal/queries/users/ -v` after each DAL commit
2. **Router tests**: Run `pytest tests/routers/users/ -v` after each router commit
3. **Full suite**: Run `pytest tests/ -x` after final commit to ensure no regressions
4. **Lint/format**: Run pre-commit hooks (ruff) before each commit
5. **Type check**: Verify no type errors introduced via DAL method signatures

---

## Additional E2E Test Coverage & Cases Covered

This section documents the comprehensive E2E test suite added to achieve 91% coverage of the users router. All tests follow the pattern of HTTP-level testing via `AsyncClient` + `ASGITransport` with real MongoDB and mocked external services.

### Test File Organization

```
tests/routers/users/
├── conftest.py                      # Shared fixtures and mock factories
├── test_e2e_account_mgmt.py        # Account suspension/activation
├── test_e2e_affiliate_extended.py  # Affiliate program & copilot registration
├── test_e2e_auth0_login.py         # Auth0 login callback
├── test_e2e_auth_misc.py           # Email setting, calendar sync, heartbeat
├── test_e2e_integrations.py        # Link/unlink integrations
├── test_e2e_join_org.py            # Organization creation/joining
├── test_e2e_referrals.py           # Referral flow
├── test_e2e_simple_endpoints.py    # Simple GET/POST endpoints
├── test_reader_replacements.py     # DAL reader method tests
├── test_refresh_integrations.py    # Integration refresh logic
├── test_set_email.py               # Email setting logic
├── test_simple_reads.py            # Simple read operations
└── test_unit_helpers.py            # Helper function unit tests
```

### Detailed Test Coverage by Endpoint

#### 1. Account Management (9 tests)

**`test_e2e_account_mgmt.py`** - Account lifecycle management

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestSuspendUserAccountM2M` | 3 tests | Success (CRM + calendar cleanup), user not found, already suspended |
| `TestSuspendUserAccount` | 1 test | Success via non-M2M endpoint |
| `TestActivateUserAccount` | 5 tests | Success (integration reactivation), calendar resync trigger, user not found, org not found, account already active |

**Key behaviors tested**:

- CRM integration state changes (`SUSPENDED` state event)
- Calendar integration cleanup (stops watchers)
- Integration status updates across all integrations
- State event tracking (`SUSPENDED`, `ACTIVATED`)
- Background task enqueueing for calendar resync

#### 2. Affiliate Program & Copilot Registration (12 tests)

**`test_e2e_affiliate_extended.py`** - Affiliate program features

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestUpdateAffiliateLink` | 4 tests | New link creation, link update, PayPal email update, user not found |
| `TestUpdateAffiliatePaypalEmail` | 3 tests | New PayPal email, update existing, user not found |
| `TestRegisterUserCopilot` | 5 tests | New registration, existing registration (usages increment), referral attribution, contactForSales flag, user not found |

**Key behaviors tested**:

- Affiliate link generation and validation
- PayPal email storage for payouts
- Copilot public registration with referral tracking
- Usage counter increments on repeat registration
- ContactForSales flag handling

#### 3. Auth0 Login Callback (5 tests)

**`test_e2e_auth0_login.py`** - Auth0 OAuth callback handling

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestOnUserLoginViaAuth0` | 5 tests | New user multi-org routing, legacy flow routing, cache invalidation, background task orchestration, response shape validation |

**Key behaviors tested**:

- Multi-org vs legacy Auth0 flow detection
- User creation with system identity
- Organization join/create logic
- Cache invalidation for user info
- Background task enqueueing (org setup)
- Response format (`{user, organization}`)

#### 4. Email Setting, Calendar Sync & Heartbeat (35 tests)

**`test_e2e_auth_misc.py`** - Authentication and sync miscellaneous endpoints

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestSetUserEmail` | 4 tests | Set email if unset, handle already-set case, user not found, validation errors |
| `TestSyncUserCalendarEvents` | 26 tests | Google/Outlook/Zoho calendar sync success, suspended user handling, user not found, creds not found, expiration checks, watcher setup/update, calendar cleanup |
| `TestDesktopAppHeartbeat` | 5 tests | New heartbeat creation, heartbeat update, user not found, invalid timestamp, future timestamp rejection |

**Key behaviors tested**:

- Email setting with validation
- Google Calendar: token refresh, watcher update, background sync trigger
- Outlook Calendar: MS Graph credential handling, watcher setup retry on update failure
- Zoho Calendar: token refresh, watcher setup
- Suspended user handling (returns early without error)
- Desktop app presence tracking with timestamp validation

#### 5. Integration Linking/Unlinking (12 tests)

**`test_e2e_integrations.py`** - Calendar and email integration management

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestLinkCalendar` | 4 tests | Google Calendar link success, calendar not provided (400), creds not fetched (403), user not found (404) |
| `TestLinkIntegrationToAccount` | 4 tests | Google email link success, no calendar integration (400), creds not fetched (403), user not found (404) |
| `TestUnlinkIntegrationFromAccount` | 4 tests | Email unlink success, nonexistent email (404), unsupported type (400), user not found (404) |

**Key behaviors tested**:

- Calendar integration creation with `INITIALIZED` state event
- Email account linking with provider validation
- Integration unlinking with `CANCELLED` state event append
- Credential fetching and validation
- Provider-specific error handling

#### 6. Organization Creation & Joining (8 tests)

**`test_e2e_join_org.py`** - Organization lifecycle

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestCreateNewOrg` | 4 tests | New org creation (admin role), org with invites, ineligible invite email (400), user already in org (400) |
| `TestJoinExistingOrg` | 4 tests | Join existing org (member role), nonexistent org (404), wrong email domain (403), user not found (404) |

**Key behaviors tested**:

- Organization creation with admin role assignment
- Email domain-based organization discovery
- Invite email validation and processing
- Domain matching for org eligibility
- Already-in-org guard (prevents duplicate membership)
- User role assignment (admin for new org, member for join)

#### 7. Referral Flow (10 tests)

**`test_e2e_referrals.py`** - Referral program endpoints

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestAddReferralInfo` | 3 tests | New referral (enqueues background task), existing referral (no action), user not found (404) |
| `TestGetReferrals` | 3 tests | Returns referrals for affiliate, empty for non-affiliate, user not found (404) |
| `TestSendReferralEmail` | 4 tests | Send to new recipient, send to existing non-Sybill recipient, Sybill user rejection (400), no affiliate info (400) |

**Key behaviors tested**:

- Referral info storage with attribution
- Background task enqueueing for referral processing
- Referral recipient validation (must not be existing Sybill user)
- Email send event tracking
- Affiliate validation before referral
- Referral doc upsert with email send history

#### 8. Simple GET/POST Endpoints (19 tests)

**`test_e2e_simple_endpoints.py`** - Various utility endpoints

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestGetUserInfo` | 3 tests | Success with org, user not found, user without org |
| `TestGetUserStats` | 3 tests | Success, bad date range, value error handling |
| `TestGetWrapped2025` | 2 tests | Success, user not found |
| `TestGetExtendedInfo` | 2 tests | Success, user not found |
| `TestGetDesktopAppHeartbeat` | 2 tests | Success, user not found |
| `TestFindUsersBySystemIdentPrefix` | 1 test | Success |
| `TestGetCalendarLastSyncedAt` | 3 tests | Success, suspended user, user not found |
| `TestSetOnboardingInfo` | 2 tests | Success, user not found |
| `TestValidateInviteEmailsExistingUserNoAccount` | 1 test | Existing user without account is eligible |
| `TestGetAdditionalDashboardUserInfoByEmail` | 1 test | With extended info |

**Key behaviors tested**:

- User info retrieval with organization data
- User stats calculation with date range validation
- Wrapped 2025 data retrieval
- Extended info handling
- Desktop app heartbeat retrieval
- System identity prefix search
- Calendar last synced timestamp
- Onboarding info upsert
- Invite email eligibility validation
- Dashboard user info by email

#### 9. Integration Refresh Logic (13 tests)

**`test_refresh_integrations.py`** - Integration token refresh and setup retry

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestRefreshIntegrationsGeneral` | 3 tests | User not found, suspended user, skip refresh for long expiration |
| `TestRefreshGoogleCalendar` | 1 test | Token refresh success |
| `TestRefreshOutlookCalendar` | 3 tests | Token refresh success, setup retry when update fails, MS Graph creds not found |
| `TestRefreshStripeIntegration` | 1 test | Token refresh success |

**Key behaviors tested**:

- Expiration time checking (skip if >30 days remaining)
- Google Calendar OAuth token refresh
- Outlook Calendar MS Graph token refresh
- Outlook watcher setup retry on update failure
- Stripe OAuth token refresh
- Suspended user early return
- Credential validation before refresh

#### 10. DAL Reader Method Tests (7 tests)

**`test_reader_replacements.py`** - Direct DAL reader method testing

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestGetUserByPrincipal` | 2 tests | Found, not found |
| `TestValidateInviteEmails` | 3 tests | New email eligible, active Sybill user not eligible, invited user not eligible |
| `TestGetEligibleOrgs` | 2 tests | Returns eligible orgs with member count, user not found |

**Key behaviors tested**:

- System identity principal lookup
- Invite eligibility validation (checks `isSybillUser` and invite status)
- Organization eligibility with member counts
- Case-insensitive email matching

#### 11. Simple Read Operations (12 tests)

**`test_simple_reads.py`** - Basic read endpoints

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestShowUser` | 2 tests | Found, not found |
| `TestShowUserExtraInfo` | 2 tests | Found, not found |
| `TestGetUserByEmail` | 2 tests | Found, not found |
| `TestGetDashboardUserRef` | 3 tests | With extended info, without extended info, not found |
| `TestGetDashboardUserInfoByEmail` | 2 tests | Found, not found |
| `TestGetOnboardingInfo` | 3 tests | Exists, no onboarding data, user not found |

**Key behaviors tested**:

- User document retrieval by ID
- User extra info retrieval
- User lookup by email (case-insensitive)
- Dashboard user reference construction
- Onboarding info retrieval with fallback handling

#### 12. Email Setting Logic (2 tests)

**`test_set_email.py`** - Email setting with validation

| Test Class | Test Cases | Scenarios Covered |
|------------|-----------|-------------------|
| `TestSetUserEmail` | 2 tests | Set email if unset, handle already-set case |

**Key behaviors tested**:

- Primary email setting for new users
- Already-set email handling (no overwrite)

#### 13. Helper Function Unit Tests (33 tests)

**`test_unit_helpers.py`** - Isolated unit tests for complex helper functions

| Function Under Test | Test Count | Scenarios Covered |
|---------------------|------------|-------------------|
| `_handle_auth0_login` | 3 tests | New user without org, existing user with invite, existing user with missing org (404) |
| `_finalize_user_setup` | 1 test | Integration/preference userId setup, system identity addition |
| `_setup_user_integrations` | 2 tests | Google Calendar integration, Stripe integration |
| `_setup_org_background_tasks` | 1 test | Task enqueueing |
| `_process_invite_emails` | 3 tests | New user invites, existing Sybill user skip, already-invited skip |
| `_initialize_integrations_after_org_join` | 7 tests | Google/Outlook/Zoho calendar initialization, email integration setup, creds failure handling, already-initialized skip |
| `_suggested_messages_for_onboarding` | 4 tests | No email integration, no provider info, Outlook returns empty, user not found |
| `_handle_legacy_auth0_login` | 4 tests | New user creates org+user, existing user with invite, joins existing org, unsupported identity rejection |
| `_handle_existing_calendar_cleanup` | 2 tests | Stops active calendar watcher, skips if no active calendar |
| `_sync_user_calendar_common` | 8 tests | Google/Outlook/Zoho calendar resync, user not found (404), suspended user (early return), creds not found, watcher update failure retry, update result None (400) |
| `_sync_proxycurl_and_upsert_person` | 2 tests | Success with person, success without person |
| `_suggested_messages_google_mail` | 3 tests | Filtered message returns, same-domain skip, missing author skip |
| `_setup_unified_to_entities_and_trigger_sync` | 2 tests | Success, existing integration deletion |

**Key behaviors tested**:

- Auth0 identity handling (multi-org vs legacy)
- User setup finalization (integration/preference userId)
- Integration initialization after org join
- Calendar provider detection and setup
- Email integration setup with credential validation
- Suggested messages for onboarding (email filtering)
- Calendar sync orchestration (Google/Outlook/Zoho)
- Suspended user early returns
- Watcher update failure recovery
- Proxycurl LinkedIn enrichment
- Unified CRM entity setup

### Test Infrastructure & Patterns

#### Fixtures in `conftest.py`

```python
@pytest.fixture
async def test_client(mock_request, mocker):
    """Extended test client with mocked dependencies."""
    # Mock external services
    mocker.patch("routers.users.api_tokens", return_value=mock_api_tokens())
    mocker.patch("routers.users.email_client", return_value=mock_email_client())
    mocker.patch("routers.users.rate_limiter", return_value=mock_rate_limiter())

    # Mock calendar/CRM services
    mocker.patch("sybill_py.components.calendar.google.GoogleCalendarService")
    mocker.patch("sybill_py.components.calendar.outlook.OutlookCalendarService")
    mocker.patch("sybill_py.components.crm.salesforce.SalesforceService")

    # Return AsyncClient for HTTP testing
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client
```

#### Common Test Helpers

```python
async def create_test_user_with_account(org_id: UUID, email: str) -> User:
    """Create user with userAccountInfo for testing."""

async def create_test_org(acc_cn: str, email_domains: list[str]) -> OrgInfo:
    """Create organization for testing."""

async def create_test_integration(user_id: UUID, type: str, state: str) -> dict:
    """Create integration document for testing."""

async def create_test_calendar_integration(user_id: UUID, provider: str) -> dict:
    """Create calendar integration with watcher for testing."""
```

#### Mocking External Services

All tests mock external API calls:

- **Auth0**: User profile fetching, token exchange
- **Stripe**: OAuth token refresh
- **Google Calendar**: Token refresh, watcher setup/update
- **Outlook Calendar**: MS Graph token refresh, watcher setup
- **Zoho Calendar**: Token refresh, watcher setup
- **Email services**: SendGrid, SES
- **CRM services**: Salesforce, HubSpot state changes
- **Background tasks**: Task enqueueing (verified via `mock.call_args`)

### Edge Cases & Error Handling Covered

#### HTTP Error Codes

- **400 Bad Request**: Invalid input, already-in-org, Sybill user referral rejection, unsupported integration type
- **403 Forbidden**: Credentials not fetched, wrong email domain for org join
- **404 Not Found**: User not found, org not found, integration not found, email not found

#### Null/Empty Handling

- User without org
- User without extended info
- User without onboarding data
- No affiliate info
- No calendar integration
- Empty referral list

#### Idempotency & State Guards

- Already suspended account (no duplicate state event)
- Already activated account (no duplicate state event)
- Already initialized integration (skip re-initialization)
- Already invited user (skip duplicate invite)
- Already-set email (no overwrite)
- Existing referral (no duplicate background task)

#### Time-Based Edge Cases

- Calendar sync skip for long expiration (>30 days)
- Desktop app heartbeat future timestamp rejection
- Email send event timestamp tracking
- State event timestamp tracking

### Coverage Gaps & Future Work

**Not yet covered** (requires extensive setup):

- File upload endpoints
- Complex multi-step flows (e.g., full onboarding from signup to first call)
- Webhook endpoints (calendar events, CRM sync notifications)
- Admin-only endpoints (bulk operations, data exports)
- Rate limiting behavior (requires concurrent requests)
- Database transaction rollbacks (requires MongoDB transaction support)

**Coverage by file**:

- `routers/users.py`: **91% coverage** (up from 42%)
- `routers/users/creation_helpers.py`: **87% coverage**
- `routers/users/integration_helpers.py`: **84% coverage**

### Running the Full Test Suite

```bash
# Full users router test suite (~17 minutes)
pytest tests/routers/users/ -v

# Specific test file
pytest tests/routers/users/test_e2e_referrals.py -v

# With coverage report
pytest tests/routers/users/ --cov=routers.users --cov-report=term-missing

# Parallel execution (faster, requires pytest-xdist)
pytest tests/routers/users/ -n auto
```

### Test Maintenance Notes

1. **Shadow tests can be deleted** after production validation (see Follow-up #2)
2. **Mock external services** to avoid flaky tests and external dependencies
3. **Use real MongoDB** for data consistency and query validation
4. **Factory functions** in `conftest.py` should be extended for new test scenarios
5. **HTTP-level testing** preferred over direct function calls for endpoint coverage
6. **Background task verification** via `mock.call_args` inspection (tasks not actually executed)
