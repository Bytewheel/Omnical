# PLAN_NEXT_AGENT_MIRROR_SYNC_UI_FINISH — finish §18.42 (mirror UI + per-user timezones) + §18.43 (contacts import + plain-language help)

Date: 2026-10-13
Repos:
- **Code**: `/home/burningserenity/router-dav/rustical` — branch `omnical-scheduling`, **everything UNCOMMITTED** (do not commit unless the user asks).
- **CI/deploy**: `/home/burningserenity/router-dav` (Omnical-Code) — untouched so far.
- Planning repo (`~/Documents/ByteWheel/omnical`) — this file. The parent plan is
  `PLAN_NEXT_AGENT_MIRROR_SYNC_UI.md` (same directory) — **read its §1, §6–§9 first**;
  this file assumes it.

## 0. Where the previous agent stopped

Interrupted **mid-clippy-fix-loop**. `cargo check --workspace --all-targets` was green,
`cargo fmt --all` had been run once (edits continued after it), the JS bundle was
rebuilt, and exactly **3 clippy errors remained** (§3.1 below — two still-unfixed,
one probably fixed). **The frontend test suite has never been run** — §3.2–§3.4 are
already-known test bugs found by reading, not running. Fix §3 first, then run §4.

## 1. Scope

1. **§18.42** (parent plan): the aphy mirror sync exposed in the profile's Calendars
   section + per-user IANA timezones (owner's setting → deployment
   `[caldav] dailyview_timezone` → UTC).
2. **Mid-session user additions (both already implemented — finish them)**:
   - **Contacts import** from other sources/files (Google et al.): `.vcf` (universal)
     + Google/Outlook contacts CSV.
   - **Plain-language instructions** for non-technical people (profile intro + the
     app-token help).

## 2. State at handoff — all of this COMPILES (last full `cargo check --workspace --all-targets` was clean)

### 2.1 §18.42 app-layer wiring (DONE)

- `src/app.rs`: `AppConfig` += `mirror_targets: Vec<MirrorTargetConfig>`,
  `mirror_timezone: String`; `AppStores` += `mirror_store: Arc<dyn MirrorStore>`;
  `make_app` (the positional shim) gained **+3 params** after `print_store`:
  `mirror_store`, then `smtp_accounts`, then `mirror_targets`, `mirror_timezone`
  (note the order: smtp sits between the store and the two config values).
  `make_app_for` builds `Arc<MirrorUiContext>` **only when `!mirror_targets.is_empty()`**
  (None ⇒ no context ⇒ UI absent, byte-for-byte the unconfigured build) and passes it
  to `frontend_router`.
- `src/store_bundle.rs` `app_stores()`: populates `mirror_store`.
- `src/tenancy.rs` `app_config_for()`: `mirror_targets` from `config.mirror.targets`,
  `mirror_timezone` from `config.caldav.dailyview_timezone()` (UTC fallback).
  **This is the production (router) path.**
- `src/lib.rs` `cmd_serve`: `make_app` call passes `mirror_store.clone()`,
  `config.scheduling.smtp.clone()`, `config.mirror.targets.clone()`,
  `mirror_timezone.clone()`; the background-loop spawn now also passes
  `principal_store.clone()`.
- `src/tasks.rs` `run_mirror_target(target, cal_store, mirror_store, auth_provider,
  deployment_timezone, shutdown)`: the owner zone is resolved **per tick**
  (`get_principal(&target.principal)` → `.timezone` → `resolve_timezone`), so a user
  changing their zone mid-uptime is picked up. A missing owner row degrades to the
  deployment rung, never fails the sync.
- `src/commands/mirror.rs`: the **13**-tuple destructure now names `auth_provider`
  (4th element); per-target owner tz lookup + `resolve_timezone`; deployment tz from
  `[caldav] dailyview_timezone`.
- All `make_app`/`AppConfig`/`AppStores` construction sites updated:
  `tests/integration_tests/mod.rs` (mirror store + `vec![]` + `"UTC"`),
  `tests/tenant_app.rs`, `tests/miss_cost.rs`.
- The `mirror` module doc in `src/lib.rs` (`pub use rustical_caldav::mirror;`) and
  `src/config.rs` re-exports are from the handoff (§2.1 of the parent plan) — done.

