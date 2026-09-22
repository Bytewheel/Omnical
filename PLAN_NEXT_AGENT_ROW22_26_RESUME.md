# Plan — next agent: run the row-26 LIVE checklist (row 22 is DONE + torn down; only row 26 + final PLAN.md updates remain)

**COMPLETED 2026-09-22 ~22:25 UTC (session 7): the row-26 LIVE checklist is
ALL GREEN** and the PLAN.md row-26 matrix cell + §17.8.7 pointer are updated
(see the Step 3 record below). Nothing remains of this file's scope. The only
remaining item-6 live work anywhere is the phone-based registration (user's
device — ask the user).

Session 5 (2026-09-22 ~22:08 UTC) finished step 1 (gates, rebuild, redeploy,
backup, reset-link hygiene). Session 6 (2026-09-22 ~22:10–22:20 UTC) ran
the FULL row-22 live checklist on the post-fix deploy — ALL GREEN — and
completed the row-22 teardown back to the row-20/21 account shape (record
below). Remaining: the row-26 checklist, then the final PLAN.md updates.
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

## Step 2 DONE — record (session 6, 2026-09-22 ~22:10–22:20 UTC): ROW-22 LIVE, ALL GREEN + teardown

Pre-flight state matched the recorded baseline exactly (sources 0; account
subs 3 / tokens 6 incl. diag-row22; global subs 9 / tokens 68 / sources 0;
live 216 cal objects / 209 contacts / 36 calendars / 12 addressbooks;
rustical PID 17364 = the session-5 post-fix deploy, still listening
127.0.0.1:4000). Fresh portal login (pass-stored password) → 303, session
saved to `jar.txt`; export URL via dav-tls → 200, `ssl_verify_result=0`,
3 VEVENTs. DAV collection paths confirmed live:
`/caldav/principal/{user%40domain}/{cal}/`, object hrefs `{uid}.ics`, DAV
Basic auth = principal-id + app-token value.

- **(1) Add:** POST add srcfeed export URL → `imported` → **303 in 0.54 s**
  — the tuple-form fix works live on the real public URL (server-side fetch
  through public DNS + NAT hairpin to dav-tls). `calendar_sources` +1 row
  (id `5ee3a3e3-36b8-4699-88ff-0c69bf00c872`, provider_host
  `0115d8cf.duckdns.org`, last_fetch_at stamped, success 1); imported =
  exactly e1/e2/e3 live; calendar-query REPORT through dav-tls → 207 with
  exactly 3 hrefs.
