# A virtual appliance, for the tests that need one

**Written 2026-09-30, right after the item 14 deploy. Read before building.**

Item 14 is deployed (`a3701c2d`) and rows 49 and 51 are still open. This file
is about closing row 49 properly, and about being precise about which half of
row 51 a virtual machine **cannot** close no matter how good it is.

---

## What a VM closes, and what it does not

This is the whole plan, stated first because everything else follows from it.

| Row | Claim | A VM |
|---|---|---|
| **49** | backup → restore **on a second unit** | **Closes it.** Two independent guests, a real archive produced by one and restored by the other, no shared state. This is the row. |
| 51 (a) | stripped binary ≤ 35 MiB of a 35 MiB overlay | Already enforced every build. Unchanged. |
| 51 (b) | `df -k /` on the appliance overlay | **Cannot close it. Do not try.** |

**Why 51(b) is out of reach, precisely.** Row 51 is about an f2fs overlay with
~16 MiB free, and the failure it exists to catch is `sysupgrade` running out of
space on a *flash-backed overlay*. A guest's root is either a qcow2 file on the
host's filesystem or an in-memory initramfs. `statvfs("/")` inside the guest
therefore reports the **host's** free space — currently ~200 GiB — so every
assertion of the form "free space < 35 MiB, therefore refuse" is *unreachable*:
you cannot make the host run out of disk to prove the guest refuses.

A VM can test the panel's **logic** — that an unmeasured filesystem is not
treated as room, that `has_room` requires `2 × size`, that the button refuses.
`tests/appliance_panel.rs` already does all three with a fake `statvfs` path. A
VM adds a real `statvfs` call, which is worth having, and it adds **nothing** to
the row. Say so in the row's cell rather than ticking it.

The honest replacement for 51(b) is a **loopback-backed ext4 image sized to the
overlay budget** — `losetup` a 48 MiB file, mkfs, mount it, and point
`[appliance] overlay_mount` at it. That makes the *filesystem genuinely small*,
so the refusal path is reachable for real. It is still not f2fs and still not
flash, so it is a **rehearsal** of 51(b), not the row. Build it if it is cheap;
do not report it as the row.

---

## The one decision that shapes everything: aarch64/TCG vs x86_64/KVM

Verified on this host, 2026-09-30:

```
$ uname -m
x86_64
$ qemu-system-aarch64 -accel help
Accelerators supported in QEMU binary:
tcg                      # <- and nothing else
$ qemu-system-x86_64 -accel help
Accelerators supported in QEMU binary:
tcg mshv kvm             # <- kvm
```

**KVM cannot accelerate an aarch64 guest on an x86_64 host.** That is a property
of KVM, not a packaging problem: the guest and the host must share an
architecture. So there are two guests, and they are not substitutes.

### Guest A — aarch64, TCG. **Fidelity.**
- Runs the **exact deployed binary**: `out/rustical`, aarch64-musl, UPX-packed.
- Same architecture, same libc, same static build as the router.
- **Emulated.** Expect roughly 10–30× slowdown. An OpenWrt boot under TCG is
  tens of seconds, not seconds. `/dev/kvm` is present and in use for nothing here.
- Machine type: `virt` with an explicit `-cpu cortex-a72` and **512 MiB RAM**.
  OpenWrt `armsr/armv8` images are built for exactly this board.

### Guest B — x86_64, KVM. **Speed.**
- **Fast** — a boot in a second or two, so a real test suite is practical in CI.
- Needs its own binaries: `scripts/build-rust.sh x86_64-unknown-linux-gnu`
  already exists and is described in-tree as the "host build for smoke tests".
  That is a *host* build, not an OpenWrt one — see the trap below.
- The `virtio` disk and NIC are what make it fast; the appliance's f2fs overlay
  behaviour is not reproduced at all.

### Recommendation

**Build both, A first.** A exists to answer "would this work on the router",
which is the only question a laptop can answer that the current test suite
cannot. B exists to make the panel's tests runnable on every commit, which is a
different and cheaper goal. Conflating them is how you end up with a fast test
suite that proves nothing about the appliance.

---

## Images: where they come from

