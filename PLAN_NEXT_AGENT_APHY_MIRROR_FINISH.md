# PLAN — aphy mirror finish: series ops in the omnical calendar, §18.41 record, commits

**Date:** 2026-10-13 (early hours)
**Context:** the follow-through on `PLAN_NEXT_AGENT_APHY_IMPORT_RENDER.md` (§18.41 of
`PLAN_DEPLOYMENTS.md`). The §18.41 push mirror is **built, deployed, and verified** — the
phone pipeline works, the web calendar renders — with exactly **one functional gap open**
(series operations in the aphy web calendar) plus the documentation/commit housekeeping.
This file is the handoff: read §1 (state) first, do §2 (the gap), then §3 (the record and
the commits).

**Repos:**
- `~/router-dav` — Omnical-Code (CI, packaging, deploy; the `rustical` submodule pointer)
- `~/router-dav/rustical` — Omnical-Server, branch `omnical-scheduling`
- `~/Documents/ByteWheel/omnical` — this planning repo
- `~/Documents/ByteWheel/dailyview-mod` — the DailyView app source (diagnosis oracle; do not modify)

---

## 1. Where things stand (all verified, 2026-10-13 ~02:00)

### 1.1 The two-calendar layout — live and converged

| aphy calendar | contents | consumed by | state |
|---|---|---|---|
| `Default` (the stock MP02's account-URL calendar) | **phone feed only**: 38 flattened personal instances + 3 suffixed one-shots + 602 suffixed liturgical one-shots (643 objects), no masters, no ORGANIZER | the MP02's DailyView | ✓ user-verified: Trash Mon 19:00 and Recycling Thu 19:00 on their own days |
| `omnical` (created via MKCALENDAR this session) | **canonical**: 6 personal masters+one-shots + 602 liturgical + 1 CAS (609 objects), ORGANIZER/CREATED-stamped, RRULE intact | the aphy web UI | ✓ week view renders (user confirmed; 10-min events are thin slivers); ✗ **series ops missing — the open item** |

- Router: `/usr/sbin/rustical` sha `4a78aa61` == `out/rustical`; healthy; current.
- Cadence: cron on the router `*/30 * * * * /usr/sbin/rustical --config-file /etc/rustical/config.toml mirror sync` (the serve-path background loop is skipped because the router runs **tenancy**; the loop guard is `!tenancy_enabled`).
- Convergence verified: a full `mirror sync` now reports `0 put, 0 deleted, N unchanged` for all seven targets (a run takes seconds; the first run after any UID-scheme change churns deletes+puts for minutes — expected, the state diff handles it).

### 1.2 What was discovered about aphy (Cyrus + Roundcube) — the §18.41 diagnosis

Nine live bisect rounds against a scratch aphy calendar established:

1. **The week/day grid gate:** a `VEVENT` renders in aphy's week/day views only if it
   carries an `ORGANIZER` (or a `CREATED`). Month view renders everything. khal never
   writes ORGANIZER — that single absence was the entire original §18.39/§18.40 symptom.
2. **Account-wide UID uniqueness (RFC 6638 `unique-scheduling-object-resource`):** Cyrus
   rejects a `PUT` whose VEVENT UID already exists in *another calendar of the same
   account* — `403` with a `C:unique-scheduling-object-resource` error body naming the
   conflicting href. This is why the same event cannot live in both `Default` and
   `omnical` under one UID, and it is the load-bearing constraint behind the whole
   suffixed-UID feed design. It was found live: every omnical one-shot PUT 403'd while
   the plain-UID copies sat in Default.
3. **Kolab-style UID parsing:** `{uid}-{YYYYMMDDTHHMMSSZ}`-suffixed UIDs parse as series
   overrides of that master. With the master in `omnical` and the `-stamp` family in
   `Default` (cross-calendar), the web's series model for the master appears to break —
   the current best explanation for the missing series ops (the open item). Confirmed
   earlier in the session: master + family in the *same* calendar → series ops present;
   split across calendars → series ops absent.
4. **Thin bars:** 10-minute events render in week/day as near-invisible slivers. At least
   two "doesn't render" answers during the bisect were really "renders, too short to
   see." Do not re-litigate early rounds on this account.
5. Restated from §18.36/§18.37 (still true): all-day `VALUE=DATE` files one day early west
   of UTC (feed copies must carry the local-midnight pass — the mirror applies
   `dailyview::apply_all_day_pass` to non-recurring feed copies); the MP02 never expands
   an `RRULE` and files every body at its master's `DTSTART`.

### 1.3 Credentials and source shapes (do not re-derive)

- aphy DAV: **username `burningserenity@aphy.io`** (email form — the bare name
  form 401/403s), password = first line of `pass show secrets/aphy.io/burningserenity/dav`
  (stored this session). The parent entry `secrets/aphy.io/burningserenity` holds a
  **different, web-login password** — the user believes they are the same; they are not.
  Endpoint root `https://dav.aphy.app/dav/`, home
  `/dav/calendars/user/burningserenity@aphy.io/`, Default at `.../Default/`.
- **Do not** probe aphy with repeated failed auth attempts — Cyrus/nginx throttles the
  source for a long window and it cost an hour this session. One careful try with the
  pair above; if it 401s, stop and check pass first.
- Source calendars (tenant DB
  `/usr/local/share/rustical/tenants/ff77b536-0e6f-4e43-b3a4-e7a10641ec75/db.sqlite3`):
  - `nfcalaway@novo-ordo.com` / `personal` — 6 objects (3 weekly masters: Trash/Church/
    Recycling UIDs `RGKW68…`, `CKCPMZ6VQ…`, `TOFZMTQJ…`; 3 past one-shots)
  - `nfcalaway@novo-ordo.com` / `cec120ff-36a1-4787-bca1-ff4dbb6f4be4` — 602 all-day
    one-shot liturgical events (`hol-rc-1962-*@omnical` UIDs)
  - `nicholas@carltonaudio.com` / `Personal` + `/CAS`, `nicholas@hawksnestsoftware.com` /
    `Internal Shared` — mirrored to omnical only; most of their objects are **UID-blocked
    by their own aphy-native copies in Default** (accepted invites: "Purple" `203057C8…`,
    `AJ9UKL4Y…`, `4A6SI53B…`, `vdirsyncer-update-test-20260911`, `25Z10RT2…`) → those
    PUTs 403 every sync (now visible as WARN lines), which is **benign and correct**: the
    native copies own the UIDs and render on the web anyway. Do not "fix" this.
- Scratch artifacts are cleaned up: the `omnical-test` calendar (all 20 test objects +
  the calendar), the `mirror-debug` calendar, and the probe objects are all deleted.

### 1.4 Commit state

- **rustical** (`omnical-scheduling`, clean tree at HEAD): the §18.41 mirror shipped
  across commits `257249d7` (feature: config/store/engine/CLI/loop) → `c1f4c7c1`+
  `a2bdd88b` (CREATED/LAST-MODIFIED inherit the source's own DTSTAMP — bodies must be
  byte-stable across syncs or every run re-PUTs everything) → `d6fa8e9f`
  (`phone_feed_only` targets) → tracing init for manual syncs → HEAD = "mirror: log the
  target by name — its Debug carried the remote password" (the WARN span dumped the
  remote password; fixed with `fields(target = %target.name)`). **Not pushed — the user
  pushes.**
