# Omnical — the aphy week/day render gap: characterize the channel, then decide

**Date:** 2026-10-12 (evening)
**Context:** the follow-through on PLAN_DEPLOYMENTS.md **§18.36–§18.40 — read those first**; they
carry the diagnosis, the shipped code, and the verified pipeline. This plan picks up at the
verified dead end: the phone pipeline **works** with the phone-shaped export, aphy's web
week/day views **still drop every imported event** (while aphy-native events render in the same
views), and the flattened import **removed the collective operations** on recurring events that
the canonical import used to give the user on aphy's web UI.

**Repos:**
- `~/router-dav` — Omnical-Code (CI, packaging, deploy; the `rustical` submodule pointer)
- `~/router-dav/rustical` — Omnical-Server, the product itself (all the code lives here)
- `~/Documents/ByteWheel/omnical` — this repo (plans; PLAN_DEPLOYMENTS.md is the running log)
- `~/Documents/ByteWheel/dailyview-mod` — the DailyView app's source (GPLv3), the **diagnosis
  oracle** for anything phone-side; do not modify (the phone cannot run a mod build anyway)

---

## 0. The pipeline — verified in the app source; do not re-derive without cause

1. The stock MP02's DailyView syncs **aphy.io** (`dav.aphy.app`), never omnical — the device is
   locked, `dailyview-mod` cannot install. **Every** omnical→phone byte flows: omnical export
   `.ics` → **manual import into the aphy web UI** → aphy stores → phone syncs aphy.
2. Phone internals (`dailyview-mod/ACalDAV/src/com/aphy/…`): event bodies come from
   `calendar-multiget` REPORT `<C:calendar-data/>` (`CaldavFacade.getEvent`); every event is
   filed at its **master's `DTSTART`** (`CalendarEvent.getTimestamp`: UTC millis − rawOffset −
   DST) into one flat SQLite table (`UserDBHelper.cal_info`, **`UID UNIQUE`**, wiped +
   reinserted on every manual sync); the 4-day list is a plain SQL window
   `DtStart between [local midnight, +4d)`; **no `RRULE` is ever expanded**; `parseIcs` keeps
   only the **first** `VEVENT` of each body.
3. Consequences, all confirmed live:
   - a **canonical** export (recurrence masters) is invisible on the phone forever (master
     `DTSTART` sits outside the 4-day window);
   - **flattened** concrete instances are correct on the phone (verified 2026-10-12: Trash,
     Church, Recycling all show on their own days in DailyView);
   - `VALUE=DATE` all-day events file at UTC midnight → **one day early** west of UTC; the
     export's all-day pass (DATE → `TZID`-qualified local midnight) fixes that on the phone
     (§18.38) — keep it.

## 1. Shipped & verified state (as of 2026-10-12 ≈15:51 — all current)

- **rustical @ `43af1493`** (branch `omnical-scheduling`): §18.36 GET all-day wire shim;
  §18.37 REPORT wire flattening (both **inert by design** for this deployment — the phone never
  syncs omnical; kept as insurance, do not remove without the user); §18.38 the export route
  `?dailyview=1` + **one** `Download` button in the UI (the shaped export — the second
  "(Punkt MP02)" button was collapsed in §18.40); §18.39 `DTEND` materialized, no `DURATION`
  lines; §18.40 instances served in the deployment's own zone (`TZID`-local), the form aphy
  itself writes. All committed/pushed; tree clean.
- **Omnical-Code @ `dec13b0`/`503ad68`**: `.github/workflows/test.yml` baseline pins —
  **121** integration tests, name digest `c8bb7123…` (committed; matches the tree).
- **Deployed:** `/usr/sbin/rustical` on the router = sha `e7ce754d…` == `out/rustical` (built
  15:49). The running binary **is** current.
- **Gates green at `43af1493`:** `cargo fmt --all` clean; `SQLX_OFFLINE=true cargo clippy
  --workspace --all-targets -- -D warnings` zero; caldav lib **73/73**; integration **121/121**.
