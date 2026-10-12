# PLAN_NEXT_AGENT_PRINTING — paper + sticker printing for events/tasks/contacts, with print-state tracking

**Date:** 2026-10-12
**Context:** a new greenfield feature, not a §18 row and not a follow-through on §18.36–§18.40.
The deployment's calendars also live on **paper on the wall**: the printouts are sticker
sheets (and letter-paper lists) that get cut apart and stuck onto a physical calendar.
The ask, verbatim from the user:

1. Print events from the **Profile (per calendar)**, **Calendar**, and **Tasks** tabs; print
   contacts from the **Contacts** and **Profile (per addressbook)** tabs.
2. Support printing on **8.5×11 paper** and **stickers**.
3. An option to print **only new or all** — where "new" means **previously unprinted**, so
   **we keep track of what is printed**.

**Repos** (same trio as `PLAN_NEXT_AGENT_APHY_IMPORT_RENDER.md`):
- `~/router-dav/rustical` — Omnical-Server, all the code. Head at writing: `43af1493`, tree clean.
- `~/router-dav` — Omnical-Code (CI, packaging, deploy).
- this repo — plans.

---

## 0. What exists now (verified 2026-10-12, all paths `crates/` relative to `rustical/`)

- The five data surfaces the feature prints from:
  - aggregate Calendar tab: `frontend/src/routes/calendar_view.rs:374` `route_events` —
    `user_calendars()` walks `memberships()`, every calendar, 6-week grid;
  - per-calendar grid: `calendar_view.rs:401` `route_calendar_events` (also the target of the
    Profile → Calendars tile);
  - Tasks tab: `routes/tasks.rs:92` `route_tasks` — every calendar carrying `VTODO`, masters
    only, recurring never expanded (`has_rruleset` chip);
  - aggregate Contacts tab: `routes/addressbook.rs:190` `route_contacts` —
    `user_addressbooks()` = same membership walk;
  - per-addressbook contacts: `addressbook.rs:119` `route_addressbook_contacts`, plus the
    Profile → Addressbooks tile (`routes/addressbooks.rs:95` `build_addressbook_section`).
- Reusable view helpers (currently `pub(super)`, promotion spec in §4.4):
  `text_prop` (`calendar_view.rs:150`), `display_timezone` (`:109`), `format_time` (`:160`),
  `user_calendars` (`:357`), `user_addressbooks` (`addressbook.rs:178`), `contact_item`
  (`addressbook.rs:55`), `format_time_short` (`tasks.rs:187`), `parse_month` (`:127`).
- The store substrate: sqlite via `sqlx`, migrations at
  `store_sqlite/migrations/` (naming `YYYYMMDDHHMMSS_name.{up,down}.sql`; newest is
  `20261011130000_calendar_source_objects`). The precedent for a **new side table + trait** is
  `store_sqlite/src/subscription_store.rs`: a small store wrapping `SqliteCalendarStore`'s
  pool, **runtime** `sqlx::query(...)` (the `.sqlx/` offline cache deliberately does not
  cover extension stores — keep it that way), trait in `store/src/subscription_store.rs`,
  `pub use` re-export in `store/src/lib.rs`.
- Wiring: `src/lib.rs:120` `get_store_bundle` builds every `Arc<dyn …Store>` from one
  `SqliteCalendarStore` clone each; `StoreBundle` struct (`src/lib.rs:~80`) carries them;
  `src/app.rs:~149` `run()` threads them to `frontend_router(...)` (`src/app.rs:377`, already
  `#[allow(clippy::too_many_arguments)]`, 13 positional args).
- The portal renders `Section`s through `pages/user.html` (nav: Profile/Calendar/Tasks/
  Contacts) and standalone pages through `layouts/default.html` (blocks: `title`, `imports`,
  `header` — which contains the logout form — and `content`; footer is inside
  `.content-pane`).
- The per-calendar "Download" export button lives in
  `frontend/public/templates/components/sections/calendars_section.html:156` (a GET form to
  `/caldav/principal/…?dailyview=1`, §18.38–§18.40 — **do not touch that path**); the
  addressbook tile's Download is `addressbooks_section.html:79`.