- **(2) Edit propagation:** PUT edited `e2` (SUMMARY "Feed event two
  EDITED", same UID) into srcfeed → 201; portal Refresh → 303 (1.5 s);
  imported `e2` updated, same href `e2.ics`; e1/e3 untouched.
- **(3) Add/delete propagation:** PUT new `e4` → 201 → Refresh → e4.ics
  appears in imported (4 hrefs); DELETE `e4` → 200 → Refresh → gone (back
  to 3). Tombstone nuance: the DAV DELETE tombstoned srcfeed/e4
  (`deleted_at` set); the refresh's copy-removal hard-deletes imported/e4.
- **(4) Mass-delete abort:** DELETE e1+e2 from srcfeed → Refresh re-renders
  **200 with banner "Refresh failed: mass-delete aborted."**; imported
  e1/e2/e3 all still live (rows intact).
- **(5) SSRF negatives** (each 200 re-render, banner, NO calendar_sources
  row; count stayed exactly 1): `http://…` → "Only HTTPS URLs are
  allowed"; `https://192.168.1.1/x.ics`, `https://127.0.0.1/x.ics`,
  `https://[::1]/x.ics` → "Refused: private-range address".
- **(6) Remove keeps copy:** portal Remove → 303; sources 0; imported still
  e1/e2/e3 live.
- **(7) Teardown (all green):** DELETE `imported`/`imported2`/`srcfeed`
  with `X-No-Trashbin: 1` → 200 ×3 (hard delete; PROPFIND → 404). The
  srcfeed subscription `ee496590…` was NOT cascade-deleted by the calendar
  DELETE — revoked via CLI (`subscriptions remove`), which printed
  "Subscription ee496590… removed". Diag token removed via CLI
  (`principals app-token remove … bbdef0d9-…`) → DAV auth now 401; the
  srcfeed export URL now 404. Account back to the row-20/21 shape exactly:
  personal+tasks calendars (live), 2 registration subs, 5 registration
  tokens. Global: **subs 8, tokens 67, sources 0** (the 9→8 / 68→67 the
  prep-state paragraph predicted). Live counts now **210 cal objects / 209
  contacts / 33 calendars / 12 addressbooks** (216−3 prep events and 36−3
  prep calendars went with the deleted prep calendars — the 216/209/36
  baselines INCLUDED the 2026-09-14 prep; 213→210 after the post-run
  tombstone sweep removed the 3 orphaned live `_vdirtest` objects —
  see below). Router /tmp fully cleaned
  (all dnsprobe*, rustical-diag, row22t/d, rustical-row2226* incl. the
  sqlite `-wal`/`-shm` sidecars, pcap/pid leftovers); rustical PID 17364
  never restarted; no scratch principal in prod.

## Step 3 DONE — record (session 7, 2026-09-22 ~22:23–22:26 UTC): ROW-26 LIVE, ALL GREEN

Safety net FIRST: `~/backups/omnical/db-pre-row26-20260922.sqlite3`
(3,878,916 B, 0600, `PRAGMA integrity_check` = ok); pre-flight matched the
recorded baseline exactly (flag 0; 8 subs / 67 tokens / 0 sources;
33/210/209/12/11 live; changelog 916; resets 5/5 used). rustical PID 17364
+ dav-tls PID 3765 both untouched for the whole run. All HTTP through
dav-tls on the public URL (`--resolve` hairpin, `ssl_verify_result=0`);
logread -f captured to a file before the probes (survives ssh exit when
backgrounded with redirects — the earlier `setsid` problem doesn't apply to
plain `cmd &`).

- **(1) Flag flip:** sqlite3 `UPDATE principals SET needs_password_change=1`
  → 1 row, verified = 1.
- **(2) Gate:** FRESH login POST (pass-stored password) → 303; GET
  `/frontend/user` → 303 → `/frontend/user/{u}/password`; GET
  `/frontend/user/{u}/linked-platforms` → 303 → same; the password page
  itself → 200, form rendered with `minlength="12"` (deployed
  `min_password_length`).
- **(3) Negatives** (each 200 re-render, form present, flag stayed 1 after
  all three): wrong current → "Current password is incorrect."; short new
  ("short") → "Password must be at least 12 characters."; mismatched
  confirm → "The new passwords do not match."
- **(4) Rotation:** correct current + 21-char new → **303 to the user
  page**; flag cleared (sqlite3 = 0); the SAME session's user page → 200
  (gate lifted — note GET `/frontend/user` 303s canonically to
  `/frontend/user/{u}` even when ungated); OLD password login → **401**
  (WARN "Failed password login attempt" in the logread capture); NEW
  password login → 303.
- **(5) Restore:** `cat pw | ssh router /usr/sbin/rustical principals edit
  live-test-20260914@example.com --password` → "Principal … updated";
  original password logs in again (303); flag stays 0.
- **(6) Passwordless-never-gated:** offline-covered, no live check (gate
  condition `password.is_some()`), per the checklist.

Evidence: logread capture = INFO "password changed (forced after first
calendar join)" + the login WARN, ZERO ERROR lines (pulled to
`/tmp/opencode/row2226/row26/row26-logread.log`, chmod 600 — it contains
cleartext passwords, see finding a). Post-state = baseline exactly (flag 0;
8 subs / 67 tokens / 0 sources; 33/210/209/12/11 live; changelog 916;
resets 5/5 used; portal sessions are process-memory only — no sessions
table, logins leave no DB rows). Router /tmp cleaned (backup copy + logread
log removed; logread stopped — busybox has no `pkill`, used `kill <pid>`).
Dev artifacts: `/tmp/opencode/row2226/row26/` (item2/3/4.sh, jar26.txt —
portal session valid until the next rustical restart, HTML/XML responses,
logread log). pw files removed locally (original lives in pass).

