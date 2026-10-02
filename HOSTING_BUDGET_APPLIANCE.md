# Omnical appliance — hosting requirements

**Companion to `HOSTING_BUDGET.md`.** That document covers the server options
(1 vCPU-and-up VM through a resold multi-tenant service). This one covers
**Option 3: the flashed appliance** — Omnical running as a firmware image on
router-class hardware.

**Tag convention is inherited from the main document.** **[M]** measured,
**[D]** derived with arithmetic shown, **[P]** planning placeholder, **[X]** does
not exist and is listed in §8.

---

## 1. What the appliance is, and why it exists

A single flashable firmware image that turns compatible router-class hardware
into an Omnical node that **re-provisions itself after every flash**.

The reason it exists is a specific, documented failure. A router firmware
upgrade (`sysupgrade`) wipes `/usr/sbin/rustical`, `/usr/sbin/dav-tls`, both
custom `/etc/init.d` scripts and the health watchdog. This was verified against
the router's real preservation list — ground truth from an actual
`sysupgrade -b` tarball listing, not assumptions (`PLAN.md:1773-1783`):

| Survives a `sysupgrade` | Wiped, must be redeployed |
|---|---|
| The database, via `/etc/sysupgrade.conf` | `/usr/sbin/rustical`, `/usr/sbin/dav-tls` |
| `/etc/rustical/` — config **including SMTP credentials and TLS certs** | `/etc/init.d/rustical`, `/etc/init.d/dav-tls` |
| `/etc/crontabs/root` — all five cron lines, via `/lib/upgrade/keep.d/busybox` | `/usr/bin/rustical-watchdog` |
| All `/etc/config/*` UCI — firewall rules, uhttpd | opkg-installed packages (`sqlite3-cli`) |
| dropbear keys, `authorized_keys`, network config, `passwd`/`shadow` | |

For a router on your desk, the recovery is a laptop, an SSH session and a
re-run of the deploy script — idempotent, verified live, **~10 s** for the
binary swap (`PLAN.md:1796`). For a retail unit there is no laptop, no SSH and no
secret store. Hence the image.

**How it survives a flash — three mechanisms, because there are three code paths:**

| What | Mechanism | Survives |
|---|---|---|
| binaries, init scripts, watchdog | **package-owned files** — opkg restores a package's file list | a fresh flash |
| init scripts on an *in-place* upgrade | `omnical.keep.d`, a `sysupgrade -b` keep list | a keep-settings upgrade |
| `/etc/rustical`, `/usr/local/share/rustical` | `/etc/sysupgrade.conf`, re-asserted idempotently by the postinst | data + config, every time |
| the base `config.toml` | installed as a **`.sample`**; the postinst copies it **only if there is nothing to preserve** | a preserved config, byte-identical |

That last row is the one that matters and the one tested hardest. The postinst
runs on every flash, forever, with nobody watching it. An earlier draft re-seeded
the config unconditionally, which on a post-flash boot would silently revert a
production unit to defaults — losing per-tenant SMTP credentials and the RSVP
signing secret — and the only symptom would be bounced invites minutes later
with no obvious cause. That is worse than a box that does not boot. The CI gate
runs the real postinst against a fake root containing a config **with secrets
in it**, asserts the output is byte-identical, then runs it twice more to prove
idempotence.

---

## 2. Hardware requirements

### 2.1 Reference unit — the only fully measured machine **[M]**

ThinkPenguin TPE-R1400, verified 2026-09-04 (`PLAN.md:150-165`):

| Component | Specification |
|---|---|
| SoC | RockChip **RK3328**, 4× Cortex-A53, aarch64 |
| Firmware | libreCMC v6.8 `r0-44c488c9`, kernel 5.15.210, target `rockchip/armv8` |
| **RAM** | **1,011,084 KB total; ~828 MB available** (127 MB used by the existing nym/mullvad/bind stack) |
| **Root** | `/dev/root` squashfs 3.8 MB, read-only |
| **Overlay** | `/dev/loop0` **f2fs, 98.4 MB total, 42.2 MB free at install** |
| tmpfs | `/tmp` ~494 MB, RAM-backed, wiped every boot |
| Package feeds | 2,435 packages listed |

