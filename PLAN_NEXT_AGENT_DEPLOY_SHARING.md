# Plan — next agent: finish the sharing/subscribe-URL effort (deploy + verify)

Written 2026-09-22 evening. Everything below is **implemented, tested, and
cross-built**. The remaining work is **deploy + post-deploy verification**, one
small cleanup decision, and the commit question. Do not redo completed work.

## Objective (this effort)

Two user reports this session:

1. **"I have somehow lost the ability to share the CAS calendar from the
   nicholas@carltonaudio.com account."** — Root-caused (see A). The fix is in
   the tree but **not deployed**; the router still runs the stale binary from
   15:51 that predates the fixes.
2. **"We need a way to generate that subscription URL for the logged-in user,
   from their Calendars page."** — Implemented as a per-calendar "Subscribe
   URL" button + shown link on the Calendars screen (see B). Also **not
   deployed**.

## Environment / key paths

- Repo: `/home/burningserenity/router-dav/rustical` — branch `omnical-scheduling`,
  ALL WORK UNCOMMITTED.
- Deploy root: `/home/burningserenity/router-dav` — `deploy.sh`,
  `scripts/build-rust.sh`, `scripts/render-router-config.sh`, `out/`.
- Planning docs: `/home/burningserenity/Documents/ByteWheel/omnical/` (PLAN.md
  §17.7 share links, §17.10 guest shares, §17.12 calendar credentials).
- Router: SSH alias `router` (root). Binary `/usr/sbin/rustical`; config
  `/etc/rustical/config.toml`; DB `/usr/local/share/rustical/db.sqlite3`
  (**LIVE user data**); init `/etc/init.d/rustical`; watchdog
  `/usr/bin/rustical-watchdog` (cron `*/5`, fenced by `/tmp/rustical-deploy.lock`);
  `dav-tls` TLS proxy `192.168.1.21:8443 → 127.0.0.1:4000`.
  Public URL: `https://0115d8cf.duckdns.org:8443`.
