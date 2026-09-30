# Next agent: two leftovers, and how not to repeat what went wrong

**Written 2026-09-30. Read this before touching either item.**

`PLAN_DEPLOYMENTS.md` is the authoritative plan and this file is not part of it.
This file exists because both remaining items were mis-scoped once already, and
in both cases the mis-scoping looked like a decision. It records what is actually
true, what is deliberately not true, and the specific traps.

Three of the last four things shipped were wrong in ways that would have passed
every test. That is not a coincidence and it is not bad luck — it is what happens
when a gate is written from a description instead of run against something real.
**Both items below have a rehearsal step, and the rehearsal is not optional.**

---

## State as of this handoff

Three repos, all clean and pushed.

| Repo | Path | Remote | Head |
|---|---|---|---|
| Server | `/home/burningserenity/router-dav/rustical` | `Bytewheel/Omnical-Server` (`omnical-scheduling`) | `35049a53` |
| Packaging | `/home/burningserenity/router-dav` | `Bytewheel/Omnical-Code` | `1bb1b07` |
| Plan | `/home/burningserenity/Documents/ByteWheel/omnical` | `Bytewheel/Omnical` | `e3fc0be` |

Gates green at handoff: **827 workspace tests, 382 all-features, pinned
integration 98/98**. `tests/integration_tests/` is frozen — do not edit it.

### Pushing

The SSH agent socket is at `~/.ssh/agent/`, **not** `~/.ssh/agent.*`:

```sh
export SSH_AUTH_SOCK=/home/burningserenity/.ssh/agent/s.9A30FBhmm9.agent.QMzfCNSGnQ
export GIT_SSH_COMMAND="ssh -o BatchMode=yes -i /home/burningserenity/.ssh/id_burningserenity -o IdentitiesOnly=yes"
```

No local key file authenticates on its own (`ssh -T git@github.com` fails with
each of them) — the agent is what holds the working key. A `Permission denied
(publickey)` here is a missing `SSH_AUTH_SOCK`, not a lost key.

### Do not re-litigate these

They were investigated and closed. Reopening them wastes a session.

- **`POST /register` CSRF.** It exists, at `src/register.rs:370`, tested at
  `:1506`. `PLAN_DEPLOYMENTS.md` §18.20 listed it as missing for weeks. It was
  not.
- **Bounded readahead queue.** Not built, deliberately. A `posix_fadvise`
  prototype measured no effect outside the noise floor and was reverted. The
  cold-tenant budget (~102 ms) is **accepted**. `mmap_size` is kept because it
  measured.
- **Invite code prefill.** Shipped in `5366b5a8`.

---

## Item 14 — appliance panel: Data / Network / Firmware

### What is true now

The **Support** section is done and gated (`tests/support_bundle.rs`, 13 tests,
row 50 green). The three remaining sections are not built.

The plan's own note says they "are thin wrappers over `backup`/`restore` and a
`sysupgrade` call, and building them well needs a unit to test against." That
second half is where this goes wrong, so read the next part.

### The plan is wrong that these need a unit

**Everything except the flash write is testable on this machine.** Items 12, 13
and 14 were all completed "where they can be proved without hardware" (§18.22),
and the same split works here:

| Section | Needs a unit? | Why |
|---|---|---|
| **Data** — list restore points, trigger a backup, restore | **No** | `backup`/`restore` already run anywhere; §18.5 proves it on any machine |
| **Network** — addresses, interfaces, public URL | **No** | read-only from the OS; this box has interfaces |
| **Firmware** — current version, binary size, overlay free | **No** | read-only from the filesystem |
| **Firmware** — the `sysupgrade` call itself | **Yes** | writes flash |

So: build three sections, and make the one destructive call **injectable** so it
is testable with a fake. The precedent is already in the tree — item 13's setup
mode does exactly this for the restart, which is also a real side effect:

```rust
// src/setup_mode.rs:231 — the pattern to copy
pub trait SetupRestart: Send + Sync + std::fmt::Debug {
    fn request_restart(&self);
}
pub struct ProcdRestart;                       // the real one: SIGTERM to ourselves
```