**Minimum for an appliance:** aarch64, 1 GB RAM, ≥98 MB overlay, and either a
libreCMC or OpenWrt/libreCMC base image. Anything that satisfies the reference
unit is known to work; anything richer is untested.

**Not available in the firmware feeds** — this constrains what you can add:
python3, php, node, nginx, lighttpd, stunnel, haproxy, radicale (`PLAN.md:164`).
**Available:** `sqlite3-cli`, `ca-bundle`, `curl`, `socat` — note `socat` is
compiled *without* OpenSSL and **cannot terminate TLS**.

### 2.2 The 35 MiB overlay budget **[M]**

This is the appliance's central physical constraint, and it is enforced
automatically.

| | Bytes | Share of budget |
|---|---|---|
| Budget | **36,700,160 B (35 MiB)** | 100% |
| `rustical`, aarch64-musl, UPX-packed | 5,350,168 | 14.6% |
| `dav-tls` | 620,884 | 1.7% |
| init scripts, watchdog, keep-list, config sample | 14,189 | 0.04% |
| **Current total payload** | **6,097,363 B (5.81 MiB)** | **16.6%** |

Enforced twice, independently: `scripts/build-rust.sh:30,146-155` fails the
build, and the CI workflow applies the same 35 MiB gate to **both** binaries
(`build.yml:107-119`) plus a static+UPX assertion. The budget has **not** been
raised and **must not** be raised.

Size history, for context on headroom: 29.0 MiB → 15.9 MiB → 4.85 MiB. Stock
upstream 0.16.1 with no optimisations is **35,844,376 B — over budget by itself**,
which is what started the campaign.

**The overlay floor that matters more than the payload.** `sysupgrade` requires
**≥ 5 MB free** before it will proceed (`FLOOR_KB=5120`), and the nightly backup
monitors against that same floor and logs a warning when crossed. Observed overlay
free space across the deployment, which shows how tight this really is **[M]**:

| Point in the deployment | Overlay free |
|---|---|
| Factory, before install | 42.2 MB |
| After first install | 12.3 MB |
| After bulk data push, **before** WAL checkpoint | **2.6 MB — below the 5 MB floor** |
| After `PRAGMA wal_checkpoint(TRUNCATE)` | 9.3 MB |
| Steady state, recent deploys | 6.6 – 27.5 MB |

**A recorded failure worth reading before you commit hardware** (`PLAN.md:2681-2687`):
during a deploy, **ENOSPC — the overlay could not hold a second copy of the
binary next to the running one.** The `scp` failed mid-write, leaving a 26 MB
partial file that had to be removed by hand. A running process's blocks are only
reclaimed when it exits. The fix is to stage binaries in `/tmp`, which is tmpfs
and therefore **costs zero overlay** — so the deploy does stop → copy → start,
with the copy on RAM.

**Planning consequence:** budget the overlay for **data, not for binaries**, and
expect a bulk import to drive free space to roughly 2.6 MB before checkpointing.
A 128 MB overlay gives meaningful margin over the reference unit's 98.4 MB.
If your units will hold more than a few hundred MB of calendar data, the overlay
becomes the binding constraint on the whole design and you need a larger flash
device or an external-data design that does not exist yet.

### 2.3 Memory budget **[M]**

| Consumer | Amount |
|---|---|
| `rustical` resident set, deployed single-tenant | **21 MB** |
| `dav-tls` resident set | **not measured [X]** — and it has no memory limit (§7) |
| Hard cap enforced on `rustical` | **256 MiB** `RLIMIT_AS`, `procd_set_param limits memory="268435456"` |
| Rationale in the init script | "hard cap well under free RAM"; sized for one tenant on 1 GB |
| `dav-tls` | **no limit at all** |