### 2.2 §18.42 frontend wiring (DONE)

- `crates/frontend/Cargo.toml`: `rustical_caldav.workspace = true` (no cycle).
- **NEW `crates/frontend/src/routes/mirror.rs`**:
  - `pub struct MirrorUiContext { store: Arc<dyn MirrorStore>, targets:
    Vec<MirrorTargetConfig>, deployment_timezone: String, in_flight:
    Mutex<HashSet<String>> }` — **not** a bare `Extension<String>` (that type is the
    subscriptions public URL).
  - `MirrorTargetUi { name, remote_label, kind_label, last_run,
    last_run_relative }` — the `can_sync` field was **removed**; the template gates
    the button on `CalendarTile.can_generate` instead.
  - `remote_label()` (host + last path segment), `kind_label()` (`const`),
    `mirror_targets_for(ctx, principal, calendar_id)`.
  - `route_post_mirror_sync`: gates `user_id == user.id`; missing context ⇒ **404**
    (password-reset pattern); `form.principal == user.id || is_admin`; calendar
    exists; targets resolved **from the context, never the form**; in-flight
    try-insert ⇒ "already syncing" line; **awaits**
    `rustical_caldav::mirror::sync_target` per target with the **owner's** zone
    (`deps.auth_provider.get_principal(&target.principal)` → `resolve_timezone`);
    per-target result lines land in `CalendarsExtras::mirror_synced`. Lock poison
    handled via `unwrap_or_else(PoisonError::into_inner)`.
- `crates/frontend/src/lib.rs`: `frontend_router` += `mirror:
  Option<Arc<MirrorUiContext>>` (before `smtp_accounts`); `Extension(mirror)`
  layered; user_router mounts `/{user}/calendar/mirror-sync` and `/{user}/timezone`;
  `pub use routes::mirror::MirrorUiContext`; imports updated.
- `routes/user.rs`: `ProfileDeps` += `mirror` (extracted as
  `parts.extensions.get::<Option<Arc<MirrorUiContext>>>().cloned()`); `ProfileExtras`
  += `timezone_banner`; `ProfileSection` += `timezone_options:
  Vec<(&'static str, bool)>` (name, selected), `timezone_banner`, `timezone_open`;
  `render_profile_page` builds the sorted `chrono_tz::TZ_VARIANTS` list with
  selected flags; **`route_post_timezone_save`** (validates
  `chrono_tz::Tz::from_str`, empty ⇒ clear, `auth_provider.set_timezone`, re-renders
  with the banner and the refreshed `user.timezone` so the `<select>` shows it
  selected).
- `routes/calendar.rs`: `CalendarTile` += `mirrors: Vec<MirrorTargetUi>`;
  `CalendarsExtras`/`CalendarsSection` += `mirror_synced: Option<Vec<String>>`;
  `has_banner()` includes it; `build_calendars_section` += `mirror:
  Option<&Arc<MirrorUiContext>>` (now 9 args, `#[allow(clippy::too_many_arguments)]`).
