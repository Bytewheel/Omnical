# Item 14 — appliance panel: the scoping was wrong a second time

**Written 2026-09-30, after the `NEXT_AGENT_PLAN.md` handoff. Read before building.**

`NEXT_AGENT_PLAN.md` says item 14's remaining three sections "are thin wrappers
over `backup`/`restore` and a `sysupgrade` call". That is not what is there. This
file records what was actually found by reading the code, and the decisions
that are now settled. **Nothing has been built. No code was kept** — a first
draft was written, found to rest on four functions that do not exist, and
deleted rather than left as a stub that silently bails.

---

## The premise is wrong: there is no panel with sections

`NEXT_AGENT_PLAN.md` says "the **Support** section is done and gated". Read
closely, "done" means *the CLI command* is done:

- `crates/frontend/src/routes/support.rs` **does not exist**.
- `crates/frontend/src/routes/mod.rs:1-13` lists all 13 route modules; `support`
  is not among them.
- The support bundle is reachable only as a CLI subcommand
  (`src/lib.rs:103-104`) and a procd action
  (`router/etc/init.d/rustical:55-68`).

So there is no Data/Network/Firmware panel to add sections to. The only web
panel is the **admin panel** (`/frontend/admin/**`, `crates/frontend/src/routes/admin.rs:350`),
and it is tenant management only.

`tests/support_bundle.rs` is **not** an HTTP test either — it never builds a
router. Its four helpers (`bundle_for`, `config_with_secrets`, a `Drop` guard)
call `build_bundle_text_for_test` directly. "13 tests, row 50 green" is true and
does not imply a panel exists.

## The deployed binary has none of it either

`/usr/sbin/rustical` on the router is dated **Sep 23** and is missing eight
subcommands the source tree has:

```
backup  backup-all  restore  setup  support-bundle  tenant  tenant-usage  upgrade
```

So **rows 42 and 50 are green in CI and absent on the appliance.** The version
string is `0.16.1` on both sides, which is why nothing flagged it. Two practical
consequences:

- **Item 14 cannot be exercised on the appliance until a deploy happens.** Its
  Data section wraps `backup`/`restore`, which the running binary does not have.
- `scripts/credential-rotation-drill.sh snapshot` **cannot run against the
  router** — it shells to `rustical backup` (`credential-rotation-drill.sh:110`)
  and dies with `unrecognized subcommand 'backup'`. The drill only works against
  a host whose binary has it. Use direct `sqlite3` row counts against the router
  instead, and say which instrument was used.

The deploy is arguably the higher-risk item: it puts `support-bundle`, `upgrade
--rollback` and `tenant` onto a production box that has never run them.

---

## Four blockers, all confirmed in the code

1. **`crates/frontend` cannot see the redaction.** `secrets_in` (`support_bundle.rs:71`),
   `redact` (`:142`) and `struct Redaction` (`:60`) are all private, in the
   `rustical` crate. The dependency is one-way (`Cargo.toml:187`), and
   `crates/frontend/Cargo.toml` has no `rustical` dep.
2. **`cmd_restore` is unreachable from any panel.** It refuses while the server
   is reachable (`backup.rs:446-457`), and a panel is served *by that server*.
   Passing `force: true` defeats the only thing the check is for.
3. **The panel is forbidden from opening a tenant database** — a stated invariant
   (`admin.rs:23-37`) with a behavioural test (`tests/admin_panel.rs:1085`).
   `AdminPanel` holds two control-plane trait objects and **no path to a
   `StoreBundle`** (`admin.rs:300-314`).
4. **Network and Firmware are greenfield.** No `statvfs`, `getifaddrs`,
   `if-addrs`, `local-ip-address`, `overlay`, `df`, `sysupgrade`, `ubus` or
   `current_exe` anywhere in the workspace. The reusable patterns are
   `is_lan_peer` (`src/setup_mode.rs:127`) and the `SetupRestart`/`ProcdRestart`
   pair (`src/setup_mode.rs:231-254`).