The 256 MiB rlimit is a documented hard fact and it constrains the appliance
permanently: **200 tenants all warm is ~190 MiB of pool memory at 0.93 MiB per
resident pool**, which is most of that budget. This is the strongest argument in
the whole design for one SQLite file per tenant — per-tenant files mean per-tenant
write contention is already isolated, where a shared database would serialise
every tenant's writes.

**Conclusion: the appliance is a single-tenant SKU.** Not a policy decision — the
memory cap makes it arithmetic.

---

## 3. Software payload and build

### 3.1 What is in the image

An OpenWrt package, `omnical`, with `PKG_LICENSE := AGPL-3.0-only`, depending on
`sqlite3-cli`. Nothing is compiled inside the SDK — both binaries are built
outside it by `scripts/build-rust.sh`, deliberately, because an SDK toolchain
would be a second redundant way to produce the same artefact.

The package installs: two binaries to `/usr/sbin`, two init scripts to
`/etc/init.d`, the watchdog to `/usr/bin`, the keep-list to
`/lib/upgrade/keep.d/omnical`, the base config to
`/etc/omnical-config.toml.sample` (**as a sample, not as the live config**), and
creates `/usr/local/share/rustical` — which must hold the database because
**`/var` is tmpfs and vanishes on every reboot**.

### 3.2 Building an image

```sh
scripts/build-rust.sh aarch64-unknown-linux-musl   # both binaries, ~3.5 min clean
scripts/build-firmware.sh --staged-only            # stage + check the budget, nothing else needed
scripts/build-firmware.sh /path/to/imagebuilder     # the actual image
```

- The **staging step needs nothing but the two binaries** and prints the combined
  payload against the 35 MiB budget.
- The **image step needs an OpenWrt/libreCMC ImageBuilder**. There is **no cached
  SDK or ImageBuilder** on the build machine — every image is a download.
- Build toolchain: Rust 1.97.0, edition 2024, clang 22 as CC, zig 0.16.0 for the
  musl sysroot, rust-lld, `opt-level = "z"` + LTO + `codegen-units = 1` + strip.
- Toolchain note: the `zig`-as-`CC` route is marked **broken at link** in the
  script's own header. Use clang.

### 3.3 The gate that verifies all of this — with no hardware **[M]**

```sh
scripts/firmware-gate.sh    # no router, no ImageBuilder, no binaries needed
```

A 2-second CI job. It exercises the postinst against a fake rootfs and asserts the
recipe installs the right files, the keep-list is correct, the watchdog cron line
is added **without duplicating on re-run**, and a config containing secrets comes
out **byte-identical**.

It includes a **mutation step** that breaks the clobbering guard on purpose and
**requires the gate to fail** — because a gate that cannot fail gets read as
coverage. Four mutations run in CI. Worth noting that two earlier `sed`-based
mutation versions silently mutated nothing and passed; the current one asserts the
replacement actually happened.

---

## 4. First boot and provisioning **[M]**

A flashed unit boots into a **browser setup wizard**, LAN-only, that writes the
config, the control database, tenant #1 and the administrator — then sends itself
`SIGTERM` and restarts into normal mode.

Design points that matter for a retail unit:

- The LAN requirement is a **per-request peer check, not a bind.** Pinning an
  address is precisely what breaks on DHCP-assigned retail hardware (§5.1).
- `X-Forwarded-For` is deliberately **ignored** during setup. Nothing is
  configured yet, and any such header is an attacker's.
- The wizard refuses to start if an `init.d` script requires a config to exist —
  an earlier draft did exactly that, which would **brick the wizard**, since a
  factory flash has no config.

19 tests in `tests/setup_mode.rs` cover this path.

---

## 5. Blockers and constraints specific to this SKU

### 5.1 `dav-tls` WAN-IP hard-pinning — **the one real piece of new work** 🔴

`dav-tls` **hard-pins `WAN_IF=eth0` and `WAN_IP=192.168.1.21`**, and **refuses to
start if the live address differs.** The check reads the actual interface address
and aborts with a message telling the operator the DHCP lease changed.

