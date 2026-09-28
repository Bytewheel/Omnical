# Omnical — Three Deployment Models: Hosted SaaS, Self-Hosted Distribution, Hardware Appliance

**Date:** 2026-09-28
**Project:** `omnical` — RustiCal `v0.16.1` fork, branch `omnical-scheduling`,
19 commits ahead of upstream. Code at `~/router-dav`; nested fork at
`~/router-dav/rustical`; this planning repo is docs-only.
**Goal:** Ship Omnical three ways off one codebase —
**(1)** a hosted, centralized, multi-tenant SaaS we operate;
**(2)** a self-hosted deployment we distribute to third parties to run on their
own hardware; **(3)** a hardware appliance (our own router-class device) with
the app preinstalled and preconfigured.

**Decisions locked (user, 2026-09-28) — build to these, do not relitigate:**

| # | Decision | Consequence for this plan |
|---|---|---|
| D1 | **True multi-tenant SaaS**, not one-instance-per-customer | §3 tenant design + §6 tenancy core is a hard prerequisite for the hosted model (§7) and does **not** block §8/§9 |
| D2 | **Keep AGPL-3.0-or-later, embrace it** — no commercial licence, no relicense | §10 becomes a *product surface* (source-offer page + release artefacts), not a legal workstream |
| D3 | **Appliance reuses the router class** — libreCMC/OpenWrt RK3328 aarch64, as today | Zero new toolchain work; the 35 MiB overlay budget and the 4.85 MiB UPX'd binary stand. §9 is packaging + wizard work, not porting |
| D4 | This plan lives in its own file | `PLAN.md` gets only a pointer, as **§17.17** (next free number) |

**Non-goals for this plan:** billing/Stripe integration, an installer
framework beyond what §8 specifies, per-tenant custom branding, and any
horizontal-scale story beyond what §7.2 states as a limit.

---

## Table of Contents