- CI (`~/router-dav/.github/workflows/test.yml`) pins **only**
  `cargo test --test run_integration_tests` — **121** tests + a name digest. New tests go in
  `tests/http_integration.rs` (not pinned, and **not** the frozen `tests/integration_tests/`).
- `tests/http_integration.rs:994` `test_portal_profile_sections_and_contacts` is the harness
  to copy: `test_runner_with` spawns the real server, CLI creates the principal + password,
  `SqliteCalendarStore::new(db, send, false).insert_calendar(Calendar { … })` seeds a calendar
  (`:1325` has the exact literal), `CalendarObject::from_ics` (`crates/ical/src/calendar_object.rs:97`)
  parses ICS fixtures, and a `reqwest` cookie client logs in via `POST /frontend/login`.

## 1. Decisions (made once, here — do not re-litigate in the implementation)

**D1 — Printing happens in the browser.** The print routes render HTML pages with
`@page { size: letter }` CSS; the user's own print dialog does the paper. No server-side PDF:
nothing in the workspace renders PDFs, the aarch64-musl deploy binary stays dependency-free,
and the browser's dialog handles printers, copies, and page ranges better than we ever will.

**D2 — One data set, two layouts.** `?format=` selects the layout only; the query (scope,
month, kind) is identical. Default `format=stickers` (the stated use is stickers on a
physical calendar); every print page carries toggle links that preserve the other params.

**D3 — Sticker geometry is fixed and letter-sized.** `@page` letter (8.5×11 in) with 0.5 in
margins → 7.5×10 in printable. Sticker cells **1.75 × 1.9 in**, **4 columns × 5 rows = 20 per
sheet**, dashed cut outlines. The server **chunks** the sticker list into `Vec<Vec<Sticker>>`
(sheets of 20) — deterministic pagination, because flex/grid page-breaking across print pages
is unreliable in exactly the browsers this household uses.

**D4 — "New" is tracked per *collection + object + event instance*, not per master.** The
wall-calendar semantics demand it: a weekly series ("Trash", Mondays) printed for October
must show up as **new again in November** — next month's instances are new stickers. Tasks and
contacts are whole-object marks (no recurrence expansion). An event whose date is *edited*
changes its instance key and becomes new again (correct: the new date needs a new sticker); a
summary-only edit does not — "all" scope always reprints.

**D5 — Marking printed is an explicit POST, not a side effect of rendering.** The print page
carries a "Mark these N as printed" form (one hidden input per rendered sticker) that POSTs to
the same URL. Rendering must not consume: opening a preview you never print would silently
empty "new", and that is a lie the user can't see. The POST marks, then redirects back with
`?marked=N` + a one-time banner. Self-correcting: forgot to mark → the same items are "new"
next time → you notice.

**D6 — Scope default is `new`; events take `?date=YYYY-MM`** (default = current month,
same `parse_month` fallback as the grid). No other filters on print: tasks print all
statuses (completed sort last, `.done` style), so "new vs all" stays the one and only knob.

**D7 — Print state is per-collection and shared across members** (keyed by the *owning*
principal). A group calendar printed by one member is printed for all — family semantics:
once the stickers exist on paper, nobody needs to print them twice. `printed_by` records who
did it. The gate on every print route is the **view** gate (`user.is_principal(&owner)` /
`user_id == user.id`, identical to the tab it prints from) — reading and printing are the same
trust class; marking print-ness is not a write to anyone's data.

**D8 — A new `PrintStore` trait + sqlite impl**, not columns on `calendarobjects`/
`addressobjects`: the rotation drill's protected-table semantics (`app_tokens`-only allowlist)
stay untouched, and per-instance keys (D4) don't fit a master row anyway. Runtime queries,
no `.sqlx` churn (`subscription_store.rs` precedent).

**D9 — The `kind` discriminator column is mandatory.** This deployment's own principal has a
calendar and an addressbook that can share the id `personal`; `(principal, collection_id)` is
not unique across kinds.