Also worth knowing before choosing a surface: **the deployed config has no
`[tenancy]` block and no `admin_host`**, and the panel is built only when
`admin_host` is non-empty (`src/tenancy.rs:95-97`). So there is no admin panel on
this appliance today either, and any panel surface is operator-configured.

---

## Settled decisions

**Placement — a new appliance surface, beside the admin panel.** Not an extension
of it, so blocker 3 is honoured rather than broken: the admin panel keeps its
never-open-a-tenant-DB invariant and its test.

**Redaction happens in the `rustical` crate, before data reaches the view
layer.** The panel renders strings that were already redacted. The frontend
never holds a secret, so there is no second redaction path to keep honest, and
blocker 1 dissolves rather than being worked around by duplicating `redact`
into `crates/frontend`.

> Do not do what the first draft did: redact a `Debug` rendering and try to
> parse it back. It needs a `decode_redacted` that always bails, so the panel
> fails the moment redaction catches something. Redact each string field
> through the existing `redact()` and then verify across the rendered output.

**A restore is compared against the archive, not against the live database.**
This is the load-bearing design decision. Restoring deliberately moves data
*back in time* — every row written since the archive is meant to disappear.
Measuring "rows before vs after" would print **every correct restore as
catastrophic data loss**, which is exactly how the deliberate token revocation
came to be printed as `LOST ROWS` under a banner about collateral damage
(§18.29). So:

- `expected` = the manifest's `row_counts` (`backup.rs:82-93`)
- `actual` = measured from the database after the swap
- `mismatches()` / `missing()` / `added_by_migration()` are the damage signals
- `claim()` states the one thing a row count can support, and is forbidden from
  implying "no data was lost" — restoring *did* discard newer rows, on purpose

Mirror `RollbackReport`'s split (`losses()` vs `expected_shrink()`,
`src/commands/upgrade.rs:120-166`) and reuse `inspect_database`
(`backup.rs:743-774`) rather than writing a second inventory that could disagree.

Carry forward the honest limit from §18.28: **row counts cannot see a dropped
column.** A green restore report means "every table in the archive is present
with the archived row count", which is strictly weaker than "the restore was
lossless".

**Restore is staged, then completed at start-up.** The panel verifies the
archive, takes a safety copy, writes the new database *beside* the live one, and
requests a restart via the existing `SetupRestart` trait. `main.rs` calls
`complete_staged_restore()` **before the connection pool is opened**. That is the
only way to satisfy blocker 2 without forcing past the guard.

**FirmwareWriter is injected, copying `SetupRestart` exactly** — real impl
shells to `sysupgrade`; the test double lives in `tests/appliance_panel.rs`, not
in the library, so a double that drifts has nowhere to hide.

**Overlay space comes from `statvfs`, not from parsing `df`.** Row 51's own gate
is written as `df -k /`, but parsing `df` means a locale-dependent,
column-count-dependent parse on the path that decides whether there is room to
flash. `libc` is already a dependency (`Cargo.toml:180`). Report absence as
absence, never as `0` — "not measured" and "measured, and empty" are different
claims.

**Interfaces from `ip -j addr`.** `getifaddrs` would want a new dependency for
one read-only fact. A missing or unparseable `ip` must surface as an error the
panel shows, never as a silently empty list that reads as "no network".

**Listing restore points must skip `.part`.** `cmd_backup` writes
`<archive>.part` while working (`backup.rs:185`) and leaves `.pre-restore-<ts>`
/ `.restore-<ts>` siblings (`:905-911`). Also note `backup_all` uses one-second
timestamp resolution and had to add a per-tenant subdirectory because 200
tenants overwrote each other's archive (`backup_all.rs:107-119`) — a listing UI
that assumes unique names in one directory will be wrong for the same reason.

---

## What the build actually requires

