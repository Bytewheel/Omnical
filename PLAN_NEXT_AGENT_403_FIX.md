# Continuation plan — Omnical 403 fix (RRULE UNTIL normalisation + default ORGANIZER)

**Date:** 2026-09-18
**Repo:** `~/router-dav/rustical`, branch `omnical-scheduling`
**Original task brief:** the khal-created event
`~/.calendars/omnical/Internal Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics`
403s on vdirsyncer PUT (pair `omnical_calendar_hawksnest`, cron `*/15`). Root cause
(established, do not re-litigate): caldata 0.16.2 rejects khal's floating
`RRULE;UNTIL=20261204T090000` next to `TZID`'d DTSTART
(`DtStartUntilMismatchTimezone`) → `valid-calendar-data` precondition → 403.
Two server-side fixes were specified: (1) normalise floating UNTIL to UTC on
import; (2) default the event creator as ORGANIZER on organizer-less
attendee-carrying events.

---

## 1. Status: BOTH FIXES ARE IMPLEMENTED AND ALL CODE GATES PASS

Working tree state (uncommitted — do NOT commit unless the user explicitly asks):

```
 M crates/caldav/src/calendar_object/methods.rs   (+59/-13 net: import closure, stamp_default_organizer, stored_ics)
 M crates/caldav/src/scheduling/tests.rs          (+207: KHAL_EVENT const + 4 tests)
 M crates/ical/src/calendar_object.rs             (+4: import calls normalize_rrule_until)
 M crates/ical/src/lib.rs                         (+3: mod normalize; pub use normalize::*)
 M crates/scheduling/src/ics.rs                   (+126: default_organizer + 5 tests)
?? crates/ical/src/normalize.rs                   (NEW, 539 lines incl. 15 tests)
```

### Fix 1 — `normalize_rrule_until` (done)

- `crates/ical/src/normalize.rs`: public `normalize_rrule_until(ics: &str) -> Cow<'_, str>`.
  Walks unfolded logical lines with component-depth tracking (local walker; no
  scheduling-crate dependency). Per VEVENT/VTODO/VJOURNAL at VCALENDAR depth:
  captures DTSTART shape (`TZID`/UTC-Z/floating/`VALUE=DATE`/other), rewrites
  RRULE `UNTIL=<floating 15-digit DATE-TIME>` only when DTSTART is tz-qualified:
  TZID → interpret via `NaiveDateTime::and_local_timezone(tz).earliest()`
  (chrono-tz, deterministic on DST folds; nonexistent local times left
  untouched rather than guessed), re-emit `…Z`; UTC DTSTART → append `Z`.
  Nested components (VALARM, VTIMEZONE subcomponents) never contribute.
  Untouched bodies return `Cow::Borrowed`; rewritten bodies rejoin CRLF with
  trailing-CRLF guard (same as `scheduling::ics::normalize_caladdresses`).
  Idempotent. Quoted-param-safe line splitting (`:`/`;` outside quotes).
- Wired into `CalendarObject::import` (`crates/ical/src/calendar_object.rs:79`)
  — the DAV PUT path (`put_event`) and the calendar-level import route both
  inherit it. `from_ics` (store load) untouched, per the brief.
- 15 unit tests incl. the real-world event verbatim, summer/EDT date,
  folded RRULE, VTODO, all-day/floating/already-UTC/unknown-TZID untouched
  (all `Cow::Borrowed`), per-component zones, RRULE-before-DTSTART,
  VALARM-depth guard, idempotence, and end-to-end
  `CalendarObject::import(REAL_EVENT)` → Ok (the 403 regression test).

### Fix 2 — default ORGANIZER stamping (done)

- `crates/scheduling/src/ics.rs`: new `default_organizer(ics, organizer) -> String`
  inserts `ORGANIZER:mailto:<organizer>` before the first ATTENDEE (depth 1) of
  the first VEVENT; no-op if that VEVENT already has an ORGANIZER or no
  ATTENDEE (VALARM attendees and second VEVENTs out of scope). Unfold→rejoin
  CRLF, trailing-CRLF guard. 5 unit tests (folded khal ATTENDEE line included).