**D10 — The sticker/paper view model is kind-agnostic** (`title` + `lines` + `color`), so
one template serves events, tasks, and contacts; kind-specific cooking happens in Rust
(the house rule: templates are presentation-only).

**D11 — Print pages are standalone, not `UserPage` sections.** A printout with a nav bar and
a logout button is wrong. `pages/print.html` extends `layouts/default.html` and **empties the
`header` block** (askama: an empty child block override is all it takes), and print CSS hides
the footer.

## 2. Routes (all under the `/frontend/user` nest, `frontend/src/lib.rs`)

| Route | Method | Prints | Gate |
|---|---|---|---|
| `/{user}/events/print` | GET+POST | events, all calendars, `?date` month | `user_id == user.id` |
| `/{user}/calendar/{calendar}/events/print` | GET+POST | events, one calendar | `user.is_principal(&owner)` |
| `/{user}/tasks/print` | GET+POST | VTODOs, every `Todo` calendar | `user_id == user.id` |
| `/{user}/contacts/print` | GET+POST | contacts, all addressbooks | `user_id == user.id` |
| `/{user}/addressbook/{addressbook}/contacts/print` | GET+POST | contacts, one addressbook | `user.is_principal(&owner)` |

Every route POSTs to itself (the mark form's action is the same path); the POST body
semantics are identical in all five cases (the paragraph below).

Query params (GET): `format=paper|stickers` (default `stickers`), `scope=new|all` (default
`new`), `date=YYYY-MM` (events pages only). POST body: `marks: Vec<String>` where each value
is `"{object_id}\u{1f}{instance_key}"` (`""` key for tasks/contacts), the exact strings the
GET rendered as hidden inputs. POST does not re-validate the keys against the store (the
worst case is a mark on a non-existent object row, which upserts harmlessly into a state
nobody reads); it **must** keep `(principal, collection_id, kind)` from the URL, never from
the body.

## 3. Data layer

### 3.1 Migration `store_sqlite/migrations/20261012130000_print_state.{up,down}.sql`

```sql
-- up
CREATE TABLE print_state (
    principal     TEXT NOT NULL,     -- owning principal of the collection
    collection_id TEXT NOT NULL,     -- calendar or addressbook id
    kind          TEXT NOT NULL,     -- 'calendar' | 'addressbook' (D9: ids collide)
    object_id     TEXT NOT NULL,
    instance_key  TEXT NOT NULL,     -- '': whole object; events: start ("%Y%m%d" | "%Y%m%dT%H%M%SZ")
    printed_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    printed_by    TEXT NOT NULL,
    PRIMARY KEY (principal, collection_id, kind, object_id, instance_key)
);
CREATE INDEX idx_print_state_collection ON print_state (principal, kind, collection_id);

-- down
DROP TABLE print_state;
```

`migration_roundtrip.rs` will roll it down — the `DROP` is the whole down, the index dies with
the table.

### 3.2 Trait `store/src/print_store.rs` (re-export `pub use print_store::*;` in `store/src/lib.rs`)

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum PrintCollectionKind { Calendar, Addressbook }
// + as_str/try_from like SubscriptionKind

#[derive(Debug, Clone)]
pub struct PrintMark {
    pub principal: String,
    pub collection_id: String,
    pub kind: PrintCollectionKind,
    pub object_id: String,
    pub instance_key: String,
    pub printed_by: String,
}

#[async_trait]
pub trait PrintStore: Send + Sync + 'static {
    /// Upsert (idempotent; re-printing refreshes printed_at/printed_by).
    async fn mark_printed(&self, marks: &[PrintMark]) -> Result<(), Error>;
    /// The (object_id, instance_key) pairs already printed for one collection.
    async fn printed_keys(
        &self,
        principal: &str,
        kind: PrintCollectionKind,
        collection_id: &str,
    ) -> Result<HashSet<(String, String)>, Error>;
}
```

### 3.3 Impl `store_sqlite/src/print_store.rs` (`SqlitePrintStore::new(cal_store: SqliteCalendarStore)` wrapping the pool, exactly `subscription_store.rs`'s shape)

- `mark_printed`: one `INSERT … ON CONFLICT(principal, collection_id, kind, object_id,
  instance_key) DO UPDATE SET printed_at=CURRENT_TIMESTAMP, printed_by=excluded.printed_by`
  per mark, looped `.execute(self.cal_store.db_pool())` like `invite_store.rs` does — a
  half-marked run is re-POSTable (the form is still on the page), so no transaction
  complexity.
- `printed_keys`: `SELECT object_id, instance_key FROM print_state WHERE principal=? AND
  kind=? AND collection_id=?` via `.fetch_all(self.cal_store.db_pool())` → collect.
- Export from `store_sqlite/src/lib.rs` (`pub use print_store::SqlitePrintStore;`).

### 3.4 Wiring

- `StoreBundle` (src/lib.rs): `pub print_store: Arc<dyn PrintStore>`, built beside the share
  store: `Arc::new(SqlitePrintStore::new(cal_store.clone()))`.
- `frontend_router` (`frontend/src/lib.rs:99`) gains the 14th arg `print_store:
  Arc<dyn PrintStore>` and layers `.layer(Extension(print_store))` with the others.
- Two call sites to update: `src/app.rs:377` (thread from `StoreBundle`) and
  `crates/frontend/src/routes/password_reset.rs:598` (test: build a real
  `SqlitePrintStore` from its `cal_store` clone, like it already does for
  `SqliteCollectionShareStore`).
- Handlers extract `Extension(print_store): Extension<Arc<dyn PrintStore>>` — the exact
  pattern of `routes/share.rs:691`.

## 4. Frontend

### 4.1 New module `frontend/src/routes/print.rs` (all five GETs + the POSTs)

View models (cooked in Rust, D10):

```rust
pub struct Sticker { pub title: String, pub lines: Vec<String>, pub color: String }
pub struct PaperRow { pub label: String, pub lines: Vec<String>, pub color: String }