Both targets were confirmed present on `downloads.openwrt.org` on 2026-09-30,
latest release **25.12.5**:

```
releases/25.12.5/targets/armsr/armv8/
  openwrt-25.12.5-armsr-armv8-generic-ext4-combined.img.gz
  openwrt-25.12.5-armsr-armv8-generic-squashfs-combined.img.gz
releases/25.12.5/targets/x86/64/
  openwrt-25.12.5-x86-64-generic-ext4-combined.img.gz
```

**There is no cached SDK or ImageBuilder on this machine** — `find /` for
`*openwrt*` / `*ImageBuilder*` returns nothing. Every image is a download.
Cache them under `router-dav/out/` (already gitignored) and checksum them.

`ext4-combined` over `squashfs-combined`, deliberately: **`squashfs` is
read-only**, so a rootfs on it cannot be written to, and the payload
(`postinst`, `/etc/sysupgrade.conf`, the watchdog cron line) is precisely a set
of *writes to files this package does not own*. That is what
`packaging/firmware/omnical/Makefile` says in its own header comment, and it is
the reason `ext4` is the right choice for a test guest. `squashfs` is fine for
the `sysupgrade` **image** a real unit flashes — different job.

---

## Reuse, not reinvention

Four things in `router-dav/` already do most of this work. Read them before
writing anything.

| Script | What it already does | What the VM needs from it |
|---|---|---|
| `scripts/build-rust.sh` | cross-compiles + UPX + the 35 MiB gate | nothing; it already emits `aarch64-unknown-linux-musl` and `x86_64-unknown-linux-gnu` |
| `scripts/build-firmware.sh` | stages the payload into `packaging/firmware/omnical/files/`, checks the overlay budget | its `--staged-only` mode is exactly what a VM guest needs, with no SDK |
| `scripts/firmware-gate.sh` | the row-51-adjacent structural gate: "the recipe installs this", "the postinst does that", "sysupgrade.conf gains this" | **run it against the guest's actual rootfs**, not just the staging dir |
| `deploy.sh` | renders config from `pass`, stages to `/tmp`, ENOSPC-guarded swap, starts, health-checks | the *shape* to copy for guest provisioning — especially the fence and the guard |

The `postinst` is the interesting one. It is written to run on a **factory
flash**, on **every sysupgrade**, and on a **re-registered unit** — and
`firmware-gate.sh:155-179` already exercises all three against a temp tree. A VM
guest is a fourth case: a rootfs that exists but has never been booted. If
`postinst` is not idempotent for that case, the guest will show it, and that is
worth finding.

---

## Steps

### 1. A single VM, booted, before anything clever

Do not start with two guests, libvirt, or a provisioning script. Get one guest
booting and reachable, by hand, and write down the exact command. Everything
after this is that command plus a second copy.

- `qemu-img convert -f qcow2 -O qcow2 <downloaded>.img.gz → guest-a.qcow2`
  (gunzip first; `qemu-img` does not read `.gz`).
- Boot A: `qemu-system-aarch64 -M virt -cpu cortex-a72 -m 512 -nographic \
   -drive if=none,file=guest-a.qcow2,format=qcow2,id=hd0 \
   -device virtio-blk-device,drive=hd0 -netdev user,id=n0 -device virtio-net-device,netdev=n0`
- First boot gives you a root shell on the console with no password. Set one.
- `passwd`, then `opkg update`.

Record the **first-boot-to-SSH** time. If it is minutes rather than seconds,
that number is the budget for every test that reboots, and it belongs in the
plan's honest-limits section rather than being discovered by the next person.

### 2. Install the payload

- `scripts/build-firmware.sh --staged-only` for the aarch64 payload.
- Copy `files/rustical`, `files/dav-tls`, `files/init.d/*`,
  `files/rustical-watchdog` into the guest at their real paths
  (`/usr/sbin/`, `/etc/init.d/`, `/usr/bin/`), `chmod 755`.
- **Do not run `postinst` by hand and call it done.** Run it and then verify the
  *observable* state it claims to produce — `/etc/sysupgrade.conf` has the two
  lines, the crontab has the watchdog, `/etc/rustical/config.toml` exists. A
  postinst that exits 0 and leaves the system half-provisioned is the exact
  failure this whole exercise is for.

