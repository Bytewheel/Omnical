# PLAN_NEXT_AGENT_MIRROR_SYNC_UI — the §18.42 mirror sync in the web UI + per-user timezones

Date: 2026-10-13
Repos:
- **Code**: `/home/burningserenity/router-dav/rustical` — branch `omnical-scheduling`, **all changes UNCOMMITTED** (do not commit unless the user asks).
- **CI/deploy**: `/home/burningserenity/router-dav` (Omnical-Code) — untouched so far.
- Planning repo (`~/Documents/ByteWheel/omnical`) — this file; docs-only.

## 1. User request (what this effort delivers)

1. **Expose the aphy sync ability in the web UI**, in the calendars section of the
   Profile page (user wording: "the calendars/contacts/tasks in the Profile page" —
   mirrors are calendars only; no contact/task mirror exists).
2. **Per-user timezones** — "not always have everything UTC only": users set their
   IANA zone on the profile page; the mirror flattens each target's phone feed in the
   **calendar owner's** zone (owner's setting → deployment `[caldav]`
   `dailyview_timezone` → `UTC`).

## 2. State at handoff: engine + store layer DONE and compiling

`cargo check --workspace --all-targets` passed **after** every change below (the
two migration-guard edits in `tests/`+`src/commands/upgrade.rs` came after the last
full check but are test-code edits; run the gates yourself, §6).

### 2.1 Engine moved to the caldav crate (the load-bearing refactor)

- `src/mirror.rs` **deleted**; moved wholesale to
  `crates/caldav/src/mirror.rs`, adapted: `rustical_caldav::dailyview` →
  `crate::dailyview`; the module doc says why it lives in caldav (§18.42: the
  frontend must run the same engine, and the frontend cannot import the root
  crate; caldav owns `dailyview`).
- `MirrorConfig` + `MirrorTargetConfig` + their `const fn` defaults moved out of
  `src/config.rs` into the same module. `src/config.rs` re-exports them:
  `pub use rustical_caldav::mirror::{MirrorConfig, MirrorTargetConfig};` — every
  existing `crate::config::Mirror…` reference keeps resolving.
- `src/lib.rs`: `pub mod mirror;` → `pub use rustical_caldav::mirror;` —
  `crate::mirror::sync_target` keeps resolving for the CLI (`src/commands/mirror.rs`)
  and the loop (`src/tasks.rs`).
- `crates/caldav/Cargo.toml`: gained `anyhow.workspace = true` and
  `reqwest.workspace = true` (both exist as workspace deps; same reqwest that ran
  the engine from the root crate — rustls — against dav.aphy.app today).
- `crates/caldav/src/lib.rs`: `pub mod mirror;` registered.

### 2.2 Run recording (new `mirror_runs` table)

- `crates/store/src/mirror_store.rs`: new `MirrorRun { ran_at, put, deleted,
  unchanged, error: Option<String> }`; trait gained `record_mirror_run` (default
  `Err(ReadOnly)`) and `last_mirror_run` (default `Ok(None)`) — glob-exported via
  `pub use mirror_store::*`.
- `crates/store_sqlite/src/mirror_store.rs`: both implemented. Insert + prune to
  `KEEP_RUNS_PER_TARGET = 20` **in one transaction**; `last_mirror_run` reads the
  newest via free fn `run_from_row` (i64→usize with loud errors). Runtime
  `sqlx::query` throughout — no `.sqlx/` churn, per the module header.
- Migration `20261013120000_mirror_runs.{up,down}.sql` (table + index
  `idx_mirror_runs_mirror_ran_at`).
- **The engine records its own runs**: `sync_target` wraps `sync_target_inner` and
  calls `record_mirror_run` on success (stats) and failure (error string) — every
  surface (loop, cron CLI, future web button) leaves the same receipt. A recording
  failure is a `warn!`, never a failed sync.

### 2.3 Per-user timezone (new `principals.timezone` column)

- Migration `20261013130000_principal_timezone.{up,down}.sql`:
  `ALTER TABLE principals ADD COLUMN timezone TEXT NOT NULL DEFAULT '';`
  (empty string = unset; same shape as the `needs_password_change` precedent).
- `Principal` (`crates/store/src/auth/principal.rs`): `pub timezone: String` with
  `#[serde(default)]`.
- **34 files** gained `timezone: String::new(),` after their `needs_password_change:`
  line (perl bulk edit — all `Principal` literal sites; `cargo check --workspace
  --all-targets` confirms nothing was missed).
- `principal_from_row` + both SELECT lists in
  `crates/store_sqlite/src/principal_store.rs` carry `p.timezone`
  (`row.get("timezone")`). `insert_principal`'s macro INSERT is deliberately
  untouched (defaults to `''`, no `.sqlx` churn).
