# Omnical — User-Managed Sharing (API + Frontend)

**Date:** 2026-09-08
**Project:** `omnical` (RustiCal `omnical-scheduling` branch)
**Goal:** Let registered users create their own shared group calendars/addressbooks and
control member access from the web UI — no admin CLI needed.

---

## 1. Problem Statement

Today, only the admin (via `ssh router rustical principals …`) can:
- Create group principals
- Assign/remove memberships
- Create collections inside group principals

Users can create their **own** calendars/addressbooks from the frontend (via MKCOL over
DAV) but cannot share with anyone else or manage access. The existing CalDAV ACL model
(RFC 3744 privilege sets) is too heavy — we want a simple "invite member / remove
member" metaphor backed by RustiCal's built-in `PrincipalType::Group` + `memberships`
table, which already gives read/write access to all group collections for all members.

---

## 2. Target Flow (User Perspective)

1. Alice (logged into the web UI) clicks **"New Group"**.
2. She names it ("Book Club"), optionally picks an icon/color.
3. She adds Bob and Chris (existing Omnical users) to the group.
4. Server creates `bookclub` as a `PrincipalType::Group`, adds memberships, seeds
   a shared calendar (`bookclub`), tasks list (`bookclub-tasks`), and addressbook
   (`bookclub-contacts`).
5. Bob and Chris now see **Book Club** in their calendar/contacts apps (via DAVx5,
   Thunderbird, Apple) — same as the existing `family` group.
6. Alice can later add/remove members or delete the group (and its collections).

---

## 3. Architecture

### 3.1 Component Map

```
Browser (Lit Web Components)
    │  fetch() JSON
    ▼
Axum Router  (/frontend/api/v1/…)
    │  authenticated via session (existing Principal extractor)
    ▼
ApiService  (new crate: crates/api/)
    │  calls AuthenticationProvider + CalendarWriteStore + AddressbookWriteStore
    ▼
Store Layer  (existing: SqlitePrincipalStore, SqliteCalendarStore, SqliteAddressbookStore)
```

### 3.2 Route Tree (additions)

All new routes live under `/frontend/api/v1/` so the existing session
auth works (session cookie set by login, `Principal` extracted automatically).

```
/frontend/api/v1/
├── groups/
│   ├── GET                    → list all groups the user belongs to
│   ├── POST                   → create a new group (name, optional color)
│   └── :group_id/
│       ├── GET                → group details (name, members, collections)
│       ├── DELETE             → delete group (owner only)
│       └── members/
│           ├── GET            → list members
│           ├── POST           → add member { user_id }
│           └── :member_id/
│               └── DELETE     → remove member
├── users/
│   └── GET?q=                 → search existing users (for member autocomplete)
└── collections/
    └── POST                   → create a collection inside a group
        { group_id, type: "calendar"|"addressbook", name, color }
```

### 3.3 Authorization Model

- **Group owner** = the user who created the group. Only the owner can add/remove members
  or delete the group. Owner is stored in a new `group_owners` table.
- **Group member** = any principal with a membership row pointing at the group. Members get
  read/write to all group collections (this is the existing RustiCal security model —
  membership = full access).
- **Self-service only** — users cannot touch groups they don't own (except listing
  groups they belong to).

### 3.4 Database Changes (Migration)

```sql
-- New migration: 20260908120000_group_ownership
CREATE TABLE group_owners (
    group_id TEXT NOT NULL,   -- the group principal id
    owner_id TEXT NOT NULL,   -- the user principal id who created it
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (group_id),
    FOREIGN KEY (group_id)   REFERENCES principals (id) ON DELETE CASCADE,
    FOREIGN KEY (owner_id)   REFERENCES principals (id) ON DELETE CASCADE
);
```

No changes to existing tables. Groups are still `principals` with
`principal_type = "GROUP"` and memberships are still in the `memberships` table.

### 3.5 New Crate: `crates/api/`

```
crates/api/
├── Cargo.toml          (depends on: axum, serde, serde_json, rustical_store, …)
├── src/
│   ├── lib.rs          (router builder: pub fn api_router(…) -> Router)
│   ├── groups.rs       (list, create, get, delete)
│   ├── members.rs      (list, add, remove)
│   ├── users.rs        (search)
│   ├── collections.rs  (create collection inside group)
│   └── error.rs        (ApiError → axum IntoResponse)
```

### 3.6 Frontend Changes