- Router is aarch64 musl; binaries are UPX-packed (so `strings`/`file` output on
  the router looks empty/odd — expected, don't be fooled).

## A. Root cause of the CAS "lost share" report (diagnosed live — do not re-diagnose)

- Verified 2026-09-22 ~18:40 with a temporary app token (minted, used,
  **cleaned up** — share revoked, token removed): the Share page renders every
  tile and every form for `nicholas@carltonaudio.com`, and the guest-invite
  POST for CAS **works**. Nothing is broken account-side.
- Root cause: the **deployed** binary predates the `share_section.html` fixes.
  The one-time guest-credential banner was nested inside
  `{% for invite in entry.invites %}` — CAS has no registration invites, so
  after the user minted a CAS guest share at 18:10 (see step 3 below) the
  credential banner never rendered. App-token secrets are stored hashed and
  **cannot** be re-displayed → the credential is effectively lost.
- Fix already in tree: in `crates/frontend/public/templates/components/sections/share_section.html`
  the guest banner, the tile-level error `<p class="error">`, and the
  "Create share link" button were moved out of wrong nesting (banner+error into
  tile level; create-button now renders when `entry.url` is `None`, gated on
  `enabled`). This also fixed **7 long-failing integration tests**
  (`frontend_share` suite); full run is now **93 passed / 0 failed**.

## B. Calendars-page subscribe URL (implemented + tested)

- `crates/frontend/src/routes/calendar.rs`:
  - `CalendarTile` gains `subscribe_url`, `sub_id`, `can_subscribe`.
  - `render_calendars_page(…)` now takes `sub_store: Option<&Arc<dyn SubscriptionStore>>`
    and `base_url`, and looks up each calendar's existing subscription
    (kind=Calendar, matching collection_id) like the Share page does.
  - New `route_calendar_subscribe` → `POST /{user}/calendar/subscribe`
    (mints-or-reuses via `ensure_subscribe_url`, then redirects back to the
    Calendars page). Ownership gate: own principal or `can_write` group (same
    rule as the Share page share links). Errors render as the page's error
    banner; subscriptions disabled → 503.
- `crates/frontend/src/lib.rs`: route mounted at `/{user}/calendar/subscribe`.
- `crates/frontend/public/templates/components/sections/calendars_section.html`:
  "Subscribe URL" button in the actions row (only when
  `tile.can_subscribe && tile.subscribe_url.is_none()`), and a metadata block
  showing the export URL with **Copy** and **Revoke** (reuses the existing
  `POST /{user}/share/{sub_id}/revoke`).
- `ensure_subscribe_url` in `crates/frontend/src/routes/share.rs` is now
  `pub(super)` (shared by the share + calendar route modules).
- 2 new integration tests in `tests/integration_tests/frontend_calendars.rs`:
  `test_calendar_subscribe_mints_url_shown_on_tile`,
  `test_calendar_subscribe_rejects_foreign_and_unknown`.
- Local e2e (scratch server) verified: button → 303 → URL on tile →
  `/export/{token}.ics` serves `200`.
- Gating: everything only appears when `[subscriptions] enabled` (sub_store
  present). The router config already enables subscriptions with `public_url`.

## C. Earlier uncommitted work on this branch (context; do not redo)

- Guest share email carries the credential-less subscribe link:
  `build_guest_invite(..., subscribe_url: Option<&str>)` in
  `crates/scheduling/src/mime.rs`; `route_share_guest_invite` mints/reuses a
  subscription for the shared calendar and includes it in the email + banner
  (`ShareSection.guest_share_subscribe_url`).
- CLI `rustical guest-share add` mints/reuses the subscription and prints the
  subscribe URL (src/commands/guest_shares.rs).
- Platform invite email: `rustical invites create --send`
  (src/commands/invites.rs), `build_registration_invite` (mime.rs), and the
  untracked `scripts/invite-user.sh` (SSH wrapper; **remember `git add`**).

## D. Build/test state

- Cross-build is **fresh**: `scripts/build-rust.sh aarch64-unknown-linux-musl`
  completed; `out/rustical` (~4 MiB after UPX, fits 35 MiB budget) and
  `out/dav-tls` are current with the tree. Only rebuild if sources change.
- Baseline: `cargo test --test run_integration_tests` → **93 passed, 0 failed**.
  `cargo fmt` applied; clippy on `rustical_frontend`/`rustical` shows only
  pre-existing warnings.

## Next steps (in order)

### 1. Deploy to the router

```sh
cd /home/burningserenity/router-dav && ./deploy.sh
```

- Requires `pass` to resolve the 7 SMTP identities (render script fail-fasts
  before any service stop).
- deploy.sh stages binaries in `/tmp` (tmpfs), stops rustical, swaps the
  binary, re-pushes the rendered config + init scripts + watchdog, starts,
  runs `health`, then releases the fence `/tmp/rustical-deploy.lock`.
- `dav-tls` is only restarted if its sha changed (likely unchanged → skipped).

### 2. Post-deploy verification (exact checks)

- Health: `ssh root@router -- "/usr/sbin/rustical --config-file /etc/rustical/config.toml health"`
  (deploy.sh runs it too; also `curl -s http://127.0.0.1:4000/ping`).
- Diag pattern for logged-in pages (no passwords needed) — used earlier,
  works well:
  1. `ssh root@router -- "/usr/sbin/rustical --config-file /etc/rustical/config.toml principals app-token create --name diag-share <principal>"` → prints `{prefix}_{secret}`.
  2. Basic-auth curl on the router loopback to
     `http://127.0.0.1:4000/frontend/user/<urlencoded principal>/calendar`
     and `/share`.
  3. **ALWAYS clean up**: revoke any share minted during testing, then
     `principals app-token remove <principal> <id>` (find id via
     `principals app-token list <principal>`).
- Verify the new UI as `nicholas@carltonaudio.com`:
  - Calendars page shows **Subscribe URL** buttons; the CAS tile already shows
    its existing link
    `https://0115d8cf.duckdns.org:8443/export/o7HEGu03f2lxiPv9uUkjB1505F17b9JsPOjhtKyGMV2moJEvvaz13edgczVZhYru.ics`
    (subscription existed since 2026-09-10).
  - Mint a guest invite for CAS from the Share page → the one-time banner MUST
    appear (Server URL / Username / App token + "Prefer a subscription link …"
    line). This is the CAS fix. Revoke the diag share afterwards.
- Export feed serves through dav-tls:
  `curl -sk https://0115d8cf.duckdns.org:8443/export/o7HE….ics | head -3`
  (or loopback `http://127.0.0.1:4000/export/…`).

### 3. The orphaned 18:10 CAS share (small decision)

Share `30343a84-eaf6-4511-a63e-ef30c2b621a7`
(guest `guest-d151aece-d941-4d26-ae07-74230819fd20`, no target_email, created
2026-09-22 18:10:03) is **active but its credential was never shown or
delivered** (hashed secret, unrecoverable — this is the "lost" share that
triggered the report). Recommend revoking it and telling the user to re-mint
from the web UI:

```sh
ssh root@router -- "/usr/sbin/rustical --config-file /etc/rustical/config.toml guest-share revoke 30343a84-eaf6-4511-a63e-ef30c2b621a7"
```

Leaving it is also harmless (nobody can use it) — ask the user if unsure.

### 4. Commit question — ASK THE USER, do not commit unilaterally

Uncommitted files:

- `crates/frontend/src/routes/share.rs`
- `crates/frontend/src/routes/calendar.rs`
- `crates/frontend/src/lib.rs`
- `crates/frontend/public/templates/components/sections/share_section.html`
- `crates/frontend/public/templates/components/sections/calendars_section.html`
- `crates/scheduling/src/mime.rs`
- `src/commands/guest_shares.rs`
- `src/commands/invites.rs`
- `tests/integration_tests/frontend_calendars.rs`
- untracked: `scripts/invite-user.sh`

Suggested split (or one commit — user's call):
1. share_section template nesting fixes (banner/error/create-button) + the
   7 test fixes they unlock;
2. subscribe link in guest share email + CLI `guest-share add` printout;
3. Calendars-page Subscribe URL feature (route + template + tests);
4. platform invite `--send` + `scripts/invite-user.sh`.

### 5. Optional follow-ups

- Update the planning doc (PLAN.md §17.7/§17.10/§17.12): Calendars-screen
  subscribe button; guest email carries the subscribe link; the banner-nesting
  incident + fix.
- If the CAS symptom is reported again after deploy: check `logread -e rustical`
  and the `collection_shares` rows first — the code path is verified end to end.

## Gotchas

- Router DB is **live user data**: prefer read-only `sqlite3` queries; always
  clean up diag tokens/shares created for testing.
- busybox on the router: no `stat`, no `base64`, UPX binaries defeat `strings` —
  use `wc -c`, loopback curl, `md5sum`.
- CLI flag order: `rustical --config-file <cfg> <subcommand> …` (top-level
  option precedes the subcommand).
- URL-encode the principal in frontend paths (emails contain `@`):
  `/frontend/user/nicholas%40carltonaudio.com/...`.
- deploy.sh is order-critical (config's serde `deny_unknown_fields` — binary is
  swapped while stopped, config lands before start) and has an ENOSPC guard +
  watchdog fence; it is idempotent and battle-tested — just run it.
- After any further code change: `cargo test --test run_integration_tests`
  (93-passing baseline), `cargo fmt`, clippy on `rustical_frontend` + `rustical`,
  then `scripts/build-rust.sh aarch64-unknown-linux-musl` before deploying.