Add the equivalent (`trait FirmwareWriter`, real impl shelling to `sysupgrade`,
fake impl writing to a `tempfile::TempDir` asserting the arguments). Do **not**
ship a `sysupgrade` call you cannot run a test against — that is how rows 49/51
stay red.

`row 51`'s binary half is already enforced every build (5,350,168 B of 35 MiB).
Only the `df -k /` half needs hardware. `row 49` (backup → restore on a second
unit) needs a second unit and will stay open; do not mark it green.

### Traps

- **The panel is not the setup-mode server.** `src/setup_mode.rs` runs its own
  `Router` on a LAN-only bind, entered from the filesystem and left by writing a
  config. The panel is a route on the real app (`crates/frontend/src/routes/`,
  which already holds `admin.rs`, `user.rs`, `app_token.rs`, `groups.rs`). Copying
  setup mode's router wholesale would give the panel setup mode's deliberate
  property — **it answers 404 for anything it does not define** — and there are
  ~13 route modules to reconcile against.
- **The restore action destroys data.** It is a *Data* section button that
  overwrites the live database from an archive. It needs the same treatment
  `rustical upgrade --rollback` got in `35049a53`: a verified backup as a
  precondition, and a row-count report afterwards. Both are reusable — the
  helpers are `resolve_database_path` and the manifest's `row_counts`.
- **Do not assume a restore can be un-done from the panel.** There is no undo.
- **Network section: redaction.** Whatever you surface becomes support-bundle
  adjacent. Row 50's rule is *"the command verifies before writing and writes
  nothing if a secret survives"* — the panel must not be the path that leaks what
  the bundle refuses to.

### Tests

New `tests/appliance_panel.rs`. Model it on `tests/admin_panel.rs` (exists) and
`tests/setup_mode.rs` (13 tests, the closest precedent for a panel with side
effects). At minimum: the three sections render; **the restore button requires
confirmation and a passing row-count check**; a network view with a secret in it
is redacted; the fake firmware writer is invoked with the expected arguments and
nothing else is.

---

## Item — RFC 8525 WebDAV-Push notification

**This is a new feature, not a configuration task.** It is the only item on the
leftover list that is greenfield.

### What is true now

§12 row 36 is `NOT TESTABLE, NOT STARTED`. The reason, recorded honestly:

- `crates/dav_push/src/endpoints.rs` is **24 lines** and routes exactly one path,
  `DELETE /push_subscription/{id}` — the unsubscribe call. That is all of it.
- There is **no WebSocket dependency anywhere in the workspace.**
- `src/tenancy.rs:272` deliberately *drops* the per-tenant update receiver —
  `let _ = bundle.take_update_recv();` — rather than draining it, "so a
  notification cannot be lost to a receiver nobody is reading." The comment is
  correct and the event source already exists.
- The DB tables already exist: `davpush_subscriptions`, `davpush_vapid_key`.
- `crates/dav_push/src/vapid.rs` is **155 lines** — VAPID is done.

So the substrate is mostly built. What is missing is the wire.

### THE trap: a test that passes because the feature is missing

`tests/trusted_proxies.rs:490` is not ordinary coverage. It is a deliberate
tripwire named `there_is_no_webdav_push_socket_to_test`, and it asserts the
**absence** of the thing you are about to add:

```rust
for dependency in ["tokio-tungstenite", "tungstenite", "async-tungstenite"] {
    assert!(
        !cargo.contains(dependency),
        "a WebSocket dependency ({dependency}) has appeared: WebDAV-Push notification may \
         now be implemented, and row 36 needs a real test"
    );
}
```

It also greps `src/tenancy.rs` for `take_update_recv`.

**Adding a WebSocket dependency will fail this test on purpose.** That is the
test working. Do not "fix" it by deleting the assertion, and do not satisfy it by
choosing a socket library whose crate name is not in that list — the absence of
those three strings is how the row knows to be revisited, and evading it by
renaming leaves row 36 falsely claiming coverage. **Convert it**: replace the
absence assertion with the real row-36 test it was asking for.

### Scope, smallest honest version

1. `PUT` on the subscription URL to create/renew (RFC 8525 §4) — the missing
   half. `register.rs` (105 lines) and `subscription.rs` (39 lines) are the
   existing pieces.
