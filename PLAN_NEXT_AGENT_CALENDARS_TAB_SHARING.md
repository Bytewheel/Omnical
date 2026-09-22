# PLAN_NEXT_AGENT_CALENDARS_TAB_SHARING — root-cause the CAS Share-tab report; kill the Share tab (controls move to Calendars/Addressbooks tabs); per-client subscribe instructions; CSS cleanup

Date: 2026-09-22
Repos:
- **Code**: `/home/burningserenity/router-dav/rustical` — branch `omnical-scheduling`. Last commits:
  `4c5f20c8` (password-reset + novo-ordo sender), `c2b2492c` (invites --send), `2db3986f`
  (Calendars Subscribe URL), `b89701c8` (guest banner fix). Do not commit unless the user asks.
- **Deploy scripts**: `/home/burningserenity/router-dav/scripts/{build-rust.sh,render-router-config.sh,deploy.sh}`.
- This planning repo (`~/Documents/ByteWheel/omnical`) is docs-only.

## 1. User request (what this effort delivers)

1. **Root-cause the CAS Share-tab report.** The user (logged in as
   `nicholas@carltonaudio.com`) still reports "no controls on the CAS calendar
   on the Share tab" — repeated after two deploys on 2026-09-22, so the
   previous diagnosis (banner nesting, §17.13) did not address their actual
   observation. Find out what they actually see (see §4).
2. **Remove the Share tab entirely.** All share controls move onto the
   **Calendars tab** (and addressbook share links onto the Addressbooks tab).
   The Share section/page/nav entry goes away.
3. **Per-client subscribe instructions in the UI.** For every calendar tile,
   viewable instructions for **Google Calendar, Apple Calendar, Outlook**
   (plus DAVx5/Thunderbird for the CalDAV path) — see §6 for the exact
   content matrix. This exists because the user asked: *"the share we have
   does caldav, is there no subscribe link we can use this way, does it
   always have to be credentials?"* — answer in §2, baked into the UI.
4. **Clean up the CSS** (§7): one stylesheet, no malformed/duplicated inline
   styles, nothing hidden by accident.

Out of scope: iMIP (unchanged, organizer-identity), the password-reset flow
(deployed, but its LIVE checklist was never run — fold it into §9), export
router semantics (already live; see §3).

## 2. THE ANSWER to "does it always have to be credentials?" (bake into UI)

- **Credential-less = read-only.** The `/export/{token}.ics` URL is a *live*
  feed: every GET re-queries the store (`src/export.rs:79-88`, no cache, no
  ETag — verified live 2026-09-22: the CAS feed served 4 VEVENTs matching the
  4 live DB rows, newest event updated *after* the 2026-09-10 mint). Anyone
  with the URL gets current events *when their client re-fetches*. "A view
  from a single moment in time" happens only when the user **downloads/imports**
  the .ics instead of subscribing, or when their client's poll interval is
  slow (Google: can lag many hours).
- **Read-write ("use the calendar in full") = credentials, always.** CalDAV
  writes must authenticate the actor; there is no standard credential-less
  write URL (and it would be unsafe). The minimal-credential form is the
  §17.10 guest share: username + one app token.
- **Platform matrix (hard limits — state plainly in the UI):**
  - **Apple iOS/macOS**: both paths work — subscribe (read-only) *and* CalDAV
    account (full read-write).
  - **Android (DAVx5 etc.)**: CalDAV = full access; Google Calendar app on
    Android can only consume a Google-internal calendar, so use the web
    "From URL" path (read-only) or DAVx5.
  - **Google Calendar**: **cannot add an external CalDAV server at all** —
    read-only ICS subscription only ("From URL", slow refresh). No mechanism
    exists to give a Google user full access to this server's calendar.
  - **Outlook (outlook.com / new Outlook)**: no external CalDAV — read-only
    "Subscribe from web" only. Classic Outlook desktop (Windows): read-only
    Internet Calendar subscription.
  - **Thunderbird / generic CalDAV clients**: full read-write.
- UI terminology (use consistently, replace "share link" wording):
  - "Subscribe link (read-only)" — the .ics URL.
  - "Full access (CalDAV)" — guest share credentials / own per-calendar app token.

## 3. Verified facts — do NOT re-derive (all checked live on 2026-09-22)