### 3. Make the config reproducible without a secret store

The router's config comes from `pass` on a laptop. A CI guest has no `pass` and
must not be handed real SMTP passwords to test a panel. The right split:

- **Never** put a real credential in a VM. Use a fixture config with obviously
  fake values — and note that this makes the guest useless for testing
  *delivery*, which is fine, because delivery is `dav-tls`'s job and is already
  covered elsewhere.
- Reuse `tests/tenant_support`'s shape, not `pass`. The one thing worth
  reproducing exactly is the **tenancy** config, because that is what the
  appliance panel reads.

### 4. Row 49: two units, one archive

This is the deliverable, and it is small once step 1 works.

1. Guest A: `rustical backup --out-dir /tmp/archives` → one archive.
2. Copy it to guest B over the host (both are files on this machine).
3. Guest B: restore it, with the service **stopped** — the same constraint
   `cmd_restore` enforces, and the reason the panel stages instead.
4. Assert on **B's** data afterwards, by direct `sqlite3` row counts (the
   instrument §"the drill only works against a host whose binary has it" already
   prescribes).
5. Then: run the *panel's* staged restore against B, restart B, and check the
   report the panel would show. That exercises the whole item-14 path on real
   hardware-shaped software, which is the point.

### 5. Only then: libvirt

`virsh uri` already answers `qemu:///session`, so a user-session libvirt is
available without a system daemon. Move the two hand-typed `qemu-system-*`
commands into `packaging/vm/*.xml` domain definitions once they are known to
work. **Do not do this first** — a libvirt domain that is wrong in one XML
attribute is much harder to debug than a command line, and the command line is
what you will copy into the plan.

---

## Traps

- **TCG is not KVM.** Guest A will be slow, and a timeout tuned for KVM will
  make it look hung. Set generous timeouts and say in the test name that it is
  emulated, so nobody later "optimises" it into an aarch64-on-aarch64 runner
  assumption.
- **The host build is not an OpenWrt build.** `build-rust.sh
  x86_64-unknown-linux-gnu` produces a glibc binary for this host. It is
  *executable* on a glibc OpenWrt x86_64 guest only by luck. Use the OpenWrt
  SDK's toolchain for guest B, or accept that guest B tests the *panel* and not
  the *binary*.
- **UPX and TCG.** The deployed binary is UPX-packed for size. Under emulation
  the unpacking cost is paid on every exec and is *not* representative of the
  router, where it happens once at boot. Fine for functional tests; do not read
  a timing from guest A.
- **`squashfs` is read-only.** Already covered above; it is the single most
  likely way to waste an afternoon.
- **Two guests sharing one image file** silently corrupt both. `cp`, or
  `qemu-img create -f qcow2 -b base.qcow2` for copy-on-write.
- **The `postinst` and `deploy.sh` both write `/etc/sysupgrade.conf` and the
  crontab, and both guard against duplicating lines.** Running them twice must be
  a no-op. `firmware-gate.sh` asserts this on a temp tree; assert it on a real
  guest too, because the guest is where `/etc/crontabs` directory-mtime
  behaviour (noted in `deploy.sh`, Phase 8.2) is actually real.
- **Do not point the VM at the real router.** Two services, one control plane,
  one hostname. If a test needs a second unit, the second unit is a VM.

---

## Also found on 2026-09-30, during the item 14 deploy — three pre-existing bugs

None are from `a3701c2d`; all three are older and all three are in the backup
path, which is why they are here rather than in a bug tracker. **They are not
fixed.** The appliance is deployed and healthy; these are what is still wrong
with it.

### 1. `cmd_backup` cannot find the database on this deployment

```
$ rustical backup --out-dir /tmp/x
Error: data_store.sqlite.db_url is not a file path
       (file:/usr/local/share/rustical/db.sqlite3) — nothing to back up
```

`resolve_database_path` (`src/commands/backup.rs`) parses `db_url` with sqlx and
then refuses any filename containing a `:`. The deployed config uses the
**`file:` scheme**, which sqlx keeps in the filename. The guard is in
`ab151571` and predates this work.

