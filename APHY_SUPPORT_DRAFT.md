# Draft — aphy support package (§18.41 findings; review before sending)

**To:** aphy.io support
**From:** a burningserenity@aphy.io account holder
**Subject:** Three calendar behaviors found while syncing a self-hosted CalDAV server to aphy

---

Hello,

I sync a self-hosted CalDAV calendar server (RustiCal) into my aphy
account over standard CalDAV (`PROPFIND`/`PUT`/`DELETE` against
`dav.aphy.app`), and my events are authored by khal, a standard
vdir-based CLI calendar tool. While getting events to render in the aphy
web UI I ran into three behaviors that I believe are product issues
rather than client bugs. Each is reproducible with plain ICS files and a
web browser. I'm happy to provide full transcripts or test calendars if
useful.

## 1. Week/day views silently drop events without ORGANIZER or CREATED

A `VEVENT` that lacks an `ORGANIZER` (or a `CREATED`) property renders
in the **month** view but is **absent from the week and day views** —
no error, no indication of any kind. Month view expands and renders
the very same object, so the event is stored; only the week/day grids
filter it out.

**Repro (web import):** import this file via the aphy web UI, then
compare month vs week views for its date:

```text
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//PIMUTILS.ORG//NONSGML khal / icalendar //EN
BEGIN:VEVENT
SUMMARY:Repro A - no organizer
DTSTART;TZID=America/New_York:20261019T190000
DTEND;TZID=America/New_York:20261019T190100
DTSTAMP:20261013T000000Z
UID:repro-a@example.test
SEQUENCE:0
END:VEVENT
END:VCALENDAR
```

Result: a chip in the month grid, nothing in the week/day grid. Now
import the identical event with a single line added (`ORGANIZER:…` or
`CREATED:…`) and a different UID — it renders in all three views:

```text
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//PIMUTILS.ORG//NONSGML khal / icalendar //EN
BEGIN:VEVENT
SUMMARY:Repro B - with organizer
DTSTART;TZID=America/New_York:20261019T190000
DTEND;TZID=America/New_York:20261019T190100
DTSTAMP:20261013T000000Z
UID:repro-b@example.test
SEQUENCE:0
ORGANIZER:mailto:me@example.test
END:VEVENT
END:VCALENDAR
```

The same split happens with a direct CalDAV `PUT`, so it is not
import-path-specific.

**Why it matters:** khal (and several other tools) never write
`ORGANIZER` on personal events — it is optional for non-scheduling use
in RFC 5545. Every such event looks "missing" in week/day to a user who
did nothing unusual. Either rendering such events (an event with no
organizer is a perfectly normal personal event) or surfacing a warning
in the UI would save users from concluding their data was lost.

## 2. Copying an event into a second calendar of the same account: an opaque 403

A `PUT` of a `VEVENT` whose `UID` already exists in **another calendar
of the same account** is rejected with `403 Forbidden`. The response
body does carry a CalDAV `unique-scheduling-object-resource` error
element naming the conflicting resource, so the server side is
well-behaved — but nothing surfaces the reason anywhere a user (or a
generic CalDAV client) can see it. From the client's seat it is an
ordinary PUT of an ordinary event answered "forbidden", with no
pointer to the precondition or the conflicting calendar.

**Repro:** create an event in calendar 1 (via the web UI), then `PUT`
an ICS with the **same `UID`** into calendar 2 of the same account:

```bash
curl -u 'user@example.test:***' -T event.ics \
  https://dav.aphy.app/dav/calendars/user/user@example.test/second-calendar/repro.ics
```

Result: `403 Forbidden` (response body names the conflicting href via
the `unique-scheduling-object-resource` precondition).

**Why it matters:** the account-wide UID rule (RFC 6638 scheduling
semantics applied across all of a user's calendars) is a defensible
server choice, but as shipped it means (a) a user cannot keep the same
event in two of their own calendars — e.g. a canonical copy in one
calendar and a derived feed in another — without mutating UIDs, and
(b) the failure mode is invisible: no client I tried reports *why*.
A friendlier error message in the web UI, or documentation of the
rule, would have saved me hours of diagnosis.

## 3. Ten-minute events render as near-invisible slivers

A short event (e.g. 19:00–19:10) renders in the week/day grids as a bar
roughly one pixel tall — visually indistinguishable from nothing at
all. During my own testing I twice concluded such events "did not
render" when in fact they were present as hairlines.

**Repro:** import an event with
`DTSTART;TZID=America/New_York:20261019T190000` and
`DTEND;TZID=America/New_York:20261019T190100` (with an `ORGANIZER`, per
finding 1) and look for it in the week view.

**Suggestion:** a minimum visual height for short events, or an
overflow marker when the true height is sub-renderable, would make
"it's there but small" distinguishable from "it's missing".

## Environment

- Account: `burningserenity@aphy.io` (happy to run any of these on a
  scratch calendar with you, or to share the exact test files and
  transcripts).
- Server: `dav.aphy.app`, observed 2026-10-12/13.
- Counterparty: self-hosted RustiCal (CalDAV server) + khal (vdir
  CLI) + standard CalDAV client requests; events also imported through
  the aphy web UI.
- All three behaviors are consistent across CalDAV `PUT` and web-UI
  import, ruling out my client.

Thanks — overall the CalDAV stack has been solid to work with; these
three are the only rough edges I hit, and none of them block me (I
stamp `ORGANIZER` server-side now). I'm reporting them in the hope
they save other sync users the same archaeology.

---

*Draft notes (remove before sending): evidence for finding 1 was
established by a nine-round bisect against a scratch calendar (§18.41
mechanism 1: ORGANIZER or CREATED — either suffices). Finding 2's error
body was captured live: the 403 carries a
`C:unique-scheduling-object-resource` element naming the conflicting
href (§18.41 mechanism 2). Finding 3: two "doesn't render" bisect
answers were thin-bar false negatives (§18.41 mechanism 4). Do not
include passwords or real UIDs in the sent version; the sample UIDs
above are synthetic.*