New pages/routes under `/frontend/user/{user}/`:

| Route | Template | Purpose |
|---|---|---|
| `/frontend/user/{user}/groups` | `pages/groups.html` | List user's groups, "New Group" button |
| `/frontend/user/{user}/groups/new` | `pages/group_new.html` | Create group form |
| `/frontend/user/{user}/groups/{group}` | `pages/group_detail.html` | Group details, member list, add/remove, collections |

New Lit web components (in `crates/frontend/js-components/lib/`):

| Component | File | Purpose |
|---|---|---|
| `<group-list>` | `group-list.js` | Renders list of groups from API |
| `<member-picker>` | `member-picker.js` | Autocomplete user search + add button |
| `<group-create-form>` | `group-create-form.js` | Name + color + initial members form |

The existing navigation in `templates/pages/user.html` gets a new **"Groups"** tab
linking to `/frontend/user/{user}/groups`.

---

## 4. Implementation Phases

### Phase A — Store Layer (SQLite + Trait)

1. **Migration** (`crates/store_sqlite/migrations/20260908120000_group_ownership.sql`):
   the `group_owners` table above.
2. **Trait extension** (patch `AuthenticationProvider` in `crates/store/src/auth/mod.rs`):
   - `list_groups_for_user(user_id) -> Vec<(String, String)>` — groups the user belongs to,
     with displaynames
   - `get_group_owner(group_id) -> Option<String>` — who created it
   - `set_group_owner(group_id, owner_id)` — set on group creation
   - `search_users(query) -> Vec<(String, String)>` — for autocomplete
3. **SQLite impl** (`crates/store_sqlite/src/principal_store.rs`): implement the new methods.
4. **No-op defaults**: add default implementations to the trait so existing code compiles
   without changes.

### Phase B — REST API (new crate)

1. **Scaffold** `crates/api/` with `Cargo.toml`, `lib.rs` exporting `api_router()`.
2. **Groups** (`groups.rs`):
   - `GET /groups/` — calls `list_groups_for_user`, returns JSON array
   - `POST /groups/` — calls `insert_principal` (GROUP), `set_group_owner`,
     `add_membership` for each member, seeds default collections
   - `GET /groups/:id` — returns group info + members
   - `DELETE /groups/:id` — owner check, then `remove_principal` (cascade deletes
     memberships + group_owners + collections)
3. **Members** (`members.rs`):
   - `GET /groups/:id/members/` — calls `list_members`
   - `POST /groups/:id/members/` — owner check, `add_membership`
   - `DELETE /groups/:id/members/:member_id` — owner check, `remove_membership`
4. **Users** (`users.rs`):
   - `GET /users/?q=` — calls `search_users`, returns JSON array of `{id, displayname}`
5. **Collections** (`collections.rs`):
   - `POST /collections/` — calls `insert_calendar` or `insert_addressbook` inside the
     specified group principal
6. **Wire into main app** (`src/app.rs`): mount `api_router()` under
   `/frontend/api/v1/` inside the `frontend_router()` (so session auth applies).

### Phase C — Frontend

1. **Backend routes** (Askama handlers in `crates/frontend/src/routes/`):
   - `route_groups` — renders groups list page
   - `route_group_new` — renders create form
   - `route_group_detail` — renders group detail page
2. **Lit web components** (in `crates/frontend/js-components/lib/`):
   - `<group-list>` — fetches `GET /api/v1/groups/`, renders cards
   - `<group-create-form>` — form → `POST /api/v1/groups/`, redirects to detail
   - `<member-picker>` — `GET /api/v1/users/?q=`, typeahead, add button
3. **Rebuild JS bundle** (`deno task build` or Vite in `js-components/`).
4. **Wire navigation** — add "Groups" tab to `templates/pages/user.html`.

### Phase D — Build & Deploy

1. **Cross-compile** via `~/router-dav/scripts/build-rust.sh` (clang recipe D).
2. **Deploy** via `~/router-dav/deploy.sh` (push binary + config, restart services,
   migration auto-applied on startup).
3. **Smoke test** on the local x86_64 build before deploying to the router.

---

## 5. API Contract

### `GET /api/v1/groups/`
Response:
```json
[
  {
    "id": "bookclub",
    "displayname": "Book Club",
    "owner": true,
    "member_count": 3,
    "collections": ["calendars/bookclub", "calendars/bookclub-tasks", "addressbooks/bookclub-contacts"]
  }
]
```