Affects `backup`, `restore` and `upgrade`, all of which route through it. The
guard's *intent* — "this is not a file, there is nothing to back up" — is right;
it just cannot tell a `file:` URL from a non-file one.

Measured, not inferred (sqlx 0.9, 2026-09-30):

```
file:/usr/local/share/rustical/db.sqlite3   get_filename() = "file:/usr/local/share/rustical/db.sqlite3"
sqlite:///var/lib/rustical/db.sqlite3       get_filename() = "/var/lib/rustical/db.sqlite3"
sqlite://db.sqlite3                         get_filename() = "db.sqlite3"
```

sqlx does **not** strip a `file:` scheme — it hands the whole string back as the
"filename". So the colon the guard trips on is the scheme's own colon.

**And the other implementation of this fact is worse, not better.**
`Config::sqlite_db_path` (`src/config.rs:924`) makes the same
`get_filename()` call with **no** colon check, so for a `file:` config it
returns `"file:/usr/local/share/rustical/db.sqlite3"` as if it were a path — and
a caller would then try to open a file with a colon in its name and fail
somewhere much less obvious.

So this is not "one correct copy and one wrong copy". It is **two copies, both
wrong for `file:`, failing differently**: `resolve_database_path` refuses
loudly, `sqlite_db_path` fails silently. Neither is right. One function should
own "turn `db_url` into a filesystem path", it should strip a leading
`<scheme>:` itself rather than delegating to sqlx, and both call sites should
use it. The loud refusal is the better of the two current behaviours, so if
nothing else happens, keep the guard.

### 2. `backup-all` looks for the tenant store in the wrong place

```
$ rustical backup-all --out-dir /tmp/x
FAIL  omnical: no database at
      /usr/local/share/rustical/ff77b536-....sqlite3
```

`backup_all::tenant_store_path` builds `{data_root}/{uuid}.sqlite3`. The real
store is `{data_root}/tenants/{uuid}/db.sqlite3`, which
`TenancyConfig::tenant_db_path` gets right. This is the **exact** mistake
`NEXT_AGENT_ITEM14_PANEL.md` warns about under "Decisions taken":

> The tenant store is `data_root/tenants/{uuid}/db.sqlite3`, not
> `data_root/{slug}.sqlite3`. Copying to the latter would have put the data
> somewhere nothing reads.

`backup_all.rs:74` is a second, wrong definition of a fact that already has a
correct owner. **Delete it and call `TenancyConfig::tenant_db_path`.** One
definition, so it cannot drift.

Net effect: **the appliance currently has no working backup path at all**, which
is why the panel's Data section would list zero restore points. This is the
strongest argument for building the VM in this file — row 49 is untestable
until these two are fixed.

### 3. `nightly-backup.sh` targets the pre-tenancy path

`scripts/nightly-backup.sh:7` hardcodes
`sqlite3 /usr/local/share/rustical/db.sqlite3` — the **old** file, kept as the
migration rollback. It is currently byte-identical to the live tenant store
(verified with `cmp`), so nothing is lost today, and there is **no backup cron
installed on the router at all** (checked `/etc/crontabs/root`).

The exposure is latent, not active: the first write to the tenant store after
the migration makes the old file a frozen snapshot, and this script would then
back up a database that no longer changes. Point it at the tenant store, or
better, at `backup-all` once bug 2 is fixed.

---

## What "done" looks like

- `scripts/vm-up.sh` and `scripts/vm-down.sh`, and a `just` or `make` target,
  that take a VM from nothing to a running, provisioned appliance and back.
- `tests/appliance_e2e.rs` (or a shell harness if driving a guest from Rust is
  more trouble than it is worth — say which and why) covering **row 49**:
  archive from unit A, restore into unit B, assert B's row counts.
- `firmware-gate.sh` run against a real guest rootfs, not a temp tree.
- A row-matrix update that says, in 51(b)'s cell, that a VM rehearses it and a
  flash is what closes it.
- The three bugs above fixed, or filed with an owner. A deployed appliance with
  no working backup is a worse position than one with a stale binary.
