# Plan — next agent: run row-22 + row-26 LIVE checklists (fix is DEPLOYED; only the live checks + teardown remain)

Session 5 (2026-09-22 ~22:08 UTC) finished ALL of step 1: gates green,
aarch64 musl rebuild, redeploy, safety-net backup, reset-link hygiene.
The deployed rustical now carries the tuple-form fix. Remaining: run the
row-22 + row-26 live checklists below (unchanged), then teardown.
Full narrative record: PLAN.md §17.8.7 item-7 "ROW-22 + ROW-26 LIVE" block.

## Step 1 DONE — record (session 5, 2026-09-22 22:0x UTC)

- Gates: `cargo test --test run_integration_tests` 95 passed / 0 failed;
  `cargo fmt --check` on linked_platforms.rs clean; clippy `-p
  rustical_frontend --all-targets` — zero warnings for the touched file
  (1 pre-existing `default_trait_access` warning in UNtouched
  oidc_user_store.rs). (`cargo test -p rustical_frontend` was already
  green, session 4.)
- Safety-net backup: `~/backups/omnical/db-pre-linkfix-deploy-20260922.sqlite3`
  (3,874,816 B, 0600, `PRAGMA integrity_check` = ok). Verified state at
  backup: live 216 calendar objects / 209 contacts (matches baseline);
  calendars 39 (36 live) / addressbooks 12 — unchanged; password_resets 5
  rows (baseline 3 + the 2 probe rows, both captured).
- Build: `scripts/build-rust.sh` (aarch64 musl, clang+zig-includes
  recipe), UPX'd: out/rustical 4.9M packed (fits 35 MiB budget);
  out/rustical mtime verified NEWER than the fixed source (fresh build).
- Deploy: `deploy.sh` 22:08 UTC. dav-tls sha unchanged → no restart;
  rustical swapped + restarted, NEW PID 17364 (health ok, listening
  127.0.0.1:4000, scheduling ext + IMAP ingest lines present in logread,
  overlay 73%). Deployed binary = POST-FIX.
- Reset-link hygiene (§17.14): the UNUSED probe row
  `5f94e33e-5724-44ef-960c-c2e53db2763a` (would have expired 22:46:52Z)
  was marked used_at=2026-09-22T22:08:50Z via sqlite3; the other probe row
  was already used. NO usable reset links remain. password_resets now:
  5 rows, all used.
- Restart side effects (expected): the kept portal session `jar.txt` is
  DEAD — re-login before the checklists (portal password: `pass secrets/
  omnical/live-test-20260914@example.com/portal`). In-memory rate buckets
  reset. Diag app token `diag-row22` survives (DB-backed) — `diag_token.txt`
  still valid.

## Task

Verification-matrix rows 22 (linked-platform real-URL import) + 26 (forced
password-change gate), LIVE on the router, using the kept
`live-test-20260914@example.com` account — then teardown to baseline.

## THE BUG IS ROOT-CAUSED (was: half-diagnosed)

**Root cause — a std behavior change, nothing to do with musl/router/UPX/
threads/process state:** current std (rustc 1.98.1) routes bare-hostname
`"example.com".to_socket_addrs()` through `impl TryFrom<&str> for LookupHost`
(`std/src/sys_common/net.rs`), which now REQUIRES a `host:port` string:

```rust
let (host, port_str) = try_opt!(s.rsplit_once(':'), "invalid socket address");
```

No colon → instant `io::Error { kind: InvalidInput, message: "invalid socket
address" }` — getaddrinfo is NEVER called (hence zero DNS packets, instant
fail, every hostname, fresh + long-running process identical). The SMTP/IMAP
paths always worked because tokio's `TcpStream::connect((host, port))` and the
probes use the *tuple* `(&str, u16)` impl, which does call getaddrinfo. The
route code (added in §17.8.3 under an older std where bare-host resolved with
port None) was the only bare-host call in the workspace (grep-verified: the
two sites in linked_platforms.rs).