- **The router is ALREADY running the current code.** `/usr/sbin/rustical`
  md5 `ca277d32…` == local `out/rustical` (built 16:10:49 EDT by someone
  outside any agent session — the password-reset plan doc §7 saying "deploy
  pending" is stale). Service restarted 20:11:33 UTC with a fresh config
  ("Scheduling extension enabled (8 SMTP identities…)", `[subscriptions]`
  enabled — the export URL serves 200). The `password_resets` table EXISTS on
  the router → the deployed binary includes the password-reset feature.
- **UPX warning:** `build-rust.sh` compresses binaries with `upx --lzma`, so
  `strings` finds NOTHING in them (even known strings). Verify deployed
  features via DB migrations/behavior, never via `strings`.
- **CAS calendar state** (router DB `/usr/local/share/rustical/db.sqlite3`):
  owned by `nicholas@carltonaudio.com`, collection id is literally `CAS`
  (early displayname-style ids exist alongside UUIDs). It has: one
  subscription `o7HEGu03…` (minted 2026-09-10 20:11:22), THREE guest shares
  (admin→chris@carltonaudio.com used 2026-09-22 15:59:49, edit→
  lynscarlton@gmail.com, view→burningserenity@gmail.com), zero registration
  invites. 4 live calendar objects (newest updated 2026-09-22 15:46:43).
- **Template analysis (`share_section.html`)**: given that DB state the CAS
  tile *should* render: the export-URL block with Copy + Revoke
  (`entry.url` is Some), 3 guest rows with Revoke, and all three action forms
  (own calendar → `can_invite` true). No code-level gate explains "no
  controls". Either the observation is client-side (cache / wrong principal)
  or there is a real bug that template-reading misses — §4 decides.
- **Sessions are in-memory** → every restart logs all users out; a stale
  browser tab that predates a restart shows an old page until re-login.
- Frontend sections/tabs today: `calendars`, `addressbooks`, `groups`,
  `share`, `linked_platforms`, `profile` (`impl Section for` in
  `crates/frontend/src/routes/*.rs`; templates in
  `crates/frontend/public/templates/components/sections/`).
- Share routes registered in `crates/frontend/src/lib.rs:149-172` (GET
  `/{user}/share` + POSTs `share/create`, `share/{id}/revoke`,
  `share/invite`, `share/invite/{code}/revoke`, `share/guest-invite`,
  `share/guest-invite/{id}/revoke`). The Calendars tab already has
  `/{user}/calendar/subscribe` (§17.13) and per-calendar credentials routes
  (§17.12) in `routes/calendar.rs`.
- `share.rs` is 1189 lines: structs (`InviteLink`, `GuestShareEntry`,
  `ShareEntry`, `ShareSection`), page assembly (`shareable_principals`,
  `tile_invite_links`, `build_share_entries`, `share_page`), and the POST
  handlers (incl. `ensure_subscribe_url`, guest minting + email spawn,
  `build_guest_invite` w/ subscribe_url in `mime.rs`).
- CSS: single stylesheet `crates/frontend/public/assets/style.css` (658
  lines), embedded via `EmbedService` at `lib.rs:226` (dev fallback `ServeDir`
  at :229). Malformed inline styles exist:
  `share_section.html:131,136` `style="{ height: 2em; }"` (braces — invalid),
  `:144` `style=""`; legit CSS-var `style="--color: …"` in
  `calendars_section.html:18,109` (keep); `linked_platforms_section.html`
  uses `style="display: inline"`.
- DB quirks: `subscriptions.created_at` (not `created`);
  `calendarobjects.cal_id` (not `collection_id`); `collection_shares` has
  `privilege` TEXT (`view`/`edit`/`admin`), `used_at`, `target_email`.
- CLI: `rustical principals edit` sets a password (reads stdin when not a
  tty — `src/commands/principals.rs:prompt_password_or_read_stdio`).
  `rustical gen-config` prints the full default config (confirm TOML keys).
- The 429 rate limiter (password reset) is per-IP 5/h — in live testing,
  budget GETs accordingly.

## 4. STEP 1 — root-cause the CAS Share-tab report (before any rewrite)

Do this FIRST: if there is a real rendering bug, the migrated Calendars-tab
controls would inherit it.

**Evidence pass (no user round-trip needed yet):**
1. `ssh router 'logread | grep http-request'` — find the user's browser UA
   (Firefox per earlier logs) GETs of `/frontend/user/…/share` and `/calendar`
   with timestamps and status codes. Key questions: did any `/share` GET
   happen *after* the 20:11 UTC restart? Was it 200 or a redirect (dead
   session)? Did `/share` GETs only happen before the restart (= they were
   looking at a pre-§17.13-fix page)?
2. Local repro against a copy of the production DB (definitive — the code and
   data are both known):
   ```
   mkdir -p /tmp/opencode/share-diag
   scp router:/usr/local/share/rustical/db.sqlite3 /tmp/opencode/share-diag/
   cd ~/router-dav/rustical && cargo build           # host debug build is fine
   ./target/debug/rustical gen-config > /tmp/opencode/share-diag/config.toml
   # edit config: data_store.sqlite.db_url -> /tmp/opencode/share-diag/db.sqlite3
   #   [frontend] enabled=true + allow_password_login, [subscriptions]
   #   enabled=true (+ public_url non-empty for reset link; [scheduling] not
   #   needed to render pages)
   # diag password ON THE COPY ONLY:
   echo 'diag-password-here' | ./target/debug/rustical principals edit \
       nicholas@carltonaudio.com -c /tmp/opencode/share-diag/config.toml
   #   (check `principals edit --help` for the exact flag; also verify
   #   needs_password_change=false for this principal — the
   #   password_change_gate middleware hijacks sessions otherwise)
   ./target/debug/rustical serve -c /tmp/opencode/share-diag/config.toml &
   # curl flow: GET /frontend/login (cookie jar + csrf hidden input),
   # POST credentials + csrf, then GET
   # /frontend/user/nicholas@carltonaudio.com/share -> save the HTML and
   # inspect the CAS tile (grep for 'CAS', Copy/Revoke, guest rows, forms).
   ```
3. Compare: repro HTML has controls for CAS ⟹ server-side is fine ⟹
   client-side explanation (H1/H2/H3 below). Repro lacks them ⟹ real bug —
   bisect with tracing/otel request logs, fix it, and make the fix a separate
   small commit.

**Hypotheses, ranked (discriminating test in parens):**
- H1 Stale browser page / dead session (restart at 20:11 killed sessions; a
  pre-restart tab shows the old page). (logread: no fresh `/share` GET after
  20:11; user hard-refresh Ctrl+Shift+R + re-login fixes it.)
- H2 Tile absent, not controls absent — user shorthand "not seeing the
  controls on the CAS calendar" may mean the whole tile is missing. (Ask the
  user: does the CAS tile appear on the Share tab at all? Do OTHER calendars'
  tiles show controls? — one short message, only if evidence is ambiguous.)