- `routes/mod.rs`: `pub mod mirror;` + `#[cfg(test)] pub mod test_rig;`.
- Templates (askama, `crates/frontend/public/templates/components/sections/`):
  - `calendars_section.html`: one-time results banner ("Mirror sync finished." +
    one `<li>` per line + a legend); per-tile **"Mirrors (copies on other
    servers)"** block — one line per target (`name → remote_label · kind_label ·
    last sync <relative>`, plus the last run's counts or its error) — and a
    **"Sync now"** form gated `tile.can_generate`, posting hidden `principal` +
    `calendar_id` to `/frontend/user/{user}/calendar/mirror-sync`.
  - `profile_section.html`: **NEW Timezone section** (`id="timezone-section"`,
    `open` on banner) between App tokens and Calendars — `<select>` over
    `timezone_options` with an empty "Server default" option, banner, POST
    `/{user}/timezone`.
  - `addressbooks_section.html`: **NEW per-tile "Import contacts"** block with the
    `import-contacts-form` element + plain-language copy (export vCard from the old
    service; duplicates skipped).

### 2.3 §18.43 contacts import (DONE)

- `crates/frontend/src/routes/addressbooks.rs`:
  - `ContactsImportOutcome { imported, skipped_existing, skipped_empty, failed }` +
    `summary()` (**has a grammar bug — §3.3**).
  - `ImportCandidate` enum: `Card(VcardContact)` (whole card — nothing lost:
    BDAY/ADR/PHOTO survive) | `Row(ParsedContact)`.
  - `ParsedContact` + `is_usable()`, `display_name()`, `to_vcard()` (VERSION:3.0,
    RFC 6350 escaping).
  - `csv_rows()` (RFC-4180-ish hand-rolled parser: quoted fields, `""` escape,
    commas/newlines inside quotes, CRLF/LF), `contacts_from_csv()` (keyword-based
    header matching covering Google `E-mail 1 - Value`/`Phone 1 - Value` **and**
    Outlook `E-mail Address`/`Mobile Phone`; `Type` columns excluded; ALL
    emails/phones imported), `contacts_from_vcf()` (caldata `VcardParser`),
  `stamp_uid()`, `card_from_vcard_text()`.
  - `route_post_contacts_import` — POST
    `/{user}/addressbook/{principal}/{addressbook_id}/import`, **raw request body**
    (the Lit fetch-the-File pattern, no multipart), BOM-tolerant `BEGIN:VCARD` sniff;
    duplicate handling: a `UID` already in the book is **skipped**, never
    overwritten (vcf keeps the file's UID; csv rows get fresh `imported-{uuid}`);
    `put_object` error `AlreadyExists` ⇒ counted as skip; gates:
    `user_id == user.id`, `principal == user.id || user.can_write(principal)`,
    addressbook exists; plain-text response (the dialog shows it verbatim).
  - `AddressbookTile` += `can_import` (`user.can_write`), filled by the builder.
  - **UFCS is load-bearing**: `AddressbookReadStore::get_objects(&*addr_store, …)` /
    `AddressbookWriteStore::put_object(&*addr_store, …)` — bare calls are ambiguous
    because `SqliteAddressbookStore` is also a `PrefixedCalendarStore`.
- **NEW `crates/frontend/js-components/lib/import-contacts-form.ts`** (Lit element:
dialog, file input, fetch raw body, result paragraph) — exported from `bundle.ts`;
**bundle rebuilt** via `cd crates/frontend/js-components && deno task build`
(74.27 kB → `../public/assets/js/bundle.mjs`). Rerun the build if any `.ts` changes.
- Route mounted in `lib.rs` next to the other addressbook POSTs.

### 2.4 Plain-language instructions (DONE)

- `profile_section.html`: intro paragraph 2 now walks a new user through
  app-token → "How to connect your apps" → Calendars (share links) → **Addressbooks
  (contacts import)**; the help block is retitled **"How to connect your apps…"**
  and rewritten as copy-these-three-things + per-device steps in plain words + a
  **"Moving your existing calendar or contacts in"** section (Google export steps,
  duplicate-skip promise).
- **The integration test `frontend_profile_intro.rs` pins exact strings** ("Omnical
  is where your calendars, tasks and contacts are stored", "what-is-omnical",
  "New here? What is Omnical", the localStorage script markers, and the absence of
  "This website has no calendar on it"). Paragraph 1 + the script were left
  byte-identical; only paragraph 2 and the help section changed. **Integration test
  names unchanged ⇒ the test.yml digest pin holds** (verify per §5, don't assume).

### 2.5 §18.42 store + migration-guard tests (DONE)

- `crates/store_sqlite/src/tests/mirror_store.rs`:
  `test_mirror_runs_record_last_and_prune` — record → `last_mirror_run` reads the
  newest; error strings verbatim; recording 25 runs for one target keeps the newest
  **20** (`MIN(ran_at)` asserted) and never touches another target's rows.
- `crates/store_sqlite/src/tests/principal_store.rs`:
  `set_timezone_persists_and_get_principal_carries_it` (set / get via
  `get_principal` / clear-to-empty).
- Migration guards (from the handoff, verified): `tests/migration_roundtrip.rs`
  **48 files / 22 pairs / 26 applied-ups (bare 4)**; `src/commands/upgrade.rs`
  reversible count **22**.
- `CONTENT_MUTATING_DOWNS` was **evaluated and deliberately NOT extended** with
  `20261013130000`: the list's own criteria is "destroys user-visible data **beyond
  dropping schema**" (cross-references like `add_calendar_uid`'s UIDs); the
  timezone down is a plain column drop like every other unlisted one. **Record this
  decision in §18.42 docs.**

### 2.6 Frontend TestRig + suites (WRITTEN — §3 has the known-broken parts)

- **NEW `crates/frontend/src/routes/test_rig.rs`** (`cfg(test)`): `Rig { router, db,
  cal_store, principal_store, addr_store }` — one in-memory pool, every store over
  it, `frontend_router` + `SessionManagerLayer`, a `user` principal whose app token
  `pass` authenticates via HTTP basic; helpers `insert_user`,
  `insert_calendar_with_object` (one one-shot `TESTUID1` event), `get`,
  `post_form`, `post_body`, `mirror_store()`, `router_with(mirror)` (fresh router
  over the same pool — the tests use the RETURNED router, not `rig.router`),
  `router_over` (`#[allow(clippy::unused_async)]`), `stub_caldav_server()`
  (`TcpListener` on `127.0.0.1:0`, axum `any()` route answering PROPFIND/PUT/DELETE
  over a `Mutex<HashMap<String, String>>` — **PROPFIND is static, see §3.2**),
  `target_config()`, `mirror_context()`, `urlencode()`. `#[allow(dead_code)]` on
  `request`/`get`/`post_form` (kept for future suites).