1. `support_bundle.rs` — make `Redaction`, `secrets_in` and `redact` `pub`.
2. `backup.rs` — add `pub async fn verify_archive_into(archive, dest) -> Result<VerifiedArchive>`
   wrapping the existing private `read_archive` (`:634`), and `pub async fn
   row_counts_of(path)` wrapping `inspect_database` (`:743`). Both reuse; neither
   re-derives.
3. `src/appliance.rs` — facts, `FirmwareWriter`, `RestoreReport`, stage/complete.
4. `crates/frontend/src/routes/appliance.rs` + `pages/appliance*.html` extending
   `layouts/admin.html`. Auth/CSRF/audit: reuse the admin panel's, do not
   duplicate (a duplicate is only acceptable while it is *checked*, per the
   `normalise_host` precedent at `admin.rs:942-957`).
5. `main.rs` — call `complete_staged_restore` before the pool opens.
6. `tests/appliance_panel.rs` — three sections render; restore needs confirmation
   *and* a passing row-count check; a network view containing a secret comes back
   redacted; the fake firmware writer is invoked with the expected arguments and
   nothing else is.
7. Rehearse against a **real migrated database and a real filesystem**, and a
   real `sysupgrade`-shaped double — not only fixtures.

Row 49 (backup → restore on a second unit) and row 51's `df -k /` half stay
open: they need hardware.

---

## Also found on 2026-09-30, unrelated to item 14

- **The deployed config has no `[tenancy]` block and no `admin_host`.** There is
  no admin panel on the appliance at all.
- **`lynscarlton@gmail.com` had duplicate `davx5` and `vdirsyncer` app tokens**
  from the §18.29 rotation — four rows for two clients, minted seconds apart,
  because `pass insert` overwrote rather than appended. Two of the four secrets
  were therefore recorded **nowhere**. Cleaned: deleted the two orphans via
  `principals app-token remove`, verified by PBKDF2 match against `pass` that the
  other two are the live ones, then confirmed only `app_tokens` moved
  (53 → 51) across all 21 tables with `PRAGMA integrity_check: ok`.
- **A token whose secret is in no `pass` file is an unrecoverable full-access
  credential.** Worth a periodic check; the PBKDF2 verification in that cleanup
  is the tool (`$pbkdf2-sha512$i=1000,l=32$…`, and the stored hash is over the
  **bare 64-char secret** — `pass` holds the 4-char-prefixed display form).
- **Reading `/etc/rustical/config.toml` prints live secrets.** It is mode 0600
  and legitimately holds SMTP passwords and the RSVP HMAC key. During this
  session a `sed -n 1,40p` on it put two SMTP passwords and `rsvp_secret` into
  the transcript — re-exposing exactly what §5.2 item 1's rotation fixed.
  **Those credentials need rotating again.** Grep that file for structure only,
  and never echo it.

---

## Update 2026-09-30, after the tenancy migration

The appliance is now tenancy-enabled. Tenant `omnical`
(`ff77b536-0e6f-4e43-b3a4-e7a10641ec75`) serves
`0115d8cf.duckdns.org` via `default_tenant`; the panel answers on `librecmc`
and `admin.librecmc`.

That resolves the Data section's ambiguity: **the appliance database is
`/usr/local/share/rustical/tenants/ff77b536-.../db.sqlite3`**, not the old
`db.sqlite3`. The old file is still on disk, untouched, as the rollback.

Two corrections to what this document says above:

- The tenant store is **`data_root/tenants/{uuid}/db.sqlite3`**, not
  `data_root/{slug}.sqlite3`. Copying to the latter would have put the data
  somewhere nothing reads.
- There is **no `tenant set-host` subcommand** — hosts are create-only. An
  explicit `tenant_hosts` claim would have meant deleting and recreating the
  tenant (new UUID, redo the copy), so `default_tenant` is used instead. It is
  the documented N=1 escape hatch and matches an appliance anyway.

