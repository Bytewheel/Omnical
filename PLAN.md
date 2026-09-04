# Omnical — Custom Contacts / Calendar / Tasks Server on the libreCMC Router

**Implementation Plan — v1.0**

- **Date:** 2026-09-04
- **Project:** `omnical`
- **Goal:** A self-hosted, domain-agnostic CalDAV (calendar + VTODO tasks) and CardDAV (contacts) server running on the libreCMC router at `192.168.10.1`, reachable from the public internet with real TLS via `0115d8cf.duckdns.org`, serving accounts for **any email address / any email domain**, acting as a **sync hub for accounts hosted on any provider**, with **sharing across identities** and **cross-domain invitations**.
- **Server software decision (approved):** cross-compile [RustiCal](https://github.com/lennart-k/rustical) (AGPL-3.0, Rust, single static binary) + one small custom-built component (`dav-tls`, a rustls TLS tunnel).
- **Storage decision (approved):** internal router flash + nightly backups.
- **Access/TLS decision (approved):** public hostname `0115d8cf.duckdns.org` with Let's Encrypt (DNS-01 via DuckDNS API).
- **Required clients (approved):** vdirsyncer + khal/khard, Apple Calendar/Contacts, DAVx5 + Tasks.org (Android), Thunderbird.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Target Architecture](#2-target-architecture)
3. [Environment Inventory (verified facts)](#3-environment-inventory-verified-facts)
4. [Constraints, Decisions, and Rationale](#4-constraints-decisions-and-rationale)
5. [Phase 0 — Prerequisites](#phase-0--prerequisites)
6. [Phase 1 — Build (dev machine)](#phase-1--build-dev-machine)
7. [Phase 2 — Install on Router](#phase-2--install-on-router)
8. [Phase 3 — TLS Certificate Automation](#phase-3--tls-certificate-automation)
9. [Phase 4 — Firewall & Reachability](#phase-4--firewall--reachability)
10. [Phase 5 — Users, Groups, Collections (any email domain)](#phase-5--users-groups-collections-any-email-domain)
11. [Phase 6 — Sync Hub for Any Provider (vdirsyncer)](#phase-6--sync-hub-for-any-provider-vdirsyncer)
12. [Phase 7 — End-User Clients & Cross-Domain Invitations](#phase-7--end-user-clients--cross-domain-invitations)
13. [Phase 8 — Backups & Maintenance](#phase-8--backups--maintenance)
14. [Verification Matrix](#14-verification-matrix)
15. [Rollback / Recovery](#15-rollback--recovery)
16. [Risks & Mitigations](#16-risks--mitigations)
17. [Future / Optional Enhancements](#17-future--optional-enhancements)
18. [Appendix A — RustiCal Reference Notes](#appendix-a--rustical-reference-notes)
19. [Appendix B — Key Commands & File Paths](#appendix-b--key-commands--file-paths)
20. [Appendix C — Relevant RFCs](#appendix-c--relevant-rfcs)

---

## 1. Executive Summary

The libreCMC v6.8 router (ThinkPenguin TPE-R1400, aarch64) cannot run the usual
CalDAV/CardDAV servers from packages: its opkg feeds (~2,435 packages) contain **no
python3, php, node, nginx, lighttpd, stunnel, or haproxy**, and the pre-installed socat
is compiled **without OpenSSL**. The established, proven pattern on this network is the
`router-nym` project: statically cross-compiled Rust binaries (via the zig
`aarch64-linux-musl` toolchain) pushed to the router over SCP, managed by procd init
scripts.

This plan applies that pattern to **RustiCal** — an actively developed, libre
(AGPL-3.0) CalDAV/CardDAV server in Rust (axum + sqlx + statically bundled SQLite,
embedded web frontend, WebDAV Push, group sharing, Nextcloud login flow, tested with
DAVx5/Tasks.org/Thunderbird/Apple/GNOME). RustiCal serves plain HTTP only and expects
a TLS-terminating reverse proxy; since none is installable on the router, we build
**one small custom component**: `dav-tls`, a ~100–150 line rustls TLS tunnel
(`0.0.0.0:443 → 127.0.0.1:4000`), cross-compiled with the same toolchain.

Public reachability is achieved with a Let's Encrypt certificate for
`0115d8cf.duckdns.org` (resolves to the current public IP `65.33.235.245`), issued via
**DNS-01** through the DuckDNS API (no inbound port 80 required), automated with
`acme.sh` on the dev machine, and deployed to the router on renewal. A single
port-forward (TCP 443 → `192.168.1.21:443`) is configured once on the upstream TP-Link
router.

"Works across any email domain" is delivered on four levels (all approved):

| Requirement | How it is met |
|---|---|
| **Multi-identity accounts** | RustiCal principals are arbitrary strings — one account per full email address (6 identities across 4 domains today, any future address works); auth is user-id + app token, domain-agnostic |
| **Sync hub for any provider** | vdirsyncer on the dev machine pairs Google (or any CalDAV/CardDAV provider) ↔ local vdirs ↔ Omnical, making the router the convergence point for accounts hosted anywhere |
| **Sharing across domains** | RustiCal groups + memberships; per-principal ACL sharing independent of email domain |
| **Cross-domain invitations** | Client-side iMIP/iTIP (Apple Calendar / Thunderbird email the `.ics` invites via any SMTP identity to any attendee address). RustiCal does **not** implement RFC 6638 server-side scheduling — documented limitation, optional future extension |

Total router footprint: one ~15–35 MB stripped binary + one ~2–4 MB tunnel + a SQLite
database (KBs–MBs) + TLS certs, inside the 42 MB free on the router's internal f2fs
overlay, with nightly pull-based backups to the dev machine and sysupgrade
survivability configured.

---

## 2. Target Architecture

```
                 ┌──────────────────────────────────────────────────────────┐
                 │                    CLIENTS                                │
                 │  DAVx5 + Tasks.org · Apple Calendar/Contacts ·            │
                 │  Thunderbird · vdirsyncer (dev machine sync hub)          │
                 └────────────────────────┬─────────────────────────────────┘
                                          │  HTTPS, real Let's Encrypt cert
                                          │  https://0115d8cf.duckdns.org:443
                                          ▼
                    ┌───────────────────────────────────────────┐
                    │  Upstream TP-Link router (192.168.1.1)     │
                    │  NAT port-forward: TCP 443 → 192.168.1.21  │
                    └────────────────────────┬──────────────────┘
                                          ▼
        ┌───────────────────────────────────────────────────────────────┐
        │  libreCMC v6.8 — ThinkPenguin TPE-R1400 (RK3328, 4× A53)       │
        │  WAN: 192.168.1.21 · LAN: 192.168.10.1 · ULA: fd1e:5c71:1ee8::1 │
        │                                                                 │
        │   ┌─────────────────┐      ┌──────────────────────────────┐   │
        │   │  dav-tls (NEW)  │      │  rustical (RustiCal 0.16.1)  │   │
        │   │  rustls tunnel  │─────►│  127.0.0.1:4000              │   │
        │   │  0.0.0.0:443   │      │  axum · sqlx · SQLite        │   │
        │   │  LE cert        │      │  embedded web UI + admin     │   │
        │   └─────────────────┘      └──────────────┬───────────────┘   │
        │                                            ▼                   │
        │                /usr/local/share/rustical/db.sqlite3          │
        │                (persistent overlay — NOT /var, which is      │
        │                 tmpfs and wiped every boot)                   │
        │                                                                 │
        │  procd services: rustical, dav-tls · cron · uhttpd (LuCI,     │
        │  moved to :8443) · bind authoritative DNS · wireguard tunnels  │
        └───────────────────────────────────────────────────────────────┘
                                          ▲
                                          │ nightly pull backup over SSH
                 ┌────────────────────────┴─────────────────────────┐
                 │  Dev machine (ssh alias `router`, key ~/.ssh/router) │
                 │  builds binaries (zig + rust musl), runs acme.sh,     │
                 │  runs vdirsyncer sync hub (Google ↔ vdirs ↔ Omnical)  │
                 └───────────────────────────────────────────────────────┘
```

**Data flow of the sync hub (any provider → router):**

```
Google Calendar/Contacts (any identity)  ──vdirsyncer──►  ~/.calendars, ~/.config/khard/contacts (vdirs)
                                                        │
                                                        │ vdirsyncer (caldav/carddav storages)
                                                        ▼
                                        Omnical on the router (CalDAV/CardDAV)
                                                        ▲
                                    ┌───────────────────┴──────────────────┐
                                    │  all other clients (Apple, DAVx5,   │
                                    │  Thunderbird) read/write directly    │
                                    └──────────────────────────────────────┘
```

---

## 3. Environment Inventory (verified facts)

All facts below were verified directly on 2026-09-04. They are the foundation the plan
is built on; re-verify anything that may have drifted before executing a phase.

### 3.1 Router: `router` (SSH alias → `root@192.168.10.1`, key `~/.ssh/router`)

| Item | Value |
|---|---|
| Model | ThinkPenguin TPE-R1400 (RockChip RK3328, 4× Cortex-A53, aarch64) |
| Firmware | libreCMC v6.8, revision `r0-44c488c9`, kernel 5.15.210, target `rockchip/armv8`, arch `aarch64_generic`, no taints |
| RAM | 1,011,084 KB total; ~828 MB available (127 MB used incl. nym/mullvad/bind stack) |
| Root/overlay | `/dev/root` squashfs 3.8M (read-only rom); `/dev/loop0` f2fs on overlay, 98.4M total, **42.2M free** (57% used) |
| tmpfs | `/tmp` ~494M (RAM-backed — **never put persistent data here**) |
| USB | One SuperSpeed USB device attached (the RTL8152 USB WAN NIC); `usb-storage` driver registered; a free USB host port may exist but is unconfirmed — plan does not depend on it |
| Init system | procd (+ procd-ujail, procd-seccomp available) |
| Cron | Active (`/etc/crontabs/root`): bind `srvzone` hourly at :50, root-hints update Sundays 03:30, `screech-watchdog` every 5 min |
| Web server | uhttpd (LuCI) — `listen_http` 192.168.10.1:80, 10.10.10.1:80, [fd1e:5c71:1ee8::1]:80; `listen_https` same IPs on **443** (conflicts with plan → moved to 8443); `redirect_https=1`; self-signed `/etc/uhttpd.crt` |
| DNS | bind 9.18 authoritative (OpenNIC zones, e.g. `truth.zone`), forwarding to split-dns dnsmasq instances on ports 5354–5360 (mullvad, nym1–4, direct, penguin) |
| Tunnels | WireGuard: `wg-vm1` 10.100.1.2/30, `wg-vm3` 10.100.3.2/30 (user's VMs); mullvad 10.64.70.187/32; nym exits (see nym stack from router-nym) |
| WAN | `eth0` 192.168.1.21/24 (DHCP, hostname `libreCMC`) — behind upstream router **192.168.1.1** (answers ping; TP-Link, per tplinkdns.com traces) |
| LAN | `br-lan` 192.168.10.1/24; IPv6 ULA `fd1e:5c71:1ee8::1/60`; also 10.10.10.1 in uhttpd config (address not currently on br-lan) |
| opkg feeds | `librecmc_core` (…/v6.8/targets/rockchip/armv8/packages), `librecmc_base` (…/v6.8/packages/aarch64_generic/base) — **2,435 packages listed** |
| Packages NOT available in feeds | python3, php, node, nginx, lighttpd, stunnel, haproxy, radicale — **confirmed absent** (rules out all standard DAV servers and TLS proxies) |
| Packages available & relevant | `sqlite3-cli 3410200-1` (for online DB backups), `ca-bundle 20260223-1` (`/etc/ssl/certs/ca-certificates.crt` present), `curl 8.5.0` (for duckdns update), `socat 1.7.4.4` (but compiled `#undef WITH_OPENSSL` — **cannot terminate TLS**) |
| Firewall | nftables (firewall4): zones lan/wan/vpn (+ screech_z_* zones for wg/mullvad/nym1-4/penguin); wan-input allow rules exist for DHCP/Ping/DNS etc. — pattern to follow for adding TCP 443 |

### 3.2 Dev machine (the workstation running this build)

| Item | Value |
|---|---|
| rustc | 1.97.1 — **too old for RustiCal (requires ≥ 1.98) → `rustup update stable` in Phase 0** |
| rustup targets | `aarch64-unknown-linux-musl` already installed; `x86_64-unknown-linux-gnu` for local smoke tests |
| zig | 0.16.0 (`/usr/bin/zig`) — proven as CC/AR/RANLIB/linker for musl cross-builds |
| Go | 1.26.7 (not needed for chosen approach) |
| bun | available (frontend build fallback if RustiCal's dist is not committed) |
| router-nym assets | `~/router-nym/build/bin/zig-musl-cc` wrapper scripts (CC/AR/RANLIB/NM/linker) + `scripts/build-deps.sh`, `scripts/build-rust.sh`, `deploy.sh` — **the template for this project's build/deploy** |
| SSH | `Host router` alias configured (root@192.168.10.1, IdentityFile `~/.ssh/router`, known_hosts pinned) |

### 3.3 Network / public reachability

- `0115d8cf.duckdns.org` → A record → `65.33.235.245` — matches the current public IP
  of the upstream network (verified via `dig` and `curl ifconfig.me`). The user
  confirmed this hostname is available for this project.
- Public exposure therefore requires exactly one upstream port-forward (TCP 443) plus
  a libreCMC firewall input rule. DuckDNS supports TXT records via its API →
  **Let's Encrypt DNS-01 works with no inbound port 80 at all**.

### 3.4 Identity & data ecosystem (who/what this serves)

Email identities (6, across 4 domains — any future address must also work):

1. `burningserenity@gmail.com`
2. `nfcarlton@gmail.com`
3. `nfcalaway@gmail.com`
4. `nicholas@hawksnestsoftware.com` (Google Workspace)
5. `nicholas@carltonaudio.com` (IMAP at netsol)
6. `nfcalaway@novo-ordo.com` (IMAP at novo-ordo)

Existing tooling to integrate with:

- `~/.mbsyncrc` — mbsync/mbsync channels for all identities
- `~/.config/vdirsyncer/config` — pairs syncing Google Calendar/Contacts into local vdirs
  (storages: `~/.calendars/google/`, `~/.calendars/hawksnest/`, `~/.config/khard/contacts/*`;
  `.ics`/`.vcf` vdir format)
- `~/.config/khal/config` — calendars `google`, `google_hawksnest`, `local`
- `~/.config/khard/khard.conf` — 5 addressbooks (burningserenity-gmail, carltonaudio, hawksnest, nfcalaway-gmail, nfcarlton-gmail)
- `~/.calendars/local/` — an existing "local" calendar (natural home to migrate into Omnical)

### 3.5 RustiCal (server software) — verified properties

- Version pinned: **0.16.1** (repo `lennart-k/rustical`, AGPL-3.0-or-later, ~524 stars, active)
- Rust workspace (`crates/*`): axum 0.8, sqlx 0.9 with **bundled SQLite** (single
  static binary; its own Dockerfile cross-builds `aarch64-unknown-linux-musl` with
  clang/lld — same target as our zig approach), `rust-embed` embedded frontend
  (Docker builds with no node stage → frontend dist is expected to be committed;
  verify at clone time), `password-auth` (argon2 for frontend passwords, pbkdf2 for
  app tokens), `web-push` (WebDAV Push for near-instant DAVx5 sync),
  `openidconnect` (OIDC optional at config level), `provided-listeners` (TCP/Unix only)
- HTTP layer: plain HTTP, default port **4000**, `HttpConfig { bind, host, port,
  session_cookie_samesite_strict, payload_limit_mb }` — **no built-in TLS**
  (`HttpBindConfig` enum has only `Tcp(String)` and `Unix(PathBuf)`)
- Config: TOML file or `RUSTICAL_*` env vars (`rustical gen-config` prints defaults);
  db URL via `RUSTICAL_DATA_STORE__SQLITE__DB_URL` (Docker default
  `/var/lib/rustical/db.sqlite3` — **unusable on libreCMC because /var is tmpfs**)
- Endpoints: `/.well-known/caldav`, `/.well-known/carddav`, `/caldav`,
  `/caldav/principal/<user_id>`, `/caldav-compat` (for Apple Calendar, which
  mis-handles multi-home `calendar-home-set`), `/carddav`, frontend at `/`
- Auth: HTTP Basic with **user id + app token** (tokens generated in web UI / frontend
  API). User ids: arbitrary strings, forbidden chars `:` and `$` only → **full email
  addresses work, any domain**. Groups: `rustical principals --principal-type group`
  + `rustical membership`; impersonation syntax `user$group` for clients that can't
  discover group principals
- CLI: `rustical serve`, `rustical principals`, `rustical membership`,
  `rustical gen-config`, `rustical health`
- Scheduling: **RFC 6638 NOT implemented** ("not sure yet whether to implement this")
  → cross-domain invitations are client-side iMIP only
- Sync: RFC 6578 sync-token listed as relevant/partial — clients (DAVx5 etc.) fall
  back to full PROPFIND sync when absent/partial
- Runtime needs: writable `/tmp` (SQLite temp), CA bundle at `/etc/ssl/certs` —
  both already satisfied on the router

---

## 4. Constraints, Decisions, and Rationale

| # | Constraint (fact) | Decision | Rationale |
|---|---|---|---|
| C1 | No python3/php/node in libreCMC v6.8 feeds | Do not deploy Radicale/Baïkal/Nextcloud | Literally uninstallable; static-porting Python is a maintenance dead end |
| C2 | 42 MB free on f2fs overlay; internal flash only (user preference) | RustiCal stripped must fit ≤ ~35 MB + DB; size-gate at build; mitigations: `strip`, `opt-level="z"`, LTO, `panic="abort"`; last-resort `upx --lzma`; USB stick as explicit fallback | Data itself is tiny (KB–MB); the binary is the only large artifact |
| C3 | RustiCal has no TLS; no stunnel/haproxy/nginx installable; router socat lacks OpenSSL | **Build `dav-tls`**: minimal rustls TLS tunnel `0.0.0.0:443 → 127.0.0.1:4003` (raw byte-pipe post-handshake; ALPN `http/1.1` only) | ~150 lines, pure Rust, same proven toolchain; passes PROPFIND/REPORT/PUT and WebSockets (WebDAV Push) untouched; no ALPN h2 → all DAV clients happily use HTTP/1.1 |
| C4 | uhttpd occupies 443 on LAN IPs | Move uhttpd `listen_https` to **8443** (keep :80 with redirect); `dav-tls` binds `0.0.0.0:443` | Wildcard bind conflicts with uhttpd's specific binds otherwise; LuCI remains reachable on https://192.168.10.1:8443 |
| C5 | Router behind upstream TP-Link (192.168.1.1); public IP is dynamic; `0115d8cf.duckdns.org` already tracks it | One-time upstream port-forward TCP 443 → 192.168.1.21:443; ACME via **DuckDNS DNS-01**; verify what updates duckdns (add router cron if nothing does) | No port 80 exposure needed; renewal keeps working through IP changes |
| C6 | `/var` is tmpfs (wiped per boot); sysupgrade wipes non-conffile overlay files | SQLite DB at **`/usr/local/share/rustical/db.sqlite3`**; add `/etc/rustical` + `/usr/local/share/rustical` to `/etc/sysupgrade.conf`; document that `/usr/sbin` binaries need re-deploy after sysupgrade | The single most important data-safety detail of this plan |
| C7 | RustiCal is young software | Pin 0.16.1; run the full verification matrix before cutover; keep Google pairs as source of truth until verified | Risk containment |
| C8 | RFC 6638 absent | Cross-domain invitations via client-side iMIP (Apple/Thunderbird email `.ics` from any SMTP identity to any attendee domain) | Honest limitation; server-side scheduling is a future custom extension (see §17) |
| C9 | Established `router-nym` pattern exists (zig-musl-cc wrappers, procd init.d, deploy.sh, watchdog cron) | Mirror it: `~/router-dav/` project layout with `scripts/`, `out/`, `router/` overlay tree, `deploy.sh` | Consistency and re-use of working, known-good infrastructure |

---

## Phase 0 — Prerequisites

**Owner: user (unless noted). Gate for Phase 1.**

1. **Update the Rust toolchain** (dev machine) — **DONE (2026-09-04): rustc 1.97.1 → 1.98.1,
   `aarch64-unknown-linux-musl` target confirmed installed; rustup self-updated to 1.29.1.**
   ```sh
   rustup update stable        # RustiCal 0.16.1 needs rustc >= 1.98 (have 1.97.1)
   rustup target add aarch64-unknown-linux-musl   # idempotent
   ```
2. **DuckDNS token** for account containing `0115d8cf` subdomain → store in `pass`
   — **DONE (2026-09-04): stored as `secrets/duckdns/0115d8cf.duckdns.org/token`
   (user-inserted); UUID shape checked and validated against the DuckDNS update API
   (`OK` = token valid, subdomain in account).** Needed for ACME DNS-01 in Phase 3.
3. **Upstream router (TP-Link at 192.168.1.1)**: create NAT port-forward
   **TCP 443 → 192.168.1.21:443** (external 443). No port 80 forward needed.
4. **DuckDNS updater ownership**: confirm what currently updates the A record for
   `0115d8cf.duckdns.org` (TP-Link feature? dev-machine cron?). If nothing does,
   add a cron on the router in Phase 2 (curl to the DuckDNS update API every 5 min).
   — **INVESTIGATED (2026-09-04): no updater found — dev-machine crontab/systemd
   clean, libreCMC router cron clean, user unsure; TP-Link built-in DDNS not
   verifiable without upstream admin access. Resolution: add the router cron in
   Phase 2.7 (DuckDNS updates are idempotent, so an extra updater is harmless).**
5. **App-token storage convention**: RustiCal app tokens (per client) will be
   generated in Phase 5; store them in `pass` under `secrets/omnical/…`.
6. **Confirm the identities list** (the 6 from §3.4, or a revised list) — drives
   Phase 5.

---

## Phase 1 — Build (dev machine)

**Deliverable: two static `aarch64-unknown-linux-musl` binaries in `~/router-dav/out/`
plus a local x86_64 smoke-test pass.**

> **STATUS (2026-09-04, updated):** 1.1 skeleton **DONE** · 1.2 clone+verify
> **DONE** · **1.3 DONE — rustical cross-compiled: static aarch64 binary, 26 MiB
> stripped (within 35 MiB budget), runs under qemu-aarch64 (`--version`,
> `gen-config` OK); toolchain fix recorded under 1.3** · **1.4 DONE — dav-tls
> built: static aarch64 binary 1.22 MiB stripped (`out/dav-tls`, runs under
> qemu-aarch64) + x86_64 host build 1.4 MiB
> (`out/x86_64-unknown-linux-gnu/dav-tls`); both via `scripts/build-rust.sh`
> (clang recipe D)** · 1.5 smoke test pending.
> **Nothing has been deployed to the router.**

### 1.1 Project skeleton (mirror router-nym) — **DONE (2026-09-04)**

Created exactly as specified below, plus `README.md` (sysupgrade runbook), and the
zig wrappers copied into `build/bin/` (self-contained; one local patch — see
Toolchain findings). `build-deps.sh` confirmed unnecessary. `deploy.sh` written but
NOT yet run (Phase 2). Files: `scripts/build-rust.sh`, `dav-tls/` crate,
`router/etc/rustical/config.toml`, `router/etc/init.d/{rustical,dav-tls}`,
`router/etc/sysupgrade.conf.additions`, `out/tls/`.


```
~/router-dav/
├── scripts/
│   ├── build-rust.sh      # adapted from router-nym/scripts/build-rust.sh
│   └── build-deps.sh      # NOT needed (sqlx bundles SQLite; rustls is pure Rust)
├── dav-tls/               # the custom rustls tunnel crate (new code)
│   ├── Cargo.toml
│   └── src/main.rs
├── rustical/              # cloned upstream, pinned to v0.16.1 (git submodule or vendored)
├── router/                # overlay tree deployed to the router
│   ├── etc/rustical/config.toml
│   ├── etc/init.d/rustical
│   ├── etc/init.d/dav-tls
│   └── etc/sysupgrade.conf.additions
├── out/                   # cross-compiled artifacts
└── deploy.sh              # adapted from router-nym/deploy.sh
```

### 1.2 Clone & verify RustiCal — **DONE (2026-09-04)**

Cloned to `~/router-dav/rustical`, pinned `v0.16.1` (git-clean, no submodules).
**Frontend assets ARE committed** — the rust-embed folder is
`crates/frontend/public/assets/` (`js/bundle.mjs`, `style.css`, `licenses.html`),
so no JS toolchain build is needed. Upstream's build tool is **deno** (not bun —
`crates/frontend/js-components/deno.json`, `deno task build`), irrelevant since
assets ship built-in. `.sqlx/` is committed → `SQLX_OFFLINE=true` works.

```sh
git clone https://github.com/lennart-k/rustical.git ~/router-dav/rustical
git -C ~/router-dav/rustical checkout v0.16.1   # pin the release
```

- Verify `crates/frontend/dist/` (or the `rust-embed` asset folder) contains a
  pre-built frontend. The upstream Dockerfile builds with no node stage, so it should
  be committed; **if not**: `cd crates/frontend && bun install && bun run build`
  before the cargo build.
- `SQLX_OFFLINE=true` is required for the build (`.sqlx/` is committed upstream).

### 1.3 Cross-compile RustiCal — **DONE (2026-09-04): `out/rustical`, static aarch64, 26 MiB stripped**

Reuse the zig-based toolchain wrappers from `router-nym/build/bin/` (`zig-musl-cc` and
its ar/ranlib/nm symlinks) exactly as `router-nym/scripts/build-rust.sh` does:

```sh
export CC_aarch64_unknown_linux_musl="$BIN/zig-musl-cc"
export AR_aarch64_unknown_linux_musl="$BIN/zig-musl-ar"
export RANLIB_aarch64_unknown_linux_musl="$BIN/zig-musl-ranlib"
export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_LINKER="$BIN/zig-musl-cc"
export SQLX_OFFLINE=true
cargo build --release --target aarch64-unknown-linux-musl --locked
```

Notes:
- ~~Only C dependency is sqlx's bundled SQLite~~ — **WRONG (found 2026-09-04):**
  the default build pulls **three** C dependencies — bundled SQLite (`sqlx-sqlite`),
  **vendored OpenSSL** (dav_push → `ece` + `web-push`, *not* feature-gated), and
  **aws-lc-sys** (rustls' default crypto provider, via `reqwest`/`openidconnect`/
  `web-push`). Host tools available: clang 22 (`/usr/lib/llvm/22/bin`), llvm-ar,
  cmake 4.3.4, perl, make, nasm, libclang, rust-lld (in the rustup toolchain).
- Disable nothing (RustiCal has no heavyweight default features; `debug`,
  `frontend-dev`, `opentelemetry` are opt-in).
- **Size-gate:** `strip` the binary immediately. Budget: `rustical` ≤ 35 MB stripped.
  If over: add `[profile.release] opt-level = "z", lto = true, panic = "abort",
  codegen-units = 1`, rebuild, re-strip; last resort `upx --lzma` (static musl ELF);
  if still over → surface to user with the USB-stick fallback option (deferred by
  decision, not off the table).

**Toolchain findings (2026-09-04) — RESOLVED: recipe D (clang + zig musl headers) works.**

Four cross-toolchain attempts (rustc 1.98.1, `SQLX_OFFLINE=true`,
`CARGO_PROFILE_RELEASE_STRIP=true`, aarch64-unknown-linux-musl):

| # | CC (C deps) | Linker | Result |
|---|---|---|---|
| A | zig 0.16 (`build/bin/zig-musl-cc`, router-nym pattern) | zig's own ELF linker | **All C deps compile under zig cc** (vendored OpenSSL, aws-lc-sys, SQLite, ring) ✓ — but the final link fails twice: rustc now emits `-Wl,--fix-cortex-a53-843419` which zig rejects (patched our wrapper copy to strip it — router-nym binaries already run on this RK3328 without the errata workaround), then `duplicate symbol: _start` (zig's musl crt1.o collides with rust's self-contained crt1.o). |
| B | clang 22 (upstream-Dockerfile recipe) | rust-lld + `-Clink-self-contained=yes` | aws-lc-sys (jitterentropy) fails: clang has **no musl sysroot** on this host → falls back to host glibc headers (`/system/index/include`): `__float128 is not supported on this target`. |
| C | zig (for its bundled musl headers) | rust-lld + rust's self-contained musl (upstream's link recipe) | C deps compile ✓; final link fails with **empty-name undefined symbols** referenced from zig-compiled `sqlite3.o` inside `liblibsqlite3_sys` (many sites: `sqlite3CreateIndex`, `sqlite3Select`, …). **Diagnosis CONFIRMED 2026-09-04 via `readelf -sW sqlite3.o`: 17 empty-name undefined SECTION LOCAL symbols** — a zig-cc object-emission quirk strict lld rejects. Zig-CC is unusable for lld-linked builds. |
| **D (WINNER — now the default `clang` route in `scripts/build-rust.sh`)** | **clang 22 + zig's musl include dirs** (`zig libc -target aarch64-linux-musl -includes`) | rust-lld + rust's self-contained musl | **WORKS.** Recipe: `CC_aarch64_unknown_linux_musl=clang`, `AR/RANLIB=llvm-ar/llvm-ranlib`, `CFLAGS_aarch64_unknown_linux_musl="-nostdinc -isystem <clang -print-resource-dir>/include -isystem <each zig musl include dir>"` (the resource dir is required — `-nostdinc` drops clang's builtin headers, breaking `arm_neon.h` in aws-lc), `CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_RUSTFLAGS="-Clink-self-contained=yes -Clinker=rust-lld"`. Validated first on a standalone bundled-rusqlite crate (0 empty-name symbols vs zig's 17), then full rustical build. |

Outcome of the D build (2026-09-04):
- `~/router-dav/out/rustical` — ELF 64-bit aarch64, **statically linked (no dynamic
  section), stripped, 27,839,344 bytes = 26 MiB** — within the 35 MiB budget, no
  size-mitigation steps (opt-level=z/LTO/upx) needed.
- Runtime check under `qemu-aarch64`: `rustical --version` → `rustical 0.16.1` ✓,
  `rustical gen-config` prints the default TOML ✓ (full smoke test is 1.5).
- Build time ~3.5 min from a clean aarch64 target dir.
- Note for 1.4: `dav-tls` should use the same recipe (`TOOLCHAIN=clang` is now the
  `build-rust.sh` default).

### 1.4 Build `dav-tls` (the one custom component) — **DONE (2026-09-04)**

Source complete in `~/router-dav/dav-tls/` and `cargo check` passes. Implementation
deviation from the spec above: **tokio + tokio-rustls (ring provider)** instead of
std::net + threads — the two-thread-with-Mutex design would block the server→client
direction during a blocking client read, stalling long-lived WebDAV-Push WebSockets;
tokio's split halves avoid that. aws-lc-rs deliberately not used in this crate.
Features implemented: repeatable `--listen`, `--upstream` (default wiring per §2.4:
`0.0.0.0:443` → `127.0.0.1:4000`), ALPN `http/1.1` only, SO_REUSEADDR (socket2),
`--user` privilege drop via libc getpwnam, graceful SIGTERM/SIGINT drain (≤5s,
1024-conn cap), TCP_NODELAY, TLS close_notify propagation. Own release profile
(opt-level="z", lto, strip). **Builds completed 2026-09-04** via the `clang`
default route of `scripts/build-rust.sh` (ring is the only C dep and compiles
under recipe D):

- `~/router-dav/out/dav-tls` — ELF 64-bit aarch64, **statically linked (no
  dynamic section), stripped, 1,279,960 bytes = 1.22 MiB** (well under the
  ~2–4 MB estimate; combined with rustical = ~27 MiB of the 42 MB overlay).
  Runtime check under `qemu-aarch64`: `--help` OK (full smoke test is 1.5).
- `~/router-dav/out/x86_64-unknown-linux-gnu/dav-tls` — host build for the
  1.5 smoke test, 1.4 MiB stripped, runs natively.
- Build times: ~42 s (aarch64, clean), ~49 s (x86_64, clean); x86_64 rustical
  rebuild from clean also completed (~6 min) as a 1.5 prerequisite.

Spec (as originally designed; implemented above with the tokio deviation):

- Args: `--listen 0.0.0.0:443` (repeatable), `--upstream 127.0.0.1:4000`,
  `--cert /etc/rustical/tls/fullchain.pem`, `--key /etc/rustical/tls/key.pem`,
  optional `--user nobody` (drop privileges after bind).
- Implementation: `rustls` 0.23 (pure Rust, no CC needed) + `tokio` (or plain
  `std::net` + threads — simpler is better here): accept TCP, perform TLS handshake
  with ALPN restricted to `http/1.1`, then **splice raw bytes** bidirectionally to
  the upstream socket. No HTTP parsing. `SO_REUSEADDR`, per-connection handling,
  graceful SIGTERM shutdown (for procd restart during cert deploys).
- Cert reload happens on service restart (renewal flow restarts the service).
- HTTP/2 note: no `h2` in ALPN → every CalDAV client falls back to HTTP/1.1. WebDAV
  Push WebSockets pass through untouched (they're just HTTP upgrades).
- Also buildable for `x86_64-unknown-linux-gnu` for local testing.
- Expected size: ~2–4 MB stripped.

### 1.5 Local smoke test (x86_64, dev machine) — NOT STARTED (unblocked 2026-09-04: both x86_64 binaries built)

```sh
cargo build --release --target x86_64-unknown-linux-gnu   # both crates
# run rustical with a temp db:
RUSTICAL_DATA_STORE__SQLITE__DB_URL=file:/tmp/omnical-test.db.sqlite3 \
  ./rustical serve &
# self-signed cert via openssl for dav-tls, then:
curl -vk https://127.0.0.1/.well-known/caldav -u 'smoketest:token'
curl -vk -X PROPFIND -H 'Depth: 1' https://127.0.0.1/caldav/ -u 'smoketest:token'
curl -vk -X OPTIONS https://127.0.0.1/caldav/
```

**Gate:** `OPTIONS` advertises DAV: 1, calendar-access, addressbook; `PROPFIND` returns
multistatus; `/.well-known/*` redirect correctly through dav-tls; PUT+REPORT of a test
`.ics` round-trips. Then proceed.

---

## Phase 2 — Install on Router

**Deliverable: both services installed, enabled, and running on the router
(HTTP-only, LAN-addressable; TLS wired in Phase 3.**

### 2.1 Files to deploy (via `deploy.sh`, modeled on router-nym)

| Artifact (dev) | Router target | Mode |
|---|---|---|
| `out/rustical` | `/usr/sbin/rustical` | 0755 |
| `out/dav-tls` | `/usr/sbin/dav-tls` | 0755 |
| `router/etc/rustical/config.toml` | `/etc/rustical/config.toml` | 0600 |
| `router/etc/init.d/rustical` | `/etc/init.d/rustical` | 0755 |
| `router/etc/init.d/dav-tls` | `/etc/init.d/dav-tls` | 0755 |

### 2.2 Router-side preparation (idempotent, part of deploy.sh)

```sh
# persistent data dir (overlay — NOT /var!)
mkdir -p /usr/local/share/rustical /etc/rustical/tls
# online-backup tool for the SQLite DB
opkg update && opkg install sqlite3-cli
```

### 2.3 `/etc/rustical/config.toml` (authoritative shape comes from
`rustical gen-config`; the fields we set:)

```toml
[http]
bind = "127.0.0.1:4000"          # only dav-tls talks to it
host = "0115d8cf.duckdns.org"     # used for URL generation in frontend/DAV responses
payload_limit_mb = 32

[data_store.sqlite]
db_url = "file:/usr/local/share/rustical/db.sqlite3"

[dav_push]
# leave defaults at first; enable WebSocket/WebPush transport after Phase 7 tests

[nextcloud_login]
# enabled by default — required for DAVx5 login flow
```

(Equivalent env vars if preferred: `RUSTICAL_HTTP__BIND=127.0.0.1:4000`,
`RUSTICAL_DATA_STORE__SQLITE__DB_URL=file:…`.)

### 2.4 procd init scripts

`/etc/init.d/rustical`:
- `START=95`, `STOP=10`, `USE_PROCD=1`
- `procd_set_param command /usr/sbin/rustical serve`
- `procd_set_param respawn` (threshold 3600, timeout 5, retry 0)
- `procd_set_param limits memory="256 MB"` (hard cap well under free RAM)
- `procd_set_param stdout 1` / `stderr 1` → syslog (logread)
- optional hardening later: `procd_set_param jail` (ujail) after functional verification

`/etc/init.d/dav-tls`:
- `START=96` (after rustical), similar respawn/limits
- `procd_set_param command /usr/sbin/dav-tls --listen 0.0.0.0:443 --upstream 127.0.0.1:4000 --cert /etc/rustical/tls/fullchain.pem --key /etc/rustical/tls/key.pem`
- only enabled/started in Phase 3 when certs exist (validate: `ls /etc/rustical/tls/` in
  `start_service`, refuse to start with log message if missing)

### 2.5 Free port 443 (move uhttpd HTTPS to 8443)

```sh
uci set uhttpd.main.listen_https='192.168.10.1:8443'
uci add_list uhttpd.main.listen_https='10.10.10.1:8443'
uci add_list uhttpd.main.listen_https='[fd1e:5c71:1ee8::1]:8443'
# remove the old :443 entries (uci delete_list), then:
uci commit uhttpd && /etc/init.d/uhttpd restart
```

LuCI remains at `https://192.168.10.1:8443` (the `:80` listener still redirects).
Document this change — it's the only visible LuCI-side modification of the whole plan.

### 2.6 Persistence across sysupgrade

Append to `/etc/sysupgrade.conf`:

```
/etc/rustical
/usr/local/share/rustical
```

(`/etc/init.d/*` files are preserved by default as conffiles; `/usr/sbin/rustical` and
`/usr/sbin/dav-tls` are **not** — after any sysupgrade, re-run
`~/router-dav/deploy.sh`. Document in the project README.)

### 2.7 Optional: duckdns updater cron (only if Phase 0 step 4 found nothing)

```cron
*/5 * * * * curl -s "https://www.duckdns.org/update?domains=0115d8cf&token=<TOKEN>&ip=" >/dev/null
```

(Token inserted at deploy time, chmod 600 crontab — never logged.)

### 2.8 Verify

```sh
/etc/init.d/rustical enable && /etc/init.d/rustical start
/usr/sbin/rustical health
netstat -tlnp | grep 4000          # bound to 127.0.0.1:4000 only
curl -s http://127.0.0.1:4000/.well-known/caldav -o /dev/null -w '%{http_code}\n'
```

**Gate:** service healthy, DB created at
`/usr/local/share/rustical/db.sqlite3`, survives a reboot (`reboot`, re-check).

---

## Phase 3 — TLS Certificate Automation

**Deliverable: valid Let's Encrypt cert for `0115d8cf.duckdns.org` live on
`0.0.0.0:443`, with unattended renewal.**

All on the **dev machine** (full curl/openssl tooling; router stays thin):

```sh
# 1. Install acme.sh (once)
curl https://get.acme.sh | sh -s email=<your-email>

# 2. Issue via DuckDNS DNS-01 (TXT record automation; no port 80 needed)
export DuckDNS_Token="$(pass show secrets/duckdns/0115d8cf.duckdns.org/token)"
~/.acme.sh/acme.sh --issue --dns dns_duckdns -d 0115d8cf.duckdns.org

# 3. Install-cert with a deploy hook that pushes to the router and reloads dav-tls
~/.acme.sh/acme.sh --install-cert -d 0115d8cf.duckdns.org \
  --fullchain-file ~/router-dav/out/tls/fullchain.pem \
  --key-file       ~/router-dav/out/tls/key.pem \
  --reloadcmd "scp ~/router-dav/out/tls/{fullchain.pem,key.pem} router:/etc/rustical/tls/ && \
               ssh router 'chmod 600 /etc/rustical/tls/*; /etc/init.d/dav-tls restart'"
```

Then on the router: `/etc/init.d/dav-tls enable && /etc/init.d/dav-tls start`.

Renewal is automatic (~60-day cycle, acme.sh cron). Key hygiene: key exists only on
dev machine + router, mode 0600, never in the git repo.

**Verify:**
```sh
openssl s_client -connect 0115d8cf.duckdns.org:443 -servername 0115d8cf.duckdns.org </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates -issuer     # Let's Encrypt, valid dates
curl -sI https://0115d8cf.duckdns.org/.well-known/caldav   # from an EXTERNAL network
```

---

## Phase 4 — Firewall & Reachability

**Deliverable: 443 reachable from WAN through both NATs, blocked otherwise.**

### 4.1 libreCMC firewall rule (follow the Allow-DNSv4 rule pattern)

```sh
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Dav-TLS'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='443'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='any'
uci commit firewall && /etc/init.d/firewall restart
```

(Also verify the IPv6 story: if a global GUA ever lands on the WAN interface,
the same rule covers it via `family='any'`; the ULA is LAN-internal only.)

### 4.2 Upstream port-forward

Done once by the user in Phase 0 (TCP 443 → 192.168.1.21:443). If it cannot be
configured, the fallback is a TCP relay on one of the existing wireguard VMs
(`wg-vm1`/`wg-vm3`) — noted here, not planned, since the port-forward path is
available.

### 4.3 Reachability matrix to verify

| From | To | Expected |
|---|---|---|
| LAN (192.168.10.x) | https://192.168.10.1:443 | 200/301, valid cert |
| Upstream LAN (192.168.1.x, incl. dev machine) | https://192.168.1.21:443 | 200/301, valid cert |
| Internet (phone LTE / a VM) | https://0115d8cf.duckdns.org:443 | 200/301, valid Let's Encrypt chain |
| Any | port 4000 | unreachable (bound to 127.0.0.1) |

---

## Phase 5 — Users, Groups, Collections (any email domain)

**Deliverable: 6 principals + sharing groups + per-client app tokens + seeded
collections. This is where "works across any email domain" materializes.**

### 5.1 Create principals (user id = full email address)

RustiCal user ids are domain-agnostic (only `:` and `$` are forbidden), so accounts
are literally the email addresses:

```sh
ssh router '/usr/sbin/rustical principals --help'   # confirm CLI shape on the pinned version
for u in \
  burningserenity@gmail.com \
  nfcarlton@gmail.com \
  nfcalaway@gmail.com \
  nicholas@hawksnestsoftware.com \
  nicholas@carltonaudio.com \
  nfcalaway@novo-ordo.com ; do
    ssh router "/usr/sbin/rustical principals add '$u'"   # set an argon2 frontend password when prompted
done
```

(Exact subcommand syntax per the pinned release's `principals --help`; the docs show
`rustical principals` as the interactive entry point. Any future email address on any
domain is added the same way — nothing is domain-coupled.)

### 5.2 Groups & memberships (cross-identity / cross-domain sharing)

```sh
ssh router "/usr/sbin/rustical principals add group.family --principal-type group"
ssh router "/usr/sbin/rustical membership add <owner> group.family <member>"
```

Notes from upstream docs:
- Membership = full read/write access to the group principal's collections.
- Clients that can't auto-discover member principals authenticate as
  `user$group` (principal impersonation) or add
  `/caldav-compat/principal/group.family` manually (Apple).

### 5.3 App tokens (DAV credentials)

- Log into the frontend (`https://0115d8cf.duckdns.org`) with each identity's
  frontend password; generate **one app token per client** (e.g.
  `vdirsyncer`, `davx5-nicholas`, `tb-carltonaudio`, `apple-nfcalaway`).
- Store tokens in `pass` (`secrets/omnical/<identity>/<client>`).
- Frontend passwords (argon2) are only for the web UI; DAV never sees them.

### 5.4 Collections

- RustiCal creates default calendar + addressbook per principal on first use;
  create explicitly via frontend (or MKCOL) if you want named ones (e.g.
  `personal`, `work-hawksnest`, `tasks`).
- Task lists are ordinary CalDAV calendars that accept VTODO (Tasks.org/Thunderbird
  treat them as task lists) — verify `calendar-collection` supports both VEVENT and
  VTODO component-set during the Phase 14 tests.
- Birthdays: RustiCal auto-exposes
  `/caldav/principal/<id>/_birthdays_<addressbook_id>` — bonus, no config needed.

---

## Phase 6 — Sync Hub for Any Provider (vdirsyncer)

**Deliverable: the router is the convergence hub for all identities regardless of
which provider hosts them.**

### 6.1 Extend `~/.config/vdirsyncer/config`

Keep the existing Google↔vdir pairs untouched; add RustiCal pairs per identity.
Example for one identity (repeat pattern per identity):

```ini
[pair rustical_calendar_nfcarlton]
a = "nfcarlton_calendar_local"          # existing filesystem storage (~/.calendars/google/nfcarlton@gmail.com)
b = "rustical_calendar_nfcarlton_remote"
collections = ["from a"]                 # or pin collection hrefs after discover
conflict_resolution = "b wins"           # during migration; revisit after verification

[storage rustical_calendar_nfcarlton_remote]
type = "caldav"
url = "https://0115d8cf.duckdns.org/"
username = "nfcarlton@gmail.com"
password = "<app token from pass>"

[pair rustical_contacts_nfcarlton]
a = "nfcarlton_contacts_local"
b = "rustical_contacts_nfcarlton_remote"
collections = ["from a"]

[storage rustical_contacts_nfcarlton_remote]
type = "carddav"
url = "https://0115d8cf.duckdns.org/"
username = "nfcarlton@gmail.com"
password = "<app token from pass>"
```

(App tokens can be referenced via vdirsyncer's password-command with `pass` to keep
them out of the config file.)

### 6.2 Run & schedule

```sh
vdirsyncer discover && vdirsyncer metasync && vdirsyncer sync
```

Add to cron/timer if no vdirsyncer scheduler exists today (this plan does not assume
one; check `crontab -l` / systemd timers first). Suggest every 15 min during
migration, then hourly.

### 6.3 Migration & khal/khard repointing

1. Initial sync: vdirs (Google truth) → RustiCal collections.
2. Import strays: `~/.calendars/local/*.ics` and `~/.contacts/contacts.csv`
   (convert to vCard, e.g. with `khard` tooling) via RustiCal's frontend import.
3. After the verification matrix passes, flip conflict resolution and/or repoint
   khal/khard to the RustiCal-side vdir trees so the router becomes canonical:
   ```ini
   [[omnical_nfcarlton]]
   path = ~/.calendars/omnical/nfcarlton@gmail.com/
   type = calendar
   ```

---

## Phase 7 — End-User Clients & Cross-Domain Invitations

**Deliverable: every approved client class working against the router.**

| Client | Setup | Notes |
|---|---|---|
| **DAVx5** | Login via **Nextcloud flow**: URL `https://0115d8cf.duckdns.org`, use frontend login → generates app token automatically; collections auto-discovered | WebDAV Push gives near-instant sync (enable dav_push transport per docs) |
| **Tasks.org** | Add CalDAV account via DAVx5; task lists appear as calendars with VTODO support | The "tasks" requirement of this project |
| **Apple Calendar** | Use `/caldav-compat` paths (Apple mishandles multi-home `calendar-home-set`), or install the **configuration profile** generated by RustiCal's frontend (token section) | Requires real TLS (satisfied by LE) |
| **Apple Contacts** | CardDAV account, server `0115d8cf.duckdns.org`, user id + app token, path `/carddav` | |
| **Thunderbird** | New Account → Calendar → On the Network → root URL `https://0115d8cf.duckdns.org` + app token; same for CardDAV | Group calendars discovered properly |
| **khal / khard** | via vdirsyncer hub (Phase 6) | CLI stays exactly as today, now backed by the router |

**Cross-domain invitations (iMIP):** RustiCal does not implement RFC 6638
server-side scheduling. Invitations to attendees on **any** domain are sent
client-side: Apple Calendar and Thunderbird compose an `.ics` iTIP message and email
it via whichever SMTP identity the user picks — attendee replies flow back the same
way and update the event. This works with every email domain by construction; it is a
documented product limitation that the *server itself* doesn't also maintain an
inbox/outbox/scheduling queue (see §17 for the future custom extension).

---

## Phase 8 — Backups & Maintenance

**Deliverable: nightly verified backups, health monitoring, sysupgrade runbook.**

### 8.1 Nightly pull-based backup (dev machine cron; reuses `router` SSH alias)

```sh
# 02:30 local, daily
ssh router 'sqlite3 /usr/local/share/rustical/db.sqlite3 ".backup /tmp/omnical-bu.db" \
            && tar -C /tmp -czf - omnical-bu.db /etc/rustical' \
  > ~/backups/omnical/$(date +\%F).tar.gz && rm -f ~/backups/omnical/*.tar.gz.(older than 30d)
```

- SQLite `.backup` is online/safe (no service stop).
- Retention: 30 days local; optionally rsync a weekly copy to a VM (`wg-vm1`/`wg-vm3`).
- **Restore drill:** test-restore the DB into a scratch RustiCal instance on the dev
  machine at least once (this is part of the verification matrix).

### 8.2 Monitoring

- procd respawn covers crashes; add `rustical health` to the existing
  `screech-watchdog`-style pattern if desired (every 5 min cron on router).
- `logread -e rustical | tail` for triage; keep `tracing` at warn/info.

### 8.3 Sysupgrade runbook (router firmware updates)

1. Backups are current (check nightly artifact exists).
2. `/etc/sysupgrade.conf` already preserves `/etc/rustical` + data dir (Phase 2.6).
3. After sysupgrade: re-run `~/router-dav/deploy.sh` (re-pushes binaries + init
   scripts), verify `rustical health` + external reachability.

### 8.4 Storage watch

`df -h /` weekly expectation: overlay stays ≥ 5 MB free (binary + DB + certs are the
only consumers). Add to backup cron: `ssh router df -h / | tail -1` in the log.

---

## 14. Verification Matrix

Run in order; each row is a gate. Do not migrate khal/khard canonicality (Phase 6.3)
before row 10 passes.

| # | Test | Command / Method | Expected |
|---|---|---|---|
| 1 | Service health | `ssh router /usr/sbin/rustical health` | healthy |
| 2 | Bound correctly | `netstat -tlnp` on router | rustical on 127.0.0.1:4000; dav-tls on 0.0.0.0:443; uhttpd on 8443 |
| 3 | TLS chain (external) | `openssl s_client` + `curl -sI https://0115d8cf.duckdns.org/.well-known/caldav` from LTE/VM | LE issuer, 200/301 |
| 4 | DAV headers | `curl -X OPTIONS -u user:token https://…/caldav/` | `DAV: 1, calendar-access` (and `addressbook` on /carddav/) |
| 5 | PROPFIND | `curl -X PROPFIND -H 'Depth: 1' -u … https://…/caldav/` | 207 multistatus, principal home-set lists collections |
| 6 | calendar-query REPORT | curl REPORT with time-range filter on a test calendar | 207, only matching VEVENTs |
| 7 | VTODO | PUT a VTODO; REPORT with `calendar-query` comp-filter VTODO | 207; Tasks.org sees the task |
| 8 | addressbook-query REPORT | curl REPORT on `/carddav/…` | 207, vCards match filter |
| 9 | Well-known | `curl -sI https://…/.well-known/caldav` and `…/carddav` | 30x to correct roots (client autodiscovery path) |
| 10 | vdirsyncer | `vdirsyncer discover && sync` (new pairs) | clean two-way sync incl. ETags; no items lost (diff before/after) |
| 11 | khal / khard | create/edit event & contact via CLI in the omnical vdirs | appears on server (verify via curl REPORT) and on other clients |
| 12 | DAVx5 + Tasks.org | Android account; create/edit event, contact, task | syncs both directions; WebDAV Push = near-instant when enabled |
| 13 | Thunderbird | calendar + cardbook/tasks accounts at root URL | discovers all own collections + group calendars |
| 14 | Apple Calendar/Contacts | caldav-compat path or config profile | account works; create/edit round-trips; contacts sync |
| 15 | Sharing | group collection visible to member identities (all 4 domains) | cross-domain share works via membership |
| 16 | iMIP invitation | Thunderbird invite to an external address on a different domain; attendee accepts | reply updates organizer's event |
| 17 | Reboot persistence | `reboot` router; re-check services + data | everything returns; DB intact (proves not-in-/var) |
| 18 | Restore drill | restore nightly backup tar into scratch instance on dev machine | DB opens, data present |
| 19 | Firewall hygiene | nmap 4000 from LAN/WAN; port-scan WAN IP | 4000 closed; only 22/443(+53) exposed |

---

## 15. Rollback / Recovery

Every step is reversible; nothing destructive is done to the router.

- **Services:** `/etc/init.d/dav-tls disable && stop`; `/etc/init.d/rustical disable && stop`.
- **Files:** `rm /usr/sbin/{rustical,dav-tls} /etc/init.d/{rustical,dav-tls}`; `rm -rf /etc/rustical` (keep a copy of the DB first).
- **uhttpd:** restore `listen_https` to `:443` entries; `uci commit uhttpd; restart`.
- **Firewall:** delete `Allow-Dav-TLS` rule; `uci commit firewall; restart`.
- **Upstream:** remove port-forward on TP-Link.
- **Data:** the SQLite DB is a single file — copy it off with the Phase 8 procedure at any time; restoring = stop service, replace file, start.
- **Google side is never modified by this plan** — original vdirs/Google pairs stay intact until you explicitly migrate canonicality (Phase 6.3), so the pre-Omnical state is always recoverable.

---

## 16. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| RustiCal binary > 35 MB stripped (42 MB overlay budget) | Medium | Blocks internal-flash decision | strip → size-opt profile → LTO/panic=abort → upx --lzma → escalate USB-stick decision to user |
| RustiCal is young software; protocol bugs with picky clients (Apple) | Medium | Client-specific breakage | Pin 0.16.1; full verification matrix incl. Apple before cutover; upstream has tested-client list; issues tracker active |
| No RFC 6638 scheduling | Certain (accepted) | No server-side invitations | Client-side iMIP covers invitations; future extension (§17) |
| RFC 6578 sync-token partial → slower large-collection syncs | Low | Performance only | Collections here are personal-scale (thousands of items max); acceptable |
| Frontend dist not committed upstream → extra build step | Low | Build friction | bun is installed; check at clone (Phase 1.2) |
| Dynamic IP / duckdns updater gap | Low | Brief outages on IP change | Verify updater in Phase 0; add router cron if needed (2.7) |
| Upstream port-forward impossible on TP-Link | Low | No public exposure | VM wireguard relay fallback documented (§4.2) |
| Flash wear from SQLite on f2fs | Low | Long-term hardware | Personal write volumes are tiny; SQLite WAL; nightly `.backup` keeps data safe regardless |
| Zig/rustc version drift breaks cross-build | Low | Build breakage | Pin toolchain in rustup/notes; router-nym proved zig 0.16 + musl target works |
| f2fs overlay corruption / sysupgrade data loss | Low | Data loss | sysupgrade.conf entries + nightly off-router backups + restore drill (rows 17–18) |

---

## 17. Future / Optional Enhancements

1. **RFC 6764 autodiscovery SRV/TXT records** at DNS providers for domains you control
   (`hawksnestsoftware.com`, `carltonaudio.com`, `novo-ordo.com`):
   `_caldavs._tcp.<domain> SRV 0 1 443 0115d8cf.duckdns.org`,
   `_carddavs._tcp.<domain> SRV …`, `TXT "path=/caldav"` — enables
   "just type your email address" account discovery in Apple/DAVx5/Thunderbird for
   those domains. (gmail.com etc. obviously can't be annotated — direct URL login
   still works for any address.)
2. **Server-side scheduling (RFC 6638) / iMIP gateway**: a small custom Rust service
   (same deployment pattern) that watches an IMAP inbox for `text/calendar` replies
   and injects them / sends invitations via SMTP — would make invitations fully
   server-driven while remaining domain-agnostic. Substantial; only if client-side
   iMIP proves insufficient.
3. **WebDAV Push transports** (WebSocket/WebPush) tuning for instant DAVx5 sync
   (RustiCal ships support; configure in `dav_push` after Phase 7).
4. **IPv6**: publish AAAA on duckdns once a stable GUA exists on WAN; same firewall
   rule already covers it (`family='any'`).
5. **Ujail/seccomp hardening** of both services via procd jail params, once stable.
6. **Dedicated VM off-site replica** (via existing wg-vm tunnels) running the same
   binaries — a warm standby with hourly DB copy.

---

## Appendix A — RustiCal Reference Notes

- Version pinned: **0.16.1** · repo: `https://github.com/lennart-k/rustical` · docs:
  `https://lennart-k.github.io/rustical/` · license: AGPL-3.0-or-later
- Subcommands: `serve` · `principals` (users/groups) · `membership` · `gen-config`
  (prints full default config — use it to confirm exact TOML keys before editing) ·
  `health`
- URL map: `/.well-known/caldav|carddav` → `/caldav`, `/caldav-compat`,
  `/carddav`; principals at `/caldav/principal/<user_id>`; group impersonation user
  `user$group`; birthdays auto-collection
  `_birthdays_<addressbook_id>`
- Auth model: frontend login = argon2 password; DAV/CalDAV/CardDAV = HTTP Basic with
  **user id + app token** (pbkdf2-verified random tokens, per-client, generated in web
  UI; DAVx5 can self-provision via Nextcloud login flow)
- User id constraints: no `:` or `$` characters — **full email addresses of any domain
  are valid**
- Storage: single SQLite DB (sqlx, bundled); needs writable `/tmp`; CA certs at
  `/etc/ssl/certs` (present on router)
- Upstream config env-var mapping example (Docker):
  `RUSTICAL_DATA_STORE__SQLITE__DB_URL=/var/lib/rustical/db.sqlite3` — **do not use
  /var on this router (tmpfs)**
- Not implemented: RFC 6638 scheduling (client-side iMIP instead); RFC 6578
  sync-token partial
- Features present: WebDAV Push (web-push), Nextcloud login flow, Apple configuration
  profiles, group-based sharing, deleted-calendar recovery, frontend import/export,
  OIDC (optional; not used in this plan)

## Appendix B — Key Commands & File Paths

**Router paths**
- `/usr/sbin/rustical`, `/usr/sbin/dav-tls` — binaries (re-deploy after sysupgrade)
- `/etc/rustical/config.toml` — server config (0600)
- `/etc/rustical/tls/{fullchain.pem,key.pem}` — LE cert/key (0600)
- `/usr/local/share/rustical/db.sqlite3` — THE database (persistent overlay)
- `/etc/init.d/rustical`, `/etc/init.d/dav-tls` — procd services
- `/etc/sysupgrade.conf` — includes `/etc/rustical`, `/usr/local/share/rustical`
- `/etc/crontabs/root` — existing cron home (backups run from dev machine instead)

**Dev machine paths**
- `~/router-dav/` — this project (scripts, dav-tls source, deploy.sh, out/)
- `~/router-nym/build/bin/zig-musl-cc` (+ ar/ranlib/nm symlinks) — reused wrappers
- `~/.acme.sh/` — acme.sh home, duckdns DNS-01 + install-cert deploy hook
- `~/.config/vdirsyncer/config` — extended with RustiCal pairs (Phase 6)
- `~/backups/omnical/` — nightly backup artifacts
- `pass` entries: `secrets/duckdns/0115d8cf.duckdns.org/token`, `secrets/omnical/<identity>/<client>`

**One-liners**
```sh
ssh router                                        # shell on router
ssh router '/usr/sbin/rustical health'            # health check
ssh router 'logread | grep -i rustical | tail'    # logs
ssh router 'sqlite3 /usr/local/share/rustical/db.sqlite3 ".backup /tmp/bu.db"'  # hot backup
curl -sI https://0115d8cf.duckdns.org/.well-known/caldav                        # liveness
vdirsyncer discover && vdirsyncer sync            # hub sync
```

## Appendix C — Relevant RFCs

| RFC | What it defines | Status in this deployment |
|---|---|---|
| 4918 | WebDAV core (PROPFIND, MKCOL, locks) | Implemented by RustiCal |
| 3253 | Versioning / REPORT method | Implemented (REPORT used heavily by clients) |
| 4791 | CalDAV | Implemented (VEVENT + VTODO → tasks) |
| 5545 | iCalendar (VEVENT/VTODO/VJOURNAL, iTIP) | Object format; VTODO = the tasks requirement |
| 6352 | CardDAV | Implemented |
| 6578 | Collection synchronization (sync-token) | Partial in RustiCal; clients fall back to full sync |
| 6638 | CalDAV scheduling (server-side invitations) | **Not implemented** → client-side iMIP (RFC 6047 iTIP-in-IMAP; email invitations from any domain) |
| 6764 | DAV service discovery (SRV/TXT + /.well-known) | `/.well-known` served by RustiCal; SRV records optional (§17.1) |
| 8291/8030 | WebPush | Used by RustiCal's WebDAV Push (optional enable) |

---

*End of plan. Execute phases in order; gates are called out at Phases 1, 2, 3, and the
verification matrix. Escalate to the user if any size-gate, port-forward, or Apple
client verification fails.*
