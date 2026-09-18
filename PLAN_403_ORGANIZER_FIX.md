# Omnical — Fix 403 on khal-created events (RRULE UNTIL) + default ORGANIZER stamping

**Date:** 2026-09-18
**Working tree:** `~/router-dav/rustical` (branch `omnical-scheduling`, uncommitted — check `git status` first)
**Symptom:** vdirsyncer PUT of a khal-created event to the `Internal Shared` calendar of
`nicholas@hawksnestsoftware.com` returns **403 Forbidden** every 15 min (cron), pair
`omnical_calendar_hawksnest`, collection `Internal Shared`, URL
`https://0115d8cf.duckdns.org:8443/caldav/principal/nicholas@hawksnestsoftware.com/Internal%20Shared/…`.
**User direction:** fix it in Omnical (the server), *not* in khal. Also: default the event
creator as ORGANIZER on organizer-less events.

---

## 1. Diagnosis (already established — do not re-litigate)

The 403 is **not** an ACL problem and **not** (by itself) the missing ORGANIZER. It is the
CalDAV precondition `valid-calendar-data`:

- `put_event` → `CalendarObject::import` fails → `Error::PreconditionFailed(Precondition::ValidCalendarData)`
  → **403** (`crates/caldav/src/calendar_object/methods.rs:149-161`, mapped in
  `crates/caldav/src/error.rs:12,34,86`). Router log shows `WARN … invalid calendar data` +
  body dump, then `ERROR … status=403 Forbidden` (view: `ssh router 'logread | grep -a rustical | tail -100'`).
- caldata 0.16.2 (workspace pin) rejects the khal-written recurrence rule:
  `RRULE(ValidationError(DtStartUntilMismatchTimezone { dt_start_tz: "America/New_York", until_tz: "Local", expected: ["UTC"] }))`
  — RFC 5545 §3.3.10: when DTSTART is timezone-qualified, `UNTIL` MUST be UTC. khal writes
  a floating local `UNTIL=20261204T090000` (means 09:00 America/New_York = `20261204T140000Z`).
- Reproduced standalone with caldata 0.16.2 (scratch project at `/tmp/opencode/caldata-repro`,
  `cargo run -- ~/.calendars/omnical/"Internal Shared"/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics`):
  original fails; **adding an ORGANIZER does not fix it**; converting UNTIL to `…T140000Z` fixes it.
- The stuck event (khal hand-edited, has `ATTENDEE:MAILTO:denis@hawksnestsoftware.com`, **no
  ORGANIZER**, `DTSTART;TZID=America/New_York:20260918T110000`,
  `RRULE:FREQ=WEEKLY;UNTIL=20261204T090000;INTERVAL=2`) lives at
  `~/.calendars/omnical/Internal Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics` and is
  retried by cron (`*/15 * * * * vdirsyncer sync`) forever.

Two server-side changes are required (both are Omnical-side normalisation of client input,
matching the existing philosophy — see the `import` docstring and the implicit-organizer
rule in the scheduler):

---

## 2. Fix 1 — normalise floating `RRULE`/`UNTIL` to UTC on import

**Where:** `crates/ical` (crate `rustical_ical`).