**Evidence chain (session 4):**
- Reproduced with a *verified-running* tcpdump: still zero DNS packets on a
  failing add. **Void-evidence discovery: busybox on this router has NO
  `setsid`** — every `setsid <cmd> … &` silently never ran ("setsid: not
  found" went to the redirected log). ALL prior backgrounded captures
  (previous session's + session 4's first two) are void artifacts; do not
  cite them. Pattern that works: run everything inside ONE held-open ssh
  script (see `scratch-test.sh` / `diag-run.sh`), or foreground ssh.
- **SMTP path re-probe: WORKS.** Two `POST /frontend/forgot-password`
  (zero@novo-ordo.com, fresh CSRF each, XFF-isolated rate bucket) at
  21:45:14 + 21:46:52 — both emails DELIVERED (mbsync Novo-Ordo-general
  INBOX). The serve process resolves + sends fine; "SMTP broken too" was a
  void-capture artifact. Side effects: 2 reset rows in prod DB (first
  superseded→used; second UNUSED, expires by itself ~22:46:52 UTC — mark
  used via sqlite3 if the next session starts sooner, per §17.14's
  "no usable links remain" convention); 2 audit emails in the INBOX.
- **Fresh instance of the exact deployed binary: FAILS IDENTICALLY** — not
  long-running state. Scratch serve (:4001, /tmp/rustical-row2226.toml,
  fresh scratch DB) + CLI invite mint (`--config-file` global flag works)
  + POST /register (provisions personal/tasks + auto-logs-in) → add POST
  → same instant banner. Scratch instance + IMAP ingest worked in the same
  process (ingest connected + parsed mail ~2 s after boot), proving
  process-wide DNS was fine.
- **Old probes were flawed but re-verified:** `rt.block_on(async …)` runs on
  the MAIN thread — the "worker" test never tested a worker. Corrected
  probe (`/tmp/opencode/row2226/dnsprobe3/`): true `tokio::spawn`ed worker
  tasks at default/2 MiB/64 KiB/8 MiB worker stacks + spawn_blocking +
  UPX-packed variant — ALL PASS on the router. Worker context, stack size,
  and UPX are now properly ruled out (not just by the old probes).
- **Real error obtained** by patching the two `.map_err(|_| …)` to
  `inspect_err(warn!(%host, error = ?e, …))`, rebuilding (same
  clang+zig-musl-includes+rust-lld recipe, UPX'd for fidelity), scratch-run:
  `ssrf_guard resolve failed host=example.com error=Error { kind:
  InvalidInput, message: "invalid socket address" }` (also for
  0115d8cf.duckdns.org) → led straight to the std pre-check.
- Why offline gating missed it: `tests/integration_tests/
  frontend_linked_platforms.rs` header says the domain-fetch happy path
  stays for the live gate; its tests use literal IPs (guard refusals) or
  seed sources directly. No offline test exercises a domain resolve.

## The fix (applied + DEPLOYED; still UNCOMMITTED in ~/router-dav/rustical)

`crates/frontend/src/routes/linked_platforms.rs`, both call sites:
`host.to_socket_addrs()` → `(host, 0u16).to_socket_addrs()` (ssrf_guard) /
`(host.as_str(), 0u16).to_socket_addrs()` (fetch_and_parse), with an
explanatory comment; the `inspect_err` warn! logging of the real io::Error is
KEPT. Port 0 is correct: only the IPs are consumed (ssrf_guard maps
`sa.ip()`), and reqwest 0.12.28 `resolve_to_addrs` docs: "Ports in the URL
itself will always be used instead of the port in the overridden addr."

Gates run so far: `cargo test -p rustical_frontend` GREEN (all unit suites,
incl. linked_platforms). Session 5 finished the rest: `cargo test --test
run_integration_tests` 95 passed; fmt clean; clippy clean for the touched
file (see session-5 record above for details).

## Remaining steps (in order)

1. ~~Finish gates + build + redeploy~~ DONE (session 5, see record above).
2. Row 22 checklist (below) — unchanged. REMEMBER: fresh portal login first
   (old `jar.txt` session died in the restart).
3. Row 26 checklist (below) — unchanged.
4. Teardown + PLAN.md updates + cleanup (below).

## Environment

- Router: `ssh router` (root). rustical PID 17364 (started 2026-09-22
  22:08:39 UTC, deployed binary = POST-FIX, incl. §17.14/§17.15 + EHLO fix).
  Prod config `/etc/rustical/config.toml`; DB
  `/usr/local/share/rustical/db.sqlite3` (LIVE data). Public URL
  `https://0115d8cf.duckdns.org:8443` (loopback diag:
  `curl --resolve 0115d8cf.duckdns.org:8443:192.168.1.21`; from the router
  itself, prod rustical is reachable plain at `127.0.0.1:4000`).
- Safety-net backups: `~/backups/omnical/db-pre-row2226-20260922.sqlite3`
  (pre-bug-work) + `~/backups/omnical/db-pre-linkfix-deploy-20260922.sqlite3`
  (pre-deploy, includes the 2 reset rows — the authoritative pre-row-22 state).
- Portal password: `pass secrets/omnical/live-test-20260914@example.com/portal`.
- Dev artifacts: `/tmp/opencode/row2226/` — `jar.txt` (portal session cookie
  from session 4 — DEAD since the session-5 deploy restart; keep only as a
  filename reference for the new login), `diag_token.txt` (diag
  DAV app token, prints `{prefix}_{secret}`), `srcfeed.ics`, `lp0.html`,
  `add1.html` (failing add response), `repro1.html` (repro), `fp*.html/jar`
  (forgot-password probes), `scratch-test.sh` + `diag-run.sh` (the working
  single-ssh-session pattern — reuse for any router-side scripted probes),
  `dnsprobe3/` (corrected probe source; binary + UPX variant on router),
  old `dnsprobe*/` (flawed "worker" tests — keep only as a cautionary note).
- Router tmpfs leftovers (REMOVE AT TEARDOWN): `/tmp/dnsprobe`,
  `/tmp/dnsprobe2`, `/tmp/dnsprobe2u`, `/tmp/dnsprobe3`, `/tmp/dnsprobe3u`,
  `/tmp/rustical-diag` (UPX'd diagnostic binary, PRE-fix + warn logging) +
  `/tmp/rustical-diag.log`, `/tmp/row22t/` + `/tmp/row22d/` (scratch/diag
  run artifacts incl. pcaps), `/tmp/rustical-row2226.toml` (scratch config,
  :4001 + scratch DB), `/tmp/rustical-row2226-db.sqlite3` (scratch DB, has
  `row22probe@example.com`), `/tmp/rustical-row2226.{log,pid}` (log now
  holds scratch-run output; pid stale), stale pcap/pid/log files
  (`/tmp/dnsadd.pcap`, `/tmp/dns-smtp-probe.pcap`, `/tmp/probe2.*`,
  `/tmp/tcpdump*.pid`).
- Prod DB deltas vs the §17.15 baseline (audit + expected): +2
  password_resets rows for zero@novo-ordo.com (21:45:14 used at 21:46:52;
  21:46:52 row marked used 22:08:50Z at session-5 start — NO usable links
  remain). Nothing else changed; no source rows, no calendars touched.
  Rustical restarted once (session-5 deploy).
- `logread` note: the ring buffer is tiny and flooded by dropbear lines
  (only ~3 rustical lines survive minutes later) — start `logread -f` to a
  file BEFORE any probe that needs log evidence.

## Discovered state (do not re-discover)

The live-test account carries **unrecorded 2026-09-14 19:17 prep for exactly
this test**: calendar `srcfeed` (3 VEVENTs `e1`/`e2`/`e3`), empty targets
`imported` + `imported2`, and a subscription `ee496590-829e-47b4-a0d9-7cf2aec3cf18`
(token `ZfqvG7VboY9EI2PxF66CxPqguX9NEh2ObkOXKtLSBM9uEE7powJ4SgtkGgabeGKL`,
kind calendar, collection `srcfeed`). Verified green: that export URL serves
200 through dav-tls (`ssl_verify_result=0`, the 3 events); portal login 303;
linked-platforms page 200 with `imported`/`imported2` targets. An active
diag app token exists: name `diag-row22`, id
`bbdef0d9-4b2a-4279-92da-13c63025dfef` (DAV Basic auth uses the value in
`diag_token.txt`). Teardown must return the account to its recorded
row-20/21 shape (personal + tasks calendars only, 2 registration subs,
5 registration tokens) — global counts go 9→8 subs, 68→67 tokens.

## Row 22 checklist (after redeploy)

1. Add `https://0115d8cf.duckdns.org:8443/export/ZfqvG7….ics` → `imported`
   → 303; `calendar_sources` gains 1 row; `imported` has exactly the 3
   events (sqlite3 count + calendar-query REPORT through dav-tls).
2. Provider edit propagates: PUT edited `e2` (new SUMMARY) into `srcfeed`
   via DAV with the diag token → portal Refresh on the source → `imported`
   copy updated (same UID/href).
3. Add/delete propagate: PUT new `e4` into `srcfeed` → Refresh → appears;
   DELETE `e4` → Refresh → gone (tombstone expected).
4. Mass-delete abort: replace srcfeed content so a Refresh would delete >50 %
   (e.g. delete 2 of 3 in `srcfeed`, keep the copy at 3, then... careful: the
   heuristic compares against the *materialized* calendar; deleting 2 of 3
   source-side = 2/3 > 50 % of imported's rows → Refresh must abort with
   the banner and leave rows intact).
5. SSRF negatives (each must store NO calendar_sources row): `http://` URL,
   `https://192.168.1.1/x.ics`, `https://127.0.0.1/x.ics`, `https://[::1]/x.ics`.
   (Size cap: offline-covered; live >25 MB feed not practical — skip.)
6. Remove keeps copy: portal Remove → mapping gone, `imported` still has
   its events.
7. Teardown: remove source (done in 6); DELETE `imported`, `imported2`,
   `srcfeed` calendars (DAV, header `X-No-Trashbin: 1`, expect 200);
   revoke subscription `ee496590…` (portal form path with the form's
   `principal` field — empty body 415 / field-less 422 gotcha — or CLI
   `subscriptions remove live-test-20260914@example.com <id>`);
   remove diag token (CLI `principals app-token remove live-test-20260914@example.com bbdef0d9-4b2a-4279-92da-13c63025dfef`).
   Verify: account = personal+tasks calendars, 2 registration subs, 5
   registration tokens; global subs 8, tokens 67; live cal/addr counts
   back to 216/209 modulo expected tombstones (deleted-calendar recovery
   keeps soft-deleted rows — live counts `deleted_at IS NULL` are the truth).
   Also remove ALL router /tmp leftovers listed above.

## Row 26 checklist (same session, after row 22)

1. `ssh router` sqlite3: `UPDATE principals SET needs_password_change=1
   WHERE id='live-test-20260914@example.com';` (flag flip only).
2. Fresh login → GET any portal page (user home, linked-platforms) → 303
   to `/frontend/user/{u}/password`; the password page itself renders 200.
   Note the session in `jar.txt` predates the flip — use a FRESH login.
3. Negatives (POST `/frontend/user/{u}/password`): wrong current password
   → "Current password is incorrect."; short new (< `min_password_length`)
   → length error; mismatched confirm → error; each re-renders the form,
   flag stays 1.
4. Rotation: correct current + valid new (≥12 chars) → 303 to user page;
   flag cleared (sqlite3); OLD password fails login, NEW logs in.
5. Restore: set the password back to the pass-stored original via
   `echo '<original>' | ssh router /usr/sbin/rustical principals edit
   live-test-20260914@example.com --password` (CLI reads stdin; as root it
   reads /etc/rustical/config.toml by default — no `-c` flag, §17.14
   finding); verify the original logs in again; flag stays 0.
6. Passwordless-never-gated: offline-covered — no live check needed.

## When both rows are green

Update PLAN.md: matrix rows 22 + 26 cells (LIVE: DONE + method/results),
§17.8.7 record (amend the session-4 root-cause note with deploy + results);
note the 2026-09-14 prep teardown + final counts. Remaining item-6 live
work after that: phone-based registration (user's device — ask).