- **Omnical-Code** (`~/router-dav`): commit `7cf6541` (the [mirror] render block) — **plus
  one UNCOMMITTED change**: the principal `.com` fixes in
  `scripts/render-router-config.sh` (`nicholas@carltonaudio.com`,
  `nicholas@hawksnestsoftware.com` — without `.com` the sync aborts `Error: Not found`).
  The deployed `/etc/rustical/config.toml` already carries the fix. **Commit it.**
- **This repo:** `PLAN_NEXT_AGENT_APHY_IMPORT_RENDER.md` and §18.40 are committed;
  **§18.41 is not yet written** (§3 below).
- test.yml pins: integration baseline **121**, digest **`c8bb7123…`** — unchanged and
  still matching (verified this session). No churn expected from the remaining work
  unless integration tests are added.

### 1.5 Gates (unchanged from §18.40)

`cargo fmt --all`; `SQLX_OFFLINE=true cargo clippy --workspace --all-targets -- -D warnings`
(zero); `SQLX_OFFLINE=true cargo test -p rustical --lib` (120); `SQLX_OFFLINE=true cargo
test -p rustical_caldav` (73); `SQLX_OFFLINE=true cargo test -p rustical_store_sqlite`
(57); `SQLX_OFFLINE=true cargo test --test run_integration_tests` (121). Deploy:
`scripts/build-rust.sh` then `./deploy.sh`, verify `ssh router sha256sum /usr/sbin/rustical`
== local `out/rustical`.