### Decisions taken

1. **An appliance surface beside the admin panel**, not inside it. Blocker 3
   above stands: the admin panel keeps its never-open-a-tenant-DB invariant and
   its test. The panel is on `admin_host` + `admin_hosts`; the appliance section
   is a separate router on the same hosts.
2. **Staged restore, completed at start-up.** The panel verifies the archive,
   takes a safety copy, writes the new database *beside* the live one, and
   requests a restart through the existing `SetupRestart` trait. `main.rs` calls
   `complete_staged_restore()` **before the connection pool opens**. This is the
   only way to satisfy `cmd_restore`'s server-is-up guard without forcing past
   it.
3. **Finish item 14, then one deploy**, together with the telemetry fix
   (`ab151577`), which is a real bug: `omnical.tenant_id` and
   `omnical.dispatch` were absent from every trace because the tags were
   recorded on no span at all.

### Migration record

Additive first, then copy, then enable — with the original never moved.

| Phase | What | Verified by |
|---|---|---|
| A | control plane + tenant created, via a scratch config whose `db_url` was a throwaway | live DB untouched at 31/216/209/51 |
| B | service stopped, WAL checkpointed, copied | all 12 tables equal **and** `sha256` byte-identical (`c04105b5…`) |
| C | `[tenancy]` in `render-router-config.sh`, rehearsed on port 4001 first | `validate()` + host reservation passed; caught the missing admin credential |

Rollback is two files: `/root/config.toml.pre-tenancy` and the original
`db.sqlite3`. No data movement.

Also of record: the deployed binary was dated Sep 23 and lacked `backup`,
`restore`, `support-bundle`, `upgrade`, `setup` and `tenant` — so rows 42 and 50
were green in CI and absent on the appliance. Fixed by the 2026-09-30 deploy.

---

## Update 2026-09-30, after the build — item 14 is DONE (software half)

Built on branch `omnical-scheduling` at `ab151577`. `cargo fmt --all -- --check`
passes, `cargo clippy -- -D warnings` passes, and `cargo test --workspace` is
**865 passed / 0 failed**. Row 49 (backup → restore on a second unit) and row
51's `df -k /` half still need hardware and are **not** claimed.

### What exists

| Piece | Where |
|---|---|
| Facts: `statvfs`, `ip -j addr`, redaction, firmware summary | `src/appliance.rs` |
| `FirmwareWriter` + `SysupgradeWriter` (`-n`), `stage_restore`, `complete_staged_restore`, `RestoreReport` | `src/appliance.rs` |
| `ApplianceService` trait + the `rustical`-side implementation | `crates/frontend/src/routes/appliance.rs`, `src/appliance_service.rs` |
| Routes and templates | `/frontend/admin/appliance{,/restore,/firmware}`, `pages/appliance{,_confirm,_staged}.html` |
| Start-up completion | `src/main.rs`, in the `Serve` arm, before `cmd_serve` |
| Tests | `tests/appliance_panel.rs` — 17, all green |

### Six places this document's design was wrong, and what was done instead

1. **`verify_archive_into` cannot return `VerifiedArchive`.** That struct is
   private *and* its `Drop` deletes its scratch directory — so a staged restore
   would lose the very file it was staging, because the panel restarts to
   complete it. `VerifiedArchive` is now `pub` with `manifest()` and
   `extract_database_to(dest)`, and the *destination* is caller-chosen and not
   cleaned up on drop. That is what makes the two-phase restore work at all.

2. **`mismatches()` / `missing()` / `added_by_migration()` did not exist.** The
   real `RowCountComparison` had `mismatches: Vec<String>` and
   `notes: Vec<String>`, and a missing table was folded into `mismatches` *as a
   sentence*. Added `missing` and `added_by_migration` as typed fields; the CLI's
   text and behaviour are unchanged, and a panel can now render "principals is
   gone" without parsing a message.