- H3 Wrong principal / group impersonation: logging in as
  `nicholas@carltonaudio.com$<group>` lists only the group's collections on
  the Share tab (personal CAS absent — this literally matches "I have
  somehow lost the ability to share the CAS calendar"). Note CAS IS visible
  on their Calendars tab, so the plain login is likely — but verify.
  (Ask user which username they log in with; check login POSTs in logread.)
- H4 CSS hiding controls (unlikely; forms render as plain elements unstyled).
  (grep style.css for `.actions`/display:none; check EmbedService cache
  headers — a stale cached stylesheet cannot remove server HTML.)
- H5 Real gating bug. (The repro decides; template analysis found none.)

Record the outcome in the PLAN.md section (§8 below) — even "client-side
staleness" is worth logging, because two agents mis-diagnosed it before.

## 5. STEP 2 — remove the Share tab; move controls to Calendars/Addressbooks

**Remove** (all in `crates/frontend`):
- `Section for ShareSection` (`routes/share.rs:24`), `ShareSection` struct,
  `share_page` handler, the GET route `/{user}/share` (`lib.rs:149`),
  `templates/components/sections/share_section.html`, and the "share" nav/tab
  entry (find where sections are enumerated for the nav — start at
  `pages/user.rs` / `UserPage` and the default layout).

**Move/merge into the Calendars tab** (`routes/calendar.rs`,
`calendars_section.html`):
- Fold `build_share_entries`' calendar half into
  `render_calendars_page`: extend `CalendarTile` with `invites`,
  `guest_shares`, `can_invite`, `created_at` (it already has `subscribe_url`,
  `sub_id`, `can_subscribe` from §17.13). Preserve the gating exactly:
  collections from own principal + `can_write` groups
  (`shareable_principals`); `can_invite` = own or `is_admin(group)`;
  invites per `tile_invite_links` rules (kind=calendar, target_group logic).
- Per-tile layout (top→bottom): title/description → **Subscribe link
  (read-only)** block (URL + Copy + Revoke, or "Create subscribe link" when
  none — keep `{% else if enabled %}` semantics) → **Full access** block
  (guest share rows + Invite-guest form + Send invite / Generate invite link
  forms; the §17.12 own-credentials block stays) → **instructions**
  disclosure (§6).
- The one-time guest-credential banner: keep the §17.13 post-mortem semantics
  EXACTLY — tile-level, gated on `guest_share_principal == entry.principal &&
  guest_share_calendar_id == entry.collection_id`, renders exactly once;
  errors at page level.
- Addressbooks (`routes/addressbooks.rs`, `addressbooks_section.html`): move
  ONLY the share-link block (URL/Copy/Revoke/Create; invites + guest shares
  are calendar-only). The addressbook .vcf export feed keeps working.

**Routes:** keep the existing POST paths (`/share/create`, `/share/{id}/revoke`,
`/share/invite…`, `/share/guest-invite…`) as-is — they are internal form
actions; renaming them is churn with no user benefit. Only their **redirect
targets** change: 303 to `/{user}/calendar` (+ `#cal-<collection_id>` anchor —
add `id="cal-{{ tile.collection_id }}"` to tiles). If you prefer clean URLs,
`/{user}/calendar/share/…` is acceptable — update every form action + test in
the same commit; do not mix both styles.

**Keep unchanged:** `ensure_subscribe_url`, guest minting + one-time app
token, fire-and-forget email (`mime::build_guest_invite`,
`smtp_accounts.first()`), export router, `get_data_stores` 11-tuple.

## 6. STEP 3 — per-client instructions UI (per calendar tile)

A `<details class="client-help">` disclosure per calendar tile ("Set up on
your device…"), two sections; askama fills URLs/username. Content (write it
into the template close to verbatim):

**A. Subscribe — read-only, no account needed** (URL + Copy button; also
offer the `webcal://` variant — swap scheme on the displayed copy, keep
https for the href):
- **Apple iPhone/iPad**: Calendar app → Calendars → Add Calendar → Subscribe
  to Calendar → paste the link. (Or just open the webcal:// link.)
- **Apple Mac**: Calendar → File → New Calendar Subscription → paste; set
  "Auto-refresh" to hourly for timely updates.
- **Google Calendar**: calendar.google.com → Settings ⚙ → Add calendar →
  **From URL** → paste → Add. Caveat copy: "Google refreshes subscribed
  calendars on its own schedule — changes can take hours (sometimes a day)
  to appear. Read-only."
- **Outlook (web / new Outlook)**: Add calendar → **Subscribe from web** →
  paste + name → Subscribe. (Read-only.)
- **Outlook classic (Windows desktop)**: File → Account Settings → Internet
  Calendars → New → paste.
- **Anything else**: any webcal/ICS-subscribing app works — the link is a
  plain live ICS feed.

**B. Full access — read + write (CalDAV)** (shown only for calendars; needs a
credential — either the guest username + app token from an "Invite guest"
mint, or the user's own per-calendar app token from the credentials block):
- **Apple iOS/macOS**: Settings → Calendar → Accounts → Add Account → Other
  → **Add CalDAV Account** → server `https://0115d8cf.duckdns.org:8443`,
  user = guest username, password = the app token.
- **Android**: install **DAVx5** → add account → CalDAV → base URL, username,
  app token.
- **Thunderbird**: Calendar → New Calendar → On the Network → CalDAV URL +
  username + token.
- **Google Calendar & Outlook.com: not possible** — neither supports adding
  an external CalDAV server; use the read-only subscribe link (A) instead.
  (This must be in the UI verbatim — it pre-empts the exact question the user
  asked.)

The one-time guest-credential banner should link to the tile's instructions
details (anchor). Terminology per §2 everywhere ("Subscribe link
(read-only)" / "Full access (CalDAV)").

## 7. STEP 4 — CSS cleanup

- Inventory `style.css` (658 lines) first; keep its conventions (or improve
  consistently — do not restyle the whole portal in this effort).
- New classes for the merged tile blocks (`.subscribe-block`, `.access-block`,
  `.client-help`, `.tile-actions`, …); move ALL inline styles into the
  stylesheet: fix the malformed `style="{ height: 2em; }"` /
  `style="{ height: 2.35em; }"` / `style=""` in the (now-migrated) share
  markup, and the `style="display: inline"` forms in
  `linked_platforms_section.html`. KEEP the `--color` CSS-var inline styles
  (legit per-tile theming).
- After the merge, no template should carry layout inline styles.
- Check `EmbedService` response headers (cache-control) for `/assets/*` — if
  long-lived, append a content-hash query param to the stylesheet link in
  `layouts/default.html` so CSS changes actually reach browsers; if it
  already sends no-cache, note that and skip.
- While in there: verify no rule hides `.actions`-style blocks (§4 H4).

## 8. Tests & gates

- Migrate every test touching the Share page: `tests/integration_tests/frontend_share*.rs`
  (the "7 long-failing" ones §17.13 fixed — keep them green through this
  move), frontend unit tests, and any TestRig flows — retarget routes/redirects
  (303 → `/{user}/calendar`), the nav absence, and `/{user}/share` returning
  404.
- New coverage: Calendars tile renders subscribe block + guest rows + forms
  for own and group calendars; Addressbooks tile renders share-link block;
  instructions details present with both URL variants; banner still exactly
  once; CAS-shaped fixture (existing subscription + 3 guest shares, no
  invites) renders all controls — regression for §4.
- Gates, in order: `cargo test -p rustical_frontend`, then
  `cargo test --workspace` (all green; the known flaky-test fix
  `mint_reset`→`random_token()` is already committed), `cargo fmt --check`,
  `cargo clippy --workspace --all-targets` (no NEW warnings; pre-existing
  `share.rs:480,749` unused `auth_provider` may disappear with the rewrite —
  good), `scripts/build-rust.sh` (aarch64-musl, ≤35 MiB, UPX — remember §3:
  don't use `strings` on the product).

## 9. Deploy + LIVE checklist (production: https://0115d8cf.duckdns.org:8443)

Deploy unchanged: `cd ~/router-dav && scripts/build-rust.sh`; push binary +
`scripts/render-router-config.sh | ssh router 'cat > /etc/rustical/config.toml'`
(0600); restart; startup log must say "Scheduling extension enabled (8 SMTP
identities, RSVP links enabled)".

Verify BOTH lists:
- **This effort:** `/frontend/user/{user}/share` → 404; Calendars tab shows
  per-calendar controls (specifically the CAS tile: subscribe block + 3 guest
  rows + forms); Addressbooks tab shows share-link blocks; instructions render
  with copyable URLs; hard-refresh the user's browser (Ctrl+Shift+R) and
  confirm they now see the controls (this also closes §4 — if they STILL see
  nothing after a hard refresh on the new build, go back to §4 H2/H3 and ask
  the two questions).
- **Carry-over from PLAN_NEXT_AGENT_PASSWORD_RESET.md §8 (deployed 20:11 UTC
  2026-09-22 but never verified):** login page shows "Forgot your password?";
  reset email arrives From `burningserenity@novo-ordo.com` with the
  `/frontend/reset-password/<64-char>` link; reset rotates the password,
  clears `needs_password_change`, old link dead, fresh request supersedes;
  unknown email gets the generic page; guest-share credential email and
  `rustical invites create --email … --send` arrive From
  `burningserenity@novo-ordo.com`; an iMIP invite from an event organized by
  `burningserenity@gmail.com` still arrives From that organizer. Mind the
  per-IP 5/h reset limiter while testing.

## 10. PLAN.md logging

Add **§17.15** (next free number — §17.14 is the password-reset section) to
`PLAN.md` in this repo, mirroring §17.13's style: requests, the §2
credentials answer, the §4 root-cause OUTCOME (whatever it turns out to be),
the tab-removal design, instructions matrix, gates, and LIVE results.
Mark "deploy + live checklist pending" until §9 is actually run.

## 11. Commit conventions

Only commit when the user asks (they asked for the previous effort; assume
nothing). Suggested messages if they do:
- investigation fix (if §4 yields a standalone bug):
  `frontend: fix missing share controls on <context> (root cause: …)`
- the main effort (one commit, or split investigation/feature if cleaner):
  `frontend: fold sharing into the Calendars/Addressbooks tabs; drop the Share tab; per-client subscribe instructions; css cleanup`
Repo/branch conventions: rustical → `omnical-scheduling`; router-dav `main`
for script changes. NEVER commit the tracked build artifacts in router-dav
(`build/cargo-target/*`, `out/rustical`, root `rustical`) — they are deploy
outputs, currently stale, refreshed by the next build.

## 12. Gotchas

- The plan doc you are reading replaces PLAN_NEXT_AGENT_PASSWORD_RESET.md's
  §7 "deploy pending" — that deploy happened at 20:11 UTC 2026-09-22 outside
  any agent session. Don't re-deploy "to be safe" before building the new
  work; §9 covers it.
- Askama templates are compile-time — template changes need a rebuild (and
  askama compile errors are the usual failure mode; run `cargo check -p
  rustical_frontend` early and often).
- SQLx offline metadata: new queries must use runtime `sqlx::query` (see
  invite-store header comment); `SQLX_OFFLINE=true` is set by build-rust.sh.
- In-memory sessions: every deploy restart logs users out — tell the user to
  expect re-login.
- The export feed is deliberately GET/HEAD-only, no ETag, 404 for everything
  unknown — keep those properties.
- `webcal://` display variant: scheme-swap only, never store it in the DB
  (the token must keep working over plain https).
- Don't break the `{% else if enabled %}` "Create subscribe link" branch or
  the exactly-once banner gating — both are burn scars from §17.13.
- Dev-loop shortcut: the frontend has a dev-mode `ServeDir` for assets
  (`lib.rs:229`) — template/CSS iteration can run on the host build without
  cross-compiling; only the final gate needs build-rust.sh.

---

## 13. PROGRESS LOG — session 1 (2026-09-22, stopped mid-effort at user request)

### Done
- **§4 root-cause: CLOSED — H1 (stale browser tab / dead session), no server bug.**
  Evidence: (a) `logread` since the 20:11:33 UTC restart shows zero
  `/frontend` page GETs from the user's Firefox — only `/favicon.ico` 404s —
  so their observation predates the restart (in-memory sessions died with
  it; an open tab kept showing the old page); (b) local repro on a copy of
  the production DB (login `nicholas@carltonaudio.com`, diag password set
  on the COPY only) rendered the CAS Share tile with **every** control
  server-side: export URL + Copy + Revoke, 3 guest rows, 3 minting forms.
  H2–H5 not needed. Closure = post-deploy hard refresh (§9 first checklist
  item covers it; if they still see nothing after that, go back to H2/H3).
- **§5 tab removal + merge: implemented.** Share section/nav/GET route/§
  template deleted; `share.rs` reduced to the POST handlers (redirects now
  303 → `/{user}/calendar#cal-{id}` / `/{user}/addressbook#ab-{id}`;
  revoke routes learn the collection from the store before deleting so the
  anchor is right); calendar tiles gained the full-access block (invite
  forms, guest rows, §17.12 credentials) and addressbook tiles the
  share-link block. Banners keep §17.13 exactly-once tile gating, now on
  the Calendars page; errors page-level. Invite rows render only where
  `can_invite` (hardening: view-only group members no longer see
  registration URLs).
- **§6 per-client instructions: implemented** (`<details class="client-help">`
  per tile, both sections, both URL variants incl. Copy webcal://, the
  verbatim Google/Outlook "not possible" line, `caldav_url` prefilled).
- **§7 CSS cleanup: implemented** (new classes `.subscribe-block`,
  `.access-block`, `.client-help`, `.share-url-row`, `.block-label`,
  `.muted`, `form.inline-form`, `.error`, `.success`; all malformed inline
  styles gone incl. `style="display:inline"` in linked-platforms +
  group_detail; `--color` kept; `input[type=email]` styled globally —
  only other user is the forgot-password form; `EmbedService` now sends
  `Cache-Control: no-cache` — verified live, no hash param needed).
- **§8 gates (partial):** `cargo test -p rustical_frontend` 22 ✓;
  integration suite **95/95** ✓; `cargo test --workspace` all green;
  `cargo fmt --check` ✓; clippy clean for all touched production code.

### Remaining (resume order)
1. One clippy nit: `tests/integration_tests/frontend_share.rs:959`
   "empty line after doc comment" — the `/// ─────` separators above the
   guest-shares tests; make them plain `//` comments or drop the blank
   line. Then re-run `cargo clippy -p rustical --all-targets` (should show
   no warnings in `tests/integration_tests/frontend_*`) and
   `cargo test --test run_integration_tests`.
2. `scripts/build-rust.sh` (aarch64-musl, ≤35 MiB, UPX; don't `strings` it).
3. §9 deploy (`~/router-dav/scripts/deploy.sh` flow: build, push binary +
   `render-router-config.sh | ssh router 'cat > /etc/rustical/config.toml'`
   (0600), restart; startup log must say "Scheduling extension enabled
   (8 SMTP identities, RSVP links enabled)") + **both** LIVE checklists
   (this effort + the password-reset carry-over).
4. Mark §17.15 in PLAN.md with LIVE results (it currently says
   "deploy + live checklist pending").
5. Commits only when the user asks (§11 suggested messages).

### Working tree (uncommitted, branch `omnical-scheduling`)
Modified: `crates/frontend/{src/lib.rs, src/assets.rs, src/routes/{share,calendar,addressbooks}.rs}`,
`public/templates/{pages/{user,group_detail}.html, components/sections/{calendars,addressbooks,linked_platforms}_section.html}`,
`public/assets/style.css`; deleted: `components/sections/share_section.html`;
tests: `frontend_share.rs` (rewritten), `frontend_password.rs` (gate tests
retargeted to `/calendar`), `frontend_calendars.rs` (credential-row wording).

### Diag environment (kept for resuming)
`/tmp/opencode/share-diag/`: `config.toml` (bind 127.0.0.1:4321, prod-DB
COPY at `db.sqlite3` — diag password `diag-Passw0rd-2026-x` set on
`nicholas@carltonaudio.com` **on the copy only**, never on the router),
`jar.txt` (session cookie — re-login after restart), saved HTML
(`share.html` = pre-rewrite Share page, `cal.html`/`ab.html` = new pages,
`guest.html`/`inv.html` = banner renders). The server may still be
running on 127.0.0.1:4321; restart with
`(cd ~/router-dav/rustical && ./target/debug/rustical -c /tmp/opencode/share-diag/config.toml serve &)`
— kill with `pkill -x rustical` (NOT `pkill -f share-diag`, that matches
your own shell). Live smoke checks already passed on this instance:
subscribe-mint 303→#cal-Personal, ab-create 303→#ab-personal, guest-invite
banner exactly once + anchor, invite gen + revoke 303→#cal-CAS, `/share`
404, nav has no Share entry, `cache-control: no-cache` on assets, CAS tile
renders all controls against real production data (§4 regression).

---

## 14. PROGRESS LOG — session 2 (2026-09-22, stopped at user request mid-live-checklist)

### Done (in resume-list order)
- **Resume item 1 (clippy nit): DONE.** The `/// ─────` separators above the
  guest-shares tests (`tests/integration_tests/frontend_share.rs:957-959`)
  are now plain `//` comments. Re-ran the gates: `cargo clippy -p rustical
  --all-targets` → **zero** warnings in `tests/integration_tests/frontend_*`
  (remaining warnings are pre-existing in untouched files, e.g. `api.rs:97`);
  `cargo test --test run_integration_tests` → **95/95** ✓.
- **Resume item 2 (build): DONE.** `scripts/build-rust.sh` (aarch64-musl,
  clang recipe): rustical UPX'd to 4.9 MiB (fits the 35 MiB C2 budget),
  dav-tls 596 KiB. (Reminder: never `strings` the UPX product.)
- **Resume item 3 (deploy): DONE — 21:00:11 UTC 2026-09-22, via
  `~/router-dav/deploy.sh`** (the real one, at the repo ROOT — not
  scripts/deploy.sh, which does not exist). Safety-net first:
  hot SQLite backup to
  `~/backups/omnical/db-pre-1715-deploy-20260922.sqlite3` (3.9 MB, 0600,
  `PRAGMA integrity_check` = ok, live counts 216 cal / 209 addr — note the
  calendar live count is NOT the old 517; later sessions cleaned test
  events, this backup is the current baseline). Deploy output green:
  startup log "Scheduling extension enabled (8 SMTP identities, RSVP links
  enabled)" + "IMAP iMIP ingestion enabled (7 mailboxes)"; health ok;
  127.0.0.1:4000; dav-tls binary sha-unchanged → not restarted; overlay
  27.5 M free. In-memory sessions died with the restart (users must
  re-login).
- **§9 LIVE checklist, this effort: ALL GREEN** (public URL, curl session as
  `nicholas@carltonaudio.com`):
  - `/{user}/share` → **404**; nav lists Calendars/Addressbooks/Groups/
    Linked platforms/Profile only (no Share).
  - **CAS tile (Calendars page): 22/22 checks PASS** — `id="cal-CAS"`
    anchor, Subscribe link (read-only) block with `/export/…` URL + Copy +
    Revoke, all 3 guest rows (chris@carltonaudio.com, lynscarlton@gmail.com,
    burningserenity@gmail.com) with Revoke, invite-guest form, share/invite
    form, Full access (CalDAV) label, instructions disclosure (Apple
    iPhone/Mac, Google From URL, Outlook Subscribe from web, DAVx5,
    Thunderbird, verbatim Google/Outlook "not possible" line, webcal Copy
    button, caldav URL prefilled). Closes §4 per H1 — every control is
    server-side on the new build; the user just needs one hard refresh +
    re-login.
  - **Addressbooks tiles**: `id="ab-personal"` / `id="ab-family"` anchors;
    share-link block renders ("Share link (read-only)" + "Create share
    link" button — the no-subscription branch). Full create→revoke cycle
    proven live on `personal`: POST share/create → 303 → `#ab-personal`;
    tile then shows URL + Copy + Revoke; public export URL → **200**; POST
    share/{id}/revoke → 303 → `#ab-personal`; export URL → **404**. DB
    back at baseline (test subscription revoked).
  - **Form-call gotchas for curl testing (browser forms send these
    implicitly):** a POST with no Content-Type → **415**; with the content
    type but an empty body → **422**; the revoke form requires its hidden
    `principal` field; the share POSTs carry **no CSRF token** (session-auth
    form actions, by design — only the forgot/reset-password endpoints use
    CSRF, session-bound and rotated on every render).
  - Live session artifacts kept in `/tmp/opencode/live1715/` (jar.txt,
    fjar.txt, page HTML dumps).
- **§9 carry-over checklist (password-reset): PARTIAL — found ONE REAL BUG.**
  - ✓ Login page shows "Forgot your password?" (it did not before the
    §17.14 deploy; `reset_available()` gate satisfied).
  - ✓ POST `/frontend/forgot-password` for a real account
    (`zero@novo-ordo.com`, deliberately flagged
    `needs_password_change=1` in the router DB for the upcoming
    flag-clearing test) → 200, byte-identical no-enumeration page
    ("If an account with that email address exists…").
  - ✗ **The reset email BOUNCED.** `logread`: `could not email password
    reset link: SMTP error: SMTP error 554 after RCPT TO:<zero@novo-ordo.com>:
    5.7.1 <omnical.local>: Helo command rejected: Host not found`.
    Root cause: the hand-rolled SMTP client identifies itself with
    `EHLO omnical.local` (`crates/scheduling/src/smtp.rs:19` const);
    smtp.novo-ordo.com runs Postfix `reject_unknown_helo_hostname`.
    Gmail (the app sender before §17.14) tolerated the placeholder, so
    the sender switch surfaced it on its first live send. NOT a §17.14
    regression — the SMTP client was never compliant for strict receivers.
  - **Fix implemented on dev (uncommitted, branch `omnical-scheduling`),
    gates green, NOT yet built/deployed:**
    `smtp.rs`: `EHLO_NAME` is now a `static OnceLock<String>` +
    `pub fn set_ehlo_name_from_url(&str)` (parses scheme/userinfo/port,
    IPv6-literal-safe; RFC-5321 "localhost" fallback — always resolvable);
    both EHLO sites use it. Set at startup in `src/lib.rs` (serve path,
    only when `[subscriptions] public_url` is configured — not the
    bind-address fallback, whose host would be an IP literal) and in
    `src/commands/invites.rs` `--send` branch (the CLI is a separate
    process with its own EHLO). Gates: 4 new unit tests incl. host
    extraction (`cargo test -p rustical_scheduling smtp` 4/4 ✓),
    `cargo fmt --check` clean, clippy clean for touched code
    (fixed one pedantic `map_unwrap_or` in the new `ehlo_name()`).
    Note: two clippy warnings remain in `src/lib.rs:238` (wildcard
    single-variant) and `src/commands/invites.rs:114` (match on bool) —
    both PRE-EXISTING on those lines, not from this fix.

### Remaining (resume order — session 3)
1. `cd ~/router-dav && scripts/build-rust.sh`, then `./deploy.sh` (repo
   root). Startup-log line to expect: unchanged (the fix is silent — EHLO
   name is derived, not logged).
2. Re-run the email-dependent carry-over items (rate limiter: 5 POSTs/h
   per IP shared across forgot/reset endpoints; 1 already spent, expires
   within the hour):
   - Reset email arrives From `burningserenity@novo-ordo.com`, subject
     "Reset your Omnical password", body =
     `https://0115d8cf.duckdns.org:8443/frontend/reset-password/<64-char>`,
     1-hour expiry note. **Mailbox check:** `zero@novo-ordo.com` is an
     IMAP **folder** of the nfcalaway@novo-ordo.com account; read it with
     `timeout 120 mbsync Novo-Ordo-zero` (mbsyncrc channel;
     `CertificateFile ~/.local/share/novo-ordo.pem` handles the chain
     quirk) then read `~/.cache/zero@novo-ordo.com/`. The empty result
     in session 2 was the bounce, not a missing folder — re-verify the
     folder appears once mail actually lands.
   - Open the link → "Set a new password" form; POST a new password
     (needs the form's CSRF from a fresh GET); then: OLD frontend
     password (`pass secrets/omnical/zero@novo-ordo.com/frontend`) fails
     to login, NEW one logs in and lands normally
     (`needs_password_change` cleared — it is currently **1** on the
     router from session 2's setup). **Then restore:** set the password
     back to the pass-stored original via
     `ssh router` + `echo '<original>' | /usr/sbin/rustical principals
     edit zero@novo-ordo.com -c /etc/rustical/config.toml` (reads stdin
     headless) and verify login again — net state change zero.
   - Supersedes: POST forgot-password twice; first link → generic invalid
     page, second → form.
   - Unknown email → byte-identical "reset sent" page (compare body with
     the known-email response from session 2, saved at
     `/tmp/opencode/live1715/fp1.html` — note this file's csrf rotates).
   - Guest-share credential email From check: mint via the Calendars-tile
     guest-invite form on a LOW-IMPACT calendar (e.g. zero@novo-ordo.com's
     own `personal` — MKCOL a test calendar there first, do NOT touch the
     real CAS guest rows), verify email From + credentials, then revoke
     the share + remove the guest principal + delete the test calendar
     (full teardown per §17.10 pattern).
   - `rustical invites create --email <controlled> --send` arrives From
     `burningserenity@novo-ordo.com`; then revoke/delete the invite row
     (cleanup per §17.8 pattern).
   - iMIP invite From organizer: PUT a test event on
     burningserenity@gmail.com's `personal` (DAV app token from pass) with
     an EXTERNAL attendee that is NOT a local principal — use Gmail
     plus-addressing (e.g. `nfcalaway+imiptest@gmail.com`, delivers to
     the nfcalaway@gmail.com mailbox, not a principal) — verify the
     REQUEST email arrives From burningserenity@gmail.com; delete the
     event after (tombstone expected, live counts per the note in §17.15).
3. Mark §17.14/§17.15 in PLAN.md (session 2 already updated both Status
   blocks; just flip the §17.14 email items to done once green).
4. Commits only when the user asks. Suggested messages:
   - the EHLO fix: `scheduling/smtp: send EHLO with the public hostname (strict receivers reject unresolvable HELO names)`
   - plus the pending §17.15 main-effort commit (§11).

## 15. PROGRESS LOG — session 3 (2026-09-22, EHLO-fix deploy + FULL §17.14 email checklist GREEN)

### Resume-list outcome — every item GREEN
1. **Build + deploy: DONE.** Dev state re-verified first (`cargo test -p
   rustical_scheduling smtp` 4/4, `cargo fmt --check` clean; the diff in
   `crates/scheduling/src/smtp.rs`, `src/lib.rs`, `src/commands/invites.rs`
   matches §13 session-2's description exactly — EHLO OnceLock from the
   public URL host, "localhost" RFC-5321 fallback). Safety-net backup
   FIRST: `~/backups/omnical/db-pre-ehlo-deploy-20260922.sqlite3` (3.9 MB,
   integrity ok, live 216 cal / 209 addr — same convention as §17.15:
   `calendarobjects`/`addressobjects` WHERE `deleted_at IS NULL`).
   `scripts/build-rust.sh`: rustical 4.85 MiB UPX'd (35 MiB budget),
   dav-tls 596 KiB. `./deploy.sh` (repo root) 21:17:07 UTC: startup log
   unchanged as predicted (the fix is silent), health ok, dav-tls
   sha-unchanged not restarted, overlay 27.5 M free.
2. **Email checklist: ALL 8 items GREEN** (details in PLAN.md §17.14
   Status; short form here):
   - Reset email (zero@novo-ordo.com): From
     `burningserenity@novo-ordo.com`, subject "Reset your Omnical
     password", 64-char one-time link, 1 h expiry note. **The EHLO fix
     works** — smtp.novo-ordo.com accepted the send (DKIM/SPF/DMARC pass).
   - No-enumeration: unknown-email POST body byte-identical to
     known-email body AND to session-2's `/tmp/opencode/live1715/fp1.html`
     after csrf normalization (all three embed a rotated csrf).
   - Supersedes: second POST invalidates the first link (A → generic
     invalid page, B → "Set a new password" form). DB timeline confirms
     chaining: session-2's stale token was marked used by POST A, A by
     POST B, B at redeem; all `password_resets` rows now carry `used_at`.
   - Reset flow: POST → 303 `/frontend/login` (no auto-login); OLD
     password 401; NEW password 303 → `/frontend/user` lands normally;
     `needs_password_change` cleared 1→0; original restored via
     `rustical principals edit zero@novo-ordo.com --password` (CORRECTION
     of §13's command: the CLI has no `-c` flag — as root it reads
     /etc/rustical/config.toml by default; `--password` reads stdin);
     original 303 again, test password 401. Net state change zero.
   - Guest-share email: MKCOL `gs-test-20260922` on zero (DAV davx5
     token) → guest-invite form (POST re-renders 200 with the one-time
     banner, NOT 303) → email to `nfcalaway+gshare20260922@gmail.com`
     From `burningserenity@novo-ordo.com` with Server URL / guest
     username / one-time app token / subscribe URL. Full teardown done:
     share revoked (303 → #cal anchor), calendar DELETE 200, then
     subscription row + guest principal + guest app token + share row
     removed; DB verified back to baseline (app_tokens 67, active shares
     7, subscriptions 9, guest principals 17, invites 15). One expected
     tombstone stays in `calendars` (deleted-calendar recovery; live
     counts unchanged).
   - `rustical invites create --email nfcalaway+invite20260922@gmail.com
     --created-by live-diag-ehlo-20260922 --send`: link emailed From
     `burningserenity@novo-ordo.com`, subject "Invitation to Omnical
     calendar server"; invite row deleted (invites back to 15 = baseline).
   - iMIP REQUEST From organizer: **gotcha** — a PUT whose ICS carries
     `METHOD:REQUEST` is IGNORED (scheduler `handle_put` early-returns:
     stored objects must not carry METHOD). Re-PUT as a plain VEVENT →
     logread "scheduling: emailed REQUEST to nfcalaway+imiptest@gmail.com
     (attempt 1)", email From `burningserenity@gmail.com` (the ORGANIZER
     identity, NOT the novo-ordo app sender — §17.14's iMIP-untouched
     claim confirmed live), subject "Invitation: …". Test event deleted
     (200; CANCEL to the same plus-address expected; tombstone remains).
3. **PLAN.md §17.14 Status + §17.15 tail: updated** (this session).
4. **Commits: NOT made** (user has not asked). Still-pending suggested
   messages (unchanged from §13): the EHLO fix
   `scheduling/smtp: send EHLO with the public hostname (strict receivers reject unresolvable HELO names)`
   + the §17.15 main-effort commit per §11.

### Live-ops findings (record for future sessions)
- **zero@novo-ordo.com is an IMAP alias**, not a folder — mail lands in
  nfcalaway@novo-ordo.com's INBOX (Received/Delivered-To headers prove
  it). The `Novo-Ordo-zero` mbsync channel (`Patterns
  "zero@novo-ordo.com"`) matches NOTHING on the server — use
  `mbsync Novo-Ordo-general` and read `~/.cache/nfcalaway@novo-ordo.com/`
  INBOX. §13 session-2's "re-verify the folder appears" premise was wrong.
  (Optional cleanup: drop or repoint the dead channel in ~/.mbsyncrc.)
- **Rate limiter records BEFORE the CSRF check** — a "form expired" 400
  still consumes per-IP budget (session 3 spent exactly one such 400).
  Per-IP buckets key on client-supplied `X-Forwarded-For` (dav-tls is a
  raw TCP splice, no proxy headers added), so live diag can give each
  logical test its own bucket; the `<global>` bucket (no XFF) carried
  session-2's 2 entries — far under the 30/h cap.
- **sqlite3 CLI on the router runs with foreign_keys OFF** — deleting a
  principal does NOT cascade; orphaned rows (e.g. the guest's app_token)
  must be removed explicitly, then diffed against the pre-deploy backup.
- Artifacts for this session in `/tmp/opencode/live1715/s3/` (jar/HTML
  dumps, mbsync logs, test ICS files).
