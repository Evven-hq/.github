# Friend Groups — MVP Plan

**Status:** Draft for team review \n
**Owner:** Jagdeep \n
**Scope:** Permanent 1:1 expense relationships between existing Evven users \n

---

## 1. Product Requirements (PRD)

### 1.1 Problem statement

Right now, tracking shared expenses with one specific person requires creating a full `Group` — naming it, remembering it exists, navigating into it. Most 1:1 expense relationships (roommate, partner, one friend you split things with regularly) don't need a "group" mental model at all. Friend Groups give users a permanent, always-there relationship with one other person, discoverable without hunting through a group list.

### 1.2 User stories

- As a user, I want to add another Evven user as a friend by their user code, so we have a permanent space to track what we owe each other.
- As a user, I want to see a list of my friends with their current balance (owe me / I owe them / settled), so I can check status at a glance without opening each one.
- As a user, I want to open a friend and see our shared expense history and record new expenses or settlements, using the same flows I already know from groups.
- As a user, I want to remove a friend I no longer split expenses with, but only once we're settled up, so I don't lose track of a debt.
- As a user, I want re-adding someone I previously unfriended to restore our shared history, not start over.

### 1.3 Scope (MVP)

- Send a friend request by `user_code`; the recipient must accept before the relationship is active
- View and act on incoming/outgoing friend requests
- List friends with live pairwise balance
- View a single friend: balance, expense history, settlement history
- Create expenses and settlements within a Friend Group (via existing group expense/settlement endpoints)
- Unfriend (soft-remove), blocked while a balance is outstanding
- Re-add a previously-unfriended user reactivates the same relationship (history intact)

### 1.4 Non-goals (explicitly out of scope for MVP)

- Renaming or customizing a Friend Group (no name field is user-facing)
- More than 2 members in a Friend Group, or converting a Friend Group into a normal group
- Ghost/ placeholder-contact friends — separate, unrelated effort, not touched by this plan
- Notifications on friend-add, new expense, or settlement — no notification system exists yet
- Blocking or privacy controls (e.g., preventing someone from re-adding you) — MVP assumes goodwill between users who already know each other's user code
- Premium friend-limit gating — deferred to a later phase, see §6

### 1.5 UX flows