- `AuthenticationProvider` trait: `set_timezone` (default `Err(ReadOnly)`,
  mirroring `update_password`); `SqlitePrincipalStore::set_timezone` implemented
  as a runtime `UPDATE principals SET timezone = ? WHERE id = ?`.

### 2.4 Timezone cascade helper (in the engine module)

`rustical_caldav::mirror::resolve_timezone(owner_timezone, deployment_timezone) ->
String` — owner's zone → deployment's → `"UTC"`. **The owner, not the acting
user**: an admin syncing a group member's calendar must not re-stamp that
member's feed in the admin's zone. Doc comment on the fn says this.

### 2.5 Migration guards updated (both migrations counted)

- `tests/migration_roundtrip.rs`: 44/20/24 → **48 files / 22 pairs / 26
  applied-ups** (bare stays 4), doc comment extended.
- `src/commands/upgrade.rs`: reversible count 20 → **22**, doc comment extended.
- **Check** (not yet evaluated): whether `20261013130000`'s down (drops a
  user-set column) belongs in that module's `CONTENT_MUTATING_DOWNS` list —
  evaluate against the list's own criteria and add it if they say so.

## 3. The remaining work, in order

### 3.1 Wiring through the app layers (`src/app.rs`, `src/lib.rs`, tenancy, bundle)

1. `AppConfig` gains `mirror_targets: Vec<MirrorTargetConfig>` and
   `mirror_timezone: String` (config axis). `AppStores` gains
   `mirror_store: Arc<dyn MirrorStore>` (store axis — the bundle already holds
   it: `b.mirror_store`).
2. `make_app` (the positional shim) gains **+3 params** — `mirror_store`,
   `mirror_targets`, `mirror_timezone` — following the `print_store` precedent.
   Its callers: `cmd_serve` (`src/lib.rs:452`, `mirror_timezone` is already
   computed at `src/lib.rs:447`) and the integration suite
   (`tests/integration_tests/mod.rs` — pass
   `Arc::new(SqliteMirrorStore::new(cal_store.clone()))`, `vec![]`, `"UTC"`).
   Let the compiler find every site (`--all-targets` check).
3. `store_bundle.rs app_stores()`: populate `mirror_store`.
4. `tenancy.rs app_config_for`: fill `mirror_targets` from `config.mirror.targets`
   and `mirror_timezone` from `config.caldav.dailyview_timezone()` (UTC fallback).
   **The router (production) runs tenancy** — this path is the one that matters.
5. `make_app_for`: build the frontend `MirrorUiContext` (§3.2) and pass it to
   `frontend_router`. No targets ⇒ no context ⇒ the UI vanishes entirely
   (byte-for-byte current build when unconfigured — house rule).
6. `cmd_serve` background loop + `rustical mirror sync` CLI: resolve **per-target
   tz per run** — look up the target's owner via the principal store
   (`get_principal(&target.principal)` → `.timezone`), then
   `resolve_timezone(&owner_tz, &deployment_tz)`. The CLI's 14-tuple destructure
   (`src/commands/mirror.rs:92`) currently ignores the auth provider (4th
   element) — destructure it. The loop (`tasks::run_mirror_target`) ticks every
   `interval_secs`, so pass it the auth provider + deployment tz and resolve
   inside the tick (a user may change their zone mid-uptime).

### 3.2 Frontend wiring (`crates/frontend`)

1. `Cargo.toml`: add `rustical_caldav.workspace = true` (no cycle: caldav does
   not depend on frontend).
2. New `MirrorUiContext` (suggest `routes/mirror.rs`):
   ```rust
   pub struct MirrorUiContext {
       pub store: Arc<dyn MirrorStore>,
       pub targets: Vec<MirrorTargetConfig>,
       pub deployment_timezone: String,
       /// Per-target in-flight guard: a second click while a sync runs
       /// answers "already syncing" instead of stacking runs.
       pub in_flight: Mutex<HashSet<String>>,
   }
   ```
   **Not** separate extensions: `Extension<String>` is already taken by the
   subscriptions public URL, so the timezone must ride inside a distinct type.