---

## 2. The open work: series operations in the `omnical` calendar

**Symptom (user, 2026-10-13):** clicking the recurring Trash in the `omnical` web calendar
"seems to not be there" for series ops (no edit-this-occurrence / this-and-future /
all-events options). Earlier in the session, when a master and its `-stamp` family lived in
the *same* calendar (Default), the user confirmed series ops present.

**Hypothesis (best-supported, §1.2 item 3):** the feed's `{uid}-{stamp}` UIDs in Default
parse as Kolab recurrence overrides of the master in omnical; the cross-calendar
master/override split breaks Roundcube's series resolution, so the master presents as a
plain single event.

**The fix — designed but NOT applied (a late-night python splice missed its match and
wrote nothing; the tree is clean):** re-key every phone-feed UID with a trailing `-m`
marker segment, which breaks the trailing-datetime Kolab parse while preserving
determinism and the `{uid}-` family prefix (the adoption sweep still claims stale
old-scheme objects, and the state diff auto-deletes them on the first post-change run).

Apply with the **Edit tool against the actual file bytes** (the failed heredoc approach is
the lesson — fmt had reflowed the target text). In `~/router-dav/rustical/src/mirror.rs`:

1. Add a helper near `override_uid`:

   ```rust
   /// The feed form of a UID: a trailing `-m` marker segment. aphy's
   /// Roundcube parses `{uid}-{datestamp}` UIDs as Kolab recurrence
   /// overrides of that master, and with the master in the web calendar
   /// and the family in the phone calendar the series model breaks
   /// (§18.41's live finding: series ops vanish). The marker breaks the
   /// datetime-suffix parse while staying deterministic (re-syncs
   /// upsert) and inside the `{uid}-` family (the adoption sweep still
   /// claims stale old-scheme objects).
   fn feed_uid(uid: &str) -> String {
       format!("{uid}-m")
   }
   ```

2. In the `phone_instances` loop of `sync_target`, re-key the instance: the desired key,
   the href, and the body's `UID:` line all become `feed_uid(&instance_uid)` (use
   `override_uid` on the instance body for the line rewrite; `block_uid` reads the block's
   UID).
3. In the non-recurring feed path, the suffixed one-shot UID
   (`format!("{uid}-{stamp}")`) becomes `feed_uid(&format!("{uid}-{stamp}"))` for key,
   href and body alike.
4. Update the `object_dtstart_stamp_and_override_uid_key_the_feed_copy` unit test (it pins
   the current `{uid}-{stamp}` scheme) and add an assertion that the feed form carries
   the marker. Add a unit test for `feed_uid` if trivial.
5. Gates (§1.5), commit, `scripts/build-rust.sh`, `./deploy.sh`, then on the router run
   `rustical mirror sync` — expect: personal-to-default ~41 put + 41 deleted (old-scheme
   out, `-m` scheme in), liturgical-to-default 602 put + 602 deleted, omnical targets
   unchanged; the *next* run converges to 0/0/N. Nothing to clean by hand.

**Verification — ask the user to click TRASH in the `omnical` calendar specifically**
(not Church: its `ORGANIZER` is `nfcalaway@novo-ordo.com`, a foreign identity — Roundcube
may legitimately treat foreign-organized events as read-only; that would be a *different*
cause and probably correct behavior). Then one more phone manual sync to confirm the feed
renaming did not disturb the 4-day list (the phone wipes and refetches each sync, so
renamed UIDs are a non-event for it).