1. New module `crates/ical/src/normalize.rs`, wired in `crates/ical/src/lib.rs`
   (`mod normalize; pub use normalize::*;` or a narrow re-export).

   Public function:

   ```rust
   /// Convert RFC-5545-invalid floating UNTIL DATE-TIMEs to UTC when the
   /// component's DTSTART is timezone-qualified (khal writes
   /// `UNTIL=20261204T090000` for TZID'd events; it means the event's zone).
   /// Returns Cow so untouched bodies are zero-copy.
   pub fn normalize_rrule_until(ics: &str) -> std::borrow::Cow<'_, str>
   ```

   Semantics (walk **unfolded logical lines** with component tracking, same style as
   `crates/scheduling/src/ics.rs::unfold` (line 185) + `parse_event`'s depth tracking —
   write a small local walker; the ical crate must NOT depend on the scheduling crate):

   - Track components (`BEGIN:VEVENT`/`VTODO` … `END:…`; skip properties inside nested
     components such as VALARM — depth ≥ 2 must not contribute DTSTART/RRULE).
   - Per component, capture DTSTART (params `TZID`, `VALUE=DATE`; value form) and each
     RRULE logical line.
   - Rewrite an RRULE **only when** the component's DTSTART is tz-aware or UTC **and** the
     RRULE has `UNTIL=<floating DATE-TIME>` (exactly `\d{8}T\d{6}`, no `Z`, split on `;`):
     - `TZID=<zone>` DTSTART → interpret the naive UNTIL in that zone via chrono-tz
       (`NaiveDateTime::and_local_timezone(tz).earliest()` — deterministic on DST folds),
       re-emit as `UNTIL=YYYYMMDDTHHMMSSZ` (note: an RRULE with no `UNTIL`, or `UNTIL`
       already UTC, or DATE-valued `UNTIL` (8 digits) for all-day DTSTARTs → leave untouched).
     - UTC DTSTART (value ends `Z`, no `TZID`) → floating UNTIL is taken as UTC: append `Z`.
     - Unparseable/unknown `TZID` (e.g. `Local`) → leave untouched (parser rejects as today).
     - Floating DTSTART (no TZID, no Z) + floating UNTIL is RFC-correct → leave untouched.
     - `VALUE=DATE` DTSTART → leave untouched (out of scope).
   - **Splicing:** if nothing needs rewriting return `Cow::Borrowed(ics)`. If something
     changes, rebuild the body from the unfolded logical lines joined with CRLF (folding
     is lost — acceptable and precedented: see the doc comment on
     `scheduling::ics::normalize_caladdresses`, line 430, which does exactly this;
     RustiCal re-folds on serialisation, and the store serialises via `get_ics()` anyway).
     Keep the trailing CRLF if the input ended with a newline (same guard as
     `normalize_caladdresses`).
   - Must be **idempotent** (running it twice yields the same bytes).
   - `chrono-tz` is already a dependency of `crates/ical` (Cargo.toml line 16) — no dep changes.

2. Wire into `CalendarObject::import` (`crates/ical/src/calendar_object.rs:79-88`):

   ```rust
   let ics = normalize_rrule_until(ics);
   let parser = IcalObjectParser::from_slice(ics.as_bytes()).with_options(options.unwrap_or_default());
   ```

   - `import`'s doc comment already declares this purpose ("iCalendar data coming from
     outside that might need to be normalised"). It has exactly **one** call site:
     caldav `put_event` (verified by grep) — the DAV PUT path, i.e. exactly where client
     quirks arrive.
   - **Do NOT touch `CalendarObject::from_ics`** (line 93): that path loads trusted,
     already-normalised data from the store.

### Fix 1 tests (new `#[cfg(test)]` module in `normalize.rs`; run `SQLX_OFFLINE=true cargo test -p rustical_ical`)

Use the **real-world event** (paste verbatim from
`~/.calendars/omnical/Internal Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics`, it is
short — khal PRODID, VTIMEZONE America/New_York, VEVENT with the folded ATTENDEE line,
one VALARM). Required cases:

- tz-aware DTSTART + floating UNTIL → `UNTIL=20261204T140000Z` (Dec = EST, UTC-5), rest of
  the line (`FREQ`, `INTERVAL`) preserved, order-independent of param positions; and
  end-to-end `CalendarObject::import(real_event)` now returns `Ok` (this is the 403 regression test).
- a spring/summer UNTIL date (EDT, UTC-4) proves chrono-tz per-date handling.
- UTC `Z` DTSTART + floating UNTIL → `Z` appended.
- folded RRULE line (UNTIL token split across a fold) → still fixed.
- VTODO with the same DTSTART/RRULE shape → fixed.
- all-day DTSTART (`VALUE=DATE`) + DATE UNTIL → byte-identical (borrowed) output.
- floating DTSTART + floating UNTIL → untouched (Cow::Borrowed).
- already-UTC `UNTIL=…Z` → untouched (Cow::Borrowed).
- idempotence: `normalize(normalize(x)) == normalize(x)`.

---

## 3. Fix 2 — default ORGANIZER to the event creator

**Where:** `crates/caldav/src/calendar_object/methods.rs` (`put_event`) + a new text-surgery
helper in `crates/scheduling/src/ics.rs` (caldav already depends on `rustical_scheduling`).

