# Omnical — Roundcube-style web calendar UI (server-rendered grid)

**Date:** 2026-10-11
**Working tree:** `~/router-dav/rustical` (branch `omnical-scheduling` — check `git status` first)
**Practices:** this is a plan; code lives in `~/router-dav`. All changes are in `crates/frontend`
(plus minimal re-exports from `crates/ical` if the event builder needs one). No new dependency.

**Direction chosen** (from the options review): *Option 3 — extend the server-rendered Askama
portal with a month/week grid and form POSTs*, the "Roundcube analog" (Roundcube is
server-rendered HTML with incremental Ajax; we keep the server-rendered part and add no JS
framework). Scope: **read-only month view, create/edit/delete events, tasks (VTODO) and
contacts (CardDAV)**.

**What exists today:** `crates/frontend` is a management portal only — calendar/addressbook
*settings* (create, credentials, share/invite links, "set up on your device"). There is **no
event view**: `pages/calendar.html` renders the calendar's subscribe-link + per-client
instructions, not a grid. Events are only reachable through external CalDAV clients
(DAVx⁵/HawkBook, vdirsyncer, Apple, Thunderbird, Evolution). The JS that does exist is one
TypeScript→ESM `bundle.mjs` of Lit-style web components (`assets/js/bundle.mjs`, ~870 KB of
assets total) — used by `edit-calendar-form`, `delete-button`, etc. The whole frontend mounts
under `/frontend` inside the auth layer (`src/app.rs`), routing in `crates/frontend/src/lib.rs`.

---

## 1. The three confirm points, resolved against the code (not re-litigate)

### 1.1 Recurrence expansion (`expand_recurrence`) — **usable, with two known limits**

The grid will reuse the exact expansion the CalDAV `REPORT` time-range filter already uses:
`crates/caldav/src/calendar_object/resource.rs:63` calls
`self.object.get_inner().expand_recurrence(Some(start_utc), Some(end_utc))`.

- Implementation is caldata 0.16.2 (workspace pin, `Cargo.toml` line 132 with
  `chrono-tz` + `vtimezones-rs`):
  - `IcalCalendarObject::expand_recurrence` → `rustical_ical::CalendarObject::get_inner()`
    (`crates/ical/src/calendar_object.rs:108`).
  - `IcalEvent::expand_recurrence`
    (`caldata …/component/ical/component/event/mod.rs:184`): builds an `rrule-rs` set from
    `RRULE` + `RDATE` + `EXRULE` + `EXDATE`, honors `RECURRENCE-ID` overrides including
    `THISANDFUTURE`, and re-derives each instance's `DTEND` from the master duration.
  - **Cap:** `rrule_set.all(2048)` (event/mod.rs:210) — a month/week window is always far
    under this; a >2048-instance daily recurrence only truncates a >~5.6-year window, which
    the grid never renders.
  - Non-recurring events (no `RRULE`) return the master as-is (`event/mod.rs:193`).

- **Limit A — expansion is Event-only.** `IcalCalendarObject::expand_recurrence`
  (calendar_object.rs:277) matches `CalendarInnerData::Event(..)` and returns `Cow::Borrowed`
  for every other variant — including `Todo`. So a recurring VTODO renders its **master** in
  the tasks view; do not attempt per-instance task expansion in this pass (see §4).
- **Limit B — instances are UTC or floating-local.** `to_utc_or_local()` (event/mod.rs:196)
  leaves a floating (`Tz::Local`) time floating, and the expanded object's `timezones`/
  `vtimezones` maps are emptied (calendar_object.rs:288-290). Rendering must map instances
  into the calendar's `Calendar.timezone_id` for display; a floating-local start is rendered
  as wall-clock in that zone. This is **identical to what DAV reports today** (the REPORT
  path expands the same way), so it is accepted behaviour, not a regression. Note it in the
  template with a code comment.

### 1.2 Read vs write gates — **membership = read; `can_write` + `!is_read_only` = write**

Authorization is already modelable from `Principal` (`crates/store/src/auth/principal.rs`)
and `Privilege` (`privilege.rs`, `View | Edit | Admin`):

- `Principal::memberships()` = self + group ids (principal.rs:61); `is_principal(p)` =
  `id == p || memberships.contains(p)` (principal.rs:51).
- The existing Calendars screen lists **exactly** the calendars of `memberships()`:
  `routes/calendar.rs:200-203` (`for group in user.memberships() { get_calendars(group) }`),
  and `route_calendar` gates with `!user.is_principal(&owner)` (calendar.rs:394). So a
  `View`-privilege group member **can read** the group's calendars — same on the grid.