**If the marker does not restore series ops:** next hypotheses, in order —
(a) the web UI's series options require the event's calendar to own the whole family, in
which case ask the user whether series ops matter more than the phone feed's current
clean shape (the pre-verification state — master + instances in one calendar — had series
ops; that layout also had the month-duplicates the user rejected), and
(b) inspect what Roundcube's event editor keys series ops on via the user's screen
(screenshot of the event card) before any further code.

---

## 3. The record and the commits (after §2 resolves, either way)

1. **§18.41 in `PLAN_DEPLOYMENTS.md`** (append-only, the repo's habit: one commit per
   section). It should record, compactly:
   - the §3.1 channel test's outcome (PUT ≡ import — both landed month-only pre-fix),
     the §3.2 byte bisect's outcome (native bytes render, ours don't → the field hunt),
     and the **four mechanisms of §1.2** as the actual diagnosis (ORGANIZER/CREATED gate;
     account-wide RFC 6638 UID uniqueness with the live 403 evidence; Kolab
     `-datestamp` override parsing and its cross-calendar series consequence; thin bars);
   - the mirror design as shipped: `[mirror]` config (`[[mirror.targets]]`,
     `phone_feed_only`, `phone_instances`, `horizon_days`, `interval_secs`,
     organizer stamping), the `mirror_objects` state table (migration
     `20261012200000_mirror_objects`, count guard now 20), hash-based change detection
     with the body-stability rule (stamps inherit the source's DTSTAMP), the adoption
     sweep, the tracing fixes (CLI warnings; the password-in-logs fix);
   - the two-calendar layout and the user's decisions (Default = phone feed;
     everything else = `omnical`; suffixed feed UIDs);
   - live results: the §1.1 matrix, converged sync numbers, the benign 403 set
     (§1.3), the known limits (foreign-organized events may be read-only on the web;
     liturgical events leave the phone — accepted, they were one-day-early there
     historically via §18.36 and the feed copies now carry the pass);
   - the cron cadence and why (tenancy skips the serve-path loop);
   - the export route (`?dailyview=1`, the Download button) stays as-is: still useful for
     one-off imports, byte shapes phone-proven.
2. **Commit Omnical-Code** (the uncommitted principal `.com` fix; message per house style).
3. **Push** — the user pushes both repos (per §4 of the session plan).
4. **Offer the aphy support package** (the original §3.3): the ORGANIZER/CREATED week/day
   gate, the account-wide UID-uniqueness UX (a same-UID copy attempt gives a bare 403),
   and the thin-bar rendering are product behaviors worth reporting as their customer.
   The repro material is all in this file's §1.2; draft only if the user wants it.
5. Ask the user the one unanswered layout question: keep the `Default` calendar displayed
   in the aphy web UI (it holds the phone feed + their aphy-native events) or hide it.

## 4. Small follow-ups (optional, none blocking)

- The perpetual 403 WARNs for the native-duplicate objects (§1.3) could be quieted by
  matching the `unique-scheduling-object-resource` precondition and recording a
  "skipped: UID owned by a native copy" state instead of retrying — cosmetic; the retries
  are harmless (2-3 PUTs per sync).
- Check that the router's crontab line survives `sysupgrade` (the watchdog's cron is
  persisted via deploy.sh's `sysupgrade.conf` handling; the mirror cron was added by hand
  this session and may not be).
- The `[mirror] enabled = false` on the router is intentional (tenancy); a future
  per-tenant loop is §7 work, not §18.41's.

## 5. Do not redo

- The §18.36/§18.37 UA wire shims (inert by design — the phone never syncs omnical).
- The §18.39/§18.40 export shapes (`flattened_instance`, the Download button).
- The 90-day horizon and 30-minute cadence (revisit only if the feed's purpose changes).
- Do not recreate scratch calendars on the aphy account without deleting them after
  (§1.2's UID rules make leftovers actively harmful — they shadow real calendars).