2. A notification socket, and draining `take_update_recv()` into it.
3. VAPID signing onto each notification (`vapid.rs` is done — use it, do not
   re-implement).
4. The edge must preserve `Upgrade`. §7.3.2 already documents that a stripping
   proxy degrades push to polling; row 36's gate is *"the socket is open (verify
   explicitly, not by 'sync works')"* — the parenthetical is the whole point.

Decide and **write down** whether the socket is per-process or shared. This
matters: §7.3.4's session problem is exactly this shape (per-process sessions
break at k>1 behind a load balancer), and row 36 is *"WebDAV-Push upgrade
survives the LB"*. A per-process socket that works on one box is not row 36.

### Tests

Convert the tripwire into the real gate: a DAVx5-shaped subscription created over
a real socket, an object mutated, and a notification **received** — verified
explicitly, not inferred from a subsequent successful sync. Then the LB case
separately, which is the actual row.

---

## The one instruction that applies to both

**Run it against something real before believing it.**

The pattern that has now produced three silent defects:

| What | Why every test passed |
|---|---|
| `CONTENT_MUTATING_DOWNS` pinned `20260701120000` | a version that does not exist — the UID-destruction warning **never fired for the one migration it exists to warn about** |
| rotation drill's protected-table list | `addressbook_objects`, `davpush_vapid`, `group_ownership` — **three table names that do not exist** |
| `verify` read a file only `selftest` wrote | **the selftest passed and the real run crashed with `FileNotFoundError`** |

All three were lists or wiring **asserted against themselves rather than against
the codebase**. Two for two, from two different sessions. And the two honest
limits were only found by running:

- Row counts **cannot see a dropped column.** Rolling back past
  `add_calendar_uid` reported `calendarobjects 3 -> 3` — every event survived,
  true — while the `uid` column was gone and unreported, because no *table*
  shrank. A green report meant "no table lost rows", a strictly weaker claim
  than the banner above it implied.
- A **real** rehearsal of item 21 found `losses()` reporting the *intended*
  token revocation as `LOST ROWS` under a "this is what DROP COLUMN means"
  banner. How a deliberate revocation gets printed as collateral damage.

So, for both items:

- Exercise them against a **real migrated database and a real filesystem** — not
  only fixtures. `rustical serve` on a scratch config creates one; §18.27 did it
  with `[data_store.sqlite] db_url = "sqlite:///tmp/…?mode=rwc"`.
- **Every hand-maintained list gets a test that checks it against the codebase.**
  Panel section names, route names, table names, version pins. If the list is
  checked against a copy of itself it is decoration.
- **Distinguish the change you meant to make from the damage.** The pattern to
  copy is `RollbackReport::losses()` vs `expected_shrink()` in
  `src/commands/upgrade.rs`, and `scripts/credential-rotation-drill.sh`'s
  single allowed table (`app_tokens`). A check that cannot separate those two
  will either fail on success or pass on damage.
- **A green result means exactly what it measures.** Print the claim, not an
  implication.

## Gates before either commit

```sh
cargo fmt --all --check
cargo test --workspace                                        # was 827
cargo test --workspace --all-features                          # was 382
cargo test --test run_integration_tests                       # was 98, DO NOT EDIT tests/integration_tests/
cd .. && scripts/credential-rotation-drill.sh selftest        # §18.27
```

`clippy` is **not** a gate in this repo (180 warnings, and CI does not run it),
but new files should be clean — `src/commands/upgrade.rs` is at 0.

## Do not

- Do not edit `tests/integration_tests/`. It is the frozen 98.
- Do not delete or weaken `there_is_no_webdav_push_socket_to_test` — convert it.
- Do not force-push, and do not rewrite the plan's historical sections. §18.24
  records a finding that §18.25 withdrew, and both stay: a plan that hides its
  own corrections is a plan nobody can audit.
- Do not touch `out/tls/` or `out/*.mobileconfig`. §5.2 item 1's rotation
  handled the live credentials; `out/` stays ignored and must stay that way.
- Do not mark row 49 or row 51 green without the hardware they need.
