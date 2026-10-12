# Next agent: green the CI coverage job, deploy the profile-collapse release, live test it

**Written 2026-10-11 ~23:45 local. Read this before touching anything.**

This continues one effort with three remaining checkpoints: (1) GitHub CI's
coverage job must go green (a fix is pushed; the run was **still in
progress** at handoff), (2) deploy the already-built aarch64 binaries to the
router with the new flash-free HawkBook-APK flow, (3) live test the new
portal end to end against the router.

**The owner saw a "CI failed" email — that was the OLDER run** (38178466941,
commit `9aa82200`, pushed *before* this effort's fixes). The run for our
commits is 38184346444 and had Check ✓ / Test Suite ✓ / Lints ✓ with Test
Coverage still running. Do not re-diagnose from the email; look at the run.

---

## State at handoff

| Repo | Path | Branch | Head | Pushed |
|---|---|---|---|---|
| Server | `~/router-dav/rustical` | `omnical-scheduling` | `c99ef966` | ✓ (origin `Bytewheel/Omnical-Server`) |
| Packaging/deploy | `~/router-dav` | `main` | `9b4dc39` | ✓ (origin `Bytewheel/Omnical-Code`, submodule pointer carried) |
| Plan | this repo | `main` | `e3fc0be`… | untouched (this file is uncommitted) |

Server commits:
- `347ccbea` **frontend: the tab pages collapse into the profile; add a Contacts page** — Calendars/Addressbooks/Groups/Linked-platforms are collapsible `<details>` sections of `/frontend/user/{user}` (App tokens likewise, in that vertical order), old URLs are deep links that pre-expand their section, one-time banners expand their section, nav is Profile/Calendar/Tasks/Contacts, plus a new aggregate Contacts page (list + addressbook-picker form + create). Includes `tests/http_integration.rs::test_portal_profile_sections_and_contacts` and the 2s→30s startup-budget widening.
- `c99ef966` **serve: keep the readiness permit so the coverage job's servers cannot miss it** — `notify_waiters()` → `notify_one()` at both `src/lib.rs` readiness sites (TCP + Unix arms).

Deploy-repo commit `9b4dc39` **deploy: the HawkBook APK moves off the router's flash** — see the deploy section.

**Build: DONE.** `out/rustical` = 5,525,920 B (UPX'd, size gate: "5 MiB of 35 MiB budget"), `out/dav-tls` = 1,312,528 B. Log: `/tmp/opencode/build-aarch64.log`. (The box was simultaneously building Gentoo gcc — load ~30, everything nice'd and slow; slow ≠ stuck.)

### The coverage failure: history, diagnosis, fix

- The tarpaulin **Test Coverage** job has failed on *every* CICD run since ≈2026-09-30. All other jobs were green before and after our changes.
- Diagnosis (from run 38178466941's log): `tests/http_integration.rs` — 4 of 6 server-spawning tests panicked at the old `http_integration.rs:43` readiness wait (`Elapsed`): `test_ping`, `test_initial_setup`, `test_principal_impersonation`, `test_register_disabled_unmounted`.
- Root cause: the server's readiness signal was `Notify::notify_waiters()`, which wakes only listeners already polled at the signal instant and stores **no permit**. On CI's crowded runner the spawned server thread could finish binding before the test thread polled `.notified()` — wakeup lost forever, timeout certain. `notify_one()` stores the permit; a late waiter wakes immediately. The 2s→30s budget is belt-and-suspenders for ptrace-slowed startup.
- **You cannot reproduce locally**: `cargo-tarpaulin` 0.37.5 is installed, but ptrace is blocked in this environment (`Failed to trace! Error: EPERM: Operation not permitted`). The CI run IS the instrument. Don't waste a session trying.

### First action: check the run

```sh
gh run view 38184346444 -R Bytewheel/Omnical-Server --json status,conclusion,jobs \
  --jq '{status, conclusion, jobs: [.jobs[] | {name, conclusion}]}'
```

- **Green** → skip to the Deploy section.
- **Red** → `gh run view 38184346444 -R Bytewheel/Omnical-Server --log-failed | less` and read the `http_integration` portion *before* changing anything:
  - `Elapsed` again (30s still not enough under ptrace) → the clean escape hatch is to skip the server-spawning tests **only** under the coverage job: set `RUSTICAL_SKIP_SERVER_TESTS=1` in the workflow's coverage step env, and make `test_runner`/`test_runner_with` return early (eprint + skip `inner`) when it is set. The Test Suite job (uninstrumented) keeps running them — it is green. This is preferable to fighting tarpaulin.
  - A different test/binary failing under ptrace → handle that on its own merits.
  - tarpaulin-side errors (`Unexpected signal when starting test`, trace errors) → check tarpaulin's `--timeout` (per-test no-response, default 60s) and `--post-test-delay` before blaming the tests.
- Any change: run the gates below, commit, push (method below), watch to completion. One fix per push; keep the loop short.

### Push auth (the old handoff's SSH instructions are STALE)

The `~/.ssh/agent/` sockets from NEXT_AGENT_PLAN.md no longer hold the GitHub
key (all tested: `Permission denied (publickey)`), and there is no agent in
this session. The working method, used for both pushes above:

```sh
git -c credential.helper='!gh auth git-credential' push https://github.com/Bytewheel/Omnical-Server.git omnical-scheduling
git -c credential.helper='!gh auth git-credential' push https://github.com/Bytewheel/Omnical-Code.git main
```

(`gh` is authenticated as `burningserenity`.) The owner also said they will
push if it doesn't work for you.

---

## Deploy runbook (after CI is green)

The binaries are built. `./deploy.sh` is idempotent and self-fencing: stages
via `/tmp` (tmpfs), takes a watchdog fence (`/tmp/rustical-deploy.lock`), has
an ENOSPC guard, renders the config from `pass` (SMTP secrets go straight to
the router's 0600 config — never to logs), and health-checks after start.
Overlay at handoff: **95% (4.9 MB free)** — the binary swap frees the old
~5.4 MB first; it fits.

1. `cd ~/router-dav && ./deploy.sh`
2. What is NEW this deploy (commit `9b4dc39`):
   - `hawkbook_apk = "/tmp/hawkbook.apk"` in the rendered config; the init
     script fetches the APK into tmpfs from `http://192.168.10.241:8123/hawkbook.apk`
     right before start (skips if already staged); deploy.sh stages
     `/tmp/hawkbook.apk` itself and **retires the old flash copy**
     `/usr/local/share/rustical/downloads/hawkbook.apk` (~17.6 MB back —
     overlay should drop to ~77%).
   - The static server: `~/router-dav/scripts/serve-hawkbook-apk.py`, running
     (nohup) and in this user's crontab `@reboot`. It serves ONLY
     `/hawkbook.apk` from `out/hawkbook.apk` — never serve the `out/`
     directory (`out/tls` holds the LIVE Let's Encrypt key). Verified
     byte-identical fetch from the router already.
   - DHCP reservation on the router (committed to uci): `devhost`
     (`d4:93:90:12:5e:98`) → `192.168.10.241`, so the embedded fetch address
     is stable.
3. Pre-deploy backups ALREADY taken (do not re-take; verify if paranoid):
   - `/tmp/opencode/omnical-tenant-predeploy-20261011.sqlite3` — the LIVE
     tenant DB, `PRAGMA integrity_check` = ok, 34 principals.
   - `/tmp/opencode/omnical-control-predeploy-20261011.sqlite3` — control DB.
   - **Live DB layout (tenancy enabled, `default_tenant = "omnical"`,
     `admin_host = "librecmc"`)**:
     - LIVE family data: `/usr/local/share/rustical/tenants/ff77b536-0e6f-4e43-b3a4-e7a10641ec75/db.sqlite3` (+WAL)
     - control DB: `/usr/local/share/rustical/control.sqlite3`
     - `/db.sqlite3` at the FS root is the OLD pre-tenancy DB (Sep 30) —
       not the live one; leave it alone.
4. After deploy: service status, `df -k /` (expect ~22 MB free), the live
   test below. dav-tls restarts only if its sha changed.

## Live-test runbook

Script: `/tmp/opencode/live-test-portal.sh` (MODE=old|new). A bug it had at
handoff (unbound `TOKEN` — `CURL` referenced the token before it was minted)
is **fixed** — `C()` is now defined after the mint; syntax-checked.

Mechanism, deliberately prompt-free and secret-free: create principal
`citest` via the router CLI (**no password**), mint an app token via CLI
(`principals app-token create --name livetest citest` — the token goes to a
shell variable and is never printed), then every HTTP call is Basic auth with
that token — the auth middleware accepts app-token Basic on portal routes,
and passwordless principals skip the forced-password gate (both behaviors are
pinned by the frozen suite).

- `MODE=old` dry-runs the mechanics against the PRE-deploy binary (old page
  shape: tabs, plain Groups list, no collapsible sections). It had not run
  yet at handoff — the script died on the unbound var before creating
  anything (router verified: 0 `citest`). Run it before deploying if the
  router is still on the old binary; skip it if the deploy already happened.
- `MODE=new` is the real gate, post-deploy: five collapsed sections
  (`app-tokens-section`, `calendars-section`, `addressbooks-section`,
  `groups-section`, `linked-platforms-section`), nav Tasks+Contacts, no
  Addressbooks/Groups/Linked-platforms tabs, calendars deep-link expansion,
  the full Contacts flow (CardDAV MKCOL with the app token → picker offers
  `citest|people` → aggregate create 303 → contact listed with its
  addressbook label), unauthenticated requests rejected, HawkBook download
  sha256 byte-identical to `out/hawkbook.apk`, and `citest` removed again
  (verified via `principals list`).
- Tenancy/hosts: the portal is reached as
  `https://0115d8cf.duckdns.org:8443` (the LE cert CN) via
  `--resolve 0115d8cf.duckdns.org:8443:192.168.1.21`; dav-tls listens on
  `192.168.1.21:8443` and forwards to `rustical` on `127.0.0.1:4000`. The
  default tenant answers for this host; the admin panel is on Host
  `librecmc`.
- If CLI-created `citest` were ever invisible to the web (wrong tenant DB),
  the assertions catch it immediately; the CLI uses the same config and
  default tenant, so it should not happen.

## Router space (the owner's question — answered)

Surveyed read-only. **Verdict: nothing significant is freeable** — the
overlay is small (98 MB) and everything on it is live: `/usr/local/share/`
22 MB (the 17.6 MB APK — the one real win, retired by this deploy — plus
4.3 MB live tenant data and control DB), `/usr/lib` 14 MB of in-use
DNS/TLS/sqlite shared libs, rustical+dav-tls ~6 MB, `/usr/bin` tools ~3.7 MB
(tcpdump, openssl, socat…), the named DNS stack ~2.5 MB, the live DB 3.8 MB.
`/root` holds ~100 KB of audit/rollback artifacts (`minted-tokens.tsv`,
`config.toml.pre-tenancy`, `pre-token-rotation/`) — leave them. opkg cache is
~0.4 MB at most. After the APK retirement the box sits ~77% full; further
headroom would mean removing diagnostic tools or audit artifacts — owner's
call, not yours.

## Gates before any commit/push

```sh
cd ~/router-dav/rustical
cargo fmt --all --check
cargo +1.97.0 clippy -- -D warnings            # the exact CI gate (1.97 toolchain installed)
cargo +1.97.0 test --all-features --workspace  # 921 green at handoff
cargo test --test run_integration_tests        # 114/114 — tests/integration_tests/ is FROZEN
cargo test --test http_integration            # 7, incl. the new portal test
```

Note: `cargo clean` plus a `build/cargo-target` wipe happened earlier in this
effort (~101 GB reclaimed); the first build of anything is cold. `rustup
component add --toolchain 1.97.0… clippy` was already done.

## Do not

- Do not edit `tests/integration_tests/` (the frozen 114).
- Do not touch `out/tls/`, `out/*.mobileconfig` (live credentials), the
  tenant DB, or family data. `citest` is the only principal you may
  create/remove.
- Do not print the minted app token, SMTP secrets, or the rendered config.
- Do not run tarpaulin locally expecting signal — EPERM; CI is the instrument.
- Do not delete the old pre-tenancy `/db.sqlite3` (rollback source).
- Do not re-render or re-push the router config outside `deploy.sh`.
- Do not serve the `out/` directory over HTTP (live TLS key inside); the
  one-file `serve-hawkbook-apk.py` exists precisely because of that.
- Do not force-push; do not rewrite the plan repo's history (append only).

## Suggested order

1. Check run 38184346444 (command above). Green → onward. Red → diagnose from
   `--log-failed`, one fix per push, gates first.
2. Confirm binaries (`ls -l ~/router-dav/out/rustical out/dav-tls`) — done at
   handoff: 5,525,920 B and 1,312,528 B.
3. (Only if the router still runs the old binary: `MODE=old
   /tmp/opencode/live-test-portal.sh` — mechanics dry-run; it also cleans up
   after itself.)
4. `./deploy.sh` — watch for "rustical healthy"; then `df -k /`.
5. `MODE=new /tmp/opencode/live-test-portal.sh` — the live gate; expect all
   checks ok and `citest` removed.
6. Report to the owner: CI run link (green), deploy summary, live-test pass
   counts, overlay delta (95% → ~77%), APK download verified.
7. If the owner wants the audit trail in this plan repo, append a dated
   section to the relevant PLAN file describing the deploy — ask first;
   never rewrite existing sections.