3. `frontend_router` gains `mirror: Option<Arc<MirrorUiContext>>` and always
   layers `Extension<Option<Arc<MirrorUiContext>>>` (the `None` case is a 404 in
   the handler — the password-reset endpoints' pattern). Every existing
   `frontend_router` call site (password_reset.rs TestRig, share/print tests,
   integration suite) needs the new argument — the compiler finds them.
4. `ProfileDeps` gains `mirror: Option<Arc<MirrorUiContext>>` extracted exactly
   like `sub_store` (`parts.extensions.get::<Option<…>>()`).

### 3.3 Mirror UI (Calendars section of the profile)

1. `CalendarTile` gains `mirrors: Vec<MirrorTargetUi>` — per target matching
   `(target.principal, target.calendar) == (calendar.principal, calendar.id)`:
   `name`, remote label (host + last path segment of `target.url`), kind
   (canonical vs phone feed: `phone_feed_only`/`phone_instances`), and
   `last_run: Option<MirrorRun>` (one `last_mirror_run` query per target).
   `build_calendars_section` fills it from the mirror context.
2. `calendars_section.html`: a "Mirrors" block per tile that has targets — one
   line each (`name → remote · kind · last sync …`), last-sync rendered
   relative with chrono-humanize (tz-independent, the app-tokens pattern) —
   plus one **"Sync now"** button per tile, gated on `can_generate`
   (owner/admin, same as credentials). Form posts hidden `principal` +
   `calendar_id`, exactly like the subscribe/credentials forms.
3. POST route `/{user}/calendar/mirror-sync` (handler in `routes/mirror.rs`):
   - Gates: `user_id == user.id`; `form.principal == user.id ||
     user.is_admin(&form.principal)`; calendar exists (fail fast like the share
     routes); targets resolved **from the config**, never from the form.
   - In-flight: try-insert each target's name into `in_flight`; already-present ⇒
     report "already syncing" for that target, don't stack.
   - **Await** `rustical_caldav::mirror::sync_target(...)` per target
     (converged runs take seconds; the button exists for exactly this). Timezone:
     `resolve_timezone(owner.timezone, ctx.deployment_timezone)` — the OWNER's
     principal row, looked up via `auth_provider.get_principal`.
   - Re-render the profile with a one-time banner listing per-target results
     ("name: N put, N deleted, N unchanged" / "name: failed — {err}"), and set
     `CalendarsExtras::mirror_synced`-style fields so `has_banner()` expands the
     section (house rule: a banner must not sit unseen inside a fold).
   - **Not gated on `[mirror] enabled`** — the button works like the CLI, which
     also ignores that switch (it gates only the background loop).
   - No CSRF token: profile POSTs don't carry one (session SameSite=Lax + the
     auth gates — the credentials/subscribe pattern).
   - **Never log the target struct** — its `Debug` carries the remote password;
     log `target.name` only (§18.41's own lesson, already encoded in the engine).
4. Route mount in `crates/frontend/src/lib.rs` user_router, next to the other
   calendar POSTs.

### 3.4 Timezone UI (profile page)

1. New collapsible section ("Timezone", id `timezone-section`) in
   `profile_section.html`, between App tokens and Calendars: a `<select>` built
   from `chrono_tz::TZ_VARIANTS` (sorted; current value selected; an empty
   "Server default" option when unset) + save POST. Validate with
   `chrono_tz::Tz::from_str` before `auth_provider.set_timezone(&user.id, tz)`.
   chrono-tz is already a frontend dependency.
2. POST route `/{user}/timezone`: save, then re-render the profile with a
   success/error banner in that section; `ProfileSection`/`ProfileExtras` gain
   the banner + an `timezone_open` that expands on a banner.
3. The absolute "last sync" times (if rendered absolutely) use the **acting
   user's** zone; the *feed content* keeps using the owner's zone (§2.4).

### 3.5 Consumers that need nothing new (verify, don't re-derive)

- The web calendar view and the `?dailyview=1` export stay as they are
  (browser-tz / deployment config) — this effort's scope is the mirror + the
  setting.
- `[mirror] enabled = false` on the router stays intentional (tenancy skips the
  serve-path loop; the cron line runs the syncs).

## 4. Tests to write

1. **store_sqlite**: `record_mirror_run`/`last_mirror_run` round-trip + prune at
   20; `set_timezone` persists and `get_principal` carries it
   (`tests/principal_store.rs` + a mirror_store test in the same style as the
   existing store tests).
2. **frontend** (the `password_reset.rs` `TestRig` pattern — in-memory sqlite +
   `frontend_router` + session layer, oneshot requests):
   - profile GET renders the mirror block only for configured calendars, with
     last-run text (seed a `mirror_runs` row via `sqlx`);
   - **POST sync end-to-end against a local stub CalDAV server**: bind
     `tokio::net::TcpListener` on `127.0.0.1:0`, serve with a tiny axum router
     (`axum::routing::any` dispatching on method — the engine's PROPFIND/PUT/
     DELETE all answer), point the target's `url` at it, then assert the banner
     stats, the `mirror_objects` state row and the `mirror_runs` row;
   - gates: other-user POST ⇒ 401/403; a calendar with no targets ⇒ the error
     banner, no run recorded;
   - timezone POST: valid zone saves (page re-renders with it selected);
     garbage zone rejected with the banner.
3. **Baseline shifts to re-derive** (the 9 engine tests moved from the root
   crate into caldav): `cargo test -p rustical --lib` was 120 → expect ~111;
   `-p rustical_caldav` was 73 → expect ~82. Record the true numbers.

## 5. CI pins (Omnical-Code repo, `~/router-dav`)

`test.yml` pins the integration baseline (121) and a digest (`c8bb7123…`).
Adding `-p rustical_frontend` tests does not move the integration baseline, but
migration-count tests and any added integration tests do. Check the workflow
file and move the pins to match reality **before** pushing; the mirror plans
(§1.5 of `PLAN_NEXT_AGENT_APHY_MIRROR_FINISH.md`) list the full gate commands.

## 6. Gates (in the rustical repo)

```
cargo fmt --all                  # review the diff; the repo has pre-existing drift
SQLX_OFFLINE=true cargo clippy --workspace --all-targets -- -D warnings
SQLX_OFFLINE=true cargo test -p rustical --lib
SQLX_OFFLINE=true cargo test -p rustical_caldav
SQLX_OFFLINE=true cargo test -p rustical_store_sqlite
SQLX_OFFLINE=true cargo test -p rustical_frontend
SQLX_OFFLINE=true cargo test --test run_integration_tests --test migration_roundtrip --test http_integration
cargo test --workspace           # full sweep at the end
```

## 7. Deploy + live verification (production, `ssh router`)

The deployed box runs **tenancy** (single tenant `omnical`, DB
`/usr/local/share/rustical/tenants/ff77b536-0e6f-4e43-b3a4-e7a10641ec75/db.sqlite3`),
7 mirror targets (two-calendar layout: `Default` = phone feed, `omnical` =
canonical; sources in `PLAN_NEXT_AGENT_APHY_MIRROR_FINISH.md` §1.3), cron
`*/30 * * * * /usr/sbin/rustical --config-file /etc/rustical/config.toml mirror sync`.

1. `cd ~/router-dav && scripts/build-rust.sh` then `./deploy.sh`; verify
   `ssh router sha256sum /usr/sbin/rustical` == local `out/rustical`. Both
   migrations run automatically on restart.
2. Live checklist:
   - [ ] nfcalaway's `personal` tile shows its two targets + last-sync line
     (should read the cron's 0/0/N within seconds of a cron tick);
   - [ ] "Sync now" converges (banner "0 put, 0 deleted, N unchanged"), and the
     same-named run appears in `mirror_runs` (cron's runs appear too);
   - [ ] benigh 403 WARNs for aphy-native duplicate UIDs still appear and are
     still benign — **do not "fix"** (§1.3 of the aphy plan);
   - [ ] timezone section: set `America/New_York`, save, re-login shows it
     selected; the owner's zone feeds the phone-feed flattening (request a reset
     link or edit an event and re-sync to see local DTSTARTs);
   - [ ] a phone manual sync still files Trash Mon 19:00 / Recycling Thu 19:00
     on their own days (renamed `-m` UIDs are a non-event for the phone).
3. Do not touch the cron line; do not enable `[mirror]`.

## 8. Docs + commits

- `PLAN_DEPLOYMENTS.md` §18.42 (append-only, one commit per section, mirror the
  §18.41 style): the engine move to `rustical_caldav::mirror`, the runs table +
  per-user timezone migrations, the UI (tiles + timezone section), the owner-zone
  cascade, the in-flight guard, live results.
- Commit the rustical changes **only if the user asks** (they have not); the
  user pushes both repos. The Omnical-Code pins move only if the baselines did.

## 9. Do not redo / landmines

- The engine, config types, and their tests are **already moved** (§2.1) — do not
  re-derive; `cargo check --workspace --all-targets` is green on this state.
- Don't add a per-target `timezone` config key — the user's own setting wins
  (that is the point of §1.2); the deployment `[caldav]` key remains the fallback.
- Don't gate the button on `[mirror] enabled`; don't mount mirror routes when no
  targets are configured.
- Don't log `MirrorTargetConfig` (password in `Debug`); don't put the timezone in
  a bare `Extension<String>` (taken).
- The `zero@novo-ordo.com` mbsync channel being broken is unrelated to this
  effort (found in the password-reset thread: dots can't appear in Dovecot
  mailbox names).
- `X-Forwarded-For` trust, EHLO-from-public_url and all §18.36–§18.41 wire
  shapes are settled — leave them.