**Rule (user request: "add the event creator as the organizer, by default"):** when a VEVENT
with **at least one ATTENDEE** and **no ORGANIZER** and no `METHOD:` is PUT, and the acting
user either **owns the target calendar** (path principal == user id) or **is an attendee**,
stamp `ORGANIZER:mailto:<user.id>`. This *materialises* exactly the implicit-organizer rule
the scheduler already applies (`crates/scheduling/src/scheduler.rs:167-181`: "No ORGANIZER …
the acting user acts as the organizer if they are an attendee or own the calendar") —
behaviour stays identical, but the stored object, etag, GETs, iTIP REQUESTs and REPLY routing
now carry a real ORGANIZER. Do **not** stamp attendee-less events, METHOD'd (iTIP) bodies,
or events that already have an ORGANIZER; do not touch VTODO/VJOURNAL
(`parse_event` is VEVENT-only — keep that scope).

### Implementation

1. New helper in `crates/scheduling/src/ics.rs`, next to `add_method` (line 397) /
   `set_attendee_partstat` (line 515), same unfold→walk→rejoin pattern:

   ```rust
   /// Insert `ORGANIZER:mailto:<organizer>` before the first ATTENDEE property
   /// (depth 1) of the first VEVENT. Unfolded rejoin, CRLF, trailing-CRLF guard —
   /// mirror add_method/normalize_caladdresses.
   #[must_use]
   pub fn default_organizer(ics: &str, organizer: &str) -> String
   ```

2. In `put_event` (`methods.rs`), after the (now-fixed) `CalendarObject::import` succeeds:

   ```rust
   // Omnical: default the organizer to the event creator. khal-lineage clients
   // write ATTENDEEs without an ORGANIZER; stamp the acting user — the same user
   // implicit scheduling would act on — so the stored object carries one.
   let mut body = body;
   let mut object = /* the imported object */;
   if let Some(info) = rustical_scheduling::ics::parse_event(&body)
       && info.method.is_none()
       && info.organizer.is_none()
       && !info.attendees.is_empty()
   {
       let is_attendee = info.attendees.iter().any(|a| a.email.eq_ignore_ascii_case(&user.id));
       if is_attendee || principal.eq_ignore_ascii_case(&user.id) {
           body = rustical_scheduling::ics::default_organizer(&body, &user.id);
           object = /* re-import `body`; propagate any error through the existing
                        ValidCalendarData 403 arm — stamped bodies parse, so this is
                        practically infallible */;
       }
   }
   ```

   (Note `parse_event` lower-cases emails and handles `MAILTO:`/principal-URL CAL-ADDRESSes
   via `Line::as_email`, line 82; it also drops attendees identical to the organizer —
   read lines 242-330 before touching anything.)

3. Feed scheduling the **final** form: change the `scheduler.handle_put` call (line 169-179)
   to pass `object.get_ics()` instead of `&body`, so `handle_put`/`organizer_put` build
   REQUEST/CANCEL payloads from the stamped, UNTIL-normalised ICS (the stored copy is the
   regenerated `get_ics()` form anyway — the sqlite store serialises the object, not the raw
   body; verify with `put_object` in `crates/store_sqlite`). ETag (line 162) is computed
   from the final object and therefore matches the stored copy — vdirsyncer stays stable
   (first sync stores + GET-backs the normalised copy, then no-ops).

4. `crates/caldav/src/calendar/methods/import.rs` (calendar-level import route): it goes
   through `CalendarObject::import`, so it inherits Fix 1 for free; do NOT add organizer
   stamping there (bulk import, no acting-user semantics needed).

### Fix 2 tests (caldav crate; rigs exist in `crates/caldav/src/scheduling/tests.rs` —
`scheduling_app()`/`request()`/`propfind_inbox()` helpers, PUT pattern at lines 224-247 —
and `crates/caldav/src/calendar/tests.rs`; match their style)

- **403 regression (the production case):** PUT the real-world khal event verbatim as the
  calendar owner into their own calendar → **201** (was 403); GET the object back → contains
  `ORGANIZER:mailto:<user>` **and** `UNTIL=20261204T140000Z`.
- PUT of an event that already has an ORGANIZER → ORGANIZER preserved verbatim (not
  rewritten, not duplicated).
- PUT of an attendee-less khal event → stored copy has **no** ORGANIZER line.
- Stamp happens with scheduler `None` too (put_event must not condition on
  `scheduler.is_some()` — check the plain calendar test rig for this).