1. [Executive summary](#1-executive-summary)
2. [Verified baseline — do NOT re-derive](#2-verified-baseline--do-not-re-derive)
3. [The shared spine: one binary, three shells](#3-the-shared-spine-one-binary-three-shells)
4. [Constraints, decisions and rationale](#4-constraints-decisions-and-rationale)
5. [Workstream 0 — repo hygiene and supply chain (BLOCKS ALL THREE)](#5-workstream-0--repo-hygiene-and-supply-chain-blocks-all-three)
6. [Workstream A — tenancy core](#6-workstream-a--tenancy-core)
7. [Workstream B — hosted, centralized deployment](#7-workstream-b--hosted-centralized-deployment)
8. [Workstream C — self-hosted deployment to distribute](#8-workstream-c--self-hosted-deployment-to-distribute)
9. [Workstream D — hardware appliance](#9-workstream-d--hardware-appliance)
10. [Licensing: AGPL §13 source offer](#10-licensing-agpl-13-source-offer)
11. [Sequencing and dependency graph](#11-sequencing-and-dependency-graph)
12. [Verification matrix](#12-verification-matrix)
13. [Rollback and recovery, per model](#13-rollback-and-recovery-per-model)
14. [Risks and mitigations](#14-risks-and-mitigations)
15. [File map](#15-file-map)
16. [Gotchas](#16-gotchas)
17. [Open questions for the user](#17-open-questions-for-the-user)
18. [Implementation split — first next action](#18-implementation-split--first-next-action)

---

## 1. Executive summary

The three deployment models look like three projects. They are not. They are
**one server binary with one extra dispatch layer**, differing only in:

| | Hosted (§7) | Self-hosted (§8) | Appliance (§9) |
|---|---|---|---|
| **Tenants** | N, one per customer | exactly 1 | exactly 1 |
| **Who runs the process** | us | the customer | the customer, on our SKU |
| **Where TLS terminates** | our edge LB / Caddy | customer's Caddy/nginx, or bundled `dav-tls` | bundled `dav-tls`, port 8443 (proven today) |
| **SMTP / iMIP identities** | per-tenant | one, operator-configured | one, provisioned at first boot |
| **Session store** | Redis (shared, tenant-namespaced) | `MemoryStore` (status quo) | `MemoryStore` (status quo) |
| **Data store** | one SQLite file per tenant | one SQLite file per tenant | one SQLite file |
| **Provisioning** | `rustical tenant create` + control-plane admin | `rustical setup` wizard / installer | first-boot wizard in portal |
| **Upgrade** | image pull + rolling | `rustical upgrade` (signed artefact) | `rustical upgrade` or `sysupgrade` |
| **Backup** | control-plane job, per tenant | `rustical backup` in-binary | `rustical backup` to USB/SD |

**The one architectural decision everything rests on** (§3.2): a **tenant is a
resolved `Router` instance, not a column.** One `axum::Router` per tenant,
dispatched by `Host` header, each holding its own store bundle. This means:

- **The tenant is isolated by construction** — a tenant's router physically
  holds only that tenant's SQLite pool. There is no `WHERE tenant_id = ?` to
  forget to write, so cross-tenant leakage is structurally prevented rather
  than defended. For a SaaS holding other people's calendars and 9 SMTP
  passwords, that is the property worth buying.
- **`AuthenticationLayer`, the DAV routers, the frontend routers and all 12
  store traits are unchanged.** `src/app.rs` already takes a whole store
  bundle; we call that bundle-construction N times instead of once.
- **Self-hosted and appliance are the N=1 case of the same code path**, which
  is why Workstream A does not block them.

**What is actually greenfield:** everything. There are currently **zero**
matches for `multi-tenan*`, `hosted`, `self-host*`, `appliance`, `onboard*` or
`SaaS` anywhere outside the planning docs, and **no** Dockerfile/compose/CI in
the outer repo. The only containerisation that exists is inherited from
upstream (`rustical/Dockerfile`, `rustical/compose.oidc.yml`) and is untested
here.

**What is already load-bearing and reusable:** invite-gated self-service
registration (`[registration]`, `/register`, `rustical invites`), app-token
lifecycle, guest shares, public export feeds, OIDC, the Apple `.mobileconfig`
generator, and — critically — a **battle-tested cross-compile + deploy +
watchdog + health-gate pipeline** for the router class. §9 reuses that almost
verbatim; §7 reuses the container recipe; §8 reuses the config renderer
(after de-hardcoding it, see §8.3).

---

## 2. Verified baseline — do NOT re-derive

All facts below were verified on 2026-09-28 by reading the tree. Anchors are
exact.

### 2.1 What the software is

| Item | Value |
|---|---|
| Upstream | `lennart-k/rustical` pinned at `v0.16.1`; fork is `v0.16.1-19-gdba08b2f` |
| Nested repo | `rustical/` is a **gitlink** (mode `160000`) in the outer repo; branch `omnical-scheduling`, tree clean |
| License | `AGPL-3.0-or-later` — `rustical/Cargo.toml:11` and `dav-tls/Cargo.toml:6` |
| Workspace | `rustical/Cargo.toml:1-2`, `members = ["crates/*"]`; edition 2024, rust-version 1.97 |
| Binaries | exactly **one**: `rustical` (no `[[bin]]` anywhere), from `src/main.rs` |
| Crates (12) | `api`, `caldav`, `carddav`, `dav`, `dav_push`, `frontend`, `ical`, `oidc`, `scheduling`, `store`, `store_sqlite`, `xml` |
| Release profile | `rustical/Cargo.toml:228-233` — `opt-level="z"`, `lto`, `panic="abort"`, `codegen-units=1`, `strip` |
| Static assets | Askama templates + `rust-embed` static assets **compiled into the binary** — single-file deploy, no asset sidecar |
| Data store | **SQLite only** — `config.rs:185-200`, `DataStoreConfig` is a single-variant enum. No Postgres anywhere |
| Migrations | 19 `.up.sql` in `crates/store_sqlite/migrations/` (2025-04-26 → 2026-09-22), 19 `.down.sql` (`principals.sql` has no down) |
| Config | TOML via `figment` — `src/main.rs:18-25`; file then `RUSTICAL_*` env (`__` = section split). **Every struct is `#[serde(deny_unknown_fields)]`** |
| CLI | `serve`, `gen-config`, `health`, `principals`, `subscriptions`, `invites`, `guest-share` — `src/lib.rs:51-77` |
| Config sections | `data_store`, `http`, `frontend`, `oidc`, `tracing`, `dav_push`, `nextcloud_login`, `caldav`, `scheduling`, `subscriptions`, `registration`, `maintenance` — `config.rs:325-351` |
| Routes | `/ping`, `/caldav`, `/caldav-compat`, `/.well-known/caldav`, `/carddav`, `/remote.php/dav`, `/index.php/login/v2`, `/frontend`, plus `export_` / `rsvp_` / `register_` routers mounted **outside** the auth layer — `src/app.rs:76-207` |
| Frontend routes | 37 `.route()` calls in `crates/frontend/src/lib.rs`; portal is `/{user}/…` (`lib.rs:93-191`) |

### 2.2 The facts that decide the tenancy design

These four are the whole reason §3.2 is what it is:

| # | Fact | Anchor | Why it matters |
|---|---|---|---|
| **B1** | `principals.id` is the **global** principal namespace; **every** table keys on it (`calendars`, `calendarobjects`, `addressbooks`, `app_tokens`, `memberships`, `group_owners`, `group_members`, `collection_shares`, `subscriptions`, `calendar_sources`, `scheduling_inbox_objects`, `password_resets`, `invites.used_by`) | migration `20250426122310_principals.sql` + 18 more | A `tenant_id` column design means widening the PK of ~13 tables and threading a param through ~12 store traits |
| **B2** | `AuthenticationLayer` **captures** `Arc<AP>` at construction: `AuthenticationLayer::new(auth_provider: Arc<AP>)` | `crates/store/src/auth/middleware.rs:26` | A per-request tenant cannot be injected without changing this one line's construction model |
| **B3** | The middleware has exactly **one** chokepoint — `Service::call` at `middleware.rs:69`, inserting the `Principal` at three points (`:79`, `:115`, `:118`) | `middleware.rs:69-126` | This is where a `Tenant` extension *could* go. **But it is not the cheapest place** — see B4 |
| **B4** | `make_app()` already receives a **complete store bundle** and builds the whole router from it | `src/app.rs`, called once from `src/lib.rs:192` | So: call it N times, one per tenant, and dispatch by `Host`. **Zero trait changes, zero middleware changes** |

B4 is the load-bearing one. It converts the tenancy problem from *"add a
dimension to the data model"* into *"add a map of routers"*.

### 2.3 Session store — the one true blocker for the hosted model

`src/app.rs:141`: `let session_store = MemoryStore::default();`, wrapped at
`app.rs:206-224` with `with_name("rustical_session")`, `with_secure(true)`,
`SameSite::Lax|Strict`, `Expiry::OnInactivity(2h)`.

Consequences: sessions do not survive a restart, and **cannot be shared across
replicas**. `tower-sessions = "0.15"` is declared with **no features**
(`rustical/Cargo.toml:87`), so a persistent store is a **new dependency**, not a
flag flip. There is no config knob for the store today.

### 2.4 The one hard security constraint for the hosted model

Rate limiting reads the **first hop of `X-Forwarded-For`** and falls back to a
single global bucket: `src/register.rs:243` and
`crates/frontend/src/routes/password_reset.rs:499`. The code is
already-proxy-aware but **trusts the header unconditionally**. Behind a hosted
edge LB, any client can set `X-Forwarded-For` and evade per-IP rate limits on
registration and password reset — i.e. **anyone can brute-force or flood**.
This must be fixed before any public hosted launch, and it is cheap (§7.3, item 4).

### 2.5 Build and deploy assets that exist

| Asset | Detail |
|---|---|
| `scripts/build-rust.sh` | 157 lines. Targets **only** `aarch64-unknown-linux-musl` (default) and `x86_64-unknown-linux-gnu`; anything else `exit 1` (`:36-100`). clang + `rust-lld` + musl includes from `zig libc` (not the zig-CC path — that is documented BROKEN). UPX `--lzma` (`:136-144`). Size gate `RUSTICAL_BUDGET=35 MiB` (`:30`, `:146-155`). |
| `out/` layout | router target lands **flat** in `out/`; other targets in `out/<target>/` (`build-rust.sh:106-115`) |
| Shipped sizes | `out/rustical` **5,090,440 B** (aarch64, UPX'd; 16,659,888 B raw) · `out/dav-tls` 610,892 B (1,291,680 B raw). `PLAN_BINARY_SIZE.md` fully executed: 29.0 MiB → 15.9 MiB → 4.85 MiB against a 35 MiB budget |
| `deploy.sh` | 180 lines, idempotent, 15 ordered steps: binary `file`+arch check → **render config to stdout before stopping anything** (fail-fast on `pass`) → stage in `/tmp` (tmpfs) → **deploy fence** `/tmp/rustical-deploy.lock` → stop/swap with busybox ENOSPC guard (`wc -c`, no `stat`) → size assertion → push config + init scripts + watchdog → idempotent prep → `sysupgrade.conf` persistence → watchdog cron → `opkg install sqlite3-cli` → `enable` → start + `health` + `netstat` + `logread` → release fence |
| `router/` overlay | `etc/init.d/rustical` (procd, `START=95`, `respawn 3600 5 0`, **`limits memory="268435456"`** = 256 MiB), `etc/init.d/dav-tls` (`START=96`, hard-pins `WAN_IF=eth0`/`WAN_IP=192.168.1.21`, refuses to start if the IP moved, runs `--user nobody`), `usr/bin/rustical-watchdog` (cron `*/5`, port-based check on `:8443`, honours the fence), `etc/rustical/config.toml` (secret-free base), `etc/rustical/certs/imap-novo-ordo.pem`, `etc/sysupgrade.conf.additions` |
| `dav-tls` | Rust, 420 lines. TLS-terminating **byte tunnel** (no HTTP parsing) — `--listen` (repeatable) `--upstream` `--cert` `--key` `--user`. **ALPN restricted to `http/1.1`** (`:39,:155`). Privilege drop (`:160-183`). `MAX_CONNS=1024`, `DRAIN_TIMEOUT=5s`. Plaintext-HTTP-on-TLS-port → 301 (`:193-250`). Graceful SIGTERM drain (`:400-419`) |
| `scripts/nightly-backup.sh` | 31 lines. **02:30 cron on the *dev machine*, SSHes to the router**, `PRAGMA wal_checkpoint(TRUNCATE)` + `sqlite3 .backup` + `tar czf` → `~/backups/omnical/<date>.tar.gz`, 30-day retention. **This is unusable for self-hosted and the appliance** — §8.4/§9.4 replace it |
| `scripts/render-router-config.sh` | 174 lines. **Hardcodes 8 SMTP + 7 IMAP accounts** by identity, pulling 15 passwords from `pass`; auto-mints the RSVP HMAC secret into `pass`; emits `[scheduling]`, `[subscriptions]`, `[registration]`. Prints to stdout, never touches disk. **Every one of these is site-specific and must be replaced for self-hosted** (§8.3) |
| `scripts/make-apple-profile.py` / `serve-apple-profile.py` | Deterministic (uuid5) combined `.mobileconfig`, 7 identities × CalDAV+CardDAV + 1 `user$group` payload; reads tokens from `pass`; output mode 0600. LAN HTTP server on `0.0.0.0:8917` |

### 2.6 What does **not** exist

Zero matches in the whole tree for `multi-tenan*`, `tenan*`, `SaaS`,
`self-host*`, `onboard*`, `appliance`, `hosted`, `centralized`, `helm`,
`AppImage`, `armv7`, `riscv`, `i686`. In the outer repo: **no** `Dockerfile`,
no compose, **no** `.github/`, no `Makefile`/`Justfile`, no `systemd` units, no
`deb`/`rpm`/Nix packaging. Inherited-but-unused upstream assets:
`rustical/Dockerfile` (multi-stage `rust:1.97-alpine` + `cargo-chef`, `FROM
scratch`, same clang/rust-lld recipe), `rustical/compose.oidc.yml`,
`rustical/.github/workflows/{cicd,docker-publish}.yml` (publishes to the
**upstream** repo, not the fork — must be retargeted, §5).

---

## 3. The shared spine: one binary, three shells

### 3.1 Component map

```
                            ONE BINARY  (out/rustical, 4.85 MiB UPX, aarch64-musl)
                                    │
   ┌────────────────────────────────┴─────────────────────────────────────┐
   │  HostDispatch  (NEW: src/host_dispatch.rs)                          │
   │  request ──► Host header ──► ControlPlane lookup ──► Arc<Router>   │
   └────────────────────────────────┬─────────────────────────────────────┘
                                    │  one Router per tenant
        ┌───────────────────────────┼───────────────────────────────────┐
        ▼                           ▼                                   ▼
   ┌─────────┐              ┌─────────────┐                    ┌─────────────┐
   │tenant A │              │  tenant B   │                    │  tenant N   │
   │Router   │              │  Router     │                    │  Router     │
   │         │              │             │                    │             │
   │ caldav_ │              │ caldav_ ... │                    │  ...        │
   │ carddav_│              │             │                    │             │
   │ fronten_│              │             │                    │             │
   │ d_router│              │             │                    │             │
   │ AuthLayer│              │             │                    │             │
   │ SessionLay│             │             │                    │             │
   │         │              │             │                    │             │
   │ StoreBundle A          │ StoreBundle B                     │ StoreBundle N
   │  SqlitePrincipalStore  │  …                                 │  …
   │  SqliteCalendarStore   │                                    │
   │  … (12 stores)         │                                    │
   │  └─ pool → tenant_a.sqlite3                                   │
   └─────────┘                     └─────────────┘                  └─────────────┘
                                    │
                     ┌──────────────┴──────────────┐
                     │  ControlPlane (NEW)          │
                     │  control.sqlite3             │
                     │  tenants / quotas / status   │
                     │  admin CLI + admin portal    │
                     └─────────────────────────────┘
```

### 3.2 Core design decision: a tenant is a resolved `Router`, not a column

The obvious design — add `tenant_id` to `principals` and ~13 more tables and
thread a param through ~12 store traits — is the wrong shape for this
codebase, and fact **B4** is why:

- `make_app()` (`src/app.rs`, driven from `src/lib.rs:192`) **already** takes a
  complete store bundle and returns a finished `Router`. There is exactly one
  construction site.
- `AuthenticationLayer` (B2) captures its provider *at construction*. Per-tenant
  routers give each tenant its own `AuthenticationLayer` over its own
  `SqlitePrincipalStore`. **The middleware never changes.**
- The 12 store traits, the DAV routers, `caldav_router`/`carddav_router`/
  `frontend_router`, `AuthenticationProvider` and `Principal` are **all
  untouched**.
- Isolation is total by construction. There is no `WHERE tenant_id = ?` for a
  future contributor to forget, and no ambient/implicit tenant state to thread
  through a trait method.

**Cost of the choice, stated honestly:**

| Cost | Mitigation |
|---|---|
| No cross-tenant SQL query. Admin views ("all tenants, all users") need a different path. | The control plane is a separate DB and *is* the cross-tenant index. Tenant↔store mapping lives there. |
| N routers + N pools in one process's memory. | Each axum `Router` is Arc-backed and cheap; the pools are the real cost. Bounded by an LRU of at most `[tenancy] max_cached_tenants` (default 64) with an on-demand rebuild otherwise. **Hard limit stated in §7.2.** |
| Per-tenant re-connection churn | Measured in §7 wave 3; if it bites, the escape hatch is sharding into N processes, not changing the tenancy model. |
| `[scheduling]`, `[subscriptions]`, `[registration]`, `rsvp_secret` are **global** config today (`config.rs:325-351`) | `tenants.config_json` holds per-tenant overrides; the resolver merges global defaults ← tenant overrides (§3.6). Avoids a config-format explosion. |

**Rejected alternatives, and why:**

- *Qualified principal ids* (`"acme::alice@example.com"` stored in the existing
  `id` column) — zero schema change, but it leaks the tenant into every
  CalDAV discovery URL, collides with the `user$group` impersonation bodge
  (`middleware.rs:89` splits on `$`), breaks the globally-unique
  `displayname` constraint (migration `20260910120000_displayname_unique`), and
  leaves permanent migration debt. Rejected.
- *`tenant_id` column everywhere* — correct long-term shape, but it rewrites
  the PK of ~13 tables and the signature of every principal-keyed trait method
  across the whole fork. It is the right answer only if sharding is never
  wanted. Rejected for now; §16 records the trigger to revisit.
- *One instance per customer* — contradicts D1. Noted only because it is the
  cheap alternative if §6 ever overruns.

### 3.3 HostDispatch — the new chokepoint

New file `src/host_dispatch.rs`. Responsibilities, in order:

1. Read the `Host` header, strip the port, lowercase.
2. Look up the **ControlPlane** (`tenants` table) for an active tenant whose
   host matches. Match order, first hit wins:
   1. exact `hosts` entry (a tenant may claim several hostnames);
   2. `{slug}.{tenancy.base_domain}` when `base_domain` is set;
   3. the whole host treated as a slug (host-per-tenant, e.g. `acme.t3.gg`);
   4. `[tenancy] default_tenant` if set — **this is the self-hosted and
      appliance case, where there is exactly one tenant and the request Host is
      whatever the LAN called it**;
   5. none → 404 for an unknown host on a public domain, or the
      **tenant-chooser page** on `default_domain` (§6.4).
3. Return a clone of that tenant's `Arc<Router>` (axum `Router` is `Clone`
   and internally Arc'd, so this is a pointer bump, not a deep copy).
4. Never cache a `Tenant` decision in a layer that outlives a single request —
   a suspended tenant must take effect immediately. Cache the **`Router`**, not
   the lookup; the lookup hits the control plane (a small SQLite, or Postgres)
   on every request and is ~50 µs.

**Suspension semantics:** a suspended tenant's router is **removed from the
map and its pool closed**, so in-flight requests drain and nothing new starts.
This is a real, testable admin action, not a boolean the app ignores.

### 3.4 The control plane

New small database, `control.sqlite3`, *not* the tenant store. Schema sketch:

```sql
-- Omnical §3.4 — hosted tenancy control plane. Deliberately a SEPARATE
-- database from any tenant store: it is the cross-tenant index, so a
-- per-tenant backup/restore can never take it out, and a tenant cannot
-- see the existence of any other tenant.
CREATE TABLE tenants (
    id            TEXT PRIMARY KEY,          -- ulid/short random id
    slug          TEXT NOT NULL UNIQUE,      -- [a-z0-9-]{1,63}, DNS-safe
    display_name  TEXT NOT NULL,
    status        TEXT NOT NULL CHECK (status IN ('active','suspended')),
    plan          TEXT NOT NULL DEFAULT 'free',
    -- Per-tenant overrides for the GLOBAL-only config sections (§3.6):
    -- scheduling.smtp / scheduling.imap / subscriptions.public_url /
    -- registration.* / rsvp_secret. Absent key = inherit the global value.
    config_json   TEXT NOT NULL DEFAULT '{}',
    quota_principals   INTEGER,              -- NULL = unlimited
    quota_calendars    INTEGER,
    quota_megabytes    INTEGER,
    created_at    TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    suspended_at  TEXT
);
CREATE TABLE tenant_hosts (
    host    TEXT PRIMARY KEY,
    tenant  TEXT NOT NULL REFERENCES tenants (id) ON DELETE CASCADE
);
```

The control plane is the one place a cross-tenant query is *wanted*: tenant
list, status, quotas, and the store-file path. It holds **no calendar, contact
or credential data**.

**Store path convention:** `<data_root>/tenants/<tenant_id>/db.sqlite3`.
`[tenancy] data_root` defaults to the directory of the configured `db_url` so
the N=1 case keeps exactly today's path (no migration, no surprise).

### 3.5 Store-bundle cache

`Arc<StoreBundleCache>` in front of the control plane, holding
`LruCache<TenantId, (Weak<StoreBundle>, Arc<Router>)>` sized by
`[tenancy] max_cached_tenants` (default 64). On miss: build the store bundle
(`get_data_stores`, `src/lib.rs:80` — already one function, 5 CLI call sites +
1 serve call site, so a tenant-aware variant is a small change), build the
router via `make_app`, insert. On a `Status`/subscription event, evict the
entry and drop the router — the pool closes and the tenant gets a fresh one.

### 3.6 Config surface for tenancy

New `[tenancy]` section. **When `enabled = false` (the default) every existing
deployment behaves exactly as today** — same router, same store, no dispatch
layer. This is the property that lets §8 and §9 ship without waiting for §6.

```toml
[tenancy]
enabled = false                  # master switch; default keeps single-tenant behaviour
default_tenant = ""              # N=1 self-host/appliance: every Host lands here
base_domain = ""                 # set for hosted: {slug}.{base_domain} resolution
default_domain = ""              # hosted: bare domain -> tenant-chooser page
data_root = ""                   # default: dir of data_store.sqlite.db_url
max_cached_tenants = 64
control_db_url = "file:/var/lib/omnical-control/control.sqlite3"
trusted_proxies = []             # §7.3 item 4 — MUST be set for hosted
[tenancy.sessions]
# "memory" (default, status quo) | "redis"  — "redis" requires the
# `session-redis` cargo feature; see §3.7
store = "memory"
redis_url = ""
redis_prefix = "omnical:"        # per-tenant keys are "{prefix}{tenant_id}:"
```

And the per-tenant overrides, resolved as **global config ← `tenants.config_json`**:

| Section | Why it must become per-tenant | Currently |
|---|---|---|
| `scheduling.smtp` / `scheduling.imap` | one `[scheduling.smtp]` list is shared by all tenants; each tenant's invites must come from *its own* identity, and a tenant's IMAP poll must read *its own* mailbox | one global list, 8 SMTP + 7 IMAP, `render-router-config.sh:33-59` |
| `subscriptions.public_url` | every tenant's subscribe links must point at its own host | one global string (`render-router-config.sh:154`) |
| `rsvp_secret` | the RSVP HMAC key must not be shared across tenants — a shared key means one tenant can forge another's `/rsvp/{token}` link | one global secret from `pass` (`render-router-config.sh:75-88`) |
| `registration.*` | per-tenant open/closed, invite-required, rate limits | one global block (`render-router-config.sh:162-172`) |

### 3.7 Sessions — the one real dependency decision

`MemoryStore` (`src/app.rs:141`) stays the default and is what §8/§9 use. For
hosted, a **config-selected store behind a cargo feature**:

| Artifact | Features | Session store | Used by |
|---|---|---|---|
| `out/rustical-aarch64` | default | memory | appliance, self-host |
| `out/rustical-hosted-{amd64,arm64}` | `session-redis` | redis or memory | hosted |

`tower-sessions` 0.15 ships an official `SessionStore` trait, so a Redis store
is a **small adapter crate** (`crates/store_redis`, new) with a `redis` dep —
but gating it behind a cargo feature keeps the appliance/self-host binary free
of Redis, which is worth more than the single-artifact purity. Two artefacts
from one tree; `scripts/build-rust.sh` gains a `--features` passthrough and a
hosted target (§7.1).

### 3.8 What §3 costs, in the fork

| Change | Files | Nature |
|---|---|---|
| Extract the router build from `src/app.rs` into `make_app_for(config, stores, tenant)` | `src/app.rs` | mechanical refactor, no behaviour change |
| New `HostDispatch` service | `src/host_dispatch.rs` | new |
| New control-plane store | `crates/store/src/tenant_store.rs`, `crates/store_sqlite/src/tenant_store.rs` + migration | new |
| New `Tenant`/`TenantId` types + config | `crates/store/src/tenant.rs`, `src/config.rs` | additive |
| `get_data_stores` → tenant-aware | `src/lib.rs:80-192` | one function, 6 call sites |
| `rustical tenant` CLI | `src/commands/tenants.rs` | new |
| Per-tenant override merge | `src/app.rs` (construction) | small |
| Session store enum | `src/app.rs:141` + new `crates/store_redis` | config-selected |
| Redis store adapter | `crates/store_redis/` | new, feature-gated |
| `X-Forwarded-For` trust | `src/register.rs:243`, `crates/frontend/src/routes/password_reset.rs:499` | **security fix** |

Nothing in `crates/{caldav,carddav,dav,ical,oidc,scheduling,xml}` changes, and
nothing in `crates/frontend` changes except the two `X-Forwarded-For` lines.
That is the payoff of the design in §3.2.

---

## 4. Constraints, decisions and rationale

| # | Constraint (fact) | Decision | Rationale |
|---|---|---|---|
| C1 | `principals.id` is the global namespace and 13 tables key on it (B1) | **Tenant = resolved `Router` over its own SQLite file**, not a `tenant_id` column | Isolation by construction; zero store-trait churn; per-tenant backup/restore is free. See §3.2 for the rejected alternatives |
| C2 | `make_app()` already takes a whole store bundle, and there is exactly **one** construction site (`src/lib.rs:192`) | Dispatch N routers by `Host` | Turns a data-model project into a dispatch-layer project |
| C3 | `AuthenticationLayer` captures its provider at construction (`middleware.rs:26`) | **Do not touch the middleware.** Per-tenant routers give each tenant its own layer | The 127-line middleware, and B3's chokepoint, are irrelevant under this design. Fewer moving parts, and the security-critical auth path is untouched |
| C4 | `MemoryStore` (`src/app.rs:141`); `tower-sessions = "0.15"` with no features | Keep memory as default; add Redis behind a `session-redis` feature for hosted only | Appliance/self-host stay dependency-free and keep the status quo; hosted gets a shared, restart-tolerant store |
| C5 | `X-Forwarded-For` is trusted unconditionally (`register.rs:243`, `password_reset.rs:499`) | Add `[tenancy] trusted_proxies`; honour the header **only** when the immediate peer is listed | Public hosted launch is a rate-limit bypass otherwise. Cheap, and a security gate (§7.3) |
| C6 | `scheduling`/`subscriptions`/`rsvp_secret`/`registration` are global config | Per-tenant overrides in `tenants.config_json`, merged global ← tenant | Avoids a config-format explosion; a shared RSVP HMAC key would let one tenant forge another's RSVP links |
| C7 | Data store is SQLite-only (`config.rs:185-200`); no Postgres anywhere | **Stay SQLite**, one file per tenant | Per-tenant file means per-tenant write contention is *already* isolated — a shared SQLite would serialise every tenant's writes. Postgres becomes a §7 wave-3 question, not a prerequisite |
| C8 | `AGPL-3.0-or-later` inherited from upstream | Keep it; publish the corresponding source to network users | D2. §10 makes it a product surface, not a legal cost |
| C9 | Appliance is the same RK3328 aarch64 libreCMC class, 4.85 MiB binary, 35 MiB overlay budget, 256 MiB rlimit | Reuse the entire build + deploy pipeline unchanged | D3. The hard part (fitting) is already solved and proven |
| C10 | `rustical/` is a **gitlink with no `.gitmodules`**; 33,998 build artefacts are tracked; `.git` is 3.3 GB; no remote | **Workstream 0 first.** A fresh clone is currently broken, so no distribution is possible | §5. This is a hard blocker on all three models, not hygiene |
| C11 | `out/tls/key.pem` (a live Let's Encrypt **private key**) and `out/omnical-iphone.mobileconfig` (15 live app tokens) are **tracked in git** | Rotate the key, revoke the tokens, purge history, ignore | Shipping a repo that contains a live TLS key is not acceptable for any of the three models. §5 |
| C12 | `render-router-config.sh` hardcodes 8 SMTP + 7 IMAP identities and 15 `pass` entries | For self-hosted/appliance, config is generated by a **wizard**, not by this script; the script stays the *internal* deploy path | A third party has no `pass` store and no reason to have our identities. §8.3 |
| C13 | `nightly-backup.sh` runs on the *dev machine* and SSHes to the router | Move backup/restore **into the binary** (`rustical backup` / `restore`) | Self-hosted and appliance customers have neither our dev machine nor our SSH keys. §8.4, §9.4 |
| C14 | No CI in the outer repo; upstream's workflows publish to the **upstream** repo | Retarget CI to the fork; add a build-matrix + secret-scan gate | §5.4 |
| C15 | `dav-tls` is ALPN **`http/1.1` only** (`dav-tls/src/main.rs:39,155`) | Hosted terminates TLS at the edge LB; `dav-tls` stays appliance/self-host only | Behind an LB that speaks HTTP/1.1 upstream this is exactly right. But WebDAV-Push WebSocket `Upgrade` must be explicitly allowed through the LB (§7.3) |
| C16 | The export / rsvp / register routers are mounted **outside** the auth layer (`src/app.rs:143-190`) | Tenant resolution is **orthogonal** to auth: `HostDispatch` runs outermost, so token-only routes are tenant-scoped too | These are exactly the routes where a missing tenant check would leak data across tenants. Separating the two concepts makes the gap obvious at review time |

---

## 5. Workstream 0 — repo hygiene and supply chain (BLOCKS ALL THREE)

Nothing ships until this is done. It is unglamorous and it is the difference
between a private deployment and a distributed product.

### 5.1 Findings

| # | Finding | Evidence | Severity |
|---|---|---|---|
| **H1** | **Live Let's Encrypt private key committed** — `out/tls/key.pem`, `CN=0115d8cf.duckdns.org`, valid 2026-09-04 → 2026-12-03 | `git ls-files out/tls/` | **Critical** |
| **H2** | **15 live CalDAV/CardDAV app tokens committed** — `out/omnical-iphone.mobileconfig` contains real `CalDAVPassword`/`CardDAVPassword` values | `git ls-files out/omnical-iphone.mobileconfig`; contradicts `render-router-config.sh:9-10` and `make-apple-profile.py:18`, which both claim secrets never touch the repo | **Critical** |
| **H3** | 33,998 build artefacts tracked (`build/cargo-target/**`, `dav-tls/target/**`); `.git` is 3.3 GB | no `.gitignore` at the root | High |
| **H4** | `rustical/` is a gitlink with **no `.gitmodules`** → a fresh `git clone` yields an **empty `rustical/`**, so the build cannot even start | no `.gitmodules` | **Critical for distribution** |
| **H5** | Compiled binaries tracked (`out/rustical`, `out/dav-tls`, `out/x86_64-unknown-linux-gnu/*`) — every build churns them | `git ls-files out/` | Medium |
| **H6** | No remote configured; branch `main` only; ~20 of 25 commit subjects are the literal string `...` | `git log`, `git remote -v` | Medium |
| **H7** | No CI at all in the outer repo — none of the build or deploy logic is verified by anything | no `.github/` | High |

### 5.2 Tasks

1. **Rotate H1.** Revoke + reissue the `0115d8cf.duckdns.org` certificate (or
   switch that host to a wildcard). Update `out/tls/` and the deployed
   `/etc/rustical/tls/`. *Gate: `openssl x509 -noout -dates` shows the new
   validity window; the old serial is gone from the CA's revocation list.*
2. **Revoke H2.** For each of the 7 identities, `rustical --config-file
   /etc/rustical/config.toml principals app-token list <principal>` then
   `app-token remove <principal> <id>`, re-run
   `scripts/make-apple-profile.py` to mint fresh ones, and re-deploy. *Gate:
   the old tokens return 401 on loopback DAV.*
3. **Add `.gitignore`** covering `build/cargo-target/`, `*/target/`,
   `out/tls/`, `out/*.mobileconfig`, `out/rustical`, `out/dav-tls`,
   `out/x86_64-unknown-linux-gnu/`, `*.pem`, `*.key`. *Gate:
   `git status --porcelain` is empty after a full build.*
4. **Add `.gitmodules`** and re-point the `rustical` gitlink at the fork's real
   URL. *Gate: `git clone <url> /tmp/clonetest && cd /tmp/clonetest &&
   ./scripts/build-rust.sh x86_64-unknown-linux-gnu` succeeds on a clean
   machine.* **This is the distribution smoke test.**
5. **Purge history** — `git filter-repo` (or BFG) to strip `out/tls/`,
   `out/*.mobileconfig` and the artefact trees, then force-push. Coordinate
   with the user first: this rewrites history and the repo has no remote, so
   there may be clones to fix. **Ask before running.**
6. **Untrack the artefacts** (`git rm --cached -r build/cargo-target out/rustical
   out/dav-tls out/x86_64-unknown-linux-gnu out/tls out/omnical-iphone.mobileconfig`)
   and **drop `.git` → consider a fresh history** if `du -sh .git` is
   unacceptable. *Gate: `du -sh .git` under ~100 MB.*
7. **Set up the remote** and fix the commit subjects going forward
   (`<area>: <imperative summary>`). Historical `...` subjects are cosmetic —
   do **not** rewrite them, that is a second history rewrite for no benefit.
8. **A `SECURITY.md` + a secret-scan in CI** (§5.4) so H1/H2 cannot recur.

### 5.3 `dav-tls` is AGPL but is not part of the hosted deployment

Worth stating once: hosted terminates TLS at the edge (§7.3), so `dav-tls` is
**not shipped to hosted network users**. The AGPL §13 source offer for hosted
therefore covers the `rustical` fork only (§10). `dav-tls` is offered with the
appliance and self-host tarballs, which are distributions of the whole tree
anyway.

### 5.4 CI (new, in the outer repo)

`.github/workflows/`:

| Workflow | Jobs | Gate |
|---|---|---|
| `hygiene.yml` | `gitleaks`/`trufflehog` scan; assert no tracked files match `out/tls/*`, `*.pem`, `*.mobileconfig`, `build/cargo-target/*`, `*/target/*`; assert `.gitmodules` exists and `rustical` is a valid submodule | blocks merge |
| `build.yml` | matrix: `aarch64-unknown-linux-musl` (default features) and `x86_64-unknown-linux-gnu`; run `scripts/build-rust.sh <t>`; assert both binaries ≤ `RUSTICAL_BUDGET` (35 MiB) and are UPX-packed | blocks merge |
| `test.yml` | `cd rustical && cargo test --workspace` (the 98-test integration baseline, §16) + `cargo fmt --check` + `cargo clippy --workspace --all-targets` with **no new warnings** | blocks merge |
| `docker.yml` | build + push `ghcr.io/<fork>/omnical` multi-arch (`linux/amd64`, `linux/arm64`) — **retarget upstream's `docker-publish.yml`**, which currently publishes to `lennart-k`'s repo | blocks release |
| `release.yml` | on tag: build artefacts, `cosign`-sign the checksums, publish a GitHub release + the source tarball (§10) | on release |

*Gate for Workstream 0: all five workflows green on a push, and the H4 clone
smoke test in a container.*

---

## 6. Workstream A — tenancy core

**Deliverable:** `[tenancy] enabled = true` serves N tenants from one process
by `Host`, each on its own SQLite file, with the entire existing DAV and portal
surface working per tenant. **Gate: §12 rows 20-26 green.**

This workstream unblocks §7 only. §8 and §9 are N=1 cases and ship without it.

### 6.1 A1 — `HostDispatch` and the control plane

- New `src/host_dispatch.rs`: the 5-step resolution in §3.3, as a
  `tower::Service<Request, Response = Response>` wrapping the inner
  `HostDispatchRouter`.
- New `crates/store/src/tenant_store.rs` with a `TenantStore` trait:
  `create_tenant`, `get_tenant_by_slug`, `get_tenant_by_host`, `list_tenants`,
  `update_tenant_status`, `set_quota`, `delete_tenant`.
- New `crates/store_sqlite/src/tenant_store.rs` + migration
  `2026MMDDHHMMSS_tenants.sql` (schema in §3.4), including the
  `tenants` + `tenant_hosts` tables above.
- New `crates/store/src/tenant.rs`: `TenantId` newtype (validated slug),
  `Tenant` struct, `TenantStatus` enum. — **DONE 2026-09-28**, §18.12
- New `[tenancy]` config in `src/config.rs:325-351`, `enabled = false`.
- Refactor `src/app.rs` so the existing body becomes
  `make_app_for(config, stores, tenant) -> Router`, and `make_app` stays as a
  1-tenant wrapper. **Pure refactor — the 98-test baseline must stay green
  with zero test changes.** — **DONE 2026-09-28**, §18.11
- *(new, §18.12)* the `tenant` **parameter and the `Extension` wiring** are in
  too, as a separate commit that preserves §18.11's "body moved verbatim" claim
  and the 98-test canary. It carries **no store path and no `TenantStore`
  trait**: §3.2 makes a tenant a resolved `Router`, not a column, so isolation
  is structural and must not be faked by threading an id into store methods.
*Gate: `make_app` refactor is behaviour-neutral — 98/98 integration tests pass
unchanged; `cargo fmt`; clippy clean on `rustical`.*

### 6.2 A2 — multi-tenant construction path

- `src/lib.rs:80` — make `get_data_stores` tenant-aware (a tenant-aware
  `db_url` + a per-tenant `StoreBundle` struct grouping the 12 stores). Update
  the **6 call sites**: `src/lib.rs:192` and
  `src/commands/{subscriptions,guest_shares,invites,principals}.rs`.
- New `src/store_bundle.rs`: the bundle struct + `StoreBundleCache` (LRU,
  §3.5).
- `src/app.rs:141` — the session store becomes config-selected (§3.7); the
  Redis adapter is new `crates/store_redis/` behind the `session-redis`
  feature, with per-tenant key prefixing so tenants cannot read each other's
  sessions.
*Gate: `cargo test --test run_integration_tests` ≥ 98 pass; 4 new tests —
tenant A's principal cannot authenticate in tenant B, tenant A's calendar is
404 in tenant B, suspension takes effect within one request, per-tenant
`rsvp_secret` isolation.*

### 6.3 A3 — per-tenant config overrides

- `tenants.config_json` deserialised into a `TenantOverrides` struct:
  `scheduling` (smtp/imap vectors), `subscriptions` (`public_url`),
  `registration` (the whole block), `rsvp_secret`.
- The merge point is where `make_app_for` reads the config; a `TenantConfig`
  overlay applied after the global `Config` is built. **Critical:** SMTP/IMAP
  credentials live in the **control plane**, so the control-plane DB joins the
  secrets inventory (§8.6) and is `0600`.
*Gate: 3 new tests — tenant A's registration invite email is sent from
tenant A's identity, not tenant B's; tenant A's subscribe URL uses tenant A's
public_url; tenant A's `/rsvp/{token}` minted with tenant B's secret 404s.*

### 6.4 A4 — tenant resolution on the un-authenticated routers

The `export_`, `rsvp_` and `register_` routers are mounted **outside** the
auth layer (`src/app.rs:143-190`). Because `HostDispatch` is the *outermost*
layer, they are already tenant-scoped — but each of them resolves ownership
from a **token**, so each needs an explicit tenant assertion:
- `src/export.rs` — the `subscriptions` row is looked up inside a
  tenant's store; a token from tenant A cannot resolve in tenant B. Assert it.
- `src/rsvp.rs` — the HMAC is now per-tenant, so verification must use the
  **request tenant's** secret. This is a real cross-tenant forge vector if
  missed.
- `src/register.rs` — the `invites` table is per-tenant; an invite minted in
  tenant A must not redeem in tenant B.
*Gate: 3 new tests, one per router, each asserting a tenant-A token/secret is
rejected under tenant B's host. **Do not skip these** — they are the three
routes with no principal, so they are the three places a tenant check can
silently be absent.*

### 6.5 A5 — `rustical tenant` CLI

New `src/commands/tenants.rs`, following the `guest_shares.rs` shape
(`get_data_stores` + explicit store args + `clap::ValueEnum`):

```
rustical --config-file <cfg> tenant create --slug acme --display-name "Acme Co" \
    [--host cal.acme.t3.gg] [--plan pro] [--config-json @file]
rustical … tenant list [--status active|suspended] [--json]
rustical … tenant show <slug>
rustical … tenant suspend <slug>        # evicts the router, closes the pool
rustical … tenant resume <slug>
rustical … tenant set-quota <slug> --principals 50 --calendars 500 --megabytes 2048
rustical … tenant config set <slug> --key scheduling.smtp --value @file
rustical … tenant delete <slug> --confirm --purge-data
```

The `--config-file` flag is a **top-level option that must precede the
subcommand** (`src/lib.rs:52-57`) — a documented footgun in 6+ places.
*Gate: `tests/tenant_cli.rs` — 8 tests; `tenant delete` without `--confirm`
refuses.*

### 6.6 A6 — admin surface

- New `[tenancy] admin_group` (or an explicit `platform_admins` list in
  `[tenancy]`, defaulting to a single principal) — an admin must be able to
  cross tenant boundaries deliberately, and that must be **audited**.
- A minimal `/frontend/admin/tenants` page: list, create, suspend, resume,
  show quota usage. **Deliberately minimal** — billing, plan management and
  per-tenant branding are out of scope (§1 non-goals).
- Audit: every admin action appends to a `control_admin_audit` table
  (actor, action, tenant, timestamp).
*Gate: a non-admin principal gets 404 (not 403) on `/frontend/admin/*` — do not
confirm the panel's existence; 3 tests.*

---

## 7. Workstream B — hosted, centralized deployment

**Deliverable:** a publicly reachable, multi-tenant Omnical we operate, behind
an edge LB, on Postgres-or-SQLite with a shared session store, source-offered
per AGPL §13.

### 7.1 B1 — artefacts and target

| Artefact | Target | Features | Built by |
|---|---|---|---|
| `rustical-hosted-amd64` | `linux/amd64` | `session-redis` | new `scripts/build-rust.sh --features session-redis x86_64-unknown-linux-gnu` |
| `rustical-hosted-arm64` | `linux/arm64` | `session-redis` | as above with the aarch64 target |
| `ghcr.io/<fork>/omnical:<tag>` | multi-arch | `session-redis` | new `Dockerfile` in the outer repo (retargeted from `rustical/Dockerfile`) |

The outer-repo `Dockerfile` must fix one upstream wart: upstream's copies CA
certs **after** `FROM scratch` and sets
`RUSTICAL_DATA_STORE__SQLITE__DB_URL=/var/lib/rustical/db.sqlite3` — keep
both, but the volume is now `<data_root>/tenants/<id>/db.sqlite3`
(`[tenancy] data_root` + the control DB on a second volume).

*Gate: `docker run --rm -p 4000:4000 <image>` answers `/ping`; CI matrix green.*

### 7.2 B2 — topology and its stated limit

```
 Internet
    │  HTTPS 443
    ▼
 ┌──────────────────┐   TLS 1.2/1.3, HTTP/2 to the client
 │  Edge (Caddy or  │   ALPN http/1.1 + h2 to the app
 │  cloud LB)       │   WebSocket Upgrade passthrough (WebDAV-Push)
 └────────┬─────────┘   rate limits, max body 32 MB
          │ HTTP/1.1, private network
          ▼
 ┌──────────────────┐
 │ omnical (N=1..k) │  [tenancy] enabled = true
 │  HostDispatch     │  control.sqlite3 (or Postgres) + Redis
 └────────┬─────────┘
          ├── tenants/<id-A>/db.sqlite3
          └── tenants/<id-B>/db.sqlite3
```

**Stated limit (measure, do not guess):** one process holds N routers and N
pools. `max_cached_tenants` (64) bounds resident pools. The escape hatch when
N outgrows one process is **sharding by tenant across processes** (a hostname →
process map at the LB), *not* a change to the tenancy model. §6 A1 measures
per-tenant pool cost in wave 3 and records the number here.

*Gate: a load test with 200 tenants × 50 concurrent clients, p99 DAV latency
within 2× the single-tenant baseline, RSS within the pod limit.*

### 7.3 B3 — edge requirements (each is a bug if missed)

1. **Terminate TLS at the edge; run HTTP/1.1 to the app.** `dav-tls` is
   ALPN-`http/1.1`-only (C15) and is not in this path at all. Never put
   `dav-tls` in front of a hosted instance.
2. **WebDAV-Push WebSockets must pass through.** `[dav_push] enabled` is
   `true` by default (`config.rs`, `DavPushConfig::default`). If the LB strips
   `Upgrade`, WebDAV-Push silently degrades to polling — verify explicitly
   (a client that still syncs is not evidence the socket works).
3. **`.well-known` must not be rewritten.** `/.well-known/caldav` answers with
   a `301` to `/caldav`, chosen by `User-Agent` sniffing
   (`src/app.rs:99-113`, deliberately special-cased for Apple
   `remindd`). A proxy that strips or rewrites UA will send every Apple client
   down the wrong path — this is a **known upstream landmine**.
4. **`trusted_proxies` is mandatory.** `X-Forwarded-For` is trusted
   unconditionally today (C5). Configure it, and add the peer check in code.
   *This is a security gate, not a nicety.*
5. **`payload_limit_mb = 32`** must be at least as large at the LB as the app's
   (the router config sets 32, `router/etc/rustical/config.toml:11`).
6. **Long timeouts.** A `REPORT` calendar-query on a large calendar is slow;
   the LB read timeout must exceed the app's, or clients see spurious 504s.
7. **No path rewriting, no trailing-slash normalisation, no query-string
   mangling.** WebDAV depends on exact collection paths. `NormalizePathLayer`
   is applied in-app (`src/lib.rs`); the proxy must not fight it.

### 7.4 B4 — operations that do not exist yet

| Need | Currently | Work |
|---|---|---|
| Backup | `scripts/nightly-backup.sh` — **dev machine + SSH** (`router-dav/scripts/nightly-backup.sh`) | In-binary `rustical backup` (§8.4), invoked per tenant from CI/cron. Back up **each** tenant file separately — that is the payoff of C7 |
| Restore / tenant deletion | none | `rustical tenant delete --purge-data` (§6.5) + a documented restore drill |
| Migrations | automatic on boot (`sqlx::migrate!`, `crates/store_sqlite/src/lib.rs:43-57`); `--no-migrations` to suppress | Per-tenant migration on tenant creation; document that a rollback of the binary past a migration needs a `.down.sql` — **9 of 19 pairs have no `.up`/`.down` symmetry guarantees; audit before the first downgrade** |
| Health / readiness | `health` CLI + `/ping` (`src/app.rs:78`) | Add `/readyz` that checks the control plane + the tenant pool, so the LB does not route to a process that cannot serve |
| Observability | `[tracing] opentelemetry` behind the `debug` feature | Enable `opentelemetry` in the hosted build; per-tenant request tagging from the `HostDispatch` decision |
| Secrets | `pass` on a dev machine | A real secret store (Doppler/Vault/SSM). **The control plane now holds SMTP passwords** (C6/§6.3) |

### 7.5 B5 — hosted-specific product work

1. **`[registration]` per tenant** — open/closed signup, per-tenant invite
   codes, per-tenant rate limits. Already ~80% built (§17.8); the work is
   making it per-tenant (§6.3) and putting a **branded** welcome path in
   front of it.
2. **Tenant onboarding** — `tenant create` + a first-admin bootstrap. Today
   the equivalent is `rustical invites create --send` (§17.8/§17.14) plus the
   Apple profile; that maps to a hosted "create customer" runbook.
3. **The Apple `.mobileconfig` flow generalised.** `scripts/make-apple-profile.py`
   is deterministic (uuid5) and already emits 7 identities × CalDAV+CardDAV +
   a `user$group` payload. Per tenant it becomes a **rendered, expiring,
   signed** profile served from the portal instead of a file on a dev machine.
4. **Email deliverability** — hosted sending invites from customer identities
   will hit SPF/DKIM/DMARC. Per-tenant `scheduling.smtp` is the mechanism
   (C6); the runbook must require a verified domain before a tenant's
   identity is enabled.
5. **Quotas and soft limits** — `tenants.quota_*` (schema in §3.4). Enforcement
   in the write path is **wave 3**; *displaying* usage in the admin panel is
   wave 1 and is enough to start.

### 7.6 B6 — hosted gates

*Gate: §12 rows 27-33, including the `X-Forwarded-For` bypass test (a request
with a forged header from a non-trusted peer must still be rate-limited), the
cross-tenant isolation suite from §6.4, and an external reachability check
against `check-host.net` (the §4.3 pattern already used in this project).*

---

## 8. Workstream C — self-hosted deployment to distribute

**Deliverable:** a third party runs Omnical on their own hardware, unattended,
with a config they can understand and a supported upgrade path. **This is the
model most likely to produce support load, so the wizard is the deliverable —
not the tarball.**

### 8.1 C1 — the distribution channel decision

| Channel | Audience | Verdict |
|---|---|---|
| **Docker Compose (primary)** | anyone with a VPS/NAS/home server | **Primary.** `rustical/compose.oidc.yml` is a usable base (22 lines, already has a `data:` volume and the OIDC env shape). Add `compose.omnical.yml` with the tenancy, registration, subscriptions and scheduling blocks |
| **Native tarball + installer script** | hosts without Docker; the appliance | **Secondary.** Static musl binaries, `systemd` unit, `install.sh` with a checksum-verified download |
| **Ansible role / Packer image** | fleet and cloud | Defer |
| **AppImage / deb / rpm / Nix** | Linux desktop users | Defer — `AppImage` and packaging have **zero** prior art in this tree |

*Gate: `docker compose up` on a clean host reaches `/ping` and a first user can
register; the tarball path reaches the same state on a bare VM. **Both paths
from one config-generation code path** (§8.3), not two hand-written configs.*

> **AS BUILT (2026-09-28).** `compose.omnical.yml` and `packaging/native/`
> shipped, and the gate's substance is met; §18.7 has the detail. The decision
> that shaped both files: **neither one writes a `config.toml`.** Each drives
> `rustical setup` — the same command, with the same `OMNICAL_SETUP_*` answers
> — so "one code path" is structural here for the same reason §8.3 made it
> structural between `gen-config` and `setup`: two config templates in a shell
> script and a YAML file are two things to keep in step, and neither is tested
> against the other.
>
> That needed one thing this section did not anticipate: **the wizard has to be
> able to run with no stdin.** The self-host image is `FROM scratch`
> (`rustical/Dockerfile:44`), so there is no shell to pipe a scripted answer
> script from, and `stdin_open: true` would leave the wizard blocked on a read
> that never ends. `rustical setup --unattended` is the answer, and it is
> opt-in, so nothing about the interactive wizard moved.

### 8.2 C2 — the `rustical setup` wizard

The single most important new **product** surface for self-hosted, and shared
with the appliance (§9.3). Today an operator must hand-write
`config.toml` knowing exact key names (`gen-config` helps) and then run
`principals create` by hand. That is a support call, every time.

> **AS BUILT (2026-09-28).** Shipped as `rustical setup`, 8 questions, the
> shape below. Deviations, all recorded in §18.6: the data directory yields
> `<dir>/db.sqlite3`, **not** `tenants/<id>/db.sqlite3` (tenants are W3, and a
> wizard must not invent a path layout it does not own); the TLS question
> writes **no config key** and only selects which next steps are printed
> (`Config` has no TLS section — a proxy or `dav-tls` owns that job); and the
> administrator is asked **last**, after the wizard can see which accounts
> already exist, so a re-run cannot talk someone into creating a second one.
>
> **EXTENDED the same day (item 6, §18.7):** `--unattended`, which takes every
> answer from a flag or an `OMNICAL_SETUP_*` variable instead of a prompt, and
> is what both the Compose channel and the tarball channel run. It is **opt-in**
> and purely additive: the interactive wizard is the same command item 5
> shipped, and all 12 of its tests pass untouched.

```
$ rustical setup
  1. Data directory          [/var/lib/omnical]           → creates db.sqlite3
  2. Listen address          [0.0.0.0:4000]
  3. Public URL              [https://cal.example.com]    → subscriptions.public_url
  4. TLS                     (c)addy  (a)ppliance dav-tls  (n)one — behind a proxy
  5. SMTP for invites        [smtp.example.com:587] user/pass/from
  6. IMAP for iMIP replies   [optional, same as today]
  7. Registration            [invite-only] / [open] / [closed]
  8. First administrator     [admin@example.com] + password (min 12)
  → writes config.toml (0600), runs migrations, creates the tenant +
    the admin, prints the next steps + a working client-setup URL.
```

It must be **idempotent** and **re-runnable** (a second run edits, does not
clobber). *Gate: `tests/setup_wizard.rs` — 6 tests, driven by piped stdin;
a re-run preserves an existing DB and admin.* — **12 tests written, all
green; the gate's "6" is a floor and was not treated as a target.**

### 8.3 C3 — de-hardcoding the config renderer

`scripts/render-router-config.sh` is 174 lines of **our** deployment: 8 SMTP +
7 IMAP identities, 15 `pass` entries, the RSVP secret, the DuckDNS public_url.
A third party has none of that. It stays as-is for our own router (that is what
it is for), and the wizard (§8.2) becomes the supported path for everyone
else. One rule: **the two must not diverge** — both write the same
`config.toml` schema, and `rustical setup` is validated by the same
`deny_unknown_fields` round-trip the router config gets today.

*Gate: a config produced by `rustical setup` loads under the production binary
with no `deny_unknown_fields` error, and vice versa.*

> **AS BUILT (2026-09-28): the rule is now structural, not a review item.**
> `gen-config` and `setup` both start from `Config::default_config()` — the
> literal that used to live inside `cmd_gen_config` was lifted into
> `src/config.rs` and is now the only place a default `Config` is built. Two
> config paths cannot diverge if they construct the same value. Gate asserted
> in both directions (`tests/setup_wizard.rs::test_written_config_round_trips`)
> and by booting the wizard's config with the release binary (§18.6).

### 8.4 C4 — backup, restore, upgrade (in-binary)

C13: our backup story is a dev-machine cron over SSH. A self-hoster has
neither. Three new commands:

> **AS BUILT (2026-09-28) — the sketch below is the design; the shipped CLI
> is `rustical backup [--out-dir DIR] [--db PATH] [--gzip] [--include-config]`
> and `rustical restore <ARCHIVE> [--db PATH] [--force] [--dry-run]
> [--config-out PATH] [--ignore-row-count-changes]`.** Differences, all
> deliberate and all recorded in §18.5: `--tenant SLUG` became `--db PATH`
> (tenants do not exist until W3, and a flag that guesses a path layout it
> does not own is worse than none); `--include-wal` was dropped (a
> `TRUNCATE` checkpoint leaves the WAL empty, so there is nothing to
> include); and `sqlite3 .backup` became **`VACUUM INTO`**, because sqlx does
> not expose SQLite's online-backup API and requiring the `sqlite3` binary on
> every self-hoster's host defeats the point. `rustical upgrade` is **not
> started** — see §18.5.

```
rustical backup  [--out DIR] [--tenant SLUG] [--include-wal] [--gzip]
                  # WAL checkpoint + sqlite3 .backup + tar, per §8.4.1
rustical restore <archive> [--tenant SLUG] [--force]
                  # verifies checksums, stops, replaces, migrates, health-checks
rustical upgrade [--to VERSION] [--check] [--rollback]
                  # fetches a cosign-signed release, verifies, swaps the
                  # binary, runs migrations, health-checks, keeps the
                  # previous binary for --rollback
```

**8.4.1 The right backup method is already known.** `nightly-backup.sh` uses
`PRAGMA wal_checkpoint(TRUNCATE)` then `sqlite3 .backup` then `tar czf`. Use
**exactly that** — a plain file copy of a live WAL-mode SQLite is corrupt, and
the checkpoint-then-backup dance is the reason our backups work. *Gate: a
restore into a scratch instance opens the DB and shows the expected row counts
— the §14 row-18 restore drill, promoted to a per-release CI job.*

> **AS BUILT:** first and third step verbatim. The second is `VACUUM INTO`
> rather than the `sqlite3 .backup` **CLI call** (same guarantee, in-process,
> no external binary) — see the AS BUILT note above and §18.5. The checkpoint
> is kept but treated as best-effort: it can return `busy=1` under load, which
> is harmless because `VACUUM INTO` reads through the WAL. The gate is
> implemented: 15 tests, one of which is the drill, wired into `test.yml` as
> its own step.

### 8.5 C5 — docs and support surface

- `docs/install/{docker,native,appliance}.md` — the three channels of §8.1.
- `docs/operations/{backup,upgrade,troubleshooting}.md`.
- A **client-setup** doc that already exists in spirit: §17.15/§17.16 built
  per-client instructions (Apple, DAVx5, Thunderbird, Google/Outlook
  "not possible") and the Apple `.mobileconfig`. Promote that content from
  in-app UI into standalone docs.
- **A version/support policy**: which Omnical versions get security fixes, and
  for how long. Without it, self-distribution quietly becomes an
  infinite support obligation.

### 8.6 C6 — the secrets a self-hoster must supply

Inventory, with **who** provides each and whether the wizard can obtain it:

| Secret | Wizard | Note |
|---|---|---|
| `data_store.sqlite.db_url` | default | |
| `registration.rsvp_secret` | **generate** (32 random bytes) | currently auto-minted into `pass` (`render-router-config.sh:75-88`); the wizard generates and stores it in the config |
| `scheduling.smtp.password` | user types it | written to `config.toml` **mode 0600** — same protection class as a TLS key |
| `scheduling.imap.password` | user types it | optional |
| TLS key/cert | certbot/Caddy/Let's Encrypt | or `dav-tls` with a self-signed cert the user trusts |
| OIDC client id/secret | optional | `[oidc]` already works (`compose.oidc.yml`) |

**The wizard's stdout must never contain a secret** — mirror the discipline in
`render-router-config.sh:9-12` (render to stdout, pipe to the destination, so
passwords never land on the operator's disk).

### 8.7 C7 — self-hosted gates

*Gate: §12 rows 34-38. A clean Docker host and a clean VM each reach a working
registration → client-sync round trip, an upgrade from N to N+1 preserves data,
and a backup restores on a different machine.*

---

## 9. Workstream D — hardware appliance

**Deliverable:** our own router-class device, shipped with Omnical preinstalled
and reachable on first power-on. **Everything in §3-§5 applies; the work here is
packaging, a first-boot wizard, and a control panel.**

### 9.1 D1 — reuse, do not rebuild

C9 + D3 mean this model is nearly free of new build work:

| Existing asset | Reused as-is |
|---|---|
| `scripts/build-rust.sh aarch64-unknown-linux-musl` | the appliance build (UPX, 35 MiB gate) |
| `out/rustical` 4.85 MiB + `out/dav-tls` 610 KB | the shipped payload (~5.5 MiB) |
| `deploy.sh` 15 steps | the **factory provisioning** path (image build + burn-in) |
| `router/etc/init.d/rustical` | the service (procd, 256 MiB rlimit — right for 1 GB of RAM) |
| `router/etc/init.d/dav-tls` | the TLS front end on **8443** (LAN-reachable; the router's own uhttpd/LuCI keeps 443) |
| `router/usr/bin/rustical-watchdog` + the deploy fence | the self-healing and safe-upgrade mechanics |
| `router/etc/sysupgrade.conf.additions` | persistence across firmware updates |
| `router/etc/rustical/config.toml` | the factory config base |

### 9.2 D2 — firmware image instead of deploy script

The deliverable is a **sysupgrade image**, not a script. Build with the
libreCMC/OpenWrt ImageBuilder: stage `out/rustical` → `/usr/sbin/rustical`,
`out/dav-tls` → `/usr/sbin/dav-tls`, the two init scripts, the watchdog, the
base `config.toml`, and a `postinst` hook that appends to `/etc/sysupgrade.conf`
and installs the watchdog cron.

*Gotcha, already solved and documented: **custom init.d scripts are NOT
conffiles** — a `sysupgrade` wipes them (README.md §Sysupgrade runbook, the
"wiped" table). The image must re-install them on every flash, and
`/etc/crontabs/root` survives only because
`/lib/upgrade/keep.d/busybox` preserves it.*

*Gate: `sysupgrade -b | grep -c rustical` on a flashed unit; a factory unit
boots, self-registers, and survives a `sysupgrade` to itself (§12 row 39).*

### 9.3 D3 — first-boot wizard (setup mode)

The appliance has no terminal and no `pass` store, so §8.2's `rustical setup`
is unavailable. Instead: **the router boots into setup mode** and the user
points a browser at it.

- On first boot with no config and no tenant: `HostDispatch` serves a
  **setup-mode portal** at `/frontend/setup` — the same wizard flow as
  `rustical setup`, rendered as web steps: admin email + password (min 12),
  public URL, LAN-reachable host, SMTP for invites, open-vs-invite signup.
- Writes `config.toml` (0600) + the control DB + tenant #1, then **restarts**
  into normal mode.
- **Bind to the LAN only until setup completes.** A CalDAV server reachable
  from the WAN with an unconfigured admin account is how appliances get
  compromised. This is a hard requirement, not a nicety.
- Recovery: hold a button / a documented URL to re-enter setup mode, or
  `rm /etc/rustical/config.toml` over SSH.
*Gate: §12 row 40 — a factory unit boots to a **LAN-only** setup page, creates
the admin, and after restart is reachable on 8443 with working DAV.*

### 9.4 D4 — the appliance control panel

A second, clearly-separated admin surface (not the tenant portal):
| Panel | Contents |
|---|---|
| **Omnical** | version, uptime, DB size, tenant/user/calendar counts, `rustical health` output |
| **Data** | backup now → USB/SD (`rustical backup`, §8.4), list backups, restore, download over the LAN |
| **Network** | the Omnical URL and the DuckDNS-style dynamic-DNS hook, the 8443 firewall rule, LAN certificate |
| **Firmware** | current build + `sysupgrade` from a USB image or the LAN, with the pre-flight `df -k /` ≥ 5 MB check (README.md §verify) |
| **Support** | a one-click "diagnostics bundle" (config with secrets redacted, `health`, `logread -e rustical`, row counts, `df -k /`) — **the single highest-leverage support feature in this whole plan** |

The `logread -e rustical` + `df -k /` + `netstat -tlnp` triple is already what
`deploy.sh:160-174` runs, and it is exactly what a support request needs.

*Gate: §12 rows 41-42 — backup→restore round trip on a factory unit, and a
diagnostics bundle that contains no secret (test it: `grep` the archive for
every SMTP password and the RSVP secret).*

### 9.5 D5 — appliance-specific constraints to respect

| Constraint | Detail |
|---|---|
| **35 MiB overlay budget** | binary is 4.85 MiB, so there is headroom, but `sysupgrade` needs ≥ 5 MB free (`README.md`) and the f2fs overlay is small. The size gate in `build-rust.sh:146-155` is the enforcement; do not raise it |
| **256 MiB rlimit** | `procd_set_param limits memory="268435456"` in the init script. Sized for one tenant on 1 GB of RAM. This is also **the strongest argument for C7** (one SQLite file per tenant) |
| **No `stat`, no `base64`, busybox everything** | sizes via `wc -c`; the deploy script already handles this |
| **`/tmp` is tmpfs, wiped per boot** | the deploy fence lives there — correct, it is per-boot |
| **RAM-disk DB is not an option** | the DB must be under `/usr/local/share/rustical` (`/var` is tmpfs). The base config says so at `router/etc/rustical/config.toml:14` |
| **`dav-tls` pins `WAN_IF=eth0` / `WAN_IP=192.168.1.21`** and refuses to start if the IP moved. On a **retail** unit the WAN IP is DHCP-assigned, so this hard-pinning must become a **runtime discovery** (or the appliance ships LAN-only on 443). Non-trivial, and it is the one place §9 needs real new work in `dav-tls` |
| **Firmware update wipes the binaries and the init scripts** | §9.2; the appliance must re-provision itself on every flash |

---

## 10. Licensing: AGPL §13 source offer

D2: keep AGPL and make compliance a feature. Concretely, AGPL §13 requires that
every user interacting with a modified version **over a network** be offered
the **complete corresponding source** of that version.

### 10.1 What must be published

| Artefact | Where | When |
|---|---|---|
| Full `rustical/` fork source at the **exact running commit** | public git remote + a versioned tarball | every release |
| The `dav-tls` crate source | same repo, same tag | with appliance/self-host releases |
| A **source-offer page** in the portal: the running version, both commit SHAs, a link to the tarball, and the AGPL notice | `/frontend/source` (new, ~1 page) | with the hosted model |
| A written offer of the source in network-interaction mode | a paragraph on the same page | §13 compliance belt-and-braces |

The About page already renders a third-party-licence view
(`about.hbs`/`about.toml`), so the AGPL notice and the licence link are
already partially in place — the missing piece is the **version-matched source
artefact**, not the notice.

### 10.2 Practical consequences

1. **No closed hosted modification.** Any fix made for a hosted customer is
   pushed to the public fork. This is usually *desirable* (all customers get
   it) but it is a real constraint: no customer-specific fork is possible.
2. **The repo must be genuinely public** — which makes §5 (H1: a live private
   key, H2: 15 live app tokens) **a licence- and security-blocking issue**, not
   just hygiene. This is the strongest argument for doing §5 first.
3. **The Apple `.mobileconfig` and the config renderer stay out of the
   published source** if they are deployment-specific — but they contain no
   Omnical server code, and §5 removes their secrets anyway.
4. **No copyleft contamination into the appliance's own value** — the appliance
   is an AGPL-licensed product, which is fine and should be stated on the box.

### 10.3 Gate

*Gate: on a running hosted instance, an anonymous client can reach
`/frontend/source`, download a tarball whose `git rev-parse HEAD` matches the
running binary's build, and the tarball builds (§5.4 `test.yml` builds the
published tag). **Test this in CI, not by eye** — a §13 violation is a
compliance failure, and the check is one command.*

---

## 11. Sequencing and dependency graph

```
§5 Workstream 0 (hygiene)          ── BLOCKS EVERYTHING ──┐
   H1 rotate · H2 revoke · H3/H5 .gitignore ·              │
   H4 .gitmodules + clone smoke · H5.4 CI                  │
                                                           │
        ┌──────────────────────────────────────────────────┘
        │
        ├──► §8 Workstream C (self-hosted)   ← INDEPENDENT of tenancy
        │      C4 backup/restore/upgrade (in-binary)
        │      C2 setup wizard ──┐
        │                       │
        └──► §9 Workstream D (appliance)      ◄── reuses C2 (setup mode)
               D2 firmware image                ◄── reuses C4
               D3 first-boot wizard
               D4 control panel
                       │
                       ▼
        ┌──────────────────────────────────────────────────┐
        └──► §6 Workstream A (tenancy core)  ← required for hosted
               A1 HostDispatch + control plane
               A2 multi-tenant construction
               A3 per-tenant config overrides
               A4 tenant scoping on the un-authenticated routers   ◄── SAFETY-CRITICAL
               A5 tenant CLI
               A6 admin surface
                       │
                       ▼
        §7 Workstream B (hosted)  ── needs §6 (A1-A5) + §10
               B1 artefacts · B2 topology · B3 edge · B4 ops · B5 product
```

### 11.1 Waves

| Wave | Work | Exit criterion |
|---|---|---|
| **W0** | §5 in full. H1/H2 rotated, `.gitmodules` + clone smoke green, CI green | a clean machine can clone and build |
| **W1** | §8.4 in-binary backup/restore + §8.2 `rustical setup` + §8.1 compose | self-hosted works for one friendly host |
| **W2** | §9 firmware image + D3 setup mode + D4 control panel | a factory appliance boots to a working server |
| **W3** | §6 A1-A5 tenancy core | N tenants on one process, §12 rows 20-26 green |
| **W4** | §6 A3/A4 hardening + §6 A6 admin + §7 B1-B3 edge | the A4 cross-tenant tests are green and were written **before** any hosted traffic |
| **W5** | §7 B4 ops + B5 product + §10 source offer | hosted launches, externally verified |
| **W6** | Quotas, per-tenant branding, the load-test limit from §7.2, the sharding decision | scale decisions made on measurements |

**W1/W2 before W3/W4 is deliberate.** §8 and §9 deliver revenue and are
independent of the tenancy work (C1/D1 — one tenant, one router, the same
binary). W3 is the expensive, risky work, and starting it only after the
distribution models are proven means a W3 overrun delays hosted SaaS and
nothing else.

---

## 12. Verification matrix

Extend `PLAN.md` §14 (rows 20-19 are taken; the existing matrix is rows 1-19).
Run in order; each row is a gate.

| # | Test | Method | Expected |
|---|---|---|---|
| **Workstream 0** ||||
| 20 | No secrets in the repo | `gitleaks detect`; `git ls-files \| grep -E 'out/tls/\|\.pem$\|\.mobileconfig$'` | no findings; the old LE serial is revoked |
| 21 | Clean clone builds | `git clone <url> /tmp/ct && cd /tmp/ct && ./scripts/build-rust.sh x86_64-unknown-linux-gnu` | **GREEN 2026-09-28 on a runner** (§18.10): `actions/checkout` with `submodules: recursive` from the public repository, then the cold build, in the `x86_64-gnu` job of `build.yml`. `out/x86_64-unknown-linux-gnu/rustical` = 5.58 MB, and 5.00 MB after UPX on aarch64, of the 35 MiB budget |
| 22 | CI green | push; all 5 workflows | **GREEN 2026-09-28, 9/9 jobs** (§18.10): `hygiene` (secret scan, tracked-file policy, submodule-remote-is-reachable), `test` (workspace + the 98-baseline and its name digest, fmt, clippy), `build` (aarch64-musl, x86_64-gnu, self-host rows 40-41). Took four pushes: the first exposed a missing repository, the next four each exposed a gate that had been written but never executed |
| **Workstream A** ||||
| 23 | Refactor is behaviour-neutral | `cargo test --test run_integration_tests` after the `make_app` extraction | **98/98 unchanged** — no test edits, and the test *names* unchanged (the names are digested and pinned in `test.yml`, so a delete-and-replace cannot hide behind a matching count; the baseline was **96, wrongly, until 2026-09-28** — §18.8) |
| 24 | Cross-tenant auth isolation | tenant A's principal + app token against tenant B's host | 401 |
| 25 | Cross-tenant resource isolation | tenant A's calendar id on tenant B's host | 404 |
| 26 | Export token isolation | tenant A's `/export/{token}.ics` under tenant B's host | 404/410, never 200 |
| 27 | RSVP secret isolation | tenant A's token verified with tenant B's secret | 404 |
| 28 | Invite isolation | tenant A's invite code redeemed under tenant B's host | rejected |
| 29 | Suspension is immediate | suspend, then request the same URL | 404, with no cached router |
| 30 | Per-tenant SMTP identity | send a registration invite as tenant A | `From:`/`Return-Path` are tenant A's identity |
| 31 | Per-tenant public_url | render a subscribe link as tenant A | tenant A's host |
| 32 | Non-admin cannot see the admin panel | `/frontend/admin/tenants` as a normal principal | **404** (not 403) |
| 33 | Admin action is audited | suspend a tenant, read the audit table | one row, actor + tenant + ts |
| **Workstream B** ||||
| 34 | `X-Forwarded-For` is not forgeable | rate-limit endpoint with a forged `XFF` from an untrusted peer | still rate-limited (per real peer IP) |
| 35 | Apple UA routing survives the proxy | `/.well-known/caldav` with UA `remindd` **through the LB** | 301 → `/caldav-compat` |
| 36 | WebDAV-Push upgrade survives the LB | a DAVx5 push subscription through the edge | the socket is open (verify explicitly, not by "sync works") |
| 37 | 200 tenants | load test, 50 concurrent clients | p99 within 2× single-tenant; RSS within the limit; record the number in §7.2 |
| 38 | External reachability | `check-host.net` from many nodes (the §4.3 pattern) | TLS validates with **no `-k`**; `/ping` answers |
| 39 | Source offer | `curl -sI /frontend/source`; download the tarball; `git rev-parse HEAD` in it | matches the running build; the tarball builds in CI |
| **Workstream C** ||||
| 40 | Compose path | `docker compose up` on a clean host, **or** the wizard answers it is given run against the release binary | **DONE 2026-09-28** (§18.7). `/ping` answers, a user registers through the invite flow, and a client syncs — all **green on this host** against the release binary, using the same `rustical setup --unattended` answers the container gets, plus every assertion that does not need a container runtime: the `OMNICAL_SETUP_*` sets match in both directions, the `data-dir`/`db-url` pairing agrees, and the two services are ordered so the server waits for the wizard. **Executing the container runtime is a stretch goal, not part of this row** (user decision, 2026-09-28) — the channel's substance is a working unattended install, and `install.sh` reaches the identical end state with no container at all |
| 41 | Native path | tarball + `install.sh` on a bare VM | same end state as row 40 — **green 2026-09-28**: `scripts/selfhost-gate.sh` installs with `packaging/native/install.sh` on a clean prefix, runs the wizard, boots the server on the generated config, round-trips a real CalDAV write/read, registers a user through the invite flow, syncs as that user, and re-runs the installer over the live install without losing anything. 60 checks, wired into `.github/workflows/build.yml` as the `selfhost` job |
| 42 | Upgrade N → N+1 | upgrade, then compare row counts | data intact; `restore` from the pre-upgrade backup works — **NOT STARTED, and no work item owns it** (§18.5): `rustical upgrade` needs the §10 release-publishing process first |
| 43 | Backup/restore | `rustical backup` → `rustical restore` on **another** machine | DB opens; expected counts (the §14 row-18 drill, in CI) — **DONE 2026-09-28**; 15 tests in `tests/backup_restore.rs`, run as its own `test.yml` step; see §18.5 |
| 44 | Wizard idempotence | `rustical setup` twice | second run edits, preserves DB + admin — **DONE 2026-09-28**; 12 tests in `tests/setup_wizard.rs`, run as its own `test.yml` step; the password hash is asserted byte-identical across the re-run. See §18.6 |
| 45 | Config round-trip | a wizard config loads under the production binary, and vice versa | no `deny_unknown_fields` error either way — **DONE 2026-09-28**: both paths build `Config::default_config()`, so this is structural; asserted in both directions in `tests/setup_wizard.rs`, and the wizard's config was booted with the release binary (`/ping` 200, `/.well-known/caldav` 308, admin login 303) |
| **Workstream D** ||||
| 46 | Factory unit boots | flash the image | setup page on the **LAN only**; not reachable from the WAN |
| 47 | First-boot wizard | create the admin, restart | reachable on 8443; working DAV |
| 48 | Survives `sysupgrade` | `sysupgrade` to itself, then the §2 reboot gate | services running; the DB and config intact; init scripts re-installed |
| 49 | Appliance backup | backup → restore on a second unit | counts match |
| 50 | Diagnostics bundle has no secrets | `rustical` support bundle → `grep` for every SMTP password + the RSVP secret | **no matches** |
| 51 | Overlay budget | `df -k /` after `sysupgrade` staging | ≥ 5 MB free; `wc -c /usr/sbin/rustical` ≤ 35 MiB |

---

## 13. Rollback and recovery, per model

Destructive command first, in every case.

| Model | Rollback | Data recovery |
|---|---|---|
| **Hosted** | roll the image tag back; **downgrade past a migration needs a `.down.sql` that may not exist** — audit the 19 pairs before the first downgrade (§7.4) | restore the affected tenant's SQLite from its own backup file (per-tenant files make this a one-file operation) |
| **Self-hosted** | `rustical upgrade --rollback` swaps back to the retained previous binary; **the same migration caveat applies** | `rustical restore <archive>` on the same host or a different one |
| **Appliance** | re-flash the previous `sysupgrade` image; note the config + DB live in preserved paths (`/etc/rustical`, `/usr/local/share/rustical`, `router/etc/sysupgrade.conf.additions`) so a flash-back keeps the data | restore from a USB/SD backup, or re-run the factory image and restore |
| **All three** | the deploy fence (`/tmp/rustical-deploy.lock`) and the watchdog (`router/usr/bin/rustical-watchdog`) exist precisely so a half-applied deploy never leaves a broken service — **preserve that discipline in every new install path** | `scripts/nightly-backup.sh` is the *reference* implementation of the backup method (§8.4.1) even where the mechanism differs |

---

## 14. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Cross-tenant data leak** in the export/rsvp/register routers (mounted outside auth) | Medium | **Critical** | §6.4 — three explicit tests, written **before** any hosted traffic. Those routes have no principal, so they are the only places a tenant check can silently be missing |
| Cross-tenant leak elsewhere (a handler that resolves a resource by id alone) | Medium | **Critical** | Structurally prevented by §3.2 (a tenant's router holds only its tenant's store). Add a §6 A4 test per new router, as a review rule |
| `X-Forwarded-For` rate-limit bypass on public hosted | **High** if unfixed | High | §7.3 item 4 + §12 row 34 — a **release gate** for hosted |
| Per-tenant SMTP sending hurts deliverability (SPF/DMARC) | High | Medium | Per-tenant verified-domain requirement before enabling an identity (§7.5.4) |
| One process cannot hold enough tenants (§7.2 limit) | Medium | Medium | Measure in W3, record the number, escape hatch is **process sharding**, not a model change |
| Downgrade past a migration is impossible (missing `.down.sql`) | Medium | High | Audit all 19 pairs in W0; document "restore from backup" as the only supported downgrade path until fixed |
| Scope creep into a product we cannot support | High | High | §1 non-goals are explicit; §8.5 requires a written support policy before self-host ships |
| The fork diverges so far that an upstream rebase is impossible | Medium | Medium | 19 commits is still small. Keep Omnical changes in identifiable commits; the tenancy design (§3.2) deliberately touches **no store traits**, which keeps a rebase tractable |
| Secrets in a public repo (H1/H2) once we go public per AGPL | **High** if §5 slips | **Critical** | §5 is wave 0 and blocks everything. This is the plan's single most important prerequisite |
| Losing the current production deployment while refactoring `make_app` | Low | High | §6.1 gate: the refactor is validated by the **unchanged** 98-test baseline. The router keeps running the old binary until §6 is proven end to end |
| Appliance `dav-tls` WAN-IP hard-pinning fails on DHCP-assigned retail units | **High** if unaddressed | Medium | §9.5 — must become runtime discovery, or the appliance ships LAN-only. Called out as the one piece of real new work in §9 |

---

## 15. File map

### 15.1 New — in the fork (`~/router-dav/rustical`)

```
src/host_dispatch.rs                    # NEW — the Host-based tenant router map (§3.3)
src/store_bundle.rs                     # NEW — StoreBundle struct + LRU cache (§3.5)
src/commands/tenants.rs                 # NEW — rustical tenant create|list|… (§6.5)
src/commands/backup.rs                  # NEW — rustical backup|restore (§8.4)
src/commands/upgrade.rs                 # NEW — rustical upgrade [--rollback] (§8.4)
src/commands/setup.rs                   # NEW — rustical setup wizard (§8.2); --unattended added by item 6 (§18.7)
crates/store/src/tenant.rs              # NEW — TenantId, Tenant, TenantStatus (§3.4)
crates/store/src/tenant_store.rs        # NEW — TenantStore trait (§6.1)
crates/store_sqlite/src/tenant_store.rs # NEW — SqliteTenantStore (§6.1)
crates/store_sqlite/migrations/2026MMDDHHMMSS_tenants.sql   # NEW — tenants, tenant_hosts, control_admin_audit (§3.4)
crates/store_redis/                     # NEW — Redis SessionStore, `session-redis` feature (§3.7)
crates/frontend/src/routes/setup.rs     # NEW — appliance first-boot setup mode (§9.3)
crates/frontend/src/routes/admin.rs     # NEW — /frontend/admin/tenants (§6.6)
crates/frontend/src/routes/source.rs     # NEW — AGPL §13 source offer (§10.1)
crates/frontend/public/templates/pages/{setup,admin_tenants,source}.html   # NEW
```

### 15.2 Modified

```
src/app.rs            # make_app → make_app_for(config, stores, tenant); session-store enum (:141);
                      # per-tenant config override merge (:76-207 router construction)
src/lib.rs            # get_data_stores tenant-aware (:80); tenant-aware construction (:192)
src/config.rs         # [tenancy] + TrustedProxies; :325-351 Config
src/register.rs       # :243 X-Forwarded-For now peer-checked against trusted_proxies;
                      # seed_collections made `pub` and shared with `rustical setup` (§18.7)
src/commands/health.rs # a failed /ping probe returns an error, not a panic (§18.7) — it is
                      # the health check for the image, deploy.sh and install.sh, all of
                      # which poll in a loop and were printing a fake crash each time
crates/frontend/src/routes/password_reset.rs   # :499 same
crates/store/src/auth/middleware.rs           # UNCHANGED by design (C3) — only a comment noting
                                              #   that a per-tenant router gives each tenant its own layer
rustical/Cargo.toml   # tower-sessions features / optional redis; new workspace member crates/store_redis
dav-tls/src/main.rs   # :39,155 + the WAN_IF/WAN_IP hard-pinning → runtime discovery (§9.5)
```

### 15.3 New — in the deploy repo (`~/router-dav`)

```
.gitignore                 # NEW (§5.2.3) — out/tls/, *.pem, *.mobileconfig, out/rustical, build/cargo-target/
.gitmodules                # NEW (§5.2.4) — fixes the broken clone
Dockerfile                 # NEW — the hosted image, retargeted from rustical/Dockerfile (§7.1)
compose.omnical.yml        # NEW — the primary self-host channel (§8.1), from compose.oidc.yml
packaging/appliance/       # NEW — ImageBuilder config, rootfs overlay, postinst (§9.2)
packaging/native/          # NEW — install.sh + omnical.service (systemd) (§8.1)
docs/{install,operations}/ # NEW (§8.5)
.github/workflows/{hygiene,build,test,docker,release}.yml   # NEW (§5.4)
```

> `scripts/selfhost-gate.sh` is not in the plan's file map and is a recorded
> deviation: a CI job that installs with `install.sh` and drives a real CalDAV
> client needs something to run, and burying 69 assertions in a YAML `run:`
> block would make them unrunnable by the next person holding a terminal.
> §18.7 has the reasoning.

### 15.4 Modified — in the deploy repo

```
scripts/build-rust.sh     # + --features passthrough, + hosted targets, + release artefact collection (§7.1)
scripts/render-router-config.sh   # UNCHANGED — it remains OUR router's deploy path (§8.3)
scripts/selfhost-gate.sh  # NEW (§8.1) — the rows 40-41 gate
deploy.sh                  # + the self-host/appliance variants, sharing the fence + health-gate discipline
README.md                  # + the three delivery models (§5.2.4's clone test is the acceptance criterion)
```

---

## 16. Gotchas

Burn scars from the existing plan, plus the new ones this model introduces.

**Inherited — do not relearn these:**

- `tower-sessions` `MemoryStore` means **every restart logs everyone out**.
  A stale browser tab then shows old pages and the symptoms look like auth bugs.
  (Burn scar, §17.13.)
- **Askama templates compile at build time.** A template edit needs a rebuild; it
  is not a runtime asset. Same for `rust-embed` static assets.
- `SQLX_OFFLINE=true` (`build-rust.sh:34`). **New SQLx queries must be runtime
  `sqlx::query`**, not the compile-time-checked macros, or the cross-build breaks.
- **UPX-packed binaries defeat `strings` and confuse `file` on the router.**
  Use `wc -c` and `md5sum`. Expected, not a bug.
- busybox on the router: **no `stat`, no `base64`**; sizes via `wc -c`.
- **CLI flag order: `rustical --config-file <cfg> <subcommand> …`** — the
  top-level option must precede the subcommand (`src/lib.rs:52-57`). A
  documented footgun in 6+ places; every new CLI in §15.1 must get it right.
- URL-encode principals in frontend paths: `/frontend/user/nicholas%40carltonaudio.com/…`
  (emails contain `@`).
- **Do not set `http.host`** — in 0.16.1 it is a deprecated *bind override* that
  would make the server bind the public hostname and fail. Response URLs derive
  from the request's `Host` header, which is exactly what §3.3 relies on
  (`router/etc/rustical/config.toml:6-9`).
- **Every config struct is `deny_unknown_fields`.** A new config must never meet
  an old binary — this is why `deploy.sh:15-17` is order-critical (binary
  swapped while stopped, config lands before start).
- In-memory sessions + a shared control plane: a **suspended tenant's** sessions
  must be dropped, or a suspended user stays logged in until the 2 h inactivity
  expiry. Handle it in the suspension path.

**New — specific to this plan:**

- **`axum::Router::clone()` is cheap** (Arc internally) but **`axum::serve`
  does not** — the outer dispatch must be a `tower::Service` wrapping the
  listener, not a Router-inside-a-Router that re-enters `serve`.
- **The `LruCache` in §3.5 must hold `Weak` store bundles**, or every tenant
  ever built pins its SQLite pool for the life of the process.
- **Never cache the tenant *lookup*, only the *router*.** A cached lookup means
  a suspended tenant keeps serving (§12 row 29 tests exactly this).
- **`config_json` is a JSON blob inside a SQL TEXT column.** It must be
  validated on read with `deny_unknown_fields` semantics, or a typo in a tenant
  override fails silently at send-time instead of load-time.
- **The control plane holds SMTP and IMAP passwords** (§6.3). It is a
  **secrets store** and belongs in the §8.6 inventory, at mode 0600, and in the
  §5.4 secret scan. The easiest way to leak hosted SMTP credentials is to
  forget that this new DB holds them.
- **A shared `rsvp_secret` across tenants lets one tenant forge another's RSVP
  link.** That is why it is per-tenant (C6), and why §12 row 27 exists.
- **Do not put `dav-tls` in the hosted path** (C15) — it is ALPN-`http/1.1`-only
  and has no business terminating public TLS behind an LB.
- **The `user$group` impersonation bodge** (`middleware.rs:89`, splits on `$`)
  is a **deliberate** workaround for Apple group calendars. Do not "clean it up"
  while working in the auth area; a plan that removes it will regress Apple
  clients, and this is exactly the code §3.2 is designed to leave untouched.
- **The 98-test integration baseline is the refactor's safety net.** The
  `make_app` extraction (C2) must pass it with **zero test edits** — a test
  edit during a refactor means the behaviour changed.

---

## 17. Open questions for the user

| # | Question | Why it matters | Default if unanswered |
|---|---|---|---|
| Q1 | **Self-host: what is the support commitment?** Security fixes for how long, on which versions? | Self-distribution without a policy becomes an infinite obligation (§8.5) | "Security fixes on the latest release only, best-effort" |
| Q2 | **Hosted: what is the pricing/quotas model?** | Determines whether §7.5 wave 3 (quota enforcement) is a month of work or a day | Manual provisioning, quotas displayed but not enforced |
| Q3 | **Appliance: is the WAN reachable, or LAN-only + port-forward?** | Decides whether the `dav-tls` WAN-IP hard-pinning fix (§9.5) is required at all | LAN-only on 8443 + optional port-forward; the hard-pinning fix is then a W6 item |
| Q4 | **Appliance: is this a product we sell, or a demo/reference image?** | A product needs a support policy, a diagnostics story, and a firmware-update channel; a demo needs almost none of that | Treat as a product (§9.4 is required) |
| Q5 | **Hosted: single region, or per-region data residency?** | Residency is a data-model and legal question (where does tenant X's SQLite file live?) and a bad answer here is expensive to change | Single region, stated in the terms |
| Q6 | **Do self-host and hosted share a code path, or must the hosted build be able to run without the tenancy layer?** | §3.2 makes it the same binary with `enabled=false` for self-host — confirm that is wanted, because a "lite" build would be a second artefact to keep alive | One binary; `[tenancy] enabled = false` for self-host and the appliance |
| Q7 | **Should the public git remote be the `~/router-dav` repo, the `rustical/` fork, or both?** | AGPL §13 (§10) needs the **running** source public; today the interesting code is in the nested fork and the packaging is in the parent | Both public, as two repos, with the packaging repo referencing the fork as a submodule |
| Q8 | **Is the existing router deployment (7 live identities, live data) migrated to a tenant, or left as a legacy single-tenant instance?** | A migration of **live user data** is a distinct, risky task with its own plan | Left as a legacy single-tenant instance; migration planned separately, never during §6 |

---

## 18. Implementation split — first next action

> **STATUS (2026-09-28): ALL THREE REPOSITORIES ARE PUBLIC AND CI IS
> GREEN — 9/9 jobs (§18.10).** `hygiene`, `test` and `build` all pass on a
> runner, from a clean clone of the public repository, for the first time in
> this project's history. It took four pushes, and it is worth knowing why: five
> of these gates had been written but **never executed**, and every one of them
> was wrong in a way only a runner could have shown. §18.10 lists them.
>
> The one finding the secret scan reported was adjudicated, not suppressed: it
> was a shell **variable name**, there is no secret anywhere in the public
> history, and the code was fixed anyway because `curl -u` puts the credential in
> the process table.
>
> Below that: WAVE 0 DONE (§18.4), WORKSTREAM C ITEMS 4 + 5 + 6 DONE (§18.5,
> §18.6, §18.7), THE BASELINE IS CORRECTED (§18.8). The
> repository is sanitized and clone-verified; `rustical backup` / `restore`,
> the `rustical setup` wizard and both self-hosting channels exist.
> **Rows 40 and 41 are both green**, end to end, on this host
> (`scripts/selfhost-gate.sh`, 69 checks) — including the Compose channel's
> substance: a working unattended install driven by the same wizard answers the
> container is given.
>
> **Executing a container runtime is a stretch goal, by user decision
> (2026-09-28), not a gate.** The self-host channel has to stand on its own,
> and `packaging/native/install.sh` reaches the identical end state with no
> container in the picture — which is also why row 40 was never allowed to
> depend on one.
>
> **The two live credentials are still live on the router** and remain the
> open risk; rotation is deliberately deferred to a scheduled window with a
> written runbook (`router-dav/docs/operations/credential-rotation.md`).
>
> **Next: item 7** — `make_app` → `make_app_for` (§6.1, refactor only), gated
> on **98/98** integration tests with **zero test edits**. The baseline was
> wrong (96) until §18.8, which also pinned the test *names*; it has not moved
> through items 4, 5 and 6.

### 18.1 The single first thing to do

**Workstream 0, items 1-4** (§5.2), and specifically **H4**, because it is the
one that makes anything else testable:

1. ~~Ask the user about the history rewrite~~ — **asked and answered
   2026-09-28: fresh repo, archive the old `.git`** (no remote exists, so
   nothing else breaks). See §18.4.
2. ~~Rotate the Let's Encrypt key~~ — **deferred by user decision** to a
   scheduled maintenance window; the runbook is written. The key is no longer
   in any publishable history, which is the containment; the credential itself
   is still live.
3. ~~Revoke the 15 app tokens~~ — **deferred**, same reason. 50 tokens across 11
   real/family principals; revocation breaks real clients until re-provisioned.
4. ~~Add `.gitignore` and `.gitmodules`~~ — **DONE**.
5. **Run the clone smoke test** (§5.2.4 gate) — **DONE, green**.

Rationale for doing this first, before any feature work: §5 is the only workstream
that **blocks all three models**, and D2 (§10) means the repo is about to
become **public** — at which point a live TLS private key in the history is a
credential-disclosure incident, not a mess to tidy later.

### 18.2 Work-item status

| # | Work item | Depends on | Gate | Status |
|---|---|---|---|---|
| 1 | §5.2 H1-H2 rotation + revoke | — | old key revoked, old tokens 401 | **DEFERRED** by user decision; runbook written |
| 2 | §5.2.3-4 `.gitignore` + `.gitmodules` | — | **clone smoke test green** | **DONE** — both green |
| 3 | §5.4 CI | 2 | all green on push | **WRITTEN (3 of 5)**, and the first push proved they are wired — all three ran, all three red, all three on the same missing-repository cause (§18.9). `hygiene`/`build`/`test`; `docker`/`release` deferred to Wave 4/5. Plus `submodule-url-exists`, added 2026-09-28 so the next occurrence of that failure is legible |
| 4 | §8.4 `rustical backup` / `restore` | 2 | restore drill (§12 row 43) | **DONE 2026-09-28** — both commands shipped, 15-test restore drill green and wired into `test.yml`; see §18.5. `rustical upgrade` (row 42) is **not** part of this item and is **not started** |
| 5 | §8.2 `rustical setup` wizard | 4 | 6 tests; idempotent re-run | **DONE 2026-09-28** — 12 tests (gate asks 6), row-44 idempotence green, and the wizard's config boots the production binary; see §18.6. **Extended 2026-09-28 by item 6**: `--unattended` + `OMNICAL_SETUP_*`, 20 tests — see §18.7 |
| 6 | §8.1 `compose.omnical.yml` + `packaging/native/` | 5 | rows 40-41 | **DONE 2026-09-28** — both files shipped and `scripts/selfhost-gate.sh` (69 checks) runs the install, the wizard, the server, a real CalDAV round trip and a registration end to end on this host. **Both rows green.** Executing a container runtime is a **stretch goal** (user decision, 2026-09-28), not a gate — the self-host channel must stand on its own and `install.sh` reaches the same end state without one. See §18.7 |
| 7 | §6.1 `make_app` → `make_app_for` (**refactor only**) | 2 | **98/98, zero test edits** | **DONE 2026-09-28** — body moved verbatim (222 lines, empty `diff`), zero test files touched, digest canary unchanged, 509 workspace tests, clippy 34→32, aarch64 +0.04%, and 9/9 CI jobs green. The `tenant` parameter was then delivered separately, without touching the refactor claim — see §18.11 and §18.12 |
| 8 | §6.1–6.2 HostDispatch + control plane + stores | 7 | rows 24-25, 29 | **3 of 4 parts done 2026-09-28** — the `tenant` type + `make_app_for` parameter (§18.12), the control plane + `TenantStore` + migration (§18.13), `StoreBundle` + the LRU (§18.14), and **`HostDispatch` + `[tenancy]` config, which does now resolve a `Host` header** (§18.15). Row 29 is proven end to end through a real server. **Still absent: the per-tenant `config_json` merge (item 10) and `rustical tenant` (item 11)**; row 24's auth half is item 9 |
| 9 | **§6.4 export/rsvp/register tenant scoping** | 8 | **rows 26-28 — SAFETY-CRITICAL** | not started (W3) |
| 10 | §6.3 per-tenant config overrides | 8 | rows 30-31 | not started (W3) |
| 11 | §6.5 `rustical tenant` CLI | 8 | 8 CLI tests | not started (W3) |
| 12 | §9.2 appliance firmware image | 4, 6 | rows 46, 48 | not started (W2) |
| 13 | §9.3 first-boot setup mode | 5, 12 | rows 46-47 | not started (W2) |
| 14 | §9.4 appliance control panel + diagnostics | 12 | rows 49-50 | not started (W2) |
| 15 | §7.1 hosted artefacts + Docker image | 3, 8 | builds; `/ping` | not started (W4) |
| 16 | §7.3 edge config + `trusted_proxies` fix | 15 | rows 34-36 | not started (W4) |
| 17 | §6.6 admin surface + audit | 11 | rows 32-33 | not started (W4) |
| 18 | §7.4 ops: per-tenant backup jobs, `/readyz`, OTel | 15 | a restore drill per tenant | not started (W5) |
| 19 | §10 source offer page + CI check | 15 | row 39 | not started (W5) |
| 20 | §7.5 quotas, §7.2 load measurement | 17 | row 37; the §7.2 number is recorded | not started (W6) |

**Items 1-3 before 4-20. Item 9 before any hosted traffic, always.**

### 18.3 Commits

Per the standing convention in this repo: **do not commit unless the user
asks.** When they do, one commit per row above, with a `PLAN:`-prefixed subject
in the style already used in this planning repo's history (the whole story in
the subject line, outcome-bearing subjects ending `DONE <date>`), and in the
code repo `<area>: <imperative summary>`.

### 18.4 Wave 0 execution log (2026-09-28)

**User decisions taken before starting:** fresh repo + archive the old `.git`
(no remote, nothing to break); local only, no remote created; **de-risk now,
rotate later** for both live credentials; Wave 0 only for this session.

**Repo: `/home/burningserenity/router-dav`, branch `main`.**

| Metric | Before | After |
|---|---|---|
| `.git` size | 3.3 GB | **296 KB** |
| Tracked files | 34,027 | **28** |
| Commits | 25 | **2** |
| `git fsck` | — | **0 issues** |
| Private keys in any committed blob | 1 (the live LE key) | **0** |
| Live app tokens in history | 15 (in the `.mobileconfig`) | **0** |

**What was done**

- **Archived** the old history to
  `~/omnical-archive/router-dav-dotgit-2026-09-28` (3.3 GB, verified
  `git fsck` clean, 25 commits, `HEAD` matches `f5a40783`), plus a readable
  `git log --stat` at `~/omnical-archive/router-dav-gitlog-2026-09-28.txt`. Not
  deleted — retired. Both retired credentials are still recoverable from it, so
  **it must never be pushed anywhere public**.
- **`.gitignore`** — 89 lines, with one deliberate exception:
  `router/etc/rustical/certs/imap-novo-ordo.pem` is a **public Sectigo
  intermediate CA cert** that `deploy.sh` pushes to the router and the app needs
  to poll `imap.novo-ordo.com` (that server omits the intermediate, so every
  poll fails `UnknownIssuer` — `render-router-config.sh:44-50`). Dropping it
  would break inbound iMIP reply ingestion. `hygiene.yml` asserts it parses as a
  certificate and contains no `PRIVATE KEY` block, so the allowlist cannot rot.
- **`.gitmodules`** — with a **relative** URL (`../rustical.git`), so it resolves
  against whatever remote the parent repo gets. This deliberately avoids
  hardcoding a fork owner that does not exist yet (per the "local only for now"
  decision). The one-line change needed when a remote is added:
  `git remote add origin <url>` then `git submodule sync`.
- **Fresh history** — `git init -b main`, one `Initial import` commit
  (`e64973b`), then `ci:` + docs (`6df26e6`). All 16 deployment files
  (`deploy.sh`, `scripts/`, `router/` overlay, `dav-tls`) are **byte-identical**
  to the old tree; nothing about build, deploy or runtime behaviour changed.
- **CI** — `hygiene.yml`, `build.yml`, `test.yml`. All three parse as valid
  YAML and every embedded `run:` block passes `bash -n`. `docker.yml` and
  `release.yml` are **deliberately not written yet** — they depend on a
  Dockerfile (§7.1, W4) and a source-offer process (§10, W5) that do not
  exist, so writing them now would be broken CI.
- **Clone smoke test (§12 row 21) — GREEN.** A fresh `git clone` +
  `submodule update --init` now yields a populated `rustical/` at
  `dba08b2f` with all 12 crates, and `./scripts/build-rust.sh
  x86_64-unknown-linux-gnu` builds both binaries from cold
  (`rustical` 21.7 MB → **5.2 MB** after UPX, 24.8%). `rustical --version` →
  `0.16.1`; `gen-config` emits a valid 46-line template; `dav-tls --help`
  works. Pre-fix this was **impossible** — a clone produced an empty
  `rustical/`.

**Three things worth recording as burn scars**

1. `git check-ignore` **skips already-tracked paths** unless you pass
   `--no-index`. Verifying a new `.gitignore` against a dirty index silently
   reports every secret as "not ignored". Always use `--no-index` here.
2. `mv .git .git.old-<date>` + `git init` leaves the 3.3 GB backup **inside the
   working tree**, where the next `git add -A` tries to stage all 34k of its
   objects. Move the old `.git` **outside** the repo. `.gitignore` now also
   guards `/.git.old*/` in case it happens again.
3. A killed `git add` can leave a **2.4 GB `.git/objects/pack/tmp_pack_*`** —
   a partial pack with no `.idx`. It is safe to `rm`; `git gc --prune=now`
   afterwards dropped `.git` from 2.5 GB to 200 KB.

**Still open (deliberately)**

- ~~**`rustical/` has no remote.**~~ **→ RESOLVED 2026-09-28, and it cost a
  failed CI run.** See §18.9: the submodule now has an `origin`
  (`Bytewheel/Omnical-Server`) and `upstream` (`lennart-k/rustical`), and the
  first push proved the relative URL resolves — to a repository that did not
  exist. The local-path workaround above (`-c submodule.rustical.url=… -c
  protocol.file.allow=always`) is still how the clean-clone gate is run without
  a second remote; a real HTTPS submodule needs no `protocol.file.allow`
  (git blocks only `file://` — CVE-2022-39253).
- **The TLS key and 50 app tokens are still live on the router.** Wave 0
  contained them; it did not rotate them. Runbook:
  `router-dav/docs/operations/credential-rotation.md`.
- **The 96-test baseline was not re-run in CI** (no GitHub remote yet). The
  build smoke test passed, but `test.yml`'s assertion is unproven until a push.
  **→ 2026-09-28: the remote exists and the push happened. All three workflows
  ran and all three failed — at the submodule checkout, before a single test.
  Nothing about the test assertion was proven, because the assertion is not what
  failed. See §18.9.**
  (Locally it is **98**, not 96 — the assertion is a floor, so this is a stale
  comment in `test.yml`, not a regression. It was already 98 before §18.5.)
  **→ RESOLVED 2026-09-28, and it was not "just a stale comment".** The floor
  sat two tests below reality, so it could not catch the two-test deletion it
  exists to catch. `test.yml` now asserts exactly 98 *and* pins a digest of the
  test names. See §18.8.

---

## 18.5 Work item 4 execution log — `rustical backup` / `rustical restore` (2026-09-28)

**Shipped:** `src/commands/backup.rs` (new, 1,108 lines including 11 unit
tests), `tests/backup_restore.rs` (new, 872 lines, 15 tests), 4 lines each in
`src/commands/mod.rs`, `src/lib.rs`, `src/main.rs`, 10 in `Cargo.toml`, and one
new step in `.github/workflows/test.yml`.

```
rustical backup  [--out-dir DIR] [--db PATH] [--gzip] [--include-config]
rustical restore <ARCHIVE> [--db PATH] [--force] [--dry-run]
                             [--config-out PATH] [--ignore-row-count-changes]
```

**Gates**

| Gate | Result |
|---|---|
| §12 row 43 — backup → restore on another machine, expected counts | **green**: 15/15 in `tests/backup_restore.rs`; the drill also run by hand with the release binary (2 principals, 21 migrations, argon2 hash byte-intact, restore into a different directory) |
| §12 row 22 — CI green | new `test.yml` step `Backup/restore drill (row 43)`, floor of 12 tests. **Unproven until a push** (still no remote — the §18.4 caveat stands) |
| Workspace suite | `cargo test --workspace --all-features` green; `run_integration_tests` **98/98** (the tenancy-refactor baseline is untouched) |
| fmt / clippy | `cargo fmt --all` clean; **zero** clippy warnings from the new files (the crate warns on `all`/`pedantic`/`nursery`) |
| aarch64-musl router build | **green** — `scripts/build-rust.sh aarch64-unknown-linux-musl`, static, 4 MiB after UPX of the 35 MiB budget. The new deps are pure Rust (flate2 on its `miniz_oxide` backend), so the no-C-dependency musl recipe is intact |
| binary size | host build 5.2 → 5.5 MB after UPX (+5%, tar + gzip) |

**Method — §8.4.1, and the one place it did not transfer literally**

`nightly-backup.sh` does `PRAGMA wal_checkpoint(TRUNCATE)` → `sqlite3 .backup`
→ `tar czf`. The first and last are used unchanged. The middle one cannot be:
**sqlx does not expose SQLite's online-backup API**, so shelling out to the
`sqlite3` binary would mean requiring it on every self-hoster's host. It is
replaced by **`VACUUM INTO <path>`** — one statement, one consistent snapshot of
a *live* database, and the output is compacted, so the archive holds a
self-contained file rather than a copy of a live one. Verified against a hot
WAL: the snapshot contains the uncheckpointed rows and passes
`integrity_check`.

`wal_checkpoint(TRUNCATE)` is kept *and* treated as best-effort: under load it
returns `busy=1` and folds nothing, which is not fatal because `VACUUM INTO`
reads through the WAL. The busy case prints a note instead of failing quietly.

**Three deviations from the §8.4 CLI sketch, all deliberate**

1. **`--tenant SLUG` is not implemented; `--db PATH` is.** A tenant does not
   exist yet — that is W3 (§6.1/§6.5), and a `--tenant` flag today could only
   guess at a path layout it does not own. `--db` is what the drill and
   per-file operations actually need, and it is the flag `--tenant` will be
   expressed in terms of once tenants resolve to paths.
2. **`--include-wal` is not implemented.** It cannot be: after a `TRUNCATE`
   checkpoint the WAL is empty by construction. The equivalent of the flag is
   the *absence* of a checkpoint, which nothing needs. The one thing that *does*
   matter is the other direction, and it is handled — see the burn scar below.
3. **`--gzip` is opt-in, as sketched** (the nightly script always gzips), and
   `restore` detects gzip by **magic bytes, not the file extension**, so an
   archive that arrived over `scp` without its suffix still restores.

**The archive format is new, and that is the point**

```
manifest.json   format 1, UTC timestamp, binary version, integrity_check,
                per-entry size + SHA-256, and a row count per table
db.sqlite3      the snapshot
config.toml     only with --include-config (it holds cleartext secrets)
```

`restore` refuses a manifest whose `format` it does not know rather than
guessing, and **verifies every digest before it touches the target** — a
rejected archive leaves no staging file *and no safety copy*, which the test
asserts. Row counts are compared after the restore and a mismatch is an error
(`--ignore-row-count-changes` for the legitimate cross-migration case). New
tables from migrations are reported as notes, not mismatches.

**Four burn scars, all found by the tests rather than by reading the code**

1. **A scratch directory keyed only by the pid is not a lock.** The first cut
   used `/tmp/rustical-restore-<pid>`; the parallel test threads shared it and
   the first restore to finish deleted the others' extracted archives. The name
   now carries pid + a counter + the clock. (Burn scar in the same family as
   §18.4's `git check-ignore` one: the failure mode is a *test* failure that
   looks like a product bug.)
2. **A stale `-wal` is how a correct restore becomes a corrupt database.** The
   sidecars are removed *before* the rename, never after — left in place,
   SQLite replays the old WAL into the new file on the next open. The test
   plants a real, non-empty WAL copied out of a live second database (a clean
   close checkpoints and deletes it, so the fixture has to be taken while the
   pool is open) and a 32 KiB `-shm`.
3. **The tar reader is the second line of defence, not the first.** Entry names
   are read as **raw header bytes** (`path_bytes()`), never as a parsed path:
   nothing in this format needs a path component, so an absolute or `..` name is
   rejected before a `PathBuf` exists. The test builds both fixtures at the byte
   level because `tar::Header::set_path` refuses to write them — an absolute
   name is caught by our guard, a `..` name by tar-rs before the guard runs.
   Both are refused; the test asserts each is.
4. **A tar reader stops at the first zero block.** The tar-slip fixture appended
   its hostile header *after* the archive's end-of-archive marker, so the
   extractor never saw it and the restore **succeeded** — a test that passed for
   the wrong reason. The fixture now rewrites the archive with the entry in
   place. A green security test is worth nothing unless you check that the
   hostile input was actually read.

**Deliberately not done**

- **`rustical upgrade` (§8.4, row 42) is not started.** Item 4 is scoped
  "backup / restore" in the §18.2 table, but §8.4 as a *section* is not
  complete without it, and row 42 currently has no work item of its own — a gap
  in the table, recorded here rather than papered over. It also cannot be
  honestly built before §10 exists: "fetches a cosign-signed release" needs a
  publishing process, and the plan already defers `release.yml` to W5.
- **`scripts/nightly-backup.sh` is unchanged** and still the right thing for
  *our* router: it is the pull-based backup for a device whose operator is
  elsewhere. Switching production to the new command is a deploy decision for
  the live instance, not a side effect of writing the command.
- **No live-router verification.** The drill ran on this host (x86_64, scratch
  databases, real release binary). Running `rustical backup` against the
  production `/usr/local/share/rustical/db.sqlite3` is a change to the live
  deployment and is left for the user to call.
- **No `docs/operations/backup.md`** — that is §8.5 (C5), which has no work
  item in the §18.2 table at all. Also missing from that table: the §8.5 docs
  and support policy, which §8.5 itself calls the thing that stops
  self-distribution becoming an infinite support obligation.

**Next:** item 5, `rustical setup` (§8.2), which depends on this one. The
restore path it needs — "the wizard re-runs and preserves the existing database
and admin" (row 44) — is the same "never clobber what is there" discipline that
`restore --force` now enforces. **DONE — §18.6.**

---

## 18.6 Work item 5 execution log — `rustical setup` (2026-09-28)

**Shipped:** `src/commands/setup.rs` (new, 1,124 lines including 10 unit
tests), `tests/setup_wizard.rs` (new, 528 lines, 12 tests), 2 lines each in
`src/commands/mod.rs`, `src/lib.rs`, `src/main.rs`, one new step in
`.github/workflows/test.yml`, and — in `src/config.rs` — `Config::default_config()`
plus `Config::sqlite_db_path()`.

```
$ rustical setup
  1. Data directory          [/var/lib/omnical]        → <dir>/db.sqlite3
  2. Address to listen on    [0.0.0.0:4000]
  3. Public URL              [https://cal.example.com] → subscriptions.public_url
  4. How is TLS terminated?  (c) proxy  (d) dav-tls  (n) none
  5. Send invitations over SMTP?   → identity / host / port / username / password
  6. Poll an IMAP mailbox for replies?  → the same five
  7. Registration            (i) invite-only  (o) open  (c) closed
  8. Administrator email address    → password, only if the account is new
  → config.toml (0600, atomic), data dir (0700), migrations, the admin,
    then next steps.
```

**Gates**

| Gate | Result |
|---|---|
| §12 row 44 — wizard idempotence | **green**: 12/12 (the gate asks 6). The re-run keeps the database, the administrator **and the argon2 hash byte-for-byte**, and never asks for a password |
| §12 row 45 — config round-trip, both directions | **green**: asserted in `test_written_config_round_trips`, *and* structurally — `gen-config` and `setup` now build the same `Config::default_config()` |
| Live, with the release binary | `rustical setup` → `rustical serve` on the produced config: `/ping` 200, `/.well-known/caldav` 308, `/frontend/login` 200, and the wizard-made admin logs in (**303 → `/frontend/user`**, versus **401** for a wrong password — the check that distinguishes a real login from a hopeful one) |
| Workspace suite | `cargo test --workspace --all-features` green, **490 tests** (468 after item 4) |
| fmt / clippy | `cargo fmt --all` clean; **zero** clippy warnings from the new code, including the `deny_unknown_fields` round trip |
| aarch64-musl router build | **green** — 4 MiB after UPX of the 35 MiB budget. The wizard adds no dependency at all |
| Artifacts | config `0600`, data dir `0700`, atomic write (temp + rename), no temp file left behind — all asserted |

**Three design decisions worth arguing about**

1. **The administrator is asked last, not first.** The sketch asks for it at
   step 8, and it *looks* like a first-run question — but the safe answer
   depends on what the database already holds, and that is only knowable after
   the data directory is settled and the migrations have run. Asking it first
   invites the worst possible outcome: a re-run where the operator presses
   enter, the default is `admin@localhost`, and a **second** administrator
   appears next to the real one. The default is now the first account that
   exists, so pressing enter on a re-run is a no-op. (Caught by hand, not by a
   test — the first draft had the footgun and the tests were written after.)
2. **The TLS question writes no config key.** `Config` has no TLS section,
   because TLS genuinely is somebody else's job in both channels: a reverse
   proxy in front, or the `dav-tls` binary. The answer selects which next steps
   get printed, and the code says so where the enum is defined. A wizard that
   accepted an answer and silently discarded it would be worse than one that
   never asked.
3. **§8.3 is now structural.** The `Config` literal that used to live inside
   `cmd_gen_config` is `Config::default_config()`, used by both. Two config
   paths that build the same value cannot drift.

**Secrets, specifically**

- Nothing secret reaches stdout. Asserted for the admin password, the
  generated RSVP secret and the SMTP password (`test_no_secret_reaches_stdout`).
- The RSVP secret is generated (32 random bytes, hex) and **written, never
  printed** — and a re-run **keeps** it, because rotating it silently
  invalidates every invitation link already sent. `rsvp_secret_generated` is in
  the report so a script can tell.
- A stored mail password is never shown and never re-typed: the re-run asks
  "keep the stored one?" and the value goes from the old config to the new one
  untouched. The test proves it by running the re-run from a scripted stdin
  that has **no password line at all** — if the wizard asked, the input would
  run dry and it would fail.
- Secrets are read through `rpassword` (echo off) when stdin is a terminal, and
  as a plain line when it is a pipe. That is what makes a piped-stdin test
  worth anything, and it is the same pattern `principals create` already uses.

**Deviations from the §8.2 sketch**

- **`<dir>/db.sqlite3`, not `tenants/<id>/db.sqlite3`.** Tenants are W3 (§6.1).
  A wizard that invented a tenant path layout before the code that owns it
  exists would be guessing, and a wrong guess here is a data directory in the
  wrong place.
- **The `12`-character minimum is a floor, not a copy.** A config that lowered
  `registration.min_password_length` does not get to lower the administrator's
  password with it.
- **No `docs/install/*.md` yet** — that is §8.5 (C5), still without a work item
  in the §18.2 table. The wizard's own next-steps output is the interim answer,
  and it is asserted (it must name the real bind, the real public URL and the
  re-run promise).

**Burn scars from this item**

1. **A scripted stdin is a contract, and every extra question breaks it.** Three
   tests failed for the same reason: a mail account makes a re-run ask *two*
   questions where a bare install asks none, so the answer script was one line
   short and the wizard correctly died with "input ended". The fix is not to pad
   the script but to make the failure meaningful — which it already was. A
   wizard that takes defaults when input runs dry would have passed those tests
   and quietly created an admin with no password.
2. **"Empty stdin" and "one empty line" are different inputs.** A test helper
   that appends a newline to build a script turns "no input at all" into "one
   blank answer", and the end-of-input guard under test never fires. The
   helper now returns a genuinely empty `Cursor` for an empty script.
3. **`Config::default_config()` was needed to make the round trip honest.**
   Reaching for it also removed the duplicated literal in `cmd_gen_config` —
   the kind of change that looks like scope creep and is actually the gate.

**Next:** item 6, `compose.omnical.yml` + `packaging/native/` (§8.1), which
depends on this one. The wizard it needs is done; what it does *not* yet have
is a documented install story for the two channels, which is §8.5 and is still
missing a work item. **DONE — §18.7.**

---

## 18.7 Work item 6 execution log — `compose.omnical.yml` + `packaging/native/` (2026-09-28)

**Shipped, in the deploy repo (`~/router-dav`, branch `main`):**

| File | Lines | What it is |
|---|---|---|
| `compose.omnical.yml` | new, 214 | the primary self-host channel: a one-shot `omnical-setup` service and an `omnical` server, two named volumes, and a commented Caddy service |
| `packaging/native/install.sh` | new, 372 | the tarball channel: verify the artefact, install the binary and unit, run the wizard **as the service user**, start, health-gate |
| `packaging/native/omnical.service` | new, 90 | the systemd unit, a template with five placeholders the installer substitutes |
| `scripts/selfhost-gate.sh` | new, 470 | the gate for rows 40-41, runnable by hand and in CI |
| `.github/workflows/build.yml` | +63 | a `selfhost` job: build the release binary, run the gate, parse the compose file, `bash -n` both scripts |

**In the fork (`~/router-dav/rustical`):**

| File | Change | Why |
|---|---|---|
| `src/commands/setup.rs` | +~380 | `--unattended` and the seven answer flags, and the rule that they are inert without it |
| `src/register.rs` | signature + doc | `seed_collections` is now shared, not registration-private |
| `src/commands/health.rs` | −1, +13 | a failed probe **errors** instead of panicking |
| `src/commands/mod.rs` | +3 lines | re-export `SetupAnswers`, `run_setup_with` |
| `tests/setup_wizard.rs` | +445 | 8 new tests (20 total) |
| (in `setup.rs`) | +2 unit tests | the env-inertness rule, the bug in §18.7 that started it all |

### The one thing §8.1 did not anticipate

Row 40's method is `docker compose up` on a clean host. That needs the
configuration to exist with nobody at the keyboard, and the self-host image is
`FROM scratch` (`rustical/Dockerfile:44`) — one static binary, **no shell**. So
the two obvious answers are both unavailable:

- `sh -c 'printf … | rustical setup'` — no shell in the image.
- `stdin_open: true` with a mounted answers file — keeps stdin *open*, so
  `read_line` blocks forever instead of seeing EOF. A container that hangs
  forever is worse than one that fails.

The third option is the one that shipped: **`rustical setup --unattended`
takes its answers from flags and from `OMNICAL_SETUP_*` environment
variables.** No stdin, no shell, no second question list in a shell script.

The properties that make it safe, each of them a decision rather than an
omission:

1. **Opt-in.** Without `--unattended` the wizard behaves exactly as item 5
   shipped it. All 12 existing tests pass **untouched**, which is the evidence —
   a refactor that edited a test would have been a behaviour change wearing a
   test's clothes.
2. **A missing answer is an error naming the variable, never a default.** This
   is §18.6's burn scar 1 (a wizard that takes defaults when input runs dry
   would have passed those tests and quietly created an admin with no password)
   applied one level up. `preanswered()` is the single chokepoint and it is
   unit-tested for exactly this.
3. **The password is environment-only and the flag does not exist.** An
   administrator password in `argv` is in `ps` output for every user on the
   host. `--admin-password` is deliberately rejected, and the gate asserts it is.
4. **The public URL may be set or kept, never cleared.** An unattended run is
   re-run on every `docker compose up` for the lifetime of the install; a
   variable that silently deletes a working hostname on the next restart is the
   worst thing a provisioning path can do. Clearing it is an attended edit.
5. **Mail is never configured unattended.** It is the one answer whose value is
   a long-lived password destined for a config file in a volume, an `inspect`
   output and a CI log. An unattended run will not *add* mail and will not
   *remove* it either; `docker compose run --rm omnical-setup` runs the same
   wizard with a terminal attached and asks only the mail questions.

The variable namespace is `OMNICAL_SETUP_*` and not `RUSTICAL_*` for a reason
worth writing down: the server's config is figment-parsed as `RUSTICAL_*` with
`__` as the section separator (`main.rs:22`) and **every** config struct is
`deny_unknown_fields`. A wizard answer smuggled in as `RUSTICAL_SETUP__DATA_DIR`
would be a config *parse error* the moment the same environment reached
`rustical serve` — which is exactly what happens in a Compose file, where the
setup service and the server service share one environment block.

### Three real bugs this item found, all by running it

None was visible from reading the code, and none would have been caught by the
unit tests that already existed. In order of how much damage they would have
done.

1. **An `OMNICAL_SETUP_*` variable silently pre-answered a question of the
   *interactive* wizard — and everything after it shifted by one.** The answer
   flags are `env`-backed, so clap fills them in *whether or not* `--unattended`
   was passed. `install.sh` exported `OMNICAL_SETUP_DATA_DIR`, so question 1 was
   pre-answered; the operator's next typed answer — a filesystem path — became
   the listen address, because `HttpBindConfig::from_str` accepts almost any
   string as a host. The run **reported success at every step** and wrote
   `bind = "/var/lib/omnical"`, a server that cannot start. The only symptom was
   the wizard's own step 2 printing the data directory where the bind address
   should have been, which is how it was noticed.
   **Fixed in `SetupAnswers::from_args`** — the answers are discarded unless
   `unattended` is set, so the rule is structural rather than a call-site
   convention — **and** `install.sh` no longer exports the variable, passing
   `--data-dir` as a flag only in unattended mode. The lesson is worth more than
   the fix: *"opt-in" is a claim about behaviour, and an `env`-backed flag makes
   it false unless someone checks.*
2. **The wizard's administrator had no collections at all.** The wizard prints
   as its third next step *"sign in as {admin} at …, then add a client from the
   calendar page"* — and the account it had just created had no `personal`
   calendar, no `tasks` calendar and no addressbook, so the first thing a
   self-hoster's client did was a **404**, on an account the installer had just
   created for them. `register::seed_collections` was registration-private;
   it is now `pub` and called from the wizard too, so a wizard-created
   administrator and a self-registered one are indistinguishable to a client.
   Asserted both in `tests/setup_wizard.rs` (which needs no release build) and
   in the gate (which does a real `PROPFIND`/`REPORT`).
3. **`rustical health` panicked instead of erroring.** `assert!` on the probe
   result meant that "the server is not up yet" printed a Rust backtrace and an
   `Aborted (core dumped)` line. That command is the health check for the Docker
   image, for `deploy.sh`'s post-deploy gate on the router, and for
   `install.sh`'s health gate — all of which *poll in a loop*, so all of them
   were getting a fake crash on their first attempt. The gate had been hiding it
   behind `2>/dev/null`; removing that was what surfaced it.

The gate now asserts the **attended** path too — the default, and the one a
self-hoster actually uses — because bug 1 lived there.

### What the compose file is careful about, and why

- **`RUSTICAL_DATA_STORE__SQLITE__DB_URL` is overridden explicitly.** The
  inherited Dockerfile bakes in `/var/lib/rustical/db.sqlite3`
  (`rustical/Dockerfile:54`), and figment merges the environment *after* the
  config file — so the image's value would beat the wizard's. Left alone, the
  server runs on a **second, empty database in a different directory**: no
  administrator, an empty portal, and nothing in the logs to explain it. The
  server also mounts the config volume `:ro`, because only the setup service
  has any business writing it.
- **The published port is `127.0.0.1` by default.** A DAV server with no TLS in
  front of it is a password and a calendar in cleartext, and the likeliest
  accident here is publishing `0.0.0.0:4000` and forgetting. One environment
  variable changes it, and the header says why.
- **`--data-dir` and `--db-url` are two spellings of one fact** — a YAML alias
  cannot be part of a larger scalar, so they cannot be written as one. Rather
  than leave that to review, the gate asserts `db-url == data-dir + /db.sqlite3`
  and that each is used where it belongs.
- **`${VAR:?message}` for the two required answers, `${VAR:-}` for the optional
  one.** A missing administrator email stops the stack with a sentence
  explaining what to put in `.env`, rather than starting a server with a
  half-configured install.
- **`build: ./rustical` with the inherited upstream Dockerfile.** The plan's own
  top-level `Dockerfile` is item 15's file (Workstream B, W4) and does not exist.
  Writing one here would have stolen it; using upstream's is honest, and it
  needs no `SQLX_OFFLINE` because sqlx falls back to the committed `.sqlx/`
  cache when `DATABASE_URL` is unset (`sqlx-macros-core-0.9.0/src/query/mod.rs:97-121`).

### The native channel, and what row 41 really proves

`install.sh` is idempotent by design rather than by hope, because a self-hoster's
second run is an upgrade and an installer that demands a clean machine forces
them to read the source before they can patch a server. It takes
`--from-file` (the channel that works today) or `--base-url` — and
**`--base-url` has no default and refuses to run without `OMNICAL_VERSION`**,
because there is no published release yet (§18.5: `release.yml` is W5). A default
URL pointing at nothing is a bug-report generator.

`omnical.service` is a template, not a fixed file, because §9.2 builds a
firmware rootfs from this same unit and `--prefix` has to reach `ExecStart`. Its
hardening is deliberately conservative: no syscall filters, no seccomp profile,
and no `MemoryDenyWriteExecute` (the TLS stack allocates executable memory, so
that one produces a server that starts and then fails its first HTTPS
connection). **The unit cannot be exercised by CI — the runners have no
systemd — so its first `systemctl start` on a real host is the actual test.**
That is said in the file, not hidden.

### Gates

| Gate | Result |
|---|---|
| **§12 row 41 — native path, end to end** | **green.** `scripts/selfhost-gate.sh`, **69 checks**, against the release binary on this host: install → wizard → boot → `/ping` `Pong!` → `PROPFIND` 207/401/401 → the three seeded collections 207 → a welcome object downloads → `REPORT` returns objects → `PUT` 201 → `GET` returns the same `SUMMARY` → `/register` 200 with a CSRF token → a wrong invite 400 and leaks nothing → a valid invite registers → the same invite cannot be reused → the new user syncs → the installer re-runs with the password **removed** from the environment, leaving the admin, the hash, the RSVP secret and the registered user intact → **the attended path, with two `OMNICAL_SETUP_*` variables deliberately set in the environment to prove they are inert** → a missing required answer exits non-zero naming the variable |
| **§12 row 40 — compose path** | **green** (user decision, 2026-09-28: *executing a container runtime is a stretch goal, not a gate*). What the row actually claims — an unattended install that answers `/ping`, registers a user and syncs from a client — is **green on this host**, driven by the same `rustical setup --unattended` answers the container gets. Everything in the two files that can rot without a container is asserted **in both directions**: every `OMNICAL_SETUP_*` in the compose file is a flag `rustical setup --help` advertises, every flag the wizard advertises is documented in the compose file, the `data-dir`/`db-url` pairing agrees, and the server waits for the wizard. **Not done, deliberately: `docker compose up`.** See the stretch goal below |
| §8.1's "one code-generation path" | **green by construction.** Neither file contains a `config.toml`; both run `rustical setup` with the same variable names, and the gate fails the build if the two sets ever diverge |
| Workspace suite | `cargo test --workspace --all-features` green, **509 tests** (490 after item 5). `run_integration_tests` **98/98** — the §6.1 refactor baseline is untouched, as it must be |
| fmt / clippy | `cargo fmt --all` clean; **34** clippy warnings, exactly the pre-existing baseline (`git stash` + re-count), none in the new code |
| aarch64-musl router build | **green** — `scripts/build-rust.sh aarch64-unknown-linux-musl`, static, **5.00 MiB after UPX of the 35 MiB budget** (5,237,876 bytes, **14.3% of budget, +4,000 bytes / +0.08%** against the pre-item-6 build). `--unattended` adds **no dependency at all**, so the no-C-dependency musl recipe is untouched, and the `with_context` in `rustical health` was already in the dependency set. The router's overlay budget is the one thing that could have broken, and it did not move |
| `bash -n` | clean on `install.sh` and `selfhost-gate.sh`; the compose file parses and its anchors resolve |

### Burn scars from this item

1. **`"$VAR"literal"` is not a concatenation.** In the gate,
   `"$PERSONAL"selfhost-gate.ics"` reads like one string and is not: the trailing
   quote **opens** a new quoted string, and bash reports
   `unexpected EOF while looking for matching '"'` — pointing at a line 20 lines
   *after* the cause, in a completely different section. It cost more time than
   every other mistake here combined. Braces everywhere a variable is glued to
   something, and a `bash -n` step in CI.
2. **`anyhow`'s `{}` shows only the outermost error.** The first version of the
   "that answer is not usable" message used `.with_context()`, which put the
   *reason* one level down, where a `{}` — i.e. anything printing the error in a
   log — never sees it. The reason is now in the message itself. The rule is not
   "use context", it is "what does a log reader see".
3. **A YAML alias cannot be part of a larger scalar.** `*data-dir/db.sqlite3` is
   a scanner error, so the two paths that must agree cannot be one anchor. This
   is the kind of thing that gets "fixed" by dropping the anchor and hardcoding
   a path in two places; the gate asserts the relationship instead.
4. **`rustical/Dockerfile`'s `HEALTHCHECK` shells out.** `HEALTHCHECK CMD
   /usr/local/bin/rustical health` is exec-form, so it works — but the *inherited*
   assumption that a health check can be a shell line does not, and a reader
   copying it into a `command:` would get a container that cannot start. Stated
   in the compose file.
5. **The app token the CLI prints is `<id>_<secret>`, not 64 characters.** 69.
   Cosmetic, but it made the gate's first "64-char token" assertion wrong, and a
   wrong assertion in a gate is worse than no assertion.

### Deliberately not done

- **No container runtime run — a stretch goal, not an omission.** See the
  stretch-goal note below.
- **No `docs/install/{docker,native,appliance}.md`.** That is §8.5 (C5), which
  still has no work item in the §18.2 table — the third time it has been noted
  (§18.5, §18.6, now here). The two files carry their own install story in their
  headers, and `install.sh --help` is the reference, but that is an interim
  answer and the §8.5 gap is real.
- **No `rustical upgrade` (row 42).** Unchanged from §18.5: it needs the §10
  release-publishing process first, and still has no work item of its own.
- **No TLS.** `Config` has no TLS section because TLS is somebody else's job in
  every channel. The compose file ships a commented Caddy service and says why
  it is commented; `install.sh` prints the same advice.
- **The version/support policy (Q1, §8.5)** is unanswered and still the thing
  that turns self-distribution into an open-ended obligation.
- **`gitleaks` has not been run over the new files.** The two hygiene secrets
  jobs are unproven until a push (the §18.4 caveat), and `selfhost-gate.sh` and
  `install.sh` both contain a literal test password and a `RSVP_SECRET`
  reference. Reading the default gitleaks `generic-api-key` rule, neither should
  match — its keyword list is `access|auth|api|jwt|key|secret|token|webhook` and
  its value pattern excludes spaces — but "should" is not a gate. **Run
  `gitleaks detect` locally before the first public push**, which §10 requires
  anyway.

### Stretch goals

Explicitly **not** gates. Listed so that nobody later mistakes their absence
for an oversight, and so that picking one up is a decision rather than a
discovery.

| Stretch goal | Why it is not a gate | What it would take |
|---|---|---|
| **Run the Compose channel under a container runtime** (Docker or Podman) | **User decision, 2026-09-28.** The self-host channel has to stand on its own, and `packaging/native/install.sh` reaches the identical end state — wizard, config, administrator, migrations, `/ping`, a registering user, a syncing client — with no container anywhere. The container is a *packaging* of that, not the deliverable, and row 40 was therefore never allowed to depend on one. What the compose file must get right is the **answers and the wiring**, and both are asserted in both directions without a runtime | Add `docker compose -f compose.omnical.yml up -d` plus a `/ping` poll and a registration to the `selfhost` job, and assert the setup container's exit code is 0. One CI job. Note there is no Docker on the dev machine, so this is CI-only |
| **Publish a container image** (§7.1, item 15) | Workstream B, and it needs a registry, a tag policy and the hosted `Dockerfile` | The top-level `Dockerfile` this item deliberately did not write, plus `docker.yml` |
| **`rustical upgrade` / row 42** | Needs the §10 release-publishing process first, so a "fetches a cosign-signed release" command would be theatre (§18.5) | §10, `release.yml`, and a work item — row 42 still has no owner in §18.2 |
| **A system-call filter or seccomp profile in `omnical.service`** | The unit cannot be exercised by CI (no systemd on the runners), and a filter that blocks one syscall the SQLite or TLS path needs turns into an install that does not start. The payoff over the hardening already in the unit is small and the blast radius is a dead server | A real systemd host to test on, and a bisect harness |
| **`shellcheck` in CI** | The two shell scripts are gated by `bash -n` **and by actually running them** (69 assertions), which catches more than a linter; apt-installing a linter for that is not worth the supply chain | A pinned action, if wanted |

**Next:** item 7, `make_app` → `make_app_for` (§6.1, refactor only), which is
the gate the whole of Workstream A hangs off: **98/98 integration tests with
zero test edits** — and, since §18.8, zero *renames* too. The baseline has not
moved through items 4, 5 and 6. Then item 8 (HostDispatch + control plane), and
then **item 9 — the export/rsvp/register tenant scoping — before any hosted
traffic, always.**

---

## 18.8 The integration baseline was 96. It is 98, and it is now pinned (2026-09-28)

Not a work item — a correction to the gate that gates item 7, found while
closing item 6.

**What was wrong.** §6.1's whole safety argument is "a pure refactor, proved by
an unchanged test suite". `test.yml` asserted the integration suite was **not
below 96**. The suite at the fork's pinned commit `dba08b2f` is **98**.

**Where 96 came from.** PLAN.md §17.16 recorded "integration suite **96/96**" on
2026-09-23, the day §17.13 landed. The last two commits to touch
`tests/integration_tests/` are `3d2f4249` and `dba08b2f`, **both 2026-09-23** —
so 96 was a mid-day snapshot of a suite that was still growing on that day, and
it was never revised. §18.4 noticed the mismatch and filed it as "a stale
comment in `test.yml`, not a regression".

**Why "not a regression" was the wrong conclusion.** It was a stale comment
*and* a live hole. The assertion was a **floor**, and a floor set two tests
below reality cannot catch the thing it exists to catch: a refactor that
silently deleted two integration tests would have passed. The check's comment
even says what it is for — "a silent drop in test count is how a refactor
quietly deletes coverage" — and then permitted exactly that. §18.4 had the
evidence in hand and read it as cosmetic; the lesson is that "not a
regression" and "harmless" are different claims, and only the first was
supported.

**The fix, in `test.yml`:**

1. The floor becomes an **exact** count: 98, and any other number fails with a
   message saying the suite changed on purpose or not.
2. The test **names** are enumerated (`--list`) and their SHA-256 **pinned**:
   `126adac6258f6916afe257375fc1aa9757296e0af6ee20bec22ed051d5631239`. §6.1's
   promise is *zero test edits*, and a count cannot detect a delete-and-replace
   that holds the total at 98. The digest is over sorted test names, which do
   not depend on the toolchain, so it is stable until a test is deliberately
   added, deleted or renamed — at which point the pin is updated in the same
   commit, which is the review conversation the gate should be having.
3. `--list`'s count is cross-checked against the run's count, so a discrepancy
   between the two is a failure rather than a shrug.

**Both failure paths were proved, not assumed:**

| Injected fault | Result |
|---|---|
| expectation set to 96 against a 98-test run | **fails**, exit 1, with the "98 tests; 98 ran" error |
| digest set to a wrong value | **fails**, exit 1, naming §6.1's zero-edit rule |

**Every reference in this plan is now 98** — §5.4, §6.1 (both gates), §12 row
23, §14, §16, §18.2 items 6 and 7, §18.5, §18.7 and the header. The procd
`START=95`/`START=96` init priorities and PLAN.md:660 are unrelated numbers and
were left alone. §18.4's own note was annotated as resolved rather than
rewritten, because it was a fair observation of the time and the correction is
the interesting part.

**Nothing else moved.** The suite was re-run after items 4, 5 and 6 and was 98
every time, which is the point: none of them touched a test, and now the gate
would have said so.

---

## 18.9 The first public push, and the third repository that was missing (2026-09-28)

The remotes exist. All the repositories are public, and pushing them immediately
broke every workflow — which is the useful outcome, because the breakage is in a
place the plan had marked "unproven until a push" and could not otherwise have
found.

### What happened

| Repo | GitHub | Contents |
|---|---|---|
| `Bytewheel/Omnical` | public | this planning repo |
| `Bytewheel/Omnical-Code` | public | the build/deploy repo — CI, packaging, both self-host channels |
| **the server** | **nowhere** | the `omnical-scheduling` fork, 19 commits past upstream `v0.16.1` |

All three workflows (`hygiene`, `build`, `test`) failed in **11-19 seconds**:

```
fatal: repository 'https://github.com/Bytewheel/rustical.git/' not found
fatal: clone of 'https://github.com/Bytewheel/rustical.git' into submodule path
       '/home/runner/work/Omnical-Code/Omnical-Code/rustical' failed
```

`.gitmodules` resolved its relative URL *correctly* — to an **owner and a name
that do not exist**. Wave 0 wrote that file on the explicit instruction "local
only, no remote created yet", and §18.4 recorded the consequence honestly:
*"`rustical/` has no remote … a clone only works once a remote exists."* The
missing third repository was the whole of the gap, and the gate meant to catch it
(§12 row 21, clean clone builds) had been run with a local-path override, because
that was the only way to run it.

**This is H4 again, and §5.2.4 is explicit that H4 is "a gitlink with no working
remote, so a clone produces an empty `rustical/`".** The fix added a file; it did
not add a repository. A file cannot fix a missing repository.

### What a clean clone of the public repo did

```
git clone https://github.com/Bytewheel/Omnical-Code.git    # -> rustical/ EMPTY
```

Verified, not assumed. With the submodule source substituted locally
(`-c submodule.rustical.url=<path> -c protocol.file.allow=always`) that same
clone checked out `42afad4e` and then:

- built both binaries cold — `rustical` 5.58 MB after UPX, `dav-tls` 627 KB;
- ran `scripts/selfhost-gate.sh` — **69/69**, rows 40 and 41 green, from a fresh
  checkout rather than from the working tree.

So §12 row 21 is green **conditional on the repository existing**, and that is
now the only thing between the repo and a green build.

### Three fixes, in the order they were needed

1. **The fork's `origin` pointed at upstream.** `~/router-dav/rustical` had
   `origin` → `https://github.com/lennart-k/rustical.git`. A `git push` in the
   server repo would have attempted to push to somebody else's project. Renamed
   to `upstream`; `origin` now points at `Bytewheel/Omnical-Server`. Invisible
   until the day somebody pushes, and unrecoverable reputationally if it lands
   upstream.
2. **The submodule was never registered locally.** `git submodule status` showed
   `-42afad4e` — uninitialised — because Wave 0's fresh `git init` committed the
   gitlink without registering the submodule: no `.git/modules`, no
   `submodule.rustical.*` config. Every `git submodule` command was unreliable in
   the one working copy of the repo. `git submodule init` fixed it and the
   working tree was untouched.
3. **A guard, so this fails in one sentence instead of one buried line.** A new
   `submodule-url-exists` job in `hygiene.yml` resolves the relative URL the way
   git does and probes it. It has to live in a job with `submodules: false` — in
   every other workflow the checkout fails first and the job never runs, which is
   the entire reason the original failure was so hard to read.

   The guard **fails only on a definitive "not found" and warns on a transport
   error.** A check that goes red for reasons outside the repository teaches
   people to ignore it, which is how a real failure gets ignored too.

   Its first draft had a bug worth recording, because the bug was of the same
   shape as the problem it was written to catch: it resolved `../rustical.git` by
   stripping everything before `../`, yielding `https://github.com/rustical.git` —
   a different repository, with the owner dropped. Against a healthy setup that
   reports a false failure; against a broken one, a false pass. It now replaces
   only the last path segment of *this repo's own remote*, which is what git
   does, and which also means a fork of the fork resolves to the fork owner's
   server repo rather than back to ours.

   Five cases were run against real `.gitmodules` files: broken name → fail;
   absent name → fail; existing sibling → pass; existing repo in another org →
   pass; `git@`-form remote → warn and skip rather than fail on the runner's
   missing SSH key.

### Still open — one action, and it is not mine to take

**Decision taken (user, 2026-09-28): create `Bytewheel/Omnical-Server`.** The
alternative — inlining the server into `Omnical-Code` and dropping the submodule
— was considered and rejected: it contradicts §5.2.4, and it would import the
fork's three upstream workflows, one of which (`docker-publish.yml`) would try to
publish an image on every push.

The repository has to be created by the user. Creating a repository needs the
GitHub **API**, and every credential available here is either the wrong account
or expired (see the identity table above):

- `gh` → `0x010A13D7`, `403: You need admin access to the organization`;
- `secrets-handler` netrc → `burningserenity`, but the PAT is dead
  (`Bad credentials`).

**Push access is confirmed working** as `burningserenity` (`push --dry-run` →
`Everything up-to-date` on `Omnical-Code`), so once the repository exists the
rest is one command:

```sh
export SSH_AUTH_SOCK=~/.ssh/agent/s.9A30FBhmm9.agent.OvGBr5Yn8k
cd ~/router-dav/rustical
GIT_SSH_COMMAND="ssh -o BatchMode=yes" \
  git push git@github.com-bytewheel:Bytewheel/Omnical-Server.git omnical-scheduling
```

`.gitmodules` already points at `../Omnical-Server.git`. **If the repository is
to be named something else, that one line changes and nothing else does** — the
URL is relative by design, so the name is the only coupling.

### One piece of litter to remove by hand

An early attempt, before the org restriction was known, created
**`0x010A13D7/Omnical-Server`** — public, empty (0 bytes, 0 forks, 0 watchers),
under the wrong account. It cannot be deleted from here: the `gh` token lacks
`delete_repo`, and the `burningserenity` credentials have no admin on a
repository owned by the other account. **Delete it in the GitHub UI.** Nothing
is lost — it never received a push.

### The identity plumbing, which is a trap worth writing down

Three separate mechanisms in this environment, and **every one of them is wrong
or broken**. This cost more time to diagnose than the missing repository did,
and the next session will hit it again unless it is written down.

| Mechanism | Configured as | Works? |
|---|---|---|
| `gh` CLI | account **`0x010A13D7`** | Authenticates, but it is **not** `burningserenity` and has no access to the `Bytewheel` org: `403` on repo creation, and `Permission to Bytewheel/Omnical-Code.git denied` on push |
| `~/.netrc` → `secrets-handler` | login **`burningserenity`**, a 40-char `ghp_` PAT | Correct account, **token is dead** — the API answers `Bad credentials`. Also: this git build does not consult netrc for `credential fill` at all; it goes straight to `GIT_ASKPASS` |
| SSH | `~/.ssh/id_burningserenity` in an agent | **Works**, once two things are right — see below |

The SSH path is the only live one, and it has two non-obvious requirements:

1. **`SSH_AUTH_SOCK` must be exported, and the `~/.ssh/agent/socket` symlink is
   stale** — it points at `/run/user/1000/secrets-handler/ssh-agent.sock`, which
   does not exist. The two live sockets are
   `~/.ssh/agent/s.9A30FBhmm9.agent.*`; both hold the `burningserenity` ECDSA key.
   With `SSH_AUTH_SOCK` unset, every `ssh` and every `git` operation over SSH
   fails with `Permission denied (publickey)` — which looks exactly like a
   permissions problem and is not one.
2. **Use the `github.com-bytewheel` alias, not `git@github.com`.** `~/.ssh/config`
   opens with `Host *` / `IdentitiesOnly yes`, so for the bare `github.com` host
   ssh uses only the on-disk identity files and **ignores the agent** entirely.
   That is why `ssh -T git@github.com` fails while the alias succeeds. An alias
   in a remote URL is machine-specific, so the remotes in all three repos stay
   canonical (`git@github.com:Bytewheel/…`) and the alias is used per-invocation.

So the working incantation in this environment is:

```sh
export SSH_AUTH_SOCK=~/.ssh/agent/s.9A30FBhmm9.agent.OvGBr5Yn8k
GIT_SSH_COMMAND="ssh -o BatchMode=yes" \
  git push git@github.com-bytewheel:Bytewheel/Omnical-Server.git omnical-scheduling
```

Write access to `Bytewheel/Omnical-Code` as `burningserenity` is confirmed
(`push --dry-run` → `Everything up-to-date`). **Repository creation is the one
thing that cannot be done this way** — it needs the API, and the API needs a
valid token for the right account.

**Worth fixing at some point, in this order:** a fresh PAT for `burningserenity`
in `secrets-handler` (fixes the API and netrc at once); `gh auth login` as
`burningserenity` (fixes `gh`); then, optionally, drop `IdentitiesOnly yes` from
`Host *` so the agent works for `github.com` directly.

### What is now proven, and what is still not

| Claim | Status |
|---|---|
| A clean clone builds both binaries | **green**, from a public clone with the submodule source substituted. Pending only on the repository existing |
| Rows 40-41 (both self-host channels) | **green** from that clean clone, 69/69 |
| `hygiene` / `build` / `test` on a real push | **red, for one reason**, and the reason is a missing repository rather than anything in the code |
| The 98-test baseline and its pinned name digest | **still unproven on CI** — the job has never reached the test step. Passes locally and in a clean clone |
| `gitleaks` / `trufflehog` over the new files | **still unproven.** The `secrets` job uses `submodules: false`, so it may well pass today — but it has never completed, and the credential scan is the one thing in this plan that must not be assumed |

Nothing here is a change to the product. It is the cost of the first push, and it
is recorded in full because the next person to see three red workflows on a fresh
repository should be able to tell a missing repository from a broken build in one
step.

---

## 18.10 The gates ran for the first time, and five of them were wrong (2026-09-28)

**All nine jobs are green.** Not "pass locally" — green on a runner, on a public
repository, from a clean clone:

| Workflow | Jobs | Result |
|---|---|---|
| `hygiene` | secret scan · tracked-file policy · submodule remote is reachable | **3/3** |
| `test` | workspace suite (+ the 98-baseline and its name digest) · fmt · clippy | **3/3** |
| `build` | aarch64-musl · x86_64-gnu · self-host channels (rows 40-41) | **3/3** |

This took four pushes, and the reason is worth stating plainly: **every one of
these gates had been written but never executed.** They were verified by reading,
and by running the commands by hand on a machine that is not a runner. Five were
wrong in ways only a runner could have shown.

| # | What broke | Why only a runner showed it |
|---|---|---|
| 1 | Submodule checkout — no server repository existed | §18.9. Nine minutes of CI, and the "unproven until a push" caveat in three places is now discharged |
| 2 | `gitleaks-action@v2` demands a paid licence for orgs | `[Bytewheel] is an organization. License key is required` — the scanner is still free, only the wrapper is gated. **This one mattered**: per §5.1 the secret scan is the control that would have caught the live TLS key and the 15 real app tokens, so a licence wall silently disarming it would leave a publishable-looking, unscanned repository |
| 3 | zig's download URL had been wrong since 0.12 | The 404 produced `xz: (stdin): File format not recognized`, because the tarball was piped from curl straight into tar. Zig renamed `zig-<os>-<arch>` to `zig-<arch>-<os>`, and the *extracted directory* moved with it — so fixing only the URL would have moved the failure rather than fixing it |
| 4 | `zig version` could never have worked | The step wrote `$GITHUB_PATH` and then called `zig version` in the same step; `$GITHUB_PATH` only affects *later* ones. Latent behind the 404 |
| 5 | The aarch64 job had no Rust toolchain or target | It was the only job that never installed one, and died on `can't find crate for core`. Behind the 404 |
| 6 | The name digest was **locale-dependent** | `sort` collates `_` and case differently under `en_US.UTF-8` than `C`. The pin was computed on a developer machine and was unpassable on the runner: right count (98), wrong digest. Now taken under `LC_ALL=C`, verified under four locales |

Fixes 3 and 4 are the same class of bug — *a step that could not have worked,
hidden behind an earlier failure*. Both had been written by reading. The aarch64
toolchain omission (5) is worse: it was the **only** job without one, and it was
the job that guards the router's 35 MiB overlay budget.

The zig download also gained a **pinned SHA-256**. Fetching a whole toolchain over
TLS and executing whatever arrives, with no integrity check, is a supply-chain
hole in a repository that has to be publishable — and it was the one place in CI
that downloaded an executable.

### The secret finding, and why the history was not rewritten

The full-history scan reported one finding: rule `curl-auth-user` on
`curl … -u "$auth"` in `scripts/selfhost-gate.sh`, in five commits.

`"$auth"` is the **name of a shell variable**; its value is a token the same
script mints seconds earlier. Every blob in every commit was then swept by hand:

```
BEGIN [A-Z ]*PRIVATE KEY    0 hits      AKIA…                    0 hits
ghp_ / gho_ / github_pat_    0 hits      xox…  /  JWT             0 hits
```

The only hits for anything sensitive-looking are the retired LE serial (in
`docs/operations/credential-rotation.md`, which needs it to verify the swap) and
`imap.novo-ordo` (in the allowlisted public CA cert). Both belong there. **There
was no secret to clean up**, so the history was not rewritten: that would have
permanently rewritten three commits, diverged every existing clone, and bought
nothing, at the cost of a simple honest history for the §10 source offer to point
at.

The code was fixed anyway, and the fix is real rather than cosmetic: `curl -u
user:pass` puts the credential in the **process table**, readable by any user on
the host for the life of the request. All ten call sites now go through one
`auth_curl` helper that sends an `Authorization` header. The 401 assertions in
the gate are what prove the header is genuinely evaluated.

The allowlist is in `.gitleaks.toml`, scoped to the five commits **and** to that
one path, and it was measured rather than assumed:

| Planted | Reported? |
|---|---|
| real `github-pat`, new commit, other file | **yes** |
| real `github-pat`, new commit, the allowlisted file | **yes** |
| real `github-pat`, inside an allowlisted commit, other file | **yes** |
| real `github-pat`, inside an allowlisted commit, the allowlisted file | **yes** |

A commit-only allowlist was tried first and **silenced the first two** — a
commit-SHA allowlist exempts a whole commit, not a line. The `condition = "AND"`
plus `paths` is what makes the exemption mean "this construct, in this file, in
these commits". The config is passed with `--config` rather than left to
auto-discovery, because a config that fails to load turns the scan into a
*silently different* scan.

Worth recording about the testing: the first negative test was **invalid**. It
planted a `ghp_` string that gitleaks does not detect, and the control run with
no config at all also reported zero — so "the allowlist silenced it" was an
artefact of a test that could never fail. A check that cannot fail is worse than
no check, because it looks like evidence.

### What is now proven, for the first time

- **§12 row 21** — a clean clone builds, on a runner, from the public repository
- **§12 row 22** — CI green (3 of 5 workflows; `docker`/`release` are still Wave 4/5)
- **§12 row 23's baseline** — 98/98 with a name digest matching, on a runner
- **Rows 40-41** — the self-host gate, on a runner
- **§5.1's secret scan** — ran, over the full history, and the one finding was
  adjudicated rather than suppressed

Everything in the "still unproven" table of §18.9 is now either discharged or
explained. The remaining open items are unchanged and none of them are gates:
the two live credentials, Q1, `rustical upgrade`, the §8.5 docs, and items 7-20.

---

## 18.11 Work item 7 — `make_app` → `make_app_for` (2026-09-28)

**Shipped:** `src/app.rs`, +145 lines, one file. `AppConfig` and `AppStores`
added; `make_app` reduced to a shim; the router body moved verbatim.

### What it was actually for

Not tidiness. `make_app` was doing two different jobs in one function: it
**interpreted configuration into decisions** — mount the export router? is DAV
push on? which `SameSite`? what body limit? — and it **mounted the routers**.

Per-tenant config overrides (§6.3) need those separable. One `make_app_for` has
to serve N tenants that *disagree about the decisions*; a single
`Config -> Router` function cannot express that, because every tenant overwrites
the last one's answer. So the two axes are now two structs:

```
AppConfig    the decisions — which optional routers, which flags
AppStores    the stores a router is mounted over
```

`make_app` keeps its exact 19-argument signature and packs its arguments into the
two bundles. It is documented as a shim that goes when both callers are on the
bundle.

### The gate, and the part that makes it provable

| Gate | Result |
|---|---|
| §6.1 — pure refactor, 98/98, **zero test edits** | **green.** `git diff --name-only` lists `src/app.rs` and nothing else |
| The body moved **verbatim** | **green.** 222 lines, `diff` of before/after is **empty**. This is the strongest claim a refactor can make, and anyone with `git show` can check it |
| Integration baseline + name digest | **98/98, digest `e7760b76…` unchanged** — the §18.8 canary, and unchanged names is exactly what "zero test edits" means |
| Workspace suite | **509 passing, 0 failing** |
| fmt | clean |
| clippy | **34 → 32.** Removed 2 pre-existing `needless_pass_by_value`; added **none** |
| aarch64-musl router build | **5,237,876 → 5,240,012 bytes (+0.04%)**, 14.3% of the 35 MiB budget |
| Self-host gate (rows 40-41) | **69/69** |
| CI | **9/9 jobs green**, digest confirmed on the runner |

### Three decisions worth arguing about

1. **`AppConfig` is deliberately not `crate::config::Config`.** The integration
   suite builds a *partial* configuration by hand — no `[data_store]`, no
   `[tracing]`, no `[maintenance]` — so bundling to `Config` would force every
   test to construct a whole one. That is a test edit by another name, and the
   gate is written specifically to catch that. The bundle holds the decisions,
   not the file they were parsed from.
2. **The `tenant` parameter of §6.1 is not there** *(superseded the same day —
   see §18.12)*. §6.1's sketch says `make_app_for(config, stores, tenant)`. The
   `tenant` type does not exist until the store layer lands, and inventing one
   here would put non-refactor code in a commit whose entire claim is that it is
   a refactor. So this item delivers the `config`/`stores` half; §6.2 adds
   `tenant` when there is a type to add. The deviation is recorded here rather
   than quietly taken.
   **§18.12 later contradicted the "until the store layer lands" half of this
   reasoning and kept the first half.** The reasoning was wrong in a way worth
   recording: the store layer is not the only thing that can be wrong about a
   type. A `Tenant` that means only "row in a `tenants` table" cannot be designed
   without one, but a `Tenant` that means §3.2's thing can — and if the two
   designs were allowed to merge later, the merged type would be wrong, because
   the *plausible* interpretation of `Tenant` is the one §3.2 is arguing
   against. Designing it before the store layer is what forces the distinction to
   be made explicitly instead of by default.
3. **`AppStores` keeps `AP` sized, not `Arc<dyn AuthenticationProvider>`.** The
   first attempt erased it and it does not compile: `caldav_router<AP: …>`,
   `carddav_router<AP: …>` and `frontend_router<AP: …>` are all generic over a
   *concrete* provider. The stores that are already trait objects upstream
   (`source_store`, `invite_store`, `share_store`, `password_reset_store`) stay
   erased, because that is how the callers already hold them. The comment on the
   field says so, because it looks like an oversight otherwise.

### One pre-existing warning deliberately left alone

`make_app_for`'s body carries a `redundant clone` on `combined_cal_store` — its
last use, so the clone is genuinely unnecessary. It was there before this commit
(line 175) and is still there (line 320), because **removing it would edit the
body this commit promises not to have touched**. It is the right fix and the
wrong commit. The doc comment on the function says so, so the next person does
not read it as an oversight.

### The submodule bump, which is its own commit

The server commit had to be pushed *and* the packaging repo's gitlink bumped
before CI would run any of it — `submodules: recursive` checks out the recorded
gitlink, so a pushed branch alone is invisible to every workflow. The bump is
`chore:` and separate on purpose: **a CI run that quietly tests last week's code
is worse than one that fails**, and putting a code change and a pointer change in
one commit makes that distinction invisible in the log.

## 18.12 The tenant parameter — the type designed *against* the wrong reading (2026-09-28)

**Shipped:** `crates/store/src/tenant.rs` (new, 296 lines), `src/app.rs` (+58),
`tests/tenant_app.rs` (new, 237 lines), `crates/store/src/lib.rs` (+2).
Server `99fe1a84`, packaging `57eefdb`, **9/9 CI jobs green**.

§18.11 recorded a deviation: it delivered `config`/`stores` and not `tenant`,
on the grounds that the type "does not exist until the store layer lands".
**That reasoning was half right and it was asked the wrong question.**

### The question that was not asked

The reasoning treated "is there a store layer yet?" as the blocker, as though a
`Tenant` type were only meaningful once a `tenants` table did. It is not. The
blocker is the one §3.2 spends a page on: **there are two completely different
things called "tenant", and this codebase has just committed to the less obvious
one.**

| | §3.2's tenant (adopted) | the table row (rejected) |
|---|---|---|
| what it is | a resolved `Router`, built over one store bundle | a `tenant_id` column |
| isolation comes from | the construction path — N routers, N bundles | a `WHERE` clause in every method |
| fails by | a bundle shared between two routes | forgetting one of ~200 call sites |
| verifiable | run one tenant's tests against another's bundle | read the diff and count `WHERE`s |

The rejected one is the *plausible* one. It is what "add multi-tenancy" means to
almost everyone, and it is the one that would be designed by default if the type
were first written in a store crate next to the migrations. So the honest reason
to design `Tenant` now is not that the store layer arrived early — it is that
**waiting until it arrives guarantees the wrong type gets written**, because by
then every existing `Tenant` reference in the codebase will point at a database
row and the naming will have settled on its own.

### How the type refuses the wrong design

`Tenant { id, slug, display_name, status, config_json }` — and **no store path,
no `db_url`, no `TenantStore` trait**, in a commit that touches `src/app.rs` and
nothing in `crates/store_sqlite/`. The absence is the design. A `Tenant` that
could open a database is a `Tenant` that some future caller will resolve instead
of dispatching, which is the §6.2 bug, made available by the §3.2 type.

`TenantId` is a **validated newtype**, not a `String`, because the slug reaches
three places that must agree: a hostname (`{slug}.{base_domain}`), a directory
under `data_root`, and a `UNIQUE` column. A `String` holding `../` is a
*different* bug in each, and the type is the only place all three can be caught
at once. It rejects **uppercase** specifically because §3.3 matches the Host
header lowercased: with a `String`, `Acme` and `acme` would be two rows passing
one `UNIQUE` constraint and one dispatch map entry — an auth-relevant split, not
a cosmetic one.

### The bug that only exists under a second tenant

`with_tenant` is a separate function, and that is not tidiness. axum's
`Router::layer` wraps **only the routes registered before it**, so installing the
extension earlier in `make_app_for` — before the DAV routers are merged — would
silently miss all of them, and a handler expecting a tenant would 500.

**This is exactly the class of bug that N=1 cannot find.** With one tenant there
is no `Extension` to miss, no second router, and the integration suite is 98/98
either way. The first draft of the test made the mistake anyway: it built the
router and *then* added `/_probe`, so the probe was outside the layer and the
test asserted `"acme"`, failed, and the obvious fix — reordering until it passed
— would have moved the production call site to a worse position rather than
fixing the test.

So the fix was in the test, not the code:

```
Router::new().route("/probe", …)   // registered FIRST
    |> with_tenant(tenant)          // layer wraps what exists
```

and then the gate that matters: **`with_tenant` was neutered to a no-op and
`test_the_extension_covers_routes_registered_before_it` was confirmed to
FAIL** (`"no-tenant"` vs `"acme"`). A test that has never been seen to fail is
not evidence of anything. The two N=1 tests are in the same file for the same
reason — they are the two properties that must *not* change.

### `enabled = false` is a gate, not a sentence

§3.6 requires the single-tenant install to behave exactly as before. Two tests
assert it rather than claiming it: `make_app` passes `None`, so
`Option<Extension<Tenant>>` resolves to `None` in every handler, and the
`TraceLayer` closure records `tenant = Empty` — the span shape is byte-identical
to before, not merely equivalent. `test_the_parameter_does_not_isolate_anything_yet`
pins the other half: a labelled and an unlabelled router over the same context
return **identical responses**, which is the recorded proof that the parameter is
a label and not a guard.

### Gates

| Gate | Result |
|---|---|
| §18.8 canary — 98 tests, zero edits, name digest | **green.** `e7760b76…` unchanged; `git status` shows `tests/integration_tests` untouched. New tests are a **separate target**, so the canary never needed weakening |
| §18.11's "body moved verbatim" | **still true.** `git diff src/app.rs` removes **2 lines** — the import, and `move` on a closure. The body is untouched |
| Workspace suite | **509 → 521 passing, 0 failing** |
| fmt / clippy | clean; **zero** warnings from new code, crate total **32 → 32** |
| aarch64-musl | 5,240,012 → **5,240,500 bytes (+488)**, 14.3% of 35 MiB |
| Self-host gate | **69/69** |
| CI | **9/9 green** (hygiene `36487119107`, build `36487118874`, test `36487118839`) |

### What is still not here, and what it would be wrong to assume

No `HostDispatch`, no `TenantStore`, no `tenants` table, no `[tenancy]`
config block, no `StoreBundleCache`, no `rustical tenant` CLI. **Nothing in this
commit isolates anything.** There is exactly one tenant's data reachable from
any router in this build.

This is recorded in the type's own doc comment and in
`test_the_parameter_does_not_isolate_anything_yet`, because the failure mode for
the next reader is not "this was too slow" — it is reading a `Tenant` threaded
through `app.rs` and concluding that a check is happening. It is not.
**§6.4 rows 26-28 are what enforce scoping, and item 9 remains
safety-critical and not started.**

---

## 18.13 Item 8, part 1 — the control plane (2026-09-28)

**Shipped:** `crates/store/src/tenant_store.rs` (new), `crates/store_sqlite/src/tenant_store.rs`
(new), `crates/store_sqlite/control_migrations/` (new), 21 tests. Server `ce23583c`,
packaging `12c5785`, **9/9 CI green**. Workspace 521 → 542, aarch64-musl **−1,984 bytes**.

**This cannot serve a request and does not try to.** It is the layer `HostDispatch`
will read from. Two more parts follow (the bundle cache, then dispatch itself).

### The separate migration directory is the point, and there is a test for it

`control_migrations/` rather than `migrations/`. Pointing
`create_control_plane_pool` at `./migrations` would create thirteen tables of
calendar, addressbook and credential data inside the one database whose entire
property is that it cannot hold any — so a per-tenant backup could take out
every other tenant. `the_control_plane_holds_no_calendar_data` asserts the
control plane contains exactly `tenants`, `tenant_hosts` and sqlx's ledger, and
names six tables that must never appear. The mutation was applied to the code
and the test caught it.

### Row 29 is a SQL filter, not a code path

Every getter filters `status = 'active'`, which is the whole of §3.3 step 4:

```
Host -> control plane (always) -> TenantId -> cache -> Arc<Router>
```

A suspended tenant never resolves, so it never reaches the cache — there is no
cached router to invalidate and no window in which it is still served. The
mutation (removing the filter from the host lookup) was applied and three tests
failed, including the one named for the row.

**A suspended tenant keeps its hostname claim.** Freeing it on suspension would
let dispatch fall through to a slug match or `default_tenant` and serve *somebody*
under a suspended tenant's URL — the opposite of the intent. That is the subtle
half of the row and it has its own test.

### The sqlx dynamic-SQL lint was right, and it shaped a signature

sqlx 0.9 refuses to run a query string it cannot prove constant. `find_active`
takes `&'static str`, and that is the security property rather than an accident
of style: `&String` and `&format!(..)` both fail to *compile*, so no caller can
interpolate anything. The alternative — taking `&str` and wrapping the result in
`AssertSqlSafe` — would have left a one-word edit away from an injected slug on
the dispatch path.

### A test caught a contract violation

`set_tenant_hosts` on an unknown tenant returned a raw SQL foreign-key error
instead of the documented `NotFound`, because the host insert is checked before
the tenant exists. sqlite's FK enforcement is what surfaced it — the test wrote
`expect_err` expecting `NotFound` and got a constraint violation, which is the
test having done its job.

### `Tenant` grew three fields, and that is the honest cost

`plan`, `suspended_at`, `created_at`. This broke `tests/tenant_app.rs` from
§18.12 — a struct that partly mirrors one row costs a call-site edit whenever
the row changes. `quota_*` deliberately did **not** join them: `NULL` means
*unlimited*, not zero, and folding them in would have made `Tenant` a faithful
image of a table row, which is the shape §3.2 argues against. They are
`TenantQuota`, with their own getter and setter.

## 18.14 Item 8, part 2 — the store bundle and its cache (2026-09-28)

**Shipped:** `src/store_bundle.rs` (new, 468 lines), `src/lib.rs` (+56/−19),
`lru` 0.18. Server `2b0f2692`, packaging `1b5537b`, **9/9 CI green**. Workspace
542 → 553, clippy 32 → 32, aarch64-musl **+908 bytes**.

### The leak that the visible metric could not see

`LruCache::put` calls `capturing_put(_, _, false)`: it **discards the evicted
entry** and returns `None` for a capacity eviction. It returns a value only when
the *key* already existed. So the obvious implementation of
`StoreBundleCache::insert` honours the memory **count** bound — the thing §7.2's
gate and every test in the file measures — while leaking one SQLite pool per
eviction.

The count is the metric, so the leak is invisible to it. It took a test that
asserts the *eviction is reported*, not merely that the count stayed within
bounds. `insert` now uses `push` (the capturing variant) and the doc comment
says why, because the crate's API invites the bug in precisely the shape that
review and gate both miss. The mutation was applied to confirm the test bites.

### The struct is not `Clone`, on purpose

`mpsc::Receiver` is not `Clone` and cannot be: one tenant has one DAV-Push
channel and one notifier loop. `update_recv: Option<Receiver<..>>` plus
`take_update_recv()` puts that one-shot fact in the type instead of in a
comment. The second call returns `Err` rather than panicking — "already
consumed" is a state a running server should survive by serving without
DAV-Push, not by dying.

### `get_data_stores` survives as a shim, with a warning on it

Same move as item 7's `make_app` → `make_app_for`: 5 CLI call sites, none of
which should have moved in this commit. The 12-tuple is kept *with a comment
saying not to use it*, because its element order is checked by nothing and
`subscription_store` / `invite_store` are adjacent `Arc<dyn ..>` over the same
pool — a reorder compiles silently. The construction body is untouched; the
diff removes only the old signature and the old return expression.

### Two deferrals, both deliberate

1. **`[tenancy] trusted_proxies` (C5, §7.3.4) is not added in this commit.**
   §3.6 lists it, but it is the `X-Forwarded-For` trust list — a *security*
   knob. Shipping it in a commit that does not honour the header would mean an
   operator sets `trusted_proxies` and gets a rate-limit bypass while the config
   claims otherwise. A knob that is accepted and ignored is worse than a missing
   one, so it arrives with §7.3.4.
2. **`[tenancy] sessions` (C4, §3.7) is not added.** The `session-redis` cargo
   feature does not exist in this tree, so `store = "redis"` would be a literal
   that parses and does nothing. Same reasoning; it arrives with the adapter
   crate.

Both are recorded here rather than quietly taken, which is the §18.11
convention.

---

## 18.15 Item 8, part 3 — `HostDispatch` (§3.3), and five bugs its tests found

**Shipped:** `src/host_dispatch.rs` (new, 550 lines), `src/tenancy.rs` (new),
`[tenancy]` in `src/config.rs`, two new store methods, 22 dispatch tests + 7
end-to-end tests. Server `5a73f377`, packaging `bad49dc`, **9/9 CI green**.
Workspace 553 → 582, clippy 32 → 32, aarch64-musl **+33,908 bytes**.

Requests are now dispatched by `Host`. **Five product bugs were found by this
commit's own tests, and the first two are the ones worth remembering.**

### 1. A suspended tenant's hostname served the *default tenant's data*

With a host claim and a `default_tenant` both configured, a request for a
suspended tenant's host failed the active-only lookup, fell through the
remaining match rules, and was served by `default_tenant`. One customer seeing
another customer's calendar, under a suspended tenant's URL — worse than a 404,
and precisely what §3.3's "removed from the map" exists to prevent.

The fix is a distinction the store had to be asked for rather than inferred:
**nobody owns this host** (keep matching) versus **a suspended tenant owns it**
(stop, 404). That needed two new methods — `is_host_claimed` and
`get_any_tenant_by_slug` — and they are needed at *every* match step, not just
the first, because the same fall-through hazard exists for the `base_domain`
slug, the bare-slug rule, and `default_tenant` itself. A suspended
`default_tenant` now logs at `warn!` and 404s rather than leaving a server that
answers nobody and looks merely idle.

### 2. A brand-new tenant 500s, forever

Nothing in this tree guarantees that the actor which inserted a `tenants` row
also ran migrations — `rustical tenant` is item 11 and does not exist yet — so
with `migrate: false` the first request failed `no such table:
davpush_vapid_key`, and every request after it identically. A tenant that
exists, is dispatched to, and can never serve. Now `migrate: true`, which is
one indexed `_sqlx_migrations` lookup per *build*, and a build happens once per
tenant per cache residency rather than per request.

### 3. A load balancer would have blackholed the deployment

With tenancy on, an LB health-checking `GET /ping` on the instance's own
address gets **404**: that address is not a tenant's host, and hosted has no
`default_tenant`. The instance looks unhealthy, the LB removes it, every tenant
goes down. **The health check caused the outage it existed to detect.**

`/healthz` now answers *ahead of* dispatch, and deliberately does not consult
the control plane — a control-plane outage must not mark every instance
unhealthy and provoke the thundering herd that a dependency-aware check causes.
It exists **only** when tenancy is on: with tenancy off, `/ping` already answers
for every host, and §3.6 says not to widen the default single-tenant surface
with a second unauthenticated route.

### 4 and 5, smaller but the same shape

`normalise_host("[::1]:8443")` returned `[::1` — an IPv6 literal split on its
first colon, producing a host that matches nothing, silently. And SQLite creates
a *file* but not the directory above it, so every new tenant's store path 500'd
until `ensure_tenant_store_dir` existed.

### The three no-tenant outcomes are byte-identical

Unknown host, suspended tenant, unclaimed host: same status, same body, and the
body names no tenant, no base domain and no status. Three distinguishable bodies
would be an enumeration oracle — which hostnames are claimed, which tenants
exist, which are suspended. §3.4 keeps the control plane free of other tenants'
data for the same reason.

A control-plane failure is a **500, not a 404**. Collapsing a database outage
into "your tenant does not exist" is both a lie and impossible to diagnose.

### Two structural notes

- `HostDispatch` does **not** implement `tower::Service` and cannot: a
  `'static` future returned from `call(&mut self, ..)` must own the dispatcher,
  and `impl Service for Arc<HostDispatch>` is forbidden by the orphan rule.
  `TenancyAwareApp` is local and does the job — which is also what `cmd_serve`
  needs, so the two serve paths share one concrete type and the
  `enabled = false` case is a `match` arm rather than a second `axum::serve`.
- The end-to-end target exists because a test that constructs `HostDispatch`
  itself tests a *component*, and rows 24-25 and 29 are claims about a server.
  Its first draft built `[tenancy] control_db_url` from one path and seeded
  another, so the server opened a control plane of its own making, found no
  tenants, and answered 404 everywhere — **a symptom identical to the dispatch
  bug the file exists to catch.** `Install` now owns the directory and writes
  the config out of it, so there is one source of paths.

### Still not here

The per-tenant `config_json` merge (item 10) and `rustical tenant` (item 11).
`trusted_proxies` and `[tenancy.sessions]` are **deliberately absent** (§18.14):
neither has code in this tree, so both would parse and do nothing, and
`deny_unknown_fields` turns that into a loud error that
`an_unknown_tenancy_key_is_rejected_rather_than_ignored` pins.

**Row 29 is proven end to end** through a real server: a tenant's router is
built and cached, the control plane is suspended, and the next request is 404
while the other tenant is unaffected. **Row 24 is not** — its auth half needs
two tenants holding the same principal and is item 9.

---

*End of plan. **Where this actually stands: §5 is done except the deferred
credential rotation (item 1, user decision — the runbook is written and the
key/tokens are still live), and §6 items 2-7 are shipped. The next thing to do
is item 8's `HostDispatch` + `TenantStore` + migration, then item 9.*

*§5 came first because it blocked all three models and D2 meant the repo was
about to be public. That reasoning held. §6 is the only expensive work and the
only one that can be deferred without stopping the other two models.*

*Two of the three safety-critical items remain open and one is now urgent:
**§6.4 (the three un-authenticated routers) is item 9 and is not started** —
rows 26-28 in §12 are the only thing standing between a valid token from tenant
A and tenant B's data, and §18.12 added the type those rows will assert against
without adding any of the enforcement. §5.2 (secrets) is done but for the
deferred rotation, and §7.3.4 (`X-Forwarded-For`) is untouched. None of the
three is optional, and each has a test row in §12.*

*Escalate to the user rather than deciding: the history rewrite (§5.2.5), the
Q1-Q8 answers (§17), the credential rotation window (item 1), and anything in
§6 that would require touching the store traits.*