- `routes/mirror.rs` `mod tests` (5): profile mirror block render + last-sync line
  (seeds a `mirror_runs` row via sqlx); **POST sync end-to-end against the stub**
  (asserts banner stats, `mirror_objects` + `mirror_runs` rows, convergence on the
  second click); gates (other-principal ⇒ 403, unconfigured calendar ⇒ error
  banner + no run recorded); mismatched path-user ⇒ 401; **timezone section**
  (render/selected/save/garbage/clear, reading `get_principal().timezone`).
- `routes/addressbooks.rs` `mod contacts_import_tests` (3): vcf adds/skips/plains;
  Google-CSV column mapping + quoted comma + empty-row skip; gates + error phrases.
- `routes/password_reset.rs` TestRig: unchanged except the `frontend_router` call
  gained `None` for the mirror param.

## 3. KNOWN-BROKEN / IN-FLIGHT — fix these first, in this order

1. **Clippy (the interrupted loop).** Re-run
   `SQLX_OFFLINE=true cargo clippy --workspace --all-targets -- -D warnings`.
   Known remaining (sed-verified during the session):
   - `crates/frontend/src/routes/mirror.rs` ~line 176, `route_post_mirror_sync`
     (132/100 lines): **no allow attribute is present** — add
     `#[allow(clippy::too_many_lines)]` directly above
     `pub async fn route_post_mirror_sync<`.
   - Same file ~line 436, the **test doc comment** says "real local CalDAV stub" —
     `clippy::doc_markdown`: backtick it as `` `CalDAV` `` (or allow).
   - `crates/frontend/src/routes/test_rig.rs` `router_over`:
     `#[allow(clippy::unused_async)]` **is present** (sed-confirmed) — expect this
     one gone; if clippy still flags it, check the attribute sits directly above
     the fn.
   Fix whatever else falls out (this loop already fixed: doc-paragraph lengths,
   `uninlined_format_args`/`format_append` via `write!` + `std::fmt::Write`,
   `const fn` suggestions, `ignored_unit_patterns`, redundant closures
   (`PoisonError::into_inner`), `option_if_let_else` (`map_or`),
   `suspicious_operation_groupings` (filter locals in mirror.rs + a comment),
   `too_many_arguments` on `build_calendars_section`,
   `#[allow(clippy::too_many_lines)]` on `sync_target_inner` in
   `crates/caldav/src/mirror.rs`, dead-code allows in test_rig).
   Then **`cargo fmt --all` again** (edits continued after the last fmt run) and
   review the diff — the repo has pre-existing drift.