- **Behavioural-equivalence:** the existing test at
  `crates/scheduling/src/scheduling/tests.rs:220` (`sched-no-org-1` — organizer-less event
  with attendees, asserts 201 + REQUEST filed in the other user's inbox) must keep passing
  unchanged. Also run the whole scheduling + caldav suites; if any snapshot/insta test
  changes, re-generate with `INSTA_UPDATE=always` and review the diff (their PLAN.md §2215
  convention) — expect *only* additions of the ORGANIZER line / UNTIL `Z` where organizer-less
  attendee events are PUT through the router in tests.

---

## 4. Gates (their standard, from PLAN_SHARING.md)

```bash
cd ~/router-dav/rustical
SQLX_OFFLINE=true cargo check --workspace --all-targets   # 0 errors / 0 warnings
cargo fmt                                                  # only files touched; `cargo fmt --check` clean after
SQLX_OFFLINE=true cargo test -p rustical_ical              # new normalize tests
SQLX_OFFLINE=true cargo test -p rustical_caldav            # incl. scheduling module
SQLX_OFFLINE=true cargo test -p rustical_scheduling
SQLX_OFFLINE=true cargo test -p rustical_store_sqlite
SQLX_OFFLINE=true cargo test -p rustical --lib             # root lib (register/rsvp tests)
clippy: zero NEW warnings (pre-existing pedantic debt untouched — house rule)
```

## 5. Build, deploy, live verify

1. Cross-build: `~/router-dav/scripts/build-rust.sh` (clang recipe D). Budget: stripped
   rustical ≤ 35 MiB (recent builds ~28 MiB).
2. **Deploy restarts the production service — confirm with the user first.** Then
   `~/router-dav/deploy.sh` (pushes binary + config, restarts rustical + dav-tls on the
   router, auto-migrates DB; pre-deploy DB backup is part of the script).
3. Live verify (the stuck event clears on the next cron run — no client-side edit needed):
   - `ssh router 'logread | grep -a rustical | tail -20'` → the
     `PUT …/Internal%20Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics` line shows
     `status=201 Created`; `rustical health` OK.
   - `vdirsyncer sync omnical_calendar_hawksnest` (local) → pair completes 0 errors;
     vdirsyncer then GETs the stamped/normalised copy back into the vdir (one local update,
     then stable across two consecutive runs).
   - Bonus: the REQUEST to `denis@hawksnestsoftware.com` is now delivered by the scheduler
     from the organiser copy (internal principal → his inbox, else iMIP) — visible in the
     router log as the usual scheduling lines.

## 6. Out of scope / cautions

- No khal, vdirsyncer, or client-config changes. No `~/.config/vdirsyncer/config` changes.
- Do not touch `CalendarObject::from_ics` (store load path) or any store crate code.
- Do not stamp ORGANIZER on every event (only events that have attendees — that is where an
  organizer matters for iTIP/scheduling; if the user later wants Google-style
  always-organizer, that is a one-line condition change in `put_event`).
- Leftovers for the user to decide (do not act unilaterally): the stale sibling vdir
  `~/.calendars/omnical/hawksnest-internal-shared/` (older duplicate "Stand Up", UID
  `L9YROQM0L9T6MZSKN3P22JX9AZPHOMNLLWDQ`, odd `TRIGGER:P0D` VALARM, not covered by any
  current pair) and the orphaned `~/.vdirsyncer/status/{google_calendar*,rustical_*}` dirs.
- Doc log: add a dated status entry to `~/Documents/ByteWheel/omnical/PLAN.md` in the house
  style (diagnosis → fixes → test counts → deploy/live-verify transcript), summarising the
  403 root cause (caldata `DtStartUntilMismatchTimezone`), the two fixes, and the new
  default-organizer rule next to the §17.9.2 / implicit-scheduling notes.

## 7. Quick reference — anchors

| What | Where |
|---|---|
| PUT handler (`put_event`), import call, 403 arm, `can_write` gate, `handle_put` call | `crates/caldav/src/calendar_object/methods.rs:59-187` (import+warn: 149-161, etag: 162, store: 163, handle_put: 169-179, can_write: 80) |
| 403 mapping + `valid-calendar-data` precondition | `crates/caldav/src/error.rs:12,34,74,86` |
| `CalendarObject::import` (add normalisation here) / `from_ics` (do not touch) | `crates/ical/src/calendar_object.rs:79 / 93` |
| Implicit organizer rule to mirror | `crates/scheduling/src/scheduler.rs:167-181` (`owns_calendar`: 175) |
| Text-surgery precedents (`unfold`, `add_method`, `normalize_caladdresses`, `set_attendee_partstat`, `rebuild_line`, `Line::as_email`, `parse_event` incl. VALARM-depth guard) | `crates/scheduling/src/ics.rs:82,185,242,397,430,515` |
| caldata pin / error shape | workspace Cargo.toml `caldata = 0.16` (lock: 0.16.2), `Error::RRule(ValidationError(DtStartUntilMismatchTimezone { .. }))` |
| Real-world stuck event | `~/.calendars/omnical/Internal Shared/25Z10RT2PWIEJR2ZRHH3MTF7FJYQB8NS428S.ics` |
| Standalone caldata repro (optional sanity aid) | `/tmp/opencode/caldata-repro` (`cargo run -- <ics>`) |
| Router logs / service | `ssh router 'logread | grep -a rustical \| tail -N'` (libreCMC, procd service `rustical`) |