**Findings for the record (also in the row-26 cell):**
(a) `route_post_password_change`'s `#[instrument]` span logs the whole
`ChangePasswordForm` incl. cleartext passwords to syslog — same pre-existing
upstream pattern as the row-20 register finding (hardening list);
(b) transient `[rustical-watchdog]` cron zombie (5-min cycle) appeared and
self-reaped during the run — benign.

## Step 4 DONE — record: final PLAN.md updates applied (session 7)

Row-26 matrix cell → **LIVE: DONE 2026-09-22** with method/results; §17.8.7
"Row 26 remains" pointer → replaced with the LIVE DONE note (remaining
item-6 live work: phone-based registration only).

## Task (original, for the record — completed)

Verification-matrix row 26 (forced password-change gate), LIVE on the
router, using the kept `live-test-20260914@example.com` account (row 22 is
done + torn down to its row-20/21 shape).

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

1. ~~Finish gates + build + redeploy~~ DONE (session 5).
2. ~~Row 22 checklist~~ DONE (session 6, see record above — incl. its
   item-7 teardown + router-/tmp cleanup).
3. ~~Row 26 checklist~~ DONE (session 7 — see the Step 3 record; FRESH
   login used after the flag flip as instructed).
4. ~~Final PLAN.md updates~~ DONE (session 7 — see the Step 4 record).

## Environment

- Router: `ssh router` (root). rustical PID 17364 (started 2026-09-22
  22:08:39 UTC, deployed binary = POST-FIX, incl. §17.14/§17.15 + EHLO fix
  + the tuple-form linked-platforms fix; never restarted since). Prod
  config `/etc/rustical/config.toml`; DB
  `/usr/local/share/rustical/db.sqlite3` (LIVE data). Public URL
  `https://0115d8cf.duckdns.org:8443` (loopback diag:
  `curl --resolve 0115d8cf.duckdns.org:8443:192.168.1.21`; from the router
  itself, prod rustical is reachable plain at `127.0.0.1:4000`).
- Post-row-22 baseline (for counting): global subs **8**, app tokens
  **67**, calendar_sources **0**; live counts **210** calendar objects /
  **209** contacts / **33** calendars / **12** addressbooks / **11**
  birthday calendars — and **ZERO tombstones anywhere** after the
  post-row-22 sweep (next paragraph); totals now equal live counts
  (33 / 210 / 209 / 11), `calendarobjectchangelog` 916.

## Tombstone sweep DONE (post-row-22, 2026-09-22 ~22:35 UTC, user-directed)

Every pre-existing soft-deleted row removed from the prod DB via
EXPLICIT per-row DELETEs (one statement per row key, generated with
`quote()`, dry-run ROLLBACK pass verified before the COMMIT pass — no
blanket conditions). Removed: 3 calendar tombstones
(`nfcarlton@gmail.com/_vdirtest` + `/test-probe`,
`zero@novo-ordo.com/gs-test-20260922`), the 3 still-live-but-orphaned
objects + 3 changelog rows under `_vdirtest` (the app's FK-cascade
equivalent — sqlite3 CLI has FKs off), 336 tombstoned `calendarobjects`,
3 tombstoned `addressobjects`, 1 tombstoned `birthday_calendars` row.
Changelog entries for LIVE collections were KEPT (919→916 = only the
`_vdirtest` three): `_sync_changes` reads the changelog, not tombstone
rows — deletions still reach sync-clients. Post-state verified: 0
tombstones in calendars/calendarobjects/addressobjects/addressbooks/
birthday_calendars; `integrity_check` ok; rustical PID 17364 never
restarted; portal + dav-tls 200s after. Safety net:
`~/backups/omnical/db-pre-tombstone-cleanup-20260922.sqlite3`
(pre-state 3/336/3/1, integrity ok). For counting purposes the "raw"
totals in older records (e.g. 36 calendars / 549 calendarobjects) are
obsolete — use 33 / 210 / 209 / 11.
- Safety-net backups: `~/backups/omnical/db-pre-row2226-20260922.sqlite3`
  (pre-bug-work) + `~/backups/omnical/db-pre-linkfix-deploy-20260922.sqlite3`
  (pre-deploy, includes the 2 reset rows — the authoritative pre-row-22 state).