2. **`stub_caldav_server` PROPFIND is STATIC** (test_rig.rs): it always answers one
   `<D:href>/stub/listed.ics</D:href>`. The engine's unchanged-check needs the
   remote LIST to contain the written href (`TESTUID1.ics`), so with a static list
   the second sync re-PUTs forever and
   `sync_button_runs_the_engine_against_a_stub_remote`'s convergence assertion
   ("0 put, 0 deleted, 1 unchanged") **fails**. FIX: build the multistatus
   dynamically — for each key of the state map emit `<D:href>/{key}</D:href>`
   (basename == key == the filename the engine PUTs). First sync: empty list ⇒
   put=1; second: prev hash matches AND listed ⇒ unchanged=1. DELETE already
   removes from the map; PUT stores under the path (object filename).
3. **`ContactsImportOutcome::summary()` grammar bug** (addressbooks.rs): the
   singular branches currently read "Skipped 1 **one that was**…" / "Ignored 1
   **one row**…". Fix to `" Skipped {n} {that was|that were} already in this
   addressbook."` and `" Ignored {n} {row|rows} with nothing recognizable in
   {it|them}."` — the csv test asserts "Added 1 contact. Ignored 1 row with
   nothing recognizable in it." (the vcf re-import one, "…Skipped 2 that were
   already in this addressbook.", already matches).
4. **`import_gates_and_error_phrases` "neither vCard nor CSV" case is wrong**
   (addressbooks.rs tests): the body `"name,email\r\nnot,vcards\r\n"` **parses** as
   a CSV with a Name column ⇒ 200 "Added 1 contact.", not the expected 400. Change
   the body to a header-only CSV with no data rows (e.g. `"foo,bar\r\n"` or
   `"garbage"`) ⇒ `contacts_from_csv` yields zero candidates ⇒ 400 with the
   "No contacts found" phrase.
5. **Run `SQLX_OFFLINE=true cargo test -p rustical_frontend` and fix what falls
   out.** Likely: the vcf/csv tests assert substrings of caldata's **regenerated**
   vcf (`AddressObject::from(VcardContact)` → `get_vcf` → `inner.generate()`), not
   the source bytes — align assertions with the generator's real output
   (`EMAIL:jane@example.com`, `N:Doe;Jane`, …). The mirror profile test asserts
   "Mirrors (copies on other servers)", "last sync", "0 copied, 0 removed, 1
   unchanged", one "Sync now" occurrence. The timezone test asserts
   `value="America/New_York" selected` after save, and the three banners
   ("Timezone saved: …", "is not a timezone this server recognises",
   "Timezone cleared").
   Optional cleanup while in there: `route_post_timezone_save` currently detects
   success by sniffing its own banner string — switch to an explicit bool set in
   each arm.

## 4. Gates (parent plan §6) — in this order, all must pass

```
cargo fmt --all                                        # review the diff; pre-existing drift
SQLX_OFFLINE=true cargo clippy --workspace --all-targets -- -D warnings
SQLX_OFFLINE=true cargo test -p rustical --lib        # expect ~111 (120 − 9 engine tests moved to caldav); RECORD the true number
SQLX_OFFLINE=true cargo test -p rustical_caldav       # expect ~82 (73 + 9); RECORD
SQLX_OFFLINE=true cargo test -p rustical_store_sqlite
SQLX_OFFLINE=true cargo test -p rustical_frontend     # the new TestRig suites (§2.6)
SQLX_OFFLINE=true cargo test --test run_integration_tests --test migration_roundtrip --test http_integration
cargo test --workspace                                # full sweep last
```

If any `.ts` changed after the last build: `cd crates/frontend/js-components && deno task build`
(bundle.mjs is committed output).

## 5. CI pins (Omnical-Code, `~/router-dav/.github/workflows/test.yml`)

- Pins: integration baseline **122** + digest `d89c6bb0…` (see the workflow's
  `Assert the integration baseline did not shrink` step).