- `crates/caldav/src/calendar_object/methods.rs` (`put_event`):
  - import error handling now a closure `import(&body)?` (same
    `ValidCalendarData` 403 arm + warn logs as before);
  - new helper `stamp_default_organizer(body, user_id, principal)` gates on
    `parse_event`: no METHOD, no ORGANIZER, ≥1 ATTENDEE, and acting user is an
    attendee or owns the target calendar (path principal) — exactly the
    scheduler's implicit-organizer rule, materialised. Attendee-less events,
    METHOD'd bodies, existing ORGANIZERs, VTODO/VJOURNAL untouched;
  - `scheduler.handle_put` now receives `stored_ics = object.get_ics()` (the
    final normalised+stamped form) instead of the raw body;
  - ETag computed from the final object → matches stored copy → vdirsyncer
    stable after first sync.
- 4 caldav integration tests: (a) the production 403 regression — khal event
  verbatim PUT by owner → 201, GET back contains both
  `UNTIL=20261204T140000Z` and `ORGANIZER:mailto:user`; (b) existing
  ORGANIZER preserved verbatim; (c) attendee-less event gets no ORGANIZER;
  (d) stamping works with scheduler `None` (router built without one).
- Behavioural equivalence: existing `sched-no-org-1` test
  (organizer-less attendee event → 201 + REQUEST in inbox) passes unchanged.

### Gates already run (all green)

- `SQLX_OFFLINE=true cargo check --workspace --all-targets` — 0 errors;
  only 5 pre-existing warnings in `rustical_frontend` (identical on stashed HEAD).
- `cargo fmt` + `cargo fmt --check` — clean.
- Clippy: zero NEW warnings — verified by diffing `cargo clippy --workspace
  --all-targets` warning locations against stashed HEAD (all diffs are
  line-number shifts of pre-existing pedantic debt; house rule: untouched).
- Tests: `rustical_ical` 16, `rustical_scheduling` 61 (+1 ignored),
  `rustical_caldav` 51, `rustical_store_sqlite` 34, `rustical --lib` 36 —
  0 failures. No insta snapshot changes (all passed without regeneration).

---

## 2. Remaining work, in order

1. **Sanity re-run** (fast, confirms nothing drifted):
   ```bash
   cd ~/router-dav/rustical
   SQLX_OFFLINE=true cargo check --workspace --all-targets
   cargo fmt --check
   SQLX_OFFLINE=true cargo test -p rustical_ical -p rustical_scheduling -p rustical_caldav -p rustical_store_sqlite
   SQLX_OFFLINE=true cargo test -p rustical --lib
   ```
   Optionally `SQLX_OFFLINE=true cargo test --workspace` for crates outside
   the gate list (import behaviour change is normalisation-only; low risk).

2. **Cross-build:** `~/router-dav/scripts/build-rust.sh` (clang recipe D).
   Budget: stripped rustical ≤ 35 MiB (recent builds ~28 MiB). Check the
   output binary size and report it.

3. **Deploy — CONFIRM WITH THE USER FIRST** (restarts the production rustical
   + dav-tls services on the router; pre-deploy DB backup is part of the
   script): `~/router-dav/deploy.sh`.

4. **Live verify** (the stuck event clears on the next cron run, no
   client-side edit needed):
   - `ssh router 'logread | grep -a rustical | tail -20'` → the
     `PUT …/Internal%20Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics` line
     shows `status=201 Created`; `rustical health` OK.
   - `vdirsyncer sync omnical_calendar_hawksnest` (local) → 0 errors; vdirsyncer
     GETs the stamped/normalised copy back into the vdir (one local update,
     then stable across two consecutive runs).
   - Bonus: REQUEST to `denis@hawksnestsoftware.com` delivered by the scheduler
     from the organiser copy (internal principal → his inbox, else iMIP) —
     visible as the usual scheduling lines in the router log.
   - If anything still 403s: grab the `WARN … invalid calendar data` body dump
     from the log before changing code.