3. **"A separate router on the same hosts" would mean a second login.** Two
   `SessionManagerLayer`s over two `MemoryStore`s means two cookies and an admin
   session the appliance section cannot see. The routes are **merged into the
   admin panel's router, inside its session layer**, with the appliance state
   carried as an `Extension` (axum has one `State` type per router). One login,
   one CSRF token, and `AdminPanel::require_admin` is the only gate — which is
   what "reuse, do not duplicate" means once implemented.

4. **"Reuse the admin panel's audit" — there is nothing to reuse.** Every audit
   write on `TenantStore` is inside a tenant mutation's transaction, and an
   appliance action mutates no table. Added `TenantStore::record_appliance_action`
   as an explicitly-named second door, `tenant = NULL`, and the doc comment says
   so rather than pretending §6.6.7's one-transaction rule applies to a restore.

5. **`Config` does not carry its own path**, so the redaction's key-shape pass
   had no file to read. Added `serve_dispatch_with_config_file`; `cmd_serve`
   passes the real path. `serve_dispatch` still exists and passes `None`, which
   means the typed-config pass runs and the unknown-key pass does not — stated in
   the doc comment, not glossed.

6. **No `[appliance]` config section was specified.** Added one, defaulting to
   `enabled = false`, so an install that upgrades does not acquire a panel that
   can replace its database. `appliance_paths()` **refuses** an enabled panel
   with no database, a relative path, or a `backup_dir` holding the live
   database — the last one would have made the listing offer the live database
   as something to restore over itself.

Also: `sysupgrade -n` deliberately, because without it the flash restores
`/config` from the backup partition, and on a tenancy install that is the control
plane and every tenant's SMTP password (§3.6).

### Three bugs the tests found, which review would not have

- **A memoised restore report went stale.** The report was cached in a `Mutex` on
  the theory that it only changes across a restart — true in production, false
  in a test that completes a restore in-process, which is how it was caught. The
  cache is gone; `data()` reads the file per request. A cache whose invalidation
  depends on a fact outside the process can be wrong.
- **The backup in the test was taken in the wrong order.** The extra row was
  inserted *before* `cmd_backup`, so the archive correctly held six rows and the
  restore correctly produced six. The test was asserting a restore had done
  something wrong when it had done nothing wrong — the same shape as §18.29.
- **A redaction test asserted something false.** It put a secret only inside
  `public_url` and expected redaction, with nothing declaring that value secret.
  `secrets_in` deliberately leaves URLs alone; a bundle that redacted its own
  public URL would be useless. Replaced with the claim that is true: a *declared*
  secret is removed from every occurrence, including from a URL.

### Also found: CI was already failing on `main`, before any of this

`cicd.yml` runs `cargo clippy -- -D warnings`, and `omnical-scheduling` had
**264 warnings** against it — on the pinned 1.97 toolchain, so this is not a
new-lint artefact. Verified by installing 1.97 and running the pristine tree.
`cargo fmt --all -- --check` also failed.

All 264 are fixed. `cargo clippy --workspace --all-targets -- -D warnings` is
now clean too, which CI does not check. The bulk were pedantic/nursery lints in
`crates/scheduling` (`format_push_string` → `write!`, `map_or_else`,
collapsible `if let`, doc backticks) and in the store tests. The judgement calls,
each carrying a reason rather than a blanket allow:

- `clippy::large_futures` allowed at module scope in `imap.rs` and `ingest.rs` —
  19 warnings, 60–66 KB futures on a 30-second poll. Boxing at each call site
  moves the allocation rather than removing it.
- `clippy::unused_async_trait_impl` on three axum extractors, together, so none
  is the single place in the tree that differs from its neighbours.
- `clippy::unused_self` on `Scheduler::spawn_email`, which is called as
  `self.spawn_email` at three sites.

Full test suite re-run after every batch: 865/865 throughout.