**Flow A — Send a friend request**
1. User taps "Add Friend" from a new Friends tab/section
2. Enters the other person's user code
3. Client validates format client-side, submits
4. Success → shows a "Request sent" confirmation, does **not** open a friend detail screen yet (nothing exists to view — no expenses possible until accepted)
5. Errors surface inline (code not found, already friends, request already pending, can't add yourself)
6. **Exception:** if the other person already has a pending request out to the current user, the backend auto-accepts instead of creating a second pending row — success state in this case goes straight to friend detail, since the relationship is now active

**Flow A2 — Respond to an incoming friend request**
1. User sees a "Friend Requests" badge/section on the Friends tab (checked on load — no push notifications exist, so this is pull-based)
2. Opens the list of incoming requests, each showing the requester's name/avatar
3. Accept → relationship becomes active, navigates to friend detail
4. Reject → request is removed, no trace kept
5. User can also see their own outgoing pending requests, with a "Cancel" action (same effect as reject, initiated by the requester)

**Flow B — View friends list**
1. User opens Friends tab
2. Sees list sorted by... (default: most recent activity, fallback name) — each row shows avatar, name, balance in green/red/gray
3. Tapping a row opens friend detail

**Flow C — Friend detail → add expense**
1. User opens a friend
2. Sees balance header, recent expenses/settlements feed
3. Taps "Add Expense" → reuses the existing group expense form, pre-scoped to the 2 members, split type still selectable (though equal is the common case for 2 people)
4. Same for "Settle Up" → reuses existing settlement form

**Flow D — Unfriend**
1. From friend detail, user taps overflow menu → "Remove Friend"
2. If balance ≠ 0: blocked, dialog explains they must settle first, links to "Settle Up"
3. If balance = 0: confirmation dialog → removes, returns to Friends list

**Flow E — Re-add a removed friend**
1. User adds a friend by user_code who was previously unfriended
2. Backend detects existing `REMOVED` Friend Group between the pair, reactivates it (`status → ACTIVE`) instead of creating new
3. Full expense/settlement history from before is immediately visible again

---

## 2. Technical Design

### 2.1 Database changes

Add two columns to the existing `groups` table — no new table. A Friend Group is a `Group` with a type flag.

```python
class GroupType(Enum):
    NORMAL = "NORMAL"
    FRIEND = "FRIEND"

class GroupStatus(Enum):
    PENDING = "PENDING"   # request sent, not yet accepted
    ACTIVE = "ACTIVE"     # accepted, fully usable
    REMOVED = "REMOVED"   # unfriended, history preserved
```

| Column | Type | Default | Notes |
|---|---|---|---|
| `group_type` | `SQLEnum(GroupType)` | `NORMAL` | non-nullable |
| `status` | `SQLEnum(GroupStatus)` | `ACTIVE` | non-nullable; only meaningful for `FRIEND` type today. Note `NORMAL` groups always stay `ACTIVE` — `PENDING` is a `FRIEND`-only concept. |

**Who requested whom:** reuse the existing `Group.created_by` column — no new field needed. Both `GroupMember` rows are created at request time, so membership queries don't need a separate "invited" concept; the guard described in §2.3 does the work of blocking action until `status == ACTIVE`.

**Rejected/cancelled requests are hard-deleted**, not soft-stated. Since expense/settlement creation is blocked while `PENDING` (§2.3 guard), there's never anything attached to a `PENDING` group besides its two `GroupMember` rows — deleting those first, then the `Group` row, is safe and avoids the (still-open) group-deletion FK cascade issue entirely, because we're not relying on cascade at all.

**Enum values must be uppercase to match existing DB convention** — this codebase has three prior bugs (`SplitType`, `AuthProvider`, `Role`) from Python enum values not matching the Postgres enum casing. Don't add a fourth.

**Uniqueness enforcement:** service-layer check (query for existing `FRIEND` group between the two user IDs before insert), not a DB constraint. A true DB-level constraint on an unordered pair needs a computed/sorted column or junction table — more infrastructure than this MVP needs. This leaves a theoretical race condition (two simultaneous "add friend" requests creating duplicate groups); acceptable for MVP, revisit if it's observed in practice.

**No cascade changes needed.** Unfriending is `status = REMOVED`, not a row delete, so the existing (still-unfixed) group-deletion FK cascade gap is never triggered by this feature.

### 2.2 API endpoints

New router, `routes/friends.py`, mounted at `/friends`:

| Method | Route | Description |
|---|---|---|
| `POST` | `/friends` | Send a friend request by `user_code` (or reactivate a `REMOVED` pair, or auto-accept a mutual reverse request) |
| `GET` | `/friends` | List all `ACTIVE` Friend Groups for current user, with balances |
| `GET` | `/friends/requests` | List current user's `PENDING` requests, split into `incoming` / `outgoing` |
| `POST` | `/friends/{group_id}/accept` | Accept an incoming request — recipient only |
| `DELETE` | `/friends/{group_id}/reject` | Reject an incoming request, or cancel an outgoing one — either party while `PENDING` |
| `GET` | `/friends/{group_id}` | Friend detail: profile, balance, expense/settlement history (only for `ACTIVE`) |
| `DELETE` | `/friends/{group_id}` | Unfriend an `ACTIVE` relationship (soft-remove, `status → REMOVED`) |

All auth-gated via existing `get_current_user` dependency, same as every other route.

**Reused unmodified:**
- `POST /groups/{group_id}/expenses` — Friend Group expenses go through this exact endpoint
- `POST /groups/{group_id}/settlements` — same
- `GET /groups/{group_id}/expenses`, `GET /groups/{group_id}/settlements` — same

### 2.3 Backend services

**New: `services/friend_service.py`**

```python
async def send_friend_request(current_user_id, target_user_code, db) -> Group
async def list_friend_requests(current_user_id, db) -> dict  # {"incoming": [...], "outgoing": [...]}
async def accept_friend_request(friend_group_id, current_user_id, db) -> Group
async def reject_friend_request(friend_group_id, current_user_id, db) -> None  # also handles cancel
async def list_friends(current_user_id, db) -> list[FriendResponse]
async def get_friend_detail(friend_group_id, current_user_id, db) -> FriendDetailResponse
async def unfriend(friend_group_id, current_user_id, db) -> None
```

- `send_friend_request` — resolve `user_code` → user, reject self-add. Check for an existing `FRIEND` group between the pair (any status):
  - `ACTIVE` → 400 "already friends"
  - `PENDING`, current user was the original requester → 400 "request already pending"
  - `PENDING`, current user is the original *recipient* (i.e., the other side already requested) → treat as mutual consent, call `accept_friend_request` on the existing row instead of creating a new one
  - `REMOVED` → reactivate directly to `ACTIVE` (previously-consented relationship, no re-approval needed)
  - none found → create `Group(group_type=FRIEND, status=PENDING, created_by=current_user_id)` + 2 `GroupMember` rows, in one transaction
- `list_friend_requests` — query `PENDING` `FRIEND` groups where current user is a member; split by whether `created_by == current_user_id` (outgoing) or not (incoming)
- `accept_friend_request` — must be called by a member who is **not** `created_by`; 403 otherwise. Flips `status → ACTIVE`.
- `reject_friend_request` — callable by either member while `status == PENDING`; hard-deletes both `GroupMember` rows then the `Group` row. Used for both "recipient rejects" and "requester cancels" — same operation, no status distinction needed.
- `list_friends` — query `Group` where `group_type=FRIEND, status=ACTIVE` and current user is a member; for each, call the existing `BalanceService.get_user_balance_in_group` to get the pairwise balance.
- `get_friend_detail` — same balance call, plus expense/settlement history scoped to the group. Returns 403/404 if `status != ACTIVE` — a pending request isn't viewable as a "friend" yet.
- `unfriend` — only valid on `ACTIVE` groups. Call `ExpenseRepository.has_pending_balance` (already fixed, BUG-05), block with 400 if `True`, else set `status = REMOVED`.

**New shared guard — FRIEND groups require `ACTIVE` status for any expense/settlement activity.** Add a check at the top of `expense_service.create_expenses` and `settlement_service.record_payment`: if the target group's `group_type == FRIEND` and `status != ACTIVE`, reject with 400. This one check covers two cases at once — blocking activity on a still-`PENDING` request, and closing a latent gap that would otherwise let someone add expenses to a `REMOVED` friend group too.

**Guards added to existing `services/group_service.py`**, gated on `group.group_type == GroupType.FRIEND`:
- `add_member` → reject, 400 "Friend Groups cannot add members"
- `remove_member` → reject, 400 "Use unfriend instead"
- `update_group` (rename) → reject, 400 "Friend Groups cannot be renamed"

### 2.4 Validation rules

| Rule | Enforced where | Failure response |
|---|---|---|
| `user_code` must resolve to an existing user | `friend_service.send_friend_request` | 404 |
| Cannot request yourself | `friend_service.send_friend_request` | 400 |
| Cannot send a duplicate request while one is already outgoing | `friend_service.send_friend_request` | 400 (mutual reverse request auto-accepts instead — not an error) |
| Cannot request someone you're already `ACTIVE` friends with | `friend_service.send_friend_request` | 400 |
| Only the recipient can accept a request | `friend_service.accept_friend_request` | 403 |
| Only the two parties on a `PENDING` request can reject/cancel it | `friend_service.reject_friend_request` | 403 |
| No expense/settlement activity while `status != ACTIVE` | `expense_service.create_expenses`, `settlement_service.record_payment` | 400 |
| Friend detail not viewable while `status == PENDING` | `friend_service.get_friend_detail` | 403/404 |
| Cannot unfriend with nonzero balance | `friend_service.unfriend` | 400, message includes current balance |
| Cannot add/remove members on a `FRIEND` group | `group_service.add_member` / `remove_member` | 400 |
| Cannot rename a `FRIEND` group | `group_service.update_group` | 400 |
| Only the two members can view/act on a Friend Group | reuse existing `is_member` check | 403 |

### 2.5 Migration strategy

Single Alembic migration:
1. Add `group_type` column, backfill all existing rows to `NORMAL`
2. Add `status` column, backfill all existing rows to `ACTIVE`
3. Both non-nullable with server defaults — safe for a live table, no downtime needed given table size

No data migration for ghost/placeholder users — explicitly out of scope (§1.4).

---

## 3. Frontend Implementation

### 3.1 Screens

- **Friends List** — new top-level tab/section. Row per friend: avatar, name, balance (colored), tap → detail. Includes a "Friend Requests" entry point/badge if any incoming requests exist.
- **Add Friend** — modal or dedicated screen with a single user-code input, submit button, inline error state. Success shows a "Request sent" confirmation rather than opening a detail screen.
- **Friend Requests** — new screen/section, two lists: incoming (Accept / Reject buttons per row) and outgoing (Cancel button per row).
- **Friend Detail** — reuses much of the existing group-detail visual language: balance header, activity feed (expenses + settlements interleaved by date), "Add Expense" / "Settle Up" CTAs, overflow menu with "Remove Friend." Only reachable for `ACTIVE` relationships.
- **Unfriend confirmation** — dialog/modal, two states (blocked-by-balance vs. confirm-removal).

### 3.2 Components

- `FriendListItem` — avatar, name, balance chip (reuse the group balance chip component if one exists)
- `AddFriendModal` — code input + validation + submit
- `FriendRequestListItem` — avatar, name, Accept/Reject or Cancel action depending on incoming vs. outgoing
- `FriendDetailHeader` — balance display, reuse from group detail if it's already componentized
- `UnfriendDialog` — two variants driven by a `blocked: boolean` prop
- Expense/settlement forms — **no new components**, reuse existing group expense/settlement forms with `groupId` pointed at the Friend Group

### 3.3 State management

- Friends list: fetch on tab focus, cache with existing data-fetching pattern (React Query / SWR / whatever's already in use for groups — mirror it exactly for consistency)
- Friend requests: fetch on tab focus alongside the friends list (no push notifications exist, so this is pull-based — refetch on focus is the only signal a new request has arrived)
- Balance updates: after adding an expense or settlement inside a Friend Group, invalidate both the friend-detail query and the friends-list query so the list balance stays in sync
- Send-request success: invalidate the outgoing-requests query, show confirmation (do not navigate to a detail screen — nothing exists yet)
- Accept success: invalidate friends-list, friend-requests, and friend-detail queries, then navigate to the new friend's detail
- Reject/cancel success: invalidate friend-requests query

### 3.4 Loading / error states

- Friends list: skeleton rows while loading; empty state ("No friends yet — add one to start splitting") if list is empty
- Add Friend: disable submit while in-flight; inline error text for 404 (code not found), 400 (self-add, duplicate)
- Friend detail: skeleton for header + feed while loading; error state with retry if the fetch fails
- Unfriend: disabled confirm button while in-flight; surface the 400 balance-blocked message inline, not as a toast that disappears

### 3.5 Mobile / Desktop behavior

- Add Friend as a modal on desktop, full-screen sheet on mobile (consistent with how other add/create flows likely already behave in the app — mirror existing pattern rather than introducing a new one)
- Friends list: single column on mobile; could support a two-pane list+detail layout on desktop if that pattern exists elsewhere, otherwise single column everywhere for MVP simplicity

---

## 4. Engineering Plan

### 4.1 PR breakdown (independent, mergeable)

| # | PR | Depends on |
|---|---|---|
| 1 | Migration: `group_type` + `status` columns (3-value enum incl. `PENDING`), backfill | — |
| 2 | Model changes: `GroupType`, `GroupStatus` enums on `Group` | 1 |
| 3 | Guards in `group_service.py` (block add/remove/rename for `FRIEND` type) | 2 |
| 4 | Shared `ACTIVE`-status guard in `expense_service.py` / `settlement_service.py` | 2 |
| 5 | `friend_service.py` + schemas — request/accept/reject/list/detail/unfriend | 2, 4 |
| 6 | `routes/friends.py`, registered in `main.py` | 5 |
| 7 | Backend tests (see §4.5) | 6 |
| 8 | Frontend: Friends List screen + `FriendListItem` | 6 |
| 9 | Frontend: Add Friend modal/sheet (request-based) | 8 |
| 10 | Frontend: Friend Requests screen (incoming/outgoing) | 8 |
| 11 | Frontend: Friend Detail screen (reusing expense/settlement forms) | 8 |
| 12 | Frontend: Unfriend flow + dialog | 11 |

PRs 1–7 are backend-only and can ship ahead of any frontend work — the API surface is fully testable via Swagger/curl before frontend starts.

### 4.2 Task estimation (rough, for a small team)

| PR | Estimate |
|---|---|
| 1–2 (migration + model) | 0.5 day |
| 3 (group_service guards) | 0.5 day |
| 4 (expense/settlement ACTIVE-status guard) | 0.5 day |
| 5 (friend_service + schemas, incl. request/accept/reject logic) | 1.5–2 days |
| 6 (routes) | 0.5 day |
| 7 (backend tests) | 1.5 days |
| 8 (friends list screen) | 1 day |
| 9 (add friend / send request) | 0.5–1 day |
| 10 (friend requests screen) | 1 day |
| 11 (friend detail, reusing forms) | 1–1.5 days |
| 12 (unfriend flow) | 0.5 day |

**Total: roughly 9–10.5 engineering days** (up from the ~7–8.5 in the instant-add version — the request/accept state machine and the extra screen are the main add), splittable across 2 people (backend/frontend) in parallel once PR 6 lands.

### 4.3 Dependencies

- Backend track (1→2→3/4→5→6→7) is fully self-contained.
- Frontend track can't start until PR 6 (routes live) is merged, or at minimum until the API contract (schemas) is settled in PR 5 — frontend can build against mocked responses matching the schema shape in parallel if you want to compress the timeline.

### 4.4 Rollout order

1. Ship backend PRs 1–7 to production first, unreleased to users (no frontend entry point yet — zero user-facing risk)
2. Verify via Swagger/manual testing in production before frontend work starts
3. Ship frontend PRs 8–12 behind a feature flag if one exists, else as a normal release once backend has been stable for a few days

### 4.5 Testing checklist

- [ ] Send friend request — success case, creates `PENDING` group with both members attached
- [ ] Send friend request — target user_code not found → 404
- [ ] Send friend request — self-request → 400
- [ ] Send friend request — duplicate outgoing request → 400
- [ ] Send friend request — already `ACTIVE` friends → 400
- [ ] Send friend request — mutual reverse request exists → auto-accepts instead of creating a duplicate
- [ ] Send friend request — pair has a `REMOVED` group → reactivates directly to `ACTIVE`, history intact, no approval step
- [ ] Accept request — success, `status → ACTIVE`
- [ ] Accept request — called by the requester instead of the recipient → 403
- [ ] Accept request — called by someone not on the request at all → 403
- [ ] Reject/cancel request — either party while `PENDING` → row hard-deleted, both `GroupMember` rows gone
- [ ] List requests — correctly splits incoming vs. outgoing
- [ ] List friends — returns correct pairwise balances, only for `ACTIVE`
- [ ] List friends — empty for a user with none
- [ ] Friend detail — non-member of the group → 403
- [ ] Friend detail — member, but group is `PENDING` → 403/404
- [ ] Add expense inside a `PENDING` Friend Group → 400 (blocked by the new ACTIVE-status guard)
- [ ] Add expense inside an `ACTIVE` Friend Group — works via existing endpoint, balance updates correctly
- [ ] Settle up inside an `ACTIVE` Friend Group — works via existing endpoint, balance zeroes out correctly
- [ ] Unfriend — blocked with nonzero balance, correct error message
- [ ] Unfriend — succeeds at zero balance, `status` flips to `REMOVED`
- [ ] Add expense inside a `REMOVED` Friend Group → 400 (same guard closes this gap too)
- [ ] Guard: `add_member` on a `FRIEND` group → 400
- [ ] Guard: `remove_member` on a `FRIEND` group → 400
- [ ] Guard: rename on a `FRIEND` group → 400
- [ ] Migration — backfill correctness on existing rows (spot check a few pre-existing groups get `NORMAL`/`ACTIVE`)

---

## 5. Architecture Review

### 5.1 Future scaling issues to watch

- **N+1 balance queries in `list_friends`.** This reuses `BalanceService.get_user_balance_in_group` per friend, same N+1 pattern documented as BUG-18 for group balances. For a user with 5–10 friends this is fine; if that number grows meaningfully, this needs the same batched-query fix eventually. Not worth solving preemptively for MVP.
- **Uniqueness race condition** (§2.1) — theoretical duplicate Friend Groups from concurrent requests. Low likelihood, low blast radius (worst case: two Friend Groups with the same pair, confusing but not data-destructive). Monitor rather than pre-solve.
- **`status` column on `groups` is currently single-purpose** (only meaningful for `FRIEND` type). If normal groups ever need soft-delete/archive semantics, this column already exists and can be extended rather than requiring a new one — a small design win, not a problem.

### 5.2 Simpler alternatives considered and rejected

- **Separate `friend_groups` table instead of reusing `groups`** — rejected because it would require duplicating expense/settlement/balance logic (exactly the concern raised about the original ChatGPT ghost-tables proposal). Reusing `Group` means zero new expense engine code.
- **Instant-add without consent** — this was the original MVP plan, then reversed: instant-add lets anyone who has (or leaks, or guesses) your `user_code` permanently attach a relationship to your account with no opt-in. Request/accept closes that, and reusing the existing `status` column plus `created_by` field means it cost one new enum value and one shared guard function rather than new infrastructure.
- **A dedicated `FriendRequest` table instead of `PENDING` status on `Group`** — rejected; would need its own model, repository, and a promotion step to create the actual `Group`/`GroupMember` rows on accept. Reusing `status` keeps one code path for the whole lifecycle (`PENDING → ACTIVE → REMOVED`) instead of two related-but-separate data shapes.
- **DB-level unique constraint on the pair** — rejected in favor of a service-layer check; the sorted-pair constraint is real infrastructure for a problem that's low-risk at MVP scale.

### 5.3 Backwards compatibility

- All existing `Group`/`GroupMember`/`GroupExpense`/`Settlement` code paths are untouched — Friend Groups are additive, gated entirely behind `group_type == FRIEND` checks in `group_service.py`.
- Existing groups backfill cleanly to `NORMAL`/`ACTIVE` with no behavior change.
- No existing API contracts change — `/friends` is a net-new namespace; `/groups/*` endpoints keep their current request/response shapes.