#[derive(Template, WebTemplate)]
#[template(path = "pages/print.html")]
pub struct PrintPage {
    pub title: String,          // "Events — October 2026" | "Tasks" | "Contacts — People"
    pub subtitle: String,       // "5 new (not yet printed)" | "all 12"
    pub base_query: String,     // preserved params for the toggle links: "date=2026-10&scope=new"
    pub format_stickers: bool,
    pub sticker_sheets: Vec<Vec<Sticker>>,  // empty in paper mode
    pub paper_rows: Vec<PaperRow>,          // empty in sticker mode
    pub post_action: String,     // the same path, for the mark form
    pub marks: Vec<String>,      // hidden-input values, one per rendered sticker
    pub marked_banner: Option<usize>, // from ?marked=N (D5)
    pub nothing_new: bool,       // scope=new && empty → hint + "print all" link
    pub mark_count: usize,
}
```

Cooking rules:

- **Events** (`?date` month, default now): range = `[first_of_month, first_of_next_month)`
  as UTC bounds (same `and_time(NaiveTime::MIN).and_utc()` discipline as
  `build_grid`), per calendar in scope: `store.calendar_query(...)` →
  `expand_recurrence(Some(start), Some(end))` → per instance: bucket **by its own start**
  (`bucket_date` semantics — same limitation as the grid: a multi-day event is filed at its
  start), keep instances whose start date is in the month. Sticker: `title` = the start date
  ("Wed 14 Oct", all-day ⇒ "All day" as the first line instead of a time), lines = time range
  (`format_time`), summary, location (if present), calendar display name. Paper: one row per
  instance, labelled by date. `instance_key`: DATE ⇒ `%Y%m%d`, DATE-TIME ⇒ UTC
  `%Y%m%dT%H%M%SZ`.
- **Tasks**: per calendar with `Todo` component, `get_objects`, masters only (Limit A — never
  expand VTODO). Sticker: `title` = summary, lines = due ("due Wed 14 Oct" / "due 19:00"),
  status, calendar name. Sort: not-completed first by due, completed last (`format_time_short`,
  same sort as the tab).
- **Contacts**: `get_objects` per addressbook → `contact_item`-shaped cooking (name /
  email / phone / addressbook name), `title` = name, sort by lowercase name (same as the tab).
- **scope filter**: after cooking, fetch `printed_keys(owner, kind, collection)` once per
  collection and drop marks already in it when `scope=new`. The **marks** vec = exactly the
  rendered stickers' keys (so "Mark these N printed" is always in lockstep with what's on
  the page — in either scope).
- Timezone: `display_timezone(cookie tzid)` + calendar fallback, same as the grid.

### 4.2 Template `frontend/public/templates/pages/print.html`

- `{% extends "layouts/default.html" %}`, `{% block header %}{% endblock %}` (empties the
  chrome — D11), title block `Print — {{ title }}`.
- Screen-only toolbar (`.print-toolbar`): heading, scope/format toggle links built from
  `base_query`, a `Print…` button (`onclick="window.print()"`), the mark form (§2), the
  `marked_banner` / `nothing_new` notices, and a cut-guide note in sticker mode.
- Sticker mode: `{% for sheet in sticker_sheets %}<div class="sticker-sheet">…` cells
  `1.75×1.9in` with dashed `outline`, `title`/`lines` inside, a `--event-color` accent bar
  from `color`. Paper mode: `.paper-list` rows with `break-inside: avoid`.
- Footer (`<footer>` in the layout): `@media print { footer { display: none } }` in CSS.

### 4.3 CSS `frontend/public/assets/style.css` (append; the file is 1246 lines, served both dev (`ServeDir`) and release (`EmbedService`))

```css
@page { size: letter; margin: 0.5in; }
@media print {
  .print-toolbar { display: none; }  /* the toolbar is screen-only */
  footer { display: none; }
  body { background: none; }
}
.sticker-sheet { display: flex; flex-wrap: wrap; width: 7.5in; page-break-after: always; }
.sticker { width: 1.75in; height: 1.9in; outline: 1px dashed #999;
           outline-offset: -4px; padding: 0.1in; box-sizing: border-box;
           break-inside: avoid; overflow: hidden; }