- No integration test names were added/removed/renamed by this effort (the new
  suites live in the frontend crate; only `get_app`'s `make_app` call changed) —
  **the pins should hold unchanged. VERIFY, don't assume**:
  `cargo test --test run_integration_tests -- --list | tail -1` == 122, and the
  sha256 of the sorted `--list` output == the pinned digest. Move the pins **only**
  if they don't match (and if they do, something unrelated broke — investigate).
- The migration-count guards (48/22/26 bare-4; reversible 22) must **pass**, not be
  moved.

## 6. Docs (parent plan §8) — append-only `PLAN_DEPLOYMENTS.md`, §18.41 style, one section per topic

- **§18.42**: the engine move to `rustical_caldav::mirror` (and why — the frontend
  must run the same engine; caldav owns `dailyview`), the `mirror_runs`
  (`20261013120000`) + `principals.timezone` (`20261013130000`) migrations with the
  count guards (48/22/26, reversible 22), the **`CONTENT_MUTATING_DOWNS` decision**
  (evaluated, NOT added — plain column drop), the mirror UI (tile "Mirrors" block +
  "Sync now"), the Timezone section + the **owner-zone** cascade (owner →
  deployment → UTC, resolved per tick/per run, never the acting admin's zone), the
  in-flight guard, the loop/CLI tz resolution changes, the true test numbers.
- **§18.43**: contacts import (`.vcf` + Google/Outlook CSV, raw-body POST,
  UID-skip duplicates, plain-text responses, the `import-contacts-form` Lit
  component + bundle rebuild) and the plain-language instructions pass.
- **Commit the rustical changes only if the user asks** (they have not); the user
  pushes both repos. The Omnical-Code pins move only if the baselines did.

## 7. Deploy + live verification (parent plan §7) — ONLY after the user confirms

- `cd ~/router-dav && scripts/build-rust.sh && ./deploy.sh`; verify
  `ssh router sha256sum /usr/sbin/rustical` == local `out/rustical`. Both migrations
  run automatically on restart.
- Live checklist: nfcalaway's `personal` tile shows both targets + a last-sync line
  (the cron's 0/0/N within seconds of a cron tick); "Sync now" converges ("0 put, 0
  deleted, N unchanged") and the same-named run appears in `mirror_runs` (cron's
  runs too); the **benign 403 WARNs** for aphy-native duplicate UIDs still appear
  and are still benign — **do not "fix"** (§1.3 of the aphy plan); timezone: set
  `America/New_York`, save, re-login shows it selected, owner zone feeds the
  phone-feed flattening (edit an event + re-sync to see local DTSTARTs); a phone
  manual sync still files Trash Mon 19:00 / Recycling Thu 19:00 on their own days;
  contacts import: try a real Google "Google CSV" + vCard export live.
- Do not touch the cron line; do not enable `[mirror]`.

## 8. Do not redo / landmines

- Parent-plan §2 (engine/config move, `mirror_objects`/`mirror_runs` store layer,
  both migrations, guards) is **done** — do not re-derive.
- `get_data_stores` returns a **13**-tuple (the parent plan said 14) — `auth_provider`
  is the 4th element; the CLI destructure names it.
- No per-target `timezone` config key — the user's own setting is the point;
  `[caldav] dailyview_timezone` stays the fallback.
- The button is **NOT** gated on `[mirror] enabled`; no mirror routes mount when no
  targets are configured (`make_app_for` builds no context ⇒ UI absent
  byte-for-byte).
- **Never log `MirrorTargetConfig`** (`Debug` carries the remote password) —
  `target.name` only.
- No bare `Extension<String>` for the timezone (taken by the subscriptions public
  URL) — it rides inside `MirrorUiContext`; the layered type is
  `Extension<Option<Arc<MirrorUiContext>>>`, and the sync handler 404s on `None`.
- UFCS in `addressbooks.rs` (`AddressbookReadStore::get_objects` /
  `AddressbookWriteStore::put_object`) and the 9-arg
  `build_calendars_section` are load-bearing — bare calls are ambiguous.
- Known limit (document, don't block): UTF-16 CSV exports are not handled (only
  UTF-8 / BOM-UTF-8); the vCard path always works.
- The `zero@novo-ordo.com` mbsync breakage is unrelated (Dovecot dots).
- §18.36–§18.41 wire shapes (X-Forwarded-For trust, EHLO-from-public_url, …) are
  settled — leave them.
- Everything in the rustical repo is **uncommitted** — commit only if the user
  asks.