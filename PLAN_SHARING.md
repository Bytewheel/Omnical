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
    2. **FIXED & DEPLOYED (2026-09-09) — inbound iMIP processing (IMAP
       ingestion):** when an invite goes to an *external* attendee (or to a user
       before their Omnical principal exists — the `invite-lyns-chris` test:
       chris@carltonaudio.com replied "yes" by email while he was not yet a
       principal), the REPLY email lands in the organizer's mailbox on the
       external mail provider and is never ingested; the server-side PARTSTAT
       never updates. Fix: poll each configured identity's mailbox over IMAP
       for `METHOD:REPLY` iTIP messages and feed them into
       `Scheduler::deliver_reply`. **Live-verified end-to-end 2026-09-09** (see
       remaining-work item 5 below for the transcript).

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
        true), ca_file (optional, default none — PEM file with extra
        TLS trust anchors for providers serving an incomplete chain;
        added 2026-09-09 for the novo-ordo fix) }` +
        `SchedulingConfig.imap: Vec<ImapAccount>` and
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
       - **Remaining work — ALL DONE (2026-09-09; deployed to the router
          and live-verified end-to-end):**
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
            principal. — **DONE 2026-09-09: 12/12 pass** (`SQLX_OFFLINE=true
            cargo test -p rustical_caldav scheduling`).
         3. Run the full gate: `SQLX_OFFLINE=true cargo check --workspace
            --all-targets` 0 errors/0 warnings, `cargo fmt`, clippy, and
            all suites (dav, scheduling, caldav, store_sqlite, root
            lib+bin+http-integration+integration). — **DONE 2026-09-09: check
            clean (0w/0e after 3 fixes), fmt/clippy pre-existing debt only,
            173/173 pass.**
         4. Extend `~/router-dav/scripts/render-router-config.sh`:
            `[[scheduling.imap]]` accounts rendered from
            `pass secrets/email/<id>/imap` (hosts per provider recon:
            gmail → `imap.gmail.com:993` user=identity;
            carltonaudio → `netsol-imap-oxcs.hostingplatform.com:993`;
            novo-ordo (nfcalaway + zero, shared login) →
            `imap.novo-ordo.com:993` user `nfcalaway@novo-ordo.com`;
            all three hostnames DNS-verified 2026-09-08). — **DONE 2026-09-09.**
         5. Cross-build + deploy via the existing deploy.sh flow (config
            lands before start; pre-deploy DB backup) + live test
            (re-invite an external address, reply by email, watch the
            organizer's event PARTSTAT + scheduling inbox). — **DONE
            2026-09-09, live test PASSED end-to-end:**
            - Cross-build via `scripts/build-rust.sh` (clang recipe D):
              rustical 28 MiB stripped (within the 35 MiB budget),
              dav-tls 1.3 MiB.
            - Deploy via `deploy.sh`: config rendered with **7 SMTP + 7
              IMAP accounts** from pass, migration auto-applied on
              startup, both services running, `rustical health` OK,
              "IMAP iMIP ingestion enabled (7 mailboxes, every 120s)".
            - **Outbound external leg:** PUT (Apple-CalDAV UA) of an
              event with attendee `external-test@example.com` → 201 +
              log `scheduling: emailed REQUEST to
              external-test@example.com (attempt 1)` — Gmail SMTP
              delivery confirmed.
            - **Outbound internal leg:** PUT of an event with local
              attendee chris@carltonaudio.com → `req-live-test-2.ics`
              appears in chris's scheduling inbox (GET-verified).
            - **Inbound leg (the new code):** sent a real REPLY email
              (multipart, `text/calendar; method=REPLY`, From
              nicholas@carltonaudio.com) via msmtp to
              burningserenity@gmail.com → within one 120 s poll the log
              shows `ingest_imip_reply{mailbox_identity=
              "burningserenity@gmail.com" from=Some("nicholas@carltonaudio.com")}:
              iMIP ingest: attendee reply applied uid="live-imap-1"
              attendee="nicholas@carltonaudio.com" partstat="ACCEPTED"`
              → organizer's stored copy flipped
              `PARTSTAT=NEEDS-ACTION→ACCEPTED` (still the full event)
              AND `reply-live-imap-1-nicholas%40carltonaudio.com.ics`
              filed in the organizer's scheduling inbox. All test
              artifacts cleaned up afterwards.
            - **Test-methodology note:** `curl -d @file` strips CR/LF
              from `.ics` bodies → misleading `403
              valid-calendar-data`; use `--data-binary`.
            - **Known issue → FIXED (2026-09-09):**
              `imap.novo-ordo.com:993` serves an **incomplete chain**
              (Sectigo intermediate missing; `openssl s_client` verify
              code 21 even with a full local CA bundle — provider-side
              misconfiguration), so the 2 novo-ordo mailboxes WARNed
              `invalid peer certificate: UnknownIssuer` every poll and
              were effectively unpolled (any attendee replies to
              events organized by nfcalaway@/zero@novo-ordo.com sat
              unread in the mailbox). `smtp.novo-ordo.com:587`
              presents the full chain (verify ok), so outbound invites
              from those identities kept working. **Fix (same day):**
              optional per-account `ca_file` on
              `[[scheduling.imap]]` — extra TLS trust anchors added on
              top of the webpki roots (`tls.rs`:
              `tls_connector_with_extra_ca`, used by `imap::connect`).
              It pins the missing Sectigo intermediate ("Sectigo RSA
              Domain Validation Secure Server CA", 7F:A4:FF:…:46:76,
              valid to 2030-12-31), extracted from SMTP :587's
              complete chain (same leaf, fingerprint DD:B6:A5:…:2E:CD);
              shipped as `router/etc/rustical/certs/imap-novo-ordo.pem`
              (provenance + renewal notes in its header), deployed to
              `/etc/rustical/certs/` by deploy.sh; render script emits
              the key only for the 2 novo-ordo accounts and aborts if
              the repo copy of the PEM is missing. Chosen over a leaf
              pin (mbsync's `~/.local/share/novo-ordo.pem` approach)
              because the intermediate survives the provider's annual
              leaf renewals; webpki roots stay trusted, so a
              provider-side fix is picked up transparently.
              Live-verified against the real server: without the pin
              the handshake fails `UnknownIssuer`, with it TLS +
              greeting succeed (regression test
              `connect_novo_ordo_with_pinned_intermediate`, run via
              `cargo test -p rustical_scheduling -- --ignored`);
              scheduling suite 45/45, caldav scheduling 12/12.
            - **Deployed + live-verified 2026-09-09 17:23** (deploy.sh,
              rustical 28 MiB): cert at
              `/etc/rustical/certs/imap-novo-ordo.pem`, 2 `ca_file`
              entries live; 0 `UnknownIssuer` across 3+ poll cycles
              (old binary WARNed every cycle up to 17:22:37). No
              backlog existed in the novo-ordo mailboxes.
            - **Follow-up: attendee replies missing on events organized
              by nicholas@carltonaudio.com** (reported after the novo-
              ordo fix; that mailbox polls fine — different root cause):
              IMAP inspection showed two iMIP REPLYs in the INBOX
              (lynscarlton@gmail.com and cfcarlton@gmail.com, both
              `Accepted: …`) already flagged `\Seen` — they predated
              the ingest going live / were read by the normal mail
              client before any poll, and ingestion only searches
              UNSEEN. Re-flagged `-FLAGS (\Seen)` (UIDs 960/969) →
              next poll ingested both (17:33:08, `attendee reply
              applied`, partstat=ACCEPTED, organizer copies updated +
              REPLYs filed). **Root systemic gap:** any iMIP reply
              read before a 120 s poll (mbsync flag sync / mail
              client) is missed permanently — UNSEEN-only ingest races
              the mail client on every mailbox that is also read
              normally.
            - **DONE & DEPLOYED (2026-09-09): widen the ingest search** —
              `UID SEARCH OR UNSEEN SINCE <date>` with a 3-day recheck
              window, so read-but-recent replies are still caught. The
              systemic gap above (any iMIP reply read before a 120 s poll
              is missed permanently) is closed.
              - `imap.rs`: `RECHECK_WINDOW_DAYS = 3`; pure helper
                `recheck_since_date(today: NaiveDate) -> String` (chrono
                `%d-%b-%Y` — RFC 3501 `dd-Mon-yyyy`, zero-padded day;
                `SINCE` = INTERNALDATE); `uid_search_unseen()` now sends
                `UID SEARCH OR UNSEEN SINCE <date>` (method name kept, so
                no call-site changes).
              - `ingest.rs`: no logic change (as designed) — docs/log
                accuracy only: module doc records the candidate scope,
                `unseen` renamed `candidates`, debug log now reads
                "N candidates, …". `poll_mailbox_inner` needed no change:
                the checked-UID set skips already-examined mail,
                `MAX_PER_POLL = 25` drains larger candidate lists across
                polls, restarts re-examine only window-recent mail.
                `mark_seen` stays only-after-successful-ingest
                (re-marking a client-read reply `\Seen` is a no-op).
              - Tests: `recheck_window_dates` (fixed dates: month/year/
                leap rollovers, zero-padding, all 12 RFC 3501 month
                abbreviations) + `search_command_covers_recheck_window`
                (fake session captures the exact command line, date
                round-trips through chrono, age 2–4 days tolerating a
                UTC-midnight flip). Suite: **47 pass / 0 fail** (45 + 2
                new; 1 ignored = live novo-ordo network test). No new
                clippy warnings (crate's pre-existing debt untouched),
                rustfmt clean, `SQLX_OFFLINE=true cargo check --workspace
                --all-targets` 0e/0w.
              - Deploy: `build-rust.sh` (clang recipe D) — rustical 28 MiB
                (within the 35 MiB budget), dav-tls 1.3 MiB; `deploy.sh`
                2026-09-09 17:47 — config rendered (7 SMTP + 7 IMAP),
                migration clean, rustical healthy, "IMAP iMIP ingestion
                enabled (7 mailboxes, every 120s)".
              - **Live retest PASSED (deterministic, race-free method —
                per user direction the first-draft design, SMTP-deliver
                then sprint to flag `\Seen` before the 120 s poll, was
                rejected as a data race):** organizer
                nicholas@carltonaudio.com PUTs event uid `live-race-1`
                (attendee burningserenity@gmail.com, Apple-CalDAV UA);
                the REPLY email (`METHOD:REPLY`, `PARTSTAT:ACCEPTED`,
                From burningserenity@gmail.com, multipart with a
                `text/calendar; method=REPLY` part) is APPENDed directly
                into the organizer's netsol INBOX **already flagged
                `\Seen`** with INTERNALDATE=now — the exact end-state of
                "the mail client read it before a poll", with no timing
                race; the next poll 91 s later ingested it:
                `ingest_imip_reply{mailbox_identity=
                "nicholas@carltonaudio.com" from=Some("burningserenity@
                gmail.com")}: iMIP ingest: attendee reply applied
                uid="live-race-1" attendee="burningserenity@gmail.com"
                partstat="ACCEPTED"`; the organizer's stored copy flipped
                NEEDS-ACTION→ACCEPTED and
                `reply-live-race-1-burningserenity%40gmail.com.ics` was
                filed in the organizer's scheduling inbox (both GET-
                verified). The old UNSEEN-only search could never have
                matched a message that was `\Seen` from the moment it
                entered the INBOX. All test artifacts cleaned up (event
                DELETEd 200, inbox reply DELETEd 200, APPENDed email
                expunged by UID); note the expunge also flushed 7 older
                messages that were already `\Deleted` in that mailbox
                (standard expunge semantics). Test-methodology note: the
                PUT template carried `METHOD:REQUEST`, and stored objects
                with METHOD are deliberately not scheduling triggers
                (`handle_put` returns early) — no REQUEST was delivered,
                keeping the test footprint minimal.
              - Observation left for later: `reply-live-imap-1-nicholas@
                carltonaudio.com.ics` from the *previous* session's live
                test still sits in burningserenity@gmail.com's scheduling
                inbox — not touched by this task.
              Alternatives rejected: unread-only ingest (status quo,
              fragile), keeping scheduling mailboxes out of the
              normal mail clients (workflow constraint, unreliable).
   3. **IN PROGRESS — one-click RSVP links in invitation emails
      (2026-09-09 session, stopped mid-flight per user direction):**
      invited users should not have to download/open an `invite.ics` to
      respond — invitation emails must carry links for accept, decline,
      and maybe. Design decisions (recorded for the next session):
      - **One NEUTRAL link in the email → public response page carrying
        the actual Accept/Maybe/Decline links** (`GET /rsvp/{token}` is
        the page; `GET /rsvp/{token}?r=accept|maybe|decline` records).
        Deliberately NOT three direct action links in the email: mail
        scanners (Outlook SafeLinks & friends) prefetch URLs from
        emails — a prefetched `?r=accept` would silently record a
        response the attendee never gave; prefetching the neutral page
        is harmless. The page links are plain GETs (no JS) so they work
        in every in-app mail browser.
      - **Stateless HMAC tokens, no DB row, no migration**:
        `v1.<b64url(JSON claims)>.<b64url(HMAC-SHA256)>`, claims =
        `{uid, organizer, attendee, exp}` (serde_json), MAC input
        domain-separated with the `"omnical-rsvp-v1."` prefix, base64
        URL_SAFE_NO_PAD, 1-year TTL (every REQUEST email mints fresh
        anyway). The response word rides unsigned in `?r=` — the token
        holder can pick any of the three responses, which is exactly
        the capability the emailed invitation grants regardless.
      - **Secret + base URL from config**: `[scheduling] rsvp_secret`
      (render script to auto-generate `pass secrets/omnical/rsvp-secret`
        on first render — NOT YET IMPLEMENTED) and
        `[scheduling] rsvp_base_url` (falls back to `[subscriptions]
        public_url` in `build_extensions` — already wired). Links are
        minted only while BOTH are set; changing the secret invalidates
        all outstanding links (endpoint then 404s until re-invited).
        Startup log now reports "RSVP links enabled/disabled"; WARN if
        a secret is set but no public URL resolves.
      - **Zero new vendored crates**: `hmac 0.13` + `sha2 0.11` +
        `serde_json` were all already in Cargo.lock (hmac via pbkdf2,
        sha2/serde_json via caldav/frontend) — only `hmac` had to be
        added to the workspace-deps table. NOTE hmac 0.13 moved
        `new_from_slice` into the `KeyInit` trait: `use hmac::KeyInit`.
      - **Endpoint semantics mirror the iMIP ingest path**: the
        response is applied through the exact same `deliver_reply`
        machinery an emailed REPLY uses (organizer's stored copies get
        the new PARTSTAT + REPLY filed in their scheduling inbox; no
        extra email). Guards, in order: HMAC+expiry verify → organizer
        must be a local principal (mirror of the ingest mail-loop
        guard, else Invalid/404) → stored-copy lookup by UID across the
        organizer's memberships → STATUS:CANCELLED or no copy or
        attendee removed → Gone/410; unknown response word → 400;
        store failure → 500. Invalid vs Gone are deliberately
        distinguishable (a valid link to a dead event deserves a
        "cancelled" page, not "invalid link"), but Invalid is
        indistinguishable from a bogus path to scanners. Links mint
        only for external attendees on REQUEST (not CANCEL) emails —
        internal attendees RSVP through their CalDAV clients natively.
      **Implementation state (all on the uncommitted `omnical-scheduling`
      working tree in `~/router-dav/rustical`):**
      - **DONE, GREEN — scheduling crate: 55 pass / 0 fail**
        (`SQLX_OFFLINE=true cargo test -p rustical_scheduling`, 47
        prior + 8 new; `cargo check -p rustical_scheduling` clean):
        - `Cargo.toml`s (workspace + crate): + hmac/sha2/serde_json.
        - `config.rs`: `rsvp_secret`/`rsvp_base_url` Option fields
          (serde defaults — old configs keep parsing) +
          `rsvp_links_enabled()`; test `rsvp_link_config` (either alone
          is not enough).
        - NEW `rsvp.rs`: `RsvpClaims`, `mint_token`/`verify_token`
          (constant-time compare via `mac.verify_slice`), TTL const,
          `partstat_for_response` (accept/maybe/decline →
          ACCEPTED/TENTATIVE/DECLINED); 6 unit tests incl. tamper +
          expiry-boundary + wrong-secret + garbage.
        - `mime.rs`: `invite_body(account, event, organizer,
          rsvp_url: Option<&str>)` — with a URL the email leads with
          "Respond directly in your browser" (URL on its own line) and
          offers the attachment as the alternative; None renders the
          classic body byte-for-byte. Test updated + new
          `invite_body_with_rsvp_link`.
        - `scheduler.rs`: `RsvpEvent` (summary/when/recurring/
          organizer/attendee/partstat page data) + `RsvpError`
          (Invalid/Gone/BadResponse/Store) types;
          `rsvp_link_url()` mints per-attendee in `deliver_status`'s
          external branch (REQUEST only); public `rsvp_links_enabled()`,
          `rsvp_page_data()`, `rsvp_apply()` (info log "rsvp link:
          attendee reply applied", mirroring the ingest line); private
          `verify_rsvp_token()` + `resolve_rsvp_event()` (the guard
          chain above) + free fn `rsvp_page_event()`.
        - `lib.rs`: `pub mod rsvp` + re-exports (`RsvpClaims`,
          `RsvpError`, `RsvpEvent`).
      - **ROOT CRATE COMPILES CLEAN (2026-09-09) — was written but not
         compiling (`SQLX_OFFLINE=true cargo check -p rustical` failed with
         4 errors, all trivial, root causes identified and all now FIXED):**
        1. `src/rsvp.rs` (new, complete): public router
           `rsvp_router(Arc<Scheduler>)` → `GET /rsvp/{token}`; `?r=`
            via axum `Query<RsvpQuery>`; landing page (event details +
            three `?r=` links, current-response highlight + note),
            confirmation page ("Response recorded" + change-mind link),
            Invalid/404, Gone/410, BadResponse/400, Store/500 pages —
            all HTML, `noindex,nofollow`, `cache-control: no-store`,
            user content HTML-escaped (event summary is user data).
            **Fix 1 (×2 E0277) APPLIED: `Debug` derived on `RsvpQuery`**
            (axum's Query extractor requires it).
            **Fix 2 (E0599) APPLIED: `IntoResponse` added to the
            `axum::response` import** — the `(StatusCode, [(&str,&str);1],
            Html<String>)` tuple in `page()` needs the trait in scope.
        2. `src/lib.rs`: `pub mod rsvp;` + `build_extensions` rewritten:
           base-URL fallback + WARN + startup log now reads
           "Scheduling extension enabled (N SMTP identities, RSVP links
           enabled|disabled (no rsvp_secret / public URL))". Done, will
           compile once rsvp.rs does.
        3. `src/app.rs`: `rsvp_router` import + a mount block after the
           export-router merge. **Fix 3 (E0382) APPLIED: the second
           `caldav_router` merge now takes `scheduler.clone()` (an Arc
           refcount bump), so the RSVP mount block below it can still
           borrow `scheduler`; its comment updated accordingly.**
       - **Remaining work (next session picks up here):**
         1. ~~Apply the 3 fixes above; root crate must `cargo check`
            clean.~~ **DONE (2026-09-09):** all three fixes applied as
            described above; `SQLX_OFFLINE=true cargo check -p rustical`
            finishes **0 errors / 0 warnings**; `src/rsvp.rs` +
            `src/app.rs` rustfmt-clean (the `page()` helper was
            reformatted; the wider workspace fmt debt is pre-existing,
            left for item 5's gate).
         2. Caldav integration tests (`crates/caldav/src/scheduling/
            tests.rs`, extend `scheduling_app_with_scheduler`'s config
            with `rsvp_secret`/`rsvp_base_url`): PUT organizer event →
            mint token → `rsvp_page_data` (page data + current PARTSTAT)
            → `rsvp_apply(token, "accept")` → organizer stored copy
            flips to `PARTSTAT=ACCEPTED` + `reply-<uid>-<attendee>.ics`
            in organizer inbox (mirror the ingest test); negatives:
            garbage/expired token → Invalid, unknown UID → Gone,
            STATUS:CANCELLED → Gone, attendee removed from copy → Gone,
            `rsvp_apply(token, "bogus")` → BadResponse, foreign
            organizer (not a local principal) → Invalid. — **DONE
            2026-09-09: 19/19 pass** (`SQLX_OFFLINE=true cargo test -p
            rustical_caldav scheduling`; full caldav suite 46/46,
            scheduling 55/55, `cargo check --workspace --all-targets`
            0e/0w; tests.rs rustfmt-clean, no new clippy warnings — the
            3 clippy nits in the file predate the task). Fixture
            extended with `rsvp_secret`/`rsvp_base_url` (link minting
            itself is only reachable on the external-email branch, so
            the 12 pre-existing tests are unaffected). 7 new tests:
            the full flow — page data shows summary/organizer/attendee
            + `NEEDS-ACTION`, `accept` flips the stored copy to
            `PARTSTAT=ACCEPTED` (still the full event) + files the
            inbox REPLY (iTIP-verified), plus the change-mind apply
            (`maybe` → TENTATIVE, inbox REPLY overwritten); negatives —
            garbage/expired/forged-token → Invalid (indistinguishable,
            no validity oracle), unknown UID → Gone, STATUS:CANCELLED →
            Gone, attendee removed from the stored copy → Gone, bogus
            response word → BadResponse (pinned to precede token
            verification — even a garbage token answers 400, matching
            the route's 400-beats-404 semantics), foreign organizer →
            Invalid; every negative also asserts no REPLY was filed and
            nothing was applied.
        3. Root HTTP tests for the /rsvp route (TestRig pattern from
           `register.rs` tests): landing page 200 + three links present,
           `?r=accept` → confirmation + stored PARTSTAT, bad token →
           404, cancelled → 410, `?r=bogus` → 400, HTML escaping of
           summary.
        4. `~/router-dav/scripts/render-router-config.sh`: emit
           `rsvp_secret` into the `[scheduling]` section, reading
           `pass secrets/omnical/rsvp-secret` and AUTO-GENERATING the
           entry on first render (`pass insert` with 32 random bytes
           hex) so the deploy flow stays one command; no rsvp_base_url
           needed (falls back to `[subscriptions] public_url` =
           `https://0115d8cf.duckdns.org:8443`).
        5. Full gate: `SQLX_OFFLINE=true cargo check --workspace
           --all-targets` 0e/0w, `cargo fmt`, clippy, all suites (dav,
           scheduling, caldav, store_sqlite, root lib+bin+http-int
           +integration — register.rs TestRig precedent).
        6. Cross-build (`scripts/build-rust.sh`) + `deploy.sh` + live
           test: invite an external address from a configured identity,
           confirm the email carries the response link, click through
           accept → organizer PARTSTAT flips + inbox REPLY (same
           verification shape as the 2026-09-09 IMAP live test).
        Alternatives rejected: three direct action links in the email
        (SafeLinks prefetch records responses nobody gave; Google-style
        per-action signed links only work with an interactive login
        gate we don't have), DB-stored RSVP tokens (migration + trait
        + GC for no benefit over stateless HMAC), signing the response
        into the token (the emailed invitation grants all three
        responses anyway; the secret gates WHO can respond, not WHAT).