.sticker .sticker-title { font-weight: 600; }
/* + .paper-list / .paper-row in the same idiom as .task-list */
```

(`@page` + `.sticker-sheet` widths are the letter-math of D3; do not improvise a second
geometry.)

### 4.4 Buttons on the five source surfaces (links carrying `format`, the scope/format toggles live on the print page itself)

- `components/sections/calendar_grid_section.html` toolbar: `CalendarGridSection` gains
  `print_paper_href` + `print_stickers_href` (+`?date=` passthrough of the *displayed*
  month), aggregate → `/{user}/events/print`, single → `/{user}/calendar/{cal}/events/print`
  (built in `build_grid` alongside `new_href`).
- `components/sections/tasks_section.html` `.tasks-filters`: two links → `/{user}/tasks/print`.
- `components/sections/contacts_section.html`: two links → `/{user}/contacts/print`.
- `calendars_section.html` tile `.actions` (beside Download, `:156`): two links →
  `/frontend/user/{principal}/calendar/{id}/events/print` — **the URL path uses the owning
  `calendar.principal`**, like the Download form, so group calendars print.
- `addressbooks_section.html` tile `.actions` (beside Download, `:79`): two links →
  `/frontend/user/{principal}/addressbook/{id}/contacts/print`.
- Helper promotion: `text_prop`, `user_calendars`, `user_addressbooks`, `contact_item`
  become `pub(crate)` (they already are module-internal; `print.rs` joins the family).

## 5. Tests — `tests/http_integration.rs` (NOT `run_integration_tests` — the CI pin stays 121)

Copy the `test_portal_profile_sections_and_contacts` harness. Seed with
`insert_calendar`/`insert_addressbook` + `put_object` (fixtures parsed by
`CalendarObject::from_ics` — do not rebuild objects through caldata builders):

1. `test_print_events_stickers_new_then_all` — calendar `personal` (components `Event`),
   event `DTSTART:20261014T180000Z` + weekly `RRULE` event Mondays from 2026-10-05.
   GET `/frontend/user/user/calendar/personal/events/print?format=stickers&scope=new&date=2026-10`
   → 200; body: `@page` CSS class markers, the dated sticker, 4 recurrence stickers
   (Oct 5/12/19/26), the mark form's hidden inputs, `letter` size. POST the marks (follow
   redirect) → GET `scope=new&date=2026-10` → "nothing new" state; GET `scope=all` →
   stickers still there.
2. `test_print_events_next_month_instances_are_new` (the D4 gate) — after marking October,
   `?date=2026-11&scope=new` → the 5 November Mondays appear (proves instance keys, not
   master marks).
3. `test_print_tasks_and_contacts` — calendar with `Todo` + a VTODO with DUE; addressbook
   `people` + a vCard with TEL/EMAIL. GET `/{user}/tasks/print`, `/{user}/contacts/print`,
   `/{user}/addressbook/people/contacts/print`, and `format=paper` variants (paper rows
   render, sticker sheet absent) → 200 + content + mark round-trip on each.
4. Gates: a second principal (non-member) GET the per-collection prints → UNAUTHORIZED; own
   session GET `/{other}/events/print` → UNAUTHORIZED.
5. `migration_roundtrip` picks the new migration up automatically (down.sql drops).

## 6. What NOT to do

- Do **not** touch `crates/caldav/src/dailyview.rs` or the `?dailyview=1` export — §18.36/37
  shims are inert by design; the Download button on the tile stays as it is.
- Do **not** edit `tests/integration_tests/` (frozen) or the `run_integration_tests` pin
  (121 + digest — unchanged, no tests are added there).
- Do **not** mark printed on GET (D5), and do **not** key events per-master (D4) — the
  November test (§5.2) exists to fail both mistakes.
- Do **not** drop the `kind` column (D9) — `personal` calendar and `personal` addressbook
  under one principal is this deployment's actual shape.
- Do **not** put `.sqlx` macro queries in the new store (`subscription_store.rs`/§18.29 discipline).

## 7. Gates + housekeeping (per `PLAN_NEXT_AGENT_APHY_IMPORT_RENDER.md` §4)

```sh
cargo fmt --all
SQLX_OFFLINE=true cargo clippy --workspace --all-targets -- -D warnings   # zero
SQLX_OFFLINE=true cargo test -p rustical_caldav                            # 73+
SQLX_OFFLINE=true cargo test --test run_integration_tests                 # 121, unchanged
SQLX_OFFLINE=true cargo test --test http_integration                      # new prints included
# deploy: scripts/build-rust.sh → out/rustical, then ./deploy.sh; verify sha vs router
```

- Commit order: this plan is already committed; the implementation commits once, message in
  the house style (`frontend: …`). `PLAN_DEPLOYMENTS.md` gains the shipping note as the next
  free §18 subsection (§18.41 if the aphy session hasn't taken it — check before writing).
- CI pins: only `run_integration_tests` name-digest changes matter; none here.
- Manual verification after deploy (the rehearsal the user runs): print-preview the October
  sticker sheet from the profile tile and from the Calendar tab, the tasks and contacts
  sheets, a paper-format list, mark a run, confirm the next "new" excludes it, and confirm
  November recurrence instances appear new.

## 8. Definition of done

- [ ] Five GET print routes + their POST marks, gated like their tabs, reachable from the
      five surfaces (§2, §4.4).
- [ ] `format=stickers` renders 4×5 letter sheets; `format=paper` renders letter lists; toggle
      links preserve scope and date (D2/D3).
- [ ] `scope=new` excludes exactly what was last marked printed, per instance for events,
      per object for tasks/contacts; marking is an explicit POST + banner (D4/D5).
- [ ] Migration + trait + sqlite impl + wiring, runtime queries only (§3).
- [ ] Tests §5 green under the §7 gates; CI pin 121 untouched.