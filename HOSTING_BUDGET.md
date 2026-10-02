# Omnical — hosting budget

**Purpose:** a document to hand to a business that wants to run Omnical on their
own infrastructure, so they can size it, cost it, and decide.

**Subject:** Omnical — a CalDAV (calendar + tasks) / CardDAV (contacts) server.
A fork of [RustiCal](https://github.com/lennart-k/rustical) v0.16.1, AGPL-3.0-or-later.
Multi-tenant, one SQLite file per tenant. Self-hosted today; a hosted SaaS is
*not yet built*.

**Status of the numbers below:** every quantity is tagged.
**[M]** = measured on real hardware or real data.
**[D]** = derived from [M] figures, arithmetic shown.
**[P]** = planning placeholder — *not* measured, must be replaced before this budget is signed.
**[X]** = does not exist and must be produced; each one is a real cost item, listed in §8.

Read §2 before quoting anything. Six quantities a hosting business will ask about
have never been measured, and this document does not pretend otherwise.

Sources: `~/router-dav` at `260f7bd` (code), `PLAN.md` / `PLAN_DEPLOYMENTS.md` (plans).
Line references are given inline. No currency is used in the resource sections —
§7 is the worksheet you fill in.

---

## 1. What you would be running

One static binary serving CalDAV and CardDAV over HTTPS. It holds calendars,
contacts, tasks and passwords for real people. Behind it, per tenant, is a SQLite
database and nothing else.

Two processes, or one plus a proxy:

| Process | Role | Size **[M]** |
|---|---|---|
| `rustical` | the server. All the DAV, storage, auth, scheduling logic | **5,466,272 B** (5.21 MiB, aarch64-musl, UPX-packed) |
| `dav-tls` | optional. TLS termination, ALPN `http/1.1`, byte-splice after handshake. Skipped if your reverse proxy terminates TLS. Resolves the WAN interface at runtime (`--discover`) so a DHCP-assigned address works | **620,884 B** (0.61 MiB) |
| **Total payload** | both binaries + init scripts + config + watchdog | **6,097,363 B = 5.81 MiB** |

Binary size is gated at **35 MiB in CI on every push** and the current payload
uses 16.6% of that (`scripts/build-firmware.sh`, `build.yml:107-119`). Release
profile is `opt-level = "z"`, `lto`, `codegen-units = 1`, `strip`, which trades
roughly 5-15% throughput for size (`PLAN_BINARY_SIZE.md:34`).

Dependencies worth knowing: 633 transitive crates, and **three vendored C
libraries** statically linked — SQLite, OpenSSL, aws-lc (`rustical/Cargo.toml:131,172,149-152`).
Builds reproducibly on Rust 1.97.0, edition 2024.

What the fork adds over upstream, all of which a hosting business is implicitly
taking on: RFC 6638 scheduling (iMIP + RSVP), public token-URL export feeds,
invite-gated registration, a first-run setup wizard, and in-binary
backup/restore. Also multi-tenancy, quota columns, an admin panel and an AGPL
source-offer page — the last four are **not yet complete in the hosted sense**
(§8).

---

## 2. The honest summary, before any numbers

Three things to know before you price this:

**It is genuinely small.** One process, tens of megabytes of resident memory,
megabytes of disk per organisation. Not a distributed system. **[M]**

**It is genuinely single-writer.** Storage is SQLite, one file per tenant, and
there is no Postgres anywhere in the codebase — `DataStoreConfig` is a
single-variant enum (`PLAN_DEPLOYMENTS.md:117`). One process per machine, or you
shard tenants across processes. There is no clustering story and one is explicitly
a non-goal (`PLAN_DEPLOYMENTS.md:23-24`).

**It has never been load-tested to a throughput figure.** This is the single most
important line in the document. There is **no requests/second number, no sync
latency number, and no soak test anywhere in the repository**. The one benchmark
present (`store_sqlite/benches/insert_calendar_object.rs`) has never had its
results recorded. The latency figures that do exist come from a load harness that
states in its own module docs that it "cannot resolve" the gate it was written to
check, because three runs of identical code gave single-tenant baselines of 10 ms,
14 ms and 19 ms (`rustical/tests/load_measure.rs:54-65`). Anyone quoting a
capacity number from this project is quoting something that has not been measured.

**What that means for you as a host:** you can size the hardware today with
confidence, because memory and disk behaviour are measured and flat. You cannot
size your *service commitment* — how many organisations per node, what uptime,
what throughput — without commissioning the measurements in §8. This document
gives you the former and refuses to fake the latter.

---

## 3. Measured resource envelope

Everything in this section is **[M]**. Sources are in-code or in the plans.

### 3.1 Memory

| Condition | Resident set size | Source |
|---|---|---|
| 200 tenants, 16 cached tenant slots | 73 MiB | `tests/cache_decision.rs`, table reproduced in `load_measure.rs:347-353` |
| 200 tenants, 64 slots (shipped default) | **74 MiB** | same |
| 200 tenants, 256 slots | 74 MiB | same |
| Deployed single-tenant router, live | **21 MB** | `PLAN.md:3961-3962` |

Two findings here are worth more than the numbers:

**Memory does not scale with tenant count, until it suddenly does.** RSS is
*identical* at 16 slots and at 256 slots (73 vs 74 MiB). The cache holds only what
is genuinely resident, so `max_cached_tenants` is a ceiling and not an
allocation — raising it is free until the pools are real. The real constraint is
not the tenant count: it is that the bound must be at least the *concurrent
working set*. 32 slots thrash with 50 concurrent clients (109 ms p99); 64 do not
(`PLAN_DEPLOYMENTS.md:917-936`).

**But the marginal cost is real, and it is the number to budget from.**
**0.93 MiB per resident tenant pool.** If every tenant is warm simultaneously:

| Tenants all resident | Projected pool memory **[D]** |
|---|---|
| 25 | ~23 MiB |
| 100 | ~93 MiB |
| 200 | ~186 MiB |

**[D]** = 0.93 MiB × tenant count. The plan's own figure for 200 warm tenants is
~190 MiB (`PLAN_DEPLOYMENTS.md:934-936`), which is where the 0.93 comes from.
On the appliance channel that is most of the 256 MiB `procd` rlimit — which is
why the appliance is sized for one tenant and the server channel is not.

**Sizing rule [D]:** 128 MiB base + 0.93 MiB × (peak simultaneously active
tenants), then add OS, reverse proxy and TLS buffers. Do **not** size on tenant
count. A node with 500 tenants on it and 20 syncing at once needs 128 + 19 ≈
147 MiB, not 590 MiB.

### 3.2 Storage

| Quantity | Value | Source |
|---|---|---|
| Live production database | **3,239,936 B (3.09 MiB)** | `rotation-drills/before/manifest.json`, `integrity_check: ok` |
| Contents of that database | 31 principals, 34 calendars, 216 events, 209 contacts, 19 collection shares, 42 scheduling objects, 15 invites, 10 subscriptions | same |
| Compressed nightly backup artefact | **1,752,142 – 1,791,621 B** (~1.71 MiB) | `PLAN.md:1623`, `PLAN.md:1710-1711` |
| Compression ratio on that data | ~2:1 (3.09 MiB → 1.71 MiB) | derived |
| Backup growth over 10 days | ~5 KB/day ≈ **1.8 MB/year** | `~/backups/omnical/` timestamps |
| 30-day backup retention, steady state | **~50 MB** | 30 × 1.71 MiB |
| Staged firmware payload | 6,097,363 B = **5.81 MiB of a 35 MiB budget** | `build-firmware.sh --staged-only` |
| SQLite WAL, steady state | 0 B to 45 KB after checkpoint | `PLAN.md:1629,3316` |
| SQLite WAL, peak observed | **7 MB** during a bulk import | `PLAN.md:1042` |

**Per-organisation storage is not measured [X].** The 3.09 MiB figure is for one
deployment with 31 principals, not a clean per-tenant unit. Two facts make
extrapolating it risky: the changelog tables are large relative to live data
(`calendarobjectchangelog` holds 937 rows for 216 live events — roughly 4.3
changelog rows per live object), and the database grew **24× in six days** during
initial onboarding (151 KB → 3.5 MB, `PLAN.md:1042`), which is onboarding
bloat, not steady state. A long-run size curve does not exist.

**[P] Provisioning assumption until §8.1 is done: 50 MB per organisation** of
live database, allowing for churn and tombstones. At 500 organisations that is
24.4 GB, which is a rounding error against any realistic disk. The number is a
placeholder, and it is the one storage figure most likely to be revised.

### 3.3 CPU and concurrency

The one concurrency measurement that exists: **200 tenants, 50 concurrent
clients, 1,000 requests total** (`load_measure.rs:41-45`).

| `max_cached_tenants` | steady-state p99 | RSS | churn p99 (cold tenant) |
|---|---|---|---|
| 16 | 119 ms | 73 MiB | 139 ms |
| 32 | 109 ms | 77 MiB | 189 ms |
| **64 (shipped default)** | **12 ms (1.20× baseline)** | **74 MiB** | 102 ms |
| ~~128~~ | ~~5 ms~~ — *re-measured at 120 ms, see below* | 74 MiB | 243 ms |
| 256 | 12 ms (1.20× baseline) | 74 MiB | 4 ms |

**The 128-slot row is noise and should not be quoted.** Re-running the identical
test on the identical host on 2026-10-01, with an unrelated build loading the
machine, put that row at **120 ms instead of 5 ms — a 24× swing on no code
change.** The test's own module docs predicted exactly this: three runs of the
same code gave single-tenant baselines of 10 ms, 14 ms and 19 ms, and one run
failed a 2× gate on load alone. **Only the RSS column is a property of the
software**, and it held under load exactly as it held without it (75–86 MiB flat
from 16 to 256 slots). Budget from the RSS column and the 0.93 MiB-per-pool
figure; do not budget from a latency figure in this table.

Two warnings travel with this table and must travel with it:

**The load harness has no TLS, no edge, no network and no disk contention.** Its
own header calls these numbers "a lower bound on production latency"
(`load_measure.rs:1-31`).

**A previous, harsher reading of this same test was wrong, and was retracted.**
It reported that the shipped default of 64 slots *failed* at 200 tenants (p99 6.7×
baseline). The cause was the harness's own bug: the warmup touched all 200 tenants,
then spread clients over tenants 0-49, so every tenant the measurement then
touched had just been evicted — 50 cold requests out of 1,000, which is exactly
where p99 lives. With that fixed, the default passes (`load_measure.rs:332-353`,
`PLAN_DEPLOYMENTS.md §18.25`). Do not quote the 6.7× figure.

**The real cost that survives: a cold tenant pays ~102 ms.** Stage-by-stage
breakdown of a tenant cache miss (`tests/miss_cost.rs`, medians, 40 samples):

| Stage | Time |
|---|---|
| `ensure_tenant_store_dir` | 0.01 ms |
| `get_store_bundle` (pool + migrate) | 2.10 ms |
| config overrides merge | 0.00 ms |
| `make_app_for` (the whole axum router) | 1.04 ms |
| **warm total** | **3.7 ms** |
| first iteration, cold file | 13.8 ms |
| **real miss, loaded page cache** | **~102 ms** |

So a miss is **I/O, not CPU** — about 88 ms of the 102 ms is re-reading the
tenant's SQLite file (`PLAN_DEPLOYMENTS.md:3560-3563`). Two optimisations were
tried and reverted: `posix_fadvise(WILLNEED)` made it *worse* (102 → 144 ms), and
a readahead prototype changed nothing outside the noise floor.

**[D] Operational consequence for a host:** keep the data directory on a disk
with good random-read latency, and warm tenants you care about. A storage tier
with high IOPS latency will hit that 102 ms on every cold request; a CPU
allocation will not fix it. This is the opposite of the usual intuition and it is
worth knowing before you pick a disk tier.

No CPU-time measurement exists at all **[X]**. Every process is async Rust on
Tokio with SQLite as the only blocking work, so the expectation is that this is
IO-bound and not CPU-bound — but that is an expectation, not a measurement.

### 3.4 Network

| Quantity | Value | Source |
|---|---|---|
| Public export feed, 196 VEVENTs | **229,201 B** ≈ 1,169 B/event | `PLAN.md:3332-3333` |
| Maximum request body | **32 MB** (`payload_limit_mb`) | `PLAN.md:634` |
| dav-tls connection cap | 1,024 concurrent; over the cap, connections are **dropped**, not queued | `dav-tls/src/main.rs:37,381-384` |
| dav-tls drain timeout on shutdown | 5 s | `dav-tls/src/main.rs:38` |
| Certificate | Let's Encrypt, ~90-day validity, renewed ~60 days out | `PLAN.md:732-734` |

**Bytes-per-sync does not exist [X].** There is no per-sync transfer measurement
anywhere, so **egress cannot be budgeted from this document.** The export feed
figure is the only measured payload and it is a different thing (an unauthenticated
`.ics` download, not a sync). Two ceiling-type numbers exist and should shape your
edge config: the 32 MB body limit, and dav-tls's habit of dropping rather than
queueing past 1,024 connections.

### 3.5 Availability characteristics, as built

| Property | Value |
|---|---|
| Restart policy | `Restart=on-failure`, `RestartSec=5s` (`packaging/native/omnical.service:36-37`) |
| Deploy interruption | **~10 s** stop → swap → start, measured live (`PLAN.md:1796`) |
| Session store, single-node | **in memory** — every restart logs every user out (`PLAN_DEPLOYMENTS.md:1569`) |
| Health check | `/ping`, unauthenticated; systemd `health` subcommand |
| Data durability | SQLite, WAL mode, `busy_timeout` 5 s. **No `synchronous` pragma set** — SQLite's WAL-mode default applies |
| Clusterability | **None.** In-memory sessions cannot be shared across replicas (`PLAN_DEPLOYMENTS.md:139-148`) |
| Admin panel | **k=1 only**, enforced as a startup refusal. A tenant created on instance 1 does not resolve on instance 2 (`PLAN_DEPLOYMENTS.md:698-712`) |

The last three are the operational constraints that matter most for a hosting
commitment, and none of them are resource costs. They are in §5 and §8.

---

## 4. Deployment options

### 4.1 Option 1 — single tenant on your existing VM or container platform

One organisation. One SQLite file. No tenancy layer.

- **Baseline hardware:** 2 vCPU, 1 GB RAM, 20 GB disk. Comfortable for
  single-tenant to low-hundreds-of-users **[P]** — sized from the 74 MiB measured
  envelope plus OS and proxy overhead, not from a load test.
- **Install:** one command, or a Compose file. `packaging/native/install.sh`
  verifies the artefact, installs the binary and unit, runs the setup wizard as
  the service user, starts the service and health-gates it. It is the supported
  upgrade path and will not clobber your config, database or administrator. There
  is also `compose.omnical.yml`, which is a packaging of the same install — both
  drive the identical `rustical setup` command, and CI fails the build if the two
  ever disagree.
- **TLS is yours to provide.** The server speaks plain HTTP and says so. Put a
  reverse proxy in front, or run `dav-tls`. The compose file ships a commented
  Caddy service for this reason.
- **Hardening is already done.** The shipped unit carries `ProtectSystem=strict`,
  `NoNewPrivileges`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, ten further
  `Protect*`/`Restrict*` directives, and `ReadWritePaths` restricted to the data
  directory. The config file — which holds mail passwords in cleartext — is
  read-only to the service. **[M]**, with the caveat in §8.6.

This is the lowest-cost way to run Omnical, and the only option with no open
architecture work in it.

### 4.2 Option 2 — multi-tenant, resold to many organisations

The control plane is built (tenants, quotas, audit, admin panel, usage snapshots,
suspended-tenant enforcement). The hosted packaging is not.

- **Baseline hardware, per node:** 4 vCPU, 2 GB RAM, 100 GB disk. Holds the
  200-tenant measured envelope with room for the proxy and OS **[P]**.
- **Ceiling:** 200 tenants per process, measured. Past that, **shard by tenant
  across processes** with a hostname→process map at the load balancer. That is the
  documented escape hatch and it is a real, workable answer — but note that the
  admin panel does not survive it (k=1).
- **Sessions must move to Redis** if you ever run more than one instance. Today
  they are in-process, so they cannot be shared across replicas.
- **What exists:** per-tenant isolation *by construction* — a tenant is a resolved
  router instance holding only that tenant's SQLite pool, so there is no
  `WHERE tenant_id = ?` to forget. Five cross-tenant isolation gates are green.
- **What does not:** quota **enforcement** (columns exist; the wave that enforces
  them is not built), a published container image, a release workflow, and the
  hosted distribution story. See §8.

### 4.3 Option 3 — appliance (see the separate appliance brief)

A flashed firmware image on router-class hardware that re-provisions itself.
Reference unit: ThinkPenguin TPE-R1400, RK3328, 4× Cortex-A53, 1 GB RAM, 98.4 MB
f2fs overlay **[M]**. Deployed and running. Full requirements, the WAN-IP defect,
and the firmware lifecycle are in the companion document,
**`HOSTING_BUDGET_APPLIANCE.md`**.

---

## 5. What is not in these numbers

A hosting commitment is mostly made of things this document cannot tell you. List
plainly so nobody discovers them later.

**Hard ceilings, not bugs.**

| Ceiling | Consequence |
|---|---|
| SQLite only. One file per tenant, one writer per file | No horizontal write scale. The answer to growth is more processes, never a shared database |
| One process per machine in the current design | In-memory sessions cannot cross processes; the admin panel refuses to start at k>1 |
| 200 tenants per process | Shard by tenant at the load balancer to go further |
| Sessions in memory | **Every restart and every deploy logs every user out.** Not configurable away in this build |
| dav-tls drops connections past 1,024 | A traffic spike is refused, not queued. Behind a reverse proxy this component is usually not in the path |
| Single-region data residency | Where a tenant's file lives is decided by your deployment. Moving to per-region residency later is a data-model change, and the plans call it "expensive to change" |

**Operational reality as currently built.**

Backups are **unencrypted, single-sited, and have no offsite copy**. The
30-day retention is a hardcoded constant. The archive contains the configuration
directory — which holds SMTP and IMAP passwords in cleartext plus the RSVP
signing secret — inside a plain gzip tar. A 3-2-1 policy is not implemented; you
would be adding it.

Backup integrity is genuinely good, and worth the credit: every archive carries a
`manifest.json` with a SHA-256 per entry and a row count per table, and
`rustical restore` verifies before replacing anything. `backup-all` **exits
non-zero if any single tenant fails**, so a run that backed up 3 of 200 tenants
cannot report success. That is the right design.

**There is no monitoring, alerting, or metrics pipeline.** There is a 5-minute
health watchdog on the appliance channel, which restarts a hung-but-alive process
(`procd` respawn only covers processes that exited). On the server channel you
get systemd's restart policy and `/ping`, and nothing else. OpenTelemetry is
compiled in behind a `debug` feature flag but is not enabled in any shipped build.

**There is no secret store.** Development deployments use a `pass` store on a
laptop. The control-plane database holds every tenant's SMTP password and the
platform administrator's hash, and it is created mode `0600` specifically so it is
not world-readable — but it is a file on a disk. A production deployment needs
Doppler, Vault or SSM, which is your cost, not ours.

**Support commitment is undefined.** The plans flag this as an open question and
their own default is "security fixes on the latest release only, best-effort",
with the note that self-distribution without a policy "quietly becomes an infinite
support obligation" (`PLAN_DEPLOYMENTS.md:1190-1196`). Agree this in writing
before you sell hosting on it.

**Credentials on the current live deployment need rotating.** 69 application
tokens across 11 accounts and a live TLS private key are still valid; the rotation
runbook is marked NOT DONE, deferred 2026-09-28. This is our debt, not yours, and
it is fixed before any customer data goes anywhere. Flagged in §8.

---

## 6. Licensing — read this before costing anything

Omnical is **AGPL-3.0-or-later** (`rustical/Cargo.toml:11`,
`dav-tls/Cargo.toml:6`). The licence decision was deliberate: keep AGPL, no
commercial licence, no relicensing.

**No licence fee. No per-seat cost. No procurement conversation with a vendor.**

What you do owe, because you are offering the service over a network:

1. **Every user interacting with it over a network must be offered the complete
   corresponding source** of the version they are talking to. The fork's
   `/frontend/source` page implements this: it names the running version, both
   commit SHAs, and links a tarball. **It refuses to offer a tarball it cannot
   match to a commit** — no commit or a dirty tree means the link is withheld and
   the page says that is not acceptable under §13. CI verifies that an anonymous
   client can reach the page, download the tarball, and that the tarball builds.
2. **You cannot run a modified private version.** Any fix made for a customer is
   pushed to the public fork. For a hosting business this is usually a *benefit* —
   every customer gets the fix — but it is a constraint, and it means there is no
   such thing as a customer-specific fork or a proprietary extension layer.
3. **Keeping the repositories public is a licence obligation, not a preference.**
   Practical consequence: secret material must never reach the repository. CI
   enforces this (gitleaks plus trufflehog, full-history scan, and a 500 MB cap on
   `.git`).
4. **You may charge for hosting it.** AGPL does not restrict charging for service.

Vendored third-party code is Apache-2.0 / ISC / MIT compatible
(OpenSSL Apache-2.0, aws-lc Apache-2.0/ISC). One paid-licence wall was hit and
worked around: the `gitleaks-action` wrapper requires a paid licence key for
organisations, so CI uses the `gitleaks` CLI instead — still free, AGPL-free
(`PLAN_DEPLOYMENTS.md:2590`).

---

## 7. Cost worksheet

Fill in your own rates. Quantities in the left column are measured or derived as
marked in §3.

### 7.1 Option 1 — single tenant

| Item | Quantity | Source | Your rate | Monthly |
|---|---|---|---|---|
| vCPU | 2 | **[P]** §4.1 | | |
| RAM | 1 GB | **[P]** §4.1 | | |
| Block storage | 20 GB | **[P]** §4.1 | | |
| Backup storage, 30-day | ~50 MB measured; allow 1 GB | **[M]** §3.2 | | |
| Offsite backup copy | **not built — you supply** | §5 | | |
| Egress | **[X]** unmeasured, §8.4 | | | |
| TLS certificates | Let's Encrypt, 0 | **[M]** §3.4 | 0 | 0 |
| Operator time | ~10 s per deploy, 30 s nightly backup | **[M]** §3.5 | | |

### 7.2 Option 2 — multi-tenant, per node

| Item | Quantity | Source | Your rate | Monthly |
|---|---|---|---|---|
| vCPU | 4 | **[P]** §4.2 | | |
| RAM | 2 GB base, +0.93 MiB per peak-concurrent tenant | **[M]/[D]** §3.1 | | |
| Block storage | 100 GB node + ~50 MB per organisation | **[M]/[P]** §3.2 | | |
| Session store (Redis) | required above one instance | **[M]** §3.5 | | |
| Edge / load balancer | required; TLS 1.2/1.3, 32 MB body cap | **[M]** §3.4 | | |
| Backup storage | per-tenant archives; measure | **[M]** §5 | | |
| Secret store | required; not built | §5 | | |
| Egress | **[X]** unmeasured, §8.4 | | | |
| Operator time | see §7.4 — this is the big one | | | |

### 7.3 Option 3 — appliance, per unit

Full detail in the appliance brief. Summary:

| Item | Quantity | Source |
|---|---|---|
| Hardware | router-class, 1 GB RAM, ≥98 MB overlay, aarch64 | **[M]** §4.3 |
| Payload | 5.81 MiB of a 35 MiB overlay budget | **[M]** §3.2 |
| Firmware build | ImageBuilder for OpenWrt/libreCMC — a download, no cached SDK | **[M]** |
| Operator time per unit | the appliance exists to remove the laptop-and-SSH step; that is its whole point | |

### 7.4 Labour — the largest line, and the easiest to forget

| Task | Frequency | Measured cost |
|---|---|---|
| Deploy / binary swap | per release | **~10 s**, automated |
| Nightly backup | daily 02:30 | unattended; **fails loudly**, all-or-nothing |
| Health check | every 5 min on appliance; systemd on server | unattended |
| Certificate renewal | ~60-day cycle | **not automatic in-service.** `dav-tls` has no SIGHUP handler and no file watcher — certificates reload only on service restart, which is a visible few-seconds TLS interruption (`dav-tls/src/main.rs:6-8`) |
| Firmware flash recovery | per flash | appliance re-provisions itself via package-owned files; a *bare* router flash still needs a laptop and SSH |
| Session invalidation | per deploy | **every user re-authenticates.** Unavoidable in this build |
| Credential rotation | per incident or policy cycle | a written 3-part runbook exists, drills exist, and it is currently **NOT DONE** |
| Incident triage | per incident | **no metrics pipeline exists.** You are reading logs |

---

## 8. Open items — the real cost of doing this properly

Six measurements and four build items are absent. Each is cheap relative to what
it de-risks, and each is something a hosting business is entitled to know about
before committing.

| # | Item | Why it matters | Status |
|---|---|---|---|
| 8.1 | **Per-organisation storage curve** (30/60/90-day DB size) | The 50 MB/org planning figure in §3.2 is a placeholder. Disk is cheap, but a 10× miss on a 5,000-org node is 244 GB → 2.4 TB | not measured |
| 8.2 | **Throughput: requests/sec and sync latency under sustained load, with TLS** | The largest gap. You cannot write a capacity or SLA commitment without it. **Partial progress 2026-10-01:** the one benchmark in the repo *did not build* (fixed; `PLAN_DEPLOYMENTS.md §18.30`) and now records a first, **load-affected** figure — 1,000 events written in **91.9 ms batched** (10,900/s) vs **306.1 ms singly** (3,270/s). That is a *storage-write* rate, not a request rate, and it was measured under an unrelated build. Re-run on a quiet machine | **partially measured** |
| 8.3 | **Soak test** — hours, not 1,000 requests | Memory leaks and cache-thrash behaviour only appear over time. The existing harness is a single-shot, 1,000-request test marked `#[ignore]` | not measured |
| 8.4 | **Bytes per sync, and a monthly egress model** | Egress is typically the largest line item at scale and is completely unmodelled here | not measured |
| 8.5 | **Quota enforcement** | Columns exist (`quota_principals`, `quota_calendars`, `quota_megabytes`, all nullable = unlimited). Without enforcement a resold service has no capacity control. Plans estimate "a month of work or a day" depending on the pricing model chosen | not built |
| 8.6 | **systemd hardening verified on a real host** | The unit sets 14 hardening directives, and its own comment says CI cannot exercise it — "the first `systemctl start` on a real host is the actual test". First install on your platform **is** that test | unverified |
| 8.7 | **Published container image + release workflow** | `release.yml` and `docker.yml` do not exist. `install.sh --base-url` has no default download URL. There is no image to pull; you would build from source | not built |
| 8.8 | **Memory/CPU limits in the systemd unit** | The unit sets **no** `MemoryMax`, **no** `CPUQuota`, **no** `LimitNOFILE`, and **no** capability bounding set. The router init script caps memory at 256 MiB; the server unit caps nothing | gap |
| 8.9 | **Backup encryption + offsite copy** | Archives contain cleartext SMTP/IMAP passwords and the RSVP secret. Not encrypted, not offsite, 30-day retention hardcoded | gap |
| 8.10 | **Live credential rotation** | 69 app tokens across 11 accounts, plus a live TLS key. Runbook exists, drills exist and are proven able to fail, execution is deferred | **not done** |
| 8.11 | **dav-tls has no memory limit** and runs a multi-threaded Tokio runtime sized to host CPU count, uncapped | Its only bound is 1,024 connections | gap |
| 8.12 | **Support policy** | Undefined; the risk is an open-ended support obligation | open question |
| 8.13 | ~~`dav-tls` WAN-IP discovery~~ | Was the appliance's one blocker. **Done 2026-10-01** and measured on a real address change: 30 s rebound, survives the address disappearing, self-recovers (`PLAN_DEPLOYMENTS.md §18.30-31`) | **closed** |

Items 8.1–8.4 are measurements we can produce. 8.5–8.12 are engineering work.
**8.13 is closed** — the WAN-IP defect that was the appliance's one hard blocker
was fixed on 2026-10-01 and then *measured*: a listener moved to a new address
follows it in 30 s, survives the address disappearing, and recovers by itself
(`PLAN_DEPLOYMENTS.md §18.30` and `§18.31`).

**The one measurement still worth commissioning first is 8.2** (throughput with
TLS, on a quiet machine). The storage-write bench that had never been run now
builds and has a first, load-affected figure; there is still no requests/second
number, and that is the figure a capacity commitment needs.

**None of them blocks Option 1.** Options 2 and 3 need more of them closed before
they are a service rather than a deployment.

---

## 9. Recommendation

**Start with Option 1.** Single tenant, your existing VM or container platform,
reverse proxy in front, 2 vCPU and 1 GB. It is the cheapest path, the fully
exercised path, and the one with no open architecture work. Everything in §5 that
is scary — sessions in memory, no clustering, the admin panel's k=1 refusal —
disappears when there is exactly one tenant, because none of those ceilings are
reached.

Size it with confidence. Memory and disk behaviour are measured and flat across a
200-tenant sweep. What you cannot do yet is sell a throughput or capacity number,
because none has been measured. Commission §8.2 and §8.4 before you make that
promise to anyone — they are two days of work, and they are the difference between
hosting software and running a service.

Then decide deliberately whether Option 2 or Option 3 is worth the open items.

---

*Figures tagged [M] are measured on real hardware or real production data, with
in-code or in-plan references given in §3. [D] figures show their arithmetic.
[P] figures are planning placeholders and are named as such. [X] marks quantities
that do not exist and are listed in §8. No price appears in this document by
design; §7 is the worksheet.*