On the reference router that address is static, so it is correct there. **On a
retail unit the WAN address is DHCP-assigned and will move.** The script detects
this and refuses to start, which is the correct behaviour for a static config and
the wrong behaviour for retail hardware.

This is called out in the plans as high-likelihood/high-impact and as *the one
place where this option needs genuine new development* (`PLAN.md:1308`,
`PLAN_DEPLOYMENTS.md:1486`). Two ways out:

1. **Make it runtime discovery** — real work in `dav-tls`.
2. **Ship LAN-only on :8443** with an optional port-forward, and treat discovery
   as a later item.

Which to choose depends on an unanswered question: **is the appliance WAN-
reachable, or LAN-only with a port-forward?** That decision is still open and it
determines whether this work is required at all. It must be settled before any
appliance SKU is quoted.

#### 5.1.1 Update 2026-10-01: this is now implemented — WAN-reachable is viable ✅

The defect above is **fixed**. `dav-tls --discover eth0:8443` resolves the
interface at startup and **re-checks every 30 seconds**, rebinding when the
address moves. Full write-up in `PLAN_DEPLOYMENTS.md §18.30`; the three points a
buyer needs:

| Property | Why it matters for a retail unit |
|---|---|
| A wrong interface **fails loudly**, naming the interfaces it did see | Binding to another adapter that happens to hold a routable address looks healthy and is unreachable. A field typo becomes a ten-second fix, not a truck roll |
| A link-local (`169.254.x`) address is **refused, not bound** | That means DHCP has not completed. Binding there would be a listener that silently never receives anything — the same class of silent failure as the original bug |
| Failures **retry rather than exit** | The usual cause is an interface mid-lease, which resolves itself in seconds. Exiting would hand a two-minute DHCP gap to procd's respawn policy, which gives up after five tries in an hour |

`--listen <ip:port>` is unchanged, so a static-address install is unaffected, and
`DAV_TLS_PINNED` in the init script restores the old pinning explicitly.

**Verified:** 12 unit tests, a 29-check CI gate with a mutation step proving the
gate can fail, and a functional pass against a real interface — the discovered
address bound specifically (not `0.0.0.0`, not a hard-coded value), TLS spliced
through with the request line intact, the plaintext-HTTP 301 intact, 25 sequential
requests, clean SIGTERM shutdown, and nothing bound on an address the interface
does not hold. `dav-tls` grew 1.6%; the staged payload is **5.81 MiB of the 35 MiB
overlay budget (16.6%)**, and `firmware-gate.sh` stays green.

**Verified on a real address change** (2026-10-01; `PLAN_DEPLOYMENTS.md §18.31`).
A discovered listener was moved from `198.51.100.42` to `.43` underneath a live
process. Measured, not asserted:

| Behaviour | Measured |
|---|---|
| Rebinds to the new address | **30 s — exactly one re-resolve tick** |
| Old address stops accepting | immediately, once rebound |
| New address serves TLS correctly | HTTP/1.1, chain verifies, redirect intact |
| 20 requests on the new address | all round-tripped |
| Process restarts during any of it | **never** |
| Address removed entirely (a DHCP gap) | keeps serving the old one, retries every 5 s, **does not exit** |
| Address restored | **recovers on its own in 5 s, no restart** |

The last two are the operational point: a DHCP gap does not become an outage,
because the process never gives up, and recovery needs no SSH, no restart and no
laptop. That is what §9.2 was built for.

**What still needs your hardware.** The address-change path is the same code a real
DHCP lease takes, so it is no longer the open question. What a flashed unit adds is
the parts a dummy interface cannot exercise: **a real DHCP server issuing a
lease**, and the **procd/inittab and flash-storage** layers. One test before you
ship a SKU: boot the unit, note its WAN address, force a renewal from the upstream
router, confirm the public URL answers within ~30 seconds.