- `privilege_for(p)`: Admin for self, stored privilege (default Edit) for memberships, else
  View (principal.rs:75). `can_write(p)` = Edit|Admin (principal.rs:96).

**Grid rules, indistinguishable from DAV:**

| Action | Gate | Reference |
|---|---|---|
| view a calendar / the aggregate | `user.is_principal(&calendar.principal)` | calendar.rs:394 |
| list the aggregate | iterate `memberships()` → `get_calendars(group)` | calendar.rs:200-203 |
| show create/edit/delete forms | `user.can_write(&calendar.principal) && !cal_store.is_read_only(&cal_id)` | `is_read_only`: `calendar_store.rs:61` (mirrored / linked-platform / subscribe calendars are read-only even for the owner) |
| perform a write | re-check the same two conditions in the handler | calendar.rs:782 is the sibling `can_write` precedent |

Each route is gated identically to its existing sibling, so the grid **cannot expose more
than DAV already exposes**.

### 1.3 Contacts (CardDAV) — **full CRUD is already in the store**

`rustical_store::AddressbookStore` already has `get_objects`, `get_object`, `put_object`,
`delete_object`, `restore_object` (`crates/store/src/addressbook_store.rs:29-90`), and
`rustical_ical::AddressObject` wraps caldata's `VcardContact` with `get_vcf()` and `full_name`
(`crates/ical/src/address_object.rs:25-62`; the birthday/anniversary bridge at
`address_object.rs:64` proves FN/date fields are reachable). Contacts need **only** views +
form handlers; no new crate.

---

## 2. Design

### 2.1 Routes (new, all under the existing auth + `password_change_gate` layers)

Mount in the `user_router` in `crates/frontend/src/lib.rs` next to the existing calendar
routes (~line 125), reusing the current `CS: CalendarStore` / `AS: AddressbookStore` generics:

```
GET  /{user}/events?view=month&date=YYYY-MM              aggregate grid (all of memberships())
GET  /{user}/calendar/{calendar}/events?view=month&date=YYYY-MM   single-calendar grid
GET  /{user}/calendar/{calendar}/events/new               new-event form
POST /{user}/calendar/{calendar}/events                   create (calendar picker here if aggregate)
GET  /{user}/calendar/{calendar}/events/{object}/edit     edit form (prefilled)
POST /{user}/calendar/{calendar}/events/{object}          update (overwrite)
POST /{user}/calendar/{calendar}/events/{object}/delete   delete → trashbin
GET  /{user}/tasks?view=…                                 tasks list (VTODO, see §4)
GET/POST /{user}/addressbook/{book}/contacts + /{id}/…    contacts list + CRUD (§4)
```

The `view`/`date` are query params (month default); `Route` handlers use `axum::extract::Query`.
The URL `/{user}/events` is chosen to avoid colliding with `/{user}/calendar` (the settings
tab) and `/{user}/calendar/{calendar}` (the existing "set up on your device" page, which the
grid links to).

### 2.2 Data flow (no JSON, direct store access)

A grid handler:

1. `cal_store.get_calendar(&owner, &id, true)` (single view) or `for g in user.memberships()
   { get_calendars(g) }` (aggregate).
2. Compute the visible window: `NaiveDate` of the month start (±7 days margin for multi-week
   rows) → `CalendarQuery { time_start, time_end }` → `cal_store.calendar_query(..)`
   (`calendar_store.rs:32`), which returns `Vec<(String, CalendarObject)>`.
3. For each object: `obj.get_inner().expand_recurrence(Some(win_start_utc), Some(win_end_utc))`,
   project each event instance onto its day, split all-day vs timed, bucket by day.
4. Render `pages/calendar_view.html`: a `<table>` month grid; each event tile keyed by
   `calendar.meta.color` (already stored and used by `calendars_section.html`); prev/next
   month links are `<a href="?view=month&date=…">`. No JS needed for Phase 1.

### 2.3 Templates

- `crates/frontend/public/templates/pages/calendar_view.html` — month grid (extends
  `layouts/default.html`; the nav in `pages/user.html` adds a "Calendar" entry).
- `crates/frontend/public/templates/pages/event_form.html` — new/edit form (summary, start/end
  datetime, all-day flag, description, location, timezone prefilled from
  `calendar.timezone_id`, and — in the aggregate case — a calendar `<select>`).
- `crates/frontend/public/templates/pages/tasks.html`, `contacts.html` — Phase 3.

Reuse the existing web-component style only where a small JS nicety is wanted (e.g. a
`delete-button`-backed confirm); otherwise plain `<form method="POST">` like every other
portal form, so CSP stays as forgiving as today.

---

## 3. Phase breakdown

### Phase 1 — read-only month view (gate this first)