- **The user's last test artifact:** `~/Downloads/nfcalaway@novo-ordo.com_personal_personal.ics`
  (15:51, ~15 KB) = the current §18.40 export. A sample flattened instance (the current shape —
  keep this regardless of what follows):

  ```text
  BEGIN:VEVENT
  SUMMARY:Trash
  DTSTART;TZID=America/New_York:20261012T190000
  DTSTAMP:20260925T103702Z
  UID:RGKW68NDCGRYYVVHH55NLH5PU8Z6QH1IDKBU-20261012T230000Z
  SEQUENCE:0
  DTEND;TZID=America/New_York:20261012T190100
  END:VEVENT
  ```

  envelope: `VERSION:2.0` / `PRODID:RustiCal Export (DailyView)` / `CALSCALE:GREGORIAN` /
  `X-WR-CALNAME:personal` / one `VTIMEZONE` (America/New_York via vtimezones-rs, RRULE-bearing
  transitions) — plus a **90-day horizon** of concrete instances with per-instance **suffixed
  UIDs** (upsert-safe), `RECURRENCE-ID` dropped, alarms stripped by the expansion. Non-recurring
  events, `VTODO`s and `VJOURNAL`s pass through canonical, all-day pass applied to the whole
  assembled text.
- **`PLAN_DEPLOYMENTS.md` §18.38–§18.40 are appended but UNCOMMITTED** — commit them first
  (this repo's habit: one commit per plan section).

## 2. The symptom matrix (all four cells verified on aphy, 2026-10-12)

| What aphy holds | month view | week/day views | phone (DailyView) | series ops on aphy web |
|---|---|---|---|---|
| canonical import (RRULE masters) | ✓ | **✗** | ✗ (master `DTSTART` out of window) | ✓ |
| §18.39/§18.40 flattened import | ✓ | **✗** | **✓** | **✗** (13 standalone instances) |
| aphy-native events (created in their UI) | ✓ | **✓** | — | ✓ |

The decisive fact: **native events render in the same week view that drops both forms of
import**. So the gap is in aphy's **import-to-render path**, not in our recurrence semantics
(flattened instances have none), and §18.40 falsified the UTC-`Z`-vs-`TZID` form hypothesis
(TZID-local tested at 15:51, no change). The user also values the **series ops** the
flattening removed.

## 3. The experiments — in order of information per effort

### 3.1 The channel test (CalDAV `PUT` vs web-UI import) — run this first

Working hypothesis: aphy's week/day grids render events whose storage rows their **own create
path** wrote (web-UI create ≈ CalDAV `PUT` under the hood), while their **import** path writes
rows the grid query excludes (a missing internal column, a source tag, an unregistered tz —
whatever; the create path populates it, the import path does not). This is the shape that makes
all four matrix cells true at once.

1. Ask the user for their aphy CalDAV URL + credentials (they created the account; the stock
   phone account is `dav.aphy.app`).
2. Create a **scratch calendar on aphy**. `PUT` one flattened instance — exact bytes from the
   15:51 file, change only the `UID` — with `curl -u … -X PUT --header 'If-None-Match: *'`.
3. Check aphy's web **week view**.
   - **Renders** → the ingest channel is the discriminator. The end-state is a **CalDAV sync into
     aphy** instead of the manual web import: either a vdirsyncer pair (omnical as source, aphy
     as target) or a small script that `PUT`s the shaped export per-instance on a schedule
     (the export route already yields the bodies; a fetch-and-`PUT` loop is ~30 lines).
     Then re-test the matrix — including whether **canonical masters `PUT` to aphy** keep
     series ops *and* whether a **second (phone-feed) calendar of flattened instances** is
     still needed for the phone (it will be: `PUT` canonical masters → phone files master
     `DTSTART` → out of window again. Two aphy calendars: "personal" canonical for web ops,
     "personal (phone)" flattened for the MP02. The stock phone syncs *every* calendar it
     discovers — canonical masters file out-of-window → invisible on the phone, so no
     duplicates in-window today; future-dated one-shot events will double on the phone —
     document that cost for the user).
   - **Doesn't render** → 3.2.

### 3.2 The byte bisect (only if 3.1 fails)

1. Export an aphy-**native** event's own bytes (aphy web UI export if it has one, else CalDAV
   `GET` of a native event). Field-inventory diff against our instance: `TRANSP`, `STATUS`,
   `ORGANIZER`, `CREATED`/`LAST-MODIFIED`, `DTSTAMP` recency, `VALARM`, `UID` shape
   (`…@aphy.io`?), `X-*` props, property order, `VTIMEZONE` shape, folding, CRLF.