This also narrows §8's open item **Q3** (WAN-reachable or LAN-only): it no longer
gates engineering, because both work now. It is a product decision, not a
blocker.

### 5.2 Constraints that shape every appliance design decision **[M]**

| Constraint | Consequence |
|---|---|
| 35 MiB overlay, ~16 MiB free in the target state | Payload is 16.6% of budget. **`sysupgrade` needs ≥ 5 MB free.** Do not raise the gate |
| 256 MiB `procd` rlimit on `rustical` | Sized for **one tenant**. 200 warm tenants ≈ 190 MiB would be most of it |
| Busybox only — **no `stat`, no `base64`** | Sizes computed with `wc -c` |
| `/tmp` is tmpfs, wiped every boot | The deploy fence lives there, which is correct — it is per-boot |
| **A RAM-disk database is impossible** | `/var` is tmpfs; the DB must live under `/usr/local/share/rustical` |
| Two `crond` daemons run on the router | **Every cron job double-runs.** Benign, but wasteful and confusing |
| busybox `crond` does not rescan on file appends | A new cron line stays **inactive** unless the `/etc/crontabs` *directory* is also touched. The deploy script appends **and** touches for this reason |
| `dav-tls` has no memory limit | Its only bound is 1,024 connections, and its Tokio runtime thread count is **host CPU count, uncapped** |
| SQLite WAL is only truncated by an explicit checkpoint | A `DELETE` through the CLI can leave the WAL un-checkpointed. On 2026-09-30 a token revocation left **177,192 bytes** of WAL — copying `db.sqlite3` alone would have yielded the pre-revocation database with **no manifest symptom** |
| `dav-tls` has **no SIGHUP handler and no file watcher** | Certificates reload **only on service restart**, which is a visible few-seconds TLS interruption. Renewal is not seamless |

### 5.3 Backup on the appliance: **does not currently work** 🔴

The in-binary backup command fails on a deployed appliance for three independent
reasons, all recorded and all unfixed as of 2026-09-30
(`NEXT_AGENT_VM_APPLIANCE.md:244-325`):

1. `backup`, `restore` and `upgrade` all fail to locate the database —
   the path resolver rejects any filename containing `:`, and sqlx keeps the
   `file:` scheme in the filename.
2. `backup-all` looks in the wrong directory — it expects the file at the data
   root rather than `tenants/<uuid>/db.sqlite3`.
3. `scripts/nightly-backup.sh` hardcodes the **pre-tenancy** path, and **no backup
   cron is installed on the router at all.**

**Net effect: the appliance currently has no working backup path.** The plans state
this plainly, and it is why the admin panel's Data section would list zero restore
points. Do not quote appliance data as backed up.

### 5.4 What cannot be tested without real hardware

The plans are explicit about the boundary, and this document should not paper over
it:

| Claim | A virtual machine |
|---|---|
| Backup → restore **on a second unit** | **Closes it.** Two independent guests, a real archive from one restored by the other |
| Stripped binary ≤ 35 MiB | Already enforced every build |
| `df -k /` free-space floor on the appliance overlay | **Cannot close it.** A guest's `statvfs("/")` reports the *host's* free space (~200 GiB), so a "free space < 35 MiB, therefore refuse" assertion is **unreachable** |

The honest replacement for the third is a **loopback-backed ext4 image sized to
the overlay budget** — a 48 MiB file, `losetup`, mkfs, mount, point the panel's
`overlay_mount` at it. That makes the filesystem genuinely small so the refusal
path is reachable for real. It is still not f2fs and still not flash, so it is a
**rehearsal**, not the claim.

**Row 49 — restore on a second unit — needs two flashed units or two booted
guests. Everything else about the appliance is testable today.**

---

## 6. Operations on a deployed unit

### 6.1 Services and scheduled tasks **[M]**

Two services under `procd`, both with `respawn 3600 5 0` (retry up to 5 times
within an hour, then stop trying), both logging to syslog:

| Service | START | Notes |
|---|---|---|
| `rustical` | 95 | Binds `127.0.0.1:4000`. 256 MiB memory cap. Pre-flight checks the binary is executable; the config-existence check was **removed on purpose** so a factory flash can reach the wizard |
| `dav-tls` | 96 | Listens `192.168.1.21:8443` → `127.0.0.1:4000`. Runs as `nobody`. Pre-flight **refuses to start without both certificate files**, and again if the live IP differs from the pinned one (§5.1). No memory cap |

A **health watchdog on a 5-minute cron** (`*/5`), checking two things and always
exiting 0 so cron never sees a failure:

1. `rustical --config-file … health` — HTTP-GETs `/ping`. This catches the
   **hung-but-running** state that `procd` respawn cannot, because respawn only
   covers processes that have *exited*.
2. That `dav-tls` still holds a LISTEN socket on :8443 — checked via
   `netstat -tln` and grep, deliberately **not** `netstat -p` process resolution,
   to avoid false restarts of the public TLS front end.

A healthy run logs **nothing**. The failure path has been proven live: the
watchdog logged a restart, the service returned on a new PID, the TLS process was
untouched, and live object counts were identical.

**Why the deploy fence exists** — an incident worth knowing before you operate
these: on 2026-09-08 17:20 the health check `exec`'d the binary mid-swap, the
exec'd process held the old binary's inode, the deploy's `rm` reclaimed nothing,
and the copy died with ENOSPC — leaving a **truncated binary and a dead service.**
The watchdog now checks for `/tmp/rustical-deploy.lock` first and no-ops while a
deploy holds it. Because `/tmp` is tmpfs, a crashed deploy silences the watchdog
until reboot.

### 6.2 Certificate lifecycle

| Item | Value |
|---|---|
| Issuer | Let's Encrypt, via DuckDNS **DNS-01**, issued in ~30 s |
| Validity | ~90 days (measured: 2026-09-04 → 2026-12-03) |
| Renewal trigger | ~60 days out; `acme.sh` cron 4×/month |
| **Mechanism** | scp both files → `/etc/rustical/tls/`, `chmod 600`, **`dav-tls restart`** |
| **Cost of renewal** | **A visible TLS interruption on :8443.** No in-process reload exists |
| Public port | **:8443, not :443** — the 443 slot is occupied by an existing service |

The renewal hook is written without brace expansion on purpose: `acme.sh` runs
`reloadcmd` under POSIX `sh`, where `{a,b}` does not expand.

### 6.3 Bare-router recovery, for completeness

If a unit is flashed with a **plain libreCMC image** rather than the Omnical
image, none of §1 applies and the full laptop-and-SSH runbook is required: stage
binaries via `/tmp`, stop → swap → start, re-push both init scripts and the
watchdog, re-render the config, re-assert the `sysupgrade.conf` entries and the
cron line, reinstall `sqlite3-cli` if missing, health-check. Verified live,
idempotent, ~10 s for the swap, and it tolerates the wiped-router state. One
benign `ubus … Not found` log line on first start is expected.

---

## 7. Cost worksheet — appliance

| Item | Quantity | Source | Your rate | Per unit |
|---|---|---|---|---|
| Hardware | aarch64, 1 GB RAM, ≥98 MB overlay | **[M]** §2.1 | | |
| Larger flash, if data volume requires it | see §2.2 — the overlay is the binding constraint | **[D]** | | |
| Firmware build, per SKU | ImageBuilder download + `build-firmware.sh` | **[M]** §3.2 | | |
| Staged payload | 5.81 MiB of 35 MiB | **[M]** §2.2 | | |
| Per-unit state storage | 3.09 MiB measured for a real deployment; **no long-run curve [X]** | **[M]** §2.2 | | |
| Backup target | **not working — §5.3. You supply** | | | |
| Certificate | Let's Encrypt, 0 | **[M]** §6.2 | 0 | 0 |
| Operator time per unit | **zero after flash**, by design — this SKU exists to eliminate the laptop step | **[M]** §1 | | |
| Operator time per fleet update | firmware flash + re-provision; **a bare-router flash still needs a laptop** | **[M]** §6.3 | | |
| **Engineering, one-off** | `dav-tls` WAN discovery **or** LAN-only decision (§5.1) | | | |
| **Engineering, one-off** | fix the three backup defects (§5.3) | | | |