- Portal password: `pass secrets/omnical/live-test-20260914@example.com/portal`.
- Dev artifacts: `/tmp/opencode/row2226/` — `jar.txt` (LIVE portal session
  cookie from session 6, valid until the next rustical restart; row 26
  wants a FRESH login after the flag flip anyway), `diag_token.txt` (**DEAD
  — revoked at row-22 teardown**), session-6 run artifacts
  (`export-pre.ics`, `login*.html`, `lp-pre.html`, `add1post.html`,
  `refresh*.html`, `ssrf*.html`, `remove1.html`, `report-imported-*.xml`,
  `report-query.xml`, `e2-edited.ics`, `e4.ics`, `home-set.xml`) and the
  older session-4/5 diagnosis artifacts (`lp0.html`, `add1.html`,
  `repro1.html`, `fp*.html/jar`, `scratch-test.sh` + `diag-run.sh` — the
  working single-ssh-session pattern, `dnsprobe3/` corrected probe source,
  old `dnsprobe*/` flawed probes — cautionary only).
- Router tmpfs: ALL row-22 leftovers REMOVED at session-6 teardown
  (dnsprobe*, rustical-diag + log, row22t/ + row22d/, rustical-row2226
  config/DB/log/pid incl. sqlite `-wal`/`-shm`, pcap/pid files). Verified
  clean; unrelated `dnsmasq-exit-*` files belong to other projects.
- Prod DB deltas vs the §17.15 baseline after session 6: NONE beyond the
  recorded session-4/5 password_resets state (5 rows, all used, NO usable
  links); the row-22 test rows were all created and torn down within the
  session — account back to row-20/21 shape, global 8 subs / 67 tokens /
  0 sources. Rustical NOT restarted during session 6 (still PID 17364).
- `logread` note: the ring buffer is tiny and flooded by dropbear lines
  (only ~3 rustical lines survive minutes later) — start `logread -f` to a
  file BEFORE any probe that needs log evidence.

## Discovered state (resolved at session-6 teardown)

The unrecorded 2026-09-14 19:17 row-22 prep (calendar `srcfeed` with
e1/e2/e3, empty targets `imported` + `imported2`, subscription
`ee496590-829e-47b4-a0d9-7cf2aec3cf18`, diag app token `diag-row22`
id `bbdef0d9-4b2a-4279-92da-13c63025dfef`) is **fully torn down** —
account verified back to the row-20/21 shape (personal + tasks calendars,
2 registration subs, 5 registration tokens); global counts went 9→8 subs
and 68→67 tokens exactly as predicted. Nothing left to clean on the
account side.

## Row 26 checklist (EXECUTED — ALL GREEN, session 7; keep for the record)

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

## When row 26 is green

~~Update PLAN.md: the row-26 matrix cell (LIVE: DONE + method/results)~~
DONE (session 7 — row-26 cell + §17.8.7 pointer updated; the row-22 cell +
§17.8.7 record were already updated at session 6, incl. the 2026-09-14
prep teardown + final counts). Remaining item-6 live work after that:
phone-based registration (user's device — ask).