### `POST /api/v1/groups/`
Request:
```json
{
  "id": "bookclub",
  "displayname": "Book Club",
  "members": ["bob@gmail.com", "chris@carltonaudio.com"],
  "collections": {
    "calendar": true,
    "tasks": true,
    "addressbook": true
  }
}
```
Response: `201 Created` with the group object.

### `POST /api/v1/groups/:id/members/`
Request:
```json
{ "user_id": "dave@gmail.com" }
```
Response: `201 Created`

### `DELETE /api/v1/groups/:id/members/:member_id`
Response: `204 No Content`

### `GET /api/v1/users/?q=bob`
Response:
```json
[
  { "id": "bob@gmail.com", "displayname": "Bob Smith" },
  { "id": "bobby@example.com", "displayname": "Bobby Tables" }
]
```

### `POST /api/v1/collections/`
Request:
```json
{
  "group_id": "bookclub",
  "type": "calendar",
  "displayname": "Book Club Meetings",
  "color": "#ff6600"
}
```
Response: `201 Created`

---

## 6. Constraints & Decisions

| Constraint | Decision |
|---|---|
| No CDDL/SCIM/DAV-ACL | Simple JSON REST API; groups = RustiCal `PrincipalType::Group`; membership = full r/w |
| Frontend auth is session-based (tower-sessions) | API routes sit inside the existing session middleware — no new auth |
| Lit 3 (ES modules, Vite bundle) | Keep existing stack; no new JS framework |
| No role/permission system on `Principal` | Add `group_owners` table (separate from memberships); owner = full control |
| 42 MB overlay budget (binary + DB) | The API crate adds ~200-300 lines of Rust — negligible binary size. Frontend adds ~500 bytes to the JS bundle per new component |

---

## 7. Files Changed

### New files
```
crates/api/Cargo.toml
crates/api/src/lib.rs
crates/api/src/groups.rs
crates/api/src/members.rs
crates/api/src/users.rs
crates/api/src/collections.rs
crates/api/src/error.rs
crates/store_sqlite/migrations/20260908120000_group_ownership.sql
crates/frontend/public/templates/pages/groups.html
crates/frontend/public/templates/pages/group_new.html
crates/frontend/public/templates/pages/group_detail.html
crates/frontend/src/routes/groups.rs
crates/frontend/js-components/lib/group-list.js
crates/frontend/js-components/lib/group-create-form.js
crates/frontend/js-components/lib/member-picker.js
```

### Modified files
```
crates/store/src/auth/mod.rs           → add list_groups_for_user, get_group_owner,
                                          set_group_owner, search_users to trait
crates/store_sqlite/src/principal_store.rs → implement new trait methods
crates/frontend/Cargo.toml             → add rustical_api dependency
crates/frontend/src/lib.rs             → register new routes, mount api_router
crates/frontend/src/routes/mod.rs      → register new route modules
crates/frontend/public/templates/pages/user.html → add "Groups" nav tab
Cargo.toml                             → workspace member: crates/api
Cargo.lock                             → auto-updated
src/app.rs                             → pass auth_provider to api_router
```

---

## 8. Verification: matrix row 21

| # | Test | Command / Method | Expected |
|---|---|---|---|
| 21 | User group self-service | Login as a regular user, create group, add member, verify member sees group collections, remove member, delete group | Group appears in member's DAV clients; removal revokes access; deletion cleans up collections + memberships |

## 9. Notes / Known Issues