- New `crates/frontend/src/routes/calendar_view.rs` (module) with the read handlers + a
  `render_grid` helper; wire into `lib.rs`; add the `pages/calendar_view.html` template.
- Add the "Calendar" nav entry in `pages/user.html` pointing at `/{user}/events`.
- Gate: no writes, no JS, `cargo test` stays green, payload change ≈ negligible (pure HTML/CSS).

### Phase 2 — create / edit / delete events

- Event model: build the VEVENT with the same `caldata` builders `crates/ical` already uses
  (`IcalEventBuilder`, `IcalCalendarObjectBuilder` — see `crates/ical/src/address_object.rs:95`
  for a working builder precedent). New object id = a fresh `uuid` (the store keys objects by
  string ids: `calendar_store.rs:99`), `put_object(.., overwrite=false)`; edit =
  `put_object(.., overwrite=true)` on the existing id; delete = `delete_object(.., use_trashbin=true)`
  (restore already exists at `calendar_store.rs:122`).
- Handlers re-check the Phase-1 write gate **before** mutating, and `redirect` back to the
  grid (POST→Redirect→GET, existing idiom).
- `is_read_only` calendars and `View`-privilege memberships never see the form buttons.

### Phase 3 — tasks (VTODO) + contacts (CardDAV)

- **Tasks:** `/{user}/tasks` lists VTODO objects across calendars whose `calendar.components`
  contains `Todo`; CRUD reuses the Phase-2 handler shape. **Recurring tasks render their
  master only** (Limit A, §1.1) — note in the template, not a bug.
- **Contacts:** `/{user}/addressbook/{book}/contacts` lists `AddressObject` `full_name`
  (read-only list first), with new/edit/delete handlers call `AddressbookStore::put_object` /
  `delete_object`. Gate with the same membership/`can_write` rules, plus `is_read_only` if
  the addressbook store exposes one (mirror to calendar gate; confirm at implementation time).

---

## 4. Constraints & non-goals

- **Overlay budget 35 MiB** (appliance): this is pure server-rendered HTML + CSS against the
  existing ~870 KB assets; **no new JS payload** in Phases 1-3. (A later drag-and-drop
  sprinkle would be the only point budget pressure returns — explicitly out of scope.)
- **Memory cap 256 MiB** (`etc/init.d/rustical`, procd limit): read path is
  `calendar_query` within a bounded window, same as REPORT; no full-collection scan.
- **Reuse, do not fork semantics:** recurrence, timezone, and attendee/scheduling behaviour
  all stay in `crates/caldav` / `crates/scheduling`; the grid only *presents* what those
  already compute.
- **Non-goals:** replacing the client-instructions page, touching scheduling/RSVP, offline/PWA
  (that is `FUGUI.md` / Option-4 territory), drag-and-drop.

---

## 5. Gates

```sh
cd ~/router-dav/rustical
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace                      # was 827 at last handoff
cargo test --workspace --all-features       # was 382
```

Add frontend tests following the existing pattern in `crates/frontend/tests` (if the crate
keeps tests there) or `tests/`, covering at minimum: the membership-only listing, the
`can_write && !is_read_only` form-visibility gate, and the POST→Redirect→GET for create/edit.
Manually smoke against the host build
(`scripts/build-rust.sh x86_64-unknown-linux-gnu` + run the dev frontend) before deploy.

---

## 6. Remaining risks / follow-ups (recorded, not blocking)

> Resolved 2026-10-11 (implemented, still worth an owner look at #1):

1. **Floating-time display** (Limit B) — **resolved** as: browser `tzid` cookie first,
   then the calendar's own `Calendar.timezone_id` for single-calendar views
   (`display_timezone_fallback`, `routes/calendar_view.rs`), UTC last. Floating
   (no-zone) times keep their wall clock unchanged in every case, matching the DAV
   REPORT path. Owner confirmation optional before deploy.
2. **VTODO recurrence** (Limit A) — **decided: master-only stays.** A tasks *list*
   shows each VTODO once by design; per-instance expansion would belong to a grid.
   The list now marks recurring tasks with a "Recurring" chip (tooltip explains the
   master is shown once). caldata remains Event-only; no fork.
3. **Addressbook `is_read_only`** — **confirmed N/A.** Neither the `Addressbook`
   model nor `AddressbookStore` has any mirrored/linked-platform read-only variant;
   no guard to mirror. Documented in `routes/addressbook.rs`.
4. **Timezone list source** — **confirmed reuse:** both the `/_timezones.json`
   endpoint and the event/task form datalists serialize the same
   `vtimezones_rs::VTIMEZONES` map, now both sorted identically. No hardcoded list
   exists anywhere.