2. **Re-import the native event's exact bytes** (change only `UID`) through the web import →
   renders?
   - **Renders** → the difference is in our bytes. Bisect fields into `flattened_instance`
     (first candidates: `TRANSP:OPAQUE`, `STATUS:CONFIRMED`, `VTIMEZONE` identity — their
     importer may register only tz definitions it already knows; `DTSTAMP`/`CREATED` recency),
     one variable per import round, gates + redeploy per §4, verify on aphy's week view.
   - **Doesn't render** → aphy tags imports by **source**, not bytes. No export shape can fix
     their grid; go to 3.3.

### 3.3 The report + the decision (close this out either way)

- Package the repro for **aphy support** (the user is their customer): native event ICS bytes +
  our flattened instance + the one-sentence symptom ("native renders week/day; the same event's
  bytes re-imported do not"). Their import path is a **product bug in their system**; that is
  the actual resolution route for week/day.
- Present the trade to the user, with the matrix:
  - **(a) keep the flattened import** (current): phone ✓, web month ✓, web week/day ✗ until
    aphy fixes, series ops ✗;
  - **(b) two aphy calendars** (web-ops canonical + phone-feed flattened): series ops ✓ on the
    canonical calendar, week/day per 3.1's finding;
  - **(c) channel change to a CalDAV sync** (if 3.1 renders): the cleanest end-state — the
    manual import step disappears, week/day likely works (their own path), phone keeps the
    flattened feed.
- Whichever is chosen: **§18.41** in PLAN_DEPLOYMENTS.md documents it (append-only), plus any
  code delta (the sync script / vdirsyncer config lives in Omnical-Code, not rustical, if 3.1
  goes that way — rustical is only touched if `flattened_instance` gains fields).

## 4. Housekeeping (every step)

- Commit `PLAN_DEPLOYMENTS.md` first (§18.38–§18.40 appended, uncommitted).
- test.yml pins **only if test names change**: count `121`, digest `c8bb7123…`; re-derive via
  the pipeline in that file's header (`cargo test --test run_integration_tests -- --list |
  grep -oE '^integration_tests::.*: test$' | LC_ALL=C sort | sha256sum`).
- Gates for any rustical change: `cargo fmt --all`; `SQLX_OFFLINE=true cargo clippy
  --workspace --all-targets -- -D warnings` (zero); `SQLX_OFFLINE=true cargo test -p
  rustical_caldav` (73+); `SQLX_OFFLINE=true cargo test --test run_integration_tests` (121).
- Deploy: `scripts/build-rust.sh` (aarch64 musl → `out/rustical`) then `./deploy.sh` (stages via
  `/tmp`, brief rustical stop/swap, dav-tls sha-check → no restart). Verify currency after:
  `ssh router sha256sum /usr/sbin/rustical` == local `sha256sum out/rustical`.
- Re-import discipline: **delete the aphy calendar's imported copy before re-importing** — the
  suffixed UIDs are built to upsert, but aphy's import-vs-append behavior is unverified; stale
  rows shadow every test. Keep a scratch aphy calendar for the experiments (§3) so the user's
  real calendar is never the test bed.
- Phone check after any change: manual "Sync calendar", then the 4-day list — Trash 19:00
  today, Church 09:00 Sunday, Recycling 19:00 Thursday; St Luke all-day Oct 18 (in-window from
  Oct 15).

## 5. What NOT to redo

- The §18.36/§18.37 UA wire shims are inert for this deployment **by design** — keep them.
- The 90-day horizon: right for a manual import cadence; revisit only if §3.1 moves the
  channel to a synced feed (a short horizon — e.g. 14 days — would then stay perpetually
  fresh).
- The §18.39/§18.40 instance shape (`DTSTART`+`DTEND` `TZID`-local pair, no `DURATION`, suffixed
  UIDs, no `RECURRENCE-ID`): keep regardless of §3's outcome — it is aphy-native form, it is
  the phone's native parse path, and no renderer drops a concrete `DTSTART`+`DTEND` pair.