- **Invite response not showing on iPhone Calendar** (diagnosed 2026-09-08 — two root causes):
  1. **FIXED (2026-09-08):** `parse_event` in `crates/scheduling/src/ics.rs` did not track
     nested components, so a VALARM's `UID:` line (iOS writes one into every alarm,
     `X-WR-ALARMUID`) overwrote the VEVENT's UID. Every attendee REPLY was then matched
     against the wrong UID — `find_calendar_objects_by_uid` never found the organizer's
     stored copy, and the REPLY filed into the organizer's inbox carried the wrong UID so
     iOS discarded it too. Confirmed in production: the inbox REQUEST for an
     iPhone-organized event was stored as `req-A215B080-…ics` (the VALARM UID) instead of
     `req-77A7E01C-…ics` (the event UID). Fix: depth-tracked component parsing + 2
     regression tests (`valarm_uid_does_not_shadow_event_uid`,
     `valarm_attendee_is_not_an_event_attendee`).
   2. **IN PROGRESS — inbound iMIP processing (IMAP ingestion), 2026-09-08
      session:** when an invite goes to an *external* attendee (or to a user
      before their Omnical principal exists — the `invite-lyns-chris` test:
      chris@carltonaudio.com replied "yes" by email while he was not yet a
      principal), the REPLY email lands in the organizer's mailbox on the
      external mail provider and is never ingested; the server-side PARTSTAT
      never updates. Fix being built: poll each configured identity's mailbox
      over IMAP for `METHOD:REPLY` iTIP messages and feed them into
      `Scheduler::deliver_reply`. Interim workaround: invite the user now
      that their principal exists (internal CalDAV path works, including
      with fix #1).

      **Implementation state (all on the uncommitted `omnical-scheduling`
      working tree in `~/router-dav/rustical`, additive on top of the
      deployed scheduling build):**
      - **DONE, compiles** — new `crates/scheduling/src/tls.rs` (the
        aws-lc `tls_connector` moved out of smtp.rs, shared by SMTP+IMAP,
        regression test moved with it; smtp.rs's `debug!` transcript now
        redacts `AUTH PLAIN` lines).
      - **DONE, compiles** — `crates/scheduling/src/config.rs`:
        `ImapAccount { identity, host, port(993 implicit TLS only),
        username, password, mailbox (default INBOX), mark_seen (default
        true) }` + `SchedulingConfig.imap: Vec<ImapAccount>` and
        `imap_poll_secs` (default 120, floored to 30 in code). Old configs
        keep parsing (serde defaults); `[scheduling.imap]` empty ⇒ zero
        footprint. `toml` added as a dev-dep (already vendored).
      - **DONE, compiles** — `crates/scheduling/src/mime_parse.rs`: minimal
        RFC 5322/MIME walker — nested multipart recursion, boundary
        splitting, base64 + quoted-printable decoding, folded headers,
        extracts the first `text/calendar` part + the From/Sender address;
        reuses ics.rs's `split_params`/`unquote` (now `pub(crate)`).
      - **DONE, compiles** — `crates/scheduling/src/imap.rs`: hand-rolled
        IMAPS client (no new deps): greeting, `AUTHENTICATE PLAIN` with
        SASL-IR → `+`-continuation → quoted-`LOGIN` fallback, `SELECT`
        (UIDVALIDITY/EXISTS), `UID SEARCH UNSEEN`, `UID FETCH
        (BODY.PEEK[])` with `{N}`/`{N+}` literal framing and 2 MB oversize
        discard-while-in-sync, `UID STORE +FLAGS.SILENT (\Seen)`, `LOGOUT`.
        Credentials never logged (AUTHENTICATE/LOGIN redacted).
      - **DONE, compiles** — `crates/scheduling/src/scheduler.rs`:
        `ingest_imip_reply(mailbox_identity, reply_ics, from)` public entry
        (validates METHOD:REPLY, ORGANIZER == mailbox identity, organizer
        is a local principal — the mail-loop guard: without it
        `deliver_reply`'s remote branch would email the polled mailbox
        itself and re-ingest forever — then reuses `deliver_reply` to
        update the organizer's stored copies + file the inbox REPLY).
        `pick_reply_attendee` disambiguates multi-attendee (Outlook)
        REPLYs via the email From address, excluding NEEDS-ACTION.
        New `imap_accounts()`/`imap_poll_interval()` accessors.
      - **DONE, compiles** — `crates/scheduling/src/ingest.rs`: per-account
        poll tasks (`spawn_ingestion`, shutdown-future-per-task), process-
        local checked-UID set (cleared on UIDVALIDITY change, 100 k soft
        cap), ≤25 messages per poll, `mark_seen` only after successful
        ingest, non-iMIP mail fetched-and-untouched (stays unread, no
        flags), RetryLater disposition for transient failures. Wired into
        `cmd_serve` (root `src/lib.rs`) — no-op unless `[scheduling.imap]`
        configured. `crates/scheduling/src/error.rs` gained
        `SchedulingError::Imap`.
       - **FIXED (2026-09-08, follow-up session): unit tests GREEN —
         40 pass / 0 fail** (`SQLX_OFFLINE=true cargo test -p
         rustical_scheduling`). All code compiled; the 9 failures were
         test/parser nits, no design changes. All 7 diagnosed fixes
         below applied, plus 3 fixture nits they had been masking in
         the two fake-server flow tests: (a) the fake server sends ONE
         script entry per command, so `full_session_flow`'s two FETCH
         responses had to be merged into one reply entry; (b) the
         fixtures wrote `)` right after the `{N}` literal marker, but
         RFC 3501 framing ends the response line AT `{N}` — the `)`
         closing the FETCH list only arrives after the literal bytes
         (which is what `read_unit` was built for); (c) the first
         literal was `{14}` for a 15-byte body (`Hello, world!\r\n`):
         1. `imap::tests::fetch_uid_parsing` (+`full_session_flow`,
            `oversize…` which fail downstream of it):
           `parse_fetch_uid` collects digits from `text[after..]` — that
           slice starts AT the separating space, so
           `take_while(is_ascii_digit)` yields nothing → None. Fix:
           `text[after..].trim_start()` first.
        2. `imap::tests::login_handles_continuation` ("unexpected
           continuation: + "): the fake-server tests skip `connect()`, so
           the scripted greeting is never consumed — `login()` reads the
           GREETING as `first`, and read_response then trips over the
           real `+ ` line. Fix: `with_session` must consume one line
           (the greeting) after constructing the session.
        3. `config::tests::imap_defaults`: test parses
           `[[scheduling.imap]]`-shaped TOML into `SchedulingConfig`
           whose fields are top-level — parse the field list into
           `ImapAccount` directly instead.
        4. `mime_parse::folded_content_type_header`: the fixture's `\`
           line-continuations STRIP the leading folding whitespace (Rust
           string literal semantics), so the unfold path is never
           exercised — rebuild the fixture with explicit concatenation.
        5. `mime_parse::qp_decoding` "stray=z": decoder currently drops
           the malformed `=`; the (correct, lenient) test expects `=z`
           to pass through. Keep dangling end-of-input `=` dropped.
        6. `mime_parse::split_multipart…`: the epilogue after a closing
           `--B--` boundary is pushed as a part (2 parts, expected 1) —
           track a `closed` flag and skip the trailing push.
        7. `mime_parse::round_trips_own_imip_builder`: wrong assertion —
           the built mail's From IS `nick@example.com` (displayname
           None); assert Some, not None.
      - **Remaining work (next session picks up here):**
         1. ~~Apply the 7 test/parser fixes above; scheduling suite should
            be green (~40 tests).~~ **DONE 2026-09-08 — 40 pass / 0 fail**
            (`SQLX_OFFLINE=true cargo test -p rustical_scheduling`), via
            the 7 fixes plus the 3 fixture nits recorded above; caldav
            suite re-run clean afterwards (35 pass / 0 fail — the
            `qp_decode`/`split_multipart` behavior changes are
            strictly-more-correct and didn't regress it); the edits are
            rustfmt-clean and add no new clippy warnings (the crate's
            pre-existing fmt/clippy debt is untouched, left for item 3's
            gate).
        2. Add ingest integration tests to
           `crates/caldav/src/scheduling/tests.rs`: stored organizer event
           (PUT via router with vdirsyncer UA) + `ingest_imip_reply` →
           organizer copy PARTSTAT updated + `reply-<uid>-<attendee>.ics`
           in organizer inbox; negatives: foreign organizer, METHOD:
           REQUEST (invite addressed to the user), organizer not a local
           principal.
        3. Run the full gate: `SQLX_OFFLINE=true cargo check --workspace
           --all-targets` 0 errors/0 warnings, `cargo fmt`, clippy, and
           all suites (dav, scheduling, caldav, store_sqlite, root
           lib+bin+http-integration+integration).
        4. Extend `~/router-dav/scripts/render-router-config.sh`:
           `[[scheduling.imap]]` accounts rendered from
           `pass secrets/email/<id>/imap` (hosts per provider recon:
           gmail → `imap.gmail.com:993` user=identity;
           carltonaudio → `netsol-imap-oxcs.hostingplatform.com:993`;
           novo-ordo (nfcalaway + zero, shared login) →
           `imap.novo-ordo.com:993` user `nfcalaway@novo-ordo.com`;
           all three hostnames DNS-verified 2026-09-08).
        5. Cross-build + deploy via the existing deploy.sh flow (config
           lands before start; pre-deploy DB backup) + live test
           (re-invite an external address, reply by email, watch the
           organizer's event PARTSTAT + scheduling inbox).