5. **Doc log:** add a dated status entry to
   `~/Documents/ByteWheel/omnical/PLAN.md` in the house style (diagnosis →
   fixes → test counts → deploy/live-verify transcript): 403 root cause
   (caldata `DtStartUntilMismatchTimezone`), Fix 1 (floating UNTIL → UTC on
   import), Fix 2 (default-organizer rule mirroring implicit scheduling), next
   to the §17.9.2 / implicit-scheduling notes.

6. **Commit:** only if the user explicitly asks. If asked, stage exactly the
   six files listed above (incl. the untracked `normalize.rs`).

---

## 3. Gotchas / implementation notes for the next agent

- **Rust test-literal pitfall:** a `\`-line-continuation in a `"…\r\n"` string
  literal swallows leading whitespace of the next line — it silently ate the
  fold space of khal's folded `ATTENDEE`/`RSVP=TRUE` line in tests twice. The
  fixed literals either put the fold (`\r\n =TRUE…`) on one physical line or
  use `concat!` (see `KHAL_EVENT` in `crates/caldav/src/scheduling/tests.rs`).
  Preserve that when editing tests.
- `SQLX_OFFLINE=true` is required for every cargo command (no live DB).
- Clippy house rule: zero NEW warnings only. Method used here: run
  `cargo clippy --workspace --all-targets 2>&1 | grep -A1 '^warning' |
  grep '\-\->' | sort | uniq` on the working tree vs stashed HEAD and diff.
  `put_event` was refactored into `import` closure + `stamp_default_organizer`
  helper partly to stay under `clippy::too_many_lines` (100).
- The scheduler now gets `object.get_ics()` (regenerated, re-folded by caldata)
  rather than the client body — REQUEST/CANCEL payloads and the canonical-form
  change detection (`scheduling_relevant_change`) see the final stored form.
  ETag/GET consistency verified by the regression test (a).
- Delivery to external attendees no-ops without SMTP config (safe in tests);
  internal principals get inbox copies.
- Do NOT touch: `CalendarObject::from_ics`, store crates, khal/vdirsyncer
  configs, the calendar-level import route's organizer semantics (bulk import
  deliberately gets no stamping).
- Leftovers for the USER to decide (do not act unilaterally): stale sibling
  vdir `~/.calendars/omnical/hawksnest-internal-shared/` (older duplicate
  "Stand Up", UID `L9YROQM0L9T6MZSKN3P22JX9AZPHOMNLLWDQ`, odd `TRIGGER:P0D`
  VALARM, not covered by any current pair) and orphaned
  `~/.vdirsyncer/status/{google_calendar*,rustical_*}` dirs.

## 4. Quick reference — anchors

| What | Where |
|---|---|
| `normalize_rrule_until` + tests | `crates/ical/src/normalize.rs` (new) |
| import wiring | `crates/ical/src/calendar_object.rs:79` |
| `default_organizer` + tests | `crates/scheduling/src/ics.rs` (near `add_method`) |
| `put_event` (import closure, stamping, stored_ics) + `stamp_default_organizer` | `crates/caldav/src/calendar_object/methods.rs:59` |
| Implicit-organizer rule mirrored | `crates/scheduling/src/scheduler.rs:167-181` |
| 403 regression + stamping tests | `crates/caldav/src/scheduling/tests.rs` (`KHAL_EVENT`, 4 tests after `test_put_no_organizer…`) |
| Real-world stuck event | `~/.calendars/omnical/Internal Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics` |
| Standalone caldata repro | `/tmp/opencode/caldata-repro` (`cargo run -- <ics>`; src/main.rs was last overwritten with a generate()-roundtrip demo) |
| Build / deploy | `~/router-dav/scripts/build-rust.sh` / `~/router-dav/deploy.sh` |
| Router logs / service | `ssh router 'logread | grep -a rustical | tail -N'` (libreCMC, procd service `rustical`) |
| Doc log target | `~/Documents/ByteWheel/omnical/PLAN.md` (house style: diagnosis → fixes → test counts → deploy transcript) |