---

## 8. Open items for the appliance

| # | Item | Impact | Status |
|---|---|---|---|
| A.1 | **`dav-tls` WAN-IP discovery** | ~~Blocks any retail SKU~~ | **DONE 2026-10-01** — `--discover` with 30 s re-resolve. Logic unit-tested + a CI gate that can fail; the **real address-change rebind needs hardware** (§5.1.1). Was: open, decide first |
| A.1b | Verify against a **real DHCP lease** | The address-change path is measured (§18.31: 30 s rebound, self-recovery, survives a gap). What a real unit adds is the DHCP server itself, plus procd/flash | **needs a flashed unit** — lower risk than it was |
| A.2 | **Repair appliance backup** — three independent defects | **The appliance has no working backup.** Cannot be sold with data claims | **not fixed — now the top blocker** |
| A.3 | Restore on a second unit (plan row 49) | The one claim a VM genuinely closes. Needs two flashed units or two booted guests | needs hardware |
| A.4 | Real `sysupgrade` overlay-space rehearsal | 48 MiB loopback ext4 gets the refusal path reachable; still not f2fs, not flash | not done |
| A.5 | Measure `dav-tls` RSS and set a limit | Currently unbounded; runtime threads = host CPU count | not measured |
| A.6 | In-process certificate reload | Renewal is currently a visible TLS interruption | not built |
| A.7 | Long-run storage curve on the overlay | The overlay is the binding constraint; there is no size curve | not measured |
| A.8 | Decide: product or reference image? | A product needs a support policy, a diagnostics story and an update channel. A demo needs almost none of that | open question |
| A.9 | Watch for a second `crond` daemon | Every cron job currently double-runs. Benign, confusing | pre-existing |

---

## 9. Recommendation

**The WAN-address blocker is gone** (A.1, done 2026-10-01 — §5.1.1). The unit
serves a DHCP-assigned WAN, re-resolves every 30 seconds, and rebinds when the
lease moves. That question no longer gates a quote.

**Do not quote an appliance SKU until A.2 is fixed.** The backup path does not work
on the appliance today, for three independent reasons (§5.3), and an appliance
sold to a customer is holding their calendar and contacts. That is now the top
blocker, and it is engineering rather than a decision — which is the good kind,
because it can be scoped and estimated.

**A.1 is done and measured.** A listener was moved to a new address underneath a
live process and followed it in 30 s, survived the address disappearing entirely,
and recovered by itself. What remains for hardware (A.1b) is a real DHCP lease
rather than the address-change path — still worth doing, materially smaller risk
than it was yesterday.

**Do not quote data retention on an appliance until A.2 is fixed.** The backup path
does not work. The verification discipline around it is excellent — the plans say
so directly, and the CI gate that catches a config-clobbering postinst is a model
of the right thing — which makes it clear this is a known gap rather than a
surprise.

Once those two are closed: a flashed appliance is the **cheapest per-user
deployment in the options**, and its defining characteristic is that operator time
after the flash is **zero by design**. A host of, say, 200 units has no per-unit
maintenance burden — one firmware image goes out, units re-provision themselves,
and the 5-minute watchdog handles hung-but-alive processes.

The reference hardware is 1 GB of RAM and 98 MB of overlay. Before you commit to a
SKU, check what real customer data does to that overlay: a bulk import drove free
space to **2.6 MB** on the reference unit, below the 5 MB floor `sysupgrade`
requires. That number, not the binary size, is the appliance's real limit.

---

*Measured and derived figures carry in-code or in-plan references in §2-§6.
Planning placeholders and missing measurements are marked and collected in §8.
No price appears in this document; §7 is the worksheet.*